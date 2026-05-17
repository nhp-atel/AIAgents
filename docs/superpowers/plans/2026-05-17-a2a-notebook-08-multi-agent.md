# A2A Notebook 08 — `08_a2a_multi_agent.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the final notebook in the A2A learning series — the **payoff**. Stop hand-rolling JSON-RPC. Install the official **`a2a-sdk`**, rebuild the researcher and writer from notebook 01 using the SDK's server primitives, and write a coordinator (an A2A *client*) that orchestrates them. Close with a side-by-side diff: the same multi-agent coordination expressed as ~150 lines of hand-rolled JSON-RPC versus ~30 lines using the SDK.

**Architecture:** Two SDK-built A2A agents running in this notebook — researcher on `127.0.0.1:8010`, writer on `127.0.0.1:8011`. Each is built with the SDK's `AgentExecutor` pattern: `class XAgentExecutor(AgentExecutor): async def execute(self, ctx, event_queue): ...`. Both are served by Starlette + uvicorn in background threads (same pattern as previous notebooks; uvicorn is happy to host Starlette apps directly). The coordinator is a Python `async` function that uses the SDK's `A2AClient` (or equivalent) to call researcher with a topic, then forward the result to writer. Optional Section 7 shows how to swap a stub agent for a real Claude-powered one (gated on `ANTHROPIC_API_KEY` env var).

**Tech Stack:** Python 3.11+, Jupyter, **`a2a-sdk`** (new dependency, installed in the notebook), Starlette, uvicorn, httpx, threading, asyncio. Optional: `anthropic` SDK for the LLM stretch.

**A2A spec version targeted:** v0.3.0. Installs `a2a-sdk` v0.3.26 specifically (the v0.3-compatible release; the 1.x line targets the A2A 1.0 spec which has wire-format differences).

**Port assignment:** Researcher `8010`, writer `8011`. (8000/8001 still off-limits.)

---

## File Structure

- **Create:** `08_a2a_multi_agent.ipynb` — self-contained.

## Notebook Section Map

1. Title + "Why this notebook exists"
2. "What you'll learn"
3. Setup — install `a2a-sdk`, imports, background-thread helper
4. Section 1: A tour of the SDK (markdown — major classes and what they replace)
5. Section 2: Build the researcher with the SDK
6. Section 3: Build the writer with the SDK
7. Section 4: The coordinator (an A2A client orchestrating both)
8. Section 5: Run the end-to-end flow
9. Section 6: Side-by-side diff — hand-rolled vs SDK (markdown)
10. Section 7: Stretch — swap a stub for Claude (markdown only by default; runs only if API key set)
11. "What you just learned" + "Wrap-up" (no "What's missing" — this is the last notebook)
12. Cleanup

---

## Important note for the implementer

The SDK API has shifted across versions. **Use `a2a-sdk==0.3.26`** (pin in the install step) to match the patterns this plan assumes. The reference implementation this plan is modelled on is the `helloworld` example at `https://github.com/a2aproject/a2a-samples/tree/main/samples/python/agents/helloworld`.

If imports in the installed version differ from what this plan shows (the SDK rearranges modules between minor versions), **adapt the imports to what the installed version actually exports** and report the substitutions in your final report. Do not file BLOCKED for import-path drift — that's expected SDK churn.

Likely-stable surfaces:
- `from a2a.server.agent_execution import AgentExecutor, RequestContext`
- `from a2a.server.events import EventQueue`
- `from a2a.server.request_handlers import DefaultRequestHandler`
- `from a2a.server.tasks import InMemoryTaskStore`
- `from a2a.server.routes import create_agent_card_routes, create_jsonrpc_routes`
- `from a2a.types import AgentCard, AgentCapabilities, AgentSkill, AgentInterface`
- `from a2a.helpers import new_task_from_user_message, new_text_artifact, new_text_message`
- `from a2a.types.a2a_pb2 import TaskState, TaskStatus, TaskStatusUpdateEvent, TaskArtifactUpdateEvent`
- Client side: `A2ACardResolver`, `A2AClient` (or `ClientConfig`+ a factory)

The plan's cells use these symbols. If they don't exist under those paths in v0.3.26, find the equivalents and substitute. The pedagogical structure does not depend on the exact import paths.

