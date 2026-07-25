# `data` skill — Agent Skills ecosystem corpus

Research sweep of the open Agent Skills ecosystem (skills.sh + GitHub) for existing
data / analytics / pipeline / governance skills, to size the competitive landscape and
prove/disprove that the `data` wedge (a unified, vendor-neutral, faceted-router
data-engineering skill) is open.

- **As-of:** 2026-07-25 (install counts and "latest" claims are volatile; each linked to source)
- **Method:** `npx skills find` across 40+ query terms + 3 Sonnet deep-read workers
- **Scope searched:** event taxonomy/tracking plans; data contracts; ETL/ELT/CDC/streaming; dbt-style transformation/modeling; semantic/metrics layers; BI/dashboards-as-code; data quality/testing/lineage; governance/PII/retention; experiment measurement; AI-era data plane (LLM trace warehousing, RAG corpus ETL)

---

## Wedge verdict (headline)

**OPEN.** As-of 2026-07-25, no skill in the ecosystem is a unified, vendor-neutral,
faceted-router (jobs × surfaces, thin router + on-demand references) data-engineering
skill. The field is exclusively:

1. **Vendor-scoped multi-skill packs** — official packs from dbt-labs, PostHog, Google,
   Astronomer/Airflow, Databricks, DuckDB/MotherDuck, Snowflake, AWS, Confluent, Grafana,
   Looker, DataHub, Metabase, Elastic, Datadog. Each locked to one product; flat piles of
   independent how-to skills, no cross-vendor router, no vendor-neutral mental models.
2. **Individual data topic-skills inside big generic agent collections** — wshobson/agents,
   github/awesome-copilot. Standalone single-topic skills (a dbt skill, a SQL skill, an
   Airflow skill), not a coherent data-engineering system.
3. **Aggregated / curated dumps** — aradotso/data-skills, nimrodfisher/data-analytics-skills,
   gordonmurray/data-engineering-skills. Collections of tutorials, portfolios, and
   standalone how-tos; no unifying router or jobs×surfaces design.

Same posture as the `ai` skill: the topic is heavily populated at the vendor/single-topic
level but **nobody owns the neutral, decision-first, faceted-router layer.** (Deep-read
confirmation of structure per worker findings below.)

---

## Full discovery inventory

Verdict key: **COMPETE** = same neutral-router ambition (would collide) · **LIFT** = worth
mining for content/patterns · **IGNORE** = vendor-locked or off-scope reference only.

### A. Big generic agent collections (individual data topic-skills)

| Skill | URL | Installs (2026-07-25) | Verdict |
|---|---|---|---|
| wshobson/agents@dbt-transformation-patterns | skills.sh/wshobson/agents/dbt-transformation-patterns | 8.5K | LIFT |
| wshobson/agents@data-quality-frameworks | skills.sh/wshobson/agents/data-quality-frameworks | 8.6K | LIFT |
| wshobson/agents@sql-optimization-patterns | skills.sh/wshobson/agents/sql-optimization-patterns | 16.2K | LIFT |
| wshobson/agents@postgresql-table-design | skills.sh/wshobson/agents/postgresql-table-design | 22.2K | LIFT |
| wshobson/agents@airflow-dag-patterns | skills.sh/wshobson/agents/airflow-dag-patterns | 8.5K | LIFT |
| wshobson/agents@spark-optimization | skills.sh/wshobson/agents/spark-optimization | 8.3K | LIFT |
| wshobson/agents@ml-pipeline-workflow | skills.sh/wshobson/agents/ml-pipeline-workflow | 8.4K | LIFT |
| wshobson/agents@kpi-dashboard-design | skills.sh/wshobson/agents/kpi-dashboard-design | 11.8K | LIFT |
| wshobson/agents@grafana-dashboards | skills.sh/wshobson/agents/grafana-dashboards | 10.2K | LIFT/IGNORE |
| wshobson/agents@startup-metrics-framework | skills.sh/wshobson/agents/startup-metrics-framework | 8.6K | LIFT |
| github/awesome-copilot@bigquery-pipeline-audit | skills.sh/github/awesome-copilot/bigquery-pipeline-audit | 8.7K | LIFT |
| github/awesome-copilot@snowflake-semanticview | skills.sh/github/awesome-copilot/snowflake-semanticview | 8.9K | LIFT |
| github/awesome-copilot@sql-optimization | skills.sh/github/awesome-copilot/sql-optimization | 13.4K | LIFT |
| github/awesome-copilot@sql-code-review | skills.sh/github/awesome-copilot/sql-code-review | 12.1K | LIFT |
| github/awesome-copilot@postgresql-optimization | skills.sh/github/awesome-copilot/postgresql-optimization | 14K | LIFT |
| github/awesome-copilot@cosmosdb-datamodeling | skills.sh/github/awesome-copilot/cosmosdb-datamodeling | 8.6K | IGNORE |

