# FastMCP Notebook 02 — `02_fastmcp_local_stdio.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the second notebook in the FastMCP learning series — introduce the **FastMCP primitive** (`from fastmcp import FastMCP`), the `@mcp.tool` decorator that turns plain Python functions into MCP tools (with type hints generating the JSON schema and docstrings generating the description), and the **in-memory `Client(mcp)`** that exercises the server from inside the same Python process. The notebook recreates the two tools from notebook 01 (`add` and `greet`) so the diff with the raw SDK is obvious, and ends with a short markdown-only explanation of `mcp.run(transport="stdio")` for the CLI/subprocess use case (not actually run from the notebook).

**Architecture:** A single Jupyter notebook that defines one `FastMCP("demo")` server with two `@mcp.tool` functions. The notebook then plays the role of an MCP *client* using the in-memory transport: `async with Client(mcp) as client:` to call `list_tools()` and `call_tool(...)`. No HTTP, no subprocess, no API keys.

**Tech Stack:** Python 3.11+, Jupyter, `fastmcp` (2.x line). Top-level `await` is used directly inside notebook cells (Jupyter supports it).

**Companion spec:** `docs/superpowers/specs/2026-05-18-fastmcp-learning-series-design.md` (notebook 02 section).

**Port assignment:** none — this notebook uses only the in-memory `Client(mcp)` transport. Notebook 03 is where ports come in.

---

## File Structure

- **Create:** `fastmcp/02_fastmcp_local_stdio.ipynb` — the entire notebook, self-contained (no imports from notebook 01).
- **Modify:** none.
- **No separate test files** — verification is the notebook running top-to-bottom in a fresh kernel (Task 12).

The `fastmcp/` folder is created by the notebook-01 plan, but Task 1 below does **not** assume it already exists — it includes a defensive `mkdir -p fastmcp` step.

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown)
2. **"What you'll learn"** (markdown bullets)
3. **Section 1: Setup** (code — imports only)
4. **Section 2: Define a FastMCP server with one tool** (markdown + 1 code cell: `mcp = FastMCP("demo")` + `@mcp.tool` `add`)
5. **Section 3: Call it with the in-memory client** (markdown + 1 code cell: `async with Client(mcp)` + `list_tools` + `call_tool`)
6. **Section 4: Inspect the auto-generated schema** (markdown + 1 code cell: pretty-print the JSON schema FastMCP generated)
7. **Section 5: Add a second tool — see how cheap that just got** (markdown + 2 code cells: register `greet`, then call it)
8. **Section 6: About stdio** (markdown only — explains `mcp.run(transport="stdio")` without running it)
9. **"What you just learned"** (markdown)
10. **"What's missing"** (markdown — teases notebook 03, HTTP transport)

---

## Task 1: Create the notebook scaffold

**Files:**
- Create: `fastmcp/02_fastmcp_local_stdio.ipynb`

- [ ] **Step 1: Ensure the `fastmcp/` folder exists**

```bash
mkdir -p fastmcp
ls -la fastmcp/
```

Expected: the directory exists (it should already, from the notebook-01 plan, but this is idempotent).

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
python -c "import json; json.load(open('fastmcp/02_fastmcp_local_stdio.ipynb'))"
ls -la fastmcp/02_fastmcp_local_stdio.ipynb
```

Expected: no exception, file non-zero size.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/02_fastmcp_local_stdio.ipynb
git commit -m "feat(fastmcp): scaffold 02_fastmcp_local_stdio.ipynb"
```

---

## Task 2: Add title and "Why this notebook exists"

**Files:**
- Modify: `fastmcp/02_fastmcp_local_stdio.ipynb` (add cell 1)

- [ ] **Step 1: Add a markdown cell with title and motivation**

Cell content (markdown):

