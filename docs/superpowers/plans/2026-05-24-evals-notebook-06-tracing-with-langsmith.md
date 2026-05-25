# Evals Notebook 06 — `06_tracing_with_langsmith.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build notebook 06 in the Agent Evals & Observability learning series — instrument a small multi-step agent with LangSmith tracing, read the resulting span tree, debug a deliberately-broken run by localizing the failure to a specific span, and surface per-run latency and token counts. The notebook is self-contained: it builds its own small traced agent and does **not** depend on the eval harness from notebook 03 (though it references it in prose).

**Architecture:** A single Jupyter notebook that (a) checks for required env vars (`OPENAI_API_KEY`, `LANGCHAIN_API_KEY`, `LANGCHAIN_TRACING_V2`), (b) traces a single LLM call, (c) traces a 2–3 step agent with nested spans, (d) deliberately breaks the agent and reads the trace to localize the failure, (e) reads latency and token usage from the run object, and (f) names alternative observability tools. Tracing is sent to the hosted LangSmith UI — the notebook prints run URLs and instructs the learner to open them; the notebook cannot render the LangSmith UI inline.

**Tech Stack:** Python 3.11+, Jupyter, `langsmith` (SDK, `@traceable` decorator, `Client`), `langchain`, `langchain-openai` (`ChatOpenAI`), `openai`, `gpt-4o-mini`.

**Credentials required:** `OPENAI_API_KEY` (LLM calls), `LANGCHAIN_API_KEY` (LangSmith account key), `LANGCHAIN_TRACING_V2=true` (enables tracing globally). A free LangSmith account is needed; sign up at https://smith.langchain.com.

**Companion spec:** `docs/superpowers/specs/2026-05-24-evals-observability-learning-series-design.md` (notebook 06 section).

**Folder:** The `evals/` folder is created by notebook 01 of this series. Task 1 of this plan scaffolds `evals/06_tracing_with_langsmith.ipynb` inside that folder.

---

## File Structure

- **Create:** `evals/06_tracing_with_langsmith.ipynb` — the entire notebook, self-contained.
- **Modify:** none.
- **No separate test files** — verification is the notebook running top-to-bottom in a fresh kernel (Task 9).

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown)
2. **"What you'll learn"** (markdown bullets)
3. **Section 1: Setup + env-var guard** (markdown + 1 code cell — pip install note, imports, guard cell that checks all three env vars and prints setup instructions with account URL if any are missing)
4. **Section 2: Trace a single LLM call** (markdown + 1 code cell — a `@traceable`-decorated function calling `ChatOpenAI`; prints the run URL)
5. **Section 3: Trace a multi-step agent** (markdown + 2 code cells — define three `@traceable` steps forming a retrieve → synthesize → format agent; run it; explain the span tree the learner sees in the UI)
6. **Section 4: Debug a broken run** (markdown + 1 code cell — introduce a deliberate bug in step 1; run it; show how the trace isolates the failure to a specific span)
7. **Section 5: Latency, tokens, cost at a glance** (markdown + 1 code cell — fetch the run object via `langsmith.Client`; print latency, prompt tokens, completion tokens)
8. **Section 6: One-line note on alternatives** (markdown — Langfuse, Arize Phoenix)
9. **"What you just learned"** (markdown)
10. **"What's missing"** (markdown teaser pointing to notebook 07)

---

## Task 1: Scaffold the empty notebook

**Files:**
- Create: `evals/06_tracing_with_langsmith.ipynb`

- [ ] **Step 1: Verify the `evals/` folder exists**

```bash
ls -la evals/
```

Expected: the folder exists (created by notebook 01). If it does not, create it:

```bash
mkdir evals
```

- [ ] **Step 2: Create an empty notebook with the Python 3 kernel**

Write the file directly:

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

Save as `evals/06_tracing_with_langsmith.ipynb`.

- [ ] **Step 3: Verify the file is valid JSON**

```bash
python -c "import json; json.load(open('evals/06_tracing_with_langsmith.ipynb'))"
ls -la evals/06_tracing_with_langsmith.ipynb
```

Expected: no exception, file non-zero size.

