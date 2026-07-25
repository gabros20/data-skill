# Surface: Warehouse platform

Purpose: Reshape the 10 primary-job refs for a central analytical platform serving many teams and
source systems — ELT landing, staging→marts layering at scale, one semantic layer, BI as the
consuming edge, and org-wide trust gates. The team-scale ownership question (mesh vs. central) is
named but not re-derived from scratch; cost governance is a first-class axis here in a way it isn't
at product-analytics scale.

Read when:
- The request is a warehouse/lakehouse serving multiple teams or source systems, not one product's
  event stream — "central warehouse," "data platform," "dbt project for the whole company," "BI
  layer," "who owns this table."
- The presenting problem is metric drift across tools, or "which team's numbers are right."

Skip when:
- The request is one product's own usage analytics — →
  [surface-product-analytics.md](surface-product-analytics.md) (the default; most instrumentation
  asks belong there, not here).
- The concern is data flowing back into an operational system, not analytical consumption — →
  [surface-operational-activation.md](surface-operational-activation.md).
- The platform's real content is an AI product's corpus/traces — →
  [surface-ai-data-plane.md](surface-ai-data-plane.md); compose the two when the AI product's data
  plane also feeds the central warehouse.
- GDPR/CCPA/HIPAA/SOC2 applies — stack the additive [surface-regulated.md](surface-regulated.md) on
  top, it doesn't replace this surface's shape.

Inputs:
- How many teams/source systems land data here, and whether ownership is centralized or
  domain-distributed (decides the mesh-vs-central posture, §3).
- The warehouse/lakehouse engine already chosen or under evaluation, and its billing model.
- Which tools currently compute the metric in question, if the presenting problem is metric drift.

Produces:
- An ELT pipeline shape: landing → staging (1:1, no joins) → marts (wide, joins finalized).
- One semantic-layer definition per metric, consumed by every downstream tool.
- A trust-gate design: where each class of check runs, and what blocks a merge/promotion.
- A cost-governance posture matched to the engine's actual billing model.

## Contents
- Job impact table
- ELT + the staging/marts DAG
- Mesh vs. central, with the federated-governance caveat
- Cost governance is a first-class axis here

## Procedure

