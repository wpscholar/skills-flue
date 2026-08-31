# The Cloudflare target

Live source: `flue docs read guide/cloudflare-target`

## Generated Durable Objects

One DO class and Wrangler binding per exported agent function found by the `'use agent'` scan. The name derives from the agent's identity (function name, or `agentName` override); camel-case boundaries split in binding names.

```
export function SupportChat() {…}  →  class FlueSupportChatAgent
                                      env.FLUE_SUPPORT_CHAT_AGENT
```

Canonical conversation streams, immutable attachments, and accepted submissions store automatically in the DO's SQLite.

- **Renaming the agent function is a storage-identity change** unless `agentName` pins the identity. Renaming the *file* changes nothing.
- **A source-root `db.ts` is rejected at build time.**
- **Do not hand-author Flue's generated `FLUE_*` bindings.** Declare only migrations for generated classes, plus bindings for your own resources (DOs, R2, Queues, …).

## Durable agent execution

Direct HTTP prompts and `dispatch(...)` inputs enter the same per-conversation queue feeding the canonical stream.

- **Admission returns first.** Flue schedules a zero-delay wake after accepting input; the DO's **alarm** invocation claims the queued submission and awaits the complete response — all model turns and tool calls — before completing. Response start is typically tens of milliseconds on the next alarm tick.
- **Platform observability captures the whole response as one invocation.** `createCloudflareTracing()` is installed by default and adds agent-level spans.
- **Scheduled callbacks await running responses.** A DO runs one alarm at a time, so `schedule()`/`scheduleEvery()` callbacks due mid-response fire only after it settles: delivery is durable; timeliness is not guaranteed while the agent is busy. `queue()` is unaffected, and message steering is unaffected.
- **CPU limits.** Responses sit well below the default 30-second active-CPU limit; raise `limits.cpu_ms` for heavy computation.

## Local development

`vite dev` invokes the Worker in workerd via the Cloudflare plugin, with permissive CORS defaults applied by Vite middleware before workerd. Agent-set changes and `wrangler.jsonc` edits regenerate only affected files. The plugin registers agents but **mounts no routes** — `app.ts` remains the route map.

## Calling a private agent over a service binding

```ts
import { createFlueClient } from '@flue/sdk';

type Env = { AGENT_APP: Fetcher };

export default {
  async fetch(request: Request, env: Env) {
    const convo = createFlueClient({
      // The host is never dialed — only the pathname and query select a route.
      url: 'https://agent.internal/agents/support/ticket-42',
      fetch: (input, init) => env.AGENT_APP.fetch(new Request(input, init)),
    });

    const admission = await convo.send({ message: { kind: 'user', body: 'Summarize this ticket.' } });
    return Response.json(admission);
  },
};
```

All client operations work over the binding — `send`, `wait`, `abort`, `history`, `observe`, in both long-poll and SSE modes.

**Attachments gotcha:** `convo.attachmentUrl(...)` returns a URL on the placeholder host that the client never fetches automatically. Forward it through the same fetcher:

```ts
const response = await env.AGENT_APP.fetch(new Request(convo.attachmentUrl(attachmentId)));
```

## Workers AI and AI Gateway

With `'cloudflare'` available, no API key is needed — authorization and billing follow the Worker account, including the Workers AI free tier.

```ts
useModel('cloudflare/@cf/moonshotai/kimi-k2.6');
```
```jsonc
{ "ai": { "binding": "AI" } }
```

**AI Gateway is enabled by default** for all `cloudflare/...` models — caching, request logging, rate limiting, budget controls. Re-register the provider in `app.ts` to customize, target a named gateway, or opt out. See the `flue-agent-authoring` skill's `models-and-providers.md`.

## Sandboxes

- **Cloudflare Sandbox** — container-backed Linux (git, package installs, native binaries, real filesystems). Export the sandbox DO class from `cloudflare.ts`, declare its binding and container image in `wrangler.jsonc`, then wrap the RPC stub:

```ts
'use agent';
import { getSandbox } from '@cloudflare/sandbox';
import { env } from 'cloudflare:workers';
import { type AgentProps, useModel, useSandbox } from '@flue/runtime';
import { cloudflareSandbox } from '@flue/runtime/cloudflare';

export function Assistant({ id }: AgentProps) {
  useModel('anthropic/claude-sonnet-4-6');
  useSandbox(cloudflareSandbox(getSandbox(env.Sandbox, id)), { cwd: '/workspace' });
}
```

