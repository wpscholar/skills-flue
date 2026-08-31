# Migrating 1.0.0-beta.x → Flue 2

Live source: `flue docs read guide/migration`

## Read this first: persisted state resets

Flue 2 stores **schema version 8**; the beta used **version 5**. Pre-1.0 schemas are **reset-only** — the runtime rejects an older database before any application code runs. There is no in-place migration.

- Disposable state → plan a drained deployment: retire old agents (`deleted_classes` on Cloudflare) and create fresh ones. Application data stored alongside agent storage is deleted with it.
- State that must survive → export it through the beta application first, re-seed after.

## Build and dev

`flue build` and `flue dev` are removed; a Flue app is a Vite project.

- `flue dev` → `vite dev` (**port 3583 → 5173**)
- `flue build` → `vite build`
- `--target` → auto-detected from plugins, or `flue.config.ts`
- `defineConfig` moves from `@flue/cli/config` to **`@flue/runtime/config`**; drop `root` and `output`
- Gitignore `.flue-vite/` and `.flue-vite.wrangler.jsonc`

## Routing: the auto-router is gone

The beta's `flue()` router no longer exists. `app.ts` is required and mounts everything explicitly with `createAgentRouter(fn)` and `channel.route()`. Registration comes from the `'use agent'` scan, not from mounting.

- Agent-module `export const route` / `export const attachments` are **deleted**
- `POST /:id` body: `{ "message": "...", "images": [...] }` → `{ "message": { "kind": "user", "body": "..." } }` with optional top-level `initialData` and `uid`
- The `?wait` query parameter is gone — follow `streamUrl` or use the SDK's `wait()`
- Workflow routes, the `runs` export, and `WorkflowRouteHandler`/`WorkflowRunsHandler` are removed

## Agent definition

`defineAgent(async ctx => ({...}))` → an exported capitalized function in a `'use agent'` module.

| Beta config field | Flue 2 |
|---|---|
| `model` | `useModel(model, options?)` — exactly once per root render |
| `instructions` | the function's return string; `useInstruction()` appends |
| `tools` | per-tool `useTool()` |
| `skills` | per-skill `useSkill()` |
| `subagents` (profiles) | `useSubagent({ name, description, agent, model?, thinkingLevel? })` — agents are functions, not profiles |
| `thinkingLevel`, `compaction` | options to `useModel()` |
| `sandbox` + `cwd` | `useSandbox(factory, { cwd })`, at most once per render |
| `durability` | a static: `Support.durability = { maxAttempts: 5 }` |
| `profile` | removed — use custom hooks |
| `actions` | removed — express as tools with `harness: true` |
| `description` | deleted |
| `ctx.id` | `AgentProps` on the root function |
| `ctx.env` | platform imports |
| async initializers | must be synchronous — async work moves to tools, lifecycle hooks, or resource factories |

Identity is the exported function name or an `fn.agentName = '...'` string-literal static. Setup facts previously parsed out of the conversation id move to the `initialData` static + `useInitialData()`.

New: `usePersistentState`, `useInitialData`, `useDelivery`, `useDispatchMessage`, `useDataWriter`, `useAgentStart`/`useAgentFinish`, `useResponseStart`/`useResponseFinish`.

## Tools

- `run()` returns an envelope `{ output?, terminate? }`. Bare strings are still sugar; returning nothing is fine when no `output` schema is declared; **any other bare value throws at runtime**.
- `run({ input })` → `run({ data })`
- `harness: true` replaces session plumbing — `harness.session()`, `FlueSession`, `FlueSessions` are gone; `harness.fs` → `harness.sandbox`
- The `task` tool's `agent` parameter is now **required**; unnamed tasks no longer clone parent config — declare `useSubagent(GeneralSubagent)` for a blank fresh-context delegate
- New `durable: true` with `step.do(name, fn)`
- New `useMcpConnection(...)`

## Skills and markdown imports

