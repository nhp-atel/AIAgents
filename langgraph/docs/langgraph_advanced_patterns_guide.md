# LangGraph Advanced Patterns Guide

A companion reference for the five advanced LangGraph notebooks in this tutorial series.

## Quick Navigation

| Pattern | Notebook | Core Concept |
|---------|----------|-------------|
| [Advanced Supervisor](#1-advanced-supervisor) | `langgraph_multi_agent_supervisor.ipynb` | Structured routing, scratchpad, quality loops |
| [Hierarchical Teams](#2-hierarchical-teams) | `langgraph_hierarchical_teams.ipynb` | Subgraph composition, state mapping |
| [Collaboration Patterns](#3-collaboration-patterns) | `langgraph_collaboration_patterns.ipynb` | Map-Reduce, Debate, Voting with `Send()` |
| [Custom State Machines](#4-custom-state-machines) | `langgraph_custom_state_machines.ipynb` | Enum lifecycles, retry logic, audit trails |
| [Human in the Loop](#5-human-in-the-loop) | `langgraph_human_in_the_loop.ipynb` | Interrupts, approvals, collaborative editing |

---

## Decision Matrix

**"I need X — use pattern Y"**

| I need to... | Use this pattern |
|-------------|-----------------|
| Route work to specialized agents | **Advanced Supervisor** |
| Let agents share context while working | **Advanced Supervisor** (scratchpad) |
| Manage 6+ agents without bottlenecks | **Hierarchical Teams** |
| Get multiple perspectives on the same question | **Map-Reduce** |
| Stress-test a decision with opposing viewpoints | **Debate** |
| Have stakeholders weigh in on a proposal | **Voting** |
| Process items through defined lifecycle stages | **Custom State Machines** |
| Add retry/error recovery to a pipeline | **Custom State Machines** |
| Require human approval at critical points | **Human in the Loop** (Draft Review) |
| Let humans review dangerous actions before execution | **Human in the Loop** (Tool Approval) |
| Co-create content with AI assistance | **Human in the Loop** (Collaborative Editing) |

---

## 1. Advanced Supervisor

### When to Use

- You have 2-5 specialized agents in distinct domains
- Routing decisions benefit from LLM reasoning (not just rules)
- Agents need to share context (scratchpad pattern)
- You want quality assurance on agent outputs

### State Design

```python
class SupervisorState(TypedDict):
    messages: Annotated[list, add_messages]    # Conversation history
    next: str                                   # Next agent to route to
    scratchpad: Annotated[dict, merge_scratchpad]  # Shared context (merge reducer)
    iteration_count: int                        # Loop guard counter
    customer_sentiment: str                     # Metadata for routing
    resolution_status: str                      # Tracks progress
```

### Key APIs

| API | What It Does |
|-----|-------------|
| `llm.with_structured_output(RouteDecision)` | Forces LLM to return a typed Pydantic model |
| `graph.add_conditional_edges(node, func, mapping)` | Multi-way routing from a node |
| Custom reducer (`merge_scratchpad`) | Controls how state fields are merged |

### Patterns

- **Structured routing**: Use Pydantic models with `Literal` fields to constrain routing to valid agent names
- **Scratchpad**: A shared dict where agents store findings; uses a merge reducer so updates are additive
- **Iteration guard**: Cap the number of supervisor loops to prevent infinite cycles
- **Quality check loop**: A reviewer node that can send work back to agents for improvement

### Common Pitfalls

- Forgetting to initialize all state fields when invoking the graph
- Not handling the case where `next` is missing or invalid in routing functions
- Setting iteration limits too low (agents need 2-3 rounds for complex queries)
- Scratchpad growing unbounded — consider summarizing older entries

---

## 2. Hierarchical Teams

### When to Use

- You have 6+ agents spanning distinct domains
- A single supervisor's context window is getting overloaded
- Sub-teams can work somewhat independently
- You want to test sub-teams in isolation

### State Design

```python
# Each sub-team has its own state
class SubTeamState(TypedDict):
    messages: Annotated[list, add_messages]
    next: str
    team_output: str  # Final summary from this team

# Parent orchestrator has a different state
class OrchestratorState(TypedDict):
    messages: Annotated[list, add_messages]
    next: str
    market_research_output: str    # Summary from team A
    financial_analysis_output: str  # Summary from team B
    final_report: str
```

### Key APIs

| API | What It Does |
|-----|-------------|
| `subgraph = graph.compile()` | Compile a subgraph that can be invoked as a function |
| Wrapper functions | Translate between parent and child state schemas |

### Architecture Pattern

```
# 1. Build sub-team as a standalone graph
sub_graph = StateGraph(SubTeamState)
# ... add nodes and edges ...
sub_app = sub_graph.compile()

# 2. Create wrapper that maps parent state → child state → parent state
def run_sub_team(state: OrchestratorState) -> dict:
    child_input = {"messages": state["messages"], "next": "", "team_output": ""}
    child_result = sub_app.invoke(child_input)
    return {"team_output": child_result["team_output"]}

# 3. Use wrapper as a node in the parent graph
parent_graph.add_node("sub_team", run_sub_team)
```

### Common Pitfalls

- Leaking internal sub-team messages to the parent (pass summaries, not transcripts)
- Not testing sub-teams independently before integration
- Going deeper than 2 levels of hierarchy (diminishing returns)
- Forgetting that each sub-team invocation is a full graph execution (cost implications)

---

## 3. Collaboration Patterns

### Map-Reduce

**When to use**: Same analysis needed from multiple perspectives or data sources.

```python
from langgraph.constants import Send

# Dispatcher returns Send objects for fan-out
def dispatcher(state):
    return [
        Send("analyst", {"perspective": "market", "topic": state["topic"]}),
        Send("analyst", {"perspective": "tech", "topic": state["topic"]}),
    ]

# Results collected via operator.add reducer
class State(TypedDict):
    analyses: Annotated[list, operator.add]  # Auto-concatenates parallel results

# Wire it up
graph.add_conditional_edges(START, dispatcher, ["analyst"])
graph.add_edge("analyst", "reducer")
```

### Debate

**When to use**: High-stakes decisions that benefit from adversarial analysis.

- Two agents argue opposing positions (bull vs bear)
- A moderator evaluates each round and decides when to conclude
- `round_number` + `max_rounds` prevents infinite debates
- Final summary synthesizes both sides into a recommendation

### Voting

**When to use**: Multi-stakeholder decisions where each role has different priorities.

```python
class Vote(BaseModel):
    voter_role: str
    vote: Literal["APPROVE", "REJECT", "CONDITIONAL"]
    confidence: float
    reasoning: str

# Each voter returns structured output
vote_llm = llm.with_structured_output(Vote)
```

- Uses `Send()` for parallel voting (same as Map-Reduce)
- Structured output ensures consistent, parseable votes
- Tally node counts votes and applies decision rules

### Common Pitfalls

- `Send()` requires the target node name and a state dict — not the full graph state
- The `operator.add` reducer only works with lists — make sure parallel outputs are wrapped in lists
- Debate agents devolving into repetition — use the moderator to cap rounds
- Voter prompts that are too similar, producing homogeneous votes

---

## 4. Custom State Machines

### When to Use

- Your workflow has well-defined lifecycle stages
- You need explicit error handling and retry logic
- Audit trail / compliance requirements
- Transitions should be deterministic (rule-based, not LLM-decided)

### State Design

```python
class DocStage(str, Enum):
    INTAKE = "intake"
    CLASSIFYING = "classifying"
    PROCESSING = "processing"
    VALIDATING = "validating"
    APPROVED = "approved"
    REJECTED = "rejected"

class DocumentState(TypedDict):
    stage: str                                        # Current lifecycle stage
    document_type: str                                # Drives branching
    extracted_data: dict                              # Structured output
    retry_count: int                                  # Error recovery counter
    max_retries: int                                  # Retry limit
    processing_log: Annotated[list, append_log]       # Audit trail
```

### Key Patterns

- **Enum lifecycle**: Stages are explicit, finite, and documented by the enum itself
- **Multi-branch routing**: 3+ branches from a single node (e.g., invoice/contract/ticket)
- **Validation gate**: Three-way output — pass / fail+retry / fail+reject
- **Retry with limits**: `retry_count < max_retries` → retry; otherwise → reject
- **Audit trail**: `processing_log` with `operator.add`-style reducer accumulates entries

### Common Pitfalls

- Using string literals instead of enum values (defeats the purpose of the enum)
- Forgetting to increment `retry_count` in the error handler
- Retry loops that don't change anything (the re-extraction should be aware of previous errors)
- Processing logs growing too large — consider periodic summarization

---

## 5. Human in the Loop

### When to Use

- Content needs human review before publishing
- Some tool calls have real-world consequences
- Output quality requires domain expertise
- Regulatory or compliance requirements mandate human oversight

### Key APIs

| API | Purpose | Example |
|-----|---------|---------|
| `interrupt(value)` | Pause graph, present data to human | `feedback = interrupt({"draft": content})` |
| `Command(resume=value)` | Resume paused graph with human input | `graph.invoke(Command(resume="approved"), config)` |
| `MemorySaver()` | In-memory state persistence | `graph.compile(checkpointer=MemorySaver())` |
| `graph.get_state(config)` | Inspect paused graph state | `state.next` shows next node |
| `graph.get_state_history(config)` | View all checkpoints | Time-travel debugging |
| `graph.update_state(config, values)` | Directly modify state | Override agent decisions |

### Three Sub-Patterns

**Draft Review Pipeline**:
```
researcher → drafter → [interrupt] → review → approve/revise/reject
```
- Human reviews AI output and provides feedback
- Supports multiple revision rounds
- Use `interrupt()` in the review node

**Tool Execution Approval**:
```
agent → classify tools → safe: auto-execute | dangerous: [interrupt] → approve/deny
```
- Classify tools as safe vs dangerous upfront
- Only interrupt for dangerous operations
- Auto-execute safe tools for smooth workflow

**Collaborative Editing**:
```
ai_draft → [interrupt: human edits] → ai_refine → [interrupt] → ... → finalize
```
- Turn-taking between AI and human
- Each round refines the content
- Human says "done" to finalize

### Common Pitfalls

- Forgetting the checkpointer — interrupts won't work without one
- Not using unique thread IDs — states from different workflows will collide
- Losing context between interrupt rounds — the checkpointer preserves everything
- Not handling the case where the human never responds (add timeouts in production)
- Using `interrupt_before`/`interrupt_after` in `compile()` vs `interrupt()` inside nodes — both work but `interrupt()` is more flexible

---

## State Design Principles

### 1. Use Reducers for Accumulation

```python
# BAD: Each node overwrites the list
findings: list

# GOOD: Each node appends; reducer handles merging
findings: Annotated[list, operator.add]
```

### 2. Separate Concerns in State

```python
# BAD: Everything in messages
class State(TypedDict):
    messages: Annotated[list, add_messages]  # Contains everything

# GOOD: Dedicated fields for structured data
class State(TypedDict):
    messages: Annotated[list, add_messages]  # Conversation
    scratchpad: dict                          # Agent findings
    metadata: dict                            # Routing context
    audit_log: Annotated[list, append_log]    # Compliance trail
```

### 3. Initialize All Fields

Every field in your state TypedDict must be initialized when invoking the graph. Missing fields cause `KeyError` in nodes.

### 4. Use Structured Output for Decisions

```python
# BAD: Parse free text
decision = llm.invoke("Route this to the right agent")
next_agent = parse_somehow(decision.content)  # Fragile

# GOOD: Structured output
class RouteDecision(BaseModel):
    next: Literal["agent_a", "agent_b", "FINISH"]
    reasoning: str

structured_llm = llm.with_structured_output(RouteDecision)
decision = structured_llm.invoke("Route this to the right agent")
next_agent = decision.next  # Type-safe, guaranteed valid
```

---

## Common API Reference

### Graph Construction

```python
from langgraph.graph import StateGraph, START, END

graph = StateGraph(MyState)
graph.add_node("name", function)
graph.add_edge(START, "first_node")
graph.add_edge("node_a", "node_b")
graph.add_conditional_edges("node", routing_func, {"option1": "node1", "option2": "node2"})
app = graph.compile()
```

### Execution

```python
# Simple invocation
result = app.invoke({"field1": "value1", "field2": "value2"})

# With memory (required for HITL)
from langgraph.checkpoint.memory import MemorySaver
memory = MemorySaver()
app = graph.compile(checkpointer=memory)
config = {"configurable": {"thread_id": "unique-id"}}
result = app.invoke(initial_state, config=config)

# Resume after interrupt
from langgraph.types import Command
result = app.invoke(Command(resume="human input"), config=config)
```

### Fan-Out with Send

```python
from langgraph.constants import Send

def dispatcher(state):
    return [Send("target_node", {"key": value}) for value in items]

graph.add_conditional_edges(START, dispatcher, ["target_node"])
```

### Visualization

```python
# Mermaid diagram (renders in Jupyter)
from IPython.display import display, Image
display(Image(app.get_graph().draw_mermaid_png()))

# Text fallback
print(app.get_graph().draw_mermaid())
```

---

## Combining Patterns

Real-world systems often combine multiple patterns:

| Combination | Example |
|------------|---------|
| Supervisor + State Machine | Supervisor manages agents; state machine handles document lifecycle |
| Hierarchical + Map-Reduce | Each sub-team uses map-reduce for parallel analysis |
| Debate + Voting | Agents debate, then a panel votes on the outcome |
| State Machine + HITL | Document pipeline with human approval at the validation gate |
| Supervisor + HITL | Supervisor routes work; human reviews before final output |

### Example: Supervisor + HITL

```python
# Add interrupt to the quality check node
def quality_check_with_human(state):
    auto_assessment = assess_quality(state)
    if auto_assessment.score < 0.8:
        human_feedback = interrupt({"draft": state["response"], "score": auto_assessment.score})
        return handle_human_feedback(human_feedback)
    return {"status": "approved"}
```

---

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|---------|
| `KeyError` on state field | Field not initialized | Initialize all fields in `invoke()` |
| Infinite loop | No iteration guard | Add `iteration_count` + `max_iterations` |
| `interrupt()` not pausing | No checkpointer | Add `checkpointer=MemorySaver()` to `compile()` |
| `Send()` results missing | Wrong reducer | Use `Annotated[list, operator.add]` for collection fields |
| Structured output errors | Pydantic validation | Check `Literal` values match actual options |
| Graph won't compile | Orphan nodes | Ensure every node has at least one incoming and outgoing edge |
| State not persisting | Missing thread_id | Always pass `config={"configurable": {"thread_id": "..."}}` |
