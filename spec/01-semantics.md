# 01. Semantics

What runs, steps, tasks and attempts are, and how each one moves from start to finish. Labels are explained in [README.md](README.md).

## 1. Terms

| Term | Meaning |
|---|---|
| Engine | The always-on Rust service |
| Host | Application code the engine delivers work to. A **worker** is always on and holds a stream. A **service** is called over HTTP and may scale to zero |
| Workflow | A registered function that orders work as named steps |
| Task | A registered function that does real work. Tasks are the only things that wait in queues |
| Run | One execution of a workflow, or of one task started directly |
| Step | A named point in a run whose result is saved |
| Call | One delivery from the engine to a host: a **task call** or a **workflow call** |
| Attempt | The engine's record of one call, with its own token |
| Entry | One task call waiting or running in a queue |
| Key | A string naming whose work a run is, such as an owner or tenant |
| Outcome | What a host reports after a call |

## 2. Runs

### 2.1 Starting a run

A client starts a run with:

| Field | Required | Meaning |
|---|---|---|
| `name` | Yes | A registered workflow or task name |
| `input` | Yes | Any JSON value |
| `key` | No | Whose work this is. Inherited by every task the run starts. Absent means "no key" |
| `priority` | No | An integer, higher first, 0 if absent. Inherited by every task the run starts |
| `idempotency_key` | No | See 2.2 |
| `start_at` | No | Do not begin before this time |

The engine commits the run and only then answers. In the durable modes the commit has reached the disk before the answer leaves ([05](05-durability-and-recovery.md) §2).

Starting a name that no host has registered is refused with `unknown_name`.

A task started directly is a run with one step, named `main`, that runs that task with the run's input. It follows every rule for task steps below. There is no workflow call.

### 2.2 Idempotent start

An idempotency key is unique per run name. Starting a run with a name and key that already exist creates nothing and returns the existing run, whatever its state and whatever input the second request carried. The key holds for as long as the run's record is kept (D6: forever, for now).

This replaces enqueueing inside the application's own database transaction, which is impossible because the application cannot open the engine's file. **Decided.**

### 2.3 Run states

| State | Meaning |
|---|---|
| `queued` | Accepted. No call for this run has been sent yet |
| `running` | At least one call for this run, of either kind, is in flight |
| `waiting` | Started, and nothing is in flight. The run is waiting on a timer, a queue, a limit or host capacity |
| `completed` | Finished with a result. Final |
| `failed` | Finished with an error. Final |
| `cancelled` | Cancelled. Final |

A status read for an id the engine does not hold answers "not found".

The engine updates the state in the same transaction as the change that causes it.

## 3. Workflows and steps

**Decided:** a workflow is a function with inline, named steps. Conditional edges are plain `if` statements and loops are plain loops.

### 3.1 What a workflow can do

| Call | Step kind | Meaning |
|---|---|---|
| `ctx.run(step, task, input)` | `task` | Run a task and return its saved result |
| `ctx.step(step, fn)` | `local` | Run a small local function inside the workflow call and save its result |
| `ctx.sleep(step, duration)` | `sleep` | Durable sleep. Nothing runs while it waits |
| `ctx.parallel(...)` | | Start several steps and wait for all of them |
| `ctx.continue_as_new(input)` | | End this run and start a successor with empty history (section 8) |

### 3.2 How a run advances

1. The engine sends a workflow call to a host of that workflow. It carries the run's input and the saved result or final error of every step so far.
2. The SDK runs the function from the top. A step with a saved result returns it at once. A step with a saved final error raises `TaskFailed`, which the code may catch.
3. At the first step with no saved result:
   - a `local` step is executed there and then, and its result is remembered for the report;
   - a `task` or `sleep` step is recorded as wanted, and the function stops. Inside `ctx.parallel`, every branch runs to its first unresolved step before the function stops, so several steps can be wanted at once.