- [ ] **Step 4: Commit**

```bash
git add evals/06_tracing_with_langsmith.ipynb
git commit -m "feat(evals): scaffold 06_tracing_with_langsmith.ipynb"
```

---

## Task 2: Add title and "Why this notebook exists"

**Files:**
- Modify: `evals/06_tracing_with_langsmith.ipynb` (add cell 1)

- [ ] **Step 1: Add a markdown cell with title and motivation**

Cell content (markdown):

```markdown
# 06 — Tracing with LangSmith

## Why this notebook exists

In **notebook 05** we built an LLM-as-judge that can score open-ended agent outputs against a rubric. We can now *measure* whether an agent is doing well. But when a score drops, we can't yet *see* why: which step produced bad output? Where did the agent spend its time? Which span consumed the most tokens?

Traditional logging is too coarse — a wall of text with no structure. What we need is **tracing**: a hierarchical record of every function call in a run, with its inputs, outputs, latency, and token usage captured per span. LangSmith provides exactly that, for free, through a hosted UI that requires no infrastructure from us.

This notebook instruments a small multi-step agent with LangSmith tracing, shows you how to read the resulting span tree, demonstrates how a trace localises a bug to a single failing span, and surfaces per-run latency and token counts. At the end, the agent outputs we've been scoring in blind eval land will have a full call graph attached.
```

- [ ] **Step 2: Verify the cell renders**

Open the notebook in Jupyter/VS Code and confirm the cell renders as markdown.

- [ ] **Step 3: Commit**

```bash
git add evals/06_tracing_with_langsmith.ipynb
git commit -m "feat(evals): add intro markdown to notebook 06"
```

---

## Task 3: Add "What you'll learn"

**Files:**
- Modify: `evals/06_tracing_with_langsmith.ipynb` (add cell 2)

- [ ] **Step 1: Add a markdown cell**

Cell content (markdown):

```markdown
## What you'll learn

- How to set up LangSmith tracing with three env vars (`OPENAI_API_KEY`, `LANGCHAIN_API_KEY`, `LANGCHAIN_TRACING_V2=true`) and a free hosted account.
- How the `@traceable` decorator from the `langsmith` SDK turns any Python function into a traced span — inputs, outputs, latency, and token counts captured automatically.
- How LangChain's `ChatOpenAI` auto-traces without any decorator when `LANGCHAIN_TRACING_V2=true` is set.
- How to build a small multi-step agent where each step is its own span, giving you a **nested span tree** in the LangSmith UI: parent run → child spans → per-span details.
- How to use a trace to **debug a broken run**: the UI (and the run object) pinpoint which span raised an error or produced malformed output, so you don't have to re-read logs.
- Where LangSmith surfaces **latency, prompt tokens, completion tokens, and cost** for each run, and how to read these programmatically via the `Client`.
- The names of two open-source alternatives (Langfuse, Arize Phoenix) so you know they exist.
```

- [ ] **Step 2: Verify the cell renders**

The bullets should render properly.

- [ ] **Step 3: Commit**

```bash
git add evals/06_tracing_with_langsmith.ipynb
git commit -m "feat(evals): add 'What you'll learn' to notebook 06"
```

---

## Task 4: Section 1 — Setup + env-var guard

**Files:**
- Modify: `evals/06_tracing_with_langsmith.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 1. Setup + Env-Var Guard

### Prerequisites

This notebook requires:

| Variable | Purpose |
|---|---|
| `OPENAI_API_KEY` | Calls to `gpt-4o-mini` for the traced agent |
| `LANGCHAIN_API_KEY` | Your LangSmith API key |
| `LANGCHAIN_TRACING_V2` | Must be the string `"true"` to enable tracing |
| `LANGCHAIN_PROJECT` *(optional)* | Groups traces under a named project in the UI; defaults to `"default"` |

**LangSmith account (free tier):**
1. Go to **https://smith.langchain.com** and sign up for a free account.
2. Navigate to **Settings → API Keys** and create a key.
3. Copy the key and set it as `LANGCHAIN_API_KEY` in your shell or `.env` file.

The guard cell below checks all required variables and tells you exactly what is missing before any LLM call is made.

> **Gotcha:** As of mid-2025 the recommended tracing env var is `LANGCHAIN_TRACING_V2=true`. Older guides may show `LANGSMITH_TRACING=true` or the deprecated `LANGCHAIN_TRACING=true`. Only `LANGCHAIN_TRACING_V2` activates the v2 trace backend reliably. Always check the current [LangSmith quickstart docs](https://docs.smith.langchain.com/setup) if traces are not appearing.

Install dependencies if needed (run once per environment):

```bash
pip install langsmith langchain langchain-openai openai
```
```

