# Evals Notebook 05 — `05_llm_as_judge.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the fifth notebook in the Evals & Observability learning series — introduce the LLM as a grader for open-ended outputs. The notebook motivates the judge (deterministic graders mis-score good summaries), defines `make_llm_judge` and `judge_pairwise`, demonstrates pairwise comparison, exposes the judge's failure modes with concrete code, and closes by validating the judge against a small human-labeled set. **This is the first notebook in the series that requires an API key.**

**Architecture:** A single self-contained Jupyter notebook. No LangSmith yet. All eval harness primitives (`Score`, `Example`, `ExampleResult`, `EvalReport`, `run_eval`) are re-declared inline so the notebook runs standalone. The LLM judge is implemented with the `openai` Python SDK, Chat Completions API, with `response_format` for JSON, default model `"gpt-4o-mini"` (cheap and swappable). An early guard cell checks for `OPENAI_API_KEY` and prints setup instructions if it is absent; remaining cells that call the API are marked to be skipped/guarded when the key is absent.

**Tech Stack:** Python 3.11+, Jupyter, `openai` (`pip install openai`), `os`, `json`, `dataclasses`, `typing`. No LangSmith, no LangChain.

**Companion spec:** `docs/superpowers/specs/2026-05-24-evals-observability-learning-series-design.md` (notebook 05 section).

**Folder:** The `evals/` folder already exists. Task 1 of this plan scaffolds `evals/05_llm_as_judge.ipynb`.

---

## File Structure

- **Create:** `evals/05_llm_as_judge.ipynb` — the entire notebook, self-contained.
- **Modify:** none.
- **No separate test files** — verification is the notebook running top-to-bottom in a fresh kernel with `OPENAI_API_KEY` set (Task 11).

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown)
2. **"What you'll learn"** (markdown bullets)
3. **Section 1: Setup + API key guard** (markdown + 2 code cells — install note, imports + inline harness re-declaration, guard cell)
4. **Section 2: Why deterministic graders fall short** (markdown + 2 code cells — show two good summaries that `exact_match`/`contains` mis-score)
5. **Section 3: Rubric grading with `make_llm_judge`** (markdown + 3 code cells — define `make_llm_judge`, run it on good/bad outputs, plug into `run_eval`)
6. **Section 4: Pairwise comparison with `judge_pairwise`** (markdown + 2 code cells — define `judge_pairwise`, compare two candidates)
7. **Section 5: The traps** (markdown + 3 code cells — position bias demo, verbosity bias note, self-preference + non-determinism note)
8. **Section 6: Evaluating the judge** (markdown + 2 code cells — human-labeled set, agreement rate)
9. **"What you just learned"** (markdown)
10. **"What's missing"** (markdown teaser to notebook 06)

---

## Task 1: Scaffold the empty notebook

**Files:**
- Create: `evals/05_llm_as_judge.ipynb`

- [ ] **Step 1: Verify the `evals/` folder exists**

```bash
ls -la evals/
```

Expected: the folder exists and contains at least `01_` through `04_` notebooks.

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

Save as `evals/05_llm_as_judge.ipynb`.

- [ ] **Step 3: Verify the file is valid JSON**

```bash
python -c "import json; json.load(open('evals/05_llm_as_judge.ipynb'))"
ls -la evals/05_llm_as_judge.ipynb
```

Expected: no exception, file non-zero size.

- [ ] **Step 4: Commit**

```bash
git add evals/05_llm_as_judge.ipynb
git commit -m "feat(evals): scaffold 05_llm_as_judge.ipynb"
```

---

## Task 2: Add title and "Why this notebook exists"

**Files:**
- Modify: `evals/05_llm_as_judge.ipynb` (add cell 1)

- [ ] **Step 1: Add a markdown cell with title and motivation**

Cell content (markdown):

```markdown
# 05 — LLM as Judge

## Why this notebook exists

In **notebook 04** we built a regression gate using only deterministic graders — exact match, `contains`, structured validation, golden outputs. Those graders are fast, free, and fully reproducible. But they break down the moment the task produces *open-ended text*: two summaries can be equally faithful and concise while sharing almost no words. An `exact_match` grader marks one correct and one wrong based purely on whether the string matches a reference. A `contains` grader tells us a keyword appears, not whether the output is actually good.

LLM-as-judge plugs that gap: we ask a language model to score an output against an explicit rubric, returning a structured score and a rationale we can read. This notebook introduces the technique, shows you how to wire it into the harness from notebook 03 as just another grader, and — critically — shows you where it goes wrong and how to check whether your judge can be trusted.

This is the **first notebook in the series that requires an OpenAI API key.** An early guard cell will stop and print instructions if the key is missing.
```

- [ ] **Step 2: Commit**

```bash
git add evals/05_llm_as_judge.ipynb
git commit -m "feat(evals): add intro markdown to notebook 05"
```

---

## Task 3: Add "What you'll learn"

**Files:**
- Modify: `evals/05_llm_as_judge.ipynb` (add cell 2)

- [ ] **Step 1: Add a markdown cell**

Cell content (markdown):

