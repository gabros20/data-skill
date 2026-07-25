# Surface: Regulated (additive)

Purpose: **Additive overlay** — stacks on top of any of the four base surfaces
([product-analytics](surface-product-analytics.md), [warehouse-platform](surface-warehouse-platform.md),
[operational-activation](surface-operational-activation.md), [ai-data-plane](surface-ai-data-plane.md))
when GDPR, CCPA, HIPAA, or SOC2 applies. It hardens the governance posture of the base surface — it
does not replace the base surface's shape. A regulated product-analytics stack is still
`surface-product-analytics.md` + this ref, not this ref alone.

Read when:
- The request names a regulatory regime explicitly (GDPR, CCPA, HIPAA, SOC2) or the data plane
  handles personal data, health data, or data under a customer compliance commitment.
- An erasure/deletion request, a consent audit, or compliance-audit prep is the presenting ask.

Skip when:
- No regulatory regime or contractual compliance obligation applies — apply the base surface alone;
  don't add this overlay's governance overhead pre-emptively.
- The ask is PII classification/masking mechanics in general, with no regulatory driver named —
  that's [governance-pii-and-retention.md](governance-pii-and-retention.md) directly; this overlay is
  what changes when a *named regime* is in scope, not general governance hygiene.

Inputs:
- Which regime(s) apply (GDPR, CCPA, HIPAA, SOC2 — state all that apply; they impose different
  obligations, not one uniform one).
- Which base surface this stacks on — the overlay's meaning depends on it (an erasure request
  against a warehouse-platform's marts is a different operation than one against an ai-data-plane's
  vector index, § composition table).
- Whether a signed data-processing agreement or customer contract already specifies retention/PII
  handling terms — those terms are the floor, not a suggestion.

Produces:
- A verified consent-gating point at collection, before the base surface's first data-touching call.
- Platform-layer masking enforced (not application-layer), matched to the base surface's storage
  layer.
- An erasure procedure that checks exception grounds before deleting, implemented as tombstone +
  scheduled hard-delete.
- A stated retention schedule and an audit trail covering access and erasure actions.

## Contents
- Consent gating, verified not assumed
- Platform-layer masking is mandatory here
- Erasure: Art. 17 with exceptions, not "delete everything"
- Retention schedules and audit trails
- PII terms belong in the contract
- Composing with each base surface

## Procedure

### 1. Consent gating, verified not assumed
Consent must gate collection *before* the first data-touching call, not be checked after the fact.
Test the "denied" path explicitly — denial does not always mean zero data transmitted: Google's
Consent Mode, for example, still sends the full page URL when `ad_storage` is denied, just without
new ad cookies. Whatever consent framework the base surface uses, verify what actually crosses the
wire under a denial; don't assume the SDK's default behavior matches the compliance claim.
Strictly-necessary collection is exempt from prior consent under GDPR but still requires purpose
disclosure; under CCPA, a "notice at collection" must appear at or before collection, listing
categories and purposes, and "sharing" explicitly includes cross-context behavioral advertising, not
just monetary sale — don't read CCPA's bar as GDPR's stricter one.

### 2. Platform-layer masking is mandatory here
Enforce PII masking at the platform layer — warehouse tag-based masking policies, catalog policy
tags, or platform-native column/row filters — never application-layer redaction alone, which any
direct warehouse query bypasses. This is the base
[governance-pii-and-retention.md](governance-pii-and-retention.md) invariant; under a named regime
it stops being a best practice and becomes the control an auditor will actually check.
Classification should be automated and decoupled from the masking action itself (classify a column's
PII category first, then apply a masking policy as a separate step) so a new column gets covered by
an existing policy the moment it's classified, not only when someone remembers to add a rule.

