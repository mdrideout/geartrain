# geartrain

A small always-on service that runs durable workflows and queued tasks for other applications.

- Rust, with one SQLite file that only the engine opens. No Postgres, no Redis.
- Push delivery with no polling: always-on workers hold one outbound stream, and services that scale to zero are called over signed HTTP.
- Workflows are ordinary functions with named steps. The engine saves each step's result.
- The engine owns every retry and records every attempt.
- Queues, limits and schedules live in one definitions file owned by the engine.
- Three durability modes trade latency against what survives a power cut.

## State

Specification only. No engine code exists yet. The specification in [spec/](spec/README.md) is the source of truth and is written before any engine code.

## Layout

```
spec/    protocol, semantics, definitions-file format, data model, durability
```

`engine/`, `sdk/`, `tests/` and `examples/` are added in later phases. See [spec/README.md](spec/README.md) for the phase plan.

## Licence

MIT. See [LICENSE](LICENSE).
