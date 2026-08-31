---
name: flue-serving-and-clients
description: >-
  Expose Flue agents and talk to them — the app.ts route map, createAgentRouter,
  authenticating and CORS-configuring conversation URLs, dispatch/init/start,
  webhook channels, cron schedules, the db.ts persistence adapter, and the
  @flue/sdk and @flue/react clients. Use when wiring routes, webhooks, cron, a
  database, or a frontend to a Flue agent.
---

# Serving Flue Agents and Talking to Them

**Flue never mounts an agent automatically.** The `'use agent'` scan makes an agent *addressable*; serving it over HTTP is a separate, explicit decision in `app.ts`.

Current API: `flue docs read guide/routing` · `guide/channels` · `guide/schedules` · `guide/database` · `sdk/overview`

## `app.ts` is the route map

```ts
import { createAgentRouter } from '@flue/runtime/routing';
import { Hono } from 'hono';
import { Support } from './agents/support.ts';
import { channel as slack } from './channels/slack.ts';

const app = new Hono();

app.route('/agents/support', createAgentRouter(Support));
app.route('/channels/slack', slack.route());

export default app;
```

Hono is convention, not requirement — the default export needs only a fetch-compatible shape (`Fetchable`). `createAgentRouter` is a **pure factory with no options**: model, durability, and initial-data schema live on the agent module, not the mount. The mount path is **not** the agent's identity, so you can move a mount with no data migration.

Conversation surface, relative to the mount:

| Route | Purpose |
|---|---|
| `POST /:id` | Deliver one message. **202 at admission**, before the agent runs. |
| `GET \| HEAD /:id` | Read the conversation (snapshot, `?view=updates`, or SSE). |
| `POST /:id/abort` | Abort in-flight and queued work for the conversation. |
| `GET /:id/attachments/:attachmentId` | Download one attachment. |

There is no "wait for the reply" mode on `POST` — follow the returned `streamUrl` or use the SDK's `wait()`.

## Security — do this before shipping

**A mounted agent has no built-in authentication. Anyone who can reach a conversation URL can send to it, read its full history, and abort its work.** There is no per-agent middleware export. Conversation ids are caller-chosen path segments, so without an ownership check any authenticated user can read another user's conversation by guessing an id.

Two checks, in your own middleware:

```ts
app.use('/agents/support/*', async (c, next) => {
  const user = await verifySession(c.req.raw);
  if (!user) return c.json({ error: 'unauthorized' }, 401);

  const [conversationId] = c.req.path.slice('/agents/support/'.length).split('/');
  if (!(await canAccessTicket(user, conversationId))) return c.json({ error: 'forbidden' }, 403);
  return next();
});
app.route('/agents/support', createAgentRouter(Support));
```

The `/*` pattern covers prompts, reads, aborts, and attachment downloads alike. Better still: **server-issued ids** derived from the authenticated principal (`user-${user.id}`), which reduces the ownership check to equality. Or keep the agent **dispatch-only** — registered but never mounted, reachable only from a verified webhook or a service binding.

**CORS:** the router sets no `Access-Control-*` headers, but `vite dev` and `vite preview` apply permissive localhost defaults — so a cross-origin setup that works locally can fail after deployment. Configure it yourself, and expose the stream coordination headers or the SDK cannot resume streams:

```ts
app.use('/agents/*', cors({
  origin: 'https://app.example.com',
  credentials: true,
  exposeHeaders: ['Stream-Next-Offset', 'Stream-Up-To-Date', 'Location'],
}));
```

## Reaching an agent from server code

```ts
const receipt = await dispatch(SupportAssistant, {
  id: event.ticketId,
  message: { kind: 'signal', type: 'support.comment.created', body: event.text, attributes: { commentId } },
});
```

- `dispatch()` — fire-and-forget; resolves at durable admission with a `DispatchReceipt`. Needs no mount.
- `init(Agent, { id })` — a handle: `dispatch()`, `read(receipt)`, `abort()`. `read()` awaits settlement and rejects with `AgentRunError` on failure or abort.
- `start()` from `@flue/runtime/node` — boots the runtime inside your own script, with no server and no `app.ts`. **`start()` scripts don't pick up `db.ts`** — pass `db:` directly.
- `uid` send conditions: omit → continue-or-create; a prior receipt's uid → continue only that incarnation; `null` → create only (409 `AgentInstanceExistsError`).

**`read()`/`wait()` promises are not durable.** If the awaiting process exits, the work continues but the local promise is gone — persist the `DispatchReceipt` and re-attach later. Side effects that must not be lost belong *inside* the agent as a tool call, not after the await.

## Channels, schedules, database

Summaries here; full detail in `references/`.

- **Channels** (`flue add channel slack`) are **inbound-only** verified webhook ingress. Outbound calls use the provider's own SDK, wrapped as tools with the destination bound in trusted code. **Channel packages do not deduplicate** — always pass the provider's redelivery-stable id as `idempotencyKey`. Never put short-lived provider tokens (`response_url`, interaction tokens) into signal messages: signals land in model context and durable history. → `references/channels.md`
- **Schedules**: Flue has no scheduler. Node uses an in-process cron in `app.ts` (fires in **every replica** and under `vite dev` — gate it); Cloudflare uses Cron Triggers, which run on **UTC** with no timezone option, through a Worker's single `scheduled` handler. → `references/schedules.md`
- **Database**: `src/db.ts` default-exports a `PersistenceAdapter`. **Node target only — a `db.ts` on Cloudflare is rejected at build time.** Without one, state is in-memory and lost on restart. A shared external database does **not** give you active-active: exactly one live Node owner per conversation. → `references/database.md`

## Clients

```ts
import { createFlueClient } from '@flue/sdk';

const conversation = createFlueClient({ url: 'https://example.com/agents/support/ticket-8472', token });
const admission = await conversation.send({ message: { kind: 'user', body: 'Summarize my case.' } });
const reply = await conversation.read(admission);
```

**One client addresses exactly one conversation** — the URL is the whole contract. There is no deployment-wide client, no agent-name addressing, no way to enumerate conversations. A new conversation is a new client with a fresh id appended to the mount URL. Construction is synchronous and makes no requests.

React:

```tsx
const agent = useFlueAgent({ url: `/api/agents/support-assistant/${conversationId}` });
```

`sendMessage()` resolves when the server **accepts** the prompt, not when generation completes. **Memoize any client you pass as `useFlueAgent({ client })`** — a new instance replaces the session on every render. → `references/sdk-and-react.md`

## Anti-patterns

- Mounting an agent without authentication middleware, or without a per-conversation-id ownership check.
- Letting clients choose conversation ids in a multi-tenant app.
- Testing CORS only under `vite dev`.
- Awaiting a settled reply inside a webhook or channel handler — providers retry slow acknowledgements. Admit and return.
- Reusing an `idempotencyKey` with a different payload (409 `submission_conflict`).
- Adding `db.ts` to a Cloudflare project, or expecting a `start()` script to read it.
