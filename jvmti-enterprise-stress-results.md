# JVMTI Enterprise Stress Results

Generated: 2026-06-22T09:20:36.5694609-04:00

| Fixture | Classes scanned | Attach ms idle | Attach ms under load | Max GC pause ms | HTTP p50 delta | Pass |
|---------|-----------------|----------------|----------------------|-----------------|----------------|------|
| tier-a | 7977 | 211 | 238 | 0 | 1.81 | yes |

## Phase 3 overhead (placeholder — run `jvmti-enterprise-stress.ps1 -Fixtures tier-a,tier-b-jpa` with `xray=1;telemetry=1`)

| Fixture | Options | Attach ms | HTTP p50 delta | telemetry_dropped | Pass |
|---------|---------|-----------|------------------|---------------------|------|
| tier-a | xray=1;telemetry=1 | _pending_ | _pending_ | _pending_ | _pending_ |
| tier-b-jpa | telemetry=1 | _pending_ | _pending_ | _pending_ | _pending_ |
| tier-c-sap | instrument=audit | _pending_ | N/A | N/A | _pending_ |

Warm `read_instances` p99 target: **< 50ms** (`scripts/jvmti-xray-bench.ps1`).
Field change → WebSocket SLA target: **< 500ms** (`scripts/jvmti-push-ws-test.ps1`).
