# Quality and Testing ⭐ (flagship)

Purpose: Make trust in a dataset a **gate**, not a dashboard — a convergent taxonomy of what to
check, where to run each check, which layer catches which class of failure, and how a change gets
reviewed before it merges. The completion condition for every other `data` job: an artifact without
tests and lineage is treated as broken, not merely undocumented.

Read when:
- Nothing yet validates a dataset/pipeline, a check keeps firing false positives or missing real
  breaks, a PR needs a pre-merge data-diff, or a tool choice (GX/Soda/Elementary/dbt tests) is
  undecided.
- An existing test suite only covers schema, and rows are silently wrong (nulls, duplicates,
  business-rule violations, distributional drift) with nothing catching it.

Skip when:
- The ask is the lineage graph, wire format, or five-pillars observability program itself — →
  [lineage-and-observability.md](lineage-and-observability.md). This ref owns the checks; that ref
  owns the standard those checks report *into* and the incident loop around them.
- The ask is the contract/schema-evolution decision (ODCS, compatibility modes) rather than runtime
  checks against real data — → `data-contracts-and-schema-evolution.md`.
- The ask is event-level validation at collection time specifically — → `event-taxonomy-and-
  tracking.md` (this ref's validate-at-ingestion principle applies there too, don't duplicate it).
- The ask is PII detection/masking mechanics — → `governance-pii-and-retention.md`; this ref only
  places "PII/sensitive column" in the taxonomy as a check category.

Inputs:
- The dataset/pipeline's failure history if any (what broke before is the cheapest signal for what
  to test next).
- Which layer the data sits at (source/staging vs marts) — decides which checks are cheap here.
- Team's existing stack (dbt project? Python-native? warehouse-only?) — decides the tool-choice axis.

Produces: a check plan mapped onto the taxonomy below, a PK floor on every model, a where-to-run
assignment per check, a tool choice with license flags resolved, and (for schema/logic changes) a
PR-time blast-radius review before merge.

## Contents
- The convergent DQ taxonomy (lift wholesale)
- Where to run checks
- Known unknowns vs unknown unknowns
- The PK floor
- dbt data tests vs unit tests
- Tool choice + license flags
- PR-time blast-radius review
- Validate-at-ingestion / quarantine
- Audit, never auto-remediate

## Procedure

### 1. The convergent DQ taxonomy — the crown jewel, lift wholesale
Five independent tools (Great Expectations, Soda, Elementary, Deequ, OpenMetadata) plus two
lineage/catalog systems (DataHub, OpenLineage) converged on nearly the same check buckets without
coordinating. That convergence *is* the evidence this taxonomy is complete enough to build a check
plan from — treat it as the reference vocabulary, not one vendor's opinion:

| Failure mode | Best-fit check category | Repos expressing it |
|---|---|---|
| Schema drift (col add/remove/type) | schema/structural | GE `expect_table_columns_to_match_set`; Soda `schema:`; Elementary `schema_changes`; OpenMetadata `tableColumnToMatchSet`; DataHub `AssertionType.DATA_SCHEMA`; OpenLineage `SchemaDatasetFacet` |
| Freshness/staleness | freshness | Soda `freshness:`; Elementary `table.freshness`; Deequ DQDL `DataFreshness`; DataHub `AssertionType.FRESHNESS` |
| Volume/row-count anomaly | volume | GE `expect_table_row_count_to_be_between`; Soda `row_count:`; Elementary `table.row_count`; Deequ `RowCount`/`RowCountMatch`; OpenMetadata `tableRowCountToBeBetween`; DataHub `AssertionType.VOLUME` |
| Completeness (nulls) | column null-rate | GE `expect_column_values_to_not_be_null`; Soda `missing:`+threshold; Elementary `null_percent`; Deequ `isComplete`; OpenMetadata `columnValuesToBeNotNull` |
| Uniqueness / duplicates | uniqueness | GE `expect_column_values_to_be_unique`; Soda `duplicate_check`; Deequ `isUnique`/`IsPrimaryKey`; OpenMetadata `columnValuesToBeUnique` |
| Domain/set validity | validity | GE `expect_column_values_to_be_in_set`; Soda `invalid:{valid_values:}`; Deequ `isContainedIn`; OpenMetadata `columnValuesToBeInSet` |
| Distributional drift/outliers | statistical/anomaly | GE KS/chi-square/z-score; Elementary anomaly-scoring; Deequ anomaly-on-metrics-over-time; whylogs profile-drift constraints |
| Referential/cross-table | pair/multi-column or cross-dataset | GE `expect_column_pair_values_to_be_equal`, `expect_multicolumn_sum_to_equal`; Deequ `ColumnCorrelation`; OpenMetadata/Deequ/data-diff `tableDiff`/`RowCountMatch`/`SchemaMatch` |
| Arbitrary business rule | custom/SQL | Soda `failed_rows_check`; Deequ `CustomSql`; OpenMetadata `tableCustomSQLQuery`; DataHub `AssertionType.SQL`/`CUSTOM`; GE `expect_query_results_to_match_comparison` |
| PR-time / pre-merge regression | value-level diff + blast-radius | Recce (Profile/Value/Top-K/Histogram/Query diff + checklist + PR gate); data-diff (archived engine); OpenMetadata `tableDiff`; Grai (re-run tests on lineage change) |
| PII / sensitive column | recognizer + anonymizer pipeline | Presidio (recognizer catalog → anonymizer operator, decoupled detect/act) — mechanics in `governance-pii-and-retention.md` |

