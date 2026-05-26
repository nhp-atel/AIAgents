# RAG Notebook 02 — `02_embeddings_and_semantic_search.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the second notebook in the RAG-from-first-principles series — introduce **embeddings** as the tool that measures relevance, the gap notebook 01 left open. The notebook shows what an embedding vector is, computes **cosine similarity by hand in NumPy**, demonstrates that semantic similarity beats keyword overlap (a query and the document that answers it can share almost no words), and builds a **brute-force top-k semantic search** over the bundled Halcyon corpus — automating the document-picking that notebook 01 did by hand. It closes by motivating chunking: one vector per whole document is too coarse for real corpora.

**Architecture:** A single self-contained Jupyter notebook that reuses the `rag/data/` Halcyon corpus created in notebook 01. It calls the OpenAI embeddings API (`text-embedding-3-small`) and does all similarity math in NumPy — no vector database yet, so the mechanic stays visible. Embeddings are effectively deterministic (no temperature), so outputs are stable across runs; the teaching point is the *relative* ordering of similarity scores.

**Tech Stack:** Python 3.11+ syntax-compatible down to **3.9** (the `rag_env` kernel is Python 3.9.2 — no `X | None` unions or `match`; PEP 585 `list[str]`/`dict[...]` are fine), Jupyter, `openai` (embeddings), `numpy`, `python-dotenv` (optional). Runs on the **`rag_env`** kernel. Requires `OPENAI_API_KEY`.

**Companion spec:** `docs/superpowers/specs/2026-05-25-rag-from-first-principles-design.md` (notebook 02 section).

**Prior notebook:** Notebook 01 created `rag/` and the five-document `rag/data/` Halcyon corpus. This notebook reuses that corpus and does not create new data files.

---

## File Structure

- **Create:** `rag/02_embeddings_and_semantic_search.ipynb` — the entire notebook, self-contained.
- **Reuses (does not modify):** `rag/data/*.md` (the Halcyon corpus from notebook 01).
- **Modify:** none.
- **No separate test files** — verification is the notebook running top-to-bottom in a fresh kernel (Task 9).

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown — recap nb01: we proved we need retrieval; now build the tool that measures relevance)
2. **"What you'll learn"** (markdown bullets)
3. **Section 1: Setup + guard** (markdown + 2 code cells — imports/optional-dotenv/key-guard; then client, `embed`, `embed_many`, `cosine` helpers)
4. **Section 2: What is an embedding?** (markdown + 1 code cell — embed a phrase, inspect the vector: dimension, dtype, norm, first values)
5. **Section 3: Cosine similarity by hand** (markdown + 1 code cell — semantic pairs score higher than lexical-overlap-but-different pairs)
6. **Section 4: Semantic search beats keyword search** (markdown + 1 code cell — a query and its answer doc share ~no words; keyword overlap ≈ 0 while cosine is high)
7. **Section 5: Brute-force semantic search over the corpus** (markdown + 2 code cells — embed all docs once, then rank docs by cosine for several queries; the right doc ranks #1)
8. **Section 6: What we built and where it breaks** (markdown only — one vector per whole doc is too coarse → motivate chunking)
9. **"What you just learned"** (markdown bullets)
10. **"What's missing"** (markdown teaser to notebook 03)

---

## Task 1: Scaffold the empty notebook

**Files:**
- Create: `rag/02_embeddings_and_semantic_search.ipynb`

- [ ] **Step 1: Confirm the corpus from notebook 01 exists**

```bash
ls -la rag/data
```

Expected: the five Halcyon `.md` files are present (`company_overview.md`, `product_porter_p1.md`, `product_porter_p2.md`, `product_porter_p3.md`, `support.md`). If they are missing, notebook 01 has not been run — stop and report, do not invent a corpus.

- [ ] **Step 2: Create the empty notebook bound to the `rag_env` kernel**

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
      "version": "3.9"
    }
  },
  "nbformat": 4,
  "nbformat_minor": 5
}
```

Save as `rag/02_embeddings_and_semantic_search.ipynb`.

- [ ] **Step 3: Verify the file is valid JSON**

```bash
python -c "import json; json.load(open('rag/02_embeddings_and_semantic_search.ipynb'))"
ls -la rag/02_embeddings_and_semantic_search.ipynb
```

Expected: no exception, file non-zero size.

- [ ] **Step 4: Commit**

```bash
git add rag/02_embeddings_and_semantic_search.ipynb
git commit -m "feat(rag): scaffold 02_embeddings_and_semantic_search.ipynb"
```

---

## Task 2: Add intro markdown — title, "Why this notebook exists", and "What you'll learn"

**Files:**
- Modify: `rag/02_embeddings_and_semantic_search.ipynb` (add cells 1–2)

- [ ] **Step 1: Add the title + motivation markdown cell**

Cell content (markdown):

```markdown
# 02 — Embeddings and Semantic Search

