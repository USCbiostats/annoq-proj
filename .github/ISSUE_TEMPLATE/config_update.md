---
name: Configuration update
about: Change a configuration value in the AnnoQ pipeline (env, mappings, compose, SLURM, endpoint)
title: "[config] "
labels: configuration
---

<!-- Companion to the /annoq-config Claude skill, which locates the config surface,
     applies the change safely, and validates it. -->

## Summary
<!-- What configuration needs to change and why. -->

## Config surface
- [ ] Stage 1 — annoq-data-builder (SLURM `sbatch`, `config.py`, resource paths, PANTHER API)
- [ ] Stage 2 — annoq-database (ES connection, index name/settings, `annoq_mappings.json`, loading strategy)
- [ ] Stage 3 — annoq-api-v2 (`docker-compose.yaml`, env, `anno_tree.json`, deps)
- [ ] Stage 4 — annoq-site-v2 (`src/lib/environment.ts` dataset + api-v2 URL, `VITE_ANNOQ_API_V2`, codegen target) — **annoq.org / HRC**
- [ ] Stage 4 — annoq-site (environment files / target api-v2 URL, build config, codegen target) — **topmed.annoq.org / TOPMed**
- [ ] Consumer — annoq-py / AnnoQR (API base URL, request settings)
- [ ] Consumer — Annoq_Overrepr_Workflow / SNPWay (api-v2 endpoint, backend port)

## File(s) and environment
- File(s):
- Environment (local / staging / prod):

## Change
- **Current value:**  <!-- do NOT paste secrets -->
- **New value:**

## Shared contract?
- [ ] Touches ES index/mappings, the annotation tree, or the api-v2 endpoint URL
      (requires downstream re-index / regenerate / redeploy — see docs/pipeline.md)

## Validation plan
<!-- How the change will be verified for the target environment. -->
