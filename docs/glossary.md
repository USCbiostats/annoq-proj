# Glossary

Domain and technical terms used across the AnnoQ pipeline.

## Genomics / domain

- **AnnoQ** — the platform: annotates human genomic variants with 500+ functional attributes
  and serves them via API and web UI.
- **Variant** — a difference from the reference genome at a position. The AnnoQ pipeline can
  process **SNVs** and **indels**, but the **currently deployed datasets contain SNPs only**.
- **SNV (Single Nucleotide Variant)** / **SNP (Single Nucleotide Polymorphism)** — a single-base
  substitution; the only variant type in the currently deployed AnnoQ datasets.
- **Indel** — an insertion or deletion of bases. Supported by the pipeline but **not present in
  the currently deployed datasets**.
- **VCF (Variant Call Format)** — the standard text format for describing variants; the raw
  input to the pipeline.
- **TSV** — tab-separated form of variant/annotation data used during conversion.
- **Annotation** — functional information attached to a variant (predicted effect,
  conservation score, protein function, regulatory role, etc.).
- **Unique id** — AnnoQ's per-variant document id, formed from
  `chromosome + position + reference + alternate`.
- **RSID** — a dbSNP reference SNP identifier (e.g. `rs12345`); one of the supported query modes.
- **SNPWay** — the SNP→gene mapping + PANTHER overrepresentation workflow; served at
  snpway.annoq.org and also exposed via the annoq-py and AnnoQR clients.
- **Overrepresentation analysis** — functional enrichment testing that finds gene functions
  statistically over-represented in a gene set (via PantherDB); the core of SNPWay.
- **PantherDB** — the PANTHER database/service used for overrepresentation testing.

## Datasets & deployments

- **HRC r1.1** — Haplotype Reference Consortium reference panel, version r1.1; the dataset
  behind the **production** UI (**annoq-site-v2**, React) at annoq.org.
- **TOPMed: Freeze 8** — the TOPMed program's Freeze 8 dataset; the dataset behind the **beta**
  UI (**annoq-site**, Angular 9) at topmed.annoq.org (TOPMed work lands on issue branches of the
  annoq-site codebase — there is no standing `TopMed` branch).
- **Production deployment** — annoq.org (HRC r1.1), served by **annoq-site-v2**.
- **Beta deployment** — topmed.annoq.org (TOPMed Freeze 8), served by **annoq-site**.
- **SNP-only** — a property of the currently deployed datasets: they contain SNPs, not indels.
- **Deployment stack** — a full vertical slice of the pipeline (data → database/ES → api-v2 →
  site) for one dataset, with its own code branch and its own running api-v2 and database/ES
  instances. There are two: the **HRC stack** (production, default branches) and the **TOPMed
  stack** (beta, issue branches). They run on separate infrastructure and can carry different
  versions.
