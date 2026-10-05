# 03. Protocol

The engine's HTTP API, the call, the outcome, the worker stream and signed invocation. **Decided:** work reaches always-on workers over server-sent events with outcomes posted back over HTTP, and reaches scale-to-zero services as signed HTTP requests. No gRPC. Both modes are in the first milestone.

**Default D2 (handoff P10):** the shapes below start from the handoff's protocol draft and fill its gaps.

## 1. Transport and authentication

- All bodies are JSON in UTF-8 unless stated.
- Times are RFC 3339 strings in UTC. Durations are numbers of seconds.
- **Default D5.** Every request to the engine carries `Authorization: Bearer <token>`. Tokens are static and listed in the engine's configuration, each with one role:

| Role | May |
|---|---|
| `client` | Start, read and cancel runs |
| `worker` | Register, open a stream, post outcomes |
| `admin` | Everything, including definitions, queue pause, service sync and catalog removal |

  Several tokens may hold the same role, so a token can be rotated without downtime.
- **Default D5.** The engine serves plain HTTP and is deployed behind a proxy that terminates TLS, or on a private network. Outbound calls to services use HTTPS and verify certificates.
- The engine never logs inputs, results or saved step values.

### 1.1 Errors

A refused request has a 4xx or 5xx status and this body:

```json
{"error": {"code": "unknown_name", "message": "no workflow or task is registered as 'delete.jbo'"}}
```

| Status | Used for |
|---|---|
| 400 | A malformed request |
| 401, 403 | Missing or wrong token; a token without the needed role |
| 404 | `not_found`, `unknown_name` |
| 409 | `stale_call`, `unrouted_task`, `unknown_limit`, `worker_gone`, `in_use` |
| 422 | Definitions that failed validation, with every problem listed |

## 2. Endpoints

| Method and path | Role | Purpose |
|---|---|---|
| `POST /v1/runs` | client | Start a run |
| `GET /v1/runs/{id}` | client | Status, steps and attempts |
| `POST /v1/runs/{id}/cancel` | client | Cancel a run |
| `POST /v1/workers` | worker | Register a connected worker's catalog and capacity |
| `GET /v1/stream?worker_id=` | worker | The worker's event stream |
| `POST /v1/calls/{call_id}/outcome` | worker | Report an outcome |
| `PUT /v1/definitions` | admin | Apply the definitions file. The body is the TOML text |
| `GET /v1/definitions` | admin | The definitions currently applied |
| `POST /v1/queues/{name}/pause`, `/resume` | admin | Pause or resume a queue |
| `POST /v1/services/{name}/sync` | admin | Fetch a service's catalog |
| `DELETE /v1/catalog/{name}` | admin | Remove a retired workflow or task name. Refused with `in_use` while it has work waiting or running |
| `GET /v1/metrics` | admin | Metrics, in Prometheus text format (phase 6) |
| `GET /v1/health` | none | 200 when the store thread is answering |

## 3. Runs

### 3.1 Start

`POST /v1/runs`

```json
{
  "name": "delete.job",
  "input": {"job_id": "j-81", "next_invocation": 1},
  "key": "owner-group-123",
  "priority": 0,
  "idempotency_key": "delete:j-81",
  "start_at": "2026-10-05T09:00:00Z"
}
```

Only `name` and `input` are required. The answer is sent after the commit.

```json
{"run_id": "…", "state": "queued", "created": true}
```

The status is 201 when a run was created, and 200 with `"created": false` when the idempotency key matched an existing run.

### 3.2 Read

`GET /v1/runs/{id}`

```json
{
  "run_id": "…",
  "name": "delete.job",
  "kind": "workflow",
  "state": "waiting",
  "key": "owner-group-123",
  "priority": 0,
  "result": null,
  "error": null,
  "created_at": "…", "started_at": "…", "finished_at": null,
  "continued_from": null, "continued_as": null,
  "steps": [
    {"step": "item-7", "kind": "task", "task": "delete.item", "state": "resolved",
     "attempts": [
       {"attempt": 1, "host": "worker:deleter-2", "queued_at": "…", "started_at": "…",
        "finished_at": "…", "outcome": "error", "error": {"code": "upstream_timeout", "message": "…"},
        "next_retry_at": "…"},
       {"attempt": 2, "host": "worker:deleter-1", "queued_at": "…", "started_at": "…",
        "finished_at": "…", "outcome": "ok"}
     ]},
    {"step": "wait-7", "kind": "sleep", "state": "waiting", "until": "…"}
  ]
}
```

