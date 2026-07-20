# CLR Phase 1 Spike

## Goals

- Validate MMF seqlock ring between host (`axon-clr`) and injector (`axon-clr-bootstrapper`)
- Prove remote `LoadLibrary` inject path on Windows
- Bench host-side delta drain latency (`clr-mmf-bench.ps1`)

## Results

| Check | Script | Status |
|-------|--------|--------|
| MMF attach | `clr-phase1-spike.ps1` | Pass (mmf_only) |
| 10k delta drain | `clr-mmf-test.ps1` | Pass |
| Remote inject | `AXON_CLR_INJECT=1` + bootstrapper DLL | Implemented (`inject.rs`) |

## vs ICorProfiler

| Dimension | ICorProfiler | axon-clr |
|-----------|--------------|----------|
| Hot path | Managed callback ~µs–ms | MMF write ~ns |
| Discovery | Metadata APIs | Raw MethodTable walk |
| Verdict | Rejected for hot path | Adopted per `shaurya_CLR.md` |

## Next

- Live SOH root scan in bootstrapper (replace catalog JSON)
- CFG assessment for JIT trampolines on Tier A
