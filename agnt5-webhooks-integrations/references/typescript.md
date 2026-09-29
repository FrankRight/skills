# Webhooks, chat bots and the client in TypeScript

Verified against `@agnt5/sdk` **0.10.5**. Same section order as the Python SKILL.md. Signature
verification, delivery semantics, event names and Studio setup are platform-side and apply
unchanged; this file covers the TypeScript code and the actual input shape.

## Declare a trigger

```typescript
import { workflow, webhook, event } from '@agnt5/sdk';
import type { Context } from '@agnt5/sdk';

export const triageIssue = workflow(
  'triage_issue',
  async (ctx: Context, input: TriggeredRun) => {
    const envelope = webhookEnvelope(input);            // see below
    const payload = JSON.parse(envelope.body);          // raw body string — parse it yourself
    const issue = payload.data.issue;
    return ctx.step('triage', () => triage(ctx, { issue, key: envelope.idempotency_key }));
  },
  { triggers: [webhook('sentry', { event: 'issue.created' })] },
);

export const welcome = workflow('welcome', async (ctx: Context, input: TriggeredRun) => { /* ... */ }, {
  triggers: [event('user.signed_up')],
});
```

`webhook(source, { event, triggerId? })` and `event(name, { triggerId? })` return a
`TriggerSpec` for `WorkflowOptions.triggers`. `source` is `standard | sentry | stripe | github | slack`.

**Leave `filterExpression`, `inputMapping`, `batchWindowMs` and `delayExpression` unset.** The
types accept them, but the gateway skips any trigger that sets one, so the workflow never
starts and nothing reports why (AGNT5-1376). Filter and reshape inside the workflow instead.

## What the workflow receives

**Not** the bare envelope shown in the docs. A triggered run's input is:

```typescript
interface TriggeredRun {
  event: {
    id: string;
    name: string;                 // 'sentry.issue.created'
    data: WebhookEnvelope;        // the envelope from the SKILL.md
    source: string;
    timestamp_ns: number;
  };
  deployment_id: string;
  target_kind: string;
  target_ref: string;
}

interface WebhookEnvelope {
  _webhook: true;
  source: string;
  integration_id: string;
  event_type: string;
  idempotency_key?: string;       // absent for Slack
  timestamp: number;
  headers: Record<string, string>; // lowercased keys
  body: string;                   // raw request body
}

function webhookEnvelope(input: TriggeredRun | WebhookEnvelope): WebhookEnvelope {
  return '_webhook' in input ? input : input.event.data;   // tolerate both shapes
}
```

Verified against the gateway's event dispatch and a live run; the docs' `event.body` at the
top level is wrong. Key side effects off `event.data.event_type` +
`event.data.idempotency_key` (or `event.id`), and put them in `ctx.step` — delivery is
at-least-once and replays re-run bare code (AGNT5-1373).

## Chat bots (Slack, Discord, Teams, Telegram)

```typescript
import { Agent, ChatBot, LM, Worker } from '@agnt5/sdk';

const agent = new Agent({
  name: 'support-bot', model: LM.anthropic(), modelName: 'anthropic/claude-sonnet-5',
  instructions: 'You are a helpful support agent.',
});

const bot = new ChatBot(agent, [
  { platform: 'slack', botToken: process.env.SLACK_BOT_TOKEN!, signingSecret: process.env.SLACK_SIGNING_SECRET! },
]);
bot.onMention(async (evt) => `You said: ${evt.message?.content ?? ''}`)   // return null to stay silent
   .onSlashCommand(async (evt) => `ran ${evt.command} ${evt.args ?? ''}`);

const worker = new Worker('support-bot');
worker.registerAgents([bot]);        // register the bot, not the bare agent
await worker.run();
```

- `new ChatBot(agent, adapters: PlatformConfig[])`. Configs are discriminated by `platform`:
  `{ platform: 'slack', botToken, signingSecret, appToken? }`,
  `{ platform: 'discord', botToken, publicKey, applicationId }`,
  `{ platform: 'teams', appId, appPassword, tenantId? }`,
  `{ platform: 'telegram', botToken, webhookSecret? }`.
