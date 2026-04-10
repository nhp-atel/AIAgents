# AI Agents — Learn to Build Intelligent Agent Systems

Welcome! This repo is your hands-on guide to building **AI agents** — programs that can reason, use tools, and collaborate to accomplish complex tasks autonomously.

## What Are AI Agents?

Think of an AI agent as an LLM (like GPT-4) that can **do things**, not just answer questions. Instead of a single prompt-in, text-out interaction, an agent can:

- **Reason** about what steps are needed to complete a task
- **Use tools** — search the web, query databases, run calculations, call APIs
- **Loop** — inspect tool results, decide if more work is needed, and keep going
- **Collaborate** — work alongside other specialized agents in a team

This is the difference between asking ChatGPT a question and having an AI assistant that can actually research a topic, analyze data, and write a report — all on its own.

## What You'll Find in This Repo

### Tutorial Notebooks

#### Foundations
| Notebook | Framework | What You'll Build |
|----------|-----------|-------------------|
| [`crewai_agents.ipynb`](crewai_agents.ipynb) | **CrewAI** | Role-based agent teams that collaborate like a real crew |
| [`langgraph_agents.ipynb`](langgraph_agents.ipynb) | **LangGraph** | Graph-based agents with full control over execution flow |

#### Advanced LangGraph Patterns
| Notebook | Pattern | What You'll Build |
|----------|---------|-------------------|
| [`langgraph_multi_agent_supervisor.ipynb`](langgraph_multi_agent_supervisor.ipynb) | **Advanced Supervisor** | Customer support center with structured routing, scratchpad, and quality loops |
| [`langgraph_hierarchical_teams.ipynb`](langgraph_hierarchical_teams.ipynb) | **Hierarchical Teams** | Startup due diligence system with sub-teams and subgraph composition |
| [`langgraph_collaboration_patterns.ipynb`](langgraph_collaboration_patterns.ipynb) | **Collaboration** | Map-Reduce, Debate, and Voting patterns using the `Send()` API |
| [`langgraph_custom_state_machines.ipynb`](langgraph_custom_state_machines.ipynb) | **State Machines** | Document processing pipeline with retry logic and error recovery |
| [`langgraph_human_in_the_loop.ipynb`](langgraph_human_in_the_loop.ipynb) | **Human-in-the-Loop** | Content publishing pipeline with approval gates and collaborative editing |

All notebooks use real OpenAI API calls and build progressively — no prior agent experience needed for the foundations.

### Companion Guides

Detailed written walkthroughs live in the [`docs/`](docs/) folder:

- [`docs/langgraph_guide.md`](docs/langgraph_guide.md) — LangGraph concepts, patterns, and code explained
- [`docs/crewai_guide.md`](docs/crewai_guide.md) — CrewAI concepts, patterns, and code explained
- [`docs/langgraph_advanced_patterns_guide.md`](docs/langgraph_advanced_patterns_guide.md) — Reference guide for all 5 advanced LangGraph patterns

## Prerequisites

- **Python 3.10+**
- **Familiarity with LLM APIs** — you should know what a prompt, completion, and model are
- **An OpenAI API key** — sign up at [platform.openai.com](https://platform.openai.com) if you don't have one

No prior experience with AI agents, LangChain, or multi-agent systems is required. The notebooks will teach you everything from scratch.

## Quick Start

1. **Clone the repo**
   ```bash
   git clone https://github.com/nhp-atel/AIAgents.git
   cd AIAgents
   ```

2. **Install dependencies** (each notebook handles this in its first cell, or you can install upfront)
   ```bash
   pip install langgraph langchain langchain-openai langchain-community
   pip install crewai 'crewai[tools]'
   ```

3. **Set your OpenAI API key**
   ```bash
   export OPENAI_API_KEY="your-key-here"
   ```

4. **Open a notebook and start learning**
   ```bash
   jupyter notebook langgraph_agents.ipynb
   ```

## Learning Path

If you're brand new to agents, here's the recommended order:

### Stage 1: Foundations
1. **CrewAI** (`crewai_agents.ipynb`) — Higher-level, faster to get results. Understand the *why* behind agents.
2. **LangGraph Basics** (`langgraph_agents.ipynb`) — Lower-level control. Understand *how* agents work: state machines, routing, tool loops.

### Stage 2: Advanced Multi-Agent Patterns (LangGraph)
3. **Advanced Supervisor** (`langgraph_multi_agent_supervisor.ipynb`) — Structured routing with Pydantic, shared scratchpads, quality check loops.
4. **Hierarchical Teams** (`langgraph_hierarchical_teams.ipynb`) — Teams of teams via subgraph composition when one supervisor isn't enough.
5. **Collaboration Patterns** (`langgraph_collaboration_patterns.ipynb`) — Map-Reduce for parallel analysis, Debate for adversarial reasoning, Voting for group decisions.
6. **Custom State Machines** (`langgraph_custom_state_machines.ipynb`) — Explicit lifecycle stages, multi-branch routing, retry/error recovery.
7. **Human-in-the-Loop** (`langgraph_human_in_the_loop.ipynb`) — Approval gates, tool review, and collaborative human-AI editing.

## What Each Notebook Covers

### LangGraph Notebook
| Section | What You'll Learn |
|---------|-------------------|
| Setup | Install packages, configure API key |
| Core Components | State, Nodes, Edges, Tools — the building blocks |
| ReAct Agent | Build a single agent that reasons and uses tools in a loop |
| Memory | Add conversation memory so the agent remembers past messages |
| Multi-Agent Team | Build a Supervisor + Researcher + Writer team |

### CrewAI Notebook
| Section | What You'll Learn |
|---------|-------------------|
| Setup | Install packages, configure API key |
| Core Components | Agents, Tasks, Crews, Processes — the building blocks |
| Single Agent | Build one agent with a tool and run it |
| Sequential Crew | Chain 3 agents: Researcher, Analyst, Writer |
| Hierarchical Crew | Let a manager agent delegate tasks dynamically |
| Custom Tools | Build your own tools with `@tool` and `BaseTool` |

## Which Framework Should I Use?

| | CrewAI | LangGraph |
|---|--------|-----------|
| **Best for** | Quick prototypes, role-based teams | Production systems, complex logic |
| **Mental model** | A team of people with job titles | A flowchart with decision points |
| **Control level** | High-level, opinionated | Low-level, flexible |
| **Learning curve** | Gentle | Steeper |
| **Setup time** | Minutes | More setup, more power |

## License

This project is open source and available for learning purposes.
