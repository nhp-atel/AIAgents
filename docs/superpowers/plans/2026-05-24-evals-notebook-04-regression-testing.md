# Evals Notebook 04 — `04_regression_testing.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build notebook 04 in the Agent Evals & Observability learning series — show how to use the eval harness built in notebook 03 to diff two agent versions and gate on quality regressions. The notebook introduces `compare_reports` (per-example delta table) and `regression_gate` (pass/fail decision rule), runs both against `agent_v1` (good) and `agent_v2` (deliberately regressed), and prints a clear PASS/FAIL banner. Everything is fully deterministic — no API key.

**Architecture:** A single self-contained Jupyter notebook. All shared harness primitives (`Score`, `Example`, `ExampleResult`, `EvalReport`, `run_eval`, and the five deterministic graders) are re-declared inline in Section 1 so the notebook runs alone without importing from earlier notebooks. Two new functions are defined fresh in this notebook: `compare_reports` and `regression_gate`. Two deterministic agent stubs (`agent_v1`, `agent_v2`) are defined in Section 3.

**Tech Stack:** Python 3.11+, Jupyter, `pydantic` v2 (for `make_schema_grader`), `re`, `json`, standard library only. No external API keys.

**Companion spec:** `docs/superpowers/specs/2026-05-24-evals-observability-learning-series-design.md` (notebook 04 section).

**Folder:** `evals/` — the folder is created by the series' first notebook plan. Task 1 of this plan scaffolds the empty notebook file inside the existing folder.

---

## File Structure

