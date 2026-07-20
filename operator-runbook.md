# Axon Operator Runbook

**Version:** 0.1 | **Product:** Nelieo Substrate Protocol (NSP) | **Target:** Axon Daemon Operators

---

## 1. Quick Start

### Installation

```bash
# Linux / macOS
curl -fsSL https://releases.nelieo.com/axon/install.sh | sh

# Windows (PowerShell as Administrator)
irm https://releases.nelieo.com/axon/install.ps1 | iex
```

Binary installs to `/usr/local/bin/axon` (Linux/macOS) or `C:\Program Files\Nelieo\axon.exe` (Windows).

### First Run

```bash
# 1. Generate your first API key
axon key create --name "my-agent"
# → Prints key: nsp_v1_<key_bytes>

# 2. Start the daemon
axon daemon start

# 3. Verify it's healthy
curl http://127.0.0.1:8765/substrate/v1/apps
```

---

## 2. Configuration Reference

### Config File Location

| OS | Path |
|----|------|
| Linux | `/etc/nelieo/axon.toml` (system) or `~/.config/nelieo/axon.toml` (user) |
| macOS | `~/Library/Application Support/Nelieo/axon.toml` |
| Windows | `C:\ProgramData\Nelieo\axon.toml` |

Environment variable override: prefix any config key with `AXON_` (e.g., `AXON_SERVER_PORT=9000`).

### Full Config with Defaults

```toml
[server]
host = "127.0.0.1"       # Never bind to 0.0.0.0 — local only
port = 8765
request_timeout_ms = 120000
shutdown_timeout_ms = 10000

[auth]
# !! PRODUCTION: require_key must always be true !!
require_key = true
# Platform-specific default if not set:
# Linux/macOS: /opt/nelieo/data/axon_keys.json
# Windows:     C:\ProgramData\Nelieo\axon_keys.json
keys_file = ""

[cors]
# Leave empty in production. Only add specific origins you control.
allowed_origins = []

[watcher]
scan_interval_ms = 2000          # How often to scan for new processes
max_tracked_processes = 256      # Hard cap on concurrent probe targets
cold_probe_concurrency = 2       # Max simultaneous V8 cold heap snapshots
probe_cycle_timeout_ms = 150000  # Max time for a single probe cycle

[registry]
db_path = "schema-registry.db"   # SQLite schema registry
ttl_days = 30                    # Schemas older than this are evicted
max_entries = 512                # Hard cap on registry entries

[rate_limit]
state_query_per_second = 100     # GET /apps, GET /state/* per API key
action_exec_per_second = 10      # POST /action per API key
bulk_ops_per_second = 5          # Bulk operations per API key
rate_bucket_ttl_hours = 1        # Idle bucket cleanup interval
```

---

## 3. API Key Management

```bash
# List all keys
axon key list

# Create a new key with optional description
axon key create --name "sdk-user-001" --description "Primary SDK test key"

# Revoke a key by ID
axon key revoke --id <key_id>

# Rotate a key (revoke + create in one step)
axon key rotate --id <key_id>
```

**Key file format** (`axon_keys.json`):
```json
{
  "keys": [
    {
      "id": "k_abc123",
      "name": "my-agent",
      "key_hash": "<hmac-sha256-hash>",
      "created_at": "2026-06-15T00:00:00Z",
      "last_used_at": "2026-06-15T12:00:00Z",
      "description": "Primary agent key"
    }
  ]
}
```

> **Security:** The `key_hash` field stores a HMAC-SHA256 hash of the key, not the key itself. The raw key is shown only at creation time and cannot be recovered. Store it in a secrets manager.

---

## 4. Daemon Operations

### Start / Stop / Status

```bash
axon daemon start          # Start as foreground process
axon daemon start --bg     # Start as background daemon
axon daemon stop           # Graceful shutdown
axon daemon status         # Check running status
axon daemon restart        # Stop + start
```

### systemd Service (Linux)

```ini
# /etc/systemd/system/axon.service
[Unit]
Description=Nelieo Axon Substrate Protocol Daemon
After=network.target

[Service]
Type=simple
User=nelieo
ExecStart=/usr/local/bin/axon daemon start
Restart=always
RestartSec=5
Environment=RUST_LOG=info

[Install]
WantedBy=multi-user.target
```

```bash
systemctl enable axon
systemctl start axon
journalctl -u axon -f
```

### Windows Service

```powershell
# Install as Windows service
axon service install

# Manage via Services or:
Start-Service Axon
Stop-Service Axon
Get-Service Axon
```

