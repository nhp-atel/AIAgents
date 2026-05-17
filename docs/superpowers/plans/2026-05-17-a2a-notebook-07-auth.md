# A2A Notebook 07 — `07_a2a_auth.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the seventh notebook in the A2A learning series — **authentication**. Cover three modes: no-auth (made explicit), bearer token, and OAuth2 client credentials. Then revisit the webhook flow from notebook 06 and **sign** the callback so the receiver can verify the server actually sent it.

**Architecture:** A single Jupyter notebook with three FastAPI apps:

- App #1: the researcher on `8010` — accepts authenticated `message/send` requests; declares its auth requirements in its Agent Card.
- App #2: a mocked OAuth2 identity provider on `8012` — accepts `client_credentials` requests at `POST /token` and returns a synthetic access token.
- App #3: a callback receiver on `8011` — receives the researcher's webhook POST, verifies the `X-A2A-Signature` HMAC header before trusting the payload.

The notebook walks each auth mode as a self-contained demo, then ties them together: bearer-authenticated `message/send` + signed webhook delivery.

**Tech Stack:** Python 3.11+, Jupyter, FastAPI, uvicorn, httpx, pydantic v2, threading. Uses `hmac` / `hashlib` from the standard library for webhook signing — no new dependency. JWT is mentioned for production context but not implemented; HMAC-SHA256 with a shared secret teaches the same concept.

**A2A spec version targeted:** v0.3.0. Auth is modeled with OpenAPI-style `securitySchemes` (named scheme definitions) and `security` (required scheme combinations) per notebook 02.

**Port assignment:** `8010` researcher, `8011` callback receiver, `8012` mocked IdP.

---

## File Structure

- **Create:** `07_a2a_auth.ipynb` — self-contained.

## Notebook Section Map

1. Title + "Why this notebook exists"
2. "What you'll learn"
3. Setup
4. Section 1: A2A auth lives in the Agent Card (markdown)
5. Section 2: Pattern 0 — no auth, made explicit (markdown + brief code)
6. Section 3: Pattern 1 — bearer token (markdown + server + client demo with/without token)
7. Section 4: Pattern 2 — OAuth2 client credentials (markdown + mocked IdP + client gets token + calls researcher)
8. Section 5: Pattern 3 — signed push notifications (markdown + server signs webhook + receiver verifies)
9. "What you just learned"
10. "What's missing"
11. Cleanup

---

## Task 1: Scaffold

**Files:** Create `07_a2a_auth.ipynb`.

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
python -c "import json; json.load(open('07_a2a_auth.ipynb'))"
```

- [ ] **Step 3: Commit**

```bash
git add 07_a2a_auth.ipynb
git commit -m "feat(a2a): scaffold 07_a2a_auth.ipynb"
```

---

## Task 2: Title + intro markdown

**Files:** Modify `07_a2a_auth.ipynb` (2 markdown cells).

- [ ] **Step 1: Title + "Why"**

```markdown
# 07 — Authentication: Bearer, OAuth2, and Signed Webhooks

## Why this notebook exists

Every endpoint and webhook we've built so far (notebooks 02–06) trusts anyone who can reach the URL. That's fine on localhost; it's a serious problem anywhere else.

A2A leans on **OpenAPI's auth model**: the Agent Card declares one or more **`securitySchemes`** (named definitions of *how* auth works) and a **`security`** list (which combinations are required to call this agent). A client reads the card, picks a scheme it can satisfy, and attaches the right credentials.

This notebook walks three real schemes — bearer token, OAuth2 client credentials, and a signed webhook — using the same researcher we've been building, but now with auth enforced.

> *Targets A2A spec v0.3.0. We use HMAC-SHA256 in a header for the signed webhook because it teaches the concept with no extra dependencies; production A2A typically uses JWS/JWT for the same purpose.*
```

- [ ] **Step 2: "What you'll learn"**

```markdown
## What you'll learn

