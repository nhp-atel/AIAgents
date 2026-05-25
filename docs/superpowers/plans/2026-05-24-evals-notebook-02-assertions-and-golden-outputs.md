# Evals Notebook 02 — `02_assertions_and_golden_outputs.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the second notebook in the Evals & Observability learning series — introduce the four families of **deterministic graders** (exact/contains/regex matching, structured-output validation, golden outputs, and a composing loop) and the shared `Score` dataclass that all graders return. No API key — every grader runs against pre-recorded, stubbed strings.

**Architecture:** A single self-contained Jupyter notebook. All code is inline — no shared importable module. Agent outputs are deterministic strings or dicts defined in-cell. Graders are plain callables. A temporary golden file is written to the OS scratch directory and cleaned up at the end.

**Tech Stack:** Python 3.11+, Jupyter, `dataclasses` (stdlib), `re` (stdlib), `pathlib` (stdlib), `pydantic` v2.

**Companion spec:** `docs/superpowers/specs/2026-05-24-evals-observability-learning-series-design.md` (notebook 02 section).

**Folder:** `evals/` already exists (created in notebook 01's plan). Task 1 here scaffolds the empty notebook JSON only — no `mkdir` needed.

---

## File Structure

- **Create:** `evals/02_assertions_and_golden_outputs.ipynb` — the entire notebook, self-contained.
- **Modify:** none.
- **No separate test files** — verification is the notebook running top-to-bottom in a fresh kernel (Task 9).

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown)
2. **"What you'll learn"** (markdown bullets)
3. **Section 1: Setup** (markdown + 1 code cell — imports + `Score` dataclass definition)
4. **Section 2: Exact / contains / regex matching** (markdown + 2 code cells — grader definitions + pass/fail demos)
5. **Section 3: Structured-output validation with pydantic** (markdown + 2 code cells — `Classification` model + `make_schema_grader` + pass/fail demo)
6. **Section 4: Golden outputs** (markdown + 3 code cells — write golden file, `make_golden_grader`, pass/fail demo, mutation demo)
7. **Section 5: All graders share one signature** (markdown + 1 code cell — composing loop over one output)
8. **"What you just learned"** (markdown)
9. **"What's missing"** (markdown teaser pointing to notebook 03)
10. **Cleanup** (markdown + code — delete temp golden files)

---

## Task 1: Scaffold the empty notebook

**Files:**
- Create: `evals/02_assertions_and_golden_outputs.ipynb`

- [ ] **Step 1: Verify `evals/` already exists**

```bash
ls -la evals/
```

Expected: directory exists, contains at least `01_why_agents_are_hard_to_test.ipynb`. Do NOT create the folder.

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

Save as `evals/02_assertions_and_golden_outputs.ipynb`.

- [ ] **Step 3: Verify the file is valid JSON**

```bash
python -c "import json; json.load(open('evals/02_assertions_and_golden_outputs.ipynb'))"
ls -la evals/02_assertions_and_golden_outputs.ipynb
```

Expected: no exception, file non-zero size.

- [ ] **Step 4: Commit**

```bash
git add evals/02_assertions_and_golden_outputs.ipynb
git commit -m "feat(evals): scaffold 02_assertions_and_golden_outputs.ipynb"
```

---

## Task 2: Add title and "Why this notebook exists"

**Files:**
- Modify: `evals/02_assertions_and_golden_outputs.ipynb` (add cell 1)

- [ ] **Step 1: Add a markdown cell with title and motivation**

Cell content (markdown):

```markdown
# 02 — Assertions and Golden Outputs

## Why this notebook exists

In **`01_why_agents_are_hard_to_test.ipynb`** we saw how a naive `assert agent(x) == "expected"` unit test flakes the moment an agent varies its phrasing — and we named the three forces behind that fragility: non-determinism, no single ground truth, and multi-step compounding failures. The closing insight was: *"We can't `assert equals` our way out of this. We need graders that score outputs, not match them exactly."*

This notebook builds those graders — the cheapest, most reliable layer of the eval stack. Each grader is a plain callable that accepts an example and a candidate output and returns a `Score`. No LLM, no external service, no API key required.
```