```markdown
# 02 — Minimum Viable FastMCP

## Why this notebook exists

In **`mcp/02`** and in **notebook 01 above**, we built an MCP server using the raw `mcp` SDK. The result worked, but every tool cost us a hand-written JSON schema, a manual dispatch branch in `call_tool`, and explicit stdio plumbing. Two tools turned into a noticeable pile of boilerplate. The closing note of notebook 01 was: *"every tool means another schema by hand — there should be a framework."*

This notebook introduces that framework: **FastMCP**.

We'll build the same two tools from notebook 01 (`add` and `greet`) — but this time the type hints generate the JSON schema, the docstrings generate the tool description, and a single decorator (`@mcp.tool`) registers each tool. We'll also introduce FastMCP's **in-memory `Client`**, which lets us exercise the server from inside the same Python process — no subprocess, no socket, no transport at all. That client is what we'll lean on throughout the rest of the series for inline examples and (in notebook 08) unit tests.
```

- [ ] **Step 2: Verify the cell renders**

Open the notebook in Jupyter/VS Code and confirm the cell renders as markdown.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/02_fastmcp_local_stdio.ipynb
git commit -m "feat(fastmcp): add title and motivation to notebook 02"
```

---

## Task 3: Add "What you'll learn"

**Files:**
- Modify: `fastmcp/02_fastmcp_local_stdio.ipynb` (add cell 2)

- [ ] **Step 1: Add a markdown cell with "What you'll learn"**

Cell content (markdown):

```markdown
## What you'll learn

- How to create a FastMCP server: `from fastmcp import FastMCP; mcp = FastMCP("demo")`.
- How `@mcp.tool` turns a plain Python function into an MCP tool — with type hints driving the JSON schema and the docstring driving the tool description.
- How to exercise a FastMCP server from the same Python process using the in-memory `Client(mcp)` — no subprocess, no transport.
- How to list tools and call a tool by name with `await client.list_tools()` / `await client.call_tool(...)`.
- Why the auto-generated schema is the central convenience FastMCP buys you: adding a second tool costs nearly nothing.
- What `mcp.run(transport="stdio")` is for (CLI/subprocess deployment) and why we don't run it from the notebook.
```

- [ ] **Step 2: Verify the cell renders**

Open the notebook and confirm the bullet list renders.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/02_fastmcp_local_stdio.ipynb
git commit -m "feat(fastmcp): add 'What you'll learn' to notebook 02"
```

---

## Task 4: Section 1 — Setup (imports)

**Files:**
- Modify: `fastmcp/02_fastmcp_local_stdio.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 1. Setup

We only need two names from `fastmcp`: `FastMCP` (the server class) and `Client` (the in-memory client we'll use to exercise it). FastMCP exposes both from the package root.

If the import fails with `ModuleNotFoundError`, install FastMCP in your active environment:

```
pip install fastmcp
```

This notebook targets the current stable `fastmcp` 2.x line.
```

- [ ] **Step 2: Add the setup code cell**

```python
import json

from fastmcp import Client, FastMCP

print("Setup OK")
```

- [ ] **Step 3: Run the cell**

Expected output:

```
Setup OK
```

If imports fail, run `pip install fastmcp` and re-execute.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/02_fastmcp_local_stdio.ipynb
git commit -m "feat(fastmcp): add setup cell to notebook 02"
```

---

## Task 5: Section 2 — Define a FastMCP server with one tool

**Files:**
- Modify: `fastmcp/02_fastmcp_local_stdio.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 2. Define a FastMCP Server with One Tool

A FastMCP server is a single object: `FastMCP("server-name")`. That object becomes the registration point for everything the server exposes — tools (now), resources and prompts (notebook 07), authentication (notebook 04), and so on.

A **tool** is just a Python function decorated with `@mcp.tool`. FastMCP reads the function's type hints to build the JSON schema for its inputs, and reads the docstring to populate the tool's description. We don't write either by hand.

> **Quirk:** Compare this with notebook 01, where we wrote an `inputSchema` dict for every tool. In FastMCP the type hints **are** the schema — `a: int, b: int` becomes a JSON Schema requiring two integer properties. The docstring **is** the description — what the client sees when it lists tools.

