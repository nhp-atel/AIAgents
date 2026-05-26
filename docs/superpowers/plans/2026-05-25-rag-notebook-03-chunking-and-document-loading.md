# RAG Notebook 03 — `03_chunking_and_document_loading.ipynb` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the third notebook in the RAG-from-first-principles series — turn real documents into retrievable units. The notebook loads the markdown corpus **plus a new PDF** (read with `pypdf`), normalizes everything to text with `source`/`page` metadata, motivates why whole-document vectors are too coarse, demonstrates three chunking strategies (naive fixed-size, token-based, and recursive/structure-aware via `langchain-text-splitters`) with overlap, builds a canonical list of chunk records carrying metadata, and shows that **chunk-level** retrieval returns the exact passage (with its source and page) instead of a whole averaged document. It closes by motivating a real vector index (notebook 04).

**Architecture:** A single self-contained Jupyter notebook that reuses the `rag/data/` Halcyon corpus and adds one bundled PDF. The PDF is authored once at build time with `matplotlib` (available on the build machine) and committed; the notebook only ever *reads* it with `pypdf` (in `rag_env`). Chunking uses `langchain-text-splitters`; embeddings use `text-embedding-3-small`; similarity math stays in NumPy — still no vector database.

**Tech Stack:** Python **3.9**-compatible (the `rag_env` kernel is Python 3.9.2 — no `X | None` unions or `match`; PEP 585 `list[...]`/`dict[...]` are fine but the existing notebooks use bare `list`/`dict`, follow that), Jupyter, `openai` (embeddings), `pypdf` (read PDF), `langchain-text-splitters` (chunking), `tiktoken`, `numpy`, `python-dotenv` (optional). PDF authored once with `matplotlib`. Runs on the **`rag_env`** kernel. Requires `OPENAI_API_KEY`.

**Companion spec:** `docs/superpowers/specs/2026-05-25-rag-from-first-principles-design.md` (notebook 03 section).

**Prior notebooks:** Notebook 01 created `rag/data/*.md`. Notebook 02 introduced `embed`, `embed_many`, `cosine` (re-declared here). This notebook adds `rag/data/porter_p2_field_guide.pdf`.

---

## File Structure

- **Create:** `rag/data/porter_p2_field_guide.pdf` — a 2-page fictional PDF (authored once with matplotlib, committed as sample data).
- **Create:** `rag/03_chunking_and_document_loading.ipynb` — the entire notebook, self-contained.
- **Reuses (does not modify):** `rag/data/*.md` (the five markdown docs from notebook 01).
- **Modify:** none.
- **No separate test files** — verification is the notebook running top-to-bottom in a fresh kernel (Task 10).

## Notebook Section Map

1. **Title + "Why this notebook exists"** (markdown — recap nb02: one vector per doc is too coarse; now load + chunk)
2. **"What you'll learn"** (markdown bullets)
3. **Section 1: Setup + guard** (markdown + 2 code cells — imports/optional-dotenv/key-guard; then client, `embed`, `embed_many`, `cosine`)
4. **Section 2: Loading documents** (markdown + 2 code cells — `load_documents` reads `.md` + the PDF via `pypdf` into page-level units with metadata; then inspect the PDF extract)
5. **Section 3: Why chunk?** (markdown only — granularity and the multi-topic-blur problem)
6. **Section 4: Chunking strategies** (markdown + 2 code cells — naive fixed-size vs recursive; then token-based + overlap)
7. **Section 5: Chunks with metadata** (markdown + 1 code cell — `chunk_documents` builds the canonical chunk records)
8. **Section 6: Chunk-level retrieval beats document-level** (markdown + 1 code cell — embed chunks, query, return the exact passage with source/page)
9. **"What you just learned"** (markdown bullets)
10. **"What's missing"** (markdown teaser to notebook 04)

---

## Task 1: Author the bundled PDF and scaffold the empty notebook

**Files:**
- Create: `rag/data/porter_p2_field_guide.pdf`
- Create: `rag/03_chunking_and_document_loading.ipynb`

- [ ] **Step 1: Confirm the markdown corpus from notebook 01 exists**

```bash
ls -la rag/data
```

Expected: the five Halcyon `.md` files are present. If missing, notebook 01 has not been run — stop and report.

