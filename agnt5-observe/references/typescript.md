# Observing TypeScript runs

Verified against `@agnt5/sdk` **0.10.5**. The `agnt5 inspect` / `agnt5 logs` CLI, Studio
pages and MCP tools in the SKILL.md are language-independent. This file covers what is
different for a TypeScript worker and the code-side logging, span and capture APIs.

## Traces: no spans for TypeScript runs yet

TypeScript workers do not export trace spans, so for a TS run
`agnt5 inspect trace -r <runId>` and Studio's Trace tab have no span tree to show. Use instead:

```bash
agnt5 inspect runs describe <runId>            # status, duration, step count, error
agnt5 inspect logs -r <runId> --severity ERROR  # your ctx.logger lines
```

The run's *journal events* (`workflow.step.*`, `function.*`, `agent.*`, `lm.*`,
`tool_call.*`) are still recorded and drive the Studio run timeline, scorers and
`client.getEvents(runId)` (`{ events: [{ eventType, data, sequence, correlationId }] }`).

Every failed run is reported as `EXECUTION_ERROR` regardless of the thrown error.
Log the real type before rethrowing:

```typescript
try {
  return await riskyCall();
} catch (err) {
  ctx.logger.error('riskyCall failed', { name: (err as Error).name, message: (err as Error).message });
  throw err;
}
```

## Logs from code

```typescript
import { getLogger, setLogLevel } from '@agnt5/sdk';

ctx.logger.info('Sending email', { to, attempt: ctx.attempt });   // run logs + stdout

const log = getLogger('billing');       // module-level; records are attributed to the current run
log.debug('details', { key: 'value' });
setLogLevel('DEBUG');                   // 'DEBUG' | 'INFO' | 'WARN' | 'ERROR'; AGNT5_DEBUG=1 does the same at startup
```

Extra fields go in the second argument as an object (`ctx.logger.info('msg', { key })`), not as
keyword arguments. Plain `console.log` reaches only the worker's stdout (deployment logs),
never `agnt5 inspect logs -r`.

## Spans from code

```typescript
import { withSpan, spanContext, span, getCurrentSpanInfo } from '@agnt5/sdk';

const rows = await withSpan('db-query', async (s) => {
  s.setAttribute('table', 'users');
  return db.query('SELECT * FROM users');
}, { componentType: 'function', attributes: { db: 'primary' } });

const s = spanContext('manual'); try { /* ... */ } finally { s.end(); }
const traced = span('process-order')(async (orderId: string) => { /* ... */ });
getCurrentSpanInfo();   // { traceId, spanId } | undefined
```

These stamp `traceId`/`spanId` onto log records emitted inside them (that correlation is what
`agnt5 inspect logs` shows). Whether the span itself is exported depends on the native
binding; until TypeScript runs record spans, treat them as log correlation, not as trace
structure.

## Automatic capture of OpenAI / OpenAI Agents SDK / Vercel AI SDK / Google ADK calls

`worker.run()` calls `autoEnable()` from `@agnt5/sdk/integrations`, which patches whichever
of these npm packages is installed: `openai`, `@openai/agents`, `ai` (Vercel AI SDK), and
`@google/adk`. Calls made inside a component appear as `lm.*` / `agent.*` / `tool_call.*`
journal events tagged `capture_mode=observed` and `source=<library>` (e.g. `vercel_ai`).
There are no `agnt5[...]` extras; install the library itself.

| Env var (TypeScript) | Effect |
|---|---|
| `AGNT5_CAPTURE=off` | Disable all capture (`off`, `0`, `false`, `no`) |
| `AGNT5_CAPTURE_OPENAI=0` / `AGNT5_CAPTURE_OPENAI_AGENTS=0` / `AGNT5_CAPTURE_VERCEL_AI=0` / `AGNT5_CAPTURE_GOOGLE_ADK=0` | Disable one library |
| `AGNT5_LLM_CAPTURE_CONTENT=off` | Omit prompt/response text from captured events |

`AGNT5_CAPTURE_CONTENT_MODE` and `AGNT5_CAPTURE_MAX_CONTENT_CHARS` are Python-only; the
TypeScript content switch is the boolean `AGNT5_LLM_CAPTURE_CONTENT`.

Vercel AI SDK: capture registers through the AI SDK's global telemetry (AI SDK 7+). To force
telemetry on calls that do not set `experimental_telemetry`, wrap the namespace:

```typescript
import * as ai from 'ai';
import { wrapAISDK } from '@agnt5/sdk/integrations';

const { generateText, streamText } = wrapAISDK(ai);   // also generateObject / streamObject
```

`enableVercelAICapture()`, `enableOpenAICapture()`, `enableOpenAIAgentsCapture()` and
`enableGoogleADKCapture()` are exported for explicit enabling (each returns `Promise<boolean>`).
Capture is best-effort: a missing event is not proof the call did not happen; calls made
outside a component context are not journaled.

## Metrics and machine-readable output

Identical to the SKILL.md (Studio Analytics/Metrics, MCP tools, `--output json`).

## TypeScript pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| Empty trace for a TS run | no span export | logs + journal events |
| Error code always `EXECUTION_ERROR` | worker collapses codes | log `err.name` yourself |
| `console.log` lines missing from `agnt5 inspect logs -r` | console is stdout only | use `ctx.logger` / `getLogger` |
| `AGNT5_CAPTURE_CONTENT_MODE=redacted` has no effect | Python-only variable | `AGNT5_LLM_CAPTURE_CONTENT=off` |
| OpenAI calls from a script are not captured | no ambient component context | run them inside a `fn()` / workflow |

## Source

https://agnt5.com/docs/run/deploying · https://agnt5.com/docs/integrations/third-party/openai-sdk (TypeScript tab)
