# FastMCP Notebook 08 — `08_fastmcp_real_llm_end_to_end.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the eighth and final notebook in the FastMCP learning series — the payoff. Stand up two `FastMCP` apps (`research_mcp`, `writer_mcp`), `mount()` them under a `main_mcp` so a single HTTP endpoint exposes tools from both, serve `main_mcp` over `streamable-http` on port 8020 on a background daemon thread, and drive it from a real LLM tool-use loop. Default LLM is **Anthropic Claude** (`claude-sonnet-4-6`) via the `anthropic` SDK; fallback is **OpenAI** (`gpt-4o-mini`) via the `openai` SDK when `ANTHROPIC_API_KEY` is unset and `OPENAI_API_KEY` is set. Close the notebook with an in-memory `Client(main_mcp)` pytest-style test (called directly from a cell — no `pytest` CLI), then a reflection cell on what FastMCP bought us across the eight notebooks.

**Architecture:** A single Jupyter notebook. Two small `FastMCP` instances (`research_mcp` with `lookup_topic`, `writer_mcp` with `format_as_bullet_list` and `make_title`) get mounted under `main_mcp = FastMCP("main")` via `main_mcp.mount(...)`. The mounted server is launched with `main_mcp.run(transport="streamable-http", host="127.0.0.1", port=8020)` on a daemon thread (same pattern as notebooks 03–07). The notebook then opens a `fastmcp.Client("http://127.0.0.1:8020/mcp/")`, calls `list_tools()`, converts the FastMCP tool definitions into Anthropic / OpenAI tool-format dicts via two small helper functions, and runs an LLM tool-use loop until the model returns a final text answer. The closing in-memory test uses `Client(main_mcp)` (the bare server object — no socket) to assert `lookup_topic("octopus")` works.

**Tech Stack:** Python 3.11+, Jupyter, `fastmcp` 2.x, `anthropic` SDK (default LLM driver), `openai` SDK (fallback LLM driver), threading, asyncio. Same baseline stack as notebooks 03–07 plus the LLM SDKs.

**Companion spec:** `docs/superpowers/specs/2026-05-18-fastmcp-learning-series-design.md` (notebook 08 section).

**FastMCP spec version targeted:** `fastmcp` 2.x (current stable). Transport: `streamable-http`. HTTP endpoint path: `/mcp/`. The `mount()` API used here is the most-likely FastMCP 2.x form (`main_mcp.mount(prefix="...", server=sub_mcp)`) — see the `> **API note:**` callouts in Task 6 for the fallback if the installed version exposes a different signature.

**Port assignment:** Primary FastMCP HTTP server on `127.0.0.1:8020`. (Per spec: 8000/8001 are taken by the user's dev server and 8010/8011 by the `a2a/` series. Port 8021 is reserved for the secondary FastMCP app in notebook 05; this notebook only needs one HTTP port because `mount()` exposes both mounted servers through `main_mcp`.)

**LLM API key:** This is the **only** notebook in the series that requires an API key. Setup checks for `ANTHROPIC_API_KEY` (default) or `OPENAI_API_KEY` (fallback) and `raise RuntimeError` with a helpful message if neither is set. Default model: `claude-sonnet-4-6`. Fallback model: `gpt-4o-mini`.

---

## File Structure

- **Create:** `fastmcp/08_fastmcp_real_llm_end_to_end.ipynb` — the entire notebook, self-contained (no imports from prior notebooks).
- **Modify:** none.
- **No separate test files** — the closing in-memory test lives inside the notebook and is invoked from a cell. Verification is the notebook running top-to-bottom in a fresh kernel (Task 17).

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown — includes the API-key note)
2. **"What you'll learn"** (markdown bullets)
3. **Setup: imports, API-key check, threaded server helper** (markdown header + 1 code cell)
4. **Section 2: Build the research server** (markdown header + 1 code cell — `research_mcp` with `lookup_topic`)
5. **Section 3: Build the writer server** (markdown header + 1 code cell — `writer_mcp` with `format_as_bullet_list`, `make_title`)
6. **Section 4: Mount them under a single main server** (markdown header + 1 code cell — `main_mcp.mount(...)`, in-memory tool listing)
7. **Section 5: Serve over HTTP** (markdown header + 1 code cell — daemon-thread launcher on port 8020)
8. **Section 6: Wire the LLM to the MCP tools** (markdown header + 1 code cell — two converters + LLM client selection)
9. **Section 7: Run the loop** (markdown header + 1 code cell — full Anthropic/OpenAI tool-use loop driving a real prompt)
10. **Section 8: Unit-test the server with the in-memory client** (markdown header + 1 code cell — `async def test_lookup_topic_returns_facts(): ...` + invocation)
11. **"What you just learned"** (markdown)
12. **"What FastMCP bought us"** (markdown — closing reflection on the eight-notebook arc)
13. **Section 9: Cleanup** (markdown header + 1 code cell — daemon-thread liveness check, kernel-exit cleanup note)

(No `## What's missing` cell — this is the series finale.)

---

## Task 1: Create the notebook scaffold

**Files:**
- Create: `fastmcp/08_fastmcp_real_llm_end_to_end.ipynb`

- [ ] **Step 1: Confirm the `fastmcp/` directory exists**

```bash
ls -la fastmcp/
```

