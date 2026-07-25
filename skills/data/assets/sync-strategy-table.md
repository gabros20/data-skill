# Sync Strategy Decision Table (fillable)

A vendor-neutral ingestion-strategy taxonomy keyed by table shape — pick one row per source
table, don't default every table to the same sync type. Source:
[ingestion-and-movement.md](../references/ingestion-and-movement.md).

| Table shape | Sync strategy | Why | Watch-outs |
|---|---|---|---|
| Small / dimension (low row count, mutable) | `full_refresh` | Simplest correct option when the resync cost is negligible | Resync cost grows with row count — re-evaluate once "small" stops being true |
| Large, has a reliable `updated_at`/bookmark column | `incremental` (cursor) | Pulls only changed rows since the last bookmark | Bookmark column must be monotonic and reliably updated by the source, or rows silently go missing; changing a `start_date` alone does **not** re-pull history — state precedence wins |
| Append-only / event logs | `append` | Rows are never updated, only added — no need to diff | If the source *can* mutate historical rows, `append` will miss it — confirm append-only is a real invariant, not an assumption |
| Source needs updates *and* deletes captured, at low source-DB overhead | `CDC` (log-based) | Tails the transaction/commit log (WAL/binlog/redo); low query load, preserves ordering, captures deletes | CDC is **not** infinite streaming — each sync reads from last log position to sync-start; first sync still snapshots (`SELECT *`) before log-tailing. Newly created tables can land ahead of connection state — a manual refresh is needed before CDC picks them up. Confirm the source DB's logical-decoding/CDC prerequisites are actually enabled |
| Near-real-time, event-driven, source can push | `webhook` | Lowest latency, no polling | Requires the source to support push delivery and at-least-once redelivery on your end; dedupe on your side, don't assume exactly-once |

## Delivery-safety invariants (apply regardless of chosen strategy)

- **At-least-once delivery, never a bare `SELECT *`.** Chunk large full-refresh/backfill reads with
  predicates so a failed sync resumes from a checkpoint, instead of restarting the whole table.
- **CDC metadata columns** (e.g. `_ab_cdc_*`-style: lsn/log position, updated_at, deleted_at) ride
  along with every CDC row — surface them downstream rather than discarding them; they're the
  audit trail for what changed and when.
- **State precedence.** Some replication runners only emit "state" after a stream (or the whole
  run) completes — a mid-run failure can lose fine-grained resumption. Know which granularity your
  tool actually gives you before relying on it.

## Per-source decision log

| Source table | Row count / shape | Bookmark column (if any) | Strategy chosen | Reverify date |
|---|---|---|---|---|
| | | | | |

**Self-audit:** every source table above has a stated strategy AND a reason it fits its row —
"we've always done full_refresh" is not a reason.
