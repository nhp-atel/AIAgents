# A2A Notebook 03 — `03_a2a_first_task.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the third notebook in the A2A learning series — the first **end-to-end A2A protocol exchange**. The learner hand-builds a JSON-RPC 2.0 envelope, sends a `message/send` request, parses the returned `Task` with its embedded `Artifact`, and reads every byte going over the wire.

**Architecture:** A single Jupyter notebook that extends notebook 02's researcher with a `POST /` JSON-RPC handler. Pydantic models capture the A2A v0.3.0 wire format (`Message`, `TextPart`, `Task`, `TaskStatus`, `Artifact`) plus the generic JSON-RPC 2.0 envelope. The notebook drives the server end with manually-constructed envelopes via `httpx.post`, then walks through the request and response byte-by-byte. Concludes with two error cases (unknown method, topic-not-found) to make the JSON-RPC error response and the `failed` task state concrete.

**Tech Stack:** Python 3.11+, Jupyter, FastAPI, uvicorn, httpx, pydantic v2, threading. Same stack as notebooks 01–02.

**Companion spec:** `docs/superpowers/specs/2026-05-17-a2a-protocol-learning-series-design.md` (notebook 03 section).

**A2A spec version targeted:** v0.3.0. Method names match the spec verbatim. The `message/send` shape mirrors what `https://a2a-protocol.org/v0.3.0/specification/` documents.

**Port assignment:** Researcher runs on `127.0.0.1:8010` — same port as notebook 02 (ports 8000/8001 are off-limits because the user runs an unrelated dev server there).

---

## File Structure

- **Create:** `03_a2a_first_task.ipynb` — the entire notebook, self-contained (no imports from notebooks 01 or 02).
- **Modify:** none.
- **No separate test files** — verification is the notebook running top-to-bottom in a fresh kernel (Task 11).

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown)
2. **"What you'll learn"** (markdown bullets)
3. **Setup: imports and server helpers** (code — same pattern as earlier notebooks)
4. **Section 1: JSON-RPC 2.0 in 60 seconds** (markdown only — concept primer)
5. **Section 2: A2A message types** (code — pydantic models for `TextPart`, `Message`, `TaskStatus`, `Artifact`, `Task`, and JSON-RPC envelope)
6. **Section 3: The server's `message/send` handler** (markdown + 1 code cell with FastAPI app)
7. **Section 4: The client's first request** (markdown + 1 code cell building the envelope and POSTing)
8. **Section 5: Reading the wire** (markdown + 1 code cell pretty-printing the raw request and response)
9. **Section 6: Error cases** (markdown + 2 code cells: unknown method, then topic-not-found)
10. **"What you just learned"** (markdown)
11. **"What's missing"** (markdown — teases notebook 04, task lifecycle and multi-turn)
12. **Cleanup** (code)

---

## Task 1: Create the notebook scaffold

**Files:**
- Create: `03_a2a_first_task.ipynb`

- [ ] **Step 1: Write an empty notebook with the Python 3 kernel**

Write the file directly:

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

- [ ] **Step 2: Verify it's valid JSON**

```bash
python -c "import json; json.load(open('03_a2a_first_task.ipynb'))"
ls -la 03_a2a_first_task.ipynb
```

Expected: no exception, file non-zero size.

- [ ] **Step 3: Commit**

```bash
git add 03_a2a_first_task.ipynb
git commit -m "feat(a2a): scaffold 03_a2a_first_task.ipynb"
```

---

## Task 2: Add title and "Why this notebook exists"

**Files:**
- Modify: `03_a2a_first_task.ipynb` (add cells 1–2)

- [ ] **Step 1: Title + "Why" markdown cell**

