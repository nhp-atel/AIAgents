# A2A Notebook 05 — `05_a2a_streaming.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the fifth notebook in the A2A learning series — `message/stream`. The server emits incremental progress over **Server-Sent Events**; the client consumes them with `httpx.stream()`. Replaces the polling loop from notebook 04 with a single open connection that yields `TaskStatusUpdateEvent`s and `TaskArtifactUpdateEvent`s as the work progresses, ending with a final status update marked `final: true`.

**Architecture:** A single Jupyter notebook with one researcher FastAPI app. The new endpoint is still `POST /` but routes on `req.method == "message/stream"` to a `StreamingResponse`. The response body is a sequence of SSE `data:` lines, each carrying a complete JSON-RPC 2.0 response wrapping one event. The server uses an inline generator (no background thread) — the work *is* the streaming; `time.sleep` between yields simulates real work. Client uses `httpx.stream("POST", ...)` and iterates lines.

**Tech Stack:** Python 3.11+, Jupyter, FastAPI, uvicorn, httpx, pydantic v2, threading. Same stack as notebooks 01–04.

**Companion spec:** `docs/superpowers/specs/2026-05-17-a2a-protocol-learning-series-design.md` (notebook 05 section).

**A2A spec version targeted:** v0.3.0. Method: `message/stream`. Events: `taskStatusUpdateEvent`, `taskArtifactUpdateEvent`. Each SSE `data:` line is a JSON-RPC 2.0 Response object whose `result` is the event itself.

**Port assignment:** Researcher on `127.0.0.1:8010`.

---

## File Structure

- **Create:** `05_a2a_streaming.ipynb` — self-contained.
- **Modify:** none.

## Notebook Section Map

1. Title + "Why this notebook exists"
2. "What you'll learn"
3. Setup
4. Section 1: SSE in 60 seconds (markdown)
5. Section 2: Event models (`TaskStatusUpdateEvent`, `TaskArtifactUpdateEvent`)
6. Section 3: The streaming researcher (server)
7. Section 4: Consuming the stream (client)
8. Section 5: Polling vs streaming (markdown recap)
9. "What you just learned"
10. "What's missing"
11. Cleanup

---

## Task 1: Scaffold

**Files:** Create `05_a2a_streaming.ipynb`.

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

- [ ] **Step 2: Verify JSON**

```bash
python -c "import json; json.load(open('05_a2a_streaming.ipynb'))"
```

- [ ] **Step 3: Commit**

```bash
git add 05_a2a_streaming.ipynb
git commit -m "feat(a2a): scaffold 05_a2a_streaming.ipynb"
```

---

## Task 2: Title + intro markdown

**Files:** Modify `05_a2a_streaming.ipynb` (2 markdown cells).

- [ ] **Step 1: Title + "Why"**

```markdown
# 05 — Streaming: `message/stream` over Server-Sent Events

## Why this notebook exists

In **notebook 04** the client polled `tasks/get` to learn about state changes. Every polling interval that didn't produce news was wasted work, and the client always lagged the server by up to one interval. With many concurrent users on slow tasks, that adds up to a lot of useless requests.

A2A's answer is `message/stream`. Same semantics as `message/send` — *"please do this thing"* — but the server keeps the HTTP connection open and **pushes** events as the work progresses: status updates, artifacts, and a final status that closes the stream.

This notebook builds a streaming researcher, walks through the wire format (Server-Sent Events wrapping JSON-RPC responses), and consumes the stream from a client with no polling at all.

> *Targets A2A spec v0.3.0.*
```

- [ ] **Step 2: "What you'll learn"**

```markdown
## What you'll learn

- The Server-Sent Events (SSE) wire format and the `text/event-stream` content type.
- How A2A wraps each SSE event as a complete JSON-RPC 2.0 response.
- The two event kinds: **`TaskStatusUpdateEvent`** (state transitions, with `final: true` on the last one) and **`TaskArtifactUpdateEvent`** (output deliveries).
- How to implement the server side with FastAPI's `StreamingResponse` and a generator.
- How to consume the stream on the client side with `httpx.stream()`.
- Why streaming is strictly better than polling for any non-trivial task — and the one tradeoff (held-open connections) that motivates push notifications in notebook 06.
```

