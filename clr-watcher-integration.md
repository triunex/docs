# CLR watcher integration

## Slots

- `clr_session_slot: Mutex<Option<ClrSession>>`
- `clr_anchor_slot: ArcSwapOption<ClrProbeAnchor>`

## `probe_clr` cycle

1. Cold: `ClrSession::attach` + `dump_heap` (stub catalog MVP)
2. Warm: MMF `drain_deltas` + anchor-valid re-read (future)
3. Degraded: heartbeat stale → take session, clear anchor, retry

## Env

- `AXON_CLR_XRAY`, `AXON_CLR_TELEMETRY`, `AXON_CLR_INCLUDE`

## API

- `execute_clr` via `WatcherHandle::get_clr_session`
- Internal target: `clr:Namespace.Type#Method`
