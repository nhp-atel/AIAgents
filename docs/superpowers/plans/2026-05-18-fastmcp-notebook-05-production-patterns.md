# FastMCP Notebook 05 — `05_fastmcp_production_patterns.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the fifth notebook in the FastMCP learning series — demonstrate the four framework features that pay off in non-toy apps: `lifespan` for resource startup/teardown, the auto-injected `Context` parameter for in-tool logging and progress, structured error handling via `ToolError`, and `mount()` for composing multiple FastMCP apps under one server. By the end the learner can manage server-lifetime resources, emit progress from long-running tools, surface clean MCP error responses, and compose multi-app servers — all things the raw `mcp` SDK either does awkwardly or not at all.

**Architecture:** A single Jupyter notebook that builds one primary `FastMCP("main")` instance with a `lifespan` that opens an in-memory sqlite connection. Four feature demonstrations are layered onto this primary instance: (1) a tool that reads from the lifespan-managed sqlite connection, (2) a tool with a `ctx: Context` parameter that emits `ctx.info` and `ctx.report_progress` updates over a slow loop, (3) a tool that raises `ToolError` and is observed returning a clean MCP error, (4) a secondary `FastMCP("math")` instance with one tool that gets `mount()`ed into the primary. Every example is driven through the in-memory `Client(parent_mcp)` — no HTTP, no threads, no ports — because this notebook is feature-dense and an HTTP server would add unrelated noise.

**Tech Stack:** Python 3.11+, Jupyter, `fastmcp` (2.x line), `sqlite3` (stdlib), `asyncio` (stdlib), `contextlib.asynccontextmanager` (stdlib). No FastAPI, no uvicorn, no httpx — the in-memory client makes those unnecessary here.

**Companion spec:** `docs/superpowers/specs/2026-05-18-fastmcp-learning-series-design.md` (notebook 05 section).

**Port assignment:** none. This notebook uses the in-memory `Client(parent_mcp)` exclusively. The spec reserves `8020` and `8021` for FastMCP HTTP usage in this series, but `mount()` is in-process composition (not network proxying), so no second port is needed.

---

## File Structure

- **Create:** `fastmcp/05_fastmcp_production_patterns.ipynb` — the entire notebook, self-contained (no imports from earlier notebooks in the series).
- **Modify:** none.
- **No separate test files** — verification is the notebook running top-to-bottom in a fresh kernel (Task 14).

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown)
2. **"What you'll learn"** (markdown bullets)
3. **Section 1: Setup — imports** (markdown + 1 code cell)
4. **Section 2: Lifespan — opening and closing resources cleanly** (markdown + 3 code cells: define lifespan, build server + tool, drive it through in-memory client)
5. **Section 3: The Context parameter — logging and progress from inside a tool** (markdown + 2 code cells: define progress tool, call it and observe progress events)
6. **Section 4: Structured errors** (markdown + 2 code cells: define error-raising tool, call it and inspect the structured failure)
7. **Section 5: Composition with mount()** (markdown + 2 code cells: define secondary `FastMCP("math")` + mount it, then call the mounted tool through the primary client)
8. **"What you just learned"** (markdown recap)
9. **"What's missing"** (markdown — teases notebook 06, validation & safety)
10. **Section 6: Cleanup** (markdown + 1 code cell — no server to stop, but include the standard cleanup pattern for consistency with other notebooks in the series)

---

## Task 1: Create the `fastmcp/` folder and the notebook scaffold

**Files:**
- Create: `fastmcp/05_fastmcp_production_patterns.ipynb`

- [ ] **Step 1: Create the `fastmcp/` directory if it does not already exist**

```bash
mkdir -p fastmcp
ls -la fastmcp
```

Expected: the directory exists (either freshly created or already present from earlier notebooks in the series).

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
python -c "import json; json.load(open('fastmcp/05_fastmcp_production_patterns.ipynb'))"
ls -la fastmcp/05_fastmcp_production_patterns.ipynb
```

Expected: no exception, file non-zero size.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/05_fastmcp_production_patterns.ipynb
git commit -m "feat(fastmcp): scaffold 05_fastmcp_production_patterns.ipynb"
```

---

## Task 2: Add title and "Why this notebook exists"

