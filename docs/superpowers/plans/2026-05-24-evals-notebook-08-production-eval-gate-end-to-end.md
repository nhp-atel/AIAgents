# Evals Notebook 08 — `08_production_eval_gate_end_to_end.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the capstone notebook of the evals & observability series — assemble the full toolkit (deterministic graders, regression gate, LLM judge, LangSmith experiment) into a CI-style production eval gate for a realistic research-and-summarize agent. The learner sees exactly how every piece taught in notebooks 01–07 snaps together into a single runnable gate that PASS/FAILs on agent quality, and ends with pointers to the `langgraph/`, `mcp/`, and `a2a/` agents they can now evaluate with this toolkit.

**Architecture:** A single self-contained Jupyter notebook. All prior abstractions (`Score`, `Example`, `ExampleResult`, `EvalReport`, `run_eval`, deterministic graders, regression gate helpers, LLM judge, LangSmith adapter) are **re-declared inline** in the notebook so it runs alone with credentials. The agent under test (`research_agent` and a deliberately-regressed `research_agent_v2`) is defined in the notebook with a canned corpus — no live web access needed. LangSmith experiment logging uses `langsmith.evaluate` with the `as_langsmith_evaluator` adapter declared inline.

**Tech Stack:** Python 3.11+, Jupyter, `openai`, `langsmith`, `langchain`, `langchain-openai`. Default LLM model: `"gpt-4o-mini"`. Requires `OPENAI_API_KEY` and a free LangSmith account with `LANGCHAIN_API_KEY` set (and optionally `LANGCHAIN_TRACING_V2=true`).

**Companion spec:** `docs/superpowers/specs/2026-05-24-evals-observability-learning-series-design.md` (notebook 08 section and full arc).

**Folder:** `evals/` already exists. Task 1 scaffolds the empty notebook inside it.

---

## File Structure

- **Create:** `evals/08_production_eval_gate_end_to_end.ipynb` — the entire notebook, self-contained.
- **Modify:** none.
- **No separate test files** — verification is the notebook running top-to-bottom in a fresh kernel (Task 11).

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown — recap arc 01→07)
2. **"What you'll learn"** (markdown bullets)
3. **Section 1: Setup + guard** (markdown + code cells — imports; OpenAI client; LangSmith client; env-var guard cell; re-declared toolkit in logical groups)
4. **Section 2: A realistic agent** (markdown + code — `research_agent` and `research_agent_v2`)
5. **Section 3: The full offline suite** (markdown + code — dataset assembly, mixed graders, `run_eval`, `summary_table()`)
6. **Section 4: The regression gate** (markdown + code — `run_eval` on both agents, `regression_gate`, `compare_reports`, PASS/FAIL banner)
7. **Section 5: CI-style gate** (markdown + code — `run_gate()` returning 0/1, illustrative `pytest` snippet, prose CI note)
8. **Section 6: Into the platform + online monitoring** (markdown + code — LangSmith `evaluate`, printed experiment URL, conceptual online-monitoring cell)
9. **"What you just learned"** (markdown — full recap from nb01 `assert ==` to production gate)
10. **"Where to go next"** (markdown — pointers to `langgraph/`, `mcp/`, `a2a/`)

*(No "What's missing" — this closes the series.)*

---

## Task 1: Scaffold the empty notebook

**Files:**
- Create: `evals/08_production_eval_gate_end_to_end.ipynb`

- [ ] **Step 1: Verify the `evals/` folder exists**

```bash
ls -la evals/
```

Expected: the folder exists and contains `01_` through `07_` notebooks.

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

Save as `evals/08_production_eval_gate_end_to_end.ipynb`.

- [ ] **Step 3: Verify the file is valid JSON**

```bash
python -c "import json; json.load(open('evals/08_production_eval_gate_end_to_end.ipynb'))"
ls -la evals/08_production_eval_gate_end_to_end.ipynb
```

Expected: no exception, file non-zero size.

- [ ] **Step 4: Commit**

```bash
git add evals/08_production_eval_gate_end_to_end.ipynb
git commit -m "feat(evals): scaffold 08_production_eval_gate_end_to_end.ipynb"
```

---

## Task 2: Add title, "Why this notebook exists", and "What you'll learn"

**Files:**
- Modify: `evals/08_production_eval_gate_end_to_end.ipynb` (add cells 1–2)

- [ ] **Step 1: Add the title + motivation markdown cell**

Cell content (markdown):

```markdown
# 08 — Production Eval Gate, End to End

## Why this notebook exists

Here is the climb we've made:

- **Notebook 01** showed why `assert agent(x) == "expected"` flakes — agents are non-deterministic, have no single ground truth, and fail in multi-step ways.
- **Notebook 02** gave us the cheapest reliable tools: `exact_match`, `make_contains`, `make_schema_grader` — deterministic graders that score outputs without touching an LLM.
- **Notebook 03** turned those graders into a reusable harness: `Score`, `Example`, `EvalReport`, and `run_eval(agent, dataset, graders) → EvalReport`.
- **Notebook 04** introduced the regression gate: run the harness on two agent versions, diff the reports with `compare_reports`, and let `regression_gate` return `True`/`False` — fully offline.
- **Notebook 05** plugged in `make_llm_judge` for open-ended outputs that deterministic graders can't reach, and looked hard at judge bias.
- **Notebook 06** instrumented an agent with LangSmith tracing, read spans in the UI, and debugged a broken run from its trace.
- **Notebook 07** moved the eval harness into LangSmith via `evaluate` and `as_langsmith_evaluator`, tracked cost and latency per experiment, and contrasted offline vs. online evaluation.

**This notebook assembles all of it.** We build a realistic research-and-summarize agent, run a full offline suite mixing deterministic graders and an LLM judge, wire the regression gate into a single `run_gate()` cell, and push the suite into LangSmith as an experiment — all in one notebook that stands alone.
```

- [ ] **Step 2: Add the "What you'll learn" markdown cell**

Cell content (markdown):

