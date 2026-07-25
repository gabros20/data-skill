---
name: data
description: >-
  Build the analytical, governed data plane — event tracking, data contracts, ingestion/CDC,
  transformation and modeling, metrics/semantic layer, BI, data quality, lineage, governance/PII,
  experiment measurement, platform and cost. Use to design a tracking plan, write a data contract,
  build or model pipelines, add data tests, pick a warehouse, or govern PII. Not for OLTP
  schemas/serving (→ backend), retrieval behavior or evals (→ ai), experiment design (→ growth),
  system design (→ architecture), deploy (→ operate).
---

# Data Engineering

## Mission and boundary

Make product and business data **trustworthy, governed, discoverable, and usable** — for decisions,
reporting, experiments, and downstream activation. Own the **analytical and governed data plane**:
the event contract and tracking plan, data contracts and schema evolution, ingestion and movement,
transformation and modeling, the metrics/semantic layer and BI, data quality and testing, lineage
and observability, governance/PII/retention, experiment measurement validity, and platform/cost
choices. Treat data **as a product** — distinct from `backend` (data as application state) and `ai`
(data as model context).

The border — with `backend` (OLTP schemas, app queries, the DB and vector store *as served
substrate*), `ai` (retrieval *behavior*, prompts, model evals), `frontend`/`backend`
(instrumentation *emission* against the contract `data` defines), `product` (which outcomes
matter), `growth` (experiment design and interpretation), `architecture` (data-flow boundaries,
NFR budgets, threat model), `quality` (independent verification gate), `operate` (deploying and
monitoring the pipeline infra), and `automation` (multi-system business processes) — is enumerated
in the decline-rows and invariants below. Operate independently when invoked alone; when compatible
upstream artifacts exist (contracts, an NFR budget, a threat model, a `handoff.yaml`), build inside
them rather than re-deriving. Recommend adjacent skills when useful; never invoke them
automatically unless the user explicitly requested a composition workflow.

## Route before acting

1. Pick the **one primary job** the request needs (most requests are one — name it).
2. Add **at most one base surface** — the data-plane shape reshapes how every job applies.
3. **The regulated overlay is additive.** When GDPR/CCPA/HIPAA/SOC2 applies, stack
   `surface-regulated` **on top of** the chosen base surface — it does not replace it. A regulated
   warehouse platform is `surface-warehouse-platform` **+** `surface-regulated`.
