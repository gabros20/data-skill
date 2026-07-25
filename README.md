# data — Data Engineering

**Visual guide:** [data-skill.vercel.app](https://data-skill.vercel.app)

Make **product and business data trustworthy, governed, and usable** — the event contract and
tracking plan, data contracts and schema evolution, ingestion and movement, transformation and
modeling, the metrics/semantic layer and BI, data quality and testing, lineage and observability,
governance/PII/retention, experiment measurement validity, and platform/cost choices, across any
warehouse or stack. `data` is the **analytical and governed data plane** of the
[Digital Product Skills](https://github.com/gabros20) family, a peer of `frontend`, `backend`, and
`ai` in the Build stage: an agent skill for treating data **as a product** — distinct from `backend`
(data as application state) and `ai` (data as model context).

It sits between `architecture` (which decides data-flow boundaries, NFR budgets, and the threat
model) and `quality`/`operate` (which independently verify and run the monitoring platform). It owns
the **analytical plane** — contracts, transformation, metrics, trust gates, governance — not the
serving substrate or the model cognition. It is a **warehouse-, tool-, and stack-agnostic router
grounded in the current platform landscape, not a single-vendor dbt/warehouse tutorial.**

## What it does

A **faceted router**: pick one primary job × at most one base data-plane surface (the regulated
overlay stacks additively on top).

**Primary jobs** — event taxonomy & tracking · data contracts & schema evolution · ingestion &
movement · transformation & modeling · metrics, semantics & BI · **quality & testing ⭐** · lineage &
observability · governance, PII & retention · experiment measurement foundations · platform & cost.

**Surface overlays** — product analytics (default) · warehouse platform · operational activation ·
AI data plane · **regulated** (additive). The base surface reshapes how every job applies for that
data-plane shape; `surface-regulated` stacks on top of whichever base surface applies rather than
replacing it.

**The differentiator:** the convergent data-quality taxonomy is the flagship — not an afterthought:
9 core DQ buckets plus PR-time blast-radius diff plus PII, known-vs-unknown-unknowns layering (rule
tests catch the former, statistical/anomaly baselines the latter), and a PK-test floor that's
non-negotiable. The through-line is **trust is a gate, not a dashboard** — untested data is broken
data, and every artifact names a concrete pick, what NOT to build, and the trade-off, capped at 3–5
decisions per pass. And a distinctive value-add: **small-data-first cost discipline** — the COST rule
(a distributed system must beat one laptop thread before its complexity is justified), narrow-the-DAG
staging, one-authoritative-definition metrics, and platform-layer governance (app-layer PII redaction
is bypassed by any query).

Running through all of it is the flagship invariant — **retrieval-first / anti-staleness**: warehouse
prices, tool versions, spec revisions, and licenses churn quarterly, so every volatile fact is
date-stamped and re-verified against the live source — reading the LICENSE *file*, never a repo's API
license field, because this stack is a minefield of stale-root-LICENSE traps and open-core splits.
The `handoff.yaml` even carries a `volatile_facts` block so a stale value can't leak downstream. Full
attribution in [SOURCES.md](skills/data/SOURCES.md).

## Install

```bash
npx skills add gabros20/data-skill -g -y
```

Or use it in Codex with `$data`.

## Examples

```text
$data design a tracking plan and event contract for our new checkout flow

$data add data quality tests and anomaly baselines to our dbt project

$data our experiment results feel off — check for SRM before we trust them

$data help me pick a warehouse and orchestrator for a startup on a budget

$data build the governed RAG-corpus pipeline for our AI product's retrieval index
```

## Route by job

| User intent | Reads | Contribution |
|---|---|---|
| Design/repair event tracking — naming grammar, tracking plan, instrumentation QA, consent-aware collection | [event-taxonomy-and-tracking](skills/data/references/event-taxonomy-and-tracking.md) | One enforced naming grammar, entity-first modeling, validate-at-ingestion/quarantine, consent gates collection ("denied" is tested) |
| Write/enforce a data contract; evolve a schema safely across producers and consumers | [data-contracts-and-schema-evolution](skills/data/references/data-contracts-and-schema-evolution.md) | ODCS contract shape, the 7 compatibility modes + who-upgrades-first, contract-as-executable-test, write-audit-publish under the brand |
| Move data — ELT/ETL, sync strategy, CDC, streaming decision, reverse-ETL, failed-sync diagnosis | [ingestion-and-movement](skills/data/references/ingestion-and-movement.md) | Sync-type decision table (by table shape), at-least-once/resumability semantics, CDC + outbox, symptom→cause→fix with destructive-action gating |
| Transform and model — staging→marts layering, materialization, SQL style, modeling choice | [transformation-and-modeling](skills/data/references/transformation-and-modeling.md) | Narrow-the-DAG/widen-the-tables, lint-rule thresholds, materialization ladder, Kimball/OBT/Vault trade-offs |
| Define metrics once — semantic layer, BI serving, dashboards-as-code vs GUI, NL-to-data | [metrics-semantics-and-bi](skills/data/references/metrics-semantics-and-bi.md) | Metric-type taxonomy, one-authoritative-definition anti-drift, code-first vs GUI-first BI line, 4-tier NL fallback |
| Prove the data is right — checks, anomaly detection, PR-time diff, test placement ⭐ | [quality-and-testing](skills/data/references/quality-and-testing.md) | Convergent DQ taxonomy (9 core buckets + PR-diff + PII), known- vs unknown-unknowns layering, PK test floor, blast-radius diff before merge |
| Trace lineage and observe pipelines — impact analysis, incident response for data | [lineage-and-observability](skills/data/references/lineage-and-observability.md) | OpenLineage model + facet gotchas, five pillars/data downtime, blast-radius workflow, catalog choice |
| Govern the data — PII classification/masking, retention/erasure, catalogs, mesh-vs-central | [governance-pii-and-retention](skills/data/references/governance-pii-and-retention.md) | Platform-layer masking (app-layer is bypassed), Art.17-with-exceptions erasure, tombstone + scheduled hard-delete |
| Make experiment measurement trustworthy — assignment, SRM, variance reduction, peeking | [experiment-measurement-foundations](skills/data/references/experiment-measurement-foundations.md) | SRM-before-trust, deterministic salted assignment + namespaces, CUPED + non-applicability, peek-safe sequential testing |
| Pick the platform — warehouse/lakehouse/engine choice, table formats, orchestrator, cost, licenses | [platform-and-cost](skills/data/references/platform-and-cost.md) | Small-data-first (COST rule), workload-shape-before-pricing, Iceberg-REST interop, task-vs-asset-vs-flow orchestrators |

**Surface overlay (add at most one base; the regulated overlay stacks additively):**

| Data plane | Reads | Reshapes |
|---|---|---|
| Product analytics — a product emitting events, funnels, product metrics (default / start-simplest) | [surface-product-analytics](skills/data/references/surface-product-analytics.md) | Tracking plan first, identity resolution, small-data default engine, self-serve vs SQL split |
| Warehouse platform — a central analytical platform serving an org | [surface-warehouse-platform](skills/data/references/surface-warehouse-platform.md) | ELT + staging→marts spine, semantic layer + BI serving, org-scale trust gates, mesh-vs-central |
| Operational activation — warehouse data flowing back into operations | [surface-operational-activation](skills/data/references/surface-operational-activation.md) | CDC/outbox + reverse-ETL, freshness SLAs, idempotency under at-least-once |
| AI data plane — the data substrate of an AI product | [surface-ai-data-plane](skills/data/references/surface-ai-data-plane.md) | RAG corpus as governed ETL, LLM-trace/token warehousing, point-in-time correctness, eval datasets as governed data |
| **Regulated ⭐ (additive)** — GDPR/CCPA/HIPAA/SOC2 applies, stacks on rather than replacing the base | [surface-regulated](skills/data/references/surface-regulated.md) | **Verified consent gating, mandatory platform-layer masking, Art.17-with-exceptions erasure ops, retention schedules, audit trails** |

Plus [handoff](skills/data/references/handoff.md) — the `handoff.yaml` companion (with
`volatile_facts`) to `backend`, `ai`, `quality`, `operate`, `growth`, and `frontend` when downstream
work is expected.

## Outputs

A working data asset plus the decision record behind it: a tracking plan or data contract, an
ingestion/transformation pipeline, a semantic-layer metric definition, a DQ test suite with anomaly
baselines, lineage/observability wiring, a PII/governance audit, an experiment-measurement validity
check — plus fillable checklists and a `handoff.yaml` companion (carrying `volatile_facts`) that
feeds `backend`/`ai`/`quality`/`operate`/`growth`/`frontend` downstream, grounded in what each
artifact names and date-stamped for re-verification, never a frozen template dump.

## Reference and asset library

16 references under `skills/data/references/` (10 primary jobs, 5 surface overlays, 1 handoff
contract) and 9 fillable assets under `skills/data/assets/`:

- **Assets:** `tracking-plan-starter.md` · `data-contract-starter.odcs.yaml` ·
  `dq-check-matrix.md` · `sync-strategy-table.md` · `schema-compatibility-cheatsheet.md` ·
  `warehouse-cost-checklist.md` · `metric-definition-template.md` · `pii-audit-checklist.md` ·
  `handoff.yaml`

## Digital Product lifecycle

`data` is the analytical and governed data plane, a peer of `frontend`, `backend`, and `ai` in the
**Build** stage: it consumes `architecture`'s data-flow boundaries, NFR budgets, threat model, and
`handoff.yaml` upstream, and hands its contracts, DQ gates, and lineage/telemetry to `quality` and
`operate` downstream. It *references* rather than owns the serving substrate (`backend`: OLTP
schemas, the DB/vector store as served infra), model cognition (`ai`: retrieval behavior, prompts,
evals), instrumentation *emission* (`frontend`/`backend`: client/server code emitting against the
contract `data` defines), experiment design (`growth`: the OEC and interpretation), and business
outcomes (`product`: which metrics matter). It works standalone with no upstream artifact required,
and never silently invokes a sibling skill.

## Repository layout

```text
skills/data/  runtime skill (SKILL.md + 16 references + 9 assets + agents/openai.yaml)
evals/        activation · traversal · output · compression-ablation fixtures
research/     4-channel research corpora + build-gate synthesis
docs/         installation and usage
.codex-plugin/  plugin manifest
```

## Documentation & releases

- [docs/installation.md](docs/installation.md) · [docs/usage.md](docs/usage.md) ·
  [docs/recipes.md](docs/recipes.md)
- [CHANGELOG.md](CHANGELOG.md) — release history (current: v0.1.0)
- [SOURCES.md](skills/data/SOURCES.md) — source attribution and license posture
- [CONTRIBUTING.md](CONTRIBUTING.md) — local validation (`scripts/lint-skill`, `scripts/check-sync`)

## License

MIT — see [LICENSE](LICENSE).
