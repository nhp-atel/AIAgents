# A2A Notebook 01 — `01_without_a2a.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first notebook in the A2A learning series — a self-contained demonstration of why an agent-to-agent protocol is needed, using two FastAPI services that integrate via custom REST and break when one side's contract changes.

**Architecture:** A single Jupyter notebook that defines two FastAPI apps (`researcher` and `writer`) in separate cells, runs each in a background thread via `uvicorn`, exercises them with `httpx` integration code, then demonstrates the brittleness by changing the writer's input schema and watching the integration fail. No A2A protocol code yet — that's the point.

**Tech Stack:** Python 3.11+, Jupyter, FastAPI, uvicorn, httpx, pydantic v2, threading.

**Companion spec:** `docs/superpowers/specs/2026-05-17-a2a-protocol-learning-series-design.md` (notebook 01 section).

---

## File Structure

- **Create:** `01_without_a2a.ipynb` — the entire notebook (markdown + code cells, no imports from other notebook files).
- **Modify:** none.
- **No separate test files** — verification is the notebook itself running top-to-bottom in a fresh kernel. The "fresh kernel run" is the test (Task 8).

The notebook is intentionally self-contained: a learner clones the repo, opens this notebook, and everything they need is in the notebook itself.

## Notebook Section Map

The finished notebook will have these sections, in order:

