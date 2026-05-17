# A2A Protocol Learning Series — Design

**Date:** 2026-05-17
**Author:** Nimesh Patel (with Claude)
**Status:** Approved design, pending implementation plan

## Purpose

Teach the **Agent2Agent (A2A) protocol** from first principles through a runnable, progressive 8-notebook series. The series mirrors the pedagogical structure of the existing MCP notebook series (`01_..._.ipynb` through `08_..._.ipynb`) in this same directory.

The learner should finish able to:

1. Explain why a standard agent-to-agent protocol exists and what problem it solves.
2. Read and produce a valid Agent Card.
3. Implement an A2A server that handles task send, retrieval, cancellation, streaming, and push notifications.
4. Implement an A2A client that discovers an agent and exchanges typed messages and artifacts with it.
5. Add bearer-token and OAuth2 authentication on top of any A2A endpoint.
6. Compose multiple A2A agents into a coordinated multi-agent system using the official Python SDK.

## Non-goals

The following are **explicitly out of scope** for this series:

- Production deployment concerns (TLS, load balancing, observability, infra).
- Comparison between A2A and MCP. (Possible follow-on notebook later.)
- LangGraph or CrewAI integration. (Possible follow-on notebook later.)
- A2A in languages other than Python.
- Building a real identity provider for OAuth2 — the IdP is mocked.

## Audience and prerequisites

The learner already understands:

- Python (async/await, typing, pydantic).
- HTTP basics (methods, status codes, headers).
- The general "agent" concept (they have built the MCP series and LangGraph notebooks in the same directory).

The learner does **not** need prior knowledge of:

- A2A or any agent-to-agent protocol.
- JSON-RPC 2.0 (introduced in notebook 03).
- Server-Sent Events (introduced in notebook 05).

## Pedagogical arc

The series follows the same shape as the MCP series:

1. **Show the pain** (notebook 01) — what life looks like without the protocol.
2. **Introduce the standard** (notebook 02) — the minimum viable A2A primitive.
3. **Layer in capabilities** (notebooks 03–07) — task lifecycle, streaming, async, auth.
4. **Tie it together** (notebook 08) — a real multi-agent system using the official SDK.

Every notebook is **runnable end-to-end** in Jupyter with no external infrastructure beyond a Python virtualenv. Each notebook ends with a **"What you just learned"** recap and a **"What's missing"** teaser motivating the next notebook.

## Common stack

| Concern | Choice |
|---|---|
| Python | 3.11+ |
| Notebooks | Jupyter |
| HTTP client | `httpx` |
| HTTP server | `FastAPI` + `uvicorn` |
| Typed models | `pydantic` v2 |
| Official A2A SDK | `a2a-sdk` (introduced only in notebook 08) |
| LLM | Any (Claude or OpenAI), used **only** in notebook 08 |

