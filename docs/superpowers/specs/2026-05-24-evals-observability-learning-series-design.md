# Agent Evals & Observability Learning Series — Design

**Date:** 2026-05-24
**Author:** Nimesh Patel (with Claude)
**Status:** Approved design, pending implementation plan

## Purpose

Teach **how to evaluate AI agents and observe them in production** — the missing production leg of this repo. The existing series cover how agents are *wired together* (CrewAI, LangGraph) and how they *communicate* (MCP, A2A). None of them answer the questions every shipped agent eventually faces: *How do I know it works? How do I stop it from regressing? What is it actually doing when it runs?*

The series lives in a **new top-level folder `evals/`**, sibling to `mcp/`, `a2a/`, `langgraph/`, and `crewai/`. It is a progressive 8-notebook series (`01_..._.ipynb` through `08_..._.ipynb`), mirroring the pedagogical structure of the existing series: build from first principles, then introduce the real tool.

The learner should finish able to:

1. Explain *why* agents are uniquely hard to test (non-determinism, no single ground truth, multi-step failure modes) and why traditional unit tests flake.
2. Write **deterministic graders** — exact/contains match, structured-output and JSON-schema validation, golden outputs — and know when each is appropriate.
3. Build a reusable **eval harness**: model an eval as `(example, output) → score`, run a suite over a dataset, and aggregate metrics (pass rate, accuracy).
4. Set up **regression testing**: version an eval dataset, run it across two agent versions, and catch a regression as a gate.
5. Use **LLM-as-judge** responsibly — rubric grading, pairwise comparison — and understand its failure modes (judge bias, the need to evaluate the judge itself).
6. **Trace and observe** an agent with LangSmith: read spans, debug a multi-step run, and see latency, token, and cost data.
7. Run **datasets and experiments** in LangSmith, compare runs, and distinguish offline from online evaluation.
8. Wire a full eval suite (deterministic + LLM-judge + LangSmith experiments) into a **CI-style regression gate** for a realistic agent.

## Non-goals

The following are **explicitly out of scope** for this series:

- Building a self-hosted observability stack (collectors, dashboards infra, alerting pipelines). We use LangSmith's hosted UI.
- A survey of every eval/observability vendor (Langfuse, Phoenix, Braintrust, etc.). We name alternatives once and move on.
- Statistical rigor beyond practical metrics (no significance testing, confidence intervals, or inter-annotator agreement math beyond a sentence of intuition).
- Re-teaching how to *build* agents — we build the smallest toy agent each notebook needs and link back to the framework series rather than re-explaining agent construction.
- Fine-tuning or training judges. The LLM-judge is prompt-engineered, not trained.
- Load/performance testing and SLO engineering as a discipline. We surface latency/cost as observable signals, not as a capacity-planning topic.

## Audience and prerequisites

The learner already understands:

- Python (functions, classes, basic typing; light async only where a traced agent needs it).
- What an LLM, prompt, and completion are; what a tool-using agent loop looks like (as taught in `langgraph/` or `mcp/`).
- Basic testing intuition (what an assertion is, what "a test passes" means).

The learner does **not** need prior knowledge of:

- Any eval framework or methodology.
- LangSmith or any observability/tracing tool.
- The internals of the agents being evaluated — each toy agent is small and self-contained.

## Pedagogical arc — the "test pyramid for agents"

The series climbs from cheap, fast, reliable checks up to expensive, fuzzy, observable ones, then closes by automating the whole thing:

1. **Feel the pain** (01) — a toy agent and a naive unit test that flakes. Why agents resist normal testing.
2. **Deterministic graders** (02) — the cheapest reliable checks: exact match, structured validation, golden outputs.
3. **Generalize into a harness** (03) — an eval is `(example, output) → score`; build a reusable runner and dataset abstraction.
4. **Regression testing** (04) — version a dataset, diff two agent versions, gate on regressions — still fully deterministic, still no key.
5. **LLM-as-judge** (05) — introduce the LLM where deterministic graders can't reach; rubric and pairwise grading, and the judge's own failure modes.
6. **Tracing & observability** (06) — instrument the agent; read spans; debug multi-step runs in LangSmith.
7. **Datasets, experiments & cost** (07) — LangSmith datasets/experiments, offline vs online eval, cost/latency/token tracking.
8. **Production eval gate** (08) — the payoff: a realistic agent with a full offline suite + LLM judge + LangSmith experiments wired as a CI-style regression gate.

This deterministic → fuzzy → observable → automated ramp maps directly onto the no-key (01–04) → key (05–08) boundary.

Every notebook is **runnable end-to-end** in Jupyter. Each opens with a "Why this notebook exists" + "What you'll learn" pair, builds straight through with `### Try it` cells, and closes with a "What you just learned" recap and a "What's missing" teaser motivating the next notebook (omitted in notebook 08).

## Common stack

