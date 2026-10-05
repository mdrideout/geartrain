# 04. Data model

The SQLite schema. **Decided:** one SQLite file that only the engine opens, through one connection on one thread. **Default D2 (handoff P10):** the tables follow the handoff's draft (section 8).

The SQL below fixes the tables, their meaning and their invariants. Phase 2 may adjust a column's type or add a column when the code needs it, and must update this document in the same change.

## 1. Conventions

- Times are `INTEGER` milliseconds since the Unix epoch, UTC.
- JSON values are stored as `TEXT`, exactly as received.
- "No key" is stored as the empty string, so key columns are never `NULL` and can sit in primary keys. The API never shows the empty string as a key.
- Boolean columns are `INTEGER` 0 or 1.
- Store functions take `now` as an argument and never read the clock. The scheduling core is then a deterministic function of a command sequence and an injected clock, testable without a network or real time.

## 2. Definitions

Written only by applying the definitions file ([02](02-queues-and-limits.md) §6), except `queue_def.paused`.

```sql
CREATE TABLE meta (
  key   TEXT PRIMARY KEY,
  value TEXT NOT NULL
) WITHOUT ROWID;
-- Rows: 'schema_version', 'definitions_toml' (the file as last applied), 'definitions_applied_at'.

CREATE TABLE limit_def (
  name      TEXT PRIMARY KEY,
  kind      TEXT NOT NULL CHECK (kind IN ('slots', 'rate')),
  capacity  INTEGER NOT NULL,      -- slots: pool size. rate: burst
  rate_n    INTEGER,               -- rate only: units gained per period
  period_ms INTEGER,               -- rate only
  per_key   INTEGER NOT NULL DEFAULT 0
) WITHOUT ROWID;

CREATE TABLE queue_def (
  name   TEXT PRIMARY KEY,
  ord    TEXT NOT NULL CHECK (ord IN ('fifo', 'round-robin')),
  paused INTEGER NOT NULL DEFAULT 0,
  seq    INTEGER NOT NULL DEFAULT 0   -- counter that stamps qkey.last_served
) WITHOUT ROWID;

CREATE TABLE queue_uses (
  queue      TEXT NOT NULL REFERENCES queue_def(name),
  limit_name TEXT NOT NULL REFERENCES limit_def(name),
  units      INTEGER NOT NULL,
  PRIMARY KEY (queue, limit_name)
) WITHOUT ROWID;

CREATE TABLE route (
  task  TEXT PRIMARY KEY,           -- a task waits in exactly one queue
  queue TEXT NOT NULL REFERENCES queue_def(name)
) WITHOUT ROWID;

CREATE TABLE schedule (
  name         TEXT PRIMARY KEY,
  cron         TEXT NOT NULL,
  target       TEXT NOT NULL,       -- workflow or task name
  input        TEXT NOT NULL,       -- JSON
  key          TEXT NOT NULL DEFAULT '',
  priority     INTEGER NOT NULL DEFAULT 0,
  overlap      TEXT NOT NULL CHECK (overlap IN ('allow', 'skip')),
  missed       TEXT NOT NULL CHECK (missed IN ('run-once', 'skip')),
  next_fire_at INTEGER NOT NULL,
  last_run_id  TEXT                 -- first run of the chain it last started; used by overlap = 'skip'
) WITHOUT ROWID;

CREATE TABLE service (
  name           TEXT PRIMARY KEY,
  url            TEXT NOT NULL,
  secret_env     TEXT NOT NULL,     -- name of the environment variable; never the secret
  max_concurrent INTEGER NOT NULL,
  synced_at      INTEGER
) WITHOUT ROWID;
```

## 3. Catalog and hosts

Written by worker registration and service sync.