1. **Title + "Why this notebook exists"** (markdown)
2. **"What you'll learn"** (markdown bullets)
3. **Setup: imports and helpers** (code)
4. **The researcher service** (markdown + code: FastAPI app, run in thread)
5. **The writer service** (markdown + code: FastAPI app, run in thread)
6. **Integration code** (markdown + code: client that calls both)
7. **The breakage** (markdown + code: change writer's schema, watch integration fail)
8. **"What you just learned"** (markdown)
9. **"What's missing"** (markdown, teasing notebook 02)
10. **Cleanup** (code: shut down the servers)

---

## Task 1: Create the notebook scaffold

**Files:**
- Create: `01_without_a2a.ipynb`

- [ ] **Step 1: Create an empty notebook with the right kernel**

Run from the project root:

```bash
jupyter nbconvert --to notebook --output 01_without_a2a.ipynb /dev/stdin <<'EOF'
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
EOF
```

If `jupyter` isn't available, instead create the file directly with the above JSON content as `01_without_a2a.ipynb`.

- [ ] **Step 2: Verify the file exists and opens**

Run:

```bash
ls -la 01_without_a2a.ipynb
```

Expected: file exists, non-zero size. Open it in Jupyter or VS Code and confirm it shows an empty notebook (no cells) with the Python 3 kernel.

- [ ] **Step 3: Commit**

```bash
git add 01_without_a2a.ipynb
git commit -m "feat(a2a): scaffold 01_without_a2a.ipynb"
```

---

## Task 2: Add title and "Why this notebook exists"

**Files:**
- Modify: `01_without_a2a.ipynb` (add cells 1–2)

- [ ] **Step 1: Add a markdown cell with title and motivation**

Cell content (markdown):

```markdown
# 01 — Without A2A: The Pain of Custom Agent Integration

## Why this notebook exists

Imagine you have two services that need to collaborate:

- A **researcher** that gathers facts on a topic.
- A **writer** that turns those facts into a paragraph.

In a world without a standard agent-to-agent protocol, the only way to connect them is to design a custom REST API for each service, then hand-roll the glue code. That works — until one side changes its contract. Then the glue breaks, and every other consumer of that service breaks with it.

This notebook reproduces that pain in ~100 lines of code. Once you've felt it, the rest of the series will introduce **A2A** — a standard that makes this problem mostly go away.
```

- [ ] **Step 2: Add a markdown cell with "What you'll learn"**

Cell content (markdown):

```markdown
## What you'll learn

- How to stand up two FastAPI services in a notebook and call them with `httpx`.
- Why ad-hoc REST contracts between agent-like services don't scale.
- What a "breaking change" looks like in practice, and why coordination is expensive without a shared protocol.
- The motivation for the rest of this series: **A2A** as a standard for agent-to-agent communication.
```

- [ ] **Step 3: Verify cells render**

Open the notebook in Jupyter, confirm both cells show as rendered markdown (not raw text).

- [ ] **Step 4: Commit**

```bash
git add 01_without_a2a.ipynb
git commit -m "feat(a2a): add intro markdown to notebook 01"
```

---

## Task 3: Add the setup cell (imports and server helpers)

**Files:**
- Modify: `01_without_a2a.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 1. Setup

We'll use:

- **FastAPI** to define each service.
- **uvicorn** running on a background thread so we can both serve and call the API from the same notebook.
- **httpx** as our HTTP client.
- **pydantic** for typed request/response models.

The helper `run_server_in_thread(app, port)` starts a uvicorn server in a daemon thread and returns a handle we can use to shut it down at the end of the notebook.
```

- [ ] **Step 2: Add the setup code cell**

Cell content (code):

```python
import socket
import threading
import time

import httpx
import uvicorn
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

_servers: list[uvicorn.Server] = []


def run_server_in_thread(app: FastAPI, port: int) -> uvicorn.Server:
    """Start `app` on localhost:`port` in a background daemon thread.

    Returns the uvicorn Server object so the caller can stop it later.
    """
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


def _port_is_free(port: int) -> bool:
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        return s.connect_ex(("127.0.0.1", port)) != 0


def stop_server(server: uvicorn.Server, port: int) -> None:
    """Stop a single server and wait for its port to be released."""
    server.should_exit = True
    for _ in range(50):
        if _port_is_free(port):
            break
        time.sleep(0.05)
    else:
        raise RuntimeError(f"Port {port} still bound after stop")
    if server in _servers:
        _servers.remove(server)


def shutdown_all_servers() -> None:
    """Stop every server we started in this notebook."""
    for server in list(_servers):
        server.should_exit = True
    _servers.clear()


print("Setup OK")
```

- [ ] **Step 3: Run the cell and verify output**

In Jupyter, execute the setup cell. Expected output:

```
Setup OK
```

No exceptions, no warnings about already-installed packages.

If imports fail, install dependencies first:

```bash
pip install fastapi "uvicorn[standard]" httpx pydantic
```

- [ ] **Step 4: Commit**

```bash
git add 01_without_a2a.ipynb
git commit -m "feat(a2a): add setup cell to notebook 01"
```

---

## Task 4: Build the researcher service

**Files:**
- Modify: `01_without_a2a.ipynb` (add markdown header + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 2. The Researcher Service

The researcher takes a `topic` and returns a list of "facts" about it. We're not using a real LLM — this is a stub that returns canned facts, because the point of this notebook is the **integration**, not the intelligence.

The researcher's API contract: `POST /research` with `{"topic": str}` → `{"topic": str, "facts": list[str]}`.
```

- [ ] **Step 2: Add the researcher app cell**

```python
researcher_app = FastAPI()


class ResearchRequest(BaseModel):
    topic: str


class ResearchResponse(BaseModel):
    topic: str
    facts: list[str]


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


@researcher_app.post("/research", response_model=ResearchResponse)
def research(req: ResearchRequest) -> ResearchResponse:
    facts = FACTS_BY_TOPIC.get(req.topic.lower())
    if facts is None:
        raise HTTPException(status_code=404, detail=f"No facts on file for {req.topic!r}")
    return ResearchResponse(topic=req.topic, facts=facts)


researcher_server = run_server_in_thread(researcher_app, port=8000)
print("Researcher running on http://127.0.0.1:8000")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Researcher running on http://127.0.0.1:8000
```

- [ ] **Step 4: Add a verification code cell**

```python
resp = httpx.post("http://127.0.0.1:8000/research", json={"topic": "octopuses"})
print(resp.status_code)
print(resp.json())
```

- [ ] **Step 5: Run the verification cell**

Expected output:

```
200
{'topic': 'octopuses', 'facts': ['Octopuses have three hearts.', 'They can change color in under a second.', 'Each of their arms has its own neural cluster.']}
```

- [ ] **Step 6: Commit**

```bash
git add 01_without_a2a.ipynb
git commit -m "feat(a2a): add researcher service to notebook 01"
```

---

## Task 5: Build the writer service

**Files:**
- Modify: `01_without_a2a.ipynb` (add markdown header + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 3. The Writer Service

The writer takes a `topic` and a list of `facts` and returns a one-paragraph summary. Like the researcher, it's a stub — it just joins the facts together with a templated lead-in.

The writer's API contract: `POST /write` with `{"topic": str, "facts": list[str]}` → `{"paragraph": str}`.

Note how this contract was designed **independently** of the researcher's. The researcher returns `{"topic", "facts"}`; the writer expects `{"topic", "facts"}`. The fact that they happen to line up is a coincidence we (the integrator) have to maintain by hand.
```

- [ ] **Step 2: Add the writer app cell**

```python
writer_app = FastAPI()


class WriteRequest(BaseModel):
    topic: str
    facts: list[str]


class WriteResponse(BaseModel):
    paragraph: str


@writer_app.post("/write", response_model=WriteResponse)
def write(req: WriteRequest) -> WriteResponse:
    bullets = " ".join(req.facts)
    paragraph = f"Here is what we know about {req.topic}: {bullets}"
    return WriteResponse(paragraph=paragraph)


writer_server = run_server_in_thread(writer_app, port=8001)
print("Writer running on http://127.0.0.1:8001")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Writer running on http://127.0.0.1:8001
```

- [ ] **Step 4: Add a verification code cell**

```python
resp = httpx.post(
    "http://127.0.0.1:8001/write",
    json={"topic": "octopuses", "facts": ["They have three hearts.", "They can change color."]},
)
print(resp.status_code)
print(resp.json())
```

- [ ] **Step 5: Run the verification cell**

Expected output:

```
200
{'paragraph': 'Here is what we know about octopuses: They have three hearts. They can change color.'}
```

- [ ] **Step 6: Commit**

```bash
git add 01_without_a2a.ipynb
git commit -m "feat(a2a): add writer service to notebook 01"
```

---

## Task 6: Build the integration code

**Files:**
- Modify: `01_without_a2a.ipynb` (add markdown header + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 4. Hand-Rolled Integration

Now the glue: a function `research_and_write(topic)` that calls the researcher, then forwards its output to the writer, then returns the paragraph.

Read this code carefully. Notice:

- We have to **know the URLs** of both services.
- We have to **know the request/response shapes** of both services.
- We have to **manually translate** between them (in this case, the shapes match exactly — but only because we got lucky).
- There is **no machine-readable description** of either service. The only way another engineer learns what's available is by reading our code, the services' source, or whatever docs we happen to write.
```

- [ ] **Step 2: Add the integration code cell**

```python
RESEARCHER_URL = "http://127.0.0.1:8000"
WRITER_URL = "http://127.0.0.1:8001"


def research_and_write(topic: str) -> str:
    research_resp = httpx.post(f"{RESEARCHER_URL}/research", json={"topic": topic})
    research_resp.raise_for_status()
    research_data = research_resp.json()

    write_resp = httpx.post(
        f"{WRITER_URL}/write",
        json={"topic": research_data["topic"], "facts": research_data["facts"]},
    )
    write_resp.raise_for_status()
    write_data = write_resp.json()

    return write_data["paragraph"]


print(research_and_write("octopuses"))
```

- [ ] **Step 3: Run the cell and verify**

Expected output (one line, wrapped here for readability):

```
Here is what we know about octopuses: Octopuses have three hearts. They can change color in under a second. Each of their arms has its own neural cluster.
```

- [ ] **Step 4: Commit**

```bash
git add 01_without_a2a.ipynb
git commit -m "feat(a2a): add hand-rolled integration to notebook 01"
```

---

## Task 7: Demonstrate the breakage

**Files:**
- Modify: `01_without_a2a.ipynb` (add markdown header + 3 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 5. The Breakage

Now imagine the writer team decides their API should be more "RESTful". They rename the request field `facts` to `bullet_points` and ship a new version. They tell their team in Slack. They don't tell us.

Watch what happens.
```

- [ ] **Step 2: Add a code cell that stops the writer and deploys a v2 in its place**

```python
# Simulate the writer team shipping a breaking change:
# stop the old writer, define a new app with a renamed field, start it on the same port.

stop_server(writer_server, port=8001)

writer_app_v2 = FastAPI()


class WriteRequestV2(BaseModel):
    topic: str
    bullet_points: list[str]   # was: facts


@writer_app_v2.post("/write", response_model=WriteResponse)
def write_v2(req: WriteRequestV2) -> WriteResponse:
    bullets = " ".join(req.bullet_points)
    paragraph = f"Here is what we know about {req.topic}: {bullets}"
    return WriteResponse(paragraph=paragraph)


writer_server = run_server_in_thread(writer_app_v2, port=8001)
print("Writer v2 running on http://127.0.0.1:8001 — now expects 'bullet_points' instead of 'facts'")
```

- [ ] **Step 3: Add a code cell that calls the integration**

```python
try:
    print(research_and_write("rome"))
except httpx.HTTPStatusError as e:
    print(f"BROKE: {e.response.status_code}")
    print(e.response.json())
```

- [ ] **Step 4: Run all three cells in order and verify the breakage**

The first cell stops the old writer and reports `Writer v2 running on ... — now expects 'bullet_points' instead of 'facts'`. The second cell calls `research_and_write` and produces:

```
BROKE: 422
{'detail': [{'type': 'missing', 'input': {...}, 'loc': ['body', 'bullet_points'], 'msg': 'Field required'}, {'type': 'extra_forbidden' or similar, 'loc': ['body', 'facts'], ...}]}
```

(The exact `detail` payload may differ slightly between pydantic / FastAPI versions; the important part is the 422 status and the `bullet_points` / `facts` field complaints.)

- [ ] **Step 5: Add a markdown cell reflecting on the breakage**

```markdown
### What just happened?

The writer's HTTP server is still up. The researcher is still up. But the integration code is broken because:

- The writer changed its input schema.
- The integration code had no way to know.
- There was no shared description of the writer's contract that we could check against.
- There was no version negotiation, no capability discovery, no nothing.

Now imagine you're not running two services. Imagine you're running twenty, written by five different teams. Every integration is bespoke. Every breaking change is a multi-team coordination problem.

**This is the problem A2A is designed to solve.**
```

- [ ] **Step 6: Commit**

```bash
git add 01_without_a2a.ipynb
git commit -m "feat(a2a): demonstrate breakage in notebook 01"
```

---

## Task 8: Closing recap, teaser, and cleanup

**Files:**
- Modify: `01_without_a2a.ipynb` (add 2 markdown cells + 1 code cell)

- [ ] **Step 1: Add the "What you just learned" markdown cell**

```markdown
## What you just learned

- How to stand up two FastAPI services side-by-side in a notebook.
- What hand-rolled integration code looks like when there's no shared agent protocol.
- How a one-field rename on one service breaks every consumer that hard-coded the old contract.
- That coordinating breaking changes across multiple agent services is **the** problem to solve.
```

- [ ] **Step 2: Add the "What's missing" markdown cell**

```markdown
## What's missing

We have no machine-readable description of either service. A consumer can't ask "what skills do you have, what inputs do you take, and what authentication do you require?" and get a structured answer.

In **notebook 02**, we introduce the first piece of A2A: the **Agent Card**, served at `/.well-known/agent.json`. The card is a JSON document that answers exactly that question, and it's the entry point for everything else in the protocol.
```

- [ ] **Step 3: Add the cleanup code cell**

```python
shutdown_all_servers()
print("All servers stopped.")
```

- [ ] **Step 4: Run the cleanup cell and verify**

Expected output:

```
All servers stopped.
```

After running this, `httpx.post("http://127.0.0.1:8000/research", json={"topic": "octopuses"})` should now raise a `ConnectError` because the server is down. (Don't add this assertion to the notebook — just verify manually if you want.)

- [ ] **Step 5: Commit**

```bash
git add 01_without_a2a.ipynb
git commit -m "feat(a2a): add closing recap and cleanup to notebook 01"
```

---

## Task 9: End-to-end verification in a fresh kernel

**Files:** none (verification only).

- [ ] **Step 1: Restart the kernel**

In Jupyter: Kernel → Restart Kernel. (Or in VS Code: click the "Restart" button at the top of the notebook.)

This guarantees the notebook isn't accidentally relying on state from a previous run.

- [ ] **Step 2: Run all cells top to bottom**

In Jupyter: Kernel → Restart Kernel and Run All Cells.

- [ ] **Step 3: Verify the run**

Expected, in order:

1. Setup cell prints `Setup OK`.
2. Researcher cell prints `Researcher running on http://127.0.0.1:8000`.
3. First verification prints status `200` and the octopus facts dict.
4. Writer cell prints `Writer running on http://127.0.0.1:8001`.
5. Second verification prints status `200` and the octopus paragraph.
6. Integration cell prints the full octopus paragraph from the researcher + writer pipeline.
7. Contract-change cell prints `Writer now expects 'bullet_points' instead of 'facts'`.
8. Breakage cell prints `BROKE: 422` followed by the error detail.
9. Cleanup cell prints `All servers stopped.`

No cell raises an unhandled exception. If any cell errors out, fix it (most likely a port conflict — kill any process on ports 8000/8001 with `lsof -ti tcp:8000,8001 | xargs kill -9` and re-run).

- [ ] **Step 4: Verify no leftover servers**

After the run, in a terminal:

```bash
lsof -iTCP:8000 -sTCP:LISTEN
lsof -iTCP:8001 -sTCP:LISTEN
```

Expected: no output (the cleanup cell stopped them).

- [ ] **Step 5: Commit the final notebook outputs**

```bash
git add 01_without_a2a.ipynb
git commit -m "chore(a2a): commit clean fresh-kernel run of notebook 01"
```

---

## Done

After Task 9 passes, notebook 01 is complete. The next plan to write is for `02_a2a_agent_card.ipynb`, which introduces the Agent Card and the `/.well-known/agent.json` endpoint.
