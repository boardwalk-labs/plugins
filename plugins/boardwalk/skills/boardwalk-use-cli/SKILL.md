---
name: "boardwalk-use-cli"
description: "Use when a user wants to install, configure, authenticate against, or drive the first-party Boardwalk CLI, the `boardwalk` command for authoring, validating, running, shipping, and operating agent workflows. A workflow is a TypeScript/JavaScript program file whose pure-literal `meta` compiles to the manifest and that calls `agent(prompt)` plus durable primitives (secrets, sleep, phases, output, artifacts, workflows.call, humanInput, step.run, now/random/uuid). Covers install, scaffolding (init), local run (dev) and validation (check, including the determinism gate), bundling (build), OAuth login, deploy, triggering and cancelling runs, inspecting runs and usage, human-in-the-loop inputs (inputs/respond), managing workflows, secrets, environments, variables, and inference providers, the managed model catalog (models), webhook URLs (header-based, secret never in the URL), self-hosted runners (runner), and project linking."
allowed-tools: Bash
---

# Use the Boardwalk CLI

Use this skill whenever the user needs to install, configure, or drive the first-party `boardwalk` CLI, to scaffold a workflow, run it locally, validate it, sign in, deploy it, trigger a run, cancel one, inspect runs and usage, or manage workflows, secrets, environments, variables, and inference providers. This is the canonical reference for the CLI surface.

New to Boardwalk? Read the **`boardwalk-overview`** skill first. It covers what the platform is and the workflow mental model (a workflow is a TypeScript program, not YAML), which this reference assumes you already have.

## What a Boardwalk workflow is

A workflow is a **TypeScript/JavaScript program file** (or a package directory), whose pure-literal `meta` compiles to the **manifest** (the control-plane contract). There is no YAML and no DSL. The body calls `agent(prompt)` for LLM work (`model` optional, chosen per call) and durable primitives (`secrets.get`, `sleep`, `output`, `workflows.call`, `humanInput`, `step.run`, and the durable `now`/`random`/`uuid`) for everything else. See `boardwalk-overview` for the full model and `equip-agents` for giving an `agent()` skills, tools, MCP, and memory.

**The program must be deterministic across a restart.** A run restarts from the top on a crash and replays from the top on a resume (after a `sleep` or `humanInput`), so a value that changes on the second pass silently corrupts the run. Bare `Date.now()`, `new Date()`, `performance.now()`, `Math.random()`, and `crypto.randomUUID()`/`getRandomValues()` are therefore **blocked**. Use the SDK's durable `await now()`, `await random()`, and `await uuid()`, which record their result once and return the same value on replay. `check`, `deploy`, and `run` enforce this gate (`build`/`dev` only warn); `--allow-nondeterminism` overrides it. Raw I/O like bare `fetch` is advisory-only (legitimate in a non-suspending script), but wrap it in `step.run` if it precedes a `sleep`, so a resume doesn't re-fire it.

## Installation

Requires **Node.js ≥ 24**.

```bash
# Zero-install execution
npx @boardwalk-labs/cli --help

# Or install globally: exposes the `boardwalk` command
npm install -g @boardwalk-labs/cli
boardwalk --help
```

## The author loop

The CLI is built around a tight local-first loop: **scaffold → run locally → validate → ship.**

### `boardwalk init [dir]`: scaffold a project

```bash
boardwalk init my-workflow          # scaffold into ./my-workflow
boardwalk init                      # scaffold into the current directory
boardwalk init my-workflow --template <name>
```

Creates a starter workflow: the program file, `package.json`, `.env.example`, and `.gitignore`. `--template <name>` selects the starting point and defaults to the built-in `hello`. It never overwrites existing files.

### `boardwalk dev <file|dir>`: run it now, locally

```bash
boardwalk dev ./index.ts
boardwalk dev ./index.ts --input '{"who":"world"}'   # trigger payload exposed to the program as `input`
boardwalk dev ./index.ts --env .env.local            # resolve secrets from this env file (default: .env)
boardwalk dev ./index.ts --verbose                   # stream EVERY channel (agent turns, tool calls, logs)
boardwalk dev ./index.ts --stream output | jq        # just the result, pipe-friendly
boardwalk dev ./index.ts --stream phase,log
boardwalk dev ./index.ts --org my-team               # org to bill managed inference to
boardwalk dev ./index.ts --token bwk_xxx             # mint the inference key with this bearer (CI/headless)
```