- Import attributes (`with { type: 'skill' }`) removed — the specifier decides
- An import resolving to `SKILL.md` packages the directory and returns a `SkillReference`
- Any other `.md` import is plain text; wrap it in `defineSkill({ name, description, instructions })` to make it a skill
- `?raw` / `?url` keep their Vite meanings
- Manual `session.skill(...)` invocation is gone — steer activation by naming the skill in instructions or `harness.prompt()` text

## Sandboxes

The implicit virtual sandbox is gone. Without `useSandbox()` an agent has no filesystem, no `read`/`write`/`edit`/`bash`/`grep`/`glob`, and `harness.sandbox` throws.

```ts
useSandbox(bash(() => new Bash({ fs: new InMemoryFs() })))  // add just-bash to deps
```

Create remote providers **lazily inside** `createSandbox(options)` — `options.id` carries the conversation id there. The beta's eager module-scope `await Sandbox.create()` must move inside.

## Workflows are removed

`defineWorkflow`, `invoke()`, runs and run events are gone. Replace with the smallest fit:

1. **One model operation returning a value** → `init()` handle: `dispatch()` then `read(receipt)`. Failure/abort rejects with `AgentRunError`. `read(receipt, { signal })` cancels only the local wait — call `abort()` to stop the run.
2. **Checkpointed side effects inside an agent** → a `durable: true` tool with `step.do(...)`.
3. **Multi-step orchestration with its own durability** → your own orchestrator (Cloudflare Workflows, Inngest, Temporal) calling the `init()` handle. Run inspection has no framework replacement — reconcile from your orchestrator plus `submission_settled` events.

Retry policy moves to the `durability` static. Scheduled workflows: cron now dispatches a signal instead of calling `invoke()`. Standalone Node scripts boot with `start()` from `@flue/runtime/node`.

## Dispatch

- `dispatch({ agent: 'name', ... })` removed — pass the agent function
- Creation seed field is `initialData`
- `uid` send conditions: omit → continue-or-create; a prior receipt's uid → continue only that incarnation (`AgentInstanceNotFoundError`/404); `null` → create only (`AgentInstanceExistsError`/409, carrying the existing uid)
- **`dispatchId` → `submissionId`** (new ids are `sub_`-prefixed and opaque); `FlueEvent` no longer carries `dispatchId`; telemetry emits `flue.submission.id`

## Channels

- Mount explicitly: `app.route('/channels/slack', channel.route())` at the old auto-mount path so registered webhooks keep working
- `conversationKey` → `instanceId`, `parseConversationKey` → `parseInstanceId`, no aliases; error classes renamed. **Zendesk's `ticketKey`/`parseTicketKey` keep their names.**
- Connectors now pass structured facts as `initialData` — prefer `useInitialData()`; `parseInstanceId()` is an escape hatch
- Hand-written channels use `createChannelRouter(routes)` from `@flue/runtime`

## Database

- Adapters take your **driver**, not a connection string: `postgres(process.env.DATABASE_URL!)` → `postgres({ query, transaction, close })`
- `db.ts` moves from `.flue/db.ts` to the source root (`src/db.ts`); `start()` scripts take `db:` directly
- Custom adapters: `RunStore` and `EventStreamStore` deleted; `AgentSubmissionStore` gained settlement and lease methods; contract test suites ship in `@flue/runtime/test-utils`

## Providers

Providers are now Pi objects registered with `setProvider()`; `registerProvider()`, `registerApiProvider()`, and `ProviderRegistrationError` are removed. Add `@earendil-works/pi-ai`.

- `registerProvider('ollama', {...})` → `setProvider(createProvider({ id, auth, models, api }))` with full `Model` objects (no catalog hydration)
- Patch a built-in by registering under its ID and reusing its models: `models: anthropicProvider().getModels().map(m => ({ ...m, baseUrl }))`
- `apiKey` → `auth.apiKey.resolve()`
- `registerProvider('cloudflare', {...})` → `cloudflareBindingProvider({ binding, gateway })` from `@flue/runtime/cloudflare/workers-ai`
- `storeResponses` and `telemetry` overrides removed with no replacement
- Tests: `registerFauxProvider(...)` → `fauxProvider(...)` + `setProvider(faux.provider)`; there is no `.unregister()`

