# Flue configuration reference

Live source: `flue docs read reference/configuration` · `flue docs read cli/overview`

## `flue.config.ts`

```ts
import { defineConfig } from '@flue/runtime/config';

export default defineConfig({ target: 'node' });
```

Optional file at the project root; must default-export the config object; every field optional.

```ts
interface FlueConfig {
  target?: 'node' | 'cloudflare';
  app?: string;
  db?: string;
  cloudflare?: string;
  agents?: string;
  providers?: string[];
  tracing?: boolean;
}
```

| Field | Meaning / default |
|---|---|
| `target` | `'node'` (self-starting server) or `'cloudflare'` (Workers). Unset by default; `flue()` auto-detects from the Vite plugin array. Explicit config overrides detection. |
| `app` | Path to `app.ts`. Defaults to a source-root lookup. **Required by `vite dev` and `vite build`.** |
| `db` | Path to `db.ts`. **Node target only.** Unset → built-in in-memory SQLite. |
| `cloudflare` | Path to `cloudflare.ts`. Defaults to a source-root lookup. |
| `agents` | Glob narrowing the `'use agent'` scan, relative to the source root (e.g. `'agents/**/*.ts'`). Default: the whole source root, recursively. |
| `providers` | Provider IDs registered at server start. Default: all built-ins. **Exhaustive when set.** |
| `tracing` | Agent tracing on the Cloudflare target. Default enabled; `false` removes it from the build. |

## The `flue()` Vite plugin

```ts
import { flue } from '@flue/vite';
function flue(config?: FlueConfig): Plugin[];
```

Inline options merge **over** the discovered file config — inline wins.

What it does:
1. Resolves the project — discovers `flue.config.ts`, locates entry modules (`app.ts` required).
2. Scans for `'use agent'` modules and registers discovered agents in the generated bootstrap.
3. Transforms agent modules — embeds each agent's identity as an immutable string literal.

Behavior by consumer:

| Consumer | Behavior |
|---|---|
| `vite dev` | Loads config, requires `app`, scans agents, watches config and agent-set changes. |
| `vite build` | Resolves once; requires `app` and a successful scan. Forces build config on the Node target. |
| `vite preview` | Validates config; does not require entries or scan agents. |
| `flue run` | Discovers config from cwd; **ignores `target` and `agents`**; always runs Node-locally. |

## CLI

`@flue/cli`, single `flue` binary, Node ≥ 22.19. Every command prints its payload to **stdout** and everything else to **stderr**, so piping stdout is safe. Global flags: `--help`/`-h`, `--version`/`-v` — there are no others, and every command rejects flags it does not declare.

| Command | Purpose |
|---|---|
| `flue init [directory]` | Interactive project scaffold; adds `@flue/cli` as a devDependency. |
| `flue run <path>` | Run one agent module locally — no server, no `app.ts`. |
| `flue add [kind] [name\|url]` | Fetch a blueprint implementation guide. Bare `flue add` lists the catalog. |
| `flue update <kind> <name\|url>` | Blueprint guide for updating an existing integration. Both args required. |
| `flue docs [read\|search]` | Offline docs bundled with the installed CLI version. |

### `flue run`

```
flue run <path> -m|--message <text> [--id <id>] [--name <agent>] [--data '<json>'] [--uid <uid>] [--new] [--json] [--env <path>]
```

- Module path and `-m`/`--message` are required.
- `--name` selects among several agents exported from one module.
- `--data '<json>'` is instance creation data, validated by the `initialData` static.
- `--uid` / `--new` are send conditions; `--new` plus a deterministic `--id` makes conversation creation exactly-once in CI.
- `--json` swaps the plain reply for a result envelope (outcome, reply, conversation id).
- Stdout is the reply only; everything else goes to stderr.
- Conversations persist between invocations in the configured database, or `node_modules/.cache/flue/run.db` without one.
- **`flue run` never loads `app.ts`** — put `setProvider()` and `observe()` registrations in the agent module if they must work under the CLI.
- Removed beta flags: `--server`, `--header`, `--target`, `--root`, `--output`, `--config`, and workflow names.
- Delivers only `kind: 'user'` messages — there is no signal form from the CLI.

### `flue docs`

```bash
flue docs                       # list every page with path and title
flue docs read guide/sandboxes  # also accepts /docs/guide/sandboxes/ or the full URL
flue docs search "durable execution"   # JSON, up to 8 results
```

Offline; consistent with the installed CLI version. Invalid page references exit 1.

### `flue add` / `flue update`

Blueprints are Markdown guides retrieved at runtime from `https://flueframework.com/cli/blueprints/`. `--print` writes the guide to stdout regardless of coding-agent detection.

Kinds: `channel`, `database`, `sandbox`, `tooling`.

## Environment variables

| Provider | Variable |
|---|---|
| `anthropic` | `ANTHROPIC_API_KEY` |
| `openai` | `OPENAI_API_KEY` |
| `google` | `GEMINI_API_KEY` |
| `groq` | `GROQ_API_KEY` |
| `cloudflare-workers-ai`, `cloudflare-ai-gateway` | `CLOUDFLARE_API_KEY` |

The Cloudflare-target `cloudflare/...` provider needs no key — authorization and billing follow the Worker.

Load order: `vite dev` uses Vite's standard `.env`, `.env.local`, `.env.<mode>`, `.env.<mode>.local`. `flue run` loads only the root `.env` unless `--env <path>`. Built servers load nothing.

## Generated files to gitignore

- `.flue-vite/`
- `.flue-vite.wrangler.jsonc`
- `dist/`
