---
name: agnt5-webhooks-integrations
description: Connect AGNT5 to the outside world - trigger workflows from Standard Webhooks, Sentry, Stripe, GitHub, or Slack events and internal event() triggers, the trigger envelope the handler receives (event["data"]["body"] plus deployment_id/target_kind/target_ref kwargs; filter/input-mapping/batch/delay options exist but are not dispatched yet), signature verification and idempotent delivery; run an agent as a Slack/Discord/Teams/Telegram chat bot with ChatBot; and call deployed workflows from your own app with Client.run/submit, pending receipts and idempotency keys. Use for "receive a Stripe/GitHub/Sentry webhook", "start a workflow when X happens", "build a Slack bot", "call my AGNT5 workflow from my backend", or a triggered workflow failing with an unexpected keyword argument.
---

# AGNT5 Webhooks and Integrations

> **TypeScript or Go?** This file shows the Python API. Read [references/typescript.md](references/typescript.md) or [references/go.md](references/go.md) first: same sections, the exact signatures for that SDK, and what it does not support.

A webhook lets an external system start a workflow by POSTing an event to AGNT5. The gateway
verifies the signature, turns the delivery into a durable event, and starts every workflow
subscribed to it. AGNT5 handles receipt, verification, deduplication, and dispatch — you only
write the workflow and declare what it listens for.

## Declare a trigger

```python
import json
from agnt5 import WorkflowContext, webhook, workflow

@workflow(name="triage_issue", triggers=[webhook("sentry", event="issue.created")])
async def triage_issue(ctx: WorkflowContext, event: dict, **_) -> dict:
    payload = json.loads(event["data"]["body"])   # raw body string — parse it yourself
    issue = payload["data"]["issue"]
    ...
```

The gateway starts the workflow with **four keyword arguments**: `event`, `deployment_id`,
`target_kind`, `target_ref`. Accept the extras with `**_` (or name them) — a handler declared
as `async def h(ctx, event: dict)` fails with `TypeError: got an unexpected keyword argument
'deployment_id'` (live-verified). `webhook` is importable from `agnt5` but not listed in
`agnt5.__all__`, so `from agnt5 import *` does not bring it in.

`source` is one of `standard`, `sentry`, `stripe`, `github`, `slack`. A single event can fan
out to multiple workflows — every workflow whose trigger matches `{source}.{event}` starts
independently.

Internal events use `event("user.signed_up")` the same way. Both `webhook()` and `event()`
also accept `filter_expression=`, `input_mapping=`, `batch_window_ms=`, `delay_expression=`
and `trigger_id=`. **Leave the first four unset**: the current gateway skips any trigger that
sets one of them as unsupported (counted in the event's `skipped_unsupported_count`; the
workflow does not start; AGNT5-1376), and their expression syntax is undocumented. Filter
inside the workflow instead.

## What the workflow receives

`event` is the gateway's trigger envelope; the webhook delivery sits under `event["data"]`:

```json
{
  "event": {
    "id": "…", "name": "sentry.issue.created", "source": "…", "timestamp_ns": 1733337600000000000,
    "data": {
      "_webhook": true,
      "source": "sentry",
      "integration_id": "int_abc123",
      "event_type": "sentry.issue.created",
      "idempotency_key": "req_9f3c…",
      "timestamp": 1733337600,
      "headers": { "sentry-hook-resource": "issue" },
      "body": "{\"action\":\"created\",\"data\":{ … }}"
    }
  },
  "deployment_id": "…", "target_kind": "deployment_id", "target_ref": "…"
}
```

`body` is the raw request body as a **string** — parse it yourself so you operate on exactly
the bytes that were signature-verified. `headers` keys are lowercased. For `event()` triggers
`event["data"]` is whatever the publisher sent; `target_kind` is `deployment_id` or
`environment_ref`.

## Set up an integration (once per source)

1. **Studio → Integrations → New**, pick the source.
2. Pick the **environment** whose deployment should receive triggers.
3. Provide the **signing secret** — AGNT5 generates one for GitHub and Standard Webhooks
   (copy it into the publisher); paste the provider-issued one for Stripe, Slack, Sentry.
4. Copy the **webhook URL** (`…/v1/webhooks/{source}/{integration_id}`) into the provider's
   webhook settings.

## Signature verification (handled automatically, know the model)

| Source | Header | Signed payload | Replay window |
|---|---|---|---|
| `standard` | `webhook-signature` (`v1,<base64>`) | `{id}.{timestamp}.{body}` | 5 min |
| `sentry` | `sentry-hook-signature` (hex) | raw body | n/a |
| `stripe` | `Stripe-Signature` (`t=…,v1=…`) | `{timestamp}.{body}` | 5 min |
| `github` | `X-Hub-Signature-256` (`sha256=…`) | raw body | n/a |
| `slack` | `X-Slack-Signature` (`v0=…`) + timestamp | `v0:{timestamp}:{body}` | 5 min |