## Why this notebook exists

Notebook 01 left us with a precise problem: we need to send the model only the *relevant* documents, but to do that we have to *measure* relevance — and we hand-picked the right document ourselves, which is exactly the step that needs automating. Keyword matching is a weak proxy: a question about "how long the robot runs" should match a document that says "battery life," even though they share no words.

This notebook introduces the tool that solves it: **embeddings**. An embedding turns a piece of text into a vector of numbers, positioned so that text with similar *meaning* lands close together — even when the words differ. We'll look at what an embedding actually is, measure closeness with **cosine similarity** (computed by hand in NumPy, so nothing is hidden), watch semantic similarity beat keyword overlap, and then build a tiny search that ranks our Halcyon documents by relevance to a question. That ranking is the first real retrieval step — the automated version of notebook 01's hand-picking.

Requires an `OPENAI_API_KEY` (for the embeddings API). No vector database yet — just NumPy, so you can see exactly how it works.
```

- [ ] **Step 2: Add the "What you'll learn" markdown cell**

Cell content (markdown):

```markdown
## What you'll learn

- What an **embedding** is: a fixed-length vector of floats representing a piece of text, and how to get one from `text-embedding-3-small`.
- How to measure similarity between two embeddings with **cosine similarity**, written out in NumPy.
- Why **semantic** similarity beats **keyword** matching — a question and its answer can share almost no words yet still be close in embedding space.
- How to build a **brute-force top-k semantic search**: embed a corpus, embed a query, rank by cosine, return the best matches.
- Why embedding *whole documents* is only a starting point — and why notebook 03 needs to chunk them.
```

- [ ] **Step 3: Commit**

```bash
git add rag/02_embeddings_and_semantic_search.ipynb
git commit -m "feat(rag): add intro and What You'll Learn to notebook 02"
```

---

## Task 3: Section 1 — Setup + guard + helpers

**Files:**
- Modify: `rag/02_embeddings_and_semantic_search.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 1. Setup

This notebook calls the OpenAI **embeddings** API and does all the math in NumPy. We load `OPENAI_API_KEY` from the environment (and from a local `.env` file if `python-dotenv` is installed — real environment variables always win), then define three small helpers:

- `embed(text)` — embed a single string into a NumPy vector.
- `embed_many(texts)` — embed a list of strings in one API call (cheaper and faster than looping).
- `cosine(a, b)` — cosine similarity between two vectors, a number in `[-1, 1]` where higher means more similar.
```

- [ ] **Step 2: Add the imports + key-guard code cell**

```python
import os

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
    print("This notebook calls the OpenAI embeddings API.")
    print()
    print("Set it and restart the kernel:")
    print("  export OPENAI_API_KEY=sk-...")
    raise SystemExit("Set OPENAI_API_KEY and restart the kernel to continue.")

print("OPENAI_API_KEY set ✓")
```

- [ ] **Step 3: Add the client + helpers code cell**

```python
import numpy as np
from openai import OpenAI

EMBED_MODEL = "text-embedding-3-small"

openai_client = OpenAI()  # reads OPENAI_API_KEY from the environment