**Server execution pattern:** each notebook spins up its FastAPI server via a background `uvicorn.Server` running on a thread, bound to a fixed localhost port (`8010` for the primary agent; `8011` for the secondary app in notebook 06's webhook receiver). Ports `8000`/`8001` are avoided because the user runs an unrelated dev server there. A `shutdown()` cell at the bottom of each notebook stops the server cleanly.

**No API keys for notebooks 01–07.** The "agents" are stub implementations that return canned responses. The focus is the protocol, not LLM reasoning. Only notebook 08 requires a real LLM API key.

## A2A spec version

This series targets the **current published A2A specification** at the time of writing (Google's `google/A2A` repo, v0.3.0). Method names and message shapes match that spec (e.g., `message/send`, `tasks/get`, `tasks/cancel`, `message/stream`, `tasks/pushNotificationConfig/set`). The Agent Card lives at `/.well-known/agent-card.json` in v0.3.0 (earlier drafts used `/.well-known/agent.json` without the hyphen). Where the spec is still in flux, the notebook calls this out explicitly so the learner can update for newer versions.

## Notebooks

### 01 — `01_without_a2a.ipynb` — The pain
**Goal:** motivate the existence of A2A.

- Two FastAPI services: a `researcher` and a `writer`, each with a hand-rolled REST API.
- Implement integration code that calls one from the other.
- Change the writer's input format midway through the notebook and watch the integration code break.
- Notebook closes: *"Every team is reinventing this wheel. There should be a standard."*

### 02 — `02_a2a_agent_card.ipynb` — Discovery
**Goal:** introduce the Agent Card primitive.

- Serve `/.well-known/agent-card.json` from a FastAPI endpoint.
- The card includes: `protocolVersion`, `name`, `description`, `url`, `version`, `capabilities`, `defaultInputModes`, `defaultOutputModes`, `skills[]`, `securitySchemes`, `security`.
- Client side: `httpx.get` the card, parse it into a `pydantic` model.
- Compare to OpenAPI: similar idea, narrower scope, JSON Schema for skill inputs/outputs.

### 03 — `03_a2a_first_task.ipynb` — Sync request/response
**Goal:** the first end-to-end protocol exchange.

- Introduce JSON-RPC 2.0 envelope (`jsonrpc`, `id`, `method`, `params`).
- Server implements `message/send`: accepts a `Message` with `role: "user"` and `parts: [{kind: "text", text: "..."}]`; returns a `Task` with state `completed` and an `Artifact`.
- Client builds the JSON-RPC request **by hand**, parses the response **by hand**.
- The learner reads every byte going over the wire.

### 04 — `04_a2a_task_lifecycle.ipynb` — States and multi-turn
**Goal:** the task state machine and asynchronous-but-not-streaming work.

- States: `submitted → working → (input-required | completed | canceled | failed)`.
- Implement `tasks/get` for polling.
- Build a server that returns `input-required` mid-task ("I need clarification..."), accept a follow-up message tied to the same task ID, complete.
- Implement `tasks/cancel`.
- Closes: *"Polling is wasteful. There must be a better way."*

### 05 — `05_a2a_streaming.ipynb` — SSE
**Goal:** real-time incremental updates.

- `message/stream` returns Server-Sent Events.
- Server yields `TaskStatusUpdateEvent` and `TaskArtifactUpdateEvent` chunks.
- Client consumes with `httpx.stream()`.
- Build a "slow agent" that emits incremental progress so the learner sees updates land in real time.

### 06 — `06_a2a_push_notifications.ipynb` — Async / webhooks
**Goal:** long-running tasks without holding open a connection.

- Client registers a webhook via `tasks/pushNotificationConfig/set`.
- Server returns immediately with a `submitted` task.
- Work runs in background (simulated with `asyncio.sleep`); on completion the server POSTs the final task to the client's webhook URL.
- **A second FastAPI app** runs in the same notebook on port 8001 to receive the callback. The learner inspects the signed payload.

### 07 — `07_a2a_auth.ipynb` — Authentication
**Goal:** make any A2A endpoint authenticated.

- The Agent Card's `securitySchemes` (OpenAPI-style scheme definitions) and `security` (required scheme combinations) fields declare supported authentication.
- Cover three modes:
  - **No auth** (the implicit default in 01–06, now made explicit).
  - **Bearer token** — declared in card, sent as `Authorization: Bearer ...`, validated server-side.
  - **OAuth2** — declared with `scopes`, walkthrough of the **client credentials** flow (no user-consent UI needed in a notebook). The IdP is **mocked** (a tiny FastAPI route that issues tokens).
- Cover **signed push notifications under auth**: the server signs the webhook callback with a JWT so the client can verify it.

### 08 — `08_a2a_multi_agent.ipynb` — End-to-end with the official SDK
**Goal:** the payoff. Stop hand-rolling.

- Install `a2a-sdk`.
- Rebuild the researcher + writer from notebook 01, but expose A2A endpoints via the SDK's server helpers.
- Add a **coordinator** (a third Python process in the notebook, acting purely as an A2A *client*): receives a user request, calls the researcher via A2A, forwards the artifact to the writer via A2A, streams the final result back to the user.
- This is the **only** notebook that uses a real LLM (Claude or OpenAI) so the agents have genuine reasoning behind the protocol.
- Final cell: side-by-side diff of the same coordination logic, hand-rolled (50+ lines per call) vs. SDK (~5 lines per call). The "why the protocol matters" payoff.

## File layout

```
01_without_a2a.ipynb
02_a2a_agent_card.ipynb
03_a2a_first_task.ipynb
04_a2a_task_lifecycle.ipynb
05_a2a_streaming.ipynb
06_a2a_push_notifications.ipynb
07_a2a_auth.ipynb
08_a2a_multi_agent.ipynb
```

All notebooks live in the working directory root, alongside the existing MCP series.

## Style conventions

Each notebook opens with:

- A `## Why this notebook exists` markdown cell.
- A `## What you'll learn` bulleted list.

Body sections use numbered headers (`## 1. ...`, `## 2. ...`) with `### Try it` cells containing exercises the learner can run and modify.

Each notebook closes with:

- `## What you just learned` — bulleted recap.
- `## What's missing` — one paragraph motivating the next notebook (omitted in notebook 08, which closes the series).

This matches the existing MCP series style exactly.

## Success criteria

The series is complete when:

1. All 8 notebooks exist in the working directory.
2. Each notebook runs top-to-bottom in a fresh kernel with no errors.
3. Notebooks 01–07 require no external API keys.
4. Notebook 08 runs given a single LLM API key (Claude or OpenAI).
5. Every JSON-RPC request and response shown matches the A2A spec the notebooks claim to target.
6. The closing diff in notebook 08 demonstrably contrasts hand-rolled and SDK code paths for the same operation.

## Open questions deferred to implementation planning

- Exact `a2a-sdk` version pin (latest at time of writing).
- Whether to use `requests-mock`-style assertions in any "verify the wire format" exercises, or just let the learner read JSON.
- Whether notebook 08's LLM uses tool calling or a simple completion loop. (Probably simple — the focus is the protocol, not tool use.)

These do not affect the design and will be decided in the implementation plan.
