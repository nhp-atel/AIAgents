# FastMCP Learning Series — Design

**Date:** 2026-05-18
**Author:** Nimesh Patel (with Claude)
**Status:** Approved design, pending implementation plan

## Purpose

Teach the **FastMCP framework** by doing — building the same MCP scenarios already covered in the `mcp/` series, but with FastMCP instead of the raw `mcp` SDK. By doing each scenario twice (once raw, once with FastMCP), the learner directly feels what FastMCP adds: decorator-driven tools, auto-generated JSON schemas from type hints, ergonomic resources and prompts, server composition, built-in auth, and an in-memory client that makes server code unit-testable.

The series lives in a **new top-level folder `fastmcp/`**, sibling to the existing `mcp/`, `a2a/`, `langgraph/`, and `crewai/` folders. It is a progressive 8-notebook series (`01_..._.ipynb` through `08_..._.ipynb`), mirroring the pedagogical structure of the existing series.

The learner should finish able to:

1. Explain what FastMCP is, what it abstracts, and when to reach for it instead of the raw `mcp` SDK.
2. Build a FastMCP server that exposes tools, resources, and prompts using decorators and Python type hints.
3. Serve the same FastMCP app over stdio or HTTP (streamable-HTTP transport) by swapping a runtime flag.
4. Add bearer-token and OAuth2 authentication using FastMCP's built-in auth provider.
5. Use the `Context` parameter for in-tool logging and progress reporting, and `lifespan` for startup/teardown.
6. Compose multiple FastMCP apps with `mount()` and drive them from a real LLM in a tool-use loop.
7. Write unit tests for MCP tools using the in-memory `Client(server)` — no subprocess required.

## Non-goals

The following are **explicitly out of scope** for this series:

- Production deployment concerns (TLS, load balancing, observability infra).
- Re-teaching MCP concepts the `mcp/` series already covers — we link back instead of re-explaining.
- Side-by-side compare cells in every step. Quirk callouts appear only where FastMCP differs materially from the raw SDK.
- Building a real identity provider for OAuth2 — the IdP is mocked.
- FastMCP in languages other than Python.
- Comparing FastMCP to other MCP frameworks. (Possible follow-on later.)

## Audience and prerequisites

The learner already understands:

- Python (async/await, typing, pydantic).
- HTTP basics (methods, status codes, headers).
- The MCP protocol at the level taught in the `mcp/` 01–08 series (server primitives: tools, resources, prompts; transports: stdio, HTTP).

The learner does **not** need prior knowledge of:

- FastMCP.
- The internal structure of the raw `mcp` SDK beyond what `mcp/02` already shows.

## Pedagogical arc

The series follows the same shape as the `mcp/` and `a2a/` series:

1. **Show the pain** (notebook 01) — what the raw SDK looks like for a minimal server, to set the baseline.
2. **Introduce the framework** (notebook 02) — minimum viable FastMCP server over stdio.
3. **Layer in capabilities** (notebooks 03–07) — HTTP transport, auth, production patterns, safety, resources/prompts.
4. **Tie it together** (notebook 08) — real LLM driving FastMCP tools end-to-end, multi-server composition via `mount()`, in-memory client for tests.

Every notebook is **runnable end-to-end** in Jupyter with no external infrastructure beyond a Python virtualenv. Each notebook opens with a 1-paragraph recap pointing at its `mcp/0X` counterpart, then builds the FastMCP version straight through. Inline **"Quirk:"** callouts surface FastMCP-specific behavior only where it materially differs from the raw SDK. Each notebook ends with a **"What you just learned"** recap and a **"What's missing"** teaser motivating the next notebook (omitted in notebook 08).

## Common stack

| Concern | Choice |
|---|---|
| Python | 3.11+ |
| Notebooks | Jupyter |
| Framework under study | `fastmcp` (2.x line — the actively developed one) |
| Raw-SDK baseline (notebook 01 only) | `mcp` (official Anthropic SDK) |
| HTTP client | `httpx` |
| Typed models | `pydantic` v2 (FastMCP uses it natively) |
| LLM | Any (Claude default, OpenAI fallback), used **only** in notebook 08 |

**Server execution pattern.** Notebooks 03–08 run FastMCP over HTTP by calling `mcp.run(transport="streamable-http", port=…)` on a background `threading.Thread`. Notebook 02 runs over stdio. The in-memory `Client(mcp)` (introduced in notebook 02) does not need any transport at all and is used throughout for examples that don't require a real socket. A `shutdown()` cell at the bottom of each HTTP notebook stops the server cleanly.

