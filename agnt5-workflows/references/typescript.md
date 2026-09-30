# Workflows and functions in TypeScript

Verified against `@agnt5/sdk` **0.10.5**. Read the Python SKILL.md first; this file maps each
section to the TypeScript API and lists what TypeScript does not have.

## Defining a workflow

```typescript
import { workflow } from '@agnt5/sdk';
import type { Context } from '@agnt5/sdk';
import { createAccount, sendWelcomeEmail } from './functions.js';

export const onboardingWorkflow = workflow(
  'onboarding_workflow',
  async (ctx: Context, input: { userEmail: string }) => {
    const account = await ctx.step('create_account', () =>
      createAccount(ctx, { userEmail: input.userEmail }),
    );
    await ctx.step('send_welcome_email', () => sendWelcomeEmail(ctx, { accountId: account.id }));
    return { status: 'done', accountId: account.id };
  },
  { inputSchema: { type: 'object', properties: { userEmail: { type: 'string' } }, required: ['userEmail'] } },
);
```

`workflow(name, handler, options?)`. The handler is `(ctx: Context, input)` — one input object
(the JSON the caller sends), not spread parameters. Registration happens when the module is
evaluated, so `app.ts` must `import './src/workflows.js'` (with the `.js` suffix under ESM).

| Python `@workflow(...)` | TypeScript `WorkflowOptions` |
|---|---|
| `name=` | `name` (overrides the positional name) |
| `cron="0 9 * * *"` | `cron: '0 9 * * *'` |
| `triggers=[event(...), webhook(...)]` | `triggers: [event('user.signed_up'), webhook('stripe', { event: 'payment_intent.succeeded' })]` |
| `chat=True` | none — see "Not available" |
| — | `inputSchema` / `outputSchema` (JSON Schema; TS types are erased, so Studio only shows fields you declare) |

## Defining a function

```typescript
import { fn } from '@agnt5/sdk';
import type { Context } from '@agnt5/sdk';

export const sendEmail = fn('send_email')
  .retry({ maxAttempts: 3, initialIntervalMs: 1000, maxIntervalMs: 60000 })
  .backoff({ type: 'exponential', multiplier: 2 })
  .timeout(10_000)
  .inputSchema({
    type: 'object',
    properties: { to: { type: 'string' }, subject: { type: 'string' }, body: { type: 'string' } },
    required: ['to', 'subject', 'body'],
  })
  .run(async (ctx: Context, input: { to: string; subject: string; body: string }) => {
    ctx.logger.info('Sending email', { to: input.to, attempt: String(ctx.attempt) });
    return `Sent to ${input.to}`;
  });
```

**`ctx.logger` attribute values must be strings.** The worker hands them to a native logger
that throws `StringExpected` (Failed to convert JavaScript value ... into rust type String) for
numbers, booleans and objects, and the throw fails the run. The `meta` type is
`Record<string, any>`, so `tsc` does not catch it: wrap values in `String(...)` or
`JSON.stringify(...)`.

Builder methods (all before `.run`): `retry(RetryPolicy)`, `backoff(BackoffPolicy)`,
`timeout(ms)`, `inputSchema(JSONSchema)`, `outputSchema(JSONSchema)`, `flowControl(...)`,
`priority(n)`, `maxConcurrency(n)`. No bare-number `retry(3)` shorthand: `RetryPolicy` is
`{ maxAttempts?, initialIntervalMs?, maxIntervalMs? }` (defaults 3 / 1000 / 60000),
`BackoffPolicy` is `{ type: 'constant' | 'linear' | 'exponential', multiplier? }`.

`Context` gives you `ctx.runId`, `ctx.attempt` (0 = first try), `ctx.logger.info(msg, meta)`
(string values only), `ctx.signal` (AbortSignal), `ctx.sleep(ms)`. There is one `Context` type
for functions and workflows; the `FunctionContext` / `WorkflowContext` split does not exist.

## Steps: the unit of durable work