```markdown
# 03 — The First Task: `message/send` and the JSON-RPC Envelope

## Why this notebook exists

In **notebook 02** we built an Agent Card. We *announced* what the researcher can do — but we never actually *called* it through A2A. The researcher served a JSON document about itself, nothing more.

This notebook closes that loop. We add a single endpoint to the researcher (`POST /`) that speaks the A2A protocol: it accepts a **JSON-RPC 2.0** envelope wrapping a `message/send` request and returns a typed `Task` containing the result.

We hand-build the request. We hand-parse the response. We print the raw bytes both sides exchange. By the end, you should be able to look at any A2A request or response in the wild and know what every field means.

> *Targets A2A spec v0.3.0. The wire format below matches `https://a2a-protocol.org/v0.3.0/specification/` exactly.*
```

- [ ] **Step 2: "What you'll learn" markdown cell**

```markdown
## What you'll learn

- The shape of a **JSON-RPC 2.0** request and response envelope, the wire format A2A uses for almost all method calls.
- The structure of an A2A **Message** (with `role`, `parts`, `messageId`) and its **TextPart** children.
- The structure of an A2A **Task** as a result: `id`, `contextId`, `status`, `artifacts`, `kind`.
- The valid **TaskState** values and which ones mean "done."
- How A2A reports **errors**: JSON-RPC `error` objects with standard codes, *not* HTTP 4xx status codes.
- How to construct A2A requests by hand — useful when debugging real agents in the wild.
```

- [ ] **Step 3: Verify cells render**

Open in Jupyter and confirm markdown renders correctly.

- [ ] **Step 4: Commit**

```bash
git add 03_a2a_first_task.ipynb
git commit -m "feat(a2a): add intro markdown to notebook 03"
```

---

## Task 3: Add the setup cell

**Files:**
- Modify: `03_a2a_first_task.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Markdown cell**

```markdown
## 1. Setup

Same pattern as previous notebooks — FastAPI on a background thread so we can serve and call it from this kernel. Re-defined here because each notebook is self-contained.
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
from pydantic import BaseModel, Field, ValidationError

_servers: list[uvicorn.Server] = []


def run_server_in_thread(app: FastAPI, port: int) -> uvicorn.Server:
    """Start `app` on localhost:`port` in a background daemon thread."""
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

- [ ] **Step 3: Run the cell**

Expected output: `Setup OK`.

- [ ] **Step 4: Commit**

```bash
git add 03_a2a_first_task.ipynb
git commit -m "feat(a2a): add setup cell to notebook 03"
```

---

## Task 4: Section 1 — JSON-RPC 2.0 in 60 seconds

**Files:**
- Modify: `03_a2a_first_task.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Markdown cell**

```markdown
## 2. JSON-RPC 2.0 in 60 seconds

A2A's primary transport is **JSON-RPC 2.0** — a tiny, language-agnostic protocol for calling remote methods. Every request is a JSON object with four fields:

| Field | Type | Notes |
|---|---|---|
| `jsonrpc` | `"2.0"` | Always the literal string `"2.0"`. |
| `id` | string or int | The caller picks any unique value; the server echoes it back. |
| `method` | string | The method being called, e.g. `"message/send"`. |
| `params` | object | Method-specific arguments. |

Every response is also a JSON object with `jsonrpc`, `id` (echoing the request's), and **either** `result` (on success) **or** `error` (on failure) — never both.

JSON-RPC errors use a small set of standard codes:

| Code | Meaning |
|---|---|
| `-32700` | Parse error (malformed JSON) |
| `-32600` | Invalid request (envelope shape wrong) |
| `-32601` | Method not found |
| `-32602` | Invalid params |
| `-32603` | Internal server error |

A2A layers its own meaning on top of `result` (always a `Task` or related A2A object) but the envelope is plain JSON-RPC 2.0 you'd recognize from any other RPC system.

**Key point for HTTP-minded readers:** if an A2A call fails, the response is typically **HTTP 200** with a JSON-RPC `error` field in the body, not HTTP 4xx/5xx. The HTTP status describes the transport; JSON-RPC describes the call.
```