### B. Dedicated "data" skill collections (closest to competitor — curated/aggregated)

| Skill/Repo | URL | Installs | Verdict |
|---|---|---|---|
| aradotso/data-skills (whole collection) | skills.sh/aradotso/data-skills | 1.3–2.1K each | COMPETE? (verify structure) |
| ↳ @data-engineering-medallion-pipeline | skills.sh/aradotso/data-skills/data-engineering-medallion-pipeline | 1.7K | LIFT |
| ↳ @apache-airflow-orchestration | skills.sh/aradotso/data-skills/apache-airflow-orchestration | 2.1K | LIFT |
| ↳ @snowflake-dbt-airbnb-analytics | skills.sh/aradotso/data-skills/snowflake-dbt-airbnb-analytics | 1.4K | IGNORE |
| ↳ @enterprise-data-engineering-pipeline-ssis-pyspark | (same repo) | 1.7K | IGNORE |
| ↳ @data-engineering-patterns-fabric-databricks | (same repo) | 1.3K | IGNORE |
| ↳ @datatalks-data-engineering-zoomcamp / study-material / portfolios | (same repo) | 1.7–2.1K | IGNORE (aggregated tutorials) |
| nimrodfisher/data-analytics-skills@data-catalog-entry | skills.sh/nimrodfisher/data-analytics-skills/data-catalog-entry | 121 | COMPETE? (small) |
| gordonmurray/data-engineering-skills@iceberg | skills.sh/gordonmurray/data-engineering-skills/iceberg | 18 | COMPETE? (tiny) |
| jeffallan/claude-skills@spark-engineer | skills.sh/jeffallan/claude-skills/spark-engineer | 2.7K | IGNORE |
| melodic-software/claude-code-plugins@data-modeling | skills.sh/melodic-software/claude-code-plugins/data-modeling | 138 | LIFT? |
| lyndonkl/claude@data-schema-knowledge-modeling | skills.sh/lyndonkl/claude/data-schema-knowledge-modeling | 131 | IGNORE |

### C. Vendor-scoped official packs (IGNORE for compete; mine for neutral patterns)

| Vendor pack | Notable skills | URL root | Installs |
|---|---|---|---|
| dbt-labs/dbt-agent-skills | using-dbt-for-analytics-engineering, building-dbt-semantic-layer, adding-dbt-unit-test, running-dbt-commands, answering-NL-questions-with-dbt, fetching-dbt-docs | skills.sh/dbt-labs/dbt-agent-skills | 580–676 |
| posthog/skills + posthog-for-claude + ai-plugin | setting-up-a-data-warehouse-source, auditing-warehouse-data-health, diagnosing-failed-warehouse-syncs, posthog-instrumentation, querying-posthog-data, exploring-llm-traces | skills.sh/posthog | 98–1.5K |
| astronomer/agents (Airflow) | airflow, warehouse-init, tracing-upstream-lineage, annotating-task-lineage, migrating-airflow-2-to-3, airflow-hitl, managing-astro-local-env | skills.sh/astronomer/agents | 664–1.2K |
| google/skills | bigquery-basics, bigquery-ai-ml, bigquery-bigframes, datalineage-bigquery-asset-impact-analysis, datalineage-summary | skills.sh/google/skills | 289–9.6K |
| databricks/databricks-agent-skills | databricks-core, databricks-pipelines, databricks-apps, databricks-lakebase, databricks-iceberg | skills.sh/databricks/databricks-agent-skills | 72–1.1K |
| aws/agent-toolkit-for-aws | ingesting-into-data-lake, querying-data-lake, exploring-data-catalog, querying-aws-sagemaker-catalog, aws-messaging-and-streaming | skills.sh/aws/agent-toolkit-for-aws | 1.1–3.1K |
| duckdb/duckdb-skills | query, read-file, attach-db, install-duckdb, duckdb-docs, read-memories | skills.sh/duckdb/duckdb-skills | 451–790 |
| motherduckdb/agent-skills | motherduck-security-governance | skills.sh/motherduckdb/agent-skills | 239 |
| snowflake-labs / adobe/skills@snowflake / jezweb@snowflake-platform | cortex-code, snowflake, snowflake-platform, streamlit-snowflake | skills.sh/snowflake-labs | 323–703 |
| confluentinc/agent-skills | confluent-cloud-cdc-tableflow | skills.sh/confluentinc/agent-skills | 57 |
| grafana/skills | dashboarding | skills.sh/grafana/skills | 2.9K |
| looker-open-source/looker-skills | creating-looker-dashboard | skills.sh/looker-open-source/looker-skills | 23 |
| datahub-project/datahub-skills | datahub-lineage | skills.sh/datahub-project/datahub-skills | 137 |
| metabase/metabase | analytics-events, metabase-data-app-semantic-layer | skills.sh/metabase/metabase | 14–1.3K |
| getnao/nao | add-semantic-layer, write-context-rules | skills.sh/getnao/nao | 135–151 |
| revosai/skills | create-cubes, query-semantic-model, visualize-semantic-model | skills.sh/revosai/skills | 66–67 |
| elastic/agent-skills | kibana-streams, observability-llm-obs | skills.sh/elastic/agent-skills | 2.0–2.1K |
| datadog-labs/agent-skills | agent-observability-eval-pipeline, llm-obs-trace-rca | skills.sh/datadog-labs/agent-skills | 134–237 |
| supabase/supabase | telemetry-standards | skills.sh/supabase/supabase | 217 |