`ctx.step(name, () => ..., { key })` is the only checkpointed call. A direct `myFn(ctx, input)`
inside a workflow is **not** checkpointed and runs again on every replay (HITL resume, durable
sleep, crash recovery). The product docs page that says `fn(...).run(...)` calls
checkpoint automatically is wrong for 0.10.5.

| Call style | Checkpointed | Use when |
|---|---|---|
| `await ctx.step('charge', () => chargeCustomer(ctx, input), { key: orderId })` | Yes | Always, inside a workflow |
| `await chargeCustomer(ctx, input)` | No | Outside a workflow, unit tests |

- The step name is required. Reusing a name in a loop or in `Promise.all` needs a distinct
  `key` per call, or replay matches the wrong checkpoint.
- The callback must return JSON-serialisable data (`undefined` inside the result is dropped).
- `fn().retry()` is applied when the function is the top-level component of a run, but **not**
  when it runs inside `ctx.step`. Retry inside the step yourself:

```typescript
import { executeWithRetry } from '@agnt5/sdk';

const charge = await ctx.step(
  'charge',
  () => executeWithRetry(() => chargeCustomer(ctx, { orderId }), {
    retryPolicy: { maxAttempts: 3 },
    backoffPolicy: 'exponential',
    context: ctx,
  }),
  { key: orderId },
);
```

`ctx.run()` and `ctx.task()` do not exist in TypeScript.

## Running in parallel

There is no `ctx.parallel` / `ctx.gather` / `ctx.batch` / `ctx.map`. Use `Promise.all` over
`ctx.step` calls, each with its own name or key:

```typescript
const [sales, inventory, customers] = await Promise.all([
  ctx.step('fetch_sales', () => fetchSales(ctx, { reportId })),
  ctx.step('fetch_inventory', () => fetchInventory(ctx, { reportId })),
  ctx.step('fetch_customers', () => fetchCustomers(ctx, { reportId })),
]);

// named results
import { gather } from '@agnt5/sdk';
const data = await gather({
  revenue: ctx.step('revenue', () => fetchRevenue(ctx, {})),
  users: ctx.step('users', () => fetchActiveUsers(ctx, {})),
});

// many items with a concurrency cap (no batch()/map() helper exists)
const results: Doc[] = [];
for (let i = 0; i < docIds.length; i += 10) {
  const slice = docIds.slice(i, i + 10);
  results.push(
    ...(await Promise.all(
      slice.map((docId) => ctx.step('process_document', () => processDocument(ctx, { docId }), { key: docId })),
    )),
  );
}
```

`parallel([...])` and `gather({...})` from `@agnt5/sdk` are thin wrappers over `Promise.all`;
they add no checkpointing. `batchExecute`, `fanOut`, `race`, `withTimeout`, `retryWorkflow`
and `executeChildWorkflow` run child *workflows* in-process on the parent's `ctx` (no separate
run) and `withTimeout` leaks its timer; prefer `ctx.step` + `Promise.all`.
`client.batch()` is a gateway call from outside a workflow, not in-workflow fan-out.

## Durable sleep

```typescript
await ctx.step('send_confirmation', () => sendConfirmation(ctx, { userId }));
await ctx.sleep(24 * 60 * 60 * 1000, 'wait_24h');   // milliseconds, not seconds
await ctx.step('send_follow_up', () => sendFollowUp(ctx, { userId }));
```

The sleep is durable only when the managed runtime negotiates `durable_suspension_v1`; the
local `ContextImpl` falls back to `setTimeout`. Give every sleep a name. Resume replays the
workflow from the top, so everything before the sleep must be inside `ctx.step`.
`sleep(ctx, ms, name)` from `@agnt5/sdk` is the same call. While it sleeps the run reports
`status: 'paused'` (like a `waitForUser` question) and `client.run()` returns at the sleep;
see `agnt5-human-in-the-loop` for telling the two apart.

## State

Only run-scoped state exists, directly on `ctx`:

```typescript
await ctx.set('phase', 'started');
const phase = await ctx.get<string>('phase', 'unknown');
await ctx.delete('phase');
```

No `ctx.state`, `ctx.session`, `ctx.user`, `ctx.memory`, `ctx.conversation`. The exported
`StateManager` / `SessionContext` / `UserContext` / `ScopedState` classes take a
`MemoryStateAdapter` and live only in the worker process; `ConversationMemory` and
`SemanticMemory` are standalone and in-memory by default (see `agnt5-agents-tools`).

## Idempotent side effects

```typescript
export const chargeCustomer = fn('charge_customer').run(
  async (ctx: Context, input: { orderId: string; total: number }) => {
    const key = ctx.activation?.idempotencyKey ?? `charge:${input.orderId}`;
    return stripeCharge(input.orderId, input.total, { idempotencyKey: key });
  },
);
```

`ctx.activation` (`{ activationId, attempt, idempotencyKey }`) is set while a durable step,
tool or model call is executing; it is `undefined` for a bare call, so keep the fallback.

## Triggers

```typescript
export const dailyReport = workflow('daily_report', async (ctx: Context) => ({ ok: true }), {
  cron: '0 9 * * *',
});
```

`event(name, { triggerId?, filterExpression?, inputMapping?, batchWindowMs?, delayExpression? })`
and `webhook(source, { event, ...same })` return a `TriggerSpec`. The input a triggered workflow
receives is an event envelope, not the raw body — see `agnt5-webhooks-integrations`
`references/typescript.md`. HITL pauses: `agnt5-human-in-the-loop`.

## Registering with the worker

Functions, workflows and tools register on import. Agents do not: call
`worker.registerAgents([...])` before `worker.run()`. `autoRegister` is typed on
`PlatformWorkerOptions` but never read.

```typescript
import { Worker } from '@agnt5/sdk';
import './src/functions.js';
import './src/workflows.js';

process.on('unhandledRejection', (err) => console.error('unhandledRejection', err)); // one stray rejection must not exit the worker

const worker = new Worker('my-service', { serviceVersion: '0.1.0' });
await worker.run();
```

## Not available in TypeScript

- `ctx.parallel`, `ctx.gather`, `ctx.batch`, `ctx.map`, `ctx.run`, `ctx.task`
- `chat=True` workflows (multi-turn is handled by the caller via `client.workflow(name).chat(message, sessionId)`; the workflow itself sees one run per message)
- `ctx.state` / `ctx.session.state` / `ctx.user.state`, `ctx.memory`, `ctx.conversation`
- `retries=3` / `backoff="exponential"` shorthands (always pass objects)
- Sync handlers wrapped in a thread pool (every handler is async or returns a value directly)
- `ctx._is_replay`
- Trace spans for any run: use `ctx.logger` and events instead

## TypeScript pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| A charge/email repeats after a HITL resume or sleep | direct `fn(ctx, ...)` call in the workflow body | wrap in `ctx.step(name, ..., { key })` |
| `.retry()` ignored inside a workflow | retry only applies to top-level function runs | `executeWithRetry` inside the step |
| `agnt5 run <function>` exits 1 while the run later succeeds | CLI prints the first failed attempt; platform keeps retrying | check `agnt5 inspect runs describe <runId>` |
| Worker process exits mid-run | unhandled promise rejection | `process.on('unhandledRejection', ...)` in `app.ts` |
| Every failure shows `EXECUTION_ERROR` | worker maps all errors to one code | log `err.name` / `err.constructor.name` yourself |
| Run fails with `StringExpected` from a log line | non-string value in `ctx.logger` meta | `String(value)` / `JSON.stringify(value)` |
| `Promise.all` of steps replays the wrong values | same step name, no key | give each call a `key` |
| Studio shows no input fields | TS types are erased | `inputSchema` on `fn()` and `workflow()` |

## Source

https://agnt5.com/docs/build/workflows · https://agnt5.com/docs/build/functions (TypeScript tabs)
