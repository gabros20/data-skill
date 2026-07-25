# Governance, PII & Retention

Purpose: Decide where PII protection is actually enforced across the data platform (not just
where it's collected), how erasure/retention requests are honored without breaking warehouse
architecture, and which catalog fits an org's governance model.

Read when:
- Classifying, masking, or redacting PII in a warehouse or lakehouse.
- Handling a "right to be forgotten" / GDPR / CCPA deletion request against analytical stores.
- Choosing a data catalog for governance, lineage, or access control.
- Deciding where PII-detection logic should run — ingest-time, query-time, or app-time.
- Evaluating "data mesh" as a governance model for a multi-domain data org.

Skip when:
- The ask is consent gating at the point of collection (Consent Mode, cookie banners) — →
  `event-taxonomy-and-tracking.md`; this ref governs data already at rest, that ref governs
  whether it gets collected at all.
- The ask is the schema-level declaration of which fields are PII (ODCS `pii`/`classification`
  field attributes) — → `data-contracts-and-schema-evolution.md`; this ref owns runtime
  enforcement, that ref owns the contract that declares it.
- The ask is tracing how far a PII column's exposure has spread downstream — →
  `lineage-and-observability.md`.
- The ask is a security/compliance boundary spanning systems beyond the data plane (a SOC2
  program, org-wide IAM design) — → `architecture`; this ref implements PII controls inside
  that boundary, it doesn't set the boundary.

Inputs:
- Where the field is queried from today (Snowflake / BigQuery / Databricks / other) — decides
  which native tagging mechanism applies.
- Which regulation(s) apply (GDPR, CCPA, HIPAA, none) — decides which erasure/notice
  obligations exist.
- Whether deletion requests must be honored against append-only/historical tables — decides
  tombstone-vs-mutation.
- Existing catalog, if any — decides incremental rollout vs from-scratch governance build.

Produces: a named platform-native enforcement mechanism for every PII field (not app-layer
redaction alone); an erasure procedure that checks legal exceptions before deleting and purges
via tombstone + scheduled hard-delete; a catalog choice with the tradeoff stated; a mesh-vs-
central governance stance with the federated-governance pairing made explicit.

## Contents
- Platform-layer enforcement is the invariant
- Presidio: decoupled detect → act
- GDPR Article 17 — erasure with exceptions
- CCPA — notice-at-collection and the "sharing" trap
- PII in the contract, not only the platform
- Catalog choice
- Data mesh — the principle survives, the program is contested

## Procedure

### 1. Platform-layer enforcement is the invariant — not app-layer redaction
State this plainly to anyone proposing PII protection in application code: any redaction
implemented only in an app or an API response is bypassed by anyone with direct
warehouse/lakehouse query access — a BI tool, an ad-hoc SQL client, or another pipeline.
Enforce at the layer where the query actually runs.

| Platform | Mechanism | Gotcha |
|---|---|---|
| Snowflake | Classification auto-assigns semantic category + privacy category (`IDENTIFIER`/`QUASI_IDENTIFIER`/`SENSITIVE`) → system tags auto-applied → tag-based masking policies **auto-mask the moment a column is tagged** | (docs.snowflake.com/en/user-guide/governance-classify, 2026-07-25) — no explicit GDPR/CCPA citation on Snowflake's own page; cite the regulation directly, not the vendor doc, for a compliance claim |
| BigQuery | Policy-tag-based: Data Catalog taxonomies → tags on columns (1 tag/column, depth ≤5, ≤1,000 tags/table) → IAM roles gate access; layers **on top of** dataset ACLs — both required, neither alone suffices | UPDATE/DELETE/MERGE on a protected column requires the **Fine-Grained Reader** role even though plain INSERT does not — a masking setup that "worked" in a read-only test can still fail once a pipeline needs to write |
| Databricks Unity Catalog | Three-level namespace (`catalog.schema.object`); row/column-level security via filters; automatic lineage + sensitive-data classification | Auto-enablement applies only to workspaces created after 2023-11-08 — older workspaces need manual enablement, don't assume it's already on |

The shared shape across all three: classification and enforcement are **one platform-native
operation**, not two — tag a column, masking/access-control follows automatically. Treat any
design that redacts PII downstream in BI or an app response as a stopgap, never the control.

### 2. Presidio — decoupled detect → act, for what native tagging can't reach
When PII lives in free text (support tickets, LLM traces, unstructured logs) rather than a
typed column, warehouse tagging doesn't apply — use a dedicated detect/act pipeline. Presidio's
architecture is the reference shape: **Analyzer** (recognizers, detection only) feeds
**Anonymizer** (operators, action only), kept separate so one detection pass can drive multiple
actions.

- Recognizer catalog: generic (credit_card, crypto, date, email, iban, ip, mac, phone, url);
  ~20 country-specific jurisdictions (US ssn/itin/passport/npi/mbi; UK nino/nhs; DE
  tax_id/vat_id; …) via checksum/regex/context; NER engines (`gliner`, `huggingface_ner`,
  `medical_ner`); classic NLP (spacy/stanza/transformers); and the newest addition,
  **LLM-assisted recognition** (`langextract` + Azure OpenAI) — pluggable alongside the
  regex/NER recognizers, not a replacement for them.
- Anonymizer operators: `mask`, `redact`, `replace`, `hash`, `encrypt`/`decrypt` (AES), `keep`,
  `custom` — pick by sensitivity and reversibility need (hash/encrypt are reversible by key,
  mask/redact are not).
- **Gotcha (2026-07-25):** Presidio's docs state it is "transitioning to a community-owned
  project" under a new org (`data-privacy-stack/presidio`). Treat `microsoft/presidio` as
  canonical as-of this date but re-verify the org before pinning a version in a build doc —
  mid-migration projects move.
- License: MIT.

### 3. GDPR Article 17 — erasure has real exceptions; "delete everything" is the wrong default
Article 17 grants six erasure grounds, but **Article 17(3) carves out exceptions** that
override them: exercising the right of free expression/information; legal or public-interest
compliance; public health; **archival/research/statistical purposes if erasure "would render
impossible or seriously impair" that purpose**; and defense of legal claims. An erasure
procedure that unconditionally purges every row on request is as often non-compliant with the
*spirit* of the exceptions as it is compliant with the request — check which ground (if any)
applies before deleting, never after.

Two operational consequences follow directly:
- **Cascading deletion.** When the data was made public, the controller must take "reasonable
  steps, including technical measures," to notify other processors holding copies — the
  obligation doesn't stop at your own warehouse.
- **Tombstone + scheduled hard-delete, not in-place history mutation.** Real-time erasure is in
  tension with append-only/snapshot/backup warehouse architecture. The resolvable pattern:
  mark the row tombstoned (logically deleted, excluded from all reads) immediately, then run a
  scheduled job that physically purges it — including from backups/snapshots on their own
  retention cycle — rather than mutating historical fact tables in place, which breaks
  point-in-time reproducibility for every other row in that table.

(gdpr-info.eu/art-17-gdpr/, 2026-07-25)

### 4. CCPA — notice-at-collection, and "sharing" is broader than it sounds
CCPA's collection-time obligation: a **"notice at collection"** must appear at or before
collection, listing categories, purposes, a privacy-policy link, and — where applicable — a
"Do Not Sell or Share" link. Two distinctions matter for a data-plane design:
- Minimization under CCPA is **purpose-limitation-based** (purposes must be disclosed) — not a
  hard technical minimization mandate the way GDPR treats it.
- **"Sharing" explicitly includes cross-context behavioral advertising**, not just monetary
  sale — a pipeline feeding an ad-tech platform for retargeting is "sharing" under CCPA even if
  no money changes hands, and needs the same notice/opt-out surface as an outright sale.

(oag.ca.gov/privacy/ccpa, 2026-07-25)

### 5. PII belongs in the contract, not only the platform
A field's PII status shouldn't be tribal knowledge rediscovered at query time — declare it
where the schema is declared. ODCS's `models.fields` carries `pii` and `classification`
attributes alongside `type`/`required`/`primaryKey` — the contract is where a consumer learns a
field is sensitive *before* they query it; platform tagging (§1) is what enforces the
consequence. See [data-contracts-and-schema-evolution.md](data-contracts-and-schema-evolution.md)
for the full field spec — this ref owns what happens once that tag exists.

### 6. Catalog choice — pick by "runs its own tests" vs "ingests results" vs "RBAC-first"
Three shapes recur, license-flagged:

| Catalog | Shape | Distinguishing capability | License |
|---|---|---|---|
| DataHub | Entity/aspect metadata platform | Independently-versioned **aspects** per entity (`schemaMetadata`, `ownership`, `globalTags`, `testResults`, `incidentsSummary`, …) — an extensible envelope that *ingests* external test/DQ results rather than running its own | Apache-2.0 (root; verify Cloud-gated features per module) |
| OpenMetadata | Catalog that **runs its own tests** natively | Ships a full Great-Expectations-equivalent test taxonomy in-catalog (`tableRowCountToBeBetween`, `tableDiff` — a Recce-style regression diff — built in); `TestSuite → TestCase → TestDefinition` with incident triage | Apache-2.0 |
| Apache Atlas | Hadoop-ecosystem governance + type system | `Referenceable → Asset → DataSet/Process/Infrastructure` type hierarchy; treats its own metamodel as schema-versioned/migrated; integrates with Ranger for RBAC/ABAC | Apache-2.0 |

Decision rule: pick OpenMetadata when the org wants the catalog itself to own and run
data-quality checks; pick DataHub when checks already run elsewhere (dbt tests, Soda, Great
Expectations) and the catalog's job is aggregating results into one governance surface; pick
Atlas only inside an existing Hadoop/Ranger estate.

(github.com/datahub-project/datahub, github.com/open-metadata/OpenMetadata,
github.com/apache/atlas — 2026-07-25)

### 7. Data mesh — the principle survives, the program is contested
Zhamak Dehghani's four founding principles (martinfowler.com/articles/data-mesh-principles.html,
2019, canon):
1. Domain-oriented decentralized ownership.
2. Data as a product (the domain data product owner role, measured on quality/lead-time/
   satisfaction).
