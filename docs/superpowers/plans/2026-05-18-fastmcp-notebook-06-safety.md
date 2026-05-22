# FastMCP Notebook 06 — `06_fastmcp_prompt_injection_and_safety.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the sixth notebook in the FastMCP learning series — demonstrate safe handling of user-controlled tool inputs with FastMCP. By the end the learner can declare Pydantic models as tool parameters (getting input validation + JSON-schema generation for free), allowlist resource URIs, sanitize tool outputs, and refuse dangerous arguments such as filesystem path traversals.

**Architecture:** A single Jupyter notebook that defines one `FastMCP` instance with a handful of deliberately-paranoid tools and one parameterized resource. All interaction is via the **in-memory `Client(mcp)`** — no HTTP, no subprocess, no auth, no real LLM. A temp sandbox directory at `/tmp/fastmcp-notebook-06/` is created at the top of the notebook (with two small text files) and deleted at the bottom, so the `read_file` example never touches real user files.

**Tech Stack:** Python 3.11+, Jupyter, `fastmcp` 2.x, `pydantic` v2, the stdlib (`pathlib`, `re`, `shutil`, `tempfile` indirectly via a fixed path under `/tmp/`).

**Companion spec:** `docs/superpowers/specs/2026-05-18-fastmcp-learning-series-design.md` (notebook 06 section).

**Port assignment:** None — this notebook is entirely in-process via `Client(mcp)`. No background HTTP server.

---

## File Structure

- **Create:** `fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb` — the entire notebook, self-contained (no imports from previous notebooks in the series).
- **Modify:** none.
- **No separate test files** — verification is the notebook running top-to-bottom in a fresh kernel (Task 13).

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown)
2. **"What you'll learn"** (markdown bullets)
3. **Section 1: Setup** (markdown + 1 code cell: imports + create the temp sandbox)
4. **Section 2: Pydantic models as tool parameters** (markdown + Quirk callout + 2 code cells: define `CreateUser` + the tool, then call with valid and invalid payloads via `Client(mcp)`)
5. **Section 3: Allowlisted resources** (markdown + 2 code cells: define the resource, then fetch a valid slug and a disallowed one)
6. **Section 4: Output sanitization** (markdown + 2 code cells: define the sanitizer + the tool, then call with control-character and over-long input)
7. **Section 5: Refusing dangerous arguments** (markdown + 2 code cells: define `read_file`, then call with a valid file and a traversal attempt)
8. **"What you just learned"** (markdown)
9. **"What's missing"** (markdown — teases notebook 07, resources & prompts in depth)
10. **Section 6: Cleanup** (code: delete the temp sandbox)

---

## Task 1: Create the notebook scaffold

**Files:**
- Create: `fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb`

- [ ] **Step 1: Confirm the `fastmcp/` directory exists**

```bash
ls -la fastmcp/
```

Expected: directory exists with at least notebooks 01–05 already present from earlier work in the series. If it doesn't exist yet, create it:

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
python -c "import json; json.load(open('fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb'))"
ls -la fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb
```

Expected: no exception, file non-zero size.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb
git commit -m "feat(fastmcp): scaffold 06_fastmcp_prompt_injection_and_safety.ipynb"
```

---

## Task 2: Add title and "Why this notebook exists"

**Files:**
- Modify: `fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb` (add cells 1–2)

- [ ] **Step 1: Add a markdown cell with title and motivation**

Cell content (markdown):

