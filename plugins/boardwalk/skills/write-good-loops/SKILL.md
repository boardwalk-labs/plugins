---
name: "write-good-loops"
description: "Use when a user wants to build an agent LOOP in Boardwalk — a workflow that iterates until a goal is reached rather than running once. Covers find/fix/verify loops, poll-until-healthy, drain-a-queue, and run-on-a-schedule maintainers. Teaches the core loop shape (plain control flow over agent()), the layered exits every loop needs (a verifier, a hard iteration cap, a budget, no-progress detection), choosing the topology (one long run with while+sleep vs a recurring cron trigger of many short runs vs workflows.schedule for runtime-computed cadences), splitting the maker from the checker for verification, carrying durable state across runs, and never paying for idle wait. Pairs with use-boardwalk-cli for scaffolding, validating (boardwalk check), and running the loop."
allowed-tools: Read, Write, Edit, Bash
---

# Write good loops

Use this skill whenever the user wants an agent to **keep going until a job is done** instead of
running a single prompt: find every bug in a diff, drain a queue, watch a service until it's
healthy, or maintain a repo every night. A loop is how you hand the whole find → act → check → repeat
cycle to the workflow, so the user defines the goal once instead of re-prompting each step.

A Boardwalk workflow is a TypeScript/JavaScript program file (see `use-boardwalk-cli` for what a
workflow is and how to scaffold/validate/run one). A loop is **not** a platform feature you
configure — it's ordinary control flow in that program. So the whole skill is about writing that
control flow well: giving it exits, shaping it right, verifying its output, and carrying state.

## The core shape

The smallest good loop runs until the work runs **dry**, not for a fixed number of passes. A fixed
"do 3 passes" either stops early or wastes the last passes; looping to dryness adapts to how much is
actually there. Track what's been seen in plain code, ask the agent only for what's new, stop after a
couple of empty rounds.

```ts
import { phase, agent, input, output, type WorkflowMeta } from "@boardwalk-labs/workflow";

export const meta = {
  slug: "loop-until-done",
  triggers: [{ kind: "manual" }],
  budget: { max_usd: 2 },           // a non-negotiable exit (see below)
} satisfies WorkflowMeta;

const { text, maxRounds = 8 } = input as { text: string; maxRounds?: number };
const seen = new Set<string>();
const all: { title: string; detail: string }[] = [];
let dry = 0;
let round = 0;

phase("Hunt");
while (dry < 2 && round < maxRounds) {           // two layered exits in one condition
  round += 1;
  const known = all.map((f) => f.title).join("; ");
  const { findings } = await agent<{ findings: { title: string; detail: string }[] }>(
    `Find issues NOT already in this list; return only NEW ones, empty if none remain.
Known: ${known || "(none yet)"}\n\nMaterial:\n${text}`,
    { schema: FINDINGS_SCHEMA },
  );
  let added = 0;
  for (const f of findings) {
    const key = f.title.trim().toLowerCase();
    if (key && !seen.has(key)) { seen.add(key); all.push(f); added += 1; }
  }
  dry = added === 0 ? dry + 1 : 0;               // count empty rounds; stop after 2 in a row
}
output({ rounds: round, found: all.length });
```

Dedupe in **code**, never by trusting the model not to repeat itself. Everything below makes a loop
you can leave running.

## Always give a loop exits

A loop with no explicit stopping logic is the single most expensive mistake — it runs until the
budget is gone. Don't rely on one exit; **layer** them so no single failure runs the loop away:

1. **A goal check** — the loop ends when the work is genuinely done, confirmed by a *separate*
   verifier (next section), not by the agent grading itself.
2. **A hard iteration cap** — a plain `round < maxRounds` bound, so a non-converging loop still
   terminates.
3. **A budget** — set `budget.max_usd` (and `max_duration_seconds`) in `meta`. Breaching a budget
   *terminates* the run; it doesn't truncate silently.
4. **No-progress detection** — if a round adds nothing new, count it and bail after a few in a row. A
   loop confidently spinning in place is worse than one that stops.

```ts
export const meta = {
  slug: "nightly-cleanup",
  triggers: [{ kind: "manual" }],
  budget: { max_usd: 5, max_duration_seconds: 1800 },  // the runaway backstop
} satisfies WorkflowMeta;
```

## One long run, or many short ones

There are two ways to shape a recurring loop and they are **not interchangeable**. The question is
whether it's *one converging job* or an *open-ended cadence*.

- **Bounded goal loop → one run.** A single run iterates and stops; state lives in memory for free
  (the `seen` set, counters). Use it when the work converges and ends: comb this diff until dry,
  drain this queue, hit this target. That's the example above. A `while`/`for` plus a long `sleep`
  between iterations keeps it one run even across waits.
