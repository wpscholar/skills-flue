---
name: flue-agent-authoring
description: >-
  Write or debug a Flue agent function — the 'use agent' directive, durable
  identity, agent hooks (useModel, usePersistentState, useInitialData,
  useDelivery, useDataWriter, useInstruction, lifecycle hooks), model
  specifiers, thinking levels, compaction, and agent statics. Use when creating,
  editing, or troubleshooting any agent module in a Flue project.
---

# Flue Agent Authoring

An agent is a **function that returns its system instructions**, re-rendered before every model call. Hooks declare what it can reach. It is React-shaped, but the differences bite — read "Where it diverges from React" below.

Current API: `flue docs read reference/agent-hooks-api` · `flue docs read guide/building-agents`

## The shape

```ts
'use agent';
import { type AgentProps, useModel, useTool } from '@flue/runtime';
import { searchIssues } from '../tools/search-issues.ts';

export function TriageAgent({ id }: AgentProps) {
  useModel('anthropic/claude-sonnet-4-6');
  useTool(searchIssues);
  return `Investigate issue #${id} and recommend the next action.`;
}

TriageAgent.agentName = 'triage-agent';
```

## Registration and identity — the highest-cost mistake

- `'use agent'` is a plain string **before any imports or statements**.
- The build registers **every exported, capitalized function** in a marked module as a top-level agent. One file may export several.
- **The function name becomes the agent's durable identity**, keying its conversation storage — and on Cloudflare, its generated Durable Object class. Renaming the function is a storage migration. Pin it with a string-literal `agentName` static so the function can be renamed freely. Do this on every agent.
- Identities must match `/^[A-Za-z][A-Za-z0-9]*(?:-[A-Za-z0-9]+)*$/`. Duplicates or invalid names throw at registration.
- Renaming the *file* changes nothing.
- Delegate functions used only by `useSubagent` must stay **unexported**, or they register as real agents.

Registration makes an agent addressable by `dispatch()` and `init()`. Serving it over HTTP is a separate decision in `app.ts` — see the `flue-serving-and-clients` skill.

## Rules of hooks

| Rule | Detail |
|---|---|
| `useModel()` exactly once, always | A render that never calls it cannot start. Twice throws. |
| Resource hooks may be conditional | `useTool`, `useSkill`, `useSubagent` can appear and disappear between renders — unlike React. |
| `useDataWriter` names are identity-invariant | Declare them unconditionally, unique names. A delta between renders throws. |
| `useSandbox` at most once | Only its *presence* is observable across renders. |
| Renders are pure reads | Setters, data writers, and `useDispatchMessage`'s dispatcher throw if called during a render. |
| Hooks only inside a render | Elsewhere: `[flue] <hook>() was called outside an agent function.` |
| Duplicate names throw | Within one render, for tools, skills, subagents, and state keys alike. |

Custom hooks are plain functions prefixed `use` that call other hooks; they record as if the body had called them.

Full signatures, subagent-render behavior, and the resource-narration contract: `references/hooks-api.md`.

## Value scoping — the subtlest gotcha

**Submission-scoped** (read once, when the agent wakes to process an accepted input): the `useModel` model, `thinkingLevel`, and `compaction`; the `useSandbox` factory and `cwd`; `useMcpConnection` definitions. A different value computed by a mid-run re-render **latches and applies to the *next* submission** — so an agent that escalates its own model finishes the current response on the old one.

**Per-render**: tool, skill and subagent sets, and instruction text.

There is no per-message model parameter on `dispatch()` or the HTTP surface — model choice belongs to the agent function.

## Statics

```ts
interface AgentStatics {
  agentName?: string;               // string literal in 'use agent' modules
  initialData?: v.GenericSchema;    // valibot; validated once at instance creation
  durability?: { maxAttempts?: number; timeoutMs?: number };  // defaults 10 / 3_600_000
}
```

Statics are applied by the platform while the agent function is *not* running, so `durability` stays in force even across a crash in the agent's own render.

## Models

`useModel('provider-id/model-id', options?)` — everything before the **first** `/` is the provider; the rest may contain slashes (`openrouter/moonshotai/kimi-k2.6`).

```ts
useModel('anthropic/claude-opus-4-6', {
  thinkingLevel: 'high',
  compaction: { reserveTokens: 30000, keepRecentTokens: 16000, model: 'anthropic/claude-haiku-4-5' },
});
```

- `thinkingLevel`: `'off' | 'minimal' | 'low' | 'medium' | 'high' | 'xhigh'` (default `'medium'`). Only reaches the wire for reasoning-capable models.
- `compaction`: `reserveTokens` (model-aware, capped at 20 000), `keepRecentTokens` (default 8000), `model`. `compaction: false` disables the *threshold* trigger only — overflow recovery and `harness.compact()` still compact.
- `useModel` is not callable in a subagent render; a delegate's model comes from its `useSubagent` definition.

Specifiers, catalog metadata, custom providers and Workers AI: `references/models-and-providers.md`.

## Persistent state and data flow

```ts
const [phase, setPhase] = usePersistentState('phase', 'gathering');
setPhase((previous) => next(previous));   // updater — the render value is a snapshot
```

Values must be JSON-serializable. Writes commit **atomically with the unit of work that made them**, which is what makes persistent state the correct guard for at-least-once callbacks.

- `useInitialData<T>()` — creation data. **Write-once**: the `initialData` static validates it at instance creation, and data sent to an existing instance is silently ignored.
- `useDelivery()` — the message currently in front of the model. Read authorization facts from a signal's `attributes` here, never from tool arguments.
- `useDataWriter(name, { schema })` — streams named data parts to the client. **Strictly one-way out**: the model never sees them and a write never re-renders the agent.
- `useInstruction(text)` — appends to the returned instructions. This, not a skill, is how you add always-on content.

## Event hooks

`useAgentStart` (async; once per delivered message — the place to load data before the model runs), `useAgentFinish` (at every would-stop point), `useResponseStart`/`useResponseFinish` (synchronous, once per response; return values merge onto response metadata).

**Callbacks run at-least-once.** Durable outcomes commit atomically per seam and never duplicate, but an external side effect inside one — an email, a page — may rarely happen twice. Guard those with persistent state.

## Prompt-cache discipline

Instructions are rebuilt from scratch every render, and the tool set is re-declared. Both feed the provider's prompt cache.

- Interpolating volatile data (`Date.now()`, a live counter) into instructions busts the cache on every turn. Keep it out of production agents.
- Gate tools on state that changes **rarely**. The safe exception is a tool unlocked by a completed tool call — current Anthropic models (except Haiku) load the new definition where it appeared and the cache survives.
- Skill catalog changes and `activate_skill` do **not** invalidate the cache; a sandbox swap does.

## Anti-patterns

- Renaming an agent function without an `agentName` static (silent storage migration, and a Wrangler failure on Cloudflare).
- Exporting a delegate function from a `'use agent'` module.
- Computing a derived state value from the render snapshot instead of an updater function.
- Expecting a mid-run `useModel` change to affect the response in flight.
- Using a skill for content the agent should always have — that is `useInstruction`.
- Doing async work in the agent body. The function must return synchronously; returning a promise throws. Async work goes in `useAgentStart`, a tool, or a resource factory.
