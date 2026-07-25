# Platform & Cost

Purpose: Decide compute/storage shape and $ before choosing a vendor — start single-node until
measured pain justifies more, price the actual workload shape rather than the sales tier, pick a
table format/catalog for interop, pick an orchestrator by its core unit, and read every license
file yourself, because this stack's licensing shapes are more numerous and consequential than most
teams assume.

Read when:
- Choosing a warehouse/lakehouse/engine, sizing compute, or reviewing a bill that "grew on its
  own."
- Picking a table format (Iceberg/Delta) or catalog, or deciding whether a distributed engine is
  justified at all.
- Picking an orchestrator (Airflow/Dagster/Prefect/Kestra) by its actual unit of work.
- Any commercial claim about an open-source/source-available data tool — before recommending it in
  a resold or hosted context.

Skip when:
- The ask is per-table sync strategy or CDC mechanics — → `ingestion-and-movement.md` (this ref
  owns whether streaming/distributed compute is cost-justified at all, not how a given sync runs).
- The ask is the metric-type/semantic-layer decision — → `metrics-semantics-and-bi.md`.
- The ask is schema/registry compatibility mode — → `data-contracts-and-schema-evolution.md`.
- The ask is deploying/monitoring the chosen platform in production — → `operate`.

Inputs:
- Actual query/workload sizes (P50/P90/P99 scanned bytes), not an assumed "big data" problem.
- Current or projected monthly spend, and whether it's variable (on-demand) or committed
  (reserved/slots).
- Whether output will be resold or hosted for third parties — decides which license shapes are
  load-bearing.
- Existing table format / catalog in use, if any.

Produces: a single-node-vs-distributed call backed by measured workload evidence, a $/workload-
shape read (not brand loyalty), a table-format + catalog pick, an orchestrator pick by core unit,
and a license flag on every recommended tool with a re-verify pointer.

## Contents
- Small-data-first (the flagship call)
- Warehouse cost is a workload-shape problem
- Table formats and the catalog interop contract
- Orchestrator decision table (canonical home)
- The licensing map (canonical home)

## Procedure

### 1. Small-data-first — the flagship call
Before sizing a warehouse, measure the actual workload. Jordan Tigani (MotherDuck founder, ex-
BigQuery founding engineer), "Big Data is Dead" (motherduck.com/blog/big-data-is-dead, fetched
2026-07-25): the majority of BigQuery customers store **<1TB total**; heavy-user median storage is
"much less than 100GB"; **90% of queries scan <100MB**. Hannes Mühleisen (DuckDB co-founder), "The
Lost Decade of Small Data?" (duckdb.org/2025/05/19/the-lost-decade-of-small-data.html, 2026-07-25):
median Redshift/Snowflake scan is "a doable 100MB," 99.9th-percentile <300GB — and a 2012 16GB
MacBook ran full TPC-H against a 265GB database via DuckDB in under 34 min/query. *"Yes, we really
did lose a full decade"* chasing Hadoop/Spark/cloud-warehouse complexity for workloads a laptop
could serve. Modern single machines (64 cores/256GB RAM) are ~100x more capable than 2006
hardware — the gap that originally justified distributed systems has largely closed.

Frank McSherry's **COST rule** ("Scalability! But at what COST?", 2015, frankmcsherry.org)
generalizes this into a standing test: **a distributed system must beat the "Configuration that
Outperforms a Single Thread" before its complexity is justified** — across published
Spark/Giraph/GraphLab benchmarks on 128 cores, none consistently outperformed a single laptop
thread running a better algorithm. Apply it before adopting Spark/Kafka/a multi-node warehouse for
a workload that hasn't measured its own size.

**The Postgres/DuckDB-as-warehouse seam** is where this consensus is landing in practice, not just
polemic: Postgres 18's async I/O makes sequential scans materially faster; `pg_duckdb` embeds
DuckDB directly inside a Postgres process so DuckDB's vectorized OLAP engine runs against Postgres
data with zero export step; `pg_lake` lets Postgres own the Iceberg catalog while delegating
analytical queries to DuckDB, giving transactional lakehouse updates plus fast analytical reads
from a stack the team already runs. **Default to single-node (DuckDB/Postgres) until a measured
bottleneck says otherwise** — a family invariant, not a one-off preference; caveat that MotherDuck
and DuckDB's own blogs are self-interested (single-node vendors) even though Tigani's BigQuery-side
numbers are independently plausible.

