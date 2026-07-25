# Lineage and Observability

Purpose: Give a data platform a standard for **where data came from, what depends on it, and
whether it's currently healthy** — the OpenLineage wire model, the five-pillars/incident-response
program built on top of it, and the blast-radius workflow that turns lineage into a pre-change
safety check rather than a diagram nobody consults.

Read when:
- Nothing tracks dataset provenance/dependencies, an incident needs root-causing across a pipeline,
  a catalog choice (Marquez/DataHub/OpenMetadata) is undecided, or a change needs a downstream-impact
  check before it ships.
- Existing lineage is a static diagram that goes stale, not a live graph fed by running jobs.

Skip when:
- The ask is which checks to run and where — → [quality-and-testing.md](quality-and-testing.md);
  this ref owns the lineage/observability standard those checks report *into*, not the taxonomy of
  checks itself (don't duplicate the DQ taxonomy here — cross-reference it).
- The ask is the contract/schema-evolution rule for a producer↔consumer relationship — →
  `data-contracts-and-schema-evolution.md`.
- The ask is catalog-level PII classification/access control — → `governance-pii-and-retention.md`.

Inputs:
- Which systems already emit lineage events (Airflow/Spark/dbt/Dagster integrations exist for
  OpenLineage) vs need instrumenting from scratch.
- The catalog decision context: does the team want a catalog that *runs* tests, or one that
  *ingests* results from GX/Soda/dbt elsewhere.
- The specific asset and change under review, when the ask is a blast-radius check.

Produces: an OpenLineage-conformant event stream (or a stated gap), a catalog choice with its
auth/trust-boundary caveat, a five-pillars observability program wired to an incident loop, and (for
a proposed change) a depth-classified downstream-impact table with explicit traversal bounds.

## Contents
- OpenLineage — the wire model
- The DQ-assertion wire format
- Five pillars + the data-downtime incident loop
- Blast-radius / impact-analysis workflow
- Catalog choice: runs-own-tests vs ingests-results
- Contested: "database lineage is not data lineage"

## Procedure

### 1. OpenLineage — the wire model (verbatim mechanics)
The spec's core objects: **Run** (one execution, `UUID` RFC-4122 mandatory), **Job** (the process,
unique `name` within a `namespace`), **Dataset** (unique `name` within a datasource namespace, e.g.
`db.host.database.schema.table`), and **Facet** (atomic named metadata attached to any of the above).
Lifecycle: a run emits a sequence of **RunEvents** — at minimum one `START` and one terminal event
(`COMPLETE`/`FAIL`/`ABORT`); events are additive, not overwritten.

**The classic integration bug:** re-emitting a facet with the same name **replaces** the previous
value, it does not merge with it. An integration that emits a partial facet on each event and expects
the receiver to accumulate fields across events will silently lose data. Every facet emission must
carry its complete current state.

**Custom facet naming:** type `{prefix}{Name}{Entity}Facet` (PascalCase), key `{prefix}_{name}`
(camelCase). `_schemaURL` must be **git-sha-pinned** — pointing at a branch HEAD makes the schema
reference mutable, which breaks reproducibility for anyone validating an old event against it later.

### 2. The DQ-assertion wire format — the convergence bridge
`DataQualityAssertionsDatasetFacet` is where check results from any tool in
[quality-and-testing.md](quality-and-testing.md)'s taxonomy land on the wire: an array of
`{assertion, success, column, severity, name, expected, actual, …}`, where `assertion` free-text-
classifies the check kind (`not_null`/`unique`/`row_count`/`freshness`/`custom_sql`) and `severity` is
`error` (blocking) vs `warn`. A companion `DataQualityMetricsInputDatasetFacet` carries the raw
metrics (row/byte/null/distinct/avg/min/max/quantiles) a check was evaluated against. Treat this pair
as the target shape for wiring GX/Soda/Elementary/dbt-test results into lineage — the taxonomy itself
lives in the other reference, this is just the transport.

### 3. Five pillars + the data-downtime incident loop
Monte Carlo's five-pillar framing, now the industry-common shape: **Freshness** (update cadence),
**Distribution/Quality** (null rates, value ranges), **Volume** (row-count completeness), **Schema**
(structural change), **Lineage** (upstream/downstream metadata). "Data downtime" names the failure
state — data that's partial, erroneous, missing, or otherwise inaccurate — and the response is an
explicit loop borrowed from SRE: **detect → alert → investigate root cause → recommend/apply fix**.
Lineage is what makes root-causing tractable: without it, "investigate" means manually tracing SQL by
hand across every hop.

