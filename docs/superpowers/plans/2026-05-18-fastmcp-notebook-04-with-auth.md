# FastMCP Notebook 04 — `04_fastmcp_with_auth.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the fourth notebook in the FastMCP learning series — **authentication**. Cover three modes: no-auth (made explicit), bearer-token validated against a static RSA-signed JWT, and OAuth2 client credentials with a mocked IdP. Every mode is wired through FastMCP's first-class `FastMCP(..., auth=...)` constructor parameter — not user-wired middleware.

**Architecture:** A single Jupyter notebook running two FastAPI/FastMCP apps:

- App #1: a FastMCP server on `127.0.0.1:8020` — exposes one tool (`research_topic`) and gets re-instantiated three times across the notebook with different `auth=` configurations (none, then bearer, then OAuth2).
- App #2: a mocked OAuth2 identity provider on `127.0.0.1:8021` — a tiny FastAPI app exposing `POST /token` that accepts `client_id` / `client_secret` form data and returns a signed JWT.

The notebook walks each auth mode as a self-contained demo, restarting the FastMCP server in place between sections so each section reads as a complete example.

**Tech Stack:** Python 3.11+, Jupyter, `fastmcp` 2.x, FastAPI, uvicorn, httpx, pydantic v2, `cryptography` (for the RSA keypair), `PyJWT` (for signing/verifying JWTs), threading. No real LLM, no real IdP.

> **API note:** FastMCP 2.x's auth provider class names have shifted between minor releases (e.g., `BearerAuthProvider`, `JWTAuthProvider`, `RemoteAuthProvider`). This plan instructs the engineer to **try `from fastmcp.server.auth import BearerAuthProvider` first**; if that import fails, check `dir(fastmcp.server.auth)` (or the FastMCP 2.x docs) for the current name in the installed version. Same applies to the OAuth2 provider class. Do not guess: verify against the installed version and amend the import.

**Port assignment:** `8020` FastMCP server, `8021` mocked IdP.

**Companion spec:** `docs/superpowers/specs/2026-05-18-fastmcp-learning-series-design.md` (notebook 04 section).

---

## File Structure