| Concern | Choice |
|---|---|
| Python | 3.11+ |
| Notebooks | Jupyter |
| Eval graders (01–04) | Pure Python + `pydantic` v2 for structured validation; no framework |
| LLM-as-judge (05–08) | OpenAI model used as a grader |
| Observability/eval platform (06–08) | **LangSmith** (`langsmith` SDK + hosted UI) |
| Agent construction | Minimal `langchain` / `langgraph` toy agents, kept as small as each notebook needs |
| LLM | **OpenAI** by default (`OPENAI_API_KEY`), matching the `langgraph/` and `crewai/` series; Anthropic as a noted fallback |
| Used in notebooks | LLM: **05–08**; LangSmith account: **06–08** |

**No API keys or accounts for notebooks 01–04.** These use deterministic graders against pre-recorded or stubbed agent outputs, so the eval *mechanics* are taught with zero external dependency. Notebooks 05–08 require an `OPENAI_API_KEY` (for the LLM-judge and traced agents) and a free **LangSmith account + `LANGCHAIN_API_KEY`** (06 onward, where the hosted UI is used).

**No local servers.** Unlike the `mcp/`, `a2a/`, and `fastmcp/` series, this series does not stand up HTTP servers, so no port allocation is needed. LangSmith is a hosted service reached over HTTPS.

## LangSmith version

This series targets the **current stable `langsmith` SDK and the hosted LangSmith app** at the time of writing. Where the SDK surface is known to shift (notably the `evaluate()` / `Client.evaluate` API and decorator names like `@traceable`), the notebook calls it out so the learner can adapt for newer versions. Notebooks 06–08 assume the learner has created a free LangSmith account and set `LANGCHAIN_API_KEY` (and `LANGCHAIN_TRACING_V2=true` where tracing is enabled).

## Notebooks

### 01 — `01_why_agents_are_hard_to_test.ipynb` — Feel the pain
**Goal:** establish *why* this series exists before introducing any tooling.