4. The SDK reports one outcome: `ok` with the function's result, `suspended` with the wanted steps, `continue_as_new`, or `error`. Any of the first three also carries the local results produced during this call.
5. For `suspended`, in one transaction the engine saves the local results, creates a step and a queue entry for each wanted task, creates a step and a timer for each wanted sleep, and ends the attempt.
6. Each task entry waits in its queue, is admitted against its limits ([02](02-queues-and-limits.md)) and is sent as a task call. Its result, or its final error after retries, becomes that step's saved result.
7. When **every** step wanted by the last suspension has resolved, the run is ready for its next workflow call. This repeats until the function returns.

Two properties follow:

- A task call carries only its own input. It never replays the workflow, and it may live in a different service or language.
- A workflow call is short. It runs only until the next unresolved step.

### 3.3 How workflow calls wait

**Default D13.** A workflow call belongs to no queue and takes no limit units. It waits only for a host of that workflow to have a free slot.

- A run that is ready for a workflow call is stamped with the time it became ready.
- When a host has a free slot, the engine sends ready workflow calls before task calls, oldest ready time first. Workflow calls are short and each one unblocks more work, so they go first.
- A host has one capacity number that covers both kinds of call.
- If no host of the workflow is connected or registered, the run waits. Nothing times out.

### 3.4 Rules for workflow code

The SDKs must enforce or document these.

- **Determinism.** Code between steps runs again on every workflow call. A branch may depend only on the input and on saved step results. Anything that can differ between calls (the clock, randomness, a database read) belongs inside a step.
- **Step names are the save keys.** A name must be unique within a run and the same on every replay. In a loop, put the iteration in the name.
- **History grows.** Every workflow call carries every saved result. A long loop should call `continue_as_new` periodically. No step cap is imposed.

### 3.5 Detecting a changed workflow

Changing a workflow while runs are in flight can break them. Mismatches must fail loudly.

- Each saved step travels with its kind and, for a task step, its task name.
- The SDK must report a non-retryable `error` with code `history_mismatch` when the function asks for a step whose saved kind or task name differs from what the function now requests, or when the function finishes or suspends while a saved step was never asked for.
- The engine must refuse a `suspended` outcome that wants a step name already saved in that run, and fail the run with `history_mismatch`.
- The advice to authors is to version workflow names (`delete.job.v2`) when the step structure changes.

## 4. Tasks and attempts

A task must be idempotent, because delivery is at least once. **Decided.**

### 4.1 The life of a task step

1. **Wanted.** A workflow call wants it, or a task is started directly. The engine creates the step and one entry.
2. **Ready.** The entry waits in its queue. It becomes claimable at its ready time: now, the run's `start_at`, or a retry time.
3. **Claimed.** The dispatcher admits it against its limits, takes its units, picks a host and creates an attempt. All of that is one transaction.
4. **Sent.** After that transaction commits, the call goes to the host.
5. **Ended.** The attempt ends in exactly one of the ways in 4.2. Its limit units are released in the same transaction.
6. **Resolved or retried.** Section 5 decides which. A resolved step has a saved result or a saved final error, and its entry is deleted.

### 4.2 Attempts

An attempt is the engine's record of one call. It holds a random token, sent as `call_id`. There are no leases and no heartbeats.

An attempt ends when the first of these happens:

| Ending | Recorded outcome |
|---|---|
| The host reports an outcome that quotes the token | `ok`, `error`, `limited`, `suspended` or `continue_as_new` |
| The attempt's `execution_timeout` passes, if it has one | `timeout` |
| Its host is lost (section 6) | `lost` |
| The engine restarted while the call was in flight to an invoked service | `engine_restart` |
| The run was cancelled, and the call then ended in any of the ways above | The same outcome, with the attempt flagged as cancelled |

Once an attempt has ended its token is dead. **An outcome is accepted only for an attempt that is still open.** A late outcome from a host that was timed out or declared lost is refused and changes nothing. This is the rule the prototype enforced by worker name and lease expiry, restated for attempt tokens.