- **Create:** `fastmcp/04_fastmcp_with_auth.ipynb` — the entire notebook, self-contained.
- **Modify:** none.
- **No separate test files** — verification is the notebook running top-to-bottom in a fresh kernel (Task 14).

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown)
2. **"What you'll learn"** (markdown bullets)
3. **Section 1: Setup** (code — imports, threaded-server helper, RSA-keypair helper, JWT helper)
4. **Section 2: No auth (the default we've been using)** (markdown + 2 code cells: build a no-auth FastMCP server, call it from a client)
5. **Section 3: Bearer-token auth** (markdown + Quirk callout + 4 code cells: generate RSA keypair, build server with `BearerAuthProvider`, demonstrate rejection without token, demonstrate success with token)
6. **Section 4: OAuth2 client credentials (with a mocked IdP)** (markdown + 4 code cells: build mocked IdP, build server validating IdP-issued tokens, fetch a token, call the server with it)
7. **"What you just learned"** (markdown)
8. **"What's missing"** (markdown — teases notebook 05, production patterns)
9. **Section 5: Cleanup** (code: shut down both servers)

---

## Task 1: Create the notebook scaffold

**Files:**
- Create: `fastmcp/04_fastmcp_with_auth.ipynb`

- [ ] **Step 1: Verify the `fastmcp/` folder exists at the repo root**

```bash
ls -la "/Users/nimeshpatel/Documents/AI Agents/fastmcp/"
```

Expected: directory listing (it should already exist from notebooks 01–03). If it does not, create it: `mkdir -p "/Users/nimeshpatel/Documents/AI Agents/fastmcp"`.

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
python -c "import json; json.load(open('fastmcp/04_fastmcp_with_auth.ipynb'))"
ls -la fastmcp/04_fastmcp_with_auth.ipynb
```

Expected: no exception, file non-zero size.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/04_fastmcp_with_auth.ipynb
git commit -m "feat(fastmcp): scaffold 04_fastmcp_with_auth.ipynb"
```

---

## Task 2: Add title and "Why this notebook exists"

**Files:**
- Modify: `fastmcp/04_fastmcp_with_auth.ipynb` (add cells 1–2)

- [ ] **Step 1: Add a markdown cell with title and motivation**

Cell content (markdown):

```markdown
# 04 — FastMCP With Auth: Bearer Tokens, the Easy Way

## Why this notebook exists

In **notebook 03** we took the same `FastMCP` instance from notebook 02 and served it over HTTP — same tool definitions, just a different transport flag. The server happily accepted any request that reached `http://127.0.0.1:8020/`. That's fine on localhost; it's a serious problem the moment the server is reachable by anything else.

This notebook adds authentication. The key insight is that FastMCP treats auth as a **first-class constructor parameter** on the server — `FastMCP(..., auth=…)` — rather than middleware the user has to wire up by hand. Three modes, three constructor configurations, the tool code itself never changes.

> *We target the current stable `fastmcp` 2.x release. FastMCP's auth provider class names have shifted between minor versions; this notebook calls out where to look if your installed version disagrees.*
```

- [ ] **Step 2: Add a markdown cell with "What you'll learn"**

Cell content (markdown):

```markdown
## What you'll learn

- That "no auth" is the implicit default in `FastMCP(...)` — and that this is a deliberate choice you should make explicit.
- How to add **bearer-token** auth by passing a `BearerAuthProvider` (or its current 2.x equivalent) to the `FastMCP(..., auth=…)` constructor.
- How a request without an `Authorization` header is rejected (HTTP 401), and how a request with a valid token succeeds.
- How to generate an RSA keypair with `cryptography` and sign a JWT the FastMCP bearer provider will accept.
- How to add **OAuth2 client-credentials** auth: a tiny mocked Identity Provider on a sibling port, plus a FastMCP server configured to trust tokens issued by that IdP.
- Where FastMCP's auth provider lives in the package, and how to discover the correct class name if your installed version uses a different one.
```

- [ ] **Step 3: Verify cells render**

Open the notebook in Jupyter/VS Code and confirm both cells render as markdown.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/04_fastmcp_with_auth.ipynb
git commit -m "feat(fastmcp): add intro markdown to notebook 04"
```

---

## Task 3: Add the setup cell

**Files:**
- Modify: `fastmcp/04_fastmcp_with_auth.ipynb` (add markdown header + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 1. Setup

Same threaded-server pattern as notebook 03, plus two extras specific to this notebook:

- An **RSA-keypair helper** for the bearer-auth section (the bearer provider in FastMCP 2.x typically validates JWTs against a public key, so we generate a keypair once and reuse it).
- A **JWT signing helper** that wraps `PyJWT` for the same reason.

We also import `fastmcp.server.auth` lazily and inspect its contents — the exact class name has shifted between FastMCP 2.x minor versions, and we want the notebook to be honest about that.
```

- [ ] **Step 2: Add the setup code cell**

```python
import asyncio
import json
import threading
import time
import uuid

import httpx
import jwt  # PyJWT
import uvicorn
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import rsa
from fastapi import FastAPI, Form
from fastapi.responses import JSONResponse

import fastmcp
from fastmcp import Client, FastMCP

print(f"fastmcp version: {fastmcp.__version__}")

# Inspect the auth module so the engineer can see what's actually exported in
# the installed FastMCP version. The names below have shifted between 2.x
# minor releases; we list what's there before importing a specific class.
import fastmcp.server.auth as _auth_mod

print("\nfastmcp.server.auth exports:")
for name in sorted(n for n in dir(_auth_mod) if not n.startswith("_")):
    print(f"  • {name}")

_servers: list[uvicorn.Server] = []


def run_server_in_thread(app, port: int) -> uvicorn.Server:
    """Start a FastAPI/Starlette `app` on localhost:`port` in a daemon thread."""
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


def run_fastmcp_in_thread(mcp: FastMCP, port: int) -> threading.Thread:
    """Run a FastMCP instance over streamable-http on a background thread."""
    def _target():
        mcp.run(transport="streamable-http", host="127.0.0.1", port=port)
    thread = threading.Thread(target=_target, daemon=True)
    thread.start()
    # Poll the port until it's accepting connections.
    deadline = time.time() + 5.0
    while time.time() < deadline:
        try:
            httpx.get(f"http://127.0.0.1:{port}/", timeout=0.2)
            break
        except Exception:
            time.sleep(0.05)
    else:
        raise RuntimeError(f"FastMCP server on port {port} did not start in time")
    return thread


def shutdown_all_servers() -> None:
    for server in list(_servers):
        server.should_exit = True
    _servers.clear()


def generate_rsa_keypair() -> tuple[bytes, bytes]:
    """Return (private_pem, public_pem) for a fresh 2048-bit RSA key."""
    key = rsa.generate_private_key(public_exponent=65537, key_size=2048)
    private_pem = key.private_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PrivateFormat.PKCS8,
        encryption_algorithm=serialization.NoEncryption(),
    )
    public_pem = key.public_key().public_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PublicFormat.SubjectPublicKeyInfo,
    )
    return private_pem, public_pem


