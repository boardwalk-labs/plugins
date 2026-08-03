---
name: "boardwalk-overview"
description: "Orientation for an agent that is new to Boardwalk: what the platform is and how it fits together, before driving the CLI or writing a workflow. Boardwalk runs agent workflows, which are TypeScript or Python programs (not YAML, a GUI, or a chatbot) that call models on a schedule, webhook, or on demand. Use when a user mentions Boardwalk and the agent lacks the mental model. Points to boardwalk-use-cli (scaffold, run, ship) and write-good-loops (iterating loops)."
---

# What Boardwalk is

Boardwalk runs **agent workflows**: TypeScript or Python programs that call models, run on
schedules or webhooks, and keep a permanent record of every run. A workflow is plain code, so you
author it in your own editor, with your own packages and tests, and ship it with one deploy.

It is **the control plane for agent workflows**: the place an org builds, versions, triggers, and
runs them, with durability, secrets, budgets, and full run history handled for you. It is code-first,
not a no-code builder and not a chatbot. You write the program; Boardwalk runs it unattended and
keeps the trace.

## A workflow is a typed function

**A Boardwalk workflow is a `run` function plus a small descriptor.** No YAML, no node editor, no
framework to subclass. Two files: the function does the work, and `workflow.jsonc` tells the
platform how to deploy it.

`src/index.ts`:

```ts
import { agent, secrets } from "@boardwalk-labs/workflow";

export default async function run(): Promise<string> {
  const token = await secrets.get("GITHUB_TOKEN");
  const issues = await fetch("https://api.github.com/issues", {
    headers: { Authorization: `Bearer ${token}` },
  }).then((r) => r.text());

  return await agent(`Summarize these open issues for a morning digest:\n${issues}`);
}
```

`workflow.jsonc`:

```jsonc
{
  "$schema": "https://boardwalk.sh/schemas/workflow.json",
  "slug": "morning-digest",
  "triggers": [{ "kind": "cron", "expr": "0 9 * * 1-5" }],     // when it runs
  "permissions": { "secrets": [{ "name": "GITHUB_TOKEN" }] },  // what it may read
}
```

- **The signature is the contract**, Lambda-style: `run(input, context) → Output`. `input` is the
  trigger's payload — annotate it (`run(input: Payment)`) and the deploy derives its schema, so the
  dashboard's run form and callers know the shape; leave it bare for raw JSON. The **return value is
  the run's output**, persisted and handed to whoever called the run. `context` (param 1) is
  read-only run metadata (ids, trigger, actor, attempt); declare it only when you need it.
- **Capabilities are imports** — `agent`, `secrets`, `sleep`, and the rest, like `import boto3` in a
  Lambda. Nothing that acts is passed into your function.
- **The descriptor is policy, read as data**: triggers, permissions, budget — what the control plane
  must know without running your code. It declares no model and no I/O shapes.

A workflow is a **package**: the directory holding those files. The code that ships is whatever the
entry imports, plus a `skills/` folder and `README.md` by convention. Python is a peer, not a port:
the same shape is a module-level `async def run(input: Lead) -> Score` (Pydantic models as the
contract) beside the same `workflow.jsonc`.

## How the pieces fit

- **`agent(prompt, opts?)` is the LLM step.** It runs an agent loop (a model plus its tools) and
  returns the final text, or `schema`-validated JSON. The engine's coding tools (read, write, edit,
  ls, grep, glob, bash, webfetch, and more) are on by default, so a plain `agent(prompt)` already
  works in the run's workspace. Scope them with `builtins`.
- **The workflow names no model; each `agent()` call chooses one.** `model` is optional and picked
  per call: omit it to use Boardwalk's managed inference, or name one to pin it. `provider` defaults
  to the managed lane, or names your own keys. A workflow that does no model work names no model.
- **Durable primitives do everything else, in plain code:** `secrets.get`, `sleep`, `phase()`,
  `artifacts.write`, `workflows.call` (invoke another workflow and wait; `workflows.run` fires one
  without waiting), and `humanInput` (pause for a person). `parallel([...])` runs independent work
  at once; `shell` runs a command in the workspace; `auth.idToken`/`auth.apiToken` mint short-lived
  credentials; `usage.get()` reads live budget state; `computer.openBrowser()` opens a real in-VM
  browser your program sets up and can hand to an agent (see `equip-agents`).
- **Each `agent()` can be equipped per call**, on top of the default built-in tools, with reusable
  `skills`, inline `tools`, `mcp` servers, and persistent `memory`. See the `equip-agents` skill.

## Where a workflow runs

Deploy it to Boardwalk and the platform runs it — on its schedule, webhook, or API, with durability
and model routing handled for you. Iterate with `boardwalk check` (validate the package locally),
`boardwalk deploy . --org <org> --run` (deploy + trigger a real run), and `boardwalk runs <id> --logs`.
There is no local run mode: unit tests call `run(input)` directly over `installTestHost` stubs, and
live execution is a real run, pointed at a dev environment when you're iterating.

A hosted run can also be pinned to your own machine with a self-hosted runner
(`"runs_on": { "kind": "self-hosted" }` in `workflow.jsonc`). See `boardwalk-use-cli`.

## Gotchas that trip up a newcomer

Get these right before writing code:

- **Write plain TypeScript — `Date.now()` and `Math.random()` just work.** There is no determinism
  gate. On the hosted fleet a `sleep` or `humanInput` snapshots the whole machine and resumes the
  exact heap, so a wait loses nothing. A crash (or a wait on a substrate without snapshots) restarts
  the program from the top, Lambda-style, so make side effects safe to re-run: use idempotent keys
  or upserts, and put must-not-repeat work behind `workflows.call`, which re-attaches to a finished
  child instead of running it twice. An already-answered `humanInput` gate is never re-asked.
- **Secrets never reach the model.** Your program reads them with `secrets.get` (the trusted layer),
  and the SDK redacts known secret values from all `agent()` context. Never `console.log` a secret,
  since run logs are kept.
- **A long `sleep` is not billed.** It releases the machine and resumes later, so idle time is free.
  Use it instead of a polling loop.
- **Set `budget.max_usd`.** A runaway agent loop spends real money, and the budget is the backstop.
  A breach pauses the run for approval — never a silent kill — and a program can watch its own spend
  with `usage.get()`.

## Watch the words

The top-level unit you build is a **workflow**. The word **agent** means the inner LLM loop it calls
via `agent()`, not you (the coding agent reading this) and not a Claude Code subagent.

## Where to go next

- To author a workflow well (typed I/O, match the model, keep it legible, guardrails, surviving
  restarts): use **`write-good-workflows`**.
- To build a workflow that iterates until a goal is reached (find/fix/verify, drain a queue, poll
  until healthy, a nightly maintainer): use **`write-good-loops`**.
- To give an `agent()` skills, tools, MCP servers, or memory, and structure a multi-file package: use
  **`equip-agents`**.
- To scaffold, run, validate, deploy, trigger, or inspect a workflow, and for every `boardwalk`
  command (secrets, environments, inference providers, self-hosted runners): use **`boardwalk-use-cli`**.

The full authoring contract (every primitive, the descriptor fields, the run-event format) is in the
`@boardwalk-labs/workflow` package's `SPEC.md`.