```markdown
## What you'll learn

- How to re-use and compose the full eval toolkit inline (types, harness, graders, gate, LLM judge, LangSmith adapter) in a single self-contained notebook.
- How to build a small but realistic **research-and-summarize agent** (`research_agent`) with a tool step (canned search results) and an LLM synthesis step, entirely self-contained.
- How to assemble a **full offline eval suite** mixing `make_contains` / `make_schema_grader` deterministic graders with an `make_llm_judge` faithfulness rubric.
- How to detect a **regression** using `regression_gate` + `compare_reports`, and print a clear PASS / FAIL banner.
- How to express the entire suite as a **CI-style gate**: a `run_gate()` cell returning `0` or `1`, and an illustrative `pytest` / CI sketch.
- How to log the suite as a **LangSmith experiment** via `evaluate` + `as_langsmith_evaluator` and print the experiment URL.
- What **online monitoring** looks like conceptually: sampling live traffic and scoring it with the same graders.
```

- [ ] **Step 3: Commit**

```bash
git add evals/08_production_eval_gate_end_to_end.ipynb
git commit -m "feat(evals): add title, arc recap, and What You'll Learn to notebook 08"
```

---

## Task 3: Section 1 — Setup + guard + re-declared toolkit

**Files:**
- Modify: `evals/08_production_eval_gate_end_to_end.ipynb` (add 1 markdown cell + 4–5 code cells)

This is the largest setup section. Split into logical groups: (a) imports + clients, (b) env-var guard, (c) types + harness, (d) deterministic graders, (e) regression tools, (f) LLM judge, (g) LangSmith adapter.

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 1. Setup + Guard

This notebook requires two credentials:

- `OPENAI_API_KEY` — for the LLM synthesis step inside the agent and for the LLM judge.
- `LANGCHAIN_API_KEY` — for the LangSmith experiment in Section 6. Get a free key at <https://smith.langchain.com>. Also set `LANGCHAIN_TRACING_V2=true` to enable automatic tracing.

The guard cell below checks both. Sections that need LangSmith are clearly marked; you can run Sections 1–5 with only `OPENAI_API_KEY`.

After the guard, we re-declare the full eval toolkit inline — types, harness, graders, regression tools, LLM judge, and LangSmith adapter — so this notebook is self-contained and does not import from earlier notebooks.
```

- [ ] **Step 2: Add the imports + clients code cell**

```python
import json
import os
import textwrap
from dataclasses import dataclass, field
from typing import Any, Callable

from openai import OpenAI

# OpenAI client (used by the agent's synthesis step and by make_llm_judge)
openai_client = OpenAI()  # reads OPENAI_API_KEY from env

print("Imports OK")
```

- [ ] **Step 3: Add the env-var guard code cell**

```python
_missing: list[str] = []
_openai_ok = bool(os.environ.get("OPENAI_API_KEY"))
_langsmith_ok = bool(os.environ.get("LANGCHAIN_API_KEY"))

if not _openai_ok:
    _missing.append("OPENAI_API_KEY")
if not _langsmith_ok:
    _missing.append("LANGCHAIN_API_KEY")

if _missing:
    print("=" * 60)
    print("MISSING CREDENTIALS — some sections will be skipped")
    print("=" * 60)
    for var in _missing:
        print(f"  • {var} is not set")
    print()
    print("Setup instructions:")
    print("  export OPENAI_API_KEY=sk-...")
    print("  export LANGCHAIN_API_KEY=ls__...")
    print("  export LANGCHAIN_TRACING_V2=true")
    print()
    print("Sections 1–5 run with OPENAI_API_KEY only.")
    print("Section 6 (LangSmith experiment) requires LANGCHAIN_API_KEY.")
else:
    print("✓ OPENAI_API_KEY set")
    print("✓ LANGCHAIN_API_KEY set")
    print("All sections will run.")
```

- [ ] **Step 4: Add the types + harness re-declaration code cell**

```python
# ── Types + harness (re-declared from notebooks 02–03) ─────────────────────

@dataclass
class Score:
    key: str
    score: float          # 0.0–1.0
    passed: bool
    comment: str = ""


@dataclass
class Example:
    input: dict[str, Any]
    expected: Any = None
    metadata: dict[str, Any] = field(default_factory=dict)


@dataclass
class ExampleResult:
    """The output and all scores for one example."""
    example: Example
    output: Any
    scores: list[Score]

    def mean_score(self) -> float:
        """Average score across all graders for this example."""
        if not self.scores:
            return 0.0
        return sum(s.score for s in self.scores) / len(self.scores)


@dataclass
class EvalReport:
    results: list[ExampleResult]

    def pass_rate(self, key: str | None = None) -> float:
        scores = [
            s for r in self.results for s in r.scores
            if key is None or s.key == key
        ]
        if not scores:
            return 0.0
        return sum(1 for s in scores if s.passed) / len(scores)

    def mean_score(self, key: str | None = None) -> float:
        scores = [
            s for r in self.results for s in r.scores
            if key is None or s.key == key
        ]
        if not scores:
            return 0.0
        return sum(s.score for s in scores) / len(scores)

    def summary_table(self) -> str:
        keys = sorted({s.key for r in self.results for s in r.scores})
        header = f"{'grader':<25}  {'pass_rate':>9}  {'mean_score':>10}"
        sep = "-" * len(header)
        rows = [header, sep]
        for k in keys:
            rows.append(
                f"{k:<25}  {self.pass_rate(k):>9.2%}  {self.mean_score(k):>10.3f}"
            )
        rows.append(sep)
        rows.append(
            f"{'OVERALL':<25}  {self.pass_rate():>9.2%}  {self.mean_score():>10.3f}"
        )
        return "\n".join(rows)


def run_eval(
    agent: Callable[[dict], Any],
    dataset: list[Example],
    graders: list[Callable[[Example, Any], Score]],
) -> EvalReport:
    results: list[ExampleResult] = []
    for ex in dataset:
        output = agent(ex.input)
        scores = [g(ex, output) for g in graders]
        results.append(ExampleResult(example=ex, output=output, scores=scores))
    return EvalReport(results=results)


print("Types + harness declared OK")
```

- [ ] **Step 5: Add the deterministic graders re-declaration code cell**

```python
# ── Deterministic graders (re-declared from notebook 02) ───────────────────

def exact_match(example: Example, output: Any) -> Score:
    passed = str(output) == str(example.expected)
    return Score(
        key="exact_match",
        score=1.0 if passed else 0.0,
        passed=passed,
        comment=f"expected {example.expected!r}, got {output!r}",
    )