def sign_jwt(private_pem: bytes, *, issuer: str, audience: str, subject: str, ttl_seconds: int = 3600) -> str:
    """Sign a minimal JWT (RS256) with the given private key."""
    now = int(time.time())
    payload = {
        "iss": issuer,
        "aud": audience,
        "sub": subject,
        "iat": now,
        "exp": now + ttl_seconds,
    }
    return jwt.encode(payload, private_pem, algorithm="RS256")


print("\nSetup OK")
```

- [ ] **Step 3: Run the cell**

Expected output (the version string and the exports list will vary by installed FastMCP version — the engineer should read what's actually printed):

```
fastmcp version: 2.x.y

fastmcp.server.auth exports:
  • BearerAuthProvider
  • OAuthProvider
  • <other names depending on installed version>

Setup OK
```

If imports fail:

```bash
pip install "fastmcp>=2" "httpx" "uvicorn[standard]" "fastapi" "pydantic>=2" "cryptography" "PyJWT[crypto]"
```

> **API note:** Read the "exports" list carefully. The next task imports `BearerAuthProvider` directly; if it isn't in the list, the engineer must substitute the current name (e.g., `JWTAuthProvider`) before continuing. Do not invent a name that isn't in the printed list.

- [ ] **Step 4: Commit**

```bash
git add fastmcp/04_fastmcp_with_auth.ipynb
git commit -m "feat(fastmcp): add setup cell to notebook 04"
```

---

## Task 4: Section 2 — No auth (the default we've been using)

**Files:**
- Modify: `fastmcp/04_fastmcp_with_auth.ipynb` (add markdown header + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 2. No Auth (the Default We've Been Using)

Notebook 03's server was unauthenticated — anything that could reach `http://127.0.0.1:8020/` could call any tool. That's the default when you instantiate `FastMCP("name")` with no `auth=` argument. It's a perfectly reasonable default for local development, but it should be a *choice*, not an accident.

We rebuild the same `research_topic` tool from notebook 03 here, run it on port 8020, and demonstrate that an in-process `Client(mcp)` can call it with no credentials at all. This is also a chance to re-establish the calling pattern before we start changing the constructor.
```

- [ ] **Step 2: Add the no-auth server code cell**

```python
no_auth_mcp = FastMCP("researcher-no-auth")


@no_auth_mcp.tool
def research_topic(topic: str) -> list[str]:
    """Return canned facts about a well-known topic."""
    facts = {
        "octopuses": [
            "Octopuses have three hearts.",
            "They can change color and texture in milliseconds.",
        ],
        "rome": [
            "Rome was founded, per legend, in 753 BCE.",
            "The Roman Republic predates the Empire by ~500 years.",
        ],
    }
    return facts.get(topic.lower(), [])


async def call_research(client: Client, topic: str) -> list[str]:
    """Call the research_topic tool through any FastMCP Client and return the facts."""
    result = await client.call_tool("research_topic", {"topic": topic})
    # FastMCP returns a CallToolResult; `data` is the structured payload.
    return result.data


# In-memory client — no HTTP, no auth, no transport. Just a direct call into
# the FastMCP instance.
async def _run_no_auth_inmemory():
    async with Client(no_auth_mcp) as client:
        facts = await call_research(client, "octopuses")
        for fact in facts:
            print(f"  • {fact}")


asyncio.run(_run_no_auth_inmemory())
```

- [ ] **Step 3: Run, expect**

```
  • Octopuses have three hearts.
  • They can change color and texture in milliseconds.
```

- [ ] **Step 4: Add the HTTP-call cell**

```python
# Now serve the same instance over HTTP on 8020 and call it with an HTTP client.
# This is exactly the notebook-03 pattern — no auth headers anywhere.
no_auth_thread = run_fastmcp_in_thread(no_auth_mcp, port=8020)
print("No-auth FastMCP server on http://127.0.0.1:8020")


async def _run_no_auth_http():
    async with Client("http://127.0.0.1:8020/mcp") as client:
        facts = await call_research(client, "rome")
        for fact in facts:
            print(f"  • {fact}")


asyncio.run(_run_no_auth_http())