`dev` derives and validates the manifest (precise errors before anything runs), bundles the program, executes it in-process, and streams the run-event log. Secrets resolve from `.env` (or `--env <path>`) and their **values never print**.

**Managed inference needs a login.** An `agent()` call that names no provider uses Boardwalk's managed lane, which the local engine reaches with a short-lived key minted from your `boardwalk login` session (billed to `--org`, or the project link's org). So a managed-`agent()` run needs a one-time login; an agent-free workflow, or one whose `agent()` names its own provider key, runs with **no account**.

**Exit code is the run's verdict:** `0` completed, `1` failed, `130` cancelled (Ctrl-C). That makes `dev` usable as a CI gate.

### `boardwalk check <file|dir>`: validate without running

```bash
boardwalk check ./index.ts
boardwalk check ./index.ts --allow-nondeterminism   # downgrade the determinism gate to a warning
```

Everything `dev` validates, without executing: full manifest-schema validation (the same schema every engine enforces), an esbuild compile proving every import resolves, **and the determinism gate**, a blocking error on bare `Date.now()`/`Math.random()`/`crypto.randomUUID()` and friends (fix with `now()`/`random()`/`uuid()`, or pass `--allow-nondeterminism`). No auth, no network, so it is safe in CI on every commit. `deploy` and `run` run the same gate; `build` and `dev` print the warnings without blocking.

### `boardwalk build <file|dir>`: bundle to one deployable file

```bash
boardwalk build ./index.ts                 # writes <slug>.mjs in the cwd
boardwalk build ./index.ts --out dist/wf.mjs
```

Bundles the workflow to a single `.mjs` (the SDK left external, `meta` intact). This is what a self-hosted server loads from its `BOARDWALK_WORKFLOWS_DIR`.

## Run-event channels

Every engine emits the same typed event stream on five channels: `lifecycle`, `phase`, `output`, `log`, `agent`. The default view is quiet (`lifecycle` + `phase` + `output`); `--verbose` adds `agent` turns, tool calls, and `log`; `--stream <channels>` picks a subset (e.g. `--stream output`). The same flags work on `dev` and on `runs --logs`/`--follow`.

## Authenticate

```bash
boardwalk login                   # browser OAuth (PKCE) → stores a least-privilege session
boardwalk login --scopes admin    # elevated session: manage secrets/providers, delete workflows
boardwalk login --token bwk_xxx   # store an API key instead of the browser flow
boardwalk whoami                  # show the current stored session (scope + expiry)
boardwalk status                  # API host + login (verified live) + project link
boardwalk logout                  # remove local credentials
```

`login` opens a browser for an OAuth PKCE flow and stores a scoped session locally. The default login is least-privilege (deploy, trigger, read runs, list secret/provider names). Writing secrets, wiring inference providers, and deleting workflows need `--scopes admin` (you must be an org admin). For headless/CI use, pass an API key (`bwk_...`) with `--token`, or set `BOARDWALK_API_KEY`.

## Ship it

### `boardwalk deploy <file|dir>`: create or update a workflow

```bash
boardwalk deploy ./index.ts --org my-team
boardwalk deploy ./index.ts --org my-team --dry-run   # print the plan (create vs update), write nothing
boardwalk deploy ./index.ts --token bwk_xxx           # use this Bearer token for this call
```

### `boardwalk run <file|dir>`: deploy, trigger a real run, wait for the result