- [ ] **Step 2: Add the guard code cell**

```python
import os
import sys

# ---------------------------------------------------------------------------
# ENV-VAR GUARD — all required variables must be present before we proceed.
# ---------------------------------------------------------------------------
REQUIRED = {
    "OPENAI_API_KEY": (
        "Your OpenAI API key. Get one at https://platform.openai.com/api-keys"
    ),
    "LANGCHAIN_API_KEY": (
        "Your LangSmith API key. Sign up free at https://smith.langchain.com, "
        "then go to Settings → API Keys."
    ),
    "LANGCHAIN_TRACING_V2": (
        'Must be set to the string "true" to enable LangSmith tracing. '
        "Example: export LANGCHAIN_TRACING_V2=true"
    ),
}

missing = {k: v for k, v in REQUIRED.items() if not os.getenv(k)}

if missing:
    print("=" * 70)
    print("MISSING REQUIRED ENVIRONMENT VARIABLES")
    print("=" * 70)
    for var, instructions in missing.items():
        print(f"\n  {var}")
        print(f"    {instructions}")
    print("\nSet the missing variables and re-run this cell before continuing.")
    print("=" * 70)
    # Raise so downstream cells don't silently run without credentials.
    raise EnvironmentError(
        f"Missing env vars: {list(missing.keys())}. See instructions above."
    )

# Optional: show which LangSmith project traces will appear under.
project = os.getenv("LANGCHAIN_PROJECT", "default")
print("All required env vars are set.")
print(f"LangSmith project: '{project}'")
print("Traces will appear at: https://smith.langchain.com")
```

- [ ] **Step 3: Run the guard cell with all env vars set**

Expected output (project name may differ):

```
All required env vars are set.
LangSmith project: 'default'
Traces will appear at: https://smith.langchain.com
```

If any variable is missing the cell raises `EnvironmentError` with instructions. No subsequent cell will run until the guard passes — this is intentional.

- [ ] **Step 4: Add the imports code cell**

```python
from langsmith import traceable, Client
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage

# Default model — cheap and fast for teaching demos.
MODEL = "gpt-4o-mini"

llm = ChatOpenAI(model=MODEL, temperature=0)
ls_client = Client()

print(f"LLM: {MODEL}")
print("LangSmith Client ready.")
```

Expected output:

```
LLM: gpt-4o-mini
LangSmith Client ready.
```

- [ ] **Step 5: Commit**

```bash
git add evals/06_tracing_with_langsmith.ipynb
git commit -m "feat(evals): add setup and env-var guard to notebook 06"
```

---

## Task 5: Section 2 — Trace a single LLM call

**Files:**
- Modify: `evals/06_tracing_with_langsmith.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 2. Trace a Single LLM Call

The simplest trace is one LLM call. Because `LANGCHAIN_TRACING_V2=true` is set, every `ChatOpenAI` invocation is **automatically traced** — no decorator needed. LangSmith captures the prompt messages, the completion, token counts, and latency and sends them to the UI.

We wrap the call in a `@traceable`-decorated function to give the run a named span (`answer_question`) that appears as the **parent** in the trace tree.

After the cell runs, the printed URL opens the run in the LangSmith UI. Click it and look at:
- **Inputs** tab: the exact messages sent to the model.
- **Outputs** tab: the completion text.
- **Metadata** tab: latency (ms) and token breakdown (prompt / completion / total).

### Try it

> **Gotcha:** The `@traceable` decorator and its `run_type` parameter are part of the `langsmith` SDK and have been stable since `langsmith>=0.1`. If you see `TypeError: traceable() got an unexpected keyword argument`, check that your installed version is recent: `pip install --upgrade langsmith`.
```

