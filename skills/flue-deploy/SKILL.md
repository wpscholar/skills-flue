---
name: flue-deploy
description: >-
  Deploy a Flue application and reason about its durability — vite build, the
  Node server, the Cloudflare Worker with its generated Durable Objects and
  wrangler migrations, secrets per target, and the accepted-work and recovery
  contract. Use when deploying a Flue app, adding, renaming or removing an agent
  on Cloudflare, or debugging interrupted or retried runs.
---

# Deploying Flue

`vite build` produces the artifact. The target is auto-detected from the Vite plugin array, or set with `target` in `flue.config.ts`.

Current API: `flue docs read guide/deploy` · `guide/cloudflare-target` · `guide/node-target` · `guide/durability`

## Node

```bash
vite build
node dist/server.mjs
```

`dist/server.mjs` is the self-starting server; `dist/app.mjs` is the non-listening chunk it imports.

- **The built server does not load `.env`.** Supply real environment variables (`set -a; source .env; set +a` if you must).
- Port `3000`, override with `PORT`.
- **Dependencies are externalized, not bundled** — ship `node_modules` alongside or install them in the container.
- Without a `db.ts` adapter, conversation state is process memory and is lost on restart. Configure a durable adapter before production.
- **One live process must own a given conversation.** A shared database lets a *replacement* process recover accepted work; it does not make two concurrent owners safe. Multi-replica deployments must route each conversation to one owner.
- `vite preview` tests the built artifact locally — but applies the **same permissive CORS as `vite dev`**, so it will not catch a missing CORS config.

Recovery behavior, `local()`, and `sqlite()` on this target: `references/node-target.md`.

## Cloudflare

`flue()` must appear **before** `@cloudflare/vite-plugin` in the plugins array. Flue generates one Durable Object class and binding per exported agent function:

```
export function SupportChat() {…}  →  class FlueSupportChatAgent
                                      env.FLUE_SUPPORT_CHAT_AGENT
```

Gitignore `.flue-vite/` and `.flue-vite.wrangler.jsonc` — both are generated. Your `wrangler.jsonc` is **never modified**; the build merges it and `vite build` writes the finalized config into `dist/`. Deploy with plain `wrangler deploy` from the project root.

You must author two things:

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "my-flue-worker",
  "compatibility_date": "2026-06-01",          // minimum 2026-04-01, validated at build
  "compatibility_flags": ["nodejs_compat"],     // required — Flue's runtime uses Node APIs
  "migrations": [
    { "tag": "v1", "new_sqlite_classes": ["FlueSupportChatAgent"] }
  ]
}
```

**Do not hand-author Flue's generated `FLUE_*` bindings.** Declare only migrations for generated classes, plus bindings for your own resources.

### Migrations — the append-only rule

Migration history is user-authored: an ordered, **append-only** record of your deployments. Never rewrite or reorder deployed entries. Generated agent classes require DO SQLite — use `new_sqlite_classes`, **not** legacy `new_classes`.

Adding an agent is three steps: the exported function in a `'use agent'` module, the mount in `app.ts` (optional for dispatch-only agents), and a `new_sqlite_classes` entry.

| Change | Migration |
|---|---|
| Add an agent | `new_sqlite_classes: ["FlueNewAgent"]` |
| Remove a deployed agent | `deleted_classes: ["FlueOldAgent"]` — **without it Wrangler fails**, because the history references a class the Worker no longer provides |
| Rename an agent (function rename or `agentName` edit) | `renamed_classes: [{ from, to }]` — preserves stored conversations |
| Re-mount at a different URL | **None.** The mount path is not the identity. |

Set `agentName` on every agent from the start and this problem mostly disappears.

`cloudflare.ts` carries Worker-level non-HTTP handlers (`scheduled`, queues, email) and your own DO/Workflow classes as named exports. **It must not export a default `fetch` handler** — HTTP composition stays in `app.ts`. Update `run_worker_first` to cover your actual mounts, e.g. `["/api/*", "/agents/*", "/channels/*"]`.

Extending the generated DO with Agents SDK capabilities (`onStart`, `schedule`, `scheduleEvery`, `queue`), service bindings, Workers AI, and the sandbox options: `references/cloudflare-target.md`.

## Secrets

| Target | Mechanism |
|---|---|
| Node | The host's own environment. Built servers read only what is supplied at start. |
| Cloudflare | `npx wrangler secret put ANTHROPIC_API_KEY`; `.dev.vars` locally. |
| `cloudflare/...` models | No key at all — authorization and billing follow the Worker. |

## The durability contract

Every input — an HTTP prompt, `dispatch(...)`, a channel delivery, a scheduled trigger — is admitted as a **submission**, recorded durably *before* any model work begins.

> Every accepted submission reaches exactly one durable terminal outcome — `completed`, `failed`, or `aborted` — no matter how many crashes happen in between.

The discipline is **at-least-once execution over exactly-once recording**. Work that committed durably never re-runs; work interrupted before committing re-runs on the next attempt.

What that means in practice:

- **Interrupted ordinary tool calls are never re-executed.** They settle with an explicit unknown-outcome error the model sees. Use `durable: true` + `step.do` for work that must complete.
- **Event-hook callbacks run at-least-once.** Durable effects commit atomically; an external side effect inside one — an email, a page — may rarely happen twice. Guard with `usePersistentState`, whose writes commit atomically with the unit of work that made them.
- **Sandbox files are not durable.** A recovered attempt re-initializes the environment; anything an interrupted attempt wrote to an ephemeral workspace is gone.
- **`read()` / `wait()` promises are not durable.** Persist the `DispatchReceipt` to re-attach from any process.
- **Flue does not checkpoint arbitrary TypeScript.** The checkpoint boundary is the agent. Outside it, use a workflow engine and treat Flue like any other service — putting the dispatch in its own step so the receipt is checkpointed the moment it exists.

Defaults are **10 attempts, 1 hour per submission**, overridable per agent:

```ts
IssueTriage.durability = { maxAttempts: 5, timeoutMs: 7_200_000 };
```

The timeout is total wall-clock from the first attempt's start; turn-boundary joins and response continuations do not extend it. Recovery classification, tool-batch repair, and per-target recovery behavior: `references/durability.md`.

## Anti-patterns

- Renaming or removing a deployed agent without the matching storage migration.
- Using `new_classes` instead of `new_sqlite_classes`.
- Editing or reordering a deployed migration entry.
- Committing `.flue-vite.wrangler.jsonc`, or hand-writing `FLUE_*` bindings.
- Shipping a Node build without `node_modules` or without a durable `db.ts`.
- Treating `vite preview` as a CORS check.
- Assuming a redeploy is safe for in-flight durable work — a redeploy that removes a tool's `durable` flag or a subagent silently withdraws the recovery contract for calls in flight.
