# Metrics, Semantics, and BI

Purpose: Decide how to give a metric exactly one governed definition across every tool that
consumes it, which metric shape (simple/ratio/derived/cumulative) a given business question needs,
and whether the BI layer on top should be code-first or GUI-first.

Read when:
- The request is "two dashboards show different revenue numbers," "build a semantic layer,"
  "should this be a metric in dbt/Cube/Malloy," or "pick a BI tool."
- The team is arguing over a metric's definition and needs a single authoritative home for it.

Skip when:
- The ask is the underlying marts/staging models a metric is built on — →
  `transformation-and-modeling.md`.
- The ask is data-quality checks on the underlying tables — → `quality-and-testing.md`.
- The ask is warehouse/engine cost or licensing comparison — → `platform-and-cost.md`.
- The ask is experiment-metric validity (guardrails, SRM, CUPED) rather than reporting metrics —
  → `experiment-measurement-foundations.md`.

Inputs:
- The metric(s) in dispute or under design, and every tool currently computing them.
- Whether a semantic layer already exists (dbt Semantic Layer/MetricFlow, Cube, Malloy, LookML) or
  this is a greenfield decision.

Produces:
- A metric-type classification (simple/ratio/derived/cumulative) with its shape.
- A semantic-layer-or-not decision with the tradeoff named.
- A BI-tool recommendation on the code-first/GUI-first axis, not just a feature comparison.

## Contents
- The metric-drift problem
- MetricFlow metric-type decision table (canonical home)
- Malloy: why a semantic layer at all
- Semantic-layer skepticism (the counterweight)
- Code-first vs. GUI-first BI — the real dividing line
- dbt's 4-tier NL-question fallback
- Dashboards are for historians — decision-first serving

## Procedure

### 1. The metric-drift problem
The reason a semantic layer exists at all: two tools computing "revenue" differently is a
correctness failure, not a stylistic difference. dbt's Semantic Layer/MetricFlow explicitly frames
its job as solving **metric drift** via one authoritative definition consumed across every
downstream tool (Tableau, Excel, Hex, Mode) instead of each tool re-deriving the aggregation.
Benn Stancil (Mode founder) named the gap before the category existed — *"The missing piece of the
modern data stack"* (benn.substack.com/p/metrics-layer): there was no central repo for metric
definitions; he proposed a metrics layer that does for metrics what dbt did for models.
**Consensus-adjacent** — this essay is widely credited with spawning the semantic-layer product
category.

### 2. MetricFlow metric-type decision table (canonical home)
`dbt-labs/metricflow` (Apache-2.0) is the metrics engine behind the dbt Semantic Layer. Its
semantic-manifest shape: a `semantic_model` (name, `node_relation`, `primary_entity`, `entities`
typed foreign/primary, `dimensions` typed time with `time_granularity`, measures) plus `metric`
definitions. This is the canonical home for the metric-type table below — lift it, don't
re-derive it:

| Metric type | Use for | Shape |
|---|---|---|
| `simple` | direct aggregation of one measure | `expr` + `agg` |
| `ratio` | business ratios (e.g. bookings-per-booker) | `numerator`/`denominator`, each with an optional `filter` |
| `derived` | algebraic combination of other metrics (e.g. `booking_value * 0.05`) | `expr` + `metrics:[]`; each input metric carries its own `filter`/`offset_window`/`offset_to_grain` for period-over-period |
| `cumulative` | trailing/to-date windows (MTD, trailing-2-month) | `cumulative_type_params`: `window`, `grain_to_date`, `period_agg` |

Metric filters use `{{ Dimension('booking__is_instant') }}` syntax, including cross-entity joins
via `entity_path`.

**License history (as-of 2026-07-25, re-verify before citing):** MetricFlow's README states its
own arc — *"v0–0.140.0 = Affero GPL. v0.150.0–0.208.2 = BSL. v0.209.0+ = Apache 2.0."* Current
`main` is Apache-2.0. **Gotcha:** the README still carries a legacy "source-available software"
header — don't repeat that stale characterization without the version caveat. MetricFlow now
anchors the cross-vendor **Open Semantic Interchange (OSI)** initiative with Snowflake — a shared
metric-definition interchange format, still early; re-verify its scope before depending on it.

### 3. Malloy: why a semantic layer at all
`malloydata/malloy` (MIT) states the argument for encoding a metric once better than most vendor
docs (verbatim-usable): metrics like "on-time arrival" have ambiguous real-world definitions — the
US DOT defines it as `arr_delay < 15` while *excluding* cancelled and diverted flights — and a
naive `count(where) / count()` silently mishandles those edge cases differently in every query
that writes it out by hand. Encoding the rule once, in a `source`/`measure` definition, keeps
every downstream query consistent instead of re-litigating the edge case per analyst. Malloy
compiles to SQL on BigQuery/Snowflake/DuckDB/Postgres/Trino/Databricks, and every query leaves a
reusable semantic model as a byproduct.

