# AnnoQ Architecture

AnnoQ is a four-stage pipeline that turns raw genomic variant files into an interactive,
queryable annotation service, plus a set of **consumers** (client libraries and apps) that
query the API. Each repository has a well-defined input/output contract; understanding those
contracts is the key to localizing any change.

## Stage overview

```
 Stage 1                Stage 2                 Stage 3                Stage 4
 ───────                ───────                 ───────                ───────
 annoq-data-builder ──▶ annoq-database ──────▶  annoq-api-v2 ───────▶  annoq-site-v2 (HRC)
 build annotations      index into ES           serve GraphQL          annoq-site  (TOPMed)
                                                                       browse/query UI

 Artifacts flowing between stages:
   1→2  Annotated VCF/TSV  +  annoq_mappings.json  +  doc_type.pkl  +  anno_tree.json
   2→3  Populated Elasticsearch indices (schema defined by the ES mappings)
   3→4  GraphQL schema + endpoint (https://api-v2.annoq.org)
```

## The shared contracts

The stages are loosely coupled but bound by a few **shared artifacts**. When these change,
the change is inherently cross-repo.

| Artifact | Produced by | Consumed by | What it defines |
|----------|-------------|-------------|-----------------|
| `annoq_mappings.json` (ES mappings) | data-builder | database | Field names, types, and analyzers for the ES index |
| `doc_type.pkl` | data-builder | database | Per-field data-type metadata used during conversion/indexing |
| `anno_tree.json` / `api_mapping_anno_tree.json` | data-builder | api-v2 | The annotation category tree exposed to clients |
| Elasticsearch index schema | database | api-v2 | The actual queryable fields; api-v2 generates GraphQL types from this |
| GraphQL schema | api-v2 | site (both stage-4 repos) | The typed query surface the web UI calls |

**Rule of thumb:** a new or renamed annotation field touches *all four* stages — build it
(1), index it (2), expose it in GraphQL (3), display/filter it in the UI (4).

## Stage 1 — annoq-data-builder

Prepares annotation data. Three logical parts:

1. **WGSA pipeline (v0.95):** runs ANNOVAR, VEP, and SnpEff over VCF files via SLURM batch
   jobs on HPC. Config generated per input file (`wgsa_095_pipeline/work_scripts/`:
   `config.py`, `sbatch.py`, `sbatch.temp`).
2. **PANTHER & enhancer annotations:** a Java module adds PANTHER protein-function and
   enhancer data; `tools/api_extractor/panther_gene_extractor.py` pulls from the PANTHER API.
3. **Downstream data generation:** `annotation_tree_gen.py` emits the JSON + Elasticsearch
   mappings; `mappings_data_type_gen.py` emits pickle files for indexing.

**Output:** annotated VCF results plus the mapping/pickle/tree artifacts consumed by stages 2–3.

## Stage 2 — annoq-database

Converts and indexes.

1. **Conversion:** VCF/TSV → JSON documents, adding a unique id from
   chromosome + position + ref + alt (`scripts/run_jobs.sh`, Python converters).
2. **Indexing:** creates/recreates ES indices with the custom mappings and bulk-loads the
   JSON (`scripts/run_es_job.sh`, `src/reinit.py`, `src/index_es_json.py`). Multiple loading strategies
   (parallel / streaming / standard). Kibana + Logstash configs included for ops.

**Output:** a populated Elasticsearch 8.5 cluster — the source of truth queried by the API.

## Stage 3 — annoq-api-v2

The query layer. FastAPI + Strawberry GraphQL. Its defining trick: it **dynamically
generates 500+ GraphQL types** from the ES schema mappings (via `datamodel-codegen` and
`scripts/class_generators/`) instead of hand-writing them. Runs under Docker Compose.

**Output:** the public GraphQL endpoint (`https://api-v2.annoq.org/docs`).

## Stage 4 — the web UI (annoq-site-v2 / annoq-site)

**Stage 4 is split by stack.** Two different UI codebases are in production, one per stack, and
both consume the same api-v2 GraphQL contract — so the shared contracts above apply to both
unchanged.

