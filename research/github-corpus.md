# `data` skill — GitHub Source Corpus

Research-only mining of GitHub for the best data-engineering / analytics / governed-data-plane knowledge, to seed the `data` member of the Digital Product Skills family (peers: product, design, architecture, frontend, backend, ai).

**Method:** github-relay wide-net discovery (52 search shards + 2 curated `hydrate` passes → **794-repo corpus**, `research/corpus.json`), offline star-skeptical rank, then **4 parallel Sonnet deep-read workers** raw-fetching READMEs / docs / spec files / LICENSE files (no relay account contention). Every version/license/"latest" claim is linked to its exact source and **date-stamped as-of 2026-07-25**. Nothing valuable lives only in an agent's head — it is here or linked.

**Anti-hallucination note:** GitHub's API `license` field is *unreliable* for the modern data stack — it reports the stale root `LICENSE` and mislabels split/source-available repos (Snowplow shows "Apache-2.0" but ships SLULA; PostHog/Langfuse show "Other"). Every license below was read from the actual file, path linked.

Seam with the shipped `ai` skill: `ai` owns retrieval *behavior*; `data` owns the governed *pipeline* (chunking/embedding jobs, corpus freshness/lineage/PII) + the vector store and LLM-trace logs *as datasets*. That seam is covered in §10.

---

## 1. Event Taxonomy & Tracking Plans

### snowplow/snowplow — schema-first behavioral tracking (the flagship pattern)
`https://github.com/snowplow/snowplow` · schema registry: `https://github.com/snowplow/iglu-central`
- **Lift — Iglu self-describing schema.** Every event/entity is a versioned JSON Schema with a `self` block: `{vendor, name, format:"jsonschema", version:"MODEL-REVISION-ADDITION"}`. Naming = reverse-DNS `vendor` + snake_case `name`. `additionalProperties:false` enforced — strict contracts, **not** append-only JSON blobs.
- **Tracking plan = a curated set of these self-describing schemas** (contexts/entities) attached to a canonical event envelope (page_view, page_ping, structured event, self-describing event). Event *shape* and event *context* are **separately versioned** schemas composed at send-time.
- **Invariant — schema evolution rule:** `MODEL-REVISION-ADDITION`. Breaking change bumps MODEL (old consumers keep working on old MODEL); additive-compatible change bumps REVISION/ADDITION. This is the single cleanest evolution convention in the corpus — lift verbatim.
- **Registry mechanics:** Iglu Central is a Maven/RubyGems-like schema store (JSON Schema/Avro/Thrift), resolved client-side by a "resolver" config; PR-based dev registries via S3/raw GitHub.
- **LICENSE (CRITICAL, two-tier, changed 2024, as-of 2026-07-25):** meta-repo root LICENSE is **stale Apache-2.0**, but the actual pipeline component `snowplow/enrich` = **Snowplow Limited Use License Agreement (SLULA) v1.1, Nov 2024** — non-OSI, source-available, Non-Production/Non-Commercial only, no redistribution. Source `https://raw.githubusercontent.com/snowplow/enrich/master/LICENSE`. The schema repo `iglu-central` **is** genuinely Apache-2.0 (`https://raw.githubusercontent.com/snowplow/iglu-central/master/README.md`). **GOTCHA:** GitHub API reports the meta-repo as Apache-2.0 — misleading; it reflects only the stale root file, not the per-component pipeline license. This is the corpus's headline "stale-LICENSE trap."

### rudderlabs/rudder-server — warehouse-first CDP, Segment-compatible
`https://github.com/rudderlabs/rudder-server`
- **Lift:** Segment-API-compatible event spec (track/identify/page/screen/group) — events written for Segment work unmodified. Warehouse-first destination model; JS user-transformation framework enriches events pre-delivery.
- **LICENSE:** **Elastic License 2.0 (ELv2)** — non-OSI, source-available (no SaaS-competing use, no license-key circumvention). `https://raw.githubusercontent.com/rudderlabs/rudder-server/master/LICENSE`.

### PostHog/posthog — product analytics event model
`https://github.com/PostHog/posthog` (details in §9 — product analytics + experiments)
- **Contrast to lift:** `$autocapture` + custom named events with arbitrary JSON props — **looser than Iglu, no schema-first contract by default**; "actions" are saved filters over raw events, not a taxonomy-enforcement layer. Good teaching contrast: schema-first (Snowplow) vs capture-everything (PostHog).

---

## 2. Data Contracts

**Decision (unambiguous, as-of 2026-07-25): use ODCS.** The older datacontract.com spec (DCS) is officially deprecated in favor of ODCS.

### bitol-io/open-data-contract-standard (ODCS) — the standard going forward
`https://github.com/bitol-io/open-data-contract-standard` · current **v3.1.0** · Linux Foundation (LF AI & Data)
- **Lift — contract section list:** Fundamentals, Schema, References, Data quality, Support & communication channels, Pricing, Team, Roles, SLA, Infrastructure & servers, Custom & other properties. Official JSON Schema at `schema/odcs-json-schema-latest.json`; media type `application/odcs+yaml;version=3.1.0`.
- **Fundamentals fields:** `apiVersion`, `kind: DataContract`, `id`, `name`, `version`, `status` (proposed/draft/active/deprecated/retired), `tenant`, `tags`, `domain`, `authoritativeDefinitions`, `description.{purpose,limitations,usage}`. `dataProduct` field **deprecated since v3.1.0**.
- **Data-quality — four rule tiers (decision axis):** **Text** (human description) · **Library** (predefined metric: `nullValues`, `missingValues`, `invalidValues`, `duplicateValues`, `rowCount`, each with `mustBe`/`mustBeLessThan`/`mustBeGreaterThan`) · **SQL** (custom query) · **Custom** (vendor-specific: Soda / Great Expectations / dbt tests / Monte Carlo).
- **SLA fields:** `slaProperties[]` with `property`, `value`/`valueExt`, `unit` (ISO), `element`, and a **`driver` enum: regulatory | analytics | operational** — a decision axis for SLA priority. Plus `scheduler`/`schedule` (cron).
- **LICENSE:** **Apache-2.0** (README: "available for you as an Apache 2.0 license").

### datacontract/datacontract-specification (DCS) — deprecated but shape worth studying
`https://github.com/datacontract/datacontract-specification`
- **DEPRECATED** — README: "With the release of ODCS v3.1.0, we deprecate the Data Contract Specification… migrate to ODCS… supported in Data Contract CLI and Entropy Data until the end of 2026."
- **Lift — annotated shape (still a good field checklist):** `info` (title/version/owner/status/contact); `servers` (type/environment/location/format + per-role access description); `terms` (usage, limitations, policies incl. privacy-policy/license links, billing, `noticePeriod`); `models.fields` with `type/required/unique/primaryKey/references/pii/classification/examples/tags` + field-level `quality` (sql: `mustBeBetween`/`mustBeLessThan`) + `lineage.inputFields` (upstream provenance); `definitions` (reusable `$ref` field defs); **`servicelevels`**: availability, retention, latency (+ source/processed timestamp fields), freshness, frequency (batch/streaming, interval/cron), support (hours, responseTime), backup (recoveryTime/recoveryPoint).
- **LICENSE:** **MIT** (Copyright 2023 Data Mesh Architecture).

### datacontract/datacontract-cli — contract-as-executable-test
`https://github.com/datacontract/datacontract-cli`
- **Lift:** `datacontract test <url>` connects to the live server and runs schema+quality checks against the real dataset — the contract *is* the test. Supports both DCS and ODCS; `export --format odcs` does DCS→ODCS migration.
- **LICENSE:** **MIT** (Copyright 2025 Entropy Data GmbH). `https://raw.githubusercontent.com/datacontract/datacontract-cli/main/LICENSE`. (API shows "Other" — mismatch; file is clean MIT.)

