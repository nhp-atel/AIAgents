# RAG Notebook 01 — `01_why_naive_rag_fails.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first notebook in the RAG-from-first-principles series — establish *why* the naive "stuff the whole corpus into the prompt" approach fails, before introducing any retrieval machinery. The notebook answers a question correctly from a single relevant document, then shows that without knowing which document is relevant you must send *everything* — and that this is wasteful (multiples more tokens, cost, and latency for the same answer), hits a hard context-window ceiling as the corpus grows, and risks "lost in the middle" quality problems. It closes by motivating retrieval (find the relevant chunk) and the embeddings that notebook 02 builds.

**Architecture:** A single self-contained Jupyter notebook plus a small bundled corpus under `rag/data/`. The corpus describes a **fictional** company ("Halcyon Robotics") so the model cannot answer from pretraining — every correct answer must come from the supplied context. The notebook makes real OpenAI chat-completion calls and uses `tiktoken` to count tokens; the always-reproducible teaching spine is the token/cost/waste measurement, not a flaky "the model gets it wrong" claim.

**Tech Stack:** Python 3.11+, Jupyter, `openai` (chat completions), `tiktoken` (token counting), `python-dotenv` (optional). Runs on the user's **`rag_env`** kernel (display name "RAG (.venv)"), which already has these. Requires `OPENAI_API_KEY`.

**Companion spec:** `docs/superpowers/specs/2026-05-25-rag-from-first-principles-design.md` (notebook 01 section and overall arc).

**Folder:** This is the **first** notebook in the series and the `rag/` folder does not yet exist. Task 1 creates `rag/` and `rag/data/`.

---

## File Structure

- **Create:** `rag/` (folder) and `rag/data/` (folder).
- **Create:** `rag/data/company_overview.md`, `rag/data/product_porter_p1.md`, `rag/data/product_porter_p2.md`, `rag/data/product_porter_p3.md`, `rag/data/support.md` — the bundled fictional corpus (markdown). The PDF the spec mentions is added in notebook 03, where document loading is the topic.
- **Create:** `rag/01_why_naive_rag_fails.ipynb` — the entire notebook, self-contained.
- **Modify:** none (the repo `README.md` gets a `rag/` section in a later task of the series, not here).
- **No separate test files** — verification is the notebook running top-to-bottom in a fresh kernel (Task 9).

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown)
2. **"What you'll learn"** (markdown bullets)
3. **Section 1: Setup + guard** (markdown + 2 code cells — imports/optional-dotenv/key-guard; then client, `count_tokens`, pricing constants, and the `ask()` helper)
4. **Section 2: The corpus** (markdown + 1 code cell — load `rag/data/*.md` and print per-doc token counts)
5. **Section 3: Naive RAG that works — when you already know the relevant doc** (markdown + 1 code cell — answer a question from the single relevant document)
6. **Section 4: But you don't know which doc — so you stuff everything** (markdown + 1 code cell — stuff-everything vs only-relevant; same answer, multiples more tokens/cost/latency)
7. **Section 5: The breaking point** (markdown + 1 code cell — token math: context-window ceiling and per-query cost at scale; "lost in the middle" gotcha)
8. **Section 6: What we actually need — retrieval** (markdown only — motivates finding the relevant chunk → embeddings)
9. **"What you just learned"** (markdown bullets)
10. **"What's missing"** (markdown teaser to notebook 02)

---

## Task 1: Create the `rag/` folder, the bundled corpus, and scaffold the empty notebook

**Files:**
- Create: `rag/` and `rag/data/` (folders)
- Create: the five corpus markdown files
- Create: `rag/01_why_naive_rag_fails.ipynb`

- [ ] **Step 1: Create the folders**

```bash
mkdir -p rag/data
ls -la rag
```

Expected: `rag/` exists and contains an empty `data/` subfolder.

- [ ] **Step 2: Create `rag/data/company_overview.md`**

Write this exact content:

