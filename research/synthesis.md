# `data` skill — Research Synthesis & Build-Gate Proposal

**As-of:** 2026-07-25. Synthesized from the four research corpora in this directory
(`skills-corpus.md`, `github-corpus.md`, `web-canon-corpus.md`, `x-corpus.md`) — 4 channels,
~300 unique sources, every volatile claim date-stamped in its corpus. This document is the
decision artifact for the build gate.

---

## 1. Wedge verdict: OPEN

No unified, vendor-neutral, faceted-router (jobs × surfaces) data-engineering skill exists
as-of 2026-07-25. Triple-confirmed by independent deep-reads. The field is exclusively:

1. **Official vendor packs** (dbt, PostHog, Google/BigQuery, Astronomer/Airflow, Databricks,
   Snowflake, AWS, Confluent, DuckDB, Grafana, Metabase, Elastic, Datadog…) — dense demand
   signal, all single-vendor, no cross-vendor decision layer.
2. **Single-topic skills in generic grab-bags** (wshobson/agents ×175, awesome-copilot ×391).
3. **Aggregated dumps** (one literally bot-scraped, incl. a cracked-software repo it ingested).

Closest router mechanics found: jeffallan `spark-engineer` (thin router + dispatch table +
`related-skills` frontmatter) and astronomer's `airflow` entrypoint — both single-persona or
single-vendor. **The neutral, decision-first layer that routes across the whole data plane is
unclaimed.** (Residual: Snowflake/Databricks/Fivetran/Airbyte/Segment pack internals were
listed but not deep-read; none appeared router-shaped in discovery.)

## 2. Mission (from the signed-off separation of concerns)

`data` owns the **analytical and governed data plane**: making product/business data
**trustworthy, governed, discoverable, and usable** for decisions, reporting, experiments,
and downstream activation. Data-as-a-product — distinct from `backend`
(data-as-application-state) and `ai` (data-as-model-context).

Seams (drawn in the clarification, confirmed by research):
- **vs `backend`** — operational/OLTP plane vs analytical plane; `data` reads from
  operational stores, never serves app traffic.
- **vs `ai` (RAG three-way seam)** — `ai` owns retrieval *behavior*; `backend` owns the
  vector store as served infra; **`data` owns the governed pipeline + vector store / LLM
  traces as datasets.** Research anchor: Databricks' governed-RAG-pipeline doc (2026-06-30)
  frames RAG ingestion as governed ETL with PII tagging + poisoning screens — the seam is in
  the primary literature.
- **vs `frontend`/`backend` (instrumentation)** — `data` defines the event contract and
  validates trustworthiness; siblings emit against it.
- **vs `product`/`growth`** — `product` picks the outcomes; `growth` designs/interprets
  experiments; `data` guarantees measurement validity (SRM, assignment, exposure).
- **vs `architecture`/`operate`/`quality`/`automation`** — implements within decided
  boundaries; doesn't deploy/monitor infra, independently verify, or orchestrate
  multi-system business processes.

## 3. Proposed shape: 10 primary jobs × 5 surfaces (4 base + 1 additive) + handoff

### Primary jobs (one per request)

