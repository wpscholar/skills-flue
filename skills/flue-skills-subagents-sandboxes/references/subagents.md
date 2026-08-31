# Subagents in Flue

Live source: `flue docs read guide/subagents` · `flue docs read reference/agent-hooks-api` (`#usesubagent`)

> The `.../guide/subagents/index.md` raw-markdown mirror serves a stale May 2026 snapshot documenting a removed API (`defineAgentProfile`, `defineAgent(() => ({...}))`, `subagents: [...]`). Several `index.md` mirrors are stale this way — trust `flue docs` or the rendered HTML page.

A subagent is a named delegate an agent hands a focused task to. The delegate works in its own fresh context, with its own instructions and capabilities, and **only its final answer returns** to the parent's conversation.

## Declaring

```ts
'use agent';
import { useModel, useSubagent } from '@flue/runtime';

function Summarizer() {                    // NOT exported
  return 'You summarize support cases in three sentences.';
}

export function CaseAgent() {
  useModel('anthropic/claude-sonnet-4-6');
  useSubagent({ name: 'summarizer', description: 'Summarizes one support case.', agent: Summarizer });
  return 'Investigate the case. Delegate the summary to the `summarizer` subagent.';
}
```

Three required fields: `name`, `description`, `agent`. Optional `model` and `thinkingLevel` override inheritance — routing a delegate to a cheaper model is a common pattern.

**`Summarizer` is deliberately not exported.** A delegate's agent function is not a registered agent: it has no `useModel()` call, no conversation id, and no HTTP surface. The build registers every *exported* capitalized function in a `'use agent'` module as a top-level agent — so keep delegates unexported, or define them in ordinary modules.

Conditional declaration is allowed and narrated. Declaring two delegates with the same name in one render throws.

## How delegation works

Every agent's tool set includes a framework-owned **`task`** tool; delegates are cataloged by name and description in an **"Available Agents"** section of the system prompt. When the model calls `task`:

1. The runtime renders the delegate's `agent` function — fresh, at delegation time, in its own frame — into the child's instructions, tools, and skills.
2. The child runs as a **detached session in the parent's environment**, with its own context window, and runs to completion.
3. **Only the child's final message returns** as the `task` result — not intermediate reasoning, tool calls, or file reads.

The `task` tool is always present with a **fully static spec** (changing it would rewrite the serialized tools block and invalidate the provider's prompt cache), but its required `agent` parameter only resolves against declared delegates — with an empty roster the tool is inert. The model may also pass an optional `cwd` to point the child at a different working directory, and can forward images from the conversation by attachment id.

Two properties follow from fresh-context design:

- **The prompt is the entire briefing.** The child does not see the parent's conversation. Parent instructions should tell the model to delegate with complete, self-contained prompts.
- **Tasks parallelize.** Tool calls in one batch execute in parallel — five independent checks become five concurrent child sessions, each with its own context window instead of the parent's.

Nested `useSubagent()` is allowed; **the runtime caps delegation depth at four levels** (harness invocations count against the same cap). Child sessions write their own durable records, so an interrupted task resumes.

A harness tool's `harness.prompt(...)` conversation can also delegate to declared subagents — application code can *require* a particular delegate by naming it in the instruction.

## What a subagent inherits

**Inherits the parent's environment:**
- the sandbox and its harness tools (`read`, `write`, `bash`, …)
- workspace context discovered from the working directory (`AGENTS.md`, workspace skills)
- the parent's model and reasoning effort, unless overridden

**Inherits nothing about the parent's conversation:** not its history, instructions, tools, skills, or subagents, and **not its persistent state or initial data**.

```ts
function Reproducer() {
  useTool(searchIssues);
  useSkill(reproduceSkill);
  return 'You reproduce one reported issue. Write your findings to report.md.';
}

export function Triage() {
  useModel('anthropic/claude-sonnet-4-6');
  useSubagent({
    name: 'reproducer',
    description: 'Sets up the reproduction for one issue and writes report.md.',
    agent: Reproducer,
  });
  return 'Investigate the reported issue. Delegate the reproduction to the `reproducer` subagent.';
}
```

Because parent and child share a sandbox, **files are a natural hand-off surface**.

### Hooks inside a delegate render

| Throws | Works |
|---|---|
| `usePersistentState()` — durable state belongs to the parent instance | `useTool()` |
| `useSandbox()` — delegates share the parent's environment | `useSkill()` |
| `useModel()` — the model comes from the definition | `useInstruction()` |
| `useMcpConnection()` | nested `useSubagent()` |
| `useDataWriter()`, `useDispatchMessage()`, all four event hooks | custom hooks |

`useInitialData()` returns `undefined` rather than throwing; `useDelivery()` returns the parent's task prompt.

## `GeneralSubagent`

```ts
import { GeneralSubagent, useModel, useSubagent } from '@flue/runtime';

export function Researcher() {
  useModel('anthropic/claude-sonnet-4-6');
  useSubagent(GeneralSubagent);
  return 'Answer questions about this codebase. Fan independent research out to the `flue-general` subagent, one question per task.';
}
```

Mounts under the **framework-reserved name `flue-general`**. Its agent function is deliberately empty — the child gets the shared environment (sandbox tools, workspace context, parent's model) and **no instructions, tools, or skills of its own**. Everything it knows comes from the task prompt, so prompts to it must be complete briefings. It is **opt-in**.

## `defineSubagent()`

```ts
import { defineSubagent } from '@flue/runtime';

function IssueClassifier() {
  return 'Return the likely product area and urgency for the reported issue.';
}

export const issueClassifier = defineSubagent({
  name: 'issue_classifier',
  description: 'Classifies support issues for routing.',
  agent: IssueClassifier,
});
```

Validates at **module load** rather than first render and returns the definition frozen. Per-mount overrides spread cleanly:

```ts
useSubagent({ ...issueClassifier, model: 'anthropic/claude-haiku-4-5' });
```

## When to delegate

- Exploratory work would flood the parent's context but produces a short answer — research, codebase exploration, log analysis.
- One phase of a workflow needs different instructions, tools, or skills than the rest of the conversation.
- Independent pieces of work can run in parallel, each in its own context window.
- A class of work should run on a different model or reasoning effort than the parent.

Against the neighbours: a **tool** is a bounded function your application code executes — reach for one when the work is deterministic. A **skill** adds instructions and resources to the *current* agent — reach for one when the agent needs guidance, not isolation. And a subagent is **not** a second registered agent: no conversation id, no persistent state, no address. When another party in your system should message an agent over time, register a real agent and `dispatch()` to it.

## Durability

A subagent task runs as a child session with its own durable conversation stream. Recovery reattaches to the child's transcript, resumes it to completion, and commits its real final result as the parent's tool outcome — recursively, and for every task in an interrupted parallel batch.

- A delegate has **no durability configuration of its own** — resumed child work runs inside the parent's attempt, under the parent's retry budget and timeout.
- **A delegate removed or renamed by a redeploy cannot be resumed** under any retry: that one call settles with an error outcome and the parent continues.
- The interrupted marker carries the child's conversation id, so the transcript stays inspectable.
