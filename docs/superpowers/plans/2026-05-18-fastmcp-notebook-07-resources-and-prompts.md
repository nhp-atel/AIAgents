# FastMCP Notebook 07 — `07_fastmcp_resources_and_prompts.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the seventh notebook in the FastMCP learning series — show how FastMCP's decorator surface extends from tools to the **other two MCP primitives**: **resources** (read-only data identified by a URI) and **prompts** (reusable prompt templates). The learner ends able to define static and parameterized resources with `@mcp.resource`, define parameterless and parameterized prompts with `@mcp.prompt`, and exercise both from the in-memory `Client(mcp)` via `list_resources()`, `list_resource_templates()`, `read_resource()`, `list_prompts()`, and `get_prompt()`.

**Architecture:** A single Jupyter notebook that defines one `FastMCP("demo")` server with one static resource (`config://app`), one parameterized resource (`notes://{topic}`), one parameterless prompt (`explain_octopuses`), and one parameterized prompt (`summarize_topic`). No tools — they're covered exhaustively by notebooks 02–06. The notebook plays the role of an MCP *client* using the in-memory transport throughout: `async with Client(mcp) as client:` to list and fetch resources and prompts. No HTTP, no subprocess, no API keys.

**Tech Stack:** Python 3.11+, Jupyter, `fastmcp` (2.x line). Top-level `await` is used directly inside notebook cells (Jupyter supports it).

**Companion spec:** `docs/superpowers/specs/2026-05-18-fastmcp-learning-series-design.md` (notebook 07 section).

**Port assignment:** none — this notebook uses only the in-memory `Client(mcp)` transport. No background server is started.

---

## File Structure

- **Create:** `fastmcp/07_fastmcp_resources_and_prompts.ipynb` — the entire notebook, self-contained (no imports from prior notebooks).
- **Modify:** none.
- **No separate test files** — verification is the notebook running top-to-bottom in a fresh kernel (Task 13).

The `fastmcp/` folder is created by the notebook-01 plan, but Task 1 below does **not** assume it already exists — it includes a defensive `mkdir -p fastmcp` step.

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown)
2. **"What you'll learn"** (markdown bullets)
3. **Section 1: Setup** (code — imports + create `mcp = FastMCP("demo")`)
4. **Section 2: Static resources** (markdown + 2 code cells: define `config://app`, list resources, read it)
5. **Section 3: Parameterized resources** (markdown + 2 code cells: define `notes://{topic}` with an in-memory notes dict, list resource templates, read `notes://octopus` and `notes://rome`)
6. **Section 4: Prompts** (markdown + 2 code cells: define a parameterless prompt and a parameterized prompt, list prompts, fetch each)
7. **Section 5: All three primitives side by side** (markdown only — recap table mapping Tool / Resource / Prompt to what it is / when to use it / decorator)
8. **"What you just learned"** (markdown)
9. **"What's missing"** (markdown — teases notebook 08, real LLM driving FastMCP tools end-to-end)
10. **Section 6: Cleanup** (markdown + 1 code cell — minimal, since nothing was started)

---

## Task 1: Create the notebook scaffold

**Files:**
- Create: `fastmcp/07_fastmcp_resources_and_prompts.ipynb`

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
python -c "import json; json.load(open('fastmcp/07_fastmcp_resources_and_prompts.ipynb'))"
ls -la fastmcp/07_fastmcp_resources_and_prompts.ipynb
```

Expected: no exception, file non-zero size.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/07_fastmcp_resources_and_prompts.ipynb
git commit -m "feat(fastmcp): scaffold 07_fastmcp_resources_and_prompts.ipynb"
```

---

## Task 2: Add title and "Why this notebook exists"

**Files:**
- Modify: `fastmcp/07_fastmcp_resources_and_prompts.ipynb` (add cell 1)

- [ ] **Step 1: Add a markdown cell with title and motivation**

Cell content (markdown):

