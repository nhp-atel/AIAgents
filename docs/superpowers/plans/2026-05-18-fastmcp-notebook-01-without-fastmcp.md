# FastMCP Notebook 01 — `01_without_fastmcp.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first notebook in the FastMCP learning series — re-implement a minimal stdio MCP server using the **raw `mcp` SDK** (NO FastMCP), with hand-written `inputSchema` JSON per tool and a manual dispatch branch per tool inside `@server.call_tool()`. The notebook defines **two** tools so the boilerplate cost becomes visible, then closes with the line that motivates the rest of the series: *"Every tool means another schema by hand, another handler dispatch branch. There should be a framework."*

**Architecture:** A single Jupyter notebook that (a) writes a raw-SDK MCP server out to a temp `.py` file, then (b) spawns it as a subprocess via `mcp.client.stdio.stdio_client(StdioServerParameters(command="python", args=[<temp.py>]))` and drives it from a `ClientSession`. No HTTP, no background server, no API keys. The temp file is deleted at the end.

**Tech Stack:** Python 3.11+, Jupyter, the official `mcp` Python SDK (`Server`, `stdio_server`, `Tool`, `TextContent`, `ClientSession`, `stdio_client`, `StdioServerParameters`), `asyncio`, `tempfile`, `pathlib`.

**Companion spec:** `docs/superpowers/specs/2026-05-18-fastmcp-learning-series-design.md` (notebook 01 section).

**Folder:** This is the **first** notebook in the series, and the `fastmcp/` folder does not yet exist. Task 1 of this plan creates the folder.

---

## File Structure

- **Create:** `fastmcp/` (folder)
- **Create:** `fastmcp/01_without_fastmcp.ipynb` — the entire notebook, self-contained.
- **Modify:** none.
- **No separate test files** — verification is the notebook running top-to-bottom in a fresh kernel (Task 10).

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown)
2. **"What you'll learn"** (markdown bullets)
3. **Section 1: Setup** (markdown + 1 code cell — imports + helper that writes a server-source string to a temp `.py` file and returns the path)
4. **Section 2: Build a one-tool server the raw way** (markdown + 1 code cell that writes a one-tool raw-SDK server to a temp file)
5. **Section 3: Run it and call its tool** (markdown + 1 code cell that spawns the temp server via `stdio_client` and calls `add`)
6. **Section 4: Add a second tool — and feel the cost** (markdown + 1 code cell that writes the two-tool version + 1 code cell that runs both tools through the same client)
7. **"What you just learned"** (markdown)
8. **"What's missing"** (markdown teaser pointing to notebook 02 — FastMCP)
9. **Cleanup** (code — delete both temp `.py` files)

---

## Task 1: Create the `fastmcp/` folder and scaffold the empty notebook

**Files:**
- Create: `fastmcp/` (folder)
- Create: `fastmcp/01_without_fastmcp.ipynb`

- [ ] **Step 1: Create the `fastmcp/` folder**

```bash
mkdir fastmcp
ls -la fastmcp
```

Expected: the folder is created and is empty.

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

Save as `fastmcp/01_without_fastmcp.ipynb`.

- [ ] **Step 3: Verify the file is valid JSON and opens as a notebook**

```bash
python -c "import json; json.load(open('fastmcp/01_without_fastmcp.ipynb'))"
ls -la fastmcp/01_without_fastmcp.ipynb
```

Expected: no exception, file non-zero size.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/01_without_fastmcp.ipynb
git commit -m "feat(fastmcp): scaffold 01_without_fastmcp.ipynb"
```

---

## Task 2: Add title and "Why this notebook exists"

**Files:**
- Modify: `fastmcp/01_without_fastmcp.ipynb` (add cell 1)

- [ ] **Step 1: Add a markdown cell with title and motivation**

Cell content (markdown):

```markdown
# 01 — Without FastMCP: The Pain

## Why this notebook exists