3. Self-serve data platform.
4. **Federated computational governance** — decentralized autonomy + globally-enforced
   interoperability standards, automated by the platform.

The backlash is real and comes from senior infra voices, not fringe accounts: Erik Bernhardsson
(Modal founder, ex-Spotify) calls it *"another weird data fad… enforcing proper ownership of
data makes sense, but I think the rest will be used to rationalize data silos"*
(x.com/bernhardsson/status/1400461693197570051, contested); Zach Wilson's counter is blunter —
*"learn master data management instead"* (x.com/Zachly/status/1773491085777797167, contested).

**The durable primitive to teach:** domain ownership works **only paired with** principle 4 —
federated, platform-enforced governance. Ownership without that pairing is exactly what the
backlash describes: silos with extra ceremony. Don't recommend "data mesh" as a program;
recommend domain ownership *and* verify a federated governance layer (shared classification
standards, shared platform, shared access model) exists alongside it.

Adopt incrementally, not as a big-bang re-org: dbt Labs' own guidance for dbt Mesh
(multi-project cross-`{{ ref() }}`, Groups, Access modifiers, Model Contracts) is to **start
single-project and adopt Mesh only when triggered** — model-count performance degradation,
team-workflow decoupling need, or security/isolation requirements — explicitly warning against
"over-engineer[ing] governance early." (docs.getdbt.com/best-practices/how-we-mesh/mesh-1-intro,
2026-07-25)