### D. Analytics / event-tracking / marketing-analytics (event taxonomy in scope)

| Skill | URL | Installs | Verdict |
|---|---|---|---|
| coreyhaines31/marketingskills@analytics-tracking | skills.sh/coreyhaines31/marketingskills/analytics-tracking | 58.3K | LIFT (tracking-plan content) |
| coreyhaines31/marketingskills@analytics | skills.sh/coreyhaines31/marketingskills/analytics | 39.2K | LIFT |
| coreyhaines31/marketingskills@ab-testing | skills.sh/coreyhaines31/marketingskills/ab-testing | 36.4K | LIFT |
| accoil/product-tracking-skills@product-tracking-design-tracking-plan | skills.sh/accoil/product-tracking-skills/... | 27 | LIFT (dead-center: event taxonomy) |
| thinkingaiagenticengine/ae-cli@ae-generate-tracking-plan | skills.sh/thinkingaiagenticengine/ae-cli/... | 48 | LIFT |
| cognyai/claude-code-marketing-skills@gtm-tracking-plan / gtm-setup | skills.sh/cognyai/... | 28–30 | IGNORE (GTM-specific) |
| markdown-viewer/skills@data-analytics | skills.sh/markdown-viewer/skills/data-analytics | 2.8K | LIFT? |
| code.deepline.com@deepline-analytics | skills.sh/code.deepline.com/deepline-analytics | 10.5K | IGNORE (vendor) |
| apify/agent-skills@apify-content-analytics | skills.sh/apify/agent-skills/apify-content-analytics | 2.6K | IGNORE |

### E. Governance / PII / retention / quality (standalone)

| Skill | URL | Installs | Verdict |
|---|---|---|---|
| patricio0312rev/skills@data-retention-archiving-planner | skills.sh/patricio0312rev/skills/data-retention-archiving-planner | 194 | LIFT |
| patricio0312rev/skills@pii-redaction-logging-policy-builder | skills.sh/patricio0312rev/skills/... | 194 | LIFT |
| ruvnet/ruflo@pii-detect | skills.sh/ruvnet/ruflo/pii-detect | 666 | LIFT? |
| alirezarezvani/claude-skills@data-quality-auditor | skills.sh/alirezarezvani/claude-skills/data-quality-auditor | 577 | LIFT? |
| hubspot/agent-cli-skills@crm-data-quality | skills.sh/hubspot/agent-cli-skills/crm-data-quality | 754 | IGNORE (CRM) |
| majesticlabs-dev/majestic-marketplace@great-expectations / etl-patterns | skills.sh/majesticlabs-dev/majestic-marketplace | 48–73 | LIFT? |
| claude-office-skills/skills@etl-pipeline | skills.sh/claude-office-skills/skills/etl-pipeline | 3.7K | LIFT? |