Step states are `waiting`, `running`, `resolved`, `failed` and `cancelled`. Saved step values are not included; `result` and `error` are the run's own. An unknown id answers 404 `not_found`.

### 3.3 Cancel

`POST /v1/runs/{id}/cancel` has no body and answers with the same shape as a read. Semantics are in [01](01-semantics.md) §7.

## 4. Connected workers

### 4.1 Register

`POST /v1/workers`

```json
{
  "name": "deleter",
  "capacity": 10,
  "tasks": [
    {"name": "delete.item",
     "uses": {"x-write": 1},
     "retries": 3, "backoff_factor": 2.0, "backoff_max_seconds": 30,
     "schedule_timeout": 86400, "execution_timeout": 120}
  ],
  "workflows": [
    {"name": "delete.job", "retries": 2}
  ]
}
```

- `capacity` is how many calls of either kind the worker runs at once. The engine never has more than that many open on it.
- Every option except `name` is optional ([01](01-semantics.md) §5.1).
- The catalog entry for a name is replaced by the latest registration of it, from any host.
- Registration is refused as a whole with `unrouted_task` if a task has no queue in the definitions, or `unknown_limit` if a `uses` names a limit that does not exist. The error lists every offending name.

```json
{"worker_id": "…"}
```

### 4.2 The stream

`GET /v1/stream?worker_id=…` answers `text/event-stream` and stays open.

| Event | Data | Meaning |
|---|---|---|
| `call` | A call (section 5) | Run this |
| `cancel` | `{"call_id": "…"}` | The run was cancelled. Stop if possible, then report an outcome as usual |

- The engine sends a comment line as a keep-alive at the interval set by `stream_keepalive_seconds` in its configuration. Keep-alives are not polling: they carry no question.
- The stream is one way. Outcomes travel as separate posts.
- When the stream closes, the worker reconnects with the same `worker_id`. The rules for the grace period, for re-sent calls and for a refused late reconnect (`worker_gone`) are in [01](01-semantics.md) §6.
- A worker must never ask the engine for work on an interval.

### 4.3 Reporting an outcome

`POST /v1/calls/{call_id}/outcome` with an outcome body (section 6).

| Answer | Meaning |
|---|---|
| 200 | Recorded. Also returned when this attempt already ended with a reported outcome, so a retried post is harmless |
| 409 `stale_call` | The attempt ended some other way (timeout, lost host, engine restart) or never existed. The worker drops the result |

The worker retries the post on a network failure or a 5xx answer. The engine answers only after the outcome is committed.

## 5. The call

The same body is a `call` event on a stream and the request body sent to a service.

```json
{
  "call_id": "f3a9…",
  "kind": "task",
  "name": "delete.item",
  "run_id": "…",
  "step": "item-7",
  "attempt": 2,
  "key": "owner-group-123",
  "input": {},
  "trace": "00-…"
}
```

| Field | Meaning |
|---|---|
| `call_id` | The attempt's token. The outcome must quote it |
| `kind` | `task` or `workflow` |
| `name` | The task or workflow name |
| `run_id`, `step`, `attempt` | For correlation in application logs. `step` is absent on workflow calls. `attempt` starts at 1 and counts every attempt of that step, including ones that did not count as failures |
| `key` | The run's key, or absent |
| `input` | The task's input, or the run's input for a workflow call |
| `trace` | A W3C `traceparent` value |
| `steps` | Workflow calls only. Every saved step of the run |

`steps` on a workflow call:

```json
"steps": {
  "new-id":  {"kind": "local", "ok": "a81f"},
  "item-7":  {"kind": "task", "task": "delete.item", "ok": {"next_action": "wait", "retry_after_seconds": 60}},
  "item-8":  {"kind": "task", "task": "delete.item", "error": {"code": "gone", "message": "…"}},
  "wait-7":  {"kind": "sleep", "ok": null}
}
```

A step has `ok` or `error`, never both. The kind and task name are what the SDK checks to detect a changed workflow ([01](01-semantics.md) §3.5).

## 6. The outcome