### confluentinc/schema-registry — schema evolution & compatibility modes
`https://github.com/confluentinc/schema-registry`
- **Lift — DECISION TABLE, compatibility modes** (enumerated from `CompatibilityLevel.java`):

  | Mode | Meaning | Upgrade order |
  |---|---|---|
  | `NONE` | no checking | — (fast migrate, no external consumers) |
  | `BACKWARD` (default) | new schema reads data from *previous* schema | consumers upgrade first |
  | `BACKWARD_TRANSITIVE` | new schema reads *all* previous | consumers first |
  | `FORWARD` | *previous* schema reads data from new schema | producers upgrade first (edge devices, shipped SDKs) |
  | `FORWARD_TRANSITIVE` | *all* previous read new | producers first |
  | `FULL` | backward + forward vs previous | independent order |
  | `FULL_TRANSITIVE` | backward + forward vs *all* previous | independent order, bidirectional coupling |

- **LICENSE (split, verified):** server `core/` = **Confluent Community License v1.0** (non-OSI, source-available, no SaaS-competing offering); `client-*`/`avro-*` = **Apache-2.0**. `https://raw.githubusercontent.com/confluentinc/schema-registry/master/LICENSE` + `LICENSE-ConfluentCommunity` + `client/LICENSE`.

---

## 3. Ingestion & Movement (ETL / ELT / CDC / Streaming)

### airbytehq/airbyte — self-hosted ELT, 600+ connectors
`https://github.com/airbytehq/airbyte`
- **Lift — Airbyte Protocol (connector architecture):** Source/Destination "Actors," each a Docker image speaking JSON over STDIN/STDOUT: `spec()`, `check(config)`, `discover()→Catalog`, `read(ConfiguredCatalog, State)→AirbyteMessage stream`, `write()`. Catalog = Streams (schema + supported sync modes); ConfiguredCatalog = user-selected streams + sync mode + cursor/PK. Protocol versioned independently (semver).
- **Replication modes:** Full Refresh · Incremental (cursor) · CDC (log-based). CDC is **not** infinite streaming — each sync reads from last log position up to sync-start; first sync snapshots (`SELECT *`) before log-tailing. Adds `_ab_cdc_*` metadata columns (lsn, log_file/pos, updated_at, deleted_at).
- **Invariant — at-least-once delivery.** Never a single `SELECT *`; chunks with predicates (`CTID >= X AND CTID < Y`) so a failed sync resumes from a checkpoint ("Resumable Full Refresh").
- **Gotcha:** newly created tables put the CDC snapshot ahead of connection state → manual refresh required before CDC picks them up.
- **LICENSE (dual, confirmed):** platform/core = **Elastic License v2 (ELv2)** — free to self-host/modify, but **no** offering Airbyte as a hosted/managed service, no circumventing license-key gating, no stripping notices (since v0.30.0 / 2021-09-27). **Connectors + CDK = MIT** (open regardless of platform). `https://github.com/airbytehq/airbyte/blob/master/LICENSE` + `docs/community/licenses/{license-faq.md,mit-license.md}`.

### meltano/meltano — Singer-spec runner / declarative EL
`https://github.com/meltano/meltano`
- **Lift — Singer mechanics:** extractors (taps) run in **discovery mode** → JSON **Catalog** (streams + JSON-schema + metadata); Meltano applies `select`/`schema`/`metadata` rules on the fly → **Configured Catalog** (avoids hand-editing huge catalogs). `select` (property-level, discovered catalogs only) vs `select_filter` (stream-level, always applied) — an easily-confused distinction.
- **Replication methods** (stream metadata `replication-method`): `LOG_BASED` (binlog/WAL) · `INCREMENTAL` (bookmark col e.g. `updated_at`) · `FULL_TABLE`.
- **State:** taps emit STATE after each stream; Meltano merges by "State ID" (extractor+loader+environment) so retries resume mid-stream. **Gotcha:** some loaders only emit state after *all* work — a mid-run failure loses resumption granularity. `--full-refresh` ignores state; **changing `start_date` alone does NOT re-pull history** (state precedence wins).
- **LICENSE:** **MIT** (Copyright Arch Data, Inc.). `https://github.com/meltano/meltano/blob/main/LICENSE`.

### dlt-hub/dlt — Python EL(T) library, schema evolution flagship
`https://github.com/dlt-hub/dlt`
- **Lift — schema inference & evolution.** Infers schema from data, normalizes nested/JSON into relational tables. **Schema contracts** (three modes, per `tables`/`columns`/`data_type`): `evolve` (accept+adapt) · `freeze` (reject record) · `discard` (drop offending row/column). Declared via decorator kwargs: `primary_key=`, `write_disposition="merge"` (upsert), `columns={...}` (type/PII hints), `schema_contract={...}`.
- **Incremental:** `dlt.sources.incremental("updated_at")` injected as resource arg; `.last_value` is the bookmark; pipelines resumable via `dlt.attach(pipeline_name=...)`.
- **Read-back:** `pipeline.dataset()` → typed table access; can lift a loaded table into an **Ibis** expression compiling to the destination dialect lazily. Positions itself as LLM/agent-friendly.
- **LICENSE:** **Apache-2.0**. `https://github.com/dlt-hub/dlt/blob/devel/LICENSE.txt`.

### estuary/flow — unified batch + streaming
`https://github.com/estuary/flow`
- **Lift:** Captures (source → millisecond "collections" as JSON in your own bucket) → Derivations (stateful stream transforms/joins) → Materializations (collection → target). Built on Gazette streaming broker; **backfill-then-tail is first-class** ("seamlessly transition to low-latency streaming on reaching the present"), not bolted on.
- **LICENSE (dual, path-split):** default = **Business Source License 1.1 (BSL)**, Licensor Estuary Technologies; Additional Use Grant forbids reselling Flow as a "Data Processing Service"; **Change Date 2029-11-20 → converts to Apache-2.0** automatically. `go/protocols/` carve-out is Apache-2.0. `https://github.com/estuary/flow/blob/master/LICENSE-BSL`.

### debezium/debezium — CDC on Kafka Connect
`https://github.com/debezium/debezium`
- **Lift:** one connector per upstream DB; each DB change → a Kafka message (one topic/table), durably ordered; consumers resume exactly where they left off. **Embedded engine** runs a connector inside your app (no Kafka broker) for apps not needing durability.
- **Pattern — transactional Outbox** as first-class: an "Outbox Event Router" SMT reads an app's outbox table via CDC and republishes clean domain events — the canonical reliable microservice data-exchange pattern. `documentation/.../integrations/outbox.adoc`.
- **Gotcha:** Postgres logical-decoding plugin choice (decoderbufs default, wal2json, pgoutput) changes connector behavior/tests.
- **LICENSE:** **Apache-2.0** (Antlr grammars in `debezium-ddl-parser` separately MIT). `https://github.com/debezium/debezium/blob/main/LICENSE.txt`.

### redpanda-data/connect (formerly Benthos) — Go stream processor
`https://github.com/redpanda-data/connect`
- **Lift:** declarative single-YAML topology (inputs → processors → outputs); rich CDC connector set (Postgres/MySQL/Mongo/Oracle/MSSQL); OTel-native tracing.
- **LICENSE CHANGE (confirmed — no longer plain-MIT Benthos):** dual, directory-split. **Apache-2.0** for the majority of connectors; **RCL (Redpanda Community License)** for gated Enterprise features (source-available, non-OSI). **No top-level `LICENSE` file** — lives only in `licenses/` (a scanner gotcha). `https://github.com/redpanda-data/connect/blob/main/licenses/README.md`.

### Multiwoven/multiwoven — open-source reverse-ETL
`https://github.com/Multiwoven/multiwoven`
- **Lift:** warehouse/DB → business tools (Salesforce/HubSpot/Slack/Klaviyo/Zendesk/Ads/Sheets). Define a **Model** (SQL record set) then a **Sync** (map model fields → destination fields on a schedule). Rails control-plane + React UI + Ruby-gem integrations framework. OSS alternative to Hightouch/Census.
- **LICENSE:** **AGPLv3** — copyleft; network-use triggers source-disclosure. Materially different risk profile from everything else in this cluster — flag it. `https://github.com/Multiwoven/multiwoven/blob/main/LICENSE`.