```sql
CREATE TABLE task_def (
  name                 TEXT PRIMARY KEY,
  retries              INTEGER NOT NULL DEFAULT 0,
  backoff_factor       REAL,        -- NULL: retry immediately
  backoff_max_ms       INTEGER,     -- NULL: no cap
  execution_timeout_ms INTEGER,     -- NULL: the engine's default applies
  schedule_timeout_ms  INTEGER      -- NULL: waits indefinitely
) WITHOUT ROWID;

CREATE TABLE task_uses (
  task       TEXT NOT NULL REFERENCES task_def(name) ON DELETE CASCADE,
  limit_name TEXT NOT NULL REFERENCES limit_def(name),
  units      INTEGER NOT NULL,
  PRIMARY KEY (task, limit_name)
) WITHOUT ROWID;

CREATE TABLE workflow_def (
  name                 TEXT PRIMARY KEY,
  retries              INTEGER NOT NULL DEFAULT 0,
  backoff_factor       REAL,
  backoff_max_ms       INTEGER,
  execution_timeout_ms INTEGER      -- NULL: the engine's default applies
) WITHOUT ROWID;
-- A name is in task_def or workflow_def, never both. Enforced in code.

CREATE TABLE worker (
  id              TEXT PRIMARY KEY,
  name            TEXT NOT NULL,
  capacity        INTEGER NOT NULL,
  connected       INTEGER NOT NULL,
  disconnected_at INTEGER,          -- set when the stream closes; the grace timer runs from here
  registered_at   INTEGER NOT NULL
) WITHOUT ROWID;

CREATE TABLE serves (
  host_kind TEXT NOT NULL CHECK (host_kind IN ('worker', 'service')),
  host      TEXT NOT NULL,          -- worker.id or service.name
  name      TEXT NOT NULL,          -- task or workflow name
  PRIMARY KEY (name, host_kind, host)
) WITHOUT ROWID;
```

A worker row and its `serves` rows are deleted when its grace period passes. Catalog rows stay until an administrator removes the name, so a name remains startable while its workers are briefly away.

## 4. Runs, steps, entries and attempts

