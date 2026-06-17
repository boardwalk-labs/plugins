---
name: "use-boardwalk-cli"
description: "Use when a user wants to install, configure, authenticate against, or drive the first-party Boardwalk CLI — the `boardwalk` command for authoring, validating, running, shipping, and operating agent workflows. A Boardwalk workflow is a TypeScript/JavaScript program file whose pure-literal `meta` compiles to the manifest; it calls `agent(prompt)` for LLM work and durable primitives (secrets, sleep, phases, output, artifacts, workflows.call) for everything else. Covers installation, scaffolding (init), running locally (dev), local validation (check), bundling (build), browser OAuth login, deploying, triggering runs, cancelling, inspecting runs and usage, managing workflows/secrets/inference providers, webhook URLs, project linking, auth precedence, the run-event channels, and self-hosting knobs."
allowed-tools: Bash
---

# Use the Boardwalk CLI

Use this skill whenever the user needs to install, configure, or drive the first-party `boardwalk` CLI — to scaffold a workflow, run it locally, validate it, sign in, deploy it, trigger a run, cancel one, inspect runs and usage, or manage workflows, secrets, and inference providers. This is the canonical reference for the CLI surface.

## What a Boardwalk workflow is

A workflow is a **TypeScript/JavaScript program file** (e.g. `index.ts`) — or a package directory containing one. The program exports a pure-literal `meta` object that the platform compiles to the **manifest** (the control-plane contract: slug, optional title, triggers, runtime). The program body does the work:

- `agent(prompt, opts?)` runs an LLM loop. **`model` is optional and chosen per call:** omit it to use Boardwalk's managed inference lane, or name one (e.g. `anthropic/claude-sonnet-4.5`) to pin it. `provider` is optional too — it defaults to the managed `boardwalk` lane; name your own (a built-in vendor or a configured provider) to use BYO keys. The workflow itself declares **no** model or provider.
- Durable primitives — `secrets.get`, `sleep`, phases, `output`, `artifacts.write`, `workflows.call` — run in deterministic code. Secrets never reach the LLM context.

There is no YAML and no DSL: the program file *is* the source of truth, and `meta` is a derived projection of it.

## Installation

Requires **Node.js ≥ 24**.

```bash
# Zero-install execution
npx @boardwalk-labs/cli --help

# Or install globally — exposes the `boardwalk` command
npm install -g @boardwalk-labs/cli
boardwalk --help
```

## The author loop

The CLI is built around a tight local-first loop: **scaffold → run locally → validate → ship.**

### `boardwalk init [dir]` — scaffold a project

```bash
boardwalk init my-workflow          # scaffold into ./my-workflow
boardwalk init                      # scaffold into the current directory
boardwalk init my-workflow --template <name>
```

Creates a starter workflow: the program file, `package.json`, `.env.example`, and `.gitignore`. `--template <name>` selects the starting point and defaults to the built-in `hello`. It never overwrites existing files.

### `boardwalk dev <file|dir>` — run it now, locally

```bash
boardwalk dev ./index.ts
boardwalk dev ./index.ts --input '{"who":"world"}'   # trigger payload exposed to the program as `input`
boardwalk dev ./index.ts --env .env.local            # resolve secrets from this env file (default: .env)
boardwalk dev ./index.ts --verbose                   # stream EVERY channel (agent turns, tool calls, logs)
boardwalk dev ./index.ts --stream output | jq        # just the result — pipe-friendly
boardwalk dev ./index.ts --stream phase,log
boardwalk dev ./index.ts --org my-team               # org to bill managed inference to
```

`dev` derives and validates the manifest (precise errors before anything runs), bundles the program, executes it in-process, and streams the run-event log. Secrets resolve from `.env` (or `--env <path>`) and their **values never print**.

**Managed inference needs a login.** An `agent()` call that names no provider uses Boardwalk's managed lane, which the local engine reaches with a short-lived key minted from your `boardwalk login` session (billed to `--org`, or the project link's org). So a managed-`agent()` run needs a one-time login; an agent-free workflow, or one whose `agent()` names its own provider key, runs with **no account**.

**Exit code is the run's verdict:** `0` completed, `1` failed, `130` cancelled (Ctrl-C). That makes `dev` usable as a CI gate.

### `boardwalk check <file|dir>` — validate without running

```bash
boardwalk check ./index.ts
```

Everything `dev` validates, without executing: full manifest-schema validation (the same schema every engine enforces) plus an esbuild compile proving every import resolves. No auth, no network — safe in CI on every commit.