```markdown
## What you'll learn

- Why deterministic graders mis-score open-ended outputs, motivating the need for a judge.
- How to set `OPENAI_API_KEY` and what the guard cell does when it is absent.
- How to define `make_llm_judge(rubric, client)` — a factory that returns a grader with the same `(example, output) -> Score` signature as every other grader in this series.
- How to write a concrete rubric, run the judge on good and bad outputs, and read the `Score` with its `rationale` field.
- How to plug `make_llm_judge` into `run_eval` alongside deterministic graders.
- How to define `judge_pairwise(client, prompt, output_a, output_b)` and why relative judgments are often more reliable than absolute scores.
- The three main traps: **position bias**, **verbosity bias**, and **self-preference / non-determinism** — with a concrete demonstration of position bias.
- How to validate the judge against a small human-labeled set and compute an agreement rate, so the judge is not just another untested component.
```

- [ ] **Step 2: Commit**

```bash
git add evals/05_llm_as_judge.ipynb
git commit -m "feat(evals): add 'What you'll learn' to notebook 05"
```

---

## Task 4: Section 1 — Setup + API key guard

**Files:**
- Modify: `evals/05_llm_as_judge.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 1. Setup + API Key Guard

This notebook uses `openai` for the LLM judge. Install it if needed:

```bash
pip install openai
```

The cells below (a) import everything and re-declare the shared harness primitives inline, then (b) check for `OPENAI_API_KEY`. **If the key is missing, the guard cell prints setup instructions and raises `SystemExit` — all subsequent API-calling cells are safe to skip.**

To get an API key: visit https://platform.openai.com/api-keys, create a key, and export it in your shell before launching Jupyter:

```bash
export OPENAI_API_KEY="sk-..."
```

Anthropic users: you can swap in `anthropic` SDK calls with the same pattern — the `make_llm_judge` factory accepts any callable you pass as `client`. The default model shown here is `"gpt-4o-mini"` (inexpensive and good enough for grading).
```

- [ ] **Step 2: Add the imports + inline harness re-declaration code cell**

```python
# ── stdlib ────────────────────────────────────────────────────────────────────
import os
import json
from dataclasses import dataclass, field
from typing import Any, Callable

# ── openai SDK ────────────────────────────────────────────────────────────────
from openai import OpenAI  # pip install openai

# ══════════════════════════════════════════════════════════════════════════════
# Shared harness — re-declared inline so this notebook is self-contained.
# These match the canonical definitions from notebooks 03 & 04 exactly.
# ══════════════════════════════════════════════════════════════════════════════

@dataclass
class Score:
    key: str
    score: float          # normalised to [0, 1]
    passed: bool
    comment: str = ""


@dataclass
class Example:
    input: Any
    expected: Any = None
    metadata: dict = field(default_factory=dict)


@dataclass
class ExampleResult:
    example: Example
    output: Any
    scores: list[Score] = field(default_factory=list)


@dataclass
class EvalReport:
    results: list[ExampleResult]

    @property
    def pass_rate(self) -> float:
        all_scores = [s for r in self.results for s in r.scores]
        if not all_scores:
            return 0.0
        return sum(s.passed for s in all_scores) / len(all_scores)

    @property
    def mean_score(self) -> float:
        all_scores = [s for r in self.results for s in r.scores]
        if not all_scores:
            return 0.0
        return sum(s.score for s in all_scores) / len(all_scores)

    def summary_table(self) -> None:
        """Print a simple per-example summary table."""
        print(f"{'#':<4} {'passed':<8} {'mean score':<12} {'comment'}")
        print("-" * 60)
        for i, r in enumerate(self.results):
            scores = r.scores
            if not scores:
                print(f"{i:<4} {'—':<8} {'—':<12} no graders")
                continue
            passed = all(s.passed for s in scores)
            mean = sum(s.score for s in scores) / len(scores)
            comments = "; ".join(s.comment for s in scores if s.comment)
            print(f"{i:<4} {'✓' if passed else '✗':<8} {mean:<12.3f} {comments[:60]}")
        print("-" * 60)
        print(f"Pass rate: {self.pass_rate:.1%}   Mean score: {self.mean_score:.3f}")


Grader = Callable[[Example, Any], Score]


def run_eval(
    agent: Callable[[Any], Any],
    dataset: list[Example],
    graders: list[Grader],
) -> EvalReport:
    """Run `agent` on every `Example`, apply every grader, return a report."""
    results: list[ExampleResult] = []
    for example in dataset:
        output = agent(example.input)
        scores = [grader(example, output) for grader in graders]
        results.append(ExampleResult(example=example, output=output, scores=scores))
    return EvalReport(results=results)


print("Harness re-declared OK.")
```

- [ ] **Step 3: Add the API key guard code cell**

```python
# ── API key guard ─────────────────────────────────────────────────────────────
# This cell must run before any cell that calls the OpenAI API.
# If OPENAI_API_KEY is not set, it prints setup instructions and stops the
# notebook so subsequent cells that call the API are safe to skip.

_api_key = os.getenv("OPENAI_API_KEY")

if not _api_key:
    print(
        "┌─────────────────────────────────────────────────────────────────┐\n"
        "│  OPENAI_API_KEY is not set.                                     │\n"
        "│                                                                 │\n"
        "│  This is the first notebook in the series that needs a key.    │\n"
        "│  Steps:                                                         │\n"
        "│    1. Visit https://platform.openai.com/api-keys               │\n"
        "│    2. Create a new secret key.                                  │\n"
        "│    3. In your terminal (before launching Jupyter):              │\n"
        "│         export OPENAI_API_KEY=\"sk-...\"                         │\n"
        "│    4. Restart the Jupyter kernel and re-run from the top.       │\n"
        "│                                                                 │\n"
        "│  Anthropic alternative: replace `from openai import OpenAI`    │\n"
        "│  with `import anthropic` and adapt the client calls in         │\n"
        "│  make_llm_judge / judge_pairwise to use the Messages API.       │\n"
        "└─────────────────────────────────────────────────────────────────┘"
    )
    raise SystemExit(
        "Set OPENAI_API_KEY and restart the kernel to continue."
    )

client = OpenAI()  # reads OPENAI_API_KEY from environment automatically
DEFAULT_MODEL = "gpt-4o-mini"

print(f"OpenAI client ready. Default model: {DEFAULT_MODEL}")
print("Key found (first 8 chars):", _api_key[:8] + "…")
```

