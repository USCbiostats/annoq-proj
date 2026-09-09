---
name: annoq-doc-sync
description: Keep AnnoQ documentation in sync across repos when something changes. Many repos (annoq-proj, api-v2, annoq-py, AnnoQR, SNPWay, annoq-site, annoq-site-v2) restate the SAME api-v2 facts — endpoints, pagination/field limits, query modes, the 500+ attribute set, the annotation tree. Use whenever the API contract or another shared fact is modified, or as a periodic audit, to find every place that documents it and update them together so they don't drift.
---

# AnnoQ Documentation-Sync Playbook

> **Note:** `annoq-proj/CLAUDE.md` restates several shared facts (stack model, api-v2 endpoints,
> datasets, SNP-only, deprecation/release status). Treat it as a documentation location too —
> include it whenever those facts change.

AnnoQ documentation is **duplicated across repos**. The same facts about **api-v2** — its
endpoint, pagination and field limits, supported query modes, the "500+ attributes" figure, and
the annotation tree — are restated in the hub docs, in api-v2's own README, and in every
consumer (annoq-py, AnnoQR, SNPWay, annoq-site, annoq-site-v2). When one of those facts changes
in code but only some docs are updated, the rest **silently drift** and mislead users.

This skill's job: given a change (or on a periodic audit), find **every** place a shared fact is
documented and update them all consistently, from a single source of truth.

> Related: `/annoq-bugfix`, `/annoq-feature`, `/annoq-config`. Those change behavior; **this one
> chases the documentation that describes it.** Run this after any of them touch a shared contract.

## Source of truth

Code and the generated schema are authoritative — **prose docs must conform to them**, never the
reverse. When docs disagree with the code, fix the docs (or file a bug if the code is wrong).

| Shared fact | Authoritative source |
|-------------|----------------------|
| GraphQL schema / available attributes | **annoq-api-v2** — the generated schema + `/docs` playground |
| Pagination cap, max fields per request | annoq-api-v2 (server enforcement) |
| Annotation tree / categories | `anno_tree.json` (built in data-builder, served by api-v2) |
| Elasticsearch mappings / field names | `annoq_mappings.json` (data-builder → database) |
| Endpoint URLs | deployment config (HRC api `api-v2.annoq.org`; TOPMed api `api-v2.topmed.annoq.org`; dev `enrichment-dev.annoq.org`) |
| Deployment URLs & datasets | deployment config / running sites (annoq.org → api-v2.annoq.org = HRC r1.1; topmed.annoq.org → api-v2.topmed.annoq.org = TOPMed Freeze 8) |
| Which site repo serves which URL | the served HTML: React/Vite builds have `<div id="root">` + `/assets/index-*.js` (annoq-site-v2); Angular builds have `runtime.*.js`/`main.*.js` (annoq-site) |
| Stack/branch structure | **`git ls-remote --heads`** on each repo (never prose) + deployment/infra config. There is **no `TopMed` branch**; defaults are `master` except annoq-site-v2 (`main`). **Stage 4 is split by repo, not ref:** annoq-site-v2 = HRC, annoq-site = TOPMed |
| Deploy provenance (which ref a live site is built from) | the deploy config / whoever runs the release — **not** inferable from branch names. Confirmed: topmed.annoq.org = the **issue-19** line |
| SNP-only vs indel data | the actual indexed dataset (currently SNP-only) |
| Client capabilities | the client's own source (annoq-py, AnnoQR) |

## Shared-fact → where-it's-documented map

When one of these changes, update **all** listed locations.

