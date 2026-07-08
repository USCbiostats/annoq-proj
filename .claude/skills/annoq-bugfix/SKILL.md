---
name: annoq-bugfix
description: Playbook for fixing a bug somewhere in the AnnoQ pipeline (data-builder → database → api-v2 → site). Use when a value is wrong/missing, indexing fails, a query breaks, or the UI misbehaves, and you need to localize the failing stage before patching. Routes work to the correct component repo.
---

# AnnoQ Bug-Fix Playbook

AnnoQ is a four-stage pipeline. A bug's *symptom* and its *cause* are often in different
stages, so the first job is always **localization**: trace the value backwards until you find
the stage where it first goes wrong. Only then patch.

> Read `docs/architecture.md` and `docs/pipeline.md` in this repo for the stage contracts and
> the backward-tracing procedure before you start if you are unfamiliar with the pipeline.

## Stage map

| Stage | Repo | Owns |
|------:|------|------|
| 1 build | annoq-data-builder | annotation content, ES mappings, tree/pickle artifacts |
| 2 index | annoq-database | VCF/TSV → JSON conversion, ES index creation + bulk load |
| 3 API | **annoq-api-v2** (current) | GraphQL schema (generated from ES), resolvers, FastAPI service |
| 4 UI | annoq-site | Angular 9 components, GraphQL client queries, rendering |

**API consumers** (also query api-v2 — a bug may live here, or here be a *symptom* of a stage-3 bug):

| Consumer | Repo | Owns |
|----------|------|------|
| Python client | annoq-py | Python wrapper; encodes API limits (10k pagination / 20 fields) |
| R client | AnnoQR | R wrapper; default base URL `enrichment-dev.annoq.org` |
| SNPWay app | Annoq_Overrepr_Workflow | SNP→gene + PANTHER overrepresentation; live at snpway.annoq.org |
| Next-gen UI | annoq-site-v2 (React, unreleased) | Experimental UI rewrite |

> **annoq-api** is the deprecated original API (Flask/REST) — do not fix new issues there;
> they belong in api-v2 unless a legacy consumer specifically requires it.

## Procedure

### 1. Reproduce and capture
- Get exact reproduction: the variant/query/URL, expected vs actual, and environment.
- **Identify which deployment stack** the report is about — the pipeline runs as two parallel
  stacks, each with its own branch, api-v2 instance, and database/ES instance:
  - **HRC stack** (production) — annoq.org → API `api-v2.annoq.org`, `main` branch, dataset HRC r1.1.
  - **TOPMed stack** (beta) — topmed.annoq.org → API `api-v2.topmed.annoq.org`, TopMed branch,
    dataset TOPMed Freeze 8.
  A value can differ between stacks and still be correct, so pin the stack before calling it a bug.
- **Confirm which api-v2 instance the site targets** — a frequent false bug is a UI pointed at
  the wrong stack's API. See `docs/architecture.md` → Parallel deployment stacks.

### 2. Localize by tracing backwards
Walk upstream, stopping at the first stage where the value is already wrong:

1. **UI wrong?** Run the same query in the GraphQL playground (`/docs` on api-v2).
   - Playground correct → bug is in **annoq-site (4)**.
2. **GraphQL wrong/missing?** Query Elasticsearch directly (Kibana or `_search`).
   - Is the field in the generated GraphQL schema at all? If ES is correct but GraphQL isn't
     → bug is in **annoq-api-v2 (3)** (type generation or resolver).
3. **ES wrong/missing?** Inspect the source JSON/converted document.
   - Source correct but index wrong → bug is in **annoq-database (2)** (conversion/mapping/load).
4. **Source data wrong?** → bug is in **annoq-data-builder (1)** (annotation built incorrectly).

**If the symptom is in a consumer** (annoq-py, AnnoQR, SNPWay/snpway.annoq.org): run the
equivalent query directly against api-v2 (`/docs` playground). If the API returns the right
result, the bug is in the **consumer** (client wrapper or SNPWay logic); if not, keep tracing
upstream into the API and beyond. Watch for API-limit assumptions (annoq-py: 10k pagination /
20 fields; AnnoQR base URL).

Use the "Tracing a value backwards" section of `docs/pipeline.md`.

### 3. Locate the repo and the branch
- Prefer a sibling checkout (`../annoq-<stage>`). If absent, offer to
  `gh repo clone USCbiostats/annoq-<stage>`.
- **Check out the branch for the affected stack** (`main` for HRC, the TopMed branch for TOPMed).
  data-builder, api-v2, and annoq-site each have both branch lines.
- Read that repo's own README/CLAUDE.md and match its conventions (see `docs/repositories.md`
  for key paths and gotchas per repo).

### 4. Write a failing test / minimal check first
- **api-v2 (3):** add/extend a `pytest` case reproducing the wrong result.
- **site (4):** reproduce in the component or an e2e spec.
- **database (2):** a small conversion/index check against sample JSON.
- **data-builder (1):** verify the generated artifact for the affected field.

### 5. Patch the smallest correct surface
- Fix at the stage where the value first goes wrong — do **not** paper over an upstream bug
  downstream (e.g. don't remap a bad field in the UI if the index is wrong).
- Match surrounding code style; keep the change minimal.

### 6. Verify locally, then check downstream ripple
- Re-run the failing test; confirm it passes.
- **Check the re-run matrix** in `docs/pipeline.md`. If the fix touched a shared contract
  (field name, ES mapping, GraphQL schema), the downstream stages must be regenerated/re-run:
  - mapping/schema change → re-index (2) → regenerate GraphQL types (3) → codegen client (4).
- If the fix changed the **api-v2 schema, annotation tree, or pagination/field limits**, also
  check the API **consumers** (annoq-py, AnnoQR, SNPWay) — they may need matching updates.
- **Does the fix also belong on the other stack's branch?** If the bug is dataset-agnostic
  (a code bug, not an HRC-only vs TOPMed-only data issue), the fix likely needs to be applied to
  **both branches** (`main` and TopMed) and validated against **both api-v2 instances**. State
  explicitly which stack(s) you patched and which still need it.
- Flag any required downstream work explicitly to the user, even if those repos aren't checked out.

### 7. Summarize
- State: the stage/repo, root cause, the fix, tests run, and any downstream re-runs needed.
- If the root cause spans stages, recommend filing follow-ups against the other repos.
- **If the fix changed a shared contract** (api-v2 schema, endpoint, limits, annotation tree, or a
  documented behavior), run **`/annoq-doc-sync`** to update every repo that documents it.

## Common pitfalls
- Fixing the symptom's stage instead of the cause's stage.
- Forgetting that api-v2 GraphQL types are **generated** — a resolver fix won't expose a field
  that isn't in the ES index yet.
- Ignoring the unique-id format (`chrom+pos+ref+alt`) when documents appear "missing" — they
  may be overwritten by id collisions.
- ES/Angular version sensitivity (ES 8.5.x, Angular 9) — check versions before deep debugging.
- Debugging the wrong stack, or assuming the two api-v2/database instances are identical — they
  are separate and may run different data/schema versions.
- Fixing only one branch when the bug is dataset-agnostic and affects both stacks.
