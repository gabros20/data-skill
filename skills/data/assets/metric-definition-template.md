# Metric Definition Template (fillable)

A semantic/metrics layer has exactly one job: **one authoritative definition per metric, consumed
by every tool and agent.** If two tools compute "revenue" differently, the layer has already
failed — this template exists to make that failure structurally hard. Source:
[metrics-semantics-and-bi.md](../references/metrics-semantics-and-bi.md).

## Per-metric definition

| Field | Fill in |
|---|---|
| Metric name | |
| Type — `simple` / `ratio` / `derived` / `cumulative` (pick one, see table below) | |
| Owner (person/team accountable for the definition) | |
| Grain (the lowest level this metric is computed at, e.g. per-user-per-day) | |
| Filters (what's excluded, e.g. `is_test_account = false`) | |
| Authoritative home (the ONE tool/repo where this is defined — every other surface reads from here) | |

## Metric-type decision table (MetricFlow shape — lift the axis, not necessarily the tool)

| Type | Use for | Shape |
|---|---|---|
| `simple` | Direct aggregation of one measure | `expr` + `agg` (e.g. `sum(order_total)`) |
| `ratio` | Business ratios (e.g. bookings-per-booker) | `numerator` / `denominator`, each with its own optional `filter` |
| `derived` | Algebraic combination of other metrics (e.g. `booking_value * 0.05`) | `expr` + `metrics: []`; each input metric can carry its own `filter`/`offset_window`/`offset_to_grain` for period-over-period comparisons |
| `cumulative` | Trailing/to-date windows (MTD, trailing-2-month) | `cumulative_type_params`: `window`, `grain_to_date`, `period_agg` |

- A naive ad-hoc `count(where ...) / count(...)` silently mishandles edge cases the business
  actually cares about (e.g. "on-time arrival" excluding cancelled/diverted flights). Encode the
  rule **once**, here, so every downstream query is consistent — that's the entire argument for a
  semantic layer.
- Ambiguous definitions (what counts as "active user"? does a refund subtract from `revenue` or
  get excluded entirely?) belong in the Filters/Grain rows above, spelled out, not left implicit
  in whoever's SQL got there first.

## Registry (one row per metric, keep this the single source of truth)

| Metric name | Type | Owner | Grain | Filters | Authoritative home |
|---|---|---|---|---|---|
| | | | | | |

## Drift check (run before adding a new BI dashboard or agent tool that touches metrics)

- [ ] This metric already exists in the registry above — reused, not redefined
- [ ] If genuinely new, it's added to the registry with a named owner before first use elsewhere
- [ ] No SQL query outside the authoritative home recomputes this metric's logic independently

**Self-audit:** any metric computed in more than one place without a stated reason is drift —
treat it as a defect, not a style preference.