### 2. Warehouse cost is a workload-shape problem
$ anchors, date-stamped 2026-07-25, re-verify before quoting (pricing pages change without
notice):
- **BigQuery**: on-demand analysis **$6.25/TiB scanned** (first 1 TiB/month free, 10MiB/query
  minimum metering); capacity slots **$0.04/slot-hour (Standard) / $0.06 (Enterprise) / $0.10
  (Enterprise Plus)**, discounted ~25-40% on 1yr/3yr commitments, 50-slot reservation minimum.
  On-demand fits sporadic workloads; slots fit predictable sustained volume — the choice is
  variable-vs-fixed cost, not a maturity ladder. (cloud.google.com/bigquery/pricing)
- **Snowflake**: 8 warehouse sizes (X-Small→6X-Large), each doubling compute + credit cost —
  **sizing is exponential, not linear** (2X-Large is 32x X-Small, not 2x). Billed per-second,
  60-second minimum. Auto-suspend/auto-resume is on by default and is the single biggest
  idle-cost lever; multi-cluster warehouses trade cost for concurrency, not efficiency.
  (docs.snowflake.com/en/user-guide/warehouses-overview)

The practitioner framing that ties this together: *"Your Snowflake bill is a design decision you
made months ago against sample data. Partitioning strategy, clustering keys, warehouse sizing,
materialization choices — teams make all of these while developing against a few million rows,
then find out what they actually cost when production volume arrives"* (@Pradeep891730,
x.com/Pradeep891730/status/2080688457760776277). Treat right-sizing, auto-suspend, and
partitioning/clustering choices as design decisions made *now*, against production-scale data, not
inherited defaults from a dev-scale prototype.

### 3. Table formats and the catalog interop contract
Both **Apache Iceberg** and **Delta Lake** are Apache-2.0/open-governance, both give ACID via
atomic metadata swaps + time travel + schema evolution over Parquet. Iceberg: hidden partitioning
(partition values derived via transforms — queries filter on logical columns, Iceberg derives
predicates), free partition-spec evolution, broadest non-JVM native implementations
(Go/Rust/C++/Python), stable per-field integer IDs make renames free. Delta: one unifying
transaction-log mental model (numbered JSON commits + periodic Parquet checkpoints), deepest
Spark-native tooling; **gotcha** — ACID strength is bounded by the underlying object store's
atomic-write/consistent-list guarantees. Databricks' **UniForm** sidesteps the format choice
entirely by generating Iceberg- and Hudi-compatible metadata over the same underlying Parquet
files — since all three formats sit on Parquet, metadata is the only real difference
(databricks.com/blog/delta-uniform-universal-format-lakehouse-interoperability, 2026-07-25).

**The real interop contract is the Iceberg REST Catalog API**, not the file format. Unity Catalog
implements it (format-agnostic — works with Delta/Iceberg/Parquet without data movement); Apache
Polaris implements it (Doris/Flink/Spark/Dremio/StarRocks/Trino all connect through it). "Catalog
wars" in 2026 is a contest over governance/AI features layered on a shared REST contract, not
incompatible wire protocols — pick a catalog for its governance features, not because it locks out
a table format.

### 4. Orchestrator decision table (canonical home)
Pick by core unit, not by feature checklist (github-corpus §11, precise wording, all four
Apache-2.0):

| Tool | Core unit | Model | Best fit |
|---|---|---|---|
| **Airflow** | Task in a DAG | Imperative, mostly-static DAG structure | Stable, task-sequenced batch pipelines; huge operator ecosystem |
| **Dagster** | **Asset** (software-defined) | Declarative — declare the *data* you want, the graph is inferred | Data/asset-lineage-centric teams; testability dev→prod |
| **Prefect** | Flow/Task (Python functions) | Imperative Python, retries/branching as decorators | "Just Python," dynamic/ad-hoc workflows, fast script→prod |
| **Kestra** | Flow (YAML) | Declarative, language-agnostic, event+schedule unified | Polyglot orchestration; event-driven triggers; low-code UI+Git round-trip |

Dagster's own framing draws the line cleanly: designed for "developing and maintaining data
assets" vs. Airflow's "author workflows that orchestrate tasks." Airflow treats idempotency as an
opinion, not an enforced property — tasks should ideally be idempotent and pass only small XCom
metadata between them, delegating heavy data movement to external systems.

**Dated fact**: Prefect is acquiring Dagster — confirmed via Nick Schrock's (Dagster founder)
letter on the Dagster blog, dated **2026-07-13** (dagster.io/blog). Separate product identities
are retained; no customer action required. Enter this as a confirmed, dated fact, not speculation
— but re-verify closer to any build decision, since integration plans can shift.

