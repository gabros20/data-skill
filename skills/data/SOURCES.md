# Sources

The `data` skill was built from a 4-channel research pass (skills marketplaces, GitHub, practitioner
discourse, and canonical docs/writings), 2026-07-25 — full source-linked corpora live in this repo's
`research/` directory. Sources are grouped by topic below with their license where it affects reuse.
**All tool versions, warehouse prices, spec revisions, and licenses were verified against live
sources on 2026-07-25 and are date-stamped in the references — re-verify before relying on any of
them (this layer churns quarterly). Never trust a repo's API license field; read the LICENSE file.**

## Event taxonomy & tracking
- Segment/Twilio Spec (6 call types, reserved properties) — https://www.twilio.com/docs/segment/connections/spec
- Avo — naming conventions & tracking-plan design — https://www.avo.app/docs/data-design/best-practices/naming-conventions
- Snowplow — tracking design + SchemaVer — https://docs.snowplow.io/docs/fundamentals/tracking-design-best-practice/ (**pipeline components = SLULA v1.1, non-OSI; the meta-repo's root Apache-2.0 LICENSE is stale; `iglu-central` schemas are genuinely Apache-2.0**)
- Segment Protocols (validate/quarantine; paid addon) — https://www.twilio.com/docs/segment/protocols
- Google Consent Mode — https://developers.google.com/tag-platform/security/guides/consent · GDPR cookies — https://gdpr.eu/cookies/ · CCPA — https://oag.ca.gov/privacy/ccpa

## Data contracts & schema evolution
- Open Data Contract Standard v3.1.0 (Bitol / LF AI & Data, Apache-2.0; ex-PayPal) — https://bitol-io.github.io/open-data-contract-standard/
- Data Contract Specification (MIT; **deprecated in favor of ODCS, supported until end-2026**) + datacontract-cli (MIT) — https://github.com/datacontract/datacontract-cli
- Confluent Schema Registry — compatibility modes — https://docs.confluent.io/platform/current/schema-registry/fundamentals/schema-evolution.html (server = Confluent Community License, non-OSI; `client-*`/`avro-*` = Apache-2.0)
- Chad Sanderson — "The Rise of Data Contracts" (2022) · "Shift Left Manifesto" v1/v2 (2025/2026) · "The True Cost of Data Debt" — https://dataproducts.substack.com/ (note: the X handle @ChadSanderson is an impostor; cite the Substack)
- Counterweight: Zach Wilson — "data terms that are mostly hype… learn write-audit-publish + quality checks" — https://x.com/Zachly/status/1773491085777797167

## Ingestion & movement
- Airbyte protocol / CDC / resumability docs — https://github.com/airbytehq/airbyte (**platform ELv2; connectors + CDK MIT**)
- Meltano + Singer spec — https://github.com/meltano/meltano (MIT) · dlt — https://github.com/dlt-hub/dlt (Apache-2.0; evolve/freeze/discard schema contracts)
- Debezium (CDC + Outbox pattern) — https://github.com/debezium/debezium (Apache-2.0) · Estuary Flow — https://github.com/estuary/flow (**BSL 1.1 → Apache-2.0 on 2029-11-20**)
- Redpanda Connect, ex-Benthos — https://github.com/redpanda-data/connect (**Apache-2.0 + RCL split; no root LICENSE file**) · Multiwoven reverse-ETL — https://github.com/Multiwoven/multiwoven (**AGPLv3**)
- Fivetran ETL-vs-ELT + pricing/activations (Census acquired 2025-05-01) — https://www.fivetran.com/

## Transformation, modeling & SQL
- dbt Labs — How We Structure — https://docs.getdbt.com/best-practices/how-we-structure/1-guide-overview · Materializations/Incremental — https://docs.getdbt.com/docs/build/materializations · How We Style Our SQL — https://docs.getdbt.com/best-practices/how-we-style/2-how-we-style-our-sql
- dbt-project-evaluator (lint-rule decision tables) — https://github.com/dbt-labs/dbt-project-evaluator · dbt-utils — https://github.com/dbt-labs/dbt-utils (Apache-2.0)
- dbt licensing — https://www.getdbt.com/licenses-faq (**dbt Core Apache-2.0 "indefinitely"; dbt Fusion = ELv2/proprietary, free-to-use ≠ open-source**)
- SQLMesh — https://github.com/TobikoData/sqlmesh (Apache-2.0, LF) · SQLFluff — https://github.com/sqlfluff/sqlfluff (MIT) · dbt-expectations (Apache-2.0, **unmaintained — flagged**)

## Metrics, semantic layer & BI
- MetricFlow / dbt Semantic Layer — https://github.com/dbt-labs/metricflow (**license history AGPL→BSL→Apache-2.0 v0.209+; README "source-available" header is stale**)
- Cube (headless BI) — https://docs.cube.dev/docs/introduction (Apache-2.0/MIT per package) · Malloy — https://github.com/malloydata/malloy (MIT) · LookML — https://docs.cloud.google.com/looker/docs/what-is-lookml
- Evidence — https://github.com/evidence-dev/evidence (MIT) · Rill (agent-first BI-as-code) — https://github.com/rilldata/rill (Apache-2.0) · Lightdash · Metabase (**AGPL-3.0 + commercial**) · Superset (Apache-2.0)
- benn stancil — metrics-layer / Is BI dead? / Down with the DAG / Disband the analytics team — https://benn.substack.com/

