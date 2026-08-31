# Flue agent hooks — signatures and contracts

Live source: `flue docs read reference/agent-hooks-api` · `flue docs read reference/agent-api`

All symbols export from `@flue/runtime` unless noted.

The hooks: `useModel()` (required), `useSandbox()`, `useTool()`, `useMcpConnection()`, `useSkill()`, `useSubagent()`, `useInstruction()`, `usePersistentState()`, `useInitialData()`, `useDelivery()`, `useDispatchMessage()`, `useDataWriter()`, `useAgentStart()`, `useAgentFinish()`, `useResponseStart()`, `useResponseFinish()`.

## Agent function types

```ts
type AgentFunction<TProps = void> = TProps extends void
  ? () => string | undefined | void
  : (props: TProps) => string | undefined | void;

type Agent = AgentFunction<AgentProps> & AgentStatics;

interface AgentProps { id: string; }
```

- Must return **synchronously**; returning a promise throws.
- Return value must be a string, `undefined`, or `void`.
- Renders never nest — directly invoking another agent function throws.
- Only the **root** agent function receives props. Subagents are called with no arguments and must close over values.

Durable identity resolves in order: the build-stamped binding → the `agentName` static → the function's own `name`. Must match `/^[A-Za-z][A-Za-z0-9]*(?:-[A-Za-z0-9]+)*$/`.

## `useModel()`

```ts
function useModel(model: string, options?: UseModelOptions): void;

interface UseModelOptions {
  thinkingLevel?: ThinkingLevel;
  compaction?: false | CompactionConfig;
}

type ThinkingLevel = 'off' | 'minimal' | 'low' | 'medium' | 'high' | 'xhigh';

interface CompactionConfig {
  reserveTokens?: number;    // default model-aware, capped at 20 000
  keepRecentTokens?: number; // default 8000
  model?: string;            // defaults to the session model
}
```

Unknown option fields throw. An unresolvable specifier fails the submission at initialization. Unset `thinkingLevel` → the harness substitutes `'medium'`.

> The guide page and the reference page disagree on whether `'max'` is a valid `thinkingLevel`. Trust `flue docs read reference/agent-hooks-api` for the installed version.

## `useSandbox()`

```ts
function useSandbox(sandbox: SandboxFactory, options?: { cwd?: string }): void;
```

At most once per render; throws in a subagent render. `createSandbox()` runs once per initialized harness. Re-renders never rebuild the environment — only **presence** is observable, re-read at every turn boundary. A factory swap while staying attached takes effect at the next submission's initialization. A value without `createSandbox` throws; a non-function `tools` property throws.

## Resource hooks

```ts
function useTool(tool: ToolDefinition): void;
function useMcpConnection(definition: McpConnectionDefinition): void;
function useSkill(skill: SkillReference | SkillDefinition): void;
function useSubagent(subagent: SubagentDefinition): void;
function useInstruction(text: string): void;
```

- `useTool` — one flat tool set. Duplicate names throw `ToolNameConflictError`. An unmounted tool cannot be called at all.
- `useMcpConnection` — connects when a submission initializes, all declared servers in parallel; tools mount as `mcp__<server>__<tool>`. Definitions read at first connect (`auth` excepted — resolved per request). A failed connect fails the submission unless `optional: true`. Throws in a subagent render.
- `useSkill` — one always-present catalog line per skill; the model pulls instructions with `activate_skill`, delivered as a tool result so the prompt prefix never changes.
- `useSubagent` — the `task` tool is always present with a fully static spec; the roster lives in the system prompt's "Available Agents" section; with an empty roster the tool is inert.
- `useInstruction` — text lands after the returned instruction, in call order, joined with blank lines. Non-empty after trimming. Works in root and subagent renders, any number of times.

## `usePersistentState()`

```ts
function usePersistentState<T>(name: string, defaultValue: T): [T, StateSetter<T>];
function usePersistentState<T = unknown>(name: string): [T | undefined, StateSetter<T | undefined>];
```

JSON-serializable values, keyed by name. The render value is a snapshot — pass an updater `(previous) => next` for derived writes. Every write is a record in the canonical conversation stream, committed atomically with the unit of work that made it: a write from a tool commits with that turn's tool batch, a write from an event hook with the hook seam's checkpoint. If recovery settles the batch as interrupted, the write never happened.

## `useInitialData()` and the `initialData` static

```ts
'use agent';
import * as v from 'valibot';
import { useInitialData, useModel } from '@flue/runtime';

export function Triage() {
  useModel('anthropic/claude-opus-4-6');
  const data = useInitialData<v.InferOutput<typeof Triage.initialData>>();
  return `Triage GitHub issue #${data!.issue} end-to-end.`;
}