### `boardwalk build <file|dir>` — bundle to one deployable file

```bash
boardwalk build ./index.ts                 # writes <slug>.mjs in the cwd
boardwalk build ./index.ts --out dist/wf.mjs
```

Bundles the workflow to a single `.mjs` (the SDK left external, `meta` intact). This is what a self-hosted server loads from its `BOARDWALK_WORKFLOWS_DIR`.

## Run-event channels

Every engine — local `dev` and the hosted platform alike — emits the same typed event stream. Each event belongs to one **channel**: `lifecycle`, `phase`, `output`, `log`, `agent`. The flags mean the same thing everywhere (`dev`, and `runs --logs`/`--follow`):

- default → `lifecycle` + `phase` + `output` (quiet, readable)
- `--verbose` → all five channels, including `agent` turns and tool calls
- `--stream <channels>` → a comma-separated subset (e.g. `--stream output` for just the result)

## Authenticate

```bash
boardwalk login                   # browser OAuth (PKCE) → stores a least-privilege session
boardwalk login --scopes admin    # elevated session: manage secrets/providers, delete workflows
boardwalk login --token bwk_xxx   # store an API key instead of the browser flow
boardwalk whoami                  # show the current stored session (scope + expiry)
boardwalk status                  # API host + login (verified live) + project link
boardwalk logout                  # remove local credentials
```

`login` opens a browser for an OAuth PKCE flow and stores a scoped session locally. The default login is least-privilege (deploy, trigger, read runs, list secret/provider names). Writing secrets, wiring inference providers, and deleting workflows need `--scopes admin` (you must be an org admin). For headless/CI use, pass an API key (`bwk_…`) with `--token`, or supply one per-command (below).

## Ship it

### `boardwalk deploy <file|dir>` — create or update a workflow

```bash
boardwalk deploy ./index.ts --org my-team
boardwalk deploy ./index.ts --org my-team --dry-run   # print the plan (create vs update), write nothing
boardwalk deploy ./index.ts --token bwk_xxx           # use this Bearer token for this call
```

### `boardwalk run <file|dir>` — deploy, trigger a real run, wait for the result

```bash
boardwalk run ./index.ts --org my-team --input '{"who":"world"}'
boardwalk run ./index.ts --org my-team --no-wait      # trigger and exit without waiting
```

### `boardwalk cancel <runId>` — cancel a queued or in-flight run

```bash
boardwalk cancel run_01H…
```

## Inspect & operate

Once a workflow is deployed, drive and observe it from the terminal instead of the dashboard (or curl).

### `boardwalk runs` — list runs, show one, or stream its log

```bash
boardwalk runs                              # the org's recent runs
boardwalk runs --workflow my-flow --status failed --limit 20
boardwalk runs <runId>                      # one run's summary (status, duration, tokens, error)
boardwalk runs <runId> --logs               # its event log (--verbose / --stream for tools + raw turns)
boardwalk runs <runId> --follow             # live-tail over SSE until it finishes (Ctrl-C aborts)
boardwalk runs <runId> --json               # raw JSON, for piping
```

Acting on a single run by id needs no `--org` — the run resolves its own org.

### `boardwalk usage` — spend and activity

```bash
boardwalk usage --org my-team               # runs, compute, tokens, credit, autonomy, cache-hit rate
boardwalk usage --org my-team --days 30 --json
```

### `boardwalk workflows` — inspect and manage

```bash
boardwalk workflows                         # the org's workflows (slug, title, triggers, last run)
boardwalk workflows show <id|slug>          # manifest projection + version history
boardwalk workflows disable <id|slug>       # pause every trigger (reversible)
boardwalk workflows enable <id|slug>        # resume a disabled workflow's triggers
boardwalk workflows delete <id|slug> --yes  # irreversible; needs an elevated login
```

### `boardwalk webhook <id|slug>` — inbound webhook URL

```bash
boardwalk webhook <id|slug>                 # show the inbound URL for a webhook-triggered workflow
boardwalk webhook <id|slug> --rotate        # regenerate the secret, reveal the working URL once (admin)
```

### `boardwalk secrets` — manage the org's secrets (values never returned)

```bash
boardwalk secrets                                   # names / scope / kind only
echo "$TOKEN" | boardwalk secrets set GITHUB_TOKEN  # pipe the value → stays out of shell history
boardwalk secrets set DEPLOY_KEY --from-file ./key  # …or from a file (--value also accepted)
boardwalk secrets set MY_KEY --scope org --kind api_key --description "…"
boardwalk secrets delete GITHUB_TOKEN --yes
```

