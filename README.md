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

| Notebook | Framework | What You'll Build |
|----------|-----------|-------------------|
| [`langgraph_agents.ipynb`](langgraph_agents.ipynb) | **LangGraph** | Graph-based agents with full control over execution flow |
| [`crewai_agents.ipynb`](crewai_agents.ipynb) | **CrewAI** | Role-based agent teams that collaborate like a real crew |

Both notebooks start simple and build up progressively — no prior agent experience needed.

### Companion Guides

Detailed written walkthroughs for each framework live in the [`docs/`](docs/) folder:

- [`docs/langgraph_guide.md`](docs/langgraph_guide.md) — LangGraph concepts, patterns, and code explained
- [`docs/crewai_guide.md`](docs/crewai_guide.md) — CrewAI concepts, patterns, and code explained

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

1. **Start with CrewAI** (`crewai_agents.ipynb`) — it's higher-level and faster to get results. You'll build a working multi-agent team in minutes and understand the *why* behind agents.

2. **Then try LangGraph** (`langgraph_agents.ipynb`) — it gives you lower-level control. You'll understand *how* agents work under the hood: state machines, conditional routing, and tool execution loops.

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
