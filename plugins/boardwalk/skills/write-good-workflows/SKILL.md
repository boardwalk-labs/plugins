---
name: "write-good-workflows"
description: "Use when authoring or improving a Boardwalk workflow program to make it correct, cheap, and legible, not just working. Covers the shape of a workflow (the run function plus the workflow.jsonc descriptor, growing into a package), typed input and output (native types drive the run form and the output contract), the SDK primitives (agent, phase, sleep, workflows.call, parallel), making a run legible with phases and console.log, writing efficient workflows (match the model per agent() call, scope tools and output, prompt caching, parallelism, don't pay to wait, budget and reuse guardrails), and surviving crashes and restarts. For iterating loops see write-good-loops; for giving an agent() skills/tools/MCP/memory see equip-agents; for running and deploying see boardwalk-use-cli."
allowed-tools: Read, Write, Edit, Bash
---

# Write a good workflow

A workflow that runs is not yet a good one. A good workflow is **cheap** (the model is almost always
the biggest line on the bill), **legible** (a run is a permanent record you debug from), and **safe
across a restart**. This skill is the craft of authoring the program. For what Boardwalk is, see
`boardwalk-overview`; for running and shipping it, see `boardwalk-use-cli`.

## The shape

A workflow is a package of two files: a `run` function (the work) and `workflow.jsonc` (the
deployment descriptor). The platform calls `run(input, context)` with the trigger's payload;
whatever you return is the run's output. Deterministic code (fetch, parse, hold secrets)
interleaves freely with model calls (`agent()`).

`src/index.ts`:

```ts
import { agent, phase, secrets } from "@boardwalk-labs/workflow";

export default async function run(): Promise<string> {
  phase("Fetch issues");
  const token = await secrets.get("GITHUB_TOKEN");
  const issues = await fetch("https://api.github.com/issues", {
    headers: { Authorization: `Bearer ${token}` },
  }).then((r) => r.json());

  phase("Summarize");
  return await agent(`Write a morning digest of these issues:\n${JSON.stringify(issues)}`);
}
```

`workflow.jsonc`:

```jsonc
{
  "$schema": "https://boardwalk.sh/schemas/workflow.json",
  "slug": "morning-digest",
  "triggers": [{ "kind": "cron", "expr": "0 9 * * 1-5" }],
  "permissions": { "secrets": [{ "name": "GITHUB_TOKEN" }] },
}
```

Python is the same shape: a module-level `async def run(input: Lead) -> Score` in `main.py`, with
Pydantic models as the contract. Two files are the starting point, not the ceiling. When a workflow
grows, the package grows with it: the entry imports your helper modules normally, and ships bundled
assets next to it (a `skills/` folder, prompt templates, a rubric). `boardwalk deploy .` bundles
whatever the entry imports. See `equip-agents`.

**Write a `README.md` at the package root.** It renders as the workflow's landing page in the
dashboard, beside the config from `workflow.jsonc`. Write it for whoever gets paged at 3am, not for
whoever wrote the code: what this workflow is for, what it touches, what it costs, what to do when
it fails. The descriptor already tells the reader the schedule and the budget — don't restate them.
It always ships, by convention.

## Type the input and output

Your native types are the I/O contract — no schema literal, no wrapper:

```ts
interface Alert  { service: string; message: string }
interface Triage { action: "page" | "ticket" | "ignore"; reason: string }

export default async function run(input: Alert): Promise<Triage> { /* ... */ }
```

Annotate the input and the deploy derives its schema from the real type checker, so the dashboard
renders an **accurate input form** and callers know the shape; a field it can't render degrades to a
raw-JSON box with a warning — never a wrong widget, never a blocked deploy. A bare `run(input)` is
the zero-ceremony floor: raw JSON, Lambda-style. At runtime the payload is best-effort converted
(an ISO string arrives as a real `Date`); there is no pre-run gate, so validate in code if the
input can be hostile. The **return** is validated against your declared type automatically — a
mismatch fails the run — and a `void` return persists `null`, which is first-class for a workflow
whose product is its side effects.

`context` (param 1, declare it only when needed) is read-only metadata: `runId`, `trigger`, `actor`,
`attempt`, `environment`, `workspaceDir`, `signal`. Data only — everything that acts is an import.

## The descriptor

`workflow.jsonc` is policy the platform enforces around your code, read as data (comments and
trailing commas welcome; never executed). The fields worth knowing: `slug` (required, the stable
identity), `triggers` (required: `cron`, `webhook`, `manual`, `workflow_run`, or a provider event —
`github`, `linear`, `jira`, `notion`. A `cron` trigger may add a static `input` object passed to
every scheduled run, and a `webhook` trigger NAMES one of the org's webhooks,
`{ "kind": "webhook", "name": "stripe-prod" }`, created once with
`boardwalk webhooks create`), `permissions` (the `secrets`
allowlist a run may `secrets.get`; `id_token` for `auth.idToken`), `budget` (`max_usd`,
`max_tokens`, `max_compute_seconds` — all metered; a breach pauses the run for approval, never a
hard kill), `concurrency` (`unlimited` default, `serial`, or per-entity
`{ "mode": "serial", "key": "refund-${input.customerId}" }`), `workspace` (directories to persist
between runs), and `runs_on` (the machine, default `boardwalk/linux`). The workflow declares **no**
model and no I/O schemas — those come from your types.