def embed(text: str) -> np.ndarray:
    """Embed a single string into a NumPy vector."""
    resp = openai_client.embeddings.create(model=EMBED_MODEL, input=text)
    return np.array(resp.data[0].embedding, dtype=np.float32)


def embed_many(texts: list[str]) -> np.ndarray:
    """Embed a list of strings in ONE API call.

    Returns a matrix of shape (len(texts), embedding_dim); row i is the
    embedding of texts[i]. The API preserves input order.
    """
    resp = openai_client.embeddings.create(model=EMBED_MODEL, input=texts)
    return np.array([d.embedding for d in resp.data], dtype=np.float32)


def cosine(a: np.ndarray, b: np.ndarray) -> float:
    """Cosine similarity between two vectors: the cosine of the angle between
    them. 1.0 = identical direction, 0.0 = unrelated (orthogonal), -1.0 = opposite.
    """
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))


print("Setup OK")
print(f"Embedding model: {EMBED_MODEL}")
```

- [ ] **Step 4: Run the three cells and verify**

Expected: the guard prints `OPENAI_API_KEY set ✓`; the helpers cell prints `Setup OK` and the embedding model name. No exceptions.

- [ ] **Step 5: Commit**

```bash
git add rag/02_embeddings_and_semantic_search.ipynb
git commit -m "feat(rag): add setup, key guard, and embed/cosine helpers to notebook 02"
```

---

## Task 4: Section 2 — What is an embedding?

**Files:**
- Modify: `rag/02_embeddings_and_semantic_search.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 2. What Is an Embedding?

An embedding is a list of numbers — a vector — that represents a piece of text. The embedding model is trained so that texts with similar meaning produce vectors that point in similar directions. `text-embedding-3-small` returns a **1536-dimensional** vector for any input, from a single word to a full paragraph.

The cell below embeds a short phrase and inspects the result: how many dimensions it has, its data type, its length (norm), and its first few values. The exact numbers aren't meaningful on their own — what matters is that *every* piece of text becomes a vector of the same shape, so we can compare any two of them.
```

- [ ] **Step 2: Add the code cell**

```python
vec = embed("warehouse robot")

print("Embedding of 'warehouse robot':")
print(f"  dimensions : {vec.shape[0]}")
print(f"  dtype      : {vec.dtype}")
print(f"  vector norm: {np.linalg.norm(vec):.4f}")
print(f"  first 8    : {np.round(vec[:8], 4)}")
```

- [ ] **Step 3: Run the cell and verify**

Expected: `dimensions : 1536`, `dtype : float32`, a vector norm very close to `1.0000` (these embeddings are normalized to unit length), and eight small floats. Exact float values are not asserted.

- [ ] **Step 4: Commit**

```bash
git add rag/02_embeddings_and_semantic_search.ipynb
git commit -m "feat(rag): add 'what is an embedding' inspection cell to notebook 02"
```

---

## Task 5: Section 3 — Cosine similarity by hand

**Files:**
- Modify: `rag/02_embeddings_and_semantic_search.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 3. Cosine Similarity by Hand

To compare two embeddings we measure the **angle** between their vectors with cosine similarity:

```
cosine(a, b) = (a · b) / (‖a‖ · ‖b‖)
```

It ranges from -1 to 1; for these embeddings, more-similar text gives a higher value. We defined `cosine` in NumPy in Section 1 — nothing is hidden.

The cell below scores four sentence pairs. Two pairs *mean* the same thing but use different words; two pairs *share* words but mean different things. Watch the meaning-based pairs score higher — that's the whole reason embeddings beat keyword matching.

