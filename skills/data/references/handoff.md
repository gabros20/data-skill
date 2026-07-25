# Handoff

Purpose: Package a completed `data` artifact (tracking plan, data contract, transformation/DAG, DQ
suite, lineage/observability config, governance policy) so downstream skills can build against,
monitor, or independently verify it without re-deriving what was decided, plus a compact
`handoff.yaml` companion for pipeline routing.

Read when:
- Finishing a `data` build that `backend` or `frontend` is expected to emit events/writes against,
  `ai` is expected to ground retrieval in, `quality` is expected to independently verify, `operate`
  is expected to deploy/monitor, or `growth` is expected to design experiments on top of.
- Deciding whether this run should emit a machine-readable companion at all.

Skip when:
- The request is standalone with no downstream consumer in view ("design this tracking plan," "pick
  a sync strategy," "write this DQ check") — produce the artifact and stop; an unread `handoff.yaml`
  is clutter, not diligence.
- You need a specific job's own procedure — that's one of the 10 job references
  (`event-taxonomy-and-tracking.md`, `quality-and-testing.md`, etc.) or a surface ref. This reference
  covers what happens after the dataset/pipeline/contract is decided, not how to design it.

Inputs: the completed artifact (tracking plan, ODCS contract, dbt/SQLMesh project, DQ suite, lineage
config, governance policy), the upstream artifacts `data` built against when running inside a
pipeline (architecture's data-classification/NFR decisions, backend's operational-store schema,
ai's retrieval-config assumptions), and the decisions/assumptions/risks recorded while building.

Produces: a `handoff.yaml` companion beside the artifact, filled from
[assets/handoff.yaml](../assets/handoff.yaml) — never a duplicate of the tracking plan, contract
text, SQL, or DQ-suite code.

## Contents
- Standalone vs pipeline
- What `data` consumes
- What `data` hands downstream
- The handoff.yaml template
- Per-consumer needs
- The seam discipline
- Validation
- Failure modes

## Standalone vs pipeline

`data` runs in two modes, read from context, not a flag. **Standalone**: the request is
self-contained ("design this tracking plan," "write a DQ check for this table," "pick a sync
strategy") with no downstream work named — deliver the artifact and stop. **Pipeline**: the request
or an orchestrating controller names downstream work explicitly (`backend`/`frontend` emit against
this contract, `ai` grounds retrieval in this pipeline, `quality` verifies this independently,
`operate` monitors this), or a multi-skill run is already in motion — emit the `handoff.yaml`
companion. Default to standalone; only emit the companion when a downstream consumer is actually
expected to read it.

## What `data` consumes

When `architecture` has already run, `data` designs the pipeline/contract **inside** the decided
shape rather than re-deriving it:

| Artifact (from `architecture`) | What `data` takes from it |
|---|---|
| Solution-architecture doc | the system boundary the data plane serves — one product, a central platform, an AI product's corpus |
| NFR budgets | freshness/latency/cost targets the pipeline and DQ gates must meet |
| Data-classification decisions | which fields are PII/regulated, feeding directly into `governance-pii-and-retention.md` |
| Build-vs-buy decision on ingestion/warehouse | whether this is a from-scratch pipeline or wraps a managed ELT/warehouse service |

When `backend` or `ai` have already run (or run in parallel), `data` consumes their contracts rather
than inventing its own: the operational store's schema from `backend` (CDC reads from its
actually-implemented shape, not an assumed one), and retrieval/freshness assumptions from `ai` when
building the governed pipeline behind a RAG corpus. When none of these ran (fully standalone
`data`), say so explicitly rather than inventing a contract to fill the gap.

## What `data` hands downstream

`data` hands distinct artifacts to six distinct consumers — **do not conflate them.**

| Artifact | Primary consumer | What they take from it |
|---|---|---|
| Tracking plan / event contract (naming grammar, reserved properties, schema) | `frontend` / `backend` | what to instrument against — `data` defines the contract, the siblings emit events that satisfy it |
| Operational-store read contract (CDC schema, sync-strategy assumptions, freshness SLA) | `backend` | what its own schema changes must not silently break — `data` reads from it, `backend` owns it |
| Vector-store/dataset schema and freshness cadence (governed RAG pipeline) | `ai` | what the retrieval design can assume is current, and how it's kept that way |
| Data contracts, DQ suite, and lineage config | `quality` | the starting point for *independent* verification — `quality` re-derives its own adversarial data-quality/compliance pass, it does not simply re-run `data`'s own DQ suite and call that independent |
| Pipeline/warehouse config, freshness/cost telemetry hooks, retention-schedule enforcement points | `operate` | what to deploy and monitor — dashboards/alerts on freshness lag, DQ failure rate, and cost drift; `data` instruments the hooks, `operate` owns the dashboard/alert/deploy pipeline |
| Measurement-validity guarantees (SRM check, assignment mechanism, exposure logging) | `growth` | the floor an experiment design can trust — `data` guarantees the measurement is valid, `growth` designs and interprets the experiment itself |

Each consumer is a consumer, not a co-author: `data` ships the contract, pipeline, and DQ suite; it
does not write the UI instrumentation code, own the operational store, design the retrieval
behavior, run the independent compliance audit, own the monitoring dashboard, or design the
experiment.

## The handoff.yaml template

`assets/handoff.yaml` is the fillable schema. Field meanings follow the same seam discipline as the
sibling `architecture` and `ai` skills' handoffs: only `objective` is required; every other field may
be empty, and a consumer must tolerate it absent rather than treat absence as a schema violation.
`artifacts_created` entries use `type` values: `tracking-plan`, `data-contract`, `pipeline-config`,
`dq-suite`, `lineage-map`, `semantic-model`. `recommended_next` names the actual next skill(s) —
`frontend`, `backend`, `ai`, `quality`, `operate`, `growth` — never a vague "further work needed."

## Per-consumer needs

**`frontend`/`backend`** need the tracking plan or event contract in its exact naming/schema shape —
a contract described loosely produces events that don't match what the pipeline expects. **`ai`**
needs the vector-store/dataset schema and freshness cadence confirmed in its actually-implemented
shape when built in parallel — flag any deviation from what `data` specified, don't let it surface as
a silent retrieval-quality regression. **`quality`** needs the data contracts and DQ suite as a
*starting point*, not a substitute for independent verification — it re-derives its own adversarial
data-quality/compliance pass against the same pipeline. **`operate`** needs the freshness/cost
telemetry hooks and pipeline config to monitor against, not a re-explanation of the pipeline — a
pointer to what's instrumented and what it emits. **`growth`** needs to know the measurement-validity
guarantees already in place (SRM check wired, assignment deterministic and salted, exposure logged)
before it designs an experiment on top — a guarantee `data` didn't actually build shouldn't be
assumed by an experiment design.

## The seam discipline

The tracking plan, contract, pipeline, and DQ suite are the real deliverable. `handoff.yaml` is a
routing index into them, never a copy. If a consumer needs more than the index gives, the artifact
is incomplete or under-documented — the yaml should not grow to compensate. Only `objective` is
required; every downstream skill must tolerate every other field absent.

## Validation

Before emitting: `objective` is one sentence and non-empty; every path in `artifacts_created`
resolves to a real tracking plan/contract/pipeline/DQ-suite artifact; `assumptions` and `risks` trace
to entries recorded during the build, not invented at handoff time; every `unresolved` item names an
owner; `recommended_next` names an actual sibling skill, not a vague call for more work; if data
classification was consumed from `architecture`, the PII/regulated fields are flagged explicitly for
`quality`/`operate` rather than left implicit; every volatile fact (engine version, pricing, license,
compatibility-mode default) referenced from the artifact carries its date stamp and re-verify note,
per this skill's anti-staleness invariant — don't let a frozen version or price slip into the handoff
uncaveated.

## Failure modes

- **Duplicating the tracking plan, contract text, SQL, or DQ-suite code into handoff.yaml** — defeats
  its purpose as an index; keep it a pointer.
- **Emitting handoff.yaml in standalone mode** when nothing downstream will read it — skip it.
- **`status: complete` with a non-trivial `validation.remaining`** — use `partial` and say what's
  outstanding (e.g. "contract published, quality's independent DQ pass not yet run").
- **Handing `quality` the same DQ suite `data` already ran and calling that independent
  verification** — `quality`'s value is a fresh, adversarial pass, not a rerun.
- **A tracking plan or contract handed to `frontend`/`backend` without the exact schema** — sibling
  instrumentation will drift from what the pipeline expects; the contract must be exact, not
  descriptive.
- **`growth` assuming a measurement guarantee `data` never actually built** — state explicitly which
  of SRM/assignment/exposure-logging are in place before `growth` designs on top of them.
- **Silently invoking `frontend`, `backend`, `ai`, `quality`, `operate`, or `growth`** instead of
  naming it in `recommended_next` — the never-auto-invoke boundary rule from the main SKILL.md
  applies at the handoff seam too.
- **An `unresolved` item with no owner** — assign one or drop the item until it has one.