def make_contains(substring: str, key: str | None = None) -> Callable[[Example, Any], Score]:
    _key = key or f"contains:{substring[:20]}"

    def grader(example: Example, output: Any) -> Score:
        text = output.get("summary", "") if isinstance(output, dict) else str(output)
        passed = substring.lower() in text.lower()
        return Score(
            key=_key,
            score=1.0 if passed else 0.0,
            passed=passed,
            comment=f"{'found' if passed else 'missing'}: {substring!r}",
        )

    return grader


def make_schema_grader(
    required_keys: list[str],
    key: str = "schema",
) -> Callable[[Example, Any], Score]:
    def grader(example: Example, output: Any) -> Score:
        if not isinstance(output, dict):
            return Score(key=key, score=0.0, passed=False, comment="output is not a dict")
        missing = [k for k in required_keys if k not in output]
        passed = len(missing) == 0
        return Score(
            key=key,
            score=1.0 if passed else 0.0,
            passed=passed,
            comment=f"missing keys: {missing}" if missing else "all required keys present",
        )

    return grader


print("Deterministic graders declared OK")
```

- [ ] **Step 6: Add the regression tools re-declaration code cell**

```python
# ── Regression tools (re-declared from notebook 04) ────────────────────────

def compare_reports(
    baseline: EvalReport,
    candidate: EvalReport,
) -> list[dict]:
    """Return a per-example delta table between baseline and candidate.

    Each entry:
        {
            "input": the example's input value,
            "baseline_score": mean score (across all graders) for baseline,
            "candidate_score": mean score (across all graders) for candidate,
            "delta": candidate_score - baseline_score,
            "regressed": True iff delta < 0,
        }

    Pairs examples by position — baseline and candidate must have been run
    on the same dataset in the same order (same length required).
    """
    if len(baseline.results) != len(candidate.results):
        raise ValueError(
            f"Reports have different lengths: "
            f"{len(baseline.results)} vs {len(candidate.results)}. "
            "Both must be run on the same dataset."
        )
    rows: list[dict] = []
    for b_result, c_result in zip(baseline.results, candidate.results):
        b_score = b_result.mean_score()
        c_score = c_result.mean_score()
        delta = c_score - b_score
        rows.append(
            {
                "input": b_result.example.input,
                "baseline_score": b_score,
                "candidate_score": c_score,
                "delta": delta,
                "regressed": delta < 0,
            }
        )
    return rows


_EPSILON = 1e-9


def regression_gate(
    baseline: EvalReport,
    candidate: EvalReport,
    min_mean: float = 0.0,
    allow_regressions: bool = False,
) -> bool:
    """Return True (PASS) iff ALL three conditions hold:

    1. candidate.mean_score() >= baseline.mean_score() - _EPSILON
       (no aggregate regression vs baseline)
    2. candidate.mean_score() >= min_mean
       (candidate meets an absolute floor; default 0.0 = no floor)
    3. allow_regressions=True OR no per-example delta < 0
       (no individual example got worse, unless you opt in)

    Returns False (FAIL) if any condition is violated.
    """
    cand_mean = candidate.mean_score()
    base_mean = baseline.mean_score()
    if cand_mean < base_mean - _EPSILON:
        return False
    if cand_mean < min_mean:
        return False
    if not allow_regressions:
        for row in compare_reports(baseline, candidate):
            if row["delta"] < 0:
                return False
    return True


print("Regression tools declared OK")
```

- [ ] **Step 7: Add the LLM judge re-declaration code cell**

```python
# ── LLM judge (re-declared from notebook 05) ───────────────────────────────

def make_llm_judge(
    rubric: str,
    client: OpenAI,
    model: str = "gpt-4o-mini",
    key: str = "llm_judge",
) -> Callable[[Example, Any], Score]:
    """Return a grader that asks an LLM to score the output against `rubric`.

    The LLM must respond with JSON: {"score": <0.0–1.0>, "passed": <bool>, "comment": <str>}.
    """
    system_prompt = textwrap.dedent(f"""
        You are a strict eval judge. Score the agent output against the rubric below.
        Respond ONLY with valid JSON (no markdown):
        {{"score": <float 0.0–1.0>, "passed": <bool>, "comment": "<one sentence>"}}

        Rubric:
        {rubric}
    """).strip()

    def grader(example: Example, output: Any) -> Score:
        text = json.dumps(output, ensure_ascii=False) if isinstance(output, dict) else str(output)
        user_msg = f"Input: {json.dumps(example.input)}\n\nOutput to score:\n{text}"
        response = client.chat.completions.create(
            model=model,
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": user_msg},
            ],
            temperature=0,
        )
        raw = response.choices[0].message.content.strip()
        try:
            parsed = json.loads(raw)
            return Score(
                key=key,
                score=float(parsed["score"]),
                passed=bool(parsed["passed"]),
                comment=parsed.get("comment", ""),
            )
        except Exception as exc:
            return Score(key=key, score=0.0, passed=False, comment=f"judge parse error: {exc}")

    return grader


print("LLM judge declared OK")
```

- [ ] **Step 8: Add the LangSmith adapter re-declaration code cell**

```python
# ── LangSmith adapter (re-declared from notebook 07) ───────────────────────
# NOTE: Section 6 (LangSmith experiment) is skipped if LANGCHAIN_API_KEY is missing.

def as_langsmith_evaluator(
    grader: Callable[[Example, Any], Score],
) -> Callable[[dict], dict]:
    """Wrap a grader function so LangSmith's `evaluate()` can call it.

    LangSmith passes evaluators a dict with keys 'inputs', 'outputs', and
    optionally 'reference_outputs'. We reconstruct an Example + call the grader.
    """
    def evaluator(run_dict: dict) -> dict:
        example = Example(
            input=run_dict.get("inputs", {}),
            expected=run_dict.get("reference_outputs"),
        )
        output = run_dict.get("outputs", {})
        score_obj = grader(example, output)
        return {
            "key": score_obj.key,
            "score": score_obj.score,
            "comment": score_obj.comment,
        }
    evaluator.__name__ = getattr(grader, "__name__", "grader")
    return evaluator


