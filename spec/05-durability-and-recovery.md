# 05. Durability, recovery and operation

## 1. The store

- **Decided.** One engine process, always on. It owns the timers and the disk.
- **Decided.** One dedicated OS thread owns the only SQLite connection. Every read and write reaches it as a command on a channel.
- **Default D9.** There is no second, read-only connection. It can be added under WAL if reads ever become a measured bottleneck.
- **Group commit.** Commands that arrive while a commit is in flight are executed together in the next transaction. Each caller is answered only after that transaction commits.
- **Nothing leaves before its commit.** An API answer, a call to a host and a timer being armed all wait for the transaction that produced them.
- WAL needs a local disk. A cloud block device is fine. A network filesystem is not supported.

## 2. Durability modes

**Decided:** three modes, selected by one setting, with WAL in every mode. The default is durable enqueue.

| Mode | SQLite sync level | What waits for the disk | After a power cut |
|---|---|---|---|
| `relaxed` | `NORMAL` | Nothing | Recently acknowledged work can be lost |
| `durable-enqueue` (default) | `FULL` for transactions that accept new work, `NORMAL` otherwise | Accepting new work | No accepted work is lost. A task may run twice |
| `strict` | `FULL` always | Every state change | No accepted work is lost. Fewest repeats |

All three survive a crash of the engine process with nothing lost. The modes differ only on power loss or an operating-system crash.

### 2.1 What counts as accepting new work

In `durable-enqueue` mode a batch is committed at `FULL` if it contains any of these, and at `NORMAL` otherwise:

| Command | Why |
|---|---|
| Starting a run | The caller was told the run exists |
| Applying definitions | The operator was told the policy changed |
| Syncing a service's catalog | The operator was told the catalog is stored |
| A workflow outcome whose `saved` is not empty | **Default D12**, below |

Everything else is committed at `NORMAL`: task outcomes, workflow outcomes with nothing saved, claims, timer firings, schedule firings, worker registration. Losing one of those to a power cut makes the engine repeat something, which at-least-once delivery already allows:

- a lost claim or task outcome means the task runs again;
- a lost schedule firing fires again, because the stored next firing time was not advanced, and its idempotency key prevents a double start;
- a lost worker registration does not matter, because a power cut closes every stream and workers register again.

**Default D12.** A local step exists to pin down a value that must not change between replays, such as a generated identifier. If its saved value were lost after a task had already been sent that value, the replay would generate a different one, and the task would run again with different input. That is not a repeat. So a workflow outcome that saves a local result is flushed at `FULL`, and because nothing leaves before its commit, no task can receive the value before it is on disk. The cost is one flush per workflow call that executed a local step.

What remains in `durable-enqueue` mode, and must be stated in the documentation: after a power cut a task may run again, and a task that is idempotent in effect may still return a different result the second time. Work that depended on the first result is lost in the same power cut, because SQLite loses whole transactions from the end of the log and never from the middle. So the engine's own records stay consistent, but the outside world may have seen effects of the lost work. `strict` mode removes this.

### 2.2 Implementation rules

- `PRAGMA journal_mode=WAL` always. WAL at `FULL` is as durable as the rollback journal and was measured about three times faster.
- In `durable-enqueue` mode the store thread inspects each batch and sets `PRAGMA synchronous` before the transaction, switching only when the needed level changes.
- On macOS a plain `fsync` does not force data to the medium. The durable modes set `PRAGMA fullfsync=1` and `PRAGMA checkpoint_fullfsync=1`. These do nothing on other systems.
- The dispatcher claims several entries per transaction, so `strict` mode is not held to one flush per task.

## 3. Engine configuration

Configuration is what the engine needs before it can open its file. It is separate from the definitions file, which is policy stored in the database.

The engine reads one TOML file, named on the command line.

| Setting | Meaning | If absent |
|---|---|---|
| `database` | Path of the SQLite file | Required |
| `listen` | Address and port to serve on | Required |
| `durability` | `relaxed`, `durable-enqueue` or `strict` | `durable-enqueue` |
| `worker_reconnect_grace` | Seconds to wait for a disconnected worker (D11) | 30 |
| `stream_keepalive_seconds` | Interval between keep-alive comments on a stream | 15 |
| `[[token]]` with `role` and `value_env` | One per API token. `value_env` names the environment variable holding it | At least one `admin` token is required |

Secrets are never in a file: tokens and service signing secrets are read from environment variables named in configuration and definitions.

## 4. Recovery

After any stop, clean or not, startup rebuilds everything from the tables. Nothing in memory is trusted, because nothing in memory survived.

In order:

1. **Open and migrate.** Apply the schema version check ([04](04-data-model.md) §7). Set the journal and sync pragmas.
2. **Recompute counters** from their source rows: `limit_state.in_use` from `claim`, `run.open_wants` from `step`, `qkey.next_ready_at` from `entry`.
3. **End attempts that were in flight to services.** Their HTTP requests died with the engine. Each ends as `engine_restart`, its claims are released, and its entry is ready again at once without counting a failed attempt (D16). A workflow call that was in flight to a service makes its run ready for a workflow call again.
4. **Keep attempts on connected workers.** Every worker is marked disconnected as of now and given a fresh grace timer. A worker that reconnects in time carries on, and its open calls are re-sent. One that does not is handled as lost ([01](01-semantics.md) §6).
5. **Catch up schedules.** For each schedule whose next firing time has passed, apply its `missed` rule (D7), then store the next future firing.
6. **Re-arm the timer** from the earliest row of `timer`. Deadlines that passed while the engine was down fire immediately, in order.
7. **Start serving,** then run the dispatcher once.

Steps 2 to 5 are one transaction.

A power cut in `relaxed` or `durable-enqueue` mode may have removed the last few transactions. Recovery does not need to know: the file is a consistent earlier state, and the steps above apply to it unchanged. A worker may then report an outcome for an attempt the file no longer contains. That is answered `stale_call`, and the work is done again under a new attempt.

## 5. Backup

Only the engine opens its file, so the engine takes its own backups.

- An administrator asks for a backup to a path on the engine's machine. The store thread runs `VACUUM INTO` that path, which writes a consistent, compacted copy.
- Writes wait while it runs. The time grows with the size of the file.
- Restoring is stopping the engine, replacing the file, and starting it. Recovery (section 4) then runs as usual. Workers reconnect and register again.
- The command and its endpoint are built in phase 6. Nothing in the earlier phases may prevent it.

The engine is a single node. Its durability rests on one disk plus these backups. That is an accepted risk (handoff 16), not something this specification solves.

## 6. Verification

What phase 2 and 3 must prove, from handoff 12:

- **Semantics tests.** The list in [02](02-queues-and-limits.md) §4.
- **Deterministic core tests.** Feed the store command sequences with an injected clock and assert on state.
- **Crash injection.** Kill the engine process at every commit boundary, restart it, and check every invariant in [04](04-data-model.md) §6, plus: no accepted run is lost, and no step result is saved twice.
- **Delivery tests.** A worker that disconnects mid-task and reconnects inside and outside the grace period; a service that times out, returns each status in [03](03-protocol.md) §6.1, or is unreachable.
- **Idle check.** With a worker connected and no work, the engine uses no CPU beyond keep-alives.
- **Benchmarks.** Repeat the handoff's measurements (section 13) on Linux with a cloud block device. The existing numbers are from one laptop.

Power loss itself is hard to test. The design relies on SQLite's documented behaviour at each sync level.
