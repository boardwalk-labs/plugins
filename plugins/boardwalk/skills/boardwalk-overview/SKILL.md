---
name: "boardwalk-overview"
description: "Explains what Boardwalk is and how it fits together — the orientation an agent needs the first time it encounters the platform, before driving the CLI or writing a workflow. Use when a user mentions Boardwalk and the agent lacks the mental model: what Boardwalk is (a platform for running agent workflows unattended — in the cloud, on a self-hosted server, or locally), what a workflow is (a TypeScript/JavaScript program, NOT YAML, a GUI, or a chatbot), how the pieces fit (the agent() LLM leaf, durable primitives, secrets), where runs execute, and the constraints that trip up newcomers (determinism, secrets never reach the model, per-call models). Routes to the sibling skills: boardwalk-use-cli to scaffold/run/ship, and write-good-loops to build iterating loops. Best loaded before boardwalk-use-cli when Boardwalk is new."
---

# What Boardwalk is

Read this before you touch the CLI or write any code, if Boardwalk is new to you. It exists because
a third-party agent lands on the other Boardwalk skills — which are command and pattern reference —
without knowing what the thing *is*. This skill is the mental model; the other two are how you drive it.

**Boardwalk is a platform for running agent workflows** — programs that call LLMs, and do real work
around them, **unattended**: on a schedule, on a webhook, or on demand. You (an AI coding agent) author
the workflow; Boardwalk runs it durably in the cloud (or on the user's own server, or locally) and
handles the parts a raw script can't: durability across restarts, secrets, human-in-the-loop pauses,
long sleeps that cost nothing, cross-workflow composition, and billing. Think "the control plane where
an org builds and runs its agent workflows," not a chatbot and not a drag-and-drop automation GUI.

## A workflow is a program, not a config

This is the single most important thing to get right, and the one newcomers get wrong: **a Boardwalk
workflow is a TypeScript/JavaScript file** (or a package directory containing one). There is **no YAML,
no DSL, and no visual builder** — the program file *is* the source of truth.

```ts
import { agent, output, secrets, type WorkflowMeta } from "@boardwalk-labs/workflow";

export const meta = {
  slug: "morning-digest",
  title: "Morning Digest",
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

Two parts:

- **`export const meta`** is a **pure literal** — a static object (slug, triggers, permissions, budget,
  …). Boardwalk reads it *without running your code* to derive the **manifest**, the control-plane
  contract for how the workflow is triggered and run. It declares **no model** (see below).
- **The module body is the program.** Importing the file *is* running it: top-level `await` is normal,
  ordinary TypeScript throughout (any import, any control flow, any npm dependency), and `output(value)`
  declares the run's result. Trigger input arrives as the `input` global.

## How the pieces fit

- **`agent(prompt, opts?)` is the LLM leaf** — it runs an agent loop (the model plus its tools) and
  returns the final text, or `schema`-validated JSON. The engine's coding tools (read/write/edit/ls/
  grep/glob/bash/webfetch/…) are on by default, so a plain `agent(prompt)` can already work in the run's
  workspace; scope them with `builtins`.
- **The workflow names no model; each `agent()` call does.** `model` is optional and chosen **per call** —
  omit it and the managed lane routes automatically, or name one to pin it; `provider` defaults to
  Boardwalk's managed inference, or names your own keys. A workflow that does no LLM work names no model
  at all.
- **Durable primitives do everything else, in deterministic code:** `secrets.get`, `sleep`,
  `phase()`, `output`, `artifacts.write`, `workflows.call` (invoke another workflow, durably),
  `humanInput` (pause for a person), `step.run` (run a side effect once across a resume), and the durable
  `now()` / `random()` / `uuid()`. `parallel([...])` runs independent work concurrently.

## Where a workflow runs

The **same file runs on three engines**, with the same manifest and event stream:

1. **Locally** — `boardwalk dev ./index.ts`. No account needed; a one-time `boardwalk login` only if a
   step uses Boardwalk's managed models.
2. **Self-hosted** — the open-source engine on the user's own server.
3. **The hosted Boardwalk platform** — `boardwalk deploy`, then triggered on its schedule/webhook/API,
   with durability and automatic model routing managed for you.

Even on the hosted platform, a specific run can be pinned to the user's own hardware with a **self-hosted
runner** (`runs_on: { kind: "self-hosted" }` in `meta`) — see `boardwalk-use-cli`.

## Constraints that trip up a newcomer

Correct these before you write code — an agent that doesn't know them ships broken workflows:

- **The program must be deterministic across a restart.** A run restarts from the top on a crash and
  replays from the top after a `sleep`/`humanInput`, so a value that changes on the second pass corrupts
  the run. **Bare `Date.now()`, `new Date()`, `Math.random()`, `crypto.randomUUID()` are blocked** at
  `check`/`deploy`/`run` — use the durable `await now()` / `random()` / `uuid()` instead. Wrap other
  nondeterministic I/O in `step.run(name, fn)` when it precedes a `sleep`.
- **Secrets never reach the model.** Your program reads them with `secrets.get` (the trusted layer); the
  SDK redacts known secret values from all `agent()` context. So prompt injection can't exfiltrate a
  secret — but also, never `console.log` one (run logs are persisted).
- **A long `sleep` is free.** It releases the machine and resumes later; idle time isn't billed. Use it
  instead of a polling busy-loop.
- **Set `budget.max_usd`.** A runaway agent loop spends real money; the budget is the backstop.

## What Boardwalk is NOT

- **Not YAML / a GUI / a DSL** — a workflow is a code file.
- **Not Zapier or a no-code automation tool** — it's code-first and agent-native.
- **Not a chatbot or an IDE assistant** — it runs unattended workflows, not an interactive session.
- **Not a model host** — it *routes* inference (managed, or your own provider keys); it runs no GPUs.
- **Watch the words:** the top-level unit you build is a **workflow**; **"agent"** means the inner LLM
  loop it calls via `agent()`, *not* you (the coding agent reading this) and *not* a Claude Code subagent.

## Where to go next

- **To scaffold, run locally, validate, deploy, trigger, or inspect a workflow — and every `boardwalk`
  command, secrets, environments, inference providers, and self-hosted runners** → use the
  **`boardwalk-use-cli`** skill.
- **To build a workflow that iterates until a goal is reached** (find/fix/verify, drain a queue,
  poll-until-healthy, a nightly maintainer) → use the **`write-good-loops`** skill.

The full authoring contract — every primitive, the manifest fields, the run-event wire format — lives in
the `@boardwalk-labs/workflow` package's `SPEC.md`.
