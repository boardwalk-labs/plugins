# Boardwalk plugins

Official plugins that let agent harnesses drive the [Boardwalk](https://boardwalk.sh) CLI.

One shared skill set (`use-boardwalk-cli`) packaged for **Claude Code, Codex, Cursor, and OpenClaw**. The skills document the first-party `boardwalk` CLI so a model can scaffold, run, validate, deploy, trigger, and operate workflows on the user's behalf.

## Layout

The canonical plugin payload (skills + Codex manifest) lives under `plugins/boardwalk/` so the Codex marketplace installer (`npx codex-plugin add`) finds it where it expects. Per-harness manifests for the other three harnesses live at the repo root and reference the same shared skills:

- `.claude-plugin/` — Claude Code plugin + marketplace manifest
- `.cursor-plugin/` — Cursor plugin manifest
- `.agents/plugins/` — Codex marketplace catalog
- `openclaw.plugin.json` — OpenClaw plugin manifest
- `plugins/boardwalk/.codex-plugin/` — Codex plugin manifest
- `plugins/boardwalk/skills/` — shared skill definitions (all harnesses point here)

The skills are single-source-of-truth: a change lands in all four plugins at once, with no copy step.

## Install

### Claude Code

```bash
claude plugin marketplace add boardwalk-labs/plugins
claude plugin install boardwalk@boardwalk-labs
```

### Codex

```bash
npx codex-plugin add boardwalk-labs/plugins
```

Then open `/plugins` in Codex and enable `boardwalk`.

### Cursor

Cursor Marketplace publication is pending. Until then, install from a local checkout by symlinking the repo into your Cursor plugins directory:

```bash
ln -s "$(pwd)" ~/.cursor/plugins/local/boardwalk
```

### OpenClaw

```bash
openclaw plugins install ./
```

## What it does

Installs the `use-boardwalk-cli` skill, which gives the model the `boardwalk` CLI surface: scaffolding (`init`), running locally (`dev`), validating (`check`), bundling (`build`), authenticating, deploying and triggering (`deploy` / `run` / `cancel`), inspecting runs and usage (`runs` / `usage`), and managing workflows, secrets, and inference providers, plus project linking, auth precedence, the run-event channels, and self-hosting knobs. The CLI itself ships separately as [`@boardwalk-labs/cli`](https://www.npmjs.com/package/@boardwalk-labs/cli).

## License

MIT — see [LICENSE](./LICENSE).
