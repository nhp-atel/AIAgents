# Evals Notebook 03 — `03_building_an_eval_harness.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the third notebook in the Evals & Observability learning series — generalize the ad-hoc graders from notebook 02 into a **reusable eval harness** by introducing three formal dataclasses (`Example`, `ExampleResult`, `EvalReport`) and a `run_eval(agent, dataset, graders) -> EvalReport` runner. The notebook re-declares `Score` and the deterministic graders from nb 02 in attribute form (with a Gotcha calling out the `example.expected` change), defines a small sentiment-classification dataset, runs a deterministic stub agent through the harness, and reads the resulting report — all with no API key.

**Architecture:** A single self-contained Jupyter notebook. No shared module; every name re-declared inline. The agent under eval is a pure-Python stub; examples are fully in-process. The `dataclasses` module and `pydantic` (already used in nb 02) are the only non-stdlib dependencies.

**Tech Stack:** Python 3.11+, Jupyter, `dataclasses` (stdlib), `pydantic` v2, `re` (stdlib), `json` (stdlib). No API key, no network, no external services.

**Companion spec:** `docs/superpowers/specs/2026-05-24-evals-observability-learning-series-design.md` (notebook 03 section).

**Folder:** `evals/` already exists. Task 1 scaffolds the empty notebook JSON.

---

## File Structure

- **Create:** `evals/03_building_an_eval_harness.ipynb` — the entire notebook, self-contained.
- **Modify:** none.
- **No separate test files** — verification is the notebook running top-to-bottom in a fresh kernel (Task 9).

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown)
2. **"What you'll learn"** (markdown bullets)
3. **Section 1: Setup** (markdown + 1 code cell — re-declare `Score` and all deterministic graders from nb 02 in attribute form; Gotcha on `example.expected`)
4. **Section 2: The data model** (markdown + 1 code cell — define `Example`; build `dataset: list[Example]` with 5 sentiment-classification cases)
5. **Section 3: A toy agent to evaluate** (markdown + 1 code cell — define `classify_agent(input: dict) -> str`; show it gets one wrong deliberately)
6. **Section 4: The runner** (markdown + 2 code cells — define `ExampleResult`, `EvalReport` with `pass_rate`/`mean_score`/`summary_table`, and `run_eval`; run it to produce a report)
7. **Section 5: Reading the report** (markdown + 1 code cell — print `summary_table()`, `pass_rate()`, `pass_rate("exact_match")`, `mean_score()`; per-example breakdown loop; explain pass rate vs mean score)
8. **Section 6: Try it** (markdown + 1 code cell — swap in a buggier agent and re-run to watch metrics move)
9. **"What you just learned"** (markdown)
10. **"What's missing"** (markdown teaser → notebook 04 `04_regression_testing.ipynb`)

---

## Task 1: Scaffold the empty notebook

**Files:**
- Create: `evals/03_building_an_eval_harness.ipynb`

- [ ] **Step 1: Verify the `evals/` folder exists**

```bash
ls -la evals/
```

Expected: folder exists, contains at least `01_why_agents_are_hard_to_test.ipynb` and `02_assertions_and_golden_outputs.ipynb`.

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

Save as `evals/03_building_an_eval_harness.ipynb`.

- [ ] **Step 3: Verify the file is valid JSON**

```bash
python -c "import json; json.load(open('evals/03_building_an_eval_harness.ipynb'))"
ls -la evals/03_building_an_eval_harness.ipynb
```

Expected: no exception, file non-zero size.

- [ ] **Step 4: Commit**

```bash
git add evals/03_building_an_eval_harness.ipynb
git commit -m "feat(evals): scaffold 03_building_an_eval_harness.ipynb"
```

---

## Task 2: Add title and "Why this notebook exists"

**Files:**
- Modify: `evals/03_building_an_eval_harness.ipynb` (add cell 1)

- [ ] **Step 1: Add a markdown cell with title and motivation**

Cell content (markdown):

```markdown
# 03 — Building an Eval Harness

## Why this notebook exists

In **`02_assertions_and_golden_outputs.ipynb`** we built five grader functions — exact match, contains, regex, schema validation, golden outputs — each following the same shape: take an example and an agent output, return a score. That was a useful pattern, but we applied each grader by hand, one call at a time, against a single example. When you have a dataset of dozens or hundreds of examples and several graders to apply, doing that by hand doesn't scale.

This notebook generalizes the pattern into a **reusable eval harness**: a small set of dataclasses that model the inputs (`Example`), the per-example results (`ExampleResult`), and the aggregated report (`EvalReport`), plus a `run_eval(agent, dataset, graders)` function that wires them together. By the end you'll be able to add a new grader or a new dataset and re-run your entire eval suite in one call.

No API key needed — the agent is a deterministic Python stub and all graders run in-process.
```

