---
name: "boardwalk-use-cli"
description: "Use when a user wants to install, configure, authenticate against, or drive the first-party Boardwalk CLI, the `boardwalk` command for authoring, validating, running, shipping, and operating agent workflows. A workflow is a package: a TypeScript or Python `run` function plus a `workflow.jsonc` deployment descriptor, calling `agent(prompt)` and durable primitives (secrets, sleep, phases, artifacts, workflows.call, humanInput) and returning the run's output. Covers install, scaffolding (init, incl. --python), validation (check), building the artifact (build), OAuth login, deploy (deterministic org resolution), triggering and cancelling runs, inspecting runs and usage, human-in-the-loop inputs (inputs/respond), managing workflows, secrets, environments, variables, persistent workspaces, and inference providers, the managed model catalog (models), webhook URLs (header-based, secret never in the URL), self-hosted runners (runner), and project linking."
allowed-tools: Bash
---

# Use the Boardwalk CLI

Use this skill whenever the user needs to install, configure, or drive the first-party `boardwalk` CLI, to scaffold a workflow, validate it, sign in, deploy it, trigger a run, cancel one, inspect runs and usage, or manage workflows, secrets, environments, variables, and inference providers. This is the canonical reference for the CLI surface.

New to Boardwalk? Read the **`boardwalk-overview`** skill first. It covers what the platform is and the workflow mental model (a workflow is a typed function, not YAML), which this reference assumes you already have.

## What a Boardwalk workflow is

