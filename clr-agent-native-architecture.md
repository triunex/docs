# CLR agent-native architecture

In-process bridge (production path):

| Layer | Component | Role |
|-------|-----------|------|
| A — Profiler COM shim | `tools/axon-clr-profiler` (`ICorProfilerCallback`) | Earliest CLR hook; loads bootstrapper + MMF |
| B — Bootstrapper | `tools/axon-clr-bootstrapper` | Remote-inject path for running processes |
| C — Managed agent | `tools/axon-clr-managed-agent` (ClrMD) | Heap enumeration (incl. frozen segments), reflection invoke |
| D — MMF host | `axon-clr::mmf`, `ClrSession` | Seqlock ring, command/result slots |

Cross-process `ReadProcessMemory` ghost walks are **degraded fallback only** when the in-process bridge is unavailable.

## Attach strategies

1. **Profiler** — `COR_ENABLE_PROFILING` / `CORECLR_PROFILER` at process launch (CoreCLR + .NET Framework)
2. **Remote inject** — `AXON_CLR_INJECT=1` + `axon_clr_bootstrapper.dll` for running GUI / Framework apps
3. **Startup hook** — `DOTNET_STARTUP_HOOKS` for modern `dotnet` CLI targets (CoreCLR 3+ only)
4. **Passive** — host creates MMF and waits for profiler, startup hook, or inject-loaded agent

## Build prerequisites (Windows E2E)

- .NET SDK 6 / 8 / 9
- .NET Framework 4.8 targeting pack (for `net48` fixtures and agent)
- Rust toolchain + `cargo build -p axon-clr-bootstrapper`
- CMake (for `tools/axon-clr-profiler` COM shim)
- PowerShell 5.1+; run `scripts/clr-run-all-tests.ps1` or tiered scripts under `scripts/clr-*.ps1`

Managed agent TFMs: `target/release/clr-agent/{net6.0,net8.0,net48}/`. Bootstrapper selects `hostfxr` (CoreCLR) or `mscoree` (.NET Framework).

## Degradation

Stale MMF heartbeat >5s → watcher drops session and re-attaches in-process bridge.
