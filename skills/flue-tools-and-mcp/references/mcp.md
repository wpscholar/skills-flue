# MCP connections in Flue

Live source: `flue docs read guide/mcp` · `flue docs read reference/agent-api` (`#mcpconnectiondefinition`)

## Connecting

```ts
'use agent';
import { useMcpConnection, useModel } from '@flue/runtime';

export function ProjectAssistant() {
  useModel('anthropic/claude-sonnet-4-6');
  useMcpConnection({
    name: 'linear',
    url: 'https://mcp.linear.app/mcp',
    auth: process.env.LINEAR_API_KEY,
  });
  return 'Manage Linear issues and projects for the team.';
}
```

Tools mount as **`mcp__<server>__<tool>`** (e.g. `mcp__linear__create_issue`) so they cannot collide with your own.

## Connection behavior

- Flue connects when the agent starts working on a message and reuses the connection while the instance stays live. You never open or close connections yourself.
- All declared servers connect **in parallel**, inside request context, on every target.
- A server that can't be reached **fails the run** with an error; the next message retries. `optional: true` runs without that server's tools instead, and the model is told they are unavailable (via a `resources` signal with `resource="mcp"`, emitted once per degraded submission).
- Definitions are read at first connect — **`auth` excepted, which resolves per request**.
- The declaration may be conditional, like a tool mount. `useMcpConnection` throws in a subagent render.
- Default request timeout is 60 s.

## Authentication

`auth` accepts a **string** (static bearer token) or a **function** (resolved on every request, so tokens can rotate and revoke).

```ts
export function Assistant() {
  const { userId } = useInitialData<{ userId: string }>();

  useMcpConnection({
    name: 'linear',
    url: 'https://mcp.linear.app/mcp',
    auth: () => tokenStore.get(userId, 'linear'),
  });

  return 'Help this user manage their Linear issues.';
}
```

> Flue never stores or manages your tokens. It is your responsibility to own any OAuth flow, token storage, and refresh token logic.

Authorizing mid-conversation, via a persistent flag:

```ts
const [linearReady, setLinearReady] = usePersistentState('linear-ready', false);

if (linearReady) {
  useMcpConnection({ name: 'linear', url: LINEAR_MCP_URL, auth: () => tokenStore.get(userId, 'linear') });
}

useAgentStart(async () => {
  if (!linearReady && (await tokenStore.has(userId, 'linear'))) setLinearReady(true);
});
```

If several agents need to share one user's authorization, put the integration in your application — an application-owned integration service can itself be an MCP server that agents connect to.

## Allowlist

```ts
useMcpConnection({
  name: 'linear',
  url: 'https://mcp.linear.app/mcp',
  auth: process.env.LINEAR_API_KEY,
  tools: ['create_issue', 'search_issues', 'get_issue'],
});
```

**If the allowlist names a tool the server doesn't expose, the connection fails with an error.**

## `defineMcpConnection()`

```ts
import { defineMcpConnection } from '@flue/runtime';

export const linear = defineMcpConnection({
  name: 'linear',
  url: 'https://mcp.linear.app/mcp',
  auth: process.env.LINEAR_API_KEY,
});
```

```ts
useMcpConnection(linear);
useMcpConnection({ ...linear, tools: ['search_issues'] }); // override fields per mount
```

## Security

> An MCP server you connect to can influence your agent: its tool descriptions enter the prompt, and its tool results enter the conversation. Treat a server you don't control like any other third-party dependency, and consider using the `tools` allowlist to limit the surface area of your exposure.

## Advanced: `createMcpConnection()`

The lower-level function beneath the hook. It connects, discovers tools, and returns them as `ToolDefinition` values so trusted code can filter or wrap them before mounting with `useTool`.

```ts
const linear = await createMcpConnection({ name: 'linear', url: LINEAR_MCP_URL, auth: TOKEN });

export function ProjectAssistant() {
  useModel('anthropic/claude-sonnet-4-6');
  for (const tool of linear.tools) useTool(tool);
  return 'Manage Linear issues and projects for the team.';
}
```

Also usable from a standalone Node script through the JavaScript API. Use this when you need to rename, narrow, or wrap a server's tools rather than mount them verbatim.
