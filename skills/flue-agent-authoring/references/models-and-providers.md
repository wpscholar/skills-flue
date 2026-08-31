# Flue models and providers

Live source: `flue docs read guide/models` · `flue docs read reference/provider-api`

Flue's model layer is [Pi](https://pi.dev/). `useModel()` is a declaration, not a client — it returns nothing, and no API key passes through agent code.

## Specifiers

`'provider-id/model-id'` — everything up to the **first** `/` is the provider; the rest is the provider's own model ID and may itself contain slashes.

```
anthropic/claude-sonnet-4-6
openai/gpt-5.5
openrouter/moonshotai/kimi-k2.6
cloudflare/@cf/moonshotai/kimi-k2.6
```

Default provider set is Pi's built-in catalog: `anthropic`, `openai`, `google`, `amazon-bedrock`, `google-vertex`, `groq`, `mistral`, `xai`, `deepseek`, `cerebras`, `together`, `fireworks`, `openrouter`, and more (`https://pi.dev/docs/latest/providers`).

Catalog metadata per model — wire protocol, endpoint, context window, output-token limit, cost rates, reasoning support, accepted input modalities — decides when compaction triggers, whether a thinking level reaches the wire, and whether images can be attached. An unknown specifier fails fast, before any request is sent.

## Trimming the build

```ts
// vite.config.ts
flue({ providers: ['anthropic', 'openai'] })
```

**The list is exhaustive** — a specifier naming any other provider fails at resolution. On the Cloudflare target include `'cloudflare'` when agents use `cloudflare/...` models. Omit the field to keep the full set.

## Credentials

```
ANTHROPIC_API_KEY="sk-ant-..."
OPENAI_API_KEY="sk-..."
GEMINI_API_KEY="..."          # note: google, not GOOGLE_API_KEY
GROQ_API_KEY="..."
```

Shell-exported values win over file values. Deployed servers read only the real environment. For credentials that are not environment variables, use a custom provider's `auth.apiKey.resolve()` — it runs per request, so rotation works.

## Changing models mid-conversation

```ts
export function Reviewer() {
  const [escalated, setEscalated] = usePersistentState('escalated', false);
  useModel(escalated ? 'anthropic/claude-opus-4-6' : 'anthropic/claude-haiku-4-5');

  useTool({
    name: 'escalate_review',
    description: 'Escalate when the change is too complex for a quick pass.',
    async run() {
      setEscalated(true);
      return 'Escalated. A stronger model will take over.';
    },
  });

  return 'Review the proposed change and leave actionable feedback.';
}
```

**Submission-scoped.** The response where `escalate_review` fires finishes on the cheap model; the conversation's next message runs on the strong one.

## Custom providers

Requires `npm install @earendil-works/pi-ai`. Register at module top level in `app.ts`, **before any agent runs** — but note that `flue run` never loads `app.ts`, so put the registration in the agent module when it must work under the CLI. Registrations key on the provider `id`; each call replaces that ID's previous provider, including a built-in.

```ts
// src/app.ts — a local Ollama server
import { createProvider, envApiKeyAuth } from '@earendil-works/pi-ai';
import { openAICompletionsApi } from '@earendil-works/pi-ai/api/openai-completions.lazy';
import { setProvider } from '@flue/runtime';

setProvider(
  createProvider({
    id: 'ollama',
    auth: { apiKey: { name: 'Ollama (keyless)', resolve: async () => ({ auth: {} }) } },
    models: [
      {
        id: 'llama3.1:8b',
        name: 'Llama 3.1 8B (local)',
        api: 'openai-completions',
        provider: 'ollama',
        baseUrl: 'http://localhost:11434/v1',
        reasoning: false,
        input: ['text'],
        cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
        contextWindow: 128000,
        maxTokens: 8192,
      },
    ],
    api: openAICompletionsApi(),
  }),
);
```

**The runtime trusts your metadata blindly:** `reasoning: false` silently drops a `thinkingLevel`; `input: ['text']` replaces attached images with an `"(image omitted)"` placeholder; `contextWindow: 0` reads as unknown, so threshold compaction cannot engage.

## Gateway / proxy pattern

Re-register under the built-in ID so agent specifiers do not change and catalog metadata rides along:

```ts
import { createProvider } from '@earendil-works/pi-ai';
import { anthropicMessagesApi } from '@earendil-works/pi-ai/api/anthropic-messages.lazy';
import { anthropicProvider } from '@earendil-works/pi-ai/providers/anthropic';
import { setProvider } from '@flue/runtime';

setProvider(
  createProvider({
    id: 'anthropic',
    auth: {
      apiKey: { name: 'Gateway key', resolve: async () => ({ auth: { apiKey: process.env.GATEWAY_KEY } }) },
    },
    models: anthropicProvider().getModels().map((model) => ({
      ...model,
      baseUrl: 'https://gateway.example.com/anthropic',
    })),
    api: anthropicMessagesApi(),
  }),
);
```

## Cloudflare Workers AI (Cloudflare target only)

The `cloudflare` provider ID registers automatically and runs models on Workers AI — no API key, no external account; authorization and billing follow the Worker, including the free tier.

```ts
useModel('cloudflare/@cf/moonshotai/kimi-k2.6');
```

```jsonc
// wrangler.jsonc
{ "ai": { "binding": "AI" } }
```

Everything after `cloudflare/` is passed as the model ID to `env.AI.run(...)` — a Workers AI ID (`@cf/...`) or a gateway-supported ID (`openai/gpt-5.5`).

**Billing gotcha:** `cloudflare/openai/gpt-5.5` bills through Cloudflare; plain `openai/gpt-5.5` uses Flue's direct OpenAI provider and its API key. Completely different paths.

**AI Gateway is on by default** for every `cloudflare/...` call — caching, logging, rate limiting, budget controls. To customize or opt out:

```ts
// src/app.ts
import { setProvider } from '@flue/runtime';
import { cloudflareBindingProvider } from '@flue/runtime/cloudflare/workers-ai';
import { env } from 'cloudflare:workers';

setProvider(
  cloudflareBindingProvider({
    binding: env.AI,
    gateway: { id: 'my-gateway', cacheTtl: 300, metadata: { tenant: 'acme' } },
    // ...or `gateway: false` to bypass AI Gateway entirely.
  }),
);
```

Also available from **any** target as ordinary catalog providers, both authenticating with `CLOUDFLARE_API_KEY`:

- `cloudflare-workers-ai/...` — URL-backed Workers AI
- `cloudflare-ai-gateway/...` — URL-backed AI Gateway

## Testing

Pi's `fauxProvider(...)` from `@earendil-works/pi-ai` plus `setProvider(faux.provider)`. There is no `.unregister()`.
