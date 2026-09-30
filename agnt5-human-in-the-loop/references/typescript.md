# Human-in-the-loop in TypeScript

Verified against `@agnt5/sdk` **0.10.5**. Same section order as the Python SKILL.md.

## `ctx.waitForUser()` parameters

`ctx.waitForUser(question, options?)` → `Promise<string | null>`. The question is positional;
everything else is one options object.

| Option | Default | Description |
|---|---|---|
| `inputType` | `'text'` | `'text' \| 'approval' \| 'select' \| 'multiselect'` (`HITLInputType`) |
| `options` | `undefined` | `HITLOption[]` = `{ id: string; label: string }[]` (required for approval/select/multiselect) |
| `allowCustom` | `false` | Adds a free-text "Something else" choice |
| `skippable` | `false` | Adds a Skip button |

## Input types and what comes back

```typescript
// text
const name = await ctx.waitForUser('What should we call this report?');

// approval — returns the chosen id as a string
const decision = await ctx.waitForUser('Deploy to production?', {
  inputType: 'approval',
  options: [{ id: 'approve', label: 'Approve' }, { id: 'reject', label: 'Reject' }],
});
if (decision === 'reject') return { status: 'cancelled' };

// select — returns the chosen id
const fmt = await ctx.waitForUser('Which output format?', {
  inputType: 'select',
  options: [{ id: 'pdf', label: 'PDF' }, { id: 'markdown', label: 'Markdown' }],
});

// multiselect — the answer is a JSON array *string*, e.g. '["market","tech"]'
const topics = selections(await ctx.waitForUser('Which topics?', {
  inputType: 'multiselect',
  options: [{ id: 'market', label: 'Market' }, { id: 'tech', label: 'Tech' }],
}));
```

Answers submitted through the resume API arrive as strings: multiselect is a JSON-encoded
array (`'["a","c"]'`), a skipped question arrives as the string `"null"` (not only `null`),
approval/select return the id string that was sent. The product docs' comma-separated
multiselect format is not what the platform sends. Normalise once:

```typescript
function skipped(raw: string | null): boolean {
  return raw === null || raw === 'null' || raw === '';
}

function selections(raw: string | null): string[] {
  if (skipped(raw)) return [];
  try {
    const parsed = JSON.parse(raw!);
    if (Array.isArray(parsed)) return parsed.map(String);
  } catch { /* not JSON — fall through */ }
  return raw!.split(',').map((s) => s.trim()).filter(Boolean);
}
```

`skippable: true` → check `skipped(raw)` and use a default. You can call `waitForUser` any
number of times, including inside `if` branches; each executed call gets the next pause index.

## Replay safety (read before generating HITL code)

`waitForUser` throws `WaitingForUserInputError` the first time; the worker reports the pause
and the run suspends. When the user answers, the platform re-dispatches the run and **the
workflow handler runs again from the top**; `waitForUser` finds the stored answer for that
pause index and returns it.

```
First run:   generateDraft() → waitForUser() → throws (paused)
On resume:   generateDraft() → waitForUser() → returns stored answer → publish()
```

**Rule: checkpoint side effects, never guard or catch the pause.**

```typescript
const draft = await ctx.step('draft', () => generateDraft(ctx, { topic }));        // replay returns the cached draft
await ctx.step('notify', () => notifyReviewer(ctx, { draft }));                    // sent once

const decision = await ctx.waitForUser(`Approve this draft?\n\n${draft}`, {        // never wrapped
  inputType: 'approval',
  options: [{ id: 'approve', label: 'Approve' }, { id: 'discard', label: 'Discard' }],
});
```

- A bare `await generateDraft(ctx, ...)` before the pause runs again on every resume
  — this is the single most common TypeScript HITL bug.
- A `try { ... } catch (err) { ... }` around the pause (or around an agent that holds HITL
  tools) swallows the suspension and the run "completes" with your fallback value. If you
  must catch, rethrow pauses:

```typescript
import { isWaitingForUserInput } from '@agnt5/sdk';

try {
  answer = await ctx.waitForUser('...');
} catch (err) {
  if (isWaitingForUserInput(err)) throw err;
  throw err; // or handle non-pause errors
}
```

- There is no `ctx.isReplay` / `ctx._is_replay`; log lines before a pause print twice.
- `waitForUser` has no timeout option: a run waits until someone answers.

## Multi-step HITL with state

```typescript
const name = await ctx.waitForUser('What is your name?', { inputType: 'text' });
await ctx.set('user_name', name);                       // run-scoped, async

const role = await ctx.waitForUser(`Hi ${name}, what is your role?`, {
  inputType: 'select',
  options: [{ id: 'engineer', label: 'Engineer' }, { id: 'manager', label: 'Manager' }],
});
```

`ctx.set/get/delete` are the only state calls (no `ctx.state`). Values written before a pause
are restored on resume together with the checkpoints.

## Agent-level HITL tools

```typescript
import { Agent, LM, workflow, AskUserTool, RequestApprovalTool } from '@agnt5/sdk';
import type { Context, ContextImpl } from '@agnt5/sdk';

export const agentWithHitl = workflow(
  'agent_with_hitl',
  async (ctx: Context, input: { task: string }) => {
    const agent = new Agent({
      name: 'assistant',
      model: LM.openai(),
      modelName: 'openai/gpt-4o-mini',
      instructions: 'Ask for clarification when ambiguous. Request approval before changes.',
      tools: [new AskUserTool(ctx as ContextImpl), new RequestApprovalTool(ctx as ContextImpl)],
    });
    const result = await agent.run(input.task, ctx);     // not inside ctx.step, not inside try/catch
    return { response: result.output };
  },
);
```