### 2. Where to run checks — a cross-repo convention, not a preference
- **Schema, volume, freshness** — cheapest at **source/staging**. Catch upstream breakage before it
  propagates into every downstream model.
- **Uniqueness, referential, business-rule** — at **marts**. Grain and joins are finalized there;
  running them upstream tests a shape that hasn't settled yet.
- **Distributional/anomaly** — at **all layers, over time**. These need history to establish a
  baseline, which means they require a metrics-history store (DataHub/OpenMetadata provide one via
  `incidentsSummary`/`dataQualityReport`) — you cannot bolt anomaly detection onto a system with no
  memory of its own past.

### 3. Known unknowns vs unknown unknowns — you need both layers
Rule-based tests (§1's schema/volume/completeness/uniqueness/validity/custom-SQL rows) catch **known
unknowns** — failure modes you can name in advance. They cannot scale to every way data can go wrong.
**Unknown unknowns** — a metric that's never been anomalous before, suddenly is — need statistical/ML
anomaly detection layered on top (Monte Carlo's framing: this is why "data downtime" tooling exists
alongside plain tests, not instead of them). Skipping either layer leaves a class of failure
invisible: rule tests alone miss novel drift; anomaly detection alone misses a hard business-rule
violation nobody encoded as a baseline.

### 4. The PK floor
Every model gets a primary-key test as the non-negotiable minimum: `unique` + `not_null` on the PK
column(s), or `dbt_utils.unique_combination_of_columns` for a composite key. This is the cheapest
possible check and the one every convergent tool implements (GE `expect_column_values_to_be_unique`,
Deequ `IsPrimaryKey`, dbt generic tests) — treat its absence on any model as a build blocker, not a
style preference.

### 5. dbt data tests vs unit tests — different axes, don't conflate
- **Data tests** (`unique`, `not_null`, `accepted_values`, `relationships`, plus custom singular
  tests) run against **real production data, post-materialization**. They answer "is the data
  correct right now."
- **Unit tests** run SQL **logic** against **static mocked inputs, pre-materialization**. They
  answer "is this model's transformation logic correct," independent of what production data
  currently looks like.
- dbt Labs' own guidance: run unit tests in **dev/CI only**, exclude from prod runs
  (`--exclude-resource-type`) — they test logic once per change, not data every run.
  `dbt build` sequences unit tests → materialization → data tests, in that order.

### 6. Tool choice — decision axis, with license flags
| Shape | Tool | Fits | License flag |
|---|---|---|---|
| Standalone Python engine, heaviest/most expressive | Great Expectations | Teams wanting the full expectation taxonomy outside any one warehouse/framework | Apache-2.0 |
| YAML DSL + SaaS cloud | Soda / SodaCL | Teams wanting a check language decoupled from SQL, with a hosted UI | **soda-core = Elastic License 2.0 (ELv2) — non-OSI, source-available; no hosted-service offering, no license-key circumvention.** Flag before recommending as "just pip install it and self-host at scale." |
| dbt-native package | Elementary | Teams already on dbt wanting the cheapest-to-adopt observability layer, ML anomaly monitors included | Apache-2.0 (both `elementary` CLI/report and `dbt-data-reliability` package — confirm you're reading both repos, they're separate) |
| Colocated dbt tests/macros | dbt generic tests + `dbt_utils` | Cheapest-ops option: quality as SQL assertions living next to the transformation that produced the data | Apache-2.0. **`dbt-expectations` carries an explicit "no longer actively supported" notice — do not recommend it as a safe default without flagging that staleness risk; prefer native dbt tests + dbt_utils + Elementary for anything it would have covered.** |

Rule of thumb: GX and Soda are heavier/most expressive but add separate infrastructure; Elementary
and colocated dbt tests are cheapest to adopt inside an existing dbt project but weaker on ML/
anomaly detection unless paired with Elementary's monitors.

### 7. PR-time blast-radius review
A dedicated review step sits **after CI's unit/schema/data tests and before merge**: run the change
against base vs PR environments and produce a lineage-aware diff (Profile/Value/Top-K/Histogram/
Query diffs, no hand-written SQL required) plus a re-runnable **checklist artifact** documenting what
was reviewed. The question this answers is different from "did the tests pass" — it's "did this
change touch what we expected, and nothing else." Recce is the canonical pattern (Apache-2.0);
OpenMetadata's `tableDiff` and Grai's lineage-triggered re-test are the same idea expressed inside a
catalog. Treat this step as required for any schema or transformation-logic change, not optional
polish.

### 8. Validate at ingestion, quarantine — never silently drop or accept
Checks belong at the boundary where data enters the system, not only downstream. On a violation:
**quarantine** the record (route it to a held/rejected area for inspection) — never silently drop it
(loses the failure signal) and never silently accept it (the exact failure mode this whole reference
exists to prevent). Segment Protocols' pattern — validate incoming events against the plan, block
non-conforming ones, route them to a quarantined source — generalizes past events to any ingestion
boundary: a landing table, a CDC stream, a file drop.

### 9. Audit, never auto-remediate
A check plan's job is to detect and report, not to silently fix. Pattern (from PostHog's warehouse-
health tooling): audit → grouped, severity-ranked, cascade-aware report → **hand off for a human
decision**, never auto-remediate. Any recovery action that would resync, backfill, or otherwise
mutate data is a destructive action and needs explicit confirmation before running — "never resync
first, confirm before destructive ops." This applies even when the fix looks obvious; an automated
"fix" that runs before root cause is understood can compound the failure.