We'll start with one tool, `add`, mirroring notebook 01. (No-parens form `@mcp.tool` is the idiomatic FastMCP 2.x usage; `@mcp.tool()` works too.)
```

- [ ] **Step 2: Add the server-definition code cell**

```python
mcp = FastMCP("demo")


@mcp.tool
def add(a: int, b: int) -> int:
    """Add two integers and return the sum."""
    return a + b


print(f"Server: {mcp.name}")
print(f"Defined tool: add")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Server: demo
Defined tool: add
```

- [ ] **Step 4: Commit**

```bash
git add fastmcp/02_fastmcp_local_stdio.ipynb
git commit -m "feat(fastmcp): define FastMCP server with add tool in notebook 02"
```

---

## Task 6: Section 3 — Call it with the in-memory client

**Files:**
- Modify: `fastmcp/02_fastmcp_local_stdio.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 3. Call It with the In-Memory Client

To actually invoke the tool, we need an MCP **client**. FastMCP ships one: `from fastmcp import Client`. When you construct it with the server *object* — `Client(mcp)` — it talks to the server directly inside the same Python process. No subprocess. No socket. No transport.

> **Quirk:** The in-memory `Client(mcp)` is what makes FastMCP servers unit-testable. Same server code, no infrastructure required to exercise it. We'll lean on this throughout the rest of the series, and use it directly as a test harness in notebook 08.

Jupyter supports top-level `await` in cells, so we can use `async with Client(mcp) as client:` directly — no `asyncio.run(...)` wrapper required.
```

- [ ] **Step 2: Add the client-invocation code cell**

```python
async with Client(mcp) as client:
    tools = await client.list_tools()
    print(f"Tools exposed by '{mcp.name}': {[t.name for t in tools]}")

    result = await client.call_tool("add", {"a": 2, "b": 3})
    print(f"add(2, 3) -> {result.data}")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Tools exposed by 'demo': ['add']
add(2, 3) -> 5
```

- [ ] **Step 4: Commit**

```bash
git add fastmcp/02_fastmcp_local_stdio.ipynb
git commit -m "feat(fastmcp): exercise add tool via in-memory Client in notebook 02"
```

---

## Task 7: Section 4 — Inspect the auto-generated schema

**Files:**
- Modify: `fastmcp/02_fastmcp_local_stdio.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 4. Inspect the Auto-Generated Schema

The schema FastMCP generated isn't a black box — every `Tool` object the client receives carries its `inputSchema` as a regular dict, and we can pretty-print it.

Recall notebook 01: for `add` we hand-wrote something like:

```python
inputSchema={
    "type": "object",
    "properties": {
        "a": {"type": "integer"},
        "b": {"type": "integer"},
    },
    "required": ["a", "b"],
}
```

FastMCP derived an equivalent schema from the `a: int, b: int` type hints alone. Let's confirm.
```

- [ ] **Step 2: Add the schema-inspection code cell**

```python
async with Client(mcp) as client:
    tools = await client.list_tools()
    add_tool = next(t for t in tools if t.name == "add")

    print(f"name:        {add_tool.name}")
    print(f"description: {add_tool.description}")
    print("inputSchema:")
    print(json.dumps(add_tool.inputSchema, indent=2))
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
name:        add
description: Add two integers and return the sum.
inputSchema:
{
  "type": "object",
  "properties": {
    "a": {
      "type": "integer"
    },
    "b": {
      "type": "integer"
    }
  },
  "required": [
    "a",
    "b"
  ]
}
```

(Exact key order or minor schema metadata fields like `"title"` may vary slightly across `fastmcp` 2.x versions. The point is: `type: object`, two integer properties `a` and `b`, both required, and a `description` that matches the docstring verbatim.)

- [ ] **Step 4: Commit**

```bash
git add fastmcp/02_fastmcp_local_stdio.ipynb
git commit -m "feat(fastmcp): inspect auto-generated schema in notebook 02"
```

---

## Task 8: Section 5 — Add a second tool

**Files:**
- Modify: `fastmcp/02_fastmcp_local_stdio.ipynb` (add markdown header + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 5. Add a Second Tool — See How Cheap That Just Got

In notebook 01, adding a second tool meant: another `inputSchema` dict, another `if name == "greet"` branch in the dispatch handler, possibly another import. Let's add `greet` to the FastMCP server and see what it costs.

> **Quirk:** This is the whole pitch. One decorator, one function. No schema, no dispatch branch, no registration call. The cost of adding a tool drops to "write the function."
```