**Files:**
- Modify: `fastmcp/05_fastmcp_production_patterns.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Add a markdown cell with title and motivation**

Cell content (markdown):

```markdown
# 05 — Production Patterns: Lifespan, Context, Structured Errors, and Mount

## Why this notebook exists

In **notebooks 02–04** we built FastMCP servers, swapped their transport between stdio and streamable-HTTP, and bolted on authentication — three things the raw `mcp` SDK makes you wire up by hand. But none of those examples touched anything stateful. Every tool was a pure function. Every call returned a canned string. That's fine for learning the surface, and it's roughly the same place `mcp/05` leaves you.

Real servers need more than that. They open database connections at startup and close them at shutdown. Their long-running tools want to stream progress back to the caller instead of going dark for thirty seconds. They want to fail loudly with a clean error message when the input is wrong, not crash with a stack trace the client has to guess at. And once a codebase has more than a handful of tools, it wants to split them across modules without standing up a second server.

This notebook covers the four FastMCP features that address those needs: `lifespan`, the `Context` parameter, `ToolError`, and `mount()`. All four are driven through the **in-memory `Client(server)`** — no HTTP, no background threads. The in-memory client makes feature-dense notebooks like this one readable: each section is *here is the feature, here is one tool that uses it, here is one client call that shows the effect.*
```

- [ ] **Step 2: Verify the cell renders**

Open the notebook in Jupyter/VS Code and confirm the cell renders as markdown.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/05_fastmcp_production_patterns.ipynb
git commit -m "feat(fastmcp): add intro markdown to notebook 05"
```

---

## Task 3: Add "What you'll learn"

**Files:**
- Modify: `fastmcp/05_fastmcp_production_patterns.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Add a markdown cell with "What you'll learn"**

Cell content (markdown):

```markdown
## What you'll learn

- How to use `lifespan` to open a resource (e.g., a sqlite connection) at server start and close it at server stop, and how a tool reads from that resource through `ctx.request_context.lifespan_context`.
- How the `Context` parameter is **auto-injected** by FastMCP based on parameter type — the client never passes it and it never appears in the tool's external schema.
- How to call `await ctx.info(...)` for structured server-side logs and `await ctx.report_progress(...)` to push progress updates to the client during a long-running tool.
- How to raise `ToolError` to surface a clean MCP error response from inside a tool, instead of leaking a Python stack trace.
- How to compose two `FastMCP` apps with `mount()` so a client calling the primary sees tools from both — a pattern with no raw-SDK analogue.
- Why all of the above is driven through the in-memory `Client(server)` in this notebook: feature density.
```

- [ ] **Step 2: Verify the cell renders**

The bullet list should render correctly.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/05_fastmcp_production_patterns.ipynb
git commit -m "feat(fastmcp): add 'what you'll learn' to notebook 05"
```

---

## Task 4: Add the setup cell

**Files:**
- Modify: `fastmcp/05_fastmcp_production_patterns.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 1. Setup

We need the FastMCP server and client classes, the `Context` type, the `ToolError` exception, `sqlite3` and `asyncio` from the stdlib, and `asynccontextmanager` for the lifespan. Each notebook in this series is self-contained, so we re-import here rather than rely on earlier notebooks.

> **API note:** This notebook is written against the FastMCP 2.x line. The names below — `FastMCP`, `Client`, `Context`, and `ToolError` — match the public API at time of writing. If a newer FastMCP release renames any of them (`ToolError` in particular has been the most likely to move between minor versions), update this cell to match your installed version.
```

- [ ] **Step 2: Add the setup code cell**

```python
import asyncio
import sqlite3
from contextlib import asynccontextmanager

from fastmcp import FastMCP, Client, Context
from fastmcp.exceptions import ToolError

print("Setup OK")
```

- [ ] **Step 3: Run the cell**

Expected output: `Setup OK`. If imports fail:

```bash
pip install "fastmcp>=2"
```

- [ ] **Step 4: Commit**

```bash
git add fastmcp/05_fastmcp_production_patterns.ipynb
git commit -m "feat(fastmcp): add setup cell to notebook 05"
```

---

## Task 5: Section 2 (part 1) — Define the lifespan and the primary server

