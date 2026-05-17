# A2A Notebook 06 — `06_a2a_push_notifications.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the sixth notebook in the A2A learning series — **push notifications**. The client registers a webhook with `tasks/pushNotificationConfig/set`, the server returns immediately with a `submitted` task, the worker finishes asynchronously and POSTs the final `Task` to the webhook URL. The learner stands up a **second** FastAPI app in the same notebook to receive the callback.

**Architecture:** Two FastAPI apps in one notebook. App #1 = the researcher (port `8010`) with a task store and background worker (lifted from notebook 04). App #2 = a tiny callback receiver (port `8011`) that captures the inbound POST. On task completion, the researcher's worker looks up any push-notification config for that task and POSTs the final `Task` JSON to the configured URL.

**Tech Stack:** Python 3.11+, Jupyter, FastAPI, uvicorn, httpx, pydantic v2, threading. Same stack as notebooks 01–05.

**A2A spec version targeted:** v0.3.0. Methods: `message/send`, `tasks/pushNotificationConfig/set`. The webhook payload is the serialized `Task` object — no envelope. Auth on push notifications is deferred to notebook 07.

**Port assignment:** Researcher on `127.0.0.1:8010`. Callback receiver on `127.0.0.1:8011`.

---

## File Structure

- **Create:** `06_a2a_push_notifications.ipynb` — self-contained.

## Notebook Section Map

