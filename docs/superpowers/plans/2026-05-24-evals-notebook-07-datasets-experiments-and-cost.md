# Evals Notebook 07 — `07_datasets_experiments_and_cost.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the seventh notebook in the Agent Evals & Observability series — move the offline eval harness into **LangSmith** by uploading a dataset, adapting our own graders into LangSmith evaluators via `as_langsmith_evaluator(grader)`, running platform-native experiments with `evaluate(...)`, comparing two agent versions, and reading cost/latency/token data from experiment results.

**Architecture:** A single Jupyter notebook that (a) re-declares the minimal harness (`Score`, `Example`, `exact_match`, `make_llm_judge`) inline for self-containment, (b) guards against missing credentials early, (c) uploads an eval dataset to LangSmith via `client.create_dataset` + `client.create_examples`, (d) adapts our graders to LangSmith's evaluator calling convention via `as_langsmith_evaluator`, (e) runs and compares two experiments with `evaluate(...)`, and (f) reads aggregate latency/token/cost data from experiment results. No local servers. LangSmith is a hosted service reached over HTTPS.

**Tech Stack:** Python 3.11+, Jupyter, `openai` (`gpt-4o-mini` by default), `langsmith` (SDK), `langchain`, `langchain-openai`. Requires `OPENAI_API_KEY` and a free LangSmith account (`LANGCHAIN_API_KEY`, `LANGCHAIN_TRACING_V2=true`).

**Companion spec:** `docs/superpowers/specs/2026-05-24-evals-observability-learning-series-design.md` (notebook 07 section).

**Folder:** The `evals/` folder is assumed to already exist (scaffolded by Task 1 of notebook 01's plan). Task 1 of this plan creates only the notebook file.

---

## File Structure

- **Create:** `evals/07_datasets_experiments_and_cost.ipynb` — the entire notebook, self-contained.
- **Modify:** none.
- **No separate test files** — verification is the notebook running top-to-bottom in a fresh kernel with valid credentials (Task 11).

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown)
2. **"What you'll learn"** (markdown bullets)
3. **Section 1: Setup + credential guard** (markdown + 2 code cells — install/imports, env-var guard)
4. **Section 2: Re-declare the minimal harness** (markdown + 1 code cell — `Score`, `Example`, `exact_match`, `make_llm_judge`)
5. **Section 3: Offline vs online eval** (markdown conceptual cell, no code)
6. **Section 4: Upload a dataset to LangSmith** (markdown + 1 code cell)
7. **Section 5: The evaluator bridge** (markdown + 1 code cell — `as_langsmith_evaluator`, adapts both `exact_match` and an LLM-judge instance)
8. **Section 6: Run an experiment** (markdown + 1 code cell — define target agent, call `evaluate(...)`, print experiment URL, read aggregate scores)
9. **Section 7: Compare experiments across agent versions** (markdown + 1 code cell — second experiment with changed agent, print both URLs)
10. **Section 8: Cost, latency, and tokens** (markdown + 1 code cell — read aggregate data from results)
11. **"What you just learned"** (markdown)
12. **"What's missing"** (markdown teaser → notebook 08)

---

## Task 1: Scaffold the empty notebook

**Files:**
- Create: `evals/07_datasets_experiments_and_cost.ipynb`

- [ ] **Step 1: Verify the `evals/` folder exists**

```bash
ls -la evals/
```

Expected: folder is present. If absent, create it: `mkdir evals`.

- [ ] **Step 2: Create an empty notebook with the Python 3 kernel**

Write the file with this minimal skeleton:

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

Save as `evals/07_datasets_experiments_and_cost.ipynb`.

- [ ] **Step 3: Verify the file is valid JSON**

```bash
python -c "import json; json.load(open('evals/07_datasets_experiments_and_cost.ipynb'))"
ls -la evals/07_datasets_experiments_and_cost.ipynb
```

Expected: no exception; file non-zero size.

- [ ] **Step 4: Commit**

```bash
git add evals/07_datasets_experiments_and_cost.ipynb
git commit -m "feat(evals): scaffold 07_datasets_experiments_and_cost.ipynb"
```

---

## Task 2: Add title and "Why this notebook exists"

**Files:**
- Modify: `evals/07_datasets_experiments_and_cost.ipynb` (add cell 1)

- [ ] **Step 1: Add a markdown cell with title and motivation**

Cell content (markdown):

```markdown
# 07 — Datasets, Experiments, and Cost

## Why this notebook exists

In **notebook 06** we instrumented an agent with LangSmith tracing and learned to read individual run spans in the UI. That gave us *observability* for a single execution. But observability is reactive — you see what happened after the fact. What we want next is *proactive quality control*: run the agent against a curated dataset on demand (or in CI), record every input/output/score pair in the platform, and compare results across agent versions without leaving the browser.

LangSmith calls this **datasets + experiments**. This notebook moves the offline eval harness we built in notebooks 03–05 into that platform workflow: we upload our dataset, adapt our graders into LangSmith evaluators, trigger an experiment run with `evaluate(...)`, and read back aggregate scores, latency, and estimated cost.
```

