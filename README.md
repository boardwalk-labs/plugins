# Boardwalk plugins

Official plugins that let agent harnesses drive the [Boardwalk](https://boardwalk.sh) CLI.

Two shared skills (`boardwalk-use-cli` and `write-good-loops`) packaged for **Claude Code, Codex, Cursor, OpenClaw, and OpenCode**. `boardwalk-use-cli` documents the first-party `boardwalk` CLI so a model can scaffold, run, validate, deploy, trigger, and operate workflows on the user's behalf; `write-good-loops` teaches how to author an agent loop that iterates until a goal is reached (layered exits, bounded-vs-recurring topology, maker/checker verification, durable state).

## Layout

The canonical plugin payload (skills + Codex manifest) lives under `plugins/boardwalk/` so the Codex marketplace installer (`npx codex-plugin add`) finds it where it expects. Per-harness manifests for the other three harnesses live at the repo root and reference the same shared skills:

- `.claude-plugin/` — Claude Code plugin + marketplace manifest
- `.cursor-plugin/` — Cursor plugin manifest
- `.agents/plugins/` — Codex marketplace catalog
- `openclaw.plugin.json` — OpenClaw plugin manifest
- `plugins/boardwalk/.codex-plugin/` — Codex plugin manifest
- `plugins/boardwalk/skills/` — shared skill definitions (all harnesses point here)

OpenCode needs no manifest: it loads Agent Skills natively and reads the same `plugins/boardwalk/skills/` folder directly (see [OpenCode](#opencode) under Install).

The skills are single-source-of-truth: a change lands in every harness at once, with no copy step.

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

### OpenCode

OpenCode has its own plugin system, but it is a different shape: OpenCode plugins are JavaScript/TypeScript hook modules (custom tools and lifecycle hooks) listed under `plugin` in `opencode.json`, not skill bundles, so the Boardwalk skill does not ship as an OpenCode plugin. Instead OpenCode loads Agent Skills natively and reads the same skill with nothing to install. From a checkout of this repo, link the skill into OpenCode's skills directory:

```bash
mkdir -p ~/.config/opencode/skills
ln -s "$(pwd)/plugins/boardwalk/skills/boardwalk-use-cli" \
  ~/.config/opencode/skills/boardwalk-use-cli
```

If you already installed the Claude Code plugin, OpenCode also discovers skills under `~/.claude/skills/`, so the Boardwalk skill is available there with no extra step.

## What it does

Installs two skills. `boardwalk-use-cli` gives the model the `boardwalk` CLI surface: scaffolding (`init`), running locally (`dev`), validating (`check`), bundling (`build`), authenticating, deploying and triggering (`deploy` / `run` / `cancel`), inspecting runs and usage (`runs` / `usage`), and managing workflows, secrets, environments, variables, and inference providers, plus project linking, auth precedence, the run-event channels, and self-hosting knobs. `write-good-loops` teaches the model to author an agent loop that iterates until a goal is reached: the core loop shape, the layered exits every loop needs (verifier, iteration cap, budget, no-progress), bounded-vs-recurring topology, maker/checker verification, durable cross-run state, and not paying to wait. The CLI itself ships separately as [`@boardwalk-labs/cli`](https://www.npmjs.com/package/@boardwalk-labs/cli).

## License

MIT — see [LICENSE](./LICENSE).
