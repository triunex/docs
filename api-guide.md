# NSP Agent API Guide

**Version:** v1 | **Base URL:** `http://127.0.0.1:8765/substrate/v1`

For the machine-readable API specification, see `../nsp-specification/openapi.yaml`.

---

## Authentication

Every request must include your API key:

```bash
curl -H "Authorization: Bearer nsp_v1_<your_key>" \
  http://127.0.0.1:8765/substrate/v1/apps
```

API keys are issued via `axon key create`. See the [Operator Runbook](./operator-runbook.md) for key management.

---

## Quickstart: Your First Agent Loop

```python
import requests, time

BASE = "http://127.0.0.1:8765/substrate/v1"
HEADERS = {"Authorization": "Bearer nsp_v1_YOUR_KEY"}

# 1. Find Gmail
apps = requests.get(f"{BASE}/apps", headers=HEADERS).json()
gmail = next(a for a in apps["apps"] if a["app_id"] == "gmail")
pid = gmail["pid"]

# 2. Read state
state = requests.get(f"{BASE}/state/{pid}", headers=HEADERS).json()
unread = state["objects"].get("inbox.unread_count", 0)
print(f"Unread: {unread}")

# 3. Execute a reversible action
resp = requests.post(f"{BASE}/action", headers=HEADERS, json={
    "app_id": "gmail",
    "action": "open_compose",
    "verify": False,
}).json()
print(resp)

# 4. Execute an irreversible action (requires verify=true)
resp = requests.post(f"{BASE}/action", headers=HEADERS, json={
    "app_id": "gmail",
    "action": "send_email",
    "parameters": {"to": "bob@example.com", "subject": "Hello"},
    "verify": True,
    "verify_expression": "changed(inbox.sent_count)",
    "verify_timeout_ms": 8000,
}).json()
print(resp["verification"])
```

---

## Streaming State Changes

```python
import websocket, json

def on_message(ws, msg):
    event = json.loads(msg)
    print(f"PID {event['pid']}: changed keys = {event['changed_keys']}")

ws = websocket.WebSocketApp(
    "ws://127.0.0.1:8765/substrate/v1/watch/12345",
    header=["Authorization: Bearer nsp_v1_YOUR_KEY"],
    on_message=on_message
)
ws.run_forever()
```

---

## Action Safety Reference

| Reversibility | Minimum `confidence` | Requires `verify=true` |
|---------------|---------------------|------------------------|
| `Read` | None | No |
| `ReversibleWrite` | 0.80 | No |
| `IrreversibleWrite` | 0.95 | **Yes** |

> **Note:** The effective confidence is `min(schema.confidence, state.confidence)`. If the probe ran in DOM fallback mode, state confidence may be 0.70–0.85, which blocks irreversible actions until a high-quality V8 probe cycle completes. Use `GET /state/{pid}?fresh=true` to trigger a fresh cycle.

---

## Error Handling

```python
resp = requests.post(f"{BASE}/action", headers=HEADERS, json={...})
if resp.status_code == 422:
    err = resp.json()
    if err["code"] == "confidence_too_low":
        # Wait for a better probe cycle
        time.sleep(5)
        # Retry with fresh=true
    elif err["code"] == "irreversible_not_confirmed":
        # Add verify=true to the request
        pass
elif resp.status_code == 429:
    retry_after = int(resp.headers.get("Retry-After", 1))
    time.sleep(retry_after)
```

---

## CDP Builtin Actions (No Schema Required)

These work on any V8 target without a registered schema:

```python
# Navigate
requests.post(f"{BASE}/action", headers=HEADERS, json={
    "app_id": "chrome",
    "action": "navigate",
    "parameters": {"url": "https://example.com", "wait_for_network_idle": True},
})

# Click element
requests.post(f"{BASE}/action", headers=HEADERS, json={
    "app_id": "chrome",
    "action": "click",
    "parameters": {"selector": "button#submit"},
})

# Type text
requests.post(f"{BASE}/action", headers=HEADERS, json={
    "app_id": "chrome",
    "action": "type",
    "parameters": {
        "selector": "input#search",
        "text": "nelieo",
        "clear_first": True,
    },
})

# Read text
result = requests.post(f"{BASE}/action", headers=HEADERS, json={
    "app_id": "chrome",
    "action": "read_text",
    "parameters": {"selector": "h1.title"},
}).json()
print(result["result"])
```

---

## Verify Expressions Reference

Expressions are a safe, restricted DSL — not arbitrary JavaScript.

| Pattern | Example | Description |
|---------|---------|-------------|
| `field == value` | `inbox.unread_count == 0` | Equality |
| `field != value` | `compose.is_open != false` | Inequality |
| `field > value` | `inbox.sent_count > 5` | Numeric greater-than |
| `field >= value` | `confidence >= 0.80` | Numeric ≥ |
| `changed(field)` | `changed(inbox.unread_count)` | Field changed pre→post |
| `exists(field)` | `exists(compose.draft_id)` | Field exists in post-state |
| `context.url == "url"` | `context.url == "https://sent"` | URL comparison |
| `prev.field != field` | `prev.inbox.unread_count != inbox.unread_count` | Explicit pre/post comparison |

> **Security:** `fetch(`, `eval(`, `XMLHttpRequest`, `import(`, `document.cookie` and similar patterns are rejected by the expression validator. Expressions are evaluated in the Axon process, not in the browser tab.
