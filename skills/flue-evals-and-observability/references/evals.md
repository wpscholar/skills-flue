# Evals in Flue

Live source: `flue docs read guide/evals` · `flue docs read ecosystem/tooling/vitest-evals`

## Setup

```ts
// vitest.evals.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['src/evals/**/*.eval.ts'],
    testTimeout: 60_000,
  },
});
```

```json
// package.json
{ "scripts": { "evals": "vitest run --config vitest.evals.config.ts" } }
```

## In-process

```ts
// src/evals/service-health.eval.ts
import { init } from '@flue/runtime';
import { start } from '@flue/runtime/node';
import { afterAll, expect, it } from 'vitest';
import { ServiceStatus } from '../agents/service-status.ts';

const flue = await start({ agents: [ServiceStatus] });
afterAll(() => flue.stop());

it('checks live service status before answering', async () => {
  const toolsCalled: string[] = [];

  // No id: init() mints a fresh conversation for this case.
  const agent = init(ServiceStatus);
  const receipt = await agent.dispatch('Is the checkout service currently operational?');
  const reply = await agent.read(receipt, {
    onEvent: (chunk) => {
      if (chunk.type === 'tool-input') toolsCalled.push(chunk.toolName);
    },
  });

  expect(reply.text).toContain('operational');
  expect(toolsCalled).toContain('get_service_status');
});
```

Fresh conversations per case prevent cross-contamination. The full runtime — hooks, durability, sandboxes — behaves identically to production.

**Gotcha: agents depending on build-resolved imports like `SKILL.md` must be evaluated over HTTP instead**, because a `start()` script does not go through the Vite build.

## Over HTTP

```ts
import { createFlueClient } from '@flue/sdk';
import { expect, it } from 'vitest';

const mountUrl = process.env.FLUE_AGENT_URL ?? 'http://127.0.0.1:5173/agents/service-status';

it('checks live service status before answering', async () => {
  const conversation = createFlueClient({ url: `${mountUrl}/eval-${crypto.randomUUID()}` });

  const admission = await conversation.send({
    message: { kind: 'user', body: 'Is the checkout service currently operational?' },
  });
  await conversation.wait(admission);

  const { messages } = await conversation.history();
  const reply = messages.findLast((message) => message.role === 'assistant');
  const text = reply?.parts.filter((p) => p.type === 'text').map((p) => p.text).join('') ?? '';

  expect(text).toContain('operational');
});
```

This exercises routing and middleware alongside agent behavior.

## vitest-evals

```bash
flue add tooling vitest-evals --print | claude
```

The blueprint generates `src/evals/harness.ts` — a harness that drives one conversation per case through `@flue/sdk`, converting replies, tool calls, and usage metrics into normalized results.

```ts
import { expect } from 'vitest';
import { describeEval, toolCalls } from 'vitest-evals';
import { createFlueAgentHarness } from './harness.ts';

const harness = createFlueAgentHarness({
  agentUrl: process.env.FLUE_AGENT_URL ?? 'http://127.0.0.1:5173/agents/service-status',
});

describeEval('service status agent', { harness }, (it) => {
  it('checks live service status before answering', async ({ run }) => {
    const result = await run('Is the checkout service currently operational?');

    expect(result.output).toContain('operational');
    expect(toolCalls(result).map((call) => call.name)).toContain('get_service_status');
  });
});
```

## Judges

Judges are scorers that grade results and fail tests below a threshold. LLM-backed judges run on a separate **judge harness** — the judge's own model connection, configured separately from the agent under evaluation.

```ts
describeEval('service status agent', { harness, judgeHarness }, (it) => {
  it('reports status consistent with the reference answer', async ({ run }) => {
    const result = await run('Is the checkout service currently operational?');

    await expect(result).toSatisfyJudge(FactualityJudge(), {
      expected: 'The checkout service is currently operational.',
      threshold: 0.6,
    });
  });
});
```

Built-in: `FactualityJudge`, `ToolCallJudge`, `StructuredOutputJudge`. Custom judges via `createJudge(...)`.

**Prefer deterministic assertions first.** A judge adds a second nondeterministic system to every failure investigation.

## Running

```bash
pnpm run evals
FLUE_AGENT_URL=https://preview.example.com/agents/service-status pnpm run evals
vitest-evals serve vitest-results.json
```

In-process suites need provider credentials in the environment. HTTP suites additionally need a reachable target — start the dev server or point at a deployment.

**Keep evals as a separate CI job** from unit tests: slower, token-consuming, and capable of failing for reasons unrelated to the code change. The `vitest-evals` blueprint adds an `evals:json` script writing `vitest-results.json`, publishable via the `getsentry/vitest-evals` GitHub Action.

## What to assert on

| Assert on | Not on |
|---|---|
| Tool names called, and their order where it matters | Exact reply wording |
| Structured data emitted via `useDataWriter` (`reply.data`) | Token counts as a pass/fail gate |
| Response metadata stamped by `useResponseFinish` | Reasoning text |
| Settlement outcome (`AgentRunError` vs a reply) | Turn counts |
| Substrings that any correct answer must contain | Formatting or punctuation |
