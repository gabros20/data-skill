# Ingestion & Movement

Purpose: Pick the sync strategy that fits a table's actual shape — full refresh, incremental,
append, CDC, or webhook — default to ELT over ETL, and diagnose/gate recovery when a sync breaks,
without reaching for a distributed pipeline the workload doesn't need.

Read when:
- Connecting a new source, choosing ELT vs ETL, or picking a per-table sync/replication strategy.
- CDC is on the table (Postgres/MySQL replication, Debezium, Outbox) and needs a justification.
- A sync is failing, stuck, or has produced bad rows and needs a root cause and a recovery action.
- Someone proposes reverse-ETL or streaming and the decision needs a capability-vs-hype check.

Skip when:
- The ask is the naming/capture of the event itself, before it moves anywhere — →
  `event-taxonomy-and-tracking.md`.
- The ask is the registry-level contract (compatibility mode, ODCS) the moved data must honor — →
  `data-contracts-and-schema-evolution.md`.
- The ask is whether streaming is cost-justified on workload-shape grounds, or warehouse/
  orchestrator choice — → `platform-and-cost.md` (owns the COST rule and the orchestrator table).
- The ask is ongoing trust in data already landed (tests, anomaly detection) rather than one-time
  sync health — → `quality-and-testing.md`.

Inputs:
- Table shape: row count, whether a reliable `updated_at`/PK exists, append-only vs mutable,
  source's native change-feed support.
- Latency requirement stated in minutes, not "real-time" — decides full_refresh/incremental/append
  vs CDC/webhook.
- Current sync status, `latest_error`, `last_synced_at` — for diagnosis, not just setup.
- Whether the target is genuinely reverse-ETL or an ETL tool wearing a different label.

Produces: a per-table sync-type decision, a CDC/Outbox design when justified, a schema-contract
mode for the ingestion library in use, a root-cause + recovery action for a broken sync (with
destructive actions gated behind explicit confirmation), and a reverse-ETL/vendor-category call.

## Contents
- ELT vs ETL
- The sync-type decision table (canonical home)
- CDC: log-based mechanics and the Outbox pattern
- Schema contracts in motion
- Streaming — justify before reaching for it
- Reverse-ETL: capability vs vendor category
- Diagnosing a broken sync

## Procedure

### 1. ELT vs ETL
**ELT is the default**: land raw data first, transform in-warehouse with SQL. Decoupling
extraction/loading from transformation means transformation delays never block landing, and cheap
cloud storage/compute makes "land everything, shape later" affordable
(fivetran.com/blog/etl-vs-elt, 2026-07-25). **ETL still wins** in two narrow cases: schemas that
are stable and well-established, and compliance contexts requiring transformation — PII
scrubbing/masking — *before* data lands in a shared warehouse (same source; masking-before-landing
policy itself is `governance-pii-and-retention.md`'s job, this ref only flags the fork it creates).

### 2. The sync-type decision table (canonical home)
Pick per table, keyed by shape — not by vendor default (PostHog `setting-up-a-data-warehouse-
source` skill, MIT, github.com/PostHog/skills, 2026-07-25; lifted verbatim, vendor-neutral
taxonomy):

| Table shape | Sync type | Why |
|---|---|---|
| Small / dimension (<50k rows, no natural ordering column) | `full_refresh` | Simple and always correct |
| Large, has `updated_at`/`modified_at` | `incremental` | Much cheaper per sync than re-pulling everything |
| Append-only immutable (logs, events) | `append` | Preserves history; never rewrites |
| Postgres with CDC enabled, near-real-time need | `cdc` | Requires primary keys + CDC prerequisites on the source |
| Source supports webhooks (e.g. Stripe) | `webhook` | Near-real-time push; still needs a bulk sync_type as a safety net for tables that don't support webhooks |

For incremental/append/CDC, prefer `updated_at` over `created_at` as the high-water mark —
`created_at` misses late-arriving updates. Integer-only tables fall back to a monotonically
increasing PK.

**Airbyte's replication-mode semantics** (Apache-2.0 connectors/CDK, ELv2 platform — no
self-hosting-as-a-service, 2026-07-25): Full Refresh · Incremental (cursor) · CDC (log-based). CDC
is *not* infinite streaming — each sync reads from the last log position to sync-start; the first
sync snapshots (`SELECT *`) before log-tailing. **Invariant: at-least-once delivery** — never a
bare `SELECT *`; chunk with predicates (`CTID >= X AND CTID < Y`) so a failed sync resumes from a
checkpoint ("Resumable Full Refresh"), not from zero. **Gotcha**: a newly created source table
sits ahead of connection state, so CDC won't pick it up until a manual schema refresh.

