# Contributing

Thanks for helping improve the Boardwalk plugins. This repo ships the same Boardwalk integration to four agent harnesses — Claude Code, Codex, Cursor, and OpenClaw — from a single shared source tree.

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
    └── skills/                            # shared skills (consumed by all four harnesses)
        └── use-boardwalk-cli/SKILL.md
```

The Codex marketplace installer (`npx codex-plugin add …`) hard-codes `<repo>/plugins/<plugin-name>/` as the source path it copies into `~/.codex/plugins/<plugin-name>/`, so the Codex manifest must live inside `plugins/boardwalk/.codex-plugin/`. The other three harnesses read manifests from the repo root and follow each manifest's `"skills"` field into `./plugins/boardwalk/skills/`. The skills are single-source-of-truth — any change lands in all four plugins at once with no copy step.

## Prerequisites

- Git
- At least one supported harness installed: Claude Code, Codex, Cursor, or OpenClaw
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

CI runs the same validation plus a Codex install smoke test. Bump the `version` field in every manifest together when you change the shipped payload.