**Files:**
- Modify: `fastmcp/05_fastmcp_production_patterns.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 2. Lifespan — opening and closing resources cleanly

A `lifespan` is an async context manager that FastMCP enters once at server start and exits once at server stop. Anything you `yield` from it becomes the **lifespan context** — a dict-like blob that every tool can pull from at call time. In a real app you'd put a database connection pool, an HTTP client, or any other resource here whose lifetime should equal the server's lifetime.

We'll open a sqlite in-memory connection at server start, seed it with a tiny `facts` table, and close it at shutdown.

> **Quirk:** `lifespan` cleanly bookends server start and stop, so resources (DB connections, HTTP pools) live exactly one server-lifetime. The raw `mcp` SDK requires you to wire this up by hand around the `stdio_server` / HTTP runner; FastMCP makes it a constructor argument.

> **API note:** FastMCP 2.x accepts the lifespan as `FastMCP(lifespan=my_lifespan)` where `my_lifespan` is an `@asynccontextmanager`-decorated async generator that **takes the server instance as its single argument** and **yields a dict** of resources. Inside a tool, those resources are reachable as `ctx.request_context.lifespan_context["key"]`. If your installed FastMCP version differs (some pre-2.x prereleases used a no-arg lifespan or yielded a dataclass instead of a dict), adjust the signature and the access path to match.
```

- [ ] **Step 2: Add the lifespan + server code cell**

```python
@asynccontextmanager
async def sqlite_lifespan(server: FastMCP):
    """Open a sqlite connection at server start, close it at shutdown."""
    print("[lifespan] opening sqlite connection")
    conn = sqlite3.connect(":memory:")
    conn.execute("CREATE TABLE facts (topic TEXT PRIMARY KEY, fact TEXT)")
    conn.executemany(
        "INSERT INTO facts (topic, fact) VALUES (?, ?)",
        [
            ("octopus", "Octopuses have three hearts."),
            ("rome", "Rome was founded in 753 BC, traditionally."),
            ("python", "Python is named after Monty Python, not the snake."),
        ],
    )
    conn.commit()
    try:
        yield {"db": conn}
    finally:
        print("[lifespan] closing sqlite connection")
        conn.close()


parent_mcp = FastMCP("main", lifespan=sqlite_lifespan)
print(f"Created FastMCP server: {parent_mcp.name!r}")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Created FastMCP server: 'main'
```

(The `[lifespan] opening sqlite connection` line does **not** print yet — the lifespan only runs when a client connects.)

- [ ] **Step 4: Commit**

```bash
git add fastmcp/05_fastmcp_production_patterns.ipynb
git commit -m "feat(fastmcp): add lifespan + primary server to notebook 05"
```

---

## Task 6: Section 2 (part 2) — Define a tool that reads from the lifespan resource

**Files:**
- Modify: `fastmcp/05_fastmcp_production_patterns.ipynb` (add 1 code cell)

- [ ] **Step 1: Add the tool code cell**

```python
@parent_mcp.tool
async def lookup_fact(topic: str, ctx: Context) -> str:
    """Return a stored fact about a topic, looked up from the lifespan sqlite connection."""
    conn: sqlite3.Connection = ctx.request_context.lifespan_context["db"]
    row = conn.execute("SELECT fact FROM facts WHERE topic = ?", (topic,)).fetchone()
    if row is None:
        return f"No fact stored for topic {topic!r}."
    return row[0]


print(f"Registered tool: lookup_fact")
```

- [ ] **Step 2: Run the cell and verify**

Expected output:

```
Registered tool: lookup_fact
```

- [ ] **Step 3: Commit**

```bash
git add fastmcp/05_fastmcp_production_patterns.ipynb
git commit -m "feat(fastmcp): add lookup_fact tool using lifespan resource"
```

---

## Task 7: Section 2 (part 3) — Drive `lookup_fact` through the in-memory client