### 5. The licensing map (canonical home)
**Never trust a repo's GitHub API `license` field for this stack — read the LICENSE file.**
Snowplow's meta-repo and the GitHub API both still say Apache-2.0 while the actual pipeline license
changed underneath. Three shapes, dated 2026-07-25:

**A. Open-core** (permissive core + a proprietary `ee/`/`enterprise/` directory): PostHog (MIT
core + PostHog Enterprise License `ee/`), GrowthBook (MIT + GrowthBook Enterprise License),
Langfuse (MIT + Langfuse Enterprise License `ee/`), Confluent Schema Registry (Apache clients +
Confluent Community License server), Redpanda Connect (Apache-2.0 majority + RCL enterprise — **no
root LICENSE file**, only `licenses/`, a scanner gotcha).

**B. Whole-product source-available** (one non-OSI license over the entire product, not just an
enterprise carve-out): Airbyte platform (**ELv2** — self-host/modify freely, no reselling as a
hosted service; connectors + CDK stay MIT), soda-core (ELv2), RudderStack server (ELv2), Arize
Phoenix (ELv2), dbt Fusion engine (ELv2, 5 named crates stay Apache-2.0; dbt Core itself stays
Apache-2.0), Estuary Flow (**BSL 1.1**, Additional Use Grant forbids reselling as a "Data
Processing Service," converts to Apache-2.0 on **2029-11-20**).

**C. Pipeline-itself tightened** (the headline trap): Snowplow's core pipeline (`snowplow/enrich`)
runs **SLULA v1.1** (Nov 2024) — not just an enterprise carve-out, the pipeline itself. The
meta-repo and GitHub API both still report Apache-2.0; only `iglu-central` (the schema repo) is
actually Apache-2.0. This is the clearest case for "read the LICENSE file, not the API field."

**Copyleft to flag distinctly** (network-use triggers source-disclosure — a materially different
obligation from anything above): Multiwoven (AGPLv3), Metabase (AGPL-3.0 + commercial), Grafana
(AGPL-3.0), Plausible (AGPL-3.0), Matomo (GPL-3.0).

**Cleanly permissive** (safe to depend on/recommend freely, as of this fetch): Apache-2.0 —
dbt-core, dbt-utils, SQLMesh, MetricFlow (v0.209+), Rill, Airflow, Dagster, Prefect, Kestra,
Iceberg, Delta, Debezium, dlt, Great Expectations, Elementary, Deequ, whylogs, Recce, OpenLineage,
Marquez, DataHub, OpenMetadata, Atlas, Feast, OpenLLMetry, Spotify Confidence, Superset, ODCS. MIT
— DuckDB, Polars, Malloy, Evidence, Meltano, Presidio, the datacontract spec + CLI, data-diff,
sqlfluff.

**The teaching example — MetricFlow's license arc**: `v0–0.140.0` = AGPL → `v0.150.0–0.208.2` =
BSL → `v0.209.0+` = **Apache-2.0** (per its own README, 2026-07-25). Current `main` is genuinely
Apache-2.0 — but the README still carries a legacy "source-available software" header. **A tool's
license is a point-in-time fact with a version boundary, not a fixed property of the project** —
check the version actually being pulled, not the project's reputation.

## Validation
- [ ] A single-node (DuckDB/Postgres) option was considered and rejected on measured workload
      evidence before reaching for a distributed warehouse/engine.
- [ ] Every quoted $/TiB, $/slot-hour, or credit figure carries the 2026-07-25 date stamp and a
      re-verify pointer to the live pricing page.
- [ ] Warehouse sizing accounts for exponential (not linear) cost scaling, and auto-suspend is on
      before go-live.
- [ ] A table format was chosen alongside its catalog's Iceberg-REST-API support, not the format
      alone.
- [ ] The orchestrator pick names its core unit (task/asset/flow) and why, not just a feature
      comparison.
- [ ] Every recommended tool's license was verified against its LICENSE file, not the GitHub API
      `license` field, with the shape (A/B/C or copyleft) named.

## Failure modes and handoff
- **The real ask is per-table sync strategy or whether CDC/streaming is justified for one
  pipeline** — → `ingestion-and-movement.md`.
- **The real ask is the metric-type/semantic-layer decision** — → `metrics-semantics-and-bi.md`.
- **The real ask is production deployment/monitoring of the chosen platform** — → `operate`.
- **A license fact is being copied from memory or a stale doc** — re-verify the LICENSE file at
  build time; a "verified" claim frozen into a doc is exactly the anti-pattern this reference
  teaches against.
- **The Prefect/Dagster acquisition or any vendor pricing figure is more than a few weeks old** —
  re-verify before it gates a build decision.