# Stop this server before the next section spins up a new one on the same port.
shutdown_all_servers()
print("\nNo-auth server stopped.")
```

- [ ] **Step 5: Run, expect**

```
No-auth FastMCP server on http://127.0.0.1:8020
  • Rome was founded, per legend, in 753 BCE.
  • The Roman Republic predates the Empire by ~500 years.

No-auth server stopped.
```

> **API note:** The exact URL path the HTTP `Client` connects to depends on the FastMCP 2.x version. Recent versions mount streamable-http at `/mcp` by default. If the URL `http://127.0.0.1:8020/mcp` returns 404 or the client raises a connection error, try the bare `http://127.0.0.1:8020/` first, then consult `mcp.http_app` / `mcp.run` in the installed FastMCP docs for the correct path. The same caveat applies in Tasks 6, 7, 11, and 13 — keep the URL consistent with whatever this task confirms works.

- [ ] **Step 6: Commit**

```bash
git add fastmcp/04_fastmcp_with_auth.ipynb
git commit -m "feat(fastmcp): demo no-auth FastMCP server in notebook 04"
```

---

## Task 5: Section 3 markdown — Bearer-token auth (concept + Quirk callout)

**Files:**
- Modify: `fastmcp/04_fastmcp_with_auth.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 3. Bearer-Token Auth

The card-carrying example of FastMCP's "auth is a constructor parameter" design:

```python
from fastmcp.server.auth import BearerAuthProvider

mcp = FastMCP(
    "researcher-bearer",
    auth=BearerAuthProvider(public_key=public_pem, issuer="...", audience="..."),
)
```

That one keyword argument is the whole change. Every tool registered with `@mcp.tool` is now behind bearer-token validation; the tool functions themselves are unchanged. The provider validates the `Authorization: Bearer <jwt>` header on each incoming MCP request, rejects with HTTP 401 if the token is missing, malformed, expired, or signed by the wrong key, and otherwise lets the call through.

FastMCP 2.x's `BearerAuthProvider` typically expects a **JWT signed with an RSA private key** and verifies it with the matching public key. That's not the only possible bearer style (opaque tokens, HMAC-signed JWTs, etc.), but it's the path of least resistance with the built-in provider. We generate a one-shot keypair in this notebook, sign a JWT with the private key, and configure the server with the public key.

> **Quirk:** Auth is a first-class `FastMCP(..., auth=...)` constructor parameter, not middleware the user wires by hand. The same tool definitions move between no-auth, bearer-auth, and OAuth2 setups by changing only the constructor call.

> **API note:** If `from fastmcp.server.auth import BearerAuthProvider` raises `ImportError`, the class has been renamed in your installed FastMCP version. Re-read the exports list printed by the Setup cell and substitute the current name (commonly `JWTAuthProvider` or `RemoteAuthProvider`). The constructor signature can also shift — `public_key=` vs `jwks_uri=` vs `verifier=`. The Setup cell printed the actual exports; the docstring of the class you choose will show its required arguments.
```

- [ ] **Step 2: Commit**

```bash
git add fastmcp/04_fastmcp_with_auth.ipynb
git commit -m "feat(fastmcp): explain bearer-token auth in notebook 04"
```

---

## Task 6: Section 3 — Generate the RSA keypair and a JWT

**Files:**
- Modify: `fastmcp/04_fastmcp_with_auth.ipynb` (add 1 code cell)

- [ ] **Step 1: Add the keypair + JWT code cell**

```python
# Generate one RSA keypair for the bearer demo. In production, the public key
# would come from your IdP's JWKS endpoint; here the same notebook plays both
# roles (issuer and verifier).
BEARER_ISSUER = "https://demo-issuer.local"
BEARER_AUDIENCE = "fastmcp-researcher"

bearer_private_pem, bearer_public_pem = generate_rsa_keypair()
print(f"Private key:  {len(bearer_private_pem)} bytes")
print(f"Public key:   {len(bearer_public_pem)} bytes")

# Sign a JWT the client will send. The same key signs it; the server-side
# provider will verify it against the matching public key.
bearer_jwt = sign_jwt(
    bearer_private_pem,
    issuer=BEARER_ISSUER,
    audience=BEARER_AUDIENCE,
    subject="demo-user",
)
print(f"\nSigned JWT (truncated): {bearer_jwt[:32]}…")
```

- [ ] **Step 2: Run, expect (the exact lengths and JWT prefix vary, but the shape should match)**

```
Private key:  1704 bytes
Public key:   451 bytes

Signed JWT (truncated): eyJhbGciOiJSUzI1NiIsInR5cCI6Ikp…
```

- [ ] **Step 3: Commit**