```markdown
# 06 — Prompt Injection & Safety: Validation and Untrusted Input

## Why this notebook exists

Notebooks 02–05 treated tool inputs as if they came from a friendly developer at the REPL. They don't. In production, MCP tool arguments come from an LLM that is itself fed user-controlled text — which means **every tool argument is effectively attacker-controlled**. Resource URIs, file paths, structured payloads: all of them. A tool that quietly does what its arguments say can be turned against you by a single injected instruction in a user message.

The defensive posture matters more than any single trick. This notebook walks through four concrete patterns FastMCP makes easy:

1. **Pydantic models as tool parameters** — validate input *before* the tool body runs, and get the JSON schema for free.
2. **Allowlisted resources** — refuse URIs you didn't pre-approve.
3. **Output sanitization** — strip control characters and bound length on the way out.
4. **Refusing dangerous arguments** — reject path traversals and out-of-sandbox reads.

We run everything through the **in-memory `Client(mcp)`** — no HTTP, no LLM, no API keys. The point is the framework's affordances, not the wire format.

> *Previously in `mcp/05`:* the raw-SDK series introduced production patterns (lifespan, structured errors). FastMCP gives us those same tools *plus* native Pydantic validation on parameters, which removes the whole class of "I forgot to validate that field" bugs that raw-SDK servers tend to grow.
```

- [ ] **Step 2: Add a markdown cell with "What you'll learn"**

Cell content (markdown):

```markdown
## What you'll learn

- How to declare a **Pydantic `BaseModel` as a tool parameter** so FastMCP validates input and generates the JSON schema in one step.
- How to **allowlist URIs** in a `@mcp.resource(...)` handler and refuse anything outside the list.
- How to **sanitize tool output** — strip ANSI/control characters and bound length — so a malicious string can't bleed into your terminal or downstream UI.
- How to **refuse dangerous arguments** like filesystem path traversals (`../etc/passwd`) before they reach `open()`.
- Why "validate at the edge" beats "validate inside every tool body" for MCP servers.
```

- [ ] **Step 3: Verify cells render**

Open the notebook in Jupyter/VS Code and confirm both cells render as markdown.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb
git commit -m "feat(fastmcp): add intro markdown to notebook 06"
```

---

## Task 3: Add the setup cell (imports + temp sandbox)

**Files:**
- Modify: `fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 1. Setup

We import FastMCP, Pydantic, and a few stdlib helpers. We also create a small **sandbox directory** at `/tmp/fastmcp-notebook-06/` with two harmless text files — this is the *only* directory the `read_file` tool later in this notebook is allowed to read from. Everything outside it is rejected. The sandbox is deleted at the very end of the notebook (Section 6).

> **API note:** `ToolError` is FastMCP's canonical way to signal a refusal from inside a tool body — FastMCP serializes it into a proper MCP error response. The import path is `from fastmcp.exceptions import ToolError` in 2.x; if your local `fastmcp` version raises `ImportError`, try `from fastmcp import ToolError` and pin the API note in a follow-up commit.
```

- [ ] **Step 2: Add the setup code cell**

```python
import re
import shutil
from pathlib import Path

from fastmcp import Client, FastMCP
from fastmcp.exceptions import ToolError
from pydantic import BaseModel, Field, field_validator

# --- Sandbox the filesystem-read example to a temp directory we own. ---
SANDBOX_DIR = Path("/tmp/fastmcp-notebook-06")
SANDBOX_DIR.mkdir(parents=True, exist_ok=True)
(SANDBOX_DIR / "hello.txt").write_text("hello from the sandbox\n", encoding="utf-8")
(SANDBOX_DIR / "notes.txt").write_text("line 1\nline 2\nline 3\n", encoding="utf-8")

# Resolve once so we can compare against later — symlinks, .., etc. all
# normalize through this single anchor.
SANDBOX_ROOT = SANDBOX_DIR.resolve()

# The FastMCP server instance every tool/resource in this notebook attaches to.
mcp = FastMCP("safety-demo")

print("Setup OK")
print(f"Sandbox: {SANDBOX_ROOT}")
print("Files :", sorted(p.name for p in SANDBOX_ROOT.iterdir()))
```

- [ ] **Step 3: Run the cell**

Expected output:

```
Setup OK
Sandbox: /tmp/fastmcp-notebook-06
Files : ['hello.txt', 'notes.txt']
```

If imports fail with `ModuleNotFoundError: No module named 'fastmcp'`, the FastMCP series prerequisite (`pip install fastmcp pydantic`) hasn't been satisfied — install per notebook 02.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb
git commit -m "feat(fastmcp): add setup + sandbox cell to notebook 06"
```

