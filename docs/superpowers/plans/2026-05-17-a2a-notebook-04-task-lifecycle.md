# A2A Notebook 04 — `04_a2a_task_lifecycle.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the fourth notebook in the A2A learning series — the task lifecycle. The learner sees a task move through `submitted → working → completed`, polls progress with `tasks/get`, cancels a running task via `tasks/cancel`, and runs a multi-turn conversation driven by the `input-required` state. The notebook closes by making polling feel wasteful, motivating notebook 05 (streaming).

**Architecture:** A single Jupyter notebook with one researcher FastAPI app that maintains a thread-safe in-memory task store and spawns a background worker per `message/send` call. The worker simulates ~1.5s of work and transitions the task through `submitted → working → completed`/`failed`/`canceled`. The same handler routes `tasks/get`, `tasks/cancel`, and follow-up `message/send` calls (those with `taskId` set) for the multi-turn `input-required` flow.

**Tech Stack:** Python 3.11+, Jupyter, FastAPI, uvicorn, httpx, pydantic v2, threading. Same stack as notebooks 01–03.

**Companion spec:** `docs/superpowers/specs/2026-05-17-a2a-protocol-learning-series-design.md` (notebook 04 section).

**A2A spec version targeted:** v0.3.0. Method names: `message/send`, `tasks/get`, `tasks/cancel`. Task states match the v0.3.0 `TaskState` literal from notebook 03.

**Port assignment:** Researcher on `127.0.0.1:8010` (ports 8000/8001 are off-limits).

---

## File Structure

- **Create:** `04_a2a_task_lifecycle.ipynb` — self-contained.
- **Modify:** none.
- **No separate test files.** Fresh-kernel run is the test (Task 12).

## Notebook Section Map

1. Title + "Why this notebook exists"
2. "What you'll learn"
3. Setup (imports + helpers, copied forward from nb03)
4. Section 1: The task lifecycle (markdown — ASCII state diagram)
5. Section 2: Models (full A2A model set from nb03 + new `TaskQueryParams` and `TaskIdParams`)
6. Section 3: The slow researcher (server with task store + background worker + handlers)
7. Section 4: Polling with `tasks/get` (client: send, then loop polling until terminal state)
8. Section 5: Cancelling a running task (`tasks/cancel`)
9. Section 6: Multi-turn with `input-required` (server asks for clarification, client sends follow-up `message/send` with `taskId`)
10. Section 7: Polling feels wasteful (markdown reflection motivating notebook 05)
11. "What you just learned"
12. "What's missing"
13. Cleanup

---

## Task 1: Create the notebook scaffold

**Files:** Create `04_a2a_task_lifecycle.ipynb`.

- [ ] **Step 1: Write an empty notebook with the Python 3 kernel**

```json
{
  "cells": [],
  "metadata": {
    "kernelspec": {
      "display_name": "Python 3",
      "language": "python",
      "name": "python3"
    },
    "language_info": {
      "name": "python",
      "version": "3.11"
    }
  },
  "nbformat": 4,
  "nbformat_minor": 5
}
```

- [ ] **Step 2: Verify valid JSON**

```bash
python -c "import json; json.load(open('04_a2a_task_lifecycle.ipynb'))"
```

Expected: no exception.

- [ ] **Step 3: Commit**

```bash
git add 04_a2a_task_lifecycle.ipynb
git commit -m "feat(a2a): scaffold 04_a2a_task_lifecycle.ipynb"
```

---

## Task 2: Title and "Why this notebook exists"

**Files:** Modify `04_a2a_task_lifecycle.ipynb` (add 2 markdown cells).

- [ ] **Step 1: Title + "Why" markdown cell**

```markdown
# 04 — Task Lifecycle: States, Polling, Cancel, and Multi-Turn

## Why this notebook exists

In **notebook 03** every `message/send` call returned a `Task` that was already `completed` (or `failed`). The agent did its work synchronously inside the request handler and the client got the answer in the same HTTP round-trip.

That's fine for fast lookups. Real agents do things that take seconds to minutes: hitting external APIs, running models, waiting on humans. A2A models that with a small but powerful state machine on the `Task` object itself, plus three methods that operate on long-running tasks:

- `tasks/get` — *"what's the status of task X right now?"*
- `tasks/cancel` — *"stop working on task X."*
- A second `message/send` with `taskId` set — *"here's the clarification you asked for on task X."*

This notebook builds a "slow researcher" that runs work in a background thread, exposes all four methods, and walks through every transition you can plausibly hit.

> *Targets A2A spec v0.3.0.*
```