- How A2A's `securitySchemes` / `security` fields work and how they map onto OpenAPI's auth model.
- The shape of a **bearer-token** scheme: declared in the card, attached as `Authorization: Bearer …`, validated server-side.
- The shape of an **OAuth2 client-credentials** flow: client posts `client_id` / `client_secret` to a token endpoint, receives an `access_token`, and uses it as a bearer.
- How to **sign** a webhook callback with HMAC-SHA256 in an `X-A2A-Signature` header, and how the receiver verifies it.
- Why "no auth" is itself a security choice that should be declared, not implied.
- The path from these primitives to production-grade A2A auth (JWT, mTLS, real OAuth flows).
```

- [ ] **Step 3: Commit**

```bash
git add 07_a2a_auth.ipynb
git commit -m "feat(a2a): add intro markdown to notebook 07"
```

---

## Task 3: Setup

**Files:** Modify `07_a2a_auth.ipynb` (markdown header + code cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 1. Setup

Same helpers as previous notebooks, plus `hmac` and `hashlib` for webhook signing.
```

- [ ] **Step 2: Setup code cell**

```python
import hashlib
import hmac
import json
import threading
import time
import uuid
from datetime import datetime, timezone

import httpx
import uvicorn
from fastapi import FastAPI, HTTPException, Request, Header
from fastapi.responses import JSONResponse
from pydantic import BaseModel, Field

_servers: list[uvicorn.Server] = []


def run_server_in_thread(app: FastAPI, port: int) -> uvicorn.Server:
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


def now_iso() -> str:
    return datetime.now(timezone.utc).isoformat()


print("Setup OK")
```

- [ ] **Step 3: Run, expect `Setup OK`**

- [ ] **Step 4: Commit**

```bash
git add 07_a2a_auth.ipynb
git commit -m "feat(a2a): add setup cell to notebook 07"
```

---

## Task 4: Section 1 — Auth lives in the Agent Card