---

## Task 4: Section 2 — Pydantic models as tool parameters (intro markdown)

**Files:**
- Modify: `fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 2. Pydantic models as tool parameters

The cheapest, highest-leverage validation in FastMCP is just: **type the parameter with a Pydantic model**. FastMCP introspects the model and uses it for two things at once:

1. **Input validation** — incoming arguments are coerced/validated against the model *before* your tool body runs. If a field is missing, the wrong type, or fails a `field_validator`, FastMCP rejects the call with a structured error.
2. **JSON schema generation** — the model's schema becomes the tool's `inputSchema`. No hand-written JSON.

The tool body only ever sees a fully-validated instance — there's no need for a defensive `if not isinstance(...)` ladder at the top of every tool.

> **Quirk:** because FastMCP uses Pydantic natively, declaring a Pydantic param model gives you input validation **and** schema generation in one step — no separate JSON Schema writing, no separate validator code. The raw `mcp` SDK requires you to hand-write the `inputSchema` JSON and then re-validate the args yourself inside `call_tool`.
```

- [ ] **Step 2: Commit**

```bash
git add fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb
git commit -m "feat(fastmcp): explain Pydantic-as-param concept in notebook 06"
```

---

## Task 5: Section 2 — Define the model and the tool

**Files:**
- Modify: `fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb` (add 1 code cell)

- [ ] **Step 1: Add the model + tool code cell**

```python
class CreateUser(BaseModel):
    """A new user the caller wants to create."""

    username: str = Field(..., min_length=3, max_length=32)
    email: str = Field(..., min_length=5, max_length=254)
    age: int = Field(..., ge=13, le=120)

    @field_validator("username")
    @classmethod
    def username_is_alnum(cls, v: str) -> str:
        if not re.fullmatch(r"[A-Za-z0-9_]+", v):
            raise ValueError("username must be letters, digits, or underscore")
        return v

    @field_validator("email")
    @classmethod
    def email_has_at(cls, v: str) -> str:
        # Deliberately minimal — full RFC 5322 is its own rabbit hole.
        if "@" not in v or "." not in v.split("@", 1)[1]:
            raise ValueError("email must contain '@' and a '.' in the domain")
        return v


@mcp.tool
def create_user(user: CreateUser) -> dict:
    """Pretend to create a user. Returns the canonical, validated payload."""
    # If we got here, `user` is already validated — no defensive checks needed.
    return {
        "status": "created",
        "username": user.username,
        "email": user.email,
        "age": user.age,
    }


print("Registered tool:", "create_user")
```

- [ ] **Step 2: Run the cell and verify**

Expected output:

```
Registered tool: create_user
```

- [ ] **Step 3: Commit**

```bash
git add fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb
git commit -m "feat(fastmcp): add CreateUser Pydantic-param tool in notebook 06"
```

---

## Task 6: Section 2 — Call with valid and invalid payloads

**Files:**
- Modify: `fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb` (add 1 code cell)

- [ ] **Step 1: Add the in-memory client call cell**

```python
async def demo_create_user() -> None:
    async with Client(mcp) as client:
        # --- Valid payload: passes Pydantic validation and the tool body runs.
        good = await client.call_tool(
            "create_user",
            {"user": {"username": "ada", "email": "ada@example.com", "age": 36}},
        )
        print("VALID  ->", good.data)

        # --- Invalid payload: missing `age`, bad `username`, bad `email`.
        # FastMCP rejects the call *before* the tool body runs.
        try:
            await client.call_tool(
                "create_user",
                {"user": {"username": "ad", "email": "not-an-email"}},
            )
        except Exception as exc:
            print("INVALID->", type(exc).__name__)
            # Print a few lines of the message so the schema-error structure is visible
            # without dumping a wall of text.
            msg = str(exc)
            for line in msg.splitlines()[:8]:
                print("        ", line)


await demo_create_user()
```

