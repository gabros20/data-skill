# Data Skill — Web Canon Corpus

**Scope:** Canonical, current data-engineering / analytics knowledge swept from the open web for the `data` skill (analytical & governed-data-plane member of the Digital Product Skills family). Research only.

**As-of date:** 2026-07-25. Every version/price/license claim is date-stamped to its fetch. Claims that could not be verified from a primary source are explicitly flagged **[UNVERIFIED]** — do not teach those as fact without a fresh fetch.

**Method:** 4 parallel Sonnet cluster-workers, each fetching primary/canonical URLs (WebSearch budget was exhausted session-wide mid-run, so most findings come from direct WebFetch of known-canonical URLs; coverage gaps where a URL guess 404'd are flagged). Cluster-3 gaps were patched by the research lead with targeted WebFetch calls — see the **[LEAD-PATCHED, verified 2026-07-25]** annotations.

---

# Cluster 1 — Collection & Contracts

## Event taxonomy & tracking-plan design

**Segment spec (6 call types)** — Identify (who), Track (what action), Page (what web page), Screen (what app screen), Group (what account/org), Alias (past identity/merge). All calls share common fields: `event`, `properties`, `traits`, `context`, `timestamp`, `messageId`, `anonymousId`, `userId`. https://www.twilio.com/docs/segment/connections/spec (2026-07-25)

**Track call reserved properties**: `revenue` (Number, decimal $), `currency` (ISO 4217, defaults USD), `value` (Number, abstract non-monetary value e.g. signups). Invariant: "only use reserved properties for their intended meaning" — Segment auto-maps these to destination-specific formats. `userId` auto-populates in browser/mobile libs from a prior Identify; `anonymousId` is the pseudo-unique fallback. Event names should be human-readable, not generic ("Video Recorded", "Order Completed" — not "Event 12"). https://www.twilio.com/docs/segment/connections/spec/track (2026-07-25)

**Object-Action naming framework**: pair an object (Button, Form, Order) with a past-tense action (Clicked, Submitted, Completed) → "Form Submitted". Rules: object-first then action; always past tense; pick one casing (snake_case / Title Case / camelCase) and never mix; no spaces as delimiters where casing requires. Segment, Amplitude, Mixpanel, PostHog, mParticle all converge on this convention. Advanced variant for scale: `Category:Object_Action` (e.g. `app:account_created`) to namespace by origin/domain. https://growthmethod.com/object-action-framework/ (secondary/aggregator source, cross-checked against Segment/Avo primary docs, 2026-07-25)

**Avo naming conventions** — four independently-configurable dimensions: (1) casing — snake_case/camelCase/Title Case/lower case; (2) format/word order — `[object][action]`, `[action][object]`, or `[context][object][action]`; (3) tense — past vs present, held consistent; (4) allowed vocabulary — canonical terms locked (e.g. always "game," never "match"). Avo enforces a **global namespace**: blocks duplicate events differing only in casing, auto-suggests corrected casing, warns on near-duplicate event names before creation. Anti-pattern called out explicitly: never let an event property and a user property share the same name — they represent different semantic states and produce "dangerously misleading" analytics if conflated. https://www.avo.app/docs/data-design/best-practices/naming-conventions (2026-07-25)

**Snowplow tracking design** — "entity-first" design principle: "If a piece of information is likely to be relevant to multiple different events, it belongs in an entity, not as a property" (entities = reusable structures attached to events, not duplicated per-event). Naming: tracking plans use descriptive Title Case per business domain ("Ecommerce Checkout Flow"); event specs use verb-noun Title Case ("Add To Cart"); underlying data structures/properties use snake_case (`add_to_cart`, `user_id`). Schema granularity is a real design choice: group related actions into one schema with a `type` discriminator (needs strong governance) vs. separate schema per action (cleaner independent evolution). https://docs.snowplow.io/docs/fundamentals/tracking-design-best-practice/ (2026-07-25)

**Snowplow self-describing events (structure)**: event data is wrapped in a self-describing JSON envelope pointing to an Iglu schema URI, e.g. `iglu:com.my_company/viewed_product/jsonschema/1-0-0`, nested inside the standard `unstruct_event` schema. The `self` block carries vendor, name, format (always `jsonschema`), version. https://docs.snowplow.io/docs/fundamentals/schemas/ (2026-07-25)

**Snowplow SchemaVer** (MODEL-REVISION-ADDITION; fetched page described it in major/minor/patch terms — same 3-part structure):
- MODEL/major bump: breaking change — changing a field's type, adding a new **required** field, required↔optional flips that break historical validation on some warehouses. Not backward-compatible.
- REVISION/minor bump: may invalidate *some* historical data depending on warehouse (Redshift vs Snowflake/BigQuery/Databricks differ on required→optional conversions).
- ADDITION/patch bump: fully backward-compatible — adding a new **optional** field, widening a size limit without a type change.
https://docs.snowplow.io/docs/fundamentals/schemas/versioning/ (2026-07-25) — warehouse-dependent compatibility nuance is a real gotcha worth teaching. **[UNVERIFIED]** the literal term "MODEL-REVISION-ADDITION" vs the major/minor/patch phrasing the fetched page used — cross-check before quoting as verbatim Snowplow vocabulary.

## Data contracts

**Chad Sanderson's canonical definition** (originally chadsanderson.com, now redirects to onebeforex.com — company rebrand; essay dated **2022-08-22**): "API-like agreements between Software Engineers who own services and Data Consumers that understand how the business works in order to generate well-modeled, high-quality, trusted, real-time data." Names the failure mode data contracts fix as the **"GIGO Cycle"** — production DBs treated as non-consensual APIs, producers unaware of downstream analytics usage → untrustworthy data → cascading pipeline failures.
- Producer responsibilities: emit data in consumer-required shapes, maintain backward-compatible versioning, notify consumers of schema changes, provide semantic docs.
- Consumer responsibilities: define explicit data requirements iteratively upfront, express business semantics (entities/events/attributes) in the contract, document use cases and SLAs, state data-quality expectations.
- Enforcement: strong CI/CD + change management, schema versioning via Protobuf, event-driven (Kafka) architecture, SLAs backed by on-call.
https://dataproducts.substack.com/p/the-rise-of-data-contracts (his Substack "Data Products" is the live canonical home; chadsanderson.com redirects here) (2026-07-25)

**PayPal's open-source Data Contract Template** → evolved into the **Open Data Contract Standard (ODCS)**, now governed by **Bitol** (AIDA User Group nonprofit + Linux Foundation AI & Data, partnership formed Nov 2023). Current version **v3.0.0**, license **Apache 2.0**. Core sections: Fundamentals, Schema, Data Quality, Support/communication channels, Pricing, Team/roles/responsibilities, SLA, Infrastructure/server specs, Custom properties. JSON Schema provided for IDE validation. https://github.com/paypal/data-contract-template and https://bitol-io.github.io/open-data-contract-standard/v3.0.0/home/ (2026-07-25)

**Schema registry as enforcement (Confluent)** — Confluent Schema Registry supports **Avro, Protobuf, and JSON Schema (JSON_SR)** natively. Seven named compatibility modes (default **BACKWARD**, non-transitive):

| Mode | Rule | Who upgrades first |
|---|---|---|
| BACKWARD | new schema reads old data; checked against latest version only | consumers first |
| BACKWARD_TRANSITIVE | new schema reads data from *all* prior versions | consumers first |
| FORWARD | old schema reads new data; checked vs immediately-preceding version only | producers first |
| FORWARD_TRANSITIVE | old schema reads data from all future/new versions, transitively | producers first |
| FULL | both backward and forward, vs latest version | either order |
| FULL_TRANSITIVE | both, transitively across all versions | either order, max flexibility |
| NONE | no compatibility checking | simultaneous upgrade or new topic required |

https://docs.confluent.io/platform/current/schema-registry/fundamentals/schema-evolution.html (2026-07-25)

## Instrumentation QA

**Segment Protocols** (Business-plan addon only — not on Free/Team tiers, event-stream + Engage sources only): tracking plan = "central source of truth"; validates incoming events/properties against the plan and raises a "violation" for any mismatch; can **block non-conforming events** and route them to a quarantined source; supports renaming events/properties post-hoc without code changes. https://www.twilio.com/docs/segment/protocols (2026-07-25)

**Avo Inspector / Audit**: flags events that break the org's casing convention; flags "rogue" events not present in the tracking plan; surfaces near-duplicate event names from spelling/casing drift. https://www.avo.app/docs/data-design/best-practices/naming-conventions (2026-07-25). **[UNVERIFIED]** dedicated Inspector doc page (avo.app/docs/inspector/*) 404'd — treat Inspector specifics beyond this as unverified.

**Typewriter**: Segment's codegen tool generating strongly-typed tracking functions from a tracking plan (JSON-Schema-based), catching type/shape drift at compile time. **[UNVERIFIED]** current maintenance status not re-confirmed — flag before teaching as current tooling.

## Client-side vs. server-side tracking

**Server-side GTM** (Google's own docs): full producer control over data before routing ("only you have access to the data in the server until you choose to send it elsewhere"); cross-device/cross-platform measurement through one funnel; full control over data shaping/routing. Operational gotcha: must be installed on a first-party domain and switched to production mode before go-live. https://developers.google.com/tag-platform/tag-manager/server-side/intro (2026-07-25). **[UNVERIFIED]** ad-blocker-resilience/latency claims were NOT stated on the fetched page — "server-side evades ad blockers because it's first-party" is industry-common reasoning, not a Google-confirmed claim here.

## Consent & privacy-aware collection

**Google Consent Mode**: four core signals — `ad_storage`, `analytics_storage`, `ad_user_data`, `ad_personalization`, each `granted`/`denied`. Pattern: `gtag('consent','default',{...})` before any measurement call fires, then `gtag('consent','update',{...})` after banner interaction. When `ad_storage` is denied: no new ad cookies, but "data sent to Google will still include the full page URL" (denial ≠ no data transmitted); `ads_data_redaction` can strip ad-click IDs; analytics can still use **URL passthrough** (cookieless). https://developers.google.com/tag-platform/security/guides/consent (2026-07-25)

**GDPR at collection point**: consent required before any non-strictly-necessary cookie fires; strictly-necessary cookies exempt from prior consent but still require purpose disclosure; withdrawal must be as easy as granting; IP addresses/online identifiers are personal data once capable of contributing to identification — the legal basis for IP-anonymization-at-capture as a default. https://gdpr.eu/cookies/ (2026-07-25)

**CCPA at collection point**: "notice at collection" must be shown at or before collection, listing categories, purposes, privacy-policy link, and (if applicable) a "Do Not Sell or Share" link. Minimization is purpose-limitation-based (must disclose purposes), not a hard technical mandate like GDPR's. "Sharing" explicitly includes cross-context behavioral advertising, not just monetary sale. https://oag.ca.gov/privacy/ccpa (2026-07-25)

---

# Cluster 2 — Transformation & Semantics

## dbt current state (docs fetched 2026-07-25)

**dbt Core vs dbt Platform** — https://docs.getdbt.com/docs/introduction
- Two runtime generations: **dbt Core v1** (legacy, Python-based, still maintained) and **dbt Core v2**, powered by the new **Fusion Engine** (Rust-based), "the current default when installing dbt."
- **dbt Platform** (formerly "dbt Cloud"): hosted scheduling, CI/CD, docs, monitoring, Studio IDE, Developer(free)→Enterprise+.

**dbt Fusion Engine licensing** — https://www.getdbt.com/licenses-faq + https://docs.getdbt.com/docs/fusion/fusion-availability
- **dbt Core**: Apache 2.0, open source, unchanged. Verbatim: *"dbt Labs will continue maintaining dbt Core under the Apache 2 license indefinitely."*
- **dbt Fusion**: **proprietary** — "dbt Product Licensing Agreement," NOT Apache 2.0. Restrictions (quoted): may not "redistribute dbt Fusion to others as a managed service" without giving end users "the full capabilities of dbt Fusion, including features gated behind login"; may not "circumvent dbt Fusion's license key functionality or attempt to derive Fusion's source code"; may not remove/obscure license/trademark notices.
- Fusion is **free to download/use locally**; core features (v1 workflows, syntax checking, lint, docs-lite, LSP-lite) free-forever, no login; **premium features require a free dbt Platform account login** (full LSP, SQL comprehension/type-checking, column-level lineage, full docs v2).
- GA date cited by docs: **June 1, 2026**. Adapter status at fetch: Snowflake/BigQuery/Databricks/Redshift = Preview; Spark/DuckDB (CLI only) = Beta.
- **[UNVERIFIED]** dbt Labs acquiring SDF Labs / exact Fusion-announcement blog post: could not be directly verified — candidate blog URLs 404'd and WebSearch was exhausted. Fusion is widely reported as stemming from the SDF acquisition, but no primary-source link/date. Do NOT present an acquisition date as fact.

## Layering conventions — dbt Labs "How We Structure" (verbatim quotes)

https://docs.getdbt.com/best-practices/how-we-structure/1-guide-overview
- **Core principle**: move data "from *source-conformed* to *business-conformed*."
- **Staging** (`stg_`): 1:1 with source tables; only place `source()` is used; **materialize as views**. Allowed: renaming, type casting, basic computations, categorizing. **Forbidden: joins and aggregations** — "joins are almost always a bad idea here."
- **Intermediate** (`int_`): naming `int_[entity]s_[verb]s.sql` (e.g. `int_order_items_summed_to_orders`), verb-oriented. **Default materialization: ephemeral** (or views in a custom schema). Should **never be exposed** to end users. Three uses: structural simplification, re-graining, isolating complex logic.
- **Marts** (business entity names, e.g. `customers.sql`; `fct_`/`dim_` for fact/dimension splits): **"Wide and denormalized"** is the explicit modern-stack recommendation — *"storage is cheap and it's compute that is expensive."* Anti-pattern: **"Too many joins in one mart"** (>4-5 concepts → use an intermediate layer). **Materialize as tables or incremental**.
- **DAG shape invariant** (verbatim): *"Narrow the DAG, widen the tables... several arrows going **into** our post-staging models is great and expected, several arrows coming **out** is a red flag."*

## Materializations & incremental strategies

https://docs.getdbt.com/docs/build/materializations, https://docs.getdbt.com/docs/build/incremental-models
- **view**: no storage cost, always fresh, slow if stacked — the default starting point.
- **table**: fast reads, long rebuilds, no auto-append — BI-facing models and slow shared upstream transforms.
- **incremental**: inserts/updates only new records — "advanced usage pattern," move to it "only after `dbt run`s become too slow."
- **ephemeral**: inlined as CTE, can't be queried/`ref()`'d by operations, no contracts support — lightweight early-DAG logic only.
- **materialized view**: DB-managed refresh; Snowflake uses Dynamic Tables instead; Postgres lacks native support.
- **Incremental strategies**: `merge` (upsert on `unique_key`), `delete+insert`, `append` (default with no `unique_key` — can duplicate), `insert_overwrite` (partition-based, ignores `unique_key`), `microbatch`. **Gotcha**: `unique_key` columns must not be null; duplicate `unique_key` values can fail/duplicate; use `on_schema_change` for added/removed columns.
- Python models support only `table`/`incremental`.

## Modeling: Kimball / OBT / Data Vault

- dbt's own "How We Structure" marts guidance leans **OBT-flavored**: wide/denormalized marts over normalized fact/dimension splits, citing cheap storage / expensive compute, while still supporting `fct_`/`dim_` naming for genuine star schemas.
- **[UNVERIFIED]** dedicated dbt Kimball-vs-OBT-vs-Data-Vault comparison docs 404'd/redirected. The marts-page quote is the only directly verified dbt stance. General tradeoffs (background theory, not link-verified this pass): Kimball = reusable conformed dimensions, more joins/ETL; OBT = BI simplicity + columnar economics, redundancy/harder reuse; Data Vault (hubs/links/satellites) = auditability/historization/parallel loads at enterprise scale, needs a downstream marts layer before user-facing use.

## SQL style guides

**dbt Labs official style guide** — https://docs.getdbt.com/best-practices/how-we-style/2-how-we-style-our-sql (verbatim rules): tenets "Clarity" + "Consistency," enforce with **SQLFluff**. 4-space indent, 80-char max, lowercase keywords, trailing commas, explicit `as`. "Fields before aggregates/window functions"; "Aggregations as early as possible before joining." `group by`/`order by` numbers preferred. Joins: `union all` over `union`; always prefix columns with table name when joining 2+; explicit `inner join`; repeated `right join` is a smell. **Import CTEs** (all `ref()`/`source()` at top, column-pruned) vs **Functional CTEs** (one logical unit, verbosely named); final line is `select *` from the final CTE.
- **[UNVERIFIED]** GitLab data-team SQL style guide exists (https://handbook.gitlab.com/handbook/business-technology/data-team/platform/sql-style-guide/) but only nav/TOC retrieved — don't cite specific GitLab rules as confirmed. Matt Mazur's dbt style guide URL 404'd — not located.
- **sqlfluff** — https://sqlfluff.com/ — "The SQL Linter for humans," dialect-flexible, Jinja/dbt-aware, lints+fixes+parses. **sqlfmt** — https://sqlfmt.com/ — opinionated formatter for dbt SQL, no config, playground runs client-side.

## Semantic / metrics layers

- **dbt Semantic Layer / MetricFlow** — https://docs.getdbt.com/docs/build/semantic-models: "Semantic models are the foundation for data definition in MetricFlow, which powers the dbt Semantic Layer." Entities have 4 types (primary, unique, foreign, natural); dimensions are group-by axes. Explicitly solves the **metric-drift problem** via one authoritative definition consumed across tools (Tableau, Excel, Hex, Mode).
- **Cube (headless BI)** — https://docs.cube.dev/docs/introduction: "built on an open-source semantic layer," "centralizes metric definitions, joins, access rules, and caching upstream of every BI tool, application, and AI agent." **Cubes** (measures+dimensions) + **Views** (curated datasets). "Semantic SQL" adds a `MEASURE` function so every query is validated against the semantic layer before hitting the warehouse — anti-drift for AI-generated SQL too.
- **LookML** — https://docs.cloud.google.com/looker/docs/what-is-lookml: "write SQL expressions once, in one place, and Looker uses the code repeatedly to generate ad hoc SQL." **Views** + **Models** + **Explores**; git-native by design, predates "metrics as code" by a decade.

## BI / dashboards-as-code

- **Lightdash** — https://docs.lightdash.com/: connects to a dbt project via git/dbt-Cloud-API/CLI; reuses dbt-defined metrics rather than duplicating; auto-syncs on commit. Current positioning shifting to "Agentic Analytics."
- **Evidence.dev** — https://docs.evidence.dev/: "renders a BI website from markdown files" containing SQL — code-driven, fully git-versionable. Requires SQL.
- **Metabase** — https://www.metabase.com/docs/latest/: OSS BI, GUI-first, **no git-versioning/dashboards-as-code** capability found in docs.
- **Apache Superset** — https://superset.apache.org/: Apache 2.0, 40+ chart types, Jinja templating, config in a metadata DB by default (YAML export/import exists in community knowledge, not verified here).
- **Hex** — https://hex.tech/: "AI-powered analytics platform," agentic notebooks + data apps; git integration ("export to GitHub/GitLab"). Positioning heavily agentic/AI-first now — treat as current marketing framing.

---

# Cluster 3 — Platform & Movement

> **[LEAD-PATCHED items, verified 2026-07-25]** are called out inline. The worker flagged Debezium, the streaming-overkill source, and BigQuery $ figures as gaps, and flagged the Prefect→Dagster claim for second-sourcing — all now resolved below.

## ELT vs ETL

- **ELT is the modern default** — decouples extraction/loading from transformation so transformation delays no longer block landing; enabled by the collapse in cloud storage/compute cost; SQL-based in-warehouse transforms let analysts own transformation logic. https://www.fivetran.com/blog/etl-vs-elt (2026-07-25)
- **ETL still wins** in narrow cases: stable/well-established schemas, and compliance-critical contexts requiring transformation (PII scrubbing/masking) *before* data lands in a shared warehouse. (same source)
- Caveat: Fivetran is an ELT-tool vendor — directionally consensus but not neutral.

## Warehouse / lakehouse landscape

**Snowflake cost model** (https://docs.snowflake.com/en/user-guide/warehouses-overview, 2026-07-25):
- 8 warehouse sizes (X-Small→6X-Large), each doubling compute + credit burn: X-Small=1 credit/hr … 6X-Large=512 credits/hr.
- Billed per-second, 60-second minimum. Auto-suspend/auto-resume ON by default — the single biggest idle-cost lever.
- Multi-cluster warehouses (Enterprise+) scale concurrency without resizing.
- Right-sizing is exponential, not linear (2X-Large ≠ 2× X-Large — it's 32×).

**BigQuery pricing** **[LEAD-PATCHED, verified 2026-07-25 from https://cloud.google.com/bigquery/pricing]**:
- **On-demand analysis: $6.25 per TiB scanned; first 1 TiB per month free.** (10 MiB/query minimum metering; up to 2,000 slots/project, no commitment.)
- **Capacity compute (per slot-hour, pay-as-you-go w/ autoscaling): Standard $0.04, Enterprise $0.06, Enterprise Plus $0.10.** 1yr/3yr commitments discount these (e.g. Enterprise ~$0.048 / ~$0.036). Min 50 slots/reservation.
- **Storage**: active logical ~$0.02/GiB-month ($0.000027397/GiB-hour, verified on page); long-term logical ~$0.01/GiB-month; physical-storage active ~$0.04/GiB-month.
- Tradeoff: on-demand = variable cost for sporadic workloads; slots = predictable baseline for sustained heavy volume. BI Engine adds separate GiB-based memory-caching reservations.
- (Worker's original note: the reservations-intro page deferred $ figures to the pricing page and left them unverified — the LEAD-PATCH above resolves that.)

**Databricks / Unity Catalog**: governance layer that is "format-agnostic" — works with Delta, Iceberg, Parquet without data movement; an "Open Sourcing Unity Catalog" initiative referenced; implements the open Iceberg REST Catalog API so Iceberg-native clients can read UC-governed tables. https://www.databricks.com/product/unity-catalog (2026-07-25)

**DuckDB / MotherDuck**: DuckDB = MIT-licensed, in-process/embedded OLAP engine, zero external deps, columnar-vectorized, runs edge→multi-TB server→browser (DuckDB-Wasm). Serious analytical power without the operational burden of a distributed server. https://duckdb.org/why_duckdb (2026-07-25)

## Open table formats

| Format | Governance/License | Differentiator | Version (fetched 2026-07-25) |
|---|---|---|---|
| **Apache Iceberg** | Apache Software Foundation, Apache 2.0 | Hidden partitioning, schema/partition evolution without rewrites, time travel, multi-engine | 1.11.0 stable — https://iceberg.apache.org/ |
| **Delta Lake** | Linux Foundation, open governance | ACID/serializable isolation, metadata to billions of partitions, unified batch+streaming exactly-once | 4.3.1 on Spark 4.1.0 — https://delta.io/ |
| **Apache Hudi** | Apache Software Foundation | Record-level indexing/upserts, minute-level incremental processing — most "OLTP-on-lake" leaning | https://hudi.apache.org/ |

- **Format convergence via UniForm**: Delta's Universal Format generates Iceberg- and Hudi-compatible metadata over the *same* Parquet files (metadata is the only real difference since all three sit on Parquet). Effectively sidesteps the format war. https://www.databricks.com/blog/delta-uniform-universal-format-lakehouse-interoperability (2026-07-25)

## Catalog wars

- **Apache Polaris**: Apache-licensed (ASF-governed), "open-source, fully-featured catalog for Apache Iceberg," implements the Iceberg REST API (Doris, Flink, Spark, Dremio OSS, StarRocks, Trino). v1.6.0 as fetched. https://polaris.apache.org/ (2026-07-25). **[UNVERIFIED]** Snowflake-origin/incubating status not surfaced on this fetch — treat as recalled, not source-confirmed.
- Net: the **Iceberg REST Catalog spec** is becoming the interop lingua franca — Unity, Polaris et al. all implement it; "catalog wars" is now about governance/AI features layered on a shared REST contract, not incompatible wire protocols.

## Orchestrators

- **Airflow 3.0**: released **2025-04-22** (https://airflow.apache.org/blog/airflow-three-point-oh-is-here/, 2026-07-25). Headline: DAG versioning, client-server Task Execution API/Task SDK (AIP-72, multi-language starting with Go), React+FastAPI UI rewrite, Datasets → event-driven "Data Assets" with Watchers, scheduler-managed backfills, Flask AppBuilder moved to a provider package.
  - **Current version as of fetch: Airflow 3.3.0, released 2026-07-06** — https://airflow.apache.org/docs/apache-airflow/stable/release_notes.html (2026-07-25). Date-stamp explicitly; Airflow ships frequently.
- **Dagster**: asset-based paradigm — data outputs are first-class (`@dg.asset`), dependencies derive from data lineage not execution order. Prefer when lineage/governance/catalog visibility matters more than procedural sequencing. https://docs.dagster.io/concepts/assets/software-defined-assets (2026-07-25)
- **Prefect**: dynamic Python-native workflows (`@flow`/`@task`), no DAG declared up front; durability built-in. Apache 2.0 core + Prefect Cloud. https://www.prefect.io/ (2026-07-25)
- **[LEAD-PATCHED, SECOND-SOURCED & CONFIRMED 2026-07-25]** **Prefect is acquiring Dagster.** Confirmed via the Dagster blog's featured post — **"Prefect is Acquiring Dagster," a letter from Nick Schrock (founder/creator of Dagster), dated 2026-07-13** — https://dagster.io/blog. This consolidates the two leading orchestrators (separate identities retained, no customer action required). Enter as dated fact (not disputed).

## Ingestion

- **Fivetran**: MAR (Monthly Active Rows) = inserts + updates per calendar month, excludes unchanged rows and initial syncs. Free tier: 500K MAR (connections) / 3,500 MAR (activations) / 5,000 model runs. Tiers gate sync frequency (15-min → 1-min) and compliance (PCI DSS L1, CMK). Estimator-based pricing, no public per-unit $. https://www.fivetran.com/pricing (2026-07-25)
- **Airbyte**: AI-agent-operations pricing (1K–10K AOs/mo, overage $0.004–0.005/AO); volume-based Standard ($10/mo+)/Plus ($500/mo, 50 credits, 15-min sync); capacity-based Pro (dedicated Data Workers, no per-row/GB charge); free self-managed **Core** (OSS). https://airbyte.com/pricing (2026-07-25)
- **dlt**: open-source Python **library** (`pip install dlt`), runs anywhere Python runs. Auto schema inference/normalization of nested data, incremental loading, schema evolution. Code-first/developer-owned vs Fivetran/Airbyte platforms; dltHub is the optional commercial layer. https://dlthub.com/docs/intro (2026-07-25)
- **Meltano**: open-source ETL on the **Singer spec** (taps/targets); SDK for custom connectors; 600+ connectors via Meltano Hub. https://meltano.com/ (2026-07-25)

## CDC & streaming

- **Kafka**: current version **4.3** in the releases dropdown at fetch (https://kafka.apache.org/documentation/, 2026-07-25). Ships fast — re-verify before quoting.
- **Debezium** **[LEAD-PATCHED, verified 2026-07-25]**: latest stable **v3.6.0.Final, released 2026-07-01**; prior maintained line 3.5.x (v3.5.2.Final, 2026-06-02). Source: https://github.com/debezium/debezium/tags (debezium.io/documentation and /releases both 403 to automated fetch). **Log-based CDC mechanics**: Debezium is built on Kafka Connect and tails the database transaction/commit log (Postgres WAL, MySQL binlog, Oracle redo, SQL Server change tables), converting row-level INSERT/UPDATE/DELETE into ordered change events with before/after images — low source-DB overhead, no query polling, preserves ordering. (Mechanics from canon; primary Debezium docs were unreachable, so not directly quoted.)
- **"You don't need streaming" discourse** **[LEAD-PATCHED, verified 2026-07-25]** — the streaming-specific canonical essay could not be located (WebSearch exhausted), but two adjacent canonicals anchor the "distributed/streaming is usually overkill" argument:
  - **Frank McSherry, "Scalability! But at what COST?" (2015-01-15)** — https://www.frankmcsherry.org/graph/scalability/cost/2015/01/15/COST.html. COST = "Configuration that Outperforms a Single Thread." Across published Spark/Giraph/GraphLab/GraphX PageRank + connectivity benchmarks on 128 cores, "none of the scalable systems consistently out-perform a single thread" laptop; with a better algorithm a single thread did graph connectivity in ~8s vs 200s+ for clusters. **Rule: a distributed system must beat one laptop thread before its complexity is justified.**
  - **Jordan Tigani, "Big Data is Dead" (MotherDuck)** — see below.
  - General shape: batch/micro-batch is sufficient for most latency needs; streaming infra (Kafka/Debezium/Flink) is justified only by a genuine sub-minute-latency or true event-driven-architecture requirement — not "it feels real-time-ish."

## Reverse-ETL / activation

- **Fivetran acquired Census** (per TechCrunch, dated **2025-05-01**, referenced on Fivetran's page). `getcensus.com/reverse-etl` now 301-redirects to `fivetran.com/data-movement/activations` — Census is being folded into Fivetran's activation line. https://www.fivetran.com/data-movement/activations (2026-07-25). **Load-bearing, date-sensitive** — re-verify near publication.
- **Reverse ETL** = syncing enriched warehouse data back into operational SaaS tools (CRM/marketing/support) without manual API code. **Data activation** = the applied use of that synced data. 200+ destinations cited. (same source)
- **Hightouch**: "Composable CDP" — real-time activation from the warehouse as an alternative to monolithic CDPs (Salesforce, HubSpot, Google/Facebook Ads, The Trade Desk). https://hightouch.com/ (2026-07-25)

## Cost economics / FinOps for data

- Snowflake: right-sizing (exponential credit cost by tier) + auto-suspend are the two dominant levers; multi-cluster trades cost for concurrency, not efficiency.
- BigQuery: on-demand vs slots is a variable-vs-fixed decision gated by query-volume predictability; BI Engine caching is a separate lever.
- Cross-cutting: most warehouse spend problems are **workload-mismatch** problems — provisioning distributed-scale compute for sub-100MB-per-query workloads — not genuine big-data needs.

## Small-data movement — "Big Data is Dead"

Jordan Tigani (MotherDuck, ex-BigQuery founding engineer) — https://motherduck.com/blog/big-data-is-dead/ (fetched 2026-07-25; **exact publication date not shown on the page — don't date-stamp beyond fetch date**):
- Majority of BigQuery customers stored **<1TB**; median heavy user **<100GB**.
- **90% of queries processed <100MB of data** among $1,000+/yr customers.
- Recent data dominates access: "the most recent month might have 5% of data but 80% of data accesses."
- A PB-scale query costs ~$5,000 at retail on-demand — illustrating waste of distributed provisioning for occasional big queries.
- Modern single machines (64 cores / 256GB RAM) are ~100× more capable than 2006 hardware, closing the gap that justified distributed systems.
- Caveat: MotherDuck (DuckDB vendor) blog — directionally credible (Tigani ex-BigQuery), self-serving for single-node positioning; treat stats as Tigani's analysis.

---

# Cluster 4 — Trust, Governance & AI-Era Data

## Data quality frameworks

**Great Expectations (GX)** — https://docs.greatexpectations.io/docs/core/introduction/ (2026-07-25). Python validation library: Expectations (declarative assertions), Expectation Suites, Data Context, Checkpoints (suite + validation run). Docs-nav version: **1.19.1** (0.18.x legacy). **[UNVERIFIED]** version number as shown on docs nav, not cross-verified; Data Context/Checkpoint internals behind deeper pages not fetched.

**Soda / SodaCL** — https://docs.soda.io/quickstart (2026-07-25). YAML check language: `row_count` (volume), `schema` (structural), `missing` (nulls), `invalid` (ranges/valid_values), `failed_rows` (custom expressions). Architecture: **Soda Cloud** (UI) + **Soda Core** (`pip install soda-core`) + **Soda Runner**. **[UNVERIFIED]** license tiers (Dataset Standard/Team/Enterprise) not confirmed from fetch.

**Elementary** — https://docs.elementary-data.com/ (2026-07-25). "Data observability and governance platform for the AI era," dbt-native. **Elementary OSS** (dbt package) vs **Elementary Cloud** (AI agents, incident management, governance UI). ML anomaly monitors: freshness, volume, schema, distribution — same five-pillar shape as Monte Carlo. Claims column-level lineage ingestion→BI.

**dbt tests** — https://docs.getdbt.com/docs/build/data-tests (2026-07-25). SQL `select` statements returning failing rows (pass = zero rows). **Singular** (`.sql` in `tests/`) + **Generic** (4 built-in: `unique`, `not_null`, `accepted_values`, `relationships`, via `data_tests:` YAML). `--store-failures` persists failing rows; `severity: warn|error` + `error_if`/`warn_if` thresholds. Invariant: every model's primary key gets `unique` + `not_null`.

**dbt unit tests** — https://docs.getdbt.com/docs/build/unit-tests (2026-07-25). Validate SQL *logic* pre-materialization on static mocked inputs; different axis from data tests (production data, post-materialization). Defined in `models/*.yml`, SQL-models-only. dbt Labs recommendation: run in dev/CI only, exclude from prod (`--exclude-resource-type`) to save compute; `dbt build` sequences unit tests → materialization → data tests.

**dbt_utils** — https://github.com/dbt-labs/dbt-utils (2026-07-25). Apache-2.0, dbt-Labs-maintained. Adds `equal_rowcount`, `expression_is_true`, `recency`, `equality`, `at_least_one`, `not_constant`, `mutually_exclusive_ranges`, `unique_combination_of_columns`, `cardinality_equality`.

**dbt-expectations** — https://github.com/calogica/dbt-expectations (2026-07-25). Ports GX-style tests into dbt macros. Apache 2.0, dbt ≥1.7. **GOTCHA**: repo carries an explicit "no longer actively supported" notice — maintenance risk, don't present as a safe default without flagging staleness.

**Comparison**: GX = standalone Python engine (heaviest, most expressive, separate infra); Soda = YAML DSL + SaaS cloud; Elementary = dbt-native package (cheapest to adopt inside dbt); dbt tests/utils/expectations = "quality as SQL assertions colocated with transformation" (cheapest ops, weakest ML/anomaly detection — that's what Elementary/Monte Carlo/Soda add).

## Anomaly detection & "data downtime"

Monte Carlo (Barr Moses, 2019) — https://montecarlo.ai/blog-what-is-data-observability/ (2026-07-25). **"Data downtime"** = periods when data is "partial, erroneous, missing, or otherwise inaccurate." Sizing heuristic (Monte Carlo's own illustrative rule, not an industry formula): annual incidents ≈ (table count ÷ 15) × avg detect+resolve time. Core split: observability covers **known unknowns** (rule-based tests) *and* **unknown unknowns** (ML/statistical anomaly detection because tests can't scale to every failure mode) — the decision rule for when to add anomaly detection on top of dbt tests.

## Data observability — five pillars

Same source: **Freshness** (update cadence), **Distribution/Quality** (null rates, value ranges), **Volume** (row-count completeness), **Schema** (structural change), **Lineage** (upstream/downstream metadata). Incident loop: detect → alert → investigate root cause → recommend/apply fix — SRE incident response applied to data pipelines.

## Lineage — OpenLineage / Marquez

**OpenLineage** — https://openlineage.io/docs/ (2026-07-25). Open spec, governed as an **LF AI & Data Graduate project**. Core model: **datasets, jobs, runs**, extended via **facets** (user-defined metadata). Rationale: standardize integration so "integration efforts are shared across projects." **[UNVERIFIED]** columnLineage-facet field details — only the general facets index was retrievable; direct fetch of the OpenLineage.json spec still needed.

**Marquez** — https://marquezproject.ai/ (2026-07-25). Reference implementation of OpenLineage; originally open-sourced by WeWork, now under LF AI. Real-time metadata collection, visual lineage graph UI, Lineage API for root-cause/backfill. Integrates with Airflow, Spark, Flink, dbt, Dagster. **[UNVERIFIED]** SPDX license string not confirmed from fetch.

## PII & privacy operations

**Snowflake classification/tagging/masking** — https://docs.snowflake.com/en/user-guide/governance-classify (2026-07-25). Automated: classification assigns semantic category + privacy category (IDENTIFIER / QUASI_IDENTIFIER / SENSITIVE) → system tags auto-applied → user tags can map to system tags → **tag-based masking policies auto-mask the moment classification tags a column**. **[UNVERIFIED]** no explicit GDPR/CCPA citation on this page — compliance framing is generic; don't attribute a specific regulatory claim to Snowflake docs here.

**BigQuery column-level security** — https://docs.cloud.google.com/bigquery/docs/column-level-security-intro (2026-07-25). Policy-tag-based: Data Catalog taxonomies → tags on columns (1 tag/column, depth ≤5, ≤1,000 tags/table) → IAM roles gate access. Layers on top of dataset ACLs (both required). Gotcha: read access not needed for INSERT into a protected column, but UPDATE/DELETE/MERGE require Fine-Grained Reader.

**GDPR Article 17 (right to erasure)** — https://gdpr-info.eu/art-17-gdpr/ (2026-07-25). Six erasure grounds vs Art.17(3) exceptions (free expression, legal/public-interest compliance, public health, archival/research/statistical if erasure "would render impossible or seriously impair" the purpose, legal-claims defense). When data was made public, controller must take "reasonable steps, including technical measures" to notify other processors — the operational root of **cascading deletion**. Warehouse tension: real-time erasure vs append-only/backup/snapshot architecture — justification for **tombstone + scheduled hard-delete** rather than in-place mutation of historical tables.

**Data catalogs / governance platforms**:
- **DataHub** — https://docs.datahub.com/docs/features (2026-07-25). OSS (Docker/K8s) + managed **DataHub Cloud (Acryl)**. Built-in ownership, PII tracking, anomaly detection, assertions, data contracts.
- **OpenMetadata** — https://docs.open-metadata.org/latest (2026-07-25). Discovery/lineage/governance; RBAC, data-quality dimensions, data contracts, glossary, 80+ connectors, MCP integration. **[UNVERIFIED]** license & DataHub comparison not in fetched content.
- **Unity Catalog** — https://docs.databricks.com/en/data-governance/unity-catalog/index.html (2026-07-25). Three-level namespace `catalog.schema.object`; securable objects; managed vs external; row/column-level security via filters; automatic lineage + sensitive-data classification; system-table auditing. Gotcha: auto-enabled only for workspaces created after Nov 8, 2023.

## Org discourse — data mesh

Zhamak Dehghani, "Data Mesh Principles and Logical Architecture" — https://martinfowler.com/articles/data-mesh-principles.html (2026-07-25). Four principles:
1. **Domain-oriented decentralized ownership** — ownership follows business-domain boundaries, not a central team.
2. **Data as a product** — discoverable, secure, understandable, trustworthy, high-quality; introduces the **domain data product owner** role (measured on quality, consumption lead time, satisfaction).
3. **Self-serve data platform** — abstracted infra so generalist domain engineers build data products without big-data specialization.
4. **Federated computational governance** — decentralized autonomy + globally enforced interoperability standards, automated by the platform. Quote (Holling): *"Placing a system in a straitjacket of constancy can cause fragility to evolve."*
Tradeoff: quality ownership shifts upstream to the domain; topology static/monolithic → dynamic/polyglot; error-handling prevention-only → detect-and-recover.

**dbt Mesh** — https://docs.getdbt.com/best-practices/how-we-mesh/mesh-1-intro (2026-07-25). Multi-project: cross-project `{{ ref() }}` (Enterprise+), cross-project lineage, **Groups**, **Access modifiers**, **Model Versions**, **Model Contracts**. Adoption triggers: model-count perf degradation, decouple team workflows, reliability/communication decay, security/isolation. dbt Labs guidance: start single-project, adopt Mesh incrementally — "don't over-engineer governance early."

## Experiment measurement validity

**Sample Ratio Mismatch (SRM)** — Kohavi's Trustworthy Online Controlled Experiments (https://exp-platform.com/, 2026-07-25; direct MS SRM article 404'd, via exp-platform summary): observed traffic-split deviates from intended ratio; "a symptom for a variety of data quality issues" — like a fever, tells you something is wrong, not what.

**OEC & guardrails** — short-term OEC movement can be a bad proxy for long-term value (price increases lift short-term revenue, suppress LTV) → pair a primary/OEC metric with guardrail metrics that must not regress.

**Twyman's Law**: "any figure that looks interesting or different is usually wrong" — surprising results demand investigation before trust.

**Assignment vs exposure/triggered analysis** — distinguish assigning users to a variant from actually exposing them; triggered/exposure-based analysis restricts measurement to users who experienced the change and needs **dilution correction** to translate back to a population effect.

**Common pitfalls**: multiple-comparison false positives, Simpson's paradox during gradual rollouts, bot/robot traffic, unchecked textbook-stats assumptions, metric misinterpretation, survivorship bias, continuous monitoring without a stopping rule (the **peeking problem**).

**CUPED** — https://docs.growthbook.io/statistics/cuped (2026-07-25). Adjusted metric = post-exposure avg − θ × pre-exposure avg. Reported (in GrowthBook docs, not independently verified): Netflix (2016) ~40% variance reduction; Microsoft (2022) ~equivalent to 20% more traffic. Needs a correlated pre-period covariate; default 14-day lookback. Non-applicable to: quantile metrics, legacy ratio metrics, bandit goal metrics, Mixpanel-sourced metrics, custom user-value aggregations.

**Sequential testing** (peeking fix), three vendors (2026-07-25):
- **GrowthBook** — https://docs.growthbook.io/statistics/sequential — Asymptotic Confidence Sequences (Waudby-Smith 2023); tuning N* (default 5,000); wider CIs by design.
- **Statsig** — https://docs.statsig.com/stats-engine/sequential-testing — **mSPRT** (Zhao et al.); auto-adjusts p-values/CIs per look; limit to a few key metrics.
- **Eppo** — https://docs.geteppo.com/statistics/confidence-intervals/ — guaranteed confidence level regardless of peek count, at the cost of wider intervals.
Converged industry pattern: **sequential testing trades interval width for peek-safety** (Confidence Sequences vs mSPRT), a mature standard, not one vendor's trick.

## AI-era data plane

**LLM trace/token-log warehousing** — Langfuse, https://langfuse.com/docs/tracing (2026-07-25). Hierarchy: **Trace** (request lifecycle) → **Observations** (nested LLM/tool/retrieval calls, each with timing/input/output/cost) → **Sessions** (multi-turn) → **Environments** (deployment stage). Token usage + cost tracked natively per observation. Positioned as the data substrate evals run against. Open-source, self-hostable.

**RAG corpus pipeline as governed ETL** — Databricks AI Cookbook, https://learn.microsoft.com/en-us/azure/databricks/generative-ai/tutorials/ai-cookbook/quality-data-pipeline-rag (2026-07-25, page dated 2026-06-30). Explicit governed-ETL framing: "raw source data should be ingested and stored in a target table... ensures data preservation, traceability, and auditing." Stages: corpus ingestion → parsing (**explicit PII/HIPAA tagging as metadata**) → enrichment → dedup (MinHash/LSH) → **filtering incl. PII-detection-driven exclusion + data-poisoning screening** → chunking (fixed/paragraph/format/semantic — no one-size-fits-all) → embedding (model tradeoffs, truncation risk) → indexing (chunks + metadata co-located in a Delta table backing the vector index). Strongest "RAG-as-governed-ETL" primary doc found — anchor the sub-topic here.

**Evals data structure** — DeepEval, https://deepeval.com/docs/getting-started (2026-07-25). `Golden` objects (min. an `input`) form an `EvaluationDataset`; single-turn cases carry `input`/`actual_output`/optional `expected_output`; multi-turn use role/content `Turn` sequences; metrics score 0–1 vs a `threshold` (default 0.5). Local results persist via `DEEPEVAL_RESULTS_FOLDER` (git-versionable); Confident AI cloud adds regression comparison + production trace monitoring. Open source.

---

# Candidate Invariants (the skill could teach these as non-negotiables)

1. **One naming grammar, enforced.** Pick a single event-naming grammar org-wide (object-action, past tense, one casing) and enforce it with a namespace/lint check — inconsistent casing/tense is the #1 cause of fragmented analytics.
2. **Reserved properties are a contract.** Never overload a reserved/semantic property (`revenue`, `currency`, `value`; user-property names) with ad-hoc meaning — destinations and pipelines depend on it.
3. **Model shared data as entities, not repeated properties.** If data describes something used across multiple events, it belongs in an entity/trait, not a per-event property.
4. **Every schema is explicitly versioned, and the team knows which bump is safe per warehouse** (Redshift's required↔optional rules differ from Snowflake/BigQuery/Databricks).
5. **A data contract is bidirectional** — producers owe backward-compatible, notified schema evolution; consumers owe explicit documented requirements upfront. Default to the strictest registry compatibility mode (FULL/FULL_TRANSITIVE) you can.
6. **Validate at ingestion; quarantine, don't silently drop or accept.** Events are checked against the tracking plan on the way in.
7. **Consent gates collection before the first measurement call — and "denied" is tested, not assumed** (URL/IP data can still leak when ad-storage is denied).
8. **Staging is 1:1 with sources, view-materialized, no joins/aggregations.** That logic lives downstream.
9. **Narrow the DAG, widen the tables.** Many inputs into a model is healthy; many outputs (esp. repeated `right join`s) is a smell. Marts default wide/denormalized; >4-5 joins → split into an intermediate model.
10. **Materialization is a ladder** — view → table → incremental — climbed only when the previous tier's build/query time is the real bottleneck. Don't pre-optimize into incremental.
11. **"Free" ≠ "open source."** dbt Core is Apache 2.0 forever; dbt Fusion is proprietary (dbt Product Licensing Agreement) even though free to use locally.
12. **A metrics/semantic layer has one job: a single authoritative metric definition consumed by every tool/agent.** If two tools compute "revenue" differently, it has failed (metric drift).
13. **ELT is the default; ETL survives only where schema is frozen or transform-before-landing is a compliance requirement.**
14. **Warehouse cost is a workload-shape problem before a pricing-model problem** — most orgs pay distributed-scale prices for single-node-scale queries. Auto-suspend + right-sizing are the highest-leverage levers before touching architecture.
15. **Table-format choice barely matters at storage (Parquet underneath all three); it matters at the catalog/REST-API layer** — evaluate catalogs by Iceberg-REST-spec compliance, not brand.
16. **Streaming (Kafka/Debezium/Flink) is justified by a genuine sub-minute-latency or event-driven need — not "it feels real-time."** A distributed system must beat one laptop thread (COST) before its complexity is warranted; the median analytical workload fits a single node.
17. **Rule-based tests catch known unknowns; only statistical/ML anomaly detection catches unknown unknowns — you need both layers.** Every primary key gets `unique` + `not_null` as the floor.
18. **Enforce PII masking at the platform layer** (Snowflake tag-based masking, BigQuery policy tags, Unity Catalog filters) — application-layer redaction is bypassed by any warehouse query.
19. **"Delete everything on request" is wrong — GDPR Art.17 has real exceptions;** erasure logic must check exception grounds, and warehouse deletion is tombstone + scheduled hard-delete, not in-place history mutation.
20. **Every experiment result clears an SRM check before it's trusted**, and continuous monitoring uses a peeking-safe estimator (confidence sequences, mSPRT, or a pre-registered fixed horizon) — "check it whenever" is not a valid method.
21. **A RAG ingestion pipeline is ETL and must be governed as such** — raw-source landing table, lineage/auditability, PII/toxicity filtering, and dedup before chunks reach an embedding index.
22. **Data mesh's domain ownership only works with federated, platform-enforced governance** — decentralizing without guardrails just recreates silos with extra steps; start central, adopt mesh incrementally.

---

# Top 20 Canonical Sources

1. Segment/Twilio Spec (event taxonomy) — https://www.twilio.com/docs/segment/connections/spec
2. Avo — naming conventions & tracking-plan design — https://www.avo.app/docs/data-design/best-practices/naming-conventions
3. Snowplow — tracking design best practice + SchemaVer — https://docs.snowplow.io/docs/fundamentals/tracking-design-best-practice/
4. Chad Sanderson, "The Rise of Data Contracts" (2022-08-22) — https://dataproducts.substack.com/p/the-rise-of-data-contracts
5. Open Data Contract Standard v3.0.0 (Bitol/LF, ex-PayPal) — https://bitol-io.github.io/open-data-contract-standard/v3.0.0/home/
6. Confluent Schema Registry — compatibility/evolution — https://docs.confluent.io/platform/current/schema-registry/fundamentals/schema-evolution.html
7. Google Consent Mode — https://developers.google.com/tag-platform/security/guides/consent
8. dbt Labs — How We Structure (staging/intermediate/marts, DAG rules) — https://docs.getdbt.com/best-practices/how-we-structure/1-guide-overview
9. dbt Labs — How We Style Our SQL — https://docs.getdbt.com/best-practices/how-we-style/2-how-we-style-our-sql
10. dbt Labs — Materializations & Incremental Models — https://docs.getdbt.com/docs/build/materializations
11. dbt Labs — Licenses FAQ (Core Apache 2.0 vs Fusion proprietary) — https://www.getdbt.com/licenses-faq
12. dbt Labs — Semantic Models / MetricFlow — https://docs.getdbt.com/docs/build/semantic-models
13. Cube — headless BI / semantic SQL — https://docs.cube.dev/docs/introduction
14. Apache Airflow 3.0 announcement + release notes (Airflow 3.3.0, 2026-07-06) — https://airflow.apache.org/blog/airflow-three-point-oh-is-here/
15. Apache Iceberg (v1.11.0) + Delta UniForm interoperability — https://iceberg.apache.org/ , https://www.databricks.com/blog/delta-uniform-universal-format-lakehouse-interoperability
16. BigQuery pricing ($6.25/TiB on-demand; slot tiers $0.04/$0.06/$0.10) — https://cloud.google.com/bigquery/pricing
17. Jordan Tigani, "Big Data is Dead" + Frank McSherry, "Scalability! But at what COST?" (2015) — https://motherduck.com/blog/big-data-is-dead/ , https://www.frankmcsherry.org/graph/scalability/cost/2015/01/15/COST.html
18. Monte Carlo — What is Data Observability (five pillars + data downtime) — https://montecarlo.ai/blog-what-is-data-observability/
19. Zhamak Dehghani — Data Mesh Principles (Martin Fowler) — https://martinfowler.com/articles/data-mesh-principles.html
20. Kohavi et al., Trustworthy Online Controlled Experiments (SRM, guardrails, peeking) — https://exp-platform.com/ ; and Databricks AI Cookbook — governed RAG data pipeline — https://learn.microsoft.com/en-us/azure/databricks/generative-ai/tutorials/ai-cookbook/quality-data-pipeline-rag

---

# Open Gaps (flagged for a follow-up pass — do NOT teach as confirmed)

- **dbt/SDF acquisition** date + exact Fusion-announcement blog post (candidate URLs 404'd).
- **Segment Typewriter** current maintenance status; **Avo Inspector** dedicated docs (404'd).
- **Snowplow** literal "MODEL-REVISION-ADDITION" vocabulary vs major/minor/patch phrasing.
- **Kimball vs OBT vs Data Vault** dbt-authored comparison doc (404'd) — only the marts-page OBT lean is verified.
- **GitLab / Matt Mazur** SQL style-guide bodies (nav-only / 404).
- **Streaming-specific** "you don't need Kafka" canonical essay — anchored on COST + Big Data is Dead instead ("You Are Not Google" WebFetch returned empty).
- **SodaCL** license tiers; **Marquez** SPDX license; **OpenLineage** columnLineage facet field schema; **GX** Data Context/Checkpoint internals; **OpenMetadata vs DataHub** feature comparison.
- WebSearch budget was exhausted session-wide mid-run, so several sub-topics rest on direct-URL fetches with gaps where a guess 404'd; a fresh-search pass would close them.