```markdown
# Halcyon Robotics — Company Overview

Halcyon Robotics is a warehouse-automation company founded in 2019 and headquartered in Portland, Oregon. The company designs autonomous mobile robots (AMRs) that move inventory inside fulfillment centers.

Halcyon was founded by Dr. Priya Nair and Marcus Webb. As of 2025 the company employs roughly 240 people and operates across North America and Western Europe.

The company's mission is "to make warehouses safer and faster by letting robots handle the heavy, repetitive work." All of Halcyon's robots are sold under the "Porter" product line.
```

- [ ] **Step 3: Create `rag/data/product_porter_p1.md`**

Write this exact content:

```markdown
# Porter P1 — Compact Mover

The Porter P1 is Halcyon's entry-level autonomous mover, introduced in 2021. It is designed for light-duty shelf-to-station transport in smaller facilities.

- Maximum payload: 80 kg
- Battery life: 9 hours per charge
- Top speed: 1.5 meters per second
- Charge time: 45 minutes to 80%
- List price: $18,000

The P1 navigates using lidar and a downward-facing floor-marker camera. It is best suited for facilities under 5,000 square meters.
```

- [ ] **Step 4: Create `rag/data/product_porter_p2.md`**

Write this exact content:

```markdown
# Porter P2 — Standard Hauler

The Porter P2, released in 2023, is Halcyon's most popular robot for mid-sized fulfillment centers.

- Maximum payload: 150 kg
- Battery life: 12 hours per charge
- Top speed: 2.0 meters per second
- Charge time: 60 minutes to 80%
- List price: $27,500

The P2 adds a swappable battery pack, so a depleted unit can return to service in under two minutes with a charged pack.
```

- [ ] **Step 5: Create `rag/data/product_porter_p3.md`**

Write this exact content:

```markdown
# Porter P3 — Heavy Lifter

The Porter P3 is Halcyon's heavy-duty robot, launched in 2024 for high-throughput warehouses.

- Maximum payload: 400 kg
- Battery life: 8 hours per charge
- Top speed: 1.8 meters per second
- Charge time: 90 minutes to 80%
- List price: $46,000

The P3 uses a reinforced chassis and dual lidar units for precise navigation around large loads.
```

- [ ] **Step 6: Create `rag/data/support.md`**

Write this exact content:

```markdown
# Halcyon Robotics — Support & Warranty

All Porter robots ship with a standard two-year warranty covering parts and labor. An extended warranty of up to five years is available for an additional 15% of the list price.

Technical support is available Monday through Friday, 6:00 AM to 8:00 PM Pacific Time, by phone and email. Customers on the Premium support plan receive 24/7 coverage and a four-hour on-site response guarantee.

Spare parts are stocked in regional depots in Portland, Dallas, and Rotterdam. Software updates are delivered over the air at no additional cost for the life of the robot.
```

- [ ] **Step 7: Create the empty notebook bound to the `rag_env` kernel**

Write the file directly:

```json
{
  "cells": [],
  "metadata": {
    "kernelspec": {
      "display_name": "RAG (.venv)",
      "language": "python",
      "name": "rag_env"
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

Save as `rag/01_why_naive_rag_fails.ipynb`.

- [ ] **Step 8: Verify the corpus and notebook**

```bash
ls -la rag/data
python -c "import json; json.load(open('rag/01_why_naive_rag_fails.ipynb'))"
python -c "import tiktoken; enc=tiktoken.encoding_for_model('gpt-4o-mini'); import pathlib; [print(p.name, len(enc.encode(p.read_text()))) for p in sorted(pathlib.Path('rag/data').glob('*.md'))]"
```

Expected: five `.md` files listed; notebook is valid JSON; each doc prints a token count in roughly the 60–130 range (exact numbers vary slightly by tokenizer version).

- [ ] **Step 9: Commit**

```bash
git add rag/data rag/01_why_naive_rag_fails.ipynb
git commit -m "feat(rag): scaffold rag/ series with Halcyon corpus and notebook 01"
```

---

## Task 2: Add intro markdown — title, "Why this notebook exists", and "What you'll learn"

**Files:**
- Modify: `rag/01_why_naive_rag_fails.ipynb` (add cells 1–2)

- [ ] **Step 1: Add the title + motivation markdown cell**

Cell content (markdown):

```markdown
# 01 — Why Naive RAG Fails: The No-Retrieval Baseline

