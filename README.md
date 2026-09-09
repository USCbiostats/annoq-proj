# AnnoQ Project Hub

Central **project-information and workflow hub** for the AnnoQ platform — a system for
annotating human genomic variants (SNVs and indels) with 500+ functional attributes and
serving them through APIs, client libraries, and web applications.

This repository does **not** contain application source code. It is the coordination layer:
it documents how the ecosystem fits together and provides **Claude Code skills** (repeatable
playbooks) for the three most common kinds of work — **fixing a bug**, **implementing a
feature**, and **updating a configuration** — across the component repositories.

> **Live services:**
> UI (production, HRC r1.1) → <https://annoq.org> — **annoq-site-v2** (React)
> (API <https://api-v2.annoq.org>) ·
> UI (beta, TOPMed Freeze 8) → <https://topmed.annoq.org> — **annoq-site** (Angular 9)
> (API <https://api-v2.topmed.annoq.org>) ·
> SNPWay overrepresentation → <https://snpway.annoq.org>
>
> **Data note:** the deployed datasets currently contain **SNPs only (no indels)**, even though
> the annotation pipeline itself handles both SNVs and indels.

---

## The core pipeline at a glance

```
┌────────────────────┐   JSON +    ┌────────────────────┐    ES     ┌────────────────────┐   GraphQL  ┌──────────────────────┐
│ annoq-data-builder │  ES mappings│  annoq-database    │  indices  │   annoq-api-v2     │    query   │  Stage 4 — web UI    │
│                    │────────────▶│                    │──────────▶│                    │───────────▶│  (split by stack)    │
│ Build annotations: │             │ VCF/TSV → JSON,    │           │ FastAPI +          │            │ annoq-site-v2 React  │
│ WGSA (ANNOVAR/VEP/ │             │ bulk-index into    │           │ Strawberry GraphQL │            │   → annoq.org (HRC)  │
│ SnpEff) + PANTHER  │             │ Elasticsearch 8.5  │           │ over 500+ attrs    │            │ annoq-site Angular 9 │
│ (Shell/Java/Py)    │             │ (Python/Bash)      │           │ (Python 3.11)      │            │   → topmed.annoq.org │
└────────────────────┘             └────────────────────┘           └─────────┬──────────┘            └──────────────────────┘
    HPC / SLURM                       ES cluster + Kibana                      │ api-v2.annoq.org
                                                                               │
                          ┌────────────────────────────┬──────────────────────┴┐
                          ▼                            ▼                        ▼
                   ┌─────────────┐            ┌─────────────┐        ┌────────────────────┐
                   │  annoq-py   │            │   AnnoQR    │        │ Annoq_Overrepr_    │
                   │ Python      │            │ R package   │        │ Workflow (SNPWay)  │
                   │ client lib  │            │ client lib  │        │ snpway.annoq.org   │
                   └─────────────┘            └─────────────┘        └────────────────────┘
                            API consumers / clients (all query api-v2)
```

> **Stage 4 is split by stack.** **annoq-site-v2 (React) is released** and serves
> **annoq.org (HRC r1.1)**. **annoq-site (Angular 9)** is **superseded on HRC but still the TOPMed
> beta UI** at topmed.annoq.org. Until the **TOPMed cutover** — in progress under
> [annoq-site#78](https://github.com/USCbiostats/annoq-site/issues/78), ending in a **single site
> serving TOPMed with HRC as a filter** — UI changes land in **both** site repos. See below.

Data flows **left → right** through the core pipeline; a set of **consumers** query the API.
A change to a shared contract (a field, an ES mapping, the GraphQL schema) can ripple from the
builder all the way out to the clients. The skills in this repo are built around localizing
work to the right stage(s) and repo.

## Repositories

### Core pipeline (build → serve)

| Stage | Repository | Role | Stack |
|------:|------------|------|-------|
| 1 | [annoq-data-builder](https://github.com/USCbiostats/annoq-data-builder) | Build annotation data (WGSA + PANTHER/enhancer); emit JSON & ES mappings | Shell, Java, Python, SLURM |
| 2 | [annoq-database](https://github.com/USCbiostats/annoq-database) | Convert VCF/TSV → JSON, create indices, bulk-load Elasticsearch | Python, Bash, Elasticsearch 8.5, Kibana |
| 3 | [annoq-api-v2](https://github.com/USCbiostats/annoq-api-v2) | **Current API** — GraphQL query layer over Elasticsearch (dynamic types) | Python 3.11, FastAPI, Strawberry, Docker |
| 4 (HRC) | [annoq-site-v2](https://github.com/USCbiostats/annoq-site-v2) | **Production UI** at annoq.org, consuming the API | React + TypeScript, Vite/Vitest, Node 20+ |
| 4 (TOPMed) | [annoq-site](https://github.com/USCbiostats/annoq-site) | **TOPMed beta UI** at topmed.annoq.org; superseded on HRC | Angular 9, TypeScript, SCSS |

> **Stage 4 is split by stack.** [annoq-site-v2](https://github.com/USCbiostats/annoq-site-v2) —
> a **React + TypeScript** app (Vite/Vitest, no Angular) — is **released** and is the production
> UI at **annoq.org (HRC r1.1)**. [annoq-site](https://github.com/USCbiostats/annoq-site)
> (Angular 9) is **superseded on HRC but still the TOPMed beta UI** at **topmed.annoq.org**.
> Until the **TOPMed cutover**, stage-4 work lands in **both site repos** so the two stacks
> behave the same.

### Parallel deployment stacks (HRC & TOPMed)

The core pipeline is deployed as **two independent, parallel stacks** — one per dataset. Each
stack has its **own code refs** across the pipeline (data, api, site), its **own api-v2 instance**,
and its **own database/Elasticsearch instance**. They do not share running infrastructure.

> **Branch reality check.** There is **no standing `TopMed` branch** on any AnnoQ repo. Default
> branches are **`master`** for annoq-data-builder, annoq-database, annoq-api-v2 and annoq-site;
> **`main`** only for annoq-site-v2. TOPMed runs on **named issue branches in two lines** — the
> **issue-19** line is what topmed.annoq.org is deployed from, and the **issue-78** line carries
> all post-release work (not yet deployed) — it is the **TOPMed-cutover** line, tracked by
> [annoq-site#78](https://github.com/USCbiostats/annoq-site/issues/78). See
> [architecture.md](docs/architecture.md#topmed-refs--what-topmedannoqorg-is-built-from) for the
> per-repo refs. Confirm any ref before assuming a name:
> `git ls-remote --heads https://github.com/USCbiostats/<repo>.git`

```
 HRC stack  (production, default branches)
   HRC r1.1 data ─▶ database/ES instance A ─▶ api-v2.annoq.org ─▶ annoq.org
                                                                  (annoq-site-v2, React)

 TOPMed stack  (beta, issue branches)
   TOPMed Freeze 8 ─▶ database/ES instance B ─▶ api-v2.topmed.annoq.org ─▶ topmed.annoq.org
                                                                           (annoq-site, Angular 9)
```

| Stack | Dataset | Code refs | api-v2 endpoint | Database/ES instance | Site URL | Site repo | Status |
|-------|---------|-------------|-----------------|----------------------|----------|-----------|--------|
| **HRC** (production) | HRC r1.1 | default branch (`master`; `main` in annoq-site-v2) | [api-v2.annoq.org](https://api-v2.annoq.org) | instance A | [annoq.org](https://annoq.org) | **annoq-site-v2** (React) | Live |
| **TOPMed** (beta) | TOPMed: Freeze 8 | issue-19 line (deployed) · issue-78 line (in flight) | [api-v2.topmed.annoq.org](https://api-v2.topmed.annoq.org) | instance B | [topmed.annoq.org](https://topmed.annoq.org) | **annoq-site** (Angular 9) | Beta |

Both stacks currently serve **SNP data only (no indels)**.

> **Working implication:** a bug fix, feature, or config change frequently has to be applied to
> **both stacks' refs** and validated against **both api-v2 instances**. Always establish *which
> stack* a report or task concerns first. Note that the two stacks now run **different UI
> codebases** — a stage-4 change usually means editing **both site repos until the TOPMed
> cutover**. See the skills in [`.claude/skills/`](.claude/skills/).

### API (historical)

| Repository | Status | Role | Stack |
|------------|--------|------|-------|
| [annoq-api](https://github.com/USCbiostats/annoq-api) | ⚠️ **Deprecated — replaced by annoq-api-v2** | Original API: Flask wrapper for Elasticsearch-based annotation search (REST) | Python (Flask), Shell |

### API consumers / clients

| Repository | Status | Role | Stack |
|------------|--------|------|-------|
| [annoq-py](https://github.com/USCbiostats/annoq-py) | Released | Python client library for the AnnoQ API + SNPWay workflows | Python 3.7+ |
| [AnnoQR](https://github.com/USCbiostats/AnnoQR) | Released | R client package for the AnnoQ API + SNPWay workflows | R 3.5+ (`httr`, `jsonlite`) |
| [Annoq_Overrepr_Workflow](https://github.com/USCbiostats/Annoq_Overrepr_Workflow) | Released → [snpway.annoq.org](https://snpway.annoq.org) | SNPWay: SNP→gene mapping + PANTHER overrepresentation analysis; **uses api-v2** (GraphQL + download) | FastAPI (Python) + React/Vite frontend |

Both **stage-4 site repos** (annoq-site-v2, annoq-site) are also api-v2 consumers; they are listed
under the core pipeline above rather than here.

Full details on every repo: [`docs/repositories.md`](docs/repositories.md).

## Using the Claude Code skills

When you open Claude Code in this repo (or a workspace where these repos are checked out
side by side), invoke the relevant playbook:

| You need to… | Skill | What it does |
|--------------|-------|--------------|
| Fix a bug | `/annoq-bugfix` | Localize the failing stage/repo, reproduce, patch, verify downstream |
| Add a feature | `/annoq-feature` | Trace the feature across stages/consumers, plan cross-repo changes, implement, test |
| Change a config | `/annoq-config` | Identify config surface (env, mappings, compose, SLURM, endpoints), apply and validate safely |
| Keep docs in sync | `/annoq-doc-sync` | Find every place a shared api-v2 fact is documented (across repos) and update them together so docs don't drift |

> Run `/annoq-doc-sync` after any of the other skills touches a **shared contract** (api-v2
> schema, endpoints, pagination/field limits, the annotation tree, or a repo's status) — those
> facts are restated in many repos and drift silently otherwise.

Each skill lives in [`.claude/skills/`](.claude/skills/) and encodes the AnnoQ-specific
knowledge needed to route work to the correct repo. See each `SKILL.md` for details.

## Recommended local layout

The skills work best when the repos you're touching are cloned as siblings so cross-repo
changes are easy to trace:

```
annoq/
├── annoq-proj/                 # ← this repo (docs + skills)
├── annoq-data-builder/         # core pipeline
├── annoq-database/
├── annoq-api-v2/
├── annoq-site/                 # stage 4 — TOPMed beta UI (Angular 9)
├── annoq-py/                   # clients / consumers
├── AnnoQR/
├── Annoq_Overrepr_Workflow/    # SNPWay
└── annoq-site-v2/              # stage 4 — HRC production UI (React)
```

Clone the ones you need:

```bash
gh repo clone USCbiostats/annoq-data-builder
gh repo clone USCbiostats/annoq-database
gh repo clone USCbiostats/annoq-api-v2
gh repo clone USCbiostats/annoq-site
gh repo clone USCbiostats/annoq-py
gh repo clone USCbiostats/AnnoQR
gh repo clone USCbiostats/Annoq_Overrepr_Workflow
gh repo clone USCbiostats/annoq-site-v2
```

## Repository layout

```
annoq-proj/
├── README.md                       # This file (for humans)
├── CLAUDE.md                       # Always-on context auto-loaded by Claude Code
├── docs/
│   ├── architecture.md             # System architecture & data flow
│   ├── repositories.md             # Deep dive on each component repo
│   ├── pipeline.md                 # End-to-end pipeline runbook
│   └── glossary.md                 # Domain & tech terminology
├── .claude/
│   └── skills/
│       ├── annoq-bugfix/SKILL.md   # Bug-fix playbook
│       ├── annoq-feature/SKILL.md  # Feature-implementation playbook
│       ├── annoq-config/SKILL.md   # Configuration-update playbook
│       └── annoq-doc-sync/SKILL.md # Documentation-sync playbook
└── .github/
    └── ISSUE_TEMPLATE/             # Bug / feature / config issue templates
```

## Documentation

- [Architecture](docs/architecture.md) — how the stages connect, the shared contracts, and the API consumers.
- [Repositories](docs/repositories.md) — per-repo purpose, structure, inputs/outputs, and gotchas.
- [Pipeline runbook](docs/pipeline.md) — end-to-end flow from raw VCF to live UI and clients,
  including the **generated-artifact map**: what stage 1 produces, where it lands, and which path
  each file must be copied to.
- [Glossary](docs/glossary.md) — VCF, WGSA, PANTHER, SNV, SNPWay, RSID, Strawberry, and other terms.