- [ ] **Step 2: Add the code cell**

```python
@traceable(run_type="chain", name="answer_question")
def answer_question(question: str) -> str:
    """Trace a single LLM call. LANGCHAIN_TRACING_V2 auto-traces ChatOpenAI."""
    response = llm.invoke([HumanMessage(content=question)])
    return response.content


# Run it.
answer = answer_question("What is the capital of France, in one word?")
print(f"Answer: {answer}")

# LangSmith captures the run ID. We can build the UI link from it.
# The run URL format is stable as of mid-2025; check LangSmith docs if the
# link format changes in a future SDK release.
import langsmith as lsm
# Retrieve the last run associated with this project.
runs = list(ls_client.list_runs(project_name=os.getenv("LANGCHAIN_PROJECT", "default"), limit=1))
if runs:
    run = runs[0]
    run_url = f"https://smith.langchain.com/o/{run.session_id}/projects/p/{run.session_id}/r/{run.id}"
    print(f"\nTrace URL (open in browser): https://smith.langchain.com")
    print(f"Run ID: {run.id}")
    print("Go to https://smith.langchain.com and open the most recent run in your project.")
else:
    print("Run submitted to LangSmith. Open https://smith.langchain.com to view it.")
```

- [ ] **Step 3: Verify expected output**

Expected output (model text may vary slightly):

```
Answer: Paris

Run ID: <uuid>
Go to https://smith.langchain.com and open the most recent run in your project.
```

The exact answer text and run ID will vary. What matters structurally: `Answer:` line is non-empty and the run ID line contains a UUID. Open the printed URL to confirm the trace appears in the LangSmith UI under your project.

- [ ] **Step 4: Commit**

```bash
git add evals/06_tracing_with_langsmith.ipynb
git commit -m "feat(evals): add single-LLM-call trace section to notebook 06"
```

---

## Task 6: Section 3 — Trace a multi-step agent

**Files:**
- Modify: `evals/06_tracing_with_langsmith.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 3. Trace a Multi-Step Agent

A single span is useful but the real power of tracing shows up when the agent has multiple steps. We build a small 3-step agent:

1. **`retrieve_fact(topic)`** — returns a canned fact string (no network call; the interesting part is the span, not the retrieval logic).
2. **`synthesize_answer(topic, fact)`** — calls the LLM to turn the raw fact into a complete sentence.
3. **`format_response(raw_answer)`** — applies a light post-processing transform (uppercases the first letter, ensures a trailing period).

Each step is decorated with `@traceable`, so each becomes its own **child span** nested under the parent `run_agent` span. The LangSmith UI will show:

```
run_agent  (parent span)
├── retrieve_fact   inputs: {topic}    outputs: {fact string}
├── synthesize_answer  inputs: {topic, fact}  outputs: {llm response}
│     └── ChatOpenAI  (auto-traced by LangChain)
└── format_response  inputs: {raw_answer}  outputs: {formatted string}
```

Per span you can read: exact inputs/outputs, wall-clock latency, and (for LLM spans) token counts. This is the view that answers *"what happened inside that agent run?"*

### Try it
```

- [ ] **Step 2: Add the agent definition code cell**