| Tool | Model-facing name / args | Behaviour |
|---|---|---|
| `new AskUserTool(ctx)` | `ask_user` `{ question }` | Pauses for a text reply; returns `string \| null` |
| `new RequestApprovalTool(ctx)` | `request_approval` `{ action, details? }` | Pauses with Approve/Reject; returns the chosen id |

Both constructors are typed `ContextImpl`, so cast the workflow `ctx`. Create the agent inside
the workflow handler with the live `ctx`; a module-level agent has no context to suspend. The
tool names are exported from the root `@agnt5/sdk` package (no `agnt5.tool` submodule).

## Answering a pause from your own backend

Same rules as the SKILL.md: the run reports `paused` for a question and for a durable
`ctx.sleep()`, only the newest `workflow.paused` event tells them apart, and resume and cancel
need a key with the `workflow` scope (`--scopes run,workflow`; a `run`-only key gets 403
`INSUFFICIENT_SCOPES`). Unfinished runs are missing from `agnt5 inspect runs ls`; find them
with `GET /v1/runs?component_name=<workflow>`. TypeScript workers emit no
`approval.requested`; the question sits in the `workflow.paused` metadata. `Client` has no
resume method and `client.getEvents()` drops each event's `metadata`, so read the events with
`fetch`:

```typescript
import { Client } from '@agnt5/sdk';

const gatewayUrl = process.env.AGNT5_GATEWAY_URL ?? 'https://gw.agnt5.com';
const headers = { 'X-API-KEY': process.env.AGNT5_API_KEY!, 'Content-Type': 'application/json' };
const client = new Client({ gatewayUrl, apiKey: process.env.AGNT5_API_KEY });

interface GatewayEvent { event_type: string; data?: Record<string, unknown>; metadata?: Record<string, string> }

/** Metadata of the question a paused run waits on; undefined while it sleeps or runs. */
async function pendingQuestion(runId: string): Promise<Record<string, string> | undefined> {
  if ((await client.getStatus(runId)).status !== 'paused') return undefined;
  const res = await fetch(`${gatewayUrl}/v1/runs/${runId}/events`, { headers });
  if (!res.ok) throw new Error(`events: HTTP ${res.status}`);
  const { items } = (await res.json()) as { items: GatewayEvent[] };
  const latest = items.filter((e) => e.event_type === 'workflow.paused').at(-1);
  return latest?.metadata?.pause_reason === 'user_input_required' ? latest.metadata : undefined;
}

async function answer(runId: string, userResponse: string): Promise<boolean> {
  if (!(await pendingQuestion(runId))) return false;   // sleeping, running or finished
  const res = await fetch(`${gatewayUrl}/v1/workflows/resume/${runId}`, {
    method: 'POST', headers, body: JSON.stringify({ user_response: userResponse }),
  });
  return res.ok;
}
```

`client.run()` returns at the first pause with `status: 'paused'`. That response has
`isPending === true`, and `client.waitForResult()` keeps polling a paused run until its timeout,
then throws `RunError`; check `res.status === 'paused'` before waiting. Cancel with
`POST /v1/runs/{runId}/cancel` and an optional `{ reason }` body.

## Edge cases

- **Unexpected text**: `Number(raw)` and check `Number.isNaN`; `raw` may be `null`.
- **Conditional pauses** inside `if` are fine; the pause index counts executed calls only.
- **`ctx.waitForSignal(name)`** is on `Context` but throws `ConfigurationError` in the local
  context and fails on managed workers too. Do not use it for external events;
  poll from a `ctx.step` or start the workflow from a webhook trigger instead. (The serverless
  runtime has its own signal support — see `agnt5-serverless`.)
- **Unit tests**: build a `ContextImpl`, call `setUserResponse(pauseIndex, answer)` for each
  pause, then invoke the handler; `waitForUser` returns the seeded answers instead of throwing.

## Not available in TypeScript

- `ctx._is_replay`
- `ctx.state.set(...)` (sync) — use `await ctx.set(...)`
- `agnt5.tool.AskUserTool` submodule import path (root export instead)
- A timeout on `waitForUser`
- `ctx.waitForSignal` on managed workers
- `ctx.session.state` between pauses (run-scoped `ctx.get/set` only)
- A resume method on `Client` (POST `/v1/workflows/resume/{runId}` with `fetch`)

## TypeScript pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| LLM call / email repeats after the user answers | side effect not in `ctx.step` | `ctx.step(name, ..., { key })` before the pause |
| Run completes with a fallback instead of pausing | `try/catch` swallowed `WaitingForUserInputError` | rethrow with `isWaitingForUserInput(err)` |
| `topics.split(',')` yields `['["a"', '"c"]']` | multiselect answer is a JSON string | `JSON.parse` first (`selections()` above) |
| `note === null` never true after Skip | skip arrives as the string `"null"` | treat `null`, `'null'`, `''` as skipped |
| `new AskUserTool(ctx)` does not type-check | constructor takes `ContextImpl` | `ctx as ContextImpl` |
| Approval never times out | no timeout support | add an operator-side deadline outside the run |
| `waitForSignal` throws | unsupported on this runtime | webhook trigger or polling step |
| Answer lands on the wrong question | resume sent while the run was in a durable sleep | resume only when `pendingQuestion()` returns metadata |
| Resume or cancel returns 403 `INSUFFICIENT_SCOPES` | key has only the `run` scope | create it with `--scopes run,workflow` |

## Source

https://agnt5.com/docs/build/human-in-the-loop · https://agnt5.com/docs/build/tools (TypeScript tabs)
