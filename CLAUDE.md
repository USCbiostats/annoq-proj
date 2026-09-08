# CLAUDE.md — annoq-proj

Standing context for any Claude Code session in this repo. For the full picture read
[`README.md`](README.md) and [`docs/`](docs/); this file is the terse, always-on version.

## What this repo is

The **coordination hub** for the AnnoQ platform — documentation + Claude skills. It contains
**no application source code**. Application code lives in the sibling repositories (below).
When a task is about *building/running* AnnoQ, the work belongs in one of those repos, not here.

## Skills — use them

When the task matches, invoke the skill rather than improvising:

- `/annoq-bugfix` — fixing a bug anywhere in the pipeline.
- `/annoq-feature` — implementing a feature (often spans repos/stacks).
- `/annoq-config` — changing a configuration value.
- `/annoq-data-build` — running the data-builder stage after WGSA: add functional + HRC
  annotation columns, then generate/distribute the tree, mappings, and doc-type artifacts.
- `/annoq-doc-sync` — **run this after any change to a shared contract** (api-v2 schema,
  endpoints, pagination/field limits, annotation tree, dataset, or repo status) to keep the
  docs across repos from drifting.

## The pipeline (4 stages)

`annoq-data-builder` → `annoq-database` → `annoq-api-v2` → **web UI (stage 4)**
(build annotations → index into Elasticsearch → GraphQL API → browse/query UI)

- **Current API is `annoq-api-v2`** (FastAPI + Strawberry GraphQL). `annoq-api` is **deprecated**
  — do not put new work there.
- api-v2 GraphQL types are **generated from the ES schema** — a field must exist in the index
  before the API can expose it.
- API **consumers** also query api-v2: `annoq-py` (Python), `AnnoQR` (R),
  `Annoq_Overrepr_Workflow` / SNPWay (snpway.annoq.org).
- **Stage 4 is split by stack.** `annoq-site-v2` (**React** + TypeScript, Vite, no Angular) is
  **released** and is the production UI at **annoq.org (HRC r1.1)**. `annoq-site` (Angular 9) is
  **superseded on HRC but still the TOPMed beta UI** at **topmed.annoq.org**. Until the **TOPMed
  cutover**, a stage-4 change lands in **both site repos** — get both, or say which you skipped.

## Two parallel deployment stacks — critical

The pipeline is deployed **twice**. Each stack has its own branch, api-v2 instance, and
database/ES instance. **Always establish which stack a task concerns first.**

| Stack | Code refs | Dataset | api-v2 | Site | Site repo |
|-------|-----------|---------|--------|------|-----------|
| **HRC** (production) | default branch (`master`; `main` in annoq-site-v2) | HRC r1.1 | `https://api-v2.annoq.org` | annoq.org | `annoq-site-v2` (React) |
| **TOPMed** (beta) | issue-19 line (deployed) / issue-78 line (in flight) | TOPMed: Freeze 8 | `https://api-v2.topmed.annoq.org` | topmed.annoq.org | `annoq-site` (Angular 9) |

**There is no standing `TopMed` branch** — don't look for one. Defaults are `master` everywhere
except `annoq-site-v2` (`main`). TOPMed runs on **named issue branches in two lines**:

| Line | data-builder / database / api-v2 | annoq-site | State |
|------|----------------------------------|------------|-------|
| **issue-19** (annoq-site#19, closed) | `annoq-site-19-add-update-metadata-for-top-med-data` | `issue-19-load-topmed` | **deployed** at topmed.annoq.org |
| **issue-78** (annoq-site#78, open) | `annoq-site-78-add-hrc-mapping-info` | `issue-78-add-hrc-mapping-info` | **in flight**, not deployed |

**annoq-site#78 is the TOPMed-cutover umbrella** ("Integrate TopMed website into Annoq.org"), so
the issue-78 line *is* the cutover work line — the cutover is **in progress**. Its end state is a
**single site serving TOPMed with HRC as a filter** (`Mapped_in_HRC=Y` via api-v2's
`search_hrc`), so HRC stops being a separate deployment/dataset user-facing. The issue's one-line
body is not the design — go by the implementation. New TOPMed work belongs on that line and should
reference #78.

So **what is live ≠ what is in flight**: topmed.annoq.org serves the issue-19 line. Establish which
line a task concerns, not just which stack. Check refs with
`git ls-remote --heads https://github.com/USCbiostats/<repo>.git`.

- The stack split spans **data-builder, api-v2, and annoq-site**. The stage-4 split is by
  **repo**, not ref: HRC is served by `annoq-site-v2` (`main`), TOPMed by `annoq-site` (a TOPMed
  issue branch).
- A dataset-agnostic code change usually must land on **both stacks' refs** and be re-indexed against
  **both** database instances. A per-instance config value targets **one** stack — get the right one.
- The two api-v2/database instances are **separate** and may run different data/schema versions —
  never assume they're identical.
- Both stacks currently serve **SNPs only (no indels)**. This is a property of the deployed
  *datasets*, not the pipeline code (which handles SNVs and indels). State it accurately.

## Working rules

- **Code/generated schema is the source of truth; prose docs conform to it, never the reverse.**
  If docs disagree with code, fix the docs (or file a bug if the code is wrong).
- Keep shared facts phrased **identically everywhere** (e.g. always "10,000-result pagination
  cap", always "SNP-only") so `/annoq-doc-sync` greps stay reliable.
- Sibling repos may not be checked out. If one is missing, offer to
  `gh repo clone USCbiostats/<repo>`; if you can't reach it, **list the exact edits it still
  needs** instead of silently skipping.
- This repo is currently tracked inside a larger parent git repo (the home directory), not its
  own. Don't stage/commit unless asked; confirm the intended repo first.

## Branch & commit naming (applies to all repos)

An issue is filed against **one** repo (its *owning repo*), but a fix often spans several. Name
branches and commits so every repo points back to the owning issue.

- **Branch name:**
  - In the **owning repo** (where the issue lives): `issue-<num>-<short-desc>` —
    e.g. annoq-site#78 → [`issue-78-add-hrc-mapping-info`](https://github.com/USCbiostats/annoq-site/tree/issue-78-add-hrc-mapping-info).
  - In **every other repo** touched by the fix: prefix with the owning repo —
    `<owning-repo>-<num>-<short-desc>` — e.g. in annoq-data-builder →
    [`annoq-site-78-add-hrc-mapping-info`](https://github.com/USCbiostats/annoq-data-builder/tree/annoq-site-78-add-hrc-mapping-info).
- **Commit message:**
  - In the **owning repo**: `For #<num>` — optionally followed by a description.
  - In **every other repo**: reference the owning repo's issue —
    `For #USCbiostats/<owning-repo>/issues/<num>` — optionally followed by a description.

## Sibling repos (recommended sibling checkout layout)

`annoq-data-builder`, `annoq-database`, `annoq-api-v2`, `annoq-site-v2` (stage 4, HRC),
`annoq-site` (stage 4, TOPMed), `annoq-py`, `AnnoQR`, `Annoq_Overrepr_Workflow` — see
[`docs/repositories.md`](docs/repositories.md).