- Handlers are chainable methods, not decorators: `onMention`, `onMessage`, `onReaction`,
  `onSlashCommand`, `onAction`, each `(event: ChatEvent) => Promise<string | null>`.
  `ChatEvent`: `{ eventType, message?: ChatMessage, channelId?, threadId?, user?, challenge?,
  emoji?, command?, args?, actionId? }`; `ChatMessage`: `{ id, platform, channelId, threadId?,
  author: { id, name, platform }, content, attachments, isMention, isDm, metadata }`.
- Import from the root package. The `@agnt5/sdk/chat` path in the class comment is not an
  export, and `worker.register(bot)` does not exist — use `registerAgents`.
- The worker routes an inbound chat event to the bot by the wrapped agent's `name`; the same
  agent stays callable directly.

## AI provider credentials

Same as Python: `.env` locally (`agnt5 dev` loads it), `agnt5 secrets set` / Studio for
deployments. `LM.openai()` etc. read `OPENAI_API_KEY`-style variables when `apiKey` is
omitted.

## Integrating an existing application (non-webhook)

```typescript
import { Client } from '@agnt5/sdk';

const client = new Client({
  gatewayUrl: process.env.AGNT5_GATEWAY_URL,   // default http://localhost:34181 — set it in backends
  apiKey: process.env.AGNT5_API_KEY,
});

// blocking — waits for the result
const res = await client.run('onboarding_workflow', { userEmail: 'ada@example.com' }, {
  componentType: 'workflow',                   // default 'function'
  idempotencyKey: `onboard:${userId}`,
  waitTimeoutMs: 300_000,
});
console.log(res.status, res.output);           // res.isSuccess / res.isPending / res.raiseForStatus()

// fire-and-forget
const sub = await client.submit('onboarding_workflow', { userEmail }, { componentType: 'workflow', idempotencyKey: `onboard:${userId}` });
const final = await client.waitForResult(sub.runId, 600_000);

// fluent alternative
await client.workflow('onboarding_workflow').run({ userEmail }, { idempotencyKey: `onboard:${userId}` });
```

`RunResponse`: `runId`, `status` (`'completed' | 'failed' | 'awaiting_input' | 'running' | ...`),
`output`, `error`, `durationMs`, `traceId`, getters `isSuccess`, `isPending`, `isError`,
`raiseForStatus()`. `client.events(component, input, opts)` streams SSE events. Always pass
`idempotencyKey` from a stable business id. Full client surface: `agnt5-client`.

## Not available in TypeScript

- `AsyncClient` (the client is async-only) and `https://gw.agnt5.com` as the default URL
- `@bot.on_mention` decorators (chainable methods instead) and `Worker(agents=[bot])`
  (`worker.registerAgents([bot])`)
- `agnt5.chat` / `@agnt5/sdk/chat` import paths (root exports)
- A typed envelope helper: define `TriggeredRun` / `WebhookEnvelope` yourself as above

## TypeScript pitfalls

| Symptom | Cause | Fix | Ticket |
|---|---|---|---|
| `JSON.parse(input.body)` throws `undefined` | body lives at `input.event.data.body` | use `webhookEnvelope(input)` | — |
| Side effects run twice per delivery | at-least-once delivery + bare calls replay | `ctx.step` keyed by `idempotency_key` | AGNT5-1373 |
| Slack retries start duplicate runs | Slack has no delivery id | dedupe on the Slack `event_id` inside `body` | — |
| Bot never answers | bare agent registered, or bot module not imported | `worker.registerAgents([bot])` | — |
| A handler rejection kills the worker | unhandled promise rejection | `process.on('unhandledRejection', ...)` + try/catch in handlers | AGNT5-1352 |
| Backend `Client` calls `localhost:34181` in production | default gateway URL | set `AGNT5_GATEWAY_URL` / `gatewayUrl` | — |

## Source

https://agnt5.com/docs/build/webhooks · /docs/integrations/event-sources/overview · /docs/integrations/ai-providers (TypeScript tabs)