## Why this notebook exists

You have an LLM and a pile of documents the model was never trained on — a company wiki, a product catalog, a stack of PDFs. You want the model to answer questions about them. The simplest possible approach is: **paste all the documents into the prompt and ask.** No vector database, no embeddings, nothing fancy.

This notebook builds exactly that, and shows where it breaks. It works beautifully when you already know which single document holds the answer. The moment you *don't* — which is the real situation — you're forced to send everything, every time. We'll measure what that costs: multiples more tokens, money, and latency for the identical answer, a hard ceiling once the corpus outgrows the context window, and the quality risk of burying the relevant fact among hundreds of irrelevant ones.

The fix is **retrieval**: find the few relevant pieces and send only those. By the end of this notebook you'll feel exactly why that matters — which is the foundation the rest of the series builds on.

Requires an `OPENAI_API_KEY`. Our corpus describes a *fictional* company, so the model can't cheat from memory — every correct answer has to come from the context we provide.
```

- [ ] **Step 2: Add the "What you'll learn" markdown cell**

Cell content (markdown):

```markdown
## What you'll learn

- How "naive RAG" — putting documents directly in the prompt — works, and the one situation where it's perfectly fine.
- Why **not knowing which document is relevant** forces you to send the whole corpus on every query.
- How to **measure the cost of stuffing**: token counts (with `tiktoken`), dollar cost, and latency — and see the same answer cost multiples more.
- The **hard ceiling**: how to compute when a growing corpus simply won't fit in the model's context window, and what stuffing would cost per query at scale.
- The **"lost in the middle"** quality risk of long, mostly-irrelevant contexts.
- Why the answer is **retrieval** — send only the relevant pieces — which motivates embeddings (notebook 02).
```

- [ ] **Step 3: Commit**

```bash
git add rag/01_why_naive_rag_fails.ipynb
git commit -m "feat(rag): add intro and What You'll Learn to notebook 01"
```

---

## Task 3: Section 1 — Setup + guard

**Files:**
- Modify: `rag/01_why_naive_rag_fails.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 1. Setup

This notebook calls the OpenAI chat API and counts tokens with `tiktoken`. We load `OPENAI_API_KEY` from the environment (and from a local `.env` file if `python-dotenv` is installed — real environment variables always win).

The guard cell below stops with a clear message if the key is missing. After it, we set up the OpenAI client, a `count_tokens` helper, illustrative pricing constants, and an `ask(question, context)` helper that answers a question using only the supplied context and reports how many tokens, dollars, and seconds it took.
```

- [ ] **Step 2: Add the imports + key-guard code cell**

```python
import os
import time

# Optional: load a local .env if python-dotenv is installed. Real env vars win.
try:
    from dotenv import load_dotenv
    load_dotenv()
except ImportError:
    pass

# ── Key guard ──────────────────────────────────────────────────────────────
if not os.environ.get("OPENAI_API_KEY"):
    print("=" * 60)
    print("OPENAI_API_KEY is not set.")
    print("=" * 60)
    print()
    print("Every notebook in the RAG series calls the OpenAI API")
    print("(this one for the chat model; later ones for embeddings too).")
    print()
    print("Set it and restart the kernel:")
    print("  export OPENAI_API_KEY=sk-...")
    raise SystemExit("Set OPENAI_API_KEY and restart the kernel to continue.")

print("OPENAI_API_KEY set ✓")
```

- [ ] **Step 3: Add the client + helpers code cell**

```python
import tiktoken
from openai import OpenAI

DEFAULT_MODEL = "gpt-4o-mini"

# gpt-4o-mini pricing (illustrative — verify current rates at openai.com/pricing).
PRICE_INPUT_PER_1M = 0.15    # USD per 1M input (prompt) tokens
PRICE_OUTPUT_PER_1M = 0.60   # USD per 1M output (completion) tokens

openai_client = OpenAI()  # reads OPENAI_API_KEY from the environment

_enc = tiktoken.encoding_for_model(DEFAULT_MODEL)


def count_tokens(text: str) -> int:
    """Count tokens the way the model will, using tiktoken."""
    return len(_enc.encode(text))