- [ ] **Step 2: Commit**

```bash
git add evals/07_datasets_experiments_and_cost.ipynb
git commit -m "feat(evals): add intro markdown to notebook 07"
```

---

## Task 3: Add "What you'll learn"

**Files:**
- Modify: `evals/07_datasets_experiments_and_cost.ipynb` (add cell 2)

- [ ] **Step 1: Add a markdown cell**

Cell content (markdown):

```markdown
## What you'll learn

- How to upload an eval dataset to LangSmith using `Client.create_dataset` and `Client.create_examples`.
- How to adapt a grader that follows our `(example, output) -> Score` contract into a LangSmith evaluator using `as_langsmith_evaluator(grader)` — and why the LangSmith evaluator signature varies by SDK version.
- How to run a platform-native experiment with `evaluate(target, data=..., evaluators=[...])` — the LangSmith equivalent of notebook 03's `run_eval`.
- How to compare two experiments side-by-side in the LangSmith UI — the platform equivalent of notebook 04's regression diff.
- Where to find aggregate latency, token counts, and estimated cost in experiment results, and how to reason about the cost of running LLM-judge evaluators at scale.
- The difference between **offline evaluation** (run a fixed dataset on demand or in CI) and **online evaluation** (sample and score live production traffic).
```

- [ ] **Step 2: Commit**

```bash
git add evals/07_datasets_experiments_and_cost.ipynb
git commit -m "feat(evals): add 'What you'll learn' to notebook 07"
```

---

## Task 4: Section 1 — Setup and credential guard

**Files:**
- Modify: `evals/07_datasets_experiments_and_cost.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 1. Setup + Credential Guard

This notebook requires:

1. **`OPENAI_API_KEY`** — used by the target agent (calls `gpt-4o-mini`) and by the LLM-judge evaluator.
2. **A free LangSmith account** — sign up at [smith.langchain.com](https://smith.langchain.com) if you haven't already.
3. **`LANGCHAIN_API_KEY`** — your LangSmith API key. Copy it from *Settings → API Keys* in the LangSmith UI and export it in your shell or add it to a `.env` file:
   ```
   export LANGCHAIN_API_KEY="ls__..."
   export LANGCHAIN_TRACING_V2="true"
   export OPENAI_API_KEY="sk-..."
   ```

The guard cell below checks both variables and prints setup instructions if either is missing. **Do not continue past this cell until both checks pass.**
```

- [ ] **Step 2: Add the pip install cell**

```python
# Install required packages (safe to re-run; pip is idempotent)
%pip install --quiet langsmith langchain langchain-openai openai
```

- [ ] **Step 3: Add the credential guard cell**

```python
import os
import sys

missing: list[str] = []

if not os.environ.get("OPENAI_API_KEY"):
    missing.append("OPENAI_API_KEY")

if not os.environ.get("LANGCHAIN_API_KEY"):
    missing.append("LANGCHAIN_API_KEY")

if missing:
    print("=" * 60)
    print("MISSING CREDENTIALS — notebook will not run correctly.")
    print("=" * 60)
    for var in missing:
        print(f"\n  {var} is not set.")
    print("\nSetup instructions:")
    print("  1. OPENAI_API_KEY:    https://platform.openai.com/api-keys")
    print("  2. LANGCHAIN_API_KEY: https://smith.langchain.com  (free account)")
    print("     In the LangSmith UI: Settings → API Keys → Create API Key")
    print("\nThen set the env vars and restart this kernel:")
    print("  export OPENAI_API_KEY='sk-...'")
    print("  export LANGCHAIN_API_KEY='ls__...'")
    print("  export LANGCHAIN_TRACING_V2='true'")
    raise EnvironmentError(f"Missing required credentials: {', '.join(missing)}")

# LangSmith tracing (optional for experiments, but good hygiene)
os.environ.setdefault("LANGCHAIN_TRACING_V2", "true")

print("Credentials OK.")
print(f"  OPENAI_API_KEY   : ...{os.environ['OPENAI_API_KEY'][-4:]}")
print(f"  LANGCHAIN_API_KEY: ...{os.environ['LANGCHAIN_API_KEY'][-4:]}")
print(f"  LANGCHAIN_TRACING_V2: {os.environ['LANGCHAIN_TRACING_V2']}")
```

Expected output (credentials present):

```
Credentials OK.
  OPENAI_API_KEY   : ...XXXX
  LANGCHAIN_API_KEY: ...XXXX
  LANGCHAIN_TRACING_V2: true
```

- [ ] **Step 4: Commit**

```bash
git add evals/07_datasets_experiments_and_cost.ipynb
git commit -m "feat(evals): add setup and credential guard to notebook 07"
```

