# AI Agents — Learn to Build Intelligent Agent Systems

A hands-on repo for learning **AI agents** — programs that reason, use tools, and collaborate to accomplish complex tasks autonomously. Organized as five progressive learning series, one per topic.

## Repository layout

```
.
├── crewai/      Role-based agent teams (CrewAI)
├── langgraph/   Graph-based agents and advanced multi-agent patterns
├── mcp/         The Model Context Protocol from first principles
├── a2a/         The Agent2Agent (A2A) protocol from first principles
└── evals/       Agent evaluation & observability from first principles
```

Each folder is self-contained: notebooks build progressively from a "no-framework" baseline up to production-grade patterns, and a `docs/` subfolder (where applicable) holds longer-form guides.

## What are AI agents?

An AI agent is an LLM (like Claude or GPT-4) that can **do things**, not just answer questions:

- **Reason** about what steps are needed to complete a task
- **Use tools** — search the web, query databases, run calculations, call APIs
- **Loop** — inspect tool results, decide if more work is needed, keep going
- **Collaborate** — work alongside other specialized agents

This is the difference between asking a chatbot a question and having an assistant that researches a topic, analyzes data, and writes a report — all on its own.

---

## crewai/ — Role-based teams

Higher-level framework. Define agents with roles, give them tasks, let them collaborate. Fast to get results.

| Notebook | What you'll build |
|---|---|
| [`crewai/crewai_agents.ipynb`](crewai/crewai_agents.ipynb) | Sequential and hierarchical crews; agents with custom tools |

Companion guide: [`crewai/docs/crewai_guide.md`](crewai/docs/crewai_guide.md).

---

## langgraph/ — Graph-based agents

Lower-level than CrewAI. You define a state machine; the framework runs it. Better for production systems with complex routing, retries, and human checkpoints.

| Notebook | Pattern | What you'll build |
|---|---|---|
| [`langgraph/pre_langgraph_ai_agents_first_principles.ipynb`](langgraph/pre_langgraph_ai_agents_first_principles.ipynb) | **First principles** | The agent loop hand-built in pure Python before bringing in LangGraph |
| [`langgraph/langgraph_agents.ipynb`](langgraph/langgraph_agents.ipynb) | **Foundations** | State, nodes, edges, tools; a ReAct agent; a Supervisor+Researcher+Writer team |
| [`langgraph/langgraph_multi_agent_supervisor.ipynb`](langgraph/langgraph_multi_agent_supervisor.ipynb) | **Advanced Supervisor** | Customer support center with structured routing, scratchpad, quality loops |
| [`langgraph/langgraph_hierarchical_teams.ipynb`](langgraph/langgraph_hierarchical_teams.ipynb) | **Hierarchical Teams** | Startup due diligence system with sub-teams and subgraph composition |
| [`langgraph/langgraph_collaboration_patterns.ipynb`](langgraph/langgraph_collaboration_patterns.ipynb) | **Collaboration** | Map-Reduce, Debate, and Voting patterns using the `Send()` API |
| [`langgraph/langgraph_custom_state_machines.ipynb`](langgraph/langgraph_custom_state_machines.ipynb) | **State Machines** | Document processing pipeline with retry logic and error recovery |
| [`langgraph/langgraph_human_in_the_loop.ipynb`](langgraph/langgraph_human_in_the_loop.ipynb) | **Human-in-the-Loop** | Content publishing pipeline with approval gates and collaborative editing |

Companion guides:
- [`langgraph/docs/langgraph_guide.md`](langgraph/docs/langgraph_guide.md) — concepts, patterns, and code explained
- [`langgraph/docs/langgraph_advanced_patterns_guide.md`](langgraph/docs/langgraph_advanced_patterns_guide.md) — reference for all five advanced patterns

---

## mcp/ — The Model Context Protocol

A progressive series that teaches **MCP** — the standard for agent-to-tool communication — from the ground up. Each notebook builds on the previous one; the closing notebook ties everything together with a real LLM.