```bash
git add fastmcp/04_fastmcp_with_auth.ipynb
git commit -m "feat(fastmcp): generate RSA keypair and JWT for bearer demo in notebook 04"
```

---

## Task 7: Section 3 — Build the bearer-protected FastMCP server

**Files:**
- Modify: `fastmcp/04_fastmcp_with_auth.ipynb` (add 1 code cell)

- [ ] **Step 1: Add the bearer-server code cell**

```python
from fastmcp.server.auth import BearerAuthProvider

bearer_provider = BearerAuthProvider(
    public_key=bearer_public_pem.decode("utf-8"),
    issuer=BEARER_ISSUER,
    audience=BEARER_AUDIENCE,
)

bearer_mcp = FastMCP("researcher-bearer", auth=bearer_provider)


@bearer_mcp.tool
def research_topic(topic: str) -> list[str]:
    """Return canned facts about a well-known topic."""
    facts = {
        "octopuses": [
            "Octopuses have three hearts.",
            "They can change color and texture in milliseconds.",
        ],
        "rome": [
            "Rome was founded, per legend, in 753 BCE.",
            "The Roman Republic predates the Empire by ~500 years.",
        ],
    }
    return facts.get(topic.lower(), [])


bearer_thread = run_fastmcp_in_thread(bearer_mcp, port=8020)
print("Bearer-protected FastMCP server on http://127.0.0.1:8020")
```

- [ ] **Step 2: Run, expect**

```
Bearer-protected FastMCP server on http://127.0.0.1:8020
```

> **API note:** `BearerAuthProvider`'s exact constructor parameters can vary across FastMCP 2.x minor versions — some accept `public_key=` as a PEM string, others as bytes, and some expect `jwks_uri=` instead. If construction raises, inspect `help(BearerAuthProvider.__init__)` and adjust. The semantic intent (this server verifies RS256 JWTs signed by `bearer_private_pem`, issued by `BEARER_ISSUER`, scoped to `BEARER_AUDIENCE`) does not change.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/04_fastmcp_with_auth.ipynb
git commit -m "feat(fastmcp): build bearer-protected FastMCP server in notebook 04"
```

---

## Task 8: Section 3 — Demonstrate rejection without a token

**Files:**
- Modify: `fastmcp/04_fastmcp_with_auth.ipynb` (add 1 code cell)

- [ ] **Step 1: Add the no-token rejection cell**

```python
# First: try to call the server with NO Authorization header. We expect 401.
# We use raw httpx here (instead of the FastMCP Client) so the failure is
# visible at the HTTP layer — easier to inspect than the SDK's exception path.

unauthed_envelope = {
    "jsonrpc": "2.0",
    "id": str(uuid.uuid4()),
    "method": "tools/call",
    "params": {"name": "research_topic", "arguments": {"topic": "octopuses"}},
}

r = httpx.post(
    "http://127.0.0.1:8020/mcp",
    json=unauthed_envelope,
    headers={"Accept": "application/json, text/event-stream"},
    timeout=5.0,
)
print(f"No token:    HTTP {r.status_code}")
print(f"  body (truncated): {r.text[:200]}")
```

- [ ] **Step 2: Run, expect a 401 (the exact body wording is provider-dependent — anything in the 401/unauthorized family is correct)**

```
No token:    HTTP 401
  body (truncated): {"detail":"Missing or invalid Authorization header"}
```

> **API note:** Some FastMCP versions return the 401 directly with a JSON body; others wrap it in a JSON-RPC error envelope (HTTP 200 with an `error.code` of `-32001` or similar). Either is a valid "rejected" outcome — the point of the cell is that an unauthenticated request does not reach the tool. If you see a 200 with an MCP-level error, update the expected-output comment in the cell to match what your version actually returns.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/04_fastmcp_with_auth.ipynb
git commit -m "feat(fastmcp): demo no-token rejection on bearer server in notebook 04"
```

---

## Task 9: Section 3 — Demonstrate success with a valid token

**Files:**
- Modify: `fastmcp/04_fastmcp_with_auth.ipynb` (add 1 code cell)

- [ ] **Step 1: Add the authed-success cell**

