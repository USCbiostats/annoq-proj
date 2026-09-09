---
name: annoq-data-build
description: Playbook for the annoq-data-builder post-WGSA build — after WGSA/WGSAdd has produced base annotation VCFs, add PANTHER/GO/Reactome/Enhancer functional annotations (Java module) and the HRC mapping columns (TOPMed), then generate and distribute the tree/mappings/pickle files consumed by annoq-database, annoq-api-v2 and both stage-4 site repos (annoq-site-v2, annoq-site). Use when running or documenting the data-builder stage, regenerating anno_tree.json / annoq_mappings.json / doc_type.pkl, or adding annotation columns to the VCF.
---

# AnnoQ Data-Builder Run Playbook (post-WGSA)

This is the **execution playbook for stage 1** (`annoq-data-builder`). WGSA (Part 1 of the
data-builder README) produces base annotation VCFs; this playbook covers what happens **after**
that: adding the functional + HRC annotation columns, then generating the metadata/mapping
artifacts every downstream repo needs.

> The authoritative, always-current commands live in **`annoq-data-builder/README.md`**
> (Part 2, Part 3, Part 3.1). Code/README is the source of truth — if this skill and the README
> disagree, trust the README and fix whichever is wrong. For *planning* a cross-repo feature use
> `/annoq-feature`; for config values `/annoq-config`; after a contract change `/annoq-doc-sync`.

## Where it sits

```
WGSA (Part 1) ──▶ [HRC merge, TOPMed only] ──▶ [Part 2] add functional annotations + clean
                                                                   │
                                          [Part 3] generate tree/mappings/pickle
                                                                   │
  annoq-database (index) ◀── annoq-api-v2 (schema) ◀── annoq-site + annoq-site-v2 (tree/terms)
```

Two kinds of output: (a) **annotated VCFs** (functional + HRC columns) for annoq-database to
index, and (b) **generated metadata files** (`anno_tree.json`, `annoq_mappings.json`,
`api_mapping_anno_tree.json`, `doc_type.pkl`, `panther_terms.json`) copied into the sibling repos.

## Stack awareness (do this first)

The pipeline runs as two parallel stacks (see `CLAUDE.md` / `docs/architecture.md`):
- **HRC stack** — each repo's **default branch** (`master`; `main` in annoq-site-v2), dataset
  HRC r1.1.
- **TOPMed stack** — dataset TOPMed Freeze 8, on a named **issue branch**.