**Port allocation** (avoiding 8000/8001 because the user runs an unrelated dev server there, and 8010/8011 because the `a2a/` series uses those):

- `8020` — primary FastMCP HTTP server in notebooks 03–08.
- `8021` — secondary FastMCP app, mounted/composed in notebooks 05 and 08.

**No API keys for notebooks 01–07.** Tools return canned, deterministic responses so the focus stays on the framework. Notebook 08 is the only one that requires a real LLM API key (`ANTHROPIC_API_KEY` by default, `OPENAI_API_KEY` as fallback).

## FastMCP version

This series targets the **current stable `fastmcp` 2.x release** at the time of writing. Where the API is known to be in flux between minor versions (notably auth provider naming), the notebook calls this out explicitly so the learner can update for newer versions. Notebook 01 uses the raw `mcp` SDK (same version `mcp/02` uses) purely as the baseline-pain demonstration.

## Notebooks

### 01 — `01_without_fastmcp.ipynb` — The pain
**Goal:** baseline the cost of building MCP without FastMCP.

- Re-implement the stdio server from `mcp/02` using the raw `mcp` SDK: `Server()`, `@server.list_tools()` and `@server.call_tool()` handlers, hand-written `inputSchema` JSON for every tool, manual `stdio_server()` runner setup.
- Add a second tool to make the boilerplate cost obvious (every new tool means another hand-written JSON schema).
- Notebook closes: *"Every tool means another schema by hand, another handler dispatch branch. There should be a framework."*
- No FastMCP yet — this notebook exists purely to set the contrast.

### 02 — `02_fastmcp_local_stdio.ipynb` — Minimum viable FastMCP
**Goal:** introduce the FastMCP primitive.

- `from fastmcp import FastMCP; mcp = FastMCP("demo")`.
- Define tools with `@mcp.tool` on plain Python functions; docstrings become tool descriptions, type hints become the JSON schema.
- Run over stdio (`mcp.run(transport="stdio")`).
- Introduce the **in-memory `Client(mcp)`** — call tools directly from the same Python process without spawning a subprocess. Used heavily in later notebooks for examples and tests.
- **Quirks flagged:** auto-schema from type hints; docstring → tool description; in-memory client.

### 03 — `03_fastmcp_http_server.ipynb` — Swap the transport, not the code
**Goal:** show how transports are runtime flags in FastMCP, not code rewrites.

- Take the same `FastMCP` instance from notebook 02 and serve it over HTTP via `mcp.run(transport="streamable-http", port=8020)` on a background thread.
- Drive it with an `httpx` MCP client to call tools over the wire.
- Briefly contrast `streamable-http` vs SSE transports — when to pick which.
- **Quirks flagged:** transport is a runtime arg, no code change in the tool definitions.

### 04 — `04_fastmcp_with_auth.ipynb` — Bearer tokens, the easy way
**Goal:** add authentication using FastMCP's built-in auth provider.

- Cover three modes:
  - **No auth** (the implicit default in 02–03, now made explicit).
  - **Bearer token** — configured via `FastMCP(..., auth=BearerAuthProvider(...))`, sent as `Authorization: Bearer ...`, validated server-side.
  - **OAuth2** — using FastMCP's OAuth provider helper, client credentials flow. IdP is **mocked** (a small FastAPI route that issues tokens), matching the `a2a/07` pattern.
- **Quirks flagged:** auth is a first-class server param, not middleware the user wires by hand.

### 05 — `05_fastmcp_production_patterns.ipynb` — Lifespan, Context, mounting
**Goal:** the framework features that pay off in non-toy apps.

- `lifespan` async context manager for startup/teardown (e.g., open a sqlite connection once at server start, close it at shutdown). Compare with `mcp/05`'s equivalent.
- The `Context` parameter — a tool that declares `ctx: Context` gets the runtime context auto-injected; use it to emit progress updates and structured logs from inside a tool call.
- Structured error handling via custom exceptions that FastMCP serializes to MCP error responses.
- **`mount()`** — compose two `FastMCP` apps under one server (primary at `8020`, secondary mounted at a sub-path). No equivalent exists in the raw SDK.
- **Quirks flagged:** `Context` is auto-injected by parameter type, not passed by the client; `mount()` has no raw-SDK analogue.

### 06 — `06_fastmcp_prompt_injection_and_safety.ipynb` — Validation and untrusted input
**Goal:** safe handling of user-controlled tool inputs.

