# Experiment Measurement Foundations

**Scope guard:** this reference owns measurement **validity** — whether an experiment's
underlying data can be trusted at all: assignment integrity, sample ratio, variance reduction,
peeking safety. It does **not** own experiment **design** (what to test, how to size a rollout)
or **interpretation** (what a validated result means for a decision) — both belong to `growth`.
Decline and route to `growth` when the request is "should we run this test" or "what does this
lift mean for the roadmap"; use this ref when the request is "can this experiment's numbers be
trusted."

Purpose: Decide whether an experiment's underlying data is trustworthy before any result is
read — assignment correctness, SRM, exposure-vs-assignment population, variance-reduction
technique, and peek-safety of the monitoring method.

Read when:
- Building or auditing an experiment-assignment pipeline (deterministic bucketing,
  namespace/mutual-exclusion design).
- A result "looks interesting" and needs a validity check before anyone acts on it.
- Choosing a variance-reduction (CUPED) or sequential-monitoring approach for an
  experimentation platform.
- Someone wants to check results continuously ("peek") without invalidating the test.

Skip when:
- The ask is experiment **design** — what to test, hypothesis framing, rollout percentage,
  success-metric selection for a specific initiative — → `growth`.
- The ask is **interpreting** a validated result for a product/business decision — →
  `growth`/`product`; this ref stops at "the data is trustworthy," not "here's what to build."
- The ask is instrumenting the events an experiment reads (event names/properties) — →
  `event-taxonomy-and-tracking.md`.
- The ask is general anomaly detection unrelated to an experiment's assignment integrity — →
  `quality-and-testing.md`.

Inputs:
- Whether assignment is deterministic and reproducible (hash-based) or stored/mutable state.
- Intended vs observed traffic split per arm — required for an SRM check.
- Whether the metric can use a pre-period covariate — needed for CUPED.
- How often results will be checked before the planned end date — decides whether peek-safe
  monitoring is required.

Produces: an SRM check result (pass/fail) gating whether any other result from the experiment
is read; a stated assignment mechanism (deterministic salted hash + namespace) with an audit
trail; a variance-reduction and peek-safety method matched to the metric and monitoring
cadence; a guardrail-metric set reported alongside the primary/OEC metric.

## Contents
- SRM-before-trust
- Deterministic assignment: the PlanOut model
- Assignment vs exposure, and dilution correction
- CUPED
- Peek-safe sequential testing
- Twyman's Law and common pitfalls
- OEC + guardrails
- Warehouse-native experimentation

## Procedure

### 1. SRM-before-trust — the fever check
Sample Ratio Mismatch: the observed traffic split across arms deviates from the intended
ratio. Kohavi's framing (Trustworthy Online Controlled Experiments, exp-platform.com,
2026-07-25): SRM is "a symptom for a variety of data quality issues" — like a fever, it tells
you *something* is wrong without telling you *what*. **No result from an experiment should be
read or acted on until SRM has been checked and passed** — a failed SRM check invalidates every
downstream metric, no matter how clean the lift number looks.

Automate it: Vermeer, Anderson & Acebal's "Automated Sample Ratio Mismatch (SRM) Detection and
Analysis" (2022-07-06, lukasvermeer.nl) reflects consensus among experimentation practitioners
— *"SRM checks are relatively straightforward to automate and can be useful for data quality
monitoring even for companies who rely on third-party experimentation platforms."* Run it as a
standing check, not a one-off audit.

### 2. Deterministic assignment — the PlanOut model
Assignment integrity starts upstream of any metric: `facebookarchive/planout` (archived, but
the model is canonical) established the reference shape:
- **Hash-based, stateless assignment.** `UniformChoice`/`WeightedChoice` hash `(experiment +
  variable + unit_id)` → the same arm every time, **without persisting assignment state** in a
  lookup table that can drift or fall out of sync.
- **Namespaces** enforce mutual exclusion — a unit can't land in two conflicting experiments'
  arms simultaneously.
- **Self-logging.** Assignment objects log their own inputs, checksum, salt, and parameters —
  the audit trail is baked into the primitive, not bolted on after the fact.
Check any modern assignment system (in-house or vendor) against this shape: deterministic +
salted, namespace-exclusive, self-logging. A system that stores assignment in a mutable table
without a deterministic hash underneath it is a design smell — reassignment on retry, replay,
or backfill silently changes who's in which arm.

### 3. Assignment vs exposure/triggered analysis — and dilution correction
Distinguish **assignment** (a unit was bucketed into a variant) from **exposure** (the unit
actually experienced the change). Triggered/exposure-based analysis restricts measurement to
units that were actually exposed — the more sensitive analysis, but it requires **dilution
correction** to translate the effect back to the full assigned population; reporting a
triggered-only effect as if it applied to everyone assigned overstates the population-level
impact.

### 4. CUPED — variance reduction, with a non-applicability list
CUPED (Controlled-experiment Using Pre-Experiment Data): adjusted metric = post-exposure
average − θ × pre-exposure average, using a correlated pre-period covariate. Default lookback:
14 days. Reported effect sizes (GrowthBook docs, not independently verified — flag as
vendor-reported): Netflix (2016) ~40% variance reduction; Microsoft (2022) ~equivalent to 20%
more traffic. (docs.growthbook.io/statistics/cuped, 2026-07-25)