- [ ] **Step 2: Verify the cell renders**

Open the notebook in Jupyter/VS Code and confirm the cell renders as markdown with correct heading levels.

- [ ] **Step 3: Commit**

```bash
git add evals/03_building_an_eval_harness.ipynb
git commit -m "feat(evals): add intro markdown to notebook 03"
```

---

## Task 3: Add "What you'll learn"

**Files:**
- Modify: `evals/03_building_an_eval_harness.ipynb` (add cell 2)

- [ ] **Step 1: Add a markdown cell**

Cell content (markdown):

```markdown
## What you'll learn

- How to model an evaluation as three composable pieces: an `Example` (input + expected), an `ExampleResult` (example + output + scores), and an `EvalReport` (aggregated results with metrics).
- How to write `run_eval(agent, dataset, graders) -> EvalReport` — a runner that applies every grader to every example and collects structured results.
- How to read an `EvalReport`: `pass_rate()`, `mean_score()`, per-grader breakdowns, and `summary_table()`.
- Why pass rate and mean score tell different stories about the same run.
- A small but real Gotcha: graders from notebook 02 read `example["expected"]` (dict form). Now that `Example` is a dataclass, they must read `example.expected` (attribute form). The graders in this notebook are re-declared to match.
```

- [ ] **Step 2: Verify the cell renders**

The bullets should render properly, including the Gotcha bullet.

- [ ] **Step 3: Commit**

```bash
git add evals/03_building_an_eval_harness.ipynb
git commit -m "feat(evals): add 'What you'll learn' to notebook 03"
```

---

## Task 4: Section 1 — Setup (re-declare Score and graders in attribute form)

