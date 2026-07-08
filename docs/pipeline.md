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
   │        artifacts: annoq_mapping.json, doc_type.pkl, anno_tree.json
   ▼  [annoq-database]
 (2) convert VCF/TSV → JSON (add unique id = chrom+pos+ref+alt)
     create ES index from annoq_mapping.json
     bulk-load JSON into Elasticsearch 8.5
   │        artifact: populated ES indices
   ▼  [annoq-api-v2]
 (3) generate GraphQL types from ES schema
     serve FastAPI + Strawberry GraphQL
   │        artifact: GraphQL endpoint (api-v2.annoq.org)
   ▼  [annoq-site]
 (4) codegen GraphQL client, render Angular UI
   │
   ▼
 value visible / queryable at annoq.org
```

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
| Stage 4 (UI only) | Build/deploy site only |

**api-v2 is a shared contract for more than the site.** Its consumers — `annoq-py` (Python),
`AnnoQR` (R), and `Annoq_Overrepr_Workflow` / SNPWay (snpway.annoq.org) — all query it. A schema,
field-name, annotation-tree, or pagination/limit change can break them too, not just annoq-site.
Include them in impact analysis for any stage-3 change.

## Tracing a value backwards (debugging)

Start where the symptom appears and walk upstream until the value is correct:

1. **UI wrong** → check the Angular component/query (4). Does the GraphQL playground return
   the right value? If yes, the bug is in the UI.
2. **GraphQL wrong/missing** → check api-v2 (3). Is the field in the generated schema? Query
   Elasticsearch directly (Kibana / `_search`). If ES is right, the bug is in the API layer.
3. **ES wrong/missing** → check annoq-database indexing (2). Does the source JSON/VCF hold the
   right value? If yes, the bug is in conversion/indexing or the mapping.
4. **Source data wrong** → check annoq-data-builder (1). The annotation was built incorrectly.

## Local dev quickstart per stage

| Stage | Bring-up |
|-------|----------|
| api-v2 (3) | `docker-compose up` with sample data from annoq-database; open `/docs` and the GraphQL playground |
| site (4) | `npm install` → `ng serve` → `localhost:4205`; point env at local or prod api-v2 |
| database (2) | Requires a reachable Elasticsearch; run `run_es_job.sh` against sample JSON |
| data-builder (1) | HPC/SLURM environment; heaviest to run — usually only the artifact generators are run locally |

## Environments

The pipeline runs as **two parallel stacks**, each with its own api-v2 instance and its own
database/ES instance (see [architecture.md](architecture.md#parallel-deployment-stacks-hrc--topmed)):

| Stack | Site | api-v2 endpoint | Dataset | Branch |
|-------|------|-----------------|---------|--------|
| HRC (production) | <https://annoq.org> | <https://api-v2.annoq.org> | HRC r1.1 | `main` |
| TOPMed (beta) | <https://topmed.annoq.org> | <https://api-v2.topmed.annoq.org> | TOPMed: Freeze 8 | TopMed branch |

Plus:
- **SNPWay:** <https://snpway.annoq.org>
- **Local API:** Docker Compose (see annoq-api-v2)
- **Local UI:** `localhost:4205`

> Both stacks currently serve **SNPs only (no indels)**. When debugging, first note **which
> stack** the report is about (annoq.org vs topmed.annoq.org) — the two stacks have separate
> api-v2 and database instances and may run different data/schema versions, so a value can differ
> between them and still be correct.

Always confirm which api-v2 endpoint the site is pointed at before debugging a UI issue —
a "bug" is often just the dev site talking to prod (or vice versa).