- [ ] **Step 2: Verify the cell renders**

Open the notebook in Jupyter/VS Code and confirm the cell renders as markdown.

- [ ] **Step 3: Commit**

```bash
git add evals/02_assertions_and_golden_outputs.ipynb
git commit -m "feat(evals): add intro markdown to notebook 02"
```

---

## Task 3: Add "What you'll learn"

**Files:**
- Modify: `evals/02_assertions_and_golden_outputs.ipynb` (add cell 2)

- [ ] **Step 1: Add a markdown cell**

Cell content (markdown):

```markdown
## What you'll learn

- The `Score` dataclass — the shared return type every grader in this series produces.
- When **exact match** is appropriate (structured outputs, deterministic pipelines) and why it's brittle for free-text.
- How **contains** and **regex** graders give you partial-match power without LLM overhead.
- How to validate **structured outputs** with `pydantic` v2 — confirming an agent's JSON parses into the expected schema, and capturing the validation error when it doesn't.
- The discipline of **golden outputs**: storing a known-good string to a file and diffing against it — and why keeping goldens from silently going stale is as important as writing them.
- How all four grader families share a single `(example, output) -> Score` signature, so they compose transparently in notebook 03's harness.
```

- [ ] **Step 2: Verify the cell renders**

- [ ] **Step 3: Commit**

```bash
git add evals/02_assertions_and_golden_outputs.ipynb
git commit -m "feat(evals): add 'What you'll learn' to notebook 02"
```

---

## Task 4: Section 1 — Setup (imports + `Score` dataclass)

**Files:**
- Modify: `evals/02_assertions_and_golden_outputs.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 1. Setup

All graders in this notebook share one return type: `Score`. Defining it here, once, as a `dataclass` gives us a consistent interface that notebook 03 will rely on when it builds the harness.

**Compatibility note:** In this notebook examples are plain dicts — `{"input": ..., "expected": ...}` — and graders read `example["expected"]`. Notebook 03 will formalize `Example` as a proper dataclass with `.input` and `.expected` attributes, at which point `example["expected"]` becomes `example.expected`. The grader signatures don't change — only how you access the field. This is called out again at each grader definition below.

If `pydantic` isn't installed yet:
```bash
pip install pydantic
```
```

- [ ] **Step 2: Add the setup code cell**

```python
import re
import tempfile
from dataclasses import dataclass
from pathlib import Path

from pydantic import BaseModel, ValidationError
from typing import Literal

# Track every temp file written in this notebook for cleanup at the end.
_temp_files: list[Path] = []


@dataclass
class Score:
    key: str            # grader identifier, e.g. "exact_match"
    score: float        # normalized to [0.0, 1.0]
    passed: bool
    comment: str = ""


print("Setup OK")
print(f"Score fields: {[f.name for f in Score.__dataclass_fields__.values()]}")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
Setup OK
Score fields: ['key', 'score', 'passed', 'comment']
```

If `ModuleNotFoundError: No module named 'pydantic'` appears, run `pip install pydantic` and re-run.

- [ ] **Step 4: Commit**

```bash
git add evals/02_assertions_and_golden_outputs.ipynb
git commit -m "feat(evals): add setup cell with Score dataclass to notebook 02"
```

---

## Task 5: Section 2 — Exact / contains / regex matching