**Files:**
- Modify: `fastmcp/05_fastmcp_production_patterns.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a brief markdown cell**

```markdown
Now we call the tool. Entering the `Client(parent_mcp)` async context triggers the lifespan (so we'll see the `opening sqlite connection` log line), and exiting the context tears it down (so we'll see `closing sqlite connection`). Anything in between is a normal MCP tool call.
```

- [ ] **Step 2: Add the client-call code cell**

```python
async def call_lookup_fact():
    async with Client(parent_mcp) as client:
        result = await client.call_tool("lookup_fact", {"topic": "octopus"})
        print("Tool returned:", result.data)

        missing = await client.call_tool("lookup_fact", {"topic": "unknown"})
        print("Tool returned:", missing.data)


await call_lookup_fact()
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
[lifespan] opening sqlite connection
Tool returned: Octopuses have three hearts.
Tool returned: No fact stored for topic 'unknown'.
[lifespan] closing sqlite connection
```

> **API note:** The exact accessor for tool return values in the FastMCP 2.x in-memory `Client` is `result.data` for the auto-deserialized payload. If your installed version returns a different attribute (e.g., `result.content` as a list of content parts), substitute accordingly — the underlying data is identical, only the unwrapping changes.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/05_fastmcp_production_patterns.ipynb
git commit -m "feat(fastmcp): call lookup_fact through in-memory client"
```

---

## Task 8: Section 3 (part 1) — Define a tool that uses `ctx.info` and `ctx.report_progress`

**Files:**
- Modify: `fastmcp/05_fastmcp_production_patterns.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 3. The Context parameter — logging and progress from inside a tool

FastMCP inspects each tool's signature when you register it. If one of the parameters is annotated as `Context` (or named `ctx`), FastMCP **auto-injects** the runtime context at call time. The client doesn't pass `ctx`, and `ctx` does **not** appear in the tool's external JSON schema. From the caller's perspective the tool only takes whatever other parameters you declared.

Inside the tool you can use the context for two big things:

- `await ctx.info(message)` (also `debug`, `warning`, `error`) — structured logs that get routed to the FastMCP server-side log handler.
- `await ctx.report_progress(progress, total)` — progress updates that flow back over the MCP connection so a client can show a progress bar instead of staring at a spinner.

To make the progress updates visible we'll do something deliberately slow: an `asyncio.sleep(0.1)` loop that reports 0/5, 1/5, ..., 5/5.

> **Quirk:** `Context` is auto-injected by parameter type/name, not passed by the client. The tool's external schema does NOT include `ctx`.
```

- [ ] **Step 2: Add the progress tool code cell**

```python
@parent_mcp.tool
async def slow_count(n: int, ctx: Context) -> str:
    """Count to n slowly, reporting progress along the way."""
    await ctx.info(f"slow_count starting with n={n}")
    for i in range(n + 1):
        await ctx.report_progress(progress=i, total=n)
        if i < n:
            await asyncio.sleep(0.1)
    await ctx.info(f"slow_count finished")
    return f"Counted to {n}."


print("Registered tool: slow_count")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Registered tool: slow_count
```

- [ ] **Step 4: Commit**

```bash
git add fastmcp/05_fastmcp_production_patterns.ipynb
git commit -m "feat(fastmcp): add slow_count tool using Context"
```

---

## Task 9: Section 3 (part 2) — Call `slow_count` and capture progress events

**Files:**
- Modify: `fastmcp/05_fastmcp_production_patterns.ipynb` (add 1 code cell)

- [ ] **Step 1: Add the client-call code cell**

```python
async def call_slow_count():
    progress_events: list[tuple[float, float | None]] = []

    async def on_progress(progress: float, total: float | None, message: str | None) -> None:
        progress_events.append((progress, total))
        print(f"[client] progress {progress}/{total}")

    async with Client(parent_mcp, progress_handler=on_progress) as client:
        result = await client.call_tool("slow_count", {"n": 5})
        print("Tool returned:", result.data)

    print(f"Captured {len(progress_events)} progress events on the client side.")


await call_slow_count()
```

- [ ] **Step 2: Run the cell and verify**

Expected output:

```
[lifespan] opening sqlite connection
[client] progress 0/5
[client] progress 1/5
[client] progress 2/5
[client] progress 3/5
[client] progress 4/5
[client] progress 5/5
Tool returned: Counted to 5.
Captured 6 progress events on the client side.
[lifespan] closing sqlite connection
```