- [ ] **Step 2: "What you'll learn" markdown cell**

```markdown
## What you'll learn

- The A2A task state machine and which states are terminal vs. transient.
- How to implement a server that spawns background work and returns a `submitted` task immediately.
- How `tasks/get` lets a client poll for progress.
- How `tasks/cancel` stops in-flight work cleanly.
- How a server requests more information mid-task with the `input-required` state, and how the client replies with a follow-up `message/send` whose `Message.taskId` references the original task.
- Why polling is fundamentally wasteful — setting up notebook 05's streaming.
```

- [ ] **Step 3: Commit**

```bash
git add 04_a2a_task_lifecycle.ipynb
git commit -m "feat(a2a): add intro markdown to notebook 04"
```

---

## Task 3: Setup cell

**Files:** Modify `04_a2a_task_lifecycle.ipynb` (markdown header + code cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 1. Setup

Same helpers as previous notebooks, plus `threading.Lock`, `threading.Event`, and a couple of helpers for the task store.
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
from fastapi import FastAPI
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
git add 04_a2a_task_lifecycle.ipynb
git commit -m "feat(a2a): add setup cell to notebook 04"
```

---

## Task 4: Section 1 — The task lifecycle

**Files:** Modify `04_a2a_task_lifecycle.ipynb` (1 markdown cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 2. The Task Lifecycle

A2A tasks carry a `status.state` that walks a deliberate state machine. Most calls move the task from one state to another; some states are terminal (the task is done, one way or another).

```
        ┌───────────┐
        │ submitted │  ← initial state on message/send acceptance
        └─────┬─────┘
              │ worker picks it up
              ▼
        ┌───────────┐                ┌────────────────┐
        │  working  │ ─ needs info ▶ │ input-required │
        └─────┬─────┘                └───────┬────────┘
              │                              │ client sends
              │                              │ message/send w/ taskId
              ▼                              ▼
   ┌──────────────────┐                ┌──────────┐
   │     completed    │ ◀──────────────│  working │
   └──────────────────┘  (resumed)     └─────┬────┘
                                             │
                          ┌──────────┐       │
                          │  failed  │ ◀─────┤
                          └──────────┘       │
                          ┌──────────┐       │
                          │ canceled │ ◀─────┘  (via tasks/cancel)
                          └──────────┘
```

Terminal states: **`completed`**, **`failed`**, **`canceled`**, **`rejected`**, **`auth-required`** (in the sense that further protocol action is needed before work continues).

Transient states: **`submitted`**, **`working`**, **`input-required`**, **`unknown`**.

The full v0.3.0 set: `submitted`, `working`, `input-required`, `completed`, `canceled`, `failed`, `rejected`, `auth-required`, `unknown`. (Notebook 03 already defined `TaskState` with all nine values.)

We'll exercise `submitted → working → completed`, `working → canceled`, and `working → input-required → working → completed` in this notebook.
```

- [ ] **Step 2: Verify the ASCII diagram renders inside a fenced code block**

- [ ] **Step 3: Commit**

```bash
git add 04_a2a_task_lifecycle.ipynb
git commit -m "feat(a2a): document task state machine in notebook 04"
```

---

## Task 5: Section 2 — Models

**Files:** Modify `04_a2a_task_lifecycle.ipynb` (markdown + code cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 3. Models

The A2A object set carries forward from notebook 03 (`TextPart`, `Message`, `TaskStatus`, `Artifact`, `Task`, plus the JSON-RPC envelope). We add two small param objects for the lifecycle methods:

- **`TaskQueryParams`** — params for `tasks/get`. Required: `id`. Optional: `historyLength` (truncates `Task.history` in the response).
- **`TaskIdParams`** — params for `tasks/cancel`. Just `id`.
```

- [ ] **Step 2: Models code cell**

```python
TaskState = Literal[
    "submitted",
    "working",
    "input-required",
    "completed",
    "canceled",
    "failed",
    "rejected",
    "auth-required",
    "unknown",
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


class TaskQueryParams(BaseModel):
    id: str
    historyLength: int | None = None


class TaskIdParams(BaseModel):
    id: str


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
            raise ValueError(
                "JSONRPCResponse must contain exactly one of `result` or `error`"
            )
        return self


print("Models defined.")
```

- [ ] **Step 3: Run, expect `Models defined.`**

- [ ] **Step 4: Commit**

```bash
git add 04_a2a_task_lifecycle.ipynb
git commit -m "feat(a2a): add lifecycle models to notebook 04"
```

---

## Task 6: Section 3 — The slow researcher

**Files:** Modify `04_a2a_task_lifecycle.ipynb` (markdown + code cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 4. The Slow Researcher

We rebuild the researcher from notebook 03, but with three changes:

1. **Server-side task store.** A dict of `task_id → Task`, protected by a `threading.Lock`. Per task, we also keep a `threading.Event` we can `set()` to signal cancellation, and the message history (the original message plus any follow-ups).
2. **Background workers.** Every `message/send` spawns a daemon thread that does ~1.5s of "research" in three 0.5s steps. After each step, the worker checks the cancel event. On completion, it transitions the task to `completed` (or `failed` if the topic is unknown).
3. **Multi-method routing.** The single `POST /` handler routes on `req.method` — `message/send` (with or without `taskId`), `tasks/get`, `tasks/cancel`.

The handler also recognises one special topic — `"clarify"` — which triggers the `input-required` flow demonstrated in Section 6. (We do this with a sentinel topic rather than real ambiguity detection so the demo is deterministic.)
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


# Server-side stores. All access must hold STORE_LOCK.
TASKS: dict[str, Task] = {}
TASK_HISTORY: dict[str, list[Message]] = {}
CANCEL_SIGNALS: dict[str, threading.Event] = {}
STORE_LOCK = threading.Lock()


def _set_status(task_id: str, **kw) -> None:
    with STORE_LOCK:
        TASKS[task_id].status = TaskStatus(timestamp=now_iso(), **kw)


def _set_artifacts(task_id: str, artifacts: list[Artifact]) -> None:
    with STORE_LOCK:
        TASKS[task_id].artifacts = artifacts


def _facts_artifact(topic: str, facts: list[str]) -> Artifact:
    return Artifact(
        artifactId=str(uuid.uuid4()),
        name=f"facts-about-{topic}",
        parts=[TextPart(text=fact) for fact in facts],
    )


def _agent_message(text: str) -> Message:
    return Message(
        messageId=str(uuid.uuid4()),
        role="agent",
        parts=[TextPart(text=text)],
    )


def _slow_worker(task_id: str, topic: str) -> None:
    """Simulate ~1.5s of research work, watching for cancellation."""
    cancel = CANCEL_SIGNALS[task_id]

    _set_status(task_id, state="working")

    for _ in range(3):  # three 0.5s steps
        if cancel.wait(timeout=0.5):
            _set_status(task_id, state="canceled")
            return

    facts = FACTS_BY_TOPIC.get(topic)
    if facts:
        _set_artifacts(task_id, [_facts_artifact(topic, facts)])
        _set_status(task_id, state="completed")
    else:
        _set_status(
            task_id,
            state="failed",
            message=_agent_message(f"No facts on file for {topic!r}."),
        )


researcher_app = FastAPI()


def _ok(rpc_id, task: Task) -> JSONRPCResponse:
    return JSONRPCResponse(id=rpc_id, result=task.model_dump(exclude_none=True))


def _err(rpc_id, code: int, message: str) -> JSONRPCResponse:
    return JSONRPCResponse(id=rpc_id, error=JSONRPCError(code=code, message=message))


def _new_task(initial_message: Message, state: TaskState, status_message: Message | None = None) -> Task:
    return Task(
        id=str(uuid.uuid4()),
        contextId=str(uuid.uuid4()),
        status=TaskStatus(state=state, timestamp=now_iso(), message=status_message),
        history=[initial_message],
    )


def _handle_new_message(rpc_id, msg: Message) -> JSONRPCResponse:
    topic = msg.parts[0].text.strip().lower()

    if topic == "clarify":
        # Demo the input-required flow: enter that state immediately, await a follow-up.
        task = _new_task(
            msg,
            state="input-required",
            status_message=_agent_message(
                "Which topic? Reply with one of: octopuses, rome."
            ),
        )
        with STORE_LOCK:
            TASKS[task.id] = task
            TASK_HISTORY[task.id] = [msg]
        return _ok(rpc_id, task)

    # Normal path: spawn a background worker.
    task = _new_task(msg, state="submitted")
    with STORE_LOCK:
        TASKS[task.id] = task
        TASK_HISTORY[task.id] = [msg]
        CANCEL_SIGNALS[task.id] = threading.Event()

    threading.Thread(
        target=_slow_worker, args=(task.id, topic), daemon=True,
    ).start()
    return _ok(rpc_id, task)


def _handle_followup(rpc_id, msg: Message) -> JSONRPCResponse:
    """Follow-up message on an existing task (taskId is set)."""
    with STORE_LOCK:
        task = TASKS.get(msg.taskId)
    if task is None:
        return _err(rpc_id, -32602, f"Unknown taskId: {msg.taskId}")
    if task.status.state not in ("input-required",):
        return _err(rpc_id, -32602, f"Task {msg.taskId} is not awaiting input (state={task.status.state}).")

    clarification = msg.parts[0].text.strip().lower()
    with STORE_LOCK:
        TASK_HISTORY[msg.taskId].append(msg)
    facts = FACTS_BY_TOPIC.get(clarification)
    if facts:
        _set_artifacts(msg.taskId, [_facts_artifact(clarification, facts)])
        _set_status(msg.taskId, state="completed")
    else:
        _set_status(
            msg.taskId,
            state="failed",
            message=_agent_message(f"Still no facts on file for {clarification!r}."),
        )
    with STORE_LOCK:
        return _ok(rpc_id, TASKS[msg.taskId])


def _handle_tasks_get(rpc_id, params) -> JSONRPCResponse:
    try:
        q = TaskQueryParams.model_validate(params)
    except ValidationError as e:
        return _err(rpc_id, -32602, f"Invalid params: {e.errors()}")
    with STORE_LOCK:
        task = TASKS.get(q.id)
        if task is None:
            return _err(rpc_id, -32602, f"Unknown taskId: {q.id}")
        # Build a response copy with history attached (optionally truncated).
        history = TASK_HISTORY.get(q.id, [])
        if q.historyLength is not None:
            history = history[-q.historyLength:]
        snapshot = task.model_copy(update={"history": history})
    return _ok(rpc_id, snapshot)


def _handle_tasks_cancel(rpc_id, params) -> JSONRPCResponse:
    try:
        p = TaskIdParams.model_validate(params)
    except ValidationError as e:
        return _err(rpc_id, -32602, f"Invalid params: {e.errors()}")
    with STORE_LOCK:
        task = TASKS.get(p.id)
        if task is None:
            return _err(rpc_id, -32602, f"Unknown taskId: {p.id}")
        cancel = CANCEL_SIGNALS.get(p.id)
    if cancel is not None:
        cancel.set()
    # Mark it canceled immediately so the response reflects the new state, even
    # if the worker hasn't observed the event yet.
    _set_status(p.id, state="canceled")
    with STORE_LOCK:
        return _ok(rpc_id, TASKS[p.id])


@researcher_app.post("/", response_model_exclude_none=True)
def handle_jsonrpc(req: JSONRPCRequest) -> JSONRPCResponse:
    if req.method == "message/send":
        try:
            msg = Message.model_validate((req.params or {}).get("message"))
        except ValidationError as e:
            return _err(req.id, -32602, f"Invalid params: {e.errors()}")
        if msg.taskId:
            return _handle_followup(req.id, msg)
        return _handle_new_message(req.id, msg)
    if req.method == "tasks/get":
        return _handle_tasks_get(req.id, req.params or {})
    if req.method == "tasks/cancel":
        return _handle_tasks_cancel(req.id, req.params or {})
    return _err(req.id, -32601, f"Method not found: {req.method!r}")


researcher_server = run_server_in_thread(researcher_app, port=8010)
print("Slow researcher running on http://127.0.0.1:8010")
```

- [ ] **Step 3: Run, expect `Slow researcher running on http://127.0.0.1:8010`**

- [ ] **Step 4: Commit**

```bash
git add 04_a2a_task_lifecycle.ipynb
git commit -m "feat(a2a): build slow researcher with task store and worker in notebook 04"
```

---

## Task 7: Section 4 — Polling with `tasks/get`

**Files:** Modify `04_a2a_task_lifecycle.ipynb` (markdown + code cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 5. Polling with `tasks/get`

A client `message/send` now gets back a `Task` in the `submitted` state — the work is just starting. To find out when it's done, the client calls `tasks/get` repeatedly until the task reaches a terminal state.

The helpers below wrap the JSON-RPC envelope so the demo reads naturally.
```

- [ ] **Step 2: Polling code cell**

```python
TERMINAL_STATES: set[TaskState] = {
    "completed", "failed", "canceled", "rejected",
}


def _rpc(method: str, params: dict) -> JSONRPCResponse:
    envelope = {
        "jsonrpc": "2.0",
        "id": str(uuid.uuid4()),
        "method": method,
        "params": params,
    }
    r = httpx.post("http://127.0.0.1:8010/", json=envelope, timeout=10.0)
    r.raise_for_status()
    return JSONRPCResponse.model_validate(r.json())


def send_message(text: str, task_id: str | None = None) -> Task:
    msg = {
        "messageId": str(uuid.uuid4()),
        "role": "user",
        "parts": [{"kind": "text", "text": text}],
        "kind": "message",
    }
    if task_id is not None:
        msg["taskId"] = task_id
    resp = _rpc("message/send", {"message": msg})
    if resp.error:
        raise RuntimeError(f"send_message error {resp.error.code}: {resp.error.message}")
    return Task.model_validate(resp.result)


def get_task(task_id: str, history_length: int | None = None) -> Task:
    params: dict = {"id": task_id}
    if history_length is not None:
        params["historyLength"] = history_length
    resp = _rpc("tasks/get", params)
    if resp.error:
        raise RuntimeError(f"tasks/get error {resp.error.code}: {resp.error.message}")
    return Task.model_validate(resp.result)


def cancel_task(task_id: str) -> Task:
    resp = _rpc("tasks/cancel", {"id": task_id})
    if resp.error:
        raise RuntimeError(f"tasks/cancel error {resp.error.code}: {resp.error.message}")
    return Task.model_validate(resp.result)


# Kick off a slow research task on "octopuses" and poll it to completion.
initial = send_message("octopuses")
print(f"Task {initial.id} accepted, initial state: {initial.status.state}")

poll_count = 0
task = initial
while task.status.state not in TERMINAL_STATES:
    time.sleep(0.3)
    poll_count += 1
    task = get_task(task.id)
    print(f"  poll #{poll_count}: state={task.status.state}")

print(f"Final state: {task.status.state}")
for artifact in task.artifacts:
    for part in artifact.parts:
        print(f"  • {part.text}")
```

- [ ] **Step 3: Run and verify**

Expected output (poll counts will vary slightly with timing):

```
Task <uuid> accepted, initial state: submitted
  poll #1: state=working
  poll #2: state=working
  poll #3: state=working
  poll #4: state=working
  poll #5: state=completed
Final state: completed
  • Octopuses have three hearts.
  • They can change color in under a second.
  • Each of their arms has its own neural cluster.
```

(The number of `working` polls before the `completed` poll may be 3–6 depending on jitter; that's fine.)

- [ ] **Step 4: Commit**

```bash
git add 04_a2a_task_lifecycle.ipynb
git commit -m "feat(a2a): demonstrate tasks/get polling in notebook 04"
```

---

## Task 8: Section 5 — Cancelling a running task

**Files:** Modify `04_a2a_task_lifecycle.ipynb` (markdown + code cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 6. Cancelling a Running Task

We kick off a task, wait ~0.3s, then send `tasks/cancel`. The worker sees the cancel event on its next 0.5s checkpoint and exits. `tasks/cancel` returns the task already in `canceled` state — the server sets the state synchronously so the client doesn't have to poll one more time to see it.

The worker continuing to run for a fraction of a second after `tasks/cancel` returns is a real concern in production A2A servers — you'd typically use stronger primitives (asyncio cancellation, process-level signals) for genuinely long work. For this notebook the `threading.Event` is plenty.
```

- [ ] **Step 2: Cancel code cell**

```python
task = send_message("rome")
print(f"Task {task.id} started: {task.status.state}")

time.sleep(0.3)  # Let the worker get into "working" state.

canceled = cancel_task(task.id)
print(f"After cancel:        {canceled.status.state}")

time.sleep(0.6)  # Wait long enough for the worker to actually exit.
final = get_task(task.id)
print(f"After worker exits:  {final.status.state}")
```

- [ ] **Step 3: Run and verify**

Expected output (UUID will differ):

```
Task <uuid> started: submitted
After cancel:        canceled
After worker exits:  canceled
```

- [ ] **Step 4: Commit**

```bash
git add 04_a2a_task_lifecycle.ipynb
git commit -m "feat(a2a): demonstrate tasks/cancel in notebook 04"
```

---

## Task 9: Section 6 — Multi-turn with `input-required`

**Files:** Modify `04_a2a_task_lifecycle.ipynb` (markdown + 2 code cells).

- [ ] **Step 1: Markdown cell**

```markdown
## 7. Multi-Turn with `input-required`

If the agent needs more information to complete a task, it transitions to `input-required` and attaches a `Message` to `status.message` explaining what it needs. The client then sends a second `message/send` — but this time the `Message` carries the original task's `taskId`. The server treats it as a continuation rather than a new task.

To trigger the demo path, send the literal topic `clarify`. The server immediately replies with `input-required` and a prompt; we then reply with `octopuses` (referencing the same task id) and watch it complete.
```

- [ ] **Step 2: Initial request code cell**

```python
pending = send_message("clarify")
print(f"Task {pending.id}")
print(f"  state: {pending.status.state}")
if pending.status.message:
    for part in pending.status.message.parts:
        print(f"  agent asks: {part.text}")
```

- [ ] **Step 3: Run and verify**

Expected output (UUID differs):

```
Task <uuid>
  state: input-required
  agent asks: Which topic? Reply with one of: octopuses, rome.
```

- [ ] **Step 4: Follow-up code cell**

```python
resumed = send_message("octopuses", task_id=pending.id)
print(f"Task {resumed.id}  (same id: {resumed.id == pending.id})")
print(f"  state: {resumed.status.state}")
for artifact in resumed.artifacts:
    print(f"  artifact: {artifact.name}")
    for part in artifact.parts:
        print(f"    • {part.text}")

# Inspect the history we accumulated on the server.
full = get_task(pending.id)
print()
print(f"history length: {len(full.history)}")
for i, m in enumerate(full.history):
    text = " | ".join(p.text for p in m.parts)
    print(f"  [{i}] {m.role}: {text}")
```

- [ ] **Step 5: Run and verify**

Expected output (UUIDs differ):

```
Task <uuid>  (same id: True)
  state: completed
  artifact: facts-about-octopuses
    • Octopuses have three hearts.
    • They can change color in under a second.
    • Each of their arms has its own neural cluster.

history length: 2
  [0] user: clarify
  [1] user: octopuses
```

(`history` contains only the user-side messages because the server-side store only appended user messages. That's a deliberate simplification we'll relax in later notebooks.)

- [ ] **Step 6: Commit**

```bash
git add 04_a2a_task_lifecycle.ipynb
git commit -m "feat(a2a): demonstrate input-required multi-turn in notebook 04"
```

---

## Task 10: Section 7 — Polling feels wasteful

**Files:** Modify `04_a2a_task_lifecycle.ipynb` (1 markdown cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 8. Polling Feels Wasteful

Look back at the polling cell. We sent ~4 `tasks/get` requests just to learn that the task was *still* working. Each one was an HTTP round-trip the server had to handle, and we still waited up to one polling interval after the task finished to find out.

Scale this out: a hundred concurrent users on long-running tasks, each polling every 300ms, equals 333 requests/second of pure progress checking. None of those requests are doing useful work.

A2A's answer is **`message/stream`** — the server holds the connection open and pushes status changes as Server-Sent Events. The client doesn't poll; it just consumes a stream.

That's notebook 05.
```

- [ ] **Step 2: Commit**

```bash
git add 04_a2a_task_lifecycle.ipynb
git commit -m "feat(a2a): motivate streaming via polling pain in notebook 04"
```

---

## Task 11: Closing recap, teaser, and cleanup

**Files:** Modify `04_a2a_task_lifecycle.ipynb` (2 markdown cells + 1 code cell).

- [ ] **Step 1: "What you just learned" markdown cell**

```markdown
## What you just learned

- The A2A task state machine: nine valid states, terminal vs. transient.
- How to implement a server that returns a `submitted` task immediately and runs work in the background.
- `tasks/get` for polling, with an optional `historyLength` parameter to truncate the returned history.
- `tasks/cancel` for stopping in-flight work, using a `threading.Event` (or equivalent) to coordinate with the worker.
- The `input-required` state and how a client continues a task by sending `message/send` with `Message.taskId` set.
- Why polling is wasteful at scale — every quiet poll is wasted work.
```

- [ ] **Step 2: "What's missing" markdown cell**

```markdown
## What's missing

Polling. Specifically: a way for the client to learn about state changes as they happen, without re-asking.

In **notebook 05** we wire up `message/stream` — the same `message/send` semantics, but the server holds the HTTP connection open and pushes `TaskStatusUpdateEvent` and `TaskArtifactUpdateEvent` chunks over **Server-Sent Events** as the worker makes progress. The client consumes the stream with `httpx.stream()`. No more polling.
```

- [ ] **Step 3: Cleanup code cell**

```python
shutdown_all_servers()
print("All servers stopped.")
```

- [ ] **Step 4: Run, expect `All servers stopped.`**

- [ ] **Step 5: Commit**

```bash
git add 04_a2a_task_lifecycle.ipynb
git commit -m "feat(a2a): add closing recap and cleanup to notebook 04"
```

---

## Task 12: End-to-end verification

**Files:** none.

- [ ] **Step 1: Execute the notebook top-to-bottom in a clean kernel**

```bash
/opt/anaconda3/bin/jupyter nbconvert --to notebook --execute --inplace 04_a2a_task_lifecycle.ipynb
```

Expected: succeeds with no errors. The notebook is rewritten with embedded outputs.

- [ ] **Step 2: Verify expected outputs**

The following must appear, in order, with no unhandled exception:

1. `Setup OK`
2. `Models defined.`
3. `Slow researcher running on http://127.0.0.1:8010`
4. The polling demo block: `Task <uuid> accepted, initial state: submitted` followed by 3–6 `poll #N: state=working` lines, ending with `poll #?: state=completed`, then `Final state: completed`, then the three octopus facts.
5. The cancel demo block: `started: submitted`, `After cancel: canceled`, `After worker exits: canceled`.
6. The input-required initial response: `state: input-required` + `agent asks: Which topic? Reply with one of: octopuses, rome.`
7. The resumed completion: `(same id: True)`, `state: completed`, octopus artifact + three facts, `history length: 2`, two user-role history entries (`[0] user: clarify`, `[1] user: octopuses`).
8. `All servers stopped.`

Timing-sensitive variance acceptable: the exact number of `working` polls in step 4. Anything else differing from the above is a bug.

- [ ] **Step 3: Verify port released**

```bash
lsof -iTCP:8010 -sTCP:LISTEN
```

Expected: empty.

- [ ] **Step 4: Commit clean run**

```bash
git add 04_a2a_task_lifecycle.ipynb
git commit -m "chore(a2a): commit clean fresh-kernel run of notebook 04"
```

---

## Done

After Task 12 passes, notebook 04 is complete. The next plan to write is for `05_a2a_streaming.ipynb`, which introduces `message/stream` and Server-Sent Events.
