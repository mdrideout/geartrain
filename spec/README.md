# geartrain specification, phase 1

| | |
|---|---|
| Date | 2026-10-04 |
| Owner | Matt |
| State | Draft for Matt's review. No engine code exists. |
| Source | The design handoff of 2026-10-04 ("the handoff"). Section numbers such as "handoff 6.7" refer to it. |

This specification is the source of truth for the engine and the SDKs. Where it and the handoff differ, this specification wins, and the difference is listed in section 4 below.

## 1. Documents

| File | Covers |
|---|---|
| [01-semantics.md](01-semantics.md) | Runs, steps, tasks and attempts; retries and timeouts; cancellation; `continue_as_new`; timers and schedules |
| [02-queues-and-limits.md](02-queues-and-limits.md) | The definitions-file format; queues, limits and the claiming algorithm |
| [03-protocol.md](03-protocol.md) | The engine's HTTP API, the call, the outcome, the worker stream, signed invocation |
| [04-data-model.md](04-data-model.md) | The SQLite schema, indexes, invariants and schema migrations |
| [05-durability-and-recovery.md](05-durability-and-recovery.md) | The three durability modes, group commit, crash recovery, backup, engine configuration |

## 2. Labels

Every rule in these documents carries one of three labels, or none.

| Label | Meaning |
|---|---|
| **Decided** | Matt stated or confirmed it. Do not reopen without him. |
| **Default Dn** | Matt has not decided. The specification picks an option so work can continue, and Matt can override it. Each default has a number and is listed in section 3. |
| No label | A detail that follows from decided rules or from a default. It changes if its parent changes. |

## 3. Defaults that need Matt's call

"Handoff option" means the handoff proposed or recommended it. "Spec choice" means the handoff left it open with no recommendation, or the handoff review found a gap, and this specification picked.