---

## 5. Health & Observability

### Health Endpoint

```
GET /substrate/v1/health
```

Response:
```json
{
  "status": "ok",
  "version": "0.1.0",
  "uptime_seconds": 3600,
  "tracked_processes": 12,
  "cold_probes_in_progress": 0
}
```

### Metrics (Prometheus)

```
GET /metrics
```

Key metrics:

| Metric | Description |
|--------|-------------|
| `axon_probes_total{tier,status}` | Probe completions by tier and success/failure |
| `axon_probe_duration_seconds{tier}` | Probe latency histogram |
| `axon_actions_total{reversibility,status}` | Action executions |
| `axon_rate_limit_rejections_total{endpoint}` | Rate limit hits |
| `axon_tracked_processes` | Current process count |
| `axon_registry_entries` | Schema registry size |

### Logging

```bash
# Set log level via environment
RUST_LOG=debug axon daemon start

# Log levels: error, warn, info, debug, trace
# Structured JSON logs:
AXON_LOG_FORMAT=json axon daemon start
```

---

## 6. Security Hardening Checklist

Run through this before issuing any API keys:

- [ ] `auth.require_key = true` (default — verify it's not overridden)
- [ ] `cors.allowed_origins` is empty or restricted to known origins
- [ ] `server.host = "127.0.0.1"` (never `0.0.0.0` unless firewalled)
- [ ] API keys are stored in a secrets manager (not in source code)
- [ ] `axon_keys.json` permissions are `600` (owner-read-only)
- [ ] TLS termination is handled by a reverse proxy (nginx, Caddy) for remote access
- [ ] Log rotation is configured to avoid disk exhaustion
- [ ] `max_tracked_processes` is tuned to your machine's RAM
- [ ] `cold_probe_concurrency` is `≤ 2` on machines with < 8GB RAM

---

## 7. Troubleshooting

### "No apps tracked" on `/apps`

1. Check `watcher.scan_interval_ms` isn't too high.
2. Verify the target app is running and not sandboxed (e.g., Chrome in app mode with `--remote-debugging-port`).
3. Check `max_tracked_processes` hasn't been hit: look for `[WARN] max_tracked_processes limit reached` in logs.

### Action fails with `confidence_too_low`

The probe quality is below the action's tier threshold. Solutions:
1. Wait for the next probe cycle (the watcher runs every `scan_interval_ms`).
2. Call `GET /state/{pid}?fresh=true` to force a fresh probe.
3. Check if the app is in a loading state (DOM only, no V8 heap available).

### `execution_timeout` on actions

The action gate couldn't acquire the exclusive lock in time. This usually means:
- A cold probe is running simultaneously (2-permit semaphore, can take up to 60s).
- Another action is holding the write lock.

Mitigation: Reduce `cold_probe_concurrency` or increase `probe_cycle_timeout_ms`.

### macOS: First scan takes 30s+

This was a known bug (H-13) in pre-0.1 builds where `vmmap` was spawned per process. Update to 0.1.0+.

### Registry grows unexpectedly

Check `registry.ttl_days` and `registry.max_entries`. If the DB is growing faster than expected:
```bash
axon registry status      # Show entry count and disk usage
axon registry prune       # Force eviction of expired entries
```

---

## 8. Upgrade Procedure

1. Stop the daemon: `axon daemon stop`
2. Replace the binary (or run installer)
3. Check for config changes: `axon config validate`
4. Restart: `axon daemon start`
5. Verify: `curl http://127.0.0.1:8765/substrate/v1/health`

The schema registry is forward-compatible across minor versions. Major version upgrades may require a registry migration — check release notes.

---

## 9. Frequently Asked Questions

**Q: Can Axon track sandboxed browsers (e.g., Chrome in Incognito mode)?**  
A: Yes, if Chrome is launched with `--remote-debugging-port=PORT`. Standard incognito sessions without the debug flag are not accessible.

**Q: Does Axon send data to Nelieo servers?**  
A: No. All probe data stays local. The daemon communicates only with local processes and the local schema registry SQLite database.

**Q: What permissions does Axon need?**  
A: On Linux, it needs `ptrace` capability or root for full /proc access. On macOS, it needs Accessibility permissions for window title queries. On Windows, it runs as a standard user with `PROCESS_QUERY_INFORMATION`.

**Q: How do I track a specific Chrome tab and not all tabs?**  
A: Use the CDP port: launch Chrome with `--remote-debugging-port=9222` and filter `/apps` by `cdp_port`.
