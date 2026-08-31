---
name: flue-project-setup
description: >-
  Scaffold, configure, or upgrade a Flue framework project. Use when starting a
  Flue app, editing flue.config.ts or vite.config.ts, wiring the app.ts / db.ts /
  cloudflare.ts entry modules, choosing a source root, running flue init or
  flue add, setting provider API keys, or migrating a Flue 1.0-beta project to
  Flue 2.
---

# Flue Project Setup

Flue is a Vite project. The `flue` CLI scaffolds, runs one agent, and fetches blueprints — it is **not** the build tool.

## Get current docs first

The installed CLI ships the docs for its own version, offline:

```bash
flue docs                      # list every page
flue docs read guide/project-layout
flue docs search "source root"
```

**Prefer this over the website.** It always matches the installed version, and parts of the published docs are stale:

- **Do not trust the `index.md` raw-markdown mirrors.** On several pages (`cli/overview`, `cli/run`, `cli/init`, `guide/subagents`) `<url>index.md` serves a **May 2026 pre-v2 snapshot** while the rendered HTML page is current. `/docs/cli/overview/index.md` still advertises `flue dev`, `flue build`, `flue connect` and `flue logs` — none of which exist.
- **`/docs/cli/run/` and `/docs/cli/init/`** are stale in rendered form too. They describe the removed workflow-era CLI and tell you to import `defineConfig` from `@flue/cli/config`. Use `cli/overview` and `guide/migration` instead.

## Scaffold

Requires **Node.js ≥ 22.19**.

```bash
npx flue init            # interactive: prompts for target and server setup
```

Manual: `npm install @flue/runtime @flue/cli vite @flue/vite hono` (add `@cloudflare/vite-plugin` for the Cloudflare target).

## Source root — first match wins, layouts never merge

Resolution order: `.flue/` → `src/` (recommended) → project root. **If `.flue/` exists, `app.ts`, `db.ts`, `cloudflare.ts`, and the `'use agent'` scan all resolve from it** — `src/` is ignored entirely. Pick one.

| File | Required? | Purpose |
|---|---|---|
| `src/app.ts` | **Yes** | Route map; default export is the server (`Fetchable`). |
| `src/db.ts` | No | Persistence adapter. **Node target only** — a `db.ts` on Cloudflare is rejected at build time. |
| `src/cloudflare.ts` | No | Worker-level non-HTTP handlers (`scheduled`, queues, email) and your own DO/Workflow classes. Must **not** export a default `fetch`. |
| `vite.config.ts` | No | Build config. |
| `flue.config.ts` | No | Project config; every field optional. |

## vite.config.ts

```ts
import { cloudflare } from '@cloudflare/vite-plugin'; // Cloudflare target only
import { flue } from '@flue/vite';
import { defineConfig } from 'vite';

export default defineConfig({
  plugins: [flue(), cloudflare()],
});
```

**`flue()` must come before `cloudflare()`.** The plugin resolves config, scans `'use agent'` modules, and stamps each agent's durable identity as a string literal so minification can't corrupt it.

Commands: `vite dev` (port 5173), `vite build`, `vite preview`. There is no `flue dev` or `flue build`.

## flue.config.ts

```ts
import { defineConfig } from '@flue/runtime/config'; // NOT @flue/cli/config

export default defineConfig({ target: 'node' });
```

Fields: `target` ('node' | 'cloudflare', auto-detected from the plugin array), `app`, `db`, `cloudflare`, `agents` (glob narrowing the scan), `providers`, `tracing`. Full table: `references/configuration.md`.

**`providers` is exhaustive.** `flue({ providers: ['anthropic'] })` trims the build, and any specifier naming another provider then fails at resolution. On Cloudflare include `'cloudflare'` if agents use `cloudflare/...` models. Omit the field to keep all providers.

## Credentials

`.env` at the project root — `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY` (Google), `GROQ_API_KEY`, `CLOUDFLARE_API_KEY`. Shell-exported values always win.

- `vite dev` loads Vite's standard `.env` set. `flue run` loads only the root `.env` (`--env <path>` to override).
- **Built servers never load `.env`** — supply real environment variables. Cloudflare: `wrangler secret put`, `.dev.vars` locally.
- Never commit `.env`.

## Blueprints

`flue add`/`flue update` fetch a **Markdown implementation guide for a coding agent** — they are not package installers.

```bash
flue add                                  # list the catalog
flue add channel slack --print | claude
flue add database postgres --print | claude
flue add sandbox e2b --print | claude
flue add tooling vitest-evals --print | claude
flue update channel slack --print | claude
```

Kinds: `channel`, `database`, `sandbox`, `tooling`. A provider docs URL in place of a name selects the build-from-scratch guide.

## Migrating from 1.0.0-beta

Read `references/migration-from-beta.md` before touching code. **Persisted state does not migrate** — the runtime rejects a beta database (schema v5 vs v8) before application code runs. Plan a drained deployment or export/re-seed.

## Anti-patterns

- Adding `db.ts` to a Cloudflare project (build error), or expecting it to be picked up by a `start()` script (pass `db:` directly).
- Putting `cloudflare()` before `flue()` in the plugin array.
- Mixing `.flue/` and `src/` and expecting both to be scanned.
- Trusting `/docs/cli/run/`, `/docs/cli/init/`, or any `index.md` mirror; importing `defineConfig` from `@flue/cli/config`.
- Committing `.flue-vite/` or `.flue-vite.wrangler.jsonc` — both are generated; gitignore them.
