---
name: flue-skills-subagents-sandboxes
description: >-
  Extend a Flue agent's context and environment — Agent Skills authoring and
  SKILL.md packaging, workspace skills, subagent delegation with useSubagent and
  GeneralSubagent, and sandboxes (virtual just-bash, local(), remote adapters).
  Use when adding skills, delegating work to a subagent, or giving a Flue agent
  filesystem and shell access.
---

# Flue Skills, Subagents, and Sandboxes

Three ways to extend an agent. Pick by what the work needs:

| Need | Primitive |
|---|---|
| Deterministic work your code executes | a **tool** |
| Guidance and resources for the *current* agent | a **skill** |
| Isolation — fresh context, different instructions/model, parallel work | a **subagent** |
| A filesystem and shell | a **sandbox** |

A subagent is **not** a second registered agent: no conversation id, no persistent state, no address. When another part of the system should message an agent over time, register a real agent and `dispatch()` to it.

Current API: `flue docs read guide/skills` · `guide/subagents` · `guide/sandboxes`

## Skills

Flue implements the open [Agent Skills](https://agentskills.io) format, so skills work across harnesses.

```text
src/skills/refunds/
├─ SKILL.md        # frontmatter + instructions
└─ POLICY.md       # supporting file, loaded only if read
```

```ts
import refunds from '../skills/refunds/SKILL.md';
useSkill(refunds);
```

The build recognizes the import, validates the frontmatter, and packages the whole directory. Rules:

- **`name`** must match the directory name; lowercase letters, numbers, hyphens; ≤ 64 chars. **`description`** ≤ 1024 chars and carries the entire routing decision — it is the only part the model always sees.
- Imports must be **static**. A dynamic `import('./skills/x/SKILL.md')` is a build error.
- Each skill name mounts **once per render**; twice throws.
- Packaging **refuses secrets** (`.env`, private keys, credential stores) and **symlinks** as hard errors, warns above 1 MB, and skips `node_modules`/`.git`/`dist`.
- **`allowed-tools` frontmatter is accepted but not enforced** — Flue does not restrict the session's toolset.
- A `.md` import that is not named `SKILL.md` loads as a **plain string**. Wrap it in `defineSkill({ name, description, instructions })` to make it a skill.

Activation: mounted skills appear as one catalog line each; the model calls `activate_skill` and receives the instructions as a **tool result**, so the system prompt never changes and the provider's cached prefix survives. Supporting files are served read-only at virtual paths from the application bundle — never copied into the sandbox, so an agent can't edit its own skill content.

**For content the agent should always have, don't use a skill** — import the markdown as a string and pass it to `useInstruction()`.

An agent with a sandbox also discovers **workspace skills** at `<cwd>/.agents/skills/` with no import. A malformed workspace `SKILL.md` is skipped with a warning; a malformed imported one is a build error; a name collision between the two is an error.

More: `references/skills.md`.

## Subagents

```ts
function Summarizer() {                       // deliberately NOT exported
  return 'You summarize support cases in three sentences.';
}

export function CaseAgent() {
  useModel('anthropic/claude-sonnet-4-6');
  useSubagent({ name: 'summarizer', description: 'Summarizes one support case.', agent: Summarizer });
  return 'Investigate the case. Delegate the summary to the `summarizer` subagent.';
}
```

**Keep delegate functions unexported** — the build registers every exported capitalized function in a `'use agent'` module as a top-level agent.

- The `task` tool is always present with a static spec; its required `agent` parameter only resolves against declared delegates, so an agent with no `useSubagent()` calls cannot delegate.
- The child renders fresh at delegation time, runs as a detached session with its own context window, and **only its final message returns**.
- **The task prompt is the child's entire briefing** — it sees none of the parent's conversation. Instruct the parent to delegate with complete, self-contained prompts.
- Tasks in one tool batch run **in parallel**. Delegation depth is capped at **four levels** (harness invocations count too).
- Inherits the parent's *environment*: sandbox, harness tools, workspace context (`AGENTS.md`, workspace skills), model and reasoning effort unless overridden by `model` / `thinkingLevel` on the definition.
- Inherits **nothing** of the parent's conversation: no history, instructions, tools, skills, subagents, persistent state, or initial data.
- Inside a delegate render, `useModel`, `useSandbox`, `useMcpConnection`, `usePersistentState`, `useDataWriter`, `useDispatchMessage` and the event hooks all **throw**.
- `useSubagent(GeneralSubagent)` mounts a blank fresh-context delegate under the reserved name `flue-general`. It is opt-in and has no instructions of its own.

Since parent and child share a sandbox, **files are the natural hand-off surface**. More: `references/subagents.md`.

## Sandboxes

`useSandbox()` is what adds file and shell access. Without it: no `read`/`write`/`edit`/`bash`/`grep`/`glob`, no workspace in the prompt, and `harness.sandbox` throws.

| Option | Use it when | Watch out for |
|---|---|---|
| **Virtual** — `bash(() => new Bash({ fs: new InMemoryFs() }))` from `just-bash` | Untrusted input, data reshaping, no native binaries needed | **Ephemeral** — the filesystem is rebuilt on every initialization, so files vanish between messages. Network is opt-in via `network.allowedUrlPrefixes`. |
| **`local()`** from `@flue/runtime/node` | Dev tooling, CI, coding agents, self-hosted automation | **No isolation, by design.** Never an isolation boundary for untrusted requests or multiple tenants. |
| **Remote adapters** (`flue add sandbox e2b`) — Daytona, E2B, Modal, Cloudflare Sandbox, Cloudflare Computer, Vercel, Mirage, … | Real Linux, durable workspaces, per-session isolation | Most provider SDKs have **no mid-flight cancellation** — a cancelled command keeps running remotely and its output is discarded. |

`local()` does **not** inherit `process.env`. Only shell essentials (`PATH`, `HOME`, `USER`, `LANG`, `TERM`, `TMPDIR`) pass by default — never API keys or cloud credentials. Opt in per variable via `env`, and set a key to `undefined` to drop a default. The snapshot is taken once at construction. `env: { ...process.env }` hands the model your entire host environment.

Attachment rules: at most once per render (twice throws); throws in a subagent render; the factory is lazy and receives the agent instance id, so an adapter that looks up an existing provider sandbox by that id gives each conversation a durable workspace. A conditional `useSandbox()` is re-read at every turn boundary and narrated as a full `environment` signal — but the **system prompt keeps describing the old workspace until the next compaction**, and the swap invalidates the prompt cache.

More: `references/sandboxes.md`.

## Anti-patterns

- Exporting a delegate function from a `'use agent'` module.
- A skill for always-on content (use `useInstruction`), or a subagent for deterministic work (use a tool).
- Delegating with a terse prompt that assumes the child can see the conversation.
- Storing anything that must survive in a virtual sandbox — use persistent state, or a durable workspace keyed on the instance id.
- Passing `env: { ...process.env }` to `local()` outside a fully trusted environment.
- Assuming `bash`/`grep`/`glob` exist — a sandbox adapter's `tools` function replaces the default set.
