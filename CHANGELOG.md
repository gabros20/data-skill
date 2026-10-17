# Changelog

All notable changes to **data** are documented here.

The release procedure synchronizes `.codex-plugin/plugin.json`, this changelog, git tag
`v<version>`, and the matching GitHub Release. Runtime `SKILL.md` contains no version metadata.

## [Unreleased]

### Added
- Queue unreleased changes here.

## [0.2.0] — 2026-10-17

### Added
- **surface-ai-data-plane** §2 *RAG corpus as governed ETL* sharpened: source ACL captured at
  ingest on the incremental sync cursor; parser, chunking and embedding versioned per chunk (a
  change is a re-index); a complete index version published before queries switch, atomically;
  permission revocation as deny-at-query-time against the current ACL before derived cleanup.
  Produces, Validation and a failure mode follow; new output eval `rag-corpus-governance-artifact`.

## [0.1.0] — 2026-07-25

### Added
- Initial release: faceted-router `data` skill — 10 primary jobs × 5 surfaces
  (regulated as additive overlay) + handoff, 16 references, 9 assets.
- Flagship reference: quality-and-testing (convergent 9-bucket DQ taxonomy,
  known/unknown-unknowns layering, PR-time blast-radius review).
- Universal invariants: contract-first; trust-as-a-gate; small-data-first;
  narrow-the-DAG; one-metric-one-definition; platform-layer governance;
  SRM-before-trust; retrieval-first/anti-staleness; subtract-the-buzzword.
- Behavioral eval suites: activation, traversal, output, compression-ablation.
- Research corpora + synthesis in `research/` (4 channels, date-stamped 2026-07-25).

[0.2.0]: https://github.com/gabros20/data-skill/releases/tag/v0.2.0
[0.1.0]: https://github.com/gabros20/data-skill/releases/tag/v0.1.0
