# A2A Notebook 02 — `02_a2a_agent_card.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the second notebook in the A2A learning series — introduce the **Agent Card** primitive served at `/.well-known/agent.json`, the entry point for everything else in A2A. By the end the learner can build a card on the server side, fetch and parse it on the client side, and understand how this differs from (and complements) OpenAPI.

**Architecture:** A single Jupyter notebook that defines one FastAPI app (the researcher from notebook 01, now with an Agent Card endpoint). The researcher exposes `/.well-known/agent.json` returning a typed `AgentCard` object. The notebook then plays the role of an A2A *client*: `httpx.get` the card, parse it into the same pydantic model, iterate over its declared `skills[]`, and finally call FastAPI's `/openapi.json` for side-by-side contrast.

**Tech Stack:** Python 3.11+, Jupyter, FastAPI, uvicorn, httpx, pydantic v2, threading. Same stack as notebook 01.

**Companion spec:** `docs/superpowers/specs/2026-05-17-a2a-protocol-learning-series-design.md` (notebook 02 section).

**Port assignment:** Researcher runs on `127.0.0.1:8010` (same port substitution we used in notebook 01 to avoid the user's `myapi:app` dev server on 8000).

---

## File Structure

- **Create:** `02_a2a_agent_card.ipynb` — the entire notebook, self-contained (no imports from notebook 01).
- **Modify:** none.
- **No separate test files** — verification is the notebook running top-to-bottom in a fresh kernel (Task 10).

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown)
2. **"What you'll learn"** (markdown bullets)
3. **Setup: imports and server helpers** (code — same pattern as notebook 01)
4. **Section 1: What is an Agent Card?** (markdown only — concept introduction)
5. **Section 2: Serve the researcher's Agent Card** (markdown + 2 code cells: pydantic models for the card + FastAPI app that serves it)
6. **Section 3: Discover the agent from the client** (markdown + 1 code cell: `httpx.get` the card, parse it)
7. **Section 4: Inspect the agent's skills programmatically** (markdown + 1 code cell: loop over `card.skills`, print each skill's id / description / tags / examples)
8. **Section 5: Agent Card vs. OpenAPI** (markdown + 1 code cell that fetches `/openapi.json` and contrasts the two)
9. **"What you just learned"** (markdown)
10. **"What's missing"** (markdown — teases notebook 03, the first `message/send` task)
11. **Cleanup** (code: shut down the server)

---

## Task 1: Create the notebook scaffold

**Files:**
- Create: `02_a2a_agent_card.ipynb`

- [ ] **Step 1: Create an empty notebook with the Python 3 kernel**

Write the file directly (simpler than `jupyter nbconvert --to notebook`):

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

- [ ] **Step 2: Verify the file is valid JSON and opens as a notebook**

```bash
python -c "import json; json.load(open('02_a2a_agent_card.ipynb'))"
ls -la 02_a2a_agent_card.ipynb
```

Expected: no exception, file non-zero size.

- [ ] **Step 3: Commit**

```bash
git add 02_a2a_agent_card.ipynb
git commit -m "feat(a2a): scaffold 02_a2a_agent_card.ipynb"
```

---

## Task 2: Add title and "Why this notebook exists"

**Files:**
- Modify: `02_a2a_agent_card.ipynb` (add cells 1–2)

- [ ] **Step 1: Add a markdown cell with title and motivation**

Cell content (markdown):

```markdown
# 02 — The Agent Card: How Agents Describe Themselves

## Why this notebook exists

In **notebook 01** we built two services that talked to each other through hand-rolled REST. We watched that fall over the moment one side changed its contract — because there was no machine-readable description of either service. Consumers had no way to ask: *"what skills do you have, what inputs do you take, and how do I authenticate?"*

A2A solves this with a single, simple primitive: the **Agent Card**. It's a JSON document served at a well-known URL (`/.well-known/agent.json`) that describes everything a client needs to know to start talking to an agent.

This notebook builds an Agent Card for the researcher from notebook 01, fetches it from a client, and walks through what the card actually says.
```

- [ ] **Step 2: Add a markdown cell with "What you'll learn"**

Cell content (markdown):

```markdown
## What you'll learn

- The shape of an A2A **Agent Card**: name, description, URL, capabilities, authentication, skills.
- How to serve `/.well-known/agent.json` from a FastAPI app.
- How to discover an agent's capabilities from the client side using `httpx`.
- How to parse a card into a typed `pydantic` model and iterate over its declared skills.
- Why the Agent Card is **not** the same as OpenAPI, and what each is good for.
```

- [ ] **Step 3: Verify cells render**

Open the notebook in Jupyter/VS Code and confirm both cells render as markdown.

- [ ] **Step 4: Commit**

