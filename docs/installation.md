# Installation

Install the `data` runtime pack from the independently versioned
`gabros20/data-skill` repository.

## Prerequisites

- An Agent Skills-compatible client.
- TODO: Add only real skill-specific tools, credentials, or runtime requirements.

## Install with skills.sh

```bash
npx skills add gabros20/data-skill
```

## Clone and install

```bash
git clone https://github.com/gabros20/data-skill.git
cd data-skill
./install.sh codex
```

Available targets:

| Argument | Destination |
|---|---|
| `codex` | `${CODEX_HOME:-$HOME/.codex}/skills/data/` |
| `agents` | `~/.agents/skills/data/` |
| `claude` | `~/.claude/skills/data/` |
| `cursor` | `~/.cursor/skills/data/` |
| `antigravity` | Gemini IDE and Antigravity CLI skill paths |
| `opencode` | `~/.config/opencode/skills/data/` |
| `grok` | `~/.grok/skills/data/` |
| `hermes` | `~/.hermes/skills/data/` |
| `all` | Claude, Codex, and the cross-agent path |

The installer stages a complete copy before replacing an existing installation. If replacement
fails, it restores the previous copy.

## Verify

Start a new client session and use the client's supported invocation form:

- Codex: `$data`
- Slash-command clients: `/data`
- Other clients: an `@` mention, skill tool, or natural-language trigger

For Codex, verify the installed files directly:

```bash
test -f "${CODEX_HOME:-$HOME/.codex}/skills/data/SKILL.md"
```

## Update

```bash
cd data-skill
git pull --ff-only
./install.sh codex
```

Review `CHANGELOG.md` and GitHub Releases for version history. Version metadata intentionally stays
outside runtime `SKILL.md`.

## Uninstall

```bash
rm -rf "${CODEX_HOME:-$HOME/.codex}/skills/data"
```