## Observability

- `run_start`/`run_end` → `agent_start`/`agent_end`
- `runId` → `instanceId` + `submissionId`
- Polling `getRun()` → `submission_settled` events plus your own state
- `createOpenTelemetryObserver()` → `createOpenTelemetryInstrumentation()`, registered with `instrument(...)`; `exportContent` became the instrumentation-wide `content` policy

## Clients and React

```ts
// beta
const client = createFlueClient({ baseUrl: '/api' });
await client.agents.send('support-assistant', ticketId, { message });

// now
const conversation = createFlueClient({ url: `/api/agents/support-assistant/${ticketId}` });
await conversation.send({ message, initialData });
await conversation.abort();   // aborts the conversation; there is no per-submission abort
```

`@flue/react` exports only `useFlueAgent`. `FlueProvider` and `useFlueWorkflow` are removed. **Memoize any client passed as `useFlueAgent({ client })`** — a new instance replaces the session.

## Cloudflare deployments

- **`FlueRegistry` is gone with no replacement.** Append a `deleted_classes` migration for it and every `Flue<Name>Workflow` class.
- Generated classes are per-agent: `export function Triage()` → class `FlueTriageAgent`, binding `FLUE_TRIAGE_AGENT`
- Your `wrangler.jsonc` is never modified; the build merges into gitignored `.flue-vite.wrangler.jsonc`, and `vite build` writes the finalized config into `dist/`. Deploy with plain `wrangler deploy` from the project root.
- `cloudflare.ts` moves from `.flue/cloudflare.ts` to `src/cloudflare.ts`
- Update `run_worker_first` from `["/api/*", "/_flue/*"]` to your actual mounts, e.g. `["/api/*", "/agents/*", "/channels/*"]`
- Minimum `compatibility_date` is **`2026-04-01`**, validated at build

## Checklist

1. **Pins** — replace `@flue/*@1.0.0-beta.x`; add `vite`, `@flue/vite`, `hono`, and (Cloudflare) `@cloudflare/vite-plugin`. Drop beta patches and vendored builds.
2. **Build** — author `vite.config.ts` (`flue()` before `cloudflare()`); scripts to `vite dev`/`vite build`; fix `flue.config.ts`; gitignore generated files.
3. **Routing** — explicit mounts in `app.ts`; delete `flue()` router usage; mount each channel's `route()`; decide which agents are dispatch-only.
4. **Agents** — convert initializers to exported capitalized functions in `'use agent'` modules; hooks for behavior, statics for contract, `AgentProps` for the id. Convert profiles to `useSubagent` agent functions. Agents that relied on the implicit sandbox must declare one.
5. **Tools** — `run({ input })` → `run({ data })`; adopt `harness: true`; consider `durable: true`.
6. **Skills** — delete import attributes; let `SKILL.md` imports package themselves; wrap other markdown with `defineSkill`.
7. **Workflows** — replace each with the smallest fit.
8. **Channels and database** — rename `conversationKey`/`parseConversationKey`; rewrite adapters around your own driver; move `db.ts` to the source root.
9. **Providers** — `createProvider()` + `setProvider()`; `cloudflareBindingProvider()`; optionally narrow with `flue({ providers: [...] })`.
10. **Observability** — migrate `run_*` to `agent_start`/`agent_end`/`submission_settled`, `runId` to `instanceId`/`submissionId`.
11. **Clients** — conversation-scoped clients and `useFlueAgent({ url | client })`.
12. **Deployment** — `deleted_classes` for `FlueRegistry` and workflow classes; `new_sqlite_classes` for new agents; plan the drained deployment.
13. **Verify** — typecheck, tests, a production `vite build`, and inspect the built artifact (exports, merged wrangler config) before deploying.
