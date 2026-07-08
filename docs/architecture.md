# AnnoQ Architecture

AnnoQ is a four-stage pipeline that turns raw genomic variant files into an interactive,
queryable annotation service, plus a set of **consumers** (client libraries and apps) that
query the API. Each repository has a well-defined input/output contract; understanding those
contracts is the key to localizing any change.

## Stage overview

```
 Stage 1                Stage 2                 Stage 3                Stage 4
 ───────                ───────                 ───────                ───────
 annoq-data-builder ──▶ annoq-database ──────▶  annoq-api-v2 ───────▶  annoq-site
 build annotations      index into ES           serve GraphQL          browse/query UI

 Artifacts flowing between stages:
   1→2  Annotated VCF/TSV  +  annoq_mapping.json  +  doc_type.pkl  +  anno_tree.json
   2→3  Populated Elasticsearch indices (schema defined by the ES mappings)
   3→4  GraphQL schema + endpoint (https://api-v2.annoq.org)
```

## The shared contracts

The stages are loosely coupled but bound by a few **shared artifacts**. When these change,
the change is inherently cross-repo.

| Artifact | Produced by | Consumed by | What it defines |
|----------|-------------|-------------|-----------------|
| `annoq_mapping.json` (ES mappings) | data-builder | database | Field names, types, and analyzers for the ES index |
| `doc_type.pkl` | data-builder | database | Per-field data-type metadata used during conversion/indexing |
| `anno_tree.json` / `api_mapping_anno_tree.json` | data-builder | api-v2 | The annotation category tree exposed to clients |
| Elasticsearch index schema | database | api-v2 | The actual queryable fields; api-v2 generates GraphQL types from this |
| GraphQL schema | api-v2 | site | The typed query surface the Angular app calls |

**Rule of thumb:** a new or renamed annotation field touches *all four* stages — build it
(1), index it (2), expose it in GraphQL (3), display/filter it in the UI (4).

## Stage 1 — annoq-data-builder

Prepares annotation data. Three logical parts:

1. **WGSA pipeline (v0.95):** runs ANNOVAR, VEP, and SnpEff over VCF files via SLURM batch
   jobs on HPC. Config generated per input file (`config.py`, `sbatch.py`, `sbatch.temp`).
2. **PANTHER & enhancer annotations:** a Java module adds PANTHER protein-function and
   enhancer data; `panther_gene_extractor.py` pulls from the PANTHER API.
3. **Downstream data generation:** `annotation_tree_gen.py` emits the JSON + Elasticsearch
   mappings; `mappings_data_type_gen.py` emits pickle files for indexing.

**Output:** annotated VCF results plus the mapping/pickle/tree artifacts consumed by stages 2–3.

## Stage 2 — annoq-database

Converts and indexes.

1. **Conversion:** VCF/TSV → JSON documents, adding a unique id from
   chromosome + position + ref + alt (`scripts/run_jobs.sh`, Python converters).
2. **Indexing:** creates/recreates ES indices with the custom mappings and bulk-loads the
   JSON (`run_es_job.sh`, `src/reinit`, `src/index_es_json`). Multiple loading strategies
   (parallel / streaming / standard). Kibana + Logstash configs included for ops.

**Output:** a populated Elasticsearch 8.5 cluster — the source of truth queried by the API.

## Stage 3 — annoq-api-v2

The query layer. FastAPI + Strawberry GraphQL. Its defining trick: it **dynamically
generates 500+ GraphQL types** from the ES schema mappings (via `datamodel-codegen` and
`scripts/class_generators/`) instead of hand-writing them. Runs under Docker Compose.

**Output:** the public GraphQL endpoint (`https://api-v2.annoq.org/docs`).

## Stage 4 — annoq-site

The user-facing Angular 9 SPA at annoq.org. Uses GraphQL code generation
(`graphql_codegen.ts`) to produce typed client operations against the api-v2 schema.
Includes integrated documentation. Local dev on `localhost:4205`. A React rewrite,
**annoq-site-v2**, is in early development (not released) and will eventually succeed it.

The site is deployed once per stack (see "Parallel deployment stacks" below): production at
annoq.org and beta at topmed.annoq.org, each from its own branch and pointed at its own api-v2
instance.