def ask(question: str, context: str, model: str = DEFAULT_MODEL) -> dict:
    """Answer `question` using ONLY `context`.

    Returns a dict with the answer plus token/latency/cost telemetry so we can
    measure what each call costs.
    """
    system = (
        "You are a helpful assistant. Answer the question using ONLY the provided "
        "context. If the answer is not in the context, say you don't know. Be concise."
    )
    user = f"Context:\n{context}\n\nQuestion: {question}"
    start = time.time()
    resp = openai_client.chat.completions.create(
        model=model,
        messages=[
            {"role": "system", "content": system},
            {"role": "user", "content": user},
        ],
        temperature=0,
    )
    latency = time.time() - start
    usage = resp.usage
    cost = (
        usage.prompt_tokens * PRICE_INPUT_PER_1M
        + usage.completion_tokens * PRICE_OUTPUT_PER_1M
    ) / 1_000_000
    return {
        "answer": resp.choices[0].message.content.strip(),
        "prompt_tokens": usage.prompt_tokens,
        "completion_tokens": usage.completion_tokens,
        "latency_s": latency,
        "cost_usd": cost,
    }


print("Setup OK")
print(f"Model: {DEFAULT_MODEL}")
print(f"count_tokens('hello world') = {count_tokens('hello world')}")
```

- [ ] **Step 4: Run the three cells and verify**

Expected: the guard prints `OPENAI_API_KEY set ✓`; the helpers cell prints `Setup OK`, the model name, and a small integer (2 or 3) for `count_tokens('hello world')`. No exceptions.

- [ ] **Step 5: Commit**

```bash
git add rag/01_why_naive_rag_fails.ipynb
git commit -m "feat(rag): add setup, key guard, and ask() helper to notebook 01"
```

---

## Task 4: Section 2 — The corpus

**Files:**
- Modify: `rag/01_why_naive_rag_fails.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 2. The Corpus

Our corpus is a handful of short markdown documents about **Halcyon Robotics**, a fictional warehouse-automation company. Using a made-up company is deliberate: the model has never seen it, so it genuinely cannot answer without the documents — which is the whole point of retrieval-augmented generation.

The cell below loads every `.md` file in `rag/data/` and prints its token count. These same documents are reused throughout the series; notebook 03 adds a PDF and teaches loading and chunking properly.
```

- [ ] **Step 2: Add the corpus-loading code cell**

```python
from pathlib import Path

# Resolve the data folder whether the kernel runs from rag/ or from the repo root.
DATA_DIR = Path("data")
if not DATA_DIR.exists():
    DATA_DIR = Path("rag/data")

corpus: dict[str, str] = {}
for path in sorted(DATA_DIR.glob("*.md")):
    corpus[path.name] = path.read_text(encoding="utf-8")

print(f"Loaded {len(corpus)} documents from {DATA_DIR}/:\n")
total = 0
for name, text in corpus.items():
    n = count_tokens(text)
    total += n
    print(f"  {name:<28} {n:>5} tokens")
print(f"  {'-' * 28} {'-' * 11}")
print(f"  {'TOTAL':<28} {total:>5} tokens")
```

- [ ] **Step 3: Run the cell and verify**

Expected: five documents listed (`company_overview.md`, `product_porter_p1.md`, `product_porter_p2.md`, `product_porter_p3.md`, `support.md`), each with a token count roughly 60–130, and a TOTAL in the few-hundred range. Exact counts vary by tokenizer version.

- [ ] **Step 4: Commit**

```bash
git add rag/01_why_naive_rag_fails.ipynb
git commit -m "feat(rag): add corpus loading and token counts to notebook 01"
```

---

## Task 5: Section 3 — Naive RAG that works (when you know the relevant doc)

**Files:**
- Modify: `rag/01_why_naive_rag_fails.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 3. Naive RAG That Works — When You Already Know the Relevant Doc

Here is the simplest "RAG" there is: take the one document that holds the answer, put it in the prompt, and ask. We ask about Halcyon's support hours — a fact that lives only in `support.md` and nowhere in the model's training data.

