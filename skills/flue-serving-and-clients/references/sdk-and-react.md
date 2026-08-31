# @flue/sdk and @flue/react

Live source: `flue docs read sdk/overview` · `sdk/create-flue-client` · `sdk/flue-client` · `guide/react`

## The model

`@flue/sdk` is the TypeScript client for **one agent conversation** of a deployed Flue application. The framework does not know where an application mounts its agents — `app.ts` does — so a client addresses one conversation by URL: the mount URL plus a caller-chosen conversation id.

- One conversation per client. No deployment-wide client, no agent-name addressing, no API to enumerate or delete conversations.
- A new conversation = a new client with a fresh id appended to the mount URL. The conversation is created by the first admitted send.
- Auth via bearer token or arbitrary headers on every request.
- **ESM-only**; runs anywhere `fetch` exists. Single dependency: `@durable-streams/client`.

Inside a Flue server, use `init()` and `dispatch()` from `@flue/runtime` instead — the SDK serves code outside the process.

## `createFlueClient()`

```ts
function createFlueClient(options: CreateFlueClientOptions): FlueClient;

interface HttpClientOptions {
  url: string;
  fetch?: typeof fetch;
  headers?: RequestHeaders;
  token?: string;
}
```

Construction is **synchronous and makes no network requests** — nothing verifies the URL reaches a mounted agent or that the conversation exists. The first request does, rejecting with `FlueApiError` on a non-2xx response.

| Field | Notes |
|---|---|
| `url` | Mount URL + conversation id. Trailing slashes stripped. In a browser a relative URL resolves against `location.origin`; **outside a browser a relative URL throws `TypeError`** — the one construction-time failure. |
| `fetch` | Used for **every** request, including the stream reads behind `wait()` and `observe()`. Defaults to global `fetch` bound to `globalThis`. A caller-supplied function is used as-is — bind it yourself if it is a method. |
| `headers` | Merged into every request, **after** the `token`-derived header, so a `headers` entry named `authorization` wins. |
| `token` | Sent as `authorization: Bearer <token>`. |

`RequestHeaders` may be a function (sync or async), re-evaluated once per JSON request and once per stream connection and reconnection — so an async factory can refresh a short-lived token. Headers apply only to requests the client itself makes: **`attachmentUrl()` returns a plain URL string carrying none of them.**

There is deliberately no retry or timeout configuration. Each JSON request is a single fetch cancelled per call via `AbortSignal`; stream reads take `backoffOptions` per call on `wait()` and `observe()`.

## Interface

```ts
interface FlueClient {
  readonly url: string;
  send(options: AgentPromptOptions): Promise<AgentSendResult>;
  read(target: AgentSendResult | string, options?: AgentReadOptions): Promise<AgentReadResult>;
  wait(admission: AgentSendResult, options?: AgentWaitOptions): Promise<void>;
  abort(options?: { signal?: AbortSignal }): Promise<AgentAbortResult>;
  history(options?: FlueConversationHistoryOptions): Promise<FlueConversationSnapshot>;
  observe(options?: AgentConversationObserveOptions): AgentConversationObservation;
  attachmentUrl(attachmentId: string): string;
}
```

| Method | HTTP composition |
|---|---|
| `send()` | `POST <url>` — resolves on **202 admission**, not completion. Does not carry the reply. |
| `read()` | `wait()`'s stream follow, then one `history()` read. Also accepts a **bare submission id**, so a process that persisted only the admission can re-attach later. |
| `wait()` | reads `GET <url>?view=updates` |
| `history()` | `GET <url>?view=history` |
| `observe()` | `history()` to hydrate, then the updates stream |
| `abort()` | `POST <url>/abort` — aborts the **conversation's** in-flight and queued work. There is no per-submission abort. |
| `attachmentUrl()` | resolves `<url>/attachments/<attachmentId>` |

Every HTTP method rejects with `FlueApiError` on a non-2xx response; `wait()` and `observe()` also surface stream errors. `read()` throws `FlueExecutionError` when the run fails or is aborted.

### `send()` options

```ts
interface AgentPromptOptions {
  message: DeliveredMessage;
  initialData?: unknown;
  uid?: string | null;
  signal?: AbortSignal;
}
```

