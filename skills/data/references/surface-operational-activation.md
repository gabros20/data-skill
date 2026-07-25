# Surface: Operational activation

Purpose: Reshape the 10 primary-job refs for data flowing *back* into operational systems — CDC out
of an operational store, reverse-ETL into SaaS tools, near-real-time sync — where at-least-once
delivery, idempotent writes, and safe failure recovery matter more than analytical query shape.

Read when:
- The request syncs warehouse/analytical data into an operational tool (CRM, marketing, support,
  billing) — "reverse ETL," "sync segments into Salesforce," "activate this audience."
- The request captures changes out of an operational database for downstream use — "CDC," "outbox
  pattern," "stream table changes," "near-real-time sync."
- A sync is failing and the ask is diagnosis/recovery, not new pipeline design.

Skip when:
- The data only flows into analytical consumption (warehouse, BI, product metrics) — →
  [surface-product-analytics.md](surface-product-analytics.md) or
  [surface-warehouse-platform.md](surface-warehouse-platform.md); this surface is specifically about
  the *return* trip into operations.
- The concern is the operational database itself as served infrastructure (uptime, connection
  pooling, schema migration of the app's own tables) — that's `backend`'s; this surface owns the
  data-movement contract in and out of it, not the store.
- GDPR/CCPA/HIPAA/SOC2 applies — stack the additive [surface-regulated.md](surface-regulated.md) on
  top, it doesn't replace this surface's shape.

Inputs:
- Direction and shape of the sync: CDC out of an operational store, or reverse-ETL into a downstream
  SaaS tool — state which; the failure modes differ.
- The freshness requirement — a genuine sub-minute/event-driven need, or "near-real-time" used
  loosely (decides whether streaming infrastructure is actually justified).
- If diagnosing a failure: the error signature, and whether the fix under consideration is
  destructive (backfill, resync, delete-and-reload).

Produces:
- A sync-strategy pick (full refresh/incremental/append/CDC/webhook) keyed to the actual table
  shape, not one strategy applied uniformly.
- An idempotency/at-least-once design: what a retried or replayed write does to the destination.
- A failure-diagnosis path: root cause before action, destructive recovery gated behind explicit
  confirmation.

## Contents
- Job impact table
- Choosing a sync strategy by table shape
- CDC + the Outbox pattern
- Reverse-ETL: the accepted capability vs. the contested vendor category
- Failure diagnosis and destructive-action gating

## Procedure

### 1. Job impact table
| Job | How this surface bends it |
|---|---|
| [ingestion-and-movement.md](ingestion-and-movement.md) | **The spine.** Sync-strategy choice (§2), CDC mechanics (§3), and reverse-ETL (§4) are this surface's primary content — the other jobs below bend around a movement decision, not a modeling or serving one. |
| [data-contracts-and-schema-evolution.md](data-contracts-and-schema-evolution.md) | A CDC stream's schema is the *operational* table's schema — it evolves on the app team's timeline, not the data team's. Use a schema-contract mode (evolve/freeze/discard) rather than assuming stability; a silent app-side column change should not silently corrupt a downstream sync. |
| [quality-and-testing.md](quality-and-testing.md) | At-least-once delivery means duplicates are expected, not exceptional — dedup keys and idempotent upserts are the test surface, not just null/uniqueness checks on the landed table. |
| [lineage-and-observability.md](lineage-and-observability.md) | Freshness is the primary observability signal for this surface — track lag between an operational-store change and its downstream/reverse-synced effect, and treat a growing lag as an incident, not a metric to shrug at. |
| [governance-pii-and-retention.md](governance-pii-and-retention.md) | Reverse-ETL pushes data *out* to third-party SaaS tools — classify what's leaving the governed perimeter before the sync ships, not after; this is a stricter version of the platform's masking discipline because the destination isn't warehouse-internal. |
| [platform-and-cost.md](platform-and-cost.md) | Streaming infrastructure (Kafka/Debezium/Flink) is justified by a genuine sub-minute-latency or event-driven need — not because "near-real-time" sounds better than batch. Batch/micro-batch covers most operational-activation asks; verify the freshness requirement (inputs) before reaching for streaming. |

### 2. Choosing a sync strategy by table shape
Pick the sync strategy per table, keyed to its actual shape — not one strategy for the whole
pipeline: **full refresh** for small dimension tables where re-pulling everything is cheap;
**incremental** (cursor column, e.g. `updated_at`) for large tables updated in place; **append** for
append-only/log-shaped tables with no updates to reconcile; **CDC** (log-based) when
updates/deletes must be captured with ordering and low source overhead; **webhook** for
near-real-time push where the source system can notify on change. A newly created table can put a
CDC snapshot ahead of connection state — a fresh table needs a manual refresh before CDC starts
picking it up; don't assume a new table is covered automatically. Whatever the mode, delivery is
at-least-once, never exactly-once by default: never run a single unbounded read; chunk with a
predicate so a failed sync resumes from a checkpoint instead of restarting from scratch, and design
the destination write as an idempotent upsert (a dedup/merge key) so a replayed batch doesn't
duplicate.

### 3. CDC + the Outbox pattern
Log-based CDC tails the database's own transaction/commit log (WAL, binlog, redo log) and converts
row-level inserts/updates/deletes into ordered change events with before/after images — no query
polling, low source-DB overhead, ordering preserved. The **Outbox pattern** is the reliable way to
turn an application's internal writes into clean domain events: the app writes its state change and
an outbox row in the same transaction, and CDC on the outbox table republishes it as a domain event
— avoiding the dual-write failure mode (writing to the DB and a message bus as two separate,
non-atomic operations, which silently diverges under partial failure). For a lower-infrastructure
variant of the same idea, solving it entirely within Postgres (same transaction, same commit) avoids
standing up Kafka/RabbitMQ at all when the volume doesn't justify it — match the infrastructure to
the actual throughput/latency need (platform-and-cost row above), not to what feels more "real."

### 4. Reverse-ETL: the accepted capability vs. the contested vendor category
Reverse-ETL — syncing enriched warehouse data back into operational SaaS tools (CRM, marketing,
support) without hand-written API code — is a genuinely useful capability. Its status as a *separate
vendor category* from forward ETL is actively contested by senior data-infra voices, who read "one
company pays to ETL data in, another to reverse-ETL the same data back out" as a symptom of vendor
fragmentation rather than a necessary architectural split — the field itself is consolidating
(reverse-ETL vendors folding into forward-movement platforms' activation lines is one visible
instance). Treat reverse-ETL as a movement direction (warehouse → operational tool), not a distinct
discipline requiring its own platform choice independent of the forward-ETL tooling already in
place.

### 5. Failure diagnosis and destructive-action gating
When a sync fails, diagnose before acting: match the error signature to a root cause (auth expiry,
schema drift, a missing primary key required for incremental mode, an unmet CDC prerequisite, a
network issue) before choosing a fix — a lookup table beats guessing. **Never resync first.** Confirm
the root cause, then choose the least-destructive fix that resolves it; gate any destructive recovery
action (full resync, backfill-and-overwrite, delete-and-reload) behind explicit confirmation rather
than running it automatically. An empty result or zero rows synced is not automatically a failure —
verify against the actual source state before treating "empty" as broken and inventing a problem that
isn't there.

## Validation
- The sync strategy is chosen per table by its actual shape, not applied uniformly across the
  pipeline.
- Destination writes are idempotent (dedup/merge key) against at-least-once delivery — a replayed
  batch does not duplicate.
- Freshness lag is monitored as a first-class signal, with a stated acceptable bound.
- Data leaving the governed perimeter via reverse-ETL is classified before the sync ships.
- A failing sync is diagnosed to root cause before any destructive recovery action, and that action
  is explicitly confirmed, not auto-run.

## Failure modes and handoff
- **One sync strategy applied to every table regardless of shape** — route back to §2; a
  full-refresh on a huge table or an incremental sync on a table with no reliable cursor both fail
  in predictable, avoidable ways.
- **A destructive resync run before root cause was confirmed** — the single most common
  self-inflicted incident at this surface; diagnose first (§5).
- **Dual-write to a database and a message bus treated as equivalent to CDC/Outbox** — it isn't; it
  silently diverges under partial failure (§3).
- **Streaming stood up for a freshness need that batch/micro-batch would have met** — see
  [platform-and-cost.md](platform-and-cost.md); verify the actual latency requirement first.
- **The real ask is analytical consumption of this data**, not the return trip into operations — →
  [surface-product-analytics.md](surface-product-analytics.md) or
  [surface-warehouse-platform.md](surface-warehouse-platform.md).