### F. Experiment measurement (foundations overlap)

| Skill | URL | Installs | Verdict |
|---|---|---|---|
| rampstackco/claude-skills@data-warehouse-experimentation | skills.sh/rampstackco/claude-skills/data-warehouse-experimentation | 110 | LIFT |
| launchdarkly/agent-skills@launchdarkly-experiment-setup / aiconfig-ai-metrics | skills.sh/launchdarkly/agent-skills | 2.3–2.4K | IGNORE (vendor) |
| arize-ai/arize-skills@arize-experiment / arize-instrumentation | skills.sh/arize-ai/arize-skills | 1.9–2.1K | IGNORE (vendor, AI obs) |

---

## Deep-read findings (3 Sonnet workers)

### Cluster 1 — Generic collections (wshobson/agents, github/awesome-copilot)

**wshobson/agents** — MIT (© Seth Hobson 2024). A **plugin marketplace**, not a data skill:
`plugins/<plugin>/skills/<skill>/SKILL.md`. README claims 94 plugins / 203 agents /
**175 skills** / 109 commands. Data-engineering is **scattered across 7 separate plugins**
(`data-engineering`, `database-design`, `developer-essentials`, `business-analytics`,
`machine-learning-ops`, `observability-monitoring`, `startup-business-analyst`) — no single
data entry point. Per-skill idiom = **two-tier** (SKILL.md ~95–215 lines + one flat
`references/details.md` 290–450 lines, dispatched by a single line). This is thin-ish but
**NOT faceted (jobs×surfaces)** — just one overflow file per skill, no sub-taxonomy.

- Best single artifact: `plugins/database-design/skills/postgresql/SKILL.md` (202 lines,
  monolithic) — dense "DO NOT use X; use Y" type table, PostgreSQL gotchas list, JSONB GIN
  vs `jsonb_path_ops` decision tree, workload-specific schema sections.
- `plugins/data-engineering/skills/data-quality-frameworks/SKILL.md` — **Data Quality
  Dimensions taxonomy** (Completeness/Uniqueness/Validity/Accuracy/Consistency/Timeliness ↔
  example check, lines 21-28) + testing-pyramid diagram.
- `plugins/business-analytics/skills/kpi-dashboard-design/SKILL.md` — lift-worthy
  troubleshooting section (lines 78-143): MRR-normalization CASE fix, OLAP-vs-OLTP
  pre-aggregation pattern, 2-stdev rolling anomaly detection vs static alert fatigue.
- **CONTENT DEFECT #1**: `data-quality-frameworks/SKILL.md` lines 85-99 contain an orphaned,
  truncated Python fragment rendered as a broken markdown heading
  (`## Summary: {total_passed}/{total_tables} tables passed")`) — a content-splitting defect
  introduced at the "Detailed patterns" cut point. (Lesson for us: automate the split, lint
  the seam.)