print("LangSmith adapter declared OK")
print()
print("=" * 50)
print("Full toolkit ready. Proceeding to Section 2.")
print("=" * 50)
```

- [ ] **Step 9: Run all setup cells and verify each prints OK**

Expected final output:

```
Full toolkit ready. Proceeding to Section 2.
```

No import errors. If `openai` or `langsmith` are missing: `pip install openai langsmith langchain langchain-openai`.

- [ ] **Step 10: Commit**

```bash
git add evals/08_production_eval_gate_end_to_end.ipynb
git commit -m "feat(evals): add setup, guard, and full toolkit re-declaration to notebook 08"
```

---

## Task 4: Section 2 — A realistic agent

**Files:**
- Modify: `evals/08_production_eval_gate_end_to_end.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 2. A Realistic Agent

Before we can evaluate anything we need something to evaluate. We'll build a **research-and-summarize agent** with two steps that mirror a real production agent:

1. **Tool step:** `search(query)` — returns canned "search results" from an in-memory corpus. No live web access needed; the corpus is deterministic, so the eval dataset has ground truth.
2. **LLM synthesis step:** calls `gpt-4o-mini` to write a short summary that cites the retrieved results.

The agent takes `{"query": str}` and returns `{"summary": str, "sources": list[str]}`.

We also define `research_agent_v2` — a deliberately-regressed version that drops source citations from the synthesis prompt, causing the summary to miss key facts. Section 4 uses this regression to demonstrate the gate.

> **Gotcha:** The synthesis step calls a real LLM, so `summary` text will vary between runs. This is exactly why deterministic graders alone can't cover quality — and why the LLM judge in Section 3 is needed.
```

- [ ] **Step 2: Add the canned corpus + `research_agent` code cell**

```python
# ── Canned corpus (deterministic — no live web) ────────────────────────────

CORPUS: dict[str, list[dict]] = {
    "climate change": [
        {"title": "Global Temperatures 2024", "snippet": "The average global temperature rose by 1.3°C above pre-industrial levels in 2024, driven largely by CO2 emissions from fossil fuels."},
        {"title": "Arctic Ice Loss", "snippet": "Arctic sea ice extent reached a record low in September 2024, continuing a three-decade decline linked to warming ocean temperatures."},
        {"title": "Renewable Energy Surge", "snippet": "Solar and wind capacity additions hit 500 GW globally in 2024, displacing an estimated 1.2 billion tonnes of CO2 emissions."},
    ],
    "large language models": [
        {"title": "Scaling Laws Revisited", "snippet": "Recent work shows LLM capability scales predictably with compute, data, and parameters, but emergent abilities appear at qualitatively different scales."},
        {"title": "Alignment Research", "snippet": "RLHF and constitutional AI methods have improved model helpfulness and reduced harmful outputs, though adversarial prompting remains a challenge."},
        {"title": "Inference Efficiency", "snippet": "Quantization and speculative decoding have reduced LLM inference costs by 4–8x compared to 2022 baselines, enabling wider deployment."},
    ],
    "default": [
        {"title": "No results", "snippet": "No matching documents found in the corpus for this query."},
    ],
}


def _search(query: str) -> list[dict]:
    """Return canned search results for the query (case-insensitive substring match)."""
    q = query.lower()
    for topic, results in CORPUS.items():
        if topic in q:
            return results
    return CORPUS["default"]


# ── research_agent (v1 — good agent) ──────────────────────────────────────

_SYNTHESIS_PROMPT_V1 = """\
You are a research assistant. Given search results, write a concise 2–3 sentence summary
that directly answers the query. Cite the titles of the sources you use, like [Source Title].
Be factual and specific — include numbers or named concepts from the results when relevant.

Query: {query}

Search results:
{results}
"""


def research_agent(input: dict) -> dict:
    """Good agent: retrieves results and synthesises a cited summary."""
    query = input["query"]
    results = _search(query)
    results_text = "\n".join(
        f"- [{r['title']}]: {r['snippet']}" for r in results
    )
    prompt = _SYNTHESIS_PROMPT_V1.format(query=query, results=results_text)
    response = openai_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.2,
    )
    summary = response.choices[0].message.content.strip()
    sources = [r["title"] for r in results]
    return {"summary": summary, "sources": sources}


# ── research_agent_v2 (deliberately regressed) ───────────────────────────

_SYNTHESIS_PROMPT_V2 = """\
Write a short paragraph about: {query}
"""


def research_agent_v2(input: dict) -> dict:
    """Regressed agent: ignores search results, produces generic uncited summaries."""
    query = input["query"]
    results = _search(query)  # still calls search but does NOT pass results to LLM
    response = openai_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": _SYNTHESIS_PROMPT_V2.format(query=query)}],
        temperature=0.7,
    )
    summary = response.choices[0].message.content.strip()
    # Regression: returns empty sources list even though results were retrieved
    return {"summary": summary, "sources": []}


# Quick smoke test
print("Smoke-testing research_agent on 'climate change'...")
sample = research_agent({"query": "climate change"})
print(f"  summary snippet : {sample['summary'][:120]}...")
print(f"  sources         : {sample['sources']}")
print()
print("Smoke-testing research_agent_v2 on 'climate change'...")
sample_v2 = research_agent_v2({"query": "climate change"})
print(f"  summary snippet : {sample_v2['summary'][:120]}...")
print(f"  sources         : {sample_v2['sources']}")
```

- [ ] **Step 3: Run and verify the smoke test**

Expected pattern (LLM text varies):

```
Smoke-testing research_agent on 'climate change'...
  summary snippet : In 2024, the average global temperature rose by 1.3°C above pre-industrial levels...
  sources         : ['Global Temperatures 2024', 'Arctic Ice Loss', 'Renewable Energy Surge']

Smoke-testing research_agent_v2 on 'climate change'...
  summary snippet : Climate change refers to long-term shifts in global temperatures and weather patterns...
  sources         : []
```

Key structural checks: `research_agent` returns non-empty `sources` list; `research_agent_v2` returns `sources == []`.

- [ ] **Step 4: Commit**

```bash
git add evals/08_production_eval_gate_end_to_end.ipynb
git commit -m "feat(evals): add research_agent and research_agent_v2 to notebook 08"
```

---

## Task 5: Section 3 — The full offline suite

**Files:**
- Modify: `evals/08_production_eval_gate_end_to_end.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 3. The Full Offline Suite

