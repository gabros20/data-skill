# Schema Compatibility Cheatsheet (fillable)

Confluent Schema Registry's seven named compatibility modes (enumerated from
`CompatibilityLevel.java`) — the definitive answer to "which schema change is safe, and who
upgrades first." Source:
[data-contracts-and-schema-evolution.md](../references/data-contracts-and-schema-evolution.md).
Re-verify against `docs.confluent.io` before a hard claim — registry defaults and mode names are
the kind of fact that drifts.

| Mode | Rule | Who upgrades first |
|---|---|---|
| `NONE` | No checking | — fast migrate, only safe with no external consumers |
| `BACKWARD` (registry default) | New schema can read data written with the *previous* schema | consumers upgrade first |
| `BACKWARD_TRANSITIVE` | New schema can read data written with *all* previous schemas | consumers first |
| `FORWARD` | *Previous* schema can read data written with the new schema | producers upgrade first (edge devices, shipped SDKs you don't control) |
| `FORWARD_TRANSITIVE` | *All* previous schemas can read data written with the new schema | producers first |
| `FULL` | Backward **and** forward compatible vs the previous schema | either order — independent |
| `FULL_TRANSITIVE` | Backward **and** forward compatible vs *all* previous schemas | either order, but bidirectional coupling — the strictest, safest default |

- **Default to the strictest mode you can afford** (`FULL`/`FULL_TRANSITIVE`) — it's the only pair
  that doesn't force a coordinated upgrade order between producer and consumer teams.
- **Non-transitive modes only check against the immediately-previous version**, not the whole
  history — a chain of individually-compatible changes can still break an old consumer several
  versions back. Use the transitive variant whenever consumers might lag by more than one version.
- Compatibility is set **per-subject** (topic/schema), not globally — audit per subject, don't
  assume one registry-wide setting covers everything.

## Pick a mode per contract

| Contract / topic | Mode chosen | Who upgrades first (confirmed) | Reason |
|---|---|---|---|
| | | | |

## Companion evolution conventions (use alongside, not instead of, the mode above)

- **`MODEL-REVISION-ADDITION` (SchemaVer, Snowplow; re-verify the literal term against docs.snowplow.io)** — a breaking change bumps `MODEL` (old
  consumers keep working against the old MODEL in parallel); an additive-compatible change bumps
  `REVISION`/`ADDITION` only. Good for human-facing versioning even when the registry mode above
  is the enforced mechanism.
- **Schema-contract mode (dlt-style, per table/column/data-type):** `evolve` (accept and adapt) ·
  `freeze` (reject the record) · `discard` (drop the offending row/column). Pick one per producer
  pipeline, don't leave it implicit.

**Self-audit:** every contract/topic above has a chosen mode, a confirmed upgrade order, and a
stated reason — "we've always used BACKWARD" is not a reason.