```python
# Now: include the signed JWT in the Authorization header and call through the
# FastMCP Client (so the call goes through the normal SDK path, not raw httpx).
# Different FastMCP 2.x versions expose the auth header on Client in slightly
# different ways — pass `auth=BearerAuth(...)` if your version has it, or use
# the `headers={"Authorization": ...}` escape hatch.

bearer_headers = {"Authorization": f"Bearer {bearer_jwt}"}


async def _run_bearer_http():
    async with Client(
        "http://127.0.0.1:8020/mcp",
        headers=bearer_headers,
    ) as client:
        facts = await call_research(client, "octopuses")
        for fact in facts:
            print(f"  • {fact}")


asyncio.run(_run_bearer_http())

# Stop the bearer server before Section 4 spins up the OAuth2 variant.
shutdown_all_servers()
print("\nBearer server stopped.")
```

- [ ] **Step 2: Run, expect**

```
  • Octopuses have three hearts.
  • They can change color and texture in milliseconds.

Bearer server stopped.
```

> **API note:** If `Client(..., headers=...)` is not a supported kwarg in your installed FastMCP version, the alternative is to pass an `httpx_client_factory=` (or similarly named kwarg) and have it build an `httpx.AsyncClient(headers=bearer_headers)`. Verify against the installed `Client.__init__` signature: `help(Client.__init__)`. Either path achieves the same outcome — every outgoing request carries the `Authorization` header.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/04_fastmcp_with_auth.ipynb
git commit -m "feat(fastmcp): demo authed bearer call success in notebook 04"
```

---

## Task 10: Section 4 markdown — OAuth2 client credentials (concept)

**Files:**
- Modify: `fastmcp/04_fastmcp_with_auth.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Add the markdown cell**

```markdown
## 4. OAuth2 Client Credentials (with a Mocked IdP)

Bearer tokens are great if the token comes from somewhere — a console, a CLI, or, most often in production, an **Identity Provider (IdP)** issuing tokens to clients that authenticate to it. The OAuth2 *client-credentials* flow is the no-human-in-the-loop version of that handshake:

1. Client POSTs `client_id` + `client_secret` + `grant_type=client_credentials` to the IdP's `/token` endpoint.
2. IdP verifies the credentials, signs a JWT, returns `{"access_token": "...", "token_type": "Bearer", "expires_in": 3600}`.
3. Client uses `Authorization: Bearer <access_token>` on the actual FastMCP call.

The IdP and the FastMCP server are usually run by different organizations. Here we run a tiny mocked IdP on port `8021` inside this same notebook for the demo — a FastAPI app with one route, `POST /token`, that issues RS256-signed JWTs for a known `client_id` / `client_secret` pair.

The FastMCP side is structurally identical to Section 3 — same `BearerAuthProvider`, same RSA verification — the only difference is **whose key signs the token**. In production, the FastMCP server would fetch the IdP's public key from a JWKS URL; in this notebook we generate the IdP's keypair locally and configure the server with the public half directly.
```

- [ ] **Step 2: Commit**

```bash
git add fastmcp/04_fastmcp_with_auth.ipynb
git commit -m "feat(fastmcp): explain OAuth2 client-credentials concept in notebook 04"
```

---

## Task 11: Section 4 — Build the mocked IdP

**Files:**
- Modify: `fastmcp/04_fastmcp_with_auth.ipynb` (add 1 code cell)

- [ ] **Step 1: Add the IdP code cell**

```python
# The mocked IdP gets its own RSA keypair — separate from the bearer-demo
# key in Section 3, so the two demos don't share trust roots.
OAUTH_ISSUER = "https://mocked-idp.local"
OAUTH_AUDIENCE = "fastmcp-researcher"
KNOWN_CLIENTS = {"my-client-id": "my-client-secret"}

idp_private_pem, idp_public_pem = generate_rsa_keypair()
print(f"IdP keypair generated: private {len(idp_private_pem)} bytes, public {len(idp_public_pem)} bytes")


idp_app = FastAPI()


@idp_app.post("/token")
async def issue_token(
    client_id: str = Form(...),
    client_secret: str = Form(...),
    grant_type: str = Form(...),
) -> JSONResponse:
    if grant_type != "client_credentials":
        return JSONResponse({"error": "unsupported_grant_type"}, status_code=400)
    if KNOWN_CLIENTS.get(client_id) != client_secret:
        return JSONResponse({"error": "invalid_client"}, status_code=401)
    access_token = sign_jwt(
        idp_private_pem,
        issuer=OAUTH_ISSUER,
        audience=OAUTH_AUDIENCE,
        subject=client_id,
        ttl_seconds=3600,
    )
    return JSONResponse({
        "access_token": access_token,
        "token_type": "Bearer",
        "expires_in": 3600,
    })


idp_server = run_server_in_thread(idp_app, port=8021)
print("Mocked IdP running on http://127.0.0.1:8021 (POST /token)")
```

