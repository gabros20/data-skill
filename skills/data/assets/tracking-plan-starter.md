# Tracking Plan Starter (fillable)

Source: [event-taxonomy-and-tracking.md](../references/event-taxonomy-and-tracking.md). Fill in
brackets; delete rows that don't apply. Run the QA checklist before shipping instrumentation, not
after.

## 1. Naming grammar decision (pick once, org-wide, then enforce)

| Dimension | Options | Decision |
|---|---|---|
| Word order | `[object][action]` / `[action][object]` / `[context][object][action]` | |
| Tense | past (`Order Completed`) / present | past, held consistent |
| Casing | snake_case / camelCase / Title Case / lower case — pick exactly one | |
| Namespace (at scale) | `Category:Object_Action` (e.g. `app:account_created`) — origin/domain prefix | |
| Shared-data rule | if a property is relevant to >1 event, model it as an **entity/trait**, not a repeated per-event property | entity-first |

- **Object-Action grammar** (Segment/Amplitude/Mixpanel/PostHog/Snowplow all converge here): pair
  an object (`Button`, `Form`, `Order`) with a past-tense action (`Clicked`, `Submitted`,
  `Completed`) → `Form Submitted`. Object first, then action; one casing, never mixed.
- Never let an event property and a user property share the same name — they represent different
  semantic states; conflating them produces misleading analytics.
- Lock the vocabulary: pick one canonical term per concept (e.g. always "game," never "match") and
  enforce it — don't let synonyms fragment the taxonomy.

## 2. Starter tracking-plan table

| Event | Properties | Type | Owner | Status |
|---|---|---|---|---|
| `Signup Completed` | `method` (string: email/oauth), `plan` (string) | string/enum | | draft / approved / deprecated |
| `Checkout Completed` | `revenue` (number, reserved), `currency` (ISO 4217, reserved), `order_id` | reserved + custom | | |
| | | | | |

- **Reserved properties are a contract, not a naming suggestion.** `revenue` (decimal $),
  `currency` (ISO 4217, defaults USD), `value` (abstract non-monetary magnitude, e.g. signups) —
  only use these for their intended meaning; destinations auto-map them to vendor-specific fields.
  `userId` auto-populates from a prior Identify call in browser/mobile libs; `anonymousId` is the
  pseudo-unique fallback before identification.
- Event names must be human-readable, not generic ("Video Recorded" — not "Event 12").
- `Status` drives enforcement: `draft` events may fire in staging only; `deprecated` events should
  raise a lint warning on new call sites, not silently keep collecting.

## 3. Schema evolution rule (per event/entity)

Adopt `MODEL-REVISION-ADDITION` versioning (Snowplow SchemaVer; re-verify the current term
against docs.snowplow.io before quoting as vendor vocabulary): a **breaking**
change bumps MODEL (old consumers keep working against the old MODEL); an **additive-compatible**
change bumps REVISION/ADDITION only. State per event:

| Event/entity | Current version | Last bump reason | Bump type |
|---|---|---|---|
| | | | MODEL / REVISION / ADDITION |

## 4. Consent gating (if any regulated surface applies — see `references/surface-regulated.md`)

- [ ] Consent default state set **before** the first measurement call fires (`gtag('consent',
      'default', {...})`), then updated after banner interaction.
- [ ] "Denied" is tested, not assumed — verify no ad-cookie writes AND check what still transmits
      (full page URL can still reach the collector even when `ad_storage` is denied; that's
      expected, not a bug — document it, don't hide it).
- [ ] IP-anonymization-at-capture is the default where required by the applicable regime.

## 5. Instrumentation QA checklist (run before this plan ships)

| # | Check | Pass/Fail |
|---|---|---|
| Q1 | Validated at ingestion against this plan — non-conforming events quarantined, never silently dropped or silently accepted | |
| Q2 | No two events differ only by casing/tense (near-duplicate check) | |
| Q3 | No "rogue" events firing that aren't in this plan | |
| Q4 | Every reserved property audited for correct usage (§2) | |
| Q5 | Every event/entity has a stated schema version (§3) | |
| Q6 | Consent gating verified end-to-end if a regulated surface applies (§4) | |

**Self-audit count: ____ / 6 QA checks passing.** Anything failing must be fixed before this plan
is treated as the source of truth for analysts/BI — untested instrumentation is untrusted data.