- [ ] **Step 2: Verify the table renders**

- [ ] **Step 3: Commit**

```bash
git add 03_a2a_first_task.ipynb
git commit -m "feat(a2a): introduce JSON-RPC 2.0 in notebook 03"
```

---

## Task 5: Section 2 — A2A message types

**Files:**
- Modify: `03_a2a_first_task.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Markdown cell**

```markdown
## 3. A2A Message Types

Here are the pydantic models we'll use on both sides of the conversation. They mirror the A2A v0.3.0 spec for the subset of types we need today (text-only messages, simple artifacts). Notebook 04 will extend them.

Key types:

- **`TextPart`** — the simplest content part. `{kind: "text", text: "...", metadata?}`.
- **`Message`** — what you send to and receive from an agent. Has `messageId`, `role` ("user" or "agent"), `parts[]`, and the literal `kind: "message"`.
- **`TaskStatus`** — the current state of a task. Includes a `state` (one of the values in the `TaskState` literal below), a timestamp, and optionally a `Message` with the agent's commentary.
- **`Artifact`** — a piece of structured output the agent produced. Has its own `artifactId` and a list of `parts`.
- **`Task`** — what a `message/send` call returns. Has an `id`, a `contextId` (server-assigned), a `status`, and zero or more `artifacts` and `history` messages.
- **`JSONRPCRequest` / `JSONRPCResponse` / `JSONRPCError`** — the generic envelope, A2A-agnostic.
```

- [ ] **Step 2: Models code cell**

```python
from pydantic import model_validator

# A2A v0.3.0 valid task states.
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


# Generic JSON-RPC 2.0 envelope (not A2A-specific).
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

- [ ] **Step 3: Run the cell and verify**

Expected output: `Models defined.`

- [ ] **Step 4: Commit**

```bash
git add 03_a2a_first_task.ipynb
git commit -m "feat(a2a): define A2A v0.3.0 message models in notebook 03"
```

---

## Task 6: Section 3 — The server's `message/send` handler

**Files:**
- Modify: `03_a2a_first_task.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Markdown cell**

```markdown
## 4. The Server's `message/send` Handler