- [ ] **Step 4: Commit**

```bash
git add evals/05_llm_as_judge.ipynb
git commit -m "feat(evals): add setup, harness re-declaration, and API key guard to notebook 05"
```

---

## Task 5: Section 2 — Why deterministic graders fall short

**Files:**
- Modify: `evals/05_llm_as_judge.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 2. Why Deterministic Graders Fall Short

Suppose our agent summarises a paragraph. We have a reference summary. Let's define two *clearly good* candidate summaries — both faithful, concise, and complete — and watch `exact_match` and `contains` score them.
```

- [ ] **Step 2: Add the motivating code cell**

```python
# ── Source paragraph and reference ────────────────────────────────────────────
SOURCE = (
    "The Apollo 11 mission, launched on July 16, 1969, successfully landed "
    "astronauts Neil Armstrong and Buzz Aldrin on the Moon on July 20. "
    "Armstrong became the first human to walk on the lunar surface, followed "
    "by Aldrin. Michael Collins orbited the Moon in the command module while "
    "his crewmates explored the surface. The crew returned safely to Earth "
    "on July 24, 1969."
)

REFERENCE = (
    "Apollo 11 (July 1969) landed Armstrong and Aldrin on the Moon; "
    "Collins orbited above. Armstrong was first to walk on the surface. "
    "The crew returned safely on July 24."
)

# Two summaries — both clearly good, written differently
SUMMARY_A = (
    "In July 1969, Apollo 11 brought Neil Armstrong and Buzz Aldrin to the "
    "lunar surface. Armstrong stepped out first. Michael Collins remained in "
    "orbit. All three astronauts returned to Earth safely on July 24th."
)

SUMMARY_B = (
    "Apollo 11 successfully landed on the Moon on July 20, 1969. "
    "Armstrong and Aldrin walked on the surface while Collins orbited. "
    "The mission concluded with a safe splashdown on July 24, 1969."
)

print("Source, reference, and two good candidate summaries defined.")
```

- [ ] **Step 3: Add the grader mis-scoring code cell**

```python
# ── Deterministic graders ─────────────────────────────────────────────────────

def exact_match(example: Example, output: str) -> Score:
    passed = output.strip() == str(example.expected).strip()
    return Score(key="exact_match", score=1.0 if passed else 0.0, passed=passed)


def contains_keywords(keywords: list[str]) -> Grader:
    """Grader factory: passes if ALL keywords appear in the output."""
    def grader(example: Example, output: str) -> Score:
        found = [kw for kw in keywords if kw.lower() in output.lower()]
        score = len(found) / len(keywords)
        passed = score == 1.0
        comment = f"found {len(found)}/{len(keywords)}: {found}"
        return Score(key="contains_keywords", score=score, passed=passed, comment=comment)
    return grader


# Evaluate both summaries with the reference as `expected`
example_ref = Example(input=SOURCE, expected=REFERENCE)

kw_grader = contains_keywords(["Armstrong", "Aldrin", "Collins", "July 24"])

print("=== Summary A ===")
em_a = exact_match(example_ref, SUMMARY_A)
kw_a = kw_grader(example_ref, SUMMARY_A)
print(f"  exact_match  → passed={em_a.passed}, score={em_a.score}")
print(f"  contains_kw  → passed={kw_a.passed}, score={kw_a.score:.2f}, {kw_a.comment}")

print()
print("=== Summary B ===")
em_b = exact_match(example_ref, SUMMARY_B)
kw_b = kw_grader(example_ref, SUMMARY_B)
print(f"  exact_match  → passed={em_b.passed}, score={em_b.score}")
print(f"  contains_kw  → passed={kw_b.passed}, score={kw_b.score:.2f}, {kw_b.comment}")

print()
print(
    "Both summaries are accurate and well-written.\n"
    "exact_match scores both 0 (neither matches the reference string exactly).\n"
    "contains_keywords scores both 1.0 — but can't tell us HOW good they are.\n"
    "We need a grader that reads the text and reasons about quality."
)
```

Expected output:
```
=== Summary A ===
  exact_match  → passed=False, score=0.0
  contains_kw  → passed=True, score=1.00, found 4/4: ['Armstrong', 'Aldrin', 'Collins', 'July 24']

=== Summary B ===
  exact_match  → passed=False, score=0.0
  contains_kw  → passed=True, score=1.00, found 4/4: ['Armstrong', 'Aldrin', 'Collins', 'July 24']

Both summaries are accurate and well-written.
exact_match scores both 0 (neither matches the reference string exactly).
contains_keywords scores both 1.0 — but can't tell us HOW good they are.
We need a grader that reads the text and reasons about quality.
```

- [ ] **Step 4: Commit**

```bash
git add evals/05_llm_as_judge.ipynb
git commit -m "feat(evals): add deterministic-grader motivation section to notebook 05"
```

---

## Task 6: Section 3 — Rubric grading with `make_llm_judge`