```bash
boardwalk run ./index.ts --org my-team --input '{"who":"world"}'
boardwalk run ./index.ts --org my-team --no-wait      # trigger and exit without waiting
boardwalk run ./index.ts --org my-team --environment Production   # run against an environment (its secrets + variables)
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

### `boardwalk workflows`: inspect and manage

```bash
boardwalk workflows                         # the org's workflows (slug, title, triggers, last run)
boardwalk workflows show <id|slug>          # manifest projection + version history
boardwalk workflows disable <id|slug>       # pause every trigger (reversible)
boardwalk workflows enable <id|slug>        # resume a disabled workflow's triggers
boardwalk workflows delete <id|slug> --yes  # irreversible; needs an elevated login
```

### `boardwalk webhook <id|slug>`: inbound webhook URL

```bash
boardwalk webhook <id|slug>                 # show the inbound URL + verification scheme (no secret)
boardwalk webhook <id|slug> --rotate        # regenerate the secret and reveal it ONCE (admin)
```

The **secret is never in the URL**. The URL is the bare workflow endpoint, safe to share. The secret rides in a header per the trigger's verifier preset: `token` sends it verbatim in `X-Boardwalk-Token`, `custom_header` in a caller-named header, `signature` as an HMAC-SHA256 of the raw body in `X-Boardwalk-Signature: sha256=<hex>`, and the provider presets (`github`/`stripe`/`slack`/`linear`) verify that provider's own signing scheme. Plain `webhook <ref>` prints the endpoint and scheme but no secret; `--rotate` (admin-gated) mints a new secret, invalidates the old one, and reveals the new value a single time, so reconfigure the sender afterward. A workflow gets a webhook by declaring `{ kind: "webhook", auth: "token" }` (or another preset) in `meta.triggers`.

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

The org keeps **secrets** (encrypted credentials, read in code via `secrets.get`) and non-secret **variables** (injected into the run as `process.env`), organized into **environments**. The **organization base** applies to every run; a named **environment** (e.g. `Production`) holds its own secrets + variables. A run **targets one environment** and resolves its config from it, **falling back to the org base**. The same name can hold a different value per environment. Pick the environment per run with `boardwalk run --environment <name>` (omit = the org base); it is NOT a manifest field.

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

A workflow can run on your own hardware instead of the hosted fleet: declare `runs_on: { kind: "self-hosted" }` in `meta`, then run a self-hosted runner that claims those runs.

```bash
boardwalk runner start --org my-team        # register THIS machine (a plain admin login is enough) + go online
boardwalk runner start --host               # run WITHOUT isolation: full machine access (trusted only)
boardwalk runner list --org my-team         # the org's runners
boardwalk runner remove <runnerId> --yes    # deregister
```

Runs are containerized by default (`--host` opts out); Ctrl-C drains. For fleets, mint a one-time token with `boardwalk runner pools token` and redeem it on each machine with `boardwalk runner register`.

## Project linking (the `--org` flag becomes optional)

The first successful `deploy`/`run` writes a per-directory link at `.boardwalk/project.json` (`{ orgSlug, workflowId }`) and adds `.boardwalk/` to `.gitignore`. Once a directory is linked, `--org` is optional and subsequent commands target the same workflow, a Vercel-style, rename-safe project identity. Deploy each separate project from its own directory so the links don't clobber each other.

## Quick reference

| Command | Purpose |
| --- | --- |
| `boardwalk init [dir] [--template <name>]` | Scaffold a new workflow project |
| `boardwalk dev <file\|dir>` | Run the workflow now, locally |
| `boardwalk check <file\|dir>` | Validate locally (no auth, no network) |
| `boardwalk build <file\|dir> [--out <path>]` | Bundle to one deployable `.mjs` |
| `boardwalk login [--scopes admin] [--token bwk_...]` | Authenticate (browser OAuth, or store an API key) |
| `boardwalk whoami` / `boardwalk status` / `boardwalk logout` | Inspect / verify / clear credentials |
| `boardwalk deploy <file\|dir> [--org <slug>] [--dry-run]` | Create or update a workflow |
| `boardwalk run <file\|dir> [--org <slug>] [--input <json>] [--environment <name>] [--no-wait]` | Deploy + trigger a real run |
| `boardwalk cancel <runId>` | Cancel a queued or in-flight run |
| `boardwalk runs [runId] [--logs] [--follow] [--json]` | List runs, show one, or stream its log |
| `boardwalk inputs [runId] [--json]` | List human-in-the-loop inputs awaiting a response |
| `boardwalk respond <runId> <key> [--value\|--values\|--other ...]` | Answer a pending input, resuming the run |
| `boardwalk usage [--org <slug>] [--days <n>] [--json]` | Org spend and activity |
| `boardwalk workflows [list\|show\|disable\|enable\|delete] ...` | Inspect and manage workflows |
| `boardwalk webhook <id\|slug> [--rotate]` | Show / rotate a workflow's inbound webhook URL |
| `boardwalk secrets [list\|set\|delete] ...` | Manage the org's secrets (admin to write) |
| `boardwalk environments [list\|create\|delete] ...` | Manage named environments (config sets a run targets) |
| `boardwalk variables [list\|set\|delete] [--environment <name>] ...` | Manage non-secret variables (`process.env`) |
| `boardwalk inference [list\|add\|delete] ...` | Manage BYO inference providers (admin to write) |
| `boardwalk models [list\|show] ...` | Browse the managed model catalog for `agent()` |
| `boardwalk runner start [--pool <name>] [--host] [--once]` | Run workflows on THIS machine (self-hosted runner) |
| `boardwalk runner [list\|remove\|register\|pools token] ...` | Manage self-hosted runners and fleet registration |