## Validation
- Every model has the PK floor (`unique`+`not_null` or composite equivalent) at minimum.
- The check plan maps each identified failure mode onto §1's taxonomy and assigns it to a layer per
  §2 — no check placed upstream/downstream by default without reasoning about cost.
- Both known-unknown (rule) and unknown-unknown (anomaly) layers are present where history exists to
  support anomaly baselines; their absence is stated explicitly if genuinely out of scope.
- Any schema or transformation-logic change gets a PR-time blast-radius review before merge.
- Tool recommendations carry their license flag (soda-core ELv2, dbt-expectations unmaintained) —
  never presented as a safe default without it.
- Ingestion validation quarantines rather than drops or silently accepts.

## Failure modes and handoff
- **Only schema tests exist, rows are still wrong** — add completeness/uniqueness/validity checks
  from §1 before reaching for anomaly detection; rule tests are cheaper and catch more than they're
  usually credited for.
- **Anomaly detection is noisy/ignored** — check whether a metrics-history store actually exists
  (§2); anomaly detection without baseline history produces false positives that train people to
  ignore alerts.
- **A "fix" was applied without review** — this violates §9; stop, revert if the fix was destructive,
  and route the finding back through an audit-report-confirm cycle.
- **The ask is really about the lineage graph / catalog these checks report into, not the checks
  themselves** — → `lineage-and-observability.md`.
- **The ask is a data-contract SLA or schema-evolution rule, not a runtime check** — →
  `data-contracts-and-schema-evolution.md`.