**Non-applicability list** — don't apply CUPED to: quantile metrics, legacy ratio metrics,
bandit goal metrics, Mixpanel-sourced metrics, custom user-value aggregations. Check the metric
type against this list before assuming variance reduction is free.

### 5. Peek-safe sequential testing — three vendors, one convergent pattern
Continuous monitoring without a stopping rule (the peeking problem) inflates false positives.
Three independent platforms converge on the same trade: **wider confidence intervals in
exchange for the ability to look whenever you want without invalidating the test.**

| Vendor | Method | Mechanism |
|---|---|---|
| GrowthBook | Asymptotic Confidence Sequences (Waudby-Smith 2023) | Tunable N* (default 5,000); intervals widen by design to stay valid at any look |
| Statsig | mSPRT (Zhao et al.) | Auto-adjusts p-values/CIs at every look |
| Eppo | Confidence-sequence guarantee | Guaranteed confidence level regardless of peek count, at the cost of wider intervals |

(docs.growthbook.io/statistics/sequential, docs.statsig.com/stats-engine/sequential-testing,
docs.geteppo.com/statistics/confidence-intervals/ — 2026-07-25)

The only other valid option is a **pre-registered fixed horizon**: commit to a sample
size/duration before the test starts and don't read the result early at all. Pick one of the
two — "peek occasionally without a sequential method and without a fixed horizon" is not a
valid third option, it's the failure mode both of these exist to prevent.

### 6. Twyman's Law
*"Any figure that looks interesting or different is usually wrong."* A dramatic lift is more
often a data-quality bug (SRM failure, mis-instrumented event, bot traffic) than a real effect
— investigate the surprising number before trusting it, especially when it would justify a
decision.

### 7. OEC + guardrail metrics
Pair the primary Overall Evaluation Criterion with **guardrail metrics that must not regress**
— a short-term OEC can be a bad proxy for long-term value on its own (a price increase lifts
short-term revenue while suppressing LTV). A result that improves the OEC while breaching a
guardrail is not a win; report both, don't let the headline metric hide the guardrail.

### 8. Common pitfalls checklist
- **Simpson's paradox during gradual rollouts** — aggregate lift can reverse sign relative to
  per-cohort lift when rollout percentage changes over time; segment by rollout cohort before
  trusting an aggregate number.
- **Bot/robot traffic** — inflates or skews arm populations without necessarily showing up as
  an SRM ratio problem; filter before assignment-integrity checks, not after.
- **Survivorship bias** — measuring only units that remained active/eligible through the full
  window silently excludes the units most affected by a change.
- **Multiple-comparison false positives** — checking many metrics at once inflates the chance
  one looks significant by chance; correct for the number of comparisons, not just the number
  of peeks.

### 9. Warehouse-native experimentation
GrowthBook queries the warehouse (BigQuery/Snowflake/Databricks) directly for metric
computation rather than ingesting a separate event stream, and ships SRM, sequential testing,
CUPED, Bayesian, post-stratification, and bandits as first-class stats-engine features —
confirming these aren't bespoke techniques, they're table stakes for a serious experimentation
platform. **License (open-core, verified 2026-07-25):** bulk MIT;
`packages/{back-end/src,front-end,shared/src}/enterprise` carve-out = GrowthBook Enterprise
License (commercial) — treat the enterprise-licensed directories as excluded from any
MIT-license assumption. (github.com/growthbook/growthbook)

For lighter-weight, library-only measurement-validity tooling without a full platform:
`spotify/confidence` (Apache-2.0) — Z-test/Welch's T-test/Chi-squared, CUPED-style variance
reduction, Bayesian BetaBinomial, Bonferroni correction, non-inferiority margins, sample-size/
power calculators.

## Validation
- SRM has been checked and passed before any other metric from the experiment is read or
  reported.
- Assignment is deterministic + salted-hash-based with a self-logging audit trail — not stored/
  mutable state alone.
- If exposure/triggered analysis is used, a dilution correction is applied before generalizing
  to the full assigned population.
- CUPED (if used) is checked against the non-applicability list for the metric type in
  question.
- Continuous monitoring uses either a named peek-safe sequential method or a pre-registered
  fixed horizon — never neither.
- A surprising/dramatic result triggers a Twyman's-Law investigation before being trusted.
- A guardrail-metric set is reported alongside the OEC, not the OEC alone.

## Failure modes and handoff
- **The real ask is experiment design or interpretation for a decision** — → `growth`; this ref
  stops at data validity.
- **The real ask is which events to instrument for the experiment to read** — →
  `event-taxonomy-and-tracking.md`.
- **SRM fails** — stop; do not report any other metric from the run until the
  assignment-integrity issue is found and fixed, and re-run the test if needed.
- **No stopping rule exists and the team wants to "just check periodically"** — that is the
  peeking problem; require either a named sequential method or a fixed horizon before
  continuing.
- **The metric needing variance reduction is on the CUPED non-applicability list** — don't
  apply CUPED; note it as a known limitation instead of forcing the technique.
