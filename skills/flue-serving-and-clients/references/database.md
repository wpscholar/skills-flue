# Persistence in Flue

Live source: `flue docs read guide/database` · `flue docs read reference/data-persistence-api`

**The database is a Node.js concern.** On the Cloudflare target every agent conversation is a Durable Object with its own SQLite storage — there is nothing to configure, and **a source-root `db.ts` is rejected at build time**.

## What Flue stores

Runtime durable state, not your business data:

- **Canonical conversations** — one append-only stream per conversation: user messages, assistant output, tool calls and results, compaction, recovery facts. Every later turn, reconnecting client, and crash recovery rebuilds from it.
- **Accepted submissions** — recorded durably *before* processing begins, with claims and leases tracking which process owns the work.
- **Persisted state** — every `usePersistentState` write, as a record in the conversation's stream.

Attachment payloads are stored alongside the conversation as immutable records the stream references.

Flue does **not** store sandbox files or installed dependencies, external API side effects, provider credentials, or your application's data. A durable database does not make a sandbox durable, and a durable workspace does not preserve conversation history.

## `src/db.ts`

```ts
import { sqlite } from '@flue/runtime/node';

export default sqlite('./data/flue.db');
```

Discovered by convention from the source root by `vite dev`, `vite build`, and `flue run`; relocate with the `db` field in `flue.config.ts`. At boot Flue calls the adapter's `migrate()` once, then awaits `connect()` — an unreachable or misconfigured database fails at **startup**, not in the middle of your first conversation.

**Standalone `start()` scripts don't go through the build, so they don't pick up `db.ts`** — pass the adapter directly via the `db` option.

## The in-memory default

Without `db.ts`, in-memory SQLite. Everything works within the process lifetime; **a restart loses everything.** In production this is acceptable only for genuinely disposable agents.

| Command | Without `db.ts` |
|---|---|
| `vite dev` | A cache file (`node_modules/.cache/flue/dev.db`) — history survives code reloads, resets on a cold start. |
| `flue run` | A cache file (`node_modules/.cache/flue/run.db`) — never reset, so `--id` continues conversations across invocations. |
| `vite build` | In-memory — the deployed server keeps state only for the process lifetime. |

## The built-in `sqlite()` adapter

Ships with the runtime on Node's built-in `node:sqlite`, no extra dependencies. Creates the file and missing parent directories on first boot; opens in **WAL mode**. `sqlite()` with no argument, or `sqlite(':memory:')`, gives the in-memory database. Survives process restarts and redeploys on the same machine — **not** loss of the host.

## Ecosystem adapters

```bash
flue add database postgres --print | claude
```

| Backend | Package |
|---|---|
| Postgres, Supabase | `@flue/postgres` |
| libSQL, Turso | `@flue/libsql` |
| MySQL | `@flue/mysql` |
| MongoDB | `@flue/mongodb` |
| Redis, Valkey | `@flue/redis` |

**Bring your own driver** — the adapter never picks, bundles, or configures one. You supply a runner: a `query` function, a `transaction` function, and a `close` function.

```ts
import { postgres } from '@flue/postgres';
import { Pool } from 'pg';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

export default postgres({
  query: async (text, params) => (await pool.query(text, params)).rows,
  transaction: async (fn) => {
    const client = await pool.connect();
    try {
      await client.query('BEGIN');
      const result = await fn({ query: async (text, params) => (await client.query(text, params)).rows });
      await client.query('COMMIT');
      return result;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  },
  close: () => pool.end(),
});
```

Migrations are not hand-written: `migrate()` provisions tables idempotently and **stamps a format version**, so a database written by an incompatible Flue version refuses to start rather than corrupting state.

## A shared database does not enable active-active scaling

> A durable external database lets a replacement process recover accepted work and lets replicas share conversation state, but **each agent conversation still needs exactly one live Node owner at a time.**

Multi-replica deployments must route each conversation consistently to one owner and avoid overlapping owners during replacement.

## Custom adapters

An adapter is an object with `connect()` returning three stores, plus optional `migrate()` and `close()`. Types in `@flue/runtime/adapter`.

```ts
import type { PersistenceAdapter } from '@flue/runtime/adapter';

export default {
  migrate() { /* create or verify backing storage */ },
  connect() { return { submissionStore, conversationStreamStore, attachmentStore }; },
  close() { /* release connections */ },
} satisfies PersistenceAdapter;
```

The contract has strict atomicity and ordering requirements — idempotent admission, fenced producer claims, append-only streams. **Run the contract test suites from `@flue/runtime/test-utils`**; they are the acceptance tests every built-in adapter passes.

## Choosing

| Situation | Choice |
|---|---|
| Local development | The defaults — add `db.ts` only to develop against production storage |
| Single-host Node deployment | File-backed `sqlite()` |
| State must survive host loss, or many replicas | An ecosystem adapter, with one live owner routed per conversation |
| Cloudflare deployment | Nothing to configure — Durable Object SQLite is automatic |
| A backend not in the catalog | A custom `PersistenceAdapter` |
