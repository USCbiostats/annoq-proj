---
name: Feature request
about: Add a capability to the AnnoQ pipeline (annotation field, query, or UI)
title: "[feature] "
labels: enhancement
---

<!-- Companion to the /annoq-feature Claude skill, which plans and sequences
     cross-repo work upstream → downstream. -->

## Summary
<!-- What should AnnoQ be able to do that it can't today? -->

## Feature kind
- [ ] New annotation field / source (spans all stages 1→4)
- [ ] New query / filter / aggregation over existing data (stage 3, maybe 2 & 4)
- [ ] Search / UX capability over existing API (stage 4)
- [ ] Performance / indexing capability (stage 2, maybe 1)
- [ ] Expose an already-indexed field (stages 3→4)

## Stages / repos likely affected
- [ ] Stage 1 — annoq-data-builder
- [ ] Stage 2 — annoq-database
- [ ] Stage 3 — annoq-api-v2
- [ ] Stage 4 — annoq-site
- [ ] Consumer — annoq-py (Python client)
- [ ] Consumer — AnnoQR (R client)
- [ ] Consumer — Annoq_Overrepr_Workflow / SNPWay
- [ ] Consumer — annoq-site-v2 (React, unreleased)

## Shared contracts touched
<!-- Does this move a field name, ES mapping, the annotation tree, or the GraphQL schema? -->

## User story / motivation
<!-- Who needs this and why. -->

## Acceptance criteria
<!-- How we'll know it's done and correct, end-to-end. -->

## Notes / references
