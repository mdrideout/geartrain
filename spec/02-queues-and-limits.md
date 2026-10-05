# 02. Queues, limits and the definitions file

**Decided:** queue rules live in one explicit layer, not spread through application code. **Default D2:** the model below is the handoff's proposal (P1 to P5), adopted as written.

The split is by who knows what:

| Information | Who knows it | Where it lives |
|---|---|---|
| The order of steps | The workflow author | The workflow function |
| What one piece of work consumes | The author of that code | On the task, as limit names and unit counts |
| How much exists and who goes first | The operator | The definitions file |

Code states facts about itself. The definitions file states policy. No number that expresses policy appears in application code.

Two rules:

- **Only tasks queue.** A workflow holds nothing while it sleeps or waits between steps.
- **A queue orders; a limit caps.** A queue is a waiting line with a turn-taking rule. A limit is a named pool of units that tasks in any queue draw from.

## 1. The definitions file

**Default D3.** The definitions are one TOML file. An administrator applies it with `PUT /v1/definitions`, and the engine stores it. Applications cannot create or change queues, limits, schedules or services through the API. Changing a limit needs no application deploy.

The file has four kinds of table: `limit`, `queue`, `schedule` and `service`.

```toml
[limit.owner]              # one piece of work per owner at a time
slots   = 1
per_key = true

[limit.x-write]            # three provider write calls per minute per owner
rate    = "3/minute"
per_key = true

[limit.build-disk]         # one archive build at a time on this machine
slots = 1

[queue.owner-work]
order = "round-robin"      # owners take turns
uses  = { owner = 1 }      # applies to every task in this queue
tasks = ["sync.page", "archive.apply", "delete.item", "billing.charge", "erasure.page"]

[queue.archive-build]
uses  = { build-disk = 1 }
tasks = ["archive.build"]

[schedule.stale-delete-scan]
cron     = "*/15 * * * *"
workflow = "delete.reconcile"
overlap  = "skip"

[service.billing]
url            = "https://billing.internal.example/geartrain"
secret_env     = "GEARTRAIN_SECRET_BILLING"
max_concurrent = 8
```

An unknown table or field is an error. The engine never guesses.

## 2. Queues

| Field | Required | Meaning |
|---|---|---|
| `order` | No | `"fifo"` (if absent) or `"round-robin"` |
| `uses` | No | Limit names and unit counts taken by every task in this queue |
| `tasks` | Yes | The task names that wait in this queue |

**Default D1.** A task's queue is set here, in `tasks`, and nowhere else. A task in code never names a queue.

- A task name belongs to exactly one queue. Listing it in two is an error.
- `tasks` may name a task that no host has registered yet, so definitions can be applied before a deploy.
- A queue has a paused flag, set through the API and not in the file ([03](03-protocol.md) §2). A paused queue accepts new entries and starts none. Attempts already running finish normally. This is how an operator drains work before an upgrade.

## 3. Limits

A limit is one of two kinds. Exactly one of `slots` and `rate` must be present.

| Field | Meaning |
|---|---|
| `slots = N` | A pool of N units. Units come back when the attempt ends |
| `rate = "N/period"` | A token bucket that gains N units per period. Period is `second`, `minute`, `hour` or `day` |
| `burst = N` | Rate limits only. The bucket's capacity. If absent it equals the rate's N |
| `per_key = true` | One pool per key instead of one shared pool. False if absent |

A new rate bucket starts full.

### 3.1 What a task claims

A task's claim is its queue's `uses` plus the task's own `uses`, summed per limit name.

- The task's own `uses` is declared in code as limit names and unit counts, for example `uses={"x-write": 1}`. These are facts about the code: "this makes one provider write call".
- When the amount depends on the input, the workflow passes an estimate with the step, which replaces the declared count for that one call. It may name only limits the task already declares.
- Sharing is visible. Everything that uses `owner` shares it, across queues.

### 3.2 Keys

- The key is supplied by whoever starts the run and is inherited by that run's tasks. **Default D2 (handoff P4).**
- A run with no key is ordered as one turn-taker of its own in a round-robin queue.
- **Default D14.** If a task with no key would claim a `per_key` limit, its step fails with the non-retryable code `key_required` and the task does not run. Pooling all keyless work under one hidden key would serialise it silently, which is the kind of implicit sharing this layer exists to prevent.

### 3.3 The limit lifecycle

**Default D2 (handoff P5).**

1. **Declare.** The task names what it uses (3.1).
2. **Take.** Before a task starts, the engine takes everything it claims in one transaction, all or nothing. If anything is short, the task waits and others pass it. Nothing runs while it waits.
3. **Give back.** Slot units return when the attempt ends, in every way an attempt can end ([01](01-semantics.md) §4.2), because each claim row belongs to an attempt. Rate units never return; the bucket refills with time.
4. **Correct.** An `ok` outcome may carry `used`, the real amount per limit. For a rate limit the engine adjusts the bucket by the difference between what was taken and what was used; the bucket may go below zero and is never raised above its capacity. `used` for a slots limit is ignored.
5. **Listen to the provider.** A `limited` outcome names a limit the task claims and a time T. The engine closes that pool (for that key, if the limit is per key) until T, releases the attempt's units, and re-queues the task for T without counting a failed attempt. A `limited` outcome naming a limit the task does not claim is treated as a retryable error with code `bad_outcome`.

