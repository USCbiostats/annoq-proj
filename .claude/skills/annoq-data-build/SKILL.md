---
name: annoq-data-build
description: Playbook for the annoq-data-builder post-WGSA build — after WGSA/WGSAdd has produced base annotation VCFs, add PANTHER/GO/Reactome/Enhancer functional annotations (Java module) and the HRC mapping columns (TOPMed), then generate and distribute the tree/mappings/pickle files consumed by annoq-database, annoq-api-v2 and annoq-site. Use when running or documenting the data-builder stage, regenerating anno_tree.json / annoq_mappings.json / doc_type.pkl, or adding annotation columns to the VCF.
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
WGSA (Part 1) ──▶ [Part 2] add functional annotations ──▶ [HRC merge, TOPMed only]
                                                                   │
                                          [Part 3] generate tree/mappings/pickle
                                                                   │
        annoq-database (index) ◀── annoq-api-v2 (schema) ◀── annoq-site (tree/terms)
```

Two kinds of output: (a) **annotated VCFs** (functional + HRC columns) for annoq-database to
index, and (b) **generated metadata files** (`anno_tree.json`, `annoq_mappings.json`,
`api_mapping_anno_tree.json`, `doc_type.pkl`, `panther_terms.json`) copied into the sibling repos.

## Stack awareness (do this first)

The pipeline runs as two parallel stacks (see `CLAUDE.md` / `docs/architecture.md`):
- **HRC stack** — `main` branch, dataset HRC r1.1.
- **TOPMed stack** — TopMed branch, dataset TOPMed Freeze 8.

The **HRC merge step is TOPMed-only** — it maps TOPMed hg38 variants back to HRC r1.1 and is
meaningless on the HRC stack itself. Everything else applies to both. Both deployed datasets are
**SNP-only**. Confirm the target stack before running, and use the stack's branch in every repo
(naming per `CLAUDE.md` → Branch & commit naming).

## Prerequisites
- WGSA (Part 1) output VCFs are present.
- Sibling checkouts of `annoq-site`, `annoq-database`, `annoq-api-v2` (and `annoq-api` for its
  `anno_tree.json`) on the correct stack branch. If missing, offer `gh repo clone`.
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
   `java_wgsa_add/.../diagnostics/panther_terms.json` — **copy that to**
   `annoq-site/src/@annoq.common/data/panther_terms.json` (the UI's term-label lookup).

### Part 2.1 — add HRC mapping columns (TOPMed only)
```
python3 wgsa_add/merge_hrc_topmed.py <hrc_dir> <topmed_dir> <output_dir>
```
Positional dirs of per-chromosome `.vcf` files; matches TOPMed↔HRC by chromosome and appends two
columns to each TOPMed row, then writes `merge_hrc_topmed_stats.json` into the output dir:
- **`Mapped_in_HRC`** — `Y` if the hg19-equivalent variant is in HRC r1.1, `N` if not found,
  `.` if `ref_hg19 != ref_hg38`.
- **`HRC_rs_dbSNP151`** — the HRC `rs_dbSNP151` id when `Mapped_in_HRC = Y`, else empty.

For every `Mapped_in_HRC = Y` variant it also compares **11 Uniprot columns** (`Uniprot_acc`,
`Uniprot_entry`, the five `Uniprot_mapped_to_*_flanking_region` / `*_Gene_ID` sets) between the HRC
and TOPMed rows and records per-column exact-match counts + percentages in
`merge_hrc_topmed_stats.json` under `uniprot_comparison` (denominator = `mapped_Y`). These Uniprot
columns come from Part 2, so this is another reason the merge must run **after** Part 2 — on an
unannotated VCF the script prints a `NOTE` that the Uniprot columns are absent.

These two fields must also be added to the annotation tree (below), under **HG19 Info** (node
`700`) — not as a separate empty category.

### Part 3 — generate and distribute the metadata/mapping files
1. Update `annoq-site/metadata/annotation_tree.csv` for any metadata changes, **including the two
   HRC fields** (`Mapped_in_HRC`, `HRC_rs_dbSNP151` under HG19 Info). `tools/gen_col_update_info.py`
   can help track column changes. This CSV is the **hand-maintained source of truth**.
2. Generate the tree + ES mappings + api-v2 mapping (Part 3.1):
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

## Gotchas
- **DO NOT** overwrite `annoq-site/metadata/annotation_tree.csv` with the `--output_csv`
  (`/do/not/use/annotation_tree_output.csv`) — some fields get lost. The CSV is the source; the
  `output_csv` is throwaway (that's why the path is `/do/not/use/`).
- **DO NOT** overwrite `annoq-api/data/anno_tree.json` with the `--anno_tree`
  (`/do/not/use/...`) file from `mappings_data_type_gen.py` — that script doesn't generate every
  field. Only `annotation_tree_gen`'s `--output_json` is the real `anno_tree.json`.
- The 2 new HRC fields must land in `annoq_mappings.json` **and** `doc_type.pkl` before indexing,
  or the columns are dropped/untyped and never become queryable.
- **`data/doc_type.pkl` and `data/annoq_mappings.json` are generated *and* version-controlled** —
  they are **not** regenerated on the HPC/index box. After Part 3 generates them, **commit** them
  and `git pull` on the HPC `annoq-database` checkout **before** submitting the job. The sbatch job
  hard-fails without `doc_type.pkl`; the load step needs `annoq_mappings.json`. (`.gitignore`
  excludes `output/`, not `data/`, so committing them is expected.)
- HRC merge is **TOPMed-only**; don't run it on the HRC stack.
- **Run the HRC merge AFTER Part 2 (`add_panther_enhancer` + clean), never before.** The Part-2
  step both adds the PANTHER/GO/Reactome columns **and cleans cells** — `clean_annotations.py`'s
  `remove_dots` rewrites a lone `"."` (raw dbNSFP missing marker) to `""`. If the HRC merge runs on
  the raw WGSA output instead, the result is missing ~100 PANTHER/GO/Reactome columns **and** keeps
  raw `"."` in numeric cells. `convert_to_json` skips `""` but not `"."`, so `"."` reaches a
  numeric-mapped field (e.g. `splicing_consensus_ada_score:float`) and Elasticsearch rejects every
  document (`mapper_parsing_exception`, `count=0`). Symptom check: a correct TOPMed VCF has ~830
  columns with `""` for missing; a mis-ordered one has ~724 columns with `"."`. Fix by re-running
  the merge on the panther-annotated + cleaned VCF — do **not** work around it by patching
  `convert_to_json`, which would hide the missing annotation columns.

## Hand-off downstream
Producing the artifacts is stage 1. To make the new fields live:
- **annoq-database (2):** re-create the index with the new `annoq_mappings.json` (`src/reinit.py`)
  and bulk-load converted JSON (`src/index_es_json.py`), using the new `doc_type.pkl`.
- **annoq-api-v2 (3):** regenerate GraphQL types from the new ES schema (`class_generators/`);
  refresh `data/anno_tree.json` + `data/api_mapping_anno_tree.json`.
- **annoq-site (4):** GraphQL codegen; surface the field; the tree + `panther_terms.json` drive UI.
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
  one annoq-site artifact this stage depends on is `metadata/annotation_tree.csv` (source of the
  `annoq_mappings.json` / `doc_type.pkl`). Copy the JSON to the index host, then run steps 2–5 there.