- Build a tiny toy agent (a single LLM-style step, here *stubbed deterministically* so 01 needs no key — a function that simulates an agent's variability).
- Write a naive `assert agent(x) == "expected"` unit test and show it flake when the (simulated) output varies in phrasing, ordering, or formatting.
- Enumerate the three core difficulties: **non-determinism**, **no single ground truth**, **multi-step compounding failures**.
- Closing: *"We can't `assert equals` our way out of this. We need graders that score outputs, not match them exactly."*
- No eval framework, no key.

### 02 — `02_assertions_and_golden_outputs.ipynb` — Deterministic graders
**Goal:** the cheapest, most reliable checks — no LLM required.

- **Exact / contains / regex** matching and when each fits.
- **Structured-output validation** with `pydantic`: does the agent's output parse into the expected schema? Are required fields present and well-typed?
- **Golden outputs**: store known-good outputs to files, diff against them, and the discipline of updating goldens intentionally.
- Frame each grader as a function returning a pass/fail or a score in `[0, 1]`.
- No key — graders run against pre-recorded or stubbed outputs.

### 03 — `03_building_an_eval_harness.ipynb` — Generalize the pattern
**Goal:** turn ad-hoc checks into a reusable evaluation system.

- The core abstraction: an eval is `(example, output) → score`; an **example** is `{input, expected?}`; a **dataset** is a list of examples.
- Build a small `run_eval(agent, dataset, graders) → results` runner that applies graders from notebook 02 across a dataset.
- Aggregate metrics: pass rate, mean score, per-grader breakdown.
- Present results as a readable table; introduce the idea of an eval **report**.
- No key — still deterministic graders over stubbed agent outputs.

### 04 — `04_regression_testing.ipynb` — Catch regressions before they ship
**Goal:** use the harness to compare agent versions and gate on quality.

- Version an eval dataset (treat it as a checked-in artifact).
- Run the harness against **two agent versions** (v1 and a deliberately-regressed v2) and diff the scores.
- Define a **gate**: fail if any example regresses or if aggregate score drops below a threshold — the deterministic precursor to a CI gate (fully realized in 08).
- Discuss flaky-grader hygiene and why determinism matters most here.
- No key — the whole regression loop runs offline.

### 05 — `05_llm_as_judge.ipynb` — When deterministic graders aren't enough
**Goal:** introduce the LLM as a grader, with eyes open. **First notebook needing a key.**

- Motivate: open-ended outputs (summaries, explanations, tone) can't be exact-matched.
- **Rubric grading**: an LLM scores an output against an explicit rubric, returning a structured score + rationale.
- **Pairwise comparison**: ask the LLM which of two outputs is better — often more reliable than absolute scoring.
- The traps: **judge bias** (position, verbosity, self-preference), non-determinism of the judge itself, and the key insight that *the judge needs evaluating too* — validate it against a small human-labeled set.
- Plug the LLM-judge into the notebook-03 harness as just another grader.
- Requires `OPENAI_API_KEY`.

### 06 — `06_tracing_with_langsmith.ipynb` — See what the agent actually did
**Goal:** introduce observability — read a real multi-step run. **First notebook needing LangSmith.**

- Set up LangSmith (`LANGCHAIN_API_KEY`, `LANGCHAIN_TRACING_V2=true`); explain the free-account step.
- Instrument a small multi-step LangGraph/LangChain agent with `@traceable` / automatic tracing.
- Run it and read the resulting **trace** in the LangSmith UI: spans, nested calls, inputs/outputs at each step.
- Debug a deliberately-broken run by reading its trace; observe latency and token counts per span.
- Note alternatives in one line (Langfuse, Arize Phoenix) and move on.
- Requires `OPENAI_API_KEY` + LangSmith account.

### 07 — `07_datasets_experiments_and_cost.ipynb` — Evals at scale, in the platform
**Goal:** move the offline harness into LangSmith and add the cost/latency dimension.

- Upload an eval dataset to LangSmith; run an **experiment** with `evaluate()` using both a deterministic grader and the notebook-05 LLM-judge.
- Compare experiments across agent versions in the UI (the platform-native version of notebook 04's regression diff).
- **Offline vs online eval**: scheduled/CI experiments vs sampling and scoring live production traffic.
- **Cost, latency, and token tracking**: read aggregate cost/latency from experiments; reason about the cost of evals themselves.
- Requires `OPENAI_API_KEY` + LangSmith account.

### 08 — `08_production_eval_gate_end_to_end.ipynb` — The payoff
**Goal:** assemble everything into a production-style quality gate for a realistic agent.

- Build a realistic (but still self-contained) agent — e.g. a small research-and-summarize agent with a tool call and an LLM synthesis step.
- Assemble a **full eval suite**: deterministic graders (02–03) + regression gate (04) + LLM-judge (05) + a LangSmith experiment (07), all wired together.
- Express the suite as a **CI-style gate**: a single runnable cell (and a sketch of the equivalent `pytest` / CI invocation) that passes on a good agent and fails on a regressed one.
- Touch online monitoring: how the same suite informs a sampled production-traffic check.
- Closing cell: a narrative recap of the climb from `assert equals` (01) to a production gate (08), and pointers back to the agents in `langgraph/`, `mcp/`, and `a2a/` that this tooling can now evaluate.
- Requires `OPENAI_API_KEY` + LangSmith account.

## File layout

```
evals/
  01_why_agents_are_hard_to_test.ipynb
  02_assertions_and_golden_outputs.ipynb
  03_building_an_eval_harness.ipynb
  04_regression_testing.ipynb
  05_llm_as_judge.ipynb
  06_tracing_with_langsmith.ipynb
  07_datasets_experiments_and_cost.ipynb
  08_production_eval_gate_end_to_end.ipynb
```

The new `evals/` folder sits at the repository root alongside `mcp/`, `a2a/`, `langgraph/`, and `crewai/`. The repo `README.md` gains an `evals/` section (mirroring the existing series tables) and an updated prerequisites note clarifying that 01–04 need no key, while 05–08 need an `OPENAI_API_KEY` and (06+) a free LangSmith account.

## Style conventions

Each notebook opens with:

- A `## Why this notebook exists` markdown cell (1 paragraph of motivation; for 02–08, a one-line recap of where the previous notebook left off).
- A `## What you'll learn` bulleted list.

Body sections use numbered headers (`## 1. ...`, `## 2. ...`) with `### Try it` cells the learner can run and modify. **"Gotcha:"** callouts are markdown blockquotes inserted only where eval/observability behavior is genuinely surprising (e.g., judge bias, flaky graders) — not in every cell.

Each notebook closes with:

- `## What you just learned` — bulleted recap.
- `## What's missing` — one paragraph motivating the next notebook (omitted in notebook 08, which closes the series).

This matches the existing `mcp/`, `a2a/`, and `fastmcp/` series style.

## Success criteria

The series is complete when:

1. All 8 notebooks exist under `evals/`.
2. Each notebook runs top-to-bottom in a fresh kernel with no errors (05–08 given the required key/account).
3. Notebooks 01–04 require no external API keys or accounts.
4. Notebooks 05–08 run given a single `OPENAI_API_KEY`; 06–08 additionally given a free LangSmith account + `LANGCHAIN_API_KEY`.
5. Every concept claimed in the spec is demonstrated with a runnable cell in the corresponding notebook.
6. Notebook 04's regression gate and notebook 08's CI-style gate both *pass on the good agent and fail on the regressed one* when executed.
7. The repo `README.md` documents the series consistently with the others.

## Open questions deferred to implementation planning

- Exact `langsmith` / `langchain` / `langgraph` version pins (latest stable at time of writing).
- Whether notebook 01's "agent" is a pure-Python stub or a recorded real-LLM transcript replayed deterministically. (Leaning pure-Python stub to guarantee no key.)
- The specific domain of the notebook-08 capstone agent (research-and-summarize vs a structured extraction task). (Leaning research-and-summarize.)
- Whether the `pytest` CI sketch in notebook 08 is a runnable file or an in-notebook illustration. (Leaning in-notebook illustration plus a short prose note.)

These do not affect the design and will be decided in the implementation plan.