- Tools that accept user-controlled input, with validation via Pydantic models declared directly as tool parameters — FastMCP both validates the input and generates the JSON schema from the model.
- Allowlisted resources: a resource handler that refuses URIs outside a known list.
- Output sanitization: a tool that strips control characters from its output before returning.
- Refusing dangerous tool args (e.g., a tool that would otherwise let a caller traverse the filesystem).
- **Quirks flagged:** Pydantic-native parameter models give validation + schema generation in one step.

### 07 — `07_fastmcp_resources_and_prompts.ipynb` — The other two primitives, the easy way
**Goal:** show how FastMCP's decorator surface extends to resources and prompts.

- `@mcp.resource("notes://{topic}")` for parameterized read-only data — URI parameters become function arguments automatically.
- `@mcp.resource("config://app")` for static resources.
- `@mcp.prompt` for reusable prompt templates the MCP client can pull in.
- Client side: list resources, read a resource by URI, list prompts, fetch a prompt — all using the in-memory `Client(mcp)`.
- **Quirks flagged:** URI parameters → function args; same decorator-surface ergonomics for resources and prompts as for tools.

### 08 — `08_fastmcp_real_llm_end_to_end.ipynb` — Real LLM driving FastMCP tools
**Goal:** the payoff — a real LLM orchestrating across composed FastMCP apps, plus testability.

- Stand up two FastMCP apps: a small "research" server (returns canned facts) and a small "writer" server (formats text). Mount them together so a single MCP endpoint exposes tools from both.
- A Claude (or OpenAI) tool-use loop on the client side: the model sees both servers' tools through one MCP connection and orchestrates them to fulfill a user request.
- Closing section: rewrite the LLM-facing tool-use plumbing as a **pytest-style test** using the in-memory `Client(mcp)` — demonstrating that the same server code is unit-testable without a subprocess or socket.
- The only notebook in this series that requires a real LLM API key.
- Final cell: short narrative on what FastMCP bought us across the eight notebooks — what we'd have had to hand-write per `mcp/01` and per the raw-SDK baseline in notebook 01.

## File layout

```
fastmcp/
  01_without_fastmcp.ipynb
  02_fastmcp_local_stdio.ipynb
  03_fastmcp_http_server.ipynb
  04_fastmcp_with_auth.ipynb
  05_fastmcp_production_patterns.ipynb
  06_fastmcp_prompt_injection_and_safety.ipynb
  07_fastmcp_resources_and_prompts.ipynb
  08_fastmcp_real_llm_end_to_end.ipynb
```

The new `fastmcp/` folder sits at the repository root alongside `mcp/`, `a2a/`, `langgraph/`, and `crewai/`.

## Style conventions

Each notebook opens with:

- A `## Why this notebook exists` markdown cell, which includes the 1-paragraph "previously in `mcp/0X`" recap.
- A `## What you'll learn` bulleted list.

Body sections use numbered headers (`## 1. ...`, `## 2. ...`) with `### Try it` cells containing exercises the learner can run and modify. **"Quirk:"** callouts are markdown blockquotes inserted only where the FastMCP behavior differs materially from the raw SDK equivalent — they are not present in every cell.

Each notebook closes with:

- `## What you just learned` — bulleted recap.
- `## What's missing` — one paragraph motivating the next notebook (omitted in notebook 08, which closes the series).
- `shutdown()` cell for notebooks that started a background HTTP server (03–08).

This matches the existing `mcp/` and `a2a/` series style.

## Success criteria

The series is complete when:

1. All 8 notebooks exist under `fastmcp/`.
2. Each notebook runs top-to-bottom in a fresh kernel with no errors.
3. Notebooks 01–07 require no external API keys.
4. Notebook 08 runs given a single LLM API key (Claude or OpenAI).
5. Every FastMCP feature claimed in the spec is demonstrated with a runnable cell in the corresponding notebook.
6. Notebook 08's closing test-style cell passes when executed, demonstrating the in-memory `Client(server)` testability claim.

## Open questions deferred to implementation planning

- Exact `fastmcp` 2.x version pin (latest stable at time of writing).
- Whether notebook 04's OAuth2 mock IdP reuses the `a2a/07` mock or is rebuilt in the FastMCP idiom. (Probably rebuilt, but small.)
- Whether notebook 06's "dangerous tool args" example uses a filesystem read or a shell-exec stub. (Probably filesystem read — less scary in a notebook.)
- Whether notebook 08 uses Claude or OpenAI as the default LLM. (Probably Claude, matching `mcp/08`.)

These do not affect the design and will be decided in the implementation plan.