A workflow is a **package**: a directory holding `workflow.jsonc` (the deployment descriptor — triggers, permissions, budget, read as data) and an entry exporting a `run` function the platform calls — `src/index.ts` with `export default async function run(input, context)`, or `main.py` with a module-level `async def run`. There is no YAML pipeline and no DSL. Native types on the signature are the I/O contract (the deploy derives their schemas for the dashboard's run form), and the return value is the run's output. The body calls `agent(prompt)` for LLM work (`model` optional, chosen per call) and durable primitives (`secrets.get`, `sleep`, `workflows.call`, `humanInput`) for everything else. See `boardwalk-overview` for the full model, `write-good-workflows` for authoring it well, and `equip-agents` for giving an `agent()` skills, tools, MCP, and memory.

**Write plain TypeScript — `Date.now()` and `Math.random()` just work.** There is no determinism gate. On the hosted fleet a `sleep` or `humanInput` snapshots the whole machine and resumes the exact heap, so a wait loses nothing. A crash (or a wait on a substrate without snapshots) restarts the program from the top, Lambda-style, so make side effects safe to re-run: idempotent keys or upserts, and put must-not-repeat work behind `workflows.call` (which re-attaches to a finished child instead of running it twice). An already-answered `humanInput` gate is never re-asked.

## Installation

Requires **Node.js ≥ 24**.

```bash
# Zero-install execution
npx @boardwalk-labs/cli --help

# Or install globally: exposes the `boardwalk` command
npm install -g @boardwalk-labs/cli
boardwalk --help
```

`boardwalk setup` is the one-command onboarding: it logs in, detects your coding agent (Claude Code, Codex, Cursor, ...), and installs its Boardwalk plugin + MCP server.

## The author loop

The CLI is built around a tight loop: **scaffold → validate → run → inspect.** Iterate with `boardwalk check` (validate the package, no auth), `boardwalk deploy . --org <org> --run` (deploy + trigger a real run), and `boardwalk runs <id> --logs`/`--follow` (inspect it). There is no local run mode: unit tests call `run(input)` directly over `installTestHost` stubs, and live execution is deploy + trigger — point it at a dev environment with `--environment` while iterating. To run workflows on your own machine, use a self-hosted runner (`boardwalk runner`, below).

### `boardwalk init [dir]`: scaffold a project

```bash
boardwalk init my-workflow          # scaffold into ./my-workflow
boardwalk init                      # scaffold into the current directory
boardwalk init my-workflow --python # a Python workflow (main.py + pyproject.toml)
boardwalk init my-workflow --template <name>
```

Creates the two-file package: `workflow.jsonc`, `src/index.ts` (typed `run` function), `README.md`, `package.json`, `tsconfig.json`, and `.gitignore`. `--python` swaps in `main.py` + `pyproject.toml` (dependencies resolve with `uv` at build time). `--template <name>` selects the starting point from the examples registry and defaults to the built-in `hello`, which works offline. It writes the package and nothing else: it does not vendor skills into `.claude/`, so if the project's agent needs the CLI in context, install the plugin (`claude plugin install boardwalk@boardwalk-labs`). It never overwrites existing files, and keeps a `README.md` you already have.

The scaffold is deliberately minimal: a `manual` trigger and one `agent()` call, no commented-out options. Reach for the JSON schema (`https://boardwalk.sh/schemas/workflow.json`, which every field describes) rather than expecting the generated file to enumerate what is available.

**Fill in the scaffolded `README.md`.** It ships with the package on every deploy and becomes the workflow's landing page in the dashboard, beside the config from `workflow.jsonc`. It is the only place a reader learns what the workflow is *for*: the descriptor can say how it is configured and nothing more. See `write-good-workflows` for what belongs in it.

### `boardwalk check <dir>`: validate without running

```bash
boardwalk check .
```

Everything a deploy does except the upload, all local: validates `workflow.jsonc` against the descriptor schema (the same one the server enforces, including the `concurrency.key` template syntax), esbuild-compiles the entry proving every import resolves (strip-only — your body is never type-checked), and packs the artifact. The I/O schemas derive at deploy (the backend reads the `run()` signature and returns warnings). No auth, no network, so it is safe in CI on every commit. `deploy` and `build` run the same validation.

A **Python** package resolves and freezes its dependencies here, so `check` needs `uv` on PATH — and a reasonably current one. The build always resolves wheels for the runner's CPython and platform, not yours, so your own Python version doesn't matter; but when a package has no matching wheel, uv falls back to building it from source *for your machine*, and that layer can't import on the runner.

### `boardwalk build <dir>`: build the deploy artifact

```bash
boardwalk build .                 # writes <slug>.tgz in the cwd
boardwalk build . --out dist/wf.tgz
```

Produces the exact content-addressed `.tgz` a deploy uploads: the bundled entry, the descriptor, the source tree, `skills/**` + `README.md` + the descriptor's `files` assets, and the TypeScript types harvest the backend derives the I/O schemas from. Deploy builds it for you; reach for `build` to inspect what ships.

## Run-event channels

Every engine emits the same typed event stream on five channels: `lifecycle`, `phase`, `output`, `log`, `agent`. The default view is quiet (`lifecycle` + `phase` + `output`); `--verbose` adds `agent` turns, tool calls, and `log`; `--stream <channels>` picks a subset (e.g. `--stream output`). The same flags work on `runs --logs`/`--follow`.

## Authenticate

```bash
boardwalk login                   # browser OAuth (PKCE) → stores a least-privilege session
boardwalk login --scopes admin    # elevated session: manage secrets and inference providers
boardwalk login --token bwk_xxx   # store an API key instead of the browser flow
boardwalk whoami                  # show the current stored session + the account's orgs
boardwalk status                  # API host + login (verified live) + project link
boardwalk logout                  # remove local credentials
```

`login` opens a browser for an OAuth PKCE flow and stores a scoped session locally. The default login is least-privilege (deploy, trigger, read runs, list secret/provider names). Writing secrets and wiring inference providers need `--scopes admin` (you must be an org admin); admin-only actions like deleting a workflow work from any login, gated on your org role. For headless/CI use, pass an API key (`bwk_...`) with `--token`, or set `BOARDWALK_API_KEY`.

## Ship it

### `boardwalk deploy <dir>`: create or update a workflow

```bash
boardwalk deploy . --org my-team
boardwalk deploy . --org my-team --dry-run   # print the plan (create vs update), write nothing
boardwalk deploy . --token bwk_xxx           # use this Bearer token for this call
boardwalk deploy . --yes                     # CI: skip the create confirmation
```

The org is resolved deterministically, never guessed: `--org` wins; else a single-org credential's scope is unambiguous; else the project link (below); else a hard error listing your orgs. A deploy that would **create** a new workflow confirms interactively first (`--yes` skips, for CI); updates never prompt. The workflow's identity is `(org, slug)` — the descriptor carries no org, so a fork deploys to whoever runs it. The server derives the I/O schemas from your `run` signature and prints any derivation warnings.

**Python packages deploy only from here.** The MCP tools and the web Code tab build server-side, which can't materialize a Python dependency layer, so they refuse a `.py` package (the web Code tab renders it read-only). TypeScript deploys from all three.

### `boardwalk run <workflow>`: run a DEPLOYED workflow, wait for the result

```bash
boardwalk run my-workflow --input '{"who":"world"}'      # --input becomes run(input)
boardwalk run my-workflow --no-wait                      # trigger and exit without waiting
boardwalk run my-workflow --environment Production       # run against an environment (its secrets + variables)
boardwalk run 01KV4SMQ0JFCNH9X4VQVW10STZ                 # by id, as in a dashboard URL
```

`<workflow>` is a **slug or a workflow id**, not a directory: `run` reads nothing from disk — no package, no build, no deploy — so it works from any machine that has a login, on a workflow you have no local copy of. Pass `--org` only when your login covers more than one org.

While authoring, `boardwalk deploy <dir> --run` does both in one step (it takes the same `--input` / `--environment` / `--no-wait`):

```bash
boardwalk deploy . --org my-team --run --input '{"who":"world"}'
```

`--environment <name>` picks which environment's secrets and variables the run resolves (omit = the org base). See `boardwalk environments` / `boardwalk variables` below.

### `boardwalk cancel <runId>`: cancel a queued or in-flight run

```bash
boardwalk cancel run_01H...
```

## Inspect & operate

Once a workflow is deployed, drive and observe it from the terminal instead of the dashboard (or curl).

### `boardwalk runs`: list runs, show one, or stream its log

```bash
boardwalk runs                              # the org's recent runs
boardwalk runs --workflow my-flow --status failed --limit 20
boardwalk runs <runId>                      # one run's summary (status, duration, tokens, error)
boardwalk runs <runId> --logs               # its event log (--verbose / --stream for tools + raw turns)
boardwalk runs <runId> --follow             # live-tail over SSE until it finishes (Ctrl-C aborts)
boardwalk runs <runId> --json               # raw JSON, for piping
```

Acting on a single run by id needs no `--org`. The run resolves its own org.

### `boardwalk inputs` / `boardwalk respond`: human-in-the-loop

A workflow can call `humanInput()` to pause for a person to approve, choose, or answer. A paused run goes `awaiting_input` and shows up in the org inbox until someone responds; answering resumes it.

```bash
boardwalk inputs                              # the org-wide inbox of inputs awaiting a response
boardwalk inputs <runId>                      # just one run's pending inputs
boardwalk respond <runId> <key> --value "ship it"        # a text / single-choice gate
boardwalk respond <runId> <key> --values approve,notify  # a multi-select gate
boardwalk respond <runId> <key> --other "something else" # the open "Other..." entry
```

The `key` comes from `boardwalk inputs`. A run resumes once every input in its batch is answered.

### `boardwalk usage`: spend and activity

```bash
boardwalk usage --org my-team               # runs, compute, tokens, credit, autonomy, cache-hit rate
boardwalk usage --org my-team --days 30 --json
```

For a single run's live budget state from *inside* the program, the SDK's `usage.get()` is the hook (see `write-good-loops`).

### `boardwalk workflows`: inspect and manage

```bash
boardwalk workflows                         # the org's workflows (slug, title, triggers, last run)
boardwalk workflows show <id|slug>          # manifest projection + version history
boardwalk workflows disable <id|slug>       # pause every trigger (reversible)
boardwalk workflows enable <id|slug>        # resume a disabled workflow's triggers
boardwalk workflows delete <id|slug> --yes  # irreversible; needs an elevated login
```

### `boardwalk workspace`: a workflow's persistent state

```bash
boardwalk workspace show my-flow            # what it's storing, per environment: size + last written
boardwalk workspace reset my-flow --yes     # clear it; the workflow, triggers, and history stay
```

A workflow opts into persistence with `"workspace": { "persist": [...] }` in `workflow.jsonc` or an `agent({ memory })` call. Reset exists because state that compounds eventually compounds something wrong (a poisoned cache, a memory that learned the wrong lesson); `--environment` addresses one environment's copy.

### `boardwalk webhooks`: the org's inbound endpoints

```bash
boardwalk webhooks                          # list: name, URL, verification scheme (no secrets)
boardwalk webhooks create <name>            # create one; the signing secret is shown ONCE (admin)
boardwalk webhooks rotate <name>            # new secret, shown ONCE; the old one stops working
boardwalk webhooks delete <name> --yes      # remove it and its secret (admin)
```

A webhook is an **org-level endpoint**, not a property of a workflow: create it once, point a sender at its URL, then attach any number of workflows with `{ "kind": "webhook", "name": "<name>" }` in the descriptor's `triggers`. **Every attached workflow runs on every delivery** — to split events between workflows, create a second webhook and choose which events go where on the sender's side (Stripe, Sentry, PagerDuty and most senders have an event picker).

The **secret is never in the URL**. It rides in a header per the webhook's verification preset: `token` sends it verbatim in `X-Boardwalk-Token`, `custom_header` in a header you name, `signature` as an HMAC-SHA256 of the raw body in `X-Boardwalk-Signature: sha256=<hex>`, and the provider presets (`github`/`stripe`/`slack`/`linear`/`sentry`/`pagerduty`/`standard_webhooks`) verify that sender's own scheme — for those, pass the sender's key with `--secret` (or `--secret -` to read stdin) rather than minting one.

Naming a webhook the org hasn't created yet is **not** a deploy failure: the workflow deploys and shows as not-connected until you create it.

For **GitHub, Linear, Jira and Notion** there is no webhook to create at all: those are provider triggers (`{ "kind": "github", "event": "pr.merged" }`), and the platform owns the subscription, the verification, and the dedupe. Connecting is an OAuth or app-install flow in the browser, so **there is no `boardwalk connections` command** — send the user to **Connections** in the dashboard, where the same page carries the inbound-delivery log and replay. A workflow declaring one deploys before the connection exists. See `write-good-workflows` for the event vocabularies.

### `boardwalk secrets`: manage the org's secrets (values never returned)

```bash
boardwalk secrets                                   # names / scope / kind only
echo "$TOKEN" | boardwalk secrets set GITHUB_TOKEN  # pipe the value → stays out of shell history
boardwalk secrets set DEPLOY_KEY --from-file ./key  # ...or from a file (--value also accepted)
boardwalk secrets set MY_KEY --scope org --kind api_key --description "..."
boardwalk secrets delete GITHUB_TOKEN --yes
```

Writing or deleting secrets needs `boardwalk login --scopes admin`. `--scope` is `org` (default) or `user`; `--kind` is `api_key` (default), `oauth_token`, `aws_role`, or `mcp_credential`.

### `boardwalk environments` / `boardwalk variables`: environment config (GitHub-Actions style)

The org keeps **secrets** (encrypted credentials, read in code via `secrets.get`) and non-secret **variables** (injected into the run as `process.env`), organized into **environments**. The **organization base** applies to every run; a named **environment** (e.g. `Production`) holds its own secrets + variables. A run **targets one environment** and resolves its config from it, **falling back to the org base**. The same name can hold a different value per environment. Pick the environment per run with `boardwalk run <workflow> --environment <name>` (omit = the org base); it is NOT a descriptor field.

```bash
boardwalk environments                       # named environments (the org base always applies underneath)
boardwalk environments create Production
boardwalk environments delete Production --yes

boardwalk variables                          # non-secret variables, VALUES are shown (they're not secret)
boardwalk variables set POSTHOG_PROJECT_ID 475542 --environment Production
boardwalk variables list --environment Production
boardwalk variables delete REGION --yes
```

A program reads a variable with `process.env.NAME` and a secret with `await secrets.get("NAME")`. When `process.env.X` is empty at runtime it is a *where-it's-set vs which-environment-the-run-targeted* mismatch: the variable is set in an environment the run didn't target (or only at the base), not a code/parsing bug. Fix by running against that environment (`--environment <name>`), or setting it on the org base so every run gets it. Writing/deleting needs an elevated login; never store a credential as a variable.

### `boardwalk inference`: manage BYO inference providers

```bash
boardwalk inference                                       # providers (endpoints only, never keys)
echo "$KEY" | boardwalk inference add my-openai --source openai
boardwalk inference add vllm --source openai_compatible --base-url https://vllm.internal
boardwalk inference delete my-openai --yes
```

`--source` is required: `bedrock`, `anthropic`, `google`, `openai`, `openai_compatible`, or `azure_openai` (plus `--base-url`, `--region`, `--api-version`, `--api-key` as the source needs). The name is what an `agent({ provider })` call routes to. Adding or deleting needs `--scopes admin`.

### `boardwalk models`: browse the managed model catalog

```bash
boardwalk models                                  # the managed lane's most-capable models, with prices
boardwalk models --all                            # every supported model
boardwalk models --search claude                  # filter by id or display name
boardwalk models show anthropic/claude-opus-4.8   # one model's price, context window, support
```

The catalog is the set of models an `agent({ model })` call can name on the managed lane (no key of yours needed). It is read-only and needs no elevated login. Add `--json` to any of these to pipe the raw record.

## Run on your own machine (`boardwalk runner`)

A workflow can run on your own hardware instead of the hosted fleet: declare `"runs_on": { "kind": "self-hosted" }` in `workflow.jsonc`, then run a self-hosted runner that claims those runs.

```bash
boardwalk runner start --org my-team        # register THIS machine (a plain admin login is enough) + go online
boardwalk runner start --host               # run WITHOUT isolation: full machine access (trusted only)
boardwalk runner list --org my-team         # the org's runners
boardwalk runner remove <runnerId> --yes    # deregister
```

Runs are containerized by default (`--host` opts out); Ctrl-C drains. For fleets, mint a one-time token with `boardwalk runner pools token` and redeem it on each machine with `boardwalk runner register`.

## Project linking (the `--org` flag becomes optional)

The first successful `deploy`/`run` writes a per-directory link at `.boardwalk/project.json` (`{ orgSlug, workflowId }`) and adds `.boardwalk/` to `.gitignore`. It is a local cache, never a source of truth — the org always comes from `--org`, a single-org credential, or this link, in that order, and a multi-org credential with none of them is a hard error, never a guess. Once a directory is linked, `--org` is optional and subsequent commands target the same workflow. Deploy each separate project from its own directory so the links don't clobber each other.

## Quick reference

| Command | Purpose |
| --- | --- |
| `boardwalk init [dir] [--python] [--template <name>]` | Scaffold a new workflow package |
| `boardwalk check <dir>` | Validate the package locally (no auth, no network) |
| `boardwalk build <dir> [--out <path>]` | Build the deployable artifact (`.tgz`) |
| `boardwalk setup` | One-command onboarding: login + coding-agent plugin + MCP |
| `boardwalk login [--scopes admin] [--token bwk_...]` | Authenticate (browser OAuth, or store an API key) |
| `boardwalk whoami` / `boardwalk status` / `boardwalk logout` | Inspect / verify / clear credentials |
| `boardwalk deploy <dir> [--org <slug>] [--dry-run] [--yes]` | Create or update a workflow |
| `boardwalk run <workflow> [--org <slug>] [--input <json>] [--environment <name>] [--no-wait]` | Run a DEPLOYED workflow (slug or id; no local copy) |
| `boardwalk deploy <dir> --run [--input <json>] [--environment <name>]` | Ship it, then run it once (the authoring loop) |
| `boardwalk cancel <runId>` | Cancel a queued or in-flight run |
| `boardwalk runs [runId] [--logs] [--follow] [--json]` | List runs, show one, or stream its log |
| `boardwalk inputs [runId] [--json]` | List human-in-the-loop inputs awaiting a response |
| `boardwalk respond <runId> <key> [--value\|--values\|--other ...]` | Answer a pending input, resuming the run |
| `boardwalk usage [--org <slug>] [--days <n>] [--json]` | Org spend and activity |
| `boardwalk workflows [list\|show\|disable\|enable\|delete] ...` | Inspect and manage workflows |
| `boardwalk workspace [show\|reset] <workflow> ...` | Inspect / clear a workflow's persistent workspace |
| `boardwalk webhooks [create\|rotate\|delete]` | Manage the org's inbound webhook endpoints |
| `boardwalk secrets [list\|set\|delete] ...` | Manage the org's secrets (admin to write) |
| `boardwalk environments [list\|create\|delete] ...` | Manage named environments (config sets a run targets) |
| `boardwalk variables [list\|set\|delete] [--environment <name>] ...` | Manage non-secret variables (`process.env`) |
| `boardwalk inference [list\|add\|delete] ...` | Manage BYO inference providers (admin to write) |
| `boardwalk models [list\|show] ...` | Browse the managed model catalog for `agent()` |
| `boardwalk runner start [--pool <name>] [--host] [--once]` | Run workflows on THIS machine (self-hosted runner) |
| `boardwalk runner [list\|remove\|register\|pools token] ...` | Manage self-hosted runners and fleet registration |