1. Title + "Why this notebook exists"
2. "What you'll learn"
3. Setup
4. Section 1: When SSE isn't enough (markdown)
5. Section 2: Push-notification models
6. Section 3: The callback receiver (app #2)
7. Section 4: The researcher with webhook delivery (app #1)
8. Section 5: End-to-end demo
9. Section 6: Auth note (markdown — deferred to nb07)
10. "What you just learned"
11. "What's missing"
12. Cleanup

---

## Task 1: Scaffold

**Files:** Create `06_a2a_push_notifications.ipynb`.

- [ ] **Step 1: Write empty notebook**

```json
{
  "cells": [],
  "metadata": {
    "kernelspec": {"display_name": "Python 3", "language": "python", "name": "python3"},
    "language_info": {"name": "python", "version": "3.11"}
  },
  "nbformat": 4,
  "nbformat_minor": 5
}
```

- [ ] **Step 2: Verify**

```bash
python -c "import json; json.load(open('06_a2a_push_notifications.ipynb'))"
```

- [ ] **Step 3: Commit**

```bash
git add 06_a2a_push_notifications.ipynb
git commit -m "feat(a2a): scaffold 06_a2a_push_notifications.ipynb"
```

---

## Task 2: Title + intro markdown

**Files:** Modify `06_a2a_push_notifications.ipynb` (2 markdown cells).

- [ ] **Step 1: Title + "Why"**

```markdown
# 06 — Push Notifications: `tasks/pushNotificationConfig/set` and Webhooks

## Why this notebook exists

**Streaming** (notebook 05) is great while a connection stays open. **Polling** (notebook 04) works but burns requests. Both assume the client is *online and waiting* for the task to finish.

What if the client is a serverless function that gets killed after 30 seconds? A mobile app whose user backgrounded the screen? A CI job that fires-and-forgets and only cares about the eventual outcome?

A2A's third asynchrony pattern is **push notifications**. The client registers a webhook URL once via `tasks/pushNotificationConfig/set`, hangs up, and goes about its life. When the task reaches a terminal state, the server POSTs the final `Task` object to the webhook URL. No connection held, no polling cost.

This notebook stands up two FastAPI apps in one kernel — the researcher (port 8010) and a tiny callback receiver (port 8011) — and watches a task complete via webhook.

> *Targets A2A spec v0.3.0. Auth on push notifications is deferred to notebook 07.*
```

- [ ] **Step 2: "What you'll learn"**

```markdown
## What you'll learn

- The `tasks/pushNotificationConfig/set` method and its `TaskPushNotificationConfig` params.
- The shape of the **webhook payload** A2A servers POST when a task completes: just the serialized `Task` object, no envelope.
- How to register a webhook from the client, fire-and-forget the task, and pick up the result via callback.
- How to run a second FastAPI app **in the same notebook** to act as the webhook receiver.
- When to prefer push notifications over polling or streaming.
- Why webhook auth matters (and why we're deferring it one notebook).
```

- [ ] **Step 3: Commit**

```bash
git add 06_a2a_push_notifications.ipynb
git commit -m "feat(a2a): add intro markdown to notebook 06"
```

---

## Task 3: Setup

**Files:** Modify `06_a2a_push_notifications.ipynb` (markdown header + code cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 1. Setup

Same helpers as previous notebooks.
```

- [ ] **Step 2: Setup code cell**

```python
import json
import threading
import time
import uuid
from datetime import datetime, timezone
from typing import Literal

import httpx
import uvicorn
from fastapi import FastAPI, Request
from pydantic import BaseModel, Field, ValidationError, model_validator

_servers: list[uvicorn.Server] = []


def run_server_in_thread(app: FastAPI, port: int) -> uvicorn.Server:
    config = uvicorn.Config(app, host="127.0.0.1", port=port, log_level="warning")
    server = uvicorn.Server(config)
    thread = threading.Thread(target=server.run, daemon=True)
    thread.start()
    for _ in range(50):
        if server.started:
            break
        time.sleep(0.05)
    else:
        raise RuntimeError(f"Server on port {port} did not start in time")
    _servers.append(server)
    return server


def shutdown_all_servers() -> None:
    for server in list(_servers):
        server.should_exit = True
    _servers.clear()


def now_iso() -> str:
    return datetime.now(timezone.utc).isoformat()


print("Setup OK")
```

- [ ] **Step 3: Run, expect `Setup OK`**

- [ ] **Step 4: Commit**

```bash
git add 06_a2a_push_notifications.ipynb
git commit -m "feat(a2a): add setup cell to notebook 06"
```

---

## Task 4: Section 1 — When SSE isn't enough

**Files:** Modify `06_a2a_push_notifications.ipynb` (1 markdown cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 2. When SSE Isn't Enough

| Scenario | Best fit |
|---|---|
| Client wants every progress update in real time, will stay online | **Streaming** (`message/stream`, notebook 05) |
| Client wants to know when it's done, may go away in the meantime | **Push notifications** (this notebook) |
| Client controls its own cadence, infrastructure forbids long-lived connections | **Polling** (`tasks/get`, notebook 04) |

The webhook flow:

```
client ──────── message/send ───────► server   (returns submitted Task)
client ─── pushNotificationConfig/set ───► server   (stores webhook URL for task)
client closes the connection. Goes about its day.

…time passes, server's worker finishes the task…

server ──────── POST final Task ─────► client's webhook
```

The webhook payload is just the serialized `Task` object — no JSON-RPC envelope, no event wrapper. That's a deliberate simplification: the receiver only needs to know how to parse a `Task`, not the whole A2A protocol.
```

- [ ] **Step 2: Commit**

```bash
git add 06_a2a_push_notifications.ipynb
git commit -m "feat(a2a): motivate push notifications in notebook 06"
```

---

## Task 5: Section 2 — Push-notification models

**Files:** Modify `06_a2a_push_notifications.ipynb` (markdown + code cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 3. Models

Carry forward the A2A core models, plus two new ones:

- **`PushNotificationConfig`** — what's registered. `url` (required webhook endpoint), `authentication` (deferred to nb07; modelled as a dict for now).
- **`TaskPushNotificationConfig`** — the params object for `tasks/pushNotificationConfig/set`. Binds a `PushNotificationConfig` to a specific task via `taskId`.
```

- [ ] **Step 2: Models code cell**

```python
TaskState = Literal[
    "submitted", "working", "input-required",
    "completed", "canceled", "failed", "rejected", "auth-required", "unknown",
]


class TextPart(BaseModel):
    kind: Literal["text"] = "text"
    text: str
    metadata: dict | None = None


class Message(BaseModel):
    messageId: str
    role: Literal["user", "agent"]
    parts: list[TextPart] = Field(..., min_length=1)
    kind: Literal["message"] = "message"
    taskId: str | None = None
    contextId: str | None = None
    metadata: dict | None = None


class TaskStatus(BaseModel):
    state: TaskState
    timestamp: str
    message: Message | None = None


class Artifact(BaseModel):
    artifactId: str
    name: str | None = None
    description: str | None = None
    parts: list[TextPart]
    metadata: dict | None = None


class Task(BaseModel):
    id: str
    contextId: str
    status: TaskStatus
    kind: Literal["task"] = "task"
    history: list[Message] = Field(default_factory=list)
    artifacts: list[Artifact] = Field(default_factory=list)
    metadata: dict | None = None


class PushNotificationConfig(BaseModel):
    url: str
    authentication: dict | None = None  # nb07 will give this a real type


class TaskPushNotificationConfig(BaseModel):
    taskId: str
    pushNotificationConfig: PushNotificationConfig


class JSONRPCRequest(BaseModel):
    jsonrpc: Literal["2.0"] = "2.0"
    id: str | int
    method: str
    params: dict | None = None


class JSONRPCError(BaseModel):
    code: int
    message: str
    data: dict | None = None


class JSONRPCResponse(BaseModel):
    jsonrpc: Literal["2.0"] = "2.0"
    id: str | int | None = None
    result: dict | None = None
    error: JSONRPCError | None = None

    @model_validator(mode="after")
    def _exactly_one(self) -> "JSONRPCResponse":
        if (self.result is None) == (self.error is None):
            raise ValueError("JSONRPCResponse must contain exactly one of `result` or `error`")
        return self


print("Models defined.")
```

- [ ] **Step 3: Run, expect `Models defined.`**

- [ ] **Step 4: Commit**

```bash
git add 06_a2a_push_notifications.ipynb
git commit -m "feat(a2a): add push notification models to notebook 06"
```

---

## Task 6: Section 3 — The callback receiver

**Files:** Modify `06_a2a_push_notifications.ipynb` (markdown + code cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 4. The Callback Receiver

A separate FastAPI app on port `8011`, with one endpoint — `POST /callback` — that captures whatever the researcher sends. We stash each callback in a thread-safe list so the demo cell can fish out the most recent one.

In real life this would be a publicly-reachable URL (a cloud function, a webhook proxy). On localhost it's just another `uvicorn` on a different port.
```

- [ ] **Step 2: Receiver code cell**

```python
RECEIVED_CALLBACKS: list[dict] = []
CALLBACKS_LOCK = threading.Lock()


callback_app = FastAPI()


@callback_app.post("/callback")
async def receive_callback(request: Request) -> dict:
    body = await request.json()
    with CALLBACKS_LOCK:
        RECEIVED_CALLBACKS.append(body)
    return {"received": True}


callback_server = run_server_in_thread(callback_app, port=8011)
print("Callback receiver running on http://127.0.0.1:8011 (POST /callback)")
```

- [ ] **Step 3: Run, expect `Callback receiver running on http://127.0.0.1:8011 (POST /callback)`**

- [ ] **Step 4: Commit**

```bash
git add 06_a2a_push_notifications.ipynb
git commit -m "feat(a2a): add callback receiver app in notebook 06"
```

---

## Task 7: Section 4 — The researcher with webhook delivery

**Files:** Modify `06_a2a_push_notifications.ipynb` (markdown + code cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 5. The Researcher with Webhook Delivery

This is the notebook 04 slow researcher, with two additions:

1. **`PUSH_CONFIGS`** — a `dict[task_id, PushNotificationConfig]` populated by `tasks/pushNotificationConfig/set`.
2. **Webhook delivery in the worker.** When the task transitions to a terminal state (`completed` or `failed`), the worker looks up any registered webhook and POSTs the final `Task` JSON to it. Errors during delivery are logged but don't change the task's state — webhook delivery is best-effort by spec.

We snapshot the task under the lock before returning from `message/send` (same race fix as notebook 04).
```

- [ ] **Step 2: Server code cell**

```python
FACTS_BY_TOPIC: dict[str, list[str]] = {
    "octopuses": [
        "Octopuses have three hearts.",
        "They can change color in under a second.",
        "Each of their arms has its own neural cluster.",
    ],
    "rome": [
        "Rome was founded in 753 BCE according to tradition.",
        "The Roman Empire at its peak spanned roughly 5 million km².",
        "Roman concrete used volcanic ash and is still studied today.",
    ],
}


TASKS: dict[str, Task] = {}
PUSH_CONFIGS: dict[str, PushNotificationConfig] = {}
STORE_LOCK = threading.Lock()


def _set_status(task_id: str, **kw) -> None:
    with STORE_LOCK:
        TASKS[task_id].status = TaskStatus(timestamp=now_iso(), **kw)


def _set_artifacts(task_id: str, artifacts: list[Artifact]) -> None:
    with STORE_LOCK:
        TASKS[task_id].artifacts = artifacts


def _deliver_webhook(task_id: str) -> None:
    """Best-effort POST the final task to the registered webhook (if any)."""
    with STORE_LOCK:
        config = PUSH_CONFIGS.get(task_id)
        task = TASKS.get(task_id)
    if config is None or task is None:
        return
    payload = task.model_dump(exclude_none=True)
    try:
        r = httpx.post(config.url, json=payload, timeout=5.0)
        r.raise_for_status()
    except Exception as e:
        # Best-effort delivery: log and move on.
        print(f"[webhook] delivery to {config.url} failed: {e!r}")


def _slow_worker(task_id: str, topic: str) -> None:
    _set_status(task_id, state="working")
    time.sleep(1.5)  # one chunk of work; cancellation is out of scope here

    facts = FACTS_BY_TOPIC.get(topic)
    if facts:
        _set_artifacts(task_id, [
            Artifact(
                artifactId=str(uuid.uuid4()),
                name=f"facts-about-{topic}",
                parts=[TextPart(text=fact) for fact in facts],
            ),
        ])
        _set_status(task_id, state="completed")
    else:
        _set_status(
            task_id,
            state="failed",
            message=Message(
                messageId=str(uuid.uuid4()), role="agent",
                parts=[TextPart(text=f"No facts on file for {topic!r}.")],
            ),
        )

    _deliver_webhook(task_id)


researcher_app = FastAPI()


def _ok(rpc_id, result_obj) -> dict:
    return {"jsonrpc": "2.0", "id": rpc_id, "result": result_obj}


def _err(rpc_id, code: int, message: str) -> dict:
    return {"jsonrpc": "2.0", "id": rpc_id, "error": {"code": code, "message": message}}


def _handle_message_send(req: JSONRPCRequest) -> dict:
    try:
        msg = Message.model_validate((req.params or {}).get("message"))
    except ValidationError as e:
        return _err(req.id, -32602, f"Invalid params: {e.errors()}")

    topic = msg.parts[0].text.strip().lower()
    task = Task(
        id=str(uuid.uuid4()),
        contextId=str(uuid.uuid4()),
        status=TaskStatus(state="submitted", timestamp=now_iso()),
        history=[msg],
    )
    with STORE_LOCK:
        TASKS[task.id] = task
        snapshot = task.model_copy(deep=True)

    threading.Thread(target=_slow_worker, args=(task.id, topic), daemon=True).start()
    return _ok(req.id, snapshot.model_dump(exclude_none=True))


def _handle_set_push(req: JSONRPCRequest) -> dict:
    try:
        cfg = TaskPushNotificationConfig.model_validate(req.params)
    except ValidationError as e:
        return _err(req.id, -32602, f"Invalid params: {e.errors()}")
    with STORE_LOCK:
        if cfg.taskId not in TASKS:
            return _err(req.id, -32602, f"Unknown taskId: {cfg.taskId}")
        PUSH_CONFIGS[cfg.taskId] = cfg.pushNotificationConfig
    # By spec the response echoes the stored config.
    return _ok(req.id, cfg.model_dump(exclude_none=True))


@researcher_app.post("/")
def handle_jsonrpc(req: JSONRPCRequest) -> dict:
    if req.method == "message/send":
        return _handle_message_send(req)
    if req.method == "tasks/pushNotificationConfig/set":
        return _handle_set_push(req)
    return _err(req.id, -32601, f"Method not found: {req.method!r}")


researcher_server = run_server_in_thread(researcher_app, port=8010)
print("Researcher running on http://127.0.0.1:8010")
```

- [ ] **Step 3: Run, expect `Researcher running on http://127.0.0.1:8010`**

- [ ] **Step 4: Commit**

```bash
git add 06_a2a_push_notifications.ipynb
git commit -m "feat(a2a): build researcher with webhook delivery in notebook 06"
```

---

## Task 8: Section 5 — End-to-end demo

**Files:** Modify `06_a2a_push_notifications.ipynb` (markdown + code cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 6. End-to-End Demo

The flow we'll exercise:

1. POST `message/send "octopuses"` → server returns a `submitted` task.
2. POST `tasks/pushNotificationConfig/set` with the task id and the receiver's URL → server stores the webhook.
3. Sleep ~2s to let the worker finish.
4. Inspect `RECEIVED_CALLBACKS` — the receiver should now have one entry: the final `Task` in `completed` state.

In real life the client could exit the kernel completely between steps 2 and 4 — the server-side worker doesn't care.
```

- [ ] **Step 2: Demo code cell**

```python
def rpc(method: str, params: dict) -> dict:
    envelope = {
        "jsonrpc": "2.0",
        "id": str(uuid.uuid4()),
        "method": method,
        "params": params,
    }
    r = httpx.post("http://127.0.0.1:8010/", json=envelope, timeout=10.0)
    r.raise_for_status()
    return r.json()


# Step 1: kick off the task.
send_resp = rpc("message/send", {
    "message": {
        "messageId": str(uuid.uuid4()),
        "role": "user",
        "parts": [{"kind": "text", "text": "octopuses"}],
        "kind": "message",
    },
})
task_payload = send_resp["result"]
task_id = task_payload["id"]
print(f"1. Task submitted: id={task_id} state={task_payload['status']['state']}")

# Step 2: register the webhook.
set_resp = rpc("tasks/pushNotificationConfig/set", {
    "taskId": task_id,
    "pushNotificationConfig": {
        "url": "http://127.0.0.1:8011/callback",
    },
})
print(f"2. Webhook registered for task {set_resp['result']['taskId']}")

# Step 3: wait for the worker to finish + deliver the webhook.
print("3. Sleeping 2s while the worker runs and POSTs the callback…")
time.sleep(2.0)

# Step 4: inspect the receiver's mailbox.
with CALLBACKS_LOCK:
    payloads = list(RECEIVED_CALLBACKS)

print(f"4. Receiver got {len(payloads)} callback(s).")
for payload in payloads:
    final = Task.model_validate(payload)
    print(f"   task {final.id}")
    print(f"     state: {final.status.state}")
    for artifact in final.artifacts:
        print(f"     artifact: {artifact.name}")
        for part in artifact.parts:
            print(f"       • {part.text}")
```

- [ ] **Step 3: Run and verify**

Expected output (UUIDs differ):

```
1. Task submitted: id=<uuid> state=submitted
2. Webhook registered for task <uuid>
3. Sleeping 2s while the worker runs and POSTs the callback…
4. Receiver got 1 callback(s).
   task <uuid>
     state: completed
     artifact: facts-about-octopuses
       • Octopuses have three hearts.
       • They can change color in under a second.
       • Each of their arms has its own neural cluster.
```

- [ ] **Step 4: Commit**

```bash
git add 06_a2a_push_notifications.ipynb
git commit -m "feat(a2a): demonstrate end-to-end webhook flow in notebook 06"
```

---

## Task 9: Section 6 — Why auth matters here

**Files:** Modify `06_a2a_push_notifications.ipynb` (1 markdown cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 7. A Note on Webhook Auth

We left `PushNotificationConfig.authentication` as a free-form dict because A2A defers webhook auth to the same `securitySchemes` mechanism the Agent Card uses for inbound requests (notebook 02 introduced these; notebook 07 fills them in).

In production this matters more for **webhooks** than for normal inbound requests, because the client's webhook URL might be reachable by anyone on the public internet. Without a verification mechanism, an attacker who learns the URL could POST fake task completions to the client. Two common defenses, both built on top of A2A's existing `securitySchemes`:

1. **Bearer tokens** — the client declares an auth scheme on its webhook (e.g. "bearer my-secret-token"); the server attaches `Authorization: Bearer my-secret-token` when delivering. Receiver rejects requests without it.
2. **JWT signing** — the server signs the payload (or a fingerprint of it) with a private key whose public key the receiver knows. Receiver verifies the signature before trusting the payload.

Notebook 07 wires up both schemes for **inbound** A2A endpoints and shows how the same primitives apply to webhook delivery.
```

- [ ] **Step 2: Commit**

```bash
git add 06_a2a_push_notifications.ipynb
git commit -m "feat(a2a): note webhook auth deferred to nb07 in notebook 06"
```

---

## Task 10: Closing recap, teaser, cleanup

**Files:** Modify `06_a2a_push_notifications.ipynb` (2 markdown cells + 1 code cell).

- [ ] **Step 1: "What you just learned"**

```markdown
## What you just learned

- The `tasks/pushNotificationConfig/set` method and the `TaskPushNotificationConfig` params shape.
- That A2A webhook payloads are just the serialized `Task` — no JSON-RPC envelope, no event wrapper.
- How to run a second FastAPI app in the same notebook to receive callbacks on a different port.
- That webhook delivery is best-effort by spec: the server delivers, but doesn't change the task's state on delivery failure.
- Why webhook delivery without auth is a security hole, and which A2A primitives notebook 07 will use to close it.
```

- [ ] **Step 2: "What's missing"**

```markdown
## What's missing

Authentication. Every endpoint and webhook we've built so far accepts anyone who can reach the URL. Real agents need to know who's calling, and webhook receivers need to know who's posting.

In **notebook 07** we wire up the Agent Card's `securitySchemes` / `security` fields with three patterns:

- **No auth** (the implicit default in 01–06, made explicit).
- **Bearer token** — declared in the card, attached as `Authorization: Bearer …`, validated server-side.
- **OAuth2 client credentials** — declared with scopes, walkthrough of the flow with a mocked identity provider.

We'll also revisit the webhook from this notebook and make the server **sign** its callback so the receiver can verify it.
```

- [ ] **Step 3: Cleanup code cell**

```python
shutdown_all_servers()
print("All servers stopped.")
```

- [ ] **Step 4: Run, expect `All servers stopped.`**

- [ ] **Step 5: Commit**

```bash
git add 06_a2a_push_notifications.ipynb
git commit -m "feat(a2a): add closing recap and cleanup to notebook 06"
```

---

## Task 11: Fresh-kernel verification

**Files:** none.

- [ ] **Step 1: Execute**

```bash
/opt/anaconda3/bin/jupyter nbconvert --to notebook --execute --inplace 06_a2a_push_notifications.ipynb
```

Expected: succeeds with no errors. The demo cell takes ~2.5s (1.5s worker + 1s callback overhead + the explicit sleep).

- [ ] **Step 2: Verify expected outputs**

In order:

1. `Setup OK`
2. `Models defined.`
3. `Callback receiver running on http://127.0.0.1:8011 (POST /callback)`
4. `Researcher running on http://127.0.0.1:8010`
5. The 4-line demo block:
   - `1. Task submitted: id=<uuid> state=submitted`
   - `2. Webhook registered for task <uuid>`
   - `3. Sleeping 2s while the worker runs and POSTs the callback…`
   - `4. Receiver got 1 callback(s).` + task block with state `completed` and the three octopus facts under `facts-about-octopuses`.
6. `All servers stopped.`

No cell raises. The crucial line is `Receiver got 1 callback(s).` — if it's `0`, the webhook delivery isn't reaching the receiver (usually a port collision or a `_deliver_webhook` exception).

- [ ] **Step 3: Verify ports released**

```bash
lsof -iTCP:8010 -sTCP:LISTEN
lsof -iTCP:8011 -sTCP:LISTEN
```

Expected: both empty.

- [ ] **Step 4: Commit clean run**

```bash
git add 06_a2a_push_notifications.ipynb
git commit -m "chore(a2a): commit clean fresh-kernel run of notebook 06"
```

---

## Done

After Task 11 passes, notebook 06 is complete. Next plan: `07_a2a_auth.ipynb`.