```python
# --- Step 1: retrieve a canned fact (no network; real retrieval would use a
#             vector store or search API — the span structure is identical).
@traceable(run_type="retriever", name="retrieve_fact")
def retrieve_fact(topic: str) -> str:
    """Return a pre-recorded fact about `topic`."""
    facts = {
        "photosynthesis": (
            "Photosynthesis converts carbon dioxide and water into glucose "
            "using light energy, releasing oxygen as a byproduct."
        ),
        "black holes": (
            "A black hole is a region of spacetime where gravity is so strong "
            "that nothing, not even light, can escape once it crosses the "
            "event horizon."
        ),
    }
    return facts.get(topic.lower(), f"No cached fact found for '{topic}'.")


# --- Step 2: ask the LLM to synthesise a clean answer from the raw fact.
@traceable(run_type="llm", name="synthesize_answer")
def synthesize_answer(topic: str, fact: str) -> str:
    """Use the LLM to turn a raw fact into a polished one-sentence explanation."""
    prompt = (
        f"Using only the following fact, write a single clear sentence that "
        f"explains '{topic}' to a curious 12-year-old.\n\nFact: {fact}"
    )
    response = llm.invoke([HumanMessage(content=prompt)])
    return response.content.strip()


# --- Step 3: light formatting — ensures sentence casing and a trailing period.
@traceable(run_type="chain", name="format_response")
def format_response(raw_answer: str) -> str:
    """Capitalise the first letter and ensure the sentence ends with a period."""
    answer = raw_answer.strip()
    if answer:
        answer = answer[0].upper() + answer[1:]
    if answer and not answer.endswith((".", "!", "?")):
        answer += "."
    return answer


# --- Parent span that orchestrates all three steps.
@traceable(run_type="chain", name="run_agent")
def run_agent(topic: str) -> str:
    """Run the full 3-step agent and return the formatted answer."""
    fact = retrieve_fact(topic)
    raw_answer = synthesize_answer(topic, fact)
    return format_response(raw_answer)


print("Agent functions defined.")
```

Expected output: `Agent functions defined.`

- [ ] **Step 3: Add the run code cell**

```python
result = run_agent("photosynthesis")
print(f"Agent answer: {result}")
print()
print("Open https://smith.langchain.com and find the 'run_agent' run.")
print("You should see 4 spans: run_agent > retrieve_fact, synthesize_answer, format_response.")
print("Click each span to read its inputs, outputs, latency, and token counts.")
```

- [ ] **Step 4: Verify expected output**

Expected output (model phrasing will vary):

```
Agent answer: Photosynthesis is when plants use sunlight to turn carbon dioxide and water into food, while releasing oxygen for us to breathe.

Open https://smith.langchain.com and find the 'run_agent' run.
You should see 4 spans: run_agent > retrieve_fact, synthesize_answer, format_response.
Click each span to read its inputs, outputs, latency, and token counts.
```

The exact sentence will vary across runs because `temperature=0` on `gpt-4o-mini` is not strictly deterministic. What matters structurally: the answer is a non-empty string ending with a period, and the LangSmith UI shows the nested span tree.

- [ ] **Step 5: Commit**

```bash
git add evals/06_tracing_with_langsmith.ipynb
git commit -m "feat(evals): add multi-step traced agent to notebook 06"
```

---

## Task 7: Section 4 — Debug a broken run

**Files:**
- Modify: `evals/06_tracing_with_langsmith.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 4. Debug a Broken Run

Tracing earns its keep when something goes wrong. We deliberately introduce a bug: `retrieve_fact_broken` returns a `dict` instead of a `str`. The downstream `synthesize_answer` step receives a dict where it expects a string and the LLM prompt becomes malformed.

Without tracing, debugging this means reading a long stack trace and guessing which step produced the bad value. With tracing, the LangSmith UI shows:

- `run_agent_broken` span: ERROR — the run failed.
- `retrieve_fact_broken` span: output is `{"raw": {...}}` (a dict, not a string).
- `synthesize_answer` span: the input `fact` field is the stringified dict, so the LLM produces nonsense — OR the span itself raises `TypeError` if the dict is passed unsanitised.

The key insight: **the trace pinpoints the span where the bad value was first introduced** (`retrieve_fact_broken`), not just where the failure eventually surfaced. That's the debugging value.

### Try it

> **Gotcha:** When a `@traceable`-decorated function raises an exception, LangSmith marks that span ERROR and still sends the partial trace. The parent span is also marked ERROR. You can always read a failed run's spans — you don't need a successful run to get tracing value.
```

- [ ] **Step 2: Add the code cell**

