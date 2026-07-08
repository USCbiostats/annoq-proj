# Component Repositories

Deep dive on each AnnoQ repository: what it does, how it's built, its inputs/outputs, and
things to watch out for. The **core pipeline** repos (stages 1–4) follow the data-flow order;
after them come the **historical API**, the **client libraries**, and **related applications**.

- Core pipeline: [annoq-data-builder](#1-annoq-data-builder) → [annoq-database](#2-annoq-database) → [annoq-api-v2](#3-annoq-api-v2) → [annoq-site](#4-annoq-site)
- Historical API: [annoq-api](#annoq-api-deprecated)
- Clients: [annoq-py](#annoq-py), [AnnoQR](#annoqr)
- Related apps: [Annoq_Overrepr_Workflow (SNPWay)](#annoq_overrepr_workflow--snpway), [annoq-site-v2](#annoq-site-v2)

---

## 1. annoq-data-builder

<https://github.com/USCbiostats/annoq-data-builder>

**Purpose:** Build the annotation data that powers AnnoQ — annotate human genomic variants
(SNVs/indels) and generate the artifacts the rest of the pipeline consumes.

**Stack:** Shell (~84%), Jupyter Notebook, Java, Python. Runs on HPC via SLURM.

**Structure (key paths):**
- `wgsa_095_pipeline/` — WGSA v0.95 pipeline integrating ANNOVAR, VEP, SnpEff
  - `config.py` — generates per-file configurations
  - `sbatch.py` — creates SLURM batch files
  - `sbatch.temp` — job template
- `java_wgsa_add/` — Java module adding PANTHER protein-function + enhancer annotations
- `tools/`
  - `panther_gene_extractor.py` — retrieves PANTHER API annotations
  - `annotation_tree_gen.py` — generates JSON + Elasticsearch mappings
  - `mappings_data_type_gen.py` — produces pickle files for DB indexing
- `resources/` — large annotation reference files

**Inputs:** VCF files, PANTHER annotation JSON, protein FASTA, UniProt ID-mapping files.
**Outputs:** annotated VCF results, `annoq_mapping.json`, `doc_type.pkl`, annotation tree JSON.

**Gotchas:** SLURM/HPC-bound — most scripts assume a cluster environment. Large reference
files may not be in-repo. Mapping/tree artifacts are contracts consumed downstream — changing
field names here breaks stages 2–4.

---

## 2. annoq-database

<https://github.com/USCbiostats/annoq-database>

**Purpose:** Convert annotated VCF/TSV into JSON and bulk-index it into Elasticsearch.

**Stack:** Python (~50%), Bash (~49%), Elasticsearch 8.5.x, HPC/SLURM. Kibana + Logstash configs.

**Structure (key paths):**
- `scripts/run_jobs.sh` — conversion workflow (VCF/TSV → JSON, adds unique id)
- `run_es_job.sh` — indexing entry point
- `src/reinit` — create/recreate ES indices with custom mappings
- `src/index_es_json` — bulk-load JSON into indices (parallel / streaming / standard)
- `data/` — `doc_type.pkl`, `annoq_mapping.json` (from stage 1)
- `logstash/`, `kibana/config/` — ops tooling

**Inputs:** annotated VCF/TSV + mapping/pickle artifacts from data-builder.
**Outputs:** populated Elasticsearch indices (the source of truth for the API).

**Gotchas:** ES version-sensitive (tested on 8.5.x). Unique id = chrom+pos+ref+alt — collisions
or format drift cause dedup/overwrite issues. Index mappings must match what data-builder emits
and what api-v2 expects.

---

## 3. annoq-api-v2

<https://github.com/USCbiostats/annoq-api-v2>

**Purpose:** GraphQL API over the Elasticsearch database, exposing 500+ annotation attributes.
This is the **current API** and the one all clients should target; it replaced the original
[annoq-api](#annoq-api-deprecated) (Flask/REST).

**Stack:** Python 3.11+, FastAPI, Strawberry GraphQL, Docker / Docker Compose.

**Structure (key paths):**
- `src/` — application source (schema, resolvers, ES client)
- `scripts/class_generators/` — dynamic model generation (`datamodel-codegen`)
- `data/` — `anno_tree.json`, `api_mapping_anno_tree.json`
- `sample_data/` — sample datasets for local setup
- `test/` — pytest suite
- `docker-compose.yaml`, `requirements.txt`, `playground.ipynb`

**Inputs:** Elasticsearch indices; annotation-tree config.
**Outputs:** GraphQL endpoint. Runs as **two instances**, one per stack:
`https://api-v2.annoq.org` (HRC / production, `main`) and `https://api-v2.topmed.annoq.org`
(TOPMed / beta, TopMed branch) — each backed by its own database/ES instance.

**Gotchas:** GraphQL types are **generated** from the ES schema, not hand-written — a field
must exist in the index before it can be exposed. Regenerate types after schema changes.
Local setup pairs with annoq-database sample data. Python 3.11+ required.

---

## 4. annoq-site

<https://github.com/USCbiostats/annoq-site>

**Purpose:** Web UI for AnnoQ — interactive annotation browsing/querying with integrated docs.

**Stack:** Angular 9, TypeScript (~62%), HTML, SCSS.

**Structure (key paths):**
- `src/` — application source
- `e2e/` — end-to-end tests
- `scripts/` — build/utility scripts
- `metadata/` — configuration data
- `graphql_codegen.ts` — GraphQL client code generation against the api-v2 schema

**Inputs:** the api-v2 GraphQL schema/endpoint.
**Outputs:** the public site at <http://annoq.org/> (dev: `npm install` + `ng serve`, `localhost:4205`).

**Deployments & datasets:** annoq-site is deployed as part of two **parallel stacks** (see
[architecture.md](architecture.md#parallel-deployment-stacks-hrc--topmed)). Each stack has its own
branch and points at its **own api-v2 instance**, which in turn queries its **own database/ES
instance**.

| Deployment | URL | Dataset | Branch | Talks to |
|------------|-----|---------|--------|----------|
| Production | <https://annoq.org> | **HRC r1.1** (Haplotype Reference Consortium) | `main` | `api-v2.annoq.org` → database instance A |
| Beta | <https://topmed.annoq.org> | **TOPMed: Freeze 8** | TopMed branch | `api-v2.topmed.annoq.org` → database instance B |

Both deployments currently serve **SNP data only — no indels** — even though the upstream
pipeline (data-builder → database) can process both SNVs and indels. The two stacks share the
codebase's history but run on **separate infrastructure** (distinct api-v2 and database/ES
instances) and are released independently, so they can carry different data/schema versions at
any moment. The branch split extends across the stack — **data-builder, api-v2, and annoq-site
each have a `main` line and a TopMed line**.

**Gotchas:** Angular 9 is dated — mind Node/CLI version compatibility. Client types are
codegen'd from the live/target GraphQL schema; regenerate after api-v2 changes. Keep the
target API URL (dev vs prod) correct in the environment config. A next-generation replacement,
[annoq-site-v2](#annoq-site-v2), is in development but not yet released.

---

## annoq-api (deprecated)

<https://github.com/USCbiostats/annoq-api>

> ⚠️ **Deprecated — replaced by [annoq-api-v2](#3-annoq-api-v2).** Documented here for historical
> context. New work should target api-v2; only touch this repo to support legacy consumers.

**Purpose:** The original AnnoQ API — a Flask wrapper providing REST access to Elasticsearch-based
annotation search.

**Stack:** Python (Flask, ~82%), Shell.

**Notes:** REST (not GraphQL). Configurable Elasticsearch credentials; query scripts for region
searches, field filtering, pagination, and schema inspection; responses include hit counts and
pagination. Superseded by api-v2's GraphQL interface and dynamic type generation.

---

## annoq-py

<https://github.com/USCbiostats/annoq-py>

**Purpose:** Python client library for programmatically accessing SNP data from the AnnoQ API and
the workflows built on top of it (SNPWay).

**Stack:** Python 3.7+.

**What it does:** Thin wrapper client that abstracts API calls — SNP attribute discovery; querying
by chromosome region, RSID list, or gene; SNP counting for result estimation; and SNPWay workflow
integration (gene mapping + PANTHER overrepresentation). Supports multiple input modes (VCF text,
chromosome ranges, RSID lists, gene identifiers) with configurable field selection.

**Inputs:** user SNP/variant queries. **Outputs:** structured annotation results (Python objects).

**Data source:** api-v2 — annoq-py retrieves its SNP/annotation data from **annoq-api-v2**.

**Gotchas:** respects API limits — **10,000-result pagination cap** for standard queries and a
**20-field maximum per request**. Behavior tracks the API contract; changes to api-v2's schema or
limits can affect this client.

---

## AnnoQR

<https://github.com/USCbiostats/AnnoQR>

**Purpose:** R client package for programmatically accessing SNP data from the AnnoQ API and the
workflows built on top of it (SNPWay). The R counterpart to annoq-py.

**Stack:** R 3.5+, depends on `httr` and `jsonlite`.

**What it does:** 9 core functions across four categories — attribute discovery, SNP retrieval
(by chromosome region / RSID / gene), SNP counting, and SNPWay workflows (SNP→gene mapping +
PANTHER overrepresentation). Returns structured results including gene mappings, PANTHER
annotations, enrichment analysis, and CSV-ready exports.

**Inputs:** user SNP/variant queries (VCF text, chromosome ranges, RSID lists, gene identifiers).
**Outputs:** R data structures / CSV-ready exports.

**Data source:** api-v2 — like annoq-py, AnnoQR retrieves its SNP/annotation data from
**annoq-api-v2**. The base URL is configurable (per call or via environment variables); it
defaults to `https://enrichment-dev.annoq.org` (a dev api-v2 deployment), so **verify it points
at the intended api-v2 instance** (dev vs prod `api-v2.annoq.org`) before running.

---

## Annoq_Overrepr_Workflow / SNPWay

<https://github.com/USCbiostats/Annoq_Overrepr_Workflow> · deployed at <https://snpway.annoq.org>

**Purpose:** A workflow application ("SNPWay") that queries AnnoQ and runs overrepresentation
(functional enrichment) analysis on genetic data. **Released** and deployed at snpway.annoq.org.

**Stack:** TypeScript frontend (~77%) + Python/FastAPI backend (uvicorn, ~22%).

**What it does:**
1. Takes a list of SNPs and queries the AnnoQ API (**api-v2**) to map SNPs → associated genes.
2. Runs overrepresentation analysis on the resulting gene set using PantherDB.
3. Serves a web interface for submitting data and viewing results.

**Inputs:** SNP lists. **Outputs:** gene mappings + PANTHER overrepresentation results.

**Gotchas:** depends on **api-v2** — API schema/limit changes can break it. The same SNPWay
workflow is also exposed programmatically through annoq-py and AnnoQR. Local dev runs the backend
on port 8002.

---

## annoq-site-v2

<https://github.com/USCbiostats/annoq-site-v2>

> 🚧 **Not released** — early development. Intended to eventually succeed [annoq-site](#4-annoq-site).

**Purpose:** Next-generation web UI for the AnnoQ platform.

**Stack:** **React** + TypeScript, built with Vite (tests via Vitest). REST/GraphQL endpoint and
dataset are configurable via environment variables.

**What it does:** A modernized rewrite of the Angular annoq-site frontend, providing a configurable
interface to query and interact with genomic annotations.

**Gotchas:** early stage (few commits, no releases) — treat as experimental. When it ships it will
replace annoq-site as stage 4; until then, annoq-site (Angular 9) remains the production UI.
