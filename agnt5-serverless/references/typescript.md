# TypeScript serverless reference

Verified against @agnt5/sdk 0.10.5 (`dist/serverless*.d.ts`, `dist/workerless*.d.ts`,
`dist/flow-control.d.ts`) and the September 2026 CLI.

## Entry points

| Import | Function | Host interface |
|---|---|---|
| `@agnt5/sdk/serverless` | `serve(options)` | Fetch-native: `handler(request, env?, ctx?)` / `handler.fetch(...)` returns `Response`; Cloudflare, Deno, Hono, Next.js route handlers |
| `@agnt5/sdk/serverless/cloudflare` | `serveCloudflare(options)` | Cloudflare `export default` with `fetch(request, env, ctx)`; the secret resolver is `(env, request, ctx)` |
| `@agnt5/sdk/serverless/node` | `serveNode(options)` | Node `(IncomingMessage, ServerResponse)` handler for `http.createServer`, Express, Fastify, Koa; also exposes `.fetch()` and `nodeRequestToWorkerlessRequest` / `writeWorkerlessResponse` |

All three re-export `workflow`, `event`, `webhook`, and the `Serverless*`/`Workerless*` types.
Register functions with `fn(...)` from `@agnt5/sdk`, tools with `tool(...)`, agents with
`new Agent(...)`, then pass them in `functions`, `tools`, `agents`.

```typescript
interface WorkerlessServeOptions<Env, RuntimeContext> {
  serviceName?: string;
  serviceVersion?: string;
  workflows?: WorkflowHandler[]; functions?: FunctionHandler[]; tools?: ToolHandler[]; agents?: Agent[];
  enabled?: boolean | ((request, env?, ctx?) => boolean | Promise<boolean | undefined>);
  signingSecret?: string | ((request, env?, ctx?) => string | undefined | Promise<string | undefined>);
}
```

Omit a list to serve everything in that registry; pass `[]` to serve none. The handler also
exposes `manifest()`.

## Cloudflare Workers

```typescript
import { serve, workflow } from '@agnt5/sdk/serverless';
interface Env { AGNT5_SERVERLESS_SIGNING_SECRET?: string }
export default serve<Env>({
  serviceName: 'hello-agnt5', serviceVersion: 'local',
  signingSecret: (_request, env) => env?.AGNT5_SERVERLESS_SIGNING_SECRET,
  workflows: [hello],
});
```

Set `"main": "src/agnt5-workerless.ts"` in `wrangler.jsonc`, put the secret in `.dev.vars` for
`wrangler dev` and `npx wrangler secret put AGNT5_SERVERLESS_SIGNING_SECRET` for production,
deploy with `wrangler deploy`, and sync `--provider cloudflare --immutable-ref <Version ID
from wrangler versions list>`. Roll back with `wrangler rollback`.

## Vercel / Next.js (App Router)

`agnt5 serverless init --provider vercel --runtime typescript` generates
`src/agnt5-workerless.ts` (`export const agnt5Workerless = serve({...
serviceVersion: process.env.VERCEL_GIT_COMMIT_SHA ?? process.env.VERCEL_URL ?? 'local',
signingSecret: () => process.env.AGNT5_SERVERLESS_SIGNING_SECRET })`) plus two routes:

```typescript
// app/.well-known/agnt5/route.ts
export const runtime = 'nodejs';
export function GET(request: Request) { return agnt5Workerless.fetch(request); }
// app/agnt5/invoke/route.ts
export const runtime = 'nodejs';
export const maxDuration = 25;          // raise for multi-tool agents; check your plan's cap
export function POST(request: Request) { return agnt5Workerless.fetch(request); }
```

`next.config.mjs`: `serverExternalPackages: ['@agnt5/sdk', 'better-sqlite3']`; on Next 16 build
and dev with `--webpack`. Node runtime only. `vercel env add AGNT5_SERVERLESS_SIGNING_SECRET
production < .agnt5-serverless-secret`, `vercel deploy --prod`, then sync `--provider vercel`
(`--immutable-ref` defaults to `VERCEL_DEPLOYMENT_ID`).

## Node.js / Express

```typescript
import { createServer } from 'node:http';
import { serveNode, workflow } from '@agnt5/sdk/serverless/node';
const handler = serveNode({
  serviceName: 'orders-api', serviceVersion: process.env.GIT_SHA ?? 'local',
  signingSecret: () => process.env.AGNT5_SERVERLESS_SIGNING_SECRET,
  workflows: [hello],
  // baseUrl?: string | (req) => string  - only if the host rewrites the URL
});
createServer((req, res) => { handler(req, res).catch(() => { res.statusCode = 500; res.end(); }); })
  .listen(Number(process.env.PORT ?? 8787));
// Express: mount BEFORE any JSON body parser - HMAC needs the raw body
app.all('/.well-known/agnt5', handler);
app.all('/agnt5/invoke', handler);
```

Fastify and Koa: pass the raw Node request/response objects; never rebuild the body from
parsed JSON. Sync with `--provider http --immutable-ref <git-sha>`.

## `WorkerlessContext` (implements the same `Context` interface as workers)

| Member | Notes |
|---|---|
| `invocationId`, `runId`, `attempt`, `serviceName`, `metadata`, `runtime`, `logger`, `signal` | `signal` is an `AbortSignal` that never aborts on this path |
| `step<T>(name, fn, { key? })` | Checkpointed by name; logs a warning when step names change between replays |
| `get/set/delete` | Request-scoped map, not checkpointed |
| `sleep(durationMs, name?)` | Timer suspension |
| `yieldIfNeeded(reason?)` | Budget suspension |
| `waitForUser(question, { inputType, options, allowCustom, skippable })` | Resolves to `string \| null` |
| `waitForSignal<T>(signalName, name?)` | Only implemented here; the worker `ContextImpl` throws `ConfigurationError` |
| `emit(event)` | Collected into the response |

## Flow control

```typescript
import { fn, workflow } from '@agnt5/sdk';
export const charge = fn('charge')
  .flowControl({ concurrency: { limit: 5, scope: 'component' }, idempotency: { keyExpression: 'input.order_id' } })
  .run(async (_ctx, input: { order_id: string }) => {/* ... */});
export const nightly = workflow('nightly', handler, { flowControl: { priority: 'batch', singleton: { key: 'nightly', mode: 'queue' } } });
```

`FunctionOptions`/`WorkflowOptions` also accept `flow_control` (manifest spelling),
`priority: number`, and `maxConcurrency: number` as top-level manifest metadata.
`fn('x').priority(n)` and `.maxConcurrency(n)` exist as builder methods too. `batch` is
rejected at sync in the beta.

## Offline test (vitest)

```typescript
const handler = serve({ serviceName: 'test', workflows: [hello] });   // no signingSecret -> unsigned accepted
const manifest = await (await handler.fetch(new Request('https://x/.well-known/agnt5'))).json();
const res = await handler.fetch(new Request('https://x/agnt5/invoke', {
  method: 'POST',
  body: JSON.stringify({ protocol_version: 'workerless.v1', run_id: 'r1', component_type: 'workflow', component_name: 'hello', input: { name: 'Ada' } }),
}));
expect(await res.json()).toEqual({ status: 'completed', output: { message: 'hello Ada' } });
```

Call `FunctionRegistry.clear()`, `WorkflowRegistry.clear()`, `ToolRegistry.clear()` in
`beforeEach` when tests register components by name. `verifyWorkerlessInvokeRequest(request,
bodyText, signingSecret, env, ctx)` is exported from `@agnt5/sdk/serverless` internals
(`workerless.js`) if you need to sign test requests the way the SDK verifies them.