> **Gotcha:** Cosine values from a real embedding model are not calibrated probabilities — an unrelated pair often scores 0.1–0.4, not 0.0. What matters is the *ranking*: the semantically-matched pair scores clearly higher than the lexically-overlapping-but-unrelated one. Compare scores to each other, not to an absolute threshold.
```

- [ ] **Step 2: Add the code cell**

```python
pairs = [
    # Same meaning, few shared words → should score HIGH.
    ("How long does the robot run on one charge?", "battery life per charge"),
    # Shared words ("run", "long") but different meaning → should score LOWER.
    ("How long does the robot run on one charge?", "The run was long and the day tiring."),
    # Same meaning, different words → should score HIGH.
    ("I need to buy a new car.", "looking to purchase an automobile"),
    # No relationship → should score LOW.
    ("I need to buy a new car.", "The weather is sunny today."),
]

for a, b in pairs:
    sim = cosine(embed(a), embed(b))
    print(f"  cosine = {sim:.3f}")
    print(f"     A: {a}")
    print(f"     B: {b}")
    print()
```

- [ ] **Step 3: Run the cell and verify**

Expected: four cosine scores. The two "same meaning" pairs (rows 1 and 3) score clearly higher than their neighboring "shared words but different meaning" / "unrelated" pairs (rows 2 and 4). Typical values: semantic matches ≈ 0.4–0.7, the others noticeably lower. Exact numbers vary slightly, but the structural requirement is **row 1 > row 2** and **row 3 > row 4**.

- [ ] **Step 4: Commit**

```bash
git add rag/02_embeddings_and_semantic_search.ipynb
git commit -m "feat(rag): add cosine-similarity semantic-vs-lexical demo to notebook 02"
```

---

## Task 6: Section 4 — Semantic search beats keyword search

**Files:**
- Modify: `rag/02_embeddings_and_semantic_search.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 4. Semantic Search Beats Keyword Search

Here is the concrete payoff. Take a real user question about the Porter P2 and the line in our corpus that answers it. They share almost no words — the question says "operate before recharging," the document says "battery life per charge." A keyword search would miss the connection entirely. Embeddings catch it.