**Provider triggers.** `github`, `linear`, `jira` and `notion` fire on a connected provider's events
with no URL and no secret to manage: `{ "kind": "github", "event": "pr.merged", "repos": ["acme/app"] }`
(`repos` optional, GitHub only), `{ "kind": "linear", "event": "issue.status_changed" }`. The
vocabularies are semantic outcomes, never raw vendor actions — GitHub: `pr.opened` / `pr.merged` /
`issue.opened` / `issue.commented` / `ci.completed`; Linear and Jira: `issue.created` /
`issue.status_changed` / `issue.commented`; Notion: `page.created` / `page.updated` /
`comment.created`. Anything else means a plain `webhook` trigger instead. Three things to tell the
user: the workflow **deploys before** the org has connected the provider (it shows needs-connection,
never a deploy failure, so a package stays portable), the run input is a curated typed projection of
the vendor payload (always `input.event` and `deliveryId`, plus `pr` / `issue` / `comment` /
`entity`) rather than the raw object, and a Notion payload carries **ids only**, so that program has
to fetch the content itself. The trigger plane vends no provider tokens: anything the program does
back to the provider uses the org's own credential via `secrets.get` or an MCP connection. Connect
is a browser flow in the dashboard, so there is no CLI command for it.

## The workspace

Your program runs **in** its workspace: it's the working directory and `HOME`, so a relative path is
the workspace and needs no ceremony.

```ts
writeFileSync("notes.md", "hi");   // the workspace. Same on hosted and self-hosted.
```

It's **scratch** by default — discarded when the run ends. Three ways to make something outlive the
run, and they combine:

| | Keeps |
|---|---|
| `"workspace": { "persist": ["cache", "state"] }` | exactly those directories |
| `agent(prompt, { memory: "notes" })` | that directory, declared nowhere |
| `"workspace": { "persist": true }` | the whole workspace |

Prefer naming directories. `true` keeps everything, so a clone or a `node_modules` ships to storage
on every run, toward the 512 MB snapshot cap and your storage bill — a workspace is mostly scratch
with a small part that compounds, so say which part. Each **environment** keeps its own workspace
(staging never sees production's), and two concurrent runs sharing one are last-writer-wins, so pin
`"concurrency": { "mode": "serial" }` if that would tear. When compounded state goes bad, clear it
with `boardwalk workspace show <workflow>` / `boardwalk workspace reset <workflow>` — the workflow,
its triggers, and its history are untouched.

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
- **Guardrails and reuse.** Set `budget.max_usd` (a guardrail, not the bill; a breach pauses for
  approval). Keep the default machine unless a step is CPU or memory bound. Persist expensive setup
  (a clone, an index) with `"workspace": { "persist": ["repo"] }`. Put must-not-repeat work behind
  `workflows.call`, which re-attaches to a finished child on restart rather than running it again.

## Survive a crash

A `sleep` or `humanInput` snapshots the whole machine and resumes the exact heap, so a wait loses
nothing and needs no special handling — write plain TypeScript, `Date.now()` and `Math.random()`
included. What you plan for is a **crash**: if the process dies, Boardwalk restarts the program
**from the top**, like a Lambda or a CI job, re-running every side effect along the way. So make
side effects safe to repeat — idempotent keys, upserts, "create if absent" — and put work you must
not repeat behind `workflows.call()`, which re-attaches to a finished child instead of running it
twice. An already-answered `humanInput` gate is never re-asked on a restart; its answer is durable.
`context.attempt` tells you which attempt you're on (it stays 1 until a crash-restart).

Checkpoint expensive side effects as you go. A run that clones a repo, makes changes, and pushes
should push incrementally (or persist the clone), so a crash near the end doesn't throw away the work
already done — the run restarts from the top, but the pushed commits are still there to resume from.
A crash is the one thing a persisted workspace does NOT cover: it saves at the end of a run (success
or failure) and before a `sleep`, so a hard crash resumes from the last save, not from the instant it
died.

And because `run` is a plain function, unit-test it as one: `installTestHost({ agent, secrets, ... })`
stubs the capabilities, making `run(input)` an ordinary call with no platform in sight.

## Where to go next

- Iterating until a goal is reached (find/fix/verify, drain a queue, a nightly maintainer): use
  **`write-good-loops`**.
- Giving an `agent()` skills, tools, MCP servers, or memory, and structuring a multi-file package:
  use **`equip-agents`**.
- Scaffolding, validating, running, and deploying: use **`boardwalk-use-cli`**.