In **`mcp/01_without_mcp.ipynb`** we saw what happens when an AI app owns its tools directly: no protocol, no boundary, no way for any other host to reuse them. In **`mcp/02_mcp_local_stdio.ipynb`** we introduced MCP — the protocol that draws the line between *host*, *client*, and *server*. That notebook used the raw `mcp` SDK's `Server` primitive: `@server.list_tools()` to advertise tools, `@server.call_tool()` to execute them, and a hand-written JSON Schema describing every tool's inputs.

This notebook does that same thing again — on purpose — so we can measure the cost. Each tool we add to a raw-SDK server costs us **(a)** a `Tool(...)` entry in `list_tools` with a hand-written `inputSchema` dict, and **(b)** another branch in the `call_tool` dispatcher. Two tools is enough to feel the duplication. Five tools is enough to want a framework.

That framework is **FastMCP**, and starting in notebook 02 of this series we'll use it. First, the pain.
```

- [ ] **Step 2: Verify the cell renders**

Open the notebook in Jupyter/VS Code and confirm the cell renders as markdown.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/01_without_fastmcp.ipynb
git commit -m "feat(fastmcp): add intro markdown to notebook 01"
```

---

## Task 3: Add "What you'll learn"

**Files:**
- Modify: `fastmcp/01_without_fastmcp.ipynb` (add cell 2)

- [ ] **Step 1: Add a markdown cell**

Cell content (markdown):

```markdown
## What you'll learn

- How to build an MCP server with the **raw `mcp` Python SDK** — `Server`, `@server.list_tools()`, `@server.call_tool()`, and `stdio_server`.
- How to write an `inputSchema` JSON Schema by hand for each tool.
- How to spawn a stdio MCP server as a subprocess from Python using `stdio_client` and a `ClientSession`, list its tools, and call them.
- Exactly which lines you have to add **per new tool** when no framework is helping you — so you know what FastMCP is replacing in notebook 02.
```

- [ ] **Step 2: Verify the cell renders**

The bullets should render properly.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/01_without_fastmcp.ipynb
git commit -m "feat(fastmcp): add 'What you'll learn' to notebook 01"
```

---

## Task 4: Section 1 — Setup (imports + temp-file helper)

**Files:**
- Modify: `fastmcp/01_without_fastmcp.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 1. Setup

A stdio MCP server expects to own its process's stdin and stdout — it reads JSON-RPC frames from one and writes frames to the other. That's awkward to do inside a Jupyter cell, where stdin/stdout already belong to the kernel.

The pragmatic workaround — and the way real MCP servers are actually run — is: **write the server code out to a `.py` file, then spawn it as a subprocess.** The `mcp` SDK's `stdio_client` does exactly that and hands us back a connected `ClientSession`. We use `tempfile` so the file is in a known scratch location we can delete at the end.

This cell defines a small helper, `write_server_to_temp(...)`, that takes a source string and returns the temp path. Each section uses it to materialize a server we'll then spawn.
```

- [ ] **Step 2: Add the setup code cell**

```python
import asyncio
import tempfile
from pathlib import Path

from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

# Track every temp server file we create so the cleanup cell at the bottom
# can delete them. Each entry is a pathlib.Path.
_temp_server_files: list[Path] = []


def write_server_to_temp(source: str, label: str) -> Path:
    """Write `source` to a temp .py file and remember it for cleanup.

    `label` becomes part of the filename so a glance at /tmp shows which
    section's server it belongs to.
    """
    fd = tempfile.NamedTemporaryFile(
        mode="w",
        suffix=f"_{label}.py",
        prefix="fastmcp01_",
        delete=False,
    )
    path = Path(fd.name)
    fd.write(source)
    fd.close()
    _temp_server_files.append(path)
    return path