The cell below computes a naive keyword-overlap score (what fraction of the question's words appear in the document) and the cosine similarity, side by side.
```

- [ ] **Step 2: Add the code cell**

```python
query = "How long can the Porter P2 operate before it needs recharging?"
doc = "Battery life: 12 hours per charge"

# Naive keyword overlap: fraction of query words that appear in the document.
q_words = set(query.lower().replace("?", "").split())
d_words = set(doc.lower().split())
shared = q_words & d_words
overlap = len(shared) / len(q_words)

sim = cosine(embed(query), embed(doc))

print(f"Query : {query}")
print(f"Doc   : {doc}")
print()
print(f"Keyword overlap : {overlap:.2f}   shared words: {sorted(shared)}")
print(f"Cosine (meaning): {sim:.3f}")
print()
print("Keyword search barely connects them — they share essentially no words.")
print("Embeddings see that 'operate before recharging' means 'battery life per charge'.")
```

- [ ] **Step 3: Run the cell and verify**

Expected: `Keyword overlap : 0.00` (or very close — the question and the spec line share essentially no words) with an empty or near-empty shared-words list, and a clearly positive `Cosine (meaning)` score (typically ≈ 0.3–0.6). The structural requirement: cosine is substantially higher than the keyword overlap.

- [ ] **Step 4: Commit**

```bash
git add rag/02_embeddings_and_semantic_search.ipynb
git commit -m "feat(rag): add semantic-vs-keyword search contrast to notebook 02"
```

---

## Task 7: Section 5 — Brute-force semantic search over the corpus

**Files:**
- Modify: `rag/02_embeddings_and_semantic_search.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 5. Brute-Force Semantic Search Over the Corpus

Now we automate notebook 01's hand-picking. We load the five Halcyon documents, embed each one (a single batched API call), and store the vectors in a matrix. To answer a question we embed the question, compute its cosine similarity against every document vector, and rank them. The top result is the document most likely to hold the answer — chosen automatically, no human in the loop.

This is "brute force" because we compare the query against *every* document. That's fine for five documents; notebook 04 introduces a real vector index for when there are thousands.
```

- [ ] **Step 2: Add the corpus-embedding code cell**

```python
from pathlib import Path

# Resolve the data folder whether the kernel runs from rag/ or from the repo root.
DATA_DIR = Path("data")
if not DATA_DIR.exists():
    DATA_DIR = Path("rag/data")

doc_names: list[str] = []
doc_texts: list[str] = []
for path in sorted(DATA_DIR.glob("*.md")):
    doc_names.append(path.name)
    doc_texts.append(path.read_text(encoding="utf-8"))

# Fail clearly if the corpus is missing, instead of a confusing error later.
if not doc_texts:
    raise FileNotFoundError(
        f"No .md documents found in {DATA_DIR}/. "
        "Run notebook 01 first to create the Halcyon corpus."
    )

# Embed every document once, in a single batched call.
doc_matrix = embed_many(doc_texts)
print(f"Embedded {len(doc_texts)} documents into a {doc_matrix.shape} matrix:")
for name in doc_names:
    print(f"  - {name}")
```

- [ ] **Step 3: Add the search + demo code cell**

```python
def semantic_search(query: str, top_k: int = 3) -> list:
    """Return the top_k (doc_name, score) pairs most similar to the query,
    ranked by cosine similarity against the pre-embedded document matrix.
    """
    q = embed(query)
    scored = [(doc_names[i], cosine(q, doc_matrix[i])) for i in range(len(doc_names))]
    scored.sort(key=lambda pair: pair[1], reverse=True)
    return scored[:top_k]


queries = [
    "How long can the Porter P2 run before recharging?",
    "Who founded Halcyon and where is it based?",
    "What does the warranty cover?",
]

for query in queries:
    print(f"Q: {query}")
    for name, score in semantic_search(query, top_k=3):
        print(f"   {score:.3f}  {name}")
    print()
```

- [ ] **Step 4: Run the cells and verify**

Expected: the first cell reports `Embedded 5 documents into a (5, 1536) matrix` and lists the five files. The search cell prints three queries, each with a ranked top-3. The **#1 result** should be:
- "Porter P2 ... recharging" → `product_porter_p2.md`
- "Who founded Halcyon ..." → `company_overview.md`
- "What does the warranty cover?" → `support.md`

Exact cosine scores vary slightly, but the correct document must rank first for each query. (If a #1 is wrong, that is a real signal worth reporting — but with this corpus and these queries the expected matches are robust.)

- [ ] **Step 5: Commit**

```bash
git add rag/02_embeddings_and_semantic_search.ipynb
git commit -m "feat(rag): add brute-force semantic search over the Halcyon corpus to notebook 02"
```

---

## Task 8: Section 6 + closing cells

**Files:**
- Modify: `rag/02_embeddings_and_semantic_search.ipynb` (add 3 markdown cells)

- [ ] **Step 1: Add the "What we built and where it breaks" markdown cell (Section 6)**

```markdown
## 6. What We Built — and Where It Breaks

We just built real retrieval: given a question, rank the documents by semantic relevance and return the best ones. Feed the top result into the prompt instead of the whole corpus, and you get notebook 01's "it works" quality without notebook 01's waste. That is the core of RAG.

But there's a crack already. We embedded each document as a *single* vector. A real document isn't about one thing — `support.md` covers warranty, support hours, *and* spare-part depots. Squashing all of that into one vector blurs it: a question about spare parts and a question about warranty both match the same averaged vector, and a long document's specific details get washed out. Worse, real documents are far longer than ours and won't embed well as one piece at all.

The fix is to break documents into smaller, focused **chunks** and embed those — so retrieval can return the exact passage that answers a question, not a whole averaged document.
```

- [ ] **Step 2: Add the "What you just learned" markdown cell**

```markdown
## What you just learned

- An **embedding** is a fixed-length vector (1536 dims for `text-embedding-3-small`) representing text by meaning; similar meanings point in similar directions.
- **Cosine similarity** measures the angle between two embeddings; compare scores to *each other*, not to an absolute threshold.
- **Semantic similarity beats keyword matching**: a question and its answer can share almost no words yet still be close in embedding space.
- A **brute-force semantic search** — embed the corpus, embed the query, rank by cosine — automatically picks the relevant document, replacing notebook 01's hand-picking.
- Embedding *whole documents* is too coarse: one vector blurs a multi-topic document, which is why we need chunking.
```

- [ ] **Step 3: Add the "What's missing" markdown cell**

```markdown
## What's missing

We embedded whole documents, and our documents are tiny. Real corpora are PDFs and long pages that cover many topics each — too big and too mixed to represent as a single vector. We also haven't dealt with *loading* anything beyond plain markdown.

**Notebook 03 — `03_chunking_and_document_loading.ipynb`** tackles both: loading real documents (including a PDF with `pypdf`), then splitting them into well-sized, overlapping **chunks** with metadata (source, page). Chunking is what lets retrieval return the exact passage that answers a question instead of a whole averaged document — and it's the input the vector index in notebook 04 will store.
```

- [ ] **Step 4: Commit**

```bash
git add rag/02_embeddings_and_semantic_search.ipynb
git commit -m "feat(rag): add chunking motivation and closing cells to notebook 02"
```

---

## Task 9: End-to-end verification in a fresh kernel

**Files:** none (verification only).

- [ ] **Step 1: Confirm prerequisites**

```bash
python -c "import openai, numpy; print('deps OK')"
ls rag/data/*.md
echo "OPENAI_API_KEY set: $(test -n \"$OPENAI_API_KEY\" && echo YES || echo NO)"
```

`OPENAI_API_KEY` must be available to the kernel, and the five corpus files must exist. The notebook tolerates a missing `python-dotenv` (the import is optional).

- [ ] **Step 2: Execute the notebook top-to-bottom in a clean kernel**

```bash
export OPENAI_API_KEY=$(python -c "from dotenv import dotenv_values; print(dotenv_values('.env').get('OPENAI_API_KEY',''))")
python -m jupyter nbconvert --to notebook --execute --inplace \
  --ExecutePreprocessor.timeout=300 \
  --ExecutePreprocessor.kernel_name=rag_env \
  "rag/02_embeddings_and_semantic_search.ipynb"
```

Expected: succeeds with no errors. The notebook is rewritten with embedded outputs. (Command substitution keeps the key out of the command text and output. If `rag_env` cannot be launched headlessly, substitute a kernel that has `openai` + `numpy`.)

- [ ] **Step 3: Verify structural outcomes in the executed notebook**

Confirm (exact float values vary — check structure and ranking, not exact strings):

1. **Section 1** — guard prints `OPENAI_API_KEY set ✓`; helpers cell prints `Setup OK`.
2. **Section 2** — embedding reports `dimensions : 1536`, `dtype : float32`, vector norm ≈ 1.0.
3. **Section 3** — four cosine scores; row 1 > row 2 and row 3 > row 4 (semantic match beats lexical-overlap/unrelated).
4. **Section 4** — keyword overlap ≈ 0.00 with an empty/near-empty shared-words list; cosine clearly positive and much higher than the overlap.
5. **Section 5** — `Embedded 5 documents into a (5, 1536) matrix`; for the three queries, the #1 ranked document is `product_porter_p2.md`, `company_overview.md`, and `support.md` respectively.
6. No cell raises an unhandled exception.

- [ ] **Step 4: Commit the clean run**

```bash
git add rag/02_embeddings_and_semantic_search.ipynb
git commit -m "chore(rag): commit clean fresh-kernel run of notebook 02"
```

---

## Done

After Task 9 passes, notebook 02 is complete: it introduces embeddings and cosine similarity, shows semantic search beating keyword matching, and builds a brute-force top-k search that automatically ranks the Halcyon documents by relevance — the first automated retrieval step.

The next plan to write is `2026-05-25-rag-notebook-03-chunking-and-document-loading.md`, which loads real documents (adding a PDF to `rag/data/` and reading it with `pypdf`) and splits them into well-sized, overlapping chunks with metadata — the units that the vector index in notebook 04 will store and that retrieval will return instead of whole documents.
```