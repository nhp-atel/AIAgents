# RAG from First Principles — Learning Series Design

**Date:** 2026-05-25
**Author:** Nimesh Patel (with Claude)
**Status:** Approved design, pending implementation plan

## Purpose

Teach **Retrieval-Augmented Generation (RAG) from first principles** — how an agent gets the *knowledge* it reasons over. The existing series cover how agents are wired together (CrewAI, LangGraph), how they communicate (MCP, A2A), and how they are evaluated (`evals/`). None of them answer the question every knowledge-grounded agent faces: *how do I let a model answer from a corpus it was never trained on, accurately and with citations, without stuffing everything into the prompt?*

The series lives in a **new top-level folder `rag/`**, sibling to `mcp/`, `a2a/`, `evals/`, `langgraph/`, and `crewai/`. It is a progressive 8-notebook series (`01_..._.ipynb` through `08_..._.ipynb`), mirroring the pedagogical structure of the existing series: build from first principles, then make it real, then make it good.

The learner should finish able to:

1. Explain *why* naive "put the whole document in the prompt" fails — context-window limits, cost/latency, and "lost in the middle" — and what RAG does instead.
2. Understand **embeddings and semantic search**: what an embedding is, cosine similarity, and why semantic retrieval beats keyword matching.
3. **Load and chunk documents** (markdown/text and PDF) into retrievable units with sensible size, overlap, and metadata.
4. Build, query, and persist a **FAISS vector store** with an associated metadata store, and filter by metadata.
5. Assemble a complete **retrieve → augment → generate** pipeline that returns grounded, cited answers.
6. Improve retrieval quality with **MMR, LLM-based reranking, and query transformation (rewriting / HyDE)**.
7. **Evaluate RAG** with retrieval metrics (recall@k / hit-rate) and answer faithfulness, reusing the `evals/` harness and LLM judge.
8. Build **agentic RAG**: expose retrieval as a tool the agent decides when and how to call, including multi-hop retrieval and abstention.

## Non-goals

The following are **explicitly out of scope** for this series:

- A survey of every vector database (Chroma, Pinecone, Qdrant, Weaviate, pgvector). We use FAISS and name alternatives once.
- Training or fine-tuning embedding models or rerankers. We use hosted OpenAI embeddings and prompt-engineered/LLM rerankers.
- Production retrieval infrastructure: index sharding, distributed/streaming ingestion pipelines, incremental re-indexing at scale.
- Multimodal RAG (images, tables beyond basic PDF text extraction) and GraphRAG / knowledge-graph retrieval.
- Deep hybrid/lexical search. BM25 + dense fusion (RRF) and cross-encoder/Cohere rerankers are *named* as further options in notebook 06, not built — `rank-bm25` and cross-encoder models are not in the target environment.
- Re-teaching how to *build* agents. Notebook 08's agent is the smallest tool loop the lesson needs; we link back to `langgraph/` and `mcp/` rather than re-explaining agent construction.

## Audience and prerequisites

The learner already understands:

- Python (functions, classes, basic typing; light NumPy array/vector basics).
- What an LLM, prompt, and completion are; ideally has seen a tool-using agent loop (as taught in `langgraph/` or `mcp/`).
- Basic intuition for what "similarity" between two pieces of text might mean.

The learner does **not** need prior knowledge of:

- Embeddings, vector databases, or any RAG framework/methodology.
- FAISS or any specific vector store.
- The internals of the agent built in notebook 08 — it is small and self-contained.

## Pedagogical arc — "build it, make it real, make it good"

The series climbs from a naive baseline, through the mechanics of retrieval, to a complete pipeline, then improves and measures it, and finally hands control to the agent:

1. **Feel the pain** (01) — answer from a document by stuffing it all into the prompt; watch it break as the corpus grows. Why we need retrieval.
2. **Measure meaning** (02) — embeddings and cosine similarity; brute-force semantic search by hand in NumPy.
3. **Prepare the corpus** (03) — load real documents (md/txt + PDF) and chunk them well, with metadata.
4. **Index at scale** (04) — a FAISS vector store: build, query, persist, filter.
5. **Close the loop** (05) — retrieve → augment → generate, with grounded, cited answers.
6. **Improve retrieval** (06) — MMR, LLM reranking, query rewriting, and HyDE.
7. **Prove it works** (07) — retrieval and answer-quality metrics, reusing the `evals/` harness.
8. **Hand it to the agent** (08) — retrieval as a tool the agent calls when and how it decides; multi-hop and abstention.