print("Setup OK")
```

- [ ] **Step 3: Run the cell**

Expected output: `Setup OK`. If the imports fail:

```bash
pip install mcp
```

(The `mcp` package is the official Python SDK; FastMCP is a separate distribution we don't touch in this notebook.)

- [ ] **Step 4: Commit**

```bash
git add fastmcp/01_without_fastmcp.ipynb
git commit -m "feat(fastmcp): add setup cell to notebook 01"
```

---

## Task 5: Section 2 — Build a one-tool server the raw way

**Files:**
- Modify: `fastmcp/01_without_fastmcp.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 2. Build a One-Tool Server the Raw Way

We'll build the smallest useful MCP server: one tool called `add` that takes two integers and returns their sum. With the raw SDK that's already a non-trivial amount of code:

1. Create a `Server("...")` instance.
2. Register an `@server.list_tools()` handler that returns `list[Tool]`. Each `Tool` carries a **hand-written** `inputSchema` JSON Schema describing its arguments.
3. Register an `@server.call_tool()` handler that takes `(name, arguments)`, **dispatches on `name`**, runs the right Python function, and wraps the result in a `list[TextContent]`.
4. Wire `stdio_server()` up to `server.run(read, write, server.create_initialization_options())` inside `asyncio.run(...)` so the process talks JSON-RPC over stdin/stdout.

We write the whole thing as a string and drop it on disk so we can spawn it as a subprocess in the next section.
```

- [ ] **Step 2: Add the code cell that materializes the one-tool server**

```python
ONE_TOOL_SERVER = '''\
"""One-tool MCP server, written with the raw `mcp` SDK (no FastMCP)."""
import asyncio

from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import TextContent, Tool

server = Server("one-tool-demo")


@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="add",
            description="Add two integers and return their sum.",
            inputSchema={
                "type": "object",
                "properties": {
                    "a": {"type": "integer", "description": "First addend."},
                    "b": {"type": "integer", "description": "Second addend."},
                },
                "required": ["a", "b"],
                "additionalProperties": False,
            },
        ),
    ]


@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "add":
        result = int(arguments["a"]) + int(arguments["b"])
        return [TextContent(type="text", text=str(result))]
    raise ValueError(f"Unknown tool: {name}")


async def main() -> None:
    async with stdio_server() as (read, write):
        await server.run(read, write, server.create_initialization_options())


if __name__ == "__main__":
    asyncio.run(main())
'''

one_tool_path = write_server_to_temp(ONE_TOOL_SERVER, "one_tool")
print(f"Wrote one-tool server to: {one_tool_path}")
print(f"Lines of code: {len(ONE_TOOL_SERVER.splitlines())}")
```

- [ ] **Step 3: Run the cell and verify**

Expected output (the exact temp filename will vary — only the prefix `fastmcp01_` and the suffix `_one_tool.py` are stable):

```
Wrote one-tool server to: /var/folders/.../fastmcp01_XXXXXXXX_one_tool.py
Lines of code: 36
```

If you see a different path prefix (e.g. `/tmp/...` on Linux) that's fine; the line count must match.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/01_without_fastmcp.ipynb
git commit -m "feat(fastmcp): write raw-SDK one-tool server in notebook 01"
```

---

## Task 6: Section 3 — Run it and call its tool

**Files:**
- Modify: `fastmcp/01_without_fastmcp.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 3. Run It and Call Its Tool

The `mcp` SDK ships a stdio client that spawns a server as a subprocess and hands us a `(read, write)` pair of streams talking JSON-RPC. Wrap that in a `ClientSession`, call `session.initialize()`, and we can then `list_tools()` and `call_tool(...)`.

We run the whole thing inside one `async` function and drive it with `await` (Jupyter's IPython kernel runs an event loop already, so we don't need `asyncio.run`).
```

- [ ] **Step 2: Add the code cell**