## The API and its consumers

The API is the platform's public boundary. Everything downstream of Elasticsearch talks to it.

**API history:** the original **annoq-api** (Flask, REST over Elasticsearch) has been
**deprecated and replaced by annoq-api-v2** (FastAPI + Strawberry GraphQL). New work targets
api-v2; annoq-api is retained only for legacy context.

**Consumers of api-v2:**

| Consumer | Kind | Notes |
|----------|------|-------|
| annoq-site | Web UI (Angular 9) | Production at annoq.org (HRC r1.1); beta at topmed.annoq.org (TOPMed Freeze 8); SNP-only |
| annoq-site-v2 | Web UI (React) | Next-gen, unreleased |
| annoq-py | Python client library | Wraps API + SNPWay workflows; 10k pagination / 20-field limits |
| AnnoQR | R client package | Wraps API + SNPWay workflows; default base URL `enrichment-dev.annoq.org` |
| Annoq_Overrepr_Workflow (SNPWay) | Web app + FastAPI | SNP→gene mapping + PANTHER overrepresentation; live at snpway.annoq.org |

Because these consumers depend on the API's schema and limits, **api-v2 is a shared contract
too**: changing field names, the annotation tree, or pagination/field limits can break clients
and SNPWay in addition to the site. See [repositories.md](repositories.md) for each consumer.

## Parallel deployment stacks (HRC & TOPMed)

The pipeline described above is not deployed once — it is instantiated as **two independent,
parallel stacks**, one per dataset. Each stack is a full vertical slice (data → database/ES →
api-v2 → site) with its **own code branch**, its **own api-v2 instance**, and its **own
database/Elasticsearch instance**. The stacks share the codebase's history but run on separate
infrastructure and are released independently.

```
 HRC stack (production) — main branches
   HRC r1.1 ─▶ database/ES instance A ─▶ api-v2.annoq.org ─▶ annoq.org

 TOPMed stack (beta) — TopMed branches
   TOPMed Freeze 8 ─▶ database/ES instance B ─▶ api-v2.topmed.annoq.org ─▶ topmed.annoq.org
```

| Property | HRC stack (production) | TOPMed stack (beta) |
|----------|------------------------|---------------------|
| Dataset | HRC r1.1 | TOPMed: Freeze 8 |
| Code branch line | `main` (data-builder / api-v2 / site) | TopMed branch (data-builder / api-v2 / site) |
| api-v2 endpoint | `https://api-v2.annoq.org` | `https://api-v2.topmed.annoq.org` |
| Database / ES instance | instance A | instance B (separate) |
| Site URL | annoq.org | topmed.annoq.org |
| Variant types | SNP-only | SNP-only |

**Why this matters for any change:**

- A code change (bug fix / feature) usually must be **applied to both branches** unless it is
  deliberately dataset-specific. Decide explicitly whether it belongs in one stack or both.
- Config is **per-instance**: there are two api-v2 endpoints and two database/ES instances.
  A config change (endpoint, index name, credentials) targets a specific instance — confirm which.
- When debugging, **identify the stack first** (annoq.org vs topmed.annoq.org). The same query
  can legitimately return different values across datasets/instances.
- The two stacks may run **different schema/data versions** at any given time (e.g. a field added
  on the TopMed branch but not yet on `main`). Never assume the two api-v2 instances are identical.

"SNP-only" remains a property of the *deployed datasets* in both stacks, not a limitation of the
pipeline code (data-builder/database can process SNVs and indels).

## Where a change lives

| Symptom / goal | Most likely stage |
|----------------|-------------------|
| Wrong/missing annotation value in results | 1 (build) or 2 (index) |
| Field exists in ES but not queryable via API | 3 (api-v2 type generation / resolver) |
| Query works in GraphQL playground but UI is wrong | 4 (site) |
| Indexing is slow / fails / mapping mismatch | 2 (database) |
| New annotation source to add | 1 → 2 → 3 → 4 (all stages) |
| Change to search/filter behavior | 3 (resolvers) and/or 4 (UI) |

See [pipeline.md](pipeline.md) for the end-to-end runbook and [repositories.md](repositories.md)
for per-repo detail.
