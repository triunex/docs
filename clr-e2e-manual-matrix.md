# CLR E2E manual smoke matrix

Use this checklist for apps that are not fully automated in CI. Each row lists attach mode, env vars, and expected `axon clr dump` behavior.

## Modern CoreCLR console (6 / 8 / 9)

| App | Attach | Env | Expected |
|-----|--------|-----|----------|
| `fixtures/dotnet-test-app` | Startup hook | `DOTNET_STARTUP_HOOKS`, `AXON_CLR_INCLUDE=MyApp` | `MyApp.Customer`, `MyApp.Order` |
| `fixtures/dotnet-test-app-net6` | Startup hook | same | same |
| `fixtures/dotnet-test-app-net9` | Startup hook | same | same |
| Running console (no hook) | Inject | `AXON_CLR_INJECT=1` | types via ClrMD agent |
| Launch-time attach | Profiler | `AXON_CLR_PROFILER=1`, `COR_*` / `CORECLR_*` | types via bootstrapper + agent |

## ASP.NET Core

| App | Attach | Notes |
|-----|--------|-------|
| `fixtures/dotnet-tier-a` | Startup hook | warm `http://127.0.0.1:5080/health`; watcher expects `customer` key |
| External Kestrel app | Inject or profiler | set `AXON_CLR_INCLUDE` to app namespace prefix |

## .NET Framework 4.8

| App | Attach | Notes |
|-----|--------|-------|
| `fixtures/dotnet-framework-test` | Inject / profiler | no startup hooks; invoke `MyApp.Order.Total` → `123.45` |
| `fixtures/dotnet-wpf-spike` | Inject / profiler | STA; may need longer attach timeout |
| `fixtures/dotnet-winforms-spike` | Inject | WinForms message loop |
| IIS ASP.NET (manual) | Profiler on app pool | set `COR_PROFILER` on worker process; not in CI |

## Services and GUI

| App | Attach | Notes |
|-----|--------|-------|
| `fixtures/dotnet-worker-spike` | Startup hook | `IHost` background service |
| Already-running GUI | Inject only | Excel, custom LOB — `AXON_CLR_INJECT=1` |
| Windows Service (manual) | Inject / profiler | install test service or use `sc.exe` trigger |

## Third-party OSS (automated optional)

See `fixtures/third-party/manifest.json` and `scripts/clr-third-party-smoke.ps1`. Set `AXON_CLR_THIRD_PARTY_OPTIONAL=1` to allow fetch/build failures on PRs.

## Quick commands

```powershell
. .\scripts\_clr-e2e-common.ps1
$p = Start-DotnetTestApp -Fixture dotnet-test-app -AttachMode StartupHook
$env:AXON_CLR_INJECT = '0'
.\target\release\axon.exe clr dump $p.Id --include MyApp
```

```powershell
$p = Start-DotnetTestApp -Fixture dotnet-framework-test -AttachMode Inject
$env:AXON_CLR_INJECT = '1'
.\target\release\axon.exe clr invoke $p.Id --type MyApp.Order --method Total
```
