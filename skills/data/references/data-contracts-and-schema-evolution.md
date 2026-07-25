# Data Contracts & Schema Evolution

Purpose: Turn "producers and consumers keep breaking each other's pipelines" into an explicit,
versioned, testable contract — pick the spec (ODCS), pick the compatibility mode, and know when
the branded "data contracts" framework is overkill and the boring discipline underneath is the
actual fix.

Read when:
- A dataset/topic has (or needs) an explicit producer↔consumer agreement on schema, SLA, and
  semantics — not just "the pipeline happens to work today."
- Choosing a schema-registry compatibility mode, or deciding who upgrades first across a breaking
  change.
- Someone proposes adopting "data contracts" as an initiative and the team needs to know what's
  durable versus hype.

Skip when:
- The contract is for a single event's naming/shape at capture time — → `event-taxonomy-and-
  tracking.md` (Snowplow SchemaVer is the event-level analog of this ref's registry-level modes).
- The ask is *whether* already-landed data is trustworthy (tests, anomaly detection) — →
  `quality-and-testing.md`; a contract is the pre-condition, quality checks are the ongoing gate.
- The ask is CDC/streaming mechanics moving data between systems — → `ingestion-and-movement.md`;
  this ref owns the schema agreement, not the transport.
- PII field classification inside the contract's schema — → `governance-pii-and-retention.md`.

Inputs:
- Whether a producer and consumer are organizationally separate (different teams/services) — a
  contract earns its overhead here; a single team modeling its own warehouse usually doesn't need
  the full ceremony.
- The message format in play (Avro/Protobuf/JSON Schema) — decides which registry compatibility
  tooling applies directly.
- Whether the real problem is "we don't have a contract" or "we don't have write-audit-publish +
  quality checks" — these are frequently conflated; diagnose before prescribing either.

Produces: a contract in ODCS shape (or a documented reason to deviate), a named compatibility mode
with its upgrade-order implication understood, an executable test tying the contract to the live
dataset, and — where the team is chasing the "data contracts" brand rather than the underlying
discipline — a redirect to write-audit-publish + quality checks instead.

## Contents
- Pick the spec: ODCS, not DCS
- Compatibility modes — the canonical decision table
- Contract-as-executable-test
- Bidirectional duties
- The counterweight: buy the discipline, not the brand
- Licenses — verify the file, not the API

## Procedure

### 1. Pick the spec: ODCS, not DCS
**Open Data Contract Standard (ODCS) v3.1.0** is the unambiguous current answer (Bitol/Linux
Foundation AI & Data, Apache-2.0, bitol-io/open-data-contract-standard, 2026-07-25). The older
**Data Contract Specification (DCS) is officially deprecated** — its own README states: "with the
release of ODCS v3.1.0, we deprecate the Data Contract Specification… migrate to ODCS… supported
in Data Contract CLI and Entropy Data **until the end of 2026**." Don't start a new contract on
DCS; its field shape (`servicelevels.availability/retention/latency/freshness`, field-level
`quality.sql` rules, `lineage.inputFields`) is still a useful checklist to mine, but treat it as
reference material, not a live target.

**ODCS section list**: Fundamentals, Schema, References, Data quality, Support & communication
channels, Pricing, Team, Roles, SLA, Infrastructure & servers, Custom & other properties. Official
JSON Schema ships at `schema/odcs-json-schema-latest.json`; media type
`application/odcs+yaml;version=3.1.0`. Fundamentals carry `status` (proposed/draft/active/
deprecated/retired) — model contract lifecycle explicitly, don't leave it implicit in a wiki page.

**Data-quality rule tiers (decision axis)** — four ways to express a rule, increasing in power:
**Text** (human description, no enforcement) → **Library** (predefined metric: `nullValues`,
`missingValues`, `invalidValues`, `duplicateValues`, `rowCount`, each with `mustBe`/
`mustBeLessThan`/`mustBeGreaterThan`) → **SQL** (custom query) → **Custom** (vendor-specific: Soda,
Great Expectations, dbt tests, Monte Carlo). Start at Library; drop to SQL/Custom only when the
rule genuinely can't be expressed as a named metric.

**SLA driver enum** — `slaProperties[]` carry a `driver`: `regulatory | analytics | operational`.
Use this to prioritize which SLA breach pages someone at 3am versus which just gets logged.

### 2. Compatibility modes — the canonical decision table
Confluent Schema Registry's seven compatibility modes are the field's clearest formalization of
"who has to upgrade first," and apply to any Avro/Protobuf/JSON-Schema registry, not just Kafka
(confluentinc/schema-registry, `CompatibilityLevel.java`; docs.confluent.io/platform/current/
schema-registry/fundamentals/schema-evolution.html, 2026-07-25):

| Mode | Rule | Upgrade order |
|---|---|---|
| `NONE` | No compatibility checking | Either — fast migrate, no external consumers |
| `BACKWARD` (default) | New schema reads data written with the *previous* schema; checked against latest version only | Consumers upgrade first |
| `BACKWARD_TRANSITIVE` | New schema reads data from *all* prior versions | Consumers first |
| `FORWARD` | *Previous* schema can read data written with the new schema; checked vs. immediately-preceding version only | Producers upgrade first (edge devices, shipped SDKs) |
| `FORWARD_TRANSITIVE` | All previous schemas can read data from all future versions | Producers first |
| `FULL` | Both backward and forward, vs. latest version | Independent order |
| `FULL_TRANSITIVE` | Both, transitively across all versions | Independent order, max flexibility, most constrained evolution |

Default to `BACKWARD` unless producers are shipped SDKs/edge devices you don't control the upgrade
timing of (then `FORWARD`), or independent-order deploys across many teams justify `FULL`'s extra
constraint. This table is the canonical home for compatibility-mode decisions across this skill —
other references point here rather than restating it.

### 3. Contract-as-executable-test
The contract should be runnable, not just documentation. `datacontract-cli`
(datacontract/datacontract-cli, MIT — GitHub API mislabels it "Other," the LICENSE file is clean
MIT): `datacontract test <url>` connects to the live dataset and runs schema + quality checks
against it — the contract *is* the test, not a separate artifact that drifts from reality. It
supports both ODCS and DCS, and `export --format odcs` migrates an existing DCS contract forward.
Wire this into CI as the contract's own regression gate, distinct from the ongoing dataset-quality
suite in `quality-and-testing.md`.

### 4. Bidirectional duties — a contract obligates both sides
Chad Sanderson's foundational framing (dataproducts.substack.com/p/the-rise-of-data-contracts,
2022-08-22) names the failure a contract fixes as the **"GIGO Cycle"**: production databases
treated as non-consensual APIs, producers unaware of downstream analytics usage, cascading breaks.
A contract is not a one-way schema dump from producer to consumer:
- **Producer duties**: emit data in the consumer-required shape, maintain backward-compatible
  versioning, notify consumers before schema changes, document semantics.
- **Consumer duties**: state explicit data requirements upfront (iteratively, not once), express
  business semantics in the contract, document use cases and SLAs, state data-quality expectations
  — a contract with only producer obligations isn't a contract, it's a schema.

### 5. The counterweight: don't buy the brand, buy the discipline
Zach Wilson (50k+-follower data-engineering educator) lists data contracts among terms that are
"mostly hype": *"data contracts, learn write-audit-publish pattern and data quality checks
instead"* (x.com/Zachly/status/1773491085777797167). This isn't a rejection of Sanderson's
underlying goal (schema/SLA agreements enforced upstream, near-consensus) — it's a rejection of
treating "adopt a data contracts initiative" as the deliverable. Sarah (@sarahcat21) names data
contracts as a top example of a field that "debates more than it implements"
(x.com/sarahcat21/status/1575537847607533568). **Teach both**: use §1–§4 when a real
producer/consumer boundary exists; redirect to write-audit-publish + quality checks
(`quality-and-testing.md`) when the team is reaching for the brand without the boundary.

### 6. Licenses — verify the file, not the API
GitHub's license field is unreliable for this cluster. **Confluent Schema Registry**: server
`core/` = Confluent Community License v1.0 (non-OSI, source-available); `client-*`/`avro-*` =
Apache-2.0 — split by directory, read `LICENSE-ConfluentCommunity` before assuming the repo is
open. **ODCS** = Apache-2.0. **DCS** = MIT. **datacontract-cli** = MIT (API shows "Other" —
mismatch). Confirm current terms before redistributing any of these.

## Validation
- [ ] Contract is expressed in ODCS shape (or the deviation from ODCS is a documented decision).
- [ ] A compatibility mode is named explicitly, with its upgrade-order implication understood by
      both producer and consumer teams — not left as an unstated default.
- [ ] The contract is wired to an executable test against the live dataset, not just a markdown
      file that can silently drift from reality.
- [ ] Both producer and consumer obligations are stated — not a one-sided schema dump.
- [ ] If "data contracts" was proposed as an initiative rather than a specific producer/consumer
      boundary, write-audit-publish + quality checks was considered as the leaner alternative.

## Failure modes and handoff
- **The real ask is event-level naming/versioning at capture time** — → `event-taxonomy-and-
  tracking.md`.
- **The real ask is ongoing trust in landed data, not the contract itself** — →
  `quality-and-testing.md`.
- **The real ask is the transport (CDC, streaming, batch sync) carrying data across the contract
  boundary** — → `ingestion-and-movement.md`.
- **PII fields inside the schema need classification/masking policy** — →
  `governance-pii-and-retention.md`.
- **The team wants the "data contracts" brand but has no real producer/consumer split** — redirect
  to write-audit-publish + quality checks; a contract without two independent parties is just
  documentation.
