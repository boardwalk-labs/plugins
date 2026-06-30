# Contributing

Thanks for helping improve the Boardwalk plugins. This repo ships the same Boardwalk integration to five agent harnesses (Claude Code, Codex, Cursor, OpenClaw, and OpenCode) from a single shared source tree.

## Repository layout

One repo, one canonical plugin payload under `plugins/boardwalk/`, four per-harness manifests at the top level:

```
plugins/                                  # repo root
├── .claude-plugin/{plugin.json, marketplace.json}
├── .cursor-plugin/plugin.json
├── .agents/plugins/marketplace.json      # Codex marketplace catalog
├── openclaw.plugin.json
└── plugins/boardwalk/                     # canonical plugin payload
    ├── .codex-plugin/plugin.json          # Codex manifest (lives inside the payload so codex-plugin's installer finds it)
    └── skills/                            # shared skills (consumed by every harness)
        ├── boardwalk-use-cli/SKILL.md
        └── write-good-loops/SKILL.md
```

The Codex marketplace installer (`npx codex-plugin add …`) hard-codes `<repo>/plugins/<plugin-name>/` as the source path it copies into `~/.codex/plugins/<plugin-name>/`, so the Codex manifest must live inside `plugins/boardwalk/.codex-plugin/`. The other three manifest-based harnesses read manifests from the repo root and follow each manifest's `"skills"` field into `./plugins/boardwalk/skills/`. The skills are single-source-of-truth: any change lands in every harness at once with no copy step.

OpenCode adds no manifest. It loads Agent Skills natively, so its users point it at the same `plugins/boardwalk/skills/` folder directly (the repo README's Install section covers it). It is therefore not part of the version-sync set the CI checks below enforce.

## Prerequisites

- Git
- At least one supported harness installed: Claude Code, Codex, Cursor, OpenClaw, or OpenCode
- Node.js ≥ 24 (only if you want to run the `boardwalk` CLI against your changes)

## Authoring skills

A skill is a directory under `plugins/boardwalk/skills/<skill-name>/` containing a `SKILL.md` with YAML frontmatter:

- `name` — the skill's kebab-case id (matches the directory name)
- `description` — the trigger. Be specific about *when* a model should reach for it; this is what every harness matches against.
- `allowed-tools` — the tools the skill may use (the CLI skill needs `Bash`)

Keep skill content accurate to the shipped CLI. The source of truth for the CLI surface is [`@boardwalk-labs/cli`](https://www.npmjs.com/package/@boardwalk-labs/cli) — verify commands and flags against it before documenting them.

## Validating locally

```bash
# Claude Code's official validator
npm install -g @anthropic-ai/claude-code
claude plugin validate .
```

CI (`.github/workflows/ci.yml`) runs on every pull request:

- **All manifest JSON is well-formed:** `jq` parses every per-harness manifest (the Claude validator only covers `.claude-plugin/*`, so this is what guards the Cursor / OpenClaw / Codex manifests).
- **Manifest versions are in sync:** every shipped manifest must carry the same `version`. This is what fails the build if you forget one when bumping.
- **Codex marketplace layout:** the exact `plugins/<name>/{.codex-plugin/plugin.json,skills/}` paths the `codex-plugin` installer copies must be in place.
- **Claude Code validation:** `claude plugin validate .`, the official validator.

On every push to `main` it additionally runs a **real Codex install smoke test**: `npx codex-plugin add boardwalk-labs/plugins --global --yes` against the just-pushed commit (the installer clones the repo's default branch), then asserts each plugin landed with its Codex manifest and a `SKILL.md`. It runs on push rather than on pull requests because `codex-plugin add` installs from a GitHub `org/repo`, so the commit has to be on the remote default branch first (fork-PR branches are not).

Bump the `version` field in every manifest together when you change the shipped payload; the version-sync check fails the build otherwise.