A production eval suite for `research_agent` needs to check three different things:

| Concern | Grader type | Why |
|---|---|---|
| Output shape — does it return `summary` and `sources`? | `make_schema_grader` (deterministic) | Structural contract, always cheap to check. |
| Key facts in the summary — are the numbers and named concepts from the results cited? | `make_contains` (deterministic) | Fast, reliable, low cost. |
| Faithfulness and quality — is the summary accurate, well-written, and grounded in the sources? | `make_llm_judge` rubric | LLM-only judgement; deterministic graders can't reach this. |

We assemble a `dataset` of two representative `Example`s, one per corpus topic, with `metadata` that documents which key facts are required. We then run `run_eval` and print the `summary_table()`.

> **Gotcha:** The LLM judge is non-deterministic, so `llm_judge` pass rates may vary slightly across runs. This is expected — the deterministic graders give you the stable floor; the LLM judge adds signal for quality that couldn't otherwise be checked.
```

- [ ] **Step 2: Add the dataset + graders code cell**

```python
# ── Dataset (two examples, one per corpus topic) ───────────────────────────

dataset: list[Example] = [
    Example(
        input={"query": "climate change"},
        expected=None,  # open-ended output — no single ground truth
        metadata={
            "required_facts": ["1.3°C", "Arctic", "solar", "wind"],
            "topic": "climate change",
        },
    ),
    Example(
        input={"query": "large language models"},
        expected=None,
        metadata={
            "required_facts": ["scaling", "RLHF", "quantization"],
            "topic": "large language models",
        },
    ),
]


# ── Graders ────────────────────────────────────────────────────────────────

schema_grader = make_schema_grader(
    required_keys=["summary", "sources"],
    key="output_schema",
)

# One contains-grader per required fact for the first example (climate change)
climate_graders = [
    make_contains("1.3°C",     key="contains:1.3°C"),
    make_contains("Arctic",    key="contains:Arctic"),
    make_contains("solar",     key="contains:solar"),
]

# One contains-grader per required fact for the second example (LLMs)
llm_graders = [
    make_contains("scaling",      key="contains:scaling"),
    make_contains("RLHF",         key="contains:RLHF"),
    make_contains("quantization", key="contains:quantization"),
]

faithfulness_judge = make_llm_judge(
    rubric=(
        "Score 1.0 if the summary is factually grounded in the provided search results, "
        "cites specific facts or numbers, and avoids hallucination. "
        "Score 0.5 if it is partially grounded but vague or missing key specifics. "
        "Score 0.0 if it is generic, uncited, or contradicts the search results."
    ),
    client=openai_client,
    model="gpt-4o-mini",
    key="llm_judge",
)

# Combine: schema + climate contains + llm contains + judge
# (contains graders are applied to ALL examples; for a real suite you'd tag by metadata)
graders = [schema_grader] + climate_graders + llm_graders + [faithfulness_judge]

print(f"Dataset: {len(dataset)} examples")
print(f"Graders: {len(graders)} ({len(graders)-1} deterministic + 1 LLM judge)")
```

- [ ] **Step 3: Add the run_eval + summary code cell**

```python
print("Running full offline suite on research_agent (v1)...")
report_v1 = run_eval(research_agent, dataset, graders)
print()
print(report_v1.summary_table())
print()
print(f"Overall pass rate : {report_v1.pass_rate():.0%}")
print(f"Overall mean score: {report_v1.mean_score():.3f}")
```

- [ ] **Step 4: Run the cells and verify**

Expected pattern (LLM judge scores may vary; deterministic graders should all pass for v1):

```
Running full offline suite on research_agent (v1)...

grader                    pass_rate  mean_score
---------------------------------------------------
contains:1.3°C              100.00%       1.000
contains:Arctic             100.00%       1.000
contains:RLHF               100.00%       1.000
contains:quantization       100.00%       1.000
contains:scaling            100.00%       1.000
contains:solar              100.00%       1.000
llm_judge                   100.00%       0.950
output_schema               100.00%       1.000
---------------------------------------------------
OVERALL                     100.00%       0.981
```

Deterministic graders should all be 100% for v1. LLM judge may vary.

- [ ] **Step 5: Commit**

```bash
git add evals/08_production_eval_gate_end_to_end.ipynb
git commit -m "feat(evals): add full offline eval suite (mixed graders) to notebook 08"
```

---

## Task 6: Section 4 — The regression gate

**Files:**
- Modify: `evals/08_production_eval_gate_end_to_end.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 4. The Regression Gate

`research_agent_v2` is a deliberately broken version that ignores the search results entirely and writes a generic uncited summary. We'd want a CI gate to catch this before it ships.

Steps:
1. Run `run_eval` on `research_agent_v2` with the same dataset and graders.
2. Use `compare_reports(baseline=report_v1, candidate=report_v2)` to see per-grader deltas.
3. Call `regression_gate(report_v1, report_v2)` — it returns `False` (FAIL) when the candidate regresses.
4. Print a clear PASS / FAIL banner.

> **Gotcha:** `regression_gate` returns `True` for PASS (the gate admits the candidate) and `False` for FAIL (the gate blocks the candidate). The naming follows the semantics of a CI gate: `True` = green = ship it.
```

- [ ] **Step 2: Add the regression evaluation code cell**

```python
print("Running full offline suite on research_agent_v2 (regressed)...")
report_v2 = run_eval(research_agent_v2, dataset, graders)
print()
print("=== v2 suite results ===")
print(report_v2.summary_table())
print()
deltas = compare_reports(report_v1, report_v2)
print("=== Per-example deltas: v1 (baseline) vs v2 (candidate) ===")
for row in deltas:
    flag = "  <-- REGRESSED" if row["regressed"] else ""
    print(f"  input={row['input']}  base={row['baseline_score']:.2f}  cand={row['candidate_score']:.2f}  delta={row['delta']:+.2f}{flag}")

passed = regression_gate(report_v1, report_v2)
print()
print(f"GATE: {'PASS' if passed else 'FAIL'}")
```

- [ ] **Step 3: Add the gate verdict code cell**

```python
# Also verify v1 passes its own gate (sanity check)
v1_self_gate = regression_gate(report_v1, report_v1)
print(f"Sanity check — v1 vs. itself: {'PASS' if v1_self_gate else 'FAIL'} (expected PASS)")
```

- [ ] **Step 4: Run and verify**

Expected output pattern:

```
=== Per-example deltas: v1 (baseline) vs v2 (candidate) ===
  input={'query': 'climate change'}  base=0.86  cand=0.29  delta=-0.57  <-- REGRESSED
  input={'query': 'large language models'}  base=0.86  cand=0.29  delta=-0.57  <-- REGRESSED