---

## Task 5: Section 2 — Re-declare the minimal harness

**Files:**
- Modify: `evals/07_datasets_experiments_and_cost.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 2. Re-declare the Minimal Harness

This notebook is self-contained — it re-declares the four harness pieces it borrows from earlier notebooks so it runs alone without importing from them.

- `Score` and `Example` are the core data types introduced in **notebook 03**.
- `exact_match` is the deterministic grader from **notebook 03**.
- `make_llm_judge(rubric, client, model, key)` is the factory from **notebook 05** that returns a rubric-grading evaluator.

If you have already worked through those notebooks and want to import from them, the signatures are identical — just replace the cells below with your imports.
```

- [ ] **Step 2: Add the harness code cell**

```python
from __future__ import annotations

import re
from dataclasses import dataclass, field
from typing import Any, Callable

from openai import OpenAI


# ---------------------------------------------------------------------------
# Core types (from notebook 03)
# ---------------------------------------------------------------------------

@dataclass
class Score:
    """Result of running one grader on one example output."""
    key: str
    score: float          # 0.0 – 1.0
    passed: bool
    comment: str = ""


@dataclass
class Example:
    """One item in an eval dataset."""
    input: Any
    expected: Any = None
    metadata: dict = field(default_factory=dict)


# ---------------------------------------------------------------------------
# Deterministic grader (from notebook 03)
# ---------------------------------------------------------------------------

def exact_match(example: Example, output: str) -> Score:
    """Pass iff the output string equals the expected string exactly."""
    passed = str(output).strip() == str(example.expected).strip()
    return Score(
        key="exact_match",
        score=1.0 if passed else 0.0,
        passed=passed,
        comment="" if passed else f"Expected {example.expected!r}, got {output!r}",
    )


# ---------------------------------------------------------------------------
# LLM-judge factory (from notebook 05)
# ---------------------------------------------------------------------------

def make_llm_judge(
    rubric: str,
    client: OpenAI,
    model: str = "gpt-4o-mini",
    key: str = "llm_judge",
) -> Callable[[Example, str], Score]:
    """Return a grader that scores an output against *rubric* using an LLM.

    The returned grader follows the same ``(example, output) -> Score``
    contract as all other graders in this series.
    """

    def grader(example: Example, output: str) -> Score:
        system = (
            "You are an impartial evaluator. "
            "Score the assistant output against the rubric. "
            "Reply with JSON only: "
            '{"score": <0.0-1.0>, "passed": <true|false>, "comment": "<reason>"}'
        )
        user = (
            f"Rubric:\n{rubric}\n\n"
            f"Input:\n{example.input}\n\n"
            f"Expected (if available):\n{example.expected}\n\n"
            f"Output to evaluate:\n{output}"
        )
        response = client.chat.completions.create(
            model=model,
            messages=[{"role": "system", "content": system},
                      {"role": "user", "content": user}],
            response_format={"type": "json_object"},
            temperature=0,
        )
        import json
        data = json.loads(response.choices[0].message.content)
        return Score(
            key=key,
            score=float(data.get("score", 0.0)),
            passed=bool(data.get("passed", False)),
            comment=data.get("comment", ""),
        )

    return grader


# ---------------------------------------------------------------------------
# OpenAI client (used by make_llm_judge and the target agents below)
# ---------------------------------------------------------------------------
import os
openai_client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])
DEFAULT_MODEL = "gpt-4o-mini"

print("Harness re-declared: Score, Example, exact_match, make_llm_judge — OK")
```

Expected output:

```
Harness re-declared: Score, Example, exact_match, make_llm_judge — OK
```

- [ ] **Step 3: Commit**

```bash
git add evals/07_datasets_experiments_and_cost.ipynb
git commit -m "feat(evals): re-declare harness primitives in notebook 07"
```

---

## Task 6: Section 3 — Offline vs online eval (conceptual)

**Files:**
- Modify: `evals/07_datasets_experiments_and_cost.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Add a markdown cell — no code cell in this section**

```markdown
## 3. Offline vs Online Eval

Before diving into the LangSmith APIs, one conceptual distinction that will frame everything else:

**Offline evaluation** means running a fixed, curated dataset through your agent on demand — in a notebook like this one, or in CI triggered by a pull request. The dataset is static (or versioned), the inputs are known, and you can compare scores deterministically across agent versions. This is what we built in notebooks 03 and 04, and it is what LangSmith *experiments* are designed for.

**Online evaluation** means sampling real user traffic from your production agent, running graders or LLM judges over those live inputs/outputs, and monitoring quality over time. You can't curate the inputs, so graders need to be more forgiving — and you typically score only a sample rather than every request.