- [ ] **Step 3: Commit**

```bash
git add 05_a2a_streaming.ipynb
git commit -m "feat(a2a): add intro markdown to notebook 05"
```

---

## Task 3: Setup

**Files:** Modify `05_a2a_streaming.ipynb` (markdown header + code cell).

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
from typing import Generator, Literal

import httpx
import uvicorn
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
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
git add 05_a2a_streaming.ipynb
git commit -m "feat(a2a): add setup cell to notebook 05"
```

---

## Task 4: Section 1 — SSE in 60 seconds

**Files:** Modify `05_a2a_streaming.ipynb` (1 markdown cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 2. SSE in 60 seconds

**Server-Sent Events** is a long-lived HTTP response whose body is a series of named events. It's a W3C standard from 2009, supported natively by browsers and by every HTTP client worth using.

Wire format:

```
HTTP/1.1 200 OK
Content-Type: text/event-stream

data: {"some": "json"}

data: {"another": "event"}

data: {"final": "event"}
```

Each event is one or more lines starting with a field name (`data:`, `event:`, `id:`, `retry:`), terminated by a blank line. For A2A we only use `data:` — every event is JSON on a single line.

A2A wraps **each** event as a complete JSON-RPC 2.0 response. So the `data:` payload of every SSE event in an A2A stream has the shape:

```json
{"jsonrpc": "2.0", "id": <request id>, "result": {<event object>}}
```

That means three things:

1. The JSON-RPC `id` is the same on every event of a single stream (the original request's `id`).
2. There is no envelope-level "stream complete" message — instead, the **last** event is a `TaskStatusUpdateEvent` with `final: true`.
3. SSE provides reconnection semantics (the `id:` line, plus the `Last-Event-ID` header on reconnect). A2A inherits all of that for free; we just don't use it in this notebook.
```

- [ ] **Step 2: Verify the fenced code blocks render**

- [ ] **Step 3: Commit**

```bash
git add 05_a2a_streaming.ipynb
git commit -m "feat(a2a): introduce SSE wire format in notebook 05"
```

---

## Task 5: Section 2 — Event models