---

## Task 1: Scaffold

**Files:** Create `08_a2a_multi_agent.ipynb`.

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
python -c "import json; json.load(open('08_a2a_multi_agent.ipynb'))"
```

- [ ] **Step 3: Commit**

```bash
git add 08_a2a_multi_agent.ipynb
git commit -m "feat(a2a): scaffold 08_a2a_multi_agent.ipynb"
```

---

## Task 2: Title + intro markdown

**Files:** Modify `08_a2a_multi_agent.ipynb` (2 markdown cells).

- [ ] **Step 1: Title + "Why"**

```markdown
# 08 — The Payoff: Multi-Agent with the Official SDK

## Why this notebook exists

For seven notebooks we hand-rolled every byte of A2A. JSON-RPC envelopes, task state machines, SSE frames, webhook signatures, OAuth flows. The goal was making the protocol *legible*: if you've stuck with the series, you can now look at any A2A interaction in the wild and know what's happening.

**You don't want to keep writing that code.** In production you reach for an SDK that handles the protocol mechanics so your code can focus on the agent's behavior.

This notebook installs Google's official **`a2a-sdk`**, rebuilds the researcher and writer from notebook 01 using the SDK's `AgentExecutor` pattern, and orchestrates them with an SDK-driven coordinator. It closes with a side-by-side diff that shows how much hand-rolling the SDK absorbs.

> *Pins `a2a-sdk==0.3.26` (the v0.3 compatibility line). The 1.x line targets the A2A 1.0 spec, which has wire-format differences from what we've been building.*
```

- [ ] **Step 2: "What you'll learn"**

```markdown
## What you'll learn

- How the `a2a-sdk` decomposes an A2A server: `AgentExecutor` (your code), `DefaultRequestHandler` (protocol glue), `InMemoryTaskStore` (state), and route factories for Starlette.
- How to build a typed `AgentCard` programmatically with the SDK's classes.
- How an `AgentExecutor` enqueues `TaskStatusUpdateEvent` and `TaskArtifactUpdateEvent` instances — the same events you built by hand in notebook 05, just constructed via the SDK's types.
- How an A2A *client* (the coordinator) uses `A2ACardResolver` to discover an agent and then sends typed messages.
- Why the side-by-side line count matters: it's the cost of *not* having a standard.
- Where to go next: real LLMs behind the agents, multi-language interop, production deployment.
```

- [ ] **Step 3: Commit**

```bash
git add 08_a2a_multi_agent.ipynb
git commit -m "feat(a2a): add intro markdown to notebook 08"
```

---

## Task 3: Setup

**Files:** Modify `08_a2a_multi_agent.ipynb` (markdown header + 2 code cells).

- [ ] **Step 1: Markdown cell**

```markdown
## 1. Setup

Install `a2a-sdk` (pinned to the v0.3-compatible line) and import the helpers we'll use throughout.
```

- [ ] **Step 2: Install cell**

```python
import sys
import subprocess

subprocess.check_call([sys.executable, "-m", "pip", "install", "--quiet", "a2a-sdk==0.3.26"])
print("a2a-sdk installed")
```

- [ ] **Step 3: Run and verify**

Expected: `a2a-sdk installed` (possibly preceded by pip output if the package wasn't cached). If the install fails due to network or PyPI issues, retry once; if it still fails, surface as NEEDS_CONTEXT.

- [ ] **Step 4: Imports cell**

```python
import asyncio
import threading
import time

import httpx
import uvicorn
from starlette.applications import Starlette

# SDK server surfaces. Adjust import paths if a different a2a-sdk version
# rearranged the module tree — the symbols themselves should still exist.
from a2a.server.agent_execution import AgentExecutor, RequestContext
from a2a.server.events import EventQueue
from a2a.server.request_handlers import DefaultRequestHandler
from a2a.server.tasks import InMemoryTaskStore
from a2a.server.routes import create_agent_card_routes, create_jsonrpc_routes
from a2a.types import (
    AgentCard,
    AgentCapabilities,
    AgentInterface,
    AgentSkill,
)
from a2a.helpers import (
    new_task_from_user_message,
    new_text_artifact,
    new_text_message,
)
from a2a.types.a2a_pb2 import (
    TaskState,
    TaskStatus,
    TaskStatusUpdateEvent,
    TaskArtifactUpdateEvent,
)