**Files:**
- Modify: `evals/02_assertions_and_golden_outputs.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 2. Exact / Contains / Regex Matching

The three cheapest graders form a spectrum:

| Grader | Passes when | Best for |
|---|---|---|
| `exact_match` | `output == expected` exactly | Structured strings, enum labels, short deterministic answers |
| `make_contains(substring)` | `substring in output` | Checking that a required phrase or field name appears anywhere |
| `make_regex(pattern)` | `re.search(pattern, output)` matches | Format constraints — dates, codes, capitalization, numeric ranges |

**Exact match is brittle for free text** — a single trailing space or synonym flips it to fail. Its strength is precisely that brittleness: when an output *should* be deterministic (a classification label, a formatted code), exact match punishes any deviation.

Each grader accepts `(example, output)` where `example` is a dict `{"input": ..., "expected": ...}`. The `example["expected"]` field is used by `exact_match`; the contains/regex graders ignore `expected` and grade the raw output against the substring/pattern they were constructed with. In notebook 03, `example` becomes an `Example` dataclass and `example["expected"]` becomes `example.expected` — the grader body doesn't change.
```

- [ ] **Step 2: Add the grader definitions code cell**

```python
# ── Grader 1: exact_match ────────────────────────────────────────────────────

def exact_match(example: dict, output: str) -> Score:
    """Pass iff output equals example['expected'] exactly."""
    expected = example["expected"]
    passed = output == expected
    return Score(
        key="exact_match",
        score=1.0 if passed else 0.0,
        passed=passed,
        comment="" if passed else f"expected {expected!r}, got {output!r}",
    )


# ── Grader 2: make_contains ──────────────────────────────────────────────────

def make_contains(substring: str, key: str = "contains"):
    """Factory: return a grader that passes when `substring` appears in output."""
    def grader(example: dict, output: str) -> Score:
        passed = substring in output
        return Score(
            key=key,
            score=1.0 if passed else 0.0,
            passed=passed,
            comment="" if passed else f"{substring!r} not found in output",
        )
    grader.__name__ = key
    return grader


# ── Grader 3: make_regex ─────────────────────────────────────────────────────

def make_regex(pattern: str, key: str = "regex"):
    """Factory: return a grader that passes when `pattern` matches anywhere in output."""
    compiled = re.compile(pattern)
    def grader(example: dict, output: str) -> Score:
        match = compiled.search(output)
        passed = match is not None
        return Score(
            key=key,
            score=1.0 if passed else 0.0,
            passed=passed,
            comment="" if passed else f"pattern {pattern!r} not found in output",
        )
    grader.__name__ = key
    return grader


print("Graders defined: exact_match, make_contains, make_regex")
```

- [ ] **Step 3: Run the cell**

Expected output:

```
Graders defined: exact_match, make_contains, make_regex
```

- [ ] **Step 4: Add the `### Try it` demo code cell**

```python
### Try it

# Stubbed agent outputs — no API call needed.
output_correct   = "positive"
output_wrong     = "Positive"   # different capitalisation — exact_match will catch this
output_verbose   = "The sentiment is positive, with high confidence."

example = {"input": "The food was great!", "expected": "positive"}

# exact_match: pass and fail
s1 = exact_match(example, output_correct)
s2 = exact_match(example, output_wrong)
print(f"exact_match on {output_correct!r}: passed={s1.passed}, score={s1.score}")
print(f"exact_match on {output_wrong!r}:  passed={s2.passed}, comment={s2.comment!r}")
print()

# contains: checks that "positive" appears somewhere, even in a verbose answer
contains_positive = make_contains("positive", key="contains_positive")
s3 = contains_positive(example, output_verbose)
s4 = contains_positive(example, "The sentiment is negative.")
print(f"contains on verbose:   passed={s3.passed}")
print(f"contains on negative:  passed={s4.passed}, comment={s4.comment!r}")
print()

# regex: require the output to be exactly one of our label words (full-string match)
label_regex = make_regex(r"^(positive|negative|neutral)$", key="label_format")
s5 = label_regex(example, output_correct)
s6 = label_regex(example, output_verbose)
print(f"regex on {output_correct!r}:  passed={s5.passed}")
print(f"regex on verbose:     passed={s6.passed}, comment={s6.comment!r}")
```