### 1. Job impact table
| Job | How this surface bends it |
|---|---|
| [ingestion-and-movement.md](ingestion-and-movement.md) | ELT is the default shape — decouple extraction/loading from transformation now that cloud storage/compute is cheap; pick a sync strategy (full refresh/incremental/append/CDC) per source **by table shape**, not one strategy for the whole platform. |
| [transformation-and-modeling.md](transformation-and-modeling.md) | **The spine at this surface.** Staging strictly 1:1 with a source, no joins, only staging touches `source()`; marts are wide and joins-finalized. This is where dbt's structural invariants (no source fan-out, no staging→staging, fold pass-through models into their child) actually pay for themselves — a platform serving many teams accumulates DAG debt fast without them. |
| [data-contracts-and-schema-evolution.md](data-contracts-and-schema-evolution.md) | Producer↔consumer contracts matter most here because producers (source teams) and consumers (analysts, other teams' marts) are organizationally separate — an ODCS contract per source table, with an explicit compatibility mode, is the mechanism; write-audit-publish + quality checks is the durable practice underneath the "data contracts" label. |
| [metrics-semantics-and-bi.md](metrics-semantics-and-bi.md) | **The trust payoff.** One semantic-layer definition per metric, consumed by every BI tool/agent — this is the surface where "two tools compute revenue differently" is the actual failure mode being designed against, not a hypothetical. Default posture leans code-first/versioned (dashboards-as-code) over GUI-only once more than one team edits definitions. |
| [quality-and-testing.md](quality-and-testing.md) | This is the flagship job at this surface's scale: known-unknowns (tests) at every layer, unknown-unknowns (anomaly detection) needing a metrics-history baseline, and a PR-time blast-radius diff before a model change merges — a schema change here can silently break another team's mart. |
| [lineage-and-observability.md](lineage-and-observability.md) | Cross-team lineage is load-bearing, not decorative — before changing a shared model, enumerate everything downstream (the blast-radius workflow) and confirm no other team's mart or dashboard breaks silently. |
| [governance-pii-and-retention.md](governance-pii-and-retention.md) | Access and PII-masking policy has to be centrally enforced at the platform layer (warehouse tags/policy filters), because consumers span teams that shouldn't each re-derive their own masking rule. |
| [platform-and-cost.md](platform-and-cost.md) | **Cost governance is a first-class axis here** (§4) — a platform serving many teams accumulates workload-shape drift fast; right-sizing and auto-suspend are the highest-leverage levers before any architecture change. |
| [experiment-measurement-foundations.md](experiment-measurement-foundations.md) | Same measurement-validity floor as any surface (SRM before trust) — this surface's addition is that the warehouse is usually where the experiment-assignment and outcome tables actually join, so the join itself needs the same staging discipline as any other mart. |

### 2. ELT + the staging/marts DAG
ELT — land first, transform in-warehouse with SQL — is the modern default: it decouples extraction
from transformation so landing isn't blocked on transform logic, and lets analysts who own SQL own
the transformation layer directly. The DAG discipline that keeps a multi-team platform from rotting:
staging models are atomic and 1:1 with a raw source (a `source()` reference has exactly one direct
child — no "source fan-out"); only staging models reference sources directly; no staging→staging
dependencies; fold pass-through intermediate models into their child as a CTE rather than leaving
"rejoining of upstream concepts" scattered across the DAG; marts are wide and denormalized (cheap
storage, expensive compute favors fewer joins at read time over a normalized star schema, though
`fct_`/`dim_` naming still applies where a genuine star schema is the right shape). Materialization
is a ladder (view → table → incremental) climbed on a measured bottleneck, not a default choice made
per model up front.

### 3. Mesh vs. central, with the federated-governance caveat
"Data mesh" as a full organizational program is contested — senior infra voices read it as a
rationalization for data silos as often as a real solution. What survives the backlash from the
founding text itself (Dehghani, 2019) is narrower and durable: **domain-oriented ownership** (the
team closest to a dataset hosts and serves it) plus **federated computational governance**
(centralized standards, decentralized execution) — not the full four-principle program taken as
gospel. When a request names "data mesh," name the durable primitive it's actually asking for
(domain ownership with federated governance) rather than standing up mesh tooling by name. A single
central team owning everything is the simpler default until the number of domains/teams genuinely
exceeds what one team's context can hold.

### 4. Cost governance is a first-class axis here
A warehouse bill is a design decision made against the workload's actual shape, not a fixed cost of
doing business. The two highest-leverage levers before touching architecture: auto-suspend (the
single biggest idle-cost lever on a per-second-billed warehouse) and right-sizing (credit cost scales
exponentially by warehouse tier on platforms like Snowflake — the wrong tier for a workload is pure
waste). Multi-cluster/concurrency scaling trades cost for concurrency, not for efficiency — don't
reach for it to fix a slow query. See [platform-and-cost.md](platform-and-cost.md) for the full
decision framework and verified dollar anchors; this surface's addition is that cost drift compounds
fastest where many teams share one billing account and no one owns the anomaly.

## Validation
- Staging is 1:1 with sources, no fan-out, no staging→staging edges; marts are wide with joins
  finalized.
- Every cross-team metric has exactly one semantic-layer definition; no tool computes it
  independently.
- A PR-time blast-radius diff runs before a shared model change merges.
- PII masking is enforced at the platform layer, not left to each consuming team.
- If "data mesh" was invoked, the durable primitive (domain ownership + federated governance) is
  named explicitly rather than the full program assumed.
- Cost levers (auto-suspend, right-sizing) are checked before any architectural cost fix.

## Failure modes and handoff
- **A full data-mesh program stood up by name** instead of the narrower domain-ownership +
  federated-governance primitive that actually survives scrutiny (§3).
- **A shared model changed without a blast-radius check** — another team's mart or dashboard breaks
  silently; this is exactly what lineage-and-observability's blast-radius workflow exists to catch.
- **Two BI tools computing the same metric differently** — the semantic layer is missing or being
  bypassed; route back to [metrics-semantics-and-bi.md](metrics-semantics-and-bi.md), don't patch
  each dashboard individually.
- **A distributed-scale architecture chosen to fix a cost problem** auto-suspend/right-sizing would
  have fixed — see [platform-and-cost.md](platform-and-cost.md).
- **The real ask is one product's own instrumentation**, not a multi-team platform — →
  [surface-product-analytics.md](surface-product-analytics.md).