> **API note:** FastMCP 2.x's in-memory `Client` accepts a `progress_handler=` keyword that is invoked once per `ctx.report_progress(...)` call on the server, with `(progress, total, message)` arguments. If your installed version uses a slightly different handler signature (e.g., a single object instead of three positional args, or a different keyword name like `on_progress`), adjust this cell to match. The `ctx.info(...)` log lines may or may not appear depending on whether your kernel's logging is wired up to display FastMCP's logger; the progress events are the load-bearing demonstration here.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/05_fastmcp_production_patterns.ipynb
git commit -m "feat(fastmcp): call slow_count and capture progress events"
```

---

## Task 10: Section 4 — Structured errors with `ToolError`

**Files:**
- Modify: `fastmcp/05_fastmcp_production_patterns.ipynb` (add markdown header + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 4. Structured errors

When a tool raises an arbitrary `Exception`, the MCP protocol still has to send *something* back to the client — but the client gets either a generic "internal error" or, worse, leaked details about your server's internals. The clean way to fail is to raise `ToolError("explicit reason")` from inside the tool. FastMCP catches it, packages the message into a proper MCP error response, and the client sees a structured failure with a readable reason — no stack trace.

We'll define a tool that refuses to divide by zero, then call it both ways: once with a valid input, once with the zero-divisor input.

> **API note:** The exception class lives at `fastmcp.exceptions.ToolError` in the FastMCP 2.x line. If your installed version exposes it from a different module (e.g., a top-level `fastmcp.ToolError`), adjust the import in the setup cell.
```

- [ ] **Step 2: Add the error-raising tool cell**

```python
@parent_mcp.tool
def safe_divide(numerator: float, denominator: float) -> float:
    """Divide numerator by denominator. Raises a structured error on division by zero."""
    if denominator == 0:
        raise ToolError("denominator must not be zero")
    return numerator / denominator


print("Registered tool: safe_divide")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Registered tool: safe_divide
```

- [ ] **Step 4: Add the client-call cell that exercises both paths**

```python
async def call_safe_divide():
    async with Client(parent_mcp) as client:
        ok = await client.call_tool("safe_divide", {"numerator": 10, "denominator": 2})
        print("Happy path:", ok.data)

        try:
            await client.call_tool("safe_divide", {"numerator": 10, "denominator": 0})
        except Exception as exc:
            print(f"Error path: client received {type(exc).__name__}: {exc}")


await call_safe_divide()
```

- [ ] **Step 5: Run the cell and verify**

Expected output:

```
[lifespan] opening sqlite connection
Happy path: 5.0
Error path: client received ToolError: denominator must not be zero
[lifespan] closing sqlite connection
```

> **API note:** The exact exception type re-raised on the client side may differ between FastMCP versions — some versions surface it as `ToolError`, others wrap it in a generic `McpError` whose message contains the original reason. What matters is that the client receives a **structured** failure with the original reason string, not an opaque internal error or a server-side stack trace. The `type(exc).__name__: {exc}` print pattern makes whichever class your version uses visible without breaking the cell.

- [ ] **Step 6: Commit**

```bash
git add fastmcp/05_fastmcp_production_patterns.ipynb
git commit -m "feat(fastmcp): add structured errors with ToolError"
```

---

## Task 11: Section 5 (part 1) — Define the secondary `FastMCP("math")` and mount it

**Files:**
- Modify: `fastmcp/05_fastmcp_production_patterns.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 5. Composition with `mount()`

A grown-up FastMCP codebase doesn't keep every tool in one module. You'll want to split tools across files (or across teams) and then bring them together at the edge. FastMCP supports that with `mount()`: build a secondary `FastMCP` app with its own tools, then mount it into a parent app under a prefix. A client connecting to the parent sees the parent's tools plus the mounted app's tools — all through one connection.

This is **in-process composition**, not network proxying. The mounted server doesn't need to run on its own port. We define a tiny `FastMCP("math")` with one tool, mount it under the prefix `math_`, and call its tool through the same `Client(parent_mcp)` we've been using all along.

> **Quirk:** `mount()` has no raw-SDK analogue — FastMCP-specific composition.

> **API note:** Current FastMCP 2.x accepts `parent_mcp.mount(prefix="math_", server=child_mcp)` (keyword form) or `parent_mcp.mount("math_", child_mcp)` (positional form). The argument *order* has shifted between earlier and later 2.x minors — some versions take `(server, prefix=...)`. If the positional call below errors with `TypeError`, swap to the keyword form (which is stable across versions) shown in the next cell.
```