**Files:**
- Modify: `evals/05_llm_as_judge.ipynb` (add 1 markdown cell + 3 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 3. Rubric Grading — `make_llm_judge`

A rubric judge takes an explicit scoring criterion written in prose, sends the output (and the original input + expected reference) to a language model, and asks for a structured JSON response: `{"score": <0–1 float>, "passed": <bool>, "rationale": <string>}`. The rationale is stored in `Score.comment` so we can read *why* the judge gave each score.

`make_llm_judge` is a **factory**: it takes the rubric and configuration, and returns a grader function with the standard `(example, output) -> Score` signature — making it a drop-in for `run_eval` alongside any deterministic grader.

### Try it
```

- [ ] **Step 2: Add the `make_llm_judge` definition code cell**

```python
def make_llm_judge(
    rubric: str,
    client: OpenAI,
    model: str = "gpt-4o-mini",
    key: str = "llm_judge",
) -> Grader:
    """Return a grader that scores an output against `rubric` using an LLM.

    The returned grader has signature (example, output) -> Score and can be
    passed directly to run_eval alongside deterministic graders.

    The LLM is prompted with:
      - The rubric
      - example.input (the prompt given to the agent)
      - example.expected (the reference output, if any)
      - The candidate output to score

    It must respond with JSON: {"score": <0-1 float>, "passed": <bool>,
    "rationale": <string>}. score is normalised to [0, 1] before returning.
    """

    system_prompt = (
        "You are a precise, impartial evaluator. "
        "Score the candidate output against the rubric. "
        "Respond with ONLY valid JSON matching this schema:\n"
        '{"score": <float 0-1>, "passed": <bool>, "rationale": <string>}\n'
        "No markdown fences, no extra keys."
    )

    def grader(example: Example, output: Any) -> Score:
        user_message = (
            f"## Rubric\n{rubric}\n\n"
            f"## Input given to the agent\n{example.input}\n\n"
        )
        if example.expected is not None:
            user_message += f"## Reference output\n{example.expected}\n\n"
        user_message += f"## Candidate output to score\n{output}\n\n"
        user_message += "Respond with JSON only."

        response = client.chat.completions.create(
            model=model,
            response_format={"type": "json_object"},
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": user_message},
            ],
            temperature=0,
        )

        raw = response.choices[0].message.content
        parsed = json.loads(raw)

        raw_score = float(parsed["score"])
        normalised = max(0.0, min(1.0, raw_score))  # clamp to [0, 1]
        passed = bool(parsed["passed"])
        rationale = str(parsed.get("rationale", ""))

        return Score(key=key, score=normalised, passed=passed, comment=rationale)

    return grader


print("make_llm_judge defined.")
```

- [ ] **Step 3: Add the code cell that runs the judge on good and bad summaries**

```python
# ── Rubric for summary quality ─────────────────────────────────────────────────
SUMMARY_RUBRIC = (
    "Score the summary on two equally-weighted dimensions (each 0–0.5, "
    "combined into a single 0–1 score):\n"
    "  1. FAITHFULNESS: Every claim in the summary must be supported by the "
    "source text. Penalise hallucinated facts, wrong dates, or wrong names.\n"
    "  2. CONCISENESS: The summary should convey the essential information "
    "without padding or irrelevant detail.\n"
    "A score >= 0.7 should be marked passed=true. "
    "Score < 0.7 should be marked passed=false."
)

judge = make_llm_judge(rubric=SUMMARY_RUBRIC, client=client)

# A clearly bad summary: hallucinated detail (wrong astronaut count, wrong date)
BAD_SUMMARY = (
    "The Apollo 11 mission in August 1969 sent four astronauts to the Moon. "
    "Armstrong was the only one to walk on the surface. The crew never returned."
)

print("Scoring Summary A (good)…")
score_a = judge(Example(input=SOURCE, expected=REFERENCE), SUMMARY_A)
print(f"  key={score_a.key!r}, score={score_a.score:.3f}, passed={score_a.passed}")
print(f"  rationale: {score_a.comment[:200]}")

print()
print("Scoring Summary B (good)…")
score_b = judge(Example(input=SOURCE, expected=REFERENCE), SUMMARY_B)
print(f"  key={score_b.key!r}, score={score_b.score:.3f}, passed={score_b.passed}")
print(f"  rationale: {score_b.comment[:200]}")

print()
print("Scoring Bad Summary (hallucinated facts)…")
score_bad = judge(Example(input=SOURCE, expected=REFERENCE), BAD_SUMMARY)
print(f"  key={score_bad.key!r}, score={score_bad.score:.3f}, passed={score_bad.passed}")
print(f"  rationale: {score_bad.comment[:200]}")
```

Representative expected output (exact text will vary — model responses are non-deterministic):
```
Scoring Summary A (good)…
  key='llm_judge', score=0.900, passed=True
  rationale: The summary faithfully captures all key facts from the source …

Scoring Summary B (good)…
  key='llm_judge', score=0.950, passed=True
  rationale: Accurate dates, names, and outcome. Concise and complete …

Scoring Bad Summary (hallucinated facts)…
  key='llm_judge', score=0.200, passed=False
  rationale: Incorrect: the mission was July not August, only two walked …
```

Note: exact scores and rationale text will differ across runs; structural shape (float score, bool passed, non-empty rationale string) is what matters.

- [ ] **Step 4: Add the `run_eval` integration code cell**

```python
# ── Plug the judge into run_eval alongside a deterministic grader ──────────────