GATE: FAIL

Sanity check — v1 vs. itself: PASS (expected PASS)
```

Key: `passed == False` for v2 (regression detected on both examples), `v1_self_gate == True` (sanity check). Exact scores vary because the LLM judge is involved — the structural shape (two regressed rows, `GATE: FAIL`) is what matters.

- [ ] **Step 5: Commit**

```bash
git add evals/08_production_eval_gate_end_to_end.ipynb
git commit -m "feat(evals): add regression gate section with PASS/FAIL banner to notebook 08"
```

---

## Task 7: Section 5 — CI-style gate

**Files:**
- Modify: `evals/08_production_eval_gate_end_to_end.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 5. CI-Style Gate

In CI you want a single command that exits `0` on pass and non-zero on failure. We express the full suite as a `run_gate()` function that returns `0` (pass) or `1` (fail) — a Python analogue to a CI exit code.

Below that we show an illustrative `pytest` snippet. This is an **in-notebook illustration** — it is not a separate file — but it maps directly onto a real `tests/test_no_regression.py` you could drop into a project and run with `pytest`.
```

- [ ] **Step 2: Add the `run_gate()` code cell**

```python
def run_gate(
    good_agent=research_agent,
    bad_agent=research_agent_v2,
    dataset: list[Example] = dataset,
    graders: list = graders,
    verbose: bool = True,
) -> int:
    """Run the full eval suite and regression gate. Returns 0 (pass) or 1 (fail)."""
    if verbose:
        print("Running eval on baseline (v1)...")
    baseline = run_eval(good_agent, dataset, graders)

    if verbose:
        print("Running eval on candidate (v2)...")
    candidate = run_eval(bad_agent, dataset, graders)

    passed = regression_gate(baseline, candidate, min_mean=0.0, allow_regressions=False)

    if verbose:
        print()
        if passed:
            print("EXIT 0 — PASS: candidate meets quality bar.")
        else:
            print("EXIT 1 — FAIL: regression detected.")
            for row in compare_reports(baseline, candidate):
                if row["regressed"]:
                    print(f"  input={row['input']}  base={row['baseline_score']:.3f} → cand={row['candidate_score']:.3f}  (Δ {row['delta']:+.3f})")

    return 0 if passed else 1


exit_code = run_gate()
print()
print(f"run_gate() returned: {exit_code}")
```

- [ ] **Step 3: Add the illustrative pytest snippet code cell**

```python
# ── Illustrative pytest / CI sketch (in-notebook, not a real file) ─────────
#
# In a real project you would put this in tests/test_no_regression.py and run:
#   pytest tests/test_no_regression.py
# or in CI:
#   python -c "import sys; from evals.gate import run_gate; sys.exit(run_gate())"
#
# The snippet below is illustrative — it shows exactly what the real test file
# would look like, but we keep it in a comment block so Jupyter doesn't try
# to import pytest.

PYTEST_SKETCH = '''\
# tests/test_no_regression.py  (illustrative — not executed in this notebook)

import pytest
from evals.agents import research_agent, research_agent_v2  # hypothetical module
from evals.suite import dataset, graders                     # hypothetical module
from evals.harness import run_eval, regression_gate


def test_no_regression():
    """Agent v2 must not regress below the v1 baseline on the offline suite."""
    baseline  = run_eval(research_agent,    dataset, graders)
    candidate = run_eval(research_agent_v2, dataset, graders)
    assert regression_gate(baseline, candidate, min_mean=0.0, allow_regressions=False), (
        "Regression detected: research_agent_v2 scores lower than research_agent on one or "
        "more graders. Fix the agent or update the baseline intentionally."
    )
'''

print(PYTEST_SKETCH)
print()
print("In CI (e.g. GitHub Actions) you would run:")
print("  pytest tests/test_no_regression.py --tb=short")
print("and the pipeline step would fail if the assertion raises.")
```

- [ ] **Step 4: Run and verify**

Expected:
- `run_gate()` prints `EXIT 1 — FAIL` (because v2 is regressed) and returns `1`.
- The pytest sketch prints cleanly.

- [ ] **Step 5: Commit**

```bash
git add evals/08_production_eval_gate_end_to_end.ipynb
git commit -m "feat(evals): add CI-style run_gate() and pytest sketch to notebook 08"
```

---

## Task 8: Section 6 — Into the platform + online monitoring

**Files:**
- Modify: `evals/08_production_eval_gate_end_to_end.ipynb` (add 1 markdown cell + 3 code cells)

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 6. Into the Platform + Online Monitoring

*(Requires `LANGCHAIN_API_KEY`. If it is not set, this section is skipped — a guard cell checks before making any API calls.)*

In notebook 07 we pushed individual graders into LangSmith via `evaluate`. Here we do the same for the full production suite: upload the dataset, run `evaluate` with `as_langsmith_evaluator`-wrapped graders, and print the experiment URL so you can open it in the LangSmith UI.

After the experiment run, we end with a short conceptual cell on **online monitoring** — how the same graders apply to sampled live traffic.
```

- [ ] **Step 2: Add the LangSmith guard + experiment code cell**

