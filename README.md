# wpscholar/skills-flue

Agent Skills for building with [Flue](https://flueframework.com) — the open TypeScript agent framework from the creators of Astro. Seven skills covering the whole surface: project setup, agent authoring, tools and MCP, skills/subagents/sandboxes, serving and clients, deployment and durability, evals and observability.

Companion to [wpscholar/skills](https://github.com/wpscholar/skills), which holds the general-purpose and project-type skills. This repo is Flue-only, so it uses a **flat layout** (`skills/<skill>/`) like [cloudflare/skills](https://github.com/cloudflare/skills) rather than that repo's `skills/<namespace>/<skill>/` split.

## Layout

```
skills/
  flue-project-setup/
  flue-agent-authoring/
  flue-tools-and-mcp/
  flue-skills-subagents-sandboxes/
  flue-serving-and-clients/
  flue-deploy/
  flue-evals-and-observability/
```

Each skill is a lean `SKILL.md` (the workflow, the contracts, the anti-patterns) plus `references/` files loaded on demand.

## Installing

All seven are domain skills — install at **project scope**, from inside a Flue project:

```bash
gh skill install wpscholar/skills-flue flue-project-setup              --agent claude-code --scope project
gh skill install wpscholar/skills-flue flue-agent-authoring            --agent claude-code --scope project
gh skill install wpscholar/skills-flue flue-tools-and-mcp              --agent claude-code --scope project
gh skill install wpscholar/skills-flue flue-skills-subagents-sandboxes --agent claude-code --scope project
gh skill install wpscholar/skills-flue flue-serving-and-clients        --agent claude-code --scope project
gh skill install wpscholar/skills-flue flue-deploy                     --agent claude-code --scope project
gh skill install wpscholar/skills-flue flue-evals-and-observability    --agent claude-code --scope project
```

Or everything at once:

```bash
gh skill install wpscholar/skills-flue --all --agent claude-code --scope project
```

Refresh after pushing changes here: `gh skill update --all`.

Local iteration without committing:

```bash
gh skill install ~/Dev/ai/wpscholar:skills-flue flue-deploy --agent claude-code --scope project --from-local
```

`--from-local` copies rather than links, so re-run after each edit.

**Do not install these at user scope.** Seven always-on descriptions is real dead weight in every non-Flue session, and `flue-skills-subagents-sandboxes` in particular will compete for triggers with skill-authoring work in other repos.

## The skills

| Skill | Covers |
|---|---|
| `flue-project-setup` | `flue init`, source-root resolution, `vite.config.ts` and the `flue()` plugin, `flue.config.ts`, entry modules, provider keys, `flue add`/`update`/`docs`, migrating from 1.0-beta |
| `flue-agent-authoring` | `'use agent'`, durable identity, the rules of hooks, submission scoping, statics, models and compaction, persistent state, event hooks, prompt-cache discipline |
| `flue-tools-and-mcp` | `defineTool`/`useTool`, the result envelope, `harness: true`, `durable: true` + `step.do`, conditional tools, tool authorization, `useMcpConnection` |
| `flue-skills-subagents-sandboxes` | Agent Skills packaging and workspace skills, `useSubagent`/`GeneralSubagent` and what delegates inherit, the sandbox selection matrix |
| `flue-serving-and-clients` | `app.ts`, `createAgentRouter`, auth and CORS, `dispatch`/`init`/`start`, channels, schedules, `db.ts`, `@flue/sdk`, `@flue/react` |
| `flue-deploy` | `vite build`, the Node server, Cloudflare Workers and generated Durable Objects, wrangler migrations, secrets, the durability and recovery contract |
| `flue-evals-and-observability` | Eval suites, `vitest-evals`, judges, the runtime `observe()` stream, usage and cost, Sentry/Braintrust/OTel/Cloudflare tracing, content redaction |

## How these stay current

Flue ships its own documentation inside the CLI package, offline and version-matched:

```bash
flue docs                              # list every page
flue docs read guide/sandboxes         # print one page as markdown
flue docs search "durable execution"   # JSON, up to 8 results
```

Every skill points at `flue docs` for the live API rather than trying to be a complete mirror. What the skills carry instead is the part that does not fall out of the reference: the durable contracts, the ordering rules, and the failure modes — the things that cost an afternoon when you get them wrong.

Prefer `flue docs` over the website, and over the website's raw-markdown mirrors in particular — see below.

## Documentation traps encoded here

Flue's published docs have a few known-bad spots as of August 2026. Each is called out in the relevant skill:

- **The `index.md` raw-markdown mirrors are not reliable.** Appending `index.md` to a docs URL returns markdown, but on several pages it serves a **May 30, 2026 pre-v2 snapshot** while the rendered HTML page is current (Jul 2026). Confirmed stale mirrors: `cli/overview`, `cli/run`, `cli/init`, `guide/subagents`. `/docs/cli/overview/index.md` still advertises `flue dev`, `flue build`, `flue connect`, `flue logs` and a `--target/--root/--output/--config` option set, none of which exist in v2. **Use `flue docs` (offline and version-matched) or the rendered HTML page.**
- **`/docs/cli/run/` and `/docs/cli/init/`** are stale in rendered form too, documenting the removed workflow-era CLI and `defineConfig` imported from `@flue/cli/config` (v2 uses `@flue/runtime/config`). Use `cli/overview` and `guide/migration`.
- **`thinkingLevel`** values differ between `guide/models` and `reference/agent-hooks-api`.
- **`observe()`'s `live` default** is documented as `'long-poll'` in `sdk/flue-client` and as SSE-with-fallback in `guide/migration`.

## Authoring

1. Copy an existing skill directory to `skills/<skill-name>/`.
2. `name` in the frontmatter must match the directory name and stay globally unique across every skill source (installs flatten to the directory name). Keep the `flue-` prefix.
3. `description` is injected into every session at the installed scope — keep it ~40–60 trigger-rich words covering **what it does and when to load it**.
4. Keep `SKILL.md` under ~150 lines; push detail into `references/`.
5. Test with `--from-local`, then commit and push.

Provenance metadata added by `gh skill install` points back here, so `gh skill update --all` refreshes these alongside third-party sources.
