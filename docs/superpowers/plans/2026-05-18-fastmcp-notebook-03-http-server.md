# FastMCP Notebook 03 — `03_fastmcp_http_server.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the third notebook in the FastMCP learning series — swap the transport, not the code. Take the same `FastMCP` instance with `add` and `greet` tools from notebook 02 and serve it over HTTP via `mcp.run(transport="streamable-http", host="127.0.0.1", port=8020)` on a background daemon thread. Drive it from the same `fastmcp.Client` class — this time passing a URL instead of an in-memory `FastMCP` instance — and confirm `list_tools` / `call_tool` work identically over the wire. The learner finishes the notebook understanding that transports in FastMCP are runtime flags, not code rewrites.

**Architecture:** A single Jupyter notebook that defines one `FastMCP` instance with two tools (`add`, `greet`), launches it via `mcp.run(transport="streamable-http", host="127.0.0.1", port=8020)` on a background daemon thread, then connects to it from the same notebook with `async with Client("http://127.0.0.1:8020/mcp/") as client:`. The notebook closes with a markdown contrast of `streamable-http` vs SSE and an explicit note on daemon-thread cleanup at kernel shutdown.

**Tech Stack:** Python 3.11+, Jupyter, `fastmcp` 2.x, threading, asyncio. Same stack as notebook 02 plus background-thread orchestration.

**Companion spec:** `docs/superpowers/specs/2026-05-18-fastmcp-learning-series-design.md` (notebook 03 section).

**FastMCP spec version targeted:** `fastmcp` 2.x (current stable). Transport: `streamable-http` (current MCP default; SSE is the older transport that streamable-http replaced in MCP spec rev 2025-03-26+). HTTP endpoint path: `/mcp/` by default in FastMCP 2.x.

**Port assignment:** Primary FastMCP HTTP server on `127.0.0.1:8020`. (Per spec, 8000/8001 are taken by the user's dev server and 8010/8011 by the `a2a/` series.)

---

## File Structure

- **Create:** `fastmcp/03_fastmcp_http_server.ipynb` — the entire notebook, self-contained (no imports from notebook 02).
- **Modify:** none.
- **No separate test files** — verification is the notebook running top-to-bottom in a fresh kernel (Task 12).

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown)
2. **"What you'll learn"** (markdown bullets)
3. **Setup: imports and the threaded server helper** (markdown header + code cell)
4. **Section 2: Define a FastMCP server with two tools** (markdown + code cell — the `add` and `greet` tools, identical to notebook 02)
5. **Section 3: Run it over HTTP on a background thread** (markdown + code cell — start the thread, sleep briefly, verify listening)
6. **Section 4: Call it from an HTTP MCP client** (markdown + code cell — `async with Client(...)` driving `list_tools` and `call_tool`)
7. **Section 5: `streamable-http` vs SSE** (markdown only, one paragraph)
8. **"What you just learned"** (markdown recap)
9. **"What's missing"** (markdown — teases notebook 04, bearer-token auth)
10. **Section 6: Cleanup** (markdown header + code cell — explicit note that the daemon thread exits at kernel shutdown)

---

## Task 1: Create the notebook scaffold

**Files:**
- Create: `fastmcp/03_fastmcp_http_server.ipynb`

- [ ] **Step 1: Confirm the `fastmcp/` directory exists**

```bash
ls -la fastmcp/
```