```json
{"outcome": "ok", "result": {}, "used": {"llm-tokens": 1840}, "saved": {}}

{"outcome": "error", "retryable": true, "code": "upstream_timeout", "message": "…", "retry_after_seconds": 30}

{"outcome": "limited", "limit": "x-write", "until": "2026-10-04T18:00:00Z"}

{"outcome": "suspended",
 "saved": {"new-id": {"ok": "a81f"}},
 "wants": [
   {"step": "item-8", "run_task": "delete.item", "input": {}, "uses": {"llm-tokens": 2000}},
   {"step": "wait-8", "sleep_seconds": 60}
 ]}

{"outcome": "continue_as_new", "input": {}, "saved": {}}
```

| Outcome | From | Fields |
|---|---|---|
| `ok` | Either kind | `result` (any JSON value). `used` from tasks only. `saved` from workflows only |
| `error` | Either kind | `retryable` (true if absent), `code`, `message`, optional `retry_after_seconds`, optional `stack` |
| `limited` | Tasks only | `limit`, `until` |
| `suspended` | Workflows only | `wants` with at least one item; optional `saved` |
| `continue_as_new` | Workflows only | `input`; optional `saved` |

- `saved` holds the results of local steps executed during this workflow call.
- Each item of `wants` is either a task (`step`, `run_task`, `input`, optional `uses` estimate) or a sleep (`step`, `sleep_seconds`).
- `message` should be short. `stack` is sent only when the application opts in, because run history should not collect sensitive detail by default.
- An outcome the engine cannot parse, or one not valid for the call's kind, is recorded as a retryable error with code `bad_outcome`.

### 6.1 HTTP status for invoked services

An SDK answers every call with status 200 and an outcome body whose content type is `application/vnd.geartrain.outcome+json`. **A body with that content type is authoritative, whatever the status.** This is how `suspended`, `continue_as_new` and `limited` reach the engine exactly.

When the response does not have that content type, the engine reads the status. This lets a task be written with no SDK at all:

| Status | Read as |
|---|---|
| 200 to 299 | `ok`. The result is the body parsed as JSON, or `null` if the body is empty |
| 400 to 499, except 429 | Non-retryable error |
| 429 | Retryable error. `Retry-After` is honoured as `retry_after_seconds` |
| 500 to 599 | Retryable error. `Retry-After` is honoured |
| Anything else, a failed connection, or no response | Retryable error |

A workflow needs the outcome body, so a workflow cannot be written without one.

## 7. Invoked services

### 7.1 The request

The engine sends `POST` to the service's `url` with a call as the body and these headers:

| Header | Value |
|---|---|
| `Content-Type` | `application/json` |
| `Geartrain-Timestamp` | Unix seconds when the request was signed |
| `Geartrain-Signature` | `v1=` followed by the lowercase hex HMAC-SHA256, keyed with the service's secret, of the timestamp, a `.`, and the exact body bytes |

The receiver must recompute the signature over the raw body, compare in constant time, and reject a mismatch. It must also reject a timestamp older than its tolerance, which the SDKs set to five minutes and let the application change. A request retried by the engine is signed again with a fresh timestamp.

The work must finish inside the request. A sleeping or waiting run holds no request open; the engine stores a timer and calls again when it fires.

The engine opens at most `max_concurrent` requests to a service at once.

### 7.2 Catalog sync

`POST /v1/services/{name}/sync` makes the engine send the service a signed request whose body is:

```json
{"kind": "catalog"}
```

The service answers with the same `tasks` and `workflows` lists a worker sends when registering (4.1). The engine validates and stores them under the same rules, and the sync answers with the stored catalog or the refusal.

Sync is run as a deploy step. It wakes the service once. Until a service has been synced, the engine does not know what it hosts and sends it nothing.

## 8. What every SDK provides

**Default D4.** Python is built first, then TypeScript, then Rust.

- A client to start, read and cancel runs.
- Workflow replay: return saved results, raise `TaskFailed` for saved errors, execute local steps, stop at the first unresolved step, report what is wanted, detect a changed workflow.
- A connected worker: register, open the stream, run calls up to capacity, post outcomes, reconnect, handle re-sent and cancelled calls.
- An adapter for invoked services: one route in the language's usual web framework that verifies signatures, answers catalog requests and runs calls.
- The catalog of hosted workflows and tasks with their options.

Errors a task can raise, by whatever names suit the language:

| Error | Outcome sent |
|---|---|
| Any ordinary exception | `error`, retryable |
| `NonRetryable(code, message)` | `error`, not retryable |
| `RetryAfter(seconds)` | `error`, retryable, with `retry_after_seconds` |
| `LimitReached(limit, until)` | `limited` |

A language-neutral conformance suite runs against every SDK, because all of them must replay identically. It is written in phase 4 against this document.