```python
@traceable(run_type="retriever", name="retrieve_fact_broken")
def retrieve_fact_broken(topic: str) -> dict:
    """BUG: returns a dict instead of a str. Downstream steps expect a str."""
    # Simulates a retrieval function that accidentally returns raw metadata
    # instead of the fact string.
    return {
        "topic": topic,
        "raw": {
            "content": "Photosynthesis converts CO2 and water into glucose.",
            "source": "biology_db",
            "confidence": 0.97,
        },
    }


@traceable(run_type="chain", name="run_agent_broken")
def run_agent_broken(topic: str) -> str:
    """Run the broken agent. retrieve_fact_broken returns a dict, not a str."""
    fact = retrieve_fact_broken(topic)  # Bug: fact is a dict here.
    # synthesize_answer expects fact: str — passing a dict produces a
    # malformed prompt and nonsensical LLM output (or a TypeError).
    raw_answer = synthesize_answer(topic, str(fact))  # str(dict) = ugly repr
    return format_response(raw_answer)


print("Running broken agent...")
broken_result = run_agent_broken("photosynthesis")
print(f"Broken agent output: {broken_result!r}")
print()
print("Open https://smith.langchain.com and find the 'run_agent_broken' run.")
print("Inspect the 'retrieve_fact_broken' span — its output is a dict, not a string.")
print("That span is where the bug lives. The LLM received a stringified dict as context,")
print("which explains any garbled output downstream.")
```

- [ ] **Step 3: Verify expected output**

Expected output (model text will vary and may be garbled/nonsensical because the prompt context is malformed):

```
Running broken agent...
Broken agent output: "{'topic': 'photosynthesis', 'raw': {'content': 'Photosynthesis converts CO2 ...}}."

Open https://smith.langchain.com and find the 'run_agent_broken' run.
Inspect the 'retrieve_fact_broken' span — its output is a dict, not a string.
That span is where the bug lives. The LLM received a stringified dict as context,
which explains any garbled output downstream.
```

The broken result will be a non-empty string (the LLM tries to process the dict repr as context). It may be nonsensical. What matters structurally: the cell runs without an unhandled exception (we use `str(fact)` to avoid a TypeError crash so the trace is still submitted), and the LangSmith UI shows the malformed `retrieve_fact_broken` output.

- [ ] **Step 4: Commit**

```bash
git add evals/06_tracing_with_langsmith.ipynb
git commit -m "feat(evals): add broken-run debug trace section to notebook 06"
```

---

## Task 8: Section 5 — Latency, tokens, cost at a glance

**Files:**
- Modify: `evals/06_tracing_with_langsmith.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 5. Latency, Tokens, Cost at a Glance

The LangSmith UI shows per-run latency and token usage visually, but you can also read these programmatically via the `langsmith.Client`. This is useful for building lightweight cost-monitoring scripts or for comparing runs in code (notebook 07 does this at scale inside LangSmith experiments).

The `Client.get_run()` method returns a `Run` object. The fields most useful for cost analysis are:

| Field | What it contains |
|---|---|
| `run.total_tokens` | Total tokens consumed by the run (prompt + completion) |
| `run.prompt_tokens` | Tokens in the input messages |
| `run.completion_tokens` | Tokens in the model's response |
| `run.total_cost` | Estimated USD cost (populated by LangSmith if it knows the model price) |
| `run.end_time - run.start_time` | Wall-clock latency for the run |

> **Gotcha:** `total_cost` is populated by LangSmith on the server side using its internal pricing table. If the field is `None`, either the model price is not in LangSmith's table or the SDK version doesn't expose it yet. Latency and token counts are more reliably populated. Check [LangSmith docs on cost tracking](https://docs.smith.langchain.com/observability/how_to_guides/trace_with_langchain) for current field names.

### Try it
```

- [ ] **Step 2: Add the code cell**