Cube (`cube-js/cube`, Apache-2.0 root — per-package, `cubejs-client-core` is MIT) makes the
anti-drift argument concrete for AI-generated SQL too: its "Semantic SQL" adds a `MEASURE`
function so every query — human- or agent-written — is validated against the semantic layer
*before* it hits the warehouse. Its pre-aggregation layer is a separate decision: application
databases give low latency but don't scale; warehouses scale but have unpredictable latency; Cube
Store's pre-aggregations bridge the two by condensing source data by orders of magnitude for
repeat query patterns.

### 4. Semantic-layer skepticism (the counterweight)
Simon Späti (@sspaeti) reframes the whole category as old wine in a new label: *"I'm discovering
the parallels between MVC and a 'modern' Semantic Layer… Model = raw data & metrics/business
logic; View = BI tool; Controller = Semantic Layer."*
(x.com/sspaeti/status/1848773811753107735) — the tell that the "new" abstraction is a 1970s
architecture pattern renamed for a new market. Hold both readings at once: the *problem* (one
governed metric definition) is real and worth solving; the *category framing* as something
architecturally novel is contested. Don't oversell a semantic layer as more than "MVC's
Controller, for metrics."

### 5. Code-first vs. GUI-first BI — the real dividing line
The dividing line that matters is **git-versioned config**, not open-source-vs-proprietary — a
tool can be fully OSS and still be GUI-first:
- **Code-first:** Evidence.dev (MIT) renders a BI website directly from markdown files containing
  SQL — fully git-versionable, requires SQL. Lightdash connects to a dbt project via
  git/dbt-Cloud-API/CLI and reuses dbt-defined metrics rather than duplicating them, auto-syncing
  on commit. Rill (Apache-2.0) is agent-first BI-as-code: a 3-file model
  (`models/*.yaml` → `metrics/*.yaml` → `dashboards/*.yaml`) deployed via `rill deploy` git-backed
  CI/CD, and `rill init` scaffolds agent-context files (`.claude`, `.mcp.json`) plus an MCP server
  — designed for coding-agent-authored dashboards.
- **GUI-first, despite being open-source:** Metabase (AGPL-3.0 + separately-licensed
  embedded/enterprise editions) is the easiest-to-use OSS BI tool, but has no git-versioning or
  dashboards-as-code capability. Apache Superset (Apache-2.0) stores config in a metadata database
  by default — open license, still config-in-DB rather than config-in-git.

Pick code-first when the metric/dashboard definition needs to be reviewable, diffable, and
CI-testable like any other code; pick GUI-first only when the audience is non-technical
self-serve exploration and drift risk is acceptable.

### 6. dbt's 4-tier NL-question fallback
`dbt-labs/dbt-agent-skills`' `answering-natural-language-questions-with-dbt` skill encodes a
tool-agnostic progressive-degradation pattern for answering a natural-language question against a
dbt project, worth reusing as a general shape beyond dbt itself:
**semantic layer → modified compiled SQL → model discovery → manifest/catalog.** Try the governed
semantic layer first (if one exists); fall back to a compiled-SQL model with light modification;
fall back further to discovering the right model by name/description; fall back last to the raw
manifest/catalog metadata. Each tier trades governance/correctness guarantees for reach — prefer
the highest tier that actually answers the question rather than defaulting to raw manifest search.

### 7. Dashboards are for historians — decision-first serving
Two independent critiques converge on the same failure mode: BI optimized for *insight* when the
job is *decisions*. On X: *"Your analytics dashboard is a graveyard of things that already
happened. You look at it once a week. You see a bounce rate spike. You have no idea why. So you do
nothing… Dashboards are for historians. Not builders."*
(@anujcodes_21, x.com/anujcodes_21/status/2057106666127274100). Benn Stancil's *"Is BI dead?"*
(benn.substack.com/p/is-bi-dead) makes the structural version of the same claim: the old
end-to-end BI stack "is already dead"; BI should **shrink to consumption-only** but **expand to
cover every consumption type** — not just dashboards, but wherever a decision actually gets made
(Slack, a PR, an agent's context). **Contested** — treat as a serving-design prompt ("where does
the decision actually happen, and is that where the metric shows up?"), not as license to stop
building dashboards; but weigh it before defaulting to "add another dashboard" as the answer to a
reporting request.

## Validation
- A metric in dispute has exactly one governed definition, not per-tool re-derivations.
- The metric type (simple/ratio/derived/cumulative) matches the MetricFlow table's shape, not a
  freeform expression that happens to work.
- A semantic-layer recommendation names both the drift-prevention benefit and the MVC-reframe
  counterweight — not sold as unambiguously novel.
- A BI-tool recommendation states the git-versioned-config line explicitly, not just
  "open-source vs. proprietary."
- A reporting request is checked against "does this need a dashboard, or does the decision happen
  somewhere else" before defaulting to a new dashboard.

## Failure modes and handoff
- **The ask is actually about the marts underneath the metric** — → `transformation-and-modeling.md`.
- **The ask is data quality on the source tables, not the metric layer** — →
  `quality-and-testing.md`.
- **The ask is an experiment metric's statistical validity**, not a reporting metric's definition
  — → `experiment-measurement-foundations.md`.
- **The ask is warehouse cost/licensing for the BI tool's backing store** — →
  `platform-and-cost.md`.