The rest of this notebook covers **offline evaluation** (the foundation). Notebook 08 sketches how the same suite plugs into an online monitoring loop.
```

- [ ] **Step 2: Commit**

```bash
git add evals/07_datasets_experiments_and_cost.ipynb
git commit -m "feat(evals): add offline-vs-online conceptual section to notebook 07"
```

---

## Task 7: Section 4 — Upload a dataset to LangSmith

**Files:**
- Modify: `evals/07_datasets_experiments_and_cost.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 4. Upload a Dataset to LangSmith

A LangSmith *dataset* is a versioned collection of (input, expected output) pairs stored in the platform. We create one with `client.create_dataset(...)` and populate it with `client.create_examples(...)`.

We'll use a small capital-city quiz dataset — five questions with deterministic expected answers — so `exact_match` can give us reliable signal alongside the LLM judge. The same dataset object will be reused by both experiments in sections 6 and 7.

### Try it
```

- [ ] **Step 2: Add the dataset upload code cell**

```python
import uuid
from langsmith import Client

ls_client = Client()

# Unique suffix so re-running the notebook doesn't collide with a previous run
_run_id = uuid.uuid4().hex[:8]
DATASET_NAME = f"capital-cities-quiz-{_run_id}"

# Our eval examples: input prompt → expected answer (exact, trimmed)
_raw_examples = [
    {"input": "What is the capital of France?",           "expected": "Paris"},
    {"input": "What is the capital of Japan?",            "expected": "Tokyo"},
    {"input": "What is the capital of Brazil?",           "expected": "Brasília"},
    {"input": "What is the capital of Australia?",        "expected": "Canberra"},
    {"input": "What is the capital of South Africa?",     "expected": "Pretoria"},
]

# 1. Create the dataset shell
dataset = ls_client.create_dataset(
    dataset_name=DATASET_NAME,
    description="Capital-city quiz for notebook 07 eval experiments.",
)

# 2. Populate it with examples
#    LangSmith examples use `inputs` (dict) and `outputs` (dict).
ls_client.create_examples(
    dataset_id=dataset.id,
    inputs=[{"question": ex["input"]} for ex in _raw_examples],
    outputs=[{"answer": ex["expected"]} for ex in _raw_examples],
)

dataset_url = f"https://smith.langchain.com/datasets/{dataset.id}"
print(f"Dataset '{DATASET_NAME}' created.")
print(f"  ID  : {dataset.id}")
print(f"  URL : {dataset_url}")
print(f"  Size: {len(_raw_examples)} examples")
```

Expected output (IDs will differ):

```
Dataset 'capital-cities-quiz-a1b2c3d4' created.
  ID  : 3f7a...
  URL : https://smith.langchain.com/datasets/3f7a...
  Size: 5 examples
```

- [ ] **Step 3: Commit**

```bash
git add evals/07_datasets_experiments_and_cost.ipynb
git commit -m "feat(evals): add LangSmith dataset upload to notebook 07"
```

---

## Task 8: Section 5 — The evaluator bridge

**Files:**
- Modify: `evals/07_datasets_experiments_and_cost.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 5. The Evaluator Bridge

LangSmith's `evaluate(...)` function expects *evaluators* — callables with a specific signature. The exact signature has shifted across SDK versions; the current stable form receives a dict with keys `inputs`, `outputs`, and (optionally) `reference_outputs`. Older versions passed `(run, example)` objects.

We don't want to rewrite our graders for every SDK version. Instead we write a single adapter, `as_langsmith_evaluator(grader)`, that:

1. Accepts any grader following our `(Example, str) -> Score` contract.
2. Returns a new callable with LangSmith's expected signature.
3. Inside the wrapper, extracts the question + reference answer from the LangSmith dicts, builds a minimal `Example`, calls our grader, and returns the dict LangSmith expects: `{"key": ..., "score": ..., "comment": ...}`.

