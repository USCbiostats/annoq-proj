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
3. (Pre-work) `bash tools/scripts/run_pre_work.sh ./../annoq_data` builds the interval tree,
   enhancer map, and label-free panther data.
4. Run the Java module to annotate the VCFs. It also emits
   `java_wgsa_add/.../diagnostics/panther_terms.json` — **copy that to**
   `annoq-site/src/@annoq.common/data/panther_terms.json` (the UI's term-label lookup).

### HRC merge — add HRC mapping columns (TOPMed only)
```
python3 wgsa_add/merge_hrc_topmed.py <hrc_dir> <topmed_dir> <output_dir>
```
Positional dirs of per-chromosome `.vcf` files; matches TOPMed↔HRC by chromosome and appends two
columns to each TOPMed row, then writes `merge_hrc_topmed_stats.json` into the output dir:
- **`Mapped_in_HRC`** — `Y` if the hg19-equivalent variant is in HRC r1.1, `N` if not found,
  `.` if `ref_hg19 != ref_hg38`.
- **`HRC_rs_dbSNP151`** — the HRC `rs_dbSNP151` id when `Mapped_in_HRC = Y`, else empty.

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
- HRC merge is **TOPMed-only**; don't run it on the HRC stack.

## Hand-off downstream
Producing the artifacts is stage 1. To make the new fields live:
- **annoq-database (2):** re-create the index with the new `annoq_mappings.json` (`src/reinit.py`)
  and bulk-load converted JSON (`src/index_es_json.py`), using the new `doc_type.pkl`.
- **annoq-api-v2 (3):** regenerate GraphQL types from the new ES schema (`class_generators/`);
  refresh `data/anno_tree.json` + `data/api_mapping_anno_tree.json`.
- **annoq-site (4):** GraphQL codegen; surface the field; the tree + `panther_terms.json` drive UI.
- Then run **`/annoq-doc-sync`** — a new field/tree is a shared contract.

## Local feasibility testing (small subset)
For a quick round-trip without full infra: take a subset VCF (e.g. the first N lines of one
chromosome), run the steps above to produce a **small JSON**, and load it into the **local
single-node docker ES** (`annoq-database/docker-compose-local.yml`) rather than the prod cluster.
Add the HRC fields to `annoq_mappings.json` + `doc_type.pkl` first, or they won't appear.