- `initialData` — consulted only when this send **creates** the conversation; validated against the agent's `initialData` static; ignored when continuing. Pair with `uid: null` to error instead of silently ignoring.
- `uid` — omitted: unconditional. A string: continue only that incarnation; a missing instance or mismatched uid rejects 404 `agent_instance_not_found` and nothing is delivered (cannot be combined with `initialData`). `null`: create only; an existing instance rejects 409 `agent_instance_exists`, with the existing uid in `body.error.meta.uid`.
- `signal` aborts the **HTTP request**, not agent work. That is `abort()`.

`DeliveredMessage` is `{ kind: 'user', body, attachments? }` or `{ kind: 'signal', type, body, attributes?, tagName? }`. Image attachments are capped at **14 MiB each**.

## Service bindings and custom transports

```ts
const conversation = createFlueClient({
  // The host is never dialed — only the pathname and query select a route.
  url: 'https://agent.internal/agents/support/ticket-42',
  fetch: (input, init) => env.AGENT_APP.fetch(new Request(input, init)),
});
```

All operations work over the binding, in both long-poll and SSE modes. **Attachments are the exception:** `attachmentUrl(...)` returns a URL on the placeholder host that the client never fetches automatically — forward it through the same fetcher yourself.

The same override injects a test transport.

## React

```bash
pnpm add @flue/react @flue/sdk
```

No provider or root configuration. `@flue/react` exports only `useFlueAgent`.

```tsx
export function Chat({ conversationId }: { conversationId: string }) {
  const agent = useFlueAgent({ url: `/api/agents/support-assistant/${conversationId}` });

  return agent.messages.map((message) => (
    <article key={message.id}>
      <strong>{message.role}</strong>
      {message.parts.map((part, i) => (part.type === 'text' ? <p key={i}>{part.text}</p> : null))}
    </article>
  ));
}
```

Options: `url`, `client`, `live: 'long-poll'`. Returns `messages`, `sendMessage()`, `status`, `historyReady`, `refresh()`.

- **`sendMessage()` inserts the user message immediately and resolves when the server accepts the prompt, not when generation completes.** The stream then reconciles the optimistic message with its persisted version without disrupting ordering.
- `status` distinguishes connection, submission, streaming, and error phases. `historyReady` becomes `true` once durable history loads as a coherent snapshot and stays `true` across reconnections.
- Messages are `FlueConversationMessage` objects with parts: `text`, `reasoning`, `dynamic-tool`, `file`, plus `data-*` parts from `useDataWriter` and message `metadata` from the response hooks — narrow these, don't assume them. Server-validated tool output is retained in the `dynamic-tool` part's `output`, so custom tool rendering needs no separate channel. `file` parts carry a `url`. **These types belong to Flue; the library has no runtime dependency on `ai`.**
- Transport: the SDK's materialized `observe()` layer. Live updates default to SSE with Durable Streams long-polling fallback; request long-polling with `live: 'long-poll'`. For a single point-in-time read, call the SDK client's `history()` instead. Partial text and reasoning are best-effort — the finalized canonical assistant message is authoritative.
- `refresh()` re-runs history sync and resumes live updates — use it when conversations are created out-of-band by server processes, queue workers, or webhooks.

### Custom clients

```tsx
const client = useMemo(
  () => createFlueClient({ url: `/api/agents/support-assistant/${conversationId}`, token }),
  [conversationId, token],
);
const agent = useFlueAgent({ client });
```

**Always memoize** — a new client instance replaces the session. The hook does not take ownership, so one instance can serve both the rendered conversation and programmatic `observe()`/`wait()`/`read()` calls outside the component tree, sharing one connection pool and one auth setup.

### SSR

During SSR the hook returns empty, idle state and opens no connections; a relative URL resolves to the browser origin, so the hook activates only after hydration. **Omitting both `url` and `client` keeps the hook dormant** until routing or data resolution determines the conversation identity.

Full option and type reference: the `@flue/react` package README (`https://github.com/withastro/flue/tree/main/packages/react#readme`) and the `examples/react-chat` example.

> Known doc discrepancy: `sdk/flue-client` says `observe()`'s `live` option defaults to `'long-poll'`; `guide/migration` says SSE with long-poll fallback. Check the installed version with `flue docs read sdk/flue-client`.
