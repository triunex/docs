# CoreCLR offset tables (.NET 8.0.x)

Offsets live in `crates/axon-clr/src/offsets/dotnet8_0.rs`.

| Symbol | Offset | Purpose |
|--------|--------|---------|
| `MethodTable.m_pEEClass` | `0x08` | Type info pointer |
| `MethodTable.m_wNumInterfaces` | `0x0E` | Sanity check |
| `ObjectHeader` | `0x08` | Instance field base |
| `EEClass.m_lpNSExternal` | `0x18` | Type name UTF-16 |
| `MethodDesc` entry | `0x30` | JIT trampoline target |

## Validation runbook

1. Attach WinDbg / dotnet-dump to `fixtures/dotnet-test-app`
2. Compare `MethodTable` layout for target `coreclr.dll` build
3. Update `dotnet8_0.rs` and bump `ClrShmHeader.coreclr_version`
4. Re-run `clr-heap-test.ps1` and `clr-xray-test.ps1`

Unknown runtime → `offsets::detect_from_process()` returns error; ghost walk uses versioned tables in `dotnet6_0.rs`, `dotnet8_0.rs`, `dotnet9_0.rs`, `netfx48.rs`.

## .NET Framework 4.8 (desktop CLR)

Offsets live in `crates/axon-clr/src/offsets/netfx48.rs`. Desktop `MethodTable` / `EEClass` layouts differ from CoreCLR — validate against `clr.dll` 4.8.4xx before changing constants.

| Symbol | Offset | Notes |
|--------|--------|-------|
| `MethodTable.m_pEEClass` | `0x10` | Desktop layout |
| `MethodTable` module | `0x30` | Loader module pointer |
| `EEClass.m_lpNSExternal` | `0x20` | Type name |

Managed agent for Framework is loaded via `mscoree` (`ExecuteInDefaultAppDomain`) from `tools/axon-clr-bootstrapper/src/mscoree_host.rs`, using `target/release/clr-agent/net48/axon-clr-managed-agent.dll`.
