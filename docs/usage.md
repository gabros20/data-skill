# Usage reference

## Activation

`data` should trigger on requests to build **the analytical and governed data plane** of a product —
event tracking and tracking plans, data contracts and schema evolution, ingestion/CDC and movement,
transformation and modeling, the metrics/semantic layer and BI, data quality and testing, lineage and
observability, governance/PII/retention, experiment measurement validity, and platform/cost choices.
Realistic triggers: "design a tracking plan," "write/enforce a data contract," "build or fix an ELT
pipeline," "model staging into marts," "define a metric once for the whole org," "add data quality
tests / anomaly detection," "trace lineage / diagnose a broken pipeline," "classify and mask PII,"
"check for SRM before trusting an experiment," "pick a warehouse / orchestrator / table format."

It should **not** trigger on: "design OLTP schemas, app queries, or serving APIs" (`backend`), "tune
retrieval/chunking/reranking, prompts, or model evals" (`ai`), "design or interpret an A/B experiment,
pick the OEC" (`growth`), "decide which outcomes/metrics matter to the business" (`product`), "decide
data-flow boundaries, NFR budgets, or the threat model" (`architecture`), "emit the events in
client/server code" (`frontend`/`backend`), "independently red-team / sign off before release"
(`quality`), "deploy pipelines / run the monitoring platform" (`operate`), or "orchestrate a
deterministic multi-system business process" (`automation`) — unless the request also requires the
analytical data plane.

Use `/data` as documentation shorthand only. Codex explicit invocation is `$data`; other clients use
an `@` mention, a skill tool, or natural-language activation.

## Mission and boundary

`data` owns the analytical and governed data plane — the event contract and tracking plan, data
contracts and schema evolution, ingestion and movement, transformation and modeling, the
metrics/semantic layer and BI, data quality and testing, lineage and observability,
governance/PII/retention, experiment measurement validity, and platform/cost choices. It treats data
**as a product**, distinct from `backend` (data as application state) and `ai` (data as model
context). It sits between `architecture` (which decides data-flow boundaries, NFR budgets, and the
threat model) and `quality`/`operate` (which independently verify and run the monitoring platform). It
*references* `backend`'s serving substrate (OLTP schemas, the DB/vector store as served infra),
`ai`'s retrieval behavior and evals, and `frontend`/`backend`'s instrumentation emission rather than
owning them. It consumes upstream artifacts (contracts, NFR budgets, threat model, `handoff.yaml`)
when they exist, works standalone when they don't, recommends siblings for work it does not own, and
never silently invokes them.

## Workflow

1. Pick **one primary job** and **at most one base surface**; stack `surface-regulated` additively
   on top when GDPR/CCPA/HIPAA/SOC2 applies.
2. **Retrieval-first / anti-staleness:** before quoting any warehouse price, tool version, spec
   revision, or license, re-verify it against the live source (pricing page, release notes, the
   LICENSE *file* — never a repo's API license field). Never freeze a volatile fact.
3. Read only the selected references (2–3 at most — never preload the pack).
4. Produce the artifact: a concrete pick + what NOT to build + what to remove + a trade-off, capped
   at 3–5 decisions per pass (verify-and-subtract).
5. Prove it with the trust gate appropriate to the artifact (contract test, DQ checks + anomaly
   baseline, SRM check, blast-radius diff) — never assert "it's right" from memory.
6. Emit a compact `handoff.yaml` (with `volatile_facts`) when downstream build/verify work is
   expected.

## Route by job

| User intent | Reference | Expected contribution |
|---|---|---|
| Design/repair event tracking — naming grammar, tracking plan, instrumentation QA, consent-aware collection | `references/event-taxonomy-and-tracking.md` | One enforced naming grammar, entity-first modeling, validate-at-ingestion/quarantine, consent gates collection |
| Write/enforce a data contract; evolve a schema safely | `references/data-contracts-and-schema-evolution.md` | ODCS contract shape, 7 compatibility modes + who-upgrades-first, contract-as-executable-test |
| Move data — ELT/ETL, sync strategy, CDC, streaming decision, reverse-ETL, failed-sync diagnosis | `references/ingestion-and-movement.md` | Sync-type decision table, at-least-once/resumability semantics, CDC + outbox, symptom→cause→fix |
| Transform and model — staging→marts layering, materialization, SQL style, modeling choice | `references/transformation-and-modeling.md` | Narrow-the-DAG/widen-the-tables, lint-rule thresholds, materialization ladder |
| Define metrics once — semantic layer, BI serving, dashboards-as-code vs GUI, NL-to-data | `references/metrics-semantics-and-bi.md` | Metric-type taxonomy, one-authoritative-definition, code-first vs GUI-first BI line |
| Prove the data is right — checks, anomaly detection, PR-time diff, test placement | `references/quality-and-testing.md` | Convergent DQ taxonomy (9 buckets + PR-diff + PII), known/unknown-unknowns layering, PK test floor |
| Trace lineage and observe pipelines — impact analysis, incident response for data | `references/lineage-and-observability.md` | OpenLineage model + facet gotchas, five pillars/data downtime, blast-radius workflow |
| Govern the data — PII classification/masking, retention/erasure, catalogs, mesh-vs-central | `references/governance-pii-and-retention.md` | Platform-layer masking, Art.17-with-exceptions erasure, tombstone + scheduled hard-delete |
| Make experiment measurement trustworthy — assignment, SRM, variance reduction, peeking | `references/experiment-measurement-foundations.md` | SRM-before-trust, deterministic salted assignment, CUPED, peek-safe sequential testing |
| Pick the platform — warehouse/lakehouse/engine, table formats, orchestrator, cost, licenses | `references/platform-and-cost.md` | Small-data-first (COST rule), workload-shape-before-pricing, licensing map |

Surface overlays (`surface-product-analytics`, `surface-warehouse-platform`,
`surface-operational-activation`, `surface-ai-data-plane`, `surface-regulated`) reshape the primary
job for that data-plane shape. Always stack `surface-regulated` additively on top of the base
surface — it changes the compliance posture, not the base.

## Outputs and completion

| Output | Complete when |
|---|---|
| Tracking plan / event contract | Naming grammar enforced, entities modeled, ingestion validation + quarantine defined, consent gates tested |
| Data contract | ODCS shape used, compatibility mode named, write-audit-publish wired as an executable test |
| Ingestion/movement pipeline | Sync type matches table shape, at-least-once/resumability handled, failure mode diagnosed with gating |
| Transformation/model | Staging is 1:1 and view-materialized, marts are wide, materialization justified by a measured bottleneck |
| Metric/semantic-layer definition | Single authoritative definition, consumed the same way by every tool/agent |
| DQ/quality suite ⭐ | PK floor covered, rule tests + anomaly baseline both present, blast-radius diff reviewed before merge |
| Lineage/observability wiring | Impact analysis traceable, the five pillars of data downtime instrumented |
| Governance/PII artifact | Masking enforced at the platform layer, erasure honors Art.17 exceptions, retention schedule set |
| Experiment-measurement check | SRM checked before any read, assignment deterministic/salted, peek-safety confirmed |
| Platform/cost decision | Workload shape assessed before pricing, COST rule applied, license read from the LICENSE file |

When downstream work is expected, provide the `handoff.yaml` companion with artifact paths,
decisions, constraints, risks, `volatile_facts` (fact / as-of / re-verify-at), and the recommended
next skill — and, when a threat model was consumed, state the border between data-implemented trust
gates and quality-verified controls.