- [ ] **Step 2: Run, expect (key lengths may vary slightly)**

```
IdP keypair generated: private 1704 bytes, public 451 bytes
Mocked IdP running on http://127.0.0.1:8021 (POST /token)
```

- [ ] **Step 3: Commit**

```bash
git add fastmcp/04_fastmcp_with_auth.ipynb
git commit -m "feat(fastmcp): build mocked OAuth2 IdP in notebook 04"
```

---

## Task 12: Section 4 — Build the OAuth-protected FastMCP server

**Files:**
- Modify: `fastmcp/04_fastmcp_with_auth.ipynb` (add 1 code cell)

- [ ] **Step 1: Add the OAuth-server code cell**

```python
# Same BearerAuthProvider, configured to verify tokens signed by the IdP's
# private key (i.e., the IdP is the trusted issuer).
oauth_provider = BearerAuthProvider(
    public_key=idp_public_pem.decode("utf-8"),
    issuer=OAUTH_ISSUER,
    audience=OAUTH_AUDIENCE,
)

oauth_mcp = FastMCP("researcher-oauth", auth=oauth_provider)


@oauth_mcp.tool
def research_topic(topic: str) -> list[str]:
    """Return canned facts about a well-known topic."""
    facts = {
        "octopuses": [
            "Octopuses have three hearts.",
            "They can change color and texture in milliseconds.",
        ],
        "rome": [
            "Rome was founded, per legend, in 753 BCE.",
            "The Roman Republic predates the Empire by ~500 years.",
        ],
    }
    return facts.get(topic.lower(), [])


oauth_thread = run_fastmcp_in_thread(oauth_mcp, port=8020)
print("OAuth2-protected FastMCP server on http://127.0.0.1:8020")
print("(trusts tokens signed by the mocked IdP on :8021)")
```

- [ ] **Step 2: Run, expect**

```
OAuth2-protected FastMCP server on http://127.0.0.1:8020
(trusts tokens signed by the mocked IdP on :8021)
```

> **API note:** Same caveats as Task 7 about `BearerAuthProvider`'s constructor — adjust to whatever the installed FastMCP version exposes. The semantic stays: the server verifies RS256 JWTs signed by `idp_private_pem`, issued by `OAUTH_ISSUER`, scoped to `OAUTH_AUDIENCE`.

- [ ] **Step 3: Commit**

```bash
git add fastmcp/04_fastmcp_with_auth.ipynb
git commit -m "feat(fastmcp): build OAuth2-protected FastMCP server in notebook 04"
```

---

## Task 13: Section 4 — Fetch a token and call the server

**Files:**
- Modify: `fastmcp/04_fastmcp_with_auth.ipynb` (add 1 code cell)

- [ ] **Step 1: Add the client-side cell**

```python
# 1) Fetch an access token from the mocked IdP.
def fetch_oauth_token() -> str:
    r = httpx.post(
        "http://127.0.0.1:8021/token",
        data={
            "grant_type": "client_credentials",
            "client_id": "my-client-id",
            "client_secret": "my-client-secret",
        },
        timeout=5.0,
    )
    r.raise_for_status()
    return r.json()["access_token"]


access_token = fetch_oauth_token()
print(f"Got access token from IdP: {access_token[:32]}…")


# 2) Use it to call the OAuth-protected FastMCP server.
async def _run_oauth_http():
    async with Client(
        "http://127.0.0.1:8020/mcp",
        headers={"Authorization": f"Bearer {access_token}"},
    ) as client:
        facts = await call_research(client, "rome")
        for fact in facts:
            print(f"  • {fact}")


asyncio.run(_run_oauth_http())
```

- [ ] **Step 2: Run, expect (token prefix will differ)**

```
Got access token from IdP: eyJhbGciOiJSUzI1NiIsInR5cCI6Ikp…
  • Rome was founded, per legend, in 753 BCE.
  • The Roman Republic predates the Empire by ~500 years.
```

- [ ] **Step 3: Commit**

```bash
git add fastmcp/04_fastmcp_with_auth.ipynb
git commit -m "feat(fastmcp): fetch IdP token and call OAuth2-protected server in notebook 04"
```

---

## Task 14: Closing recap, teaser, and cleanup

**Files:**
- Modify: `fastmcp/04_fastmcp_with_auth.ipynb` (add 2 markdown cells + 1 code cell)

- [ ] **Step 1: Add the "What you just learned" markdown cell**