Defaults are `master` everywhere except `annoq-site-v2` (`main`). **TOPMed has two branch lines** — no `TopMed` branch exists:
  - **issue-19 line** — `issue-19-load-topmed` (annoq-site) +
    `annoq-site-19-add-update-metadata-for-top-med-data` (data-builder / database / api-v2).
    **This is what topmed.annoq.org is deployed from.** (annoq-site#19, closed.)
  - **issue-78 line** — `issue-78-add-hrc-mapping-info` (annoq-site) +
    `annoq-site-78-add-hrc-mapping-info` (data-builder / database / api-v2). Updates since the
    release; **not deployed yet.** (annoq-site#78, open.)

The HRC-mapping columns this playbook adds (Part 2.1) are **issue-78 line** work — build them
there, not on the deployed issue-19 line. (annoq-site#78 is the **TOPMed-cutover umbrella**,
whose end state is a **single site serving TOPMed with HRC as a filter** — these columns are what
make that possible, hence the branch name.) Verify refs with
`git ls-remote --heads https://github.com/USCbiostats/<repo>.git`.

The **HRC merge step is TOPMed-only** — it maps TOPMed hg38 variants back to HRC r1.1 and is
meaningless on the HRC stack itself. Everything else applies to both. Both deployed datasets are
**SNP-only**. Confirm the target stack before running, and use the stack's branch in every repo
(naming per `CLAUDE.md` → Branch & commit naming).

## Prerequisites
- WGSA (Part 1) output VCFs are present.
- Sibling checkouts of `annoq-site`, `annoq-site-v2`, `annoq-database`, `annoq-api-v2` (and
  `annoq-api` for its `anno_tree.json`) on the correct stack branch. If missing, offer
  `gh repo clone`. **Both site repos are stage 4** — annoq-site-v2 serves annoq.org (HRC),
  annoq-site serves topmed.annoq.org (TOPMed).
- Python env: `python3 -m venv env && . env/bin/activate && pip3 install -r requirements.txt`.

## Procedure

### Part 2 — add functional annotations (PANTHER / GO / Reactome / Enhancer)
The Java module `java_wgsa_add/add_panther_enhancer` adds these columns. It needs a PANTHER
annotation file first:
1. `python3 tools/api_extractor/panther_gene_extractor.py --output panther_annot.json`
2. Copy `panther_annot.json` to the path in
   `java_wgsa_add/add_panther_enhancer/src/main/resources/add_panther_enhancer.properties`
   (or edit that property to point at the file).
3. Run the Java module to annotate the VCFs. It also emits
   `java_wgsa_add/.../diagnostics/panther_terms.json` — **copy it to both site repos** (the UI's
   term-label lookup; stage 4 is split by stack):
   - `annoq-site-v2/src/data/panther_terms.json` (HRC — annoq.org)
   - `annoq-site/src/@annoq.common/data/panther_terms.json` (TOPMed — topmed.annoq.org)

### Part 2.1 — add HRC mapping columns (TOPMed only) — runs after WGSA, before Part 2
```
python3 wgsa_add/merge_hrc_topmed.py <hrc_dir> <topmed_dir> <output_dir>
```
Run **right after WGSA (Part 1), before Part 2**. `hrc_dir` is the **raw HRC r1.1 reference VCFs**
(one per chromosome, e.g. `18.vcf` — standard 8-col `CHROM POS ID REF ALT QUAL FILTER INFO`, hg19,
bare chromosome `18`); `topmed_dir` is the WGSA-output TOPMed `.vcf`s. **SNPs only** (indels /
multiallelic ignored). Matches TOPMed↔HRC by chromosome and appends **four** columns per row, then
writes `merge_hrc_topmed_stats.json` (mapping counts only):
- **`chr_pos`** — hg38 `chr:pos`, always populated (basic info).
- **`Mapped_in_HRC`** — `Y` if the hg19-equivalent SNP is in HRC r1.1, `N` if not found,
  `.` if `ref_hg19 != ref_hg38`.
- **`HRC_chr_pos`** — hg19 `chr:pos` when `Mapped_in_HRC = Y`, else empty (HG19 info).
- **`HRC_chr_pos_ref_alt`** — hg19 `chr:posREF>ALT` (e.g. `18:10005A>T`) when `= Y`, else empty.

The HRC rsID is **not** carried — the raw HRC ID column never provides an rsID that TOPMed's own
`rs_dbSNP` lacks (verified on chr18), so HRC-by-RSID search uses `rs_dbSNP` + `Mapped_in_HRC=Y`. No
Uniprot comparison is done (the merge now runs before the Part-2 functional columns exist).

Register these in the annotation tree (below): `chr_pos` under basic info (node `1`); the three
HRC/HG19 fields under **HG19 Info** (node `700`).

### Part 3 — generate and distribute the metadata/mapping files
1. Update the annotation-tree CSV for any metadata changes. Until the switchover it lives in
   **both site repos** — update `annoq-site/metadata/annotation_tree.csv` (**still the
   authoritative copy** — see "Pending: annotation_tree.csv is moving to annoq-site-v2" below)
   **and** `annoq-site-v2/metadata/annotation_tree.csv`, keeping the two identical —
   **including the HRC
   fields** (`Mapped_in_HRC`, `HRC_chr_pos`, `HRC_chr_pos_ref_alt` under HG19 Info; `chr_pos` under
   basic info). `tools/gen_col_update_info.py` can help track column changes. This CSV is the
   **hand-maintained source of truth**.
2. Generate the tree + ES mappings + api-v2 mapping (Part 3.1). Both generators take **one**
   input CSV: pass the authoritative `annoq-site` copy until the switchover — the
   `annoq-site-v2/metadata/annotation_tree.csv` replica becomes the input at phase 2:
   ```
   python3 -m tools.annotation_tree_gen \
     --input_csv        /path/to/annoq-site/metadata/annotation_tree.csv \
     --output_csv       /do/not/use/annotation_tree_output.csv \
     --output_json      /path/to/annoq-api/data/anno_tree.json \
     --mappings_json    /path/to/annoq-database/data/annoq_mappings.json \
     --api_mappings_json /path/to/annoq-api-v2/data/api_mapping_anno_tree.json
   ```
   Then copy over: `anno_tree.json` → **both** `annoq-api/data/` and `annoq-api-v2/data/`;
   `api_mapping_anno_tree.json` → `annoq-api-v2/data/`; `annoq_mappings.json` →
   `annoq-database/data/annoq_mappings.json`.
3. Generate the ES doc-type pickle:
   ```
   python3 tools/mappings_data_type_gen.py \
     --input     /path/to/annoq-site/metadata/annotation_tree.csv \
     --output    /path/to/annoq-database/data/doc_type.pkl \
     --anno_tree /do/not/use/do_not_use_anno_tree.json  -d ,
   ```
   Copy `doc_type.pkl` → `annoq-database/data/doc_type.pkl`.

## Pending: annotation_tree.csv is moving to annoq-site-v2

The annotation-tree source of truth still lives in **`annoq-site/metadata/annotation_tree.csv`**,
even though annoq-site no longer serves annoq.org. It has been **replicated** to
`annoq-site-v2/metadata/` (repo root, alongside `README.md`), but the move is **phase 1 of 2**:

- **Now (phase 1):** `annoq-site/metadata/annotation_tree.csv` is still authoritative — edit it
  there, and keep passing that path to the generators below. The annoq-site-v2 copy is seeded from
  annoq-site **`master`** (558 rows) and is deliberately behind the `issue-78-add-hrc-mapping-info`
  version (840 rows, which adds `chr_pos`, `Mapped_in_HRC`, `HRC_chr_pos`,
  `HRC_chr_pos_ref_alt`). Pointing the generators at it today would **drop the HRC columns and
  282 rows**.
- **Phase 2 (after [annoq-site#78](https://github.com/USCbiostats/annoq-site/issues/78) merges to
  `master`):** refresh `annoq-site-v2/metadata/annotation_tree.csv` from the merged annoq-site
  `master`, verify the row count and the HRC columns, then repoint every reference below.

**Phase-2 repoint checklist** — every place the old path is documented. All of these are prose or
comments; **no code hardcodes the path** (the scripts take `--input_csv` / `--input`, and
`tools/scripts/run_pre_work.sh` uses `$INPUT_DIR`), so nothing breaks at flip time:

| Repo | File | What to change |
|------|------|----------------|
| annoq-proj | `.claude/skills/annoq-data-build/SKILL.md` | Part 3 step 1, the two generator commands, the `--output_csv` gotcha, and the local-load note |
| annoq-proj | `docs/repositories.md` | the annoq-site `metadata/` bullet (§4b) and annoq-site-v2 (§4a) |
| annoq-data-builder | `README.md` | Part 2 field registration; Part 4 steps 2–4 (both generator commands + the DO-NOT-overwrite line) |
| annoq-data-builder | `tools/gen_col_update_info.py` | the GitHub-URL comments near the bottom |
| annoq-database | `README.md` | the "generated upstream … from annoq-site/metadata" line |
| annoq-api-v2 | `docs/issue-78-hrc-mapping.md` | the "registered in annoq-site/metadata" line |
| annoq-api-v2 | `docs/superpowers/specs/2026-07-15-hrc-search-design.md` | same |
| annoq-site | `metadata/README.md` | flip the banner: this copy becomes the stale one |
| annoq-site-v2 | `metadata/README.md` | drop the "not yet canonical" banner |

Then run **`/annoq-doc-sync`** — the path is a documented shared value.

> annoq-site-v2 does **not** read this file at runtime; it builds its tree from the api-v2 response
> (`src/lib/annotations.ts`). The directory is build-time input for annoq-data-builder only, so
> hosting it there needs no site-v2 code change.

## Gotchas
- **DO NOT** overwrite `annoq-site/metadata/annotation_tree.csv` (or its
  `annoq-site-v2/metadata/annotation_tree.csv` replica) with the `--output_csv`
  (`/do/not/use/annotation_tree_output.csv`) — some fields get lost. The CSV is the source; the
  `output_csv` is throwaway (that's why the path is `/do/not/use/`).
- **DO NOT** overwrite `annoq-api/data/anno_tree.json` with the `--anno_tree`
  (`/do/not/use/...`) file from `mappings_data_type_gen.py` — that script doesn't generate every
  field. Only `annotation_tree_gen`'s `--output_json` is the real `anno_tree.json`.
- The new HRC/basic fields (`chr_pos`, `Mapped_in_HRC`, `HRC_chr_pos`, `HRC_chr_pos_ref_alt`) must
  land in `annoq_mappings.json` **and** `doc_type.pkl` before indexing, or the columns are
  dropped/untyped and never become queryable.
- **`data/doc_type.pkl` and `data/annoq_mappings.json` are generated *and* version-controlled** —
  they are **not** regenerated on the HPC/index box. After Part 3 generates them, **commit** them
  and `git pull` on the HPC `annoq-database` checkout **before** submitting the job. The sbatch job
  hard-fails without `doc_type.pkl`; the load step needs `annoq_mappings.json`. (`.gitignore`
  excludes `output/`, not `data/`, so committing them is expected.)
- HRC merge is **TOPMed-only**; don't run it on the HRC stack.
- **Order is WGSA → HRC merge → Part 2 (PANTHER/enhancer + clean) → convert.** The HRC merge runs
  on the raw WGSA output (it only needs the hg19 columns); **Part 2 must still run after it and
  before VCF→JSON conversion.** The Part-2 **Java** module (`java_wgsa_add/add_panther_enhancer`)
  both adds the functional columns **and cleans cells** — rewriting a lone `"."` (raw dbNSFP missing
  marker) to `""`. (The legacy Python `add_annotations.py` / `clean_annotations.py` path is superseded
  by this Java module — see the data-builder README's "Legacy / unused code".) If conversion runs
  on a VCF that never went through Part-2 cleaning, raw `"."` reaches a numeric-mapped field (e.g.
  `splicing_consensus_ada_score:float`) and Elasticsearch rejects every document
  (`mapper_parsing_exception`, `count=0`). Symptom check: a correct, cleaned TOPMed VCF has `""` for
  missing numeric cells; an uncleaned one keeps raw `"."`. Fix by converting the Part-2-cleaned VCF —
  do **not** patch `convert_to_json`, which would hide the issue.

## Hand-off downstream
Producing the artifacts is stage 1. To make the new fields live:
- **annoq-database (2):** re-create the index with the new `annoq_mappings.json` (`src/reinit.py`)
  and bulk-load converted JSON (`src/index_es_json.py`), using the new `doc_type.pkl`.
- **annoq-api-v2 (3):** regenerate GraphQL types from the new ES schema (`class_generators/`);
  refresh `data/anno_tree.json` + `data/api_mapping_anno_tree.json`.
- **stage 4 — both site repos:** GraphQL codegen; surface the field; the tree +
  `panther_terms.json` drive the UI. `annoq-site-v2` (HRC, annoq.org) **and** `annoq-site`
  (TOPMed, topmed.annoq.org) until the TOPMed cutover.
- Then run **`/annoq-doc-sync`** — a new field/tree is a shared contract.

## Local end-to-end load test (stage 1 → stage 2), in strict order
For a quick round-trip without full infra: take a subset VCF (e.g. the first N lines of one
chromosome), produce a **small JSON**, and load it into the **local single-node docker ES**
(`annoq-database/docker-compose-local.yml`) rather than the prod cluster.

> ⚠️ **ORDER MATTERS — generate (and verify) the JSON *before* starting the database.**
> **Why:** VCF→JSON conversion (step 1 below) is pure CPU/disk and needs **no** Elasticsearch.
> Doing it first means you (a) catch empty/bad output before spending memory + disk on an ES
> container, and (b) never bulk-load a half-written file. Bring ES up only once the JSON is
> verified. (The two are technically independent until the load step — this order is a
> discipline that avoids wasted infra and confusing failures.)

Prerequisite: the HRC fields must already be in `annoq_mappings.json` + `doc_type.pkl`
(see Part 3), or they won't be indexed/typed.

1. **Generate the JSON** (no ES required). From `annoq-database`:
   ```
   bash scripts/run_jobs.sh \
     --work_name <WORK_NAME> --base_dir <BASE_DIR> \
     --es_index <ANNOQ_ANNOTATIONS_INDEX> --local
   ```
   - Input : `<BASE_DIR>/annoq-data-builder/wgsa_add/output/<WORK_NAME>/*.vcf`
   - Output: `<BASE_DIR>/annoq-database/output/<WORK_NAME>/<vcf>/1.json`
   - **Verify before continuing:** output exists & is non-trivial; the `_index` inside the JSON
     equals `--es_index`; spot-check new fields (e.g. `grep -o '"Mapped_in_HRC": "[YN.]"' … | sort | uniq -c`).
   - If the output dir already exists, `run_jobs.sh` prompts `(yes/no)` before wiping it.

2. **Set the index name.** `annoq-database/.env` → `ANNOQ_ANNOTATIONS_INDEX` **must equal** the
   `--es_index` used in step 1 (the index name is baked into every JSON bulk line).

3. **Start local Elasticsearch** (single-node, low-mem) — only now:
   `docker compose -f docker-compose-local.yml up -d` (or copy that file to `docker-compose.yml`
   first). Wait for `curl -s localhost:9200/_cluster/health` to report green/yellow.

4. **Create the index + bulk-load** the JSON. Index creation and loading are **two separate
   module calls** — keep them separate whenever more than one file/chromosome goes into the index:

   a. **Create the index — ONCE per index** (`src.reinit` deletes + recreates it empty):
      ```
      python3 -m src.reinit \
        --index_name    <ANNOQ_ANNOTATIONS_INDEX> \
        --mappings_file data/annoq_mappings.json \
        --settings_file data/annoq_settings.json
      ```
      > ⚠️ **DO NOT re-run `src.reinit` once data is loaded — it DELETES all previously loaded
      > chromosomes.** Run it exactly once, before the first load.

   b. **Load data — once per chromosome/batch** (`src.index_es_json` only appends, never deletes):
      ```
      python3 -m src.index_es_json <annoq-database/output/<WORK_NAME>/<vcf>/>
      ```
      Repeat for each chromosome's JSON dir; it loads into the `.env` `ANNOQ_ANNOTATIONS_INDEX`
      (must equal the `--index_name` used in step a). **Check doc counts in Kibana between batches**
      (`GET <index>/_count`) to confirm each load added rows rather than replacing them.

   > `scripts/run_es_job.sh` bundles both (reinit **then** load) in one shot, so it **recreates the
   > index every run**. Use it ONLY for a fresh single-file test/rebuild — never for incremental
   > multi-chromosome loading, or each run wipes the prior chromosomes.

5. **Refresh** so docs become searchable:
   `curl -X GET localhost:9200/<ANNOQ_ANNOTATIONS_INDEX>/_refresh`.

> The prod `scripts/create-index.sh` / `scripts/refresh-es.sh` are hardcoded to `bioghost3.usc.edu`
> and the prod index — for local testing use the localhost equivalents above; **do not** edit those
> prod scripts to point at localhost.

### Generating the JSON on HPC/CARC (instead of `--local`)
Step 1 (VCF→JSON) is the heavy part and is normally run on CARC. **Drop `--local`** and the same
`run_jobs.sh` renders `src/db_batch.template` per VCF and `sbatch`-submits it:
```
cd <BASE_DIR>/annoq-database
bash scripts/run_jobs.sh --work_name <WORK_NAME> --base_dir <BASE_DIR> --es_index <ES_INDEX>
```
- Input `<BASE_DIR>/annoq-data-builder/wgsa_add/output/<WORK_NAME>/*.vcf`; output
  `<BASE_DIR>/annoq-database/output/<WORK_NAME>/<vcf>/1.json`; slurm scripts under
  `<BASE_DIR>/annoq-database/slurm/<WORK_NAME>/`. Monitor with `squeue -u $USER`.
- `db_batch.template` gotchas on CARC:
  - **SLURM account/partition** are pinned to the `pdthomas_136` / `thomas` condo. Leave as-is only
    if you actually run on that condo; otherwise set your own (`myaccount` / `sacctmgr show assoc`).
  - **Python module** line pins `python/3.9.12`; a plain `module purge && module load python`
    (any modern 3.x, e.g. 3.13) works too.
  - It activates a venv named **`venv`** (not `env`): create it once —
    `module load python && python3 -m venv venv && . venv/bin/activate && pip3 install -r requirements.txt`.
  - Needs `data/doc_type.pkl` present (see the version-control gotcha below).
- Only the JSON is produced here. Loading into ES + the site/api-v2 do **not** run on this box — the
  one site artifact this stage depends on is `metadata/annotation_tree.csv` (source of the
  `annoq_mappings.json` / `doc_type.pkl`). Until the switchover it exists in **both site repos** —
  `annoq-site/metadata/annotation_tree.csv` (**still authoritative**, so pass that one to the
  generators) and the `annoq-site-v2/metadata/annotation_tree.csv` replica.
  Copy the JSON to the index host, then run steps 2–5 there.
