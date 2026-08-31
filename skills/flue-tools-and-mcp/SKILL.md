---
name: flue-tools-and-mcp
description: >-
  Give a Flue agent the ability to act — defineTool and useTool with valibot
  schemas, the run result envelope, harness tools, durable tools with step.do,
  conditional tools, and remote MCP servers via useMcpConnection. Use when
  adding, reviewing, or debugging tools, tool authorization, or MCP connections
  in a Flue agent.
---

# Flue Tools and MCP

A tool is a function you write and describe to the model. The model decides **when** to call; your code decides **what happens**.

Current API: `flue docs read guide/tools` · `flue docs read guide/mcp`

## Shape

```ts
import { defineTool } from '@flue/runtime';
import * as v from 'valibot';

export const lookupOrder = defineTool({
  name: 'lookup_order',
  description: 'Look up one order by id and return its current status.',
  input: v.object({ orderId: v.string() }),
  async run({ data }) {
    const order = await orders.get(data.orderId);
    return { output: { status: order.status, eta: order.eta } };
  },
});
```

Mount it with `useTool(lookupOrder)`, or pass an inline definition object. `defineTool` validates at module load and returns the definition frozen — the right shape for a shared `src/tools/` directory.

## Contracts that bite

| Rule | Consequence |
|---|---|
| `input` must be a **top-level valibot object schema** | Anything else throws. |
| `run` returns an **envelope** `{ output?, terminate? }` | A bare string is sugar. Returning nothing is fine only when no `output` schema is declared. **Any other bare value throws at runtime.** |
| The description is the model's only documentation | Vague descriptions are the single most common cause of a tool being called wrongly or not at all. |
| Names must be unique, and some are reserved | `task`, `activate_skill`, `read_skill_resource` are framework-owned. Collisions throw when the tool set is assembled. |
| Throw, don't swallow | A throw inside `run` becomes an error result the model sees and can react to. It does not crash the agent. |
| `terminate: true` | Ends the agent's turn once the current tool batch settles. |

`run` also receives `signal` (an `AbortSignal` — a `run` that ignores it is *abandoned*, not awaited, so it cannot wedge the agent), `log` (streamed as conversation events the **model never sees**), and `toolCallId`.

## Built-in tools

A sandbox adds `read`, `write`, `edit`, `bash`, `grep`, `glob`; without `useSandbox()` there are none, and a sandbox adapter's `tools` function **replaces** that set — check an integration's docs before assuming `bash` exists. See the `flue-skills-subagents-sandboxes` skill.

## Harness tools (`harness: true`)

Reach the runtime from inside a tool. `run` then receives `harness`:

```ts
harness: true,
async run({ harness, data }) {
  await harness.sandbox.writeFile('contract.md', data.contract);
  const { data: report } = await harness.prompt('Review contract.md and assess the risk.', { result: Report });
  return { output: report };
}
```

- `harness.sandbox` — live file and exec verbs with **no conversation record**. **Throws when the agent declared no sandbox.**
- `harness.prompt(text, options?)` — a model operation in the harness's own scratch conversation, which repeated calls continue. `options.result` (a valibot schema) requires validated structured data; `options.tools` adds tools for just that operation.

Harness invocations are scoped to the tool call and **count against the four-level delegation cap**. Tools without the flag cannot reach the runtime at all.

## Durable tools (`durable: true`)

**Interrupted ordinary tool calls are never re-executed** — they settle with an explicit unknown-outcome error, because the runtime cannot know which side effects already happened. For work that must complete, opt in:

```ts
export const provisionWorkspace = defineTool({
  name: 'provision_workspace',
  description: 'Create the customer tenant, then seed each default project.',
  input: v.object({ customerId: v.string() }),
  durable: true,
  async run({ data, step }) {
    const tenant = await step.do('create-tenant', () => billing.createTenant(data.customerId));
    for (const project of DEFAULT_PROJECTS) {
      await step.do(`seed:${project.name}`, () => projects.seed(tenant.id, project));
    }
    return { output: { tenantId: tenant.id } };
  },
});
```

Four rules: **everything effectful goes in a step** (code between steps re-executes on recovery); **names must be deterministic** and unique within the call; **step values are JSON and should stay small** (store artifacts in the sandbox, record a pointer); and steps are **exactly-once-recorded, at-least-once-executed**, so each one around an external effect must be individually idempotent.

Flags compose: with `durable: true, harness: true` you get both, and **`harness.prompt(...)` belongs inside a step** so recovery does not re-prompt. A thrown error is not an interruption — nothing retries automatically. And **a redeploy can withdraw the contract**: if recovery finds the current render no longer declares the tool, or no longer marks it `durable`, the call falls back to the ordinary interrupted-marker path.

Full signatures, the `run` context table, and a failure-mode lookup: `references/tool-api.md`.

## Conditional tools

The tool set is program logic — declare it from scratch each render:

```ts
if (approved) useTool(publishRelease);
```

Changes announce to the model as a `resources` signal at the next turn boundary. **Changing the set rewrites the provider's tools array and invalidates its prompt cache** — gate on state that changes rarely. The safe exception is a tool unlocked by a completed tool call: current Anthropic models (except Haiku) load the definition at the point it appeared and the cache survives.

## Authorization

**A tool's arguments are model-selected inputs, not an authorization boundary.** Carry the authorized identifier in the delivered signal's `attributes`, read it with `useDelivery()`, and close over it in trusted code:

```ts
const delivery = useDelivery();
const customerId = delivery.kind === 'signal' ? delivery.attributes?.customerId : undefined;

useTool({
  name: 'lookup_customer_order',
  description: 'Look up one order belonging to this customer.',
  input: v.object({ orderId: v.string() }),   // note: no customerId here
  async run({ data }) {
    return customerId ? await orders.getStatus(customerId, data.orderId) : 'No accessible order was found.';
  },
});
```

Your route must still verify the caller. Avoid generic provider tools that expose arbitrary destinations or API methods.

## MCP

```ts
useMcpConnection({
  name: 'linear',
  url: 'https://mcp.linear.app/mcp',
  auth: process.env.LINEAR_API_KEY,        // string, or a function resolved per request
  tools: ['create_issue', 'search_issues'], // optional allowlist
  optional: true,                           // run without it rather than failing the submission
});
```

Tools mount as `mcp__<server>__<tool>`. Flue connects when a submission initializes and reuses the connection for the instance's lifetime. An unreachable server **fails the run** unless `optional: true`, and an allowlist naming a tool the server doesn't expose **fails the connection**.

**Flue never stores or manages tokens** — OAuth flow, storage, and refresh are yours; the function form of `auth` resolves per request, so rotation works. And **an MCP server can influence your agent**: its tool descriptions enter the prompt and its results enter the conversation. Treat a server you don't control as an untrusted dependency and use the allowlist to limit the surface.

`defineMcpConnection`, mid-conversation authorization, and the lower-level `createMcpConnection`: `references/mcp.md`.

## Anti-patterns

- Returning a bare object from `run` (throws) instead of `{ output: … }`.
- Trusting `data.customerId` from tool input as an authorization check.
- Reaching for `durable: true` everywhere — it is for work that must complete, and it changes recovery semantics.
- Calling `harness.prompt(...)` outside a `step.do` in a durable tool.
- Flipping tools on state that changes every turn.
- Mounting a whole third-party MCP server when three tools would do.
