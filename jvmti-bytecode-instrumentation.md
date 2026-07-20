# JVMTI Bytecode Instrumentation

Phase 3 Pillar 3 — ASM-based probes orchestrated from Rust `ClassFileLoadHook`.

## Architecture

| Layer | Technology | Role |
|-------|------------|------|
| Transforms | `tools/jvmti-injector` (ASM 9.7) | `AxonClassTransformer` inserts method-entry probes |
| Orchestration | `injector_bridge.rs` + `loadhook.rs` | Filter → JNI → Java transform → hook output bytes |
| Telemetry | `AxonProbe` JNI natives | Enqueue `field_changed` / `method_exit` IPC events |

Rust does **not** parse method bodies for production transforms — ASM handles verifier, stack maps, and Java 17+ features.

## Attach options

```
instrument=off|audit|transform
instrument_allow=com.nelieo,com.example.demo
```

## Rollout milestones

| Milestone | Mode | Behavior |
|-----------|------|----------|
| 3.0 audit | `audit` | Log only; pass-through bytes |
| 3.1 transform | `transform` | ASM probes on newly loaded classes |
| 3.2 retransform | `transform` + sweep | `RetransformClasses` on pre-loaded allowlisted classes |

## Safety policy (`policy.rs`)

- Never transform `java.*`, `jdk.*`, bootstrap classes
- Skip classes > 4MB → `instrument_skipped`
- Transform/verifier error → pass-through original bytes + `instrument_skipped`
- SAP / signed JAR: use `instrument=audit` runbook first (`jvmti-tier-c-sap.ps1`)

## Building the injector JAR

```powershell
mvn -f tools/jvmti-injector/pom.xml package
$env:AXON_JVMTI_INJECTOR_PATH = "tools/jvmti-injector/target/jvmti-injector-0.1.0-SNAPSHOT.jar"
```

## SAP Tier C runbook (summary)

1. Attach with `instrument=audit;include=com.sap` — no bytecode change
2. Review `instrument=audit` logs for class sizes and probe plan
3. Narrow `instrument_allow=` to explicit subset before `instrument=transform`
4. Record results in `docs/jvmti-enterprise-stress-results.md` §Phase 3

## Rollback

Set `instrument=off` and reattach, or detach agent — original class bytes are preserved on transform failure (pass-through).