Triage.initialData = v.object({ issue: v.pipe(v.number(), v.integer()) });
```

Supplied three ways:

```ts
await dispatch(Triage, { id: 'issue-17307', initialData: { issue: 17307 }, message: 'New issue.' });
```
```bash
flue run ./triage.ts --id issue-17307 --data '{"issue": 17307}' -m "New issue."
```
```bash
curl -X POST https://example.com/agents/triage-agent/issue-17307 \
  -H 'Content-Type: application/json' \
  -d '{"initialData": {"issue": 17307}, "kind": "user", "body": "New issue."}'
```

**Validated once at instance creation. Data sent to an existing instance is ignored and the recorded value never changes.** Pair with `uid: null` to error instead of silently continuing. Returns `undefined` (rather than throwing) inside a subagent render.

## `useDelivery()`

Returns the `DeliveredMessage` currently in front of the model. Inside a subagent render it returns the parent's task prompt. Narration signals never advance its cursor.

```ts
const delivery = useDelivery();
const customerId = delivery.kind === 'signal' ? delivery.attributes?.customerId : undefined;
```

This is the authorization surface — tool arguments are model-selected inputs, not an authorization boundary.

## `useDataWriter()`

```ts
const writeOrderCard = useDataWriter('orderCard', {
  schema: v.object({ orderId: v.string(), status: v.picklist(['loading', 'loaded']) }),
});
```

Each write arrives as a named data part on the conversation message (`{ type: 'data-orderCard', data: … }`), read on the client from `part.type === 'data-orderCard'` or from `AgentReply.data`. **Declare writers unconditionally with unique names** — a delta between consecutive renders throws with the exact added/removed names. Strictly one-way out: the model never sees data parts and a write never re-runs the agent function.

## Event hooks

```ts
useAgentStart(async ({ harness }) => { ... });   // once per delivered message; awaited
useAgentFinish(async (ctx) => { ... });          // at every would-stop point; awaited
useResponseStart(() => ({ startedAt: Date.now() }));                     // synchronous
useResponseFinish(({ metadata, response }) => ({ ... }));                // synchronous
```

- One response may absorb several delivered messages.
- `useAgentStart`/`useAgentFinish` may be async and receive a harness. `useResponseStart`/`useResponseFinish` are synchronous — a returned promise fails the submission.
- Return values from the response hooks merge onto the response's **metadata** envelope field.
- Declarations have no durable identity; identity within a render is declaration order.
- A callback throw fails the submission.
- **At-least-once.** Guard non-idempotent external effects with persistent state.
- `useAgentFinish` continuation is capped at **32 cycles per response** (not configurable).

```ts
useResponseStart(() => ({ startedAt: Date.now() }));
useResponseFinish(({ metadata, response }) => ({
  elapsed: Date.now() - (metadata.startedAt as number),
  totalTokens: response.usage.totalTokens,
}));
```

`useResponseFinish` receives the whole response's aggregate usage — the right place to stamp token counts onto response metadata for a client.

## Subagent renders

Throw: `useModel`, `useSandbox`, `useMcpConnection`, `usePersistentState`, `useDataWriter`, `useDispatchMessage`, and all four event hooks.
Work: `useTool`, `useSkill`, `useInstruction`, nested `useSubagent`, custom hooks.
Degrade: `useInitialData()` returns `undefined`; `useDelivery()` returns the parent's task prompt.

## Dynamic resource narration

Conditional tools, skills and subagents are diffed against the last-narrated snapshot; changes append as signals rather than rewriting the system prompt, so the provider's cached prefix survives.

- `resources` signal, one per changed kind (`skill`, `tool`, `subagent`, `mcp`)
- `instructions` signal when the composed instruction document changes (body is the fixed marker `"System instructions updated."`)
- `environment` signal on a `useSandbox` presence flip — always a full snapshot, superseding that boundary's delta narration
- Compaction rebaselines the snapshot; delta narration starts fresh

**Reserved signal types** — rejected at `dispatch()` admission: `resources`, `instructions`, `environment`, `stream_interrupted`, `stream_continued`, `submission_aborted`, `submission_interrupted`, `compaction`, `memory`.

## Custom hooks

```ts
function useEscalation() {
  useTool(escalateCase);
  return 'Escalate to a specialist only after you have confirmed the account and issue.';
}

export function SupportAssistant() {
  useModel('anthropic/claude-haiku-4-5');
  return `Answer support questions accurately. ${useEscalation()}`;
}
```