- **Code refs (per stack)** — HRC tracks each repo's **default branch**: `master` in
  annoq-data-builder / annoq-database / annoq-api-v2 / annoq-site, `main` in annoq-site-v2.
  TOPMed has **no standing `TopMed` branch**; its work lands on **named issue branches** in two
  lines (see [architecture.md](architecture.md#topmed-refs--what-topmedannoqorg-is-built-from)).
  The older docs called this a "TopMed branch line"; that branch does not exist. Verify with
  `git ls-remote --heads`.
- **issue-19 line** — the TOPMed refs **topmed.annoq.org is deployed from**:
  `issue-19-load-topmed` (annoq-site) + `annoq-site-19-add-update-metadata-for-top-med-data`
  (data-builder / database / api-v2). Owning issue annoq-site#19 "Load TOPMed data to the
  elasticsearch database" (closed).
- **issue-78 line** — the TOPMed refs carrying all post-release work, **not yet deployed**:
  `issue-78-add-hrc-mapping-info` (annoq-site) + `annoq-site-78-add-hrc-mapping-info`
  (data-builder / database / api-v2). Owning issue annoq-site#78 "Integrate TopMed website into
  Annoq.org" (open) — the **TOPMed-cutover umbrella**, so this is the **cutover work line**; the
  HRC-mapping search is one task under it, which is why the branches carry that name.
- **Stage-4 split** — the two stacks run **different UI codebases**: annoq-site-v2 (React) on
  annoq.org, annoq-site (Angular 9) on topmed.annoq.org. So a UI change is implemented twice,
  in two frameworks, until the **TOPMed cutover** consolidates them onto one site.
- **TOPMed cutover** — the consolidation tracked by
  [annoq-site#78](https://github.com/USCbiostats/annoq-site/issues/78) "Integrate TopMed website
  into Annoq.org" (open), the **umbrella issue** for the work. **End state (per the
  implementation): a single site serving TOPMed with HRC as a filter** — not two sites, and not
  two user-facing datasets. HRC stops being a separate deployment and becomes a
  **filter over TOPMed** (`Mapped_in_HRC=Y`, exposed by api-v2's
  `search_hrc` argument). It is **in progress, not planned**: the
  [issue-78 line](architecture.md#topmed-refs--what-topmedannoqorg-is-built-from) *is* the cutover
  work line. Note the issue's one-line body ("Maybe button to switch between websites") is an
  initial January note, **not** the design — go by the implementation.

## Annotation tools & sources

- **WGSA** — Whole-Genome Sequence Annotation pipeline (v0.95 here); orchestrates multiple
  annotation tools over VCF files.
- **ANNOVAR** — variant annotation tool integrated by WGSA.
- **VEP (Variant Effect Predictor)** — Ensembl's variant effect annotation tool.
- **SnpEff** — variant annotation and effect-prediction tool.
- **PANTHER** — protein classification system; source of protein-function annotations
  (pulled via the PANTHER API).
- **Enhancer annotations** — regulatory-region annotations added alongside PANTHER data.
- **Annotation tree** — hierarchical categorization of annotation fields
  (`anno_tree.json` / `api_mapping_anno_tree.json`) exposed to clients.

## Infrastructure & tech

- **Elasticsearch (ES)** — the search/index engine storing annotated variant documents
  (tested on v8.5.x); the source of truth for the API.
- **ES mapping** — the index schema (`annoq_mappings.json`): field names, types, analyzers.
- **Kibana** — visualization/ops UI for Elasticsearch.
- **Logstash** — log/data-ingestion tooling configured in annoq-database.
- **SLURM** — HPC batch job scheduler used by data-builder and database stages.
- **FastAPI** — Python web framework hosting the API service.
- **Strawberry** — Python GraphQL library used by api-v2 to define the schema.
- **GraphQL** — the query language/protocol exposed by api-v2 and consumed by the site.
- **datamodel-codegen** — generates Python models from schema; used to dynamically build the
  500+ GraphQL types.
- **graphql_codegen.ts** — generates typed GraphQL client operations against the api-v2 schema;
  used by both stage-4 site repos (`npm run graphql_codegen`).
- **Angular 9** — the frontend framework for annoq-site, the **TOPMed beta UI**.
- **React** — the frontend framework for **annoq-site-v2**, the **HRC production UI** at annoq.org
  (built with Vite, tested with Vitest, Node 20+); also used by the SNPWay frontend.
- **Vite** — the build/dev tooling for annoq-site-v2 (`npm run dev` on port 5173); dataset and API
  endpoint come from `src/lib/environment.ts`, overridable via `VITE_ANNOQ_API_V2`.
- **Docker / Docker Compose** — containerization. **All database containers (Elasticsearch,
  Kibana) belong to annoq-database's compose stack**; api-v2 ships a `Dockerfile` but no compose
  file and is run locally with `uvicorn`.

## Repositories & shorthand

Core pipeline:
- **Stage 1 / build** — annoq-data-builder
- **Stage 2 / index** — annoq-database
- **Stage 3 / API** — annoq-api-v2 (current API)
- **Stage 4 / UI** — **split by stack:** annoq-site-v2 (React) on annoq.org (HRC);
  annoq-site (Angular 9) on topmed.annoq.org (TOPMed)

Other repos:
- **annoq-api** — original Flask/REST API; **deprecated**, replaced by annoq-api-v2.
- **annoq-py** — Python client library for the API + SNPWay.
- **AnnoQR** — R client package for the API + SNPWay.
- **Annoq_Overrepr_Workflow** — the SNPWay app (snpway.annoq.org), uses api-v2.
- **annoq-site-v2** — React + TypeScript rewrite of the UI; **released** and serving
  **annoq.org (HRC r1.1)** as stage 4.
- **annoq-site** — Angular 9 UI; **superseded on HRC but still the TOPMed beta UI** at
  topmed.annoq.org, pending the TOPMed cutover.