```python
import time

# Run the working agent once more so we have a fresh run to inspect.
_ = run_agent("black holes")

# Give LangSmith a moment to ingest the run before we query it.
# In practice you'd query asynchronously or poll; here a short wait is fine.
time.sleep(3)

# Fetch the most recent run for this project.
project_name = os.getenv("LANGCHAIN_PROJECT", "default")
recent_runs = list(
    ls_client.list_runs(
        project_name=project_name,
        run_type="chain",
        limit=5,
    )
)

# Find the most recent run_agent run.
agent_run = next(
    (r for r in recent_runs if r.name == "run_agent"),
    None,
)

if agent_run is None:
    print("Could not find a 'run_agent' run in recent results.")
    print("This can happen if the run hasn't been ingested yet. Wait a few seconds and re-run.")
else:
    latency_s = (
        (agent_run.end_time - agent_run.start_time).total_seconds()
        if agent_run.end_time and agent_run.start_time
        else None
    )
    print(f"Run name     : {agent_run.name}")
    print(f"Run ID       : {agent_run.id}")
    print(f"Status       : {agent_run.status}")
    print(f"Latency      : {latency_s:.2f}s" if latency_s is not None else "Latency      : (not yet available)")
    print(f"Prompt tokens: {agent_run.prompt_tokens}")
    print(f"Completion   : {agent_run.completion_tokens}")
    print(f"Total tokens : {agent_run.total_tokens}")
    print(f"Total cost   : ${agent_run.total_cost:.6f}" if agent_run.total_cost else "Total cost   : (not populated for this model/version)")
    print()
    print("These numbers let you reason about cost per run before moving to")
    print("large-scale experiments (notebook 07 does this across entire datasets).")
```

- [ ] **Step 3: Verify expected output**

Expected output (numbers will vary per run):

```
Run name     : run_agent
Run ID       : <uuid>
Status       : success
Latency      : 1.43s
Prompt tokens: 87
Completion   : 34
Total tokens : 121
Total cost   : $0.000049

These numbers let you reason about cost per run before moving to
large-scale experiments (notebook 07 does this across entire datasets).
```

Token counts, latency, and cost will vary. `Total cost` may show `(not populated for this model/version)` depending on SDK version — that is acceptable. What matters structurally: `Status` is `success`, token fields are integers greater than 0, and `Latency` is a positive float.

- [ ] **Step 4: Commit**

```bash
git add evals/06_tracing_with_langsmith.ipynb
git commit -m "feat(evals): add latency/tokens/cost section to notebook 06"
```

---

## Task 9: Section 6 — Alternatives, closing recap, and teaser

**Files:**
- Modify: `evals/06_tracing_with_langsmith.ipynb` (add 3 markdown cells)

- [ ] **Step 1: Add the alternatives markdown cell**

```markdown
## 6. One-Line Note on Alternatives

**Langfuse** (https://langfuse.com) and **Arize Phoenix** (https://phoenix.arize.com) are two well-maintained open-source alternatives that support OpenTelemetry-native tracing and can be self-hosted. This series uses LangSmith because it integrates with the LangChain stack we're already using and has a generous free tier — but the tracing *concepts* (spans, parent/child relationships, inputs/outputs per span) transfer directly to any of these tools.
```

- [ ] **Step 2: Add the "What you just learned" markdown cell**

```markdown
## What you just learned

- **Tracing** adds a hierarchical call graph to every agent run: each function decorated with `@traceable` becomes a **span** with captured inputs, outputs, latency, and token counts.
- Setting `LANGCHAIN_TRACING_V2=true` enables **automatic tracing** for all LangChain components (like `ChatOpenAI`) without any code changes.
- A **nested span tree** (parent run → child spans → LLM sub-spans) lets you pinpoint exactly which step in a multi-step agent produced a given output — or a given error.
- When a run breaks, the trace **localises the failure**: you can read the output of each span and identify the first span where the data went wrong, rather than hunting through a flat log.
- The `langsmith.Client` lets you fetch run metadata programmatically: `total_tokens`, `prompt_tokens`, `completion_tokens`, `total_cost`, and wall-clock latency are all available on the `Run` object.
- LangSmith is a hosted service — traces appear in the UI at https://smith.langchain.com, not inside the notebook.
```

- [ ] **Step 3: Add the "What's missing" markdown cell**