dataset = [
    Example(input=SOURCE, expected=REFERENCE),
    Example(input=SOURCE, expected=REFERENCE),
    Example(input=SOURCE, expected=REFERENCE),
]

def summary_agent_good(inp: str) -> str:
    """Stub: always returns Summary A."""
    return SUMMARY_A

def summary_agent_bad(inp: str) -> str:
    """Stub: always returns the hallucinated bad summary."""
    return BAD_SUMMARY

graders = [
    kw_grader,    # deterministic: from section 2
    judge,        # LLM: from this section
]

print("=== Good agent ===")
report_good = run_eval(summary_agent_good, dataset, graders)
report_good.summary_table()

print()
print("=== Bad agent ===")
report_bad = run_eval(summary_agent_bad, dataset, graders)
report_bad.summary_table()
```

Representative expected output:
```
=== Good agent ===
#    passed   mean score   comment
------------------------------------------------------------
0    ✓        0.950        found 4/4: ['Armstrong', …]; The summary faithfully …
1    ✓        0.950        …
2    ✓        0.950        …
------------------------------------------------------------
Pass rate: 100.0%   Mean score: 0.950

=== Bad agent ===
#    passed   mean score   comment
------------------------------------------------------------
0    ✗        0.350        found 2/4: ['Armstrong', …]; Incorrect: wrong month …
1    ✗        0.350        …
2    ✗        0.350        …
------------------------------------------------------------
Pass rate: 0.0%   Mean score: 0.350
```

Note: LLM scores vary; the deterministic column is stable. The bad agent will consistently fail.

- [ ] **Step 5: Commit**

```bash
git add evals/05_llm_as_judge.ipynb
git commit -m "feat(evals): add make_llm_judge definition and run_eval integration to notebook 05"
```

---

## Task 7: Section 4 — Pairwise comparison with `judge_pairwise`

**Files:**
- Modify: `evals/05_llm_as_judge.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 4. Pairwise Comparison — `judge_pairwise`

Absolute rubric scores are useful, but they have a calibration problem: two different rubrics, or even two different prompts for the same rubric, can assign very different absolute values to the same output. Pairwise comparison sidesteps this: we show the judge two candidate outputs and ask only *"which is better?"* — a relative judgment that is often more reliable and robust to prompt wording.

`judge_pairwise` returns `"A"`, `"B"`, or `"tie"`.

### Try it
```

- [ ] **Step 2: Add the `judge_pairwise` definition code cell**

```python
def judge_pairwise(
    client: OpenAI,
    prompt: str,
    output_a: str,
    output_b: str,
    model: str = "gpt-4o-mini",
) -> str:
    """Ask the LLM which of two outputs is better for `prompt`.

    Returns "A", "B", or "tie".

    `prompt` is the task description / input; output_a and output_b are the
    two candidates. The judge is shown both and must choose.
    """
    system_prompt = (
        "You are a fair evaluator. Given a task prompt and two candidate "
        "outputs (A and B), decide which is better.\n"
        "Respond with ONLY valid JSON: "
        '{"winner": "A" | "B" | "tie", "reason": "<one sentence>"}\n'
        "No markdown fences, no extra keys. "
        "Be objective; do not favour longer responses."
    )
    user_message = (
        f"## Task prompt\n{prompt}\n\n"
        f"## Output A\n{output_a}\n\n"
        f"## Output B\n{output_b}\n\n"
        "Which output better addresses the task? Respond with JSON only."
    )

    response = client.chat.completions.create(
        model=model,
        response_format={"type": "json_object"},
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_message},
        ],
        temperature=0,
    )

    raw = response.choices[0].message.content
    parsed = json.loads(raw)
    winner = str(parsed.get("winner", "tie")).strip().upper()
    if winner not in {"A", "B", "TIE"}:
        winner = "TIE"
    reason = str(parsed.get("reason", ""))
    print(f"  winner={winner!r}, reason: {reason[:120]}")
    return "tie" if winner == "TIE" else winner


print("judge_pairwise defined.")
```

- [ ] **Step 3: Add the pairwise comparison code cell**

```python
# ── Compare Summary A vs Bad Summary ──────────────────────────────────────────
TASK_PROMPT = f"Summarise the following paragraph concisely and faithfully:\n\n{SOURCE}"

print("Comparing good Summary A vs hallucinated Bad Summary:")
winner = judge_pairwise(client, TASK_PROMPT, SUMMARY_A, BAD_SUMMARY)
print(f"Result: {winner!r}  (expected: 'A')\n")

print("Comparing bad Summary vs good Summary A (reversed):")
winner_rev = judge_pairwise(client, TASK_PROMPT, BAD_SUMMARY, SUMMARY_A)
print(f"Result: {winner_rev!r}  (expected: 'B')\n")

print("Comparing two good summaries (A vs B):")
winner_ab = judge_pairwise(client, TASK_PROMPT, SUMMARY_A, SUMMARY_B)
print(f"Result: {winner_ab!r}  (may be 'A', 'B', or 'tie' — both are good)")
```

Representative expected output:
```
Comparing good Summary A vs hallucinated Bad Summary:
  winner='A', reason: Summary A is factually accurate while Summary B contains incorrect …
Result: 'A'  (expected: 'A')

Comparing bad Summary vs good Summary A (reversed):
  winner='B', reason: Output B correctly states July 1969 and includes all three …
Result: 'B'  (expected: 'B')

Comparing two good summaries (A vs B):
  winner='tie', reason: Both summaries accurately capture the essential facts …