- [ ] **Step 2: Add the secondary server + mount code cell**

```python
math_mcp = FastMCP("math")


@math_mcp.tool
def multiply(a: float, b: float) -> float:
    """Multiply two numbers."""
    return a * b


parent_mcp.mount(prefix="math_", server=math_mcp)
print(f"Mounted {math_mcp.name!r} under prefix 'math_' on {parent_mcp.name!r}")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Mounted 'math' under prefix 'math_' on 'main'
```

- [ ] **Step 4: Commit**

```bash
git add fastmcp/05_fastmcp_production_patterns.ipynb
git commit -m "feat(fastmcp): define secondary math server and mount it"
```

---

## Task 12: Section 5 (part 2) — Call the mounted tool through the primary client

**Files:**
- Modify: `fastmcp/05_fastmcp_production_patterns.ipynb` (add 1 code cell)

- [ ] **Step 1: Add the client-call code cell**

```python
async def call_mounted_tool():
    async with Client(parent_mcp) as client:
        tools = await client.list_tools()
        tool_names = sorted(t.name for t in tools)
        print(f"Tools visible to the client: {tool_names}")

        result = await client.call_tool("math_multiply", {"a": 6, "b": 7})
        print(f"math_multiply(6, 7) = {result.data}")


await call_mounted_tool()
```

- [ ] **Step 2: Run the cell and verify**

Expected output:

```
[lifespan] opening sqlite connection
Tools visible to the client: ['lookup_fact', 'math_multiply', 'safe_divide', 'slow_count']
math_multiply(6, 7) = 42.0
[lifespan] closing sqlite connection
```

The key observation: the client connected only to `parent_mcp`, but `math_multiply` (from the mounted `math_mcp`) is in the tool list and is callable just like any native tool — the prefix `math_` distinguishes it.

> **API note:** The exact prefix-joining convention (`math_multiply` vs `math.multiply` vs `math/multiply`) has varied across FastMCP minors. The current 2.x convention is underscore concatenation (`{prefix}{tool_name}` where `prefix` is what you passed to `mount`), but if `list_tools()` shows a different separator on your version, adjust the `call_tool` name to match what `list_tools` reports.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/05_fastmcp_production_patterns.ipynb
git commit -m "feat(fastmcp): call mounted tool through primary client"
```

---

## Task 13: Closing recap, teaser, and cleanup

**Files:**
- Modify: `fastmcp/05_fastmcp_production_patterns.ipynb` (add 2 markdown cells + 1 markdown header + 1 code cell)

- [ ] **Step 1: Add the "What you just learned" markdown cell**

```markdown
## What you just learned

- `lifespan` is an `@asynccontextmanager` you pass to `FastMCP(lifespan=...)`. Whatever it yields becomes the lifespan context; tools reach it via `ctx.request_context.lifespan_context["key"]`. Resources opened there live exactly one server-lifetime.
- A `ctx: Context` parameter is **auto-injected** — clients never pass it and it never appears in the tool's external schema. Inside the tool you can call `await ctx.info(...)` for logs and `await ctx.report_progress(progress, total)` to stream progress updates back to the caller.
- The in-memory `Client(server)` accepts a `progress_handler=` keyword that fires once per server-side `report_progress` call — so you can observe the events from the client side.
- Raising `ToolError("reason")` from inside a tool produces a structured MCP error response on the client; raising a bare `Exception` does not. Use `ToolError` for any failure you want the caller to see meaningfully.
- `parent_mcp.mount(prefix="x_", server=child_mcp)` composes two `FastMCP` apps into one server. The client connects to the parent and sees both apps' tools through one connection — no second port, no second process. This pattern doesn't exist in the raw `mcp` SDK.
- The in-memory `Client` made all of this visible in a single notebook with no HTTP, no threads, and no ports. Real production would expose the same server over streamable-HTTP — we covered that transport in notebook 03 and will revisit it for the end-to-end demo in notebook 08.
```

- [ ] **Step 2: Add the "What's missing" markdown cell**

```markdown
## What's missing