```markdown
## What's missing

We can now trace individual runs and inspect them in the UI. But our eval harness from notebook 03 — the `run_eval(agent, dataset, graders)` loop that scores outputs in bulk — still runs entirely offline. Its results live only in a Python list.

**Notebook 07 (`07_datasets_experiments_and_cost.ipynb`)** moves that harness into LangSmith: we'll upload an eval dataset to LangSmith, run a named **experiment** using `Client.evaluate()`, and compare experiments across agent versions in the UI the same way we compared them with diff tables in notebook 04. We'll also see how LangSmith aggregates cost, latency, and token usage across an entire dataset run — giving us the cost dimension we've been missing from our offline evals.
```

- [ ] **Step 4: Verify all three cells render**

Check that the hyperlinks in the alternatives cell and the bold emphasis in the recap render correctly.

- [ ] **Step 5: Commit**

```bash
git add evals/06_tracing_with_langsmith.ipynb
git commit -m "feat(evals): add alternatives, recap, and nb-07 teaser to notebook 06"
```

---

## Task 10: End-to-end verification in a fresh kernel

**Files:** none (verification only).

**Prerequisites:** Before running this task, ensure the following are set in your shell environment:
- `OPENAI_API_KEY` — a valid OpenAI API key.
- `LANGCHAIN_API_KEY` — a valid LangSmith API key (free account at https://smith.langchain.com).
- `LANGCHAIN_TRACING_V2=true` — enables LangSmith tracing.

**Cells that require credentials:** Section 1 (guard cell, imports cell), Section 2 (LLM call), Section 3 (multi-step agent run), Section 4 (broken agent run), Section 5 (LLM call + Client query). If credentials are absent the guard cell will raise `EnvironmentError` before any LLM call is made.

- [ ] **Step 1: Execute the notebook top-to-bottom in a clean kernel**

```bash
jupyter nbconvert --to notebook --execute --inplace evals/06_tracing_with_langsmith.ipynb
```

Expected: succeeds with no errors. The notebook is rewritten with embedded outputs from the fresh run.

- [ ] **Step 2: Verify structural outcomes in the executed notebook**

Check each section for the following structural properties (exact text will vary due to model non-determinism and run UUIDs):

1. **Guard cell**: output contains `"All required env vars are set."` and a project name line.
2. **Imports cell**: output contains `"LLM: gpt-4o-mini"` and `"LangSmith Client ready."`.
3. **Section 2** (single call): `Answer:` line is a non-empty string; a run ID (UUID) is printed or `"Go to https://smith.langchain.com"` appears.
4. **Section 3** (multi-step agent): `"Agent answer:"` line ends with `.` (from `format_response`); the LangSmith instructions line is present.
5. **Section 4** (broken run): `"Running broken agent..."` is present; `"Broken agent output:"` line is a non-empty quoted string; the trace instructions are printed.
6. **Section 5** (latency/tokens): either a `Run name: run_agent` block with integer token counts and a positive latency float, OR the `"Could not find a 'run_agent' run"` fallback message (acceptable if ingestion lag exceeds the wait).
7. No cell raises an unhandled exception. `EnvironmentError` from a missing credential is the only expected exception path, and only if credentials are absent.

The LangSmith UI itself cannot be verified from the notebook — the notebook instructs the learner to open https://smith.langchain.com and confirms a run was submitted by printing the run ID or project URL. This is the correct verification boundary: the notebook's responsibility is to submit traces successfully; rendering them is the hosted UI's responsibility.

- [ ] **Step 3: Verify the run was submitted to LangSmith**

Open https://smith.langchain.com, navigate to your project (default or the value of `LANGCHAIN_PROJECT`), and confirm at least three runs appear: `run_agent` (Section 3), `run_agent_broken` (Section 4), and the `answer_question` run (Section 2). Each should show a span tree in the trace view.

- [ ] **Step 4: Commit the clean run**

```bash
git add evals/06_tracing_with_langsmith.ipynb
git commit -m "chore(evals): commit clean fresh-kernel run of notebook 06"
```

---

## Done

After Task 10 passes, notebook 06 is complete. The next plan to write is `2026-05-24-evals-notebook-07-datasets-experiments-and-cost.md`, which moves the offline eval harness into LangSmith as datasets and experiments, adds cost/latency comparison across runs, and introduces the `Client.evaluate()` API.
