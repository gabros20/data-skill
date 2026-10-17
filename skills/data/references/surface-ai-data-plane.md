# Surface: AI data plane

Purpose: Reshape the 10 primary-job refs for the data plane underneath an AI product — the RAG
corpus modeled as governed ETL, LLM traces/tokens warehoused as a queryable dataset, point-in-time
correctness for any feature/context served to a model, and eval datasets treated as governed data
like any other. Composes with the shipped `ai` skill rather than duplicating its retrieval-behavior
content.

Read when:
- The request is ingesting/chunking/embedding a corpus that an AI product retrieves from — "RAG
  pipeline," "vector store ingestion," "corpus for retrieval."
- The request is warehousing LLM traces, tokens, or cost for analysis — "log LLM calls," "token
  usage dashboard," "trace warehouse."
- The request needs point-in-time-correct features/context for a model (no label/feature leakage
  from the future relative to a query's as-of time).
- The request treats an eval dataset as a first-class governed asset (versioned, access-controlled),
  not a scratch file.

Skip when:
- The ask is retrieval *behavior* — chunking strategy, fusion, rerank, agentic decomposition,
  retrieval eval metrics — that's `ai`'s `retrieval-and-rag.md` territory entirely; this surface
  owns the governed pipeline and the traces/corpus **as datasets**, not how retrieval reasons over
  them.
- The ask is deploying/scaling the vector store itself (uptime, connection pooling, backup) — that's
  `backend`'s served-infrastructure concern; see §1's three-way seam.
- No AI product is actually involved — the ask is a standard product's own event stream — →
  [surface-product-analytics.md](surface-product-analytics.md).
- GDPR/CCPA/HIPAA/SOC2 applies — stack the additive [surface-regulated.md](surface-regulated.md) on
  top; §4 flags the regulated-RAG-specific note.

Inputs:
- Which of the three seam layers (§1) the request is actually about — retrieval behavior, served
  infrastructure, or governed dataset — state it before answering.
- The corpus's freshness/update cadence, and whether PII or regulated content can appear in source
  documents.
- Whether trace/token data needs to be queried for cost, quality, or product analytics beyond
  eval-time debugging.

Produces:
- A RAG-corpus ingestion pipeline shaped as governed ETL: landing table, PII/poisoning screens,
  dedup, chunking, source ACL and step versions per chunk, atomic index-version switch, with
  lineage and freshness like any other dataset.
- A trace/token warehousing schema (Trace → Observation hierarchy) queryable for cost and quality
  analysis, not just point-in-time debugging.
- A point-in-time-correctness discipline for any feature or context served to a model.
- Eval datasets versioned and access-controlled like production data.

## Contents
- The three-way retrieval seam, restated
- RAG corpus as governed ETL
- LLM traces/tokens as a warehoused dataset
- Point-in-time correctness
- Job impact table

## Procedure

### 1. The three-way retrieval seam, restated
State this once, canonically, before doing any work at this surface: retrieval **behavior**
(chunking, fusion, rerank, decomposition, retrieval eval) is `ai`'s. The vector store **as served
infrastructure** (deployment, connection pooling, backup, uptime) is `backend`/`operate`'s. The
vector store **and LLM traces as governed datasets** — the ingestion pipeline, source-of-truth
lineage, PII classification, access control on what gets indexed, and the trace/token warehouse — is
`data`'s. A single "the RAG app is slow" or "answers are stale" complaint can be any of the three;
name which layer a given question actually belongs to before absorbing it into this surface's scope.
This mirrors `ai`'s own `surface-rag-app.md` §3 — read that reference's framing of the same seam
from the retrieval-behavior side when composing with a live `ai` build, and consume `ai`'s
`handoff.yaml` (vector-store schema and retrieval-freshness assumptions) rather than re-deriving
what the retrieval design already decided.

### 2. RAG corpus as governed ETL
Model corpus ingestion as ETL, not a one-off script: raw source documents land in a table first —
preserving data, traceability, and auditing — before any transformation touches them. The pipeline
stages, in order: **ingestion** (raw landing) → **parsing** (explicit PII/HIPAA tagging as metadata,
not an afterthought) → **enrichment** → **dedup** (MinHash/LSH catches near-duplicate documents
before they double-count in retrieval) → **filtering**, including PII-detection-driven exclusion
**and data-poisoning screening** (untrusted or adversarial source content is a real ingestion-time
risk for a corpus a model will trust) → **chunking** (fixed/paragraph/format/semantic — no
one-size-fits-all; the chunking *strategy* choice itself is `ai`'s, per §1, but the chunking *job*
running as a scheduled, monitored pipeline step is this surface's) → **embedding** (model tradeoffs,
truncation risk) → **indexing**, with chunks and metadata co-located so the index itself carries
lineage back to its source document. Give the corpus the same freshness/lineage contract any other
dataset gets and PII classification at ingest (a decoupled detect→act pipeline: classify first, then
decide masking/exclusion as a separate step) — a governed RAG corpus is not exempt from the same
trust-gate discipline this skill applies everywhere else.

**Capture the source ACL at ingest** on the content's incremental sync cursor (commit records, then
advance it), so each chunk carries who may see it. **Version every derived step** — parser, chunking
strategy, embedding model — per chunk; changing one is a re-index, not an in-place edit. **Publish a
complete index version before switching queries**: build beside the live index, then repoint in one
atomic alias swap (elastic.co/docs/manage-data/data-store/aliases, 2026-10-17). **A revocation is
not a deletion**: update the current ACL first; then purge derived chunks, vectors, and caches
([governance-pii-and-retention.md](governance-pii-and-retention.md) §3's tombstone order). The
query-time filter is `ai`'s `surface-rag-app.md` "The three-way retrieval border" and `backend`'s
`persistence.md` *"Postgres for everything" — and its traps*.

### 3. LLM traces/tokens as a warehoused dataset
Model LLM observability data with an explicit hierarchy, not a flat log: **Trace** (one full request
lifecycle) → **Observations** (nested LLM/tool/retrieval calls within it, each carrying timing,
input/output, and cost) → **Sessions** (multi-turn grouping) → **Environments** (deployment stage).
Track token usage and cost natively per observation, not bolted on after the fact — this is the
substrate an eval suite and a cost dashboard both query against. Use the OpenTelemetry `gen_ai.*`
semantic conventions as the wire format when traces need to interoperate across tools: attributes
like `gen_ai.usage.total_tokens`, `gen_ai.usage.token_type`, `gen_ai.is_streaming`,
`gen_ai.response.finish_reason`, plus vector-DB-specific attributes
(`db.vector.query.{top_k,embeddings_count,result_count}`), give retrieval steps first-class spans
alongside generation spans — the standard, not a vendor log shape. SDKs that batch and queue
traces need an explicit flush in short-lived/serverless processes, or buffered traces are silently
lost on exit.

### 4. Point-in-time correctness
Any feature or context served to a model must be point-in-time-correct relative to the query's
as-of time — never embed or serve a document/feature version "from the future" relative to when the
query was made, the same label/feature-leakage discipline a feature store enforces for training data
(the historical-features join pattern: join to feature values as they existed at the event
timestamp, not the latest value). This applies to RAG corpora exactly as it applies to any served
feature: a corpus reindexed after a query shouldn't retroactively change what that query "should
have" retrieved when the answer is being audited or replayed.

**Regulated-RAG note:** when GDPR/CCPA/HIPAA/SOC2 applies to this AI product (→
[surface-regulated.md](surface-regulated.md)), PII tagging at the parsing stage (§2) is the gate
regulated content must pass before it reaches an embedding index — an erasure request against a
source document has to propagate to the vector index and any cached trace containing it, not just
the warehouse table.

### 5. Job impact table
| Job | How this surface bends it |
|---|---|
| [ingestion-and-movement.md](ingestion-and-movement.md) | The RAG corpus pipeline (§2) *is* this job's content at this surface — an incremental ETL job with schema-contract discipline (evolve/freeze/discard), not a one-off import script. |
| [data-contracts-and-schema-evolution.md](data-contracts-and-schema-evolution.md) | The trace schema (Trace/Observation/Session/Environment, §3) and the chunk+metadata schema (§2) are contracts like any other — version them, don't let an embedding-model swap silently change the index's schema underneath consumers. |
| [quality-and-testing.md](quality-and-testing.md) | Dedup (MinHash/LSH) and poisoning screens (§2) are this surface's quality gate — applied at ingestion, before content reaches an index a model will trust. |
| [lineage-and-observability.md](lineage-and-observability.md) | Index staleness (time since last reindex vs. corpus update cadence) is monitored as a first-class signal — the same discipline `ai`'s `surface-rag-app.md` names from the retrieval side; this surface owns producing that signal. |
| [governance-pii-and-retention.md](governance-pii-and-retention.md) | PII tagging happens at the parsing stage (§2), before chunking — classify, then decide exclusion/masking as a separate step (a decoupled detect→act model), not conflated into one pass. |
| [platform-and-cost.md](platform-and-cost.md) | Token/cost data is a first-class warehoused dataset (§3), not just a debugging log — query it the same way any cost-governance dashboard queries warehouse spend. |

## Validation
- The three-way seam (§1) is named explicitly whenever a request could plausibly belong to more than
  one of retrieval-behavior / served-infrastructure / governed-dataset.
- The corpus pipeline has a landing table, PII tagging, dedup, and a poisoning screen before
  chunking — not chunking applied directly to raw source content.
- Chunks carry source ACL and step versions; index versions switch atomically; revocation precedes
  cleanup.
- Traces are queryable by Trace/Observation, with token/cost tracked per observation, not just
  logged as unstructured text.
- Any point-in-time-sensitive feature or context is joined as-of the query time, not to the latest
  available value.
- If regulated content is in scope, the PII-tagging gate (§2/§4) is confirmed to run before content
  reaches the embedding index.

## Failure modes and handoff
- **Chunking applied directly to raw source content** with no landing table, PII tagging, dedup, or
  poisoning screen first — the corpus is ungoverned ETL wearing a RAG label; route back to §2.
- **LLM traces logged as unstructured text** instead of the Trace/Observation hierarchy with
  per-call token/cost — this surface's warehousing is exactly what makes cost and quality queries
  possible; a flat log doesn't.
- **A retrieval-behavior question absorbed into this surface's scope** (or vice versa) — restate the
  seam (§1) explicitly rather than silently answering from the wrong layer.
- **A document version served that's "from the future" relative to the query's as-of time** — the
  point-in-time discipline (§4) was skipped; this corrupts both eval results and production answers
  the same way feature leakage corrupts a training set.
- **A revoked user still retrieves a document** — the revocation waited for cleanup instead of
  reaching the current ACL first (§2).
- **Regulated content indexed before PII tagging ran** — the erasure obligation now has to reach
  into the vector index and cached traces, not just a warehouse table; catch this at ingestion (§4's
  regulated-RAG note), not after the fact.