```python
async def call_one_tool_server(server_path: Path) -> None:
    params = StdioServerParameters(command="python", args=[str(server_path)])
    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            tools = await session.list_tools()
            print("Tools advertised by the server:")
            for tool in tools.tools:
                print(f"  - {tool.name}: {tool.description}")
                print(f"    inputSchema keys: {sorted(tool.inputSchema.keys())}")

            result = await session.call_tool("add", {"a": 2, "b": 3})
            print()
            print("call_tool('add', {'a': 2, 'b': 3}) returned:")
            for block in result.content:
                print(f"  ({block.type}) {block.text}")


await call_one_tool_server(one_tool_path)
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Tools advertised by the server:
  - add: Add two integers and return their sum.
    inputSchema keys: ['additionalProperties', 'properties', 'required', 'type']

call_tool('add', {'a': 2, 'b': 3}) returned:
  (text) 5
```

If you see `RuntimeError: This event loop is already running` instead, your Jupyter kernel is older than 7.x — wrap the call in `asyncio.get_event_loop().run_until_complete(call_one_tool_server(one_tool_path))` or upgrade Jupyter.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/01_without_fastmcp.ipynb
git commit -m "feat(fastmcp): spawn raw-SDK server and call 'add' in notebook 01"
```

---

## Task 7: Section 4 — Add a second tool and feel the cost

**Files:**
- Modify: `fastmcp/01_without_fastmcp.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 4. Add a Second Tool — and Feel the Cost

So far we've paid the raw-SDK tax once: one `Tool(...)` entry, one hand-written `inputSchema`, one `if name == "...":` branch.

Watch what adding **one** more tool — a `greet(name: str) -> str` — costs us:

1. A second `Tool(...)` entry in `list_tools` with **its own** hand-written `inputSchema` (different keys, different required list, different types).
2. A second `elif name == "...":` branch in `call_tool` with **its own** argument extraction and result wrapping.

The Python logic itself is one line (`f"Hello, {name}!"`). Everything else is plumbing. That's the cost we'll erase in notebook 02.
```

- [ ] **Step 2: Add the code cell that materializes the two-tool server**

```python
TWO_TOOL_SERVER = '''\
"""Two-tool MCP server, written with the raw `mcp` SDK (no FastMCP).

Notice the duplication: each tool requires (1) its own Tool() entry with a
hand-written inputSchema, and (2) its own branch in call_tool.
"""
import asyncio

from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import TextContent, Tool

server = Server("two-tool-demo")


@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="add",
            description="Add two integers and return their sum.",
            inputSchema={
                "type": "object",
                "properties": {
                    "a": {"type": "integer", "description": "First addend."},
                    "b": {"type": "integer", "description": "Second addend."},
                },
                "required": ["a", "b"],
                "additionalProperties": False,
            },
        ),
        Tool(
            name="greet",
            description="Return a friendly greeting for `name`.",
            inputSchema={
                "type": "object",
                "properties": {
                    "name": {"type": "string", "description": "Who to greet."},
                },
                "required": ["name"],
                "additionalProperties": False,
            },
        ),
    ]


@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "add":
        result = int(arguments["a"]) + int(arguments["b"])
        return [TextContent(type="text", text=str(result))]
    elif name == "greet":
        result = f"Hello, {arguments['name']}!"
        return [TextContent(type="text", text=result)]
    raise ValueError(f"Unknown tool: {name}")


async def main() -> None:
    async with stdio_server() as (read, write):
        await server.run(read, write, server.create_initialization_options())


if __name__ == "__main__":
    asyncio.run(main())
'''