### 3. Erasure: Art. 17 with exceptions, not "delete everything"
"Delete everything on request" is the wrong default. GDPR Article 17 lists six erasure grounds
against explicit Article 17(3) exceptions — free expression, legal/public-interest compliance,
public health, and archival/research/statistical purposes when erasure "would render impossible or
seriously impair" that purpose, among others. Check exception grounds before executing an erasure;
don't execute unconditionally. The warehouse-architecture answer to the tension between real-time
erasure and append-only/backup/snapshot storage is **tombstone + scheduled hard-delete** — mark the
record for deletion immediately (so no live query surfaces it), then physically purge it on the next
scheduled maintenance window, rather than mutating historical/backup tables in place. When personal
data was made public, the controller must take "reasonable steps, including technical measures" to
notify other processors holding copies — the operational root of cascading deletion across every
system holding a copy, not just the primary warehouse.

### 4. Retention schedules and audit trails
State an explicit retention schedule per dataset class (not "keep it as long as we might need it")
and enforce it the same way a scheduled hard-delete enforces erasure. Every access to regulated data
and every erasure action needs an audit trail — who/what accessed it, when, and what an erasure
request did and when it completed — because under SOC2/HIPAA, "we deleted it" without a
corresponding audit entry is not a defensible answer to an auditor.

### 5. PII terms belong in the contract
When data is exchanged between producer and consumer under a data contract
([data-contracts-and-schema-evolution.md](data-contracts-and-schema-evolution.md)), the PII
classification of each field and the retention/erasure terms that apply to it belong in the contract
itself, not in a separate policy document a consumer has to cross-reference. A schema-evolution
change that adds a new PII-bearing field should trip the same contract-compatibility discipline as
any breaking schema change.

## Composing with each base surface
| Base surface | What the overlay typically adds |
|---|---|
| [product-analytics](surface-product-analytics.md) | Consent-gated collection at the tracking-plan level (§1); an erasure request against a user's event history needs the tombstone + scheduled hard-delete pattern (§3) applied to the event store, including any identity-merge history. |
| [warehouse-platform](surface-warehouse-platform.md) | Platform-layer masking (§2) enforced centrally across every team's marts, not per-team; retention schedules (§4) stated per dataset class across the whole platform, since cross-team access is the norm here. |
| [operational-activation](surface-operational-activation.md) | Data classified (§2) *before* it leaves the governed perimeter via reverse-ETL into a third-party SaaS tool — the destination is now outside direct platform control, so the classification gate has to run pre-sync, not after. |
| [ai-data-plane](surface-ai-data-plane.md) | The regulated-RAG note applies directly: PII tagging at the corpus-parsing stage (that surface's §2) gates content before it reaches an embedding index, and an erasure request has to propagate into the vector index and any cached LLM trace containing the erased content, not just a warehouse table. |

## Validation
- Consent gating is verified against what actually transmits under a "denied" state, not assumed
  from the SDK's default.
- Masking is enforced at the platform layer for every dataset this overlay covers.
- Every erasure request is checked against Art. 17(3) exception grounds before executing, and
  implemented as tombstone + scheduled hard-delete.
- A retention schedule is stated per dataset class, with an audit trail covering access and erasure.
- PII classification and retention terms are present in the data contract for any regulated field,
  not tracked separately from it.
- The base surface this overlay stacks on is named, and this overlay's composition row (§ above) for
  that base has been applied.

## Failure modes and handoff
- **Consent assumed compliant because "the SDK defaults to it"** without verifying what actually
  transmits under denial (§1) — test it, don't assume it.
- **Application-layer PII redaction treated as sufficient** — any direct warehouse query bypasses it;
  platform-layer masking (§2) is mandatory, not optional, once a regime applies.
- **"Delete everything" executed without checking Art. 17(3) exceptions** — legal-hold, archival, or
  public-interest data deleted that shouldn't have been (§3).
- **An erasure request that reached the warehouse table but not the vector index or cached traces**
  when this overlay stacks on ai-data-plane — see that composition row; the erasure has to propagate
  everywhere the content lives.
- **This overlay applied when no regime actually applies** — unnecessary governance overhead on data
  that was never in scope; confirm which regime applies (inputs) before adding this surface's cost.
- **PII/retention terms tracked in a side document instead of the data contract** (§5) — the contract
  and the compliance obligation drift apart the first time one is updated without the other.