Writing or deleting secrets needs `boardwalk login --scopes admin`. `--scope` is `org` (default) or `user`; `--kind` is `api_key` (default), `oauth_token`, `aws_role`, or `mcp_credential`.

### `boardwalk inference` — manage BYO inference providers

```bash
boardwalk inference                                       # providers (endpoints only, never keys)
echo "$KEY" | boardwalk inference add my-openai --source openai
boardwalk inference add vllm --source openai_compatible --base-url https://vllm.internal
boardwalk inference delete my-openai --yes
```

`--source` is required: `bedrock`, `anthropic`, `google`, `openai`, `openai_compatible`, or `azure_openai` (plus `--base-url`, `--region`, `--api-version`, `--api-key` as the source needs). The name is what an `agent({ provider })` call routes to. Adding or deleting needs `--scopes admin`.

## Project linking (the `--org` flag becomes optional)

The first successful `deploy`/`run` writes a per-directory link at `.boardwalk/project.json` (`{ orgSlug, workflowId }`) and adds `.boardwalk/` to `.gitignore`. Once a directory is linked, `--org` is optional and subsequent commands target the same workflow — a Vercel-style, rename-safe project identity. Deploy each separate project from its own directory so the links don't clobber each other.

## Auth precedence

For commands that talk to Boardwalk (`deploy`, `run`, `cancel`, `runs`, `usage`, `workflows`, `webhook`, `secrets`, `inference`, …), credentials resolve in this order:

1. `--token <token>` (per-command Bearer token)
2. `BOARDWALK_API_KEY` environment variable
3. the stored `boardwalk login` session

The API host follows the same source: an explicit `BOARDWALK_API_URL`/`BOARDWALK_API_DOMAIN` wins, otherwise the stored session's own origin (so a dev or self-hosted login just works), then the default.

## Self-hosting / pointing at another deployment

The CLI is provider-agnostic and works against any Boardwalk deployment. Override the targets via environment variables:

- `BOARDWALK_API_DOMAIN` — the deployment's domain (the simplest single knob; resolves to `https://<domain>`)
- `BOARDWALK_API_URL` — set the API base explicitly (full URL; the escape hatch for local http / non-standard ports)
- `BOARDWALK_ISSUER_URL` — the OAuth issuer origin `login` authenticates against
- `BOARDWALK_OAUTH_CLIENT_ID` / `BOARDWALK_OAUTH_PORT` — OAuth client id (default `boardwalk-cli`) and the local callback port (default `53682`)
- `BOARDWALK_CONFIG_DIR` — where credentials + config are stored

Example (drive a self-hosted or custom deployment):

```bash
BOARDWALK_API_DOMAIN=boardwalk.your-company.com boardwalk login
```

## Quick reference

| Command | Purpose |
| --- | --- |
| `boardwalk init [dir] [--template <name>]` | Scaffold a new workflow project |
| `boardwalk dev <file\|dir>` | Run the workflow now, locally |
| `boardwalk check <file\|dir>` | Validate locally (no auth, no network) |
| `boardwalk build <file\|dir> [--out <path>]` | Bundle to one deployable `.mjs` |
| `boardwalk login [--scopes admin] [--token bwk_…]` | Authenticate (browser OAuth, or store an API key) |
| `boardwalk whoami` / `boardwalk status` / `boardwalk logout` | Inspect / verify / clear credentials |
| `boardwalk deploy <file\|dir> [--org <slug>] [--dry-run]` | Create or update a workflow |
| `boardwalk run <file\|dir> [--org <slug>] [--input <json>] [--no-wait]` | Deploy + trigger a real run |
| `boardwalk cancel <runId>` | Cancel a queued or in-flight run |
| `boardwalk runs [runId] [--logs] [--follow] [--json]` | List runs, show one, or stream its log |
| `boardwalk usage [--org <slug>] [--days <n>] [--json]` | Org spend and activity |
| `boardwalk workflows [list\|show\|disable\|enable\|delete] …` | Inspect and manage workflows |
| `boardwalk webhook <id\|slug> [--rotate]` | Show / rotate a workflow's inbound webhook URL |
| `boardwalk secrets [list\|set\|delete] …` | Manage the org's secrets (admin to write) |
| `boardwalk inference [list\|add\|delete] …` | Manage BYO inference providers (admin to write) |