Each attempt record keeps: queued, started and finished times; which host ran it; the outcome; the error code and message if any; the next retry time if any.

## 5. Retries and timeouts

The engine owns every retry decision. Application code reports an outcome and never retries itself or sleeps for backoff, so a waiting retry holds no connection and no running instance. **Default D2 (handoff P6, P7).**

### 5.1 Options

Options are stated on the task or workflow in code, because they describe that piece of code and not shared capacity. They use Hatchet's names.

| Option | Applies to | Meaning | If absent |
|---|---|---|---|
| `retries` | Tasks, workflows | Failed attempts allowed after the first | 0 |
| `backoff_factor` | Tasks, workflows | Base of the exponential backoff | Retry immediately |
| `backoff_max_seconds` | Tasks, workflows | Cap on one backoff delay | No cap |
| `execution_timeout` | Tasks, workflows | How long one attempt may take | **The attempt is never timed out** |
| `schedule_timeout` | Tasks | How long a task may wait for its first attempt to start | **The task waits indefinitely** |
| `uses` | Tasks | Limit names and unit counts ([02](02-queues-and-limits.md) §3) | Uses nothing of its own |

**Default D10.** The engine ships no default for either timeout. Matt's rule is that a limit is added only after a real, documented problem, and a timeout picked by the engine would be an arbitrary one. A value on a task is the author's statement about that code. The consequence must be documented in every SDK: a task without `execution_timeout` that hangs on a still-connected worker holds its limit units until that worker disconnects.

The backoff delay before retry number `n` (the first retry is 1) is `backoff_factor ^ n` seconds, capped at `backoff_max_seconds`. With factor 2.0 and cap 10 this gives 2 s, 4 s, 8 s, 10 s, 10 s.

The engine reads options from the catalog at the moment it needs them, so a redeploy that changes an option applies to work already waiting.

### 5.2 The outcome table

Every step has its own count of failed attempts.

| The engine sees | Counts as a failed attempt? | It does |
|---|---|---|
| `ok` | No | Saves the result under the step name and advances the run |
| Retryable `error`, retries left | Yes | Schedules the next attempt after backoff |
| `error` carrying `retry_after_seconds`, retries left | Yes | Schedules the next attempt for that time instead of the backoff |
| Non-retryable `error`, or a failed attempt with no retries left | Yes | Saves the error as the step's final result and advances the run. The workflow may catch it as `TaskFailed`. If it does not, the run fails |
| `timeout` | Yes | As a retryable error |
| `lost`, or an invoked call that failed to connect or returned no usable response | Yes | As a retryable error |
| `limited` until T | No | Closes that limit for that key until T and re-queues the task for T |
| `engine_restart` | No (**Default D16**) | Re-queues the task at once. The engine's own restart must not spend the author's retries |
| Not started within `schedule_timeout` | | Fails the step with code `schedule_timeout` without running it |

### 5.3 Workflow calls

**Default D13.** A workflow call follows the same table, using the workflow's own options and one failure count per run. The count resets whenever a workflow call ends in `suspended`.

- An `error` outcome from a workflow call means the function raised, including an uncaught `TaskFailed`. With no retries left, the run fails with that error.
- `history_mismatch` is always non-retryable.
- `limited` is not valid from a workflow call and is treated as a retryable error with code `bad_outcome`.

## 6. Lost hosts

**Connected workers.** The engine learns a worker is gone when its stream closes.

- **Default D11.** The engine then waits for the worker to reconnect, for the engine setting `worker_reconnect_grace` (30 seconds unless changed). Without a wait, every network blip would re-run work that is still running. This is a deadline for one specific known thing and is the only waiting period the engine invents.
- If the worker reconnects in time with the same worker id, its attempts carry on. The engine re-sends every call still open for that worker, because a call written to a dying connection may never have arrived. The SDK must ignore a call id it is already running, and must re-post the outcome for one it has finished.
- If the grace period passes, every open attempt on that worker ends as `lost`, and the worker's record is removed. A late reconnect is refused and the worker must register again.
- A worker process that restarts registers again and gets a new worker id. Its old id runs out its grace period.
- A worker that is connected but hung is caught only by the task's `execution_timeout`.