- [ ] **Step 2: Run the cell and verify**

Expected output (exception class name and exact wording may differ slightly between FastMCP minor versions; what matters is *the second call raises*):

```
VALID  -> {'status': 'created', 'username': 'ada', 'email': 'ada@example.com', 'age': 36}
INVALID-> ToolError
         Input validation error for tool 'create_user':
         - user.username: String should have at least 3 characters
         - user.email: email must contain '@' and a '.' in the domain
         - user.age: Field required
```

> **API note:** The exception type FastMCP raises for failed input validation may be `ToolError`, `ValidationError`, or a wrapped `McpError` depending on the 2.x minor version. The key behavior to verify is that the **tool body is never entered** — add a `print` at the top of `create_user` while debugging if you need to confirm.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb
git commit -m "feat(fastmcp): demo valid+invalid create_user payloads in notebook 06"
```

---

## Task 7: Section 3 — Allowlisted resources

**Files:**
- Modify: `fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb` (add markdown + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 3. Allowlisted resources

A `@mcp.resource("docs://{slug}")` handler turns its URI template parameters into Python function arguments. The framework gives you the slug; what you *do* with it is up to you.

The mistake is to treat the slug as already-trusted because it's "just a URI parameter." It isn't — it's the same attacker-controlled string as any other tool argument, only dressed up differently. The fix is the same as for any input: **check it against an allowlist, refuse anything else.**

We'll publish three known docs (`intro`, `setup`, `faq`) and watch the resource handler reject everything else.
```

- [ ] **Step 2: Add the resource definition code cell**

```python
DOCS = {
    "intro": "Welcome to the safety demo.",
    "setup": "Run `pip install fastmcp pydantic` and you're good.",
    "faq":   "Q: Is this real?  A: No, it's a notebook.",
}


@mcp.resource("docs://{slug}")
def read_doc(slug: str) -> str:
    """Return a known doc by slug, refusing anything outside the allowlist."""
    if slug not in DOCS:
        # Listing the allowed values in the error is fine here — the set is
        # public knowledge. For a private allowlist you'd say "not found".
        raise ToolError(
            f"unknown doc slug {slug!r}; allowed: {sorted(DOCS)}"
        )
    return DOCS[slug]


print("Registered resource template: docs://{slug}")
print("Allowlisted slugs:", sorted(DOCS))
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Registered resource template: docs://{slug}
Allowlisted slugs: ['faq', 'intro', 'setup']
```

- [ ] **Step 4: Add the resource fetch cell**

```python
async def demo_docs() -> None:
    async with Client(mcp) as client:
        # --- Valid slug: returns the doc body.
        good = await client.read_resource("docs://intro")
        print("VALID  ->", good[0].text)

        # --- Disallowed slug: refused with a ToolError-derived message.
        try:
            await client.read_resource("docs://secrets")
        except Exception as exc:
            print("REFUSED->", type(exc).__name__, "|", str(exc).splitlines()[0])


await demo_docs()
```

- [ ] **Step 5: Run the cell and verify**

Expected output (the leading line of the refusal message is what we assert on; exception class may be `ToolError` or `McpError` depending on FastMCP minor version):

```
VALID  -> Welcome to the safety demo.
REFUSED-> ToolError | unknown doc slug 'secrets'; allowed: ['faq', 'intro', 'setup']
```

- [ ] **Step 6: Commit**

```bash
git add fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb
git commit -m "feat(fastmcp): demo allowlisted resource refusal in notebook 06"
```

---

## Task 8: Section 4 — Output sanitization (intro + sanitizer)