A2A method dispatch goes through a **single HTTP endpoint** per agent — usually `POST /` (whatever the Agent Card's `url` field points to). The handler routes on the JSON-RPC `method` string, NOT on the URL path.

Our researcher will:

1. Accept any JSON-RPC envelope at `POST /`.
2. If `method == "message/send"`, treat `params.message` as a request, extract the topic from the first text part, look it up in our canned data, and return a `Task` with state `completed` and an `Artifact` containing the facts.
3. If the topic isn't recognized, return a `Task` with state `failed` and an explanation message.
4. For any other method, return a JSON-RPC `error` with code `-32601` ("Method not found").
```

- [ ] **Step 2: Server app code cell**

```python
FACTS_BY_TOPIC = {
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


def _completed_task_for(topic: str, facts: list[str]) -> Task:
    return Task(
        id=str(uuid.uuid4()),
        contextId=str(uuid.uuid4()),
        status=TaskStatus(state="completed", timestamp=now_iso()),
        artifacts=[
            Artifact(
                artifactId=str(uuid.uuid4()),
                name=f"facts-about-{topic}",
                parts=[TextPart(text=fact) for fact in facts],
            ),
        ],
    )


def _failed_task_for(topic: str) -> Task:
    return Task(
        id=str(uuid.uuid4()),
        contextId=str(uuid.uuid4()),
        status=TaskStatus(
            state="failed",
            timestamp=now_iso(),
            message=Message(
                messageId=str(uuid.uuid4()),
                role="agent",
                parts=[TextPart(text=f"No facts on file for {topic!r}.")],
            ),
        ),
    )


@researcher_app.post("/", response_model_exclude_none=True)
def handle_jsonrpc(req: JSONRPCRequest) -> JSONRPCResponse:
    if req.method != "message/send":
        return JSONRPCResponse(
            id=req.id,
            error=JSONRPCError(
                code=-32601,
                message=f"Method not found: {req.method!r}",
            ),
        )

    try:
        message = Message.model_validate((req.params or {}).get("message"))
    except ValidationError as e:
        return JSONRPCResponse(
            id=req.id,
            error=JSONRPCError(code=-32602, message=f"Invalid params: {e}"),
        )

    topic = message.parts[0].text.strip().lower()
    facts = FACTS_BY_TOPIC.get(topic)
    task = _completed_task_for(topic, facts) if facts else _failed_task_for(topic)
    return JSONRPCResponse(id=req.id, result=task.model_dump(exclude_none=True))


researcher_server = run_server_in_thread(researcher_app, port=8010)
print("Researcher running on http://127.0.0.1:8010 (A2A endpoint at POST /)")
```

- [ ] **Step 3: Run the cell and verify**

Expected output: `Researcher running on http://127.0.0.1:8010 (A2A endpoint at POST /)`

- [ ] **Step 4: Commit**

```bash
git add 03_a2a_first_task.ipynb
git commit -m "feat(a2a): implement message/send handler in notebook 03"
```

---

## Task 7: Section 4 — The client's first request

**Files:**
- Modify: `03_a2a_first_task.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Markdown cell**

```markdown
## 5. The Client's First Request

We'll build the JSON-RPC envelope as a plain Python dict (no SDK) and POST it. This is exactly what an A2A client looks like at the protocol level — every higher-level SDK is just sugar over this.

The request asks: *"please research 'octopuses'."*
```

- [ ] **Step 2: Client request code cell**

```python
def send_message(base_url: str, user_text: str) -> JSONRPCResponse:
    envelope = {
        "jsonrpc": "2.0",
        "id": str(uuid.uuid4()),
        "method": "message/send",
        "params": {
            "message": {
                "messageId": str(uuid.uuid4()),
                "role": "user",
                "parts": [{"kind": "text", "text": user_text}],
                "kind": "message",
            },
        },
    }
    http_resp = httpx.post(base_url, json=envelope, timeout=10.0)
    http_resp.raise_for_status()
    return JSONRPCResponse.model_validate(http_resp.json())


rpc_resp = send_message("http://127.0.0.1:8010/", "octopuses")

if rpc_resp.error:
    print(f"ERROR {rpc_resp.error.code}: {rpc_resp.error.message}")
else:
    task = Task.model_validate(rpc_resp.result)
    print(f"Task {task.id}")
    print(f"  state:     {task.status.state}")
    print(f"  context:   {task.contextId}")
    print(f"  artifacts: {len(task.artifacts)}")
    for artifact in task.artifacts:
        print(f"    • {artifact.name}")
        for part in artifact.parts:
            print(f"        - {part.text}")
```

- [ ] **Step 3: Run the cell and verify**

Expected output (UUIDs will differ):

```
Task <uuid>
  state:     completed
  context:   <uuid>
  artifacts: 1
    • facts-about-octopuses
        - Octopuses have three hearts.
        - They can change color in under a second.
        - Each of their arms has its own neural cluster.
```

- [ ] **Step 4: Commit**

```bash
git add 03_a2a_first_task.ipynb
git commit -m "feat(a2a): send first message/send request from client in notebook 03"
```

---

## Task 8: Section 5 — Reading the wire

**Files:**
- Modify: `03_a2a_first_task.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Markdown cell**

```markdown
## 6. Reading the Wire

Let's print the raw bytes both sides exchanged. The point is to make the abstractions concrete — every A2A request and response is just JSON like this, and you can hand-write or hand-debug them any time.

Notice in the response: HTTP status is **200**, even though the JSON-RPC `id` is echoed and the result is a fully-formed `Task`. The HTTP layer says nothing about whether the call succeeded — JSON-RPC's `result` vs `error` does.
```

- [ ] **Step 2: Wire-inspection code cell**

```python
def show_wire(base_url: str, user_text: str) -> None:
    envelope = {
        "jsonrpc": "2.0",
        "id": str(uuid.uuid4()),
        "method": "message/send",
        "params": {
            "message": {
                "messageId": str(uuid.uuid4()),
                "role": "user",
                "parts": [{"kind": "text", "text": user_text}],
                "kind": "message",
            },
        },
    }
    print("=" * 60)
    print("REQUEST  (HTTP POST application/json):")
    print("=" * 60)
    print(json.dumps(envelope, indent=2))

    http_resp = httpx.post(base_url, json=envelope, timeout=10.0)

    print()
    print("=" * 60)
    print(f"RESPONSE (HTTP {http_resp.status_code} {http_resp.headers.get('content-type', '')}):")
    print("=" * 60)
    print(json.dumps(http_resp.json(), indent=2))


show_wire("http://127.0.0.1:8010/", "rome")
```

- [ ] **Step 3: Run the cell and verify**

Expected output is a request block and a response block. The response should contain:
- `"jsonrpc": "2.0"`
- a `"result"` object (no `"error"`)
- inside `result`: `"id"`, `"contextId"`, `"status"` (with `"state": "completed"`), `"kind": "task"`, `"artifacts"` containing the three Rome facts.

- [ ] **Step 4: Commit**

```bash
git add 03_a2a_first_task.ipynb
git commit -m "feat(a2a): pretty-print raw wire format in notebook 03"
```

---

## Task 9: Section 6 — Error cases

**Files:**
- Modify: `03_a2a_first_task.ipynb` (add markdown header + 2 code cells)

- [ ] **Step 1: Markdown cell**

```markdown
## 7. Error Cases

Two failure modes show off the protocol's error story.

**Case A: an unknown method.** The server returns HTTP 200 with a JSON-RPC `error` object (code `-32601`, "Method not found"). The `result` field is absent.

**Case B: a known method, but the agent can't do the work.** The server still returns HTTP 200 with a `result`, but the `Task`'s `status.state` is `failed` instead of `completed`, and the embedded `status.message` carries an explanation. The protocol succeeded; the *task* failed.
```

- [ ] **Step 2: Unknown-method code cell**

```python
unknown_envelope = {
    "jsonrpc": "2.0",
    "id": "test-unknown",
    "method": "tasks/list",  # not implemented
    "params": {},
}

resp_a = httpx.post("http://127.0.0.1:8010/", json=unknown_envelope)
print(f"HTTP {resp_a.status_code}")
print(json.dumps(resp_a.json(), indent=2))
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
HTTP 200
{
  "jsonrpc": "2.0",
  "id": "test-unknown",
  "error": {
    "code": -32601,
    "message": "Method not found: 'tasks/list'"
  }
}
```

(`result` is absent in the response, as it should be — JSON-RPC: never both.)

- [ ] **Step 4: Topic-not-found code cell**

```python
unknown_topic_resp = send_message("http://127.0.0.1:8010/", "quantum llamas")

if unknown_topic_resp.error:
    print(f"JSON-RPC ERROR {unknown_topic_resp.error.code}: {unknown_topic_resp.error.message}")
else:
    task = Task.model_validate(unknown_topic_resp.result)
    print(f"Task {task.id}")
    print(f"  state: {task.status.state}")
    if task.status.message:
        for part in task.status.message.parts:
            print(f"  agent says: {part.text}")
```

- [ ] **Step 5: Run the cell and verify**

Expected output:

```
Task <uuid>
  state: failed
  agent says: No facts on file for 'quantum llamas'.
```

(The protocol-level call succeeded — no JSON-RPC error. The task itself entered the `failed` state.)

- [ ] **Step 6: Commit**

```bash
git add 03_a2a_first_task.ipynb
git commit -m "feat(a2a): demonstrate error cases in notebook 03"
```

---

## Task 10: Closing recap, teaser, and cleanup

**Files:**
- Modify: `03_a2a_first_task.ipynb` (add 2 markdown cells + 1 code cell)

- [ ] **Step 1: "What you just learned" markdown cell**

```markdown
## What you just learned

- The JSON-RPC 2.0 envelope (`jsonrpc`, `id`, `method`, `params`) and its `result`/`error` response halves.
- A2A's `message/send` method — what its `params.message` looks like and what its `result` (a `Task`) looks like.
- The valid `TaskState` values and the meaning of `completed` vs `failed`.
- Two distinct failure modes: protocol-level errors (JSON-RPC `error` object, e.g. unknown method) vs domain-level failures (task with `state: "failed"`).
- That HTTP status is almost always `200` regardless of outcome — A2A talks about success and failure through JSON-RPC, not HTTP.
```

- [ ] **Step 2: "What's missing" markdown cell**

```markdown
## What's missing

Every `Task` we got back was already in a terminal state — either `completed` or `failed`. The agent did its work synchronously inside `handle_jsonrpc` and we waited for the answer. That's fine for fast lookups, but real agents do work that takes seconds or minutes, and they need a way to:

- Hand a partial answer back: *"I need clarification before I can finish."* → state `input-required`.
- Tell us they're still chewing on it: *"I started — check back in a minute."* → state `working`.
- Cancel a long-running task.

In **notebook 04** we explore the full task lifecycle: states, polling via `tasks/get`, multi-turn conversations driven by `input-required`, and `tasks/cancel`. We'll also feel the pain of polling — which sets up notebook 05's streaming.
```

- [ ] **Step 3: Cleanup code cell**

```python
shutdown_all_servers()
print("All servers stopped.")
```

- [ ] **Step 4: Run the cleanup cell and verify**

Expected output: `All servers stopped.`

- [ ] **Step 5: Commit**

```bash
git add 03_a2a_first_task.ipynb
git commit -m "feat(a2a): add closing recap and cleanup to notebook 03"
```

---

## Task 11: End-to-end verification in a fresh kernel

**Files:** none (verification only).

- [ ] **Step 1: Execute the notebook top-to-bottom in a clean kernel**

```bash
/opt/anaconda3/bin/jupyter nbconvert --to notebook --execute --inplace 03_a2a_first_task.ipynb
```

(If bare `jupyter` works in your environment, use that.)

Expected: succeeds with no errors. The notebook is rewritten with embedded outputs from the fresh run.

- [ ] **Step 2: Verify expected outputs in the executed notebook**

In order:

1. `Setup OK`
2. `Models defined.`
3. `Researcher running on http://127.0.0.1:8010 (A2A endpoint at POST /)`
4. The "Task ... state: completed ... artifacts: 1 ..." block with all three octopus facts.
5. The REQUEST/RESPONSE wire dump for the Rome request, response containing the three Rome facts.
6. `HTTP 200` + JSON-RPC error block for the unknown method (`-32601`, no `result`).
7. The failed task block for "quantum llamas" with `state: failed` and the agent's message.
8. `All servers stopped.`

No cell raises an unhandled exception. If a port-collision error appears, free port 8010 manually (only kill processes you recognize).

- [ ] **Step 3: Verify port is free after cleanup**

```bash
lsof -iTCP:8010 -sTCP:LISTEN
```

Expected: no output.

- [ ] **Step 4: Commit the clean run**

```bash
git add 03_a2a_first_task.ipynb
git commit -m "chore(a2a): commit clean fresh-kernel run of notebook 03"
```

---

## Done

After Task 11 passes, notebook 03 is complete. The next plan to write is for `04_a2a_task_lifecycle.ipynb`, which introduces task states, `tasks/get`, multi-turn `input-required`, and `tasks/cancel`.
