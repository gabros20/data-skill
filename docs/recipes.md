# Recipes

Use the invocation form supported by your client. The examples below use Codex's explicit `$data`
form.

## Design a tracking plan and event contract

Use when: a product needs event tracking designed or repaired before instrumentation starts, and you
want a naming grammar and a contract, not a spreadsheet of ad-hoc event names.

```text
Use $data to design a tracking plan and event contract for our new checkout flow — naming grammar,
entities, and what gets validated at ingestion.
```

Route: primary job `event-taxonomy-and-tracking` (one enforced naming grammar, entity-first modeling,
validate-at-ingestion/quarantine, consent gates collection) + `data-contracts-and-schema-evolution`
(the ODCS contract shape and compatibility mode) + surface `surface-product-analytics` (tracking plan
first, identity resolution). Retrieval-first: re-verifies current CDP/warehouse ingestion limits
before generating the schema.

## Add data quality tests and anomaly baselines to a dbt project

Use when: a dbt project has no tests, or only `not_null`/`unique`, and a bad load has shipped to a
dashboard before anyone noticed.

```text
Use $data to add data quality tests and anomaly baselines to our dbt project — cover the primary-key
floor, then layer in checks for what rule tests can't catch.
```

Route: primary job `quality-and-testing` (flagship) — the convergent DQ taxonomy (9 core buckets +
PR-diff + PII), known-unknowns via rule tests vs unknown-unknowns via statistical/anomaly baselines,
the non-negotiable PK `unique` + `not_null` floor, and a blast-radius diff reviewed before merge, not
after.

## Stand up the small-data-first analytics stack for a startup

Use when: a startup is choosing a warehouse/orchestrator for the first time and is at risk of
over-building for traffic it doesn't have.

```text
Use $data to help us pick a warehouse, orchestrator, and modeling layer for a startup on a tight
budget — start as simple as the data actually requires.
```

Route: primary job `platform-and-cost` (small-data-first / COST rule — a distributed system must beat
one laptop thread before its complexity is justified, workload-shape-before-pricing, the
task-vs-asset-vs-flow orchestrator choice) + `transformation-and-modeling` (narrow-the-DAG staging,
materialization ladder climbed only on measured bottlenecks) + surface `surface-warehouse-platform`.
Every price and license fact is date-stamped with a re-verify pointer.

## Build the governed RAG-corpus pipeline for an AI product

Use when: an AI product's retrieval index or LLM-trace warehouse needs to be built and governed as a
real dataset, not assembled ad hoc alongside the model code.

```text
Use $data to build the governed ETL for our RAG corpus and an LLM-trace warehouse for our AI
product — point-in-time correct, with eval datasets treated as governed data.
```

Route: surface `surface-ai-data-plane` (RAG corpus as governed ETL, LLM-trace/token warehousing,
point-in-time correctness, eval datasets as governed data) + primary job
`data-contracts-and-schema-evolution` for the corpus contract + `quality-and-testing` for its trust
gate. This is the seam with the `ai` skill: `data` governs the corpus and trace warehouse as a
dataset; `ai` owns the retrieval *behavior* and model evals that consume them — recommend `ai` rather
than reimplementing its side.

## Make experiment measurement trustworthy

Use when: an experiment result looks surprising and nobody has checked whether the measurement itself
can be trusted before acting on it.

```text
Use $data to check our experiment measurement before we trust the result — assignment, SRM, and
whether we're peeking safely.
```

Route: primary job `experiment-measurement-foundations` — SRM-before-trust as the gate (a surprising
result is investigated before it's believed, per Twyman's Law), deterministic salted assignment with
namespaces, CUPED variance reduction where it applies (and where it doesn't), and peek-safe
sequential testing (confidence sequences/mSPRT) instead of naive continuous monitoring. `data`
guarantees the measurement is valid; designing or interpreting the experiment itself routes to
`growth`.
