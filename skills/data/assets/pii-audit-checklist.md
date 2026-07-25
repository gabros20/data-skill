# PII Audit Checklist (fillable)

Uniform-table format per skill convention — fill "Done" and "Evidence," never narrate. The chain
is classify → tag → platform-mask → retention/erasure → verify. Enforcement lives at the platform
layer (warehouse tags/policies), never only in application code — app-layer redaction is bypassed
by any warehouse query that skips the app. Source:
[governance-pii-and-retention.md](../references/governance-pii-and-retention.md).

## 1. Classify

| # | Step | Implementation required | Done (Y/N) | Evidence / location |
|---|---|---|---|---|
| P01 | Every column reviewed for PII | Recognizer pass (regex/checksum/NER, e.g. Presidio-style: email, phone, SSN/national-id, IP, card, IBAN) — decoupled detection from action | | |
| P02 | Privacy category assigned per column | IDENTIFIER / QUASI_IDENTIFIER / SENSITIVE (or your platform's equivalent) | | |
| P03 | Classification is automated, not a one-time manual pass | Reruns on schema change / new column | | |

## 2. Tag

| # | Step | Implementation required | Done (Y/N) | Evidence / location |
|---|---|---|---|---|
| P04 | Classification maps to a system/policy tag | e.g. Snowflake system tags from classification; BigQuery Data Catalog taxonomy tags | | |
| P05 | Tag applied at the column, not the table | 1 tag/column minimum granularity (BigQuery: ≤5 depth, ≤1,000 tags/table) | | |
| P06 | Contract reflects the tag | `pii: true` + `classification:` set in the data contract (see `data-contract-starter.odcs.yaml`) | | |

## 3. Platform-mask (enforce here, not in application code)

| # | Step | Implementation required | Done (Y/N) | Evidence / location |
|---|---|---|---|---|
| P07 | Masking policy attached to the tag | Snowflake tag-based masking policy / BigQuery column-level security policy tag / Unity Catalog row-filter or column-mask | | |
| P08 | Masking triggers automatically the moment a column is tagged | Not a manual per-query redaction step | | |
| P09 | IAM/role gating layered on top of masking | Masking + dataset ACLs both required — neither alone is sufficient | | |
| P10 | Write paths checked separately from read paths | e.g. INSERT into a protected column may not need read access, but UPDATE/DELETE/MERGE typically require elevated (fine-grained) access — confirm both are covered, not just SELECT | | |

## 4. Retention / erasure (GDPR Art.17, with exceptions — never a blanket "delete everything")

**"Delete everything on request" is the wrong default.** Art.17 has six erasure grounds *and*
Art.17(3) exceptions — check exception grounds before erasing, every time:

| Exception ground | Applies here? (Y/N) |
|---|---|
| Exercising the right of freedom of expression/information | |
| Compliance with a legal obligation / public-interest task | |
| Public-health grounds | |
| Archival/research/statistical purposes — where erasure would render the purpose impossible or seriously impair it | |
| Establishment, exercise, or defense of legal claims | |

| # | Step | Implementation required | Done (Y/N) | Evidence / location |
|---|---|---|---|---|
| P11 | Erasure request checked against exception grounds above before acting | Not auto-approved | | |
| P12 | Warehouse deletion is tombstone + scheduled hard-delete | Never in-place mutation of historical/append-only/backup tables | | |
| P13 | Cascading deletion notifies downstream processors when data was previously made public | "Reasonable steps, including technical measures" per Art.17 | | |
| P14 | Retention period stated per dataset and enforced on a schedule | Not indefinite-by-default | | |

## 5. Verify denied paths (prove the controls, don't assume them)

| # | Step | Implementation required | Done (Y/N) | Evidence / location |
|---|---|---|---|---|
| P15 | A masked column, queried directly by an unauthorized role, returns masked/denied — tested, not assumed | Run the actual query as that role | | |
| P16 | An erasure request against a real (non-prod) record produces a verifiable tombstone + scheduled hard-delete | End-to-end test, not a code review | | |
| P17 | Consent "denied" state verified against actual collection behavior (see `tracking-plan-starter.md` §4) — not just the config flag | | |
| P18 | Catalog/governance platform reflects current classification+masking state (DataHub / OpenMetadata / Unity Catalog or equivalent) | Not stale from a prior audit | | |

**Self-audit count: ____ / 18 controls implemented and evidenced.** Anything below 18/18 must be
either fixed before this dataset is treated as governed, or explicitly waived with a stated reason
and a named owner — never a silent gap. If a regulated surface applies (HIPAA/SOC2/CCPA on top of
GDPR), layer its additional requirements on top of this checklist rather than replacing it — see
`references/surface-regulated.md`.
