# Boardwalk plugins

Official plugins that let agent harnesses drive [Boardwalk](https://boardwalk.sh): shared skills for the `boardwalk` CLI, plus the hosted control-plane MCP server.

Five shared skills for **Claude Code, Codex, and Cursor**: `boardwalk-overview` (the platform mental model), `boardwalk-use-cli` (the first-party CLI: scaffold, run, validate, deploy, trigger, operate), `write-good-workflows` (authoring quality), `write-good-loops` (iterating agent loops), and `equip-agents` (skills, tools, MCP, memory, and human-input gates inside a workflow). **OpenClaw and OpenCode** currently wire the `boardwalk-use-cli` skill. For Claude Code the plugin also connects the [remote MCP server](#remote-mcp-server) so the model can create, schedule, trigger, and monitor workflows directly.

## Layout

The canonical plugin payload (skills + Codex manifest) lives under `plugins/boardwalk/` so the Codex marketplace installer (`npx codex-plugin add`) finds it where it expects. Per-harness manifests for the other three harnesses live at the repo root and reference the same shared skills:

- `.claude-plugin/` — Claude Code plugin + marketplace manifest
- `.mcp.json` — remote MCP server config (Claude Code auto-loads it when the plugin is enabled)
- `.cursor-plugin/` — Cursor plugin manifest
- `.agents/plugins/` — Codex marketplace catalog
- `openclaw.plugin.json` — OpenClaw plugin manifest
- `plugins/boardwalk/.codex-plugin/` — Codex plugin manifest
- `plugins/boardwalk/skills/` — shared skill definitions (all harnesses point here)

OpenCode needs no manifest: it loads Agent Skills natively, so you link the `boardwalk-use-cli` skill in directly (see [OpenCode](#opencode) under Install).

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

## Remote MCP server

For Claude Code, the plugin also bundles the Boardwalk control-plane MCP server (`.mcp.json`, loaded automatically when the plugin is enabled). It talks to `https://api.boardwalk.sh/mcp/v1` and gives the model the full control plane: create and update workflows, add cron / rate / one-shot schedules, trigger runs, tail run output, answer human-input gates, and manage environments, variables, and secrets metadata.

Auth is a Boardwalk API key sent as a bearer header. Create one in the web console (Settings > API keys), then:

```bash
export BOARDWALK_API_KEY=bwk_...
```

Restart Claude Code (or run `/mcp` and reconnect) after setting it. Without the variable the skills still work; only the MCP connection needs it.

Any MCP-capable harness can connect the same server manually: it is a plain HTTP MCP server at `https://api.boardwalk.sh/mcp/v1`, authenticated with an `Authorization: Bearer $BOARDWALK_API_KEY` header. Point your harness's MCP config at that URL and header; from the Claude Code CLI (outside the plugin), for example:

```bash
claude mcp add --transport http boardwalk https://api.boardwalk.sh/mcp/v1 \
  --header "Authorization: Bearer $BOARDWALK_API_KEY"
```

## What it does

Installs five skills. `boardwalk-overview` orients a model that is new to Boardwalk: what the platform is and how workflows, triggers, and runs fit together. `boardwalk-use-cli` gives the model the `boardwalk` CLI surface: scaffolding (`init`), validating (`check`), building the artifact (`build`), authenticating, deploying and triggering (`deploy` / `run` / `cancel`), inspecting runs and usage (`runs` / `usage`), and managing workflows, secrets, environments, variables, and inference providers, plus project linking, auth precedence, the run-event channels, and self-hosting knobs. `write-good-workflows` covers authoring quality: the `run` function and its typed I/O, the `workflow.jsonc` descriptor, SDK primitives, run legibility, efficiency, and surviving restarts. `write-good-loops` teaches the model to author an agent loop that iterates until a goal is reached: the core loop shape, the layered exits every loop needs (verifier, iteration cap, budget, no-progress), bounded-vs-recurring topology, maker/checker verification, durable cross-run state, and not paying to wait. `equip-agents` covers giving an `agent()` call skills, inline tools, MCP servers, memory, and human-input gates. The CLI itself ships separately as [`@boardwalk-labs/cli`](https://www.npmjs.com/package/@boardwalk-labs/cli).

## The Boardwalk repos

- [`boardwalk`](https://github.com/boardwalk-labs/boardwalk) — the open-source single-node engine: cron scheduling, webhooks, durable runs, run history
- [`sdk-typescript`](https://github.com/boardwalk-labs/sdk-typescript) — `@boardwalk-labs/workflow`, the TypeScript API a workflow program imports
- [`cli`](https://github.com/boardwalk-labs/cli) — `boardwalk`: scaffold, validate, deploy, run
- [`examples`](https://github.com/boardwalk-labs/examples) — copyable workflow templates (`boardwalk init --template`)
- [`runner`](https://github.com/boardwalk-labs/runner) — self-hosted runner: your machines execute hosted-scheduled runs

Hosted platform and docs: [boardwalk.sh](https://boardwalk.sh).

## License

MIT — see [LICENSE](./LICENSE).
