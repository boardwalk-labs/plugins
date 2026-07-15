---
name: "write-good-workflows"
description: "Use when authoring or improving a Boardwalk workflow program to make it correct, cheap, and legible, not just working. Covers the shape of a workflow (the meta contract plus the script body, growing into a package), the SDK primitives (agent, phase, output, sleep, workflows.call, parallel), making a run legible with phases and console.log, writing efficient workflows (match the model per agent() call, scope tools and output, prompt caching, parallelism, don't pay to wait, budget and reuse guardrails), and surviving crashes and restarts. For iterating loops see write-good-loops; for giving an agent() skills/tools/MCP/memory see equip-agents; for running and deploying see boardwalk-use-cli."
allowed-tools: Read, Write, Edit, Bash
---

# Write a good workflow

A workflow that runs is not yet a good one. A good workflow is **cheap** (the model is almost always
the biggest line on the bill), **legible** (a run is a permanent record you debug from), and **safe
across a restart**. This skill is the craft of authoring the program. For what Boardwalk is, see
`boardwalk-overview`; for running and shipping it, see `boardwalk-use-cli`.

## The shape

A workflow is one TypeScript file with two parts: a `meta` export (the contract) and a script body
(the work). It runs top to bottom each time; top-level `await` is normal, and deterministic code
(fetch, parse, hold secrets) interleaves freely with model calls (`agent()`).

```ts
import { phase, agent, output, secrets } from "@boardwalk-labs/workflow";

export const meta = {
  slug: "morning-digest",
  triggers: [{ kind: "cron", expr: "0 9 * * 1-5" }],
  permissions: { secrets: [{ name: "GITHUB_TOKEN" }] },
};

phase("Fetch issues");
const token = await secrets.get("GITHUB_TOKEN");
const issues = await fetch("https://api.github.com/issues", {
  headers: { Authorization: `Bearer ${token}` },
}).then((r) => r.json());

phase("Summarize");
output(await agent(`Write a morning digest of these issues:\n${JSON.stringify(issues)}`));
```

One file is the starting point, not the ceiling. When a workflow grows, make it a **package**: an
entry file that imports your helper modules and ships bundled assets next to it (a `skills/` folder,
prompt templates, a rubric). `boardwalk deploy .` on the directory bundles the whole package. See
`equip-agents`.

## The meta contract

`meta` must be a **pure literal** so Boardwalk derives the manifest without running your code. The
fields worth knowing: `slug` (required, the stable identity), `triggers` (required: `cron`,
`webhook`, or `manual` — a `cron` trigger may add a static `input` object passed to every
scheduled run, e.g. `{ kind: "cron", expr: "0 9 * * *", input: { mode: "full" } }`),
`permissions.secrets` (the allowlist a run may `secrets.get`), `budget`
(`max_usd`, `max_tokens`, `max_duration_seconds` for active compute, `deadline_seconds` for
wall-clock including idle), `workspace` (directories to persist between runs), and `runs_on` (the
machine, default `boardwalk/linux`). The workflow declares **no** model.

## Make the run legible

A run is a permanent, replayable record, and your program decides how much of its story that record
tells. Use both hooks from the start; when you come back to a failed run, the log is all you have.

- **`phase("name")`** marks a stage on the `phase` channel, so the live tail and
  `boardwalk runs <id> --logs` read as named steps ("Fetch issues", "Triage", "File tickets") instead
  of one undifferentiated stream. Set one per logical stage.
- **`console.log`** lands on the `log` channel. Record the specifics: how many items you fetched,
  which branch you took and why, the id of something you created, the decision a step reached. A run
  whose log answers "what did this actually do?" is one you can debug without re-running it.

```ts
phase("Triage");
const urgent = issues.filter(isUrgent);
console.log(`${urgent.length} of ${issues.length} need attention`);
```

The default run view is quiet (`lifecycle` + `phase` + `output`), so well-named phases alone make a
run readable; `--verbose` or `--stream log` adds detail. **Never `console.log` a secret**: the log is
persisted, and secret values are redacted from the model's context, not from your own stdout.

## Write it efficiently

The model is almost always the biggest line on the bill, so spend tokens deliberately. This is not
premature optimization; it is the difference between a workflow you run once and one you run on a
schedule for months.

- **Match the model to the job.** The model is chosen per `agent()` call: a small, fast model for
  routine steps (classify, route, short summaries), a stronger one for genuinely hard steps. Unsure?
  Omit `model` (or pass `model: "auto"`) and the managed lane routes each call to a fitting model,
  with no routing fee. Raise `reasoning` only on steps that need careful multi-step thinking.
  (`boardwalk models` lists the catalog.)
- **Scope tools and output.** An agent carries the full built-in tool belt by default. Narrow it with
  `builtins: "read-only"` for analysis or `builtins: "none"` for a pure classifier or judge: every
  tool definition is tokens the model re-reads each turn. Ask for a `schema` instead of prose you
  re-prompt to reformat. Do parsing, filtering, deduping, and routing in plain code, and save the
  model for the judgement only a model can make.

```ts
const { verdict } = await agent<Verdict>(`Is this change correct? ...`, {
  builtins: "none",
  schema: VERDICT,
});
```

- **Prompt caching is automatic** on the managed lane, and it pays off **inside a single multi-turn
  `agent()` call**: keep the front of the prompt stable (instructions, shared context) and put the
  part that varies (the item, the file under review) last. Anything baked into the instructions that
  changes every run (a timestamp, a random id) makes every turn look new, so nothing is reused. A
  one-shot call has no later turn to read a cache, so just make it cheap with a small model.
- **Parallelize** independent work with `parallel([...])`: the same tokens, finished in the time of
  the slowest task instead of the sum. Multi-agent patterns (a panel of verifiers, a fan-out, a
  tournament) buy quality but spend N times the tokens; reach for them only when a task is genuinely
  large, parallel, or adversarial.
- **Don't pay to wait.** A long `sleep` suspends the run and releases its machine, so idle time is
  free. Use it instead of a polling loop.
- **Guardrails and reuse.** Set `budget.max_usd` (a guardrail, not the bill). Keep the default
  machine unless a step is CPU or memory bound. Persist expensive setup (a clone, an index) with
  `workspace: { persist: ["repo"] }`. Put must-not-repeat work behind `workflows.call`, which
  re-attaches to a finished child on restart rather than running it again.

## Survive a crash

A `sleep` or `humanInput` snapshots the whole machine and resumes the exact heap, so a wait loses
nothing and needs no special handling — write plain TypeScript, `Date.now()` and `Math.random()`
included. What you plan for is a **crash**: if the process dies, Boardwalk restarts the program
**from the top**, like a Lambda or a CI job, re-running every side effect along the way. So make
side effects safe to repeat — idempotent keys, upserts, "create if absent" — and put work you must
not repeat behind `workflows.call()`, which re-attaches to a finished child instead of running it
twice. An already-answered `humanInput` gate is never re-asked on a restart; its answer is durable.

Checkpoint expensive side effects as you go. A run that clones a repo, makes changes, and pushes
should push incrementally (or persist `/workspace`), so a crash near the end doesn't throw away the
work already done — the run restarts from the top, but the pushed commits are still there to resume
from.

## Where to go next

- Iterating until a goal is reached (find/fix/verify, drain a queue, a nightly maintainer): use
  **`write-good-loops`**.
- Giving an `agent()` skills, tools, MCP servers, or memory, and structuring a multi-file package:
  use **`equip-agents`**.
- Scaffolding, validating, running, and deploying: use **`boardwalk-use-cli`**.
