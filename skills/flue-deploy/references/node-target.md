# The Node.js target

Live source: `flue docs read guide/node-target`

## The built server

```bash
vite build
node dist/server.mjs
```

- `dist/server.mjs` — the self-starting server: HTTP routing, agent dispatch, durable conversation streaming.
- `dist/app.mjs` — the non-listening chunk the server imports.
- Default port `3000`; override with `PORT`.
- **Dependencies are externalized, not bundled.** Deploy `node_modules` alongside, or package them in a container.

## Local development

`vite dev` loads the app through Vite's module graph on **port 5173**. Flue adds: watching `'use agent'` modules without manual restarts; a server restart when `flue.config.*` changes; automatic `.env` loading (shell-exported values take precedence); and permissive CORS defaults for separately served SPAs.

`vite preview` serves the built artifact with production configuration but applies **the same developer-friendly CORS policy as `vite dev`** — so it does not validate your production CORS setup.

## State and durability

Without a database adapter the Node server uses **process-local in-memory SQLite** — ordered within one process, lost on restart. With a durable adapter, prompts and dispatched inputs queue persistently per conversation, and a replacement process can recover canonical progress and interrupted submissions.

**Node requires one live process to own a given agent conversation.** Multi-replica deployments must route each conversation consistently to the same replica to prevent unsafe overlapping ownership during transitions.

See the `flue-serving-and-clients` skill's `database.md` for adapters.

### Recovery on Node

Lease-based ownership: a coordinator inside your server process owns submission processing, and each running submission carries a short lease heartbeated by the owning process.

- **Startup reconciliation.** A replacement process scans for interrupted work at boot, requeues it, then serves immediately while it settles in the background. Ordering is preserved per conversation — recovered work runs ahead of newly delivered work, so a restart never reorders a conversation's timeline.
- **Periodic lease scans** reclaim work stranded by a fast restart (a new process booting before the old lease expires) within seconds.
- **Graceful shutdown** aborts active submissions at the turn boundary and waits; work that doesn't settle in time is left running with its lease intact and reclaimed on the next startup after expiry.

Two deployment consequences:

- **Recovery is only as durable as the database.** With the in-memory default, accepted work survives interruptions within the process lifetime but a restart loses everything.
- **One live owner per conversation.** A shared database lets a *replacement* process recover accepted work; it does not make two concurrent owners of the same conversation safe.

## `local()` sandbox

Node-only. Direct host filesystem and shell access — development tools, CI tasks, self-hosted automation.

```ts
import { local } from '@flue/runtime/node';

function local(options?: LocalSandboxOptions): SandboxFactory;
```

`LocalSandboxOptions`:
- `cwd` — working directory, default `process.cwd()`
- `env` — additional environment variables layered on the default allowlist; an `undefined` value **removes** a default

**Only shell-essential variables are exposed by default** (`PATH`, `HOME`, `USER`, `LANG`, `TERM`, `TMPDIR` and similar). API keys, tokens, and cloud credentials are deliberately excluded unless passed explicitly:

```ts
useSandbox(local({ env: { GH_TOKEN: process.env.GH_TOKEN } }));
```

There is no isolation. Do not use it as an isolation boundary for untrusted requests or multiple tenants.

## Remote sandboxes

For per-session isolation or specialized environments, remote adapters connect through the Sandbox Adapter API and run on external infrastructure — Daytona, E2B, Modal and others in the ecosystem catalog.

## `sqlite()`

```ts
import { sqlite } from '@flue/runtime/node';

function sqlite(path?: string): PersistenceAdapter;
```

Omitting `path` gives in-memory storage; passing a path persists across restarts (WAL mode; creates the file and parent directories on first boot).

## Environment and secrets

Dev `.env` load order: `.env`, `.env.local`, `.env.<mode>`, `.env.<mode>.local`. `flue run` loads **only** the project-root `.env` unless overridden with `--env <path>`.

In production, `node dist/server.mjs` reads only the environment supplied at start:

```bash
set -a; source .env; set +a
node dist/server.mjs
```

Use standard provider variable names — `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`.
