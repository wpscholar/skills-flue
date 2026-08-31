# Durability and recovery in Flue

Live source: `flue docs read guide/durability`

## The accepted-work contract

Every input — a direct HTTP prompt, `dispatch(...)`, an `init()` handle's `dispatch(...)`, a channel delivery, a scheduled trigger — is admitted as a **submission**: the payload is recorded durably *before* any model work begins.

> Every accepted submission reaches exactly one durable terminal outcome — `completed`, `failed`, or `aborted` — no matter how many crashes happen in between.

The outcome is written as a `submission_settled` record in the canonical stream. The SDK's `wait()` resolves or rejects from that record; an awaited `init().read(...)` resolves with the settled reply or rejects with the settled error.

- **Queueing.** Submissions for one conversation form a durable queue processed in admission order, one at a time. A message arriving while busy either joins the live response at a turn boundary or waits; a delivery that misses the live response runs as its own submission.
- **Attempts.** A coordinator claims the submission, runs it, settles it. An interruption consumes the attempt; recovery claims a new one, up to the retry budget.
- **Aborts.** `POST /:id/abort` (or the SDK's `abort()`) records a durable abort intent on every unsettled submission; each settles with the distinct `aborted` outcome, even when the process running the work is already gone. An abort arriving after a finished response settles `completed`.

## Recovery after an interruption

**Converge:** any partially streamed assistant output the dead attempt persisted is closed out as an aborted entry — unconditionally and idempotently, so no crash shape can leave the conversation looking mid-stream. The partial output stays preserved in history.

**Classify:**

| Durable evidence after the input | What recovery does |
|---|---|
| The input was never persisted | Requeues the submission for a clean first attempt. |
| A completed assistant response | Settles `completed` — finished work is never discarded, even past the retry budget. |
| A partial response with text or reasoning | Tells the model its stream was interrupted and continues from the durable partial. |
| A tool turn with unresolved calls | Repairs the tool batch (below), then continues the turn loop. |
| A transient provider error (rate limit, outage) | Retries the turn after a backoff, under a bounded error budget. |
| A context-overflow response | Compacts the conversation and retries the turn. |
| A durable abort intent | Settles `aborted`. |

**Tool-batch repair is deliberately conservative.** Recorded results are preserved exactly — those calls never run again. An unresolved *ordinary* call is **not** re-executed, because the runtime cannot know which side effects already happened; it settles with an explicit unknown-outcome error the model sees and can react to. Two exceptions resolve real outcomes: `durable: true` tools re-execute with completed steps replaying from records, and in-flight delegated tasks resume from their own transcripts.

**The overall discipline: at-least-once execution over exactly-once recording.**

> Work that committed durably — recorded responses, recorded tool results, committed state writes — never re-runs. Work that was interrupted before committing re-runs on the next attempt, which includes your **event hook** callbacks: their durable effects commit atomically and never duplicate, but an external side effect inside one (an email, a page) may rarely happen twice.

A recovered conversation always comes to rest in a state where the next message processes normally — an interrupted submission cannot wedge the queue behind it.

## Retry budget and timeout

Exhausting attempts or exceeding the wall-clock timeout settles the conversation to a rest state, lands a `submission_interrupted` advisory in the timeline, and settles the submission `failed` — waiters reject with the structured error, including which tool calls were left with unknown outcomes.

The timeout is enforced **preemptively**, not just between attempts:

- The coordinator supervises running attempts on its wake cadence; at the deadline it fires the attempt's abort signal.
- **A `run` that ignores its signal is abandoned rather than awaited.**
- An attempt hung below the abandonable layer is settled `failed` over the hung fiber after a short grace, its late writes fenced off. A hang delays settlement by at most the deadline plus that grace; it can never strand the submission.
- A model stream silent past its **idle timeout** fails as a transient provider error, and the turn retries under the error budget.

**Defaults: 10 attempts, 1 hour per submission.**

```ts
IssueTriage.durability = { maxAttempts: 5, timeoutMs: 7_200_000 };
```

> The static is applied by the platform while the agent function is *not* running, so it stays in force after a crash — including a crash in the agent's own render. The timeout is the total wall-clock budget from the first attempt's start; turn-boundary joins and response continuations do not extend it.

## Durable tools

`durable: true` + `step.do(name, fn)`. Each completed step durably records its returned value before resolving; on recovery the whole call re-runs, completed steps return recorded values without executing, and execution continues from the first unfinished step. Step records are keyed to the tool call id, so they carry across attempts of the same call and are scoped to it.

- **Exactly-once-recorded, at-least-once-executed.** A crash in the window between a step's function finishing and its record landing re-runs that one step — steps around external effects should be individually idempotent.
- **A redeploy can withdraw the contract.** If recovery finds the current render no longer declares the tool, or no longer marks it `durable`, the call falls back to the ordinary interrupted-marker path rather than guessing.

## Delegated tasks

A subagent task runs as a child session with its own durable conversation stream. Recovery repairing a tool batch with an unresolved `task` call does not settle it with a marker — it reattaches to the child's durable transcript, resumes the child to completion under the same recovery rules, and commits the child's real final result as the parent's tool outcome. This recurses, and several tasks interrupted in one parallel batch are all resumed before the batch commits.

A delegate has no durability configuration of its own: resumed child work runs inside the parent's attempt, under the parent's retry budget and timeout.

- **A delegate removed or renamed by a redeploy cannot be resumed under any retry** — that one call settles with an error outcome and the parent continues.
- The interrupted marker carries the child's conversation id, so the transcript remains inspectable.

## Persisted state

Every `usePersistentState` write is a record in the canonical stream, and **a write becomes durable atomically with the unit of work that made it**. A write from a tool commits with that turn's tool batch; a write from an event hook commits with the hook seam's checkpoint. If recovery settles the batch as interrupted, the write never happened.

> That atomicity is what makes persistent state the correct guard for at-least-once callbacks: a `sent` flag set by the same unit of work that sent the email cannot end up `true` while the work it guarded rolled back.

## What is deliberately NOT durable

1. **Sandbox / workspace state.** The conversation database does not store sandbox files. The virtual sandbox is ephemeral by design and a recovered attempt re-initializes it, so files an interrupted attempt wrote are gone on resume. Durable workspaces come from a sandbox adapter that keys the provider workspace on the agent instance id. Keep knowledge the agent must not lose in persistent state; keep files that must last in a durable workspace.
2. **In-flight local promises.** `init(...).read(...)` and `client.wait(...)` are not durable. If the awaiting process exits, the accepted work continues; only the local promise is gone. Persist the `DispatchReceipt` that `dispatch(...)` resolved with — for example as a workflow step's durable result — and `read(receipt)` re-attaches from any process, resolving immediately if the submission already settled.
3. **Code outside the agent.** Flue does not checkpoint arbitrary TypeScript execution. The checkpoint boundary is the agent itself. Outside it, use a workflow engine and treat Flue like any other service.
4. **External side effects.** Flue records that a tool ran and what it returned — never the effect itself. Recovery never blindly repeats uncertain effectful work, but at-least-once execution means an effect at the boundary can repeat. Design external effects to be idempotent, key them on stable ids like `toolCallId` or `step.do` names, and guard one-shot actions with persistent state.

## Orchestrating from outside

Flue guarantees the agent, not the script around it: a workflow that dies between two dispatches re-runs from its start. No special integration is required with Cloudflare Workflows, Inngest, or Temporal — call Flue like any other service, and **put the dispatch in its own step** so the receipt is checkpointed the moment it exists, with the read in a second step.

```ts
const agent = init(Reviewer, { id: `nightly-${date}` });
const receipt = await step.do('dispatch review', () => agent.dispatch(prompt));
const review = await step.do('read review', async () => {
  const reply = await agent.read(receipt);
  return { text: reply.text, data: reply.data };
});
```

- **Step timeouts compound.** Splitting also splits any per-step time bound: a 20-minute step timeout becomes up to 40 minutes end-to-end. If the operation carries one deadline, checkpoint it in the dispatch step's result and have the read step enforce the remainder.
- **The one remaining crash window** is inside the dispatch step itself — the send was admitted but the step died before checkpointing the receipt. The engine re-runs the step, which sends again, and the instance's **send condition** decides what that means: an unconditional send (no `uid`) delivers the duplicate, which joins the live response at a turn boundary so both submissions settle with the same coalesced reply; a create-only send (`uid: null`) rejects the duplicate at admission with `AgentInstanceExistsError`, which is the workflow's signal to fail or fall back.
