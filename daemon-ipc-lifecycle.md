# JVM IPC lifecycle

## Failure catalog (F1–F7)

| # | Scenario | Target behavior |
|---|----------|-----------------|
| F1 | JVM kill -9 during dump | Detect <500ms; `IpcDisconnected`; watcher degrades |
| F2 | Agent worker panic | Reader sets disconnect; unblock waiters |
| F3 | Host axon-api restart | Agent sees broken pipe; worker exits |
| F4 | PID reuse | Session generation in pipe name (future) |
| F5 | Zombie `\\.\pipe\axon-jvmti-{pid}` | Host closes handles on detach |
| F6 | Action on dead PID | 503 + Retry-After within 5s |
| F7 | Double attach same PID | Second attach rejects or supersedes |

## State machine

```
Detached → Attaching → Ready → Active
Active → Degraded (pipe disconnect) → Attaching (watcher retry)
Active → ShuttingDown → Detached
```

## Pipe layout

- Events: `\\.\pipe\axon-jvmti-{pid}` (agent → host)
- Commands: `\\.\pipe\axon-jvmti-{pid}-cmd` (host → agent)

On `ReadFile` failure the host reader sets `ConnectionState::disconnected`.
All blocking waits check `is_connected()` first.