two_tool_path = write_server_to_temp(TWO_TOOL_SERVER, "two_tool")
print(f"Wrote two-tool server to: {two_tool_path}")
print(f"Lines of code: {len(TWO_TOOL_SERVER.splitlines())}")
print(f"Delta vs. one-tool server: +{len(TWO_TOOL_SERVER.splitlines()) - 36} lines for +1 tool")
```

- [ ] **Step 3: Run the cell and verify**

Expected output (the exact temp filename will vary):

```
Wrote two-tool server to: /var/folders/.../fastmcp01_XXXXXXXX_two_tool.py
Lines of code: 60
Delta vs. one-tool server: +24 lines for +1 tool
```

(If your platform's `splitlines()` gives a different total because of a trailing newline, the delta line is still the point — every new tool costs roughly two dozen lines of schema + dispatch.)

- [ ] **Step 4: Add the code cell that drives the two-tool server**

```python
async def call_two_tool_server(server_path: Path) -> None:
    params = StdioServerParameters(command="python", args=[str(server_path)])
    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            tools = await session.list_tools()
            print("Tools advertised by the server:")
            for tool in tools.tools:
                print(f"  - {tool.name}: {tool.description}")

            add_result = await session.call_tool("add", {"a": 10, "b": 32})
            print()
            print("call_tool('add', {'a': 10, 'b': 32}) returned:")
            for block in add_result.content:
                print(f"  ({block.type}) {block.text}")

            greet_result = await session.call_tool("greet", {"name": "Ada"})
            print()
            print("call_tool('greet', {'name': 'Ada'}) returned:")
            for block in greet_result.content:
                print(f"  ({block.type}) {block.text}")


await call_two_tool_server(two_tool_path)
```

- [ ] **Step 5: Run the cell and verify**

Expected output:

```
Tools advertised by the server:
  - add: Add two integers and return their sum.
  - greet: Return a friendly greeting for `name`.

call_tool('add', {'a': 10, 'b': 32}) returned:
  (text) 42

call_tool('greet', {'name': 'Ada'}) returned:
  (text) Hello, Ada!
```

- [ ] **Step 6: Commit**

```bash
git add fastmcp/01_without_fastmcp.ipynb
git commit -m "feat(fastmcp): add second tool to surface raw-SDK boilerplate in notebook 01"
```

---

## Task 8: Closing recap and "What's missing" teaser

**Files:**
- Modify: `fastmcp/01_without_fastmcp.ipynb` (add 2 markdown cells)

- [ ] **Step 1: Add the "What you just learned" markdown cell**

```markdown
## What you just learned

- The raw `mcp` SDK exposes three primitives you wire by hand: `Server`, `@server.list_tools()`, and `@server.call_tool()`. The transport plumbing (`stdio_server`, `server.run(...)`) is also yours to set up.
- **Every** tool you add costs roughly the same amount of boilerplate: one `Tool(...)` entry in `list_tools` with a hand-written `inputSchema`, plus one branch in the `call_tool` dispatcher. Two tools doubled the schema code; the dispatcher grew an `elif`.
- The Python logic inside each tool is tiny (`a + b`, `f"Hello, {name}!"`). The framework cost dominates.
- A stdio MCP server is realistically run as a `.py` file spawned by a client — `stdio_client(StdioServerParameters(...))` is the canonical way to do that from another Python process.
```

- [ ] **Step 2: Add the "What's missing" markdown cell**

```markdown
## What's missing

We built two tools and wrote ~60 lines of server code, almost all of it schema declarations and dispatch glue. The actual *behavior* of the tools was three lines. That ratio gets worse with every tool we'd add.

Every tool means another schema by hand, another handler dispatch branch. **There should be a framework.**

In **notebook 02** we introduce that framework: **FastMCP**. We'll rebuild the same two-tool server with `@mcp.tool` decorators on plain Python functions — type hints become the JSON Schema, docstrings become the tool description, and there is no `call_tool` dispatcher to maintain. Same protocol on the wire; a fraction of the code on disk.
```

- [ ] **Step 3: Verify both cells render**

The italic emphasis on *"There should be a framework."* and the bold on **FastMCP** should display.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/01_without_fastmcp.ipynb
git commit -m "feat(fastmcp): add closing recap and notebook-02 teaser to notebook 01"
```

---

## Task 9: Cleanup cell

