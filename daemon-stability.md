# Daemon stability (Phase 4 P1)

## Goals

- `axon-api` must not hang when the JVM dies mid-probe.
- Named pipes must not block re-attach after process respawn.
- Python SDK receives predictable HTTP semantics for degraded/detached probes.

## SDK contract

| HTTP | Meaning | SDK action |
|------|---------|------------|
| 200 + `probe_metadata.extra.degraded=true` | Stale/partial state | Retry `GET /state` after 2s |
| 503 `PROBE_DETACHED` | PID gone | Stop waiting; refresh app list |
| 503 + `retry_after_ms=2000` (IPC disconnect) | Agent pipe died | Retry with backoff |
| 504 | `verify_timeout` exceeded | Surface to agent |

## Implementation

- `ProbeError::IpcDisconnected` in `axon-types`
- `PersistentSession::is_connected()` + reader-thread disconnect flag
- Watcher `probe_jvm` clears `jvm_session_slot` + anchor on IPC loss → `ProbeStatus::Degraded`
- API `probe_ipc_disconnected()` → 503 + `Retry-After`

## Verification

```powershell
./scripts/daemon-jvm-crash-test.ps1
./scripts/daemon-jvm-pipe-zombie-test.ps1
./scripts/daemon-jvm-respawn-e2e.ps1
```
