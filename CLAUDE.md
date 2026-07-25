# Claude Code repository guide

This is the released `data-skill` (runtime skill `data`, under `skills/data/`). Read [AGENTS.md](AGENTS.md)
for the ownership boundary and non-negotiable invariants — especially **retrieval-first /
anti-staleness**: never present a warehouse price, tool version, spec revision, or license as fact
without date-stamping it and re-verifying against the live source (read the LICENSE file, never a
repo's API license field).

Run `scripts/check-sync` before any release. Keep `SKILL.md` the router; keep versions out of it.