**Files:** Modify `05_a2a_streaming.ipynb` (markdown + code cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 3. Event Models

Models carry forward from previous notebooks (`TextPart`, `Message`, `TaskStatus`, `Artifact`, plus the JSON-RPC envelope). We add the two A2A streaming event types.

- **`TaskStatusUpdateEvent`** — fired on every status transition. `kind` is the literal `"taskStatusUpdateEvent"`. The `final: true` flag marks the terminal event that closes the stream.
- **`TaskArtifactUpdateEvent`** — fired when the agent produces an artifact (or a chunk of one). `kind` is `"taskArtifactUpdateEvent"`. We use the simple "one artifact, all at once" pattern; the spec also supports chunked artifacts via optional `append` / `lastChunk` flags, which we won't exercise here.
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


class TaskStatusUpdateEvent(BaseModel):
    kind: Literal["taskStatusUpdateEvent"] = "taskStatusUpdateEvent"
    taskId: str
    contextId: str
    status: TaskStatus
    final: bool = False
    metadata: dict | None = None


class TaskArtifactUpdateEvent(BaseModel):
    kind: Literal["taskArtifactUpdateEvent"] = "taskArtifactUpdateEvent"
    taskId: str
    contextId: str
    artifact: Artifact
    append: bool | None = None
    lastChunk: bool | None = None
    metadata: dict | None = None


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
git add 05_a2a_streaming.ipynb
git commit -m "feat(a2a): add SSE event models to notebook 05"
```

---

## Task 6: Section 3 — The streaming researcher

**Files:** Modify `05_a2a_streaming.ipynb` (markdown + code cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 4. The Streaming Researcher

The server lives at `POST /`. For `method == "message/stream"` it returns a FastAPI `StreamingResponse` whose body is a generator yielding SSE-formatted bytes. The generator IS the work — `time.sleep` between yields simulates ~1.5s of effort, and each `yield` pushes one event to the client over the open connection.

For `method == "message/send"` we still return a normal one-shot JSON response — same as notebook 03 — so the same agent can serve both flavors. (Real A2A agents typically support both; the Agent Card's `capabilities.streaming` field tells clients which to prefer.)
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


researcher_app = FastAPI()


def _sse(rpc_id: str | int, event: BaseModel) -> bytes:
    """Format one A2A event as an SSE `data:` frame."""
    envelope = {
        "jsonrpc": "2.0",
        "id": rpc_id,
        "result": event.model_dump(exclude_none=True),
    }
    return f"data: {json.dumps(envelope)}\n\n".encode("utf-8")


def _stream_research(rpc_id: str | int, topic: str) -> Generator[bytes, None, None]:
    task_id = str(uuid.uuid4())
    context_id = str(uuid.uuid4())

    # 1. Acknowledge: task is submitted (could skip this and go straight to working;
    #    we send it so the learner sees every transition).
    yield _sse(rpc_id, TaskStatusUpdateEvent(
        taskId=task_id, contextId=context_id,
        status=TaskStatus(state="submitted", timestamp=now_iso()),
    ))

    # 2. Working: simulate three steps of effort.
    for step in range(1, 4):
        time.sleep(0.5)
        yield _sse(rpc_id, TaskStatusUpdateEvent(
            taskId=task_id, contextId=context_id,
            status=TaskStatus(
                state="working", timestamp=now_iso(),
                message=Message(
                    messageId=str(uuid.uuid4()), role="agent",
                    parts=[TextPart(text=f"…working on step {step}/3")],
                ),
            ),
        ))

    # 3. Result.
    facts = FACTS_BY_TOPIC.get(topic)
    if facts:
        yield _sse(rpc_id, TaskArtifactUpdateEvent(
            taskId=task_id, contextId=context_id,
            artifact=Artifact(
                artifactId=str(uuid.uuid4()),
                name=f"facts-about-{topic}",
                parts=[TextPart(text=fact) for fact in facts],
            ),
        ))
        yield _sse(rpc_id, TaskStatusUpdateEvent(
            taskId=task_id, contextId=context_id,
            status=TaskStatus(state="completed", timestamp=now_iso()),
            final=True,
        ))
    else:
        yield _sse(rpc_id, TaskStatusUpdateEvent(
            taskId=task_id, contextId=context_id,
            status=TaskStatus(
                state="failed", timestamp=now_iso(),
                message=Message(
                    messageId=str(uuid.uuid4()), role="agent",
                    parts=[TextPart(text=f"No facts on file for {topic!r}.")],
                ),
            ),
            final=True,
        ))


@researcher_app.post("/")
def handle_jsonrpc(req: JSONRPCRequest):
    if req.method != "message/stream":
        # Non-streaming methods unsupported in this notebook; client should use streaming.
        body = {
            "jsonrpc": "2.0",
            "id": req.id,
            "error": {"code": -32601, "message": f"Method not found: {req.method!r}"},
        }
        return body  # FastAPI serializes as application/json

    try:
        msg = Message.model_validate((req.params or {}).get("message"))
    except ValidationError as e:
        body = {
            "jsonrpc": "2.0",
            "id": req.id,
            "error": {"code": -32602, "message": f"Invalid params: {e.errors()}"},
        }
        return body

    topic = msg.parts[0].text.strip().lower()
    return StreamingResponse(
        _stream_research(req.id, topic),
        media_type="text/event-stream",
    )


researcher_server = run_server_in_thread(researcher_app, port=8010)
print("Streaming researcher running on http://127.0.0.1:8010")
```

- [ ] **Step 3: Run, expect `Streaming researcher running on http://127.0.0.1:8010`**

- [ ] **Step 4: Commit**

```bash
git add 05_a2a_streaming.ipynb
git commit -m "feat(a2a): build streaming researcher with SSE in notebook 05"
```

---

## Task 7: Section 4 — Consuming the stream

**Files:** Modify `05_a2a_streaming.ipynb` (markdown + 2 code cells).

- [ ] **Step 1: Markdown cell**

```markdown
## 5. Consuming the Stream

`httpx.stream("POST", ...)` returns a context-managed response we can iterate over **as bytes arrive**. We split lines, look for the `data: ` prefix, and `json.loads` the payload.

Two cells: one prints every event with its kind, then a second one extracts only the final completed-state and artifacts to show the protocol-level result.
```

- [ ] **Step 2: First client cell — print every event**

```python
def stream_message(text: str):
    """Yield each parsed event from a message/stream request."""
    envelope = {
        "jsonrpc": "2.0",
        "id": str(uuid.uuid4()),
        "method": "message/stream",
        "params": {
            "message": {
                "messageId": str(uuid.uuid4()),
                "role": "user",
                "parts": [{"kind": "text", "text": text}],
                "kind": "message",
            },
        },
    }
    with httpx.stream(
        "POST", "http://127.0.0.1:8010/", json=envelope, timeout=30.0,
    ) as response:
        response.raise_for_status()
        for line in response.iter_lines():
            if not line.startswith("data: "):
                continue
            payload = json.loads(line[len("data: "):])
            if "error" in payload and payload.get("error") is not None:
                raise RuntimeError(f"JSON-RPC error: {payload['error']}")
            yield payload["result"]


for i, event in enumerate(stream_message("octopuses"), start=1):
    kind = event["kind"]
    if kind == "taskStatusUpdateEvent":
        state = event["status"]["state"]
        final = " (final)" if event.get("final") else ""
        agent_msg = ""
        msg = event["status"].get("message")
        if msg:
            agent_msg = f" — \"{msg['parts'][0]['text']}\""
        print(f"  [{i}] status: {state}{final}{agent_msg}")
    elif kind == "taskArtifactUpdateEvent":
        artifact = event["artifact"]
        print(f"  [{i}] artifact: {artifact['name']} ({len(artifact['parts'])} parts)")
    else:
        print(f"  [{i}] unknown event kind: {kind}")
```

- [ ] **Step 3: Run and verify**

Expected output (event count: 1 submitted + 3 working + 1 artifact + 1 completed = 6 events):

```
  [1] status: submitted
  [2] status: working — "…working on step 1/3"
  [3] status: working — "…working on step 2/3"
  [4] status: working — "…working on step 3/3"
  [5] artifact: facts-about-octopuses (3 parts)
  [6] status: completed (final)
```

- [ ] **Step 4: Second client cell — accumulate the result**

```python
def consume_stream(text: str) -> tuple[str, list[Artifact]]:
    """Run a streaming request and collect the final state + artifacts."""
    final_state = "unknown"
    artifacts: list[Artifact] = []
    for event in stream_message(text):
        if event["kind"] == "taskArtifactUpdateEvent":
            artifacts.append(Artifact.model_validate(event["artifact"]))
        elif event["kind"] == "taskStatusUpdateEvent" and event.get("final"):
            final_state = event["status"]["state"]
    return final_state, artifacts


state, artifacts = consume_stream("rome")
print(f"Final state: {state}")
for artifact in artifacts:
    print(f"  artifact: {artifact.name}")
    for part in artifact.parts:
        print(f"    • {part.text}")
```

- [ ] **Step 5: Run and verify**

Expected output:

```
Final state: completed
  artifact: facts-about-rome
    • Rome was founded in 753 BCE according to tradition.
    • The Roman Empire at its peak spanned roughly 5 million km².
    • Roman concrete used volcanic ash and is still studied today.
```

- [ ] **Step 6: Commit**

```bash
git add 05_a2a_streaming.ipynb
git commit -m "feat(a2a): consume SSE stream from client in notebook 05"
```

---

## Task 8: Section 5 — Polling vs streaming

**Files:** Modify `05_a2a_streaming.ipynb` (1 markdown cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 6. Polling vs Streaming

| | Polling (`tasks/get`, notebook 04) | Streaming (`message/stream`, this notebook) |
|---|---|---|
| **Connections per task** | 1 + N polls | 1 (held open) |
| **Update latency** | Up to one polling interval | Zero (event pushed as soon as it occurs) |
| **Server work for idle ticks** | One handler call per poll | Zero |
| **Backpressure** | Client controls cadence | Server controls cadence |
| **Recovery** | Just resume polling | SSE `Last-Event-ID` (out of scope today) |
| **Cost** | Many short requests | One long-lived connection |

Streaming wins almost every metric. The one real cost is the held-open connection — if your client crashes mid-stream and reconnects with a new HTTP request, the server has no way to push it pending events. For *fire-and-forget* or *come-back-later* scenarios where you don't want a connection at all, A2A offers a third pattern: **push notifications**. That's notebook 06.
```

- [ ] **Step 2: Commit**

```bash
git add 05_a2a_streaming.ipynb
git commit -m "feat(a2a): contrast polling vs streaming in notebook 05"
```

---

## Task 9: Closing recap, teaser, cleanup

**Files:** Modify `05_a2a_streaming.ipynb` (2 markdown cells + 1 code cell).

- [ ] **Step 1: "What you just learned"**

```markdown
## What you just learned

- Server-Sent Events as a long-lived HTTP response with `Content-Type: text/event-stream`.
- The A2A wrapping: every SSE `data:` line is a full JSON-RPC 2.0 response, and the inner `result` is an event object.
- Two event kinds: `TaskStatusUpdateEvent` (state transitions, with `final: true` marking the last one) and `TaskArtifactUpdateEvent` (output delivery).
- How to implement the server side with FastAPI's `StreamingResponse` and a generator that yields SSE-formatted bytes.
- How to consume the stream on the client side with `httpx.stream()` and a parser that splits lines on the `data: ` prefix.
- That streaming subsumes polling for most use cases, but a held-open connection has its own costs that push notifications address.
```

- [ ] **Step 2: "What's missing"**

```markdown
## What's missing

A held-open SSE connection is great while it lasts. But:

- Your client might be a serverless function that gets killed after 30 seconds.
- Your task might run for an hour.
- Your client might be a mobile app whose user closed the screen.

For any of these, you want the agent to *call you back* when the work is done — not require you to hold a connection. A2A supports that with **push notifications**: the client registers a webhook URL once via `tasks/pushNotificationConfig/set`, and the server POSTs the final task to that URL when it's ready.

In **notebook 06** we wire up push notifications, run a second tiny FastAPI app inside this notebook to receive the callback, and inspect the signed payload.
```

- [ ] **Step 3: Cleanup code cell**

```python
shutdown_all_servers()
print("All servers stopped.")
```

- [ ] **Step 4: Run, expect `All servers stopped.`**

- [ ] **Step 5: Commit**

```bash
git add 05_a2a_streaming.ipynb
git commit -m "feat(a2a): add closing recap and cleanup to notebook 05"
```

---

## Task 10: Fresh-kernel verification

**Files:** none (verification only).

- [ ] **Step 1: Execute**

```bash
/opt/anaconda3/bin/jupyter nbconvert --to notebook --execute --inplace 05_a2a_streaming.ipynb
```

Expected: succeeds with no errors.

- [ ] **Step 2: Verify expected outputs**

In order:

1. `Setup OK`
2. `Models defined.`
3. `Streaming researcher running on http://127.0.0.1:8010`
4. The 6-event sequence: submitted → 3× working (with "…working on step N/3" messages) → artifact `facts-about-octopuses` (3 parts) → completed (final). Event indices `[1]` through `[6]`.
5. `Final state: completed`, artifact `facts-about-rome` with three Rome facts.
6. `All servers stopped.`

No cell raises. Cells 4 and 5 each take ~1.5s to complete (three 0.5s sleeps in the generator).

- [ ] **Step 3: Verify port released**

```bash
lsof -iTCP:8010 -sTCP:LISTEN
```

Expected: empty.

- [ ] **Step 4: Commit clean run**

```bash
git add 05_a2a_streaming.ipynb
git commit -m "chore(a2a): commit clean fresh-kernel run of notebook 05"
```

---

## Done

After Task 10 passes, notebook 05 is complete. Next plan: `06_a2a_push_notifications.ipynb`.