We've built a server that opens resources cleanly, streams progress, fails politely, and composes from multiple apps — but we've been entirely trusting about the **input**. Every tool above accepts whatever the caller sends and tries to use it. If `lookup_fact` got a 10MB SQL-injection payload as its `topic`, we'd happily concatenate it into a query (we don't, because we used parameterized SQL — but nothing in the tool's signature forced that). If `slow_count` got `n=10_000_000`, it would chew the server up for sixteen minutes.

In **notebook 06** we wire input validation into the tool surface itself. FastMCP's first-class Pydantic integration means a Pydantic model declared as a tool parameter both **validates** the input and **generates** the JSON schema in one step. We'll also walk through allowlisted resources, output sanitization, and refusing dangerous tool args — the bare minimum to put a FastMCP server in front of anything untrusted.
```

- [ ] **Step 3: Add the cleanup markdown header**

```markdown
## 6. Cleanup

This notebook never started an HTTP server, so there's nothing to shut down on a network — every `async with Client(parent_mcp) as client:` block above already tore down its own lifespan when it exited. We include this cell anyway so the notebook's structure matches every other notebook in the series, where this is where the background server gets stopped.
```

- [ ] **Step 4: Add the cleanup code cell**

```python
# No background server to stop in this notebook; the in-memory client handles
# lifespan entry/exit per `async with Client(...)` block.
print("Nothing to clean up. Notebook complete.")
```

- [ ] **Step 5: Run the cleanup cell and verify**

Expected output:

```
Nothing to clean up. Notebook complete.
```

- [ ] **Step 6: Commit**

```bash
git add fastmcp/05_fastmcp_production_patterns.ipynb
git commit -m "feat(fastmcp): add closing recap, teaser, and cleanup to notebook 05"
```

---

## Task 14: End-to-end verification in a fresh kernel

**Files:** none (verification only).

- [ ] **Step 1: Execute the notebook top-to-bottom in a clean kernel**

```bash
jupyter nbconvert --to notebook --execute --inplace fastmcp/05_fastmcp_production_patterns.ipynb
```

Expected: succeeds with no errors. The notebook is rewritten with embedded outputs from the fresh run.

- [ ] **Step 2: Verify expected outputs in the executed notebook**

Cells should contain, in order:

1. `Setup OK`
2. `Created FastMCP server: 'main'`
3. `Registered tool: lookup_fact`
4. The two `Tool returned:` lines from `lookup_fact`, bracketed by `[lifespan] opening sqlite connection` and `[lifespan] closing sqlite connection`.
5. `Registered tool: slow_count`
6. Six `[client] progress` lines from 0/5 through 5/5, then `Tool returned: Counted to 5.`, then `Captured 6 progress events on the client side.`, all bracketed by the lifespan log lines.
7. `Registered tool: safe_divide`
8. `Happy path: 5.0` and `Error path: client received ...: denominator must not be zero` (the exception class name may vary between FastMCP versions; the message is the load-bearing part), bracketed by the lifespan log lines.
9. `Mounted 'math' under prefix 'math_' on 'main'`
10. The tools list `['lookup_fact', 'math_multiply', 'safe_divide', 'slow_count']` and `math_multiply(6, 7) = 42.0`, bracketed by the lifespan log lines.
11. `Nothing to clean up. Notebook complete.`

No cell raises an unhandled exception. If a FastMCP API mismatch surfaces in one of the marked `> **API note:** ...` cells, that cell's note explains what to swap; fix in place and re-run.

- [ ] **Step 3: Commit the clean run**

```bash
git add fastmcp/05_fastmcp_production_patterns.ipynb
git commit -m "chore(fastmcp): commit clean fresh-kernel run of notebook 05"
```

---

## Done

After Task 14 passes, notebook 05 is complete. The next plan to write is for `06_fastmcp_prompt_injection_and_safety.ipynb`, which covers Pydantic-native input validation, allowlisted resources, output sanitization, and refusing dangerous tool arguments. That plan lives at `docs/superpowers/plans/2026-05-18-fastmcp-notebook-06-safety.md`.