```markdown
# 07 — Resources and Prompts: The Other Two Primitives, the Easy Way

## Why this notebook exists

Notebooks **02 through 06** focused entirely on one MCP primitive: **tools**. We watched FastMCP collapse the cost of defining a tool from a hand-written JSON schema plus a dispatch branch (the raw-SDK shape from notebook 01) down to a single `@mcp.tool` decorator on a plain Python function. That's already most of what people reach for FastMCP for.

But MCP defines **three** server-side primitives, not one:

- **Tools** — functions a client can call (`add(2, 3) -> 5`).
- **Resources** — read-only data a client can fetch by URI (`config://app` -> a config blob).
- **Prompts** — reusable prompt templates a client can render with parameters (`summarize_topic(topic="octopus")` -> a list of MCP messages).

This notebook shows that FastMCP applies the **same decorator-driven ergonomics** to all three. `@mcp.resource("uri://path")` turns a plain Python function into a resource. `@mcp.prompt` turns a plain Python function into a prompt template. URI parameters in a resource path become function arguments automatically — no manual URI parsing. We'll exercise everything through the in-memory `Client(mcp)` from notebook 02, using a small handful of client methods we haven't met yet: `list_resources()`, `list_resource_templates()`, `read_resource()`, `list_prompts()`, and `get_prompt()`.
```

- [ ] **Step 2: Verify the cell renders**

Open the notebook in Jupyter/VS Code and confirm the cell renders as markdown.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/07_fastmcp_resources_and_prompts.ipynb
git commit -m "feat(fastmcp): add title and motivation to notebook 07"
```

---

## Task 3: Add "What you'll learn"

**Files:**
- Modify: `fastmcp/07_fastmcp_resources_and_prompts.ipynb` (add cell 2)

- [ ] **Step 1: Add a markdown cell with "What you'll learn"**

Cell content (markdown):

```markdown
## What you'll learn

- The three MCP server-side primitives: **tools**, **resources**, and **prompts** — and which one to reach for when.
- How to define a **static resource** with `@mcp.resource("config://app")` — a fixed URI that always returns the same payload.
- How to define a **parameterized resource** with `@mcp.resource("notes://{topic}")`, where URI parameters are auto-extracted into function arguments by FastMCP.
- Why static resources show up in `list_resources()` but parameterized resources show up in `list_resource_templates()` — and what the MCP spec means by a "resource template."
- How to define **prompts** with `@mcp.prompt` — both parameterless and parameterized — and how to fetch a rendered prompt from the client with `await client.get_prompt(name, {arg: value})`.
- Why all three primitives share the same decorator-surface ergonomics: same `mcp` object, same plain-Python-function-plus-decorator pattern.
```

- [ ] **Step 2: Verify the cell renders**

Open the notebook and confirm the bullet list renders.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/07_fastmcp_resources_and_prompts.ipynb
git commit -m "feat(fastmcp): add 'What you'll learn' to notebook 07"
```

---

## Task 4: Section 1 — Setup (imports + server object)

**Files:**
- Modify: `fastmcp/07_fastmcp_resources_and_prompts.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 1. Setup

We need the same two names from `fastmcp` we used in notebook 02 — `FastMCP` (the server class) and `Client` (the in-memory client). We'll also import `json` for pretty-printing fetched resource payloads.

If the import fails with `ModuleNotFoundError`, install FastMCP in your active environment:

```
pip install fastmcp
```

This notebook targets the current stable `fastmcp` 2.x line. We construct a single `FastMCP("demo")` server here and register one static resource, one parameterized resource, and two prompts against it across the next sections.
```

- [ ] **Step 2: Add the setup code cell**

```python
import json

from fastmcp import Client, FastMCP

mcp = FastMCP("demo")

print(f"Setup OK. Server: {mcp.name}")
```

- [ ] **Step 3: Run the cell**

Expected output:

```
Setup OK. Server: demo
```

If imports fail, run `pip install fastmcp` and re-execute.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/07_fastmcp_resources_and_prompts.ipynb
git commit -m "feat(fastmcp): add setup cell to notebook 07"
```

---

## Task 5: Section 2 — Static resources

