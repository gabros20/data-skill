# Warehouse Cost Checklist (fillable)

Warehouse cost is a **workload-shape problem before a pricing-model problem** — most orgs pay
distributed-scale prices for single-node-scale queries (COST: "a distributed system must beat one
laptop thread before its complexity is justified"; ~90% of real analytical queries scan <100MB).
Ask every question below **before** committing to an architecture, not after the bill arrives.
Source: [platform-and-cost.md](../references/platform-and-cost.md).

## 1. Workload-shape questions (ask first)

| Question | Answer |
|---|---|
| What's the actual median bytes-scanned per query, measured — not guessed? | |
| Is query volume predictable (steady) or spiky? | |
| Does any workload have a genuine sub-minute-latency or event-driven requirement — or does it just "feel real-time"? | |
| Would a single node (Postgres/DuckDB) handle this today, measured? | |
| Is the current design decision based on production data volume, or on a small sample/dev dataset? | |

**Your Snowflake/BigQuery bill is a design decision made against sample data if the answer to the
last question is "sample."** Re-run the sizing exercise against real production volume before
locking in an architecture.

## 2. Auto-suspend / right-sizing (the highest-leverage levers before touching architecture)

| Lever | Check | Status |
|---|---|---|
| Auto-suspend enabled | Idle compute is the single biggest avoidable cost — confirm it's ON, not assumed | |
| Right-sizing understood as exponential, not linear | e.g. Snowflake 2X-Large ≠ 2× X-Large — it's 32×; don't size up "to be safe" without doing this math | |
| Multi-cluster / concurrency scaling reserved for concurrency problems only | it trades cost for concurrency, not efficiency — don't reach for it to fix a slow single query | |
| On-demand vs reserved/slots decision made deliberately | on-demand = variable cost for sporadic workloads; committed capacity = predictable baseline for sustained heavy volume | |

## 3. Verified $ anchors (date-stamp before quoting — these move)

| Anchor | Value | As-of | Re-verify at |
|---|---|---|---|
| BigQuery on-demand analysis | $6.25/TiB scanned; first 1 TiB/month free | 2026-07-25 | cloud.google.com/bigquery/pricing |
| BigQuery capacity (slot-hour, pay-as-you-go) | Standard $0.04 · Enterprise $0.06 · Enterprise Plus $0.10 (1yr/3yr commit discounts further) | 2026-07-25 | cloud.google.com/bigquery/pricing |
| Snowflake warehouse sizing | 8 sizes X-Small→6X-Large, each doubling compute/credit-burn; billed per-second, 60s minimum | 2026-07-25 | docs.snowflake.com/en/user-guide/warehouses-overview |

Never copy the numbers above into a customer-facing estimate without re-fetching the source —
pricing pages change without notice.

## 4. Table-format / catalog decision (if choosing Iceberg/Delta/Hudi)

- Storage format barely matters — Parquet sits underneath all three.
- What matters is the **catalog/REST-API layer** — evaluate by Iceberg-REST-catalog-spec
  compliance, not by brand loyalty.

## 5. Orchestrator fit (task vs asset vs flow — pick the paradigm, not just the vendor)

| Paradigm | Core unit | Best fit |
|---|---|---|
| Task-in-a-DAG (e.g. Airflow) | Task | Stable, task-sequenced batch pipelines |
| Asset (declarative, e.g. Dagster) | Software-defined asset | Data/asset-lineage-centric teams needing dev→prod testability |
| Flow (imperative Python, e.g. Prefect) | Flow/Task as functions | Dynamic/ad-hoc workflows, fast script-to-prod |
| Flow (declarative YAML, e.g. Kestra) | Flow | Polyglot infra/data orchestration, event+schedule unified |

**Self-audit: ____ / 5 sections above completed with measured (not assumed) answers.** Anything
answered from a sample dataset or a guess must be re-measured against real production volume
before it drives an architecture decision.