This works perfectly. The answer is correct, grounded, and cheap, because we sent exactly the right context and nothing else.

> **Gotcha:** This only works because *we* knew `support.md` was the relevant document. We did the retrieval by hand. The entire rest of this series is about doing that step automatically — because in the real world you have thousands of documents and no idea up front which one holds the answer.
```

- [ ] **Step 2: Add the code cell**

```python
# A question answerable only from Halcyon's docs (the model can't know it).
question = "What are Halcyon Robotics' technical support hours?"

# Hand-pick the ONE relevant document.
relevant_doc = corpus["support.md"]
result = ask(question, context=relevant_doc)

print(f"Q: {question}\n")
print(f"A: {result['answer']}\n")
print(f"context tokens : {result['prompt_tokens']}")
print(f"latency        : {result['latency_s']:.2f}s")
print(f"cost           : ${result['cost_usd']:.6f}")
```

- [ ] **Step 3: Run the cell and verify**

Expected: the answer states support is available Monday–Friday, 6:00 AM–8:00 PM Pacific Time (phrasing varies). `context tokens` is small (roughly 120–180). No exception. The exact answer wording will vary between runs — the requirement is that it correctly reflects `support.md`.

- [ ] **Step 4: Commit**

```bash
git add rag/01_why_naive_rag_fails.ipynb
git commit -m "feat(rag): add working naive-RAG single-doc demo to notebook 01"
```

---

## Task 6: Section 4 — You don't know which doc, so you stuff everything

**Files:**
- Modify: `rag/01_why_naive_rag_fails.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 4. But You Don't Know Which Doc — So You Stuff Everything

In reality you can't hand-pick the relevant document, because you don't know which one it is until you've answered the question. The naive workaround is to put the **entire corpus** in the prompt every single time and let the model sort it out.

Below we ask about the Porter P2's battery life two ways: once with the whole corpus stuffed in, and once with only the relevant document. Watch the answer — it's the same — and then watch the token count, cost, and latency.

> **Gotcha:** With only five tiny documents the model usually still gets the right answer when you stuff everything. The problem isn't (yet) correctness — it's **waste**. You're paying to send four irrelevant documents to answer from one. That waste grows linearly with every document you add, forever.
```

- [ ] **Step 2: Add the code cell**

```python
# Stuff the ENTIRE corpus into one context blob.
everything = "\n\n".join(f"=== {name} ===\n{text}" for name, text in corpus.items())

q2 = "What is the battery life of the Porter P2?"

stuffed = ask(q2, context=everything)
only_relevant = ask(q2, context=corpus["product_porter_p2.md"])

print(f"Q: {q2}\n")
print("Stuff-everything approach:")
print(f"  answer        : {stuffed['answer']}")
print(f"  prompt tokens : {stuffed['prompt_tokens']}")
print(f"  cost          : ${stuffed['cost_usd']:.6f}")
print(f"  latency       : {stuffed['latency_s']:.2f}s")
print()
print("Only-the-relevant-doc approach:")
print(f"  answer        : {only_relevant['answer']}")
print(f"  prompt tokens : {only_relevant['prompt_tokens']}")
print(f"  cost          : ${only_relevant['cost_usd']:.6f}")
print(f"  latency       : {only_relevant['latency_s']:.2f}s")
print()
waste = stuffed["prompt_tokens"] / max(only_relevant["prompt_tokens"], 1)
print(f"Same answer (~12 hours). The naive approach sent {waste:.1f}x more tokens to get it.")
```

- [ ] **Step 3: Run the cell and verify**

Expected: both approaches answer that the Porter P2's battery lasts 12 hours (phrasing varies). `stuffed['prompt_tokens']` is clearly larger than `only_relevant['prompt_tokens']`, and the final line reports a multiplier greater than 1 (typically 2–4x for this small corpus). No exception. The structural requirement: the stuffed call uses meaningfully more tokens for the same correct answer.

- [ ] **Step 4: Commit**

```bash
git add rag/01_why_naive_rag_fails.ipynb
git commit -m "feat(rag): add stuff-everything vs relevant-doc waste demo to notebook 01"
```

---

## Task 7: Section 5 — The breaking point

**Files:**
- Modify: `rag/01_why_naive_rag_fails.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 5. The Breaking Point

