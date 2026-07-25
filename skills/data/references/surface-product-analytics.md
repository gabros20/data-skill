# Surface: Product analytics

Purpose: Reshape the 10 primary-job refs for a product that emits its own usage events — tracking
plan first, identity resolution across anonymous/known users, product metrics/funnels/activation as
the primary artifact, and a self-serve small-data-first query surface. **Default surface** when no
other shape is named.

Read when:
- The request is instrumenting a product/feature for usage analysis — "add analytics," "tracking
  plan," "funnel," "activation metric," "which users did X," "event schema for the app."
- No other surface is named and the system is a product generating its own behavioral data — start
  here by default.

Skip when:
- The request is a central analytical platform serving many teams/sources beyond one product's
  events — → [surface-warehouse-platform.md](surface-warehouse-platform.md).
- Data flows back into an operational system (CRM sync, triggering a workflow, near-real-time
  activation) — → [surface-operational-activation.md](surface-operational-activation.md).
- The product IS an AI feature and the ask is about the RAG corpus, LLM traces, or eval datasets —
  → [surface-ai-data-plane.md](surface-ai-data-plane.md).
- GDPR/CCPA/HIPAA/SOC2 applies — stack the additive [surface-regulated.md](surface-regulated.md) on
  top of this surface, it doesn't replace it.

Inputs:
- The product/feature and the decision the metric informs (activation, retention, funnel step,
  feature adoption) — state it before naming events; a metric with no decision behind it is a
  vanity number.
- Whether a tracking plan already exists, or this is greenfield instrumentation.
- Approximate data scale — defaults to the small-data engine (§4) unless a genuine distributed-scale
  need is stated.

Produces:
- A tracking plan: event names in one naming grammar, reserved properties respected, entities not
  duplicated per-event.
- An identity-resolution policy: what merges an anonymous session into a known user, and what
  happens to pre-merge events.
- The specific product metric/funnel definitions, each with one authoritative computation.
- The query-serving default: self-serve SQL over a small-data engine, escalated only on measured
  need.

## Contents
- Job impact table
- Tracking plan first, one naming grammar
- Identity resolution
- Self-serve, small-data-first by default

## Procedure

