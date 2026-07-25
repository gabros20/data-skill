# Transformation and Modeling

Purpose: Decide how raw source-conformed data becomes business-conformed marts — the
staging→intermediate→marts layering, when to escalate materialization, which lint thresholds mean
a model needs splitting, and where a dbt-alternative (SQLMesh) changes the defaults.

Read when:
- The request is "build/refactor a dbt project," "design the marts layer," "our DAG is a mess,"
  "should this be a view or a table," or "staging vs. intermediate vs. marts."
- A PR touches model structure, materialization config, or SQL style and needs a lint pass before
  merge.

Skip when:
- The ask is writing or reasoning about data-quality checks/tests themselves — →
  `quality-and-testing.md` (this ref only states the structural PK-test floor as a lint rule).
- The ask is choosing a warehouse/engine or reading a license matrix — → `platform-and-cost.md`
  (this ref states only the dbt Core/Fusion split as it bears on which engine runs the SQL).
- The ask is metric definitions, a semantic layer, or BI tooling — →
  `metrics-semantics-and-bi.md`.
- The ask is ingestion/sync strategy upstream of transformation — → `ingestion-and-movement.md`.

Inputs:
- Existing model DAG (or a description of the target one) and its current materializations.
- Whether the project is dbt Core, dbt Fusion, or SQLMesh — the lint thresholds and
  materialization defaults below assume dbt-shaped layering either way.

Produces:
- A staging/intermediate/marts layering decision with naming and materialization per layer.
- A lint pass against the dbt-project-evaluator thresholds (or the SQLMesh equivalent).
- A SQL-style verdict (CTE shape, linter config) when style is in question.

## Contents
- Layering: staging → intermediate → marts
- Lint-rule decision tables (dbt-project-evaluator — canonical home)
- Materialization ladder and incremental strategies
- SQLMesh contrast
- Modeling choice: Kimball / OBT / Data Vault
- SQL style essentials
- Licensing: dbt Core vs. dbt Fusion

## Procedure

### 1. Layering: staging → intermediate → marts
dbt Labs' "How We Structure" canon (docs.getdbt.com/best-practices/how-we-structure, verbatim
quotes) frames the whole layer as moving data "from *source-conformed* to *business-conformed*."
- **Staging** (`stg_`): 1:1 with a source table; the **only** place `source()` is used;
  **materialize as views**. Allowed: renaming, type casting, basic computation, categorizing.
  Forbidden: joins and aggregations — *"joins are almost always a bad idea here."*
- **Intermediate** (`int_`): named `int_[entity]s_[verb]s.sql` (e.g.
  `int_order_items_summed_to_orders`), verb-oriented. **Default materialization: ephemeral** (or
  views in a custom schema). **Never exposed** to end users. Three legitimate uses: structural
  simplification, re-graining, isolating complex logic.
- **Marts** (business-entity names, e.g. `customers.sql`; `fct_`/`dim_` for genuine fact/dimension
  splits): **"wide and denormalized"** is the explicit modern-stack recommendation — *"storage is
  cheap and it's compute that is expensive."* Anti-pattern: too many joins in one mart (>4–5
  concepts → push into an intermediate). Materialize as **table or incremental**.
- **DAG shape invariant** (verbatim): *"Narrow the DAG, widen the tables... several arrows going
  **into** our post-staging models is great and expected, several arrows coming **out** is a red
  flag."*

### 2. Lint-rule decision tables (dbt-project-evaluator — canonical home)
`dbt-labs/dbt-project-evaluator` (Apache-2.0) is the reference implementation of the layering rules
above as automated lint checks. This is their canonical home — cite this table rather than
re-deriving thresholds ad hoc.

**Naming prefixes** (`docs/rules/structure.md`):

| Model type | Prefix |
|---|---|
| Staging | `stg_` |
| Intermediate | `int_` |
| Marts | `fct_` or `dim_` |
| Other/report | `rpt_` |

**Structure rules:** staging nests in `staging/<source>/`; one `.yml` per source-subfolder;
co-locate tests with models (one `.yml` per model-type folder) — never a single centralized test
file.

**Modeling invariants (each is a lint rule):**
- Staging is atomic, 1:1 with a raw source table.
- A `{{ source() }}` has exactly **one** direct child — **source fan-out = 1**. Only staging
  models reference sources directly; no marts/intermediate → source, and never mix a model
  reference and a source reference in one model.