The waste from Section 4 is annoying at five documents. Now extrapolate. Real knowledge bases have thousands or millions of documents. Two things break:

1. **The context window is finite.** `gpt-4o-mini` accepts about 128,000 tokens. Once your corpus exceeds that, you *cannot* stuff it — not "it's expensive," but "the request is rejected."
2. **Cost scales with everything you send.** You pay for every input token on every query. Sending the whole corpus each time means paying for the whole corpus each time.

The cell below does the arithmetic for our corpus and projects it to a realistic scale.

> **Gotcha — "lost in the middle":** even when a long context *fits*, models reliably attend best to information at the very start and very end of the prompt, and worst to information buried in the middle. So a giant stuffed context doesn't just cost more — it can actively *lower* answer quality by burying the one relevant fact among hundreds of irrelevant ones. More context is not more better.
```

- [ ] **Step 2: Add the code cell**

```python
corpus_tokens = sum(count_tokens(t) for t in corpus.values())
CONTEXT_WINDOW = 128_000  # gpt-4o-mini context window, in tokens

avg_doc = corpus_tokens / len(corpus)
max_docs = int(CONTEXT_WINDOW / avg_doc)

print(f"Current corpus : {len(corpus)} docs, {corpus_tokens} tokens")
print(f"Average doc    : {avg_doc:.0f} tokens")
print()
print(f"At this size, ~{max_docs:,} documents would fill the entire "
      f"{CONTEXT_WINDOW:,}-token window.")
print(f"A 10,000-document knowledge base would need "
      f"{10_000 * avg_doc / CONTEXT_WINDOW:.0f}x the context window — it simply won't fit.")
print()

# Per-query cost of stuffing, projected to scale.
big_corpus_tokens = 10_000 * avg_doc
big_cost = big_corpus_tokens * PRICE_INPUT_PER_1M / 1_000_000
print(f"Even if it fit: 10,000 docs ≈ {big_corpus_tokens:,.0f} input tokens")
print(f"             ≈ ${big_cost:,.2f} PER QUERY, just to send the context.")
print()
print("Stuffing doesn't scale. We need to send only the relevant pieces.")
```

- [ ] **Step 3: Run the cell and verify**

Expected: prints the corpus token total and average, a `max_docs` figure (hundreds to low thousands), a multiple-of-context-window figure for 10k docs, and a per-query dollar projection. All values are pure arithmetic, so they are deterministic given the corpus. No exception.

- [ ] **Step 4: Commit**

```bash
git add rag/01_why_naive_rag_fails.ipynb
git commit -m "feat(rag): add context-ceiling and cost-at-scale math to notebook 01"
```

---

## Task 8: Section 6 + closing cells

**Files:**
- Modify: `rag/01_why_naive_rag_fails.ipynb` (add 3 markdown cells)

- [ ] **Step 1: Add the "What we actually need" markdown cell (Section 6)**

```markdown
## 6. What We Actually Need — Retrieval

Every problem in this notebook has the same root cause: **we sent the model everything because we couldn't tell what was relevant.** Section 3 proved that if you send just the right document, the answer is correct, cheap, and fast. The entire challenge of RAG is automating that one step — *finding the relevant pieces* — so you get Section 3's quality at any corpus size.

To do that we need a way to measure **relevance**: given a question, which documents (or chunks of documents) are most likely to contain the answer? Keyword matching is a weak proxy — a question about "how long the robot runs" should match a document that says "battery life," even though they share no words.

What we need is a notion of *semantic* similarity: text that *means* the same thing should score as similar even when the words differ. That is exactly what **embeddings** provide — and they're where notebook 02 begins.
```

- [ ] **Step 2: Add the "What you just learned" markdown cell**

```markdown
## What you just learned

