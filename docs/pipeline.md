# Pipeline Runbook

End-to-end flow from a raw variant file to a value visible in the AnnoQ web UI. Use this to
understand ordering, to trace where a value comes from, and to know what must re-run after a
change at a given stage.

## End-to-end flow

```
 raw VCF
   │
   ▼  [annoq-data-builder]
 (1) WGSA annotate (ANNOVAR + VEP + SnpEff)  ──▶ annotated VCF
     + PANTHER / enhancer (Java + PANTHER API)
     + generate mappings/tree/pickle
   │        artifacts: annoq_mappings.json, doc_type.pkl, anno_tree.json
   ▼  [annoq-database]
 (2) convert VCF/TSV → JSON (add unique id = chrom+pos+ref+alt)
     create ES index from annoq_mappings.json
     bulk-load JSON into Elasticsearch 8.5
   │        artifact: populated ES indices
   ▼  [annoq-api-v2]
 (3) generate GraphQL types from ES schema
     serve FastAPI + Strawberry GraphQL
   │        artifact: GraphQL endpoint (api-v2.annoq.org)
   ▼  [annoq-site-v2 (HRC)  |  annoq-site (TOPMed)]
 (4) codegen GraphQL client, render the web UI
     React/Vite → annoq.org        Angular 9 → topmed.annoq.org
   │
   ▼
 value visible / queryable at annoq.org (HRC) / topmed.annoq.org (TOPMed)
```

## Generated artifacts — what produces them, and where each one must be copied

The stages share **no filesystem**. Stage 1 writes files and you **copy them by hand** into the
repos that consume them; a missed copy is silent — the stages simply disagree. Exact commands:
[`/annoq-data-build`](../.claude/skills/annoq-data-build/SKILL.md).

### Step numbering differs between docs — map it first

The same work carries different "Part" numbers in each document. Match by **name**, not number:

| Work | data-builder README (`issue-78` line) | data-builder README (`master`) | `/annoq-data-build` skill |
|------|---------------------------------------|--------------------------------|---------------------------|
| WGSA annotate (ANNOVAR/VEP/SnpEff) | Part 1 | Part 1 | prerequisite, not covered |
| HRC mapping columns (**TOPMed only**) | Part 2 | — | Part 2.1 |
| PANTHER / GO / Reactome / Enhancer (Java module) | Part 3 | Part 2 | Part 2 |
| Generate + distribute tree / mappings / pickle | Part 4 (+ 4.1) | Part 3 (+ 3.1) | Part 3 |

**Ordering invariant:** the HRC merge runs **after** WGSA and **before** the Java
PANTHER/enhancer step (which also does the dbNSFP `.` → `""` cleanup), and both run before
VCF→JSON conversion in stage 2.

### The artifacts

| Artifact | Produced by | Written to | Copy to |
|----------|-------------|------------|---------|
| `panther_annot.json` | `tools/api_extractor/panther_gene_extractor.py -o` (PANTHER/enhancer step, first) | the path you pass to `-o` | the path in `file.panther.annot` of `java_wgsa_add/add_panther_enhancer/src/main/resources/add_panther_enhancer.properties` — **input** to the Java module, not shipped anywhere |
| **`panther_terms.json`** | the Java module `add_panther_enhancer` — `ProcessVCFParallel <inputDir> <outputDir> <workingDir>` | `<workingDir>` (the "diagnostics" dir) | **both site repos** — `annoq-site-v2/src/data/panther_terms.json` (HRC, annoq.org) **and**, until the TOPMed cutover, `annoq-site/src/@annoq.common/data/panther_terms.json` (topmed.annoq.org) |
| annotated VCFs | the same Java module run | `<outputDir>` | nowhere — read in place by stage 2 (VCF→JSON) |
| `merge_hrc_topmed_stats.json` | `wgsa_add/merge_hrc_topmed.py` (HRC-mapping step, TOPMed only) | `<output_dir>` | nowhere — mapping counts only |
| `anno_tree.json` | `python3 -m tools.annotation_tree_gen --output_json` | the path you pass | `annoq-api-v2/data/anno_tree.json` |
| `api_mapping_anno_tree.json` | `tools.annotation_tree_gen --api_mappings_json` | the path you pass | `annoq-api-v2/data/api_mapping_anno_tree.json` |
| `annoq_mappings.json` | `tools.annotation_tree_gen --mappings_json` | the path you pass | `annoq-database/data/annoq_mappings.json` |
| `doc_type.pkl` | `tools/mappings_data_type_gen.py -o/--output` | the path you pass | `annoq-database/data/doc_type.pkl` |

### The one input nothing generates

`annotation_tree.csv` is **hand-maintained** — both generators read it (`--input_csv` /
`--input`). Until the switchover it exists in **both site repos**:
`annoq-site/metadata/annotation_tree.csv` (**authoritative — pass this one to the generators**)
and the `annoq-site-v2/metadata/annotation_tree.csv` replica. Edit both, keep them identical.

### Two outputs are throwaway — never copy them back

- `annotation_tree_gen --output_csv` (`annotation_tree_output.csv`) — **never** overwrite
  `annotation_tree.csv` with it; fields get lost. Hence the documented `/do/not/use/` path.
- `mappings_data_type_gen.py --anno_tree` (`do_not_use_anno_tree.json`) — **never** overwrite
  `anno_tree.json` with it; that script does not generate every field. Only
  `annotation_tree_gen --output_json` produces the real `anno_tree.json`.

Copying is not the end: the consuming stages still have to re-run — re-create and reload the index
(2) with the new `annoq_mappings.json` + `doc_type.pkl`, regenerate GraphQL types (3), then codegen
and build **each** site (4). See the re-run matrix next.

