# Channels in Flue

Live source: `flue docs read guide/channels` · catalog: `flue docs read ecosystem`

> A channel connects an external provider — Slack, GitHub, Stripe — to your agents: verified HTTP ingress that authenticates each incoming delivery and hands your code the provider's native payload. Channels are **inbound-only**; outbound provider calls stay in your application, written against the provider's own SDK.

## Adding one

```bash
flue add channel slack --print | claude
```

For Slack this installs `@flue/slack` (ingress: verification + routes) and `@slack/web-api` (outbound), creates `src/channels/slack.ts` exporting the configured `channel` and `client`, mounts it in `app.ts`, and binds a reply tool into the target agent. Env: `SLACK_SIGNING_SECRET` (inbound), `SLACK_BOT_TOKEN` (outbound).

Catalog (17): Slack, Discord, Microsoft Teams, Google Chat, Telegram, WhatsApp, Facebook Messenger, Twilio, GitHub, Linear, Notion, Intercom, Zendesk, Stripe, Shopify, Resend, Salesforce Marketing Cloud.

## The channel module

The package verifies each request — signatures checked against the exact raw bytes, replay windows enforced, protocol handshakes such as Slack's URL verification answered internally — and calls your handler only for authenticated deliveries.

```ts
import { dispatch } from '@flue/runtime';
import { createSlackChannel } from '@flue/slack';
import { Assistant } from '../agents/assistant.ts';

export const channel = createSlackChannel({
  signingSecret: process.env.SLACK_SIGNING_SECRET!,

  // Served at POST /channels/slack/events (with the mount below).
  async events({ payload }) {
    if (payload.type !== 'event_callback') return;
    if (payload.event.type !== 'app_mention') return;

    const event = payload.event;
    const thread = {
      teamId: payload.team_id,
      channelId: event.channel,
      threadTs: event.thread_ts ?? event.ts,
    };

    await dispatch(Assistant, {
      id: channel.instanceId(thread),
      // Slack redelivers events whose acknowledgement was slow or lost; the
      // event id names the delivery, so a retry never runs a second turn.
      idempotencyKey: payload.event_id,
      initialData: { channelId: thread.channelId, threadTs: thread.threadTs, startedBy: event.user },
      message: {
        kind: 'signal',
        type: 'slack.app_mention',
        body: event.text,
        attributes: { eventId: payload.event_id },
      },
    });
  },
});
```

## Package conventions

- **Handlers select routes.** Each configured handler publishes its route (`events` → `/events`, `interactions` → `/interactions`); omit a handler and its route does not exist. Most providers expose a single `webhook` handler at `/webhook`.
- **Return values become responses.** Nothing → empty `200`; a JSON-compatible value → JSON; a `Response` passes through unchanged (Slack slash commands, Discord interactions).
- **Acknowledge quickly.** Providers retry slow acknowledgements — admit the work and return rather than awaiting agent output.
- **Deliveries can repeat.** Providers retry and may deliver an event more than once; **channel packages are stateless and do not deduplicate.** Pass the provider's redelivery-stable id as `idempotencyKey`. A redelivered event converges on the original submission — same receipt, marked `deduplicated: true`, at most one answer. **The key names the delivery, not the outcome: reusing it with a different payload rejects with a 409 `submission_conflict`.** Carrying the id in signal `attributes` too keeps it visible for tracing.

## Mounting

```ts
app.route('/channels/slack', channel.route());
// Slack's Events API endpoint is now POST /channels/slack/events
```

The URL registered with the provider is the mount plus the suffix. The dispatch-target agent needs no mount of its own.

> Unlike agent mounts, channel routes need no additional authentication middleware for the provider traffic itself — verification against the provider's secret **is** the authentication, and it happens inside the channel before your handler runs.

## Conversation ids

Conversation-shaped providers (Slack thread, GitHub issue, Teams chat) expose `instanceId()`:

```ts
channel.instanceId({ teamId, channelId, threadTs }); // "slack:v1:T0123:C0456:1721760000.123456"
```

**The id identifies the conversation; it does not authorize access to it.** `parseInstanceId(id)` recovers destination fields but is an escape hatch — prefer passing structured facts through `initialData`.

Event-feed providers (Stripe, Shopify, Notion, Resend) have **no `instanceId()` helper** — choose the id yourself per customer, order, or occurrence.

## Signals, not user messages

Channel deliveries dispatch as `kind: 'signal'`. A Slack thread or GitHub issue is a multi-participant surface the agent joins as one member — a `user` message would present every participant as the agent's own user, where a signal carries the event with its metadata intact.

- `type` — a namespaced event name you choose (`'slack.app_mention'`, `'github.issue_comment.created'`)
- `body` — plain string
- `attributes` — a string-to-string map: sender, delivery id, resource identifiers

**Security: keep short-lived provider capabilities — interaction tokens, `response_url` values — out of the dispatched message.** Signals enter model context and durable history; those values belong only in immediate request handling.

## Creation data

`initialData` is recorded once, when the dispatch creates the conversation, and ignored by every later send. When the agent declares an `initialData` schema static, the value is validated at admission, so a creating dispatch that omits or malforms it fails instead of seeding a broken conversation.

```ts
'use agent';
export function Assistant() {
  useModel('anthropic/claude-sonnet-4-6');

  const data = useInitialData<v.InferOutput<typeof Assistant.initialData>>();
  useTool(replyInThread(data));

  return 'You participate in one Slack thread. Reply with the reply_in_slack_thread tool when a response is called for.';
}

Assistant.initialData = v.object({
  channelId: v.string(),
  threadTs: v.string(),
  startedBy: v.optional(v.string()),
});
```

With the schema static required, a conversation cannot exist without valid creation data, and no `undefined` narrowing is needed.

## Outbound

> Flue has **no outbound messaging API, no reply routing, and no send-message abstraction** over providers.

OAuth installation flows, token storage, and rotation are application concerns. To let the model act, wrap exactly the needed actions as tools with the destination bound in trusted code:

```ts
export function replyInThread(ref: { channelId: string; threadTs: string }) {
  return defineTool({
    name: 'reply_in_slack_thread',
    description: 'Reply in the Slack thread bound to this conversation.',
    input: v.object({ text: v.pipe(v.string(), v.minLength(1)) }),
    async run({ data }) {
      const result = await client.chat.postMessage({
        channel: ref.channelId,
        thread_ts: ref.threadTs,
        text: data.text,
      });
      return { output: { ts: result.ts ?? null } };
    },
  });
}
```

The model selects the reply text; it cannot select the workspace, the thread, the credential, or the API method. **Avoid generic provider tools that expose arbitrary destinations or API methods** unless the application has an explicit authorization design for them.

## Providers without a blueprint

```bash
flue add channel https://developers.provider.example/webhooks --print | claude
```

Or hand-write one with `createChannelRouter(routes)` from `@flue/runtime`:

```ts
export const channel = {
  routes: [{ method: 'POST', path: '/webhook', handler: webhook }],
};
```
```ts
app.route('/channels/acme', createChannelRouter(acme.routes));
```

Rules: verify signatures against the exact **unconsumed** request body; keep every route suffix a non-empty path beginning with `/`; test valid signatures, invalid signatures, and the provider's protocol handshakes. Channels model verified HTTP delivery — long-lived sockets, polling loops, and provider-managed background transports stay in application-owned infrastructure.