- No hard-coded relations — always `ref()`/`source()`.
- **No staging→staging** dependencies.
- Fold pass-through intermediate models into their child as a CTE ("rejoining of upstream
  concepts") rather than leaving a trivial standalone model.
- A root model (0 parents) is a lineage bug unless deliberately self-contained (e.g. a
  `date_spine` calendar).

**Thresholds:**
- Model fan-out **>3** direct leaf children → push logic to BI or further upstream instead of
  fanning out from one model.
- **>7** direct refs into one model ("too many joins") → split into intermediates, each joining
  4–6 concepts.
- **≥4** chained non-materialized (view) models before a table → warn; long view chains compound
  compute at query time.
- **Materialization rule of thumb** (verbatim): *"if a view is used downstream by many models,
  change materialization to table"*; *"if a view has more complex calculations (window functions,
  joins between many tables), change materialization to table."*
- **Testing floor:** every model needs a primary-key test — `unique`+`not_null` on the grain
  column(s), or `dbt_utils.unique_combination_of_columns`. (Full DQ-check design beyond this
  structural floor is `quality-and-testing.md`'s.)
- **Exposures** must depend on `table`/`incremental` models only — never directly on a `source`,
  `ephemeral`, or `view` model.

### 3. Materialization ladder and incremental strategies
Climb this ladder only on a measured bottleneck, not by default:
- **view** — no storage cost, always fresh, slow if stacked. The default starting point for every
  new model.
- **table** — fast reads, full rebuild each run, no auto-append. Use for BI-facing models and slow
  shared upstream transforms.
- **incremental** — inserts/updates only new records. dbt's own framing: an "advanced usage
  pattern" — move to it "only after `dbt run`s become too slow," not preemptively.
- **ephemeral** — inlined as a CTE; can't be queried or `ref()`'d by operations, no contracts
  support. Lightweight early-DAG logic only.
- **materialized view** — database-managed refresh (Snowflake uses Dynamic Tables instead;
  Postgres lacks native support).
- Python models support only `table`/`incremental` — no view/ephemeral option.

**Incremental strategy choice:** `merge` (upsert on `unique_key`) · `delete+insert` ·
`append` (**default when no `unique_key` is set** — can silently duplicate rows) ·
`insert_overwrite` (partition-based, ignores `unique_key`) · `microbatch`.

**Gotchas:** `unique_key` columns must not be null — a null `unique_key` value breaks the merge
semantics; duplicate `unique_key` values can fail or duplicate rows depending on strategy/adapter;
set `on_schema_change` explicitly when columns are added or removed, or schema drift silently
breaks the incremental run.

### 4. SQLMesh contrast
`TobikoData/sqlmesh` (Apache-2.0 code + CC-BY-4.0 docs, Linux Foundation project) is the dbt
alternative worth naming when the team wants Terraform-style `plan`/`apply` semantics or is
fighting compute duplication across dev branches:
- **Virtual Data Environments** — dev environments are views over the same physical tables, so
  branching costs zero extra compute (contrast dbt's per-environment schema rebuild).
- `MODEL` DDL block + a trailing `SELECT` — no separate YAML for model config.
- Multi-dialect via SQLGlot; built-in unit tests/audits; column-level lineage and diffing.

**Model-kind decision table** (`docs/concepts/models/model_kinds.md`):

| Kind | Use for | Gotcha |
|---|---|---|
| `INCREMENTAL_BY_TIME_RANGE` | immutable, time-stamped events | needs `time_column` + `@start_date`/`@end_date` WHERE; `lookback` handles late-arriving data |
| `INCREMENTAL_BY_UNIQUE_KEY` | upsert by key | **does not dedupe automatically** — the query itself must not return duplicate keys |
| `FULL` | small aggregates only | full rewrite every run |
| `VIEW` (default) | — | **re-evaluated on every downstream reference** — a real compute cost, not free like it looks |
| `SCD Type 2` | historized dimensions | auto `valid_from`/`valid_to` |
| `EMBEDDED`, `SEED`, `EXTERNAL`, `MANAGED`, `INCREMENTAL_BY_PARTITION` | — | — |

**Philosophy (verbatim):** *"Never build a table more than once"* — SQLMesh defaults to
incremental-by-default where dbt's cultural default is closer to full-refresh-then-optimize-later.

### 5. Modeling choice: Kimball / OBT / Data Vault
dbt's own "How We Structure" marts guidance leans **OBT-flavored**: wide/denormalized marts over
normalized fact/dimension splits, citing cheap storage vs. expensive compute — while still
supporting `fct_`/`dim_` naming for genuine star schemas when the grain calls for it.

**[UNVERIFIED]** No dedicated dbt Kimball-vs-OBT-vs-Data-Vault comparison doc could be verified
(candidate docs 404'd/redirected as of 2026-07-25) — do not attribute a full three-way comparison
to dbt. Only the marts-page OBT lean above is directly sourced. General tradeoffs (background
theory, not link-verified this pass): **Kimball** (conformed fact/dimension schemas) = reusable
dimensions, more joins/ETL to build and maintain; **OBT** = BI simplicity + columnar-storage
economics, at the cost of redundancy and harder reuse across marts; **Data Vault** (hubs/links/
satellites) = auditability, historization, and parallel loads at enterprise scale, but needs a
downstream marts layer before it's user-facing — never serve Data Vault tables directly to BI.

### 6. SQL style essentials
dbt Labs' official style guide (docs.getdbt.com/best-practices/how-we-style, verbatim rules):
tenets **"Clarity"** and **"Consistency,"** enforced with **SQLFluff**. 4-space indent, 80-char
max line, lowercase keywords, trailing commas, explicit `as`. Fields before aggregates/window
functions; aggregate as early as possible, before joining. Prefer `group by`/`order by` numbers
over repeating expressions. Joins: `union all` over `union`; always prefix columns with the table
name once 2+ tables are joined; explicit `inner join` (never bare `join`); a repeated `right join`
is a style smell — restructure as `left join`.

**CTE shape — two acceptable forms:**
- **Import CTEs** — all `ref()`/`source()` calls at the top, column-pruned, one per referenced
  relation.
- **Functional CTEs** — one logical transformation unit per CTE, verbosely named.
Either way, the final line is `select *` from the final CTE — never a bare column list buried
mid-file.

**Linting:** `sqlfluff` (MIT) — dialect-flexible, Jinja/dbt-aware, lints + fixes + parses, with a
dedicated `dbt` templater for compiled Jinja SQL. **Gotcha:** `ignore_templated_areas = True` is
sqlfluff's *default* — lint errors inside Jinja-generated code are silently suppressed unless this
is explicitly turned off, which surprises teams expecting full-file coverage.

### 7. Licensing: dbt Core vs. dbt Fusion
**(as-of 2026-07-25 — re-verify before a hard licensing claim; full cross-vendor license map is
`platform-and-cost.md`'s.)** **dbt Core** is Apache-2.0; dbt Labs states it will "continue
maintaining dbt Core under the Apache 2 license indefinitely." **dbt Fusion** (the Rust engine,
GA cited as 2026-06-01) is licensed under the **Elastic License 2.0** for nearly every file —
only a handful of crates (`dbt-agate`, `dbt-auth`, `dbt-adapter`, `dbt-jinja`, `dbt-xdbc`) are
Apache-2.0. Fusion is free to download and use locally, and its core features (v1 workflows,
lint, docs-lite, LSP-lite) are free-forever without login — but premium features (full LSP,
SQL comprehension/type-checking, column-level lineage, full docs v2) require a dbt Platform
account login. **Free-to-use is not open-source** — don't recommend "just switch to Fusion" as a
drop-in without flagging the license change; dbt Core remains the fully-open path.

## Validation
- The layering matches staging (1:1, view, no joins) → intermediate (ephemeral, never exposed) →
  marts (wide, table/incremental) — not a flattened two-layer shortcut.
- No source fan-out >1, no staging→staging dependency, no model with >7 direct refs left unsplit.
- Every model has a PK test (`unique`+`not_null` or equivalent); every exposure points at a
  `table`/`incremental` model.
- A materialization escalation (view→table→incremental) is justified by a measured slowdown, not
  applied speculatively.
- Any dbt Fusion recommendation states the ELv2 license split, not just "it's the new engine."

## Failure modes and handoff
- **The ask is really about test/anomaly-check design, not structure** — → `quality-and-testing.md`.
- **The ask is a warehouse/engine choice or a licensing comparison across vendors** — →
  `platform-and-cost.md`.
- **The ask is metric definitions or a semantic/BI layer sitting on top of these marts** — →
  `metrics-semantics-and-bi.md`.
- **The DAG problem is actually upstream** (sync strategy, CDC, raw-load shape) — →
  `ingestion-and-movement.md`.
