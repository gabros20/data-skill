# Independent review — data skill v0.1.0 (2026-07-25)

Reviewer: independent Opus agent (did not author the pack). Ground truth: `research/` corpora.

## Verdict: fix-then-release → fixes applied same-day. BLOCKER 0 · S-major 1 · N-minor 2.

The `ai`-skill BLOCKER class (invented tool/model facts shipped as "verified") does NOT recur —
every volatile anchor traces to a corpus line with the same value.

| ID | Sev | Finding | Resolution |
|---|---|---|---|
| S1 | S-major | Snowplow `MODEL-REVISION-ADDITION` stamped "lift verbatim" in tracking-plan-starter (and uncaveated in event-taxonomy ref + schema-compatibility cheatsheet) while web-canon flagged the literal term [UNVERIFIED]; github corpus verified it from the Iglu `self` block — grounded, but missing the caveat the skill's own anti-staleness invariant demands | FIXED: dropped "lift verbatim", added re-verify-against-docs.snowplow.io caveats in all 3 files |
| N1 | N-minor | SKILL.md said "9-bucket DQ taxonomy"; the lifted table is 11 rows (9 convergent + PR-diff + PII) | FIXED: reworded to "9 core buckets + PR-diff + PII" |
| N2 | N-minor | `growth` routed to but unshipped | NO ACTION: established family convention (product-skill routes to it too) |

## Verified clean (spot-audited against corpora)
- $ anchors exact (BQ $6.25/TiB, slots $0.04/$0.06/$0.10; Snowflake exponential sizing).
- ODCS v3.1.0 (github corpus authoritative over web-canon's stale v3.0.0); Confluent 7 modes;
  Prefect→Dagster 2026-07-13 dated; SLULA stale-Apache trap; Estuary BSL→Apache 2029-11-20;
  MetricFlow ≥0.209 Apache; dbt Fusion ELv2 + 5 Apache crates; Debezium v3.6.0.Final.
- Open gaps correctly absent/caveated (dbt-SDF acquisition absent; streaming essay anchored on COST).
- RAG three-way seam consistent; experiment ref declines design/OEC to growth.
- Router resolves; 16/16 primacy headers + "Skip when:" + Contents; 9 Resources ↔ 9 assets;
  single-homing clean; surfaces point, don't re-teach; versions consistent at 0.1.0; no mojibake.
