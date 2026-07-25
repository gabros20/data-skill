# Data Quality Check Matrix (fillable)

The convergent DQ taxonomy — five independent tools (Great Expectations, Soda, Elementary, Deequ,
OpenMetadata) plus two catalogs (DataHub, OpenLineage) evolved to nearly the same ~9 check
buckets. Fill `Layer`, `Tool`, and `Threshold` per bucket for your stack; don't skip a bucket
without a stated reason. Source: [quality-and-testing.md](../references/quality-and-testing.md).

| # | Bucket | Layer (staging / marts / all) | Tool (name yours) | Threshold |
|---|---|---|---|---|
| 1 | Schema/structural (col add/remove/type drift) | staging | | |
| 2 | Freshness/staleness | staging | | |
| 3 | Volume/row-count anomaly | staging | | |
| 4 | Completeness (nulls) | staging or marts | | |
| 5 | Uniqueness/duplicates | marts | | |
| 6 | Domain/set validity | marts | | |
| 7 | Distributional drift/outliers (statistical/anomaly) | all (needs history) | | |
| 8 | Referential/cross-table | marts | | |
| 9 | Arbitrary business rule (custom/SQL) | marts | | |

Two buckets sit outside the per-table matrix — track them separately, not as a row above:

| Cross-cutting check | What it catches | Tool (name yours) |
|---|---|---|
| PR-time / pre-merge regression (value-level diff + blast-radius) | a change about to ship breaks downstream before merge | e.g. Recce |
| PII / sensitive column | a column carries PII that isn't classified/masked yet | e.g. Presidio (recognizer → anonymizer) — see [pii-audit-checklist.md](pii-audit-checklist.md) |

## Where to run checks (cross-repo convention)

- **Schema / volume / freshness** — cheapest at **source/staging**: catch upstream breakage before
  it propagates downstream.
- **Uniqueness / referential / business-rule** — at **marts**: grain and joins are finalized by
  then, so these checks mean something.
- **Distributional / anomaly** — at **all layers, over time**: needs history for a baseline, which
  requires a metrics-history store (a catalog's `incidentsSummary`/`dataQualityReport` equivalent).

## Known-unknowns vs unknown-unknowns (both layers required)

- **Rule-based tests** (buckets 1–6, 8–9) catch **known unknowns** — things you already thought to
  check.
- **Statistical/ML anomaly detection** (bucket 7) catches **unknown unknowns** — the failure mode
  you didn't write a rule for. One without the other is an incomplete quality gate.

## Primary-key floor (non-negotiable minimum)

Every primary key gets `unique` **and** `not_null` — this is the floor, not a target. If a table
has no test at all, this is the first pair to add.

| Table | PK column | `unique` passing | `not_null` passing |
|---|---|---|---|
| | | | |

**Self-audit count: ____ / 9 buckets assigned a layer + tool + threshold**, plus PR-time diff and
PII checks wired in separately. Any bucket left blank must carry a one-line reason (e.g. "append-
only log table, referential checks not applicable") — never a silent gap.