Unsigned/mismatched deliveries get a `401` before any workflow runs.

## Delivery semantics — make handlers idempotent

Delivery is **at-least-once**. Retries are deduped by the provider's per-delivery id
(`webhook-id` / `Request-ID` / Stripe event `id` / `X-GitHub-Delivery`) — a retry replays the
original run, it doesn't start a new one. Two exceptions where a workflow can fire more than
once for the same event:

- **Slack** has no stable per-delivery id, so its retries are never deduped.
- The idempotency cache is per-gateway-instance and time-bounded — a retry after a gateway
  restart, or on a different node in a multi-node deployment, sees a cold cache.

**Key your side effects off `event_type` + `idempotency_key` (or an id in the body) so a
re-delivery is a no-op.** AGNT5 guarantees at-least-once, not exactly-once.

## Event names by source

| Source | What you pass as `event` | Resulting name |
|---|---|---|
| `standard` | the `webhook-event` header value | `standard.<event>` |
| `sentry` | `<resource>.<action>` | `sentry.issue.created` |
| `stripe` | the body `type` | `stripe.payment_intent.succeeded` |
| `github` | `<event>.<action>` or `<event>` | `github.issues.opened` |
| `slack` | the event-callback `type` | `slack.app_mention` |

Slack events are named `slack.<event.type>`, e.g. `slack.app_mention` (needs
`app_mentions:read` scope), `slack.message`, `slack.reaction_added`. Sentry's Studio setup
also requires creating a matching custom integration in Sentry's own
**Settings → Integrations → Custom Integrations** and pasting AGNT5's webhook URL there.

## Chat bots (Slack, Discord, Teams, Telegram)

For a conversational bot, wrap an agent in `ChatBot` instead of hand-parsing `slack.*`
webhooks — AGNT5 verifies and routes the event, runs the agent, and posts the reply:

```python
import os
from agnt5 import Agent, Worker
from agnt5.chat import ChatBot, SlackConfig

agent = Agent(name="support-bot", model="anthropic/claude-sonnet-5",
              instructions="You are a helpful support agent.")
bot = ChatBot(agent=agent, adapters=[
    SlackConfig(bot_token=os.environ["SLACK_BOT_TOKEN"],
                signing_secret=os.environ["SLACK_SIGNING_SECRET"]),
])

worker = Worker(service_name="support-bot", agents=[bot])   # register the bot, not the bare agent
```

Also `DiscordConfig`, `TeamsConfig`, `TelegramConfig`. Custom routing: decorate handlers with
`@bot.on_mention`, `@bot.on_message`, `@bot.on_reaction`, `@bot.on_slash_command`,
`@bot.on_action` (return a reply string, or `None` to stay silent).

## AI provider credentials

Model calls need a provider credential (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, …). Local dev:
`.env`, see `agnt5-project-init`. Deployed workers: Studio integrations or
`agnt5 secrets set`, see `agnt5-deploy`.

## Integrating an existing application (non-webhook)

Call a deployed workflow/agent from your own backend with the SDK client (reads
`AGNT5_API_KEY` — must start with `agnt5_sk_` — and `AGNT5_GATEWAY_URL`, default
`https://gw.agnt5.com`):

```python
from agnt5 import Client

client = Client()
# waits up to wait_timeout (default 300 s), then returns a pending receipt instead of blocking
res = client.run("onboarding_workflow", {"user_email": "ada@example.com"},
                 component_type="workflow", idempotency_key=f"onboard:{user_id}")
if res.is_pending:                                   # HTTP 202 — the run is still going
    res = client.wait_for_result(res.run_id, timeout=900)
print(res.status, res.output)

# fire-and-forget — returns a run_id to poll or inspect later
sub = client.submit("onboarding_workflow", {"user_email": "ada@example.com"},
                    component_type="workflow", idempotency_key=f"onboard:{user_id}")
```

`component_type` defaults to `"function"` — pass it for workflows, agents and tools.
`AsyncClient` has the same `run` / `submit` but no `wait_for_result`: poll with
`await client.get_status(run_id)` / `await client.get_result(run_id)`. Always pass
`idempotency_key=` from a stable business id so retries from your app don't start duplicate
runs. `run` and `stream_events` (not `submit`) take `session_id=` / `user_id=` to give the run
session and user scope (`agnt5-workflows`). From a shell, `agnt5 run ... --env production`
does the same (see `agnt5-project-init`). Full client API, streaming and answering a paused
run: `agnt5-client`.

## Source

https://agnt5.com/docs/build/webhooks · https://agnt5.com/docs/integrations/event-sources/overview · https://agnt5.com/docs/integrations/ai-providers
