---
name: "use-boardwalk-cli"
description: "Use when a user wants to install, configure, authenticate against, or drive the first-party Boardwalk CLI — the `boardwalk` command for authoring, validating, running, and shipping agent workflows. A Boardwalk workflow is a TypeScript/JavaScript program file whose pure-literal `meta` compiles to the manifest; it calls `agent(prompt, { model })` for LLM work and durable primitives (secrets, sleep, phases, output, artifacts) for everything else. Covers installation, scaffolding (init), running locally with no account (dev), local validation (check), browser OAuth login, deploying, triggering real runs, cancelling, project linking, auth precedence, the run-event channels, and self-hosting knobs."
allowed-tools: Bash
---

# Use the Boardwalk CLI

Use this skill whenever the user needs to install, configure, or drive the first-party `boardwalk` CLI — to scaffold a workflow, run it locally, validate it, sign in, deploy it, trigger a real run, or cancel one. This is the canonical reference for the whole CLI surface.

## What a Boardwalk workflow is

A workflow is a **TypeScript/JavaScript program file** (e.g. `index.ts`) — or a package directory containing one. The program exports a pure-literal `meta` object that the platform compiles to the **manifest** (the control-plane contract: slug, optional title, triggers, runtime). The program body does the work:

- `agent(prompt, { model })` runs an LLM loop. **`model` is required and named per call** (e.g. `anthropic/claude-sonnet-4.5`); the workflow itself declares no model. `provider` is optional (defaults to the managed `boardwalk` lane).
- Durable primitives — `secrets.get`, `sleep`, phases, `output`, artifacts, `workflows.call` — run in deterministic code. Secrets never reach the LLM context.

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
boardwalk init my-workflow                 # scaffold into ./my-workflow
boardwalk init                             # scaffold into the current directory
boardwalk init my-workflow --template hello
```

Creates the program file, `package.json`, `.env.example`, and `.gitignore`. The default `hello` template runs green under `dev` immediately. `--template <name>` can also name any template from the Boardwalk examples registry.

### `boardwalk dev <file|dir>` — run it now, locally (no account)

```bash
boardwalk dev ./index.ts
boardwalk dev ./index.ts --input '{"who":"world"}'   # trigger payload exposed to the program as `input`
boardwalk dev ./index.ts --env .env.local            # resolve secrets from this env file (default: .env)
boardwalk dev ./index.ts --verbose                   # stream EVERY channel (agent turns, tool calls, logs)
boardwalk dev ./index.ts --stream output | jq        # just the result — pipe-friendly
boardwalk dev ./index.ts --stream phase,log
```

`dev` derives and validates the manifest (precise errors before anything runs), bundles the program, executes it in-process, and streams the run-event log. Secrets resolve from `.env` (or `--env <path>`) and their **values never print**.

**Exit code is the run's verdict:** `0` completed, `1` failed, `130` cancelled (Ctrl-C). That makes `dev` usable as a CI gate.

### `boardwalk check <file|dir>` — validate without running

```bash
boardwalk check ./index.ts
```

Everything `dev` validates, without executing: full manifest-schema validation (the same schema every engine enforces) plus an esbuild compile proving every import resolves. No auth, no network — safe in CI on every commit.

## Run-event channels

Every engine — local `dev` and the hosted platform alike — emits the same typed event stream. Each event belongs to one **channel**: `lifecycle`, `phase`, `output`, `log`, `agent`. The flags mean the same thing everywhere:

- default → `lifecycle` + `phase` + `output` (quiet, readable)
- `--verbose` → all five channels, including `agent` turns and tool calls
- `--stream <channels>` → a comma-separated subset (e.g. `--stream output` for just the result)

## Authenticate

```bash
boardwalk login              # browser OAuth (PKCE) → stores a session
boardwalk login --token bwk_xxx   # store an API key instead of the browser flow
boardwalk whoami             # show the current session
boardwalk logout             # remove local credentials
```

`login` opens a browser for an OAuth PKCE flow and stores a scoped session locally. For headless/CI use, pass an API key (`bwk_…`) with `--token`, or supply one per-command (below).

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

## Project linking (the `--org` flag becomes optional)

The first successful `deploy`/`run` writes a per-directory link at `.boardwalk/project.json` (`{ orgSlug, workflowId }`) and adds `.boardwalk/` to `.gitignore`. Once a directory is linked, `--org` is optional and subsequent `deploy`/`run` target the same workflow — a Vercel-style, rename-safe project identity. Deploy each separate project from its own directory so the links don't clobber each other.

## Auth precedence

For `deploy`, `run`, and `cancel`, credentials resolve in this order:

1. `--token <token>` (per-command Bearer token)
2. `BOARDWALK_API_KEY` environment variable
3. the stored `boardwalk login` session

## Self-hosting / pointing at another deployment

The CLI is provider-agnostic and works against any Boardwalk deployment. Override the targets via environment variables:

- `BOARDWALK_API_DOMAIN` — the deployment's domain (the simplest single knob)
- `BOARDWALK_API_URL` / `BOARDWALK_ISSUER_URL` — set the API base and the OAuth issuer explicitly
- `BOARDWALK_OAUTH_CLIENT_ID` / `BOARDWALK_OAUTH_PORT` — OAuth client id and the local callback port
- `BOARDWALK_CONFIG_DIR` — where credentials + config are stored

Example (drive a self-hosted or custom deployment):

```bash
BOARDWALK_API_DOMAIN=https://boardwalk.your-company.com boardwalk login
```

## Quick reference

| Command | Purpose |
| --- | --- |
| `boardwalk init [dir] [--template <name>]` | Scaffold a new workflow project |
| `boardwalk dev <file\|dir>` | Run the workflow now, locally — no account |
| `boardwalk check <file\|dir>` | Validate locally (no auth, no network) |
| `boardwalk login [--token bwk_…]` | Authenticate (browser OAuth, or store an API key) |
| `boardwalk whoami` / `boardwalk logout` | Show / clear the current session |
| `boardwalk deploy <file\|dir> [--org <slug>] [--dry-run]` | Create or update a workflow |
| `boardwalk run <file\|dir> [--org <slug>] [--input <json>] [--no-wait]` | Deploy + trigger a real run |
| `boardwalk cancel <runId>` | Cancel a queued or in-flight run |
