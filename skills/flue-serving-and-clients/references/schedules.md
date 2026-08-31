# Schedules in Flue

Live source: `flue docs read guide/schedules`

> Flue has no scheduler of its own — each target pairs its cron mechanism with the same dispatch surface every other delivery uses.

Three parts: a **trigger**, a **delivery** (`dispatch(agent, { id, message })`, which resolves at durable admission and does not wait for the model), and a **conversation** named by the `id`. The agent itself is ordinary — nothing marks it as scheduled.

## Node.js — `croner` in `app.ts`

```ts
import { dispatch } from '@flue/runtime';
import { Cron } from 'croner';
import { Reporter } from './agents/reporter.ts';

new Cron(
  '0 9 * * *',
  {
    timezone: 'America/New_York',
    protect: true,   // skip a fire while the previous callback is still running
    catch: (error) => console.error('Scheduled dispatch failed', error),
  },
  async () => {
    await dispatch(Reporter, {
      id: 'daily-summary',
      message: {
        kind: 'signal',
        type: 'schedule',
        body: 'Review recent activity and prepare the daily summary.',
        attributes: { scheduledAt: new Date().toISOString() },
      },
    });
  },
);
```

**Two gotchas:**
- The `Cron` is created at module load, so the schedule starts with the server — **including under `vite dev`**. Gate construction on an environment variable when development fires are unwanted.
- An in-process schedule runs in **every replica** of the server. Past one instance, gate the trigger to a single replica or move it to a platform scheduler.

## Cloudflare — Cron Triggers

```jsonc
// wrangler.jsonc
{ "triggers": { "crons": ["0 9 * * *"] } }
```

```ts
// src/cloudflare.ts
export default {
  async scheduled(controller) {
    await dispatch(Reporter, {
      id: 'daily-summary',
      message: {
        kind: 'signal',
        type: 'schedule',
        body: 'Review recent activity and prepare the daily summary.',
        attributes: {
          cron: controller.cron,
          scheduledAt: new Date(controller.scheduledTime).toISOString(),
        },
      },
    });
  },
};
```

- **Cloudflare evaluates cron expressions in UTC. There is no timezone option.**
- A Cloudflare Worker has a single `scheduled` handler, so when `crons` lists several patterns, use `controller.cron` to identify which one fired.
- `dispatch(...)` here needs no mount and bypasses HTTP middleware, admitting durably to the agent's Durable Object before resolving.
- **Nothing durable exists until `dispatch(...)` resolves**, so a handler that throws before admission delivers nothing for that fire. Keep the handler thin: dispatch and return.

**Alternative for per-conversation timers:** the Agents SDK `schedule()` / `scheduleEvery()` APIs, reachable through the per-module `extend()` extension point. Those callbacks share the conversation's Durable Object with agent execution, so one that comes due while a response is running fires only after it settles — delivery is durable, timeliness is not. A Cron Trigger is right when the schedule must address or create conversations from outside.

## What a fire delivers

Deliver as `kind: 'signal'` — a caller-defined `type`, the instruction in `body`, flat string metadata in `attributes`. It renders into model context as an XML-tagged block:

```
<signal type="schedule" scheduledAt="2026-07-17T13:00:00.000Z">
Review recent activity and prepare the daily summary.
</signal>
```

Code reads the same delivery via `useDelivery()`.

## Choosing the conversation id

| Id choice | Behavior |
|---|---|
| Fixed (`'daily-summary'`) | Every fire continues one conversation. The agent sees its previous runs and keeps persistent state across them. |
| Per fire (`` `daily-${isoDate}` ``) | Each fire creates a fresh conversation with bounded context. Pair with `initialData` to seed it. |

## Awaiting a scheduled run

```ts
const reporter = init(Reporter, { id: `daily-${isoDate}` });
const receipt = await reporter.dispatch('Review recent activity and prepare the daily summary.');
const reply = await reporter.read(receipt);
await postSummary(reply.text);
```

**The `read()` promise is not durable.** If the process dies mid-await, the run still settles, but anything after the `await` is gone — so side effects that must not be lost belong inside the agent as a tool call, not after the read.

## External schedulers (Fly, Render, Railway)

```http
POST /agents/reporter/daily-summary HTTP/1.1
Content-Type: application/json
Authorization: Bearer <scheduler-token>

{ "kind": "signal", "type": "schedule", "body": "Review recent activity and prepare the daily summary." }
```

`202` at admission. **This path requires the agent to be mounted, and a mounted agent has no built-in authentication** — put the mount behind credential-verifying middleware.

## One-shot runs from CI

```bash
flue run src/agents/reporter.ts \
  --message "Review recent activity and prepare the daily summary." \
  --id "daily-$(date +%F)"
```

`flue run` delivers one `kind: 'user'` message — **there is no signal form** from the CLI. Add `--new` with a deterministic `--id` to make conversation creation exactly-once across job retries.

## Operational behavior

- **Missed fires.** An in-process Node scheduler fires only while running; fires during downtime or a deploy are skipped, and cron libraries do not replay them on restart. Use a platform scheduler, or track the last completed run and catch up at startup. On Cloudflare the platform invokes `scheduled` on cadence with no traffic required.
- **Overlap.** Deliveries to one conversation never run concurrently — inputs are processed in accepted order, and a message arriving mid-response joins at a turn boundary. Overlapping fires against a fixed id queue or coalesce; they cannot double-run the agent. `protect: true` matters mainly when the callback awaits a settled reply rather than a fast admission. Per-fire ids are independent conversations and *do* run concurrently.
- **Durability.** With Node's in-memory default, admitted work lasts only as long as the process. On Cloudflare, admission is durable in the Durable Object and interrupted processing is reconciled — which makes delivery **at-least-once**. Design a scheduled agent's external side effects to be idempotent.
