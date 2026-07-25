# Event Taxonomy & Tracking

Purpose: Pick a naming grammar and tracking-plan structure that stays legible at scale, wire in
instrumentation QA that catches drift before it reaches the warehouse, and gate collection on
consent — so the very first row written is one `data-contracts-and-schema-evolution.md` can trust.

Read when:
- Designing or auditing event names/properties, a tracking plan, or an instrumentation spec.
- Debugging inconsistent event names, duplicate near-identical events, or a property that means
  two different things in two places.
- Wiring consent (GDPR/CCPA/Consent Mode) into what fires and what doesn't.

Skip when:
- The schema/version/compatibility-mode decision for a dataset that already has a producer and
  consumer — that's `data-contracts-and-schema-evolution.md` (this ref owns event *naming and
  capture*, that one owns the *registry contract* once data is in motion).
- The ask is which sync strategy moves the captured event into the warehouse — →
  `ingestion-and-movement.md`.
- The ask is whether existing data is trustworthy on an ongoing basis (tests, anomaly detection) —
  → `quality-and-testing.md`; this ref's "instrumentation QA" is pre-ingestion validation only.
- PII classification/masking policy for event properties — → `governance-pii-and-retention.md`.

Inputs:
- Existing event/property names (if any) — audit for casing drift, duplicate events, and
  property/trait name collisions before proposing new ones.
- Vendor/tool in use (Segment, Snowplow, PostHog, Avo, GTM) — decides which spec/QA tooling
  applies literally vs. by analogy.
- Regulatory scope (GDPR/CCPA/neither) — decides whether consent must gate collection itself.

Produces: a naming convention (grammar + casing + tense, locked), a tracking-plan shape (event ×
properties × entities), an instrumentation-QA mechanism (validate-and-quarantine, not silent
accept), and a consent gate wired before the first collection call where required.

## Contents
- Naming grammar — pick once, never mix
- Tracking-plan shape
- Event-level schema evolution
- Instrumentation QA — validate at ingestion
- Client-side vs. server-side capture
- Consent-aware collection

## Procedure

### 1. Naming grammar — pick once, never mix
**Object-Action framework** — object first (Button, Form, Order), then a past-tense action
(Clicked, Submitted, Completed) → `Form Submitted`. Five major vendors converge on this shape
independently (Segment, Amplitude, Mixpanel, PostHog, mParticle) — treat it as the default, not
one vendor's opinion (growthmethod.com/object-action-framework/, cross-checked against Segment/Avo
primary docs, 2026-07-25). At scale, namespace by origin: `Category:Object_Action` (e.g.
`app:account_created`).

Four independently-locked dimensions (Avo's decomposition — avo.app/docs/data-design/best-
practices/naming-conventions, 2026-07-25):
1. **Casing** — snake_case / camelCase / Title Case / lower case. Pick one, enforce globally.
2. **Word order** — `[object][action]`, `[action][object]`, or `[context][object][action]`.
3. **Tense** — past vs. present, held consistent across every event.
4. **Vocabulary** — canonical terms locked (always "game," never "match").

**Never let an event property and a user (trait) property share the same name** — they represent
different semantic states; conflating them produces "dangerously misleading" analytics (Avo, same
source). This is the single highest-leverage naming bug to audit for.

### 2. Tracking-plan shape
**Segment's 6-call spec** — Identify (who), Track (what action), Page, Screen, Group, Alias
(identity merge) — all share `event`, `properties`, `traits`, `context`, `timestamp`, `messageId`,
`anonymousId`, `userId` (twilio.com/docs/segment/connections/spec, 2026-07-25). Reserved
properties on Track calls have fixed meaning — don't repurpose: `revenue` (decimal $), `currency`
(ISO 4217, default USD), `value` (abstract non-monetary magnitude, e.g. signups). Event names must
be human-readable ("Order Completed"), never generic ("Event 12").

**Entity-first design (Snowplow)** — "if a piece of information is likely relevant to multiple
different events, it belongs in an entity, not as a property" — attach reusable entities to events
rather than duplicating the same fields on every event schema (docs.snowplow.io/docs/fundamentals/
tracking-design-best-practice/, 2026-07-25). Contrast: PostHog's `$autocapture` + arbitrary JSON
props is capture-everything with no schema-first contract by default — pick schema-first when
downstream consumers need a contract, capture-everything for exploratory analytics.

Schema granularity is a real design decision, not a default: group related actions into one schema
with a `type` discriminator (needs strong governance) vs. one schema per action (cleaner
independent evolution, more schemas to manage).