- [ ] **Step 5: Run the cell and verify**

Expected output:

```
exact_match on 'positive': passed=True, score=1.0
exact_match on 'Positive':  passed=False, comment="expected 'positive', got 'Positive'"

contains on verbose:   passed=True
contains on negative:  passed=False, comment="'positive' not found in output"

regex on 'positive':  passed=True
regex on verbose:     passed=False, comment="pattern '^(positive|negative|neutral)$' not found in output"
```

- [ ] **Step 6: Commit**

```bash
git add evals/02_assertions_and_golden_outputs.ipynb
git commit -m "feat(evals): add exact/contains/regex graders to notebook 02"
```

---

## Task 6: Section 3 — Structured-output validation with pydantic

**Files:**
- Modify: `evals/02_assertions_and_golden_outputs.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 3. Structured-Output Validation with Pydantic

Exact match breaks the moment an agent's JSON output has an extra space or differently-ordered keys. What we actually care about is: *does the output represent a valid instance of the expected schema?*

`pydantic` v2's `model_validate_json` (for raw JSON strings) or `model_validate` (for dicts) raises a `ValidationError` with a precise error message when the output is invalid. We capture that message in `Score.comment` so a failing eval tells you *why* it failed — wrong field type, missing required field, value outside a `Literal`, etc.

`make_schema_grader(model)` is a factory: pass it a pydantic `BaseModel` subclass and get back a grader. The grader tries to parse the output; if validation succeeds, `passed=True`; otherwise `passed=False` and `comment` carries the first validation error.
```

- [ ] **Step 2: Add the model + grader definition code cell**

```python
# ── Pydantic model used as the schema under test ─────────────────────────────

class Classification(BaseModel):
    label: Literal["positive", "negative", "neutral"]
    confidence: float  # pydantic will enforce float; we don't constrain range here


# ── Grader 4: make_schema_grader ─────────────────────────────────────────────

def make_schema_grader(model, key: str = "valid_schema"):
    """Factory: return a grader that passes when `output` is valid JSON for `model`.

    Tries model.model_validate_json(output) first (raw JSON string); if `output`
    is already a dict, falls back to model.model_validate(output).
    On ValidationError, passed=False and comment holds the error detail.
    """
    def grader(example: dict, output) -> Score:
        try:
            if isinstance(output, str):
                model.model_validate_json(output)
            else:
                model.model_validate(output)
            return Score(key=key, score=1.0, passed=True)
        except ValidationError as exc:
            first_error = exc.errors(include_url=False)[0]
            comment = f"{first_error['loc']}: {first_error['msg']}"
            return Score(key=key, score=0.0, passed=False, comment=comment)
        except Exception as exc:
            return Score(key=key, score=0.0, passed=False, comment=str(exc))
    grader.__name__ = key
    return grader


schema_grader = make_schema_grader(Classification, key="valid_schema")
print("Classification model and schema_grader defined")
```

- [ ] **Step 3: Run the cell**

Expected output:

```
Classification model and schema_grader defined
```

- [ ] **Step 4: Add the `### Try it` demo code cell**

```python
### Try it

example = {"input": "Review text here.", "expected": None}  # schema grader ignores expected

# Pass: well-formed JSON matching the schema
valid_output = '{"label": "positive", "confidence": 0.92}'
s1 = schema_grader(example, valid_output)
print(f"valid JSON:          passed={s1.passed}, score={s1.score}")

# Fail: wrong label value (not in Literal)
bad_label = '{"label": "POSITIVE", "confidence": 0.92}'
s2 = schema_grader(example, bad_label)
print(f"wrong label case:    passed={s2.passed}, comment={s2.comment!r}")

# Fail: missing required field
missing_field = '{"label": "negative"}'
s3 = schema_grader(example, missing_field)
print(f"missing confidence:  passed={s3.passed}, comment={s3.comment!r}")

# Fail: malformed JSON
malformed = "label=positive confidence=0.9"
s4 = schema_grader(example, malformed)
print(f"malformed JSON:      passed={s4.passed}, comment={s4.comment!r}")
```