**Files:**
- Modify: `fastmcp/07_fastmcp_resources_and_prompts.ipynb` (add markdown header + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 2. Static Resources

A **resource** in MCP is read-only data the client can fetch by URI. Think of it the same way you'd think of a file path or an HTTP GET endpoint — the client knows the URI, sends a `read_resource` request, and gets the payload back.

In FastMCP, you define a resource by decorating a function with `@mcp.resource("uri://path")`. The URI is the address the client uses to fetch it. The function's return value is what the client receives — FastMCP accepts plain Python types (`str`, `bytes`, `dict`) and handles serialization automatically. A `dict` becomes JSON; a `str` stays a string.

A **static resource** is one whose URI has no parameters — the same URI always returns the same payload. We'll start there with `config://app`, a small config-blob resource.

> **Quirk:** The decorator argument is the resource's URI — not a path on disk, not an HTTP route. URIs in MCP follow the `scheme://path` convention so a server can group related resources under a common scheme (here, `config://`).
```

- [ ] **Step 2: Add the static-resource definition cell**

```python
@mcp.resource("config://app")
def app_config() -> dict:
    """Return the demo app's static configuration."""
    return {
        "name": "demo",
        "version": "0.1.0",
        "features": ["resources", "prompts"],
    }


print("Registered static resource: config://app")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Registered static resource: config://app
```

- [ ] **Step 4: Add the static-resource client-exercise cell**

```python
async with Client(mcp) as client:
    resources = await client.list_resources()
    print(f"Static resources ({len(resources)}):")
    for r in resources:
        print(f"  {r.uri}  (name={r.name!r})")

    content = await client.read_resource("config://app")
    print()
    print("Read config://app:")
    for item in content:
        # Each item is a TextResourceContents (or BlobResourceContents) with a .text payload.
        print(item.text)
```

- [ ] **Step 5: Run the cell and verify**

Expected output:

```
Static resources (1):
  config://app  (name='app_config')

Read config://app:
{"name":"demo","version":"0.1.0","features":["resources","prompts"]}
```

(The exact JSON whitespace inside `item.text` may vary slightly across `fastmcp` 2.x versions — some versions pretty-print, some emit compact JSON. The key contract is: a `list_resources()` call surfaces `config://app`, and `read_resource("config://app")` returns content whose `.text` parses back to the dict we returned.)

> **API note:** `read_resource(uri)` returns an iterable of resource-content objects (FastMCP wraps `TextResourceContents` / `BlobResourceContents` from the MCP SDK). For a `dict` return value FastMCP serializes to JSON and exposes it as `.text` on a text-content item. If your installed `fastmcp` 2.x exposes the payload under a different attribute (e.g., `.content` or a `.value`), update this cell to match what `dir(item)` actually shows.

- [ ] **Step 6: Commit**

```bash
git add fastmcp/07_fastmcp_resources_and_prompts.ipynb
git commit -m "feat(fastmcp): define and exercise static resource in notebook 07"
```

---

## Task 6: Section 3 — Parameterized resources

**Files:**
- Modify: `fastmcp/07_fastmcp_resources_and_prompts.ipynb` (add markdown header + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 3. Parameterized Resources

Not every resource is static. Often a server wants to expose a *family* of resources — one per user, one per document, one per topic — under a URI pattern. MCP calls this pattern a **resource template**: a URI containing parameter placeholders (e.g., `notes://{topic}`) that the client fills in at read time.

FastMCP makes this painless. Put a placeholder in the decorator URI, and declare a matching function argument:

```python
@mcp.resource("notes://{topic}")
def notes_for(topic: str) -> str:
    ...
```

FastMCP parses the URI at read time, extracts `topic`, and passes it to your function. No manual URL parsing, no regex.

> **Quirk:** Parameterized resource URIs — `@mcp.resource("notes://{topic}")` — auto-extract URI parameters into function args. The same decorator surface that handles `@mcp.tool` parameters handles resource-URI parameters, and it works for all three primitive types (tools, resources, prompts).

There's one more thing to be aware of on the client side. The MCP spec distinguishes between **concrete resources** (a fixed URI with a known payload) and **resource templates** (a URI pattern that needs parameters before it can be read). FastMCP follows this distinction:

- `await client.list_resources()` — returns *only* concrete (static) resources, like `config://app`.
- `await client.list_resource_templates()` — returns *only* templates, like `notes://{topic}`.

A client that wants to know everything the server exposes calls both. We'll demonstrate both lists and then read two filled-in URIs against a tiny in-memory notes dict.
```

- [ ] **Step 2: Add the parameterized-resource definition cell**

```python
NOTES = {
    "octopus": "Octopuses have three hearts and blue, copper-based blood.",
    "rome": "Rome was founded, by tradition, on April 21, 753 BCE.",
}


@mcp.resource("notes://{topic}")
def notes_for(topic: str) -> str:
    """Return a one-line note about the given topic, or an empty string if unknown."""
    return NOTES.get(topic, "")


print("Registered parameterized resource: notes://{topic}")
print(f"Known topics: {sorted(NOTES.keys())}")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Registered parameterized resource: notes://{topic}
Known topics: ['octopus', 'rome']
```

- [ ] **Step 4: Add the parameterized-resource client-exercise cell**

```python
async with Client(mcp) as client:
    resources = await client.list_resources()
    templates = await client.list_resource_templates()

    print(f"Concrete resources    ({len(resources)}): {[str(r.uri) for r in resources]}")
    print(f"Resource templates    ({len(templates)}): {[t.uriTemplate for t in templates]}")
    print()

    for uri in ("notes://octopus", "notes://rome"):
        content = await client.read_resource(uri)
        for item in content:
            print(f"{uri} -> {item.text!r}")
```

- [ ] **Step 5: Run the cell and verify**

Expected output:

```
Concrete resources    (1): ['config://app']
Resource templates    (1): ['notes://{topic}']

notes://octopus -> 'Octopuses have three hearts and blue, copper-based blood.'
notes://rome -> 'Rome was founded, by tradition, on April 21, 753 BCE.'
```

(The exact key the template URI is exposed under on the client side — `uriTemplate` vs `uri_template` — may vary slightly across `fastmcp` 2.x versions, since the underlying MCP SDK uses the camelCase JSON name. If `t.uriTemplate` raises `AttributeError`, swap to `t.uri_template` or print `dir(t)` to see the actual attribute name.)

> **API note:** Calling `read_resource("notes://octopus")` returns content whose `.text` is the string the function returned (the function returned a `str`, so FastMCP passes it through unchanged). If your version of `fastmcp` 2.x returns the payload under a different attribute, the fix is local to this cell.

- [ ] **Step 6: Commit**

```bash
git add fastmcp/07_fastmcp_resources_and_prompts.ipynb
git commit -m "feat(fastmcp): define and exercise parameterized resource in notebook 07"
```

---

## Task 7: Section 4 — Prompts (definition)

**Files:**
- Modify: `fastmcp/07_fastmcp_resources_and_prompts.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 4. Prompts

The third MCP primitive is the **prompt** — a reusable prompt template the server publishes for the client to render and use. Think of it as the server saying *"here's a known-good way to ask me about topic X — call this prompt with these arguments and you'll get back a structured set of messages ready to feed to an LLM."*

In FastMCP, you define a prompt the same way you define a tool or a resource: write a plain Python function, decorate it with `@mcp.prompt`. The function returns the rendered prompt. The simplest legal return value is a **plain string** — FastMCP treats it as a single user-role message. You can also return a **list of `Message`/`PromptMessage` objects** when you want multiple turns (e.g., a system message followed by a user message).

We'll define two:

1. `explain_octopuses` — parameterless, returns a single user-role message string.
2. `summarize_topic(topic: str)` — parameterized; arguments work the same as for tools and parameterized resources.

> **Quirk:** Same decorator surface again. `@mcp.tool`, `@mcp.resource("uri://...")`, `@mcp.prompt` — three primitives, one ergonomic pattern.

> **API note:** FastMCP 2.x accepts several prompt return-type shapes — `str`, `list[str]`, and lists of typed message objects from `fastmcp.prompts` or the MCP SDK. The forms shown below (a bare string from the parameterless prompt and a single-message string from the parameterized one) are the simplest legal forms and are stable across recent `fastmcp` 2.x versions. If you want explicit role tagging (system vs user vs assistant), import `Message` / `PromptMessage` from `fastmcp.prompts` (exact module path may vary by version — check `from fastmcp import prompts; dir(prompts)`).
```

- [ ] **Step 2: Add the prompt-definition code cell**

```python
@mcp.prompt
def explain_octopuses() -> str:
    """A canned prompt that asks the model to explain why octopuses are interesting."""
    return (
        "Explain, in three short bullet points, why octopuses are biologically "
        "unusual among invertebrates."
    )


@mcp.prompt
def summarize_topic(topic: str) -> str:
    """Render a prompt asking the model to summarize the given topic in two sentences."""
    return f"Summarize the topic {topic!r} in exactly two sentences for a curious adult reader."


print("Registered prompts:")
print("  explain_octopuses    (parameterless)")
print("  summarize_topic      (takes one argument: topic)")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Registered prompts:
  explain_octopuses    (parameterless)
  summarize_topic      (takes one argument: topic)
```

- [ ] **Step 4: Commit**

```bash
git add fastmcp/07_fastmcp_resources_and_prompts.ipynb
git commit -m "feat(fastmcp): define parameterless and parameterized prompts in notebook 07"
```

---

## Task 8: Section 4 (continued) — Prompts (client exercise)

**Files:**
- Modify: `fastmcp/07_fastmcp_resources_and_prompts.ipynb` (add 1 code cell)

- [ ] **Step 1: Add the prompt client-exercise code cell**

```python
async with Client(mcp) as client:
    prompts = await client.list_prompts()
    print(f"Prompts exposed ({len(prompts)}):")
    for p in prompts:
        arg_names = [a.name for a in (p.arguments or [])]
        print(f"  {p.name}  args={arg_names}  description={p.description!r}")
    print()

    no_args = await client.get_prompt("explain_octopuses", {})
    print("get_prompt('explain_octopuses', {}):")
    for msg in no_args.messages:
        print(f"  role={msg.role}  text={msg.content.text!r}")
    print()

    with_args = await client.get_prompt("summarize_topic", {"topic": "octopus"})
    print("get_prompt('summarize_topic', {'topic': 'octopus'}):")
    for msg in with_args.messages:
        print(f"  role={msg.role}  text={msg.content.text!r}")
```

- [ ] **Step 2: Run the cell and verify**

Expected output:

```
Prompts exposed (2):
  explain_octopuses  args=[]  description='A canned prompt that asks the model to explain why octopuses are interesting.'
  summarize_topic  args=['topic']  description='Render a prompt asking the model to summarize the given topic in two sentences.'

get_prompt('explain_octopuses', {}):
  role=user  text='Explain, in three short bullet points, why octopuses are biologically unusual among invertebrates.'

get_prompt('summarize_topic', {'topic': 'octopus'}):
  role=user  text="Summarize the topic 'octopus' in exactly two sentences for a curious adult reader."
```

(Per the MCP spec, when a prompt function returns a bare `str`, the resulting message has `role="user"` and a text-content payload — that's what the output above asserts. Attribute paths on the client side — `msg.content.text` and `p.arguments` — track the MCP SDK's pydantic models; if your `fastmcp` 2.x version surfaces them differently, the fix is local to this cell.)

> **API note:** `get_prompt(name, arguments)` returns a `GetPromptResult` object with a `.messages` list. Each message has `.role` and `.content`; for text content, `.content.text` is the rendered string. If your installed version names these differently, run `print(dir(no_args))` and adjust the attribute accesses.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/07_fastmcp_resources_and_prompts.ipynb
git commit -m "feat(fastmcp): exercise prompts via in-memory Client in notebook 07"
```

---

## Task 9: Section 5 — All three primitives side by side

**Files:**
- Modify: `fastmcp/07_fastmcp_resources_and_prompts.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 5. All Three Primitives Side by Side

Now that we've defined all three — `add` (tool, from notebook 02), `config://app` and `notes://{topic}` (resources, this notebook), and `explain_octopuses` plus `summarize_topic` (prompts, this notebook) — the symmetry of the FastMCP surface is worth pausing on.

| Primitive   | What it is                                                              | When the client uses it                                                      | FastMCP decorator                  |
|-------------|-------------------------------------------------------------------------|------------------------------------------------------------------------------|------------------------------------|
| **Tool**    | A function the client invokes with arguments and gets a result back.    | "Do something for me." Side effects, computation, calls to external systems. | `@mcp.tool`                        |
| **Resource**| Read-only data identified by a URI. Static or parameterized.            | "Give me that piece of data." Config blobs, documents, per-topic notes.      | `@mcp.resource("uri://path")`      |
| **Prompt**  | A reusable, server-published prompt template the client renders.        | "Render the canonical prompt for X." Few-shot scaffolds, summarization, etc. | `@mcp.prompt`                      |

Three primitives. One decorator pattern: write a plain Python function, decorate it, register against the same `mcp` object. Same type-hint-driven argument schema, same docstring-as-description, same in-memory `Client(mcp)` to exercise everything.

That symmetry is the whole pitch of FastMCP across MCP primitives. The rest of the framework (HTTP transport in notebook 03, auth in notebook 04, `Context` and `lifespan` in notebook 05, validation in notebook 06) layers on top of this same shape.
```

- [ ] **Step 2: Verify the cell renders**

Open the notebook and confirm the table renders properly.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/07_fastmcp_resources_and_prompts.ipynb
git commit -m "feat(fastmcp): add three-primitives recap table to notebook 07"
```

---

## Task 10: Closing recap

**Files:**
- Modify: `fastmcp/07_fastmcp_resources_and_prompts.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Add the "What you just learned" markdown cell**

```markdown
## What you just learned

- MCP has **three** server-side primitives — tools, resources, and prompts — and FastMCP exposes all three through the same decorator-driven surface.
- `@mcp.resource("config://app")` defines a **static resource** at a fixed URI; the client reads it with `await client.read_resource("config://app")`.
- `@mcp.resource("notes://{topic}")` defines a **parameterized resource** (a "resource template" in MCP-spec terms); URI parameters are auto-extracted into function arguments by FastMCP — no manual URI parsing.
- The MCP spec separates concrete resources from templates, and FastMCP follows that split on the client side: `await client.list_resources()` returns the concrete ones, `await client.list_resource_templates()` returns the templates. A client that wants the full picture calls both.
- `@mcp.prompt` defines a **prompt template** the client can fetch with `await client.get_prompt(name, arguments)`. Returning a bare string yields a single user-role message; for multi-turn templates return a list of typed message objects.
- Across all three primitives, the FastMCP shape is the same: plain Python function, one decorator, type hints driving the schema, docstring driving the description, in-memory `Client(mcp)` exercising it without infrastructure.
```

- [ ] **Step 2: Verify the cell renders**

Open the notebook and confirm the bullet list renders.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/07_fastmcp_resources_and_prompts.ipynb
git commit -m "feat(fastmcp): add closing recap to notebook 07"
```

---

## Task 11: "What's missing" teaser

**Files:**
- Modify: `fastmcp/07_fastmcp_resources_and_prompts.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Add the "What's missing" markdown cell**

```markdown
## What's missing

We now know how to publish all three MCP primitives — tools, resources, and prompts — from a FastMCP server, and how to exercise them from an in-memory client. What we **haven't** done in this entire series is plug FastMCP into a real LLM. Every notebook so far has been the framework on one side and a hand-rolled client on the other; no model has been in the loop.

In **notebook 08** we close the series with the payoff. We stand up *two* FastMCP apps — a small "research" server (canned facts about topics, similar in spirit to the `notes://{topic}` resource here) and a small "writer" server (formats text into a summary). We compose them with `mount()` so a single MCP endpoint exposes tools from both, then point a real Claude (or OpenAI) tool-use loop at that composed endpoint and watch the model orchestrate across both servers to fulfill a user request. Closing section: rewrite that same plumbing as a `pytest`-style test using the in-memory `Client(mcp)` — proving the same server code is unit-testable without a subprocess or a socket. This is the only notebook in the series that requires a real LLM API key.
```

- [ ] **Step 2: Verify the cell renders**

Open the notebook and confirm the markdown renders.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/07_fastmcp_resources_and_prompts.ipynb
git commit -m "feat(fastmcp): add 'What's missing' teaser to notebook 07"
```

---

## Task 12: Section 6 — Cleanup

**Files:**
- Modify: `fastmcp/07_fastmcp_resources_and_prompts.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 6. Cleanup

This notebook never started a background server — every interaction went through the in-memory `Client(mcp)`, which doesn't open a socket or spawn a subprocess. The only thing to do here is drop the `mcp` object and confirm the kernel has nothing lingering.
```

- [ ] **Step 2: Add the cleanup code cell**

```python
del mcp
print("Cleanup OK. No servers were started, so no sockets or subprocesses to close.")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Cleanup OK. No servers were started, so no sockets or subprocesses to close.
```

- [ ] **Step 4: Commit**

```bash
git add fastmcp/07_fastmcp_resources_and_prompts.ipynb
git commit -m "feat(fastmcp): add cleanup cell to notebook 07"
```

---

## Task 13: End-to-end verification in a fresh kernel

**Files:** none (verification only).

- [ ] **Step 1: Execute the notebook top-to-bottom in a clean kernel**

```bash
jupyter nbconvert --to notebook --execute --inplace fastmcp/07_fastmcp_resources_and_prompts.ipynb
```

Expected: succeeds with no errors. The notebook is rewritten with embedded outputs from the fresh run.

- [ ] **Step 2: Verify expected outputs in the executed notebook**

Cells should contain, in order:

1. `Setup OK. Server: demo`
2. `Registered static resource: config://app`
3. The static-resource listing block: `Static resources (1):` with `config://app  (name='app_config')`, followed by `Read config://app:` and a single JSON line whose content parses back to `{"name": "demo", "version": "0.1.0", "features": ["resources", "prompts"]}`.
4. `Registered parameterized resource: notes://{topic}` followed by `Known topics: ['octopus', 'rome']`.
5. The parameterized-resource listing block: `Concrete resources    (1): ['config://app']`, `Resource templates    (1): ['notes://{topic}']`, then the two `notes://octopus -> ...` and `notes://rome -> ...` lines with the canned facts.
6. `Registered prompts:` block listing `explain_octopuses` and `summarize_topic`.
7. The prompt-exercise block: `Prompts exposed (2):` with both prompts (arg lists `[]` and `['topic']`), then the `get_prompt('explain_octopuses', {})` and `get_prompt('summarize_topic', {'topic': 'octopus'})` blocks each printing one `role=user  text=...` line.
8. `Cleanup OK. No servers were started, so no sockets or subprocesses to close.`

No cell raises an unhandled exception. If `ModuleNotFoundError: fastmcp` appears, install it (`pip install fastmcp`) and re-run. If an `AttributeError` appears against a content/template attribute (e.g., `uriTemplate`, `content.text`), check the **API note** callouts in sections 2, 3, and 4 — those cells flag which attribute paths may vary across `fastmcp` 2.x minor versions.

- [ ] **Step 3: Sanity-check that no background process is left behind**

This notebook never starts a server — only the in-memory client — so there is nothing to clean up. Confirm:

```bash
ps -ef | grep -i fastmcp | grep -v grep
```

Expected: no output (no orphaned FastMCP processes from this notebook).

- [ ] **Step 4: Commit the clean run**

```bash
git add fastmcp/07_fastmcp_resources_and_prompts.ipynb
git commit -m "chore(fastmcp): commit clean fresh-kernel run of notebook 07"
```

---

## Done

After Task 13 passes, notebook 07 is complete. The next plan to write is `2026-05-18-fastmcp-notebook-08-real-llm.md`, which stands up two FastMCP apps (a "research" server and a "writer" server), composes them with `mount()`, and drives the composed endpoint with a real Claude (or OpenAI) tool-use loop — closing the series with a `pytest`-style in-memory test that proves the same server code is unit-testable without infrastructure.