**Singer-spec state-precedence gotcha** (Meltano, MIT, 2026-07-25): taps emit STATE per stream;
the runner merges by "State ID" (extractor+loader+environment) so retries resume mid-stream.
**Changing `start_date` alone does not re-pull history — persisted state wins over the date
parameter.** Only `--full-refresh` ignores state. This is the most common "why didn't my backfill
work" bug in Singer-based pipelines.

### 3. CDC: log-based mechanics and the Outbox pattern
Debezium (Apache-2.0, latest stable v3.6.0.Final released 2026-07-01 — re-verify at
github.com/debezium/debezium/tags) tails the database's transaction/commit log (Postgres WAL,
MySQL binlog, Oracle redo, SQL Server change tables) and converts row-level INSERT/UPDATE/DELETE
into ordered change events with before/after images. This is low source-DB overhead (no query
polling) and preserves ordering — the structural reason CDC beats poll-based incremental for
high-churn tables. Runs on Kafka Connect, or embedded inside an app when no Kafka broker/durability
is needed.

**Transactional Outbox** is the canonical reliable pattern for an app that must both update its own
state and publish a domain event: write both in the same DB transaction to a local `outbox` table,
then let CDC (Debezium's Outbox Event Router SMT) tail *that* table and republish clean domain
events — avoiding the dual-write problem (DB commit succeeds, message-bus publish fails, or vice
versa) without two-phase commit. Gotcha: Postgres logical-decoding plugin choice (`pgoutput` vs
`wal2json` vs `decoderbufs`) changes connector behavior and test setup — pin it explicitly.

### 4. Schema contracts in motion
dlt (Apache-2.0, 2026-07-25) infers schema from data and normalizes nested/JSON into relational
tables, but treats every incoming record against a **schema contract** with three modes, settable
per `tables`/`columns`/`data_type`: `evolve` (accept the change, adapt the schema), `freeze`
(reject the record), `discard` (drop the offending row/column, keep the rest). Pick `evolve` for
exploratory/early pipelines, `freeze` once a downstream consumer depends on a stable shape — this
is the ingestion-time analog of the registry-level compatibility modes
`data-contracts-and-schema-evolution.md` owns once the dataset is governed.

### 5. Streaming — justify before reaching for it
Batch/micro-batch is sufficient for most latency needs; reach for streaming infra
(Kafka/Debezium/Flink) only for a genuine sub-minute-latency or true event-driven-architecture
requirement, never because it "feels real-time-ish." The cost argument for staying batch —
McSherry's COST rule and the workload-shape economics — is `platform-and-cost.md`'s job; this ref
only states the gate: **name the latency requirement in minutes before reaching for streaming, or
the decision is unjustified.**

### 6. Reverse-ETL: capability vs vendor category
The *capability* — syncing enriched warehouse data back into operational SaaS tools
(CRM/marketing/support) — is uncontested and useful; "data activation" is its applied use. The
*separate vendor category* is what senior practitioners are skeptical of: Chris Riccomini flags
reverse-ETL skepticism as natural "for those coming from a CDC background"
(x.com/criccomini/status/1404516589534212096), and Erik Bernhardsson asks the structural question
directly — "why do we have one set of startups doing ETL connectors and another one doing reverse
ETL connectors?" (x.com/bernhardsson/status/1435639674987495429), noting substantial synergy
between the two directions. **Fivetran acquired Census** (dated 2025-05-01;
`getcensus.com/reverse-etl` now redirects to `fivetran.com/data-movement/activations`) — market
evidence the category is consolidating back into general data-movement tooling rather than staying
a separate class of vendor. Treat "reverse-ETL platform" as a feature of a data-movement platform,
not an architecturally distinct system, when evaluating build-vs-buy.

Multiwoven (open-source reverse-ETL) is **AGPLv3** — copyleft, network-use triggers
source-disclosure; a materially different risk profile from the Apache/MIT tools elsewhere in this
cluster, flag it before adopting (2026-07-25).

### 7. Diagnosing a broken sync
Map the error to a root cause before picking a fix (PostHog `diagnosing-failed-warehouse-syncs`
skill, MIT, 2026-07-25 — the pattern generalizes past PostHog):

| Symptom | Root cause | Fix |
|---|---|---|
| `authentication failed` / 401 / 403 | Credentials expired or rotated | Rotate creds, update source config, reload |
| `Primary key required for incremental syncs` | Table has no PK but sync_type is incremental/CDC | Add a PK, or fall back to `full_refresh` |
| PK columns "not unique" | Declared PK isn't actually unique | Pick different PK columns |
| `column "X" does not exist` | Schema drift — tracked column removed/renamed | Re-detect fields, then update the schema config |
| `relation "..." does not exist` | Source table dropped/renamed | Remove the schema or rename source-side |
| SSL / connection refused / timeout | Network/firewall/host reachability | Source-side network fix |
| `replication slot` / `publication` / `wal_level` | CDC prerequisites broken | Check CDC prerequisites; may need a slot recreate |
| Status stuck `Running`, `last_synced_at` hours old | Orphaned job | Cancel, then reload |
| Bulk sync `Completed` but no new rows via webhook | Webhook never registered, disabled, or signing secret rotated | Check webhook registration/status, re-register or update secret |

**Recovery-action gating — never resync first.** Three actions discard data and cannot be undone:
full resync (wipe + re-import), delete synced data, destroy the source entirely. Always name the
proposed fix and get explicit confirmation before calling any of them; most transient failures
(network blips, expired creds) resolve with a plain reload that resumes from checkpoint, not a
resync from zero. Treat "just try resync" as a default to avoid, not a first move.

## Validation
- [ ] Every synced table has a sync_type chosen by shape (the table above), not by copying
      whatever the last table used.
- [ ] Incremental/append/CDC tables use `updated_at` over `created_at` where both exist.
- [ ] A stated latency requirement (in minutes) exists before any streaming/CDC-for-real-time
      decision.
- [ ] A broken sync's root cause is identified before a fix is proposed, and destructive recovery
      actions (resync, delete-data, source-destroy) are confirmed explicitly, never defaulted to.
- [ ] Reverse-ETL is evaluated as a capability of a data-movement platform, not assumed to need a
      separate vendor category.
- [ ] Every license claim (Airbyte ELv2, Multiwoven AGPLv3, Debezium/dlt Apache-2.0) is verified
      against the LICENSE file, not the GitHub API license field.

## Failure modes and handoff
- **The real ask is whether streaming is cost-justified, or which warehouse/orchestrator to use**
  — → `platform-and-cost.md`.
- **The real ask is the registry-level schema contract once data is governed**, not the
  ingestion-time schema-evolution mode — → `data-contracts-and-schema-evolution.md`.
- **The real ask is ongoing data trust (tests, anomaly baselines), not one-time sync health** — →
  `quality-and-testing.md`.
- **PII must be scrubbed/masked before landing** (the ETL exception in §1) — →
  `governance-pii-and-retention.md`.
- **Vendor pricing/version specifics move fast** (Debezium releases, Airbyte tiers) — re-verify
  against current vendor docs/release pages before citing as current.