- [ ] **Step 2: Author the sample PDF with matplotlib (one-time build step)**

The notebook *reads* PDFs with `pypdf` (in `rag_env`), but we need a sample PDF to read. Author it once with `matplotlib` (available on the build machine's default `python`). Write this to `/tmp/_make_sample_pdf.py`:

```python
import matplotlib
matplotlib.use("pdf")
matplotlib.rcParams["pdf.fonttype"] = 42  # embed TrueType so pypdf can extract the text
import matplotlib.pyplot as plt
from matplotlib.backends.backend_pdf import PdfPages

PAGE1 = """Porter P2 - Field Service Guide

Preventive Maintenance Schedule

Every 250 operating hours: inspect the drive wheels for wear and
clean the lidar lenses with a microfiber cloth.

Every 500 operating hours: replace the air filter and check the
battery contactor torque (specification: 8 Nm).

Every 1,000 operating hours: perform a full calibration of the
floor-marker camera and update the navigation firmware."""

PAGE2 = """Porter P2 - Field Service Guide

Error Codes

E101: drive motor over-temperature. Allow the unit to cool for
20 minutes before restarting.

E204: lidar obstruction detected. Clean the sensor and clear the
travel path.

E330: battery pack communication fault. Re-seat the swappable
battery; if the fault persists, replace the pack.

For all unresolved faults, contact Halcyon technical support."""

with PdfPages("rag/data/porter_p2_field_guide.pdf") as pdf:
    for body in (PAGE1, PAGE2):
        fig = plt.figure(figsize=(8.5, 11))
        fig.text(0.1, 0.92, body, va="top", ha="left", fontsize=12, family="sans-serif")
        pdf.savefig(fig)
        plt.close(fig)

print("Wrote rag/data/porter_p2_field_guide.pdf")
```

Run it from the repo root with the default python (which has matplotlib):

```bash
python /tmp/_make_sample_pdf.py
```

Expected: prints `Wrote rag/data/porter_p2_field_guide.pdf`. (If the default `python` lacks matplotlib, report it — do not invent the PDF another way without checking.)

- [ ] **Step 3: Verify the PDF has two pages and extractable text**

```bash
python -c "from pypdf import PdfReader; r=PdfReader('rag/data/porter_p2_field_guide.pdf'); print('pages:', len(r.pages)); t=r.pages[1].extract_text(); print('E204 on p2:', 'E204' in t)"
```

Expected: `pages: 2` and `E204 on p2: True`. If text isn't extractable, stop and report (the `pdf.fonttype = 42` setting is what makes it extractable — do not proceed with a non-extractable PDF).

- [ ] **Step 4: Create the empty notebook bound to the `rag_env` kernel**

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

Save as `rag/03_chunking_and_document_loading.ipynb`.

- [ ] **Step 5: Verify and commit**

```bash
python -c "import json; json.load(open('rag/03_chunking_and_document_loading.ipynb'))"
ls -la rag/data/porter_p2_field_guide.pdf rag/03_chunking_and_document_loading.ipynb
git add rag/data/porter_p2_field_guide.pdf rag/03_chunking_and_document_loading.ipynb
git commit -m "feat(rag): add sample PDF and scaffold notebook 03"
```

---

## Task 2: Add intro markdown — title, "Why this notebook exists", and "What you'll learn"

**Files:**
- Modify: `rag/03_chunking_and_document_loading.ipynb` (add cells 1–2)

- [ ] **Step 1: Add the title + motivation markdown cell**

Cell content (markdown):

```markdown
# 03 — Chunking and Document Loading

## Why this notebook exists

Notebook 02 ranked whole documents by embedding each one as a single vector. That worked because our documents are tiny and each is about one thing. Real documents aren't: a support page covers warranty *and* hours *and* spare-part depots, and a PDF manual runs for pages. Squash all of that into one vector and it blurs — a question about spare parts and a question about warranty both match the same averaged document, and specific details get washed out.

The fix is to break documents into smaller, focused **chunks** and embed those, so retrieval can return the exact passage that answers a question. But first we have to *get the documents in* — including a PDF, which needs real extraction. This notebook does both: load the corpus (markdown + a PDF, via `pypdf`) into text with `source`/`page` metadata, split it into well-sized overlapping chunks, and show chunk-level retrieval returning the precise passage — with a citation — instead of a whole averaged document.

Requires an `OPENAI_API_KEY` (we embed chunks at the end to show the payoff). Still no vector database — just NumPy.
```

- [ ] **Step 2: Add the "What you'll learn" markdown cell**

Cell content (markdown):

```markdown
## What you'll learn

- How to **load** a mixed corpus — markdown files and a **PDF** (with `pypdf`) — into a uniform list of text units carrying `source` and `page` metadata.
- Why a single embedding per document is too coarse, and why **chunking** improves retrieval granularity.
- Three **chunking strategies**: naive fixed-size character splits, **token-based** splits (`tiktoken`), and **recursive/structure-aware** splits (`langchain-text-splitters`) — and the role of **chunk overlap**.
- How to attach **metadata** (`source`, `page`, `chunk_id`) to every chunk so retrieved passages can be cited.
- How **chunk-level retrieval** returns the exact relevant passage — the fix for notebook 02's whole-document blur.
```

- [ ] **Step 3: Commit**

```bash
git add rag/03_chunking_and_document_loading.ipynb
git commit -m "feat(rag): add intro and What You'll Learn to notebook 03"
```

---

## Task 3: Section 1 — Setup + guard + helpers

**Files:**
- Modify: `rag/03_chunking_and_document_loading.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 1. Setup

We reuse notebook 02's embedding toolkit — `embed`, `embed_many`, and `cosine` — re-declared inline so this notebook stands alone. We load `OPENAI_API_KEY` from the environment (and from a local `.env` if `python-dotenv` is installed). Document loading and chunking themselves need no API; the key is used only at the end (Section 6) to embed chunks and show the retrieval payoff.
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
    print("This notebook embeds chunks at the end (Section 6) to show the")
    print("retrieval payoff. Loading and chunking themselves need no key.")
    print()
    print("Set it and restart the kernel:")
    print("  export OPENAI_API_KEY=sk-...")
    raise SystemExit("Set OPENAI_API_KEY and restart the kernel to continue.")

print("OPENAI_API_KEY set ✓")
```

- [ ] **Step 3: Add the embedding-helpers code cell**

```python
import numpy as np
from openai import OpenAI

EMBED_MODEL = "text-embedding-3-small"

openai_client = OpenAI()  # reads OPENAI_API_KEY from the environment


def embed(text: str) -> np.ndarray:
    """Embed a single string into a NumPy vector."""
    resp = openai_client.embeddings.create(model=EMBED_MODEL, input=text)
    return np.array(resp.data[0].embedding, dtype=np.float32)


def embed_many(texts: list) -> np.ndarray:
    """Embed a list of strings in ONE API call; returns a (len(texts), dim) matrix."""
    resp = openai_client.embeddings.create(model=EMBED_MODEL, input=texts)
    return np.array([d.embedding for d in resp.data], dtype=np.float32)


def cosine(a: np.ndarray, b: np.ndarray) -> float:
    """Cosine similarity between two vectors (higher = more similar)."""
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))


print("Setup OK")
print(f"Embedding model: {EMBED_MODEL}")
```

- [ ] **Step 4: Run the cells and verify**

Expected: `OPENAI_API_KEY set ✓` then `Setup OK` and the model name. No exceptions.

- [ ] **Step 5: Commit**

```bash
git add rag/03_chunking_and_document_loading.ipynb
git commit -m "feat(rag): add setup, key guard, and embedding helpers to notebook 03"
```

---

## Task 4: Section 2 — Loading documents

**Files:**
- Modify: `rag/03_chunking_and_document_loading.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 2. Loading Documents

Retrieval needs raw text, but our corpus is mixed: markdown files and a PDF. We load both into one uniform shape — a list of dicts, each with `source` (filename), `page` (a 1-based page number for PDFs, or `None` for markdown), and `text`.

Markdown is just a file read. The PDF needs real extraction: `pypdf` opens it and gives us the text of each page separately, which is why PDF pages become separate units — page numbers are useful metadata for citing where an answer came from.
```

- [ ] **Step 2: Add the `load_documents` code cell**

```python
from pathlib import Path
from pypdf import PdfReader

# Resolve the data folder whether the kernel runs from rag/ or from the repo root.
DATA_DIR = Path("data")
if not DATA_DIR.exists():
    DATA_DIR = Path("rag/data")


def load_documents(data_dir: Path) -> list:
    """Load every .md and .pdf in data_dir into a uniform list of text units.

    Each unit is a dict: {"source": filename, "page": int|None, "text": str}.
    Markdown files become one unit (page=None); each PDF page becomes its own
    unit (page=1, 2, ...), so we can cite the page an answer came from.
    """
    docs = []
    for path in sorted(data_dir.glob("*.md")):
        docs.append({"source": path.name, "page": None, "text": path.read_text(encoding="utf-8")})
    for path in sorted(data_dir.glob("*.pdf")):
        reader = PdfReader(str(path))
        for i, page in enumerate(reader.pages):
            docs.append({"source": path.name, "page": i + 1, "text": page.extract_text() or ""})
    return docs


documents = load_documents(DATA_DIR)

# Fail clearly if nothing loaded, instead of a confusing error later.
if not documents:
    raise FileNotFoundError(
        f"No documents found in {DATA_DIR}/. Run notebook 01 (and notebook 03's "
        "PDF-authoring step) first."
    )

print(f"Loaded {len(documents)} document units (markdown files + PDF pages):\n")
for d in documents:
    loc = d["source"] if d["page"] is None else f"{d['source']} p.{d['page']}"
    print(f"  {loc:<34} {len(d['text']):>5} chars")
```

- [ ] **Step 3: Add the PDF-inspection code cell**

```python
# The PDF is where loading does real work — show what pypdf extracted.
pdf_units = [d for d in documents if d["source"].endswith(".pdf")]
print(f"The PDF contributed {len(pdf_units)} page-units.\n")
print("Page 1 extracted text:")
print("-" * 60)
print(pdf_units[0]["text"])
```

- [ ] **Step 4: Run the cells and verify**

Expected: the loader lists the five markdown files (page `None`) plus two PDF page-units (`porter_p2_field_guide.pdf p.1` and `p.2`), i.e. **7 units total**. The inspection cell prints page 1's extracted text, which mentions the preventive maintenance schedule (e.g. "250 operating hours", "air filter"). No exception.

- [ ] **Step 5: Commit**

```bash
git add rag/03_chunking_and_document_loading.ipynb
git commit -m "feat(rag): add document loading (markdown + PDF) to notebook 03"
```

---

## Task 5: Section 3 — Why chunk?

**Files:**
- Modify: `rag/03_chunking_and_document_loading.ipynb` (add 1 markdown cell)

- [ ] **Step 1: Add the markdown cell**

```markdown
## 3. Why Chunk?

Now that documents are loaded, why not just embed each one whole, like notebook 02 did? Two reasons:

1. **Granularity.** `support.md` covers warranty, support hours, *and* spare-part depots. As one vector it's an average of all three topics, so it matches every related question only weakly and never strongly. Split it into a warranty chunk, an hours chunk, and a depots chunk, and a question about spare parts matches the depots chunk *precisely*.
2. **Length.** Embedding models have an input limit and lose fidelity on long text. A 30-page PDF cannot be one useful vector. Chunks keep each embedded unit short and focused.

So we split documents into **chunks**: small spans of text, each embedded on its own. The art is choosing *where* to split. Too large and you're back to the blurring problem; too small and a chunk loses the context needed to make sense. The next section compares strategies.

> **Gotcha:** A good chunk is semantically self-contained — it should still make sense on its own when a model reads it in isolation. Splitting blindly every N characters often cuts a sentence (or a number and its unit) in half, which is why structure-aware splitting and overlap matter.
```

- [ ] **Step 2: Commit**

```bash
git add rag/03_chunking_and_document_loading.ipynb
git commit -m "feat(rag): add 'why chunk' motivation to notebook 03"
```

---

## Task 6: Section 4 — Chunking strategies

**Files:**
- Modify: `rag/03_chunking_and_document_loading.ipynb` (add 1 markdown cell + 2 code cells)

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 4. Chunking Strategies

We'll compare three ways to split text, using `support.md` as the sample:

1. **Naive fixed-size** — slice every N characters. Simple, but it cuts blindly through words and sentences.
2. **Recursive / structure-aware** — `RecursiveCharacterTextSplitter` from `langchain-text-splitters` tries to split on paragraph, then sentence, then word boundaries, so chunks end at natural breaks.
3. **Token-based** — split by *token* count (what the model and the embedding API actually measure) rather than characters, using a `tiktoken` encoder.

We also show **overlap**: letting consecutive chunks share a little text so a fact that straddles a boundary isn't lost.
```

- [ ] **Step 2: Add the naive-vs-recursive code cell**

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

sample = next(d for d in documents if d["source"] == "support.md")["text"]
print(f"Sample: support.md ({len(sample)} chars)\n")

# 1. Naive fixed-size character chunks
def fixed_chunks(text: str, size: int = 200) -> list:
    return [text[i:i + size] for i in range(0, len(text), size)]

naive = fixed_chunks(sample, 200)
print(f"Naive fixed 200-char chunks: {len(naive)}")
print(f"  chunk 0 ends: ...{naive[0][-45:]!r}")
print("  ^ note how it can cut mid-word / mid-sentence\n")

# 2. Recursive / structure-aware chunks (respects paragraph & sentence breaks)
splitter = RecursiveCharacterTextSplitter(chunk_size=200, chunk_overlap=40)
recursive = splitter.split_text(sample)
print(f"Recursive 200-char chunks (overlap 40): {len(recursive)}")
print(f"  chunk 0 ends: ...{recursive[0][-45:]!r}")
print("  ^ ends on a natural boundary")
```

- [ ] **Step 3: Add the token-based + overlap code cell**

```python
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4o-mini")

# 3. Token-based splitting: chunk by token count, not character count.
tok_splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(
    chunk_size=60, chunk_overlap=15
)
tok_chunks = tok_splitter.split_text(sample)
print(f"Token-based chunks (~60 tokens, overlap 15): {len(tok_chunks)}")
for i, c in enumerate(tok_chunks):
    print(f"  chunk {i}: {len(enc.encode(c)):>3} tokens, {len(c):>3} chars")

# Overlap in action: trailing text of one chunk reappears at the start of the next.
if len(recursive) > 1:
    print("\nOverlap carries context across the boundary:")
    print(f"  end of recursive chunk 0  : ...{recursive[0][-40:]!r}")
    print(f"  start of recursive chunk 1: {recursive[1][:40]!r}...")
```

- [ ] **Step 4: Run the cells and verify**

Expected: the naive splitter produces chunks whose first chunk ends mid-text; the recursive splitter produces a comparable number of chunks ending on cleaner boundaries; the token-based cell lists each chunk's token count (each ≈ 60 or fewer). No exception. Exact counts depend on the splitter but should be small (single digits) for this short document.

- [ ] **Step 5: Commit**

```bash
git add rag/03_chunking_and_document_loading.ipynb
git commit -m "feat(rag): add chunking-strategies comparison to notebook 03"
```

---

## Task 7: Section 5 — Chunks with metadata

**Files:**
- Modify: `rag/03_chunking_and_document_loading.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 5. Chunks With Metadata

Now we chunk the *whole* corpus into the canonical list we'll embed and search. Each chunk is a dict carrying the metadata we'll need later to cite answers: a unique `chunk_id`, its `source` document, its `page` (for PDF chunks), and the `text`. This list — chunks plus metadata — is exactly what the vector store in notebook 04 will hold.

We use the recursive splitter with a slightly larger window (300 chars, 50 overlap) so each chunk is a coherent passage.
```

- [ ] **Step 2: Add the `chunk_documents` code cell**

```python
from collections import Counter


def chunk_documents(documents: list, chunk_size: int = 300, chunk_overlap: int = 50) -> list:
    """Split every document unit into chunk records carrying metadata.

    Returns a list of dicts: {"chunk_id", "source", "page", "text"}.
    """
    splitter = RecursiveCharacterTextSplitter(chunk_size=chunk_size, chunk_overlap=chunk_overlap)
    chunks = []
    cid = 0
    for d in documents:
        for piece in splitter.split_text(d["text"]):
            chunks.append({
                "chunk_id": cid,
                "source": d["source"],
                "page": d["page"],
                "text": piece,
            })
            cid += 1
    return chunks


chunks = chunk_documents(documents)
print(f"{len(documents)} document units → {len(chunks)} chunks\n")

print("First 3 chunk records:")
for ch in chunks[:3]:
    loc = ch["source"] if ch["page"] is None else f"{ch['source']} p.{ch['page']}"
    print(f"  chunk {ch['chunk_id']:>2} | {loc:<30} | {ch['text'][:48]!r}...")

print("\nChunks per source:")
for src, n in sorted(Counter(c["source"] for c in chunks).items()):
    print(f"  {src:<32} {n}")
```

- [ ] **Step 3: Run the cell and verify**

Expected: the corpus (7 document units) produces more chunks than units (each multi-paragraph doc splits into several). The per-source breakdown lists all six source files (five `.md` plus `porter_p2_field_guide.pdf`). Each printed chunk record shows `chunk_id`, a `source`/`page` location, and a text preview. No exception.

- [ ] **Step 4: Commit**

```bash
git add rag/03_chunking_and_document_loading.ipynb
git commit -m "feat(rag): add chunk-records-with-metadata builder to notebook 03"
```

---

## Task 8: Section 6 — Chunk-level retrieval beats document-level

**Files:**
- Modify: `rag/03_chunking_and_document_loading.ipynb` (add 1 markdown cell + 1 code cell)

- [ ] **Step 1: Add the section heading markdown cell**

```markdown
## 6. Chunk-Level Retrieval Beats Document-Level

Here's the payoff. We embed every chunk (one batched call), then search the same way notebook 02 searched documents — but now the unit returned is a focused **passage** with a citation, not a whole averaged document. Ask about a PDF-only error code and retrieval pinpoints the exact page; ask about spare parts and it returns the depots passage from `support.md`, not the entire support document.
```

- [ ] **Step 2: Add the chunk-search code cell**

```python
chunk_texts = [c["text"] for c in chunks]
chunk_matrix = embed_many(chunk_texts)
print(f"Embedded {len(chunks)} chunks into {chunk_matrix.shape}.\n")


def search_chunks(query: str, top_k: int = 2) -> list:
    """Return the top_k (chunk, score) pairs most similar to the query."""
    q = embed(query)
    scored = [(i, cosine(q, chunk_matrix[i])) for i in range(len(chunks))]
    scored.sort(key=lambda pair: pair[1], reverse=True)
    return [(chunks[i], s) for i, s in scored[:top_k]]


for query in [
    "What should I do about error code E204?",
    "Where are spare parts stocked?",
]:
    print(f"Q: {query}")
    for ch, score in search_chunks(query, top_k=2):
        loc = ch["source"] if ch["page"] is None else f"{ch['source']} p.{ch['page']}"
        snippet = " ".join(ch["text"].split())[:80]
        print(f"   {score:.3f}  [{loc}]  {snippet!r}...")
    print()
```

- [ ] **Step 3: Run the cell and verify**

Expected: chunks embed into a `(n_chunks, 1536)` matrix. For the **E204** query, the #1 chunk comes from `porter_p2_field_guide.pdf p.2` and its text mentions E204 / lidar obstruction. For the **spare parts** query, the #1 chunk comes from `support.md` and mentions the Portland/Dallas/Rotterdam depots. Exact scores vary, but the top chunk's `source` (and page, for E204) must be correct — this is the structural payoff and worth reporting if it's wrong.

- [ ] **Step 4: Commit**

```bash
git add rag/03_chunking_and_document_loading.ipynb
git commit -m "feat(rag): add chunk-level retrieval payoff to notebook 03"
```

---

## Task 9: Closing cells

**Files:**
- Modify: `rag/03_chunking_and_document_loading.ipynb` (add 2 markdown cells)

- [ ] **Step 1: Add the "What you just learned" markdown cell**

```markdown
## What you just learned

- **Loading** a mixed corpus into a uniform shape: markdown files plus a **PDF** read with `pypdf`, each unit carrying `source` and `page` metadata.
- Why one vector per document is too coarse — **chunking** restores granularity and keeps each embedded unit short enough to be faithful.
- Three **chunking strategies**: naive fixed-size (cuts blindly), **recursive/structure-aware** (splits on natural boundaries), and **token-based** (`tiktoken`) — plus **overlap** to preserve context across boundaries.
- Building **chunk records with metadata** (`chunk_id`, `source`, `page`, `text`) — the unit a vector store holds and retrieval returns.
- **Chunk-level retrieval** returns the exact relevant passage *with a citation*, fixing notebook 02's whole-document blur.
```

- [ ] **Step 2: Add the "What's missing" markdown cell**

```markdown
## What's missing

We're embedding every chunk and scanning all of them with a NumPy loop on each query. That's fine for a few dozen chunks, but it's a brute-force linear scan: at thousands or millions of chunks, embedding everything on the fly and comparing against every vector per query is far too slow, and we throw the embeddings away when the kernel restarts.

**Notebook 04 — `04_vector_store_with_faiss.ipynb`** introduces a real **vector store**: we build a FAISS index over the chunk embeddings once, query it in milliseconds with approximate nearest-neighbor search, keep the chunk metadata alongside it, and **persist** the index to disk so ingestion is a one-time cost. That's the last piece before we assemble the full retrieve-augment-generate pipeline in notebook 05.
```

- [ ] **Step 3: Commit**

```bash
git add rag/03_chunking_and_document_loading.ipynb
git commit -m "feat(rag): add closing cells to notebook 03"
```

---

## Task 10: End-to-end verification in a fresh kernel

**Files:** none (verification only).

- [ ] **Step 1: Confirm prerequisites**

```bash
python -c "import openai, numpy, pypdf, tiktoken, langchain_text_splitters; print('deps OK')"
ls rag/data/*.md rag/data/*.pdf
echo "OPENAI_API_KEY set: $(test -n \"$OPENAI_API_KEY\" && echo YES || echo NO)"
```

`OPENAI_API_KEY` must be available; the five `.md` files and the `.pdf` must exist. (The `deps OK` check should use the `rag_env` python if the default differs — all five imports must succeed there.)

- [ ] **Step 2: Execute the notebook top-to-bottom in a clean kernel**

```bash
export OPENAI_API_KEY=$(python -c "from dotenv import dotenv_values; print(dotenv_values('.env').get('OPENAI_API_KEY',''))")
python -m jupyter nbconvert --to notebook --execute --inplace \
  --ExecutePreprocessor.timeout=300 \
  --ExecutePreprocessor.kernel_name=rag_env \
  "rag/03_chunking_and_document_loading.ipynb"
```

Expected: succeeds with no errors. (Command substitution keeps the key out of the command text and output. If `rag_env` cannot be launched headlessly, substitute a kernel that has `openai` + `numpy` + `pypdf` + `tiktoken` + `langchain_text_splitters`.)

- [ ] **Step 3: Verify structural outcomes in the executed notebook**

Confirm (exact counts/scores vary — check structure and rankings):

1. **Section 1** — guard prints `OPENAI_API_KEY set ✓`; helpers cell prints `Setup OK`.
2. **Section 2** — 7 document units loaded (5 markdown + 2 PDF pages); the PDF-inspection cell prints page-1 text mentioning the maintenance schedule.
3. **Section 4** — naive vs recursive chunk demos print; token-based cell lists per-chunk token counts (each ≈ 60 or fewer).
4. **Section 5** — corpus chunks into more chunks than the 7 units; the per-source table lists all six source files.
5. **Section 6** — chunk matrix is `(n_chunks, 1536)`; the **E204** query's #1 chunk is from `porter_p2_field_guide.pdf p.2`; the **spare parts** query's #1 chunk is from `support.md`.
6. No cell raises an unhandled exception.

- [ ] **Step 4: Commit the clean run**

```bash
git add rag/03_chunking_and_document_loading.ipynb
git commit -m "chore(rag): commit clean fresh-kernel run of notebook 03"
```

---

## Done

After Task 10 passes, notebook 03 is complete: the corpus (now including a PDF) loads into uniform units with metadata, chunks into citable passages, and chunk-level retrieval returns the exact relevant passage. The bundled `rag/data/porter_p2_field_guide.pdf` and the `load_documents` / `chunk_documents` patterns are reused by notebooks 04+.

The next plan to write is `2026-05-25-rag-notebook-04-vector-store-with-faiss.md`, which builds a FAISS index over the chunk embeddings, queries it with approximate nearest-neighbor search, keeps chunk metadata alongside, and persists the index to disk — replacing this notebook's brute-force NumPy scan.
```