| Stack | Repo | Framework | Deployed at | Codegen / dev |
|-------|------|-----------|-------------|---------------|
| HRC (production) | **annoq-site-v2** | React + TypeScript (Vite/Vitest, Node 20+) | annoq.org | `npm run graphql_codegen` → `npm run dev` (port 5173) |
| TOPMed (beta) | **annoq-site** | Angular 9, TypeScript/SCSS | topmed.annoq.org | `npm run graphql_codegen` → `ng serve` (`localhost:4205`) |

- **annoq-site-v2** is **released** and is the production UI at **annoq.org (HRC r1.1)**. Its
  dataset and API endpoint live in `src/lib/environment.ts` (`dataset`, `annotationApiV2`),
  overridable with `VITE_ANNOQ_API_V2`. Generated types land in `src/generated/graphql.ts`;
  **codegen must run before `npm run build`**, whose typecheck depends on it.
- **annoq-site** (Angular 9) is **superseded on HRC but still the TOPMed beta UI** at
  **topmed.annoq.org**. It uses `graphql_codegen.ts` to produce typed client operations and
  carries the integrated documentation.
- Until the **TOPMed cutover** (tracked by **annoq-site#78**, in progress on the issue-78 line), a
  stage-4 change generally has to be made in **both repos** so the two stacks behave the same. The
  cutover's end state is a **single site serving TOPMed with HRC as a filter**, which collapses
  this stage-4 split — and, user-facing, the two-dataset model below. See "Parallel deployment
  stacks" below.

## The API and its consumers

The API is the platform's public boundary. Everything downstream of Elasticsearch talks to it.

**API history:** the original **annoq-api** (Flask, REST over Elasticsearch) has been
**deprecated and replaced by annoq-api-v2** (FastAPI + Strawberry GraphQL). New work targets
api-v2; annoq-api is retained only for legacy context.

**Consumers of api-v2:**

| Consumer | Kind | Notes |
|----------|------|-------|
| annoq-site-v2 | Web UI (React + TypeScript) | **Stage 4, HRC:** production at annoq.org (HRC r1.1); SNP-only |
| annoq-site | Web UI (Angular 9) | **Stage 4, TOPMed:** beta at topmed.annoq.org (TOPMed Freeze 8); SNP-only; superseded on HRC |
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

> **Branch reality check (verified 2026-09-08).** There is **no standing `TopMed` branch** on any
> AnnoQ repo. Default branches are **`master`** for annoq-data-builder, annoq-database,
> annoq-api-v2 and annoq-site; **`main`** only for annoq-site-v2. TOPMed work lands on **issue
> branches** — currently `issue-19-load-topmed` (annoq-site) and
> `annoq-site-19-add-update-metadata-for-top-med-data` (data-builder / database / api-v2). Confirm
> the ref before assuming a name: `git ls-remote --heads https://github.com/USCbiostats/<repo>.git`

### TOPMed refs — what topmed.annoq.org is built from

There is **no `TopMed` branch**. The TOPMed stack runs on **named issue branches**, in two lines:

| Repo | **issue-19 line — deployed** at topmed.annoq.org | **issue-78 line — in flight** (updates since that release) |
|------|--------------------------------------------------|------------------------------------------------------------|
| annoq-data-builder | `annoq-site-19-add-update-metadata-for-top-med-data` | `annoq-site-78-add-hrc-mapping-info` |
| annoq-database | `annoq-site-19-add-update-metadata-for-top-med-data` | `annoq-site-78-add-hrc-mapping-info` |
| annoq-api-v2 | `annoq-site-19-add-update-metadata-for-top-med-data` | `annoq-site-78-add-hrc-mapping-info` |
| annoq-site (stage 4, TOPMed) | `issue-19-load-topmed` | `issue-78-add-hrc-mapping-info` |

- **Owning issues** (both filed on annoq-site, per the branch-naming convention):
  [annoq-site#19](https://github.com/USCbiostats/annoq-site/issues/19) "Load TOPMed data to the
  elasticsearch database" (**closed** — this is the deployed line) and
  [annoq-site#78](https://github.com/USCbiostats/annoq-site/issues/78) "Integrate TopMed website
  into Annoq.org" (**open** — the active line). **#78 is the TOPMed-cutover umbrella**, in
  progress on this line. Its **end state is a single site serving TOPMed with HRC as a filter**
  (`Mapped_in_HRC=Y` via api-v2's `search_hrc`), so the HRC-mapping work is not a side
  task — it is what *makes* the single-site end state possible, which is why the cutover branches
  are named `*-add-hrc-mapping-info`.
- **What is live ≠ what is in flight.** topmed.annoq.org serves the **issue-19** line; the
  **issue-78** work (HRC-mapping search, new columns) is *not deployed there yet*. When debugging a
  TOPMed report, check which of the two you are looking at before assuming a field exists.
- The **HRC** stack tracks each repo's default branch (`master`; `main` in annoq-site-v2). Its exact
  deploy refs have not been confirmed the same way these TOPMed refs have.

```
 HRC stack (production) — default branches (`master`; `main` in annoq-site-v2)
   HRC r1.1 ─▶ database/ES instance A ─▶ api-v2.annoq.org ─▶ annoq.org
                                                             (annoq-site-v2, React)

 TOPMed stack (beta) — TOPMed issue branches
   TOPMed Freeze 8 ─▶ database/ES instance B ─▶ api-v2.topmed.annoq.org ─▶ topmed.annoq.org
                                                                          (annoq-site, Angular 9)
```

| Property | HRC stack (production) | TOPMed stack (beta) |
|----------|------------------------|---------------------|
| Dataset | HRC r1.1 | TOPMed: Freeze 8 |
| Code refs | default branch — `master` (data-builder / api-v2), `main` (annoq-site-v2) | **issue-19 line** deployed, **issue-78 line** in flight — see [TOPMed refs](#topmed-refs--what-topmedannoqorg-is-built-from); **no `TopMed` branch exists** |
| api-v2 endpoint | `https://api-v2.annoq.org` | `https://api-v2.topmed.annoq.org` |
| Database / ES instance | instance A | instance B (separate) |
| Site URL | annoq.org | topmed.annoq.org |
| Site repo (stage 4) | **annoq-site-v2** (React) | **annoq-site** (Angular 9) |
| Variant types | SNP-only | SNP-only |

**Why this matters for any change:**

- A code change (bug fix / feature) usually must be **applied to both branches** unless it is
  deliberately dataset-specific. Decide explicitly whether it belongs in one stack or both.
- **Stage 4 is split by repo, not by branch.** The HRC UI is annoq-site-v2 (React) and the TOPMed
  UI is annoq-site (Angular 9), so a UI change has to be implemented **twice, in two different
  frameworks, until the TOPMed cutover** — there is no shared branch to merge between them.
- Config is **per-instance**: there are two api-v2 endpoints and two database/ES instances.
  A config change (endpoint, index name, credentials) targets a specific instance — confirm which.
- When debugging, **identify the stack first** (annoq.org vs topmed.annoq.org). The same query
  can legitimately return different values across datasets/instances.
- The two stacks may run **different schema/data versions** at any given time (e.g. a field added
  on the TOPMed issue-78 line but not yet on the default branch, or on issue-78 but not yet
  deployed to topmed.annoq.org). Never assume the two api-v2 instances are identical.

"SNP-only" remains a property of the *deployed datasets* in both stacks, not a limitation of the
pipeline code (data-builder/database can process SNVs and indels).

## Where a change lives

| Symptom / goal | Most likely stage |
|----------------|-------------------|
| Wrong/missing annotation value in results | 1 (build) or 2 (index) |
| Field exists in ES but not queryable via API | 3 (api-v2 type generation / resolver) |
| Query works in GraphQL playground but UI is wrong | 4 (site — annoq-site-v2 for annoq.org, annoq-site for topmed.annoq.org) |
| Indexing is slow / fails / mapping mismatch | 2 (database) |
| New annotation source to add | 1 → 2 → 3 → 4 (all stages) |
| Change to search/filter behavior | 3 (resolvers) and/or 4 (UI) |

See [pipeline.md](pipeline.md) for the end-to-end runbook and [repositories.md](repositories.md)
for per-repo detail.