- [ ] **Step 5: Run the cell and verify**

Expected output (exact pydantic error wording may vary slightly across pydantic patch releases; the `passed` values and the presence of a non-empty `comment` on failures are the invariants):

```
valid JSON:          passed=True, score=1.0
wrong label case:    passed=False, comment="('label',): Value error, Input should be 'positive', 'negative' or 'neutral'"
missing confidence:  passed=False, comment="('confidence',): Field required"
malformed JSON:      passed=False, comment="('label',): Field required"
```

> **Gotcha:** pydantic v1 uses `parse_raw` / `parse_obj`; `model_validate_json` is v2-only. If you see `AttributeError: type object 'Classification' has no attribute 'model_validate_json'`, you have pydantic v1 installed. Run `pip install --upgrade pydantic` to get v2.

- [ ] **Step 6: Commit**

```bash
git add evals/02_assertions_and_golden_outputs.ipynb
git commit -m "feat(evals): add pydantic schema grader to notebook 02"
```

---

## Task 7: Section 4 — Golden outputs

**Files:**
- Modify: `evals/02_assertions_and_golden_outputs.ipynb` (add 1 markdown cell + 3 code cells)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 4. Golden Outputs

A golden output is simply a **known-good agent response stored to a file**. The grader reads the file and checks for exact equality. This is powerful for long, structured outputs (multi-paragraph summaries, formatted reports) where you want to detect *any* change — including formatting drift — without rewriting the expected string in code every time.

The discipline has two parts:

1. **Writing goldens intentionally.** A golden file is a contract: this is what the agent should produce. Don't auto-generate golden files from the current agent output without reviewing them first.
2. **Updating goldens intentionally.** When the agent legitimately improves, update the golden and commit the diff. If goldens are silently auto-updated by CI, they stop catching regressions.

> **Gotcha:** Silent golden rot. If your build pipeline regenerates goldens automatically on every run, a regressed agent "passes" forever. The golden file should be a checked-in artifact reviewed at PR time, not an output artifact.
```

- [ ] **Step 2: Add the golden file setup and grader definition code cell**

```python
# ── Write a known-good golden file ───────────────────────────────────────────

GOLDEN_CONTENT = (
    "Summary: The product received overwhelmingly positive reviews. "
    "Customers highlighted fast shipping, durable packaging, and responsive support."
)

golden_fd = tempfile.NamedTemporaryFile(
    mode="w",
    suffix="_golden.txt",
    prefix="evals02_",
    delete=False,
)
golden_path = Path(golden_fd.name)
golden_fd.write(GOLDEN_CONTENT)
golden_fd.close()
_temp_files.append(golden_path)

print(f"Golden file written: {golden_path}")
print(f"Content: {golden_path.read_text()!r}")
print()


# ── Grader 5: make_golden_grader ─────────────────────────────────────────────

def make_golden_grader(golden_path, key: str = "golden"):
    """Factory: return a grader that passes when output equals the golden file's content exactly."""
    path = Path(golden_path)
    def grader(example: dict, output: str) -> Score:
        expected = path.read_text()
        passed = output == expected
        return Score(
            key=key,
            score=1.0 if passed else 0.0,
            passed=passed,
            comment="" if passed else (
                f"output differs from golden ({golden_path}): "
                f"first diff at char {next((i for i,(a,b) in enumerate(zip(expected, output)) if a != b), min(len(expected), len(output)))}"
            ),
        )
    grader.__name__ = key
    return grader


golden_grader = make_golden_grader(golden_path, key="golden")
print("make_golden_grader defined")
```

- [ ] **Step 3: Run the cell and verify**

Expected output (the exact temp filename will vary; prefix `evals02_` and suffix `_golden.txt` are stable):

```
Golden file written: /var/folders/.../evals02_XXXXXXXX_golden.txt
Content: 'Summary: The product received overwhelmingly positive reviews. Customers highlighted fast shipping, durable packaging, and responsive support.'