Every notebook is **runnable end-to-end** in Jupyter given `OPENAI_API_KEY`. Each opens with a "Why this notebook exists" + "What you'll learn" pair, builds straight through with `### Try it` cells, and closes with a "What you just learned" recap and a "What's missing" teaser motivating the next notebook (omitted in notebook 08, which closes the series).

## Common stack

| Concern | Choice |
|---|---|
| Python | 3.11+ (the bundled `rag_env` venv) |
| Notebooks | Jupyter |
| Embeddings | OpenAI `text-embedding-3-small` (default) |
| Vector store | **FAISS** (`faiss-cpu`) — local, no server |
| Chunking | `tiktoken` (token counting) + `langchain-text-splitters` |
| Document loading | `pypdf` for PDFs; plain readers for `.md`/`.txt` |
| Similarity math (02) | NumPy, by hand, before any vector DB |
| LLM (generation, reranking, judge) | OpenAI, `gpt-4o-mini` by default |
| Eval harness (07) | reuse the `evals/` pattern (`Score`, `Example`, `run_eval`) + `make_llm_judge` |
| Agent (08) | minimal hand-rolled tool loop (no new dependency); LangGraph noted as the production path, which would add `langgraph` |
| LLM provider | **OpenAI** (`OPENAI_API_KEY`), matching `langgraph/`, `crewai/`, and `evals/` |

**`OPENAI_API_KEY` is required for all eight notebooks.** Unlike the `evals/` series (which kept 01–04 keyless), RAG needs embeddings from the start, so there is no keyless on-ramp: even notebook 01's naive baseline calls the chat API. There is **no LangSmith or other account requirement** — FAISS is local and the only external dependency is the OpenAI API.

**No local servers.** Like `evals/` and unlike `mcp/`/`a2a/`, this series stands up no HTTP servers and needs no port allocation. FAISS runs in-process; the OpenAI API is reached over HTTPS.

## Environment

The user has a **`rag_env` Jupyter kernel** (venv at `Nimesh_Playground/RAG/.venv`) already provisioned with the target stack: `faiss-cpu`, `langchain` (+ `-core`, `-community`, `-openai`, `-text-splitters`), `numpy`, `openai`, `pypdf`, and `tiktoken`. The series targets these packages and adds **no new dependencies** on its default path (notebook 08 uses a hand-rolled tool loop; the optional LangGraph variant noted in the open questions would add `langgraph`). The install line for a fresh environment:

```
pip install faiss-cpu langchain langchain-openai langchain-community langchain-text-splitters tiktoken pypdf numpy openai
```

## Notebooks

### 01 — `01_why_naive_rag_fails.ipynb` — The no-retrieval baseline
**Goal:** establish why retrieval is necessary before introducing any of its machinery.

- Answer a question by putting an entire small document into the prompt — and watch it work.
- Scale up (more documents / a long PDF) and hit the wall: context-window limits, rising cost and latency, and "lost in the middle" (a relevant fact buried in a long context is missed).
- Show irrelevant content degrading the answer even when everything *fits*.
- Define RAG conceptually: retrieve only the relevant pieces, then generate.
- Closing: *"We need to find the relevant chunks instead of sending everything — which means measuring relevance. That's embeddings."*
- Requires `OPENAI_API_KEY` (chat).

### 02 — `02_embeddings_and_semantic_search.ipynb` — Measuring meaning
**Goal:** the core mechanic of retrieval, made transparent.

- What an embedding is (a vector); generate embeddings with OpenAI `text-embedding-3-small`.
- **Cosine similarity** computed by hand in NumPy on a few sentences; show semantically related text scoring higher than lexically similar but unrelated text.
- Build a tiny brute-force semantic search over a handful of texts: embed corpus + query, rank by cosine, return top-k. No vector DB yet — NumPy keeps the mechanic visible.
- Contrast with keyword search (why "car" retrieves "automobile").
- Closing: *"This works for ten texts; real corpora have thousands of chunks. First we need chunks (03), then a real index (04)."*
- Requires `OPENAI_API_KEY` (embeddings).

### 03 — `03_chunking_and_document_loading.ipynb` — Turning documents into retrievable units
**Goal:** get real documents in, and split them well.