Result: 'tie'  (may be 'A', 'B', or 'tie' — both are good)
```

Note: the winner for the two-good-summaries comparison may be `"A"`, `"B"`, or `"tie"` and will vary across runs.

- [ ] **Step 4: Commit**

```bash
git add evals/05_llm_as_judge.ipynb
git commit -m "feat(evals): add judge_pairwise definition and comparison cells to notebook 05"
```

---

## Task 8: Section 5 — The traps (Gotchas)

**Files:**
- Modify: `evals/05_llm_as_judge.ipynb` (add 1 markdown cell + 3 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 5. The Traps

The LLM judge is a powerful tool, but it has well-documented failure modes. Before shipping a judge-based eval pipeline, you should know what these are and how to test for them.

> **Gotcha:** An LLM judge can produce confidently wrong scores. Until you validate it against human labels (section 6), it is just another untested component in your system.

### 5.1 Position Bias

Studies have shown that LLM judges tend to favour the candidate that appears **first** (position A) when two outputs are close in quality — not because it is better, but because of the model's positional preference in context. Concretely: you can sometimes flip the winner just by swapping A and B.

### 5.2 Verbosity Bias

LLMs often rate **longer** outputs as better, even when the extra length is padding. Conciseness in a rubric needs to be stated explicitly and forcefully to counteract this tendency.

### 5.3 Self-Preference and Non-Determinism

A judge based on GPT-4o-mini may subtly prefer outputs that *sound* like GPT-4o-mini output (a form of self-preference). Additionally, even with `temperature=0`, the judge's scores can vary slightly across runs due to sampling non-determinism in the API infrastructure.

### Try it — position bias demonstration
```

- [ ] **Step 2: Add the position bias demonstration code cell**

```python
# ── Position bias: construct a close pair and swap A / B ──────────────────────
# Summary A and Summary B are both good. We run pairwise 4 times in each
# ordering to see whether the winner is stable or flips.

import collections

TRIALS = 4

results_ab: list[str] = []
results_ba: list[str] = []

print(f"Running {TRIALS} trials of A vs B …")
for i in range(TRIALS):
    r = judge_pairwise(client, TASK_PROMPT, SUMMARY_A, SUMMARY_B)
    results_ab.append(r)

print(f"\nRunning {TRIALS} trials of B vs A (swapped) …")
for i in range(TRIALS):
    r = judge_pairwise(client, TASK_PROMPT, SUMMARY_B, SUMMARY_A)
    results_ba.append(r)

counts_ab = collections.Counter(results_ab)
counts_ba = collections.Counter(results_ba)

print("\n── Results ──────────────────────────────────────────────────────────")
print(f"A-first ordering  → {dict(counts_ab)}")
print(f"B-first ordering  → {dict(counts_ba)}")
print()
print(
    "If A wins more often when listed first and B wins more often when listed\n"
    "first, that is position bias at work. With temperature=0 and a clear\n"
    "quality difference the bias may not appear; it surfaces on close pairs."
)
```

Representative expected output (will vary):
```
Running 4 trials of A vs B …
  winner='A', reason: …
  winner='tie', reason: …
  winner='A', reason: …
  winner='A', reason: …

Running 4 trials of B vs A (swapped) …
  winner='tie', reason: …
  winner='A', reason: …   ← 'A' here means B in the original ordering
  winner='B', reason: …
  winner='tie', reason: …

── Results ──────────────────────────────────────────────────────────
A-first ordering  → {'A': 3, 'tie': 1}
B-first ordering  → {'tie': 2, 'A': 1, 'B': 1}

If A wins more often when listed first and B wins more often when listed
first, that is position bias at work. …
```

Note: counts are model-dependent and will vary. The important observation is that results *can* shift between orderings on a close pair.

- [ ] **Step 3: Add the verbosity + non-determinism callout code cell**

```python
# ── Verbosity bias: observe scores on padded vs concise output ─────────────────

PADDED_SUMMARY = (
    "This is a summary of the text you provided above. "
    "The text discusses the Apollo 11 mission. "
    "Apollo 11 was launched in July 1969. "
    "The astronauts Neil Armstrong and Buzz Aldrin landed on the Moon on July 20. "
    "They walked on the surface. Michael Collins was in the command module orbiting. "
    "The mission was a success. The crew came back to Earth on July 24, 1969. "
    "This was a historic event in human spaceflight history and mankind as a whole. "
    "It demonstrated the capability of the United States space program at the time. "
    "In conclusion, Apollo 11 was a very significant mission."
)

judge_verbose = make_llm_judge(
    rubric=SUMMARY_RUBRIC,
    client=client,
    key="llm_judge_padded",
)

print("Scoring concise Summary B:")
score_concise = judge_verbose(Example(input=SOURCE, expected=REFERENCE), SUMMARY_B)
print(f"  score={score_concise.score:.3f}, passed={score_concise.passed}")
print(f"  rationale: {score_concise.comment[:200]}")

print()
print("Scoring padded summary (same facts, much more verbose):")
score_padded = judge_verbose(Example(input=SOURCE, expected=REFERENCE), PADDED_SUMMARY)
print(f"  score={score_padded.score:.3f}, passed={score_padded.passed}")
print(f"  rationale: {score_padded.comment[:200]}")

print()
print(
    "If the padded summary scores >= the concise one, verbosity bias may be\n"
    "present. The rubric explicitly penalises padding — check whether the\n"
    "judge's rationale actually mentions it."
)
```