```sql
CREATE TABLE run (
  id             TEXT PRIMARY KEY,
  name           TEXT NOT NULL,
  kind           TEXT NOT NULL CHECK (kind IN ('workflow', 'task')),
  input          TEXT NOT NULL,     -- JSON
  key            TEXT NOT NULL DEFAULT '',
  priority       INTEGER NOT NULL DEFAULT 0,
  state          TEXT NOT NULL CHECK (state IN
                   ('queued', 'running', 'waiting', 'completed', 'failed', 'cancelled')),
  result         TEXT,              -- JSON, when completed
  error_code     TEXT,              -- when failed
  error_message  TEXT,
  idem_key       TEXT,
  start_at       INTEGER,
  call_ready_at  INTEGER,           -- not NULL: a workflow call is waiting for host capacity
  call_failures  INTEGER NOT NULL DEFAULT 0,  -- failed workflow calls since the last suspension
  open_wants     INTEGER NOT NULL DEFAULT 0,  -- steps from the last suspension not yet resolved
  continued_from TEXT REFERENCES run(id),
  continued_as   TEXT REFERENCES run(id),
  schedule       TEXT,              -- the schedule that started it, if any
  created_at     INTEGER NOT NULL,
  started_at     INTEGER,
  finished_at    INTEGER
) WITHOUT ROWID;

CREATE UNIQUE INDEX run_idem ON run(name, idem_key) WHERE idem_key IS NOT NULL;
CREATE INDEX run_call_ready ON run(name, call_ready_at) WHERE call_ready_at IS NOT NULL;

CREATE TABLE step (
  run_id        TEXT NOT NULL REFERENCES run(id),
  name          TEXT NOT NULL,
  seq           INTEGER NOT NULL,   -- order of creation within the run
  kind          TEXT NOT NULL CHECK (kind IN ('task', 'local', 'sleep')),
  task          TEXT,               -- task steps only
  state         TEXT NOT NULL CHECK (state IN
                  ('waiting', 'running', 'resolved', 'failed', 'cancelled')),
  result        TEXT,               -- JSON, when resolved
  error_code    TEXT,               -- when failed
  error_message TEXT,
  created_at    INTEGER NOT NULL,
  resolved_at   INTEGER,
  PRIMARY KEY (run_id, name)
) WITHOUT ROWID;

-- One task call waiting or running in a queue. Deleted when its step resolves, fails or is cancelled.
CREATE TABLE entry (
  id          INTEGER PRIMARY KEY,
  run_id      TEXT NOT NULL,
  step        TEXT NOT NULL,
  task        TEXT NOT NULL,
  queue       TEXT NOT NULL,
  key         TEXT NOT NULL DEFAULT '',
  priority    INTEGER NOT NULL DEFAULT 0,
  state       TEXT NOT NULL CHECK (state IN ('ready', 'running')),
  ready_at    INTEGER NOT NULL,     -- covers start_at, retry times and limit reopenings
  input       TEXT NOT NULL,        -- JSON
  uses        TEXT,                 -- JSON estimate passed with the step, if any
  failures    INTEGER NOT NULL DEFAULT 0,   -- failed attempts so far, compared with retries
  attempts    INTEGER NOT NULL DEFAULT 0,   -- all attempts so far; numbers the next one
  created_at  INTEGER NOT NULL,
  FOREIGN KEY (run_id, step) REFERENCES step(run_id, name)
);

CREATE UNIQUE INDEX entry_step ON entry(run_id, step);
CREATE INDEX entry_ready ON entry(queue, key, priority DESC, ready_at, id) WHERE state = 'ready';

-- One per call, of either kind. Never deleted while its run is kept.
CREATE TABLE attempt (
  id            INTEGER PRIMARY KEY,
  token         TEXT NOT NULL UNIQUE,   -- sent as call_id
  run_id        TEXT NOT NULL REFERENCES run(id),
  kind          TEXT NOT NULL CHECK (kind IN ('task', 'workflow')),
  step          TEXT,                   -- task attempts only
  number        INTEGER NOT NULL,
  host_kind     TEXT NOT NULL CHECK (host_kind IN ('worker', 'service')),
  host          TEXT NOT NULL,
  host_name     TEXT NOT NULL,          -- kept for the record after a worker row is deleted
  queued_at     INTEGER NOT NULL,       -- when this attempt's wait began
  started_at    INTEGER NOT NULL,
  timeout_at    INTEGER NOT NULL,       -- started_at plus the execution timeout in force
  finished_at   INTEGER,                -- NULL: the attempt is open
  outcome       TEXT CHECK (outcome IN ('ok', 'error', 'limited', 'suspended',
                  'continue_as_new', 'timeout', 'lost', 'engine_restart')),
  cancelled     INTEGER NOT NULL DEFAULT 0,
  error_code    TEXT,
  error_message TEXT,
  next_retry_at INTEGER
);

CREATE INDEX attempt_open ON attempt(host_kind, host) WHERE finished_at IS NULL;
CREATE INDEX attempt_run ON attempt(run_id, step, number);
```

## 5. Scheduling state