| # | Job (reference file) | Owns | Crown-jewel artifacts from research |
|---|---|---|---|
| 1 | `event-taxonomy-and-tracking` | Event naming grammar, tracking plans, instrumentation QA, consent-aware collection | Object-Action grammar (5-vendor convergent); Segment 6-call spec + reserved props; Snowplow entity-first + SchemaVer; validate-at-ingestion/quarantine; Consent Mode ("denied ≠ nothing transmitted") |
| 2 | `data-contracts-and-schema-evolution` | Producer↔consumer contracts, schema registries, evolution rules | **ODCS v3.1.0 is the unambiguous spec answer** (DCS officially deprecated); Confluent 7-mode compatibility decision table; contract-as-executable-test (datacontract-cli); Sanderson canon **counterweighted by Zach Wilson's "learn write-audit-publish instead"** |
| 3 | `ingestion-and-movement` | ELT/ETL, sync strategy, CDC, streaming-when-justified, reverse-ETL | PostHog sync-type decision table (full_refresh/incremental/append/CDC/webhook by table shape); Airbyte at-least-once + resumable-refresh semantics; Singer state-precedence gotcha; Debezium + Outbox pattern; dlt evolve/freeze/discard contracts; reverse-ETL vendor-category skepticism |
| 4 | `transformation-and-modeling` | Staging→marts layering, materialization, SQL style, modeling choice | dbt "How We Structure" verbatim invariants; **dbt-project-evaluator lint-rule decision tables** (source-fanout, ≤7 joins, chained-views, prefix conventions); materialization ladder; SQLMesh model-kind table + "never build a table more than once" contrast; Kimball/OBT/Vault tradeoffs |
| 5 | `metrics-semantics-and-bi` | Semantic/metrics layer, BI-as-code vs GUI, decision-first serving | MetricFlow metric-type table (simple/ratio/derived/cumulative); Malloy's "why a semantic layer" argument; code-first vs GUI-first BI dividing line; dbt 4-tier NL-question fallback; "dashboard graveyard" critique → build for decisions |
| 6 | `quality-and-testing` ⭐ **FLAGSHIP** | The trust gate: checks, anomaly detection, PR-time diff | **Convergent DQ taxonomy (9 buckets × 7 tools — the corpus's best artifact)**; known-unknowns (tests) vs unknown-unknowns (anomaly) split; where-to-run-checks convention; PK `unique`+`not_null` floor; Recce PR-time blast-radius diff; dbt data-vs-unit test axis |
| 7 | `lineage-and-observability` | Lineage standard, five pillars, incident response for data | OpenLineage Run/Job/Dataset+Facet model (+ replace-not-merge gotcha); DQ-assertion wire format; five pillars + "data downtime"; Google blast-radius impact-analysis workflow; Sanderson v2: "database lineage is not data lineage" |
| 8 | `governance-pii-and-retention` | PII classification/masking, erasure, catalogs, access | Platform-layer masking invariant (Snowflake tags / BQ policy tags / UC filters); GDPR Art.17 **with exceptions** → tombstone + scheduled hard-delete; Presidio decoupled detect→act; catalog choice (runs-own-tests vs ingests-results); mesh-vs-central with federated-governance caveat |
| 9 | `experiment-measurement-foundations` | Measurement validity (not experiment design — that's `growth`) | SRM-before-trust; deterministic salted-hash assignment + namespaces (PlanOut); CUPED + non-applicability list; peek-safe sequential testing (3-vendor convergent); Twyman's Law; guardrail metrics |
| 10 | `platform-and-cost` | Warehouse/lakehouse/engine choice, table formats, FinOps | **Small-data-first** (Tigani + COST + Lost Decade: 90% of queries <100MB); Postgres/DuckDB-as-warehouse consensus seam; Iceberg-REST-catalog as the real interop contract; orchestrator task-vs-asset-vs-flow table; verified $ anchors (BQ $6.25/TiB, Snowflake exponential sizing); "your bill is a design decision made against sample data" |

### Surfaces (≤1 base + additive overlay)

| Surface | Shape it reshapes jobs for | Default |
|---|---|---|
| `surface-product-analytics` | A product emitting events → tracking, product metrics, funnels, activation | **default** |
| `surface-warehouse-platform` | A central analytical platform → ELT, marts, semantic layer, BI, org-wide trust |
| `surface-operational-activation` | Data flowing back into operations → CDC, reverse-ETL, outbox, near-real-time |
| `surface-ai-data-plane` | The data plane of an AI product → RAG-corpus-as-governed-ETL, LLM-trace warehousing (Langfuse model + OTel `gen_ai.*`), point-in-time correctness (Feast), token/cost datasets |
| `surface-regulated` **(additive)** | Stacks on any base when GDPR/CCPA/HIPAA/SOC2 applies — hardens governance posture, doesn't change the base shape | additive overlay |

Plus `handoff.md` + `handoff.yaml` (with `volatile_facts`) → 16 references total (10 jobs +
5 surfaces + handoff), same envelope as `ai`.

## 4. Through-line & universal invariants

**Through-line: "contract-first, trust-as-a-gate — and subtract the buzzword."**
Every artifact ships with a contract + test + lineage; untested data is treated as broken.
And per the X-corpus meta-finding (the field "debates more than it implements"), the skill
names the durable primitive under each branded concept: data mesh → domain *ownership* with
federated governance; data contracts → *write-audit-publish + quality checks*; modern data
stack → *a database and some SQL*; semantic layer → *one governed definition per metric*.

**Universal invariants (draft — refined at build):**
1. **Contract-first.** Every dataset/event has an explicit schema, version, and evolution
   rule — and the team knows which bump is safe on *their* warehouse/registry mode.
2. **Trust is a gate, not a dashboard.** PK `unique`+`not_null` floor; validate at ingestion
   and quarantine (never silently drop/accept); rule tests for known unknowns + anomaly
   baselines for unknown unknowns; PR-time blast-radius diff before merge.
3. **Small-data-first.** Single node (DuckDB/Postgres) until measured pain; streaming only
   for a genuine sub-minute or event-driven need; a distributed system must beat one laptop
   thread (COST) before its complexity is justified.
4. **Narrow the DAG, widen the tables.** Staging 1:1/no-joins; marts wide; materialization
   is a ladder climbed on measured bottlenecks only.
5. **One metric, one definition.** The semantic layer has exactly one job; two tools
   computing "revenue" differently = failure.
6. **Governance at the platform layer.** PII masking enforced in the warehouse (tags/policy
   filters), not the app; erasure honors Art.17 exceptions via tombstone + scheduled
   hard-delete; consent gates collection *before* the first call — and "denied" is tested.
7. **No experiment read before SRM.** Assignment is deterministic + salted; monitoring is
   peek-safe or pre-registered.
8. **Retrieval-first / anti-staleness (family invariant).** Every version/price/license is
   date-stamped with a re-verify pointer; never trust a repo's API license field — read the
   LICENSE file.

## 5. Distinctive value-add (family signature)

1. **The licensing minefield, taught explicitly.** Research produced a complete 2026 map —
   three shapes: (A) open-core `ee/` (PostHog, GrowthBook, Langfuse, Confluent, Redpanda);
   (B) whole-product source-available (Airbyte ELv2, soda-core ELv2, dbt-Fusion ELv2,
   Estuary BSL→Apache-2029, Phoenix ELv2); (C) pipeline-tightened (Snowplow SLULA with a
   stale-Apache root LICENSE — the headline trap). Plus MetricFlow's AGPL→BSL→Apache arc and
   the copyleft flags (Multiwoven AGPLv3, Metabase/Grafana AGPL). No competitor teaches this.
2. **Cost economics as a first-class decision axis.** Workload-shape-before-pricing-model;
   verified $ anchors; "your Snowflake bill is a design decision made against sample data";
   auto-suspend/right-sizing before architecture.
3. **The AI-era data plane** — the assembled seam no single source packages: RAG corpus as
   governed ETL + LLM traces as warehoused datasets + point-in-time correctness + `gen_ai.*`
   conventions. Direct composition with the shipped `ai` skill via handoff.

## 6. Build plan (on approval)

- **Repo:** `~/Documents/Personal/Projects/data-skill` (public → github.com/gabros20/data-skill),
  scaffolded via `scripts/init data …` with the family site/video pipeline. Research stays in
  this repo per convention.
- **Artifacts:** thin `SKILL.md` router (decline table routes to architecture/backend/ai/
  frontend/quality/operate/automation/growth) + 16 references + ~9 assets (handoff.yaml with
  `volatile_facts`; tracking-plan starter; ODCS contract starter; DQ-check matrix; sync-type,
  compatibility-mode, orchestrator, metric-type decision tables; SQL-style lint config).
- **Evals:** activation / traversal (incl. additive-regulated stacking + decline-to-growth
  hard cases) / output / compression-ablation — mirroring the `ai` eval suite.
- **Quality bars carried from the family:** lint reference seams + encoding (two competitor
  defects found are our lint targets); independent Opus review; behavioral eval pass;
  license attribution for everything lifted (skip the two unlicensed repos).

### Build crew estimate (tokenomics gate)
~11 agents: 5–6 Sonnet writers (job-cluster references), 1 Sonnet asset/eval writer,
1 integrator (controller-adjacent), 1 independent Opus reviewer, 1 Sonnet eval-runner,
site+hero agent deferred to the release phase as with `ai`. **Estimate ≈ 6–9M tokens** —
similar envelope to the `ai` build. No relay usage in the build phase.