| # | Question | Default taken | Origin | Where |
|---|---|---|---|---|
| D1 | Where is a task's queue set? | In the definitions file, never on the task | Handoff option (17.1) | 02 §2 |
| D2 | Do the handoff's proposals P1 to P10 stand? | Yes, all ten, as written | Handoff option (17.2) | Throughout |
| D3 | Definitions format and ownership | One TOML file applied by an administrator. Applications cannot define queues through the API | Handoff option (17.3) | 02 §1 |
| D4 | Which SDK is built first? | Python | Spec choice (17.5). The handoff's examples and the reference workload are Python | 03 §8 |
| D5 | Authentication and TLS | Static bearer tokens with three roles. The engine serves plain HTTP and sits behind a proxy that terminates TLS | Handoff option for tokens; spec choice for TLS (17.6) | 03 §1 |
| D6 | Retention of finished runs | Nothing is deleted automatically. A retention setting arrives in phase 6 | Spec choice (17.7) | 04 §5 |
| D7 | Schedule firings missed while the engine was down | Each schedule starts one catch-up run at startup, however many firings were missed | Spec choice (17.8) | 01 §9 |
| D8 | Priority and fairness across keys | In a round-robin queue, priority orders work inside one key only. In a fifo queue, priority orders the whole queue | Handoff option (17.9, the prototype's behaviour) | 02 §4 |
| D9 | A second read-only connection | No. One connection on one thread | Handoff option (17.10) | 05 §1 |
| D10 | Timeouts against the "no arbitrary timeouts" rule | The engine ships no default timeout. `execution_timeout` and `schedule_timeout` are off unless the task's author sets them | Spec choice (review gap) | 01 §5 |
| D11 | Worker reconnect grace period | An engine setting, 30 seconds unless changed. This is the only waiting period the engine invents | Spec choice (review gap) | 01 §6 |
| D12 | Local step results in durable-enqueue mode | A workflow outcome that saves a local step result is flushed to disk before anything depends on it | Spec choice (review gap) | 05 §2 |
| D13 | How workflow calls wait and retry | They wait for host capacity only, are sent ahead of task calls, and retry under the workflow's own `retries` options | Spec choice (review gap) | 01 §3, §5 |
| D14 | A per-key limit used by a run with no key | The step fails with `key_required` without running | Spec choice (review gap) | 02 §3 |
| D15 | Cancellation | Queued work is dropped at once. Running calls are told to stop, and their limit units return only when they end | Spec choice (review gap) | 01 §7 |
| D16 | Attempts lost because the engine itself restarted | Recorded, retried, and not counted against `retries` | Spec choice | 01 §5 |
| D17 | Schedule time zone | Cron expressions are read in UTC only | Spec choice | 01 §9 |
| D18 | Licence | None chosen. The repository has no licence file until Matt picks one | Open (17.4) | |

Settled since the handoff: the name is **geartrain**, the repository is `mdrideout/geartrain`, and the default branch is `master`. **Decided.**

## 4. Differences from the handoff

These come from the gaps found when the handoff was reviewed.

| Handoff | This specification | Why |
|---|---|---|
| A default `execution_timeout` is implied (6.4, 6.6) | No default. A task without one is never timed out (D10) | Matt's rule against arbitrary timeouts |
| The probe recovers a stale worker by lease expiry (6.9, Appendix B) | There are no leases. An attempt ends by outcome, by its own timeout, or by its host being lost. A late outcome is refused by attempt token (01 §4) | The main design dropped leases |
| "A lost step result only causes a repeat" (6.3) | True for task results under idempotent tasks. Not true for local `ctx.step` results, so those are flushed (D12) | A regenerated identifier would differ from the one a task already used |
| Workflow calls "never wait in a queue" (6.5) | They wait for host capacity, oldest first, and hold no limits (D13) | The handoff did not say what happens when the host is full |
| HTTP statuses for invoked services omit `continue_as_new` (7.3) | An outcome body with its own content type is authoritative. The status is read only when that body is absent (03 §6.1) | Covers every outcome without inventing statuses |
| Status 206 means `suspended` and 429 means `limited` (7.3) | Neither can be expressed by a status alone, so both need the outcome body. A bare 429 is a retryable error that honours `Retry-After` (03 §6.1) | A bare status cannot name the limit or list the wanted steps |
| Catalog entries have no removal path | An administrator removes a retired name with `DELETE /v1/catalog/{name}` (03 §2) | Otherwise a deleted task would block definitions changes forever |
| Rate-limit burst is not in the definitions format (6.7) | `burst` is an optional field and defaults to the rate's count (02 §3) | The probe tested 3 per minute with a burst of 3 |
| Schema migrations and backup are not covered | 04 §6 and 05 §5 | |

## 5. Scope

In scope for the first milestone (phases 1 to 3 plus one SDK): inline steps, both delivery modes, the explicit queue layer, the three durability modes. **Decided.**

Not in this specification:

- `ctx.wait_for_event` and external events. Later phase.
- A workflow starting another workflow as a step. A step runs a task, a local function or a sleep.
- Everything in the handoff's non-goals (3.2): no embedded form, no application access to the engine's file, no polling, no gRPC, no hosted application code, no event bus, no expression language, no multi-node engine, no arbitrary caps.

## 6. Phases

Unchanged from handoff 11.

| Phase | Delivers |
|---|---|
| 1. Specification | These documents |
| 2. Engine core | Single-writer store with group commit and the three modes; definitions; queues, limits and fairness; run, step and attempt state; retries; timers and schedules; idempotent start; cancellation; crash recovery |
| 3. Delivery | Connected stream; signed HTTP invocation; outcome handling; lost-host detection |
| 4. SDKs | Client, workflow replay, connected worker and service adapter, with the conformance suite |
| 5. Later workflow features | Waiting for external events; SDK support for usage correction and provider limit signals |
| 6. Operations | Status interface and command-line tool, metrics, drain, backup, retention |

## 7. Conventions used in these documents

- **Time.** The engine stores times as integer milliseconds since the Unix epoch, UTC. The API carries times as RFC 3339 strings in UTC and durations as numbers of seconds.
- **Payloads.** Inputs, results and saved step values are JSON values. The engine stores them and never interprets or logs them. There is no size cap.
- **Identifiers.** Run ids, worker ids and call ids are opaque strings made by the engine. Clients must not parse them.
- **Names.** Workflow, task, queue, limit, schedule and service names are non-empty strings of letters, digits, `.`, `-` and `_`. Workflows and tasks share one namespace, so a name starts exactly one kind of run.
- **"Must"** marks a rule an implementation is tested against.
