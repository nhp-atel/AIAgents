# Evals Notebook 01 — `01_why_agents_are_hard_to_test.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first notebook in the Evals & Observability learning series — establish *why* evaluating agents is hard, before introducing any tooling. The notebook builds a deterministic-stub toy agent whose output *phrasing* varies across calls, writes a naive `assert` test that flakes even when the answer is semantically correct, then enumerates the three core difficulties (non-determinism, no single ground truth, multi-step compounding) with a runnable mini-demo for each. It closes by motivating the `Score` / grader shape that notebook 02 will implement. **No API key. No external dependencies beyond the Python standard library.**

**Architecture:** A single self-contained Jupyter notebook. All "LLM" behavior is a deterministic Python stub using a seeded `random.Random` — never a real network call. No temp files, no subprocesses, no servers.

**Tech Stack:** Python 3.11+, Jupyter, `random` (stdlib), `dataclasses` (stdlib). Nothing to install.

**Companion spec:** `docs/superpowers/specs/2026-05-24-evals-observability-learning-series-design.md` (notebook 01 section).

**Folder:** This is the **first** notebook in the series and the `evals/` folder does not yet exist. Task 1 of this plan creates the folder.

---

## File Structure

- **Create:** `evals/` (folder)
- **Create:** `evals/01_why_agents_are_hard_to_test.ipynb` — the entire notebook, self-contained.
- **Modify:** none.
- **No separate test files** — verification is the notebook running top-to-bottom in a fresh kernel (Task 9).

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown)
2. **"What you'll learn"** (markdown bullets)
3. **Section 1: Setup** (markdown + 1 code cell — stdlib imports and seed note)
4. **Section 2: A toy sentiment-classifier agent** (markdown + 1 code cell — the stub agent + `### Try it` demonstration cell)
5. **Section 3: The naive test — and why it flakes** (markdown + 1 code cell — the failing `assert` + loop over seeds counting pass vs fail)
6. **Section 4: The three core difficulties** (markdown + 3 mini-demo subsections, each with a markdown cell + code cell)
   - 4a. Non-determinism (same input → different surface text)
   - 4b. No single ground truth (two equally-good summaries that aren't equal)
   - 4c. Multi-step compounding (2-step pipeline where step-1 variation breaks step-2 brittle parsing)
7. **Section 5: What we actually need — graders, not matchers** (markdown only — motivates `Score` / `grader` shape)
8. **"What you just learned"** (markdown bullets)
9. **"What's missing"** (markdown teaser to notebook 02)

---

## Task 1: Create the `evals/` folder and scaffold the empty notebook

**Files:**
- Create: `evals/` (folder)
- Create: `evals/01_why_agents_are_hard_to_test.ipynb`

- [ ] **Step 1: Create the `evals/` folder**

```bash
mkdir evals
ls -la evals
```

Expected: the folder is created and is empty.

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

Save as `evals/01_why_agents_are_hard_to_test.ipynb`.

- [ ] **Step 3: Verify the file is valid JSON and opens as a notebook**

```bash
python -c "import json; json.load(open('evals/01_why_agents_are_hard_to_test.ipynb'))"
ls -la evals/01_why_agents_are_hard_to_test.ipynb
```

Expected: no exception, file non-zero size.

- [ ] **Step 4: Commit**

```bash
git add evals/01_why_agents_are_hard_to_test.ipynb
git commit -m "feat(evals): scaffold 01_why_agents_are_hard_to_test.ipynb"
```

---

## Task 2: Add title and "Why this notebook exists"

**Files:**
- Modify: `evals/01_why_agents_are_hard_to_test.ipynb` (add cell 1)

- [ ] **Step 1: Add a markdown cell with title and motivation**

Cell content (markdown):

```markdown
# 01 — Why Agents Are Hard to Test

## Why this notebook exists

You know how to build agents — the `langgraph/`, `mcp/`, and `a2a/` series in this repo cover wiring them together and making them communicate. But once an agent is running, a new set of questions arrives: *Does it actually work? How do I know it hasn't regressed? What happens when the output changes phrasing but is still semantically right?*

Traditional unit tests answer questions like "does `2 + 2` return `4`?" — the answer is deterministic, there is exactly one correct form, and every step is independent. Agents violate all three assumptions. This notebook makes that concrete before we introduce any tooling: we'll build a tiny toy agent, write the most natural test for it, watch it flake, and understand *why*. That understanding is the foundation everything else in this series builds on.

No API key. No external packages. Just Python.
```

- [ ] **Step 2: Verify the cell renders**

Open the notebook in Jupyter or VS Code and confirm the markdown renders correctly.

- [ ] **Step 3: Commit**

```bash
git add evals/01_why_agents_are_hard_to_test.ipynb
git commit -m "feat(evals): add intro markdown to notebook 01"
```

---

## Task 3: Add "What you'll learn"

**Files:**
- Modify: `evals/01_why_agents_are_hard_to_test.ipynb` (add cell 2)

- [ ] **Step 1: Add a markdown cell**

Cell content (markdown):

```markdown
## What you'll learn

- Why a naive `assert agent(input) == "expected"` unit test breaks for agents even when the agent is *correct*.
- The three root causes that make agent evaluation hard: **non-determinism**, **no single ground truth**, and **multi-step compounding failures**.
- How to quantify flakiness concretely: loop over seeds, count pass vs fail on semantically-correct outputs.
- Why we need **graders** — functions that *score* an output — instead of equality checks, and what the grader shape `(example, output) -> Score` looks like (preview only; the harness is built in notebooks 02–03).
```

- [ ] **Step 2: Verify the cell renders**

The bullets and bold text should display correctly.

- [ ] **Step 3: Commit**

```bash
git add evals/01_why_agents_are_hard_to_test.ipynb
git commit -m "feat(evals): add 'What you'll learn' to notebook 01"
```

---

## Task 4: Section 1 — Setup (imports and seed note)

**Files:**
- Modify: `evals/01_why_agents_are_hard_to_test.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 1. Setup

This notebook uses only the Python standard library: `random` for the stub agent's phrasing variation and `dataclasses` for a preview of the `Score` type in the closing section. No `pip install` needed.

One important note on seeds: the stub agent in this notebook uses a **per-call seeded `random.Random` instance** whose seed is derived from the call number, not from global state. That means:
- Running any single call multiple times gives the same result — it is reproducible.
- Running across a *range* of call numbers gives genuinely different phrasings — it looks non-deterministic.

This is intentional: it lets us demonstrate flakiness in a completely reproducible way.
```

- [ ] **Step 2: Add the setup code cell**

```python
import random
from dataclasses import dataclass, field

# We will use this throughout the notebook.
# No external dependencies — stdlib only.

print("Python stdlib only. No pip install needed.")
print(f"random module: {random.__name__}")
print(f"dataclasses module: {dataclass.__module__}")
```

- [ ] **Step 3: Run the cell**

Expected output:

```
Python stdlib only. No pip install needed.
random module: random
dataclasses module: dataclasses
```

- [ ] **Step 4: Commit**

```bash
git add evals/01_why_agents_are_hard_to_test.ipynb
git commit -m "feat(evals): add setup cell to notebook 01"
```

---

## Task 5: Section 2 — A toy sentiment-classifier agent

**Files:**
- Modify: `evals/01_why_agents_are_hard_to_test.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 2. A Toy Sentiment-Classifier Agent

Our toy agent is a sentiment classifier: given a review string, it decides whether the sentiment is positive or negative. A real LLM would return something like `"positive"` — but it wouldn't always phrase it the *same* way. Sometimes it would say `"Positive!"`, sometimes `"The sentiment is positive."`, sometimes `"positive"`, sometimes `"POSITIVE"`.

We simulate that variability with a seeded `random.Random` instance. The agent always gets the *answer* right (positive reviews return a positive-sentiment string, negative reviews return a negative-sentiment string), but the *phrasing* varies. This is the critical distinction: **semantic correctness ≠ string equality.**

The `call_count` parameter lets us drive different phrasing choices without touching global state, so every output below is fully reproducible.
```

- [ ] **Step 2: Add the agent definition code cell**

```python
# Phrasings the stub agent chooses among for each sentiment.
# All positive phrasings are semantically equivalent; same for negative.
_POSITIVE_PHRASINGS = [
    "positive",
    "Positive",
    "Positive!",
    "POSITIVE",
    "The sentiment is positive.",
    "sentiment: positive",
    "positive sentiment",
]

_NEGATIVE_PHRASINGS = [
    "negative",
    "Negative",
    "Negative!",
    "NEGATIVE",
    "The sentiment is negative.",
    "sentiment: negative",
    "negative sentiment",
]

# Positive signal words. A real LLM would do inference; we just keyword-match
# for simplicity — the agent is always "correct" in that positive reviews get
# a positive phrasing and negative reviews get a negative phrasing.
_POSITIVE_SIGNALS = {"great", "love", "excellent", "amazing", "good", "fantastic",
                     "wonderful", "best", "happy", "perfect"}


def sentiment_agent(review: str, call_count: int = 0) -> str:
    """Toy sentiment classifier that always gives the right ANSWER
    but varies its PHRASING based on call_count.

    Args:
        review: The product review text to classify.
        call_count: Drives phrasing variation. Different values → different
                    surface strings, even for the same review.

    Returns:
        A string expressing the sentiment — phrasing varies.
    """
    words = set(review.lower().split())
    is_positive = bool(words & _POSITIVE_SIGNALS)

    # Seed a fresh Random instance from call_count so output is reproducible
    # but varies across different call_count values.
    rng = random.Random(call_count)

    if is_positive:
        return rng.choice(_POSITIVE_PHRASINGS)
    else:
        return rng.choice(_NEGATIVE_PHRASINGS)


print("Agent defined. Classifies sentiment; phrasing varies by call_count.")
```

- [ ] **Step 3: Add the `### Try it` demonstration cell**

```python
### Try it

POSITIVE_REVIEW = "I love this product, it is great and amazing!"
NEGATIVE_REVIEW = "This is the worst thing I have ever bought."

print("Same positive review, different call_count values:")
for i in range(7):
    output = sentiment_agent(POSITIVE_REVIEW, call_count=i)
    print(f"  call_count={i}  ->  {output!r}")

print()
print("Same negative review, different call_count values:")
for i in range(7):
    output = sentiment_agent(NEGATIVE_REVIEW, call_count=i)
    print(f"  call_count={i}  ->  {output!r}")
```

- [ ] **Step 4: Run both cells and verify**

Expected output for the `### Try it` cell:

```
Same positive review, different call_count values:
  call_count=0  ->  'positive'
  call_count=1  ->  'Positive'
  call_count=2  ->  'positive sentiment'
  call_count=3  ->  'Positive!'
  call_count=4  ->  'POSITIVE'
  call_count=5  ->  'The sentiment is positive.'
  call_count=6  ->  'sentiment: positive'

Same negative review, different call_count values:
  call_count=0  ->  'negative'
  call_count=1  ->  'Negative'
  call_count=2  ->  'negative sentiment'
  call_count=3  ->  'Negative!'
  call_count=4  ->  'NEGATIVE'
  call_count=5  ->  'The sentiment is negative.'
  call_count=6  ->  'sentiment: negative'
```

(The exact phrasing per `call_count` index depends on `random.Random` seed behavior, which is stable across Python 3.x patch versions. If your output order differs slightly, that is acceptable — the key requirement is that all 7 phrasings appear across the 7 calls and no two adjacent rows are identical.)

- [ ] **Step 5: Commit**

```bash
git add evals/01_why_agents_are_hard_to_test.ipynb
git commit -m "feat(evals): add toy sentiment-classifier agent to notebook 01"
```

---

## Task 6: Section 3 — The naive test and why it flakes

**Files:**
- Modify: `evals/01_why_agents_are_hard_to_test.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 3. The Naive Test — and Why It Flakes

The most natural test for our agent is:

```python
assert sentiment_agent(POSITIVE_REVIEW) == "positive"
```

This is exactly how we'd test a deterministic function — and it works for `call_count=0` because the agent happens to return `"positive"` with that seed. But change the seed (which is what happens when calls happen in a different order, or when the LLM's sampling varies), and the *semantically correct* output `"Positive!"` causes the assertion to fail.

Let's make the flakiness concrete: run the same assertion across all 7 phrasing seeds and count how many pass.

> **Gotcha:** A test that passes most of the time is worse than a test that always fails — it creates false confidence and surfaces failures at random moments, making it hard to distinguish a real regression from natural variation.
```

- [ ] **Step 2: Add the code cell**

```python
NUM_SEEDS = 7  # One per phrasing in _POSITIVE_PHRASINGS

passed = 0
failed = 0
failures = []

print(f"Running: assert sentiment_agent(POSITIVE_REVIEW, call_count=i) == 'positive'")
print(f"across {NUM_SEEDS} seeds (one per distinct phrasing):\n")

for i in range(NUM_SEEDS):
    output = sentiment_agent(POSITIVE_REVIEW, call_count=i)
    try:
        assert output == "positive", f"Got {output!r}, expected 'positive'"
        passed += 1
        print(f"  seed {i}: PASS  (output={output!r})")
    except AssertionError as e:
        failed += 1
        failures.append((i, output))
        print(f"  seed {i}: FAIL  ({e})")

print()
print(f"Results: {passed}/{NUM_SEEDS} passed, {failed}/{NUM_SEEDS} failed")
print()
print("Every FAIL is a semantically correct output that the naive test rejects.")
print("The agent is not wrong — the test is.")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Running: assert sentiment_agent(POSITIVE_REVIEW, call_count=i) == 'positive'
across 7 seeds (one per distinct phrasing):

  seed 0: PASS  (output='positive')
  seed 1: FAIL  (Got 'Positive', expected 'positive')
  seed 2: FAIL  (Got 'positive sentiment', expected 'positive')
  seed 3: FAIL  (Got 'Positive!', expected 'positive')
  seed 4: FAIL  (Got 'POSITIVE', expected 'positive')
  seed 5: FAIL  (Got 'The sentiment is positive.', expected 'positive')
  seed 6: FAIL  (Got 'sentiment: positive', expected 'sentiment: positive')

Results: 1/7 passed, 6/7 failed

Every FAIL is a semantically correct output that the naive test rejects.
The agent is not wrong — the test is.
```

(Exact pass/fail per seed depends on phrasing ordering. The requirement is: at least 4 of the 7 seeds fail, and the summary line correctly reflects the counts.)

- [ ] **Step 4: Commit**

```bash
git add evals/01_why_agents_are_hard_to_test.ipynb
git commit -m "feat(evals): add naive-test flakiness demo to notebook 01"
```

---

## Task 7: Section 4 — The three core difficulties

**Files:**
- Modify: `evals/01_why_agents_are_hard_to_test.ipynb` (add 1 section-header markdown cell + 3 subsections, each with 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add the section-header markdown cell**

```markdown
## 4. The Three Core Difficulties

The flaky test above is a symptom of three deeper problems. Each has its own runnable demo below.
```

- [ ] **Step 2: Add subsection 4a markdown cell**

```markdown
### 4a. Non-Determinism — Same Input, Different Surface Text

A real LLM's temperature parameter, sampling order, and context window all affect the exact tokens produced. Our stub models this with `call_count`. The *meaning* is stable; the *string* is not. Any test that compares strings will therefore be fragile.

The cell below calls the agent 10 times on the same input, collects all unique surface strings, and prints the semantic verdict for each. Same input, same correct answer, many distinct strings.
```

- [ ] **Step 3: Add subsection 4a code cell**

```python
# 4a: Non-determinism demo
review = "This product is excellent and I love it!"

outputs = [sentiment_agent(review, call_count=i) for i in range(10)]
unique_outputs = sorted(set(outputs))

print(f"Input: {review!r}")
print(f"10 calls → {len(unique_outputs)} distinct surface strings:\n")
for s in unique_outputs:
    print(f"  {s!r}")

print()
print("All are semantically 'positive'. None are equal to each other.")
print(f"String equality test would pass {outputs.count('positive')}/10 times.")
```

Expected output:

```
Input: 'This product is excellent and I love it!'
10 calls → 7 distinct surface strings:

  'POSITIVE'
  'Positive'
  'Positive!'
  'The sentiment is positive.'
  'positive'
  'positive sentiment'
  'sentiment: positive'

All are semantically 'positive'. None are equal to each other.
String equality test would pass 1/10 times.
```

- [ ] **Step 4: Add subsection 4b markdown cell**

```markdown
### 4b. No Single Ground Truth — Many Correct Outputs

For tasks like summarization, explanation, or translation there is no single canonical correct output. Two summaries can both be excellent and completely non-identical. Storing one as the "golden answer" and asserting equality would reject all the other good answers.

The cell below constructs two clearly-good one-sentence summaries of the same article, shows they are not equal, and shows that neither is a substring of the other — so `in`-tests also fail.
```

- [ ] **Step 5: Add subsection 4b code cell**

```python
# 4b: No single ground truth demo
ARTICLE = (
    "Researchers at Stanford have developed a new battery technology "
    "that charges in under five minutes and lasts three times longer "
    "than current lithium-ion cells, potentially transforming electric vehicles."
)

# Two equally-valid one-sentence summaries a capable LLM might produce.
SUMMARY_A = (
    "Stanford researchers developed a fast-charging battery that lasts "
    "three times longer than lithium-ion, which could revolutionize EVs."
)
SUMMARY_B = (
    "A new Stanford battery technology charges in under five minutes "
    "and offers triple the longevity of lithium-ion cells."
)

print("Article:")
print(f"  {ARTICLE}\n")
print("Summary A:")
print(f"  {SUMMARY_A}\n")
print("Summary B:")
print(f"  {SUMMARY_B}\n")

print(f"Are they equal?             {SUMMARY_A == SUMMARY_B}")
print(f"Is B a substring of A?      {SUMMARY_B in SUMMARY_A}")
print(f"Is A a substring of B?      {SUMMARY_A in SUMMARY_B}")
print()
print("Both summaries are accurate and high quality.")
print("No equality or substring test can accept both and reject bad ones.")
```

Expected output:

```
Article:
  Researchers at Stanford have developed a new battery technology that charges in under five minutes and lasts three times longer than current lithium-ion cells, potentially transforming electric vehicles.

Summary A:
  Stanford researchers developed a fast-charging battery that lasts three times longer than lithium-ion, which could revolutionize EVs.

Summary B:
  A new Stanford battery technology charges in under five minutes and offers triple the longevity of lithium-ion cells.

Are they equal?             False
Is B a substring of A?      False
Is A a substring of B?      False

Both summaries are accurate and high quality.
No equality or substring test can accept both and reject bad ones.
```

- [ ] **Step 6: Add subsection 4c markdown cell**

```markdown
### 4c. Multi-Step Compounding — One Variation Breaks the Next Step

Real agents chain steps: an LLM extracts information, then downstream code parses it, then another LLM uses the parsed result. If step 1's output varies in phrasing, step 2's brittle parser may break — even though step 1 was semantically correct. The error compounds: by the time something visibly goes wrong, the failure is two steps removed from its actual cause.

The cell below models a two-step pipeline:
- **Step 1 (agent):** classifies sentiment — output phrasing varies.
- **Step 2 (parser):** extracts a boolean `is_positive` from step 1's output via a brittle exact-string check.

Step 2 works for one phrasing and silently produces the wrong boolean for all others.
```

- [ ] **Step 7: Add subsection 4c code cell**

```python
# 4c: Multi-step compounding demo

def step1_classify(review: str, call_count: int = 0) -> str:
    """Step 1: sentiment classification (phrasing varies)."""
    return sentiment_agent(review, call_count=call_count)


def step2_parse(classification: str) -> bool:
    """Step 2: brittle parser — only recognises the bare lowercase string."""
    # Real downstream code often looks like this after a quick prototype.
    return classification == "positive"


def pipeline(review: str, call_count: int = 0) -> dict:
    raw = step1_classify(review, call_count=call_count)
    is_positive = step2_parse(raw)
    return {"raw": raw, "is_positive": is_positive}


review = POSITIVE_REVIEW
print(f"Input: {review!r}\n")
print(f"{'call_count':<12} {'step1 output':<35} {'step2 result':<14} {'correct?'}")
print("-" * 75)

for i in range(7):
    result = pipeline(review, call_count=i)
    correct = result["is_positive"]  # should always be True for a positive review
    marker = "OK" if correct else "BUG <-- compounding failure"
    print(f"{i:<12} {result['raw']:<35} {str(result['is_positive']):<14} {marker}")

print()
print("Step 1 is always semantically correct.")
print("Step 2 breaks silently on 6/7 phrasings — the bug is invisible at step 1.")
```

Expected output:

```
Input: 'I love this product, it is great and amazing!'

call_count   step1 output                        step2 result   correct?
---------------------------------------------------------------------------
0            positive                            True           OK
1            Positive                            False          BUG <-- compounding failure
2            positive sentiment                  False          BUG <-- compounding failure
3            Positive!                           False          BUG <-- compounding failure
4            POSITIVE                            False          BUG <-- compounding failure
5            The sentiment is positive.          False          BUG <-- compounding failure
6            sentiment: positive                 False          BUG <-- compounding failure

Step 1 is always semantically correct.
Step 2 breaks silently on 6/7 phrasings — the bug is invisible at step 1.
```

- [ ] **Step 8: Commit**

```bash
git add evals/01_why_agents_are_hard_to_test.ipynb
git commit -m "feat(evals): add three-difficulties demos to notebook 01"
```

---

## Task 8: Section 5, closing recap, "What's missing" teaser

**Files:**
- Modify: `evals/01_why_agents_are_hard_to_test.ipynb` (add 3 markdown cells)

- [ ] **Step 1: Add the "What we actually need" markdown cell (Section 5)**

```markdown
## 5. What We Actually Need — Graders, Not Matchers

The three demos above share a common shape: *we know what a good output looks like, but we can't express that knowledge as a string equality check.* What we need instead is a **grader** — a function that takes an output and returns a *score* rather than a boolean pass/fail.

A grader for the sentiment classifier might look like:

```python
def sentiment_grader(example, output) -> Score:
    label = output.lower()
    passed = example["expected_label"] in label   # "positive" anywhere in output
    return Score(
        key="sentiment_correct",
        score=1.0 if passed else 0.0,
        passed=passed,
        comment=f"output={output!r}",
    )
```

Notice the shape: `grader(example, output) -> Score`. The `Score` dataclass (introduced in notebook 02) holds:
- `key` — which grader produced this score.
- `score` — a float in `[0, 1]`.
- `passed` — a boolean threshold judgment.
- `comment` — optional explanation.

And `example` (introduced in notebook 03) is an `{input, expected, metadata}` bundle — one row from your eval dataset.

We're not implementing these yet. We're just naming the shapes so the rest of the series can refer back to this moment: the point where we realized `assert equals` wasn't enough and decided to build something better.
```

- [ ] **Step 2: Add the "What you just learned" markdown cell**

```markdown
## What you just learned

- A toy sentiment agent can always produce the *right answer* while failing a naive `assert output == "positive"` test — because the output phrasing varies even when the semantics are correct.
- **Non-determinism:** the same input can produce many distinct surface strings; equality tests are fragile.
- **No single ground truth:** for open-ended outputs like summaries, many correct forms exist; no single golden string covers them.
- **Multi-step compounding:** a brittle parser in step 2 can silently produce wrong results whenever step 1's phrasing varies, making bugs hard to locate.
- We need **graders** — functions of shape `(example, output) -> Score` — that *score* outputs rather than match them, so semantically-correct variation doesn't register as failure.
```

- [ ] **Step 3: Add the "What's missing" markdown cell**

```markdown
## What's missing

We've named the problem and sketched the grader shape, but we haven't built any graders yet. The `Score` dataclass is undefined, `sentiment_grader` above is pseudocode, and we have no way to run a grader across a batch of examples.

**Notebook 02 — `02_assertions_and_golden_outputs.ipynb`** builds the first real graders: exact match, `contains`/regex match, and structured-output validation with `pydantic`. Each returns a proper `Score`. By the end of notebook 02 you'll have a small toolkit of deterministic graders that handle most of the cases where `assert equals` fails — no LLM required.
```

- [ ] **Step 4: Commit**

```bash
git add evals/01_why_agents_are_hard_to_test.ipynb
git commit -m "feat(evals): add grader motivation and closing cells to notebook 01"
```

---

## Task 9: End-to-end verification in a fresh kernel

**Files:** none (verification only).

- [ ] **Step 1: Execute the notebook top-to-bottom in a clean kernel**

```bash
jupyter nbconvert --to notebook --execute --inplace evals/01_why_agents_are_hard_to_test.ipynb
```

Expected: succeeds with no errors. The notebook is rewritten with embedded outputs from the fresh run.

- [ ] **Step 2: Verify expected outputs in the executed notebook**

The output cells should contain, in order:

1. Setup cell:
   ```
   Python stdlib only. No pip install needed.
   random module: random
   dataclasses module: dataclasses
   ```

2. Agent definition cell:
   ```
   Agent defined. Classifies sentiment; phrasing varies by call_count.
   ```

3. `### Try it` demonstration cell — 7 positive phrasings and 7 negative phrasings, all distinct within each group, none raising an exception.

4. Naive-test flakiness cell — at least 4 of 7 seeds fail; the summary line `Results: N/7 passed, M/7 failed` with `N < 3`.

5. Section 4a non-determinism cell — 7 distinct surface strings listed, string equality count of 1/10.

6. Section 4b no-single-ground-truth cell — `Are they equal?  False`, both substring checks `False`.

7. Section 4c multi-step compounding cell — call_count 0 shows `OK`, call_counts 1–6 show `BUG <-- compounding failure`.

No cell raises an unhandled exception.

- [ ] **Step 3: Commit the clean run**

```bash
git add evals/01_why_agents_are_hard_to_test.ipynb
git commit -m "chore(evals): commit clean fresh-kernel run of notebook 01"
```

---

## Done

After Task 9 passes, notebook 01 is complete. The next plan to write is `2026-05-24-evals-notebook-02-assertions-and-golden-outputs.md`, which introduces the `Score` dataclass, `Example` type, and the first deterministic graders (exact match, contains/regex, `pydantic` structured-output validation, and golden-file diffs) — all still no-key, running against the same stub agent from this notebook.