make_golden_grader defined
```

- [ ] **Step 4: Add the `### Try it` demo code cell**

```python
### Try it

example = {"input": "Summarize the reviews.", "expected": None}  # golden grader reads from file

# Pass: output matches the golden exactly
s1 = golden_grader(example, GOLDEN_CONTENT)
print(f"exact match:      passed={s1.passed}, score={s1.score}")

# Fail: output has a minor mutation (trailing period changed to ellipsis)
mutated_output = GOLDEN_CONTENT.replace("support.", "support...")
s2 = golden_grader(example, mutated_output)
print(f"mutated output:   passed={s2.passed}")
print(f"  comment: {s2.comment}")

# Fail: output truncated
truncated_output = GOLDEN_CONTENT[:60]
s3 = golden_grader(example, truncated_output)
print(f"truncated output: passed={s3.passed}")
print(f"  comment: {s3.comment}")
```

- [ ] **Step 5: Run the cell and verify**

Expected output (the golden file path in the comment will vary):

```
exact match:      passed=True, score=1.0
mutated output:   passed=False
  comment: output differs from golden (/var/folders/.../evals02_XXXXXXXX_golden.txt): first diff at char 139
truncated output: passed=False
  comment: output differs from golden (/var/folders/.../evals02_XXXXXXXX_golden.txt): first diff at char 60
```

- [ ] **Step 6: Add the intentional-update discussion code cell**

```python
### Updating a golden intentionally

# Suppose the agent has legitimately improved. The workflow is:
#   1. Review the new output and confirm it is better.
#   2. Overwrite the golden file.
#   3. Commit the diff — the PR diff shows exactly what changed.

new_golden_content = (
    "Summary: The product received overwhelmingly positive reviews. "
    "Customers highlighted fast shipping, durable packaging, and responsive support. "
    "No recurring complaints were found."
)

# Simulate the intentional update:
golden_path.write_text(new_golden_content)
print(f"Golden updated. New content: {golden_path.read_text()!r}")
print()

# Old output now fails (correct — it no longer matches the improved golden):
s_old = golden_grader(example, GOLDEN_CONTENT)
print(f"old output vs updated golden: passed={s_old.passed}")

# New output passes:
s_new = golden_grader(example, new_golden_content)
print(f"new output vs updated golden: passed={s_new.passed}, score={s_new.score}")
```

- [ ] **Step 7: Run the cell and verify**

Expected output:

```
Golden updated. New content: 'Summary: The product received overwhelmingly positive reviews. Customers highlighted fast shipping, durable packaging, and responsive support. No recurring complaints were found.'

old output vs updated golden: passed=False
new output vs updated golden: passed=True, score=1.0
```

- [ ] **Step 8: Commit**

```bash
git add evals/02_assertions_and_golden_outputs.ipynb
git commit -m "feat(evals): add golden output grader to notebook 02"
```

---

## Task 8: Section 5 — All graders share one signature

**Files:**
- Modify: `evals/02_assertions_and_golden_outputs.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add a markdown cell**

```markdown
## 5. Each Grader Returns a `Score` — They Compose

Every grader we defined — `exact_match`, `make_contains(...)`, `make_regex(...)`, `make_schema_grader(...)`, `make_golden_grader(...)` — has the same signature:

```python
grader(example: dict, output) -> Score
```

That uniformity is the point. Notebook 03 will build a `run_eval(agent, dataset, graders)` harness that iterates over a list of `(input, expected)` examples, calls the agent, and then passes the same `(example, output)` pair to each grader in the list — no special-casing needed.

Here we preview that composition with a single output and a mixed grader list.
```

- [ ] **Step 2: Add the composing loop code cell**

```python
### Try it — run all graders over one output