# Background-thread uvicorn helper (same as previous notebooks).
_servers: list[uvicorn.Server] = []


def run_server_in_thread(app, port: int) -> uvicorn.Server:
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

- [ ] **Step 5: Run, expect `Setup OK`**

If any SDK imports fail, the message should reveal which symbol moved. Update the import paths to match what the installed version exports and re-run.

- [ ] **Step 6: Commit**

```bash
git add 08_a2a_multi_agent.ipynb
git commit -m "feat(a2a): install a2a-sdk and import SDK helpers in notebook 08"
```

---

## Task 4: Section 1 — A tour of the SDK

**Files:** Modify `08_a2a_multi_agent.ipynb` (1 markdown cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 2. A Tour of the SDK

| SDK piece | What it replaces (vs. our hand-rolled code) |
|---|---|
| `AgentExecutor` (subclass it; implement `async execute(ctx, queue)`) | The whole `handle_jsonrpc` function from notebook 03 — method dispatch, task creation, status updates, artifact assembly. |
| `RequestContext` | Parsed `Message`, `task_id`, `context_id` — instead of plucking them out of `req.params["message"]` by hand. |
| `EventQueue` | Replaces the `_set_status` / `_set_artifacts` lock-protected stores plus the SSE generator — you `await event_queue.enqueue_event(thing)` and the SDK handles delivery for both `message/send` (last event wins) and `message/stream` (streamed). |
| `DefaultRequestHandler` | The router that maps JSON-RPC method names to `AgentExecutor` callbacks plus task lifecycle plumbing (the entire notebook 04 state machine). |
| `InMemoryTaskStore` | Our `TASKS` dict + `STORE_LOCK` from notebook 04. |
| `create_agent_card_routes` / `create_jsonrpc_routes` | The FastAPI route declarations from notebooks 02 and 03. |
| `AgentCard`, `AgentCapabilities`, `AgentSkill`, `AgentInterface` (types) | Our hand-written pydantic `AgentCard` model from notebook 02. |
| `new_task_from_user_message`, `new_text_artifact`, `new_text_message` (helpers) | Boilerplate task/artifact construction. |
| `TaskStatusUpdateEvent`, `TaskArtifactUpdateEvent` (protobuf types) | Our hand-written event models from notebook 05. |
| `A2ACardResolver`, `A2AClient` (client side) | The hand-rolled JSON-RPC envelope construction + `httpx.post` calls. |

Same protocol on the wire — what the SDK does is absorb the *mechanics* of producing and consuming it. Field names in Python switch from `defaultInputModes` (the JSON wire shape we've been using) to `default_input_modes` (the SDK's snake_case Python convention); the SDK serializes back to the spec-correct camelCase on the wire.
```

- [ ] **Step 2: Commit**

```bash
git add 08_a2a_multi_agent.ipynb
git commit -m "feat(a2a): tour the SDK surfaces in notebook 08"
```

---

## Task 5: Section 2 — Researcher with the SDK

**Files:** Modify `08_a2a_multi_agent.ipynb` (markdown + code cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 3. The Researcher, SDK-Built

The `ResearcherAgent` is just a callable that looks up facts. The `ResearcherAgentExecutor` is the SDK adapter — given a `RequestContext` and an `EventQueue`, it enqueues a `working` status update, runs the agent, then enqueues the artifact and a `completed` status update. Compare cell-by-cell against `_handle_message_send` from notebook 03 to see what the SDK absorbs.
```

- [ ] **Step 2: Researcher code cell**

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


class ResearcherAgent:
    """Pure-Python agent logic — knows nothing about A2A."""

    async def invoke(self, topic: str) -> list[str] | None:
        return FACTS_BY_TOPIC.get(topic.strip().lower())


class ResearcherAgentExecutor(AgentExecutor):
    def __init__(self) -> None:
        self.agent = ResearcherAgent()

    async def execute(self, context: RequestContext, event_queue: EventQueue) -> None:
        task = context.current_task or new_task_from_user_message(context.message)
        await event_queue.enqueue_event(task)

        await event_queue.enqueue_event(TaskStatusUpdateEvent(
            task_id=context.task_id,
            context_id=context.context_id,
            status=TaskStatus(
                state=TaskState.TASK_STATE_WORKING,
                message=new_text_message("Looking up facts…"),
            ),
        ))

        topic = context.message.parts[0].text  # SDK normalises to .text
        facts = await self.agent.invoke(topic)

        if facts is None:
            await event_queue.enqueue_event(TaskStatusUpdateEvent(
                task_id=context.task_id,
                context_id=context.context_id,
                status=TaskStatus(
                    state=TaskState.TASK_STATE_FAILED,
                    message=new_text_message(f"No facts on file for {topic!r}."),
                ),
            ))
            return

        await event_queue.enqueue_event(TaskArtifactUpdateEvent(
            task_id=context.task_id,
            context_id=context.context_id,
            artifact=new_text_artifact(name=f"facts-about-{topic.lower()}", text="\n".join(facts)),
        ))
        await event_queue.enqueue_event(TaskStatusUpdateEvent(
            task_id=context.task_id,
            context_id=context.context_id,
            status=TaskStatus(state=TaskState.TASK_STATE_COMPLETED),
        ))

    async def cancel(self, context: RequestContext, event_queue: EventQueue) -> None:
        raise NotImplementedError("cancel not supported in this demo")


researcher_card = AgentCard(
    name="Researcher",
    description="Returns canned facts on a small set of well-known topics.",
    version="0.1.0",
    default_input_modes=["text/plain"],
    default_output_modes=["text/plain"],
    capabilities=AgentCapabilities(streaming=True),
    supported_interfaces=[
        AgentInterface(protocol_binding="JSONRPC", url="http://127.0.0.1:8010"),
    ],
    skills=[AgentSkill(
        id="research_topic",
        name="Research a topic",
        description="Given a topic, return facts about it.",
        tags=["research"],
        examples=["octopuses", "rome"],
    )],
)

researcher_handler = DefaultRequestHandler(
    agent_executor=ResearcherAgentExecutor(),
    task_store=InMemoryTaskStore(),
    agent_card=researcher_card,
)

researcher_routes = []
researcher_routes.extend(create_agent_card_routes(researcher_card))
researcher_routes.extend(create_jsonrpc_routes(researcher_handler, "/"))
researcher_app = Starlette(routes=researcher_routes)

researcher_server = run_server_in_thread(researcher_app, port=8010)
print("Researcher (SDK) on http://127.0.0.1:8010")
```

- [ ] **Step 3: Run, expect `Researcher (SDK) on http://127.0.0.1:8010`**

- [ ] **Step 4: Commit**

```bash
git add 08_a2a_multi_agent.ipynb
git commit -m "feat(a2a): build researcher with a2a-sdk in notebook 08"
```

---

## Task 6: Section 3 — Writer with the SDK

**Files:** Modify `08_a2a_multi_agent.ipynb` (markdown + code cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 4. The Writer, SDK-Built

The writer takes a topic and a list of facts (joined by newlines) and emits a one-paragraph summary. Pure stub — no LLM. Built with the same SDK pattern as the researcher.
```

- [ ] **Step 2: Writer code cell**

```python
class WriterAgent:
    async def invoke(self, topic: str, fact_lines: str) -> str:
        facts = [line.strip() for line in fact_lines.splitlines() if line.strip()]
        return f"Here is what we know about {topic}: " + " ".join(facts)


class WriterAgentExecutor(AgentExecutor):
    def __init__(self) -> None:
        self.agent = WriterAgent()

    async def execute(self, context: RequestContext, event_queue: EventQueue) -> None:
        task = context.current_task or new_task_from_user_message(context.message)
        await event_queue.enqueue_event(task)

        # Input convention for this demo: message text is "<topic>\n---\n<fact_lines>".
        raw = context.message.parts[0].text
        if "\n---\n" not in raw:
            await event_queue.enqueue_event(TaskStatusUpdateEvent(
                task_id=context.task_id,
                context_id=context.context_id,
                status=TaskStatus(
                    state=TaskState.TASK_STATE_FAILED,
                    message=new_text_message("Writer expects '<topic>\\n---\\n<facts>'."),
                ),
            ))
            return
        topic, fact_lines = raw.split("\n---\n", 1)

        paragraph = await self.agent.invoke(topic.strip(), fact_lines)

        await event_queue.enqueue_event(TaskArtifactUpdateEvent(
            task_id=context.task_id,
            context_id=context.context_id,
            artifact=new_text_artifact(name="summary", text=paragraph),
        ))
        await event_queue.enqueue_event(TaskStatusUpdateEvent(
            task_id=context.task_id,
            context_id=context.context_id,
            status=TaskStatus(state=TaskState.TASK_STATE_COMPLETED),
        ))

    async def cancel(self, context: RequestContext, event_queue: EventQueue) -> None:
        raise NotImplementedError("cancel not supported in this demo")


writer_card = AgentCard(
    name="Writer",
    description="Turns a topic + facts into a one-paragraph summary.",
    version="0.1.0",
    default_input_modes=["text/plain"],
    default_output_modes=["text/plain"],
    capabilities=AgentCapabilities(streaming=True),
    supported_interfaces=[
        AgentInterface(protocol_binding="JSONRPC", url="http://127.0.0.1:8011"),
    ],
    skills=[AgentSkill(
        id="write_summary",
        name="Write a summary",
        description="Given a topic and facts, return a one-paragraph summary.",
        tags=["writing"],
        examples=["octopuses\\n---\\nThey have three hearts."],
    )],
)

writer_handler = DefaultRequestHandler(
    agent_executor=WriterAgentExecutor(),
    task_store=InMemoryTaskStore(),
    agent_card=writer_card,
)

writer_routes = []
writer_routes.extend(create_agent_card_routes(writer_card))
writer_routes.extend(create_jsonrpc_routes(writer_handler, "/"))
writer_app = Starlette(routes=writer_routes)

writer_server = run_server_in_thread(writer_app, port=8011)
print("Writer (SDK) on http://127.0.0.1:8011")
```

- [ ] **Step 3: Run, expect `Writer (SDK) on http://127.0.0.1:8011`**

- [ ] **Step 4: Commit**

```bash
git add 08_a2a_multi_agent.ipynb
git commit -m "feat(a2a): build writer with a2a-sdk in notebook 08"
```

---

## Task 7: Section 4 — The coordinator

**Files:** Modify `08_a2a_multi_agent.ipynb` (markdown + code cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 5. The Coordinator

The coordinator is an A2A *client* — it doesn't expose its own endpoint. It uses the SDK's `A2ACardResolver` to fetch each agent's card (proving discovery works), then sends a `message/send` request to each, picks the result artifact out of the response, and chains them: researcher's text becomes part of the writer's input.

If the SDK's client API differs from the names used here, swap in the equivalent — what matters is that you (a) resolve each agent's card by base URL and (b) send a typed message to its `url` and (c) read the resulting `Task.artifacts[0]`.
```

- [ ] **Step 2: Coordinator code cell**

```python
# The SDK's high-level client surfaces vary across versions. The patterns below
# use the most stable public symbols. Adapt to whatever the installed version
# exports — the wire shape is identical to what we hand-rolled in notebook 03.

from a2a.client import A2ACardResolver, A2AClient, ClientConfig  # adjust if needed


async def call_agent(base_url: str, user_text: str) -> str:
    """Send `user_text` to the agent at `base_url`; return its first artifact's text."""
    async with httpx.AsyncClient(timeout=30.0) as http:
        resolver = A2ACardResolver(base_url=base_url, httpx_client=http)
        card = await resolver.get_agent_card()

        client = A2AClient(config=ClientConfig(httpx_client=http), agent_card=card)
        try:
            response = await client.send_message(text=user_text)
        finally:
            await client.close()

    # The SDK returns a Task or a Message wrapping one; extract the first text artifact.
    task = response if hasattr(response, "artifacts") else response.result
    if not task.artifacts:
        raise RuntimeError(f"No artifacts in response from {base_url}")
    return task.artifacts[0].parts[0].text


async def coordinate(topic: str) -> str:
    facts_text = await call_agent("http://127.0.0.1:8010", topic)
    writer_input = f"{topic}\n---\n{facts_text}"
    paragraph = await call_agent("http://127.0.0.1:8011", writer_input)
    return paragraph


print("Coordinator defined.")
```

- [ ] **Step 3: Run, expect `Coordinator defined.`**

If the import line raises `ImportError` because the installed SDK uses different client class names, find the equivalents in the installed version (try `from a2a.client import …` and look at `a2a.client.__all__`) and adapt — then re-run.

- [ ] **Step 4: Commit**

```bash
git add 08_a2a_multi_agent.ipynb
git commit -m "feat(a2a): define coordinator client in notebook 08"
```

---

## Task 8: Section 5 — Run the end-to-end flow

**Files:** Modify `08_a2a_multi_agent.ipynb` (markdown + code cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 6. Run It

Jupyter supports top-level `await`, so we can call the async coordinator directly.
```

- [ ] **Step 2: Run code cell**

```python
paragraph = await coordinate("octopuses")
print(paragraph)
```

- [ ] **Step 3: Run and verify**

Expected output (whitespace exact):

```
Here is what we know about octopuses: Octopuses have three hearts. They can change color in under a second. Each of their arms has its own neural cluster.
```

If the run fails with an SDK-related error (unexpected response shape, etc.), capture the traceback and adapt the `call_agent` extraction logic to what the installed version actually returns. Don't change the agents.

- [ ] **Step 4: Commit**

```bash
git add 08_a2a_multi_agent.ipynb
git commit -m "feat(a2a): demonstrate multi-agent coordination in notebook 08"
```

---

## Task 9: Section 6 — Side-by-side diff

**Files:** Modify `08_a2a_multi_agent.ipynb` (1 markdown cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 7. Hand-Rolled vs SDK: The Diff

Same wire format, same protocol semantics — radically different line counts.

**Calling one agent with one `message/send` (hand-rolled, from notebook 03):**

```python
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
data = http_resp.json()
if data.get("error"):
    raise RuntimeError(f"JSON-RPC error: {data['error']}")
task = Task.model_validate(data["result"])
return task.artifacts[0].parts[0].text
```

That's ~17 lines, plus you maintain the `Task`, `Message`, `TextPart`, `Artifact`, `JSONRPCRequest`, `JSONRPCResponse` pydantic models (~70 more lines from notebook 03).

**The same call, SDK:**

```python
resolver = A2ACardResolver(base_url=base_url, httpx_client=http)
card = await resolver.get_agent_card()
client = A2AClient(config=ClientConfig(httpx_client=http), agent_card=card)
response = await client.send_message(text=user_text)
return response.artifacts[0].parts[0].text
```

Five lines. Zero protocol-model maintenance. Plus you get streaming support, auth handling, and webhook delivery for free — none of which we wired up in the SDK demos above, but all of which the SDK already implements.

**Server side, the multiplier is similar.** A single `AgentExecutor.execute` method replaces our `handle_jsonrpc` dispatcher, our `_handle_message_send`, our task store, our state-transition helpers, and our SSE generator. ~30 lines vs ~150.

**That's the payoff for adopting the standard.** Every team that builds A2A this way speaks the same protocol on the wire, and no team has to write protocol code.
```

- [ ] **Step 2: Commit**

```bash
git add 08_a2a_multi_agent.ipynb
git commit -m "feat(a2a): contrast hand-rolled vs SDK in notebook 08"
```

---

## Task 10: Section 7 — Stretch: swap a stub for Claude

**Files:** Modify `08_a2a_multi_agent.ipynb` (1 markdown cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 8. Stretch: Swap a Stub for Claude

The agents above return canned data — the focus of this notebook is the SDK and the multi-agent shape, not the agents' intelligence. Swapping the stub for a real LLM is a *one-method* change.

```python
# Add to imports
from anthropic import Anthropic

claude = Anthropic()  # reads ANTHROPIC_API_KEY from env

class ResearcherAgent:
    async def invoke(self, topic: str) -> list[str] | None:
        resp = await asyncio.to_thread(
            claude.messages.create,
            model="claude-sonnet-4-6",
            max_tokens=400,
            messages=[{"role": "user", "content": f"Give me 3 surprising facts about {topic} as a JSON array of strings."}],
        )
        text = resp.content[0].text
        import json, re
        match = re.search(r"\[.*\]", text, re.DOTALL)
        return json.loads(match.group(0)) if match else None
```

Everything else stays the same — the `ResearcherAgentExecutor`, the `AgentCard`, the routes, the coordinator. The agent's *intelligence* is now Claude; the *protocol* is still A2A, still served by the SDK.

We don't run this here so the notebook works offline. If you want to try it: `pip install anthropic`, set `ANTHROPIC_API_KEY`, paste the snippet over the stub `ResearcherAgent`, and re-run sections 3–6.
```

- [ ] **Step 2: Commit**

```bash
git add 08_a2a_multi_agent.ipynb
git commit -m "feat(a2a): document Claude swap-in stretch in notebook 08"
```

---

## Task 11: Wrap-up + cleanup

**Files:** Modify `08_a2a_multi_agent.ipynb` (1 markdown cell + 1 code cell).

- [ ] **Step 1: Wrap-up markdown cell**

```markdown
## What you just learned (and the whole series)

This notebook:

- `a2a-sdk`'s shape: `AgentExecutor` for behavior, `DefaultRequestHandler` for protocol, `InMemoryTaskStore` for state, route factories for HTTP, typed `AgentCard` for discovery.
- How to build two SDK-driven agents and have them collaborate via an A2A client coordinator.
- The line-count case for adopting the SDK over hand-rolling.

The whole series:

1. The pain of integration without a shared protocol (nb01).
2. The Agent Card as a discovery primitive (nb02).
3. JSON-RPC 2.0 and the first `message/send` (nb03).
4. Task lifecycle, polling, cancel, multi-turn (nb04).
5. Streaming via SSE (nb05).
6. Push notifications and webhooks (nb06).
7. Authentication (bearer, OAuth2, signed callbacks) (nb07).
8. The SDK, multi-agent coordination, the payoff (this notebook).

### Where to go from here

- **Real LLMs behind your agents** — swap any `*Agent` class for one that calls Claude / GPT / a local model.
- **Cross-language interop** — JS/Go/Java SDKs exist for A2A; an agent in any of those languages will talk to your Python agents over the same wire.
- **Production deployment** — TLS, real OAuth providers, observability, fan-out, message-queue-backed task stores. The SDK supports all of these via its extension points.
- **MCP comparison** — A2A is for agent-to-agent; MCP is for agent-to-tool. Both can coexist on the same agent. (See the MCP series in this repo.)
```

- [ ] **Step 2: Cleanup code cell**

```python
shutdown_all_servers()
print("All servers stopped.")
```

- [ ] **Step 3: Run, expect `All servers stopped.`**

- [ ] **Step 4: Commit**

```bash
git add 08_a2a_multi_agent.ipynb
git commit -m "feat(a2a): wrap up the series in notebook 08"
```

---

## Task 12: Fresh-kernel verification

**Files:** none.

- [ ] **Step 1: Execute**

```bash
/opt/anaconda3/bin/jupyter nbconvert --to notebook --execute --inplace 08_a2a_multi_agent.ipynb
```

Expected: succeeds. The install cell may take several seconds on cold cache; subsequent runs are fast.

- [ ] **Step 2: Verify expected outputs**

In order:

1. `a2a-sdk installed` (possibly preceded by pip output).
2. `Setup OK`
3. `Researcher (SDK) on http://127.0.0.1:8010`
4. `Writer (SDK) on http://127.0.0.1:8011`
5. `Coordinator defined.`
6. The full one-paragraph summary: `Here is what we know about octopuses: Octopuses have three hearts. They can change color in under a second. Each of their arms has its own neural cluster.`
7. `All servers stopped.`

No cell raises an unhandled exception. **If SDK import paths or client API shapes had to be adapted from the plan**, that is expected and not a failure — report the substitutions in your final summary.

- [ ] **Step 3: Verify ports released**

```bash
lsof -iTCP:8010 -sTCP:LISTEN
lsof -iTCP:8011 -sTCP:LISTEN
```

Expected: both empty.

- [ ] **Step 4: Commit clean run**

```bash
git add 08_a2a_multi_agent.ipynb
git commit -m "chore(a2a): commit clean fresh-kernel run of notebook 08"
```

---

## Done

After Task 12 passes, the A2A learning series is complete: eight notebooks from "two services that can't talk" to "two SDK-driven agents collaborating over A2A." No follow-on plan needed.