### bruin-data/bruin — ingestion + transform + quality in one CLI
`https://github.com/bruin-data/bruin`
- **Lift:** single Go CLI unifying ingestion (`ingestr`) + SQL/Python/R transform + built-in DQ checks; runs locally / EC2 / GitHub Actions. A lightweight "dbt+Airbyte+GE-in-one" for small/mid teams; dry-run end-to-end validation.
- **LICENSE:** **Apache-2.0** (file is `LICENSE.md`). `https://github.com/bruin-data/bruin/blob/main/LICENSE.md`.

---

## 4. Transformation & Modeling (dbt / SQL / staging→marts)

### dbt-labs/dbt-core — SQL-first transformation (the T in ELT)
`https://github.com/dbt-labs/dbt-core`
- **Lift:** select-statements-as-models; DAG built via `ref()`/`source()`. `main` now hosts **dbt Core v2.0 (alpha)** — a ground-up Rust rewrite, foundation of the Fusion engine, single self-contained binary, emits Parquet artifacts (superset of `manifest.json`). Classic Python dbt-core lives on branch `1.latest`.
- **Gotcha:** v2.0 is alpha (formats/APIs may change; Windows ARM unsupported). Two distributions now share one language spec — **logic is portable, licenses differ** (see Fusion).
- **LICENSE:** **Apache-2.0**. `https://github.com/dbt-labs/dbt-core/blob/main/LICENSE`.

### dbt-labs/dbt-fusion — the next-gen engine (CRITICAL license finding)
`https://github.com/dbt-labs/dbt-fusion`
- **LICENSE (CRITICAL, mixed, NOT fully open, as-of 2026-07-25):** `LICENSES.md` states *"Unless otherwise specified, every folder and file in this repository is licensed under the **Elastic License 2.0 (ELv2)**."* Only these crates are Apache-2.0: `dbt-agate`, `dbt-auth`, `dbt-adapter`, `dbt-jinja`, `dbt-xdbc`. Everything else = **ELv2** (source-available: no hosted/managed-service offering, no removing license-key gating). `https://github.com/dbt-labs/dbt-fusion/blob/main/LICENSES.md`.
- **Skill gotcha:** don't recommend "just use Fusion, it's free" without flagging free-to-use ≠ open-source; **dbt Core (Apache-2.0) remains the fully-open path.**

### dbt-labs/dbt-project-evaluator — decision-table goldmine (promote to a primary read)
`https://github.com/dbt-labs/dbt-project-evaluator`
- **DECISION TABLE — model naming/prefix** (`docs/rules/structure.md`):

  | Model type | Prefix |
  |---|---|
  | Staging | `stg_` |
  | Intermediate | `int_` |
  | Marts | `fct_` or `dim_` |
  | Other/report | `rpt_` |

- **Structure rules (each is a lint rule):** staging nests in `staging/<source>/`; one `.yml` per source-subfolder; **co-locate tests with models** (one `.yml` per model-type folder), not centralized.
- **Modeling invariants:** staging = atomic, **1:1 with a raw source table**; a `{{ source() }}` has exactly one direct child (no "source fanout"); **only staging models reference sources directly** (no marts/int → source; no mixing model+source in one model); no hard-coded relations (always `ref()`/`source()`); **no staging→staging** deps; fold pass-through intermediate models into the child as a CTE ("rejoining of upstream concepts"); root models (0 parents) are a lineage bug unless deliberately self-contained (e.g. `date_spine` calendar).
- **Thresholds:** model fan-out >3 direct leaf children → push logic to BI or upstream; "too many joins" default 7 direct refs → split into intermediates joining 4–6 concepts; "chained view dependencies" ≥4 non-materialized models before a table → warn.
- **Materialization rule of thumb (verbatim):** *"if a view is used downstream by many models, change materialization to table"*; *"if a view has more complex calculations (window functions, joins between many tables), change materialization to table."*
- **Testing rule:** every model needs a PK test — (`unique`+`not_null`) OR `dbt_utils.unique_combination_of_columns` OR not-null+unique. **Exposures** must depend on `table`/`incremental` models, never directly on a `source`, `ephemeral`, or `view`.
- **LICENSE:** Apache-2.0 (per org convention; not independently re-read).

### dbt-labs/dbt-utils — canonical macro library
`https://github.com/dbt-labs/dbt-utils`
- **Lift — generic tests:** `equal_rowcount`, `equality`, `expression_is_true`, `recency`, `not_accepted_values`, `relationships_where`, `mutually_exclusive_ranges`, `sequential_values`, `unique_combination_of_columns`, `accepted_range`, `cardinality_equality`. **SQL generators:** `date_spine`, `deduplicate`, `star`, `union_relations`, `generate_surrogate_key`, `pivot`/`unpivot`, `safe_divide`, `width_bucket`. **Introspective:** `get_column_values`, `get_relations_by_pattern`. Dispatch-macro pattern for cross-adapter override.
- **LICENSE:** **Apache-2.0**.

### calogica/dbt-expectations — GE-style tests as dbt tests
`https://github.com/calogica/dbt-expectations`
- **Gotcha (important):** README header — *"This package is no longer actively supported."* Flag as maintenance risk; prefer native dbt tests + dbt-utils + Elementary. Categories mirror GE (table-shape, missing/unique/type, sets/ranges, regex, aggregate mean/median/stdev/quantile, multi-column, distributional `within_n_stdevs`).
- **LICENSE:** **Apache-2.0**.

