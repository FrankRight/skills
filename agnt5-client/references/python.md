# Python client reference

Verified against agnt5 0.13.6 (`src/agnt5/client.py`, `responses.py`; signatures introspected
with `inspect.signature`).

## `Client` (sync, httpx) - use as a context manager or call `close()`

```python
Client(gateway_url=None, timeout=45.0, api_key=None, tenant_id=None, deployment_id=None)

run(component, input_data=None, component_type="function", session_id=None, user_id=None, tenant=None,
    timeout=None, headers=None, deployment_id=None, *, idempotency_key=None, wait_timeout=300.0) -> RunResponse
submit(component, input_data=None, component_type="function", metadata=None, tenant=None, deployment_id=None,
       *, idempotency_key=None) -> SubmitResponse
get_status(run_id) -> StatusResponse            # .status, .is_complete, .is_running
get_result(run_id) -> RunResponse
wait_for_result(run_id, timeout=300.0, poll_interval=1.0) -> RunResponse
get_events(run_id) -> EventsResponse            # iterable of Event(id, event_type, run_id, step_key, input_data, output_data, ...)
stream(component, input_data=None, component_type="function", tenant=None, deployment_id=None,
       *, idempotency_key=None, wait_timeout=300.0, timeout=None) -> Iterator[str]
stream_events(component, input_data=None, component_type="function", session_id=None, user_id=None, tenant=None,
              timeout=None, deployment_id=None, *, idempotency_key=None, wait_timeout=300.0) -> Iterator[ReceivedEvent]
workflow(workflow_name) -> WorkflowProxy
session(session_type, key) -> SessionProxy      # entity-backed conversation
entity(entity_type, key) -> EntityProxy
batch(component, items, component_type="function", max_concurrency=10, continue_on_failure=True,
      batch_timeout_ms=None, item_timeout_ms=None, metadata=None, timeout=None, deployment_id=None,
      *, idempotency_key=None) -> BatchResult
get_batch_status(batch_id, include_results=True, timeout=None) -> BatchStatusResult
cancel_batch(batch_id, reason=None, timeout=None) -> CancelBatchResult
eval(component, input_data=None, expected=None, scorers=None, component_type="function", session_id=None,
     user_id=None, timeout=None, deployment_id=None) -> EvalResponse
batch_eval(component, items, scorers=None, expected=None, component_type="function", max_concurrency=10,
           timeout=None, deployment_id=None) -> BatchEvalResult
close()
```

`wait_timeout` is the gateway wait (0-86400 s; 0 returns a receipt immediately); the HTTP
`timeout` defaults to `max(client.timeout, wait_timeout + 10)`. `headers=` merges extra HTTP
headers into `run()` only. 404/503/504 from the gateway come back as failed/timeout
`RunResponse`s (codes `NOT_FOUND`, `SERVICE_UNAVAILABLE`, `TIMEOUT`) rather than exceptions.

## `AsyncClient` (`async with AsyncClient() as client:`)

Same constructor; `await client.run(...)` (no `headers=`), `submit`, `get_status`,
`get_result`, `get_events`, `stream_events` (async iterator), `batch`, `batch_eval`, `eval`,
`get_batch_status`, `cancel_batch`, `close`. **Not** on `AsyncClient` in 0.13.6: `stream()`,
`wait_for_result()`, `workflow()`, `session()`, `entity()` - poll `get_result` yourself.

## Proxies

```python
wf = client.workflow("support_flow")
wf.run(session_id=None, user_id=None, wait_timeout=300.0, timeout=None, **input_fields) -> RunResponse
wf.chat(message, session_id=None, user_id=None, **extra) -> RunResponse       # chat=True workflows
wf.submit(**input_fields) -> SubmitResponse
wf.stream_events(session_id=None, user_id=None, timeout=None, wait_timeout=300.0, **input_fields)

s = client.session("Conversation", "user-alice")   # POST /v1/entity/{type}/{key}/{method}
s.chat(message, **extra) -> str
s.get_history() -> list
s.add_message(role, content) -> RunResponse
s.clear_history() -> RunResponse
```

Workflow proxies take the input as keyword arguments, so a field named `session_id` or
`timeout` must go through `client.run(...)` instead.

## Responses

```python
RunResponse: run_id, status_code, status: RunStatus, output, error: RunErrorDetail(code, message, details),
             duration_ms, trace_id, component, created_at, started_at, completed_at, failed_at, session_id, metadata
             .is_success (200 + completed)  .is_pending (202)  .is_error  .elapsed -> timedelta  .raise_for_status()
SubmitResponse: run_id, status, trace_id, component, created_at, links.self_url, .status_url
StatusResponse: run_id, status, ... .is_complete .is_running
ReceivedEvent (from stream_events): event_type, data, content_index, sequence, run_id
RunError(message, run_id=None, error_code=None, attempts=None, max_attempts=None, metadata=None)
         .was_retried  .exhausted_retries
RunStatus: PENDING ENQUEUED QUEUED STARTED RUNNING COMPLETED FAILED CANCELLED PAUSED AWAITING_INPUT AWAITING_USER_INPUT TIMEOUT UNKNOWN
```

## Patterns

```python
# fire-and-forget from a request handler, finish in a worker
sub = client.submit("generate_report", {"report_id": rid}, component_type="workflow", idempotency_key=f"report:{rid}")
... later ...
status = client.get_status(sub.run_id)
if status.is_complete:
    result = client.get_result(sub.run_id)

# answer a HITL pause (no client method in 0.13.6)
import httpx, os
httpx.post(f"{client.gateway_url}/v1/workflows/resume/{run_id}",
           headers={"X-API-KEY": os.environ["AGNT5_API_KEY"]}, json={"user_response": "approve"}).raise_for_status()

# send a signal to a serverless workflow
httpx.post(f"{client.gateway_url}/v1/runs/{run_id}/signals/payment.settled",
           headers={"X-API-KEY": os.environ["AGNT5_API_KEY"]}, json={"payload": {"reference": ref}})
```

Large serverless outputs may come back as an `output_ref` inside the raw response (`_raw`);
the Python client does not dereference them in 0.13.6 - use the TypeScript client or
`GET /v1/result/{run_id}` handling in your own code.