- [ ] **Step 2: Add the second-tool registration code cell**

```python
@mcp.tool
def greet(name: str) -> str:
    """Return a friendly greeting for the given name."""
    return f"Hello, {name}!"


print(f"Server now exposes: ", end="")
async with Client(mcp) as client:
    tools = await client.list_tools()
    print([t.name for t in tools])
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Server now exposes: ['add', 'greet']
```

- [ ] **Step 4: Add the second-tool invocation code cell**

```python
async with Client(mcp) as client:
    result = await client.call_tool("greet", {"name": "world"})
    print(f"greet('world') -> {result.data!r}")

    greet_tool = next(t for t in await client.list_tools() if t.name == "greet")
    print(f"greet.description: {greet_tool.description}")
    print(f"greet.inputSchema: {json.dumps(greet_tool.inputSchema, indent=2)}")
```

- [ ] **Step 5: Run the cell and verify**

Expected output:

```
greet('world') -> 'Hello, world!'
greet.description: Return a friendly greeting for the given name.
greet.inputSchema: {
  "type": "object",
  "properties": {
    "name": {
      "type": "string"
    }
  },
  "required": [
    "name"
  ]
}
```

(Again, minor schema-metadata fields may differ slightly across `fastmcp` 2.x versions; the core shape — one string property `name`, required — is the contract.)

- [ ] **Step 6: Commit**

```bash
git add fastmcp/02_fastmcp_local_stdio.ipynb
git commit -m "feat(fastmcp): add second tool to demonstrate near-zero cost in notebook 02"
```

---

## Task 9: Section 6 — About stdio

**Files:**
- Modify: `fastmcp/02_fastmcp_local_stdio.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 6. About stdio

So far we've used the **in-memory** transport — `Client(mcp)` passing the server object directly. That's perfect for inline examples and tests, but it's not how a *real* MCP client (Claude Desktop, Claude Code, an IDE plugin, a CI script) talks to this server. Real MCP clients spawn the server as a **subprocess** and speak the MCP protocol over its standard input and output.

The FastMCP one-liner for that is:

```python
if __name__ == "__main__":
    mcp.run(transport="stdio")
```

You'd put that at the bottom of a `.py` script (say, `demo_server.py`), and then a host MCP client would launch it with something like `python demo_server.py` and pipe the protocol over stdio. We don't run it from this notebook because doing so would block the notebook's kernel — `mcp.run(transport="stdio")` is a foreground loop that reads from the process's actual `stdin`. That's fine in a CLI deployment; it's awkward inside Jupyter.

In **notebook 03** we switch to HTTP transport, which *can* run from a notebook (on a background thread) and lets us call the server with an ordinary HTTP client. From the tool author's point of view, nothing changes: the same `@mcp.tool` functions, the same `mcp` object — just a different runtime flag.

> **Quirk:** Transports are **runtime arguments**, not code rewrites. `mcp.run(transport="stdio")` vs. `mcp.run(transport="streamable-http", port=8020)` is the entire difference between "deploy as a CLI subprocess" and "deploy as a network service." Notebook 03 makes this concrete.
```

- [ ] **Step 2: Verify the cell renders**

Open the notebook and confirm the markdown (including the fenced code block and the blockquote callout) renders cleanly.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/02_fastmcp_local_stdio.ipynb
git commit -m "feat(fastmcp): explain stdio transport in notebook 02"
```

---

## Task 10: Closing recap

**Files:**
- Modify: `fastmcp/02_fastmcp_local_stdio.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Add the "What you just learned" markdown cell**