### 3. Event-level schema evolution
Self-describing envelope (Snowplow/Iglu): `{vendor, name, format:"jsonschema", version:"MODEL-
REVISION-ADDITION"}`, e.g. `iglu:com.my_company/viewed_product/jsonschema/1-0-0`
(docs.snowplow.io/docs/fundamentals/schemas/, 2026-07-25). Naming = reverse-DNS vendor + snake_case
name; `additionalProperties:false` enforced — a strict contract, not an append-only blob.

**SchemaVer bump rule:**
| Bump | Trigger | Compatibility |
|---|---|---|
| MODEL (major) | Field type change, new **required** field, required↔optional flip | Breaking — old consumers keep working only on the old MODEL |
| REVISION (minor) | May invalidate *some* historical data | **Warehouse-dependent** — Redshift vs. Snowflake/BigQuery/Databricks differ on required→optional handling; verify per warehouse before treating as safe |
| ADDITION (patch) | New **optional** field, widened size limit, no type change | Fully backward-compatible |

The warehouse-dependent REVISION caveat is the gotcha most teams miss. For the producer/consumer
*registry contract* (Confluent compatibility modes, ODCS) once this event reaches a governed
dataset, see `data-contracts-and-schema-evolution.md`.

### 4. Instrumentation QA — validate at ingestion, never silently accept
Treat a tracking plan as enforceable, not documentation. **Segment Protocols** (Business-plan
addon only — not on Free/Team, event-stream + Engage sources only, twilio.com/docs/segment/
protocols, 2026-07-25) validates incoming events against the plan and can **block non-conforming
events, routing them to a quarantined source** rather than silently dropping or silently
accepting malformed data — the same validate-and-quarantine principle `quality-and-testing.md`
applies to warehouse-level checks, applied one layer upstream at capture time.

**Avo Inspector**: flags casing violations, "rogue" events absent from the plan, and near-duplicate
event names from spelling/casing drift. **Typewriter** (Segment): codegen'd strongly-typed tracking
functions from a JSON-Schema plan, catching type/shape drift at compile time — **[UNVERIFIED]**
maintenance status; confirm before recommending.

### 5. Client-side vs. server-side capture
Server-side (e.g. server-side GTM) gives the producer full control over data before it's routed
anywhere, enables cross-device measurement through one funnel, and must run on a first-party
domain in production mode before go-live (developers.google.com/tag-platform/tag-manager/server-
side/intro, 2026-07-25). Default to server-side when reshaping/filtering data before it reaches a
third-party destination; client-side stays simpler for a single destination, no reshaping need.

### 6. Consent-aware collection — gate before the first call
**Google Consent Mode**: four signals — `ad_storage`, `analytics_storage`, `ad_user_data`,
`ad_personalization`, each `granted`/`denied`. Set a `consent.default` state *before any
measurement call fires*, then `consent.update` after the banner interaction
(developers.google.com/tag-platform/security/guides/consent, 2026-07-25). **Headline gotcha:
"denied" is not "nothing transmitted"** — when `ad_storage` is denied, no new ad cookies are set,
but "data sent to Google will still include the full page URL"; test the denied path explicitly,
don't assume it's a no-op.

**GDPR**: consent required before any non-strictly-necessary cookie fires; strictly-necessary
cookies are exempt but still need purpose disclosure; withdrawal must be as easy as granting
(gdpr.eu/cookies/, 2026-07-25). **CCPA**: "notice at collection" shown at or before collection
(categories, purposes, privacy-policy link, "Do Not Sell or Share" link where applicable);
"sharing" explicitly includes cross-context behavioral advertising, not just monetary sale
(oag.ca.gov/privacy/ccpa, 2026-07-25). PII classification of properties is `governance-pii-and-
retention.md`'s job; this ref owns *whether the collection call fires at all*.

## Validation
- [ ] One casing, one word order, one tense — locked and documented, not left implicit per team.
- [ ] No event property shares a name with a user/trait property.
- [ ] Every event has a versioned schema (SchemaVer or equivalent) — not an unversioned JSON blob.
- [ ] Non-conforming events are validated and quarantined at ingestion, never silently dropped or
      silently accepted.
- [ ] Consent state is set before the first measurement call, and the "denied" path is tested, not
      assumed.

## Failure modes and handoff
- **The real ask is the registry-level contract (compatibility mode, ODCS) for data already in
  motion** — → `data-contracts-and-schema-evolution.md`.
- **The real ask is how the captured event reaches the warehouse** — → `ingestion-and-movement.md`.
- **The real ask is ongoing trust in already-landed data, not one-time capture QA** — →
  `quality-and-testing.md`.
- **PII fields need classification or masking policy** — → `governance-pii-and-retention.md`.
- **Vendor/tool specifics move fast** (Segment Protocols pricing tier, Typewriter maintenance
  status) — re-verify against current vendor docs before citing as current.
