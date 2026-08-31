# Skills in Flue

Live source: `flue docs read guide/skills` · spec: https://agentskills.io · validator: https://github.com/agentskills/agentskills/tree/main/skills-ref

A skill is packaged expertise loaded on demand. Three parts: a short `name` the model activates by, a `description` stating what it does **and when to use it**, and `instructions` — the full procedure, loaded only on activation. Optional supporting files stay unloaded until the model reads one.

Progressive disclosure: each mounted skill costs **one always-present catalog line** in the system prompt.

## Directory layout

```text
src/skills/refunds/
├─ SKILL.md
└─ POLICY.md
```

```markdown
---
name: refunds
description: Process a customer refund request end-to-end. Use when a customer asks for a refund or disputes a charge.
---

Follow this procedure for every refund request:

1. Confirm the order ID and the reason for the refund.
2. Read `POLICY.md` and check the eligibility rules for that reason.
3. If eligible, issue the refund with the `issue_refund` tool and confirm the amount.
4. If not eligible, explain which rule applies and offer the alternatives in the policy.
```

## Frontmatter fields

Validated against the Agent Skills specification.

| Field | Rules |
|---|---|
| `name` (required) | Lowercase letters, numbers, hyphens. No leading, trailing, or consecutive hyphens. **≤ 64 characters. Must match the skill directory name.** |
| `description` (required) | Non-empty, **≤ 1024 characters**. The only part the model always sees — it carries the entire routing decision. |
| `license` | Accepted, informational only. |
| `compatibility` | Accepted, **≤ 500 characters**, informational only. |
| `metadata` | Accepted; string-to-string map; not interpreted by Flue. |
| `allowed-tools` | **Accepted but NOT enforced.** Experimental in the spec; Flue does not restrict the session's toolset. |

Unknown fields are ignored.

## Importing and mounting

```ts
'use agent';
import { useModel, useSkill } from '@flue/runtime';
import refunds from '../skills/refunds/SKILL.md';

export function SupportAgent() {
  useModel('anthropic/claude-haiku-4-5');
  useSkill(refunds);
  return 'Answer customer support questions clearly and accurately.';
}
```

At build time Flue recognizes the import, validates the frontmatter, and packages the entire directory. The import's value is a typed `SkillReference`. No registration step, no runtime file copying — works in `vite dev`, `vite build`, and `flue run`. Type declarations for `SKILL.md` and `.md` imports ship with `@flue/runtime`. In dev, editing any file in the skill directory is hot-picked-up.

From a package:

```ts
import review from '@acme/review-skills/review/SKILL.md';
```

The package must publish `SKILL.md` and its supporting files, and must export the imported subpath if it defines package exports.

### Rules

- Imports must be **static** — a dynamic `import('./skills/x/SKILL.md')` is a **build error**.
- Each skill name mounts **once per render**; mounting the same name twice **throws**.
- Packaging skips repository noise (`node_modules`, `.git`, `dist`), **warns on files over 1 MB**, and **refuses to package secrets** — `.env` files, private keys, credential stores, and **symlinks are hard errors**.
- Mounts may be conditional; catalog changes are narrated **without invalidating the cached prompt**.

## `defineSkill()`

```ts
import { defineSkill } from '@flue/runtime';

export const escalation = defineSkill({
  name: 'escalation',
  description:
    'Escalate an unresolved case to a human specialist. Use when the customer asks for a human or the issue is out of scope.',
  instructions:
    'Summarize the case so far, tag the conversation with the escalation reason, and hand off with the `escalate_case` tool.',
});
```

`instructions` is the `SKILL.md` body. Optional `files` carries supporting resources keyed by relative path:

```ts
const reviewSkill = defineSkill({
  name: 'review-pr',
  description: 'Review a pull request against the team checklist. Use when asked to review code.',
  instructions: 'Read CHECKLIST.md, then review the diff against every item.',
  files: { 'CHECKLIST.md': checklistText },
});
```

Returns the definition frozen. Nothing is packaged at definition time — packaging happens when the skill is first needed, and the runtime writes spec-valid frontmatter itself.

**A bare `.md` import (not named `SKILL.md`) loads as a plain string** — nothing is packaged. Pass it through `defineSkill` to make it a skill:

```ts
import runbook from './incident-runbook.md'; // plain markdown text

export const incidents = defineSkill({
  name: 'incidents',
  description: 'Run the incident response procedure. Use when an outage or security event is reported.',
  instructions: runbook,
});
```

## Activation mechanics

Mounted skills appear in an **Available Skills** section of the system prompt — one line each. The runtime provides an `activate_skill` tool; the model calls it with the skill's name and receives the full instructions as the **tool result**. Because instructions arrive as a tool result, the system prompt never changes and **the provider's cached prompt prefix survives**. Activating an unknown name is not an error — the result lists the available skills.

Steering activation from instructions:

```ts
useSkill(refunds);
return 'Activate the `refunds` skill before handling any refund request.';
```

A harness tool's `harness.prompt(...)` runs with the agent's rendered configuration — same system prompt, skill catalog, tools — so naming the skill in the prompt text activates it there too, workspace-discovered skills included.

**For content the agent should *always* have, don't use a skill.** Import the markdown as a string into your instructions, or pass it to `useInstruction(...)`.

## Supporting files at runtime

Packaged supporting files travel **inside your application bundle, not the agent's workspace**. Nothing is copied into the sandbox; the runtime serves each file read-only at a virtual path, and the activation briefing lists every resource with its exact path. File-reading tools resolve those paths transparently, and a `read_skill_resource` tool is added whenever a mounted skill carries supporting files. The same bundle serves the same files on a laptop, in Node.js, or on Cloudflare — so an agent can never accidentally edit its own skill content.

## Workspace skills

An agent with a sandbox discovers skills at session start by scanning `.agents/skills/` in the sandbox's working directory. Every valid `<name>/SKILL.md` joins the catalog — no import, no `useSkill(...)`.

```text
<cwd>/.agents/skills/
└─ greet/
   └─ SKILL.md
```

- Instructions are read **from disk at activation time**, so mid-session edits are picked up.
- Supporting files are ordinary workspace files the model reads directly.
- **A malformed workspace `SKILL.md` is skipped with a warning** rather than failing the session. Imported skills are validated **strictly at build time**.
- **A workspace skill whose name collides with a declared skill is an error.**

Use workspace skills when expertise ships with the workspace (a repo checkout, a CI environment); use declared skills when it belongs to your application.
