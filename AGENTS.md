# Agent guide — `data` (Data Engineering)

This repository is the released `data-skill`: an installable agent skill for making product and
business data trustworthy, governed, and usable. The runtime skill lives under `skills/data/`; the
repository and Codex plugin use the `-skill` suffix. Read this before editing.

## Ownership boundary (non-negotiable)

`data` owns the **analytical and governed data plane** — data treated as a product, distinct from
`backend` (data as application state) and `ai` (data as model context). It owns the event contract
and tracking plan, data contracts and schema evolution, ingestion and movement, transformation and
modeling, the metrics/semantic layer and BI, data quality and testing, lineage and observability,
governance/PII/retention, experiment measurement validity, and platform/cost choices. It does **not**
own:

- OLTP schema design, app queries, migrations, serving APIs, the DB/vector store *as served infra*
  → **backend**
- retrieval *behavior* (chunk/rerank tuning), prompts, model evals, agent construction → **ai**
- designing or interpreting an A/B experiment, picking the OEC, growth loops → **growth**
- which outcomes/metrics matter to the business → **product**
- data-flow boundaries, storage strategy, NFR budgets, threat model of the data platform →
  **architecture**
- emitting the events in client/server code → **frontend** / **backend**
- independent verification / sign-off before release → **quality**
- deploying pipelines, running the monitoring platform, incident response infra → **operate**
- deterministic multi-system business processes, even when one step touches the warehouse →
  **automation**

`data` *references* those substrates; it does not re-teach or override them. It works standalone and
never silently invokes a sibling skill.

## Non-negotiable invariants

1. **Retrieval-first / anti-staleness — the flagship.** Warehouse prices, tool versions, spec
   revisions, and licenses churn quarterly. Every volatile fact is date-stamped and re-verified
   against the live source before use — read the LICENSE *file*, never a repo's API license field;
   never freeze a volatile fact or assert one as "verified-live" inside a static doc. Treat any
   pricing/version table (including this repo's) as illustrative until re-checked.
2. **Trust is a gate, not a dashboard.** Untested data is broken data. Every primary key gets
   `unique` + `not_null` as a floor; validate at ingestion and quarantine violations. Rule tests
   catch known unknowns, statistical/anomaly baselines catch unknown unknowns — ship both layers.
3. **Small-data-first.** Default to a single node until measured pain; a distributed system must
   beat one laptop thread (COST) before its complexity is justified.
4. **Contract-first + one-metric-one-definition.** Every dataset/event ships an explicit schema,
   version, and evolution rule; every metric has exactly one authoritative definition, consumed by
   every tool and agent.
5. **Verify-and-subtract, not just assemble.** Every recommendation names what to build, what NOT to
   build, and what to remove. Cap each pass at 3–5 decisions.

## Required behavior

1. Run `scripts/check-sync` (and `scripts/lint-skill`) before any release; never replace the generic
   gate with domain-only checks.
2. Keep runtime frontmatter to `name` and `description`. Versions/metadata live in
   `.codex-plugin/plugin.json`, `CHANGELOG.md`, tags, and releases — never in `SKILL.md`.
3. Keep `SKILL.md` the direct router: one primary job × ≤1 base surface (+ additive regulated
   overlay), plus the "Not this skill →" decline table. References are flat, directly linked,
   self-describing, loaded only when their conditions apply.
4. Every reference begins with `Purpose / Read when / Skip when / Inputs / Produces`; add a
   `## Contents` when it exceeds ~100 lines. Dense is acceptable; some refs intentionally exceed the
   2,500-token soft ceiling.
5. Exhaustive-coverage artifacts (DQ matrices, PII audits, compatibility tables) are uniform tables
   with a self-audit count, never a narrative.
6. Keep repository docs, eval fixtures, research, and site assets outside `skills/data/`.
7. Release through matching plugin version, changelog entry, tag, and GitHub Release; set the GitHub
   About area (family-style description, canonical site alias as homepage, base + domain topics).

Do not mark the repository complete because structural lint passes. Activation, traversal, and output
evaluations must show the skill changes agent behavior usefully.
