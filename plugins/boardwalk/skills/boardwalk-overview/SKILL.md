---
name: "boardwalk-overview"
description: "Orientation for an agent that is new to Boardwalk: what the platform is and how it fits together, before driving the CLI or writing a workflow. Boardwalk runs agent workflows, which are TypeScript programs (not YAML, a GUI, or a chatbot) that call models on a schedule, webhook, or on demand. Use when a user mentions Boardwalk and the agent lacks the mental model. Points to boardwalk-use-cli (scaffold, run, ship) and write-good-loops (iterating loops)."
---

# What Boardwalk is

Boardwalk runs **agent workflows**: TypeScript programs that call models, run on schedules or
webhooks, and keep a permanent record of every run. A workflow is plain code, so you author it in
your own editor, with your own packages and tests, and run the exact same file on your laptop, on
your own hardware, or hosted on Boardwalk.

It is **the control plane for agent workflows**: the place an org builds, versions, triggers, and
runs them, with durability, secrets, budgets, and full run history handled for you. It is code-first,
not a no-code builder and not a chatbot. You write the program; Boardwalk runs it unattended and
keeps the trace.

## A workflow is a program, not a config

**A Boardwalk workflow is one TypeScript/JavaScript file** (or a package directory of them). No YAML,
no node editor, no framework to subclass. The file has two parts: a `meta` export that declares the
contract, and a script body that does the work.

```ts
import { agent, output, secrets, type WorkflowMeta } from "@boardwalk-labs/workflow";

export const meta = {
  slug: "morning-digest",
  triggers: [{ kind: "cron", expr: "0 9 * * 1-5" }],       // when it runs
  permissions: { secrets: [{ name: "GITHUB_TOKEN" }] },    // what it may read
} satisfies WorkflowMeta;

const token = await secrets.get("GITHUB_TOKEN");
const issues = await fetch("https://api.github.com/issues", {
  headers: { Authorization: `Bearer ${token}` },
}).then((r) => r.text());

const summary = await agent(`Summarize these open issues for a morning digest:\n${issues}`);
output(summary);
```

- **`meta` is a pure literal.** Boardwalk reads it without running your code to derive the
  **manifest**, the control-plane contract: schedule, declared secrets, budget. It declares no model.
- **The script body is the program.** It runs top to bottom each time. Top-level `await` is normal,
  any import and any npm dependency work, and `output(value)` declares the result. Trigger input
  arrives as the `input` global.

A workflow can also be a **package**: a directory the CLI bundles as one unit, with an entry `index.ts`,
helper modules it imports, and a `skills/` folder or other assets. Point the CLI at the directory
instead of the file. This is what lets an `agent()` load reusable skills and resources. See the
`equip-agents` skill.

## How the pieces fit

- **`agent(prompt, opts?)` is the LLM step.** It runs an agent loop (a model plus its tools) and
  returns the final text, or `schema`-validated JSON. The engine's coding tools (read, write, edit,
  ls, grep, glob, bash, webfetch, and more) are on by default, so a plain `agent(prompt)` already
  works in the run's workspace. Scope them with `builtins`.
- **The workflow names no model; each `agent()` call chooses one.** `model` is optional and picked
  per call: omit it to use Boardwalk's managed inference, or name one to pin it. `provider` defaults
  to the managed lane, or names your own keys. A workflow that does no model work names no model.
- **Durable primitives do everything else, in plain code:** `secrets.get`, `sleep`, `phase()`,
  `output`, `artifacts.write`, `workflows.call` (invoke another workflow), `humanInput` (pause for a
  person), `step.run` (run a side effect once across a resume), and the durable `now()`, `random()`,
  `uuid()`. `parallel([...])` runs independent work at once.
- **Each `agent()` can be equipped per call**, on top of the default built-in tools, with reusable
  `skills`, inline `tools`, `mcp` servers, and persistent `memory`. See the `equip-agents` skill.

## Where a workflow runs

The same file runs under three engines, with the same manifest and event stream:

1. **Locally**, with `boardwalk dev ./index.ts`. No account needed. A one-time `boardwalk login` is
   required only if a step uses Boardwalk's managed models.
2. **Self-hosted**, the open-source single-node engine on your own hardware.
3. **Hosted on Boardwalk**, with `boardwalk deploy`, then triggered on its schedule, webhook, or API,
   with durability and model routing handled for you.

A hosted run can also be pinned to your own machine with a self-hosted runner
(`runs_on: { kind: "self-hosted" }` in `meta`). See `boardwalk-use-cli`.

## Gotchas that trip up a newcomer

Get these right before writing code:

- **The program must be deterministic across a restart.** A run restarts from the top on a crash and
  replays from the top after a `sleep` or `humanInput`, so a value that changes on the second pass
  corrupts the run. Bare `Date.now()`, `new Date()`, `Math.random()`, and `crypto.randomUUID()` are
  blocked at `check`, `deploy`, and `run`. Use the durable `now()`, `random()`, and `uuid()` instead,
  and wrap other nondeterministic I/O in `step.run(name, fn)` when it precedes a `sleep`.
- **Secrets never reach the model.** Your program reads them with `secrets.get` (the trusted layer),
  and the SDK redacts known secret values from all `agent()` context. Never `console.log` a secret,
  since run logs are kept.
- **A long `sleep` is not billed.** It releases the machine and resumes later, so idle time is free.
  Use it instead of a polling loop.
- **Set `budget.max_usd`.** A runaway agent loop spends real money, and the budget is the backstop.

## Watch the words

The top-level unit you build is a **workflow**. The word **agent** means the inner LLM loop it calls
via `agent()`, not you (the coding agent reading this) and not a Claude Code subagent.

## Where to go next

- To author a workflow well (match the model, keep it legible, guardrails, determinism): use
  **`write-good-workflows`**.
- To build a workflow that iterates until a goal is reached (find/fix/verify, drain a queue, poll
  until healthy, a nightly maintainer): use **`write-good-loops`**.
- To give an `agent()` skills, tools, MCP servers, or memory, and structure a multi-file package: use
  **`equip-agents`**.
- To scaffold, run, validate, deploy, trigger, or inspect a workflow, and for every `boardwalk`
  command (secrets, environments, inference providers, self-hosted runners): use **`boardwalk-use-cli`**.

The full authoring contract (every primitive, the manifest fields, the run-event format) is in the
`@boardwalk-labs/workflow` package's `SPEC.md`.