**Files:**
- Modify: `fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb` (add markdown + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 4. Output sanitization

Even after you've validated *inputs*, you still need to think about what's leaving the tool. Tool output flows into the LLM's context, into terminals, sometimes into HTML — and a string like `"OK\x1b[2J\x1b[Hgotcha"` can do real damage if it lands in the wrong renderer (the LLM may also "trust" embedded instructions it sees).

We handle two concerns:

1. **Strip control / ANSI-escape characters** so the output is pure printable text.
2. **Bound the length** so a tool can't unilaterally dump a megabyte into the conversation.

The pattern: wrap the tool body in a small `sanitize()` function. Tool authors don't have to remember — the function does the work in one place.
```

- [ ] **Step 2: Add the sanitizer + tool code cell**

```python
MAX_OUTPUT_LEN = 200

# Strip ANSI escape sequences (CSI / OSC) and any other C0/C1 control chars
# except whitespace we actually want (\t, \n).
_ANSI_RE = re.compile(r"\x1b\[[0-?]*[ -/]*[@-~]")
_CTRL_RE = re.compile(r"[\x00-\x08\x0b-\x1f\x7f-\x9f]")


def sanitize(text: str, max_len: int = MAX_OUTPUT_LEN) -> str:
    """Strip ANSI/control chars and truncate to `max_len` graphemes."""
    cleaned = _ANSI_RE.sub("", text)
    cleaned = _CTRL_RE.sub("", cleaned)
    if len(cleaned) > max_len:
        cleaned = cleaned[: max_len - 1] + "…"  # ellipsis
    return cleaned


@mcp.tool
def echo_safely(message: str) -> str:
    """Echo `message` after running it through the output sanitizer."""
    # Imagine `message` was looked up from some untrusted source rather than
    # passed in directly — the sanitizer fires either way.
    return sanitize(message)


print("Registered tool:", "echo_safely")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Registered tool: echo_safely
```

- [ ] **Step 4: Commit**

```bash
git add fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb
git commit -m "feat(fastmcp): add output sanitizer + echo_safely tool in notebook 06"
```

---

## Task 9: Section 4 — Call with malicious input

**Files:**
- Modify: `fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb` (add 1 code cell)

- [ ] **Step 1: Add the echo_safely demo cell**

```python
async def demo_sanitize() -> None:
    # Build the nasty string in Python so the notebook source stays readable:
    # "OK" + ANSI clear-screen + cursor-home + a null byte + "gotcha"
    nasty = "OK\x1b[2J\x1b[H\x00gotcha"
    long  = "A" * 500

    async with Client(mcp) as client:
        r1 = await client.call_tool("echo_safely", {"message": nasty})
        print("control chars stripped:")
        print("  in :", repr(nasty))
        print("  out:", repr(r1.data))

        r2 = await client.call_tool("echo_safely", {"message": long})
        print(f"long input bounded to <= {MAX_OUTPUT_LEN} chars:")
        print("  in  len:", len(long))
        print("  out len:", len(r2.data))
        print("  out tail:", repr(r2.data[-5:]))


await demo_sanitize()
```

- [ ] **Step 2: Run the cell and verify**

Expected output:

```
control chars stripped:
  in : 'OK\x1b[2J\x1b[H\x00gotcha'
  out: 'OKgotcha'
long input bounded to <= 200 chars:
  in  len: 500
  out len: 200
  out tail: 'AAAA…'
```

- [ ] **Step 3: Commit**

```bash
git add fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb
git commit -m "feat(fastmcp): demo output sanitization in notebook 06"
```

---

## Task 10: Section 5 — Refusing dangerous arguments (intro + tool)

**Files:**
- Modify: `fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb` (add markdown + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 5. Refusing dangerous arguments

Some inputs *look* fine until you act on them. The canonical example is a filesystem path: `"notes.txt"` is fine, `"../etc/passwd"` is not, and a string compare won't catch the second one because it's "valid" as a path.

The defensive pattern is:

1. Reject any path that contains `..` outright — that's the cheap check.
2. **Resolve** the path against an allowlisted base directory.
3. Verify the resolved path is **still inside** that base after resolution (symlinks, `..`, absolute paths all collapse here).
4. Only then call `open()`.

Refusal is signalled with `ToolError("reason")` so the MCP client sees a clean structured error instead of a stack trace.
```

- [ ] **Step 2: Add the read_file tool code cell**

```python
@mcp.tool
def read_file(path: str) -> str:
    """Read a file from the sandbox. Refuses anything outside SANDBOX_ROOT."""

    # Cheap-and-loud rejection of obvious traversal patterns.
    if ".." in Path(path).parts:
        raise ToolError(f"path traversal not allowed: {path!r}")

    # Resolve relative to the sandbox; if the caller gives us an absolute path,
    # `(SANDBOX_ROOT / path)` will *replace* the sandbox prefix — but the next
    # check will catch that.
    candidate = (SANDBOX_ROOT / path).resolve()

    # The actual boundary check: is `candidate` still inside the sandbox?
    if SANDBOX_ROOT not in candidate.parents and candidate != SANDBOX_ROOT:
        raise ToolError(f"path outside sandbox: {path!r}")

    if not candidate.is_file():
        raise ToolError(f"not a file: {path!r}")

    return candidate.read_text(encoding="utf-8")


print("Registered tool:", "read_file")
print("Sandbox root   :", SANDBOX_ROOT)
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Registered tool: read_file
Sandbox root   : /tmp/fastmcp-notebook-06
```

- [ ] **Step 4: Commit**

```bash
git add fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb
git commit -m "feat(fastmcp): add sandboxed read_file tool in notebook 06"
```

---

## Task 11: Section 5 — Demonstrate refusal on traversal

**Files:**
- Modify: `fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb` (add 1 code cell)

- [ ] **Step 1: Add the read_file demo cell**

```python
async def demo_read_file() -> None:
    async with Client(mcp) as client:
        # --- Valid read: file lives inside the sandbox.
        good = await client.call_tool("read_file", {"path": "hello.txt"})
        print("VALID    ->", repr(good.data))

        # --- Traversal attempt: `..` should be refused immediately.
        try:
            await client.call_tool("read_file", {"path": "../etc/passwd"})
        except Exception as exc:
            print("REFUSED  ->", type(exc).__name__, "|", str(exc).splitlines()[0])

        # --- Absolute-path attempt: even without `..`, an absolute path outside
        # the sandbox should be refused by the boundary check.
        try:
            await client.call_tool("read_file", {"path": "/etc/passwd"})
        except Exception as exc:
            print("REFUSED  ->", type(exc).__name__, "|", str(exc).splitlines()[0])


await demo_read_file()
```

- [ ] **Step 2: Run the cell and verify**

Expected output (exception class may be `ToolError` or `McpError` depending on FastMCP minor version; the leading message is what we assert on):

```
VALID    -> 'hello from the sandbox\n'
REFUSED  -> ToolError | path traversal not allowed: '../etc/passwd'
REFUSED  -> ToolError | path outside sandbox: '/etc/passwd'
```

- [ ] **Step 3: Commit**

```bash
git add fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb
git commit -m "feat(fastmcp): demo path-traversal refusal in notebook 06"
```

---

## Task 12: Closing recap, teaser, and cleanup

**Files:**
- Modify: `fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb` (add 2 markdown cells + 1 markdown header + 1 code cell)

- [ ] **Step 1: Add the "What you just learned" markdown cell**

```markdown
## What you just learned

- A Pydantic `BaseModel` used as a tool parameter gives you **input validation and JSON-schema generation in one step** — no hand-written schema, no duplicate validator code.
- Resource URI parameters are **just as untrusted as any other tool argument**; an allowlist check in the handler is the simplest safe default.
- A small **`sanitize()`** function wrapped around tool output catches ANSI escapes, control characters, and runaway length — apply it once, benefit everywhere.
- Refusing dangerous arguments is the same pattern every time: a cheap syntactic check, then **resolve and re-anchor** against an allowlisted base directory. Use `ToolError("reason")` so clients see a structured error.
- The in-memory `Client(mcp)` makes all of this easy to demonstrate (and unit-test) — no HTTP, no subprocess.
```

- [ ] **Step 2: Add the "What's missing" markdown cell**

```markdown
## What's missing

We've focused on *tools* and one small *resource*. But FastMCP's resource and prompt surfaces have their own ergonomics worth a notebook's time — URI templates, static vs. parameterized resources, the `@mcp.prompt` decorator, and how the in-memory client lists and reads each kind.

In **notebook 07** we go deep on those two primitives: `@mcp.resource("notes://{topic}")` for parameterized read-only data, `@mcp.resource("config://app")` for static configuration, `@mcp.prompt` for reusable prompt templates, and the client-side calls to list and fetch each.
```

- [ ] **Step 3: Add a markdown cell for the cleanup section header**

```markdown
## 6. Cleanup

Delete the temp sandbox directory we created in Section 1. The notebook leaves no files on disk after this cell.
```

- [ ] **Step 4: Add the cleanup code cell**

```python
if SANDBOX_DIR.exists():
    shutil.rmtree(SANDBOX_DIR)
print("Sandbox deleted:", not SANDBOX_DIR.exists())
```

- [ ] **Step 5: Run the cleanup cell and verify**

Expected output:

```
Sandbox deleted: True
```

- [ ] **Step 6: Commit**

```bash
git add fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb
git commit -m "feat(fastmcp): add closing recap, teaser, and cleanup to notebook 06"
```

---

## Task 13: End-to-end verification in a fresh kernel

**Files:** none (verification only).

- [ ] **Step 1: Execute the notebook top-to-bottom in a clean kernel**

```bash
jupyter nbconvert --to notebook --execute --inplace fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb
```

Expected: succeeds with no errors. The notebook is rewritten with embedded outputs from the fresh run.

- [ ] **Step 2: Verify expected outputs in the executed notebook**

Cells should contain, in order:

1. `Setup OK` / `Sandbox: /tmp/fastmcp-notebook-06` / `Files : ['hello.txt', 'notes.txt']`
2. `Registered tool: create_user`
3. The `VALID  -> {...}` line followed by `INVALID-> <ExcName>` and a few schema-error lines.
4. `Registered resource template: docs://{slug}` / `Allowlisted slugs: ['faq', 'intro', 'setup']`
5. `VALID  -> Welcome to the safety demo.` / `REFUSED-> <ExcName> | unknown doc slug 'secrets'; allowed: [...]`
6. `Registered tool: echo_safely`
7. The control-chars-stripped + length-bounded `echo_safely` output block.
8. `Registered tool: read_file` / `Sandbox root   : /tmp/fastmcp-notebook-06`
9. The `VALID    -> ...` + two `REFUSED  -> ...` lines from `read_file`.
10. `Sandbox deleted: True`

No cell raises an unhandled exception. The expected `INVALID->` and `REFUSED->` lines are **handled** exceptions — printed via `try/except` — and do not count as failures.

- [ ] **Step 3: Verify the sandbox is gone**

```bash
ls /tmp/fastmcp-notebook-06 2>&1 | head -1
```

Expected: `ls: /tmp/fastmcp-notebook-06: No such file or directory` (or equivalent — the directory must not exist).

- [ ] **Step 4: Commit the clean run**

```bash
git add fastmcp/06_fastmcp_prompt_injection_and_safety.ipynb
git commit -m "chore(fastmcp): commit clean fresh-kernel run of notebook 06"
```

---

## Done

After Task 13 passes, notebook 06 is complete. The next plan to write is `docs/superpowers/plans/2026-05-18-fastmcp-notebook-07-resources-and-prompts.md`, which goes deep on FastMCP's `@mcp.resource(...)` and `@mcp.prompt` decorators.