## Data quality, testing & lineage
- Great Expectations — https://github.com/great-expectations/great_expectations (Apache-2.0; the canonical check taxonomy)
- Soda Core — https://github.com/sodadata/soda-core (**ELv2, non-OSI; API license field shows NOASSERTION**) · Elementary + dbt-data-reliability — https://github.com/elementary-data/ (Apache-2.0; taxonomy lives in the separate dbt package)
- Deequ / DQDL — https://github.com/awslabs/deequ (Apache-2.0) · pandera (MIT) · whylogs (Apache-2.0) · Recce PR-time diff — https://github.com/DataRecce/recce (Apache-2.0) · data-diff (MIT, **archived 2024**)
- OpenLineage spec — https://github.com/OpenLineage/OpenLineage (Apache-2.0) · Marquez — https://github.com/MarquezProject/marquez (Apache-2.0; no default auth)
- Monte Carlo — data downtime / five pillars — https://montecarlo.ai/blog-what-is-data-observability/
- dbt tests / unit tests — https://docs.getdbt.com/docs/build/data-tests · https://docs.getdbt.com/docs/build/unit-tests

## Governance, PII & catalogs
- Snowflake classification/tag-based masking — https://docs.snowflake.com/en/user-guide/governance-classify · BigQuery policy tags — https://docs.cloud.google.com/bigquery/docs/column-level-security-intro · Unity Catalog — https://docs.databricks.com/en/data-governance/unity-catalog/index.html
- GDPR Art.17 (erasure + exceptions) — https://gdpr-info.eu/art-17-gdpr/
- Presidio — https://github.com/microsoft/presidio (MIT; **mid-transition to community "Data Privacy Stack" org — re-verify location**)
- DataHub (Apache-2.0, aspect model) — https://github.com/datahub-project/datahub · OpenMetadata (Apache-2.0, native tests) — https://github.com/open-metadata/OpenMetadata · Atlas (Apache-2.0) · grai-core (MIT-0)
- Zhamak Dehghani — data mesh principles — https://martinfowler.com/articles/data-mesh-principles.html · dbt Mesh — https://docs.getdbt.com/best-practices/how-we-mesh/mesh-1-intro
- Mesh backlash: Erik Bernhardsson — https://x.com/bernhardsson/status/1400461693197570051

## Experiment measurement
- Kohavi et al., Trustworthy Online Controlled Experiments (SRM, OEC, guardrails, peeking) — https://exp-platform.com/
- Vermeer/Anderson/Acebal — Automated SRM Detection — https://www.lukasvermeer.nl/publications/2022/07/06/automated-srm-detection-and-analysis.html
- CUPED + sequential testing — https://docs.growthbook.io/statistics/cuped · https://docs.statsig.com/stats-engine/sequential-testing · https://docs.geteppo.com/statistics/confidence-intervals/
- GrowthBook — https://github.com/growthbook/growthbook (**MIT core + enterprise dirs**) · Spotify Confidence (Apache-2.0) · PlanOut (BSD, archived — deterministic assignment model)

## Platform, cost & small data
- Jordan Tigani — "Big Data is Dead" — https://motherduck.com/blog/big-data-is-dead/ · Hannes Mühleisen — "The Lost Decade of Small Data?" — https://duckdb.org/2025/05/19/the-lost-decade-of-small-data.html
- Frank McSherry — "Scalability! But at what COST?" (2015) — https://www.frankmcsherry.org/graph/scalability/cost/2015/01/15/COST.html
- BigQuery pricing — https://cloud.google.com/bigquery/pricing · Snowflake warehouses — https://docs.snowflake.com/en/user-guide/warehouses-overview
- Apache Iceberg (Apache-2.0) — https://iceberg.apache.org/ · Delta Lake + UniForm (Apache-2.0) — https://delta.io/ · DuckDB (MIT) — https://duckdb.org/
- Orchestrators: Airflow — https://airflow.apache.org/ · Dagster · Prefect · Kestra (all Apache-2.0). **Prefect acquired Dagster Labs, announced 2026-07-13** — https://dagster.io/blog

## AI-era data plane (seam with the `ai` skill)
- Databricks AI Cookbook — governed RAG data pipeline — https://learn.microsoft.com/en-us/azure/databricks/generative-ai/tutorials/ai-cookbook/quality-data-pipeline-rag
- Langfuse tracing data model — https://langfuse.com/docs/tracing (**MIT core + `ee/` enterprise**) · OpenLLMetry `gen_ai.*` conventions — https://github.com/traceloop/openllmetry (Apache-2.0) · Arize Phoenix (**ELv2**)
- Feast (point-in-time correctness) — https://github.com/feast-dev/feast (Apache-2.0)

## Ecosystem skills studied (structure/prior art; content not lifted verbatim unless licensed)
- dbt-labs/dbt-agent-skills · posthog/skills (MIT) · astronomer/agents (Apache-2.0) · google/skills — vendor packs (patterns studied: STOP gate, triage triad, sync-type table, warehouse-init, blast-radius workflow)
- wshobson/agents (MIT) · github/awesome-copilot (MIT) · jeffallan/claude-skills (MIT, router idiom) · gordonmurray/data-engineering-skills (MIT, update-checklist footer)
- **Not lifted (unlicensed):** posthog/posthog-for-claude, lyndonkl/claude — no LICENSE file; facts re-expressed, nothing copied.

## Method
Discovery: find-skills sweep (40+ queries), github-relay (52 shards → 794-repo corpus), x-relay
(888 tweets → 126 curated), web canon (4 clusters). Deep-reads: 13 Sonnet workers under 4 Opus
leads. Corpora + synthesis: `research/` in this repo. Everything volatile is date-stamped
2026-07-25 with re-verify pointers.