| Notebook | What you'll learn |
|---|---|
| [`mcp/01_without_mcp.ipynb`](mcp/01_without_mcp.ipynb) | The pain of integrating tools without a shared protocol |
| [`mcp/02_mcp_local_stdio.ipynb`](mcp/02_mcp_local_stdio.ipynb) | MCP over local stdio transport |
| [`mcp/03_mcp_http_server.ipynb`](mcp/03_mcp_http_server.ipynb) | MCP over HTTP |
| [`mcp/04_mcp_with_auth.ipynb`](mcp/04_mcp_with_auth.ipynb) | Adding authentication |
| [`mcp/05_mcp_production_patterns.ipynb`](mcp/05_mcp_production_patterns.ipynb) | Production patterns and pitfalls |
| [`mcp/06_mcp_prompt_injection_and_safety.ipynb`](mcp/06_mcp_prompt_injection_and_safety.ipynb) | Prompt injection and tool-use safety |
| [`mcp/07_mcp_resources_and_prompts.ipynb`](mcp/07_mcp_resources_and_prompts.ipynb) | MCP resources and prompt templates |
| [`mcp/08_real_llm_end_to_end.ipynb`](mcp/08_real_llm_end_to_end.ipynb) | The payoff: a real LLM driving MCP tools end-to-end |

---

## a2a/ — The Agent2Agent (A2A) protocol

A progressive series that teaches **A2A** — the standard for agent-to-*agent* communication — from the ground up. Mirrors the MCP series' arc; closes with the official `a2a-sdk`.

| Notebook | What you'll learn |
|---|---|
| [`a2a/01_without_a2a.ipynb`](a2a/01_without_a2a.ipynb) | The pain of hand-rolled REST integration between agents |
| [`a2a/02_a2a_agent_card.ipynb`](a2a/02_a2a_agent_card.ipynb) | `/.well-known/agent-card.json` discovery |
| [`a2a/03_a2a_first_task.ipynb`](a2a/03_a2a_first_task.ipynb) | JSON-RPC 2.0 envelope, `message/send`, and the wire format |
| [`a2a/04_a2a_task_lifecycle.ipynb`](a2a/04_a2a_task_lifecycle.ipynb) | Task states, `tasks/get` polling, `tasks/cancel`, multi-turn `input-required` |
| [`a2a/05_a2a_streaming.ipynb`](a2a/05_a2a_streaming.ipynb) | `message/stream` over Server-Sent Events |
| [`a2a/06_a2a_push_notifications.ipynb`](a2a/06_a2a_push_notifications.ipynb) | `tasks/pushNotificationConfig/set` and webhook delivery |
| [`a2a/07_a2a_auth.ipynb`](a2a/07_a2a_auth.ipynb) | Bearer tokens, OAuth2 client credentials, HMAC-signed webhooks |
| [`a2a/08_a2a_multi_agent.ipynb`](a2a/08_a2a_multi_agent.ipynb) | The payoff: official `a2a-sdk` and multi-agent coordination |

Series design + per-notebook implementation plans live under `docs/superpowers/` (the project-lifecycle docs folder).

---

## evals/ — Agent Evaluation & Observability

A progressive series on how to know whether your agent actually works. It starts from why `assert agent(x) == expected` flakes, builds a deterministic grading harness and an LLM judge, adds a regression gate, then moves into LangSmith for tracing, experiments, and cost — closing with a full CI-style production eval gate.

| Notebook | What you'll learn |
|---|---|
| [`evals/01_why_agents_are_hard_to_test.ipynb`](evals/01_why_agents_are_hard_to_test.ipynb) | Why `assert agent(x) == expected` flakes — non-determinism, multi-step failures, and the need for graded evaluation. |
| [`evals/02_assertions_and_golden_outputs.ipynb`](evals/02_assertions_and_golden_outputs.ipynb) | Deterministic graders: `exact_match`, `make_contains`, `make_schema_grader` — cheap, reliable, no LLM needed. |
| [`evals/03_building_an_eval_harness.ipynb`](evals/03_building_an_eval_harness.ipynb) | The eval harness: `Score`, `Example`, `EvalReport`, and `run_eval(agent, dataset, graders) → EvalReport`. |
| [`evals/04_regression_testing.ipynb`](evals/04_regression_testing.ipynb) | The regression gate: `compare_reports` + `regression_gate` catch quality drops before they ship, entirely offline. |
| [`evals/05_llm_as_judge.ipynb`](evals/05_llm_as_judge.ipynb) | `make_llm_judge` extends the harness to open-ended outputs with explicit rubrics; judge bias and calibration. |
| [`evals/06_tracing_with_langsmith.ipynb`](evals/06_tracing_with_langsmith.ipynb) | LangSmith tracing makes multi-step agent runs visible — read spans, debug failures, observe cost and latency. |
| [`evals/07_datasets_experiments_and_cost.ipynb`](evals/07_datasets_experiments_and_cost.ipynb) | LangSmith experiments: move the offline harness into the platform, compare runs across versions, track cost at scale. |
| [`evals/08_production_eval_gate_end_to_end.ipynb`](evals/08_production_eval_gate_end_to_end.ipynb) | Capstone: full toolkit assembled into a CI-style production gate — mixed graders, regression gate, LangSmith experiment, online monitoring sketch. |

