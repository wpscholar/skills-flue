# Sandboxes in Flue

Live source: `flue docs read guide/sandboxes` · `flue docs read reference/sandbox-api`

A sandbox is an execution environment attached to an agent: a filesystem and a shell. An agent doesn't have one unless you give it one.

## What a sandbox adds

- **File and shell tools** — `read`, `write`, `edit`, `bash`, `grep`, `glob`. (A sandbox can replace this set with its own.)
- **Workspace context** — at initialization Flue composes into the system prompt: the working directory path, a directory listing, and the contents of **`AGENTS.md`** when present.
- **Workspace skills** — directories under `<cwd>/.agents/skills/`, discovered at the same time.
- **Subagents** — delegates share the parent's environment. A `task` call can scope a child to a different working directory, but never to a different sandbox.
- **Your application code** — harness tools reach the same environment as `harness.sandbox`.

Without a sandbox: no file/shell tools, no workspace in the prompt, and `harness.sandbox` **throws**. Choose the narrowest environment that supports the task.

## Attaching

```ts
useSandbox(factory);
useSandbox(factory, { cwd: '/srv/checkouts/flue' });
```

- **At most once per render.** A second call in the same render throws. It also throws inside a subagent render.
- **The factory is lazy.** Expensive work happens inside `createSandbox()`, called once when the runtime initializes the agent — never on re-renders.
- **The factory receives the agent instance id**, so adapters can key provider resources on it.
- **`cwd`** is resolved once at initialization against the sandbox's base directory. It determines where commands run and where workspace discovery happens.

## The virtual sandbox (just-bash)