### 4. Blast-radius / impact-analysis workflow — vendor-neutralized
Before changing an asset (a schema, a column, a model), don't ship blind — enumerate what depends on
it first. The workflow (generalized from Google's BigQuery asset-impact-analysis pattern):
1. Resolve the asset to its fully-qualified name (FQN) in the lineage graph — not a display name,
   which can collide across schemas/projects.
2. Traverse downstream from that FQN with an explicit, bounded depth or hop limit — an unbounded
   traversal on a real graph either times out or buries the signal in noise.
3. Classify the resulting impact by depth tier (direct consumers vs N-hop transitive consumers) into
   a table, rather than a single flat list — direct breakage and eventual/transitive breakage carry
   different urgency.
4. **Disclose the traversal bounds transparently** in the output — state how deep the search went and
   whether it was cut off, rather than presenting a partial graph as if it were complete. A
   silently-incomplete impact table is worse than none, because it reads as reassurance.

### 5. Catalog choice: runs-own-tests vs ingests-results
- **Marquez** — the reference OpenLineage implementation (DB + API + UI consuming the spec directly).
  **Gotcha:** no default auth/authz on the HTTP API — a self-hosting decision, not a library default,
  that needs to be closed before exposing it beyond a trusted network.
- **DataHub** — an aspect-based metadata platform; each entity carries independently-versioned
  aspects (`schemaMetadata`, `datasetProfile`, `upstreamLineage`, `testResults`, `incidentsSummary`,
  …) and a native `AssertionType` enum (`DATASET`/`FRESHNESS`/`VOLUME`/`SQL`/`FIELD`/`DATA_SCHEMA`/
  `CUSTOM`) that mirrors OpenLineage's assertion classification — it's an **ingestion** target for
  results computed elsewhere (GX/Soda/dbt/Deequ), not a check engine itself.
- **OpenMetadata** — the one catalog here that **runs its own tests natively**
  (`TestSuite → TestCase → TestDefinition`, including a built-in `tableDiff` regression check) rather
  than only ingesting external results. Choose it over DataHub specifically when the requirement is
  "the catalog itself executes checks," not just displays them.

The decision axis: pick a catalog that *runs* tests when the team wants one system of record for both
metadata and execution; pick one that *ingests* results when checks already run elsewhere (dbt CI,
GX, Soda) and the catalog's job is to surface and correlate them.

### 6. Contested: "database lineage is not data lineage"
Chad Sanderson, "The Shift Left Manifesto v2" (dataproducts.substack.com, 2026-07-09) — **[contested,
date-stamped]**: *"Database lineage is not data lineage… The provenance of a value… mostly does not
live in the database. It lives in the code that produced it."* The claim: table-to-table lineage
(what OpenLineage/Marquez/DataHub capture) shows *data movement*, not the business logic and
application code that actually determined a value — real provenance requires tracing into the
application layer, which none of the cataloging tools above do natively. Teach this as a real
limitation of warehouse-level lineage, not a reason to skip it — it argues for supplementing catalog
lineage with code-level tracing on high-stakes values, not for abandoning the catalog.

## Validation
- Every emitted run has at least one `START` and one terminal event; facets are re-emitted with full
  state, never partial (checked against the replace-not-merge gotcha in §1).
- `_schemaURL` references are git-sha-pinned, not branch-HEAD.
- A DQ-assertion facet exists for check results the team wants visible in lineage, distinct from the
  check taxonomy itself (owned by `quality-and-testing.md`).
- The chosen catalog's fit (runs-tests vs ingests-results) is stated, and its auth posture (Marquez:
  no default auth) is closed before external exposure.
- Any blast-radius report states its traversal depth/bounds explicitly, never presents a partial
  graph as complete.

## Failure modes and handoff
- **A facet update silently lost fields** — re-check for partial-facet emission (§1); the fix is
  emitting complete state per facet, not merging on the consumer side.
- **Root-causing an incident takes manual SQL archaeology** — lineage isn't wired up yet; instrument
  the pipeline's OpenLineage integration before the next incident, not during one.
- **The ask is "what should we test," not "how do we track what depends on what"** — →
  `quality-and-testing.md`.
- **The ask needs true code-level provenance, not table lineage** — flag §6's limitation explicitly;
  this reference's tooling won't answer it alone.