```markdown
## What you just learned

- FastMCP treats auth as a **constructor parameter** (`FastMCP(..., auth=...)`), not middleware you wire up by hand. The same tool definitions move between auth modes by changing only the constructor call.
- The "no auth" default is the implicit behavior of `FastMCP("name")` with no `auth=` — perfectly fine for local development, but it should be a deliberate choice.
- A `BearerAuthProvider` (or its current 2.x equivalent — verify the import) validates `Authorization: Bearer <jwt>` against a configured public key, rejecting unauthenticated requests with HTTP 401.
- An OAuth2 client-credentials flow is just bearer-token validation where the token comes from an Identity Provider. The FastMCP side is unchanged — only the trust root (whose public key it verifies against) differs.
- Auth provider class names and constructor signatures have shifted between FastMCP 2.x minor versions. Always inspect `dir(fastmcp.server.auth)` and `help(...)` if your installed version disagrees with a tutorial.
```

- [ ] **Step 2: Add the "What's missing" markdown cell**

```markdown
## What's missing

We have a server that knows who it's talking to. We do **not** yet have:

- A clean way to open expensive resources (database connections, model handles) once at server start and dispose of them at shutdown.
- A way for tools to emit progress updates or structured logs back to the caller while a long-running operation is in flight.
- A way to compose two FastMCP apps under one server (e.g., a "research" sub-app and a "writer" sub-app) so a single client connection sees both.

In **notebook 05** we add `lifespan` for startup/teardown, the auto-injected `Context` parameter for in-tool logging and progress, and `mount()` for composing FastMCP apps. These are the framework features that pay off once your server graduates from a toy.
```

- [ ] **Step 3: Add the cleanup code cell**

```markdown
## 5. Cleanup
```

(That is a small markdown cell preceding the code cell below.)

- [ ] **Step 4: Add the cleanup code cell proper**

```python
shutdown_all_servers()
print("All servers stopped.")
```

- [ ] **Step 5: Run the cleanup cell and verify**

Expected output:

```
All servers stopped.
```

- [ ] **Step 6: Commit**

```bash
git add fastmcp/04_fastmcp_with_auth.ipynb
git commit -m "feat(fastmcp): add closing recap and cleanup to notebook 04"
```

---

## Task 15: End-to-end verification in a fresh kernel

**Files:** none (verification only).

- [ ] **Step 1: Execute the notebook top-to-bottom in a clean kernel**

```bash
/opt/anaconda3/bin/jupyter nbconvert --to notebook --execute --inplace fastmcp/04_fastmcp_with_auth.ipynb
```

Expected: succeeds with no errors. The notebook makes a handful of HTTP calls and a few in-memory calls; total execution should be well under 30 seconds.

- [ ] **Step 2: Verify expected outputs in the executed notebook**

In order, the executed cells should contain:

1. The FastMCP version string and the `fastmcp.server.auth` exports list, ending with `Setup OK`.
2. The two octopus facts (in-memory call against the no-auth instance).
3. `No-auth FastMCP server on http://127.0.0.1:8020`, the two Rome facts, and `No-auth server stopped.`.
4. The RSA keypair byte-counts and a truncated JWT prefix.
5. `Bearer-protected FastMCP server on http://127.0.0.1:8020`.
6. `No token:    HTTP 401` (or its JSON-RPC-error equivalent — see Task 8's API note).
7. The two octopus facts again (authed call) plus `Bearer server stopped.`.
8. The IdP keypair byte-counts and `Mocked IdP running on http://127.0.0.1:8021 (POST /token)`.
9. `OAuth2-protected FastMCP server on http://127.0.0.1:8020` plus the trust note.
10. The truncated IdP access-token prefix and the two Rome facts.
11. `All servers stopped.`

No cell raises an unhandled exception. If a port-collision error appears, free the ports first: `lsof -ti tcp:8020 -ti tcp:8021 | xargs kill -9` (only kill processes you recognize).

- [ ] **Step 3: Verify ports are free after cleanup**

```bash
lsof -iTCP:8020 -sTCP:LISTEN
lsof -iTCP:8021 -sTCP:LISTEN
```

Expected: no output from either command.

- [ ] **Step 4: Commit the clean run**

```bash
git add fastmcp/04_fastmcp_with_auth.ipynb
git commit -m "chore(fastmcp): commit clean fresh-kernel run of notebook 04"
```

---

## Done

After Task 15 passes, notebook 04 is complete. The next plan to write is for `05_fastmcp_production_patterns.ipynb` — see `docs/superpowers/plans/2026-05-18-fastmcp-notebook-05-production-patterns.md`.
