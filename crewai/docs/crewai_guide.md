# CrewAI — Building AI Agent Teams with Roles

A companion guide to [`crewai_agents.ipynb`](../crewai_agents.ipynb).

---

## What is CrewAI?

CrewAI is a role-based multi-agent orchestration framework. You define agents as team members with specific roles, goals, and backstories, then organize them into crews that collaborate on tasks.

The mental model: you're assembling a team of specialists, just like you would in a real company.

---

## Core Concepts

### Agent

An autonomous unit with a persona and capabilities.

```python
from crewai import Agent

researcher = Agent(
    role="Senior Researcher",
    goal="Find comprehensive and accurate information",
    backstory="You are an expert researcher with a PhD...",
    tools=[search_tool],
    verbose=True,
)
```

| Parameter | Why It Matters |
|-----------|---------------|
| `role` | Shapes the LLM's behavior — a "Senior Researcher" acts differently than a "Junior Intern" |
| `goal` | Focuses the agent on a specific objective — keeps it on track |
| `backstory` | Gives persona and expertise context — surprisingly impactful on output quality |
| `tools` | What the agent can do beyond text generation |
| `llm` | Which model to use (defaults to GPT-4o) |
| `verbose` | Print reasoning steps for debugging |

### Task

A unit of work assigned to an agent.

```python
from crewai import Task

research_task = Task(
    description="Research the current state of AI agents...",
    expected_output="A structured report with key findings...",
    agent=researcher,
)
```

- `description` — the more specific, the better the output
- `expected_output` — acts as a quality rubric; tells the agent what "done" looks like
- `agent` — who's responsible (optional in hierarchical mode)

### Crew

The team that brings agents and tasks together.

```python
from crewai import Crew, Process

crew = Crew(
    agents=[researcher, analyst, writer],
    tasks=[research_task, analysis_task, writing_task],
    process=Process.sequential,
    verbose=True,
)
result = crew.kickoff()
```

### Process

How the crew executes tasks.

| Process | How It Works | Best For |
|---------|-------------|----------|
| **Sequential** | Tasks run in listed order. Output from task N feeds into task N+1. | Predictable pipelines (research → analyze → write) |
| **Hierarchical** | A manager agent receives all tasks and delegates dynamically. | Flexible workflows, ambiguous task ownership |

---

## Pattern 1: Single Agent + Tool

The simplest setup — one agent, one tool, one task.

```python
crew = Crew(
    agents=[researcher],
    tasks=[research_task],
    process=Process.sequential,
    verbose=True,
)
result = crew.kickoff()
print(result.raw)
```

This is your starting point for any CrewAI project. Get one agent working before adding complexity.

---

## Pattern 2: Sequential Crew

Chain multiple specialists in a pipeline.

```
Researcher → Analyst → Writer
```

Each task's output automatically becomes context for the next task. Define tasks in the order you want them executed.

**Key insight:** The output quality compounds — good research leads to good analysis leads to a great final report. Invest time in clear `expected_output` descriptions.

```python
crew = Crew(
    agents=[researcher, analyst, writer],
    tasks=[research_task, analysis_task, writing_task],
    process=Process.sequential,
)
result = crew.kickoff()

# Inspect individual task outputs
for task_output in result.tasks_output:
    print(task_output.raw)
```

---

## Pattern 3: Hierarchical Crew

Let a manager agent decide who does what.

```python
from langchain_openai import ChatOpenAI

crew = Crew(
    agents=[researcher, analyst, writer],
    tasks=[research_task, analysis_task, writing_task],
    process=Process.hierarchical,
    manager_llm=ChatOpenAI(model="gpt-4o"),
)
```

**Key differences from sequential:**
- Tasks don't need explicit `agent` assignments — the manager decides
- Execution order is flexible and determined at runtime
- More LLM calls (manager reasoning overhead) but more adaptive

### Sequential vs Hierarchical

| Aspect | Sequential | Hierarchical |
|--------|-----------|---------------|
| Predictability | High | Lower |
| Flexibility | Low | High |
| Cost | Lower | Higher (manager LLM calls) |
| Debugging | Easier (linear) | Harder (dynamic) |
| Best for | Well-defined pipelines | Complex, ambiguous workflows |

**Rule of thumb:** Start with sequential. Switch to hierarchical when task routing needs judgment.

---

## Building Custom Tools

### Option 1: `@tool` Decorator (Quick)

```python
from crewai.tools import tool

@tool("Sentiment Analyzer")
def sentiment_analyzer(text: str) -> str:
    """Analyzes sentiment of the given text."""
    # Your logic here
    return "Positive (0.78/1.0)"
```

### Option 2: `BaseTool` Class (Structured)

```python
from crewai.tools import BaseTool
from pydantic import BaseModel, Field

class ScraperInput(BaseModel):
    url: str = Field(description="URL to scrape")

class WebScraperTool(BaseTool):
    name: str = "Web Scraper"
    description: str = "Scrapes content from a URL"
    args_schema: type[BaseModel] = ScraperInput

    def _run(self, url: str) -> str:
        # Your scraping logic here
        return f"Content from {url}..."
```

**When to use which:**
- `@tool` — simple functions, quick prototyping, one-off tools
- `BaseTool` — input validation needed, reusable across projects, complex logic

---

## When to Use CrewAI

CrewAI is the right choice when:

- Your workflow maps to a **team of specialists** with clear roles
- You want to **prototype fast** — idea to working system in minutes
- Your tasks follow **delegation patterns** (research → analyze → write)
- You want **less boilerplate** than graph-based frameworks
- You need built-in **sequential and hierarchical** execution

### CrewAI vs LangGraph

| Dimension | CrewAI | LangGraph |
|-----------|--------|-----------|
| Abstraction | High-level roles | Low-level graphs |
| Mental model | Team of people | Flowchart with decisions |
| Prototyping | Very fast | More setup |
| Control | Limited | Full control |
| Execution | Sequential / Hierarchical | Any graph topology |
| Best for | Role-based teams, content pipelines | Complex state machines, production |

**Start with CrewAI** to validate your idea. **Move to LangGraph** when you need precise control.

---

## Tips for Better Crews

1. **Write specific backstories** — they matter more than you'd expect
2. **Be precise in `expected_output`** — it's the quality rubric
3. **Start with one agent** — get it working, then add more
4. **Use sequential first** — only switch to hierarchical when needed
5. **Mock your tools first** — get the agent flow right before connecting real APIs