- **Cloudflare Computer** (`flue add sandbox cloudflare-computer`) — a durable, SQLite-backed `Workspace` inside the agent's own DO, with shell execution via a just-bash Dynamic Worker. Standard tools remain, but commands run in a **JavaScript shell**: coreutils and text tools work; **native binaries and package managers do not**.

Use Computer when a durable workspace and shell-expressible work are enough; Sandbox when you need full Linux.

## Extending the generated agent

```ts
'use agent';
import { useModel } from '@flue/runtime';
import { extend } from '@flue/runtime/cloudflare';

export function Assistant() {
  useModel('anthropic/claude-sonnet-4-6');
}

export const cloudflare = extend({
  base: (Base) =>
    class extends Base {
      async onStart() { await this.scheduleEvery(60, 'heartbeat'); }
      async heartbeat() { this.setState({ ...this.state, lastHeartbeatAt: Date.now() }); }
    },
});
```

```ts
function extend<TBase extends object = CloudflareAgentLike, TEnv = any>(
  extension: CloudflareExtension<TBase, TEnv>,
): CloudflareExtension<TBase, TEnv>;
```

- `base(Base)` receives the Agents SDK `Agent` base class, applied **before** Flue defines the final generated DO subclass. It must return the received class or a subclass.
- `wrap(Final)` receives the final generated class and **must return the same class or a prototype-preserving wrapper** — subclasses are rejected. Use it for instrumentation:

```ts
export const cloudflare = extend({
  wrap: (Final) => Sentry.instrumentDurableObjectWithSentry((env) => ({ dsn: env.SENTRY_DSN }), Final),
});
```

- Both callbacks type against `CloudflareAgentLike` — a structural view of the Agents SDK base covering `state`, `setState()`, `onStart()`, `schedule()`, `scheduleEvery()`, `queue()`. Typos fail at typecheck. Pass explicit `TBase`/`TEnv` for richer shapes.
- **Do not override Flue-owned `fetch()`, `onRequest()`, `onFiberRecovered()`, or `alarm()`.**
- **Do not add a Worker cron trigger solely to reach `scheduleEvery(...)`** — the Agents SDK scheduling APIs run inside the generated DO after creation. If the application must create the first conversation, expose an authenticated bootstrap route in `app.ts` or get the DO namespace from `env`.

## Extending `cloudflare.ts`

Named exports become top-level Worker exports:

```ts
import { DurableObject } from 'cloudflare:workers';

export class SalesforceAuthCache extends DurableObject {
  async refreshIfNeeded() { return await this.ctx.storage.get('token'); }
}

export default {
  async scheduled(_controller, env) {
    await env.SALESFORCE_AUTH_CACHE.getByName('default').refreshIfNeeded();
  },
};
```

Declare each class's binding and migration in `wrangler.jsonc`. **`cloudflare.ts` must not define a default `fetch` handler** — Flue keeps HTTP composition in `app.ts`. Use it for Worker-level events (inbound email, queues, cron). To deliver scheduled input to an agent from these handlers, import the agent function and call `dispatch(Agent, { id, message })`.

## Helper functions

```ts
import { getCloudflareContext, getDurableObjectIdentity } from '@flue/runtime/cloudflare';
```

- `getCloudflareContext(): CloudflareContext` — valid only inside a Worker or DO request handler; throws otherwise. Returns `env` (bindings) plus `storage` (the DO's `{ sql }` handle). For advanced integrations such as custom Cloudflare sandbox adapters.
- `getDurableObjectIdentity(): FlueDurableObjectIdentity` — valid only inside a generated DO request handler. Returns `bindingName`, `className`, `name` (the instance name passed to `idFromName`/`getAgentByName`), and `id`.

## Recovery on Cloudflare

Ownership is structural — every conversation is a Durable Object with its own SQLite, so there is no lease protocol. Recovery is wake-driven:

- **Wake on start.** After eviction, a code deploy, or a platform reset, Flue immediately flags any attempt that was running when the previous instance died and reconciles before serving new work. The platform's fiber-recovery callback triggers the same path.
- **A durable wake schedule.** While unsettled work exists, the object keeps a short self-renewing wake. Each wake runs a bounded supervision pass — reconcile, enforce deadlines, start work — and **re-arms its successor before doing anything that can fail**, so a hung attempt can delay supervision by at most one wake, never break it. Attempt execution runs detached from the wake that started it.
- Abort intents, attempt bookkeeping, and settlement records live in the object's storage, so an abort requested while the object was evicted is honored on the next wake.