| Shared fact | Documented in |
|-------------|---------------|
| **api-v2 endpoint URL(s)** — `api-v2.annoq.org` (HRC) & `api-v2.topmed.annoq.org` (TOPMed) | annoq-proj (README, architecture, pipeline, repositories); api-v2 README; annoq-py README; AnnoQR README (base URL default); SNPWay README; annoq-site (per stack) & annoq-site-v2 env docs |
| **Pagination cap (10,000)** | annoq-proj (repositories, pipeline, skills); annoq-py README; AnnoQR README; api-v2 README |
| **Max fields per request (20)** | annoq-proj (repositories, skills); annoq-py README; AnnoQR README; api-v2 README |
| **"500+ attributes" figure** | annoq-proj (README, architecture, repositories); api-v2 README; site docs |
| **Query modes (chrom region / RSID / gene / VCF)** | annoq-proj (glossary, repositories); annoq-py README; AnnoQR README; site help |
| **Annotation tree / categories** | api-v2 `data/`; annoq-proj (architecture); site UI + integrated docs |
| **ES field names / mappings** | data-builder & database READMEs; api-v2 (generated); annoq-proj (architecture) |
| **Repo status (deprecated / unreleased / released)** | annoq-proj (README, repositories, glossary, CLAUDE.md); the repo's own README/CLAUDE.md; `.github/ISSUE_TEMPLATE/*` |
| **Successor relationships (annoq-api → annoq-api-v2; annoq-site → annoq-site-v2)** | annoq-proj (README, architecture, repositories, glossary, CLAUDE.md, the bugfix/feature/config/data-build skills, `.github/ISSUE_TEMPLATE/*`); the predecessor repo's README/CLAUDE.md; the successor repo's README |
| **Stage-4 split by stack (annoq-site-v2 = HRC/annoq.org; annoq-site = TOPMed/topmed.annoq.org)** | annoq-proj (README, architecture, pipeline, repositories, glossary, CLAUDE.md, all four other skills, `.github/ISSUE_TEMPLATE/*`); annoq-site README/CLAUDE.md; annoq-site-v2 README; annoq-api-v2 README (its "Downstream" section) |
| **Local UI dev ports / build commands (5173 + `npm run dev`; 4205 + `ng serve`)** | annoq-proj (architecture, pipeline, config skill); each site repo's README |
| **`panther_terms.json` provenance + distribution targets** | annoq-proj (`docs/pipeline.md` → "Generated artifacts", `docs/architecture.md` contracts table, data-build skill); annoq-data-builder README (both branch lines); both site repos' own docs |
| **`annotation_tree.csv` location** — in **both** site repos until the switchover (`annoq-site/metadata/` authoritative, `annoq-site-v2/metadata/` replica) | annoq-proj (`docs/pipeline.md`, `docs/architecture.md`, data-build skill, repositories); annoq-data-builder README (both branch lines); annoq-database README; annoq-api-v2 `docs/issue-78-hrc-mapping.md`; both site repos' `metadata/README.md` |
| **Stack/version facts (ES 8.5, Python 3.11, Angular 9, React)** | annoq-proj (repositories, glossary); each repo's README |
| **Deployment URLs (annoq.org, topmed.annoq.org, snpway.annoq.org)** | annoq-proj (README, architecture, pipeline, repositories); annoq-site README; SNPWay README |
| **Dataset versions (HRC r1.1 prod, TOPMed Freeze 8 beta)** | annoq-proj (README, architecture, pipeline, repositories, glossary); annoq-site README / branch docs |
| **SNP-only vs indel status of deployed data** | annoq-proj (README, architecture, pipeline, glossary, repositories); annoq-site docs |
| **Parallel-stack model (2 stacks: separate branches, api-v2 instances, database/ES instances)** | annoq-proj (README, architecture, pipeline, repositories, glossary); the skills (bugfix/feature/config); each stacked repo's branch docs |
| **Per-stack code refs (HRC = default branch, `master`/`main`; TOPMed = issue branches, **no `TopMed` branch**)** | annoq-proj (README, architecture, pipeline, repositories, glossary, CLAUDE.md, bugfix/feature/config/data-build skills); each stacked repo's README/CLAUDE.md |
| **TOPMed issue-19 line (deployed) vs issue-78 line (in flight)** | annoq-proj (README, architecture, pipeline, repositories, glossary, CLAUDE.md, bugfix/feature/config/data-build skills); annoq-site CLAUDE.md |
| **TOPMed cutover = annoq-site#78 (umbrella, in progress); end state = a single site serving TOPMed with HRC as a filter (`Mapped_in_HRC`/`search_hrc`)** | annoq-proj (README, architecture, repositories, glossary, CLAUDE.md, feature/data-build skills); annoq-site CLAUDE.md; annoq-site-v2 README |

If a fact isn't in this table but is restated in ≥2 repos, **add a row** — the map is meant to grow.

## Procedure

### 1. Identify what changed
- Name the specific fact(s) that moved (e.g. "pagination cap 10k → 50k", "endpoint host changed",
  "field `x` renamed", "annoq-site-v2 released"). Get the new authoritative value from the source
  in the table above — do not guess.

### 2. Find every occurrence
Search the hub and any checked-out sibling repos. Prefer the Grep tool; useful patterns:

- Endpoints: `api-v2\.annoq\.org`, `api-v2\.topmed\.annoq\.org`, `topmed\.annoq\.org`, `api-v2`, `enrichment-dev`, `snpway`
- Limits: `10,?000`, `\b20\b.*field`, `pagination`, `field.*(max|limit)`
- Attribute count: `500\+?`, `attributes`
- Query modes / domain: `RSID`, `chromosome`, `\bgene\b`, `VCF`
- Versions/status: `Elasticsearch 8`, `Python 3\.1`, `Angular 9`, `React`, `Node 20`, `deprecated`, `unreleased`, `not released`, `superseded`, `cutover`
- Successor language: `replace`, `succeed`, `successor`, `next-gen`, `rewrite`, `site-v2`
- Deployments/datasets: `topmed\.annoq\.org`, `annoq\.org`, `HRC`, `r1\.1`, `TOPMed`, `Freeze 8`, `SNP`, `indel`
- Stacks/branches: `stack`, `instance`, `TopMed branch`, `\bmain\b`, `\bmaster\b`, `branch line`, `parallel`, `split by stack`
- Stage-4 UI: `annoq-site\b`, `site-v2`, `4205`, `5173`, `ng serve`, `npm run dev`, `graphql_codegen`

Search across repos, e.g.:
```
# note: annoq-site-v2 may not be a sibling of the others — add its path explicitly
grep -rniE '10,?000|20 field|api-v2\.annoq\.org|500\+' \
  ../annoq-* ../AnnoQR ../Annoq_Overrepr_Workflow ../../annoq-site-v2 .
```
List every hit before editing — the point of this skill is to catch the ones you'd otherwise miss.

### 3. Update consistently
- Change every occurrence to the new authoritative value, keeping each doc's wording/format.
- Use the **same phrasing and numbers** everywhere for a given fact (e.g. always "10,000-result
  pagination cap"), so future greps stay reliable.
- Update this skill's fact→location map if a fact gained or lost a documentation site.
- Fix cross-references and links if a repo's status changed (e.g. when annoq-site-v2 is released,
  update "not released" notes and successor language in annoq-proj + both site READMEs).
- **Canonical phrasings currently in force** (reuse verbatim so greps keep working):
  - "**annoq-site-v2** is **released** — the production UI at **annoq.org (HRC r1.1)**"
  - "**annoq-site** (Angular 9) is **superseded on HRC but still the TOPMed beta UI** at
    **topmed.annoq.org**" — it is **not** deprecated
  - "**Stage 4 is split by stack**" · "pending the **TOPMed cutover**" (tracked by
    **annoq-site#78**, the cutover umbrella — describe it as *in progress*, not planned)
  - "a stage-4 change lands in **both site repos** until the TOPMed cutover"
  - "**no standing `TopMed` branch**" · "default branch (`master`; `main` in annoq-site-v2)" ·
    "TOPMed **issue branches**" · "**issue-19 line** (deployed)" · "**issue-78 line** (in flight)"
- **An issue's body is not its design.** annoq-site#78's body is a one-line January note that
  misdescribes the actual end state; the implementation is authoritative. Ask the implementer
  rather than quoting an issue body as the plan.
- **Never restate a branch name from prose.** Branch facts drifted undetected for a long time here:
  the docs claimed a `main`/`TopMed` branch line that never existed. Confirm with
  `git ls-remote --heads https://github.com/USCbiostats/<repo>.git` before writing one down.

### 4. Verify
- Re-run the searches from step 2 — confirm **no stale value remains**.
- Sanity-check the new value against the authoritative source (query `/docs`, read the code/config).
- For repos not checked out locally, **list the exact files and edits still needed** so the owner
  can apply them (or open issues/PRs against those repos).

### 5. Summarize
- Report: the fact(s) changed, the authoritative new value, every file updated (per repo), and any
  updates still pending in repos you couldn't reach.

## When to run
- Immediately after `/annoq-bugfix`, `/annoq-feature`, or `/annoq-config` touches a **shared
  contract** (schema, endpoint, limits, annotation tree, repo status).
- On a **periodic audit** — pick one row of the fact map and verify every listed location agrees.
- When onboarding a new repo that restates api-v2 facts — add it to the map.

## Common pitfalls
- Updating annoq-proj's docs but not the consumers' own READMEs (or vice versa).
- Leaving the number in one phrasing here and another there, so a future search misses it.
- Trusting prose over code — always reconcile against the authoritative source.
- Forgetting the auto-generated surfaces (api-v2 `/docs`, GraphQL playground) reflect the *code*;
  if prose disagrees with them, the prose is what's stale.
- Editing docs in the **deprecated** annoq-api instead of annoq-api-v2.
