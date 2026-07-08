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
  behind the **production** annoq-site at annoq.org.
- **TOPMed: Freeze 8** — the TOPMed program's Freeze 8 dataset; the dataset behind the **beta**
  annoq-site at topmed.annoq.org (served from a TopMed branch of the annoq-site codebase).
- **Production deployment** — annoq.org (HRC r1.1).
- **Beta deployment** — topmed.annoq.org (TOPMed Freeze 8).
- **SNP-only** — a property of the currently deployed datasets: they contain SNPs, not indels.
- **Deployment stack** — a full vertical slice of the pipeline (data → database/ES → api-v2 →
  site) for one dataset, with its own code branch and its own running api-v2 and database/ES
  instances. There are two: the **HRC stack** (production, `main`) and the **TOPMed stack**
  (beta, TopMed branch). They run on separate infrastructure and can carry different versions.
- **Branch line** — the per-stack branch that runs across multiple repos (data-builder, api-v2,
  annoq-site): `main` for HRC, a TopMed branch for TOPMed.

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
- **ES mapping** — the index schema (`annoq_mapping.json`): field names, types, analyzers.
- **Kibana** — visualization/ops UI for Elasticsearch.
- **Logstash** — log/data-ingestion tooling configured in annoq-database.
- **SLURM** — HPC batch job scheduler used by data-builder and database stages.
- **FastAPI** — Python web framework hosting the API service.
- **Strawberry** — Python GraphQL library used by api-v2 to define the schema.
- **GraphQL** — the query language/protocol exposed by api-v2 and consumed by the site.
- **datamodel-codegen** — generates Python models from schema; used to dynamically build the
  500+ GraphQL types.
- **graphql_codegen.ts** — generates typed GraphQL client operations for the Angular site.
- **Angular 9** — the frontend framework for annoq-site.
- **Docker / Docker Compose** — containerization for api-v2 local/prod deployment.

## Repositories & shorthand

Core pipeline:
- **Stage 1 / build** — annoq-data-builder
- **Stage 2 / index** — annoq-database
- **Stage 3 / API** — annoq-api-v2 (current API)
- **Stage 4 / UI** — annoq-site (current UI, Angular 9)

Other repos:
- **annoq-api** — original Flask/REST API; **deprecated**, replaced by annoq-api-v2.
- **annoq-py** — Python client library for the API + SNPWay.
- **AnnoQR** — R client package for the API + SNPWay.
- **Annoq_Overrepr_Workflow** — the SNPWay app (snpway.annoq.org), uses api-v2.
- **annoq-site-v2** — React rewrite of the UI; **not yet released**.
