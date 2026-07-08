---
name: Bug report
about: Something in the AnnoQ pipeline is producing a wrong or missing result
title: "[bug] "
labels: bug
---

<!-- Companion to the /annoq-bugfix Claude skill. Fill in what you can; the skill
     traces the value backwards to localize the failing stage. -->

## Summary
<!-- One sentence: what's wrong. -->

## Where the symptom appears
- [ ] Stage 1 — annoq-data-builder (annotation content / generated artifacts)
- [ ] Stage 2 — annoq-database (conversion / Elasticsearch index)
- [ ] Stage 3 — annoq-api-v2 (GraphQL / API)
- [ ] Stage 4 — annoq-site (web UI)
- [ ] Consumer — annoq-py (Python client)
- [ ] Consumer — AnnoQR (R client)
- [ ] Consumer — Annoq_Overrepr_Workflow / SNPWay (snpway.annoq.org)
- [ ] Consumer — annoq-site-v2 (React, unreleased)
- [ ] Not sure — needs localization

## Reproduction
<!-- Exact variant / query / URL. Prod (annoq.org / api-v2.annoq.org) or local? -->
- Variant / query / URL:
- Environment (prod / local):
- If UI: which api-v2 endpoint is it pointed at?

## Expected vs actual
- **Expected:**
- **Actual:**

## Backward-trace results (if known)
<!-- Does the value look correct in: GraphQL playground? Elasticsearch/Kibana? source JSON? -->
- GraphQL playground:
- Elasticsearch (`_search` / Kibana):
- Source data:

## Additional context
<!-- Logs, screenshots, versions (ES 8.5.x, Python 3.11+, Angular 9). -->
