---
name: flue-evals-and-observability
description: >-
  Test and instrument Flue agents — eval suites with vitest and vitest-evals,
  LLM judges, the in-process runtime observe() event stream, token usage and
  cost accounting, Sentry / Braintrust / OpenTelemetry / Cloudflare tracing, and
  redacting sensitive content from traces. Use when writing evals for a Flue
  agent or wiring its logging, metrics, or tracing.
---

# Flue Evals and Observability

Current API: `flue docs read guide/evals` · `flue docs read guide/observability`

## Evals

An eval is an automated test that runs an agent against a **live model** and asserts on observable behavior: the reply, the tools it called, the data it emitted. Two properties separate evals from unit tests — they are **nondeterministic** (assert on behavioral contracts, never exact strings) and they **consume real tokens and time**, which warrants separate config, credentials, and timeouts.

```ts
// vitest.evals.config.ts
export default defineConfig({
  test: { include: ['src/evals/**/*.eval.ts'], testTimeout: 60_000 },
});
```

Vitest's 5-second default will not survive a live model turn.

**In-process** — the full runtime, hooks, durability and sandboxes behave exactly as in production:

```ts
const flue = await start({ agents: [ServiceStatus] });
afterAll(() => flue.stop());

it('checks live service status before answering', async () => {
  const toolsCalled: string[] = [];
  const agent = init(ServiceStatus);            // no id → a fresh conversation per case
  const receipt = await agent.dispatch('Is the checkout service currently operational?');
  const reply = await agent.read(receipt, {
    onEvent: (chunk) => { if (chunk.type === 'tool-input') toolsCalled.push(chunk.toolName); },
  });

  expect(reply.text).toContain('operational');
  expect(toolsCalled).toContain('get_service_status');
});
```

**Agents that depend on build-resolved imports like `SKILL.md` must be evaluated over HTTP instead**, against `vite dev` or a deployment — which also exercises routing and middleware. `flue add tooling vitest-evals` generates a harness for `vitest-evals`, with judges (`FactualityJudge`, `ToolCallJudge`, `StructuredOutputJudge`, or `createJudge(...)`) running on their own separately configured judge harness. **Prefer deterministic assertions first**; reach for a judge only when the contract is genuinely fuzzy.

**Run evals as a separate CI job** from unit tests — slower, token-consuming, and able to fail for reasons unrelated to the change. Details and examples: `references/evals.md`.

## Two event surfaces named `observe()`

This is the first thing to get straight:

| | Conversation stream | Runtime event stream |
|---|---|---|
| Import | `createFlueClient(...).observe()` from `@flue/sdk` | `observe()` from `@flue/runtime` |
| Scope | One conversation | Every agent in the process |
| Shape | Durable, render-ready messages and data parts | Raw activity: model requests, tool executions, logs, token counts |
| Transport | HTTP | In-process only |
| For | Product UI | **Telemetry, metering, error reporting** |

They share correlation identifiers — a conversation message's `submissionId` matches the runtime events its submission produced.

## Subscribing

```ts
// src/app.ts — module top level, once at startup
import { observe } from '@flue/runtime';

observe((event) => {
  if (event.type === 'submission_settled' && event.outcome === 'failed') {
    console.error(`[${event.agentName}] submission ${event.submissionId} failed:`, event.error?.message);
  }
});
```

Three subscriber rules: **stay cheap** (subscribers run synchronously on the emission path — branch on `event.type`, return early, queue async work); **treat events as read-only** (each delivery is a detached, frozen observation); **failures are contained** (a throwing subscriber is logged and skipped, and returned promises are observed for rejection but not awaited).

Scope traps:

- The subscription is **isolate-scoped and live-only** — no durable replay, no cross-process aggregation.
- **On Cloudflare each conversation runs in its own Durable Object isolate**, so a subscriber registered from `app.ts` runs in each isolate and observes only that isolate's activity.
- **`flue run` never loads `app.ts`** — register in the agent module when a subscriber must also run under the CLI.

## Reading the stream

Events carry `v`, `eventIndex`, `timestamp`, and correlation fields (`agentName`, `conversationId`, `instanceId`, `submissionId`, `operationId`, `turnId`, `taskId`). Families: agent lifecycle (`agent_start`/`agent_end`/`idle`), `submission_settled`, operations, turns, messages and deltas, `tool_start`/`tool`, `task_start`/`task`, compaction, and `log`.

Three traps:

- **Alert on `submission_settled` outcomes, not on nested `isError`.** An agent can recover from a failed turn or tool call — nested errors are diagnostic context.
- **Streaming deltas are live progress, not authoritative state.** The assistant `message_end` event carries the completed message.
- **Don't sum usage across levels.** `turn` events are the leaves; `operation` and `compaction` roll-ups already include them. Durations overlap the same way.

**The model never sees log lines** written with a tool's or hook's `log` — they are runtime events, not conversation content, and reach only in-process subscribers. Forward them yourself.

Usage fields (`input`, `output`, `cacheRead`, `cacheWrite`, `totalTokens`, `cost`), provider diagnostics, and worked metering code: `references/observability.md`.

## Providers

No bundled dashboard. `flue add tooling sentry` (terminal failures as issues, logs, optional AI traces with content off by default) · `flue add tooling braintrust` (LLM tracing with model, tool, task and compaction spans) · `@flue/opentelemetry` (`createOpenTelemetryInstrumentation()` registered with `instrument(...)`). They compose — an error reporter and a tracer can subscribe side by side.

On Cloudflare, agent work is visible to Workers Logs and Workers Traces with no Flue-side wiring, and `createCloudflareTracing()` is installed by default. Install it yourself **only to customize** — at `app.ts` module scope, since a post-startup install throws.

## Sensitive content — decide this deliberately

Runtime events can contain prompts, system instructions, reasoning, tool arguments and results, and error details. **Both trace adapters capture conversation content by default.**

> For every exporter that leaves Cloudflare, installing an instrumentation with `instrument(...)` **is** the consent: nothing is emitted by merely deploying, but once that line exists, prompts and tool payloads flow to the receiving backend.

Every adapter takes the same two controls: `content: false` (capture off entirely) and `content: { transform }` (redact, drop by `scope.contentType`, or tighten the byte budget). Two protections hold unconditionally: `turn_request` events never leave the process, and image content blocks never carry raw bytes.

Review the retention and access controls of whatever receives your traces.

## Anti-patterns

- Mixing up the two `observe()` functions.
- Doing async work inside a subscriber instead of queueing it.
- Alerting on nested `isError` and paging on recoverable turn failures.
- Summing `turn` and `operation` usage together.
- Asserting on exact model wording in an eval.
- Running evals in the same CI job as unit tests.
- Installing a trace exporter without deciding the content policy first.