**github/awesome-copilot** — MIT (© GitHub, Inc.). Flat `skills/<name>/SKILL.md`, **391 skill
dirs**, no plugin grouping. All 6 data targets are **pure monolithic single-file, no
references/**. Several are ported VS Code Copilot prompt files (use `${selection}` template
syntax — a porting tell, not agent-native).

- Top artifact: `skills/bigquery-pipeline-audit/SKILL.md` (129 lines) — **A–F structured
  audit rubric** for Python+BigQuery pipelines with hard-fail gates (BQ job count >20,
  missing `maximum_bytes_billed`, per-date query loops), MERGE-key/idempotency rules
  (`run_id` must NOT be in the dedupe key), PASS/FAIL + ranked-patch + top-3-cost-risk output.
- `skills/cosmosdb-datamodeling/SKILL.md` — **1045 lines** (largest single skill found);
  interview-driven NoSQL modeling methodology: Container Consolidation Decision Framework
  (>50% access-overlap rule), Multi-Doc vs Separate Containers (percentage-banded 70/50/<50),
  hot-partition analysis, "🔴 CRITICAL don't-proceed-until-captured" directive. Effectively a
  mini playbook (but Cosmos-specific).
- `skills/postgresql-optimization/SKILL.md` (404 lines) — genuine "Optimization Checklist"
  (checkbox format, lines 318-346). **CONTENT DEFECT #2**: 3 section headers (lines 10, 97,
  145) carry a mojibake replacement char `�` where an emoji should be — a real source
  encoding defect.

**Cluster-1 verdict:** neither is a unified jobs×surfaces faceted-router data skill. Both are
multi-domain grab-bags where data topics are individual, uncoordinated single-topic skills.

### Cluster 2 — Dedicated data-skill collections (aradotso, gordonmurray, misc)

**aradotso/data-skills** — MIT. **Bot-scraped aggregator, NOT authored.** README states a bot
scans GitHub every 30 min for trending data-eng repos and auto-writes a SKILL.md for the top
new one. 90+ flat folders, **zero subdirectories/references anywhere**, all monolithic
(716–1158 lines) — installation guides / tutorial writeups for one tool or one person's
portfolio. Frontmatter `name/description/triggers`. Massive junk: 25+ near-identical
"harvard-artifacts-*" entries for one student project, plus a
`malware-warning-sas-analytics-pro-cracked` entry the bot ingested. **Nothing lift-worthy —
derivative scraped content, not engineering judgment.** (Despite branding, not a real
competitor.)

**gordonmurray/data-engineering-skills** — MIT. Small, original, well-curated but
**7 independent flat tool-reference skills** (`iceberg`, `paimon`, `fluss`, `flink`, `iggy`,
`lance`, `docker-compose`) — no umbrella router, no jobs taxonomy, narrow slice (lakehouse
table formats + streaming storage + Flink). `iceberg/SKILL.md` = 39 lines with a consistent
template `Current Facts → How To Use → Design Rules → **Update Checklist**`. The **Update
Checklist** ("Recheck Iceberg releases before changing versions; recheck engine v3 support
before upgrade recommendations") is a compact **anti-staleness footer** — same spirit as the
ai-skill verify-and-subtract invariant. LIFT as a per-reference footer pattern.

**jeffallan/claude-skills@spark-engineer** — MIT. A **well-built THIN ROUTER** (SKILL.md 150
lines + `## Reference Guide` dispatch table `Topic | Reference | Load When` → 5 references/)
— but one narrow compute persona among ~60 unrelated stack-persona skills. Richer frontmatter
than any other found: `metadata: {author, version, domain: data-ml, triggers, role, scope,
output-format, related-skills}`. The **`metadata.related-skills` cross-link field** (→
python-pro, sql-pro, devops-engineer) is a lift-worthy router-frontmatter idea. This is the
**closest thing to our target router mechanics**, but scoped to one persona, not the data
domain.

**melodic-software/claude-code-plugins@data-modeling** — **does not exist** (verified via repo
tree + `gh api search/code`, 0 results). False lead.

**lyndonkl/claude@data-schema-knowledge-modeling** — no LICENSE file. One narrow schema-design
skill (157 lines + resources/) buried in a 248-skill generalist mega-repo. LIFT: **Schema
Types decision table** (Relational/Document/Graph/Event-Time-Series/Dimensional × best-for /
pattern / key-decision), **hierarchy-modeling options checklist** (adjacency list / nested
sets / path enumeration / closure table + trade-offs), tight Do/Don't guardrails. Scope is
schema only — no ETL/orchestration/quality/governance.

**Cluster-2 verdict:** none is a unified jobs×surfaces data router. Closest = jeffallan's
spark-engineer for router *mechanics* + gordonmurray's Update Checklist for *anti-staleness
discipline*, but neither is scoped to the data-engineering domain. **Wedge open.**

### Cluster 3 — Vendor packs (dbt-labs, posthog, astronomer, google)

**dbt-labs/dbt-agent-skills** — official dbt. 3 plugins (`skills/dbt` 9 daily skills,
`skills/dbt-extras`, `skills/dbt-migration`). **No super-router SKILL.md**; the README table +
each skill's "Do NOT use for" section act as de-facto dispatcher (peer skills reference each
other by name). Ships an **`evals/` harness** (scenario-based A/B skill-quality tests — notable
meta-pattern worth copying). All targets are **thin router + references/**. Strict frontmatter
schema enforced by repo `CLAUDE.md` (`name` must equal dir name; only
`name/description/allowed-tools/compatibility/license/metadata/user-invocable` allowed).
Lift-worthy NEUTRAL patterns:
- **"STOP — is this a breaking change?" pre-flight gate** (`using-dbt-for-analytics-engineering`
  line 14) — reroutes to governance before any edit. Portable as a neutral schema/contract
  change detector.
- **Spec-version routing table** (`building-dbt-semantic-layer` lines 32-52): detect existing
  config → classify legacy vs latest → cross with tool-version matrix. Generic "detect
  convention then branch" pattern for any semantic/metrics layer (Cube, LookML, MetricFlow).
- **Entry-point taxonomy** (business-question-first / model-first / open-ended) + **metric-type
  taxonomy** (simple/derived/cumulative/ratio/conversion) — vendor-neutral semantic vocabulary.
- **4-tier NL-to-data fallback** (`answering-natural-language-questions-with-dbt` lines 26-72,
  mermaid): semantic layer → modified compiled SQL → model discovery → manifest/catalog. An
  excellent tool-agnostic "progressively degrade capability" framework.
- **Unit-test decision framework** (when to / when not to test, by logic complexity — regex,
  date math, window fns, joins) rather than blanket coverage. Common Mistakes / Red Flags
  tables recur across all skills.

**posthog/skills** — MIT. Large auto-generated matrix (context-mill tool, "updated on each
release"): `skills/posthog/all` = 90+ per-framework skills; `skills/omnibus/` = ~90 ops skills
(our 3 targets). No top-level data router. Our targets are thin/self-contained. Description
frontmatter is a multi-line YAML block with baked-in "Use when…" triggers + quoted example
prompts (heavier trigger-engineering than dbt). **Strongest neutral-pattern set of the four:**
- **Skill triage triad**: `setting-up-a-data-warehouse-source` (create) →
  `auditing-warehouse-data-health` (fleet-wide scan) → `diagnosing-failed-warehouse-syncs`
  (single-item root cause). A clean 3-tier neutral workflow shape to structurally copy.
- **Sync-type decision table** (`setting-up-a-data-warehouse-source` lines 163-169): full_refresh
  vs incremental vs append vs CDC vs webhook, keyed by table shape (small/dimension,
  large+updated_at, append-only/logs, near-real-time). Completely vendor-neutral ingestion
  taxonomy — LIFT wholesale into a "choosing a sync strategy" reference.
- **Severity-ranked, cascade-aware issue taxonomy** (`auditing-warehouse-data-health`) —
  source-failure-cascades-downstream; billing-limit-is-not-technical; "empty = healthy, don't
  invent problems"; audit → grouped/prioritized report → **hand off, never auto-remediate**.
- **Root-cause lookup table** (`diagnosing-failed-warehouse-syncs` lines 89-103): error-substring
  → root cause → fix (auth expiry, schema drift, missing PK for incremental, CDC prereqs,
  network). Reusable neutral diagnostic-template structure.
- **Recovery-action taxonomy with explicit destructive-action gating** (Steps 4-5, lines
  107-190): 9 lettered scenarios each → specific action, hard rule "never resync first,
  confirm before destructive ops." Directly portable governance discipline.

**posthog/posthog-for-claude** — separate, smaller, **NO LICENSE file at all** (note this).
Single `posthog-instrumentation` skill, 71 lines, monolithic code-dump — nothing structurally
novel. Skip as a lift source.

**astronomer/agents** — Apache 2.0. Single plugin (`astronomer-data`) = MCP server + ~30 skills.
The `airflow` skill (394 lines) is an **explicit router** ("entrypoint that routes to sibling
skills for authoring, testing, deploying, migrating") with real hooks wiring (SessionStart
warms cache, Stop shuts down a background Jupyter kernel). **Closest of the four to genuine
router semantics — but single-vendor (Airflow/Astro).** LIFT:
- **`warehouse-init`** (346 lines) — an 8-step schema-discovery pipeline: read config → parallel
  subagent codebase doc search → parallel per-DB warehouse introspection → categorical
  value-family discovery → merge → generate `warehouse.md` → seed lookup cache → offer to inject
  a "Quick Reference" table into CLAUDE.md/AGENTS.md. A strong neutral "prime the agent on an
  unknown warehouse" workflow.
- `tracing-upstream-lineage` (137 lines) — stepwise lineage-tracing workflow.

**google/skills** — `datalineage-bigquery-asset-impact-analysis` implements a **blast-radius /
downstream-impact workflow** (given a changed asset, enumerate everything downstream before
acting) — a lift-worthy neutral "impact analysis before schema change" pattern; `bigquery-basics`
is an intro reference. Vendor-scoped (BigQuery). _(License not captured in transcript — verify.)_

**Cluster-3 verdict:** all four are purely vendor-scoped; none is a general jobs×surfaces data
skill. astronomer's `airflow` has real router semantics but only across Airflow tasks. These
packs are the **richest mine for neutral patterns** we can lift into a vendor-neutral skill.

---

## Lift-worthy summary

Ranked patterns to mine into the vendor-neutral `data` skill (all confirmed in deep-read):

1. **Google blast-radius / downstream-impact workflow** — `datalineage-bigquery-asset-impact-analysis`.
   "Before changing an asset, enumerate everything downstream." Neutral core of any schema/
   contract-change safety gate.
2. **PostHog diagnose taxonomy + destructive-action gating** — `diagnosing-failed-warehouse-syncs`
   error-substring→cause→fix lookup table + 9-scenario recovery taxonomy with "never resync
   first, confirm before destructive ops." Best governance-discipline artifact found.
3. **dbt STOP gate + 4-tier NL fallback** — "STOP, is this a breaking change?" pre-flight
   reroute, plus semantic-layer→compiled-SQL→model-discovery→manifest progressive-degradation
   flow. Both tool-agnostic in structure.
4. **PostHog sync-type decision table** — full_refresh / incremental / append / CDC / webhook
   keyed by table shape. Vendor-neutral ingestion taxonomy, liftable wholesale.
5. **Astronomer `warehouse-init`** — 8-step parallel-subagent schema-discovery → `warehouse.md`
   + CLAUDE.md Quick-Reference injection. Neutral "prime the agent on an unknown warehouse."
6. **spark-engineer router idiom** — thin SKILL.md + `Topic | Reference | Load When` dispatch
   table + `metadata.related-skills` cross-links. The router *mechanics* to adopt.
7. **gordonmurray per-reference "Update Checklist" footer** — compact anti-staleness mechanism
   per reference file; mirrors the ai-skill verify-and-subtract invariant.
8. **dbt `evals/` harness** — scenario-based A/B skill-quality tests; a meta-pattern for
   validating our own skill.
9. Supporting decision tables: data-quality-dimensions taxonomy (wshobson), schema-types +
   hierarchy-modeling checklist (lyndonkl), bigquery-pipeline cost-audit rubric (awesome-copilot),
   metric-type + entry-point taxonomies (dbt).

**Two content defects observed in competitors** (our lint targets): (1) wshobson
`data-quality-frameworks` — orphaned truncated Python fragment as broken heading at a
content-split seam; (2) awesome-copilot `postgresql-optimization` — mojibake `�` in 3 headers.
Both are automation-without-QA failures; our build must lint reference seams and encoding.

---

## Competitive-landscape signal

Official vendor skill packs now exist for: **dbt, PostHog, Google/BigQuery, Airflow
(Astronomer), Databricks, DuckDB/MotherDuck, Snowflake, AWS, Confluent, Grafana, Looker,
DataHub, Metabase, Elastic, Datadog, Supabase.** This density confirms strong demand for
data tooling as skills — but every pack is single-vendor. The vendor-neutral, decision-first
orchestration layer that routes across all of them (choose warehouse vs lakehouse, when
CDC vs batch, ELT modeling discipline, event taxonomy, governance) is unclaimed.

**Licenses (deep-read confirmed):** MIT — wshobson/agents, github/awesome-copilot,
aradotso/data-skills, gordonmurray/data-engineering-skills, jeffallan/claude-skills,
posthog/skills. Apache 2.0 — astronomer/agents. **No LICENSE file** — posthog/posthog-for-claude
(do not lift; unlicensed), lyndonkl/claude. Not captured in transcript (verify before lifting):
dbt-labs/dbt-agent-skills, google/skills. All lifting must respect attribution.

**Deep-read coverage gap (as-of 2026-07-25):** vendor packs confirmed vendor-scoped by deep
read = dbt, PostHog, Astronomer/Airflow, Google. **NOT yet deep-read** (structure assumed from
listing only, verify before final wedge claim): **Snowflake, Databricks, Fivetran, Airbyte,
Segment**, plus AWS/Confluent/DuckDB/Elastic/Datadog. None appeared as a unified neutral router
in discovery, but their internal structure is unconfirmed — flag for a follow-up pass if the
wedge claim needs to be airtight.