**Files:** Modify `07_a2a_auth.ipynb` (1 markdown cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 2. Auth Lives in the Agent Card

Two top-level fields on the Agent Card carry all of A2A's auth contract:

| Field | Shape | Purpose |
|---|---|---|
| `securitySchemes` | `{ [name: string]: SecurityScheme }` | Named definitions of authentication methods this agent supports |
| `security` | `[{ [name: string]: string[] }]` | List of acceptable scheme combinations (logical OR between list entries, logical AND within each entry) |

A **`SecurityScheme`** mirrors OpenAPI's:

- `type: "http"` with `scheme: "bearer"` — opaque bearer tokens.
- `type: "oauth2"` with `flows: {...}` — OAuth2 flows (we use `clientCredentials`).
- `type: "apiKey"` with `name: ...` and `in: "header" | "query" | "cookie"` — API keys.
- `type: "mutualTLS"` — client certificates.
- `type: "openIdConnect"` — OIDC.

A typical card has one or two schemes and a single-element `security` list. Multiple list entries express "any of these is acceptable" (e.g., bearer OR oauth2). Multiple keys in one entry express "all of these required together" (e.g., bearer AND a separate API key for tenant routing).

For each demo below we'll show the Agent Card snippet that declares the scheme alongside the code that enforces it.
```

- [ ] **Step 2: Commit**

```bash
git add 07_a2a_auth.ipynb
git commit -m "feat(a2a): explain securitySchemes/security in notebook 07"
```

---

## Task 5: Section 2 — Pattern 0: no auth, made explicit

**Files:** Modify `07_a2a_auth.ipynb` (markdown + code cell).

- [ ] **Step 1: Markdown cell**

```markdown
## 3. Pattern 0: No Auth (Explicitly)

Notebooks 02–06 quietly assumed no auth. The Agent Card was silent on `securitySchemes` and `security`, leaving the empty defaults from notebook 02. That's not wrong — it's the *unauthenticated* default — but it's worth making explicit so a client doesn't have to guess.

```json
{
  "securitySchemes": {},
  "security": []
}
```

An empty `security` list means *"no scheme is required."* Clients should still send the headers they consider polite (e.g., `User-Agent`), but no `Authorization` is needed.
```

- [ ] **Step 2: Commit**

```bash
git add 07_a2a_auth.ipynb
git commit -m "feat(a2a): document no-auth as explicit default in notebook 07"
```

---

## Task 6: Section 3 — Pattern 1: bearer token

**Files:** Modify `07_a2a_auth.ipynb` (markdown + 3 code cells).

- [ ] **Step 1: Markdown cell**

```markdown
## 4. Pattern 1: Bearer Token

The card declares:

```json
{
  "securitySchemes": {
    "bearer": {"type": "http", "scheme": "bearer"}
  },
  "security": [{"bearer": []}]
}
```

The server requires `Authorization: Bearer <token>` on every request and rejects with HTTP 401 if missing or invalid. The token is opaque — the spec doesn't say where it comes from (could be from a console, a CLI, an OAuth flow). For this demo we hard-code one.
```

- [ ] **Step 2: Bearer server code cell**

```python
BEARER_TOKEN = "demo-token-12345"  # opaque; in production it'd come from issuance


bearer_app = FastAPI()


def _require_bearer(authorization: str | None) -> None:
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Missing or malformed Authorization header")
    token = authorization[len("Bearer "):]
    if not hmac.compare_digest(token, BEARER_TOKEN):
        raise HTTPException(status_code=401, detail="Invalid bearer token")


@bearer_app.get("/.well-known/agent-card.json")
def bearer_card() -> dict:
    return {
        "protocolVersion": "0.3.0",
        "name": "Researcher (bearer-protected)",
        "description": "Returns canned facts; requires a bearer token.",
        "url": "http://127.0.0.1:8010",
        "version": "0.1.0",
        "capabilities": {"streaming": False, "pushNotifications": False, "stateTransitionHistory": False},
        "defaultInputModes": ["text"],
        "defaultOutputModes": ["text"],
        "skills": [{
            "id": "research_topic", "name": "Research a topic",
            "description": "Given a topic, return facts.", "tags": ["research"],
            "examples": ["octopuses"],
        }],
        "securitySchemes": {"bearer": {"type": "http", "scheme": "bearer"}},
        "security": [{"bearer": []}],
    }


@bearer_app.post("/")
async def bearer_jsonrpc(request: Request, authorization: str | None = Header(default=None)) -> dict:
    _require_bearer(authorization)
    body = await request.json()
    # Trivial echo back as a completed task — focus is auth, not protocol depth.
    text = body["params"]["message"]["parts"][0]["text"]
    facts = {"octopuses": ["Octopuses have three hearts."]}.get(text.lower(), [])
    return {
        "jsonrpc": "2.0",
        "id": body["id"],
        "result": {
            "id": str(uuid.uuid4()),
            "contextId": str(uuid.uuid4()),
            "status": {"state": "completed" if facts else "failed", "timestamp": now_iso()},
            "kind": "task",
            "artifacts": [{
                "artifactId": str(uuid.uuid4()),
                "name": f"facts-about-{text.lower()}",
                "parts": [{"kind": "text", "text": fact} for fact in facts],
            }] if facts else [],
        },
    }


bearer_server = run_server_in_thread(bearer_app, port=8010)
print("Bearer-protected researcher on http://127.0.0.1:8010")
```

- [ ] **Step 3: Bearer client demo — without token**

```python
def send(text: str, headers: dict | None = None) -> httpx.Response:
    envelope = {
        "jsonrpc": "2.0",
        "id": str(uuid.uuid4()),
        "method": "message/send",
        "params": {"message": {
            "messageId": str(uuid.uuid4()), "role": "user",
            "parts": [{"kind": "text", "text": text}], "kind": "message",
        }},
    }
    return httpx.post("http://127.0.0.1:8010/", json=envelope, headers=headers or {}, timeout=5.0)


# Attempt 1: no token.
r1 = send("octopuses")
print(f"No token:    HTTP {r1.status_code}  {r1.json()}")
```

- [ ] **Step 4: Run, expect**

```
No token:    HTTP 401  {'detail': 'Missing or malformed Authorization header'}
```

- [ ] **Step 5: Bearer client demo — with token**

```python
r2 = send("octopuses", headers={"Authorization": f"Bearer {BEARER_TOKEN}"})
print(f"With token:  HTTP {r2.status_code}")
result = r2.json()["result"]
print(f"  state: {result['status']['state']}")
for art in result["artifacts"]:
    for part in art["parts"]:
        print(f"  • {part['text']}")
```

- [ ] **Step 6: Run, expect**

```
With token:  HTTP 200
  state: completed
  • Octopuses have three hearts.
```

- [ ] **Step 7: Commit**

```bash
git add 07_a2a_auth.ipynb
git commit -m "feat(a2a): implement bearer-token auth in notebook 07"
```

---

## Task 7: Section 4 — Pattern 2: OAuth2 client credentials

**Files:** Modify `07_a2a_auth.ipynb` (markdown + 3 code cells).

- [ ] **Step 1: Markdown cell**

```markdown
## 5. Pattern 2: OAuth2 Client Credentials

The card declares an OAuth2 scheme with the `clientCredentials` flow:

```json
{
  "securitySchemes": {
    "oauth2": {
      "type": "oauth2",
      "flows": {
        "clientCredentials": {
          "tokenUrl": "http://127.0.0.1:8012/token",
          "scopes": {"research": "Read research data"}
        }
      }
    }
  },
  "security": [{"oauth2": ["research"]}]
}
```

The flow:

1. Client POSTs `client_id` / `client_secret` / `grant_type=client_credentials` to the IdP's `tokenUrl`.
2. IdP returns `{access_token, token_type, expires_in}`.
3. Client uses `Authorization: Bearer <access_token>` on the actual A2A call.

The IdP and the agent are usually run by different organizations. Here we run a mocked IdP on port 8012 inside this same notebook for the demo.

The agent's server-side check is **the same** as the bearer pattern from Section 4 — it just trusts tokens issued by a known IdP. (In production you'd verify the IdP's signature on the token; for the mocked IdP we use opaque tokens and an in-memory issued-token set.)
```

- [ ] **Step 2: Mocked IdP**

```python
ISSUED_TOKENS: set[str] = set()
TOKENS_LOCK = threading.Lock()
KNOWN_CLIENTS = {"my-client-id": "my-client-secret"}


idp_app = FastAPI()


@idp_app.post("/token")
async def issue_token(request: Request) -> JSONResponse:
    form = await request.form()
    client_id = form.get("client_id")
    client_secret = form.get("client_secret")
    grant_type = form.get("grant_type")
    if grant_type != "client_credentials":
        return JSONResponse({"error": "unsupported_grant_type"}, status_code=400)
    if KNOWN_CLIENTS.get(client_id) != client_secret:
        return JSONResponse({"error": "invalid_client"}, status_code=401)
    token = f"oauth-{uuid.uuid4()}"
    with TOKENS_LOCK:
        ISSUED_TOKENS.add(token)
    return JSONResponse({"access_token": token, "token_type": "Bearer", "expires_in": 3600})


idp_server = run_server_in_thread(idp_app, port=8012)
print("Mocked IdP running on http://127.0.0.1:8012 (POST /token)")
```

- [ ] **Step 3: Replace the bearer check to trust the IdP's tokens**

```python
# The OAuth2-protected agent reuses the bearer_app's handler — but we widen the
# accepted token set: either the hard-coded BEARER_TOKEN (from Section 4) or any
# token the mocked IdP has issued.

def _require_oauth_token(authorization: str | None) -> None:
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Missing or malformed Authorization header")
    token = authorization[len("Bearer "):]
    with TOKENS_LOCK:
        if token == BEARER_TOKEN or token in ISSUED_TOKENS:
            return
    raise HTTPException(status_code=401, detail="Invalid bearer token")


# Monkey-patch the existing app's dependency. Cleaner alternatives exist (separate
# app on a different port), but reusing port 8010 keeps the demo focused.
bearer_app.dependency_overrides[_require_bearer] = _require_oauth_token
print("Researcher now accepts OAuth2-issued tokens too.")
```

- [ ] **Step 4: Client side — fetch token, then call**

```python
def fetch_oauth_token() -> str:
    r = httpx.post(
        "http://127.0.0.1:8012/token",
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
print(f"Got access token: {access_token[:12]}…")

r = send("octopuses", headers={"Authorization": f"Bearer {access_token}"})
print(f"OAuth-authed call: HTTP {r.status_code}")
result = r.json()["result"]
print(f"  state: {result['status']['state']}")
```

- [ ] **Step 5: Run, expect**

```
Got access token: oauth-xxxxxxxx…
OAuth-authed call: HTTP 200
  state: completed
```

- [ ] **Step 6: Commit**

```bash
git add 07_a2a_auth.ipynb
git commit -m "feat(a2a): implement OAuth2 client credentials in notebook 07"
```

---

## Task 8: Section 5 — Pattern 3: signed push notifications

**Files:** Modify `07_a2a_auth.ipynb` (markdown + 3 code cells).

- [ ] **Step 1: Markdown cell**

```markdown
## 6. Pattern 3: Signed Push Notifications

Webhook receivers face a unique threat: the URL is reachable by *anyone* who knows it. Auth tokens flow the other direction (client → server), so a different mechanism is needed to authenticate **server → webhook** delivery.

A2A's answer is: declare an auth scheme on the `pushNotificationConfig.authentication` field, exactly like the agent card declares its own. Common patterns:

- **HMAC signature** — server and receiver share a secret; server signs the payload with HMAC-SHA256 and sends the digest in `X-A2A-Signature`. Receiver recomputes and compares.
- **JWT signing** — server signs a JWT with a private key; receiver verifies with the public key (no shared secret needed).

We implement the HMAC variant (no extra dependencies). The same primitives apply to JWT — substitute `jwt.encode` / `jwt.decode` for `hmac.new` / `hmac.compare_digest`.
```

- [ ] **Step 2: Callback receiver with signature verification**

```python
SHARED_SECRET = b"shared-webhook-secret-do-not-reuse"
RECEIVED_VALID: list[dict] = []
RECEIVED_INVALID: list[dict] = []
RECEIVED_LOCK = threading.Lock()


callback_app = FastAPI()


def _verify_signature(body: bytes, signature_header: str | None) -> bool:
    if not signature_header:
        return False
    expected = hmac.new(SHARED_SECRET, body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature_header)


@callback_app.post("/callback")
async def receive_callback(
    request: Request,
    x_a2a_signature: str | None = Header(default=None),
) -> dict:
    body = await request.body()
    parsed = json.loads(body)
    if _verify_signature(body, x_a2a_signature):
        with RECEIVED_LOCK:
            RECEIVED_VALID.append(parsed)
        return {"received": True, "verified": True}
    with RECEIVED_LOCK:
        RECEIVED_INVALID.append(parsed)
    return JSONResponse({"received": True, "verified": False}, status_code=401)


callback_server = run_server_in_thread(callback_app, port=8011)
print("Signing-aware callback receiver on http://127.0.0.1:8011 (POST /callback)")
```

- [ ] **Step 3: Server-side webhook delivery with signing**

```python
def deliver_signed_webhook(url: str, task_payload: dict, secret: bytes) -> None:
    body = json.dumps(task_payload).encode("utf-8")
    signature = hmac.new(secret, body, hashlib.sha256).hexdigest()
    try:
        r = httpx.post(
            url,
            content=body,
            headers={"Content-Type": "application/json", "X-A2A-Signature": signature},
            timeout=5.0,
        )
        print(f"  delivery to {url}: HTTP {r.status_code} {r.json()}")
    except Exception as e:
        print(f"  delivery to {url} failed: {e!r}")


fake_task = {
    "id": str(uuid.uuid4()),
    "contextId": str(uuid.uuid4()),
    "status": {"state": "completed", "timestamp": now_iso()},
    "kind": "task",
    "artifacts": [],
}

print("Delivery with correct secret:")
deliver_signed_webhook("http://127.0.0.1:8011/callback", fake_task, SHARED_SECRET)

print("\nDelivery with WRONG secret (attacker simulation):")
deliver_signed_webhook("http://127.0.0.1:8011/callback", fake_task, b"wrong-secret")

with RECEIVED_LOCK:
    print(f"\nReceiver mailbox: {len(RECEIVED_VALID)} verified, {len(RECEIVED_INVALID)} rejected.")
```

- [ ] **Step 4: Run and verify**

Expected output:

```
Delivery with correct secret:
  delivery to http://127.0.0.1:8011/callback: HTTP 200 {'received': True, 'verified': True}

Delivery with WRONG secret (attacker simulation):
  delivery to http://127.0.0.1:8011/callback: HTTP 401 {'received': True, 'verified': False}

Receiver mailbox: 1 verified, 1 rejected.
```

- [ ] **Step 5: Commit**

```bash
git add 07_a2a_auth.ipynb
git commit -m "feat(a2a): implement signed push notifications in notebook 07"
```

---

## Task 9: Closing recap, teaser, cleanup

**Files:** Modify `07_a2a_auth.ipynb` (2 markdown cells + 1 code cell).

- [ ] **Step 1: "What you just learned"**

```markdown
## What you just learned

- A2A puts auth in the Agent Card via `securitySchemes` (named definitions) and `security` (required combinations) — same shape as OpenAPI.
- **Bearer tokens** are the simplest path: header in, header out, server validates with constant-time compare.
- **OAuth2 client credentials** add a token-issuance step at the IdP, but the agent's server-side check is the same as for an opaque bearer.
- **Signed webhooks** authenticate the *server → receiver* direction. HMAC-SHA256 with a shared secret in `X-A2A-Signature` is the simplest scheme; JWT with public-key verification is the production-grade variant.
- "No auth" is itself a security choice — declare it (`securitySchemes: {}`, `security: []`) rather than leaving the card silent.
```

- [ ] **Step 2: "What's missing"**

```markdown
## What's missing

We've been hand-rolling every byte of A2A for seven notebooks. That's been deliberate — you can now look at any A2A request, response, event, or webhook and know what it means.

In production you stop hand-rolling. Google publishes an official Python SDK, **`a2a-sdk`**, that handles the protocol mechanics so your code can focus on the agent's actual behavior. In **notebook 08** we install the SDK, rebuild the researcher + writer agents from notebook 01 using the SDK's helpers, and add a **coordinator** that orchestrates them via A2A. The notebook closes with a side-by-side diff: 50+ lines of hand-rolled JSON-RPC per agent call versus ~5 lines with the SDK.

That's the payoff. Everything we've built so far made the SDK's choices legible.
```

- [ ] **Step 3: Cleanup code cell**

```python
shutdown_all_servers()
print("All servers stopped.")
```

- [ ] **Step 4: Run, expect `All servers stopped.`**

- [ ] **Step 5: Commit**

```bash
git add 07_a2a_auth.ipynb
git commit -m "feat(a2a): add closing recap and cleanup to notebook 07"
```

---

## Task 10: Fresh-kernel verification

**Files:** none.

- [ ] **Step 1: Execute**

```bash
/opt/anaconda3/bin/jupyter nbconvert --to notebook --execute --inplace 07_a2a_auth.ipynb
```

Expected: succeeds with no errors. The notebook makes ~6 HTTP calls; total execution should be under 5 seconds.

- [ ] **Step 2: Verify expected outputs**

In order:

1. `Setup OK`
2. `Bearer-protected researcher on http://127.0.0.1:8010`
3. `No token:    HTTP 401  {'detail': 'Missing or malformed Authorization header'}`
4. `With token:  HTTP 200` + `state: completed` + `• Octopuses have three hearts.`
5. `Mocked IdP running on http://127.0.0.1:8012 (POST /token)`
6. `Researcher now accepts OAuth2-issued tokens too.`
7. `Got access token: oauth-xxxxxxxx…` (8 truncated chars) + `OAuth-authed call: HTTP 200` + `state: completed`.
8. `Signing-aware callback receiver on http://127.0.0.1:8011 (POST /callback)`
9. Signed delivery block: HTTP 200 verified, HTTP 401 rejected, mailbox `1 verified, 1 rejected.`
10. `All servers stopped.`

No cell raises an unhandled exception.

- [ ] **Step 3: Verify ports released**

```bash
lsof -iTCP:8010 -sTCP:LISTEN
lsof -iTCP:8011 -sTCP:LISTEN
lsof -iTCP:8012 -sTCP:LISTEN
```

Expected: all three empty.

- [ ] **Step 4: Commit clean run**

```bash
git add 07_a2a_auth.ipynb
git commit -m "chore(a2a): commit clean fresh-kernel run of notebook 07"
```

---

## Done

After Task 10 passes, notebook 07 is complete. Next plan: `08_a2a_multi_agent.ipynb` — the official SDK and the multi-agent payoff.
