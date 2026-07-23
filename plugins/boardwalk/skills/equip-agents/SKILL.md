---
name: "equip-agents"
description: "Use when a Boardwalk workflow needs to give an agent() more than a prompt: reusable skills, inline tools, MCP servers, persistent memory, or a human-input gate. Covers the per-call agent() capabilities (skills, tools, mcp, memory, cwd, humanInput, and builtins scoping), short-lived credentials via auth.idToken/auth.apiToken, and the workflow package that bundles a skills/ folder and other assets. A skill is a skills/<name>/SKILL.md the leaf loads on demand through the built-in skill tool (progressive disclosure), anchored to the code-review example. Pairs with boardwalk-use-cli to scaffold, build, and deploy the package."
---

# Equip an agent() with skills, tools, MCP, and memory

The engine's built-in coding tools (read, write, edit, ls, grep, glob, bash, webfetch, and more) are
on by default, so a plain `agent(prompt)` already works in the run's workspace. Everything beyond that
is added **per `agent()` call**, on its options object. None of it is declared in `workflow.jsonc`:
two `agent()` calls in the same workflow can carry different skills, tools, and memory.

## The workflow package

A workflow is a **package**: the directory holding the descriptor and the entry. Point
`boardwalk check`/`build`/`deploy` at the directory.

```text
code-review/
  workflow.jsonc        # the deployment descriptor
  package.json
  README.md             # the workflow's landing page in the dashboard
  src/index.ts          # the entry: exports run
  src/lib/parse.ts      # helper modules the entry imports normally
  skills/
    reviewer/
      SKILL.md          # a reusable procedure
      checklist.md      # a bundled resource the skill can pull on demand
```

You never list source files: the code that ships is whatever the entry imports. `skills/**` and
`README.md` ride along by convention; other non-code assets (prompt templates, fixtures) ship via
the `files` allowlist in `workflow.jsonc`. The `skills/` folder is not imported; it is bundled and
resolved by name at run time (below).

## Give an agent reusable skills

`agent(prompt, { skills: ["reviewer"] })` makes `skills/reviewer/SKILL.md` available to that leaf. It
loads with **progressive disclosure**, the same model these plugin skills use:

- The leaf sees a compact catalog: each named skill's `name` and `description` (from its SKILL.md
  frontmatter) ride the prompt.
- The model calls the built-in `skill` tool to load a skill's full body when it needs it:
  `skill({ name: "reviewer" })`.
- It pulls a bundled resource from the skill's folder only if required:
  `skill({ name: "reviewer", file: "checklist.md" })`.

```ts
import { agent } from "@boardwalk-labs/workflow";

interface Change { diff: string }

export default async function run(input: Change): Promise<{ review: string }> {
  const review = await agent(
    `Review this change. You have a "reviewer" skill: call the \`skill\` tool to load its
instructions before you start, and load its checklist resource if you need the detail.\n\n${input.diff}`,
    { skills: ["reviewer"] },
  );
  return { review };
}
```

The bundled `skills/reviewer/SKILL.md` is a plain file with `name` and `description` frontmatter, the
same format as this file. The payoff: the standing prompt stays small, the procedure stays versioned
next to the code, and every `agent()` that names it reuses it.

## Inline tools

For a one-off tool specific to this workflow, define it inline. `execute` runs in the program process,
the trusted layer, so it may read secrets; only its **return value** enters model context, with secret
values redacted. Inline tools are added on top of the built-ins.

```ts
const answer = await agent("What is the on-call rotation today?", {
  tools: [
    {
      name: "pagerduty_oncall",
      description: "Look up who is on call for a schedule right now.",
      inputSchema: { type: "object", properties: { schedule: { type: "string" } }, required: ["schedule"] },
      execute: async (input) => fetchOncall((input as { schedule: string }).schedule),
    },
  ],
});
```

## MCP servers

Attach MCP servers inline. The program supplies any credentials directly (it is the trusted layer);
`stdio` launches a local server, `http` connects to a remote one.

```ts
await agent("Summarize the open issues.", {
  mcp: [
    { name: "github", transport: "stdio", command: "npx", args: ["-y", "@modelcontextprotocol/server-github"] },
    { name: "internal", transport: "http", url: "https://mcp.internal/v1", headers: { Authorization: `Bearer ${token}` } },
  ],
});
```

## Short-lived credentials (`auth`)

When a tool or MCP server calls your own systems, don't bake in long-lived keys — mint on demand.
Both mints are actions, so they are imports (never `context` fields), and both are redacted from
model context like any secret:

- `auth.idToken(audience)` — a short-lived OIDC JWT asserting this run's identity, for keyless
  federation into AWS/GCP/Azure instead of cloud keys in secrets. Requires
  `"permissions": { "id_token": "write" }` in `workflow.jsonc`, plus a trust policy in the target
  cloud pinned to the Boardwalk issuer.
- `auth.apiToken()` — a short-lived, run-scoped bearer for Boardwalk's own API/MCP; pass it into an
  MCP `headers` block or a subprocess env explicitly.

## Persistent memory

`agent(prompt, { memory: "memory/triager" })` gives the leaf a workspace-relative directory the engine
persists across runs **automatically** (no `workspace.persist` needed). The leaf gets read/write file
tools scoped to that directory and loads its index at the start of each turn; your program code can
read and write the same files. Point two agents at the same path to share, or separate paths to isolate.

## Human input mid-loop

`agent(prompt, { humanInput: true })` gives the leaf a `human_input` tool, so the model itself can pause
the run to ask a person; it is off by default. For a gate in your own deterministic code instead, call
the top-level `humanInput()` primitive (see `boardwalk-overview`).

## Work from a subdirectory

`agent(prompt, { cwd: "checkouts/repo-a" })` re-roots the leaf's workspace view to an EXISTING
subdirectory: file tools resolve and confine there, `bash` starts there, and the leaf is told that
directory's layout — so a run that clones several repos gives each agent one checkout and clean
repo-relative paths. Create the directory in program code first (the call fails loudly on a missing
path). `memory` stays workspace-root-relative, and a `subagent` inherits its parent's `cwd`.

## Scope the built-ins

`builtins` controls which engine tools a leaf gets: `"all"` (default), `"read-only"`, `"none"`, or an
explicit subset. A judge or classifier wants `builtins: "none"` plus a `schema`; analysis wants
`"read-only"`. Fewer tools mean a smaller prompt and fewer stray turns.

## Gotchas

- **`skills/` rides by convention.** `agent({ skills })` resolves from the deployed package's
  `skills/` folder — no `files` entry needed; just put `skills/<name>/SKILL.md` in the package
  before you name it.
- **All of this is per `agent()` call, never a descriptor field.** `workflow.jsonc` declares no
  tools, skills, MCP servers, or memory. Capabilities live on each call.
- **Keep credentials in code.** The program may hold secrets (`secrets.get`) and hand them to a tool or
  MCP server; secret values are redacted from model context, so never put them in the prompt.
- **Two different SKILL.md consumers.** This file is a Claude Code plugin skill. A workflow's
  `skills/<name>/SKILL.md` is loaded by `agent({ skills })` at run time. Same file format, different
  reader; don't conflate them.

## Where to go next

- To scaffold, build, and deploy the package (directory in, one artifact out): use **`boardwalk-use-cli`**.
- For what a workflow is and how the pieces fit: use **`boardwalk-overview`**.
