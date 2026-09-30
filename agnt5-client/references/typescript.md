# TypeScript client reference

Verified against @agnt5/sdk 0.10.5 (`dist/client.d.ts`, `dist/client.js`).

## Construct

```typescript
import { Client } from '@agnt5/sdk';
const client = new Client({
  gatewayUrl: process.env.AGNT5_GATEWAY_URL,   // default is http://localhost:34181 - always set this
  apiKey: process.env.AGNT5_API_KEY,           // falls back to AGNT5_API_KEY
  tenantId: undefined,                         // X-TENANT-ID; falls back to AGNT5_TENANT_ID
  deploymentId: undefined,                     // falls back to AGNT5_DEPLOYMENT_ID (ambient; not used for run())
  timeout: 45_000, maxRetries: 0, retryDelayMs: 1_000,   // retries only for idempotent calls
});
```

## Methods

```typescript
run<T>(component, inputData?, { componentType?, sessionId?, userId?, tenant?, deploymentId?, idempotencyKey?,
                               waitTimeoutMs? /* 300000 */, timeoutMs?, maxRetries? }): Promise<RunResponse<T>>
submit(component, inputData?, { componentType?, tenant?, deploymentId?, idempotencyKey? }): Promise<SubmitResponse>
getStatus(runId): Promise<RunResponse>
getResult<T>(runId): Promise<RunResponse<T>>
getOutput<T>(runId): Promise<T>                       // dereferences output_ref
resolveOutput<T>(result: RunResponse<T>): Promise<T | undefined>
waitForResult<T>(runId, timeoutMs?, pollIntervalMs?): Promise<RunResponse<T>>
waitForOutput<T>(runId, timeoutMs?, pollIntervalMs?): Promise<T | undefined>
stream(component, inputData?, opts?): AsyncGenerator<string>
events(component, inputData?, opts?): AsyncGenerator<ReceivedEvent>   // { eventType, data, contentIndex, sequence, runId? }
getEvents(runId): Promise<{ events: EventRecord[]; runId }>            // { eventType, data, timestamp?, sequence, correlationId? }; no metadata
workflow(name): WorkflowProxy       // run(input, opts) chat(message, sessionId?, opts) submit(input, opts) events(input, opts)
session(sessionType, key): SessionProxy   // chat(message, extra?) getHistory()
entity(entityType, key): EntityProxy       // call(method, args?)
batch(component, items, { maxConcurrency?, ... , componentType?, metadata?, deploymentId?, idempotencyKey? }): Promise<BatchResult>
getBatchStatus(batchId, includeResults?): Promise<BatchStatusResult>
cancelBatch(batchId, reason?): Promise<CancelBatchResult>
eval<T>(component, inputData?, { expected?, scorers?, componentType?, deploymentId?, sessionId?, userId?, timeout? }): Promise<EvalResponse<T>>
batchEval(component, items, { scorers?, expected?, componentType?, deploymentId?, maxConcurrency?, timeout? }): Promise<BatchEvalResult>
```

`RunResponse` is a class: `runId`, `statusCode`, `status`, `output`, `outputRef`, `error`,
`durationMs`, `traceId`, `component`, `createdAt`, `startedAt`, `completedAt`, `failedAt`,
`sessionId`, `metadata`; getters `isSuccess`, `isPending`, `isError`, `elapsed`,
`hasOutputRef`; `raiseForStatus()` throws `RunError`. `RunStatus` is the same string union as
Python. No `resumeWorkflow`/`cancelRun` methods - use `fetch` against the gateway.

A human-in-the-loop question and a durable sleep both report `status: 'paused'`, and `run()`
returns at the first pause. `isPending` is `true` for `paused` (and for `awaiting_input`), and
`waitForResult()` only stops on `completed`/`failed`/`cancelled`/`timeout`, so on a paused run
it polls until `timeoutMs` and throws `RunError`. Branch on `res.status === 'paused'` before
waiting. `getEvents()` drops each event's `metadata`, which is where a question's
`pause_reason`/`pause_index`/`question` live; read `GET /v1/runs/{runId}/events` with `fetch`
for that (helper in `agnt5-human-in-the-loop` `references/typescript.md`).

## Patterns

```typescript
// run with pending fallback, large-output safe
let res = await client.run<Report>('generate_report', { reportId }, { componentType: 'workflow', idempotencyKey: `report:${reportId}` });
if (res.isPending && res.status !== 'paused') res = await client.waitForResult<Report>(res.runId, 600_000, 2_000);
res.raiseForStatus();
const report = await client.resolveOutput(res);   // undefined while the run is paused

// typed event stream from an agent
for await (const ev of client.events('support_agent', { message }, { componentType: 'agent', sessionId })) {
  if (ev.eventType === 'output.delta') process.stdout.write(ev.data.content);
  if (ev.eventType === 'run.completed') console.log(ev.data.output);
}

// answer a HITL pause / send a signal / cancel (no client methods; the key needs the `workflow` scope)
// resume only after pendingQuestion() (agnt5-human-in-the-loop) confirms a question, not a durable sleep
const headers = { 'X-API-KEY': process.env.AGNT5_API_KEY!, 'Content-Type': 'application/json' };
await fetch(`${gatewayUrl}/v1/workflows/resume/${runId}`, { method: 'POST', headers, body: JSON.stringify({ user_response: 'approve' }) });
await fetch(`${gatewayUrl}/v1/runs/${runId}/signals/payment.settled`, { method: 'POST', headers, body: JSON.stringify({ payload: { reference } }) });
await fetch(`${gatewayUrl}/v1/runs/${runId}/cancel`, { method: 'POST', headers, body: JSON.stringify({ reason: 'operator stop' }) });
```

Headers the client sends: `X-API-KEY`, `X-TENANT-ID`, `X-DEPLOYMENT-ID` (explicit per-call
only for `run`), `X-Session-ID`, `X-User-ID`, `Idempotency-Key`, `X-AGNT5-Wait-Timeout-Ms`.