- **Naive RAG** — putting documents straight into the prompt — works perfectly *when you already know which document is relevant* (Section 3).
- In reality you don't, so you're forced to **stuff the whole corpus** on every query (Section 4) — paying multiples more tokens, cost, and latency for the identical answer.
- Stuffing has a **hard ceiling**: once the corpus exceeds the model's context window it cannot be sent at all, and the per-query cost scales with the entire corpus (Section 5).
- Long, mostly-irrelevant contexts also risk **"lost in the middle"** quality degradation.
- The fix is **retrieval**: send only the relevant pieces. Doing that automatically requires measuring **semantic relevance** — the job of embeddings.
```

- [ ] **Step 3: Add the "What's missing" markdown cell**

```markdown
## What's missing

We've shown *why* we need retrieval, but we have no way to actually find relevant text yet. We hand-picked `support.md` and `product_porter_p2.md` ourselves — the very step that needs automating.

**Notebook 02 — `02_embeddings_and_semantic_search.ipynb`** introduces **embeddings**: vector representations of text where semantic similarity becomes geometric closeness. We'll compute cosine similarity by hand with NumPy, watch "battery life" match "how long it runs" even with no shared words, and build a tiny brute-force semantic search that ranks our Halcyon documents by relevance to a question — the first real retrieval step.
```

- [ ] **Step 4: Commit**

```bash
git add rag/01_why_naive_rag_fails.ipynb
git commit -m "feat(rag): add retrieval motivation and closing cells to notebook 01"
```

---

## Task 9: End-to-end verification in a fresh kernel

**Files:** none (verification only).

- [ ] **Step 1: Confirm prerequisites**

```bash
python -c "import openai, tiktoken; print('deps OK')"
echo "OPENAI_API_KEY set: $(test -n \"$OPENAI_API_KEY\" && echo YES || echo NO)"
```

`OPENAI_API_KEY` must be available to the kernel. If it is only in the repo's `.env` and the `rag_env` kernel lacks `python-dotenv`, export it before executing (the next step shows how). The notebook tolerates a missing `python-dotenv` (the import is optional).

- [ ] **Step 2: Execute the notebook top-to-bottom in a clean kernel**

```bash
export OPENAI_API_KEY=$(python -c "from dotenv import dotenv_values; print(dotenv_values('.env').get('OPENAI_API_KEY',''))")
python -m jupyter nbconvert --to notebook --execute --inplace \
  --ExecutePreprocessor.timeout=300 \
  --ExecutePreprocessor.kernel_name=rag_env \
  "rag/01_why_naive_rag_fails.ipynb"
```

Expected: succeeds with no errors. The notebook is rewritten with embedded outputs. (Command substitution keeps the key out of the command text and output. If `rag_env` cannot be launched headlessly, substitute a kernel that has `openai` + `tiktoken`.)

- [ ] **Step 3: Verify structural outcomes in the executed notebook**

Confirm (model wording varies — check structure, not exact strings):

1. **Section 1** — guard prints `OPENAI_API_KEY set ✓`; helpers cell prints `Setup OK`.
2. **Section 2** — five documents loaded with per-doc token counts and a TOTAL.
3. **Section 3** — the answer correctly reflects `support.md` (Mon–Fri, 6 AM–8 PM Pacific); `context tokens` is small.
4. **Section 4** — both approaches answer ~12 hours; `stuffed['prompt_tokens']` > `only_relevant['prompt_tokens']`; the final line reports a multiplier > 1.
5. **Section 5** — prints the corpus token total, a `max_docs` figure, the 10k-doc context-window multiple, and a per-query cost projection.
6. No cell raises an unhandled exception.

- [ ] **Step 4: Commit the clean run**

```bash
git add rag/01_why_naive_rag_fails.ipynb
git commit -m "chore(rag): commit clean fresh-kernel run of notebook 01"
```

---

## Done

After Task 9 passes, notebook 01 is complete: it establishes why stuffing the whole corpus into the prompt fails (waste, the context ceiling, lost-in-the-middle) and motivates retrieval. The bundled `rag/data/` Halcyon corpus created here is reused by every later notebook.

The next plan to write is `2026-05-25-rag-notebook-02-embeddings-and-semantic-search.md`, which introduces embeddings (`text-embedding-3-small`), cosine similarity computed by hand in NumPy, and a brute-force top-k semantic search over the Halcyon corpus — the first automated retrieval step, replacing the hand-picking we did in this notebook.
```