```python
if not _langsmith_ok:
    print("⚠ LANGCHAIN_API_KEY not set — skipping LangSmith experiment (Section 6).")
    print("  Set LANGCHAIN_API_KEY and re-run to log an experiment.")
else:
    from langsmith import Client as LangSmithClient
    from langsmith.evaluation import evaluate as ls_evaluate

    ls_client = LangSmithClient()

    # ── Upload dataset to LangSmith ────────────────────────────────────────
    DATASET_NAME = "evals-08-research-agent-v1"

    # Delete old copy if it exists so re-runs are idempotent
    try:
        ls_client.delete_dataset(dataset_name=DATASET_NAME)
        print(f"Deleted existing dataset '{DATASET_NAME}'")
    except Exception:
        pass

    ls_dataset = ls_client.create_dataset(DATASET_NAME, description="Notebook 08 capstone eval dataset")
    ls_client.create_examples(
        inputs=[ex.input for ex in dataset],
        outputs=[ex.expected for ex in dataset],
        dataset_id=ls_dataset.id,
    )
    print(f"Uploaded {len(dataset)} examples to LangSmith dataset '{DATASET_NAME}'")

    # ── Wrap a representative subset of graders for LangSmith ─────────────
    # We pick schema + one contains + LLM judge to keep the experiment concise.
    ls_evaluators = [
        as_langsmith_evaluator(schema_grader),
        as_langsmith_evaluator(make_contains("Arctic", key="contains:Arctic")),
        as_langsmith_evaluator(faithfulness_judge),
    ]

    # ── Run the LangSmith experiment ───────────────────────────────────────
    def _agent_for_langsmith(inputs: dict) -> dict:
        return research_agent(inputs)

    experiment_results = ls_evaluate(
        _agent_for_langsmith,
        data=DATASET_NAME,
        evaluators=ls_evaluators,
        experiment_prefix="nb08-production-gate",
        metadata={"notebook": "08", "agent_version": "v1"},
    )

    print()
    print("LangSmith experiment complete.")
    # experiment_results.experiment_name gives the run name; URL is in the UI
    exp_name = getattr(experiment_results, "experiment_name", "see LangSmith UI")
    print(f"Experiment name : {exp_name}")
    print(f"View results at : https://smith.langchain.com")
    print()
    print("Open the URL above and navigate to Datasets & Experiments to see")
    print("per-example scores, grader breakdowns, and cost/latency metadata.")
```

- [ ] **Step 3: Add the online monitoring conceptual cell**

```python
# ── Online monitoring — conceptual sketch ─────────────────────────────────
#
# Everything above is OFFLINE evaluation: a fixed dataset, run on-demand or in CI.
# ONLINE monitoring applies the same graders to a sample of LIVE production traffic.
#
# Conceptually:
#
#   for request, response in sample_live_traffic(rate=0.05):   # 5% sampling
#       example = Example(input=request, expected=None)
#       for grader in [schema_grader, faithfulness_judge]:
#           score = grader(example, response)
#           emit_metric(score.key, score.score)                 # → Datadog / CloudWatch / LangSmith
#
# The same graders you wrote for offline CI work online — you just swap the
# dataset source for a live traffic sampler and replace summary_table() with
# a time-series metric emission.
#
# LangSmith supports this via its "online evaluation" feature: you attach
# evaluators to a project and it scores a sampled fraction of traced runs
# automatically.

print("Online monitoring pattern printed above as a code comment.")
print("The same graders run offline (CI) and online (production sampling) —")
print("the only difference is the data source and metric emission target.")
```

- [ ] **Step 4: Run and verify**

If `LANGCHAIN_API_KEY` is set: the cell prints the experiment name and a LangSmith URL.
If not set: the cell prints the skip warning. Neither path raises an unhandled exception.

- [ ] **Step 5: Commit**

```bash
git add evals/08_production_eval_gate_end_to_end.ipynb
git commit -m "feat(evals): add LangSmith experiment and online monitoring section to notebook 08"
```

---

## Task 9: Closing cells — "What you just learned" and "Where to go next"

**Files:**
- Modify: `evals/08_production_eval_gate_end_to_end.ipynb` (add 2 markdown cells)

*(No "What's missing" cell — this is the last notebook in the series.)*

- [ ] **Step 1: Add the "What you just learned" markdown cell**

```markdown
## What you just learned

You just climbed the full test pyramid for agents — from bottom to top:

- **Notebook 01:** `assert agent(x) == "expected"` flakes because agents are non-deterministic, have no single ground truth, and fail in multi-step ways. Raw assertions can't carry a production eval.
- **Notebook 02:** Deterministic graders — `exact_match`, `make_contains`, `make_schema_grader` — are the cheapest, most reliable checks. No LLM needed.
- **Notebook 03:** The eval harness — `Score`, `Example`, `EvalReport`, `run_eval` — turns ad-hoc assertions into a composable, reusable system.
- **Notebook 04:** The regression gate — `compare_reports` + `regression_gate` — catches quality drops before they ship, entirely offline.
- **Notebook 05:** `make_llm_judge` extends the harness to open-ended outputs that deterministic graders can't reach, with explicit rubrics and awareness of judge bias.
- **Notebook 06:** LangSmith tracing makes multi-step agent runs visible — you can read spans, debug failures, and observe latency and token costs per step.
- **Notebook 07:** LangSmith experiments move the offline harness into the platform — you compare runs across agent versions and track cost/latency at scale.
- **Notebook 08 (this one):** Everything assembled into a **production eval gate**: a realistic research-and-summarize agent, a mixed grader suite, a `run_gate()` CI entry point that returns 0 or 1, a LangSmith experiment, and a conceptual sketch of online monitoring.

The toolkit you have now — deterministic graders + LLM judge + regression gate + platform integration — is the same shape used by production ML teams. The pieces are small enough to understand, and composable enough to scale.
```

- [ ] **Step 2: Add the "Where to go next" markdown cell**

```markdown
## Where to go next

You now have a working eval toolkit. Here are the agents in this repo that are ready to be evaluated with it:

**`langgraph/`** — LangGraph multi-agent workflows and supervisor patterns. These are ideal first targets: they have structured inputs and outputs, and their multi-step nature means multi-step failure modes are real. Wire `run_eval` against the LangGraph agent's `invoke` method and add a `make_schema_grader` for the output graph state.

**`mcp/`** — MCP tool-calling servers and clients. Evaluate tool selection quality (did the agent pick the right tool for the query?) with `make_contains` graders on tool names, and evaluate output faithfulness with `make_llm_judge`.

**`a2a/`** — A2A multi-agent communication protocols. The inter-agent message payloads are structured JSON, making `make_schema_grader` an excellent fit. Use `make_llm_judge` to score the coherence of agent-to-agent reasoning chains.

**`crewai/`** — CrewAI crew runs with role-based agents. Score final crew outputs with the full suite; use LangSmith experiments to compare different crew configurations.

For any of these, the starting template is the same:

```python
dataset = [Example(input={...}, expected=None, metadata={}) for ...]
graders = [make_schema_grader([...]), make_contains("..."), make_llm_judge(rubric, client)]
report  = run_eval(your_agent, dataset, graders)
print(report.summary_table())
```

The hard part — building the harness — is already done.
```