**Keys:** notebooks 01–04 need no API key; 05–08 need `OPENAI_API_KEY`; 06–08 additionally need a free LangSmith account (`LANGCHAIN_API_KEY`). Notebooks that need LangSmith skip gracefully when the key is absent.

Series design + per-notebook implementation plans live under `docs/superpowers/`.

---

## Prerequisites

- **Python 3.11+**
- **Familiarity with LLM APIs** — you should know what a prompt, completion, and model are
- **An API key** — OpenAI for the CrewAI/LangGraph notebooks; either OpenAI or Anthropic for the MCP and A2A series finales (notebooks `08_*`). The MCP and A2A series 01–07 require no API keys. For the `evals/` series: 01–04 need no key, 05–08 need `OPENAI_API_KEY`, and 06–08 additionally need a free LangSmith account (`LANGCHAIN_API_KEY`).

No prior experience with AI agents, LangChain, MCP, or A2A is required — each series starts from "no framework" and builds up.

## Quick start

1. **Clone the repo**
   ```bash
   git clone https://github.com/nhp-atel/AIAgents.git
   cd AIAgents
   ```

2. **Install dependencies for the series you want to work through**
   ```bash
   # CrewAI / LangGraph
   pip install langgraph langchain langchain-openai langchain-community crewai 'crewai[tools]'

   # MCP series
   pip install mcp fastapi 'uvicorn[standard]' httpx

   # A2A series
   pip install fastapi 'uvicorn[standard]' httpx pydantic
   # Notebook 08 only:
   pip install 'a2a-sdk[http-server]==0.3.26'

   # evals series
   pip install langsmith langchain langchain-openai openai pydantic
   ```

3. **Set your API key** (only needed for some notebooks)
   ```bash
   export OPENAI_API_KEY="your-key-here"
   # or
   export ANTHROPIC_API_KEY="your-key-here"
   # evals/ notebooks 06–08 also use LangSmith (free account):
   export LANGCHAIN_API_KEY="your-langsmith-key"
   ```

4. **Open the first notebook of any series and start**
   ```bash
   jupyter notebook langgraph/langgraph_agents.ipynb
   # or
   jupyter notebook mcp/01_without_mcp.ipynb
   # or
   jupyter notebook a2a/01_without_a2a.ipynb
   ```

## Recommended learning paths

**New to agents?** → start with `crewai/crewai_agents.ipynb` for a gentle, high-level intro, then move to `langgraph/langgraph_agents.ipynb` for the lower-level mental model.

**Comfortable with LangChain-style agents, want depth?** → walk the `langgraph/` series 1–7 in order. The advanced patterns are where production systems live.

**Want to connect agents to external tools?** → walk the `mcp/` series 01–08. Builds the protocol from scratch, then ties it together with a real LLM.

**Want agents to talk to other agents?** → walk the `a2a/` series 01–08. Builds the protocol from scratch and closes with the official SDK.

**Want to know if your agent works?** → walk the `evals/` series 01–08. Builds an evaluation harness, LLM judge, and regression gate from scratch, then wires it into LangSmith and a CI-style production gate.

## CrewAI vs LangGraph

| | CrewAI | LangGraph |
|---|---|---|
| **Best for** | Quick prototypes, role-based teams | Production systems, complex logic |
| **Mental model** | A team of people with job titles | A flowchart with decision points |
| **Control level** | High-level, opinionated | Low-level, flexible |
| **Learning curve** | Gentle | Steeper |
| **Setup time** | Minutes | More setup, more power |

## MCP vs A2A

| | MCP | A2A |
|---|---|---|
| **What it standardizes** | Agent → Tool communication | Agent → Agent communication |
| **Typical client** | An LLM calling tools | Another agent (or a coordinator) |
| **Typical server** | A tool exposing operations | An agent exposing skills |
| **Transports** | stdio, HTTP | HTTP (JSON-RPC + SSE + webhooks) |
| **Coexists with the other?** | Yes — an agent can expose A2A and consume MCP simultaneously |  |

## License

Open source. Use for learning.