```bash
git add 02_a2a_agent_card.ipynb
git commit -m "feat(a2a): add intro markdown to notebook 02"
```

---

## Task 3: Add the setup cell

**Files:**
- Modify: `02_a2a_agent_card.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 1. Setup

Same pattern as notebook 01 — we run a FastAPI app on a background thread so we can both serve and call it from this notebook. Each notebook in this series is self-contained, so the helper is re-defined here rather than imported.
```

- [ ] **Step 2: Add the setup code cell**

```python
import json
import threading
import time

import httpx
import uvicorn
from fastapi import FastAPI
from pydantic import BaseModel, Field

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


print("Setup OK")
```

- [ ] **Step 3: Run the cell**

Expected output: `Setup OK`. If imports fail:

```bash
pip install fastapi "uvicorn[standard]" httpx pydantic
```

- [ ] **Step 4: Commit**

```bash
git add 02_a2a_agent_card.ipynb
git commit -m "feat(a2a): add setup cell to notebook 02"
```

---

## Task 4: Section 1 — What is an Agent Card?

**Files:**
- Modify: `02_a2a_agent_card.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 2. What is an Agent Card?

An Agent Card is a JSON document an agent **publishes about itself**. The A2A spec mandates it lives at the path `/.well-known/agent.json` on the agent's base URL — the same `/.well-known/` convention used by OAuth, OpenID Connect, Web App Manifests, and other web standards.

A minimal card answers four questions:

| Question | Card field |
|---|---|
| *Who are you?* | `name`, `description`, `version` |
| *Where do I send requests?* | `url` |
| *What can you do?* | `capabilities`, `skills[]`, `defaultInputModes`, `defaultOutputModes` |
| *How do I authenticate?* | `authentication.schemes[]` |

The card is **machine-readable** — a client fetches it once at the start of an interaction and can then make sensible decisions: "this agent supports streaming, so I'll use `message/stream` instead of `message/send`," or "this agent requires a bearer token, so I'll attach one."

The card is **also human-readable** — `description` and `skills[].description` are free-form prose meant to help humans (and LLMs reading the card on their behalf) understand what the agent is for.
```

- [ ] **Step 2: Verify the cell renders**

The table should render properly.

- [ ] **Step 3: Commit**

```bash
git add 02_a2a_agent_card.ipynb
git commit -m "feat(a2a): explain Agent Card concept in notebook 02"
```

---

## Task 5: Section 2 — Serve the researcher's Agent Card