- **Create:** `evals/04_regression_testing.ipynb` — the entire notebook, self-contained.
- **Modify:** none.
- **No separate test files** — verification is the notebook running top-to-bottom in a fresh kernel (Task 10).

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown)
2. **"What you'll learn"** (markdown bullets)
3. **Section 1: Setup — re-declare the harness** (markdown + 1 code cell — copy all shared primitives forward from nb 03; note this inline)
4. **Section 2: A versioned dataset** (markdown + 1 code cell — ~6 `Example` objects; note that in practice you'd commit this as a JSON file)
5. **Section 3: Two agent versions** (markdown + 1 code cell — `agent_v1` and `agent_v2`; v2 deliberately mishandles 2 examples)
6. **Section 4: Run both, diff** (markdown + 1 code cell — `run_eval` each; define + call `compare_reports`; print delta table)
7. **Section 5: The gate** (markdown + 1 code cell — define `regression_gate`; show PASS for v1-vs-v1, FAIL for v1-vs-v2; PASS/FAIL banner)
8. **`### Try it` — fix v2 and re-run** (markdown + 1 code cell — fix v2's bug; re-run gate; watch it flip to PASS)
9. **Gotcha — flaky graders undermine gates** (markdown blockquote)
10. **"What you just learned"** (markdown)
11. **"What's missing"** (markdown teaser to nb 05)

---

## Task 1: Scaffold the empty notebook

**Files:**
- Create: `evals/04_regression_testing.ipynb`

- [ ] **Step 1: Confirm the `evals/` folder exists**

```bash
ls -la /Users/nimeshpatel/Documents/AI\ Agents/evals/
```

Expected: the folder exists (created by notebook 01's plan). If it doesn't exist yet:

```bash
mkdir /Users/nimeshpatel/Documents/AI\ Agents/evals
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

Save as `evals/04_regression_testing.ipynb`.

- [ ] **Step 3: Verify the file is valid JSON**

```bash
python -c "import json; json.load(open('evals/04_regression_testing.ipynb'))"
ls -la evals/04_regression_testing.ipynb
```

Expected: no exception, file non-zero size.

- [ ] **Step 4: Commit**

```bash
git add evals/04_regression_testing.ipynb
git commit -m "feat(evals): scaffold 04_regression_testing.ipynb"
```

---

## Task 2: Add title and "Why this notebook exists"

**Files:**
- Modify: `evals/04_regression_testing.ipynb` (add cell 1)

- [ ] **Step 1: Add a markdown cell with title and motivation**

Cell content (markdown):

```markdown
# 04 — Regression Testing

## Why this notebook exists

In **notebook 03** we built a reusable eval harness: `run_eval(agent, dataset, graders)` runs a list of graders against every example in a dataset and hands back an `EvalReport` with pass rates and mean scores. That harness is the tool we needed — but using it once tells you how your agent performs *today*. It doesn't tell you whether it got *worse* between yesterday and today.

This notebook adds the missing layer: **regression testing**. We version an eval dataset (treat it as a checked-in artifact that evolves deliberately, not accidentally), run the harness against two different agent versions, compute a per-example delta table, and encode a **gate** that returns PASS or FAIL automatically. This is the deterministic precursor to the CI gate assembled in full in notebook 08 — the same logic, but wired by hand here so the mechanics are transparent.

No API key is needed. Both agent versions are deterministic Python stubs.
```

- [ ] **Step 2: Verify the cell renders**

Confirm the markdown cell renders with the bold cross-references and no raw asterisks showing.

- [ ] **Step 3: Commit**

```bash
git add evals/04_regression_testing.ipynb
git commit -m "feat(evals): add intro markdown to notebook 04"
```

---

## Task 3: Add "What you'll learn"

**Files:**
- Modify: `evals/04_regression_testing.ipynb` (add cell 2)

- [ ] **Step 1: Add a markdown cell**

Cell content (markdown):

```markdown
## What you'll learn

- How to treat an eval **dataset as a versioned artifact** — a list of `Example` objects you'd commit to source control so regressions are caught on the diff, not in production.
- How to compare two `EvalReport` objects with **`compare_reports`** — a per-example delta table showing which examples regressed, held steady, or improved.
- How to encode a **`regression_gate`** — a single boolean function that returns `True` (PASS) or `False` (FAIL) based on an explicit, auditable rule combining aggregate score floor and per-example regression tolerance.
- Why **determinism in graders** is a prerequisite for gates to be meaningful — a gate built on a flaky grader is a false sense of safety.
- How this deterministic gate is the foundation for the CI-style gate in notebook 08.
```

- [ ] **Step 2: Verify the cell renders**

Bullets and bold text should display correctly.

- [ ] **Step 3: Commit**

```bash
git add evals/04_regression_testing.ipynb
git commit -m "feat(evals): add 'What you'll learn' to notebook 04"
```

---

## Task 4: Section 1 — Setup (re-declare the full harness)

**Files:**
- Modify: `evals/04_regression_testing.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 1. Setup — Re-Declare the Harness

This cell copies the shared primitives forward from notebook 03 verbatim so this notebook is fully self-contained. In a real project you'd package these into a module (e.g. `evals/harness.py`) and import from there; here we keep them inline for readability.

**Primitives declared here (identical signatures to nb 03):**
- `Score(key, score, passed, comment="")` — one grader's verdict on one example.
- `Example(input, expected=None, metadata={})` — a single eval case.
- `ExampleResult(example, output, scores)` — the output + all scores for one example.
- `EvalReport(results)` — aggregates a list of `ExampleResult`; exposes `.pass_rate(key=None)`, `.mean_score(key=None)`, `.summary_table()`.
- `run_eval(agent, dataset, graders) -> EvalReport` — runs every grader on every example.
- Graders: `exact_match`, `make_contains`, `make_regex`, `make_schema_grader`, `make_golden_grader`.
```

- [ ] **Step 2: Add the setup code cell**

```python
from __future__ import annotations

import json
import re
from dataclasses import dataclass, field
from typing import Any, Callable

from pydantic import BaseModel, ValidationError


# ---------------------------------------------------------------------------
# Core data types
# ---------------------------------------------------------------------------

@dataclass
class Score:
    """One grader's verdict on one example."""
    key: str
    score: float          # in [0.0, 1.0]
    passed: bool
    comment: str = ""


@dataclass
class Example:
    """A single evaluation case."""
    input: Any
    expected: Any = None
    metadata: dict = field(default_factory=dict)


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


class EvalReport:
    """Aggregates ExampleResults into metrics."""

    def __init__(self, results: list[ExampleResult]) -> None:
        self.results = results

    def pass_rate(self, key: str | None = None) -> float:
        """Fraction of examples where all graders (or the named grader) passed."""
        if not self.results:
            return 0.0
        if key is None:
            passed = sum(
                1 for r in self.results if all(s.passed for s in r.scores)
            )
        else:
            passed = sum(
                1 for r in self.results
                for s in r.scores
                if s.key == key and s.passed
            )
        return passed / len(self.results)

    def mean_score(self, key: str | None = None) -> float:
        """Mean score across all examples (and all graders, or the named grader)."""
        if not self.results:
            return 0.0
        if key is None:
            scores = [s.score for r in self.results for s in r.scores]
        else:
            scores = [
                s.score
                for r in self.results
                for s in r.scores
                if s.key == key
            ]
        return sum(scores) / len(scores) if scores else 0.0

    def summary_table(self) -> str:
        """A readable per-example table with scores and pass/fail."""
        lines = [
            f"{'#':<4} {'Input':<35} {'Mean Score':<12} {'Passed?':<10} Scores",
            "-" * 80,
        ]
        for i, r in enumerate(self.results):
            input_str = str(r.example.input)[:33]
            mean = r.mean_score()
            all_passed = all(s.passed for s in r.scores)
            score_detail = ", ".join(
                f"{s.key}={s.score:.2f}({'P' if s.passed else 'F'})"
                for s in r.scores
            )
            lines.append(
                f"{i:<4} {input_str:<35} {mean:<12.3f} {'PASS' if all_passed else 'FAIL':<10} {score_detail}"
            )
        lines.append("-" * 80)
        lines.append(
            f"     pass_rate={self.pass_rate():.3f}  mean_score={self.mean_score():.3f}"
        )
        return "\n".join(lines)


# ---------------------------------------------------------------------------
# Eval runner
# ---------------------------------------------------------------------------

def run_eval(
    agent: Callable[[Any], Any],
    dataset: list[Example],
    graders: list[Callable[[Example, Any], Score]],
) -> EvalReport:
    """Run every grader on every example and return an EvalReport."""
    results: list[ExampleResult] = []
    for example in dataset:
        output = agent(example.input)
        scores = [grader(example, output) for grader in graders]
        results.append(ExampleResult(example=example, output=output, scores=scores))
    return EvalReport(results)


# ---------------------------------------------------------------------------
# Deterministic graders (attribute form: graders read example.expected)
# ---------------------------------------------------------------------------

def exact_match(example: Example, output: Any) -> Score:
    """Pass iff output == example.expected (string comparison after strip)."""
    expected = str(example.expected).strip()
    actual = str(output).strip()
    passed = actual == expected
    return Score(
        key="exact_match",
        score=1.0 if passed else 0.0,
        passed=passed,
        comment="" if passed else f"expected {expected!r}, got {actual!r}",
    )


def make_contains(substring: str) -> Callable[[Example, Any], Score]:
    """Return a grader that passes iff `substring` appears in the output."""
    def grader(example: Example, output: Any) -> Score:
        passed = substring in str(output)
        return Score(
            key=f"contains({substring!r})",
            score=1.0 if passed else 0.0,
            passed=passed,
            comment="" if passed else f"{substring!r} not found in output",
        )
    return grader


def make_regex(pattern: str) -> Callable[[Example, Any], Score]:
    """Return a grader that passes iff `pattern` matches anywhere in the output."""
    compiled = re.compile(pattern)

    def grader(example: Example, output: Any) -> Score:
        passed = bool(compiled.search(str(output)))
        return Score(
            key=f"regex({pattern!r})",
            score=1.0 if passed else 0.0,
            passed=passed,
            comment="" if passed else f"pattern {pattern!r} did not match output",
        )
    return grader


def make_schema_grader(model: type[BaseModel]) -> Callable[[Example, Any], Score]:
    """Return a grader that passes iff the output parses as `model` (JSON or dict)."""
    def grader(example: Example, output: Any) -> Score:
        try:
            data = json.loads(output) if isinstance(output, str) else output
            model.model_validate(data)
            return Score(key="schema", score=1.0, passed=True)
        except (ValidationError, json.JSONDecodeError, Exception) as exc:
            return Score(key="schema", score=0.0, passed=False, comment=str(exc))
    return grader


def make_golden_grader(golden: str) -> Callable[[Example, Any], Score]:
    """Return a grader that passes iff output matches a stored golden string."""
    def grader(example: Example, output: Any) -> Score:
        passed = str(output).strip() == golden.strip()
        return Score(
            key="golden",
            score=1.0 if passed else 0.0,
            passed=passed,
            comment="" if passed else f"golden mismatch: expected {golden!r}",
        )
    return grader


print("Harness loaded — Score, Example, ExampleResult, EvalReport, run_eval, "
      "exact_match, make_contains, make_regex, make_schema_grader, make_golden_grader")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:
```
Harness loaded — Score, Example, ExampleResult, EvalReport, run_eval, exact_match, make_contains, make_regex, make_schema_grader, make_golden_grader
```

If `pydantic` is not installed: `pip install pydantic`.

- [ ] **Step 4: Commit**

```bash
git add evals/04_regression_testing.ipynb
git commit -m "feat(evals): add harness setup cell to notebook 04"
```

---

## Task 5: Section 2 — A versioned dataset

**Files:**
- Modify: `evals/04_regression_testing.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 2. A Versioned Dataset

An eval dataset is most useful when it's treated as a **checked-in artifact** — a file in your repository that changes via deliberate PRs, not silently in notebooks. Versioning it means you can:

- Diff it in code review when you add or remove examples.
- Guarantee that `baseline` and `candidate` were evaluated on *identical* inputs.
- Track coverage gaps over time.

Here we define the dataset inline (in a real project you'd load it from `evals/data/04_dataset.json`). Each `Example` has a plain-text `input` (a simple arithmetic question or lookup task that a toy agent should handle) and an `expected` answer string that the `exact_match` grader will use. Six examples are enough to show a regression clearly.
```

- [ ] **Step 2: Add the dataset code cell**

```python
dataset: list[Example] = [
    Example(input="What is 2 + 2?",          expected="4"),
    Example(input="What is 10 - 3?",          expected="7"),
    Example(input="What is 6 * 7?",           expected="42"),
    Example(input="What is 100 / 4?",         expected="25"),
    Example(input="What is 2 ** 8?",          expected="256"),
    Example(input="What is the square root of 144?", expected="12"),
]

print(f"Dataset: {len(dataset)} examples")
for i, ex in enumerate(dataset):
    print(f"  [{i}] input={ex.input!r:45s}  expected={ex.expected!r}")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:
```
Dataset: 6 examples
  [0] input='What is 2 + 2?'                           expected='4'
  [1] input='What is 10 - 3?'                          expected='7'
  [2] input='What is 6 * 7?'                           expected='42'
  [3] input='What is 100 / 4?'                         expected='25'
  [4] input='What is 2 ** 8?'                          expected='256'
  [5] input='What is the square root of 144?'          expected='12'
```

- [ ] **Step 4: Commit**

```bash
git add evals/04_regression_testing.ipynb
git commit -m "feat(evals): add versioned dataset to notebook 04"
```

---

## Task 6: Section 3 — Two agent versions

**Files:**
- Modify: `evals/04_regression_testing.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 3. Two Agent Versions

`agent_v1` is the "good" version — it handles all six examples correctly. It parses the arithmetic question with a small lookup table and a regex, so the logic is fully deterministic.

`agent_v2` is a **deliberately regressed** version. It ships a new code path for exponentiation and square roots (maybe a developer "simplified" the parser), but that change introduced a bug: it mishandles `**` (returns the wrong answer for `2 ** 8`) and doesn't recognise the phrase "square root of" at all (returns a fallback string instead of `12`). Everything else is unchanged.

Both agents take a plain string and return a plain string — the same interface the harness expects.
```

- [ ] **Step 2: Add the agents code cell**

```python
import math as _math

def agent_v1(input: str) -> str:
    """v1 — correct on all six dataset examples."""
    s = input.strip().rstrip("?").lower()
    # square root
    m = re.search(r"square root of\s+(\d+)", s)
    if m:
        n = int(m.group(1))
        return str(int(_math.isqrt(n)))
    # exponentiation
    m = re.search(r"(\d+)\s*\*\*\s*(\d+)", s)
    if m:
        return str(int(m.group(1)) ** int(m.group(2)))
    # basic arithmetic: + - * /
    m = re.search(r"(\d+)\s*([+\-*/])\s*(\d+)", s)
    if m:
        a, op, b = int(m.group(1)), m.group(2), int(m.group(3))
        if op == "+": return str(a + b)
        if op == "-": return str(a - b)
        if op == "*": return str(a * b)
        if op == "/": return str(a // b)
    return "unknown"


def agent_v2(input: str) -> str:
    """v2 — REGRESSED on examples [4] and [5].

    A developer 'simplified' the exponentiation path and dropped the
    square-root branch entirely. Both changes introduced bugs:
      - '2 ** 8'  now returns '28' instead of '256'  (mis-parsed as concat)
      - 'square root of 144' now falls through to 'unknown'
    """
    s = input.strip().rstrip("?").lower()
    # BUG: exponentiation regex removed; falls through to basic arithmetic
    # which matches '2' and '8' via the multiply branch — wait, actually the
    # '**' doesn't match '[+\-*/]', so it falls to 'unknown'... let's be
    # concrete about the exact wrong answer:
    if "**" in s:
        # Naive broken path: strip '**' and concatenate the two operands
        parts = s.split("**")
        try:
            left = re.search(r"(\d+)", parts[0]).group(1)
            right = re.search(r"(\d+)", parts[1]).group(1)
            return left + right   # e.g. "2" + "8" = "28"  — WRONG
        except Exception:
            return "unknown"
    # BUG: square-root branch removed entirely
    # (no 're.search(r"square root of ...")' here)
    # basic arithmetic: + - * /
    m = re.search(r"(\d+)\s*([+\-*/])\s*(\d+)", s)
    if m:
        a, op, b = int(m.group(1)), m.group(2), int(m.group(3))
        if op == "+": return str(a + b)
        if op == "-": return str(a - b)
        if op == "*": return str(a * b)
        if op == "/": return str(a // b)
    return "unknown"


# Quick sanity check — all six expected answers from v1
print("agent_v1 outputs:")
for ex in dataset:
    out = agent_v1(ex.input)
    status = "OK" if out == ex.expected else f"WRONG (expected {ex.expected!r})"
    print(f"  {ex.input!r:45s} -> {out!r:6s} {status}")

print()
print("agent_v2 outputs (regressions marked):")
for ex in dataset:
    out = agent_v2(ex.input)
    status = "OK" if out == ex.expected else f"REGRESSION (expected {ex.expected!r})"
    print(f"  {ex.input!r:45s} -> {out!r:10s} {status}")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:
```
agent_v1 outputs:
  'What is 2 + 2?'                           -> '4'    OK
  'What is 10 - 3?'                          -> '7'    OK
  'What is 6 * 7?'                           -> '42'   OK
  'What is 100 / 4?'                         -> '25'   OK
  'What is 2 ** 8?'                          -> '256'  OK
  'What is the square root of 144?'          -> '12'   OK

agent_v2 outputs (regressions marked):
  'What is 2 + 2?'                           -> '4'    OK
  'What is 10 - 3?'                          -> '7'    OK
  'What is 6 * 7?'                           -> '42'   OK
  'What is 100 / 4?'                         -> '25'   OK
  'What is 2 ** 8?'                          -> '28'   REGRESSION (expected '256')
  'What is the square root of 144?'          -> 'unknown' REGRESSION (expected '12')
```

- [ ] **Step 4: Commit**

```bash
git add evals/04_regression_testing.ipynb
git commit -m "feat(evals): add agent_v1 and agent_v2 stubs to notebook 04"
```

---

## Task 7: Section 4 — Run both, diff

**Files:**
- Modify: `evals/04_regression_testing.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 4. Run Both, Diff

We run the harness on each version and then call `compare_reports` to compute a per-example delta table. The key design decision: `compare_reports` uses each example's **mean score across all graders** as the single number to diff. This keeps the delta meaningful whether you run one grader or ten.

`compare_reports(baseline, candidate) -> list[dict]` returns one dict per example:

```python
{
    "input": ...,              # the example's input
    "baseline_score": float,   # mean score across all graders for baseline
    "candidate_score": float,  # mean score across all graders for candidate
    "delta": float,            # candidate_score - baseline_score (negative = regressed)
    "regressed": bool,         # True iff delta < 0
}
```

A row is marked `regressed=True` when `delta < 0` — i.e. the candidate scored strictly lower on this example than the baseline.
```

- [ ] **Step 2: Add the code cell**

```python
# Run the harness on both versions using exact_match as the sole grader.
graders = [exact_match]

report_v1 = run_eval(agent_v1, dataset, graders)
report_v2 = run_eval(agent_v2, dataset, graders)

print("=== agent_v1 report ===")
print(report_v1.summary_table())
print()
print("=== agent_v2 report ===")
print(report_v2.summary_table())
print()


# ---------------------------------------------------------------------------
# compare_reports — new function introduced in this notebook
# ---------------------------------------------------------------------------

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


# Print the delta table
deltas = compare_reports(report_v1, report_v2)

print("=== Delta table: v1 (baseline) vs v2 (candidate) ===")
header = f"{'#':<4} {'Input':<42} {'Baseline':>9} {'Candidate':>10} {'Delta':>7}  Status"
print(header)
print("-" * 85)
for i, row in enumerate(deltas):
    status = "REGRESSED" if row["regressed"] else "ok"
    print(
        f"{i:<4} {str(row['input']):<42} "
        f"{row['baseline_score']:>9.3f} "
        f"{row['candidate_score']:>10.3f} "
        f"{row['delta']:>7.3f}  {status}"
    )
print("-" * 85)
regressions = sum(1 for r in deltas if r["regressed"])
print(f"     {regressions} regression(s) detected out of {len(deltas)} examples")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:
```
=== agent_v1 report ===
#    Input                               Mean Score   Passed?    Scores
--------------------------------------------------------------------------------
0    What is 2 + 2?                      1.000        PASS       exact_match=1.00(P)
1    What is 10 - 3?                     1.000        PASS       exact_match=1.00(P)
2    What is 6 * 7?                      1.000        PASS       exact_match=1.00(P)
3    What is 100 / 4?                    1.000        PASS       exact_match=1.00(P)
4    What is 2 ** 8?                     1.000        PASS       exact_match=1.00(P)
5    What is the square root of 144?     1.000        PASS       exact_match=1.00(P)
--------------------------------------------------------------------------------
     pass_rate=1.000  mean_score=1.000

=== agent_v2 report ===
#    Input                               Mean Score   Passed?    Scores
--------------------------------------------------------------------------------
0    What is 2 + 2?                      1.000        PASS       exact_match=1.00(P)
1    What is 10 - 3?                     1.000        PASS       exact_match=1.00(P)
2    What is 6 * 7?                      1.000        PASS       exact_match=1.00(P)
3    What is 100 / 4?                    1.000        PASS       exact_match=1.00(P)
4    What is 2 ** 8?                     0.000        FAIL       exact_match=0.00(F)
5    What is the square root of 144?     0.000        FAIL       exact_match=0.00(F)
--------------------------------------------------------------------------------
     pass_rate=0.667  mean_score=0.667

=== Delta table: v1 (baseline) vs v2 (candidate) ===
#    Input                                      Baseline  Candidate   Delta  Status
-------------------------------------------------------------------------------------
0    What is 2 + 2?                                1.000      1.000   0.000  ok
1    What is 10 - 3?                               1.000      1.000   0.000  ok
2    What is 6 * 7?                                1.000      1.000   0.000  ok
3    What is 100 / 4?                              1.000      1.000   0.000  ok
4    What is 2 ** 8?                               1.000      0.000  -1.000  REGRESSED
5    What is the square root of 144?               1.000      0.000  -1.000  REGRESSED
-------------------------------------------------------------------------------------
     2 regression(s) detected out of 6 examples
```

- [ ] **Step 4: Commit**

```bash
git add evals/04_regression_testing.ipynb
git commit -m "feat(evals): add compare_reports and delta table to notebook 04"
```

---

## Task 8: Section 5 — The gate

**Files:**
- Modify: `evals/04_regression_testing.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 5. The Gate

`regression_gate` wraps `compare_reports` in an explicit, auditable pass/fail rule. The rule has three components, all of which must hold for the gate to return `True` (PASS):

1. **Aggregate floor:** `candidate.mean_score() >= min_mean` — the candidate must score at least `min_mean` overall (default `0.0`, meaning no floor; in practice you'd set this to your current baseline, e.g. `0.90`).
2. **No aggregate regression:** `candidate.mean_score() >= baseline.mean_score() - epsilon` — the candidate's mean score must not drop more than `epsilon` (1e-9, effectively zero) below the baseline's mean score.
3. **No per-example regression (unless explicitly allowed):** if `allow_regressions=False` (the default), then every per-example delta must be `>= 0` — no example is allowed to score worse in the candidate than in the baseline.

All three conditions must hold. Failing any one returns `False` (FAIL).

This gate is the deterministic precursor to the full CI gate in notebook 08, which will additionally include LLM-judge graders and LangSmith experiments.
```

- [ ] **Step 2: Add the gate code cell**

```python
_EPSILON = 1e-9  # tolerance for floating-point aggregate comparisons


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

    # Condition 1: no aggregate regression
    if cand_mean < base_mean - _EPSILON:
        return False

    # Condition 2: absolute floor
    if cand_mean < min_mean:
        return False

    # Condition 3: no per-example regression (unless allow_regressions=True)
    if not allow_regressions:
        deltas = compare_reports(baseline, candidate)
        if any(row["regressed"] for row in deltas):
            return False

    return True


def print_gate_result(label: str, result: bool) -> None:
    banner = "PASS" if result else "FAIL"
    border = "=" * 50
    print(border)
    print(f"  Gate result for {label}: {banner}")
    print(border)


# --- v1 vs v1: no change — should PASS ---
result_same = regression_gate(report_v1, report_v1)
print_gate_result("v1 vs v1 (no change)", result_same)
print(f"  baseline mean={report_v1.mean_score():.3f}  candidate mean={report_v1.mean_score():.3f}")
print()

# --- v1 vs v2: 2 regressions — should FAIL ---
result_regressed = regression_gate(report_v1, report_v2)
print_gate_result("v1 (baseline) vs v2 (candidate)", result_regressed)
print(f"  baseline mean={report_v1.mean_score():.3f}  candidate mean={report_v2.mean_score():.3f}")
print(f"  regressions: {sum(1 for r in compare_reports(report_v1, report_v2) if r['regressed'])}")
print()

# --- With allow_regressions=True: condition 3 is skipped.
#     v2's mean (0.667) < v1's mean (1.000), so condition 1 still fails. ---
result_allow = regression_gate(report_v1, report_v2, allow_regressions=True)
print_gate_result("v1 vs v2 (allow_regressions=True)", result_allow)
print(f"  Note: still FAIL because candidate mean ({report_v2.mean_score():.3f}) "
      f"< baseline mean ({report_v1.mean_score():.3f}) - epsilon")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:
```
==================================================
  Gate result for v1 vs v1 (no change): PASS
==================================================
  baseline mean=1.000  candidate mean=1.000

==================================================
  Gate result for v1 (baseline) vs v2 (candidate): FAIL
==================================================
  baseline mean=1.000  candidate mean=0.667
  regressions: 2

==================================================
  Gate result for v1 vs v2 (allow_regressions=True): FAIL
==================================================
  Note: still FAIL because candidate mean (0.667) < baseline mean (1.000) - epsilon
```

- [ ] **Step 4: Commit**

```bash
git add evals/04_regression_testing.ipynb
git commit -m "feat(evals): add regression_gate with PASS/FAIL banner to notebook 04"
```

---

## Task 9: `### Try it` — fix v2 and re-run

**Files:**
- Modify: `evals/04_regression_testing.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
### Try it

Fix `agent_v2`'s two bugs and re-run the gate. You should see it flip from FAIL to PASS.

The fix: restore the exponentiation regex (use `**` before the basic arithmetic match, same as v1) and restore the square-root branch.

Run the cell below — it defines `agent_v2_fixed`, evaluates it, and runs the gate.
```

- [ ] **Step 2: Add the code cell**

```python
def agent_v2_fixed(input: str) -> str:
    """v2 with both regressions fixed — identical behaviour to v1."""
    s = input.strip().rstrip("?").lower()
    # FIX 1: square root branch restored
    m = re.search(r"square root of\s+(\d+)", s)
    if m:
        n = int(m.group(1))
        return str(int(_math.isqrt(n)))
    # FIX 2: exponentiation regex restored (before basic-arithmetic match)
    m = re.search(r"(\d+)\s*\*\*\s*(\d+)", s)
    if m:
        return str(int(m.group(1)) ** int(m.group(2)))
    # basic arithmetic: + - * /
    m = re.search(r"(\d+)\s*([+\-*/])\s*(\d+)", s)
    if m:
        a, op, b = int(m.group(1)), m.group(2), int(m.group(3))
        if op == "+": return str(a + b)
        if op == "-": return str(a - b)
        if op == "*": return str(a * b)
        if op == "/": return str(a // b)
    return "unknown"


report_v2_fixed = run_eval(agent_v2_fixed, dataset, graders)

print("=== agent_v2_fixed report ===")
print(report_v2_fixed.summary_table())
print()

result_fixed = regression_gate(report_v1, report_v2_fixed)
print_gate_result("v1 (baseline) vs v2_fixed (candidate)", result_fixed)
print(f"  baseline mean={report_v1.mean_score():.3f}  candidate mean={report_v2_fixed.mean_score():.3f}")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:
```
=== agent_v2_fixed report ===
#    Input                               Mean Score   Passed?    Scores
--------------------------------------------------------------------------------
0    What is 2 + 2?                      1.000        PASS       exact_match=1.00(P)
1    What is 10 - 3?                     1.000        PASS       exact_match=1.00(P)
2    What is 6 * 7?                      1.000        PASS       exact_match=1.00(P)
3    What is 100 / 4?                    1.000        PASS       exact_match=1.00(P)
4    What is 2 ** 8?                     1.000        PASS       exact_match=1.00(P)
5    What is the square root of 144?     1.000        PASS       exact_match=1.00(P)
--------------------------------------------------------------------------------
     pass_rate=1.000  mean_score=1.000

==================================================
  Gate result for v1 (baseline) vs v2_fixed (candidate): PASS
==================================================
  baseline mean=1.000  candidate mean=1.000
```

- [ ] **Step 4: Commit**

```bash
git add evals/04_regression_testing.ipynb
git commit -m "feat(evals): add Try it fix cell to notebook 04"
```

---

## Task 10: Gotcha callout

**Files:**
- Modify: `evals/04_regression_testing.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Add the Gotcha markdown cell**

```markdown
> **Gotcha: flaky graders make gates meaningless.**
>
> A regression gate is only as reliable as the graders feeding it. If a grader returns different scores for the same (example, output) pair across runs — because it calls an LLM without a fixed seed, because it parses a timestamp that changes, or because it relies on external state — then `regression_gate` can flip between PASS and FAIL for *no change in agent behaviour*. That's worse than having no gate: it trains you to ignore failures.
>
> This is the reason notebooks 01–04 stay entirely deterministic. When we introduce the LLM-as-judge in notebook 05, we'll address its non-determinism explicitly — temperature, fixed seeds where the API allows, and the discipline of validating the judge itself before wiring it into a gate.
```

- [ ] **Step 2: Verify the blockquote renders**

The `> **Gotcha:**` prefix should render as a blockquote with bold text.

- [ ] **Step 3: Commit**

```bash
git add evals/04_regression_testing.ipynb
git commit -m "feat(evals): add flaky-grader Gotcha callout to notebook 04"
```

---

## Task 11: Closing recap and "What's missing" teaser

**Files:**
- Modify: `evals/04_regression_testing.ipynb` (add 2 markdown cells)

- [ ] **Step 1: Add the "What you just learned" markdown cell**

```markdown
## What you just learned

- **Versioned datasets** are the foundation of regression testing — if you don't commit the dataset, you can't audit why a gate passed or failed last Tuesday.
- **`compare_reports(baseline, candidate)`** produces a per-example delta table keyed on mean score per example; a negative delta marks a regression.
- **`regression_gate(baseline, candidate, min_mean, allow_regressions)`** encodes a three-part rule: no aggregate regression, absolute floor, no per-example regression (unless opted out). All three must hold for PASS.
- Fixing `agent_v2`'s bugs caused the gate to flip from FAIL to PASS — this is the intended workflow: gate fails on CI, developer fixes the bug, gate passes, ship.
- **Deterministic graders are a prerequisite** for a trustworthy gate. Flaky graders produce a gate you can't rely on.
- This gate is fully offline (no API key, no server). Notebook 08 will wire the same pattern into a production-grade CI-style gate with LLM-judge graders and LangSmith experiments included.
```

- [ ] **Step 2: Add the "What's missing" markdown cell**

```markdown
## What's missing

The graders we've used so far — exact match, contains, regex, schema, golden — can only score outputs that have a *known correct answer*. Ask your agent to *summarise* a document, *explain* a concept in plain language, or *rewrite* a sentence more politely: none of those have a single right answer, and `exact_match` will fail everything.

This is the boundary deterministic graders cannot cross. **Notebook 05 — `05_llm_as_judge.ipynb`** — crosses it by using a second LLM as a grader: it reads the output and a rubric, then returns a structured score and a rationale. That's the first notebook in the series that requires an API key, and it's where evaluation gets genuinely powerful — and genuinely tricky. The LLM judge introduces its own failure modes (position bias, verbosity preference, self-preference) that you have to understand and measure before trusting a gate built on it.
```

- [ ] **Step 3: Verify both cells render**

Bold function names and the series cross-reference to notebook 05 should display correctly.

- [ ] **Step 4: Commit**

```bash
git add evals/04_regression_testing.ipynb
git commit -m "feat(evals): add closing recap and notebook-05 teaser to notebook 04"
```

---

## Task 12: End-to-end verification in a fresh kernel

**Files:** none (verification only).

- [ ] **Step 1: Execute the notebook top-to-bottom in a clean kernel**

```bash
jupyter nbconvert --to notebook --execute --inplace evals/04_regression_testing.ipynb
```

Expected: succeeds with no errors. The notebook is rewritten with embedded outputs from the fresh run.

- [ ] **Step 2: Verify expected outputs in the executed notebook**

The output cells should contain, in order:

1. Section 1 setup confirmation:
   ```
   Harness loaded — Score, Example, ExampleResult, EvalReport, run_eval, exact_match, make_contains, make_regex, make_schema_grader, make_golden_grader
   ```

2. Section 2 dataset listing: 6 examples, `[0]` through `[5]`, each with the correct `expected` value.

3. Section 3 agent sanity check:
   - `agent_v1`: all 6 lines show `OK`.
   - `agent_v2`: lines `[0]`–`[3]` show `OK`, lines `[4]` and `[5]` show `REGRESSION`.

4. Section 4 delta table:
   - v1 report: `pass_rate=1.000  mean_score=1.000`.
   - v2 report: `pass_rate=0.667  mean_score=0.667`.
   - Delta table: `2 regression(s) detected out of 6 examples`; rows `[4]` and `[5]` show `REGRESSED`.

5. Section 5 gate results:
   ```
   Gate result for v1 vs v1 (no change): PASS
   Gate result for v1 (baseline) vs v2 (candidate): FAIL
   Gate result for v1 vs v2 (allow_regressions=True): FAIL
   ```

6. Try it gate result:
   ```
   Gate result for v1 (baseline) vs v2_fixed (candidate): PASS
   ```
   with `mean_score=1.000` for both baseline and candidate.

No cell raises an unhandled exception. If `nbconvert` reports `ModuleNotFoundError: No module named 'pydantic'`, install it first: `pip install pydantic`.

- [ ] **Step 3: Commit the clean run**

```bash
git add evals/04_regression_testing.ipynb
git commit -m "chore(evals): commit clean fresh-kernel run of notebook 04"
```

---

## Done

After Task 12 passes, notebook 04 is complete. The next plan to write is `2026-05-24-evals-notebook-05-llm-as-judge.md`, which introduces the LLM as a grader for open-ended outputs — the first notebook in the series that requires an `OPENAI_API_KEY`, and where rubric grading, pairwise comparison, and judge-bias failure modes are taught.