Expected: directory exists. If it does not yet (because notebooks 01–02 haven't been built), create it:

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
python -c "import json; json.load(open('fastmcp/03_fastmcp_http_server.ipynb'))"
ls -la fastmcp/03_fastmcp_http_server.ipynb
```

Expected: no exception, file non-zero size.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/03_fastmcp_http_server.ipynb
git commit -m "feat(fastmcp): scaffold 03_fastmcp_http_server.ipynb"
```

---

## Task 2: Add title and "Why this notebook exists"

**Files:**
- Modify: `fastmcp/03_fastmcp_http_server.ipynb` (add cells 1–2)

- [ ] **Step 1: Add a markdown cell with title and motivation**

Cell content (markdown):

```markdown
# 03 — FastMCP over HTTP: Swap the Transport, Not the Code

## Why this notebook exists

In **notebook 02** we built a minimum-viable FastMCP server — `add` and `greet` tools defined with `@mcp.tool` on a `FastMCP("demo")` instance — and called it from the in-memory `Client(mcp)`. Everything ran in one Python process; the tools never touched a socket.

Real agents talk to MCP servers over a wire. That wire is HTTP, and the current transport in the MCP spec is **streamable-HTTP**. The promise FastMCP makes is that switching to HTTP costs you exactly one line of code: change `mcp.run(transport="stdio")` to `mcp.run(transport="streamable-http", host="127.0.0.1", port=8020)`. The tool functions don't change. Their decorators don't change. The schemas FastMCP generated from their type hints don't change. Only the runtime flag changes.

This notebook proves that claim: same two tools, served over HTTP on a background thread, called by the same `fastmcp.Client` class (just pointed at a URL instead of at the server object).

> *Targets `fastmcp` 2.x.*
```

- [ ] **Step 2: Add a markdown cell with "What you'll learn"**

Cell content (markdown):

```markdown
## What you'll learn

- How to run a `FastMCP` instance over HTTP with `mcp.run(transport="streamable-http", host="127.0.0.1", port=8020)`.
- Why FastMCP HTTP servers are typically launched on a background `threading.Thread(daemon=True)` inside a notebook — and what tradeoff that buys.
- That `fastmcp.Client` accepts **either** an in-memory `FastMCP` instance **or** a URL — the same class drives both.
- How the MCP HTTP endpoint is conventionally served at `/mcp/` in FastMCP 2.x.
- The difference between the current **`streamable-http`** transport and the older **SSE** transport, and why MCP spec rev 2025-03-26+ chose streamable-http as the default.
- The daemon-thread cleanup pattern: no explicit `shutdown()` call is needed in FastMCP 2.x — the daemon thread exits when the kernel shuts down.
```

- [ ] **Step 3: Verify cells render**

Open the notebook in Jupyter/VS Code and confirm both cells render as markdown.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/03_fastmcp_http_server.ipynb
git commit -m "feat(fastmcp): add intro markdown to notebook 03"
```

---

## Task 3: Add the setup cell

**Files:**
- Modify: `fastmcp/03_fastmcp_http_server.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 1. Setup

We need three things in scope:

1. The `FastMCP` class (server) and `Client` class (client) from `fastmcp`.
2. `threading` so we can run `mcp.run(...)` on a background daemon thread — `mcp.run(...)` blocks the calling thread, so it can't be called directly from a notebook cell that also wants to make client calls afterward.
3. `time` for a small sleep that lets the HTTP server bind to its port before we hit it.

The helper `start_http_server_in_thread(mcp, host, port)` below takes a `FastMCP` instance and a `(host, port)` pair, launches `mcp.run(transport="streamable-http", host=host, port=port)` on a daemon thread, and returns that thread. The thread is `daemon=True` so it dies automatically when the kernel shuts down.

> **Quirk:** FastMCP's `mcp.run(...)` is **blocking**. In a script you call it as the last line of `__main__`. In a notebook, where you want to also call the server from the same process, you put it on a daemon thread and rely on daemon-thread cleanup at kernel shutdown — there is no need to call `mcp.shutdown()` or similar in 2.x.
```

- [ ] **Step 2: Add the setup code cell**

```python
import threading
import time

from fastmcp import Client, FastMCP

_server_threads: list[threading.Thread] = []


def start_http_server_in_thread(
    mcp: FastMCP,
    host: str = "127.0.0.1",
    port: int = 8020,
) -> threading.Thread:
    """Run `mcp.run(transport="streamable-http", ...)` on a daemon thread.

    Blocking call lives on the thread, so the notebook keeps control.
    The daemon flag ensures the thread is torn down when the kernel exits.
    """
    def _serve() -> None:
        mcp.run(transport="streamable-http", host=host, port=port)

    thread = threading.Thread(target=_serve, daemon=True)
    thread.start()
    _server_threads.append(thread)
    return thread


print("Setup OK")
```

- [ ] **Step 3: Run the cell**

Expected output: `Setup OK`. If imports fail:

```bash
pip install "fastmcp>=2,<3"
```

- [ ] **Step 4: Commit**

```bash
git add fastmcp/03_fastmcp_http_server.ipynb
git commit -m "feat(fastmcp): add setup cell to notebook 03"
```

---

## Task 4: Section 2 — Define a FastMCP server with two tools

**Files:**
- Modify: `fastmcp/03_fastmcp_http_server.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 2. Define a FastMCP Server with Two Tools

This is the exact same server we built in notebook 02 — copied here so this notebook is self-contained. **The tool functions are unchanged.** The decorators are unchanged. The type hints are unchanged. FastMCP will derive the same JSON schemas from these signatures that it derived in notebook 02.

The only thing that will differ from notebook 02 is what we pass to `mcp.run(...)` in the next cell.

> **Quirk:** The tool definitions did not change — only the transport flag will. That is the entire point of this notebook.
```

- [ ] **Step 2: Add the server-definition code cell**

```python
mcp = FastMCP("demo")


@mcp.tool
def add(a: int, b: int) -> int:
    """Add two integers and return their sum."""
    return a + b


@mcp.tool
def greet(name: str) -> str:
    """Return a friendly greeting for the given name."""
    return f"Hello, {name}!"


print(f"FastMCP server defined: {mcp.name!r} with 2 tools (add, greet)")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
FastMCP server defined: 'demo' with 2 tools (add, greet)
```

- [ ] **Step 4: Commit**

```bash
git add fastmcp/03_fastmcp_http_server.ipynb
git commit -m "feat(fastmcp): define add and greet tools in notebook 03"
```

---

## Task 5: Section 3 — Run it over HTTP on a background thread

**Files:**
- Modify: `fastmcp/03_fastmcp_http_server.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 3. Run It over HTTP on a Background Thread

Now we cross from in-memory to over-the-wire. We launch the same `mcp` instance with `transport="streamable-http"`. FastMCP 2.x exposes the streamable-HTTP endpoint at the path `/mcp/` by default. After kicking off the thread, we sleep briefly (1.5s) so the underlying uvicorn server has time to bind to port 8020 before we try to call it. In a production launcher you'd poll a health endpoint instead — for a notebook, a short sleep is plenty.

The thread is `daemon=True`, so it dies automatically when this kernel shuts down. We will not call any `shutdown()` API on the server; FastMCP 2.x does not require one in this notebook pattern.
```

- [ ] **Step 2: Add the server-launch code cell**

```python
server_thread = start_http_server_in_thread(mcp, host="127.0.0.1", port=8020)

# Give uvicorn a moment to bind before we call the server.
time.sleep(1.5)

print(f"Server thread alive: {server_thread.is_alive()}")
print("FastMCP HTTP server should be listening on http://127.0.0.1:8020/mcp/")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Server thread alive: True
FastMCP HTTP server should be listening on http://127.0.0.1:8020/mcp/
```

If port 8020 is already in use, free it first (only kill processes you recognize):

```bash
lsof -ti tcp:8020 | xargs kill -9
```

- [ ] **Step 4: Commit**

```bash
git add fastmcp/03_fastmcp_http_server.ipynb
git commit -m "feat(fastmcp): start HTTP server on background thread in notebook 03"
```

---

## Task 6: Section 4 — Call it from an HTTP MCP client

**Files:**
- Modify: `fastmcp/03_fastmcp_http_server.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 4. Call It from an HTTP MCP Client

Here is the payoff: the same `fastmcp.Client` class that drove the in-memory server in notebook 02 now drives the HTTP server — we just pass it a URL instead of a `FastMCP` instance. `Client` is async; we use `async with Client(url) as client:` and then `await client.list_tools()` and `await client.call_tool(name, args)`.

The endpoint path is `/mcp/` in FastMCP 2.x. If you get a connection error from `http://127.0.0.1:8020/mcp/`, try `http://127.0.0.1:8020/` — some FastMCP releases serve it at the root.

> **Quirk:** The same `Client` class drives in-memory and HTTP servers — just pass a URL. There is no separate `HTTPClient` or `RemoteClient` to learn.
```

- [ ] **Step 2: Add the client-call code cell**

```python
SERVER_URL = "http://127.0.0.1:8020/mcp/"


async def call_remote_tools() -> None:
    async with Client(SERVER_URL) as client:
        tools = await client.list_tools()
        print(f"Remote tools advertised by server: {[t.name for t in tools]}")

        add_result = await client.call_tool("add", {"a": 2, "b": 3})
        print(f"add(2, 3) -> {add_result.data}")

        greet_result = await client.call_tool("greet", {"name": "Nimesh"})
        print(f"greet('Nimesh') -> {greet_result.data!r}")


await call_remote_tools()
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Remote tools advertised by server: ['add', 'greet']
add(2, 3) -> 5
greet('Nimesh') -> 'Hello, Nimesh!'
```

If the cell errors with a connection refused on `/mcp/`, retry with `SERVER_URL = "http://127.0.0.1:8020/"` (some FastMCP releases serve the endpoint at the root). If `await` at top level is rejected by your kernel, prepend the call with `import asyncio; asyncio.run(call_remote_tools())` instead — modern Jupyter kernels accept top-level `await` natively.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/03_fastmcp_http_server.ipynb
git commit -m "feat(fastmcp): call HTTP server from fastmcp.Client in notebook 03"
```

---

## Task 7: Section 5 — `streamable-http` vs SSE

**Files:**
- Modify: `fastmcp/03_fastmcp_http_server.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 5. `streamable-http` vs SSE

MCP has had two HTTP-based transports. **SSE** (Server-Sent Events) was the original one: the server held one long-lived HTTP response open and pushed events down it as a stream of `data:` lines, while the client posted requests over a separate HTTP channel. **`streamable-http`** is the current default since MCP spec rev 2025-03-26+: a single HTTP endpoint handles both directions, with the server free to keep the response open and stream chunks back when it has more to say. `streamable-http` collapses the two-channel design of SSE into one, simplifies reconnection, and is what every new MCP client and server should target. SSE remains in the spec for backward compatibility with older servers; if you see a server documented as "SSE-only," it predates this transition. FastMCP supports both via the `transport=` flag, but we use `streamable-http` throughout this series because it is the modern default.
```

- [ ] **Step 2: Verify the cell renders**

The paragraph should render with `streamable-http` and `SSE` rendered as inline code where backticked.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/03_fastmcp_http_server.ipynb
git commit -m "feat(fastmcp): contrast streamable-http vs SSE in notebook 03"
```

---

## Task 8: Add "What you just learned"

**Files:**
- Modify: `fastmcp/03_fastmcp_http_server.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## What you just learned

- FastMCP transports are runtime flags, not code rewrites: the same `@mcp.tool` functions ran over HTTP with no change to their definitions.
- `mcp.run(transport="streamable-http", host="127.0.0.1", port=8020)` is the HTTP launcher. FastMCP 2.x serves the endpoint at `/mcp/` by default.
- Inside a notebook, `mcp.run(...)` goes on a `threading.Thread(daemon=True)` because it blocks. The daemon flag means no explicit shutdown is needed at kernel exit.
- The same `fastmcp.Client` class drives both the in-memory server (notebook 02) and the HTTP server (this notebook). The only difference is what you pass to the constructor: a `FastMCP` instance vs. a URL.
- `streamable-http` is the current MCP HTTP transport since spec rev 2025-03-26+; SSE is the older two-channel transport it replaces.
```

- [ ] **Step 2: Commit**

```bash
git add fastmcp/03_fastmcp_http_server.ipynb
git commit -m "feat(fastmcp): add 'what you just learned' recap to notebook 03"
```

---

## Task 9: Add "What's missing"

**Files:**
- Modify: `fastmcp/03_fastmcp_http_server.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## What's missing

Right now, anyone who can reach `http://127.0.0.1:8020/mcp/` can call our tools. There is no authentication, no token, no way for the server to even know *who* the caller is. That is fine on `127.0.0.1` in a notebook — it is not fine the moment this server moves anywhere a second person can hit it.

In **notebook 04** we keep the HTTP transport from this notebook and layer in authentication using FastMCP's built-in auth provider. We will cover three modes — no auth (the implicit default we have been running so far), bearer token validated server-side, and OAuth2 client credentials with a mocked IdP — and we will see how FastMCP makes auth a first-class server constructor parameter rather than middleware you wire by hand.
```

- [ ] **Step 2: Commit**

```bash
git add fastmcp/03_fastmcp_http_server.ipynb
git commit -m "feat(fastmcp): add 'what's missing' teaser to notebook 03"
```

---

## Task 10: Section 6 — Cleanup

**Files:**
- Modify: `fastmcp/03_fastmcp_http_server.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 6. Cleanup

There is no explicit shutdown call in this notebook. FastMCP 2.x's `mcp.run(...)` blocks the calling thread; we put it on a daemon thread; daemon threads are killed automatically when the Python process exits. Restart the kernel (or close the notebook) and the port is released. We confirm the thread is still alive below — it should be, until the kernel exits — and that is the whole story.

If you want to free port 8020 right now without restarting the kernel, the simplest move is **Kernel → Restart**. There is no clean per-server shutdown API surfaced here in 2.x.
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
git add fastmcp/03_fastmcp_http_server.ipynb
git commit -m "feat(fastmcp): add cleanup section to notebook 03"
```

---

## Task 11: Sanity-check the full cell sequence

**Files:** none (verification only).

- [ ] **Step 1: List cells and confirm the order**

```bash
python -c "import json; nb = json.load(open('fastmcp/03_fastmcp_http_server.ipynb')); print(len(nb['cells']), 'cells'); [print(i, c['cell_type'], (c['source'][0] if c['source'] else '<empty>').rstrip()[:80]) for i, c in enumerate(nb['cells'])]"
```

Expected: the cell list reads in this order (cell types in parentheses):

1. (markdown) `# 03 — FastMCP over HTTP: Swap the Transport, Not the Code`
2. (markdown) `## What you'll learn`
3. (markdown) `## 1. Setup`
4. (code) `import threading`
5. (markdown) `## 2. Define a FastMCP Server with Two Tools`
6. (code) `mcp = FastMCP("demo")`
7. (markdown) `## 3. Run It over HTTP on a Background Thread`
8. (code) `server_thread = start_http_server_in_thread(...)`
9. (markdown) `## 4. Call It from an HTTP MCP Client`
10. (code) `SERVER_URL = "http://127.0.0.1:8020/mcp/"`
11. (markdown) `## 5. \`streamable-http\` vs SSE`
12. (markdown) `## What you just learned`
13. (markdown) `## What's missing`
14. (markdown) `## 6. Cleanup`
15. (code) `print(f"Server thread still running: ...`

Total: 15 cells.

- [ ] **Step 2: Confirm no stray placeholder strings**

```bash
grep -nE 'TODO|FIXME|XXX|placeholder' fastmcp/03_fastmcp_http_server.ipynb || echo "OK no placeholders"
```

Expected: `OK no placeholders`.

- [ ] **Step 3: No commit needed (verification only).** If a structural issue is found, fix it in the appropriate prior task and re-run this task.

---

## Task 12: End-to-end verification in a fresh kernel

**Files:** none (verification only).

- [ ] **Step 1: Execute the notebook top-to-bottom in a clean kernel**

```bash
jupyter nbconvert --to notebook --execute --inplace fastmcp/03_fastmcp_http_server.ipynb
```

Expected: succeeds with no errors. The notebook is rewritten with embedded outputs from the fresh run.

- [ ] **Step 2: Verify expected outputs in the executed notebook**

Cells should contain, in order:

1. `Setup OK`
2. `FastMCP server defined: 'demo' with 2 tools (add, greet)`
3. `Server thread alive: True` followed by `FastMCP HTTP server should be listening on http://127.0.0.1:8020/mcp/`
4. The three client-call lines:
   ```
   Remote tools advertised by server: ['add', 'greet']
   add(2, 3) -> 5
   greet('Nimesh') -> 'Hello, Nimesh!'
   ```
5. `Server thread still running: True` followed by `(The daemon thread will exit when the kernel restarts; no explicit shutdown call is needed in 2.x.)`

No cell raises an unhandled exception. The Section 4 cell takes the full ~1.5s sleep from Section 3 plus a small async round-trip.

If a port-collision error appears, free port 8020 first: `lsof -ti tcp:8020 | xargs kill -9` (only kill processes you recognize).

If the Section 4 cell errors with a connection refused on `/mcp/`, change `SERVER_URL` in Task 6's code cell to `"http://127.0.0.1:8020/"` and re-run end-to-end. Document the working path in the cell's prose if the change was needed.

- [ ] **Step 3: Confirm port is still bound (server is daemon-threaded so it stays up until kernel exit)**

```bash
lsof -iTCP:8020 -sTCP:LISTEN
```

Expected: one entry showing the Python process owning the port — this is the daemon thread still serving inside the notebook kernel. (Unlike notebook 02 which has nothing to bind, and unlike `a2a/05` whose `shutdown_all_servers()` releases its port, this notebook leaves the port held by design until kernel shutdown.)

- [ ] **Step 4: Commit the clean run**

```bash
git add fastmcp/03_fastmcp_http_server.ipynb
git commit -m "chore(fastmcp): commit clean fresh-kernel run of notebook 03"
```

---

## Done

After Task 12 passes, notebook 03 is complete. The next plan to write is `docs/superpowers/plans/2026-05-18-fastmcp-notebook-04-with-auth.md`, which keeps the HTTP transport from this notebook and layers in FastMCP's built-in auth provider (no auth → bearer token → mocked OAuth2 client credentials).