# A classification agent output we want to evaluate multiple ways simultaneously.
output_under_test = '{"label": "positive", "confidence": 0.87}'
example_under_test = {
    "input": "The checkout experience was smooth and fast.",
    "expected": '{"label": "positive", "confidence": 0.87}',
}

graders = [
    exact_match,
    make_contains("positive", key="contains_positive"),
    make_regex(r'"confidence":\s*0\.\d+', key="has_confidence_field"),
    make_schema_grader(Classification, key="valid_schema"),
    # golden_grader is omitted here since the golden file holds a different domain's output;
    # in notebook 03 each example will carry its own grader list.
]

print(f"{'key':<30} {'passed':<8} {'score':<8} comment")
print("-" * 72)
for grader in graders:
    s = grader(example_under_test, output_under_test)
    print(f"{s.key:<30} {str(s.passed):<8} {s.score:<8.1f} {s.comment}")
```

- [ ] **Step 3: Run the cell and verify**

Expected output:

```
key                            passed   score    comment
------------------------------------------------------------------------
exact_match                    True     1.0      
contains_positive              True     1.0      
has_confidence_field           True     1.0      
valid_schema                   True     1.0      
```

- [ ] **Step 4: Commit**

```bash
git add evals/02_assertions_and_golden_outputs.ipynb
git commit -m "feat(evals): add grader composition demo to notebook 02"
```

---

## Task 9: Closing recap, "What's missing", and cleanup

**Files:**
- Modify: `evals/02_assertions_and_golden_outputs.ipynb` (add 2 markdown cells + 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add the "What you just learned" markdown cell**

```markdown
## What you just learned

- **`Score`** — `key / score / passed / comment` — is the single return type every grader in this series produces. Defining it once means graders, harnesses, and reporters share a language.
- **`exact_match`** is unforgiving and valuable: it detects any deviation in a deterministic output, including case differences, trailing whitespace, or synonym substitution.
- **`make_contains`** and **`make_regex`** give partial-match power without LLM cost — useful for checking that required phrases, field names, or format constraints appear in a longer output.
- **`make_schema_grader`** (pydantic v2) validates structured output: it confirms the output is valid JSON that conforms to your schema, and when it doesn't, `Score.comment` tells you which field failed and why.
- **`make_golden_grader`** catches any change — including formatting drift — against a stored known-good reference. The discipline: write goldens intentionally, update them intentionally, commit the diff.
- All five grader families share `(example, output) -> Score`. That signature is the contract notebook 03 depends on.
```

- [ ] **Step 2: Add the "What's missing" markdown cell**

```markdown
## What's missing

We now have five graders, but we've been calling them one-by-one against hand-written stub outputs. In the real world you have a *dataset* — a list of `(input, expected)` pairs — and you want to run every grader over every example automatically, aggregate the results, and surface a pass rate.