**Files:**
- Modify: `02_a2a_agent_card.ipynb` (add markdown header + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 3. Serve the Researcher's Agent Card

Now we build the server side. We'll:

1. Define `pydantic` models matching the A2A card schema (the shapes are stable enough to type).
2. Build a FastAPI app with one endpoint: `GET /.well-known/agent.json`.
3. Start it on `127.0.0.1:8010`.

The card declares one skill — `research_topic` — corresponding to the researcher's only ability. It also declares **`authentication.schemes: ["none"]`** because we haven't introduced auth yet (notebook 07 will).
```

- [ ] **Step 2: Add the models + app code cell**

```python
class AgentCapabilities(BaseModel):
    streaming: bool = False
    pushNotifications: bool = False
    stateTransitionHistory: bool = False


class AgentAuthentication(BaseModel):
    schemes: list[str] = Field(default_factory=lambda: ["none"])


class AgentSkill(BaseModel):
    id: str
    name: str
    description: str
    tags: list[str] = Field(default_factory=list)
    examples: list[str] = Field(default_factory=list)


class AgentCard(BaseModel):
    name: str
    description: str
    url: str
    version: str
    capabilities: AgentCapabilities = Field(default_factory=AgentCapabilities)
    authentication: AgentAuthentication = Field(default_factory=AgentAuthentication)
    defaultInputModes: list[str] = Field(default_factory=lambda: ["text"])
    defaultOutputModes: list[str] = Field(default_factory=lambda: ["text"])
    skills: list[AgentSkill] = Field(default_factory=list)


researcher_app = FastAPI()


RESEARCHER_CARD = AgentCard(
    name="Researcher",
    description="Returns canned facts on a small set of well-known topics.",
    url="http://127.0.0.1:8010",
    version="0.1.0",
    skills=[
        AgentSkill(
            id="research_topic",
            name="Research a topic",
            description="Given a topic name, return a list of facts about it.",
            tags=["research", "facts"],
            examples=["octopuses", "rome"],
        ),
    ],
)


@researcher_app.get("/.well-known/agent.json", response_model=AgentCard)
def agent_card() -> AgentCard:
    return RESEARCHER_CARD


researcher_server = run_server_in_thread(researcher_app, port=8010)
print("Researcher running on http://127.0.0.1:8010")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Researcher running on http://127.0.0.1:8010
```

- [ ] **Step 4: Add a verification cell**

```python
resp = httpx.get("http://127.0.0.1:8010/.well-known/agent.json")
print(resp.status_code)
print(json.dumps(resp.json(), indent=2))
```

- [ ] **Step 5: Run the verification cell**

Expected output:

```
200
{
  "name": "Researcher",
  "description": "Returns canned facts on a small set of well-known topics.",
  "url": "http://127.0.0.1:8010",
  "version": "0.1.0",
  "capabilities": {
    "streaming": false,
    "pushNotifications": false,
    "stateTransitionHistory": false
  },
  "authentication": {
    "schemes": [
      "none"
    ]
  },
  "defaultInputModes": [
    "text"
  ],
  "defaultOutputModes": [
    "text"
  ],
  "skills": [
    {
      "id": "research_topic",
      "name": "Research a topic",
      "description": "Given a topic name, return a list of facts about it.",
      "tags": [
        "research",
        "facts"
      ],
      "examples": [
        "octopuses",
        "rome"
      ]
    }
  ]
}
```

- [ ] **Step 6: Commit**

```bash
git add 02_a2a_agent_card.ipynb
git commit -m "feat(a2a): serve researcher Agent Card in notebook 02"
```

---

## Task 6: Section 3 — Discover the agent from the client side

**Files:**
- Modify: `02_a2a_agent_card.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 4. Discover the Agent from a Client

A client that wants to talk to an agent does one thing first: **fetch the Agent Card**. With nothing more than the agent's base URL, it can learn the agent's identity, capabilities, and skills.

We'll parse the card back into the same `AgentCard` pydantic model we defined on the server side. In a real cross-team setup the client wouldn't have access to the server's classes — it would have its own definitions of the same A2A schema, or use a shared SDK. The shapes are governed by the A2A spec, not by either side's code.
```

- [ ] **Step 2: Add the discovery code cell**

```python
def discover_agent(base_url: str) -> AgentCard:
    """Fetch and parse an agent's card from its base URL."""
    resp = httpx.get(f"{base_url.rstrip('/')}/.well-known/agent.json")
    resp.raise_for_status()
    return AgentCard.model_validate(resp.json())


card = discover_agent("http://127.0.0.1:8010")
print(f"Found agent: {card.name} (v{card.version})")
print(f"  Description: {card.description}")
print(f"  Endpoint:    {card.url}")
print(f"  Auth:        {card.authentication.schemes}")
print(f"  Streaming?   {card.capabilities.streaming}")
print(f"  # skills:    {len(card.skills)}")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Found agent: Researcher (v0.1.0)
  Description: Returns canned facts on a small set of well-known topics.
  Endpoint:    http://127.0.0.1:8010
  Auth:        ['none']
  Streaming?   False
  # skills:    1
```

- [ ] **Step 4: Commit**

```bash
git add 02_a2a_agent_card.ipynb
git commit -m "feat(a2a): add client-side agent discovery to notebook 02"
```

---

## Task 7: Section 4 — Inspect skills programmatically

**Files:**
- Modify: `02_a2a_agent_card.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 5. Inspect the Agent's Skills

The card's `skills[]` field is the menu. Each entry tells a client (or an LLM driving a client) what the agent can do. Skills are intentionally lightweight in A2A: an `id`, a human-friendly `name` and `description`, tags for routing/categorization, and example invocations.

Notice what's **not** here: full JSON schemas for inputs and outputs. The A2A spec leaves the shape of a skill's input to the **message contents** at call time — A2A messages carry parts (text, files, structured data) rather than enforcing per-skill argument schemas the way an RPC API would. This is a deliberate design choice: agents handle free-form conversational input, not tightly-typed function calls. (If you've worked through the MCP series in this repo, contrast this with MCP tools, which **do** declare JSON Schemas for their inputs.)
```

- [ ] **Step 2: Add the inspection code cell**

```python
for skill in card.skills:
    print(f"• {skill.id}: {skill.name}")
    print(f"    {skill.description}")
    print(f"    tags     = {skill.tags}")
    print(f"    examples = {skill.examples}")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
• research_topic: Research a topic
    Given a topic name, return a list of facts about it.
    tags     = ['research', 'facts']
    examples = ['octopuses', 'rome']
```

- [ ] **Step 4: Commit**

```bash
git add 02_a2a_agent_card.ipynb
git commit -m "feat(a2a): inspect skills programmatically in notebook 02"
```

---

## Task 8: Section 5 — Agent Card vs. OpenAPI

**Files:**
- Modify: `02_a2a_agent_card.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 6. Agent Card vs. OpenAPI

FastAPI gives us OpenAPI for free at `/openapi.json`. That document also describes our service in a machine-readable way. So — why does A2A need its own format?

The short answer: **they describe different things at different abstraction levels.**

| | OpenAPI | A2A Agent Card |
|---|---|---|
| **Describes** | HTTP routes, methods, request/response shapes | An agent's identity, capabilities, skills |
| **Granularity** | Per-endpoint | Per-agent |
| **Input model** | Typed per-route arguments | Free-form messages (text, files, data) |
| **Auth model** | OpenAPI security schemes | A2A authentication schemes (subset by design) |
| **Audience** | Code generators, HTTP clients | Other agents (and the LLMs/runtimes driving them) |
| **Discoverable at** | `/openapi.json` | `/.well-known/agent.json` |

OpenAPI is great if you're writing a typed HTTP client. A2A is what an LLM-driven coordinator wants when it's asking *"is there an agent here, and what is it good for?"* In practice both can coexist on the same FastAPI app — and they do, right now, on this notebook's researcher.
```

- [ ] **Step 2: Add the OpenAPI fetch cell**

```python
openapi = httpx.get("http://127.0.0.1:8010/openapi.json").json()

print("OpenAPI describes routes:")
for path, methods in openapi["paths"].items():
    for method in methods:
        print(f"  {method.upper():6} {path}")

print()
print(f"Agent Card describes one agent named {card.name!r}")
print(f"with {len(card.skills)} skill(s): {[s.id for s in card.skills]}")
```

- [ ] **Step 3: Run the cell and verify**

Expected output (paths order may differ):

```
OpenAPI describes routes:
  GET    /.well-known/agent.json

Agent Card describes one agent named 'Researcher'
with 1 skill(s): ['research_topic']
```

- [ ] **Step 4: Commit**

```bash
git add 02_a2a_agent_card.ipynb
git commit -m "feat(a2a): contrast Agent Card with OpenAPI in notebook 02"
```

---

## Task 9: Closing recap, teaser, and cleanup

**Files:**
- Modify: `02_a2a_agent_card.ipynb` (add 2 markdown cells + 1 code cell)

- [ ] **Step 1: Add the "What you just learned" markdown cell**

```markdown
## What you just learned

- The Agent Card is the entry point of A2A — a single JSON document at `/.well-known/agent.json` that describes an agent's identity, capabilities, authentication, and skills.
- The card is typed: a `pydantic` model on both sides keeps the shape stable.
- A client only needs the agent's base URL to discover everything else.
- Agent Card and OpenAPI describe **different things** at different abstraction levels — and they coexist happily.
```

- [ ] **Step 2: Add the "What's missing" markdown cell**

```markdown
## What's missing

We have an Agent Card. We know what the agent **claims** to do. But we haven't actually **called** the agent yet — there's no way for a client to say *"please research octopuses for me"* and get a result back.

In **notebook 03** we introduce the first real A2A protocol exchange: `message/send`. We'll wrap our researcher's logic behind a JSON-RPC 2.0 endpoint, hand-build the request envelope, and watch the agent return a structured `Task` with an `Artifact` containing the answer.
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

- [ ] **Step 5: Commit**

```bash
git add 02_a2a_agent_card.ipynb
git commit -m "feat(a2a): add closing recap and cleanup to notebook 02"
```

---

## Task 10: End-to-end verification in a fresh kernel

**Files:** none (verification only).

- [ ] **Step 1: Execute the notebook top-to-bottom in a clean kernel**

```bash
jupyter nbconvert --to notebook --execute --inplace 02_a2a_agent_card.ipynb
```

Expected: succeeds with no errors. The notebook is rewritten with embedded outputs from the fresh run.

- [ ] **Step 2: Verify expected outputs in the executed notebook**

Cells should contain, in order:

1. `Setup OK`
2. `Researcher running on http://127.0.0.1:8010`
3. Pretty-printed Agent Card JSON (200 status, full card body).
4. The five-line `Found agent: Researcher ...` summary.
5. The bullet list for the `research_topic` skill.
6. The OpenAPI route listing + Agent Card summary.
7. `All servers stopped.`

No cell raises an unhandled exception. If a port-collision error appears, free port 8010 first: `lsof -ti tcp:8010 | xargs kill -9` (only kill processes you recognize).

- [ ] **Step 3: Verify port is free after cleanup**

```bash
lsof -iTCP:8010 -sTCP:LISTEN
```

Expected: no output.

- [ ] **Step 4: Commit the clean run**

```bash
git add 02_a2a_agent_card.ipynb
git commit -m "chore(a2a): commit clean fresh-kernel run of notebook 02"
```

---

## Done

After Task 10 passes, notebook 02 is complete. The next plan to write is for `03_a2a_first_task.ipynb`, which introduces JSON-RPC 2.0 and the `message/send` exchange.