```sql
-- One row per (queue, key): everything claiming needs, so it never scans entries.
CREATE TABLE qkey (
  queue         TEXT NOT NULL,
  key           TEXT NOT NULL,
  next_ready_at INTEGER,            -- earliest ready_at among this key's ready entries; NULL if none
  last_served   INTEGER NOT NULL DEFAULT 0,   -- value of queue_def.seq when last served
  parked_until  INTEGER,            -- rate or closed limit: the time it can next be available
  parked_on     TEXT,               -- slots limit: the limit whose release unparks this key
  PRIMARY KEY (queue, key)
) WITHOUT ROWID;

CREATE INDEX qkey_rr ON qkey(queue, last_served) WHERE next_ready_at IS NOT NULL;

CREATE TABLE limit_state (
  limit_name   TEXT NOT NULL,
  key          TEXT NOT NULL,       -- '' for a shared pool
  in_use       INTEGER NOT NULL DEFAULT 0,  -- slots: recomputable from claim
  tokens       REAL NOT NULL DEFAULT 0,     -- rate
  refilled_at  INTEGER NOT NULL DEFAULT 0,  -- rate
  closed_until INTEGER,             -- set by a 'limited' outcome
  PRIMARY KEY (limit_name, key)
) WITHOUT ROWID;

-- Which open attempt holds how many units of which limit.
CREATE TABLE claim (
  attempt_id INTEGER NOT NULL REFERENCES attempt(id),
  limit_name TEXT NOT NULL,
  key        TEXT NOT NULL,
  units      INTEGER NOT NULL,
  PRIMARY KEY (attempt_id, limit_name)
) WITHOUT ROWID;

CREATE INDEX claim_limit ON claim(limit_name, key);

-- Every stored deadline. The timer task sleeps until MIN(fire_at).
CREATE TABLE timer (
  id      INTEGER PRIMARY KEY,
  fire_at INTEGER NOT NULL,
  kind    TEXT NOT NULL CHECK (kind IN (
            'entry_ready',        -- ref: entry.id. A start_at, retry or reopening time arrived
            'sleep_end',          -- ref: run_id, ref2: step name
            'execution_timeout',  -- ref: attempt.id
            'schedule_timeout',   -- ref: entry.id
            'worker_grace',       -- ref: worker.id
            'cron',               -- ref: schedule.name
            'unpark')),           -- ref: queue, ref2: key
  ref     TEXT NOT NULL,
  ref2    TEXT
);

CREATE INDEX timer_fire ON timer(fire_at);
CREATE INDEX timer_ref ON timer(kind, ref);
```

The handoff left open whether deadlines are columns or one table. One `timer` table is chosen because "sleep until the earliest deadline" is then a single indexed read, and recovery re-arms from one place.

**Default D6.** No row of `run`, `step` or `attempt` is deleted automatically. Matt's rule is to add a limit only after a real problem, and a retention period chosen now would be arbitrary. Phase 6 adds a retention setting and the command that applies it. Until then the file grows with history, and idempotency keys hold forever.

## 6. Invariants

The crash-injection tests check these after every restart. They must hold after every committed transaction.

1. A step in a final state has a result or an error, and it never changes afterwards.
2. An `entry` exists for a task step exactly while that step is `waiting` or `running`.
3. A `running` entry has exactly one open attempt, and a `ready` entry has none.
4. A `claim` row exists only for an open attempt.
5. For a slots limit, `limit_state.in_use` equals the sum of `claim.units` for that limit and key. It exceeds `limit_def.capacity` only if the capacity was lowered afterwards.
6. `qkey.next_ready_at` equals the minimum `ready_at` over that queue and key's ready entries, or `NULL`.
7. Open attempts on a worker never exceed its capacity. Open attempts on a service never exceed its `max_concurrent`.
8. A run in a final state has no entries, no open timers of its own and no `call_ready_at`. It may still have open attempts flagged `cancelled`.
9. `run.open_wants` equals the number of that run's steps in `waiting` or `running`.
10. Every row with a deadline has a matching `timer` row, and every `timer` row refers to a row that exists.
11. `(name, idem_key)` is unique among runs.

Counters (`in_use`, `open_wants`, `next_ready_at`) are stored for speed and are recomputed from their source rows at startup ([05](05-durability-and-recovery.md) §4). The prototype reset its counters to zero, which was only correct because it had no recovery.

## 7. Schema migrations

- `meta.schema_version` holds an integer. A new file is created at the current version.
- At startup, before anything else, the engine compares the file's version with its own.
  - Equal: carry on.
  - Older: apply each numbered migration in order, each in its own transaction that also bumps the version. A crash mid-way resumes at the next unapplied migration.
  - Newer: refuse to start, with a clear message. An older engine never opens a newer file.
- Migrations only go forward. Going back means restoring a backup taken before the upgrade.
- Migrations are plain SQL files in the engine's source, embedded in the binary.
- The procedure for an upgrade is: pause queues, let running attempts finish, take a backup, stop the old engine, start the new one.