**`03_building_an_eval_harness.ipynb`** does exactly that. It formalizes the `Example` dataclass (`.input` and `.expected` attributes replacing the `example["expected"]` dict access you've seen here), builds a `run_eval(agent, dataset, graders) -> results` runner, adds aggregate metrics (pass rate, mean score, per-grader breakdown), and presents the results as a readable table. The graders you defined here plug in without modification.
```

- [ ] **Step 3: Add a "Cleanup" markdown cell**

```markdown
## Cleanup

The temp golden files written in Section 4 live in the OS scratch directory. Deleting them here keeps a fresh-kernel re-run idempotent.
```

- [ ] **Step 4: Add the cleanup code cell**

```python
removed: list[str] = []
for path in list(_temp_files):
    try:
        path.unlink()
        removed.append(str(path))
    except FileNotFoundError:
        pass
_temp_files.clear()

print(f"Removed {len(removed)} temp file(s):")
for r in removed:
    print(f"  - {r}")
```

- [ ] **Step 5: Run the cleanup cell and verify**

Expected output (filename will vary):

```
Removed 1 temp file(s):
  - /var/folders/.../evals02_XXXXXXXX_golden.txt
```

- [ ] **Step 6: Commit**

```bash
git add evals/02_assertions_and_golden_outputs.ipynb
git commit -m "feat(evals): add closing recap, teaser, and cleanup to notebook 02"
```

---

## Task 10: End-to-end verification in a fresh kernel

**Files:** none (verification only).

- [ ] **Step 1: Execute the notebook top-to-bottom in a clean kernel**

```bash
jupyter nbconvert --to notebook --execute --inplace evals/02_assertions_and_golden_outputs.ipynb
```

Expected: succeeds with no errors. The notebook is rewritten with embedded outputs from the fresh run.

- [ ] **Step 2: Verify expected outputs in the executed notebook**

The output cells should contain, in order:

1. Setup cell:
   ```
   Setup OK
   Score fields: ['key', 'score', 'passed', 'comment']
   ```

2. Grader definitions cell:
   ```
   Graders defined: exact_match, make_contains, make_regex
   ```

3. Exact/contains/regex `### Try it` cell:
   ```
   exact_match on 'positive': passed=True, score=1.0
   exact_match on 'Positive':  passed=False, comment="expected 'positive', got 'Positive'"

   contains on verbose:   passed=True
   contains on negative:  passed=False, comment="'positive' not found in output"

   regex on 'positive':  passed=True
   regex on verbose:     passed=False, comment="pattern '^(positive|negative|neutral)$' not found in output"
   ```

4. Pydantic model + grader definition cell:
   ```
   Classification model and schema_grader defined
   ```

5. Pydantic `### Try it` cell:
   ```
   valid JSON:          passed=True, score=1.0
   wrong label case:    passed=False, comment=<non-empty string mentioning 'label'>
   missing confidence:  passed=False, comment=<non-empty string mentioning 'confidence'>
   malformed JSON:      passed=False, comment=<non-empty string>
   ```

6. Golden file setup cell:
   ```
   Golden file written: /...evals02_..._golden.txt
   Content: 'Summary: The product received overwhelmingly positive reviews. ...'

   make_golden_grader defined
   ```

7. Golden `### Try it` cell:
   ```
   exact match:      passed=True, score=1.0
   mutated output:   passed=False
     comment: output differs from golden (...): first diff at char 139
   truncated output: passed=False
     comment: output differs from golden (...): first diff at char 60
   ```

8. Intentional golden update cell:
   ```
   Golden updated. New content: '...'

   old output vs updated golden: passed=False
   new output vs updated golden: passed=True, score=1.0
   ```

9. Grader composition `### Try it` cell:
   ```
   key                            passed   score    comment
   ------------------------------------------------------------------------
   exact_match                    True     1.0      
   contains_positive              True     1.0      
   has_confidence_field           True     1.0      
   valid_schema                   True     1.0      
   ```

10. Cleanup cell:
    ```
    Removed 1 temp file(s):
      - /...evals02_..._golden.txt
    ```

No cell raises an unhandled exception.

- [ ] **Step 3: Verify no temp files leaked**

```bash
ls /tmp/evals02_* /var/folders/**/evals02_* 2>/dev/null | wc -l
```

Expected: `0`. A zero means the cleanup cell ran and nothing leaked from this execution.

- [ ] **Step 4: Commit the clean run**

```bash
git add evals/02_assertions_and_golden_outputs.ipynb
git commit -m "chore(evals): commit clean fresh-kernel run of notebook 02"
```

---

## Done

After Task 10 passes, notebook 02 is complete. The next plan to write is `2026-05-24-evals-notebook-03-building-an-eval-harness.md`, which formalizes `Example` as a dataclass, builds the `run_eval(agent, dataset, graders) -> results` harness, adds aggregate metrics (pass rate, mean score, per-grader breakdown), and presents results as a readable table — all using the graders defined here, unchanged.
