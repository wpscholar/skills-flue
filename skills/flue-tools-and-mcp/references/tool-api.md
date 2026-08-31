# Flue tool API reference

Live source: `flue docs read reference/agent-api` (`#definetool`) · `flue docs read guide/tools`

## `defineTool()`

```ts
import { defineTool } from '@flue/runtime';
import * as v from 'valibot';

export const checkInventory = defineTool({
  name: 'check_inventory',
  description: 'Check the stock level for one SKU.',
  input: v.object({ sku: v.string() }),
  output: v.object({ inStock: v.number(), warehouse: v.string() }),
  async run({ data, signal, log, toolCallId }) {
    return { output: await inventory.lookup(data.sku) };
  },
});
```

Validates and returns the definition **frozen**. `useTool` accepts either a `defineTool` result or an inline object.

### Input

Must be a **top-level valibot object schema**. Arguments are parsed before `run`; `run` receives the parsed value as `data`, fully typed. On validation failure `run` is never called — the failure returns to the model as a tool error so it can correct and retry.

### Output

`run` returns `{ output?, terminate? }`.

- `output` — JSON-serializable, JSON-stringified for the model.
- A bare `string` return is shorthand for `{ output: <string> }`.
- Returning nothing is allowed **only** when no `output` schema is declared.
- Any other bare return (object, number, boolean, array, null) **throws**.
- `terminate: true` ends the agent's turn once the current tool batch settles (same contract as `finish`/`give_up`).
- An optional `output` valibot schema validates and types the result.

### Errors

A throw inside `run` does not crash the agent — it becomes an error result the model sees. Throw rather than swallow: the model can only respond to failures it can see.

### `run` context

| Field | Notes |
|---|---|
| `data` | The parsed input. |
| `signal` | `AbortSignal` for this call. A `run` that ignores it is abandoned when the signal fires — the call fails with an `AbortError` saying the work may still be running, and the orphaned promise's result is discarded. |
| `log` | `log.info/warn/error(message, attributes?)`. Lines stream into the conversation as events your application can observe. **They are not part of the result and the model never sees them.** |
| `toolCallId` | This call's id, matching the id on the call's conversation events. |
| `harness` | Only with `harness: true`. |
| `step` | Only with `durable: true`. |

## Built-in tools (sandbox-gated)

| Tool | What it does |
|---|---|
| `read` | Read a file (truncated to 2000 lines or 50 KB; supports offset/limit) |
| `write` | Write a file, creating it and parent directories |
| `edit` | Edit a file by exact text replacement |
| `bash` | Execute a shell command, return stdout/stderr |
| `grep` | Search file contents for a regex |
| `glob` | Find files by filename pattern |

Framework-added: `task` (subagent delegation — **always present**), `activate_skill` (when the agent has skills), `read_skill_resource` (when a skill packages supporting files). **These names are reserved.**

Full behavior: `flue docs read reference/agent-behavior` (`#built-in-tools`).

## Harness tools

```ts
interface FlueHarness {
  readonly name: string;
  prompt<S extends v.GenericSchema>(text: string, options: PromptOptions<S> & { result: S }): CallHandle<PromptResultResponse<v.InferOutput<S>>>;
  prompt(text: string, options?: PromptOptions): CallHandle<PromptResponse>;
  compact(): Promise<void>;
  readonly sandbox: Sandbox;
}

interface PromptOptions<S = undefined> {
  result?: S;                  // valibot schema — requires validated structured data
  tools?: ToolDefinition[];     // extra tools for just this operation
  model?: string;
  thinkingLevel?: ThinkingLevel;
  signal?: AbortSignal;
  images?: PromptImage[];
}
```

- Handed to `harness: true` tools and to `useAgentStart`/`useAgentFinish` contexts. Scoped to the invocation that received it; no direct construction.
- `harness.prompt()` runs in the harness's own **scratch conversation**, separate from the agent's public conversation. Repeated calls continue it.
- It runs with the agent's rendered configuration — same system prompt, skill catalog, tools — so naming a skill in the prompt text activates it there too, and it can delegate to declared subagents.
- `harness.sandbox` exposes `readFile`, `writeFile`, `exec`, `cwd`, `resolvePath` and other verbs with **no conversation record**. **Throws when the agent declared no sandbox.**
- Harness invocations materialize when the call runs and close when it settles; they **count against the four-level delegation cap**, and any child conversations they open are retained on the parent conversation.

## Durable tools

```ts
durable: true,
async run({ data, step }) {
  const tenant = await step.do('create-tenant', () => billing.createTenant(data.customerId));
  ...
}
```

`step.do(name, fn)` runs `fn` once per name per tool call and durably records its return value before resolving. On recovery the whole call re-runs; completed steps return their recorded values without executing, and execution continues from the first unfinished step.

- Step records are keyed to the **tool call id**, so they carry across attempts of the same call and are scoped to it — a fresh invocation runs every step fresh.
- The model sees only the final result; step progress surfaces as the call's log events.
- Reusing a step name within one call **throws**.
- A thrown error is not an interruption — nothing retries automatically.
- **A redeploy that removes the tool, or removes its `durable` flag, withdraws the recovery contract** for in-flight calls; they fall back to the interrupted-marker path.

## Conditional tools and narration

Set changes are announced to the model in a `resources` signal at the next turn boundary ("New tool available: …"). Tool updates are announced name-only; entry counts update when a description changes or when a tool's input-schema digest changes.

> From the docs: changing the tool set rewrites the provider's tools array, which invalidates its prompt cache, so gate tools on state that changes rarely. The exception is a tool unlocked by a completed tool call — current Anthropic models (except Haiku) load its definition at the point in the conversation where it appeared, and the cache survives.

## Failure modes

| Symptom | Cause |
|---|---|
| Throws when the tool set is assembled | Duplicate tool name, or a collision with `task` / `activate_skill` / `read_skill_resource`. |
| Throws at runtime on return | Bare non-string return with no `output` schema. |
| `harness.sandbox` throws | No `useSandbox()` on the agent. |
| Tool never called, or called wrongly | Vague description. |
| Interrupted call settles with an unknown-outcome error | Ordinary (non-durable) tool. Expected behavior. |
| `AbortError` mentioning work may still be running | The `run` ignored its `signal`. |