> **Gotcha:** The LangSmith evaluator signature is not yet fully stable. At the time of writing (SDK ≥ 0.1), the common form is `def evaluator(*, inputs, outputs, reference_outputs, **kwargs) -> dict`. Older notebooks and docs may show `def evaluator(run, example)`. If you upgrade `langsmith` and get a `TypeError` on the evaluator call, check the [LangSmith SDK changelog](https://github.com/langchain-ai/langsmith-sdk/releases) and adjust the wrapper parameter names — the logic inside stays the same.

### Try it
```

- [ ] **Step 2: Add the evaluator bridge code cell**

```python
from typing import Any

def as_langsmith_evaluator(grader: Callable[[Example, str], Score]) -> Callable:
    """Adapt a ``(Example, str) -> Score`` grader into a LangSmith evaluator.

    LangSmith (SDK ≥ 0.1) calls evaluators with keyword arguments:
        ``inputs``           — the example's input dict
        ``outputs``          — the target function's return dict
        ``reference_outputs`` — the dataset's expected output dict (may be None)

    We build a minimal ``Example`` from those dicts, extract the output string,
    call our grader, and return the dict LangSmith expects.

    Args:
        grader: Any grader following the ``(Example, str) -> Score`` contract.

    Returns:
        A callable with the LangSmith evaluator signature.
    """
    def _evaluator(
        *,
        inputs: dict[str, Any],
        outputs: dict[str, Any],
        reference_outputs: dict[str, Any] | None = None,
        **_kwargs: Any,
    ) -> dict[str, Any]:
        # Build a minimal Example from LangSmith's dicts
        question = inputs.get("question", "")
        expected = (reference_outputs or {}).get("answer", None)
        example = Example(input=question, expected=expected)

        # Extract the answer string our target agent returned
        output_str = str(outputs.get("answer", outputs.get("output", "")))

        score: Score = grader(example, output_str)
        return {
            "key": score.key,
            "score": score.score,
            "comment": score.comment,
        }

    # Preserve the grader's name for readability in the LangSmith UI
    _evaluator.__name__ = f"ls_{grader.__name__ if hasattr(grader, '__name__') else 'grader'}"
    return _evaluator


# Adapt both graders we'll use in the experiments
ls_exact_match = as_langsmith_evaluator(exact_match)

_quality_rubric = (
    "The answer must name the correct capital city. "
    "Small spelling variants or 'the capital is X' phrasing are acceptable. "
    "Score 1.0 if correct, 0.0 if wrong."
)
_llm_judge_grader = make_llm_judge(
    rubric=_quality_rubric,
    client=openai_client,
    model=DEFAULT_MODEL,
    key="llm_judge_quality",
)
ls_llm_judge = as_langsmith_evaluator(_llm_judge_grader)

print("Evaluator bridge defined: as_langsmith_evaluator — OK")
print(f"  ls_exact_match : {ls_exact_match.__name__}")
print(f"  ls_llm_judge   : {ls_llm_judge.__name__}")
```

Expected output:

```
Evaluator bridge defined: as_langsmith_evaluator — OK
  ls_exact_match : ls_exact_match
  ls_llm_judge   : ls_grader
```

- [ ] **Step 3: Commit**

```bash
git add evals/07_datasets_experiments_and_cost.ipynb
git commit -m "feat(evals): define as_langsmith_evaluator bridge in notebook 07"
```

---

## Task 9: Section 6 — Run an experiment

**Files:**
- Modify: `evals/07_datasets_experiments_and_cost.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 6. Run an Experiment

An *experiment* in LangSmith is one run of a target function over the full dataset, with evaluators scoring each output. The platform records every input, output, and score, and gives you an experiment URL where you can inspect individual rows.

We use `langsmith.evaluate(target, data=..., evaluators=[...])` — this is the platform-native replacement for notebook 03's hand-rolled `run_eval`. Internally, LangSmith calls our `target` function once per dataset example, collects the outputs, calls each evaluator, and stores everything.

Our target is a small agent that calls `gpt-4o-mini` to answer geography questions. We define it as a plain synchronous function: it receives the LangSmith `inputs` dict and must return a dict.

### Try it
```

- [ ] **Step 2: Add the experiment code cell**

```python
from langsmith import evaluate as ls_evaluate


def capital_city_agent_v1(inputs: dict) -> dict:
    """Agent v1: straightforward prompt, returns the city name only."""
    question = inputs["question"]
    response = openai_client.chat.completions.create(
        model=DEFAULT_MODEL,
        messages=[
            {
                "role": "system",
                "content": (
                    "You are a geography expert. "
                    "Answer with the capital city name only — no extra words."
                ),
            },
            {"role": "user", "content": question},
        ],
        temperature=0,
    )
    answer = response.choices[0].message.content.strip()
    return {"answer": answer}


print("Running experiment (v1 agent) — this may take 15–30 s …")
results_v1 = ls_evaluate(
    capital_city_agent_v1,
    data=DATASET_NAME,
    evaluators=[ls_exact_match, ls_llm_judge],
    experiment_prefix="capital-city-v1",
    metadata={"agent_version": "v1"},
)

# The results object exposes the experiment URL and aggregate scores
experiment_url_v1 = results_v1.experiment_name  # or results_v1.url depending on SDK version
print(f"\nExperiment complete.")
print(f"  Experiment name : {results_v1.experiment_name}")

# Print aggregate scores
print("\nAggregate scores:")
try:
    summary = results_v1.to_pandas()[["exact_match", "llm_judge_quality"]].mean()
    print(summary.to_string())
except Exception:
    # Fallback: iterate rows
    rows = list(results_v1)
    em_scores = [r.get("feedback", {}).get("exact_match", None) for r in rows]
    valid = [s for s in em_scores if s is not None]
    if valid:
        print(f"  exact_match mean    : {sum(valid)/len(valid):.2f}")

print(f"\nView in LangSmith UI:")
print(f"  https://smith.langchain.com  → Experiments → capital-city-v1-*")
```

Expected output (scores are model-dependent — numbers will vary):

```
Running experiment (v1 agent) — this may take 15–30 s …

Experiment complete.
  Experiment name : capital-city-v1-20260524T...

Aggregate scores:
  exact_match          0.80
  llm_judge_quality    1.00

View in LangSmith UI:
  https://smith.langchain.com  → Experiments → capital-city-v1-*
```

> **Note:** `exact_match` may score below 1.0 even for a correct model because accents (e.g., "Brasília" vs "Brasilia") fail exact string comparison. The LLM judge should score 1.0 for semantically correct answers. This is intentional — it illustrates why you often want *both* grader types.

- [ ] **Step 3: Commit**

```bash
git add evals/07_datasets_experiments_and_cost.ipynb
git commit -m "feat(evals): add experiment run to notebook 07"
```

---

## Task 10: Section 7 — Compare experiments across agent versions

**Files:**
- Modify: `evals/07_datasets_experiments_and_cost.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 7. Compare Experiments Across Agent Versions

In **notebook 04** we diffed two agent versions by comparing score tables in Python. LangSmith gives us the platform-native version of that workflow: run a second experiment against the same dataset and open both experiments in the UI side-by-side.

We'll introduce a deliberately degraded agent (v2) that adds verbose padding to its answers — causing `exact_match` to drop — then compare the two experiment URLs.

The LangSmith *Experiments* tab lets you select two (or more) experiments on the same dataset and view a per-row diff: which examples regressed, which improved, and by how much. This is the workflow you'd use in a pull-request review.

### Try it
```

- [ ] **Step 2: Add the comparison code cell**

```python
def capital_city_agent_v2(inputs: dict) -> dict:
    """Agent v2 (deliberately degraded): adds verbose preamble to every answer.

    This causes exact_match to fail even when the city name is correct,
    simulating a prompt-change regression.
    """
    question = inputs["question"]
    response = openai_client.chat.completions.create(
        model=DEFAULT_MODEL,
        messages=[
            {
                "role": "system",
                "content": (
                    "You are a helpful geography tutor. "
                    "Always begin your answer with 'The capital city is' "
                    "followed by the city name."
                ),
            },
            {"role": "user", "content": question},
        ],
        temperature=0,
    )
    answer = response.choices[0].message.content.strip()
    return {"answer": answer}


print("Running experiment (v2 agent — degraded) — this may take 15–30 s …")
results_v2 = ls_evaluate(
    capital_city_agent_v2,
    data=DATASET_NAME,
    evaluators=[ls_exact_match, ls_llm_judge],
    experiment_prefix="capital-city-v2-degraded",
    metadata={"agent_version": "v2", "note": "verbose preamble regression"},
)

print(f"\nBoth experiments complete.")
print(f"  v1 experiment : {results_v1.experiment_name}")
print(f"  v2 experiment : {results_v2.experiment_name}")

# Print side-by-side aggregate comparison
print("\nAggregate score comparison:")
print(f"  {'Metric':<25} {'v1':>8} {'v2':>8}")
print(f"  {'-'*25} {'-'*8} {'-'*8}")

def _mean_score(results, key: str) -> str:
    try:
        col = results.to_pandas()[key]
        return f"{col.mean():.2f}"
    except Exception:
        return "n/a"

for metric in ["exact_match", "llm_judge_quality"]:
    v1_score = _mean_score(results_v1, metric)
    v2_score = _mean_score(results_v2, metric)
    print(f"  {metric:<25} {v1_score:>8} {v2_score:>8}")

print("\nTo compare in LangSmith UI:")
print("  1. Go to smith.langchain.com → Datasets → capital-cities-quiz-*")
print("  2. Click 'Compare Experiments' and select both runs.")
print("  3. Inspect the per-row diff: rows where exact_match dropped are the regression.")
```

Expected output (scores model-dependent):

```
Running experiment (v2 agent — degraded) — this may take 15–30 s …

Both experiments complete.
  v1 experiment : capital-city-v1-20260524T...
  v2 experiment : capital-city-v2-degraded-20260524T...

Aggregate score comparison:
  Metric                       v1       v2
  ------------------------- -------- --------
  exact_match                 0.80     0.00
  llm_judge_quality           1.00     0.80

To compare in LangSmith UI:
  1. Go to smith.langchain.com → Datasets → capital-cities-quiz-*
  2. Click 'Compare Experiments' and select both runs.
  3. Inspect the per-row diff: rows where exact_match dropped are the regression.
```

- [ ] **Step 3: Commit**

```bash
git add evals/07_datasets_experiments_and_cost.ipynb
git commit -m "feat(evals): add cross-version experiment comparison to notebook 07"
```

---

## Task 11: Section 8 — Cost, latency, and tokens

**Files:**
- Modify: `evals/07_datasets_experiments_and_cost.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 8. Cost, Latency, and Tokens

LangSmith captures latency and token counts for every traced run. Where the SDK surfaces aggregate cost/latency data on the experiment results object, we can read them directly. For richer breakdowns, the LangSmith UI *Experiments* tab shows per-row and aggregate latency, and the *Trace* view shows token counts per span.

Two things worth internalizing before you run eval at scale:

1. **The cost of the eval itself.** Each example in the dataset triggers one target-agent call *plus* one LLM-judge call. On a 500-example dataset with `gpt-4o-mini`, that is 500 agent calls + 500 judge calls. The judge calls can be as expensive as the agent calls. Prefer deterministic graders where they are sufficient; reserve LLM judges for the dimensions that actually require natural-language reasoning.

2. **Latency budgets for CI.** A 100-example offline suite with a judge takes roughly 2–3 minutes on `gpt-4o-mini`. Plan CI timeouts accordingly, or run the full suite only on main-branch merges and a smaller smoke subset on every PR.

> **Gotcha:** The `results.to_pandas()` latency column (if present) reflects wall-clock time including network round-trips to LangSmith. Token counts come from the OpenAI response objects captured in traces; if tracing is off, they may not be available in the results object directly — use the LangSmith UI trace view instead.

### Try it
```

- [ ] **Step 2: Add the cost/latency code cell**

```python
print("=== Cost, Latency, and Token Summary (v1 experiment) ===\n")

try:
    df_v1 = results_v1.to_pandas()
    print("Columns available in results DataFrame:")
    print(" ", list(df_v1.columns))
    print()

    # Latency — column name varies by SDK version; try common names
    for lat_col in ["latency", "execution_time", "total_latency"]:
        if lat_col in df_v1.columns:
            print(f"Latency ({lat_col}):")
            print(f"  mean  : {df_v1[lat_col].mean():.2f} s")
            print(f"  median: {df_v1[lat_col].median():.2f} s")
            print(f"  max   : {df_v1[lat_col].max():.2f} s")
            break
    else:
        print("Latency: not available in results DataFrame — see LangSmith UI trace view.")

    # Token counts — may be nested under 'prompt_tokens' / 'completion_tokens'
    for tok_col in ["total_tokens", "prompt_tokens", "completion_tokens"]:
        if tok_col in df_v1.columns:
            print(f"\n{tok_col}: {df_v1[tok_col].sum()} total across {len(df_v1)} examples")

except Exception as exc:
    print(f"Note: to_pandas() raised {type(exc).__name__}: {exc}")
    print("This is normal if the SDK version does not support DataFrame export.")
    print("Use the LangSmith UI: Experiments → select run → 'Aggregate' tab.")

print("\n--- Cost reasoning (manual estimate) ---")
_n_examples = len(_raw_examples)
# gpt-4o-mini pricing as of 2026: ~$0.15/1M input, $0.60/1M output (illustrative)
_est_tokens_per_call = 150   # rough estimate per agent call
_est_judge_tokens    = 200   # rough estimate per judge call
_agent_cost  = _n_examples * _est_tokens_per_call * 0.15 / 1_000_000
_judge_cost  = _n_examples * _est_judge_tokens    * 0.15 / 1_000_000
print(f"  Examples in dataset      : {_n_examples}")
print(f"  Estimated agent cost     : ${_agent_cost:.6f}  ({_est_tokens_per_call} tok/call × {_n_examples})")
print(f"  Estimated judge cost     : ${_judge_cost:.6f}  ({_est_judge_tokens} tok/call × {_n_examples})")
print(f"  Combined (illustrative)  : ${_agent_cost + _judge_cost:.6f}")
print()
print("At 500 examples the judge cost alone would be ~100× larger.")
print("Use deterministic graders (exact_match) where they are sufficient.")
```

Expected output (varies by SDK version and model pricing):

```
=== Cost, Latency, and Token Summary (v1 experiment) ===

Columns available in results DataFrame:
  ['input', 'output', 'reference', 'exact_match', 'llm_judge_quality', ...]

Latency: not available in results DataFrame — see LangSmith UI trace view.

--- Cost reasoning (manual estimate) ---
  Examples in dataset      : 5
  Estimated agent cost     : $0.000113  (150 tok/call × 5)
  Estimated judge cost     : $0.000150  (200 tok/call × 5)
  Combined (illustrative)  : $0.000263

At 500 examples the judge cost alone would be ~100× larger.
Use deterministic graders (exact_match) where they are sufficient.
```

- [ ] **Step 3: Commit**

```bash
git add evals/07_datasets_experiments_and_cost.ipynb
git commit -m "feat(evals): add cost and latency section to notebook 07"
```

---

## Task 12: Closing recap and "What's missing" teaser

**Files:**
- Modify: `evals/07_datasets_experiments_and_cost.ipynb` (add 2 markdown cells)

- [ ] **Step 1: Add the "What you just learned" markdown cell**

```markdown
## What you just learned

- **LangSmith datasets** are versioned collections of (input, expected output) pairs created with `client.create_dataset(...)` and `client.create_examples(...)`. They are the stable ground truth your offline experiments run against.
- **`as_langsmith_evaluator(grader)`** adapts any grader following our `(Example, str) -> Score` contract into a LangSmith evaluator. The adapter handles the impedance mismatch between LangSmith's `(inputs, outputs, reference_outputs)` dict signature and our typed dataclass interface — and is easy to update if the SDK signature changes.
- **`evaluate(target, data=..., evaluators=[...])`** is the platform-native version of notebook 03's hand-rolled `run_eval`. It runs the target function over every dataset example, calls each evaluator, stores results, and returns a results object with an experiment URL.
- **Comparing experiments** in the LangSmith UI replicates notebook 04's regression diff visually: select two experiments on the same dataset and inspect per-row score changes.
- **Offline eval** (fixed dataset, run on demand or in CI) is the foundation. **Online eval** (sample live traffic and score it) uses the same graders but on non-curated inputs.
- **LLM-judge calls cost money per example.** Use deterministic graders where they are sufficient and reserve the judge for dimensions that require natural-language reasoning.
```

- [ ] **Step 2: Add the "What's missing" markdown cell**

```markdown
## What's missing

We now have every individual component: deterministic graders (02–03), a regression gate (04), an LLM judge (05), tracing (06), and platform datasets + experiments (07). But they still live in separate notebooks. In production you want a *single runnable suite* that wires all of them together into a gate that passes on a good agent and fails on a regressed one — and that you can invoke from a CI pipeline.

That is exactly what **notebook 08 — `08_production_eval_gate_end_to_end.ipynb`** builds: a realistic research-and-summarize agent evaluated by a full suite (deterministic graders + regression gate + LLM judge + LangSmith experiment), expressed as a CI-style gate in one runnable cell, with a sketch of the equivalent `pytest` invocation.
```

- [ ] **Step 3: Commit**

```bash
git add evals/07_datasets_experiments_and_cost.ipynb
git commit -m "feat(evals): add closing recap and notebook-08 teaser to notebook 07"
```

---

## Task 13: End-to-end verification in a fresh kernel

**Files:** none (verification only).

**Prerequisites:** `OPENAI_API_KEY` and `LANGCHAIN_API_KEY` must be set in the environment where `jupyter nbconvert` runs. The notebook will raise `EnvironmentError` in the guard cell and abort if either is missing — this is intentional. Do not skip or mock the credential check.

- [ ] **Step 1: Execute the notebook top-to-bottom in a clean kernel**

```bash
jupyter nbconvert --to notebook --execute --inplace evals/07_datasets_experiments_and_cost.ipynb
```

Expected: exits 0 with no unhandled exceptions. The notebook is rewritten with embedded outputs from the fresh run. If you see `EnvironmentError: Missing required credentials`, set the env vars and re-run.

- [ ] **Step 2: Verify structural outcomes in the executed notebook**

Open the executed notebook and confirm the following structural outcomes. Exact scores and token numbers are model-dependent and are NOT asserted.

1. **Credential guard** cell printed `Credentials OK.` — did not raise.
2. **Harness cell** printed `Harness re-declared: Score, Example, exact_match, make_llm_judge — OK`.
3. **Dataset upload** cell printed a line matching `Dataset 'capital-cities-quiz-*' created.` and a URL containing `smith.langchain.com/datasets/`.
4. **Evaluator bridge** cell printed `Evaluator bridge defined: as_langsmith_evaluator — OK`.
5. **Experiment v1** cell printed a non-empty experiment name (e.g., `capital-city-v1-...`) and an aggregate scores table with at least the `exact_match` row.
6. **Experiment v2** cell printed both experiment names and a two-column score table showing a lower `exact_match` for v2 than v1 (the regression is detectable).
7. **Cost/latency** cell printed the manual cost estimate section without raising an exception.

- [ ] **Step 3: Spot-check LangSmith UI**

Navigate to `smith.langchain.com`, go to Datasets, and confirm the `capital-cities-quiz-*` dataset exists with 5 examples. Go to Experiments and confirm two experiments (`capital-city-v1-*` and `capital-city-v2-degraded-*`) are listed.

- [ ] **Step 4: Commit the clean run**

```bash
git add evals/07_datasets_experiments_and_cost.ipynb
git commit -m "chore(evals): commit clean fresh-kernel run of notebook 07"
```

---

## Done

After Task 13 passes, notebook 07 is complete. The next plan to write is the implementation plan for notebook 08 — `08_production_eval_gate_end_to_end.ipynb` — which assembles all components (deterministic graders + regression gate + LLM judge + LangSmith experiment) into a single CI-style quality gate for a realistic research-and-summarize agent.
