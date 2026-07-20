# JVMTI Phase 3 — Telemetry and Deep Introspection

Phase 3 extends the JVMTI probe with **X-Ray** (live object graphs), **push telemetry** (`field_changed` over IPC), and **bytecode instrumentation** (`instrument=audit|transform`).

## Attach options (Phase 3)

| Option | Default | Description |
|--------|---------|-------------|
| `xray=1` | off | Deep object graph walk during dump / snapshots |
| `xray_depth=` | `4` | Max recursion depth |
| `xray_max_nodes=` | `256` | Max nodes per instance |
| `sample=1` | on | Instance sampling (forced on when `xray=1`) |
| `telemetry=1` | off | FieldModification watches + push events |
| `telemetry_coalesce_ms=` | `50` | Coalesce window per `(class, field)` |
| `instrument=` | `off` | `audit` (log pass-through) or `transform` (ASM probes) |
| `instrument_allow=` | — | Comma-separated class prefix allowlist for transform |

Host `ProbeConfig` and `build_agent_options` pass `xray`, `telemetry`, `instrument`, and `instrument_allow` through to the agent.

## IPC protocol v3

### Host → Agent commands

| `op` | Payload | Behavior |
|------|---------|----------|
| `read_instances` | `object_ids?` | Warm re-read of registered instances |
| `watch_fields` | `classes[]`, `fields[]` | Enable field watches (returns `watch_id`) |
| `unwatch_fields` | `watch_id` | Tear down watches |
| `dump` / `invoke` / `shutdown` | (Phase 2) | Unchanged |

### Agent → Host events

| `op` | When |
|------|------|
| `object_snapshot` | X-Ray walk per instance |
| `field_changed` | Push telemetry / bytecode probe |
| `method_exit` | Bytecode method-entry probe (see below) |
| `telemetry_overflow` | Bounded queue backpressure |
| `instrument_applied` | ASM transform success |
| `instrument_skipped` | Policy block or transform error |

## X-Ray

- Agent: `object_graph.rs` walks instance fields with depth/cycle/collection caps.
- Registry: `object_id` + JVMTI tag + `NewGlobalRef` for warm `read_instances`; released on shutdown.
- Normalizer: `MAPPING_RULES` map e.g. `com.nelieo.probe.Customer` → `customer`.
- Watcher: warm cycles call `read_instances` when anchor valid and `xray=1`.

### Benchmarks (jvm-test-app)

| Metric | Target | Script |
|--------|--------|--------|
| Warm `read_instances` p99 | < 50ms | `jvmti-xray-bench.ps1` |
| Cold X-Ray attach | < 2s Tier A | enterprise stress |

## Push telemetry

**Path A — FieldModification:** enabled when `telemetry=1`; `watch_fields` IPC narrows targets.

**Path B — ASM probes:** `instrument=transform` + `tools/jvmti-injector` JAR; `AxonProbe.emitMethodEntry` / `emitFieldChange` natives enqueue to the same telemetry queue.

**MethodExit (native JVMTI):** intentionally not registered — requires per-method allowlists and adds hot-path callback overhead. Bytecode probes emit `method_exit`-shaped IPC events instead (Path B fallback per §7.2 decision tree).

- Host: `JvmSession::drain_telemetry()`, `watch_fields()`, `merge_field_delta()`.
- Watcher: IPC poll wakes `force_snapshot_tx`; probe cycle merges deltas.
- API: `execute_jvm` with `verify=true` returns `state_delta` from `field_snapshot`.

## Bytecode instrumentation

| Mode | Behavior |
|------|----------|
| `audit` | Log class + byte length; pass-through |
| `transform` | ASM method-entry probes via `injector_bridge.rs` + JAR |

Safety (`policy.rs`): skip `java.*`/`jdk.*`, classes > 4MB, non-allowlisted classes; pass-through on transform error.

Set `AXON_JVMTI_INJECTOR_PATH` to the built JAR or build with `mvn -f tools/jvmti-injector package`.

## CLI

```powershell
axon jvm dump <pid> --xray
axon jvm xray <pid> [--object-id obj-1]
axon jvm watch <pid> --telemetry --fields total,status
```

## Tests

| Script | Assert |
|--------|--------|
| `jvmti-xray-test.ps1` | `ada@example.com` without invoke |
| `jvmti-xray-spring-e2e.ps1` | `/state` live `Order.total` |
| `jvmti-xray-bench.ps1` | Warm p99 < 50ms |
| `jvmti-push-field-test.ps1` | `field_changed` IPC |
| `jvmti-push-ws-test.ps1` | WS notification within SLA |
| `jvmti-instrument-*-test.ps1` | audit / transform / retransform |

## Degradation

All Phase 3 features default **off** — Phase 2 attach behavior is unchanged without explicit flags.