- [ ] **Step 3: Commit**

```bash
git add evals/08_production_eval_gate_end_to_end.ipynb
git commit -m "feat(evals): add closing recap and Where to go next to notebook 08"
```

---

## Task 10: End-to-end verification in a fresh kernel

**Files:** none (verification only).

- [ ] **Step 1: Confirm prerequisites are available in the environment**

```bash
python -c "import openai, langsmith, langchain, langchain_openai; print('All deps present')"
echo "OPENAI_API_KEY set: $(test -n \"$OPENAI_API_KEY\" && echo YES || echo NO)"
echo "LANGCHAIN_API_KEY set: $(test -n \"$LANGCHAIN_API_KEY\" && echo YES || echo NO)"
```

Both keys must be set for a complete run. The notebook handles missing `LANGCHAIN_API_KEY` gracefully (Section 6 is skipped), but `OPENAI_API_KEY` is required for the agent + LLM judge (Sections 2–5).

- [ ] **Step 2: Execute the notebook top-to-bottom in a clean kernel**

```bash
jupyter nbconvert --to notebook --execute --inplace evals/08_production_eval_gate_end_to_end.ipynb
```

Expected: succeeds with no unhandled exceptions. The notebook is rewritten with embedded outputs.

- [ ] **Step 3: Verify structural outcomes in the executed notebook**

Open the executed notebook and confirm the following structural checks (model output text varies — check structure, not exact strings):

1. **Section 1 guard cell** — prints either "All sections will run." or lists which env vars are missing without raising.
2. **Section 1 toolkit cells** — each prints `... declared OK`. Final cell prints "Full toolkit ready."
3. **Section 2 smoke test** — `research_agent` smoke test output shows a non-empty `sources` list; `research_agent_v2` smoke test shows `sources == []`.
4. **Section 3 summary table** — `output_schema` grader shows 100% pass rate for `research_agent` (v1). `summary_table()` renders without error.
5. **Section 4 PASS/FAIL banner** — the gate verdict cell prints the `GATE: FAIL` banner (v2 is regressed). The sanity-check line prints `v1 vs. itself: PASS`.
6. **Section 5 `run_gate()`** — prints `EXIT 1 — FAIL` and the last line reads `run_gate() returned: 1`.
7. **Section 6 LangSmith cell** — if `LANGCHAIN_API_KEY` is set, prints experiment name and LangSmith URL without error. If not set, prints the skip warning without raising.

- [ ] **Step 4: Commit the clean run**

```bash
git add evals/08_production_eval_gate_end_to_end.ipynb
git commit -m "chore(evals): commit clean fresh-kernel run of notebook 08"
```

---

## Task 11: Document the series in the repo README

**Files:**
- Modify: `/Users/nimeshpatel/Documents/AI Agents/README.md`

- [ ] **Step 1: Add an `## evals/ — Agent Evaluation & Observability` section to `README.md`**

Place it AFTER the existing `## a2a/` section, mirroring the existing series tables. Include an 8-row markdown table:

```markdown
## evals/ — Agent Evaluation & Observability

| Notebook | What you'll learn |
|---|---|
| [`evals/01_why_agents_are_hard_to_test.ipynb`](evals/01_why_agents_are_hard_to_test.ipynb) | Why `assert agent(x) == expected` flakes — non-determinism, multi-step failures, and the need for graded evaluation. |
| [`evals/02_assertions_and_golden_outputs.ipynb`](evals/02_assertions_and_golden_outputs.ipynb) | Deterministic graders: `exact_match`, `make_contains`, `make_schema_grader` — cheap, reliable, no LLM needed. |
| [`evals/03_building_an_eval_harness.ipynb`](evals/03_building_an_eval_harness.ipynb) | The eval harness: `Score`, `Example`, `EvalReport`, and `run_eval(agent, dataset, graders) → EvalReport`. |
| [`evals/04_regression_testing.ipynb`](evals/04_regression_testing.ipynb) | The regression gate: `compare_reports` + `regression_gate` catch quality drops before they ship, entirely offline. |
| [`evals/05_llm_as_judge.ipynb`](evals/05_llm_as_judge.ipynb) | `make_llm_judge` extends the harness to open-ended outputs with explicit rubrics; judge bias and calibration. |
| [`evals/06_tracing_with_langsmith.ipynb`](evals/06_tracing_with_langsmith.ipynb) | LangSmith tracing makes multi-step agent runs visible — read spans, debug failures, observe cost and latency. |
| [`evals/07_datasets_experiments_and_cost.ipynb`](evals/07_datasets_experiments_and_cost.ipynb) | LangSmith experiments: move the offline harness into the platform, compare runs across versions, track cost at scale. |
| [`evals/08_production_eval_gate_end_to_end.ipynb`](evals/08_production_eval_gate_end_to_end.ipynb) | Capstone: full toolkit assembled into a CI-style production gate — mixed graders, regression gate, LangSmith experiment, online monitoring sketch. |
```

- [ ] **Step 2: Update Prerequisites / Quick start notes for the evals series**

Add or update the relevant section so it states:

- evals 01–04 need **no API key**
- evals 05–08 need `OPENAI_API_KEY`
- evals 06–08 additionally need a free LangSmith account (`LANGCHAIN_API_KEY`)
- pip install line for the series:

```
pip install langsmith langchain langchain-openai openai pydantic
```

- [ ] **Step 3: Add a recommended-learning-path line for the evals series**

Add a "Want to know if your agent works?" recommended-learning-path line pointing at the `evals/` series, matching the style of the existing recommended-path lines in the README.

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "docs(evals): document the evals & observability series in README"
```

---

## Done

After Task 10 passes, notebook 08 is complete and the **evals & observability series is complete** — all 8 notebooks exist and run end-to-end. Task 11 adds the `evals/` section to the repo root `README.md`, documenting all 8 notebooks with their filenames, prerequisites (no key for 01–04; `OPENAI_API_KEY` for 05–08; `LANGCHAIN_API_KEY` for 06–08), a recommended learning path entry, and the pip install line for the series — completing the series end to end.