- Load the bundled corpus: `.md`/`.txt` readers plus `pypdf` for the PDF; normalize to text with metadata (`source`, `page`).
- Why chunk at all: embeddings degrade on long text; retrieval needs the right granularity.
- Chunking strategies with concrete demos: fixed-character, token-based (`tiktoken`), and recursive/structure-aware (`langchain-text-splitters`); the role of **chunk overlap**; the size/overlap tradeoff.
- Attach metadata to each chunk (`source`, `page`, `chunk_id`).
- Closing: *"We have good chunks and can embed them — but brute-force NumPy search won't scale. Enter the vector store."*
- Requires `OPENAI_API_KEY` (embeddings, to show chunk embedding).

### 04 — `04_vector_store_with_faiss.ipynb` — A real index
**Goal:** store and search embeddings at scale.

- Why brute-force NumPy doesn't scale; one-paragraph intuition for approximate nearest-neighbor search.
- Build a **FAISS** index over the chunked corpus, with a parallel metadata store keyed by vector id.
- Query: embed the query → FAISS search → top-k chunks with similarity scores and metadata.
- **Persist** the index (and metadata) to disk and reload it, so ingestion is a one-time cost.
- **Metadata filtering** (restrict retrieval to a source/page range).
- Name alternatives once (Chroma, Pinecone, Qdrant, pgvector) and move on.
- Closing: *"We can retrieve relevant chunks fast. Now wire retrieval into generation — the full RAG loop."*
- Requires `OPENAI_API_KEY`.

### 05 — `05_the_rag_pipeline.ipynb` — Retrieve → augment → generate
**Goal:** the first complete, real RAG system.

- Assemble the loop: ingest (03) → index (04) → retrieve → build an augmented prompt → generate with the chat model.
- **Prompt construction**: how to insert retrieved context, instruct the model to ground its answer, and request citations.
- Return the answer plus cited sources (from chunk metadata).
- Demonstrate it answering questions the naive baseline (01) got wrong or couldn't fit.
- Wrap it in a reusable `rag_answer(query) -> {answer, sources}` function.
- Preview failure modes: wrong chunks retrieved, ungrounded answers, missing information.
- Closing: *"It works — but retrieval quality is everything. Bad chunks mean bad answers. Let's improve retrieval (06)."*
- Requires `OPENAI_API_KEY`.

### 06 — `06_better_retrieval.ipynb` — Reranking and query transformation
**Goal:** improve what gets retrieved before it reaches the model.

- The problem: top-k by cosine isn't always best — redundant chunks, and query/document vocabulary mismatch.
- **MMR (Maximal Marginal Relevance)** for diversity (no new dependency).
- **LLM-based reranking**: retrieve a larger candidate set, ask the chat model to score/reorder by relevance, keep the best (no new dependency).
- **Query transformation**: query rewriting and **HyDE** (Hypothetical Document Embeddings — generate a hypothetical answer, embed it, retrieve against that).
- Name hybrid search (BM25 + dense + RRF) and cross-encoder/Cohere rerankers as further options, one line each.
- Closing: *"We've made retrieval better — but how do we *know* it's better? Measure it (07)."*
- Requires `OPENAI_API_KEY`.

### 07 — `07_evaluating_rag.ipynb` — Does retrieval actually help?
**Goal:** measure RAG quality, reusing the `evals/` series harness.

- Two layers of metrics: **retrieval** (recall@k / hit-rate / MRR — did we fetch the right chunk?) and **answer** (faithfulness/groundedness and answer relevance, via an LLM judge).
- Build a small labeled eval set over the bundled corpus (questions plus the ground-truth source/chunk that answers each).
- Reuse the `evals/` pattern: a `Score`/`Example`/`run_eval`-style harness plus `make_llm_judge` for faithfulness — re-declared inline so the notebook stands alone.
- Compare configurations — naive top-k vs reranked vs HyDE — and show the numbers move.
- Mention automated frameworks (e.g. RAGAS) once; we hand-roll for understanding.
- Explicit cross-links to `evals/03` (harness) and `evals/05` (LLM judge).
- Closing: *"We can build, improve, and measure RAG. The last step: let the agent drive retrieval itself."*
- Requires `OPENAI_API_KEY`.

### 08 — `08_agentic_rag.ipynb` — The agent decides when and what to retrieve
**Goal:** the payoff — retrieval as an agent capability, not a fixed pipeline.