One thread owns the database, so step 2 cannot deadlock or leave a partial hold.

Steps 4 and 5 are fields of the outcome and are in the protocol from the start. SDK support for them may land in phase 5.

## 4. Claiming

The engine keeps one small row per (queue, key): whether that key has ready work and since when, when it was last served, and until when it is parked. Claiming never scans entries.

The dispatcher runs only when something changed: new work, an attempt ending, a host gaining capacity, a timer firing, a definitions change, a queue being resumed. **Decided (no polling).**

To claim one entry from a queue that is not paused:

1. **Pick a key.** Among the queue's keys that have ready work and are not parked:
   - `round-robin`: the key served least recently.
   - `fifo`: the key whose best ready entry comes first by priority, then ready time, then id.
2. **Take that key's best ready entry:** highest priority, then earliest ready time, then lowest id.
3. **Work out its claim** (3.1).
4. **Check every limit in the claim** for that key:
   - a `slots` limit needs enough free units;
   - a `rate` limit is refilled for the time elapsed, up to its burst, and then needs enough tokens;
   - a closed limit is unavailable until its reopening time.
5. **Check for a host** of that task with a free slot.
6. **If everything is available:** take all the units, create the attempt, mark the entry running and stamp the key as served. One transaction.
7. **If a limit is short:** park that (queue, key) and move to the next key. A rate limit or a closed limit gives an exact time, which is stored as a timer. A slots limit has no time; the key is unparked when any attempt holding that limit for that key ends.
8. **If only a host is missing:** stop claiming for that task name until a host of it gains a slot or registers.

Several entries are claimed in one transaction when several can start.

**Default D8.** In a round-robin queue, priority orders work inside one key only: a key with high-priority work still waits its turn. In a fifo queue, priority orders the whole queue. This is the prototype's behaviour.

Choosing a host: among hosts of the task that have a free slot, the one sent to least recently.

Behaviours the semantics tests must prove, ported from the handoff's probe (Appendix B) and extended for limits:

- One key flooding a round-robin queue does not starve other keys.
- A per-key `slots` limit of one holds throughout.
- A repeated idempotency key returns the original run.
- An outcome for an ended attempt is refused.
- Work on a lost host is retried.
- A rate of 3 per minute with a burst of 3 is honoured.
- Taking is all or nothing across several limits.
- Units are released on every kind of attempt ending.

## 5. Schedules and services

### 5.1 Schedules

| Field | Required | Meaning |
|---|---|---|
| `cron` | Yes | Five fields, UTC (D17) |
| `workflow` or `task` | Yes, exactly one | The name to start |
| `input` | No | A TOML value converted to JSON. `null` if absent |
| `key` | No | The run's key |
| `priority` | No | The run's priority |
| `overlap` | No | `"allow"` (if absent) or `"skip"` |
| `missed` | No | `"run-once"` (if absent, D7) or `"skip"` |

Behaviour is in [01](01-semantics.md) §9.2.

### 5.2 Services

A service is a host the engine calls over HTTP.

| Field | Required | Meaning |
|---|---|---|
| `url` | Yes | The one URL the engine posts calls to |
| `secret_env` | Yes | The name of an environment variable of the engine process that holds the signing secret. The secret itself is never in the file or the database |
| `max_concurrent` | Yes | The most calls the engine will have open to this service at once |

The engine learns what a service hosts by asking it ([03](03-protocol.md) §7). This is done at deploy time, because a service scaled to zero cannot announce itself.

## 6. Applying definitions

`PUT /v1/definitions` replaces the whole set in one transaction, or changes nothing and reports every error found.

The engine refuses definitions that:

- fail to parse, or contain an unknown table or field;
- route one task to two queues;
- name a limit in a queue's `uses` that the file does not define;
- leave a registered task without a queue;
- remove a limit that a registered task's own `uses` names;
- name a `secret_env` that is not set in the engine's environment.

In the other direction, the engine refuses to register a task that has no queue in the current definitions, or whose `uses` names a limit that does not exist ([03](03-protocol.md) §4).

Effects on work in flight:

| Change | Effect |
|---|---|
| A task moves to another queue | Its waiting entries move with it. Running attempts are untouched |
| A limit's size is lowered below what is in use | Nothing is interrupted. No new units are taken until use falls below the new size |
| A limit's size is raised | Parked keys waiting on it are unparked |
| A limit changes kind | Its state is reset: a rate bucket starts full, a slots pool is recounted from open claims |
| A queue's `uses` changes | Applies to claims made from then on |
| A schedule is added or its cron changes | Its next firing is computed from now |
| A schedule is removed | Its timer is removed. Runs it already started are untouched |
| A service is removed | Refused while any registered name is hosted only by that service and has work waiting or running |

Applying definitions counts as accepting new work for durability ([05](05-durability-and-recovery.md) §2).