## Re-run matrix: "I changed stage N, what must re-run?"

| Changed | Must also re-run / regenerate |
|---------|-------------------------------|
| Stage 1 (annotation content or field set) | Regenerate mappings/tree → re-index (2) → regenerate GraphQL types (3) → codegen + UI (4) |
| Stage 1 (mapping/tree only, same data) | Re-create index & re-index (2) → regenerate types (3) → codegen (4) |
| Stage 2 (indexing logic, same schema) | Re-index only; downstream unaffected if schema unchanged |
| Stage 2 (ES mapping/schema) | Regenerate GraphQL types (3) → codegen (4) |
| Stage 3 (resolvers/schema) | Regenerate/verify GraphQL types → codegen client (4) → check **consumers** (annoq-py, AnnoQR, SNPWay) |
| Stage 3 (internal only, schema unchanged) | Redeploy API; UI unaffected |
| Stage 3 (pagination / field limits) | Review consumers that encode those limits (annoq-py: 10k / 20 fields) |
| Stage 4 (UI only) | Build/deploy the affected site only — **both site repos** if the change should apply to both stacks |

**Stage 4 is split by stack.** **annoq-site-v2** (React + TypeScript) is **released** and serves
**annoq.org (HRC r1.1)**: `npm run graphql_codegen` → `npm run test` → `npm run build` (codegen
**must** precede the build's typecheck). **annoq-site** (Angular 9) is **superseded on HRC but
still the TOPMed beta UI** at **topmed.annoq.org**: `npm run graphql_codegen` → `ng build`. Until
the **TOPMed cutover**, a stage-4 change generally has to be made and deployed in **both repos**.

**api-v2 is a shared contract for more than the site.** Its consumers — `annoq-py` (Python),
`AnnoQR` (R), and `Annoq_Overrepr_Workflow` / SNPWay (snpway.annoq.org) — all query it. A schema,
field-name, annotation-tree, or pagination/limit change can break them too, not just annoq-site.
Include them in impact analysis for any stage-3 change.

## Tracing a value backwards (debugging)

Start where the symptom appears and walk upstream until the value is correct:

1. **UI wrong** → note the URL first: annoq.org is **annoq-site-v2** (React), topmed.annoq.org is
   **annoq-site** (Angular 9). Check that repo's component/query (4). Does the GraphQL playground
   return the right value? If yes, the bug is in the UI.
2. **GraphQL wrong/missing** → check api-v2 (3). Is the field in the generated schema? Query
   Elasticsearch directly (Kibana / `_search`). If ES is right, the bug is in the API layer.
3. **ES wrong/missing** → check annoq-database indexing (2). Does the source JSON/VCF hold the
   right value? If yes, the bug is in conversion/indexing or the mapping.
4. **Source data wrong** → check annoq-data-builder (1). The annotation was built incorrectly.

## Local dev quickstart per stage

| Stage | Bring-up |
|-------|----------|
| api-v2 (3) | Bring ES up in **annoq-database** (`docker-compose up -d`), then `uvicorn src.main:app --reload --port 8001` **from the api-v2 repo root**; open `/docs` and the GraphQL playground |
| site — HRC (4) | **annoq-site-v2:** `npm install` → `npm run graphql_codegen` → `npm run dev` → `localhost:5173` (Node 20+); endpoint via `src/lib/environment.ts` or `VITE_ANNOQ_API_V2` |
| site — TOPMed (4) | **annoq-site:** `npm install` → `ng serve` → `localhost:4205`; point env at local or prod api-v2 |
| database (2) | Requires a reachable Elasticsearch; run `scripts/run_es_job.sh` against sample JSON |
| data-builder (1) | HPC/SLURM environment; heaviest to run — usually only the artifact generators are run locally |

## Environments

The pipeline runs as **two parallel stacks**, each with its own api-v2 instance and its own
database/ES instance (see [architecture.md](architecture.md#parallel-deployment-stacks-hrc--topmed)):

| Stack | Site | Site repo | api-v2 endpoint | Dataset | Code refs |
|-------|------|-----------|-----------------|---------|-----------|
| HRC (production) | <https://annoq.org> | **annoq-site-v2** (React) | <https://api-v2.annoq.org> | HRC r1.1 | default branch (`master`; `main` in annoq-site-v2) |
| TOPMed (beta) | <https://topmed.annoq.org> | **annoq-site** (Angular 9) | <https://api-v2.topmed.annoq.org> | TOPMed: Freeze 8 | **issue-19 line** deployed; **issue-78 line** in flight ([refs](architecture.md#topmed-refs--what-topmedannoqorg-is-built-from)) |

Plus:
- **SNPWay:** <https://snpway.annoq.org>
- **Local API:** `uvicorn src.main:app --port 8001` from the annoq-api-v2 root (no compose file there)
- **Local UI:** `localhost:5173` (annoq-site-v2, `npm run dev`) · `localhost:4205` (annoq-site, `ng serve`)

> Both stacks currently serve **SNPs only (no indels)**. When debugging, first note **which
> stack** the report is about (annoq.org vs topmed.annoq.org) — the two stacks have separate
> api-v2 and database instances **and separate UI codebases** (annoq-site-v2 vs annoq-site), and
> may run different data/schema versions, so a value can differ between them and still be correct.
> For TOPMed also ask **which line**: topmed.annoq.org serves the **issue-19** branches, while the
> **issue-78** branches (HRC-mapping search, new columns) are not deployed there yet — so a field
> that exists in the code may legitimately be absent from the live site.

Always confirm which api-v2 endpoint the site is pointed at before debugging a UI issue —
a "bug" is often just the dev site talking to prod (or vice versa).