Representative expected output:
```
Scoring concise Summary B:
  score=0.950, passed=True
  rationale: Accurate and concise. Covers all key facts without padding …

Scoring padded summary (same facts, much more verbose):
  score=0.700, passed=True
  rationale: Factually accurate but includes unnecessary filler sentences …
```

Note: a well-prompted rubric with explicit conciseness criteria should penalise the padded version. If it does not, the rubric needs strengthening.

- [ ] **Step 4: Commit**

```bash
git add evals/05_llm_as_judge.ipynb
git commit -m "feat(evals): add judge traps section (position bias, verbosity, non-determinism) to notebook 05"
```

---

## Task 9: Section 6 — Evaluating the judge

**Files:**
- Modify: `evals/05_llm_as_judge.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 6. Evaluating the Judge

The key insight from this notebook: **an unvalidated judge is just another untested component.** If the judge gives wrong scores, every eval pipeline downstream of it is wrong too — silently.

The cheapest validation is a small **human-labeled set**: a handful of (input, output) pairs where a human has already decided the correct score or preference. We run the judge on the same pairs and compute an **agreement rate**: what fraction of the judge's pass/fail decisions match the human's?

A real-world threshold depends on your use case, but as a rule of thumb: below ~70% agreement on a balanced set, the judge is adding noise rather than signal.

### Try it
```

- [ ] **Step 2: Add the human-labeled validation set code cell**

```python
# ── Human-labeled validation set (defined inline) ─────────────────────────────
# Each entry: (source_text, candidate_summary, human_passed: bool)
# These represent a human reviewer's ground-truth judgments.

HUMAN_LABELED = [
    # (input, candidate, human_passed)
    (
        SOURCE,
        SUMMARY_A,
        True,   # human: good — accurate and concise
    ),
    (
        SOURCE,
        SUMMARY_B,
        True,   # human: good — accurate and concise
    ),
    (
        SOURCE,
        BAD_SUMMARY,
        False,  # human: fail — wrong month, wrong crew count, wrong outcome
    ),
    (
        SOURCE,
        (
            "Neil Armstrong walked on the Moon. "
            "The mission was called Apollo. "
            "There were some other astronauts too."
        ),
        False,  # human: fail — vague, missing key facts (dates, Collins role)
    ),
]

print(f"Human-labeled validation set: {len(HUMAN_LABELED)} examples")
for i, (src, summ, human) in enumerate(HUMAN_LABELED):
    print(f"  [{i}] human_passed={human}  summary[:60]: {summ[:60]!r}")
```

Expected output:
```
Human-labeled validation set: 4 examples
  [0] human_passed=True  summary[:60]: 'In July 1969, Apollo 11 brought Neil Armstrong and Buzz Aldr'
  [1] human_passed=True  summary[:60]: 'Apollo 11 successfully landed on the Moon on July 20, 1969.'
  [2] human_passed=False  summary[:60]: 'The Apollo 11 mission in August 1969 sent four astronauts to'
  [3] human_passed=False  summary[:60]: 'Neil Armstrong walked on the Moon. The mission was called Ap'
```

- [ ] **Step 3: Add the agreement rate computation code cell**

```python
# ── Run the judge on each example and compare to human label ──────────────────

agreement_judge = make_llm_judge(rubric=SUMMARY_RUBRIC, client=client, key="llm_judge")

agreements = []
print(f"{'#':<4} {'human':<8} {'judge':<8} {'agree':<8} rationale[:60]")
print("-" * 70)

for i, (src, summ, human_passed) in enumerate(HUMAN_LABELED):
    score = agreement_judge(Example(input=src, expected=REFERENCE), summ)
    agree = score.passed == human_passed
    agreements.append(agree)
    print(
        f"{i:<4} {str(human_passed):<8} {str(score.passed):<8} "
        f"{'✓' if agree else '✗':<8} {score.comment[:60]}"
    )

agreement_rate = sum(agreements) / len(agreements)
print("-" * 70)
print(f"Agreement rate: {agreement_rate:.1%}  ({sum(agreements)}/{len(agreements)} examples)")
print()
if agreement_rate >= 0.75:
    print("Judge agrees with human labels at >= 75% — reasonable signal for this rubric.")
else:
    print(
        "Agreement below 75% — consider revising the rubric or expanding the "
        "human-labeled set before trusting this judge in a pipeline."
    )
```

Representative expected output (will vary):
```
#    human    judge    agree    rationale[:60]
----------------------------------------------------------------------
0    True     True     ✓        The summary accurately covers all key facts …
1    True     True     ✓        Accurate and concise, covers all major points …
2    False    False    ✓        Contains factual errors: wrong month and crew …
3    False    False    ✓        Too vague; missing dates, Collins's role, and …
----------------------------------------------------------------------
Agreement rate: 100.0%  (4/4 examples)

Judge agrees with human labels at >= 75% — reasonable signal for this rubric.
```

Note: agreement rate may be lower on some runs. The 4-example set is intentionally small for illustration; in practice use at least 20–50 human-labeled pairs.

- [ ] **Step 4: Commit**

```bash
git add evals/05_llm_as_judge.ipynb
git commit -m "feat(evals): add judge validation against human-labeled set to notebook 05"
```

---

## Task 10: Closing recap + "What's missing" teaser

**Files:**
- Modify: `evals/05_llm_as_judge.ipynb` (add 2 markdown cells)

- [ ] **Step 1: Add the "What you just learned" markdown cell**