Expected: directory exists. If it does not yet (because notebooks 01–07 haven't been built), create it:

```bash
mkdir -p fastmcp
```

- [ ] **Step 2: Create an empty notebook with the Python 3 kernel**

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

- [ ] **Step 3: Verify the file is valid JSON and opens as a notebook**

```bash
python -c "import json; json.load(open('fastmcp/08_fastmcp_real_llm_end_to_end.ipynb'))"
ls -la fastmcp/08_fastmcp_real_llm_end_to_end.ipynb
```

Expected: no exception, file non-zero size.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/08_fastmcp_real_llm_end_to_end.ipynb
git commit -m "feat(fastmcp): scaffold 08_fastmcp_real_llm_end_to_end.ipynb"
```

---

## Task 2: Add title and "Why this notebook exists"

**Files:**
- Modify: `fastmcp/08_fastmcp_real_llm_end_to_end.ipynb` (add 2 markdown cells)

- [ ] **Step 1: Add the title + "Why this notebook exists" markdown cell**

Cell content (markdown):

```markdown
# 08 — Real LLM End-to-End: Composed FastMCP Servers Driving a Tool-Use Loop

## Why this notebook exists

For seven notebooks we built FastMCP servers and called them ourselves — from the in-memory `Client(mcp)`, from `fastmcp.Client(url)`, from hand-written test cells. Every tool we wrote was waiting for the same thing: **a real model to pick it up and use it**. That is what this notebook does.

We stand up two small FastMCP apps — a `research_mcp` that returns canned facts about a few topics, and a `writer_mcp` that formats text — and compose them under a single primary server with `main_mcp.mount(...)`. One HTTP endpoint on port 8020 exposes the tools from both. Then we hand the tool list to **Claude** (`claude-sonnet-4-6`), or OpenAI as a fallback, and run a real tool-use loop: the model picks `lookup_topic` to gather facts, then picks `format_as_bullet_list` to format them, then returns a final text answer. The notebook closes by rewriting the same setup as a pytest-style test using the **in-memory `Client(main_mcp)`** — proving the server is unit-testable without ever opening a socket — and ends with a reflection on what FastMCP bought us across the series.

> **This is the only notebook in the series that requires a real LLM API key.** The setup cell checks for `ANTHROPIC_API_KEY` (default driver) or `OPENAI_API_KEY` (fallback) and raises a clear error if neither is set. Notebooks 01–07 are all key-free by design; notebook 08 is the payoff, and the payoff requires a model.

> *Targets `fastmcp` 2.x. Default LLM: Anthropic Claude (`claude-sonnet-4-6`) via the `anthropic` SDK. Fallback: OpenAI (`gpt-4o-mini`) via the `openai` SDK.*
```

- [ ] **Step 2: Add the "What you'll learn" markdown cell**

Cell content (markdown):

```markdown
## What you'll learn

- How to compose two FastMCP apps under a single primary server with `main_mcp.mount(...)` so one HTTP endpoint serves tools from both.
- How a real LLM consumes MCP tools: list the tools, convert each one into the LLM provider's tool-schema format, and pass that list to the model.
- The shape of an Anthropic tool-use loop: `messages.create(...)` returns content blocks; when any block is a `tool_use`, execute it via `client.call_tool(...)`, append a `tool_result` block, send again, repeat until the model returns plain text (i.e. `stop_reason != "tool_use"`).
- The equivalent shape for the OpenAI Chat Completions tool-use loop — `tool_calls` on the assistant message, `role="tool"` reply messages — so you can fall back to either provider with the same converter pattern.
- How to write a pytest-style test for a FastMCP server using `Client(main_mcp)` directly — no subprocess, no socket, fully in-process — and invoke it from a notebook cell.
- A reflection on the **eight-notebook arc**: what FastMCP absorbed compared to the raw-SDK baseline in notebook 01.
```

- [ ] **Step 3: Verify cells render**

Open the notebook in Jupyter/VS Code and confirm both cells render as markdown.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/08_fastmcp_real_llm_end_to_end.ipynb
git commit -m "feat(fastmcp): add intro markdown to notebook 08"
```

---

## Task 3: Setup — imports, API-key check, threaded server helper

**Files:**
- Modify: `fastmcp/08_fastmcp_real_llm_end_to_end.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 1. Setup

The setup cell does three things:

1. Imports the FastMCP server and client classes, the LLM SDKs (`anthropic` and `openai`), and the standard-library helpers we need (`asyncio`, `json`, `os`, `threading`, `time`).
2. **Checks that at least one LLM API key is set.** `ANTHROPIC_API_KEY` is the default; `OPENAI_API_KEY` is the fallback. If neither is set, the cell raises a `RuntimeError` with a clear remediation message. This is the only notebook in the series with this check.
3. Defines `start_http_server_in_thread(mcp, host, port)` — the same daemon-thread helper used in notebooks 03–07. It launches `mcp.run(transport="streamable-http", host=host, port=port)` on a background `threading.Thread(daemon=True)` so the notebook keeps control of the kernel.

> **Quirk:** FastMCP's `mcp.run(...)` is **blocking**. In a notebook we wrap it on a daemon thread; the thread dies automatically when the kernel exits and the port is released. There is no explicit `shutdown()` API in 2.x.
```

- [ ] **Step 2: Add the setup code cell**

```python
import asyncio
import json
import os
import threading
import time

from fastmcp import Client, FastMCP

# Default driver: Anthropic Claude. Fallback: OpenAI. Imported lazily below so
# that a missing optional dep doesn't break the check.
HAVE_ANTHROPIC = False
HAVE_OPENAI = False
try:
    from anthropic import Anthropic  # noqa: F401
    HAVE_ANTHROPIC = True
except ImportError:
    pass
try:
    from openai import OpenAI  # noqa: F401
    HAVE_OPENAI = True
except ImportError:
    pass

ANTHROPIC_KEY = os.getenv("ANTHROPIC_API_KEY")
OPENAI_KEY = os.getenv("OPENAI_API_KEY")

if not (ANTHROPIC_KEY or OPENAI_KEY):
    raise RuntimeError(
        "Notebook 08 requires an LLM API key.\n"
        "Set ANTHROPIC_API_KEY (default driver, claude-sonnet-4-6) or\n"
        "set OPENAI_API_KEY (fallback, gpt-4o-mini), then restart the kernel.\n"
        "Notebooks 01-07 in this series are key-free; only this one drives a real LLM."
    )

# Decide which provider we will use later in Section 6.
if ANTHROPIC_KEY and HAVE_ANTHROPIC:
    LLM_PROVIDER = "anthropic"
    LLM_MODEL = "claude-sonnet-4-6"
elif OPENAI_KEY and HAVE_OPENAI:
    LLM_PROVIDER = "openai"
    LLM_MODEL = "gpt-4o-mini"
elif ANTHROPIC_KEY and not HAVE_ANTHROPIC:
    raise RuntimeError(
        "ANTHROPIC_API_KEY is set but the `anthropic` package is not installed. "
        "Run `pip install anthropic` (or fall back by unsetting ANTHROPIC_API_KEY "
        "and setting OPENAI_API_KEY with `openai` installed)."
    )
else:
    raise RuntimeError(
        "OPENAI_API_KEY is set but the `openai` package is not installed. "
        "Run `pip install openai`."
    )

_server_threads: list[threading.Thread] = []


def start_http_server_in_thread(
    mcp: FastMCP,
    host: str = "127.0.0.1",
    port: int = 8020,
) -> threading.Thread:
    """Run `mcp.run(transport="streamable-http", ...)` on a daemon thread.

    Blocking call lives on the thread, so the notebook keeps control.
    The daemon flag ensures the thread is torn down at kernel exit.
    """
    def _serve() -> None:
        mcp.run(transport="streamable-http", host=host, port=port)

    thread = threading.Thread(target=_serve, daemon=True)
    thread.start()
    _server_threads.append(thread)
    return thread


print(f"Setup OK. LLM provider = {LLM_PROVIDER!r}, model = {LLM_MODEL!r}")
```

- [ ] **Step 3: Run the cell and verify**

Expected output (when `ANTHROPIC_API_KEY` is set and the `anthropic` package is installed):

```
Setup OK. LLM provider = 'anthropic', model = 'claude-sonnet-4-6'
```

When only `OPENAI_API_KEY` is set and `openai` is installed:

```
Setup OK. LLM provider = 'openai', model = 'gpt-4o-mini'
```

If imports fail because a package is missing:

```bash
pip install "fastmcp>=2,<3" anthropic openai
```

(`anthropic` and `openai` are both optional; you only need the one for whichever provider's key you have set.)

If the API-key `RuntimeError` fires, set one of `ANTHROPIC_API_KEY` or `OPENAI_API_KEY` in your shell and restart the kernel.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/08_fastmcp_real_llm_end_to_end.ipynb
git commit -m "feat(fastmcp): add setup cell with API-key check to notebook 08"
```

---

## Task 4: Section 2 — Build the research server

**Files:**
- Modify: `fastmcp/08_fastmcp_real_llm_end_to_end.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 2. Build the Research Server

`research_mcp = FastMCP("research")` exposes one tool, `lookup_topic(topic: str) -> str`, that returns hard-coded facts about a small set of well-known topics. The data is canned on purpose — this notebook is about the framework, not about real research. The LLM will see `lookup_topic` in its tool list and decide when to call it.

The tool's docstring becomes its description (the LLM reads this to decide whether to call it). The `topic: str` type hint becomes a JSON-Schema property in the tool definition FastMCP emits, which is what the LLM eventually sees as its `input_schema`. Both happen automatically — no schema by hand.
```

- [ ] **Step 2: Add the research-server code cell**

```python
research_mcp = FastMCP("research")

FACTS_BY_TOPIC: dict[str, list[str]] = {
    "octopus": [
        "Octopuses have three hearts.",
        "They can change color in under a second.",
        "Each of their arms has its own neural cluster.",
    ],
    "rome": [
        "Rome was founded in 753 BCE according to tradition.",
        "The Roman Empire at its peak spanned roughly 5 million km^2.",
        "Roman concrete used volcanic ash and is still studied today.",
    ],
    "saturn": [
        "Saturn is the sixth planet from the Sun.",
        "Its rings are made mostly of water ice.",
        "Saturn has at least 146 known moons as of recent counts.",
    ],
}


@research_mcp.tool
def lookup_topic(topic: str) -> str:
    """Look up canned facts about a well-known topic.

    Known topics are: octopus, rome, saturn. Returns the facts as a single
    newline-separated string. Returns an explicit "no facts on file" string
    for unknown topics so the LLM can recover gracefully.
    """
    facts = FACTS_BY_TOPIC.get(topic.strip().lower())
    if facts is None:
        return f"No facts on file for topic: {topic!r}. Known topics: octopus, rome, saturn."
    return "\n".join(facts)


print(f"Research server defined: {research_mcp.name!r} with 1 tool (lookup_topic)")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Research server defined: 'research' with 1 tool (lookup_topic)
```

- [ ] **Step 4: Commit**

```bash
git add fastmcp/08_fastmcp_real_llm_end_to_end.ipynb
git commit -m "feat(fastmcp): build research_mcp with lookup_topic in notebook 08"
```

---

## Task 5: Section 3 — Build the writer server

**Files:**
- Modify: `fastmcp/08_fastmcp_real_llm_end_to_end.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 3. Build the Writer Server

`writer_mcp = FastMCP("writer")` exposes two tools: `format_as_bullet_list(text: str) -> str` and `make_title(text: str) -> str`. Both are deterministic string transforms — no LLM inside the tools. The point is to give the orchestrating LLM enough vocabulary to chain a small plan: "look up facts, then format them as a bullet list, then make a title for the result."

The LLM will see all three tools (`lookup_topic`, `format_as_bullet_list`, `make_title`) through a single MCP endpoint once we mount these two servers under `main_mcp` in the next section.
```

- [ ] **Step 2: Add the writer-server code cell**

```python
writer_mcp = FastMCP("writer")


@writer_mcp.tool
def format_as_bullet_list(text: str) -> str:
    """Format the input text as a Markdown bullet list.

    Each non-empty line of the input becomes a bullet. Lines are stripped of
    surrounding whitespace. Empty lines are dropped.
    """
    lines = [line.strip() for line in text.splitlines() if line.strip()]
    return "\n".join(f"- {line}" for line in lines)


@writer_mcp.tool
def make_title(text: str) -> str:
    """Generate a short title from the input text.

    Takes the first non-empty line, truncates to 60 characters, and applies
    title-case capitalization.
    """
    for line in text.splitlines():
        line = line.strip()
        if line:
            return line[:60].title()
    return "Untitled"


print(f"Writer server defined: {writer_mcp.name!r} with 2 tools (format_as_bullet_list, make_title)")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Writer server defined: 'writer' with 2 tools (format_as_bullet_list, make_title)
```

- [ ] **Step 4: Commit**

```bash
git add fastmcp/08_fastmcp_real_llm_end_to_end.ipynb
git commit -m "feat(fastmcp): build writer_mcp with bullet-list and title tools in notebook 08"
```

---

## Task 6: Section 4 — Mount them under a single main server

**Files:**
- Modify: `fastmcp/08_fastmcp_real_llm_end_to_end.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 4. Mount Them Under a Single Main Server

Composition is the FastMCP feature with no raw-SDK analogue. We create a third instance, `main_mcp = FastMCP("main")`, and **mount** the two domain servers under it. After mounting, a client that connects to `main_mcp` sees the tools from both underlying servers — same `list_tools()`, same `call_tool()` surface, one endpoint.

We verify with the in-memory `Client(main_mcp)` (no socket yet — we go over HTTP in the next section) that all three tools appear. Notebook 05 introduced `mount()` for production patterns; this notebook uses the same surface to make multi-server composition the LLM-facing default.

> **API note:** FastMCP 2.x's `mount()` API has shifted across minor versions. The most-common 2.x form is `main_mcp.mount(prefix="research_", server=research_mcp)` — a keyword `server=` argument plus an optional `prefix=` that becomes a tool-name prefix on the exposed tools. If the installed version uses positional-only arguments (e.g. `main_mcp.mount("research_", research_mcp)`) or a different keyword (e.g. `app=`), adapt to whatever the installed version's signature actually exposes. Confirm with `help(main_mcp.mount)` or `inspect.signature(main_mcp.mount)`. The pedagogical point — one main server exposes tools from two sub-servers — does not depend on the exact signature.

> **Quirk:** Mounted tools may or may not be exposed under a name prefix depending on the FastMCP version. In versions that prefix, you'll see `research_lookup_topic` instead of `lookup_topic` in the tool list. The LLM will still pick the right tool because the description is the same; the converter in Section 6 passes whatever name `list_tools()` returns through to the LLM unchanged.
```

- [ ] **Step 2: Add the mount + verify code cell**

```python
main_mcp = FastMCP("main")

# FastMCP 2.x most-common signature. If the installed version expects positional
# args or a different keyword (e.g. app=), adapt accordingly — see the API note above.
main_mcp.mount(prefix="research_", server=research_mcp)
main_mcp.mount(prefix="writer_", server=writer_mcp)


async def list_main_tools() -> list[str]:
    async with Client(main_mcp) as client:
        tools = await client.list_tools()
        return [t.name for t in tools]


tool_names = await list_main_tools()
print(f"main_mcp exposes {len(tool_names)} tool(s): {tool_names}")
```

- [ ] **Step 3: Run the cell and verify**

Expected output (FastMCP versions that *do* prefix mounted tool names):

```
main_mcp exposes 3 tool(s): ['research_lookup_topic', 'writer_format_as_bullet_list', 'writer_make_title']
```

Or, in FastMCP versions that do *not* prefix:

```
main_mcp exposes 3 tool(s): ['lookup_topic', 'format_as_bullet_list', 'make_title']
```

Verify only that the list has length 3 and contains a `lookup_topic`-shaped name, a `format_as_bullet_list`-shaped name, and a `make_title`-shaped name. The exact prefixing is version-dependent and does not affect the loop in Section 7.

If `main_mcp.mount(prefix="research_", server=research_mcp)` raises `TypeError` because the installed version uses different argument names, try `main_mcp.mount("research_", research_mcp)` (positional) or check `inspect.signature(main_mcp.mount)` and adapt. The cell prose already calls this out.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/08_fastmcp_real_llm_end_to_end.ipynb
git commit -m "feat(fastmcp): mount research and writer under main_mcp in notebook 08"
```

---

## Task 7: Section 5 — Serve over HTTP

**Files:**
- Modify: `fastmcp/08_fastmcp_real_llm_end_to_end.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 5. Serve over HTTP

Time to put `main_mcp` on a real socket. Same daemon-thread pattern as notebooks 03–07: call `start_http_server_in_thread(main_mcp, "127.0.0.1", 8020)`, sleep briefly so uvicorn has time to bind, then verify the thread is alive. The endpoint will be `http://127.0.0.1:8020/mcp/` (FastMCP 2.x's default streamable-HTTP path).

We deliberately serve **only `main_mcp`** over HTTP. The two sub-servers are reached *through* `main_mcp` — that's the whole point of `mount()`. One LLM connection, one tool list, three tools from two underlying services.
```

- [ ] **Step 2: Add the server-launch code cell**

```python
server_thread = start_http_server_in_thread(main_mcp, host="127.0.0.1", port=8020)

# Give uvicorn a moment to bind before we start calling the server.
time.sleep(1.5)

print(f"Server thread alive: {server_thread.is_alive()}")
print("main_mcp HTTP server should be listening on http://127.0.0.1:8020/mcp/")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Server thread alive: True
main_mcp HTTP server should be listening on http://127.0.0.1:8020/mcp/
```

If port 8020 is already in use, free it first (only kill processes you recognize):

```bash
lsof -ti tcp:8020 | xargs kill -9
```

- [ ] **Step 4: Commit**

```bash
git add fastmcp/08_fastmcp_real_llm_end_to_end.ipynb
git commit -m "feat(fastmcp): serve main_mcp over HTTP on port 8020 in notebook 08"
```

---

## Task 8: Section 6 — Tool-schema converters and LLM client

**Files:**
- Modify: `fastmcp/08_fastmcp_real_llm_end_to_end.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 6. Wire the LLM to the MCP Tools

The LLM doesn't speak MCP natively. It speaks its provider's tool-schema dialect: Anthropic wants `{"name": ..., "description": ..., "input_schema": {...}}` per tool; OpenAI wants `{"type": "function", "function": {"name": ..., "description": ..., "parameters": {...}}}` per tool. So we need a **converter**: given the list of FastMCP tools from `await client.list_tools()`, emit a list of dicts the LLM SDK accepts.

Two small helpers do the conversion — one per provider. Both read the same fields off the FastMCP `Tool` object: `name`, `description`, and the input JSON-Schema (FastMCP exposes this as `input_schema` on the tool object in 2.x; if your installed version exposes it under a different attribute such as `inputSchema` or `schema`, adapt accordingly).

We then instantiate the LLM client we decided on back in Section 1 (`LLM_PROVIDER` and `LLM_MODEL`). The actual tool-use loop lives in Section 7.

> **API note:** FastMCP `Tool` objects in 2.x expose the input JSON Schema as `.input_schema`. If the installed version uses a different attribute name, adapt the two converter helpers below — the LLM-side shape is fixed by the SDK, so adapt the MCP-side accessor only.
```

- [ ] **Step 2: Add the converters + client-instantiation code cell**

```python
MCP_URL = "http://127.0.0.1:8020/mcp/"


def _tool_input_schema(tool) -> dict:
    """Extract the input JSON Schema from a FastMCP Tool object.

    FastMCP 2.x exposes this as `.input_schema`. Older builds used `.inputSchema`.
    Default to an empty object schema if neither attribute is present.
    """
    return (
        getattr(tool, "input_schema", None)
        or getattr(tool, "inputSchema", None)
        or {"type": "object", "properties": {}}
    )


def mcp_tools_to_anthropic_format(tools) -> list[dict]:
    """Convert a list of FastMCP Tool objects to Anthropic's tools format."""
    return [
        {
            "name": t.name,
            "description": t.description or "",
            "input_schema": _tool_input_schema(t),
        }
        for t in tools
    ]


def mcp_tools_to_openai_format(tools) -> list[dict]:
    """Convert a list of FastMCP Tool objects to OpenAI Chat Completions tools format."""
    return [
        {
            "type": "function",
            "function": {
                "name": t.name,
                "description": t.description or "",
                "parameters": _tool_input_schema(t),
            },
        }
        for t in tools
    ]


if LLM_PROVIDER == "anthropic":
    from anthropic import Anthropic
    llm = Anthropic()
else:
    from openai import OpenAI
    llm = OpenAI()


async def fetch_and_convert_tools() -> tuple[list, list[dict]]:
    """Connect to main_mcp over HTTP, list its tools, and convert them to the
    selected provider's format. Returns (raw_tools, llm_tools)."""
    async with Client(MCP_URL) as client:
        raw_tools = await client.list_tools()

    if LLM_PROVIDER == "anthropic":
        llm_tools = mcp_tools_to_anthropic_format(raw_tools)
    else:
        llm_tools = mcp_tools_to_openai_format(raw_tools)
    return raw_tools, llm_tools


raw_tools, llm_tools = await fetch_and_convert_tools()
print(f"Converted {len(raw_tools)} MCP tools into {LLM_PROVIDER} format.")
print(f"First converted tool:\n{json.dumps(llm_tools[0], indent=2)}")
```

- [ ] **Step 3: Run the cell and verify**

Expected output (Anthropic path, exact tool name depending on mount-prefix behavior):

```
Converted 3 MCP tools into anthropic format.
First converted tool:
{
  "name": "research_lookup_topic",
  "description": "Look up canned facts about a well-known topic. ...",
  "input_schema": {
    "type": "object",
    "properties": {
      "topic": {
        "title": "Topic",
        "type": "string"
      }
    },
    "required": [
      "topic"
    ]
  }
}
```

OpenAI path (when the fallback is active) prints the same shape under `{"type": "function", "function": {...}}`. Exact field nesting inside `input_schema` / `parameters` (e.g. `title`, `required`) is FastMCP-generated and will vary with the FastMCP version. Verify only:

- `Converted 3 MCP tools into <provider> format.`
- The first tool's JSON has a `name`, a non-empty `description`, and an object schema with a `topic` property of type `string`.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/08_fastmcp_real_llm_end_to_end.ipynb
git commit -m "feat(fastmcp): add tool-schema converters and LLM client in notebook 08"
```

---

## Task 9: Section 7 — Run the loop

**Files:**
- Modify: `fastmcp/08_fastmcp_real_llm_end_to_end.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 7. Run the Loop

This is the payoff cell. Given a prompt — `"Research octopuses, then give me the result as a 3-bullet summary."` — the LLM:

1. Receives the user message + the three converted tool definitions.
2. Returns a response with a `tool_use` block (Anthropic) or a `tool_calls` array (OpenAI), naming the tool and its arguments.
3. We execute the tool via `await mcp_client.call_tool(name, arguments)`, get text back from the MCP server, and append a `tool_result` block (Anthropic) or a `role="tool"` message (OpenAI) with that text.
4. We send the conversation back to the model. If it returns another `tool_use` block, repeat. If it returns plain text (`stop_reason != "tool_use"` for Anthropic, no `tool_calls` for OpenAI), we have our final answer.

We bound the loop at `max_turns=8` so a misbehaving model can't spin forever. The loop also prints each tool call as it happens so the trace is visible.

> **Quirk:** The MCP client used here is the **HTTP** `Client(MCP_URL)`, not the in-memory one. The LLM is talking to MCP over the wire, exactly as a deployed agent would. The closing in-memory test in Section 8 demonstrates the *other* mode — same server object, no socket, used for unit tests.
```

- [ ] **Step 2: Add the loop code cell**

```python
USER_PROMPT = "Research octopuses, then give me the result as a 3-bullet summary."


async def run_anthropic_loop(user_prompt: str, max_turns: int = 8) -> str:
    """Drive Claude through MCP tools until it returns plain text."""
    messages: list[dict] = [{"role": "user", "content": user_prompt}]

    async with Client(MCP_URL) as mcp_client:
        for turn in range(max_turns):
            response = llm.messages.create(
                model=LLM_MODEL,
                max_tokens=2048,
                tools=llm_tools,
                messages=messages,
            )

            # If the model is done (no more tool_use blocks), gather plain text and return.
            if response.stop_reason != "tool_use":
                final = "".join(b.text for b in response.content if getattr(b, "type", None) == "text")
                return final

            # Otherwise, append the assistant's full content (preserving tool_use blocks)
            # and execute each tool_use, appending tool_result blocks in a single user turn.
            messages.append({"role": "assistant", "content": response.content})

            tool_result_blocks: list[dict] = []
            for block in response.content:
                if getattr(block, "type", None) != "tool_use":
                    continue
                print(f"  [turn {turn}] tool_use -> {block.name}({block.input})")
                mcp_result = await mcp_client.call_tool(block.name, block.input)
                # FastMCP's call_tool returns an object with `.data` (the structured payload)
                # and a `.content` list of TextContent/ImageContent. Use whichever your version
                # exposes; `.data` for primitives and `.content[0].text` for text is the typical pair.
                if hasattr(mcp_result, "data") and mcp_result.data is not None:
                    result_text = str(mcp_result.data)
                elif getattr(mcp_result, "content", None):
                    result_text = "".join(getattr(p, "text", "") for p in mcp_result.content)
                else:
                    result_text = str(mcp_result)
                tool_result_blocks.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result_text,
                })

            messages.append({"role": "user", "content": tool_result_blocks})

        raise RuntimeError(f"Tool-use loop did not converge in {max_turns} turns.")


async def run_openai_loop(user_prompt: str, max_turns: int = 8) -> str:
    """Drive OpenAI through MCP tools until it returns plain text."""
    messages: list[dict] = [{"role": "user", "content": user_prompt}]

    async with Client(MCP_URL) as mcp_client:
        for turn in range(max_turns):
            response = llm.chat.completions.create(
                model=LLM_MODEL,
                tools=llm_tools,
                messages=messages,
            )
            choice = response.choices[0]
            msg = choice.message

            # If the model returned plain text, we're done.
            if not getattr(msg, "tool_calls", None):
                return msg.content or ""

            # Otherwise: append the assistant message (with tool_calls) and a tool reply per call.
            messages.append({
                "role": "assistant",
                "content": msg.content or "",
                "tool_calls": [
                    {
                        "id": tc.id,
                        "type": "function",
                        "function": {"name": tc.function.name, "arguments": tc.function.arguments},
                    }
                    for tc in msg.tool_calls
                ],
            })

            for tc in msg.tool_calls:
                args = json.loads(tc.function.arguments or "{}")
                print(f"  [turn {turn}] tool_call -> {tc.function.name}({args})")
                mcp_result = await mcp_client.call_tool(tc.function.name, args)
                if hasattr(mcp_result, "data") and mcp_result.data is not None:
                    result_text = str(mcp_result.data)
                elif getattr(mcp_result, "content", None):
                    result_text = "".join(getattr(p, "text", "") for p in mcp_result.content)
                else:
                    result_text = str(mcp_result)
                messages.append({
                    "role": "tool",
                    "tool_call_id": tc.id,
                    "content": result_text,
                })

        raise RuntimeError(f"Tool-use loop did not converge in {max_turns} turns.")


if LLM_PROVIDER == "anthropic":
    final_answer = await run_anthropic_loop(USER_PROMPT)
else:
    final_answer = await run_openai_loop(USER_PROMPT)

print()
print("=" * 60)
print("FINAL ANSWER:")
print("=" * 60)
print(final_answer)
```

- [ ] **Step 3: Run the cell and verify**

Expected output **shape** (exact wording from the LLM will vary across runs — verify structurally, not literally):

```
  [turn 0] tool_use -> research_lookup_topic({'topic': 'octopus'})
  [turn 1] tool_use -> writer_format_as_bullet_list({'text': '...'})

============================================================
FINAL ANSWER:
============================================================
- Octopuses have three hearts.
- They can change color in under a second.
- Each of their arms has its own neural cluster.
```

Verify the output **contains the word `octopus` (or `octopuses`)** somewhere in the final answer, **at least one `tool_use`/`tool_call` line** was printed, and the final answer **renders as approximately three bullet points or three short factual sentences**. Exact bullet wording and ordering will vary by run; the LLM may also use `make_title` if it decides to add a heading. Both of these are acceptable.

Common variations that are still considered passing:

- The model calls `lookup_topic` directly (without the `research_` prefix) if the installed FastMCP version does not prefix mounted tool names.
- The model calls only `lookup_topic` and skips `format_as_bullet_list`, returning the bullet list itself instead. As long as the final answer contains `octopus` content and is roughly bulleted, this is fine.
- The model may produce 3 to 5 bullets.

If the loop raises `RuntimeError("Tool-use loop did not converge in 8 turns.")`, the model is misbehaving — re-run the cell. If the same prompt fails twice in a row, lower the temperature (Anthropic: pass `temperature=0.2` to `llm.messages.create`; OpenAI: pass `temperature=0.2` to `chat.completions.create`).

- [ ] **Step 4: Commit**

```bash
git add fastmcp/08_fastmcp_real_llm_end_to_end.ipynb
git commit -m "feat(fastmcp): run real LLM tool-use loop end-to-end in notebook 08"
```

---

## Task 10: Section 8 — Unit-test the server with the in-memory client

**Files:**
- Modify: `fastmcp/08_fastmcp_real_llm_end_to_end.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 8. Unit-Test the Server with the In-Memory Client

The same `main_mcp` we just drove from a real LLM over HTTP is also drivable in-process via `Client(main_mcp)` — pass the server object, not a URL. No socket, no thread, no port. This is the FastMCP feature that makes server code unit-testable without any of the integration ceremony you'd normally need to test a network service.

Below is a pytest-style async test function: `test_lookup_topic_returns_facts()`. It uses `Client(main_mcp)` to call `lookup_topic("octopus")` (or its prefixed name) and asserts the result contains the substring `"three hearts"`. We invoke it directly from a cell — no `pytest` CLI invocation needed. If the assertion holds we print `PASS`; if it doesn't, the cell raises and the failure is visible inline.

The structure of this function is exactly what you'd put in a `tests/test_research.py` file under pytest in a real project. The notebook just calls it directly because notebooks can `await` top-level.
```

- [ ] **Step 2: Add the test code cell**

```python
async def test_lookup_topic_returns_facts() -> None:
    """In-memory unit test for the research server through main_mcp.

    Uses Client(main_mcp) — no socket, no thread. The same server code we drove
    from a real LLM over HTTP is also drivable in-process for tests.
    """
    async with Client(main_mcp) as client:
        tools = await client.list_tools()
        tool_names = [t.name for t in tools]

        # Find whichever name the installed FastMCP version exposes for lookup_topic.
        # Some versions prefix mounted tools (research_lookup_topic), others don't.
        candidates = [n for n in tool_names if "lookup_topic" in n]
        assert candidates, (
            f"Expected a tool whose name contains 'lookup_topic' in the mounted server. "
            f"Got: {tool_names}"
        )
        lookup_name = candidates[0]

        result = await client.call_tool(lookup_name, {"topic": "octopus"})

        # FastMCP's call_tool result exposes the structured payload as `.data`
        # for primitives, and text via `.content[0].text`. Try both for portability.
        if hasattr(result, "data") and result.data is not None:
            text = str(result.data)
        elif getattr(result, "content", None):
            text = "".join(getattr(p, "text", "") for p in result.content)
        else:
            text = str(result)

        assert isinstance(text, str), f"Expected str result, got {type(text).__name__}"
        assert "three hearts" in text.lower(), (
            f"Expected the canned 'three hearts' fact in the response, got: {text!r}"
        )


await test_lookup_topic_returns_facts()
print("PASS")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
PASS
```

If the assertion `"three hearts" in text.lower()` fails, the cell raises `AssertionError` with the actual returned text — that means either the server's `lookup_topic` was edited away from the canned facts, or the result-extraction branch did not pick up the right attribute. Inspect `result` (`print(repr(result))` before the asserts) to see what attribute holds the text in your installed FastMCP version, and adjust the extraction accordingly.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/08_fastmcp_real_llm_end_to_end.ipynb
git commit -m "feat(fastmcp): add in-memory unit test for lookup_topic in notebook 08"
```

---

## Task 11: Add "What you just learned"

**Files:**
- Modify: `fastmcp/08_fastmcp_real_llm_end_to_end.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## What you just learned

- How to compose two FastMCP apps under a single primary server with `main_mcp.mount(prefix=..., server=...)`. One HTTP endpoint, tools from both underlying servers.
- How to drive a real LLM through MCP: list the tools from the MCP client, convert them to the provider's tool format (`mcp_tools_to_anthropic_format` / `mcp_tools_to_openai_format`), and pass the converted list to `messages.create` (Anthropic) or `chat.completions.create` (OpenAI).
- The shape of the tool-use loop: send messages, on `tool_use` (or `tool_calls`) execute each tool via `mcp_client.call_tool(name, args)`, append the result as a `tool_result` block / `role="tool"` message, send again, repeat until plain text comes back.
- That the same `main_mcp` server is drivable two ways: over HTTP via `Client("http://127.0.0.1:8020/mcp/")` for the LLM-facing path, and in-memory via `Client(main_mcp)` for unit tests. Same server code, no special test scaffolding.
- That a pytest-style async test function works inline in a notebook — no `pytest` CLI needed; just `await` it from a cell.
```

- [ ] **Step 2: Commit**

```bash
git add fastmcp/08_fastmcp_real_llm_end_to_end.ipynb
git commit -m "feat(fastmcp): add 'what you just learned' recap to notebook 08"
```

---

## Task 12: Add "What FastMCP bought us" closing reflection

**Files:**
- Modify: `fastmcp/08_fastmcp_real_llm_end_to_end.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## What FastMCP bought us

Eight notebooks. One framework. Here is what we did **not** write by hand at any point, compared to the raw-SDK baseline we built in notebook 01:

- **JSON Schemas.** Every tool's input schema was generated by FastMCP from the function's type hints. In the raw SDK we hand-wrote `inputSchema` for every tool, every time.
- **Tool dispatch.** The raw SDK uses `@server.list_tools()` and `@server.call_tool()` decorators on hand-written dispatcher functions that case-switch on tool name. FastMCP collapses that into one `@mcp.tool` decorator per tool.
- **Transport switching.** Same `FastMCP` instance over stdio (notebook 02) and `streamable-http` (notebooks 03–08). One runtime flag, no code rewrites. The raw SDK requires different runner setup per transport.
- **Authentication.** Notebook 04 added bearer-token and OAuth2 authentication via a single `auth=` constructor parameter on `FastMCP`. The raw SDK would have required middleware wired by hand on the underlying transport.
- **Composition.** `main_mcp.mount(...)` exposed two sub-servers as one in this notebook. There is no equivalent in the raw SDK — composition is something you'd hand-build by routing tool calls yourself.
- **Pydantic-native validation.** Notebook 06's input-validation patterns used Pydantic models directly as tool parameters; FastMCP both validates inputs and generates the schema from the same model. The raw SDK has neither integration.
- **Resources and prompts.** Notebook 07's `@mcp.resource("notes://{topic}")` parameterizes URIs as function arguments automatically; same for `@mcp.prompt`. The raw SDK exposes these as separate handler decorators with hand-written URI parsing.
- **In-memory client for tests.** Notebook 02 and this notebook's Section 8 both used `Client(server_instance)` to drive a server in-process with no socket, no subprocess. The raw SDK has no equivalent — testing requires a real stdio or HTTP round-trip.

The pedagogical bet of this series was: **build each scenario twice — once raw, once with FastMCP — and the second one is much shorter for a reason worth understanding.** The reason is everything in the list above.

### Where to go from here

- **Real LLMs across more domains.** This notebook proved one model + two composed servers; the same shape scales to ten servers + larger toolsets.
- **Production deployment.** TLS, real OAuth providers, observability, fan-out, message-queue-backed task stores. FastMCP exposes the extension points; the FastMCP docs walk through deployment.
- **A2A comparison.** A2A is for agent-to-agent communication; MCP is for agent-to-tool. Both can coexist on the same agent. See the `a2a/` series in this repo for the parallel teaching arc.
- **Cross-language interop.** MCP clients exist for JS, Go, and others; a FastMCP server serves them all over the same wire format.
```

- [ ] **Step 2: Commit**

```bash
git add fastmcp/08_fastmcp_real_llm_end_to_end.ipynb
git commit -m "feat(fastmcp): add 'what FastMCP bought us' closing reflection to notebook 08"
```

---

## Task 13: Section 9 — Cleanup

**Files:**
- Modify: `fastmcp/08_fastmcp_real_llm_end_to_end.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 9. Cleanup

Same daemon-thread cleanup story as notebooks 03–07. `mcp.run(...)` blocks the thread it's called on; that thread is a daemon; daemon threads are killed automatically when the Python process exits. Restart the kernel (or close the notebook) and port 8020 is released. We confirm the thread is still alive below — it should be, until the kernel exits — and that is the whole story.

If you want to free port 8020 right now without restarting the kernel, the simplest move is **Kernel → Restart**. There is no clean per-server shutdown API surfaced here in FastMCP 2.x.
```

- [ ] **Step 2: Add the cleanup code cell**

```python
print(f"Server thread still running: {server_thread.is_alive()}")
print("(The daemon thread will exit when the kernel restarts; no explicit shutdown call is needed in 2.x.)")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Server thread still running: True
(The daemon thread will exit when the kernel restarts; no explicit shutdown call is needed in 2.x.)
```

- [ ] **Step 4: Commit**

```bash
git add fastmcp/08_fastmcp_real_llm_end_to_end.ipynb
git commit -m "feat(fastmcp): add cleanup section to notebook 08"
```

---

## Task 14: Sanity-check the full cell sequence

**Files:** none (verification only).

- [ ] **Step 1: List cells and confirm the order**

```bash
python -c "import json; nb = json.load(open('fastmcp/08_fastmcp_real_llm_end_to_end.ipynb')); print(len(nb['cells']), 'cells'); [print(i, c['cell_type'], (c['source'][0] if c['source'] else '<empty>').rstrip()[:80]) for i, c in enumerate(nb['cells'])]"
```

Expected: the cell list reads in this order (cell types in parentheses):

1. (markdown) `# 08 — Real LLM End-to-End: Composed FastMCP Servers Driving a Tool-Use Loop`
2. (markdown) `## What you'll learn`
3. (markdown) `## 1. Setup`
4. (code) `import asyncio`
5. (markdown) `## 2. Build the Research Server`
6. (code) `research_mcp = FastMCP("research")`
7. (markdown) `## 3. Build the Writer Server`
8. (code) `writer_mcp = FastMCP("writer")`
9. (markdown) `## 4. Mount Them Under a Single Main Server`
10. (code) `main_mcp = FastMCP("main")`
11. (markdown) `## 5. Serve over HTTP`
12. (code) `server_thread = start_http_server_in_thread(main_mcp, ...)`
13. (markdown) `## 6. Wire the LLM to the MCP Tools`
14. (code) `MCP_URL = "http://127.0.0.1:8020/mcp/"`
15. (markdown) `## 7. Run the Loop`
16. (code) `USER_PROMPT = "Research octopuses, ...`
17. (markdown) `## 8. Unit-Test the Server with the In-Memory Client`
18. (code) `async def test_lookup_topic_returns_facts():`
19. (markdown) `## What you just learned`
20. (markdown) `## What FastMCP bought us`
21. (markdown) `## 9. Cleanup`
22. (code) `print(f"Server thread still running: ...`

Total: 22 cells.

- [ ] **Step 2: Confirm no stray placeholder strings**

```bash
grep -nE 'TODO|FIXME|XXX|placeholder' fastmcp/08_fastmcp_real_llm_end_to_end.ipynb || echo "OK no placeholders"
```

Expected: `OK no placeholders`.

- [ ] **Step 3: No commit needed (verification only).** If a structural issue is found, fix it in the appropriate prior task and re-run this task.

---

## Task 15: Confirm an LLM API key is set in the execution environment

**Files:** none (verification only).

- [ ] **Step 1: Confirm at least one LLM API key is available**

```bash
python -c "import os; ak=bool(os.getenv('ANTHROPIC_API_KEY')); ok=bool(os.getenv('OPENAI_API_KEY')); print('ANTHROPIC_API_KEY set:', ak); print('OPENAI_API_KEY set:', ok); assert ak or ok, 'Set one before running the notebook.'"
```

Expected: at least one of the two reads as `True`, and the assertion does not fire.

If neither is set, surface as NEEDS_CONTEXT — the executing user needs to provide a key. Do **not** attempt to source a key from anywhere else; this notebook is explicitly opt-in for API costs.

- [ ] **Step 2: No commit needed (verification only).**

---

## Task 16: Confirm the LLM SDK matching the available key is installed

**Files:** none (verification only).

- [ ] **Step 1: Confirm the relevant SDK is importable**

```bash
python -c "
import os, importlib.util
ak = bool(os.getenv('ANTHROPIC_API_KEY'))
ok = bool(os.getenv('OPENAI_API_KEY'))
if ak:
    spec = importlib.util.find_spec('anthropic')
    assert spec is not None, 'ANTHROPIC_API_KEY is set but the anthropic package is missing. pip install anthropic.'
    print('anthropic SDK present.')
elif ok:
    spec = importlib.util.find_spec('openai')
    assert spec is not None, 'OPENAI_API_KEY is set but the openai package is missing. pip install openai.'
    print('openai SDK present.')
"
```

Expected: prints `anthropic SDK present.` or `openai SDK present.` matching whichever key is set, with no assertion error.

If the assertion fires, install the missing SDK per the message and re-run. (The plan does **not** install anything automatically — installs are an environment concern.)

- [ ] **Step 2: No commit needed (verification only).**

---

## Task 17: End-to-end verification in a fresh kernel

**Files:** none (verification only).

- [ ] **Step 1: Execute the notebook top-to-bottom in a clean kernel**

```bash
jupyter nbconvert --to notebook --execute --inplace fastmcp/08_fastmcp_real_llm_end_to_end.ipynb
```

Expected: succeeds with no errors. The notebook is rewritten with embedded outputs from the fresh run. This will make at least one real LLM API call — the loop typically converges in 1–3 turns.

- [ ] **Step 2: Verify expected outputs in the executed notebook**

Cells should contain, in order:

1. `Setup OK. LLM provider = 'anthropic', model = 'claude-sonnet-4-6'` (or the OpenAI fallback line).
2. `Research server defined: 'research' with 1 tool (lookup_topic)`
3. `Writer server defined: 'writer' with 2 tools (format_as_bullet_list, make_title)`
4. `main_mcp exposes 3 tool(s): [...]` — three tools, names matching `lookup_topic` / `format_as_bullet_list` / `make_title` (possibly prefixed).
5. `Server thread alive: True` followed by `main_mcp HTTP server should be listening on http://127.0.0.1:8020/mcp/`
6. `Converted 3 MCP tools into <provider> format.` followed by a JSON dump of the first tool whose schema contains a `topic` string property.
7. At least one `[turn N] tool_use -> ...` (or `tool_call -> ...`) line, then `FINAL ANSWER:`, then a final answer that mentions octopus content and renders as roughly 3 bullet points or short factual sentences.
8. `PASS` from the in-memory unit test.
9. `Server thread still running: True` followed by the daemon-thread cleanup note.

No cell raises an unhandled exception. **Exact wording of the LLM's final answer will vary across runs — verify structurally (octopus content + ~3 bullets), not literally.**

If a port-collision error appears, free port 8020 first: `lsof -ti tcp:8020 | xargs kill -9` (only kill processes you recognize), then re-run from Task 17 Step 1.

If the tool-use loop raises `RuntimeError("Tool-use loop did not converge in 8 turns.")`, re-run the cell. If the same prompt fails twice in a row, edit the loop cell to pass `temperature=0.2` to the LLM `create` call (per the guidance in Task 9 Step 3) and re-execute.

If a FastMCP API-shape error fires (e.g. `mount()` keyword mismatch, `Tool.input_schema` missing), follow the API-note guidance inside the relevant task cell. These are expected drift points across FastMCP 2.x minor versions and adapting them is part of the implementer's job; they are **not** failures.

- [ ] **Step 3: Confirm port is still bound (server is daemon-threaded so it stays up until kernel exit)**

```bash
lsof -iTCP:8020 -sTCP:LISTEN
```

Expected: one entry showing the Python process owning the port — this is the daemon thread still serving inside the notebook kernel. (Same pattern as notebooks 03–07; the port is released at kernel exit.)

- [ ] **Step 4: Commit the clean run**

```bash
git add fastmcp/08_fastmcp_real_llm_end_to_end.ipynb
git commit -m "chore(fastmcp): commit clean fresh-kernel run of notebook 08"
```

---

## Done

After Task 17 passes, notebook 08 is complete — and **the FastMCP learning series is complete**: eight notebooks from "the pain of MCP without FastMCP" (nb01) through "a real LLM driving composed FastMCP servers end-to-end, with an in-memory unit test for the same server code" (this notebook). No follow-on plan needed. The series stands alone.