In-memory filesystem plus emulated bash, implemented entirely in TypeScript ([just-bash](https://github.com/vercel-labs/just-bash)). Most of the standard unix toolbox works — `ls`, `sed`, `awk`, `jq`, `sort`, pipes, redirects — plus `curl` for HTTP. **No real process is ever spawned.**

```ts
'use agent';
import { bash, useModel, useSandbox } from '@flue/runtime';
import { Bash, InMemoryFs } from 'just-bash';

export function ScratchWorker() {
  useModel('anthropic/claude-haiku-4-5');
  useSandbox(bash(() => new Bash({ fs: new InMemoryFs() })));
  return 'Fetch, reshape, and summarize the data the user points you at.';
}
```

Two defining properties:

- **Isolated from the host.** The model cannot reach your host filesystem, processes, or environment variables. **Network is opt-in**: pass `network: { allowedUrlPrefixes: [...] }`, or `dangerouslyAllowFullInternetAccess: true`.
- **Ephemeral.** The filesystem starts empty and is rebuilt fresh each time the runtime initializes the agent for new work. Files written while processing one message are gone by the next. Keep durable knowledge in persistent state; use a real sandbox when files must last.

Seeding files, allowlisting network:

```ts
useSandbox(
  bash(() => new Bash({
    fs: new InMemoryFs({ '/data/catalog.csv': exportCatalogCsv() }),
    network: { allowedUrlPrefixes: ['https://api.example.com/'] },
  })),
);
```

Harness staging pattern — write input in, read output out:

```ts
harness: true,
async run({ harness, data }) {
  await harness.sandbox.writeFile('document.md', data.document);
  await harness.prompt('Review document.md and write your findings to review.md.');
  return { output: { review: await harness.sandbox.readFile('review.md') } };
}
```

## The local sandbox (Node target)

```ts
import { local } from '@flue/runtime/node';

useSandbox(local({ env: { GH_TOKEN: process.env.GH_TOKEN } }));
```

`local()` binds the agent directly to the host: real filesystem, real processes via the host shell. **There is no isolation, by design.** For development tools, CI tasks, coding agents, self-hosted automation. **Do not use it as an isolation boundary for untrusted requests or multiple tenants.**

```ts
function local(options?: { cwd?: string; env?: Record<string, string | undefined> }): SandboxFactory;
```

**Environment gotcha:** the model's shell does **not** inherit your process environment. Only a short allowlist of shell essentials passes by default — `PATH`, `HOME`, `USER`, `LANG`, `TERM`, `TMPDIR` and the like — and never API keys, tokens, or cloud credentials. Anything else is an explicit per-variable opt-in through `env`; set a key to `undefined` to drop one of the defaults. `env: { ...process.env }` hands the model's shell your entire host environment, secrets included. **The snapshot is taken once when the sandbox is constructed** — later changes to `process.env` are not picked up.

Before widening the shell's credentials, consider whether a narrow application tool can perform the privileged action instead.

## Remote sandboxes (adapters)

An adapter is a small file in your project wrapping a provider's SDK into the sandbox factory contract.

```bash
flue add sandbox e2b        # creates <source-dir>/sandboxes/e2b.ts and installs the SDK
flue add sandbox <docs-url> # build-from-scratch guide against the Sandbox Adapter API
```

Catalog: boxd, Cloudflare Computer, Cloudflare Sandbox, Daytona, E2B, exe.dev, islo, Mirage, Modal, Vercel Sandbox, smolvm.

> Adapters are deliberately thin: your application creates, reuses, and deletes provider sandboxes; Flue only connects to what you hand it and **never destroys provider infrastructure**.

```ts
useSandbox({
  async createSandbox(options) {
    const client = new Daytona();
    const sandbox = await client.create();
    return daytona(sandbox).createSandbox(options);
  },
});
```

`createSandbox({ id })` receives the agent instance id. **A factory that looks up an existing provider sandbox by that id before creating one gives each conversation a durable workspace** that survives across messages and process restarts.

**Cancellation gotcha:** cancelling a running command — the model's own `timeout`, or your application aborting the surrounding task — always rejects promptly. `local()`'s process-group kill actually stops the command, but **most provider SDKs have no mid-flight cancellation: the remote command keeps running in the background after the rejection**, and its eventual output is discarded rather than appearing later in the conversation. Vercel and Mirage stop the command for real.

## Sandbox-provided tools

A sandbox factory may carry a `tools` function. When present it **replaces** the framework's default model-facing tool set for that agent — an exec-less adapter might keep `read`/`write`/`edit` but swap shell-backed `bash`/`grep`/`glob` for its own executor tool. Adapters compose these sets from exported per-tool factories (`createReadTool`, `createBashTool`, …) rather than rebuilding from scratch.

**Because capabilities vary this way, check an integration's documentation before assuming ordinary file or command tools are available.**

## Conditional attachment

```ts
if (investigating) {
  useSandbox(local({ cwd: '/srv/support/repro' }));
}
```

- Presence is read at initialization and again at **every turn boundary**. When it flips, the environment swaps before the next model call. Detaching removes the environment; file/shell tools drop with it and nothing carries over either way.
- The model is told with a single **`environment` signal** restating the complete current state — new working directory plus the full tool, skill, and subagent rosters — and warning that files and results from the previous environment may no longer be accessible.
- **The system prompt stays frozen**, still describing the workspace discovered at initialization, until the next compaction re-discovers against the current environment.
- A swap rewrites the native tools array and **invalidates the provider's prompt cache**.
- **Only presence is observable.** Factories are fresh objects every render, so replacing sandbox A with sandbox B while staying attached does not swap mid-run — it takes effect when the next submission initializes.
- Because the condition lives in persistent state it replays durably, and an adapter keyed on the instance id resolves back to the same workspace.

## Durability

Sandbox state is **deliberately not durable**. The conversation database does not store sandbox files, and the virtual sandbox is rebuilt fresh on every initialization — including on a recovered attempt, so files an interrupted attempt wrote are gone on resume. Durable workspaces come only from an adapter that keys the provider workspace on the agent instance id.

Keep knowledge the agent must not lose in **persistent state**; keep files that must last in a **durable workspace**. Conversation persistence and workspace persistence are independent choices.

## Cloudflare options

- **Cloudflare Sandbox** — container-backed Linux for git, package installs, native binaries, real filesystems. Export the sandbox DO class from `cloudflare.ts`, declare its binding and container image in `wrangler.jsonc`, then wrap the RPC stub with `cloudflareSandbox(getSandbox(env.Sandbox, id))` from `@flue/runtime/cloudflare`.
- **Cloudflare Computer** (`flue add sandbox cloudflare-computer`) — a durable, SQLite-backed `Workspace` inside the agent's own Durable Object, with shell execution via a just-bash Dynamic Worker. Standard tools remain, but commands run in a **JavaScript shell**: coreutils and text tools work; **native binaries and package managers do not**.

Use Computer when a durable workspace and shell-expressible work are enough; use Sandbox when you need a full Linux environment.