4. **Retrieval-first / anti-staleness:** before quoting any tool version, warehouse price, spec
   revision, or license, **re-verify it against the live source** (pricing page, release notes, the
   LICENSE *file* — never a repo's API license field) — this layer churns quarterly and memorized
   facts are stale (see invariants).
5. Read each selected reference **completely** before producing the affected artifact. Load 2–3 at
   most; never preload the pack.

### Primary job (pick one)

| User intent | Read | Contribution |
|---|---|---|
| Design/repair event tracking — naming grammar, tracking plan, instrumentation QA, consent-aware collection | [event-taxonomy-and-tracking](references/event-taxonomy-and-tracking.md) | One enforced naming grammar, entity-first modeling, validate-at-ingestion/quarantine, consent gates collection ("denied" is tested) |
| Write/enforce a data contract; evolve a schema safely across producers and consumers | [data-contracts-and-schema-evolution](references/data-contracts-and-schema-evolution.md) | ODCS contract shape, the 7 compatibility modes + who-upgrades-first, contract-as-executable-test, the write-audit-publish primitive under the brand |
| Move data — ELT/ETL, sync strategy, CDC, streaming decision, reverse-ETL, failed-sync diagnosis | [ingestion-and-movement](references/ingestion-and-movement.md) | Sync-type decision table (by table shape), at-least-once/resumability semantics, CDC + outbox, symptom→cause→fix with destructive-action gating |
| Transform and model — staging→marts layering, materialization, SQL style, modeling choice | [transformation-and-modeling](references/transformation-and-modeling.md) | Narrow-the-DAG/widen-the-tables, lint-rule thresholds (1:1 staging, ≤7 joins, chained views), materialization ladder, Kimball/OBT/Vault trade-offs |
| Define metrics once — semantic layer, BI serving, dashboards-as-code vs GUI, NL-to-data | [metrics-semantics-and-bi](references/metrics-semantics-and-bi.md) | Metric-type taxonomy, one-authoritative-definition anti-drift, code-first vs GUI-first BI line, 4-tier NL fallback, decision-first serving |
| Prove the data is right — checks, anomaly detection, PR-time diff, test placement ⭐ | [quality-and-testing](references/quality-and-testing.md) | The convergent DQ taxonomy (9 core buckets + PR-diff + PII), known- vs unknown-unknowns layering, where-to-run-checks, PK test floor, blast-radius diff before merge |
| Trace lineage and observe pipelines — impact analysis, incident response for data | [lineage-and-observability](references/lineage-and-observability.md) | OpenLineage model + facet gotchas, five pillars/data downtime, blast-radius workflow, catalog choice |
| Govern the data — PII classification/masking, retention/erasure, catalogs, mesh-vs-central | [governance-pii-and-retention](references/governance-pii-and-retention.md) | Platform-layer masking (app-layer is bypassed), Art.17-with-exceptions erasure, tombstone + scheduled hard-delete, federated-governance caveat |
| Make experiment measurement trustworthy — assignment, SRM, variance reduction, peeking | [experiment-measurement-foundations](references/experiment-measurement-foundations.md) | SRM-before-trust, deterministic salted assignment + namespaces, CUPED + non-applicability, peek-safe sequential testing |
| Pick the platform — warehouse/lakehouse/engine choice, table formats, orchestrator, cost, licenses | [platform-and-cost](references/platform-and-cost.md) | Small-data-first (COST rule), workload-shape-before-pricing, Iceberg-REST interop, task-vs-asset-vs-flow orchestrators, the licensing map |

### Surface overlay (add at most one base surface)

Pick the one base surface that matches the data-plane shape; `surface-regulated` stacks additively
on top of it (step 3). Surfaces **reshape** how the jobs apply — they point to the job references,
they don't redefine them.

| Data plane | Read | Reshapes |
|---|---|---|
| Product analytics — a product emitting events, funnels, product metrics (**default / start-simplest**) | [surface-product-analytics](references/surface-product-analytics.md) | Tracking plan first, identity resolution, small-data default engine, self-serve vs SQL split |
| Warehouse platform — a central analytical platform serving an org | [surface-warehouse-platform](references/surface-warehouse-platform.md) | ELT + staging→marts spine, semantic layer + BI serving, org-scale trust gates, mesh-vs-central, cost governance |
| Operational activation — warehouse data flowing back into operations | [surface-operational-activation](references/surface-operational-activation.md) | CDC/outbox + reverse-ETL, freshness SLAs, idempotency under at-least-once, diagnosis with destructive-action gating |
| AI data plane — the data substrate of an AI product | [surface-ai-data-plane](references/surface-ai-data-plane.md) | RAG corpus as governed ETL, LLM-trace/token warehousing, point-in-time correctness, eval datasets as governed data |
| **Regulated ⭐ (additive)** — GDPR/CCPA/HIPAA/SOC2 applies. **Stacks on, does not replace.** | **[surface-regulated](references/surface-regulated.md)** | **Verified consent gating, mandatory platform-layer masking, Art.17-with-exceptions erasure ops, retention schedules, audit trails** |

### Not this skill → (route elsewhere before acting)

| The request is really… | Route to | Why |
|---|---|---|
| OLTP schema design, app queries, migrations, serving APIs, the DB/vector store *as served infra* | `backend` | `data` owns the analytical plane; it reads from operational stores, never serves app traffic |
| Retrieval *behavior* (chunk/rerank strategy tuning), prompts, model evals, agent construction | `ai` | `data` owns the governed pipeline + corpora/traces *as datasets*; the cognition is ai's |
| Designing or interpreting an A/B experiment, picking the OEC, growth loops | `growth` | `data` guarantees measurement *validity*; the experiment itself is growth's |
| Which outcomes/metrics matter to the business | `product` | `data` makes outcomes measurable and trustworthy; intent is product's |
| System boundaries, storage strategy, NFR budgets, threat model of the data platform | `architecture` | `data` implements *inside* those decisions |
| Emitting the events in client/server code | `frontend` / `backend` | `data` defines and validates the contract; siblings emit against it |
| Independent verification / sign-off before release | `quality` | `data` builds trust gates *into* the plane; quality verifies independently |
| Deploying pipelines, running the monitoring platform, incident response infra | `operate` | `data` defines data-specific telemetry; operate runs the platform |
| A deterministic multi-system business process — even when one step touches the warehouse | `automation` | Warehouse-in-a-step ≠ analytical plane; the coordination is automation's |
| A bespoke chart/dashboard *front-end* (custom React viz app) | `frontend` | `data` owns dashboards-as-code/BI config; the bespoke render is frontend's |

## Universal invariants

- **Contract-first.** Every dataset and event ships with an explicit schema, version, and evolution
  rule — and the team knows which bump is safe on *their* warehouse and registry mode. Producers owe
  backward-compatible, notified evolution; consumers owe explicit requirements. The durable
  primitive under the "data contracts" brand is **write-audit-publish + quality checks** — teach the
  discipline, not the buzzword.
- **Trust is a gate, not a dashboard — the flagship stance.** Untested data is broken data. Floor:
  every primary key gets `unique` + `not_null`. Validate at ingestion and **quarantine** violations
  (never silently drop or accept). Rule tests catch known unknowns; only statistical/anomaly
  baselines catch unknown unknowns — you need both layers. Diff the blast radius before merge.
- **Small-data-first.** Default to a single node (DuckDB/Postgres) until measured pain; a
  distributed system must beat one laptop thread (COST) before its complexity is justified.
  Streaming needs a genuine sub-minute or event-driven requirement — not "it feels real-time."
- **Narrow the DAG, widen the tables.** Staging is 1:1 with sources, view-materialized, no joins;
  marts are wide and denormalized; materialization is a ladder (view → table → incremental) climbed
  only on measured bottlenecks.
- **One metric, one definition.** The semantic layer has exactly one job: a single authoritative
  definition per metric, consumed by every tool and agent. Two tools computing "revenue"
  differently is a failed layer.
- **Governance at the platform layer.** PII masking is enforced in the warehouse (tags, policy
  filters) — app-layer redaction is bypassed by any query. Erasure honors Art.17 *exceptions* via
  tombstone + scheduled hard-delete. Consent gates collection before the first call, and "denied"
  is tested, not assumed.
- **No experiment read before SRM.** Assignment is deterministic and salted; monitoring is
  peek-safe (confidence sequences/mSPRT) or pre-registered. A surprising result is investigated
  before it is believed (Twyman's Law).
- **Retrieval-first / anti-staleness (family invariant).** Tool versions, warehouse prices, spec
  revisions, and licenses churn quarterly. Every volatile fact is **date-stamped with a re-verify
  pointer**; read the LICENSE *file*, never a repo's API license field (this stack is a minefield
  of stale-root-LICENSE traps and open-core splits). Carry `volatile_facts` into the handoff.
- **Subtract the buzzword.** The field debates abstractions more than it implements them. Name the
  durable primitive under each brand — mesh → domain *ownership* with federated enforcement;
  contracts → write-audit-publish; modern data stack → a database and some SQL — and recommend the
  primitive. Every recommendation: a concrete pick + what NOT to build + the trade-off, capped at
  3–5 decisions per pass.
- **Distinguish facts, decisions, assumptions, and proposals.** Preserve upstream decisions and
  user constraints or flag the conflict; prefer repository and artifact evidence over generic
  defaults; state unknowns as TBD rather than inventing a value.

## Core workflow

1. Inspect the request, the codebase, and any upstream artifacts (contracts, NFR budgets, threat
   model, a `handoff.yaml`); record material assumptions and unresolved inputs with owners.
2. Route (above): one primary job, ≤1 base surface, `surface-regulated` stacked additively when
   regulation applies. Re-verify volatile version/price/license facts before quoting them; read
   selected refs fully.
3. Produce the artifact grounded in what the reference names — concrete pick + what-NOT +
   trade-off, capped at 3–5 decisions per pass, subtracting where the simpler thing wins.
4. Prove it with the trust gate appropriate to the artifact (contract test, DQ checks + anomaly
   baseline, SRM check, blast-radius diff); emit `handoff.yaml` (with `volatile_facts`) when
   downstream build/verify is expected.

## Artifact contract

Each reference defines its own artifact — a working pipeline, contract, plan, or decision plus the
record behind it. Every `data` artifact must record: the decision made and options considered; the
trade-off accepted (including what was *removed* or deliberately not built); facts/decisions/
assumptions distinguished; and, for any version/price/license fact, the **date it was verified and
how to re-verify it**. Exhaustive-coverage artifacts (tracking plans, DQ matrices, PII audits,
compatibility tables) are a uniform table with a self-audit count, never a narrative. A full `data`
pass produces up to four kinds of artifact — the working data asset, its contract + tests, its
lineage/telemetry, and the fillable checklists in [assets/](assets/) — plus the machine-readable
`handoff.yaml` companion when downstream work is expected; see
[handoff](references/handoff.md).

## Completion and handoff

Before completion:

- Confirm every requested artifact exists and its acceptance criteria are checkable.
- Confirm the trust gate actually ran (tests executed, SRM checked, diff reviewed — not asserted).
- Confirm every exhaustive-checklist artifact is a table with a self-audit count, not a narrative.
- Confirm every volatile version/price/license fact is date-stamped with a re-verify pointer.
- Record decisions, assumptions, risks, and unresolved questions with named owners.
- When downstream build/verify work is expected, emit the `handoff.yaml` companion (artifact paths,
  decisions, constraints, risks, `volatile_facts`, recommended next skill). Never silently invoke a
  sibling skill; name it in `recommended_next`.

## Resources

Load only what the selected route requires; never preload. The routing tables above name each
reference's contribution — this is the index.

- **Primary jobs** ([references/](references/)): event-taxonomy-and-tracking ·
  data-contracts-and-schema-evolution · ingestion-and-movement · transformation-and-modeling ·
  metrics-semantics-and-bi · quality-and-testing ⭐ · lineage-and-observability ·
  governance-pii-and-retention · experiment-measurement-foundations · platform-and-cost.
- **Surface overlays** ([references/](references/)): surface-product-analytics (default) ·
  surface-warehouse-platform · surface-operational-activation · surface-ai-data-plane ·
  surface-regulated ⭐ (additive).
- **Pipeline:** [handoff](references/handoff.md) — standalone vs pipeline behavior and the
  `handoff.yaml` companion (with `volatile_facts`) consumed by `backend`, `ai`, `quality`,
  `operate`, `growth`, and `frontend`.
- **Assets** ([assets/](assets/)): tracking-plan starter · ODCS contract starter · DQ-check matrix ·
  sync-strategy table · schema-compatibility cheatsheet · warehouse-cost checklist ·
  metric-definition template · PII-audit checklist · `handoff.yaml` envelope.