```markdown
## What you just learned

- **Deterministic graders break on open-ended text** — `exact_match` marks good summaries wrong; `contains` can't distinguish good from bad.
- **`make_llm_judge(rubric, client)`** returns a grader with the standard `(example, output) -> Score` signature. It prompts the model with the rubric + context, parses a JSON response into a `Score`, and stores the rationale in `comment`. It plugs into `run_eval` as a drop-in alongside deterministic graders.
- **`judge_pairwise(client, prompt, output_a, output_b)`** returns `"A"`, `"B"`, or `"tie"`. Relative judgments are often more reliable than absolute rubric scores for close-quality pairs.
- **Position bias** is real: swapping A and B can flip the winner on a close pair. Always test it.
- **Verbosity bias**: LLMs tend to rate longer outputs higher unless the rubric explicitly penalises padding.
- **An unvalidated judge is just another untested component.** Validate against a small human-labeled set and compute an agreement rate before trusting the judge in a pipeline.
```

- [ ] **Step 2: Add the "What's missing" markdown cell**

```markdown
## What's missing

We can now score outputs — deterministically and with an LLM judge. But we still can't **see** what a multi-step agent did internally while producing those outputs. When a score is wrong, is the agent's first step failing? Its tool call? Its final synthesis? We have no visibility.

In **notebook 06 — `06_tracing_with_langsmith.ipynb`** we instrument a real multi-step agent, run it, and read its trace in the LangSmith UI: every span, every intermediate input and output, latency per step, and token usage. That's the observability layer that makes debugging and root-cause analysis possible.
```

- [ ] **Step 3: Commit**

```bash
git add evals/05_llm_as_judge.ipynb
git commit -m "feat(evals): add closing recap and notebook-06 teaser to notebook 05"
```

---

## Task 11: End-to-end verification in a fresh kernel

**Files:** none (verification only).

**Prerequisites:** `OPENAI_API_KEY` must be set in the environment before running this task. The notebook will not execute to completion without it — the guard cell in Section 1 raises `SystemExit` when the key is absent. This is by design; document it in the verification output.

- [ ] **Step 1: Confirm `OPENAI_API_KEY` is set**

```bash
python -c "import os; k=os.getenv('OPENAI_API_KEY'); print('Key present:', bool(k)); print('Prefix:', (k or '')[:8])"
```

Expected: `Key present: True` and an 8-char prefix printed. If `Key present: False`, set the key before proceeding:

```bash
export OPENAI_API_KEY="sk-..."
```

- [ ] **Step 2: Execute the notebook top-to-bottom in a clean kernel**

```bash
jupyter nbconvert --to notebook --execute --inplace evals/05_llm_as_judge.ipynb
```

Expected: succeeds with no errors. The notebook is rewritten with outputs from the fresh run.

If `OPENAI_API_KEY` is absent, `nbconvert` will report a `SystemExit` from the guard cell and stop — that is correct behavior. Re-run after setting the key.

- [ ] **Step 3: Verify structural expected output (not exact text)**

Because judge outputs are model-dependent, verify **structure**, not exact strings. Check all of the following:

```bash
python - <<'EOF'
import json, re, sys

nb = json.load(open("evals/05_llm_as_judge.ipynb"))
outputs = []
for cell in nb["cells"]:
    for out in cell.get("outputs", []):
        text = "".join(out.get("text", out.get("data", {}).get("text/plain", [])))
        outputs.append(text)

full = "\n".join(outputs)

checks = [
    ("Harness re-declared OK", "harness re-declaration cell"),
    ("OpenAI client ready",    "API key guard passed"),
    ("key='llm_judge'",        "Score with key=llm_judge present"),
    ("passed=True",            "at least one passed=True Score"),
    ("passed=False",           "at least one passed=False Score"),
    ("rationale",              "rationale/comment field present"),
    ("winner=",                "pairwise comparison ran"),
    ("Agreement rate",         "judge validation ran"),
]

all_ok = True
for pattern, label in checks:
    found = pattern in full
    status = "OK" if found else "MISSING"
    print(f"  [{status}] {label!r}")
    if not found:
        all_ok = False

sys.exit(0 if all_ok else 1)
EOF
```

Expected: all checks print `[OK]`. If any print `[MISSING]`, re-examine that section.

The check looks for structural markers (field names, section headers) rather than exact numeric values, because LLM-generated scores vary across runs.

- [ ] **Step 4: Verify the key guard behavior when key is absent (optional but recommended)**

In a separate terminal with the key unset, confirm the guard cell raises:

```bash
OPENAI_API_KEY="" jupyter nbconvert --to notebook --execute evals/05_llm_as_judge.ipynb \
  --output /tmp/05_no_key_test.ipynb 2>&1 | grep -i "SystemExit\|OPENAI_API_KEY\|Set OPENAI"
```

Expected: output includes `SystemExit` or the guard cell's printed instructions. This confirms the guard works correctly.

- [ ] **Step 5: Commit the clean run**

```bash
git add evals/05_llm_as_judge.ipynb
git commit -m "chore(evals): commit clean fresh-kernel run of notebook 05"
```

---

## Done

After Task 11 passes, notebook 05 is complete. The next plan to write is `2026-05-24-evals-notebook-06-tracing-with-langsmith.md`, which instruments a real multi-step LangGraph/LangChain agent with `@traceable`, runs it, and reads its trace in the LangSmith UI — the first notebook requiring both `OPENAI_API_KEY` and a free LangSmith account (`LANGCHAIN_API_KEY`).