- **Open-ended cadence → many runs.** When the loop should run indefinitely (every night, forever),
  don't write one run that never ends. Add a `cron` **trigger** so each tick is its own fresh,
  independently-billed-and-capped run; a crash on one tick doesn't kill the schedule. State carries
  between runs in a persistent workspace (next section) or is re-derived from the source of truth.

```ts
export const meta = {
  slug: "repo-maintainer",
  triggers: [{ kind: "cron", expr: "0 3 * * *", timezone: "America/Anchorage" }],
  budget: { max_usd: 5 },
  workspace: { persist: true },        // carry progress between nightly runs
  concurrency: { mode: "serial" },     // never let two ticks clobber that state
} satisfies WorkflowMeta;
```

For a **fixed** cadence, the `cron` trigger in `meta` is all you need. Reach for
`workflows.schedule(slug, input, { cron })` (or `{ rate }` / `{ at }`) **only** when the schedule is
computed at runtime, or one workflow schedules another — it's the dynamic version of the same
pipeline, not a different one.

## Split the maker from the checker

An agent that decides its own work is done is goal drift in one sentence. The biggest reliability win
when looping toward a goal is to verify with a **separate** agent, prompted to *refute* rather than to
agree — only what survives the check counts.

```ts
phase("Verify");
const verdicts = await parallel(
  candidates.map((c) => () =>
    agent<{ real: boolean; reason: string }>(
      `Be a strict, skeptical reviewer. Is this a REAL correctness/security defect, not a
best-practice nice-to-have? Default to real:false if unsure.\nClaim: ${c.title}\n${c.detail}`,
      { schema: VERDICT_SCHEMA },
    )),
);
const confirmed = candidates.filter((_, i) => verdicts[i].real);
```

Verification re-reads the work once per item, so it roughly **doubles** the loop's tokens — the right
trade on output you can't afford to get wrong, skippable on low-stakes work. Keep the material compact
or batch several items per checker call when that cost grows. (Run the checks with `parallel([...])`
so they don't serialize.)

## Remember across runs

A model remembers nothing between runs, so a recurring loop must keep its own record of what it
already did or it repeats itself. Turn on a persistent workspace and read/write a progress file:

```ts
import { existsSync, readFileSync, writeFileSync } from "node:fs";
// meta.workspace = { persist: true }; meta.concurrency = { mode: "serial" };

const donePath = "/workspace/done.json";
const done: string[] = existsSync(donePath) ? JSON.parse(readFileSync(donePath, "utf8")) : [];
// ... do a bounded chunk of work, append to `done` ...
writeFileSync(donePath, JSON.stringify(done));
```

Persisted state is **last-writer-wins**, so pin `concurrency: { mode: "serial" }` whenever a loop
shares it. For work that must never be redone on a restart, put it behind `workflows.call(slug,
input)` — a restarted parent re-attaches to the existing child result instead of running it again.

## Don't pay to wait

Loops wait a lot: between polls, for a build, for a person. Never busy-wait. A long `sleep` **releases
the machine** and re-acquires one on wake, and a human gate does the same — idle time isn't billed, so
a loop can park overnight on an approval or poll for a week and cost nothing in between.

```ts
import { sleep, humanInput } from "@boardwalk-labs/workflow";

for (;;) {
  if (await isHealthy(url)) break;
  await sleep("5m");                  // releases the machine; resumes in 5m, idle free
}

const ok = await humanInput({         // run suspends until a person answers; not billed waiting
  prompt: "Open the PR?",
  input: { kind: "choice", options: ["approve", "skip"] },
});
if (ok.value === "approve") await workflows.call("open-pr", { /* ... */ });
```

## Build and ship it

Use the `boardwalk` CLI (see `use-boardwalk-cli`):

```bash
boardwalk check .                                   # validate the program (no auth/network)
boardwalk run . --org <slug> --input '{"text":"…"}' # deploy + trigger + wait
boardwalk runs <id> --logs                          # read what each round did
```

Two copyable starting points: `boardwalk init my-loop --template loop-until-done` (the bounded goal
loop) and `--template loop-with-verify` (the same loop plus the separate checker).

## Checklist for a good loop

- [ ] **Exits are layered:** a goal/verifier check, a hard `maxRounds`, `budget.max_usd`, and
      no-progress detection. Never just one.
- [ ] **Topology matches the job:** one run for a bounded goal; a `cron` trigger for an open-ended
      cadence; `workflows.schedule` only for a runtime-computed schedule.
- [ ] **The checker is a separate agent**, prompted to refute — the maker never grades itself.
- [ ] **Dedupe and route in code**, not in another `agent()` call.
- [ ] **Recurring loops persist state** (`workspace: { persist: true }` + `concurrency: serial`) or
      re-derive it from the source of truth.
- [ ] **Waits use `sleep`/`humanInput`**, never a busy-wait — idle time should be free.
- [ ] **Secrets stay in code** (`secrets.get`), never in an agent prompt.