### TobikoData/sqlmesh — dbt alternative, plan/apply + virtual environments
`https://github.com/TobikoData/sqlmesh`
- **Lift:** Terraform-style plan/apply; **Virtual Data Environments** (dev envs are views over the same physical tables — no compute duplication for branching); `MODEL` DDL block + trailing `SELECT` (no separate YAML); multi-dialect via SQLGlot; built-in unit tests/audits; column-level lineage/diffing.
- **DECISION TABLE — model kinds** (`docs/concepts/models/model_kinds.md`): `INCREMENTAL_BY_TIME_RANGE` (immutable time-stamped events; needs `time_column` + `@start_date`/`@end_date` WHERE; `lookback` for late data) · `INCREMENTAL_BY_UNIQUE_KEY` (upsert by key — **does NOT dedupe automatically**, query must not return dup keys) · `FULL` (full rewrite; small aggregates only) · `VIEW` (default; **re-evaluated every downstream reference** — compute gotcha) · `SCD Type 2` (auto `valid_from`/`valid_to`) · plus `EMBEDDED`, `SEED`, `EXTERNAL`, `MANAGED`, `INCREMENTAL_BY_PARTITION`.
- **Philosophy (verbatim):** *"Never build a table more than once"* (incremental-by-default vs dbt's full-refresh default mindset).
- **LICENSE:** **Apache-2.0** (code) + CC-BY-4.0 (docs). Linux Foundation project. `https://github.com/TobikoData/sqlmesh/blob/main/LICENSE`.

### sqlfluff/sqlfluff — SQL linter (28 dialects, dbt-aware)
`https://github.com/sqlfluff/sqlfluff`
- **Lift:** templater plugins — `jinja` (default), `raw`, `python`, `placeholder`, and a **dbt templater** for linting compiled dbt Jinja SQL. Rust-accelerated parser via `sqlfluff[rs]`.
- **Gotcha:** `ignore_templated_areas = True` by default → lint errors inside Jinja-generated code are **suppressed by default** (surprises dbt users expecting full coverage).
- **LICENSE:** **MIT** (`LICENSE.md`). `https://github.com/sqlfluff/sqlfluff/blob/main/LICENSE.md`.

**Worked-example references:** `dbt-labs/jaffle_shop` (canonical staging→mart: source→rename CTE in `stg_*`, multi-CTE join mart, staging `materialized: view` + default table); `dbt-labs/dbt-codegen` (generate staging boilerplate/YAML from sources); `dbt-labs/dbt-audit-helper` (diff/audit outputs during refactor); `dbt-labs/mrr-playbook` + `attribution-playbook` (worked dimensional examples); `dbt-labs/dbt-mcp` + `dbt-labs/dbt-agent-skills` (dbt agent tooling — directly relevant to this skill family).

---

## 5. Semantic / Metrics Layer

### dbt-labs/metricflow — the metrics engine dbt uses (+ Open Semantic Interchange)
`https://github.com/dbt-labs/metricflow`
- **Lift — semantic manifest spec** (from real fixtures): a `semantic_model` (name, `node_relation`, `primary_entity`, `entities` [type foreign/primary], `dimensions` [type time, `time_granularity`], measures) + `metric` definitions.
- **DECISION TABLE — metric types:**

  | Metric type | Use for | Shape |
  |---|---|---|
  | `simple` | direct aggregation of one measure | `expr` + `agg` |
  | `ratio` | business ratios (bookings-per-booker) | `numerator`/`denominator`, each with optional `filter` |
  | `derived` | algebraic combo of other metrics (`booking_value * 0.05`) | `expr` + `metrics:[]`; inputs carry own `filter`/`offset_window`/`offset_to_grain` (period-over-period) |
  | `cumulative` | trailing/to-date windows (MTD, trailing-2-mo) | `cumulative_type_params`: `window`, `grain_to_date`, `period_agg` |

  Metric filters use `{{ Dimension('booking__is_instant') }}` syntax incl. cross-entity joins via `entity_path`.
- **LICENSE HISTORY (unusually explicit, from README, as-of 2026-07-25):** *"v0–0.140.0 = Affero GPL. v0.150.0–0.208.2 = BSL. v0.209.0+ = Apache 2.0."* Current `main` LICENSE = **Apache-2.0** (`https://github.com/dbt-labs/metricflow/blob/main/LICENSE`). **Gotcha:** the README still carries a legacy "source-available software" header — don't repeat that stale characterization without the version caveat. Now anchors the cross-vendor **Open Semantic Interchange (OSI)** initiative (with Snowflake).

### cube-js/cube — headless semantic layer (SQL/REST/GraphQL/AI)
`https://github.com/cube-js/cube`
- **Lift — data-model YAML:** `cubes:` with `sql_table`, `joins` (`relationship: many_to_one`, `sql`), `measures` (type count/sum…), `dimensions` (type number/time, `primary_key`). Both YAML and JS-DSL `cube(...)` forms are first-class/equivalent.
- **Decision rule — pre-aggregations:** app DBs = low latency, don't scale; warehouses = scale, unpredictable latency; Cube's pre-agg layer (in Cube Store) bridges — condenses source data by orders of magnitude for repeat query patterns.
- **Invariant:** cube `name` globally unique across cubes+views; `sql_alias` needed when auto-alias exceeds DB identifier length.
- **LICENSE (as-of 2026-07-25):** **dual, per-package** — root LICENSE: default **Apache-2.0** unless a package header/LICENSE says otherwise; `cubejs-server` = Apache-2.0, `cubejs-client-core` = MIT. **No BSL/SSPL in the OSS repo** — the commercial split lives in the separately-hosted Cube Cloud product. `https://github.com/cube-js/cube/blob/master/LICENSE`.

### malloydata/malloy — semantic modeling language
`https://github.com/malloydata/malloy`
- **Lift — source/measure/dimension:** `source: X extend { measure: on_time_rate is count(){where:...} / count(){where:...} }`; `run: X -> { group_by:…; aggregate:… }`. Auto-orders by first aggregate (no manual `ORDER BY`). Compiles to SQL on BigQuery/Snowflake/DuckDB/Postgres/Trino/Databricks; every query leaves a reusable semantic model.
- **"Why a semantic layer" argument (verbatim-usable):** metrics like "on-time arrival" have ambiguous real-world definitions (US DOT: `arr_delay<15` excluding cancelled/diverted); a naive `count(where)/count()` silently mishandles edge cases — encoding the rule once keeps every downstream query consistent.
- Companion `malloydata/publisher` — OSS semantic-model server exposing `.malloy` over REST + MCP (agent-facing).
- **LICENSE:** **MIT**. `https://github.com/malloydata/malloy/blob/main/LICENSE`.

---

## 6. BI & Dashboards-as-Code

### evidence-dev/evidence — BI as code (markdown + SQL → static site)
`https://github.com/evidence-dev/evidence`
- **Lift:** SQL fenced blocks named inline, referenced by chart components: ` ```sql sales_by_category … ``` ` then `<LineChart data={sales_by_category}/>`. Query chaining via `${query_name}` (compiles to subquery); templated pages + loops. **Invariant:** page query order doesn't matter — compiler resolves dependencies (forward-refs fine).
- **LICENSE:** **MIT** (`LICENSE.md`; root `LICENSE` 404s).

### rilldata/rill — agent-first BI as code (OLAP-backed)
`https://github.com/rilldata/rill`
- **Lift — 3-file model:** `models/*.yaml` (connector + SQL, `materialize:true`) → `metrics/*.yaml` (`type: metrics_view`: dimensions, measures like `sum(price*quantity)`, `timeseries`) → `dashboards/*.yaml` (`type: explore`, references metrics_view). Deploy via `rill deploy` (git-backed CI/CD).
- **Distinctive:** `rill init` scaffolds **agent-context files** (`.claude`, `.mcp.json`) — designed for coding-agent-authored dashboards; per-metrics-view **Row Access Policies**; ships an MCP server for conversational querying.
- **LICENSE:** **Apache-2.0** (`LICENSE.md`).

### Other BI (from corpus, licenses via `research/corpus.json` SPDX + noted caveats)
- `apache/superset` — **Apache-2.0** — mature BI/exploration platform.
- `metabase/metabase` — **AGPL-3.0 + commercial** (API "Other") — easiest-to-use OSS BI; embedded/enterprise editions are separately licensed.
- `lightdash/lightdash` — Looker-alternative wired to dbt metrics (strong BI-as-code fit; deep dbt integration).
- `getredash/redash` — **BSD-2-Clause** — query-and-visualize any source.
- `streamlit/streamlit` — **Apache-2.0** — data apps (adjacent, not strictly BI).
- Grafana `grafana/grafana` — **AGPL-3.0** — observability/metrics dashboards.

---

## 7. Data Quality / Testing / Lineage / Observability

**The single most valuable synthesis in this corpus is the convergent DQ taxonomy** — five independent tools (GE, Soda, Elementary, Deequ, OpenMetadata) + two catalogs (DataHub, OpenLineage) evolved to nearly the same ~9 check buckets. Reproduced verbatim below.

### CONVERGENT DATA-QUALITY DECISION TABLE (cross-repo synthesis — lift wholesale)

| Failure mode | Best-fit check category | Repos expressing it |
|---|---|---|
| Schema drift (col add/remove/type) | schema/structural | GE `expect_table_columns_to_match_set`; Soda `schema:`; Elementary `schema_changes`; OpenMetadata `tableColumnToMatchSet`; DataHub `AssertionType.DATA_SCHEMA`; OpenLineage `SchemaDatasetFacet` |
| Freshness/staleness | freshness | Soda `freshness:`; Elementary `table.freshness`; Deequ DQDL `DataFreshness`; DataHub `AssertionType.FRESHNESS` |
| Volume/row-count anomaly | volume | GE `expect_table_row_count_to_be_between`; Soda `row_count:`; Elementary `table.row_count`; Deequ `RowCount`/`RowCountMatch`; OpenMetadata `tableRowCountToBeBetween`; DataHub `AssertionType.VOLUME` |
| Completeness (nulls) | column null-rate | GE `expect_column_values_to_not_be_null`; Soda `missing:`+threshold; Elementary `null_percent`; Deequ `isComplete`; OpenMetadata `columnValuesToBeNotNull` |
| Uniqueness / duplicates | uniqueness | GE `expect_column_values_to_be_unique`; Soda `duplicate_check`; Deequ `isUnique`/`IsPrimaryKey`; OpenMetadata `columnValuesToBeUnique` |
| Domain/set validity | validity | GE `expect_column_values_to_be_in_set`; Soda `invalid:{valid_values:}`; Deequ `isContainedIn`; OpenMetadata `columnValuesToBeInSet` |
| Distributional drift/outliers | statistical/anomaly | GE KS/chi-square/z-score; Elementary anomaly-scoring; Deequ anomaly-on-metrics-over-time; whylogs profile-drift constraints |
| Referential/cross-table | pair/multi-column or cross-dataset | GE `expect_column_pair_values_to_be_equal`, `expect_multicolumn_sum_to_equal`; Deequ `ColumnCorrelation`; OpenMetadata/Deequ/data-diff `tableDiff`/`RowCountMatch`/`SchemaMatch` |
| Arbitrary business rule | custom/SQL | Soda `failed_rows_check`; Deequ `CustomSql`; OpenMetadata `tableCustomSQLQuery`; DataHub `AssertionType.SQL`/`CUSTOM`; GE `expect_query_results_to_match_comparison` |
| PR-time / pre-merge regression | value-level diff + blast-radius | Recce (Profile/Value/Top-K/Histogram/Query diff + checklist + PR gate); data-diff (archived engine); OpenMetadata `tableDiff`; Grai (re-run tests on lineage change) |
| PII / sensitive column | recognizer + anonymizer pipeline | Presidio (recognizer catalog → anonymizer operator, decoupled detect/act) |

**Where to run checks (cross-repo convention):** schema/volume/freshness cheapest at **source/staging** (catch upstream breakage before propagation); uniqueness/referential/business-rule at **marts** (grain + joins finalized); distributional/anomaly at **all layers over time** (need history for a baseline → requires a metrics-history store, which DataHub/OpenMetadata provide via `incidentsSummary`/`dataQualityReport`).

### great-expectations/great_expectations — the canonical expectation taxonomy
`https://github.com/great-expectations/great_expectations` (branch `develop`)
- **Lift — full built-in taxonomy** (from `expectations/core/`): table-level (row/column count, columns-match-ordered-list/set), schema/type (`expect_column_to_exist`, `expect_column_values_to_be_of_type`), completeness (null/not-null/proportion), uniqueness (unique, compound-unique, multicolumn-unique), set/domain (`to_be_in_set`, `distinct_values_to_equal_set`), range/stat (between, min/max/mean/median/stdev/sum, quantile, z-score), format (regex, like-pattern, strftime, json-parseable/schema), ordering (increasing/decreasing), length, distributional (KL-divergence, KS, chi-square), multi-column (pair-equal, pair-a>b, cramers-phi, multicolumn-sum, arbitrary `expect_query_results_to_match_comparison`).
- **Invariant:** stage-agnostic — the taxonomy is the reusable vocabulary wherever a "Batch" is defined.
- **LICENSE:** **Apache-2.0**.

### sodadata/soda-core — SodaCL "data contracts engine"
`https://github.com/sodadata/soda-core`
- **Lift — check types** (`contracts/impl/check_types/`, mirrored in YAML): `schema`, `row_count`, `missing`, `invalid`, `duplicate`, `freshness`, `metric`, `aggregate`, `failed_rows`. Dataset- and column-scoped YAML with thresholds (`{metric: percent, must_be_less_than: 10}`).
- **Gotcha:** v4 package names changed (`soda-{ds}` vs legacy v3 `soda-core-{ds}`, on `v3` branch).
- **LICENSE (CRITICAL):** **Elastic License 2.0** — NOT OSI, source-available (no hosted-service, no license-key circumvention → implies key-gated parts even in this repo). API reports `NOASSERTION`. `https://raw.githubusercontent.com/sodadata/soda-core/main/LICENSE`.

### elementary-data/elementary (+ dbt-data-reliability) — dbt-native observability
`https://github.com/elementary-data/elementary` · package: `https://github.com/elementary-data/dbt-data-reliability`
- **Gotcha:** the CLI/report repo and the **actual test-taxonomy dbt package are SEPARATE repos** — deep-reading only `elementary` misses the entire monitor taxonomy.
- **Lift — monitors** (`monitors.sql`): table (`row_count`, `freshness`); any-type (`null_count`/`null_percent`); string (min/max/avg length, missing_percent); numeric (min/max/zero/avg/stdev/variance/sum); boolean (count_true/false); plus `schema_changes`. Each metric runs as static-threshold **or** anomaly (auto-baseline) test. Metrics materialize into warehouse tables → CLI reads them for lineage-enriched reports + Slack/Teams alerts.
- **LICENSE:** **Apache-2.0** (both repos).

### awslabs/deequ — Spark-native "unit tests for data"
`https://github.com/awslabs/deequ`
- **Lift — DQDL rule set** (also AWS Glue's declarative language): `RowCount`, `ColumnCount`, `DuplicateRowCount`, `Completeness`/`IsComplete`, `Uniqueness`/`IsUnique`/`IsPrimaryKey`, `ColumnCorrelation`, `DistinctValuesCount`, `Entropy`, `Mean`/`StandardDeviation`/`Variance`/`Skewness`/`Kurtosis`, `Range`, `Sum`, `CustomSql`, `ColumnLength`, `ColumnExists`, `RowCountMatch` (regression vs reference), `SchemaMatch` (schema diff vs reference), `DataFreshness`. Extras: MetricsRepository (metrics over time), profiling, anomaly detection, automatic constraint suggestion, incremental/algebraic metric computation.
- **LICENSE:** **Apache-2.0**. (Python binding: `awslabs/python-deequ`.)

### unionai-oss/pandera — dataframe schema validation
`https://github.com/unionai-oss/pandera`
- **Lift:** `DataFrameSchema`/`DataFrameModel` — type + constraint validated together (statistically-typed dataframes); `pa.Column(dtype, pa.Check.ge/isin/…)`. In-process/dataframe-native contrast to GE's suite-against-any-batch. **Gotcha:** v0.24+ moves pandas schemas to `pandera.pandas` submodule.
- **LICENSE:** **MIT**.

### whylabs/whylogs — data logging / profiling
`https://github.com/whylabs/whylogs`
- **Lift — profile-first, test-second:** `why.log(df)` → compact **mergeable** profile (combine distributed/streaming profiles over any time window); constraints defined against the profile do "data unit testing" + drift/skew monitoring. (Root README is a stub → real content in `python/README.md`.)
- **LICENSE:** **Apache-2.0**.

### datafold/data-diff — value-level diffing (ARCHIVED)
`https://github.com/datafold/data-diff`
- **Gotcha:** **archived/sunset** (May 17 2024 — "no longer actively supporting"). Frozen reference for the hash-based row-comparison / cross-DB regression pattern, not an active dependency.
- **LICENSE:** **MIT**.

### DataRecce/recce — PR-time data review (the canonical pre-merge pattern)
`https://github.com/DataRecce/recce`
- **Lift:** run against a dbt project with two environments (base vs PR) → lineage + column-level impact map, Profile/Value/Top-K/Histogram/Query diffs (no hand-written SQL), and a re-runnable **Checklist** review artifact. **Decision rule:** sits *after* CI unit/schema tests (GE/Soda/dbt) and *before* merge — verifies the diff had the intended blast radius and no unintended one. Recce Cloud adds PR-merge gating.
- **LICENSE:** **Apache-2.0**.

### OpenLineage/OpenLineage — THE lineage standard (spec)
`https://github.com/OpenLineage/OpenLineage` · spec `spec/OpenLineage.md` + JSON Schema
- **Core model:** **RunEvent** (state transitions; ≥1 START + 1 terminal COMPLETE/FAIL/ABORT; events **additive**) over **Job** (process, unique `name` in `namespace`), **Dataset** (unique `name` in datasource namespace, `db.host.database.schema.table`), **Run** (execution, **UUID** RFC-4122 mandatory), **Facet** (atomic named metadata; re-emitting same-name facet **REPLACES**, never merges — common integration bug).
- **Custom facet naming rule:** type `{prefix}{Name}{Entity}Facet` (PascalCase), key `{prefix}_{name}` (camelCase); `_schemaURL` must be git-sha-pinned/immutable, not branch HEAD (a documented gotcha).
- **DQ bridge (the wire format GE/Soda/dbt results map into):** `DataQualityMetricsInputDatasetFacet` (row/byte/null/distinct/avg/min/max/quantiles) and `DataQualityAssertionsDatasetFacet` — array of `{assertion, success, column, severity, name, expected, actual, …}` where `assertion` free-text-classifies (`not_null`/`unique`/`row_count`/`freshness`/`custom_sql`) and `severity` = `error` (blocking) vs `warn`.
- **LICENSE:** **Apache-2.0** (spec files carry SPDX header).

### MarquezProject/marquez — reference OpenLineage service
`https://github.com/MarquezProject/marquez`
- **Lift:** the DB+API+UI that consumes OpenLineage (Java/Dropwizard + Postgres; REST `/lineage`, admin `/healthcheck`/`/metrics`, beta GraphQL; React UI). LF AI & Data Graduated. Maintains 3 compatibility tiers (CURRENT/RECOMMENDED/MAINTENANCE) pinned to spec versions.
- **Gotcha:** **no default auth/authz** on the HTTP API (self-hosting security note).
- **LICENSE:** **Apache-2.0**.

---

## 8. Governance / Catalog / PII / Access

### datahub-project/datahub — entity/aspect metadata platform
`https://github.com/datahub-project/datahub`
- **Lift — aspect model** (`entity-registry.yml`): each entity carries independently-versioned/writable **aspects** (extensibility = add an aspect type without touching the entity). `dataset` aspects include `schemaMetadata, datasetProfile, datasetUsageStatistics, upstreamLineage, ownership, globalTags, glossaryTerms, deprecation, testResults, incidentsSummary, structuredProperties, icebergCatalogInfo, …`.
- **Native DQ — `AssertionType` enum:** `DATASET`, `FRESHNESS`, `VOLUME`, `SQL`, `FIELD`, `DATA_SCHEMA`, `CUSTOM` (explicit landing zone for GE/Soda/dbt/Deequ results). Separate first-class `testResults` + `incidentsSummary` aspects. **Convergence signal:** near-identical shape to OpenLineage's assertion classification and Soda's check types.
- **LICENSE:** **Apache-2.0** (root unambiguous; some Cloud features gated separately — verify per-module if embedding).

### open-metadata/OpenMetadata — catalog that runs its own tests
`https://github.com/open-metadata/OpenMetadata`
- **Lift:** JSON-Schema entity model; data assets `table/database/dashboard/pipeline/topic/mlmodel/dataContract/…`. **Ships a full GE-equivalent test taxonomy natively in the catalog** (`json/data/tests/`): `tableRowCountToBeBetween`, `tableRowInsertedCountToBeBetween`, `tableColumnCountToBeBetween`, `tableColumnToMatchSet`, `tableCustomSQLQuery`, **`tableDiff`** (regression, Recce-style, built in), `columnValuesToBe{Between,InSet,NotNull,Unique}`, `columnValuesToMatchRegex`, `columnValue{Max,Min,Mean,Median,StdDev}ToBeBetween`, etc. Organized `TestSuite → TestCase → TestDefinition` with `testCaseResolutionStatus` incident triage.
- **Decision-table takeaway:** the only catalog here that *runs its own tests* (vs DataHub's thinner assertion-result envelope). Choose by "catalog that runs tests" vs "catalog that ingests external test results."
- **LICENSE:** **Apache-2.0**.

### microsoft/presidio — PII detection + anonymization
`https://github.com/microsoft/presidio`
- **Lift — decoupled detect/act:** Analyzer (recognizers) → Anonymizer (operators), detection (entity type + span) separated from action.
  - **Recognizers:** generic (credit_card, crypto, date, email, iban, ip, mac, phone, url); country-specific ~20 jurisdictions (US ssn/itin/passport/npi/mbi; UK nino/nhs; DE tax_id/vat_id; …) via checksum/regex/context; NER (`gliner`, `huggingface_ner`, `medical_ner`); NLP engines (spacy/stanza/transformers); **LLM-assisted** (`langextract` + Azure OpenAI) — newest direction, pluggable alongside regex/NER.
  - **Anonymizer operators:** `mask`, `redact`, `replace`, `hash`, `encrypt`/`decrypt` (AES), `keep`, `custom`.
- **GOTCHA (governance):** docs say *"Presidio is transitioning to a community-owned project"* under "Data Privacy Stack" (`github.com/data-privacy-stack/presidio`). Treat `microsoft/presidio` as canonical as-of 2026-07-25 but expect the org to move; re-verify.
- **LICENSE:** **MIT**.

### grai-io/grai-core — version control for metadata / lineage-aware CI gate
`https://github.com/grai-io/grai-core` — **NOTE: `grai-io/grai` 404s; correct repo is `grai-core`.**
- **Lift:** column-level lineage across warehouses/APIs/dashboards + **centralized tests that re-run on upstream change** + GitHub-Actions CI gating (`grai-actions`). "Tests as version-controlled metadata that re-trigger when the lineage graph shows an upstream dependency changed" — a lineage-aware CI gate (like Recce) rather than a standalone check DSL. Per-source connectors: snowflake/bigquery/redshift/postgres/dbt/fivetran/metabase/looker.
- **LICENSE:** **MIT-0 (MIT No Attribution)** — confirmed by full-text read (omits the "copyright notice shall be included" clause; that's what makes it MIT-0, not MIT). `https://raw.githubusercontent.com/grai-io/grai-core/master/LICENSE`.

### apache/atlas — Hadoop-ecosystem governance + type system
`https://github.com/apache/atlas`
- **Lift — type hierarchy** (`0010-base_model.json`): `Referenceable` (has `qualifiedName`) → `Asset` (name/owner/description) → **`DataSet`** (marker for lineage endpoints) / **`Process`** (`inputs: array<DataSet>, outputs: array<DataSet>` — structurally identical to OpenLineage's Job inputs/outputs) / `Infrastructure`. Domain models (Hive/Kafka/Spark/Flink; AWS/Azure/GCP) shipped as versioned patch files.
- **Invariant worth lifting:** Atlas treats **its own metamodel as schema-versioned/migrated** (numbered `patches/` per model area) — the pattern for any catalog storing evolving structured metadata. Integrates with Ranger for RBAC/ABAC access control.
- **LICENSE:** **Apache-2.0**.

---

## 9. Experiment Data Foundations (Measurement Validity)

### growthbook/growthbook — warehouse-native experimentation + flags
`https://github.com/growthbook/growthbook`
- **Lift:** flags + experiments + product analytics; **warehouse-native** (queries BigQuery/Snowflake/Databricks directly rather than ingesting events); SQL-backed metric definitions; 24 SDKs. **Stats engine (README-named, confirms table-stakes):** CUPED (variance reduction), Sequential testing, Bayesian, Post-Stratification, Bandits, and **SRM (Sample Ratio Mismatch) checks** as a first-class guardrail.
- **LICENSE (open-core, verified):** bulk = **MIT**; carve-out dirs `packages/{back-end/src,front-end,shared/src}/enterprise` = **GrowthBook Enterprise License** (commercial). `https://raw.githubusercontent.com/growthbook/growthbook/main/LICENSE`.

### spotify/confidence — experiment analysis library (measurement validity)
`https://github.com/spotify/confidence`
- **Lift:** Python (statsmodels) for A/B analysis — Z-test / Welch's T-test / Chi-squared, CUPED-style variance reduction (linear model on pre-exposure data), Bayesian BetaBinomial, Bonferroni multiple-testing correction, non-inferiority margins, group-sequential testing, sample-size/power calculators. Measurement-validity tooling, not a full platform.
- **LICENSE:** **Apache-2.0**.

### facebookarchive/planout — deterministic assignment (foundational, archived)
`https://github.com/facebookarchive/planout`
- **Lift — deterministic hashing + namespaces:** `UniformChoice`/`WeightedChoice` hash `(experiment + variable + unit_id)` → same arm every time **without persisting assignment state**. Namespaces enforce mutually-exclusive experiment allocation (no double-assignment). Assignment objects self-log inputs/checksum/salt/params — audit trail baked into the primitive. Archived, but the model is canonical.
- **LICENSE:** **BSD** (Copyright 2014 Facebook, Inc.).

**Measurement-validity invariants to teach** (synthesized): SRM check (assignment ratio ≠ intended → the experiment is invalid, don't read the result); guardrail metrics; CUPED for variance reduction; deterministic salted-hash assignment; mutually-exclusive namespaces. `PostHog/posthog` also ships experiments over the same event store (see §1/§10).

---

## 10. AI-Era Data Plane (LLM Trace/Token Warehousing · RAG-as-ETL)

**This is the seam with the shipped `ai` skill.** `ai` owns retrieval behavior; `data` owns (a) LLM traces/tokens **as a warehoused dataset** and (b) the RAG pipeline **as governed ETL** (chunking/embedding jobs, corpus freshness/lineage/PII).

### langfuse/langfuse — LLM traces as a queryable dataset
`https://github.com/langfuse/langfuse` (built on ClickHouse; part of ClickHouse org since Jan 2026)
- **Lift — data model (v4):** `Session (1)→(n) Trace`; a **Trace** groups **Observations** sharing a `trace_id`; Observations are the atomic unit. Storage: conceptually one **observations table**, each row carrying observation data **plus a denormalized copy of trace-level attributes** (`user_id`, `session_id`, `tags`, `metadata`) for fast aggregation — a warehousing pattern to lift.
- **Observation types:** `event`, `span`, `generation` (LLM call — prompts/tokens/cost), `agent`, `tool`, `chain`, `retriever` (vector/DB lookup), `evaluator`, `embedding` (token/cost), `guardrail`.
- **Token/cost model:** usage+cost only on `generation`/`embedding`; usage-types are provider strings bucketed by substring (`input`/`output`/`cached_tokens`/`audio_tokens`); **ingested usage/cost always wins over inferred** (tokenizer + model price table at ingestion). Aggregated via a Metrics API for billing/rate-limiting.
- **Invariant/gotcha:** OTel-native (fan out to Langfuse + Datadog simultaneously); SDKs batch/queue → require explicit `flush()` in serverless/short-lived processes or buffered traces are lost on exit.
- **LICENSE (open-core):** core = **MIT**; `ee/`, `web/src/ee/`, `worker/src/ee/` = **Langfuse Enterprise License** (commercial). `https://raw.githubusercontent.com/langfuse/langfuse/main/LICENSE` + `ee/LICENSE`.

### traceloop/openllmetry — OTel GenAI semantic conventions (the standard)
`https://github.com/traceloop/openllmetry`
- **Lift:** its semantic conventions were **folded into official OpenTelemetry**. `gen_ai.*` attributes (from `semconv_ai`): `GEN_AI_USAGE_TOTAL_TOKENS`, `GEN_AI_USAGE_TOKEN_TYPE`, `GEN_AI_USAGE_REASONING_TOKENS`, `GEN_AI_IS_STREAMING`, `GEN_AI_RESPONSE_FINISH_REASON`, `GEN_AI_REQUEST_STRUCTURED_OUTPUT_SCHEMA`, plus **vector-DB attributes** `db.vector.query.{top_k,embeddings_count,result_count}` — RAG retrieval steps get first-class OTel spans alongside generation spans. Metrics: `gen_ai.client.token.usage`, `gen_ai.client.operation.duration`. `GenAISystem` enum = canonical vendor strings (openai/anthropic/cohere/mistral_ai/bedrock/vertex/…).
- **LICENSE:** **Apache-2.0**.

### Arize-ai/phoenix — AI observability (OTel + evals)
`https://github.com/Arize-ai/phoenix`
- **Lift:** OTel LLM tracing, evals, datasets/experiments; auto-instrumentation via **OpenInference** (`Arize-ai/openinference`) which defines the semantic conventions Phoenix consumes (second OTel-GenAI-conventions source alongside OpenLLMetry).
- **LICENSE:** **Elastic License 2.0 (ELv2)** — non-OSI, source-available. `https://raw.githubusercontent.com/Arize-ai/phoenix/main/LICENSE`.

### feast-dev/feast — feature store (point-in-time correctness)
`https://github.com/feast-dev/feast`
- **Lift — the invariant that transfers to RAG-as-ETL:** `get_historical_features()` does **point-in-time-correct joins** to prevent label/feature leakage. Same discipline a governed RAG corpus needs — never embed/serve a document version "from the future" relative to a query's as-of time. Offline store (training) / online store (serving) split.
- **LICENSE:** **Apache-2.0**.

**RAG-as-ETL synthesis (for the skill):** model embedding/chunking as an incremental ETL job (dlt schema-contract discipline, §3), give the vector store a **freshness/lineage** contract (OpenLineage facets, §7) and **PII classification** at ingest (Presidio, §8), warehouse LLM traces/tokens as a dataset (Langfuse model + OTel `gen_ai.*` conventions above), and enforce point-in-time correctness (Feast). No single repo packages this end-to-end — it's the assembled seam this skill should teach.

---

## 11. Orchestration (cross-cutting)

### ORCHESTRATOR DECISION TABLE (task vs asset vs flow — precise wording, lift wholesale)

| Tool | Core unit | Model | Best fit |
|---|---|---|---|
| **Airflow** (`apache/airflow`) | Task in a DAG | Imperative, mostly-static DAG structure | Stable, task-sequenced batch pipelines; huge operator ecosystem |
| **Dagster** (`dagster-io/dagster`) | **Asset** (software-defined) | Declarative — declare the *data* you want, Dagster infers the graph & keeps it current | Data/asset-lineage-centric teams; testability dev→prod |
| **Prefect** (`PrefectHQ/prefect`) | Flow/Task (Python fns) | Imperative Python with retries/branching as decorators | "Just Python," dynamic/ad-hoc workflows, fast script→prod |
| **Kestra** (`kestra-io/kestra`) | Flow (YAML) | Declarative, language-agnostic, event+schedule unified | Polyglot infra/data orchestration; event-driven triggers; low-code UI+Git |

- **Airflow:** idempotency is an *opinion not enforced* ("tasks should ideally be idempotent… not pass large data between tasks" — small XCom metadata only; delegate heavy movement to external systems). Retrofitted **Assets** (data-aware scheduling view) without becoming asset-native. First-class backfill UI. **Apache-2.0**.
- **Dagster:** you declare `@dg.asset` fns; params ARE upstream assets → graph inferred; lineage/observability baked into the asset graph. README framing (verbatim for a decision table): "designed for developing and maintaining data assets" vs Airflow's "author workflows that orchestrate tasks." **Apache-2.0**.
- **Prefect:** `@flow`/`@task` decorators; retries built into `@task`; middle ground between Airflow ceremony and Dagster asset graph — still task/flow-based imperative control flow. **Apache-2.0**.
- **Kestra:** declarative YAML, language-agnostic (any language via plugins); unifies cron + event-driven behind one **trigger** abstraction (file arrival, Kafka/Redis/Pulsar/AMQP/MQTT/SQS/PubSub); UI round-trips to YAML-as-code. **Apache-2.0**.

---

## 12. Lakehouse / Storage

### apache/iceberg — open table format (spec + Java reference)
`https://github.com/apache/iceberg` (Go/Python/Rust/C++ impls in sibling repos)
- **Lift — metadata model:** table metadata → **snapshots** → **manifest list** (+ partition stats) → **manifests** (per-file metrics). Changes = atomic-swap of the metadata file → **serializable isolation**, no reader locks, native **time travel** (old snapshot = old metadata pointer). Sequence numbers enable optimistic concurrency + safe retries.
- **Schema evolution:** add/drop/rename/reorder via **stable per-field integer IDs** (renames free; old data files stay valid). **Hidden partitioning:** partition values derived via **transforms** (`bucket[N]`, `truncate`, `year`/`month`/`day`/`hour`); queries filter on logical columns, Iceberg derives partition predicates. **Partition-spec evolution** without rewriting old data.
- **LICENSE:** **Apache-2.0**.

### delta-io/delta — transaction-log lakehouse format
`https://github.com/delta-io/delta`
- **Lift — transaction log:** every commit = numbered JSON in `_delta_log/`; periodic `.checkpoint.parquet` collapse the log; `_last_checkpoint` points to latest. Log-replay gives **ACID + time travel**. **Gotcha:** ACID is only as strong as the underlying storage's atomic-write/consistent-list guarantees (weak object stores break it). Backward-compatible reads, not forward-compatible (protocol reader/writer version gating).
- **LICENSE:** **Apache-2.0**.

### duckdb/duckdb — in-process OLAP engine
`https://github.com/duckdb/duckdb`
- **Lift:** embedded (no server); `SELECT * FROM 'file.parquet'` directly, query DataFrames in place (zero-copy interop); "friendly SQL." The de-facto local/embedded engine — e.g. dlt's default `destination="duckdb"` runs a whole pipeline in-process before targeting a warehouse; reads Iceberg/Delta via extensions.
- **LICENSE:** **MIT** (Stichting DuckDB Foundation). `https://github.com/duckdb/duckdb/blob/main/LICENSE`.

**Iceberg vs Delta rule of thumb:** both Apache-2.0, ACID via atomic metadata, time travel, schema evolution. Iceberg → fully "hidden" declarative partitioning + free partition-spec evolution; broadest non-JVM native impls (Go/Rust/C++/Python); mature REST catalog. Delta → single unifying transaction-log mental model (numbered JSON + Parquet checkpoints); deepest Spark-native tooling.

Also in corpus (query/OLAP engines, all **Apache-2.0**): `ClickHouse/ClickHouse`, `apache/doris`, `StarRocks/starrocks`, `apache/pinot`, `apache/druid`, `questdb/questdb`, `trinodb/trino`, `prestodb/presto`, `pola-rs/polars` (MIT), `ibis-project/ibis` (portable dataframe→SQL).

---

## 13. Licensing Appendix (family value-add — date-stamped 2026-07-25)

The `data`/analytics stack is a **licensing minefield** the skill should teach explicitly. Three dominant 2026 shapes:

**A. Open-core (MIT/Apache core + `ee/` or `enterprise/` dir under a proprietary Enterprise License):**
- PostHog — MIT core + `ee/` PostHog Enterprise License. API reports "Other."
- GrowthBook — MIT core + `packages/*/enterprise` GrowthBook Enterprise License.
- Langfuse — MIT core + `ee/` Langfuse Enterprise License.
- Confluent Schema Registry — Apache clients + Confluent Community License server half.
- Redpanda Connect — Apache-2.0 majority + RCL enterprise; **no root LICENSE** (only `licenses/`).

**B. Whole-product single non-OSI source-available license:**
- Airbyte platform — **ELv2** (connectors/CDK stay MIT).
- soda-core — **ELv2** (API shows NOASSERTION).
- RudderStack server — **ELv2**.
- Arize Phoenix — **ELv2**.
- dbt Fusion engine — **ELv2** (5 named crates Apache-2.0); dbt Core stays Apache-2.0.
- Estuary Flow — **BSL 1.1**, converts to Apache-2.0 on **2029-11-20**.

**C. Pipeline-itself tightened (most aggressive):**
- Snowplow — **SLULA v1.1 (Nov 2024)** on the core pipeline (`snowplow/enrich`), not just an enterprise carve-out. **Stale-LICENSE trap:** meta-repo + GitHub API still say Apache-2.0. `iglu-central` schema repo IS Apache-2.0.

**Copyleft to flag distinctly:**
- Multiwoven — **AGPLv3** (network-use source-disclosure).
- Metabase — **AGPL-3.0** + commercial. Grafana — **AGPL-3.0**. Plausible — **AGPL-3.0**. Matomo — **GPL-3.0**. Vector — MPL-2.0. Redash — BSD-2.

**Cleanly permissive (safe to depend on / recommend freely):**
- **Apache-2.0:** dbt-core, dbt-utils, SQLMesh, MetricFlow (v0.209+), Rill, Airflow, Dagster, Prefect, Kestra, Iceberg, Delta, Debezium, dlt, Great Expectations, Elementary, Deequ, whylogs, Recce, OpenLineage, Marquez, DataHub, OpenMetadata, Atlas, Feast, OpenLLMetry, Spotify Confidence, Superset, ODCS.
- **MIT:** DuckDB, Polars, Malloy, Evidence, Meltano, Presidio, pandera, datacontract spec + CLI, data-diff, sqlfluff. **MIT-0:** grai-core. **BSD:** PlanOut (2-clause: Redash).

**Anti-hallucination reminders:** (1) never trust GitHub API `license` for this stack — read the file. (2) MetricFlow README's "source-available" header is stale (current is Apache-2.0). (3) `grai-io/grai` and `dbt-labs/corp` (dbt style guide) return 404 — don't cite as live. (4) `microsoft/presidio` and `datafold/data-diff` are mid-transition / archived respectively.

---

## 14. Top 15 Lift-Worthy Sources (ranked)

1. **dbt-labs/dbt-project-evaluator** — the single densest decision-table source (naming, structure, materialization, testing, fan-out/join thresholds — each an enforceable lint rule). `https://github.com/dbt-labs/dbt-project-evaluator`
2. **OpenLineage/OpenLineage** — the lineage standard; Run/Job/Dataset+Facet model and the exact DQ-assertion wire format every tool maps into. `https://github.com/OpenLineage/OpenLineage`
3. **great-expectations/great_expectations** — the canonical, exhaustive DQ check taxonomy (the vocabulary every other tool echoes). `https://github.com/great-expectations/great_expectations`
4. **bitol-io/open-data-contract-standard (ODCS)** — the go-forward data-contract standard: schema + DQ tiers + SLA `driver` axis. `https://github.com/bitol-io/open-data-contract-standard`
5. **TobikoData/sqlmesh** — model-kind decision table + virtual-environment + incremental-by-default philosophy (sharpest contrast to dbt). `https://github.com/TobikoData/sqlmesh`
6. **dbt-labs/metricflow** — metric-type decision table + the explicit AGPL→BSL→Apache license history (a licensing teaching artifact in itself). `https://github.com/dbt-labs/metricflow`
7. **snowplow/snowplow (+ iglu-central)** — schema-first tracking + MODEL-REVISION-ADDITION evolution rule + the SLULA stale-LICENSE trap. `https://github.com/snowplow/snowplow`
8. **confluentinc/schema-registry** — the schema-compatibility-mode decision table (BACKWARD/FORWARD/FULL + transitive), the definitive answer to "how do events evolve safely." `https://github.com/confluentinc/schema-registry`
9. **airbytehq/airbyte** — connector protocol + replication-mode + CDC-snapshot semantics + the ELv2/MIT platform-vs-connector split. `https://github.com/airbytehq/airbyte`
10. **apache/iceberg** — lakehouse table-format semantics (hidden partitioning, ID-based schema evolution, snapshot time-travel) done most cleanly. `https://github.com/apache/iceberg`
11. **dagster-io/dagster** (with the §11 table) — the asset-vs-task orchestration paradigm, the precise wording for the decision. `https://github.com/dagster-io/dagster`
12. **langfuse/langfuse** — the LLM-trace-as-dataset warehousing model (Trace/Observation, denormalized token/cost) — core to the AI-era seam. `https://github.com/langfuse/langfuse`
13. **DataRecce/recce** — the PR-time data-review pattern (verify blast radius before merge) that nothing else in the family teaches. `https://github.com/DataRecce/recce`
14. **microsoft/presidio** — decoupled PII recognizer→anonymizer pipeline (the governance/PII-at-ingest primitive). `https://github.com/microsoft/presidio`
15. **dlt-hub/dlt** — Python schema-inference + three-mode schema contracts (evolve/freeze/discard) — the cleanest ingestion-contract pattern, and the RAG-as-ETL building block. `https://github.com/dlt-hub/dlt`

**Honorable mentions:** traceloop/openllmetry (OTel `gen_ai.*` conventions, incl. vector-DB span attrs), datahub-project/datahub + open-metadata/OpenMetadata (catalog aspect models + native tests), growthbook/growthbook (SRM/CUPED experimentation), debezium/debezium (CDC + Outbox), duckdb/duckdb (embedded engine glue), evidence-dev/evidence + rilldata/rill (BI-as-code, Rill is agent-first).
