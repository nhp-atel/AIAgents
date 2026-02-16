# LangGraph — Building AI Agents as Graphs

A companion guide to [`langgraph_agents.ipynb`](../langgraph_agents.ipynb).

---

## What is LangGraph?

LangGraph is a framework for building stateful, multi-step AI agent applications as **graphs**. It's part of the LangChain ecosystem but purpose-built for agent orchestration where you need fine-grained control over execution flow.

Unlike simple chain-based approaches, LangGraph lets you define **cyclic graphs** — meaning agents can loop, retry, and route dynamically based on LLM decisions.

---

## Core Concepts

### State

The central data structure flowing through the graph. Defined as a Python `TypedDict`.

```python
from typing import Annotated, TypedDict
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    messages: Annotated[list, add_messages]  # appends, doesn't overwrite
    counter: int                              # plain overwrite
```

**Reducers** control how state updates are merged. The `add_messages` reducer appends new messages to the list instead of replacing it — essential for building up conversation history.

### Nodes

Python functions that do work. Each node receives the current state and returns a **partial update** (only the keys that changed).

```python
def my_node(state: AgentState) -> dict:
    response = llm.invoke(state["messages"])
    return {"messages": [response]}
```

### Edges

Connections between nodes that define execution flow.

| Type | Method | Use Case |
|------|--------|----------|
| Normal | `graph.add_edge("a", "b")` | Always go A → B |
| Conditional | `graph.add_conditional_edges("a", fn)` | Route based on state |
| Entry | `graph.add_edge(START, "a")` | Where execution begins |
| Terminal | `graph.add_edge("a", END)` | Where execution ends |

**Conditional edges** are the backbone of agent loops — they let the LLM decide whether to call a tool, route to another agent, or finish.

### Tools

Functions the LLM can call, defined with the `@tool` decorator and bound to the model:

```python
from langchain_core.tools import tool

@tool
def get_weather(city: str) -> str:
    """Get weather for a city."""
    return "72°F, sunny"

model = ChatOpenAI(model="gpt-4o-mini")
model_with_tools = model.bind_tools([get_weather])
```

When the LLM wants to use a tool, it emits `tool_calls` in its response. A `ToolNode` (from `langgraph.prebuilt`) automatically executes those calls and returns results.

---

## Pattern 1: ReAct Agent (Single Agent + Tools)

The most common agent architecture. The agent reasons, optionally calls tools, and loops until done.

```
START → agent → tool_calls? → yes → tools → agent (loop)
                             → no  → END
```

**How it works:**
1. The agent node calls the LLM with tools bound
2. A conditional edge checks if the response contains `tool_calls`
3. If yes → route to the tool node, which executes the calls, then back to agent
4. If no → the agent is done, route to END

**Key code:**
```python
graph = StateGraph(AgentState)
graph.add_node("agent", agent_node)
graph.add_node("tools", ToolNode(tools))
graph.add_edge(START, "agent")
graph.add_conditional_edges("agent", should_continue)
graph.add_edge("tools", "agent")
agent = graph.compile()
```

---

## Pattern 2: Memory with Checkpointers

By default, each invocation is stateless. Add a checkpointer to persist state across turns.

```python
from langgraph.checkpoint.memory import MemorySaver

agent = graph.compile(checkpointer=MemorySaver())

# Same thread_id = continued conversation
config = {"configurable": {"thread_id": "1"}}
agent.invoke({"messages": [HumanMessage("Hi")]}, config=config)
agent.invoke({"messages": [HumanMessage("What did I just say?")]}, config=config)
```

- **Short-term memory**: Within a thread — the agent remembers the full conversation
- **Long-term memory**: Across threads — requires external storage (database, vector store)

---

## Pattern 3: Multi-Agent Supervisor

A supervisor LLM routes work between specialized agents.

```
START → Supervisor → "researcher" → Researcher → Supervisor
                   → "writer"     → Writer     → Supervisor
                   → "FINISH"     → END
```

**How it works:**
1. The supervisor reads the conversation and decides which worker to engage next
2. Workers do their specialized tasks and report back to the supervisor
3. The supervisor routes again until all work is done, then returns FINISH
4. Conditional edges from the supervisor handle the routing

**State includes a routing field:**
```python
class TeamState(TypedDict):
    messages: Annotated[list, add_messages]
    next: str  # "researcher", "writer", or "FINISH"
```

---

## When to Use LangGraph

LangGraph is the right choice when you need:

- Fine-grained control over execution flow
- Complex state management beyond simple message passing
- Cyclic graphs — agents that loop, retry, and self-correct
- Custom multi-agent coordination patterns
- Built-in persistence and streaming
- Production-grade reliability

### LangGraph vs CrewAI

| Dimension | LangGraph | CrewAI |
|-----------|-----------|--------|
| Abstraction | Low-level graphs | High-level roles |
| Control | Full control over every edge | More opinionated |
| Prototyping speed | Slower setup | Very fast |
| Flexibility | Any graph topology | Sequential / Hierarchical |
| Best for | Production, complex workflows | Rapid prototyping, role-based teams |

---

## Next Steps

- **Human-in-the-loop**: Use `interrupt_before`/`interrupt_after` to pause for user approval
- **Streaming**: Use `.stream()` and `.astream_events()` for real-time output
- **Deployment**: LangGraph Platform (Cloud) to deploy agents as APIs
- **Persistent storage**: SQLite or PostgreSQL checkpointers for production memory