**Files:**
- Modify: `fastmcp/01_without_fastmcp.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## Cleanup

The temp `.py` files we wrote earlier live in the OS scratch directory. They'd eventually be reaped, but cleaning them up here keeps a fresh-kernel re-run idempotent.
```

- [ ] **Step 2: Add the cleanup code cell**

```python
removed: list[str] = []
for path in list(_temp_server_files):
    try:
        path.unlink()
        removed.append(str(path))
    except FileNotFoundError:
        pass
_temp_server_files.clear()

print(f"Removed {len(removed)} temp server file(s):")
for r in removed:
    print(f"  - {r}")
```

- [ ] **Step 3: Run the cleanup cell and verify**

Expected output (filenames will differ):

```
Removed 2 temp server file(s):
  - /var/folders/.../fastmcp01_XXXXXXXX_one_tool.py
  - /var/folders/.../fastmcp01_XXXXXXXX_two_tool.py
```

- [ ] **Step 4: Commit**

```bash
git add fastmcp/01_without_fastmcp.ipynb
git commit -m "feat(fastmcp): add temp-file cleanup cell to notebook 01"
```

---

## Task 10: End-to-end verification in a fresh kernel

**Files:** none (verification only).

- [ ] **Step 1: Execute the notebook top-to-bottom in a clean kernel**

```bash
jupyter nbconvert --to notebook --execute --inplace fastmcp/01_without_fastmcp.ipynb
```

Expected: succeeds with no errors. The notebook is rewritten with embedded outputs from the fresh run.

- [ ] **Step 2: Verify expected outputs in the executed notebook**

The output cells should contain, in order:

1. `Setup OK`
2. The one-tool server materialization output:
   ```
   Wrote one-tool server to: /...fastmcp01_..._one_tool.py
   Lines of code: 36
   ```
3. The one-tool server interaction:
   ```
   Tools advertised by the server:
     - add: Add two integers and return their sum.
       inputSchema keys: ['additionalProperties', 'properties', 'required', 'type']

   call_tool('add', {'a': 2, 'b': 3}) returned:
     (text) 5
   ```
4. The two-tool server materialization output:
   ```
   Wrote two-tool server to: /...fastmcp01_..._two_tool.py
   Lines of code: 60
   Delta vs. one-tool server: +24 lines for +1 tool
   ```
5. The two-tool server interaction:
   ```
   Tools advertised by the server:
     - add: Add two integers and return their sum.
     - greet: Return a friendly greeting for `name`.

   call_tool('add', {'a': 10, 'b': 32}) returned:
     (text) 42

   call_tool('greet', {'name': 'Ada'}) returned:
     (text) Hello, Ada!
   ```
6. The cleanup output:
   ```
   Removed 2 temp server file(s):
     - /...fastmcp01_..._one_tool.py
     - /...fastmcp01_..._two_tool.py
   ```

No cell raises an unhandled exception. If `nbconvert` reports `ModuleNotFoundError: No module named 'mcp'`, install the SDK first: `pip install mcp`.

- [ ] **Step 3: Verify no temp files leaked**

```bash
ls /tmp/fastmcp01_* /var/folders/**/fastmcp01_* 2>/dev/null | wc -l
```

Expected: `0`. (On macOS the temp dir lives under `/var/folders`; on Linux it's typically `/tmp`. A single zero means nothing leaked from this run.)

- [ ] **Step 4: Commit the clean run**

```bash
git add fastmcp/01_without_fastmcp.ipynb
git commit -m "chore(fastmcp): commit clean fresh-kernel run of notebook 01"
```

---

## Done

After Task 10 passes, notebook 01 is complete. The next plan to write is `2026-05-18-fastmcp-notebook-02-fastmcp-local-stdio.md`, which introduces FastMCP itself with `@mcp.tool` decorators, auto-schema from type hints, and the in-memory `Client(mcp)` for direct in-process testing.