### 1. Job impact table
| Job | How this surface bends it |
|---|---|
| [event-taxonomy-and-tracking.md](event-taxonomy-and-tracking.md) | **The spine.** The tracking plan is the primary artifact of this surface; every other row bends around it. |
| [data-contracts-and-schema-evolution.md](data-contracts-and-schema-evolution.md) | The tracking plan *is* the contract — new properties are additive by default (don't silently break a dashboard reading an old shape); a renamed/removed event is a breaking change and follows the same compatibility discipline as any producer/consumer contract. |
| [ingestion-and-movement.md](ingestion-and-movement.md) | Mostly append-only event-stream landing — no CDC, no reverse sync. The one non-trivial movement step is identity-merge (§3), not connector plumbing. |
| [transformation-and-modeling.md](transformation-and-modeling.md) | Staging = one raw event table, 1:1, no joins. Marts are built around the object the metric needs (user, session, funnel-step) — narrow staging, wide marts, same as any surface. |
| [metrics-semantics-and-bi.md](metrics-semantics-and-bi.md) | Product metrics (activation rate, funnel conversion, retention curve) get exactly one definition in the semantic layer — the same "one metric, one number" discipline as warehouse-platform, just a smaller catalog. Default serving is self-serve SQL over the small-data engine (§4), not a BI-tool-first posture. |
| [quality-and-testing.md](quality-and-testing.md) | PK `unique`+`not_null` on `event_id` is the floor. Validate at ingestion and quarantine violations — never silently drop or silently accept a malformed event. |
| [lineage-and-observability.md](lineage-and-observability.md) | Lineage is shallow (event → mart), so the sharper observability signal is **volume**: a sudden drop in an event's row count is usually broken instrumentation, not a real behavior change — alert on it. |
| [governance-pii-and-retention.md](governance-pii-and-retention.md) | Consent-aware collection is the primary governance concern at this surface (§2); masking obligations are lighter than warehouse-platform's unless the event stream itself carries PII fields. |
| [experiment-measurement-foundations.md](experiment-measurement-foundations.md) | Product experiments run over this same event store. SRM check before any result is trusted — see that reference; this surface doesn't relax it. |
| [platform-and-cost.md](platform-and-cost.md) | Small-data-first is the default engine choice here (§4) — most product-analytics workloads never justify a distributed warehouse. |

### 2. Tracking plan first, one naming grammar
Pick a single event-naming grammar and enforce it — inconsistent casing/tense is the most common
cause of fragmented analytics. The convergent pattern across Segment, Amplitude, Mixpanel, PostHog,
and mParticle is **Object-Action**: object first, past-tense action, one casing throughout ("Form
Submitted," not "submit_form" mixed with "FormSubmit") — namespace with `Category:Object_Action`
(e.g. `app:account_created`) once the plan grows past a handful of domains. Segment's spec
distinguishes six call types (Identify, Track, Page, Screen, Group, Alias); every call shares
`event`, `properties`, `traits`, `context`, `timestamp`, `anonymousId`, `userId`. Never overload a
reserved property (`revenue`, `currency`, `value`) with ad-hoc meaning — destinations and downstream
pipelines depend on its declared semantics. Snowplow's entity-first alternative is the sharper design
principle once a plan grows large: information relevant to multiple events belongs in a reusable
entity, not duplicated as a property on every event; entity and event shape are versioned separately
(SchemaVer) and composed at send-time. Whichever grammar is chosen, validate incoming events against
the plan at ingestion and quarantine violations — never silently drop or silently accept an event
that doesn't match.

### 3. Identity resolution
Every product-analytics stream starts anonymous and merges into a known identity mid-session. State
the merge rule explicitly rather than assuming a library's default: an Identify call binds the
current `anonymousId` to a `userId`; events issued before the merge stay attributed to the anonymous
ID unless the mart layer explicitly re-attributes them post-merge — a modeling decision, not a given.
Get this wrong and funnels undercount: a user who signs up mid-funnel appears to abandon under the
anonymous ID and start fresh under the known one.

### 4. Self-serve, small-data-first by default
Most product-analytics workloads fit a single node. The COST result (a distributed system must beat
one laptop thread before its complexity is justified — Frank McSherry, 2015) and Jordan Tigani's
"Big Data is Dead" argument (most orgs query at a scale a laptop handles) both apply directly here:
default to a small-data engine (DuckDB/Postgres) and self-serve SQL over the tracking-plan-shaped
tables, and escalate to a distributed warehouse only against a measured bottleneck — not because
"product analytics" sounds like a big-data problem by name. See
[platform-and-cost.md](platform-and-cost.md) for the full decision.

## Validation
- Every event name follows the chosen grammar; reserved properties carry only their declared
  meaning.
- The identity-merge rule is stated, not assumed — pre-merge event attribution is a named decision.
- Every product metric/funnel has exactly one definition, referenced by name, not recomputed ad hoc
  per dashboard.
- Ingestion validates against the tracking plan and quarantines violations.
- The engine choice (small-data default vs. distributed) is justified against a measured need, not
  assumed by product scale alone.

## Failure modes and handoff
- **Events shipped with no naming grammar enforced** — casing/tense drift compounds into a
  fragmented plan within weeks; fix at the source, not with post-hoc dashboard patches.
- **Identity-merge rule left implicit** — funnels silently undercount at exactly the step where
  anonymous users convert; state the rule (§3) before trusting a funnel number.
- **A distributed warehouse stood up before the small-data default was measured to fail** — see
  [platform-and-cost.md](platform-and-cost.md); this is the single most common overbuild at this
  surface.
- **The real ask is a central platform serving multiple products/teams** — →
  [surface-warehouse-platform.md](surface-warehouse-platform.md); this surface stops at one product's
  own event stream.
- **The real ask is syncing this data back into an operational tool** — →
  [surface-operational-activation.md](surface-operational-activation.md).