- The limitation of fixed RAG: it always retrieves once, with the raw query; it can't decide it doesn't need to retrieve, and can't do follow-up searches.
- **Agentic RAG**: expose retrieval as a `search_corpus(query)` **tool** over the FAISS index; the agent decides whether and when to call it, can rewrite the query, and can retrieve multiple times.
- Build a small retrieval agent — a minimal hand-rolled tool loop (or light LangGraph) — over the index built earlier.
- Demonstrate **multi-hop**: a question requiring two retrievals; the agent decomposes and searches twice.
- Demonstrate **abstention**: a question answerable without retrieval; the agent skips the tool.
- Tie back: this is the `langgraph/`/`mcp/` tool-loop pattern with a retrieval tool, and it can be measured with notebook 07's harness.
- Closing cell: a narrative recap of the climb (stuff-in-context → embeddings → chunking → FAISS → pipeline → reranking → evaluation → agentic), and "where to go next" pointers — expose retrieval via `mcp/`, coordinate multiple retrieval agents via `a2a/`, and evaluate with `evals/`.
- No "What's missing" cell (this closes the series).
- Requires `OPENAI_API_KEY`.

## File layout

```
rag/
  data/                              # bundled corpus (committed): a few .md/.txt docs + one PDF
  01_why_naive_rag_fails.ipynb
  02_embeddings_and_semantic_search.ipynb
  03_chunking_and_document_loading.ipynb
  04_vector_store_with_faiss.ipynb
  05_the_rag_pipeline.ipynb
  06_better_retrieval.ipynb
  07_evaluating_rag.ipynb
  08_agentic_rag.ipynb
```

The new `rag/` folder sits at the repository root alongside `mcp/`, `a2a/`, `evals/`, `langgraph/`, and `crewai/`. The persisted FAISS index built in notebook 04 is written under `rag/` (e.g. `rag/.index/`) and git-ignored. The repo `README.md` gains a `rag/` section (mirroring the existing series tables) and a prerequisites note: all eight notebooks need an `OPENAI_API_KEY`; no other account is required.

## Style conventions

Each notebook opens with:

- A `## Why this notebook exists` markdown cell (one paragraph of motivation; for 02–08, a one-line recap of where the previous notebook left off).
- A `## What you'll learn` bulleted list.

Body sections use numbered headers (`## 1. ...`, `## 2. ...`) with `### Try it` cells the learner can run and modify. **"Gotcha:"** callouts are markdown blockquotes inserted only where RAG behavior is genuinely surprising (e.g., chunk-boundary effects, "lost in the middle", embedding/vocabulary mismatch, judge bias) — not in every cell.

Each notebook closes with:

- `## What you just learned` — bulleted recap.
- `## What's missing` — one paragraph motivating the next notebook (omitted in notebook 08, which closes the series).

This matches the existing `mcp/`, `a2a/`, and `evals/` series style.

## Success criteria

The series is complete when:

1. All 8 notebooks exist under `rag/`, plus a committed `rag/data/` corpus including at least one PDF.
2. Each notebook runs top-to-bottom in a fresh kernel with no errors, given `OPENAI_API_KEY`.
3. No external server or account beyond the OpenAI API is required; the FAISS index builds, persists, and reloads locally.
4. Notebook 05 answers at least one question correctly that the notebook 01 naive baseline gets wrong or cannot fit.
5. Notebook 07 shows a measurable difference between retrieval configurations (e.g., reranked/HyDE vs naive top-k).
6. Notebook 08 demonstrates both multi-hop retrieval and abstention (a no-retrieval answer).
7. Every concept claimed in this spec is demonstrated with a runnable cell in the corresponding notebook.
8. The repo `README.md` documents the series consistently with the others.

## Open questions deferred to implementation planning

- The specific bundled corpus topic and which PDF to include (leaning toward a single coherent topic so cross-document retrieval is meaningful).
- `text-embedding-3-small` vs `-large` (leaning `-small` for cost; the dimension is parameterized so it is a one-line change).
- Whether notebook 08 uses a hand-rolled tool loop or a light LangGraph graph (leaning hand-rolled to avoid heavy coupling, with LangGraph noted as the production path).
- Exact dependency pins and whether the notebooks declare the `rag_env` kernel or the repo's default kernel.

These do not affect the design and will be decided in the implementation plan.
