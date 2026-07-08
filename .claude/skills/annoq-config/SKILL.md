---
name: annoq-config
description: Playbook for updating a configuration in the AnnoQ pipeline (data-builder → database → api-v2 → site). Use when changing env vars, ES mappings/index settings, Docker Compose, SLURM batch settings, endpoint URLs, or the annotation tree — where a safe, validated change matters more than code.
---

# AnnoQ Configuration-Update Playbook

Configuration in AnnoQ is spread across stages and formats. The risk is that a config change
silently breaks a **shared contract** (an ES mapping, an endpoint URL, the annotation tree).
This playbook locates the config surface, applies the change safely, and validates it.

> Read `docs/architecture.md` for which config artifacts are shared contracts.

## Config surface by stage

| Stage | Repo | Common config |
|------:|------|---------------|
| 1 build | annoq-data-builder | SLURM batch settings (`wgsa_095_pipeline/work_scripts/`: `sbatch.py`, `sbatch.temp`, `config.py`), resource paths, PANTHER API params |
| 2 index | annoq-database | ES connection/creds, index name & settings, `annoq_mappings.json`, `doc_type.pkl`, loading strategy, Kibana/Logstash config |
| 3 API | **annoq-api-v2** (current) | `docker-compose.yaml`, env (ES host/index), `data/anno_tree.json` / `api_mapping_anno_tree.json`, `requirements.txt` |
| 4 UI | annoq-site | environment files (**target api-v2 URL**), build config, `graphql_codegen.ts` target, `metadata/` |

**API consumers** (their key config is *which API endpoint they point at*):

| Consumer | Repo | Common config |
|----------|------|---------------|
| Python client | annoq-py | API base URL / endpoint, request field & pagination settings |
| R client | AnnoQR | base URL (default `enrichment-dev.annoq.org`; per-call or env override) |
| SNPWay app | Annoq_Overrepr_Workflow | `.env`: `ANNOTATION_API_V2` (GraphQL) + `ANNOTATION_DOWNLOAD_V2` (download); backend port (8002 in dev); frontend build |
| Next-gen UI | annoq-site-v2 (React) | env vars for API endpoint + dataset (Vite) |

> Config for the API belongs in **annoq-api-v2**, not the deprecated **annoq-api**.

## Shared-contract configs (change with care)

These affect more than one stage — changing them is effectively a cross-stage change:

- **ES index name / mappings** — must agree across database (2) and api-v2 (3).
- **Annotation tree** (`anno_tree.json`) — produced in (1), consumed by (3), shapes the UI (4).
- **api-v2 endpoint URL** — the site (4) **and every consumer** (annoq-py, AnnoQR, SNPWay,
  annoq-site-v2) must point at the intended API (dev vs prod). AnnoQR defaults to
  `enrichment-dev.annoq.org` — an easy way to accidentally hit the dev API.
- **api-v2 pagination / field limits** — clients (annoq-py: 10k / 20 fields) encode these;
  changing them on the server can silently desync clients.

## Procedure

### 1. Identify the exact config surface
- Pin down the file(s) and stage from the table above. Confirm the target **environment**
  (local / staging / prod) — the same key often has different values per environment.
- **Confirm which deployment stack/instance** the config belongs to. The pipeline runs as two
  parallel stacks, each with its **own api-v2 instance and its own database/ES instance**:
  - **HRC stack** — `main` branch, dataset HRC r1.1, annoq.org → API `api-v2.annoq.org`.
  - **TOPMed stack** — TopMed branch, dataset TOPMed Freeze 8, topmed.annoq.org → API `api-v2.topmed.annoq.org`.
  There are therefore **two api-v2 endpoints and two databases**. A config value (endpoint, index
  name, credentials) usually targets **one instance** — get the right one. If the change is a code
  default rather than a per-instance value, it likely applies to **both branches**. See
  `docs/architecture.md` → Parallel deployment stacks.

### 2. Check whether it's a shared contract
- If it touches ES index/mappings, the annotation tree, or the API endpoint URL, plan for the
  downstream effects (see `docs/pipeline.md` re-run matrix) before editing.

### 3. Back up / capture the current value
- Record the current value (and which env). For secrets/creds, never print or commit them —
  confirm they belong in env/secret stores, not tracked files. Check `.gitignore`.

### 4. Apply the change minimally
- Edit only the intended key(s), matching the file's existing format/quoting.
- Keep environment-specific values in the correct environment file, not hardcoded.

### 5. Validate
| Change | Validation |
|--------|------------|
| ES mapping/index settings | Re-create index + re-index sample; confirm docs load and fields query (Kibana/`_search`) |
| Docker Compose / api-v2 env | `docker-compose up`; hit `/docs` and run a sample GraphQL query |
| SLURM/`sbatch` settings | Dry-run or a small test job; confirm resource requests are valid |
| Site env / endpoint URL | `ng serve`; confirm the app hits the intended api-v2 and data loads |
| Consumer endpoint (annoq-py / AnnoQR / SNPWay / site-v2) | Run a sample query/workflow; confirm it hits the intended api-v2 (watch AnnoQR's dev default) |
| Annotation tree | Confirm api-v2 exposes the new tree and the site renders it |
| `requirements.txt` / deps | Rebuild; run the test suite |

### 6. Propagate downstream if it's a shared contract
- ES mapping change → re-index (2) → regenerate GraphQL types (3) → client codegen (4).
- Annotation tree change → update api-v2 `data/` (3) → codegen + UI (4).
- Endpoint URL change → verify the site build for that environment.

### 7. Summarize
- State: the file(s), old → new value, environment, validation performed, and any downstream
  re-runs/redeploys required. Call out anything the user must apply in prod manually.
- **If the config is a documented shared value** (endpoint URL, pagination/field limits, index
  name, annotation tree), run **`/annoq-doc-sync`** so the docs that quote it stay in sync.

## Common pitfalls
- Editing an ES mapping without re-indexing — existing documents keep the old mapping.
- Changing the index name in one stage but not the other — api-v2 queries an empty/missing index.
- Committing secrets or environment-specific values into tracked config.
- Pointing the site **or a consumer** at the wrong api-v2 endpoint (dev vs prod) and mistaking
  it for a data bug — AnnoQR defaults to `enrichment-dev.annoq.org`.
- Editing config in the deprecated **annoq-api** instead of **annoq-api-v2**.
- Applying an instance config to the wrong stack (HRC vs TOPMed) — there are two api-v2
  endpoints and two databases; changing one does not affect the other.
- Treating a per-instance value as a shared default (or vice versa) — decide whether the change
  belongs to one instance or to both branches' code.
- Version-sensitive configs: Elasticsearch 8.5.x, Python 3.11+, Angular 9 — verify compatibility.