```markdown
## What you just learned

- A FastMCP server is one object: `mcp = FastMCP("name")`. Everything attaches to it.
- `@mcp.tool` registers a plain Python function as an MCP tool. Type hints → JSON schema. Docstring → tool description. No hand-written boilerplate.
- The in-memory `Client(mcp)` exercises the server from the same process — `async with Client(mcp) as client:` then `await client.list_tools()` and `await client.call_tool(...)`. No subprocess, no socket.
- Adding a second tool is exactly as expensive as defining one function — there is no per-tool schema or dispatch cost the way there was in notebook 01.
- The runtime transport (`stdio`, `streamable-http`, etc.) is a flag passed to `mcp.run(...)`, not a code rewrite. `stdio` is the right choice for CLI/subprocess deployment with hosts like Claude Desktop.
```

- [ ] **Step 2: Verify the cell renders**

Open the notebook and confirm the bullet list renders.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/02_fastmcp_local_stdio.ipynb
git commit -m "feat(fastmcp): add closing recap to notebook 02"
```

---

## Task 11: "What's missing" teaser

**Files:**
- Modify: `fastmcp/02_fastmcp_local_stdio.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Add the "What's missing" markdown cell**

```markdown
## What's missing

We can build tools fast and call them from inside Python — but no external process has talked to our server yet. Every MCP scenario beyond a single Python process needs a real transport on the wire.

In **notebook 03** we take *the exact same* `FastMCP` instance from this notebook and serve it over HTTP via `mcp.run(transport="streamable-http", port=8020)` on a background thread. We'll drive it from an `httpx`-based MCP client, watch the same `add` and `greet` tools work unchanged across the wire, and briefly contrast `streamable-http` against the older SSE transport. Same tool code; different runtime flag.
```

- [ ] **Step 2: Verify the cell renders**

Open the notebook and confirm the markdown renders.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/02_fastmcp_local_stdio.ipynb
git commit -m "feat(fastmcp): add 'What's missing' teaser to notebook 02"
```

---

## Task 12: End-to-end verification in a fresh kernel

**Files:** none (verification only).

- [ ] **Step 1: Execute the notebook top-to-bottom in a clean kernel**

```bash
jupyter nbconvert --to notebook --execute --inplace fastmcp/02_fastmcp_local_stdio.ipynb
```

Expected: succeeds with no errors. The notebook is rewritten with embedded outputs from the fresh run.

- [ ] **Step 2: Verify expected outputs in the executed notebook**

Cells should contain, in order:

1. `Setup OK`
2. `Server: demo` followed by `Defined tool: add`
3. `Tools exposed by 'demo': ['add']` followed by `add(2, 3) -> 5`
4. The schema inspection block: `name: add`, `description: Add two integers and return the sum.`, and the pretty-printed JSON schema for `add` (object with required integer `a` and `b`).
5. `Server now exposes: ['add', 'greet']`
6. `greet('world') -> 'Hello, world!'`, then `greet.description: Return a friendly greeting for the given name.`, then the pretty-printed JSON schema for `greet` (object with required string `name`).

No cell raises an unhandled exception. If `ModuleNotFoundError: fastmcp` appears, install it (`pip install fastmcp`) and re-run.

- [ ] **Step 3: Sanity-check that no background process is left behind**

This notebook never starts a server — only the in-memory client — so there is nothing to clean up. Confirm:

```bash
ps -ef | grep -i fastmcp | grep -v grep
```

Expected: no output (no orphaned FastMCP processes from this notebook).

- [ ] **Step 4: Commit the clean run**

```bash
git add fastmcp/02_fastmcp_local_stdio.ipynb
git commit -m "chore(fastmcp): commit clean fresh-kernel run of notebook 02"
```

---

## Done

After Task 12 passes, notebook 02 is complete. The next plan to write is `2026-05-18-fastmcp-notebook-03-http-server.md`, which takes this same `FastMCP` instance and serves it over HTTP on port `8020` via a background thread, driven by an `httpx`-based MCP client.