**Files:**
- Modify: `evals/03_building_an_eval_harness.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 1. Setup

We re-declare `Score` and every deterministic grader from notebook 02 here, so this notebook is fully self-contained and runs in a fresh kernel without importing from another file.

**One small change from notebook 02:** the graders there read `example["expected"]` because examples were plain dicts. In this notebook, `Example` is a dataclass, so the graders read `example.expected` (attribute access). Everything else is identical — same function names, same signatures, same `Score` return type.

> **Gotcha:** If you copy a grader verbatim from notebook 02 and forget to change `example["expected"]` to `example.expected`, you'll get a `TypeError: 'Example' object is not subscriptable`. The fix is one character: replace `[` with `.` and drop the `"]"`. This is the only breaking change between the two notebooks.
```

- [ ] **Step 2: Add the setup code cell**

```python
from __future__ import annotations

import re
import json
from dataclasses import dataclass, field
from typing import Any, Callable

from pydantic import BaseModel, ValidationError


# ---------------------------------------------------------------------------
# Score — the atomic unit every grader returns
# ---------------------------------------------------------------------------

@dataclass
class Score:
    key: str          # grader identity, e.g. "exact_match"
    score: float      # numeric value in [0, 1]
    passed: bool      # True if this grader considers the output acceptable
    comment: str = "" # optional human-readable note


# ---------------------------------------------------------------------------
# Deterministic graders — re-declared from notebook 02, now reading
# example.expected (attribute form) instead of example["expected"] (dict form)
# ---------------------------------------------------------------------------

def exact_match(example, output) -> Score:
    """Pass if str(output) == str(example.expected), case-insensitive strip."""
    expected = str(example.expected).strip().lower()
    actual = str(output).strip().lower()
    passed = expected == actual
    return Score(
        key="exact_match",
        score=1.0 if passed else 0.0,
        passed=passed,
        comment=f"expected={expected!r}, got={actual!r}",
    )


def make_contains(substring: str, key: str = "contains") -> Callable:
    """Return a grader that passes when `substring` appears in the output."""
    def grader(example, output) -> Score:
        passed = substring.lower() in str(output).lower()
        return Score(
            key=key,
            score=1.0 if passed else 0.0,
            passed=passed,
            comment=f"looked for {substring!r}",
        )
    return grader


def make_regex(pattern: str, key: str = "regex") -> Callable:
    """Return a grader that passes when `pattern` matches anywhere in the output."""
    compiled = re.compile(pattern, re.IGNORECASE)
    def grader(example, output) -> Score:
        passed = bool(compiled.search(str(output)))
        return Score(
            key=key,
            score=1.0 if passed else 0.0,
            passed=passed,
            comment=f"pattern={pattern!r}",
        )
    return grader


def make_schema_grader(model: type[BaseModel], key: str = "valid_schema") -> Callable:
    """Return a grader that passes when the output parses into `model`."""
    def grader(example, output) -> Score:
        try:
            if isinstance(output, str):
                data = json.loads(output)
            else:
                data = output
            model.model_validate(data)
            return Score(key=key, score=1.0, passed=True, comment="schema valid")
        except (ValidationError, json.JSONDecodeError, TypeError) as exc:
            return Score(key=key, score=0.0, passed=False, comment=str(exc))
    return grader


def make_golden_grader(path: str, key: str = "golden") -> Callable:
    """Return a grader that passes when output matches the text stored at `path`."""
    import pathlib
    golden = pathlib.Path(path).read_text().strip()
    def grader(example, output) -> Score:
        actual = str(output).strip()
        passed = actual == golden
        return Score(
            key=key,
            score=1.0 if passed else 0.0,
            passed=passed,
            comment=f"golden={golden[:40]!r}",
        )
    return grader


print("Setup OK — Score and graders declared (attribute form)")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Setup OK — Score and graders declared (attribute form)
```

If `from pydantic import BaseModel, ValidationError` fails: `pip install pydantic`.

- [ ] **Step 4: Commit**

```bash
git add evals/03_building_an_eval_harness.ipynb
git commit -m "feat(evals): add setup cell with Score and graders (attribute form) to notebook 03"
```

---

## Task 5: Section 2 — The data model and dataset

**Files:**
- Modify: `evals/03_building_an_eval_harness.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 2. The Data Model

An **example** pairs an `input` (a dict the agent receives) with an optional `expected` value (whatever the correct output should be). A `metadata` dict holds anything else — source, difficulty tier, tags — that we might want to filter on later without changing the core fields.

```python
@dataclass
class Example:
    input: dict
    expected: object | None = None
    metadata: dict = field(default_factory=dict)
```

A **dataset** is just a `list[Example]`. No framework required — it's a Python list you can build inline, load from a JSON file, or pull from a database.

We'll use a small sentiment-classification dataset: five product reviews with expected labels (`"positive"`, `"negative"`, or `"neutral"`). One example is deliberately tricky (a sarcastic review) so the stub agent we build in the next section gets it wrong — making the metrics more interesting than a trivial 100 % pass rate.
```

- [ ] **Step 2: Add the data model code cell**

```python
# ---------------------------------------------------------------------------
# Example dataclass
# ---------------------------------------------------------------------------

@dataclass
class Example:
    input: dict
    expected: object | None = None
    metadata: dict = field(default_factory=dict)


# ---------------------------------------------------------------------------
# Dataset: 5 sentiment-classification cases
# ---------------------------------------------------------------------------

dataset: list[Example] = [
    Example(
        input={"review": "This product is absolutely fantastic! Best purchase I've made this year."},
        expected="positive",
        metadata={"id": "s01", "difficulty": "easy"},
    ),
    Example(
        input={"review": "Terrible quality. Broke after two days and support never responded."},
        expected="negative",
        metadata={"id": "s02", "difficulty": "easy"},
    ),
    Example(
        input={"review": "It arrived on time and the packaging was fine."},
        expected="neutral",
        metadata={"id": "s03", "difficulty": "medium"},
    ),
    Example(
        input={"review": "Oh great, another product that stops working the moment the warranty expires."},
        expected="negative",
        metadata={"id": "s04", "difficulty": "hard", "note": "sarcastic — stub gets this wrong"},
    ),
    Example(
        input={"review": "Exactly what I needed. Simple, effective, good value."},
        expected="positive",
        metadata={"id": "s05", "difficulty": "easy"},
    ),
]

print(f"Dataset: {len(dataset)} examples")
for ex in dataset:
    preview = ex.input["review"][:55].replace("\n", " ")
    print(f"  [{ex.metadata['id']}] expected={ex.expected!r:10s}  {preview!r}")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Dataset: 5 examples
  [s01] expected='positive'   'This product is absolutely fantastic! Best purchase I'
  [s02] expected='negative'   'Terrible quality. Broke after two days and support nev'
  [s03] expected='neutral'    'It arrived on time and the packaging was fine.'
  [s04] expected='negative'   'Oh great, another product that stops working the moment'
  [s05] expected='positive'   'Exactly what I needed. Simple, effective, good value.'
```

- [ ] **Step 4: Commit**

```bash
git add evals/03_building_an_eval_harness.ipynb
git commit -m "feat(evals): add Example dataclass and sentiment dataset to notebook 03"
```

---

## Task 6: Section 3 — A toy agent to evaluate

**Files:**
- Modify: `evals/03_building_an_eval_harness.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 3. A Toy Agent to Evaluate

A real sentiment classifier would call an LLM. For this notebook we use a deterministic stub: a function that classifies reviews using keyword matching. It's correct on the four straightforward examples but returns `"positive"` for the sarcastic review (`s04`), which should be `"negative"`.

That one wrong answer is intentional — if the stub were perfect, every metric would be 100 % and we'd learn nothing about reading a report.

The harness doesn't care how the agent works internally. It only requires that `classify_agent(input: dict) -> str` — it takes the same `input` dict that `Example.input` holds and returns a label string.
```

- [ ] **Step 2: Add the agent code cell**

```python
def classify_agent(input: dict) -> str:
    """Deterministic stub sentiment classifier using keyword matching.

    Correct on s01, s02, s03, s05.
    Intentionally wrong on s04 (sarcastic review) — returns 'positive'.
    """
    review = input["review"].lower()

    negative_words = {"terrible", "broke", "awful", "horrible", "worst", "bad", "poor"}
    positive_words = {"fantastic", "great", "excellent", "best", "needed", "effective"}
    neutral_words  = {"arrived", "time", "packaging", "fine"}

    neg_hits = sum(1 for w in negative_words if w in review)
    pos_hits = sum(1 for w in positive_words if w in review)
    neu_hits = sum(1 for w in neutral_words  if w in review)

    if neg_hits > pos_hits and neg_hits > neu_hits:
        return "negative"
    elif neu_hits > pos_hits and neu_hits >= neg_hits:
        return "neutral"
    else:
        return "positive"


# Quick smoke test so we can see the one wrong prediction before running the harness
print("Smoke test — classify_agent predictions:")
for ex in dataset:
    prediction = classify_agent(ex.input)
    correct = "✓" if prediction == ex.expected else "✗"
    print(f"  [{ex.metadata['id']}] {correct} predicted={prediction!r:10s} expected={ex.expected!r}")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Smoke test — classify_agent predictions:
  [s01] ✓ predicted='positive'  expected='positive'
  [s02] ✓ predicted='negative'  expected='negative'
  [s03] ✓ predicted='neutral'   expected='neutral'
  [s04] ✗ predicted='positive'  expected='negative'
  [s05] ✓ predicted='positive'  expected='positive'
```

- [ ] **Step 4: Commit**

```bash
git add evals/03_building_an_eval_harness.ipynb
git commit -m "feat(evals): add classify_agent stub to notebook 03"
```

---

## Task 7: Section 4 — The runner

**Files:**
- Modify: `evals/03_building_an_eval_harness.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 4. The Runner

Three more pieces complete the harness:

- **`ExampleResult`** — holds one example, the agent's output for it, and the list of `Score`s every grader produced.
- **`EvalReport`** — wraps the full list of `ExampleResult`s and exposes three read methods: `pass_rate(key=None)`, `mean_score(key=None)`, and `summary_table()`.
- **`run_eval(agent, dataset, graders) -> EvalReport`** — loops over the dataset, calls `agent(example.input)` for each, applies every grader, and assembles the report.

The function signature `run_eval(agent, dataset, graders)` is the core abstraction of the series: you plug in any callable agent, any list of examples, and any list of grader functions, and you get back a structured report. Changing the agent, dataset, or grader list is one argument swap.
```

- [ ] **Step 2: Add the dataclasses + run_eval code cell**

```python
# ---------------------------------------------------------------------------
# ExampleResult and EvalReport
# ---------------------------------------------------------------------------

@dataclass
class ExampleResult:
    example: Example
    output: object
    scores: list  # list[Score]


@dataclass
class EvalReport:
    results: list  # list[ExampleResult]

    def _scores(self, key=None):
        """Yield all Score objects, optionally filtered to a single grader key."""
        for r in self.results:
            for s in r.scores:
                if key is None or s.key == key:
                    yield s

    def pass_rate(self, key=None) -> float:
        """Fraction of scores (optionally filtered to grader `key`) with passed=True."""
        scores = list(self._scores(key))
        if not scores:
            return 0.0
        return sum(1 for s in scores if s.passed) / len(scores)

    def mean_score(self, key=None) -> float:
        """Mean of .score values (optionally filtered to grader `key`)."""
        scores = list(self._scores(key))
        if not scores:
            return 0.0
        return sum(s.score for s in scores) / len(scores)

    def summary_table(self) -> str:
        """Printable text table: one row per grader key with pass_rate + mean_score."""
        # Collect unique grader keys in insertion order
        seen: dict[str, None] = {}
        for r in self.results:
            for s in r.scores:
                seen[s.key] = None
        keys = list(seen)

        header = f"{'Grader':<20}  {'Pass rate':>10}  {'Mean score':>10}"
        sep    = "-" * len(header)
        rows   = [header, sep]
        for k in keys:
            pr = self.pass_rate(k)
            ms = self.mean_score(k)
            rows.append(f"{k:<20}  {pr:>10.1%}  {ms:>10.3f}")
        rows.append(sep)
        overall_pr = self.pass_rate()
        overall_ms = self.mean_score()
        rows.append(f"{'OVERALL':<20}  {overall_pr:>10.1%}  {overall_ms:>10.3f}")
        return "\n".join(rows)


# ---------------------------------------------------------------------------
# run_eval
# ---------------------------------------------------------------------------

def run_eval(agent, dataset: list, graders: list) -> EvalReport:
    """Run every example in `dataset` through `agent`, apply every grader.

    Args:
        agent:   callable (input: dict) -> str | dict
        dataset: list[Example]
        graders: list of grader callables, each (example, output) -> Score

    Returns:
        EvalReport containing one ExampleResult per example.
    """
    results = []
    for example in dataset:
        output = agent(example.input)
        scores = [g(example, output) for g in graders]
        results.append(ExampleResult(example=example, output=output, scores=scores))
    return EvalReport(results=results)
```

- [ ] **Step 3: Add the run cell that calls run_eval and stores the report**

```python
# Grader set for this run
graders = [
    exact_match,
    make_contains("positive", key="contains_positive"),
]

report = run_eval(
    agent=classify_agent,
    dataset=dataset,
    graders=graders,
)

print(f"EvalReport produced: {len(report.results)} ExampleResult(s), "
      f"{sum(len(r.scores) for r in report.results)} Score(s) total")
```

- [ ] **Step 4: Run both cells and verify**

Expected output from the run cell:

```
EvalReport produced: 5 ExampleResult(s), 10 Score(s) total
```

- [ ] **Step 5: Commit**

```bash
git add evals/03_building_an_eval_harness.ipynb
git commit -m "feat(evals): add ExampleResult, EvalReport, run_eval to notebook 03"
```

---

## Task 8: Section 5 — Reading the report

**Files:**
- Modify: `evals/03_building_an_eval_harness.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 5. Reading the Report

An `EvalReport` exposes three views of the same underlying data:

- `summary_table()` — one row per grader, side by side: a quick dashboard.
- `pass_rate(key=None)` — what fraction of outputs satisfied the pass criterion. When called with a grader key, scoped to that grader only.
- `mean_score(key=None)` — the numeric average of `Score.score` values. For binary graders (0/1) this equals pass rate, but graders that return partial scores (e.g. 0.5 for a fuzzy match) show a different picture here.

A per-example breakdown loop lets you see *which* examples failed — essential for debugging the agent rather than just knowing a single aggregate number.
```

- [ ] **Step 2: Add the report-reading code cell**

```python
# Summary table
print("=== Summary table ===")
print(report.summary_table())
print()

# Aggregate metrics
print(f"Overall  pass_rate : {report.pass_rate():.1%}")
print(f"exact_match  pass_rate : {report.pass_rate('exact_match'):.1%}")
print(f"contains_positive  pass_rate : {report.pass_rate('contains_positive'):.1%}")
print(f"Overall  mean_score: {report.mean_score():.3f}")
print()

# Per-example breakdown
print("=== Per-example breakdown ===")
for r in report.results:
    ex_id = r.example.metadata.get("id", "?")
    label_line = f"[{ex_id}] output={r.output!r:12s} expected={r.example.expected!r}"
    score_parts = []
    for s in r.scores:
        mark = "✓" if s.passed else "✗"
        score_parts.append(f"{s.key}={mark}")
    print(f"  {label_line}  |  {', '.join(score_parts)}")

print()
print("Pass rate tells you the fraction of outputs that cleared the bar.")
print("Mean score is the same number for binary graders, but differs once")
print("graders return partial credit (e.g. a fuzzy similarity score in [0, 1]).")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
=== Summary table ===
Grader                 Pass rate  Mean score
----------------------------------------------
exact_match               80.0%       0.800
contains_positive         60.0%       0.600
----------------------------------------------
OVERALL                   70.0%       0.700

Overall  pass_rate : 70.0%
exact_match  pass_rate : 80.0%
contains_positive  pass_rate : 60.0%
Overall  mean_score: 0.700

=== Per-example breakdown ===
  [s01] output='positive'  expected='positive'  |  exact_match=✓, contains_positive=✓
  [s02] output='negative'  expected='negative'  |  exact_match=✓, contains_positive=✗
  [s03] output='neutral'   expected='neutral'   |  exact_match=✓, contains_positive=✗
  [s04] output='positive'  expected='negative'  |  exact_match=✗, contains_positive=✓
  [s05] output='positive'  expected='positive'  |  exact_match=✓, contains_positive=✓

Pass rate tells you the fraction of outputs that cleared the bar.
Mean score is the same number for binary graders, but differs once
graders return partial credit (e.g. a fuzzy similarity score in [0, 1]).
```

- [ ] **Step 4: Commit**

```bash
git add evals/03_building_an_eval_harness.ipynb
git commit -m "feat(evals): add report-reading section to notebook 03"
```

---

## Task 9 (was Section 6): Try it

**Files:**
- Modify: `evals/03_building_an_eval_harness.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
### Try it

Swap in a buggier agent — one that always returns `"positive"` regardless of the review — and re-run the harness with the same grader set. Watch how `exact_match` pass rate drops from 80 % to 40 % and `contains_positive` jumps to 100 %. This is the key insight: metrics only mean something in the context of *which graders you chose* and *what the agent is supposed to do*.

Try changing `graders` to include only `exact_match` and re-run, or add a `make_contains("negative", key="contains_negative")` grader and observe that the broken agent scores 0 % on it.
```

- [ ] **Step 2: Add the Try it code cell**

```python
def always_positive_agent(input: dict) -> str:
    """Broken agent: always returns 'positive', ignores the review."""
    return "positive"


bugged_report = run_eval(
    agent=always_positive_agent,
    dataset=dataset,
    graders=[
        exact_match,
        make_contains("positive", key="contains_positive"),
        make_contains("negative", key="contains_negative"),
    ],
)

print("=== Bugged agent — summary table ===")
print(bugged_report.summary_table())
print()
print("Compare to the original agent above:")
print(f"  exact_match pass_rate: original={report.pass_rate('exact_match'):.0%}  "
      f"bugged={bugged_report.pass_rate('exact_match'):.0%}")
print(f"  contains_positive pass_rate: original={report.pass_rate('contains_positive'):.0%}  "
      f"bugged={bugged_report.pass_rate('contains_positive'):.0%}")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
=== Bugged agent — summary table ===
Grader                 Pass rate  Mean score
----------------------------------------------
exact_match               40.0%       0.400
contains_positive        100.0%       1.000
contains_negative          0.0%       0.000
----------------------------------------------
OVERALL                   46.7%       0.467

Compare to the original agent above:
  exact_match pass_rate: original=80%  bugged=40%
  contains_positive pass_rate: original=60%  bugged=100%
```

- [ ] **Step 4: Commit**

```bash
git add evals/03_building_an_eval_harness.ipynb
git commit -m "feat(evals): add Try it section to notebook 03"
```

---

## Task 10: Closing recap and "What's missing" teaser

**Files:**
- Modify: `evals/03_building_an_eval_harness.ipynb` (add 2 markdown cells)

- [ ] **Step 1: Add the "What you just learned" markdown cell**

```markdown
## What you just learned

- An eval is three composable pieces: `Example` (input + expected), `ExampleResult` (example + output + scores), and `EvalReport` (aggregated results with metrics). Each is a plain Python dataclass — no framework needed.
- `run_eval(agent, dataset, graders) -> EvalReport` is the central abstraction: swap any one of the three arguments and re-run. The graders, the agent, and the dataset are all independently replaceable.
- Graders from notebook 02 carry over verbatim except for one small change: `example["expected"]` (dict form) becomes `example.expected` (attribute form). Same logic, same `Score` return type.
- `pass_rate()` and `mean_score()` measure the same underlying data through two different lenses. For binary graders they're equal; once graders assign partial credit they diverge — and that divergence is informative.
- A per-example breakdown loop is often more useful than aggregate metrics alone: it tells you *which* examples failed and *why*, making the harness a debugging tool, not just a scoreboard.
```

- [ ] **Step 2: Add the "What's missing" markdown cell**

```markdown
## What's missing

We now have a harness that can evaluate a single agent snapshot against a fixed dataset. But agents evolve — you fix a bug, change a prompt, update a dependency — and what was passing yesterday might silently break today.

In **`04_regression_testing.ipynb`** we extend this harness into a **regression gate**: treat the eval dataset as a versioned artifact, run the harness against two agent versions (v1 and a deliberately-regressed v2), diff the scores, and define a gate that fails if any example regresses or if aggregate quality drops below a threshold. Same `run_eval` abstraction, same graders, same `EvalReport` — now used to *compare* rather than just *measure*.
```

- [ ] **Step 3: Verify both cells render**

The italic *which* and the bold **`04_regression_testing.ipynb`** and **regression gate** should display correctly.

- [ ] **Step 4: Commit**

```bash
git add evals/03_building_an_eval_harness.ipynb
git commit -m "feat(evals): add closing recap and notebook-04 teaser to notebook 03"
```

---

## Task 11: End-to-end verification in a fresh kernel

**Files:** none (verification only).

- [ ] **Step 1: Execute the notebook top-to-bottom in a clean kernel**

```bash
jupyter nbconvert --to notebook --execute --inplace evals/03_building_an_eval_harness.ipynb
```

Expected: exits with code 0, no cells raise an unhandled exception. The notebook is rewritten with embedded outputs from the fresh run.

- [ ] **Step 2: Verify expected outputs in the executed notebook**

The output cells should contain, in order:

1. Setup cell:
   ```
   Setup OK — Score and graders declared (attribute form)
   ```

2. Dataset cell:
   ```
   Dataset: 5 examples
     [s01] expected='positive'   'This product is absolutely fantastic! Best purchase I'
     [s02] expected='negative'   'Terrible quality. Broke after two days and support nev'
     [s03] expected='neutral'    'It arrived on time and the packaging was fine.'
     [s04] expected='negative'   'Oh great, another product that stops working the moment'
     [s05] expected='positive'   'Exactly what I needed. Simple, effective, good value.'
   ```

3. Smoke test cell:
   ```
   Smoke test — classify_agent predictions:
     [s01] ✓ predicted='positive'  expected='positive'
     [s02] ✓ predicted='negative'  expected='negative'
     [s03] ✓ predicted='neutral'   expected='neutral'
     [s04] ✗ predicted='positive'  expected='negative'
     [s05] ✓ predicted='positive'  expected='positive'
   ```

4. run_eval call cell:
   ```
   EvalReport produced: 5 ExampleResult(s), 10 Score(s) total
   ```

5. Report-reading cell: summary table showing `exact_match 80.0%`, `contains_positive 60.0%`, `OVERALL 70.0%`, plus the per-example breakdown with `[s04]` showing `exact_match=✗`.

6. Try it cell: bugged-agent summary table showing `exact_match 40.0%`, `contains_positive 100.0%`, `contains_negative 0.0%`.

No cell raises an unhandled exception. If `nbconvert` reports `ModuleNotFoundError: No module named 'pydantic'`, install it first: `pip install pydantic`.

- [ ] **Step 3: Commit the clean run**

```bash
git add evals/03_building_an_eval_harness.ipynb
git commit -m "chore(evals): commit clean fresh-kernel run of notebook 03"
```

---

## Done

After Task 11 passes, notebook 03 is complete. The next plan to write is `2026-05-24-evals-notebook-04-regression-testing.md`, which uses the `run_eval` harness built here to compare two agent versions, diff the scores, and define a gate that fails on quality regression — still fully deterministic, still no API key.