**Invoked services.** The HTTP request is the attempt. If the connection fails, closes without a response, or the response is unusable, the attempt ends at once. There is no grace period. The engine's request waits as long as the attempt's `execution_timeout`, or indefinitely if there is none; the platform's own request cap ends the request in practice.

## 7. Cancellation

**Default D15.** Cancelling a run is one transaction:

1. The run becomes `cancelled`. This is final and immediate.
2. Its waiting entries and its sleep timers are deleted. Their steps are marked cancelled.
3. Each open attempt is flagged as cancelled. The engine tells the host: a `cancel` event on a worker's stream, or closing the HTTP request to a service.

The engine cannot stop application code. Stopping is best effort, and the SDK exposes the signal to the task. So:

- A flagged attempt keeps its limit units until it ends in one of the normal ways (section 4.2). Releasing them earlier would let a limit be exceeded by code that is still running.
- Whatever outcome that attempt reports is recorded on the attempt and otherwise ignored. It cannot change the run.

Cancelling a run that is already final changes nothing and returns its state. Cancelling a run that was continued (section 8) cancels the latest run in its chain.

## 8. `continue_as_new`

**Default D2 (handoff P8).** A workflow ends its run and starts a successor with empty history. The engine does both in one transaction, so there is never a moment with neither or both.

- The successor has a new id and the same name, key and priority. Its input is the one passed to `continue_as_new`.
- The old run becomes `completed` with no result and a `continued_as` pointer to the successor. The successor has a `continued_from` pointer back.
- The idempotency key stays on the first run of the chain. A repeated start with that key returns the first run.
- A status read returns the run asked for, including `continued_as`. The SDK client's `status` follows the chain to the latest run unless told not to, so an application holding the first id still sees the real state.
- A schedule's overlap rule (section 9) treats the whole chain as one run.

## 9. Timers and schedules

### 9.1 Timers

Every deadline is a stored time: a run's `start_at`, a retry time, the end of a sleep, an execution timeout, a schedule timeout, a worker's grace period, a limit's refill or reopening time, and a cron firing.

One timer task sleeps until the earliest stored deadline. It is reset when an earlier deadline is stored. It never wakes on an interval. **Decided (no polling).**

Deadlines are wall-clock times. The engine does not detect clock changes: if the clock jumps forward, deadlines fire early; if it jumps back, they fire late. Elapsed time used for rate-limit refill is never taken as negative.

### 9.2 Schedules

A schedule is declared in the definitions file ([02](02-queues-and-limits.md) §5) and starts a run at each cron firing.

- The run's idempotency key is the schedule name plus the firing time, so a crash around a firing cannot start it twice.
- `overlap = "skip"` drops a firing while the schedule's previous run, or any successor in its chain, is not final. `overlap = "allow"`, the value if absent, starts a run at every firing.
- **Default D17.** Cron expressions have five fields and are read in UTC.
- **Default D7.** When the engine starts and finds that one or more firings of a schedule were missed while it was down, it starts one run for that schedule, keyed to the latest missed firing, and then resumes the normal timetable. The overlap rule still applies. A schedule may set `missed = "skip"` to start nothing instead.

## 10. Guarantees

- Every accepted task runs **at least once**. It can run more than once if its success report is lost, if it is timed out or its host is declared lost while it is in fact still running, or after a power cut in the modes that allow it ([05](05-durability-and-recovery.md) §2).
- A step has **at most one saved result**. Once saved it never changes.
- **No limit is over capacity** as far as the engine's own records go. Units are tied to attempts and return when the attempt ends.
- An outcome is accepted **only from an open attempt**.
- Accepted runs survive an engine crash in every durability mode.