## Validation
- Every PII field has a platform-native enforcement mechanism named (tag/policy/filter) — not
  solely an app-layer redaction step.
- Any erasure procedure checks GDPR Art.17(3) exceptions before deleting, and physically purges
  via tombstone + scheduled hard-delete rather than mutating historical tables in place.
- If CCPA applies, a notice-at-collection surface exists and "sharing" (including
  behavioral-ad feeds) is covered by the same opt-out as sale.
- PII/classification is declared in the data contract (ODCS `pii`/`classification` fields), not
  only enforced ad hoc at the platform.
- A catalog choice states which of "runs its own tests" / "ingests external results" /
  "RBAC-first Hadoop-native" it optimizes for.
- Any "data mesh" recommendation names domain ownership **and** the federated-governance
  pairing — never ownership alone.

## Failure modes and handoff
- **The real ask is gating collection/consent before data is captured** — →
  `event-taxonomy-and-tracking.md`; this ref governs data already at rest.
- **The real ask is declaring which fields are PII in a schema/contract** — →
  `data-contracts-and-schema-evolution.md`; this ref owns enforcement once declared.
- **The real ask is tracing how far a PII column's exposure has spread downstream** — →
  `lineage-and-observability.md`.
- **"Delete everything" is requested with no exception check** — stop and confirm which
  Art.17(3) ground (if any) applies before running a purge; an unconditional delete on an
  archival/statistical dataset can itself be non-compliant.
- **A regulated-industry overlay applies (HIPAA/SOC2) beyond GDPR/CCPA** — → `surface-regulated`;
  this ref's platform-enforcement pattern is the base, the overlay hardens it further.
