---
name: annoq-feature
description: Playbook for implementing a new feature in the AnnoQ pipeline (data-builder → database → api-v2 → site). Use when adding an annotation field/source, a new query/filter capability, or UI functionality that may span multiple component repos. Helps plan and sequence cross-repo changes.
---

# AnnoQ Feature-Implementation Playbook

Features in AnnoQ frequently **span multiple stages**. Adding an annotation field, for
example, must be built (1), indexed (2), exposed in GraphQL (3), and surfaced in the UI (4).
This playbook plans the cross-repo work and sequences it correctly (upstream → downstream).

> Read `docs/architecture.md` (shared contracts) and `docs/pipeline.md` (re-run matrix)
> before planning.

## Stage map

| Stage | Repo | Owns |
|------:|------|------|
| 1 build | annoq-data-builder | annotation content, ES mappings, tree/pickle artifacts |
| 2 index | annoq-database | conversion → JSON, ES index creation + bulk load |
| 3 API | **annoq-api-v2** (current) | GraphQL schema (generated), resolvers, FastAPI service |
| 4 UI | annoq-site | Angular 9 components, GraphQL client queries, rendering |

**API consumers** (build features here when the work is client- or workflow-facing):

| Consumer | Repo | When it's the target |
|----------|------|----------------------|
| Python client | annoq-py | New programmatic query/helper for Python users |
| R client | AnnoQR | New programmatic query/helper for R users |
| SNPWay app | Annoq_Overrepr_Workflow | New overrepresentation/enrichment capability (snpway.annoq.org) |
| Next-gen UI | annoq-site-v2 (React, unreleased) | UI features targeting the future frontend |

> Target **annoq-api-v2**, not the deprecated **annoq-api**, for any API-layer feature.
> A new client-facing query capability usually means api-v2 (3) **and** the relevant
> client(s) (annoq-py / AnnoQR) and/or SNPWay.

## Procedure

### 1. Classify the feature
Decide which stages it touches:

| Feature kind | Typical stages |
|--------------|----------------|
| New annotation field/source | 1 → 2 → 3 → 4 (all), + consumers if client-facing |
| New query/filter/aggregation over existing data | 3 (resolvers) → 4 (UI), sometimes 2 (mapping/analyzer) |
| Search/UX capability over existing API | 4 only (or annoq-site-v2) |
| Performance/indexing capability | 2 (and maybe 1) |
| Expose an already-indexed field | 3 → 4 (+ annoq-py / AnnoQR if clients should surface it) |
| New programmatic client capability | annoq-py and/or AnnoQR (+ 3 if the API lacks the query) |
| New overrepresentation/enrichment capability | Annoq_Overrepr_Workflow (SNPWay) (+ 3 if new API data needed) |

### 2. Plan before coding
- Write the change list **per repo, in upstream→downstream order**, and identify which
  **shared contracts** move (ES mapping, `anno_tree.json`, GraphQL schema).
- **Decide which stack(s) the feature targets.** The pipeline runs as two parallel stacks with
  separate branches, api-v2 instances, and database/ES instances:
  - **HRC stack** — `main` branch, dataset HRC r1.1, annoq.org (API `api-v2.annoq.org`).
  - **TOPMed stack** — TopMed branch, dataset TOPMed Freeze 8, topmed.annoq.org (API `api-v2.topmed.annoq.org`).
  Most code features belong on **both branches**; a dataset-specific feature may target one. State
  the choice, and plan the same change (and re-index) for each targeted stack independently. See
  `docs/architecture.md` → Parallel deployment stacks.
- For anything non-trivial, present the plan to the user before implementing. Note repos/branches
  that aren't checked out and will need follow-up.

### 3. Implement upstream → downstream
Work in data-flow order so each stage has what the next needs.

- **Stage 1 (data-builder):** produce the annotation and update the generated
  mappings/tree/pickle (`annotation_tree_gen.py`, `mappings_data_type_gen.py`). Field naming
  here is the contract for everyone downstream — get it right.
- **Stage 2 (database):** update conversion to emit the field; update the ES mapping; re-create
  the index and bulk-load (`src/reinit`, `src/index_es_json`). Verify in Kibana/`_search`.
- **Stage 3 (api-v2):** regenerate GraphQL types from the new ES schema
  (`scripts/class_generators/`, `datamodel-codegen`); add/adjust resolvers; update `data/`
  tree config if the annotation tree changed. Add `pytest` coverage.
- **Stage 4 (site):** run GraphQL codegen (`graphql_codegen.ts`), add/extend the query and the
  Angular component to display/filter the field. Update integrated docs if user-facing.
- **Consumers (if client-facing):** surface the capability in `annoq-py` / `AnnoQR` (mind the
  API limits — 10k pagination / 20 fields) and/or `Annoq_Overrepr_Workflow` (SNPWay). These
  depend on the api-v2 contract, so they come after stage 3 is settled.

### 4. Test each stage and the seam between stages
- Unit/integration test within each repo (pytest for api-v2, e2e for site).
- **Test the seams:** does api-v2 actually return the new field from a real ES query? Does the
  site render what api-v2 returns? Verify in the GraphQL playground between stages 3 and 4.

### 5. Verify the full path
- Trace the new value end-to-end: source → ES → GraphQL → UI (the forward version of the
  backward trace in `docs/pipeline.md`).
- Confirm nothing regressed for existing fields (schema regeneration can be broad).

### 6. Summarize and hand off
- Per repo: what changed, tests run, and what still needs doing (e.g. a prod re-index, a
  deploy, or work in a repo that wasn't checked out).
- Recommend the PR order (upstream repos merge/deploy first).
- **State which stack(s)/branch(es) got the change** and which still need it — a code feature
  usually lands on both `main` and the TopMed branch, each re-indexed against its own instance.
- **If the feature changed a shared contract** (new/renamed field, api-v2 schema, annotation
  tree, limits, or a repo's status), run **`/annoq-doc-sync`** to update the docs in every repo
  that restates it — including consumer READMEs and the hub docs.

## Common pitfalls
- Coding downstream before upstream — the field won't exist to expose yet.
- Renaming a field in one stage but not the shared artifacts — silently breaks the contract.
- Forgetting to regenerate GraphQL types after an ES mapping change (api-v2 types are generated).
- Forgetting client codegen in the site after an api-v2 schema change.
- Skipping the re-index step — new mappings don't apply to already-indexed documents.
- Landing a code feature on one branch/stack only when it should be on both (`main` + TopMed),
  or forgetting to re-index the second stack's database instance.
