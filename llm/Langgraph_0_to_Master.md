# LangGraph — Complete Staff-Level Guide

_State Machines for LLM Applications · Agents · Checkpointing · Human-in-the-Loop_

---

# Part 1: Why LangGraph Exists

## The Limitation of Chains

```
LCEL chains are DIRECTED ACYCLIC GRAPHS — data flows forward, never backward.

  retrieve → generate → done

But real agentic workloads need:
  • LOOPS          "retrieval was bad, try a different query"
  • BRANCHES       "this is a SQL question, not a document question"
  • STATE          "remember what we've tried across iterations"
  • INTERRUPTS     "pause here and ask a human to approve"
  • PERSISTENCE    "resume this conversation tomorrow"
  • STREAMING      "show the user each intermediate step"

You cannot express a cycle in LCEL. LangGraph adds cycles, shared state,
and durable execution.
```

## The Mental Model

```
LangGraph = a STATE MACHINE where:
  • STATE   is a typed dict that flows through the whole graph
  • NODES   are functions that read state and return partial state updates
  • EDGES   define which node runs next (fixed or conditional)
  • The graph runs until it reaches END

  ┌─────────┐
  │  START  │
  └────┬────┘
       ▼
  ┌─────────┐      ┌──────────────┐
  │ retrieve│─────▶│ grade_docs   │
  └─────────┘      └──────┬───────┘
       ▲                  │ conditional edge
       │                  ├──── relevant ────▶ ┌──────────┐
       │                  │                    │ generate │
       └── rewrite ◀──── irrelevant            └────┬─────┘
                                                     ▼
                                                  ┌─────┐
                                                  │ END │
                                                  └─────┘
```

---

# Part 2: State — The Foundation

```python
from typing import Annotated, TypedDict, Literal
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langchain_core.messages import BaseMessage
import operator

class AgentState(TypedDict):
    # Annotated[type, reducer] — the reducer controls HOW updates merge.
    messages: Annotated[list[BaseMessage], add_messages]   # APPENDS (never replaces)
    documents: list                                         # REPLACES (default)
    visited: Annotated[list[str], operator.add]             # CONCATENATES
    iteration: int                                          # REPLACES
    query: str
    answer: str | None
```

```
THE REDUCER IS THE MOST IMPORTANT CONCEPT IN LANGGRAPH.

  NO reducer (default):
      node returns {"documents": [d1, d2]}  → state["documents"] = [d1, d2]  (OVERWRITES)

  WITH operator.add:
      state["visited"] = ["a"]
      node returns {"visited": ["b"]}       → state["visited"] = ["a", "b"]  (APPENDS)

  WITH add_messages (special, message-aware):
      • Appends new messages
      • DEDUPLICATES by message ID
      • UPDATES an existing message if the same ID is returned
      This is what makes conversation state correct under retries and parallel branches.

⚠️ CLASSIC BUG: forgetting the reducer on `messages`, so each node
   OVERWRITES the conversation instead of appending to it.
```

```python
# ── Custom reducer ──
def merge_unique_docs(existing: list, new: list) -> list:
    seen = {d.page_content[:200] for d in existing}
    return existing + [d for d in new if d.page_content[:200] not in seen]

class State(TypedDict):
    documents: Annotated[list, merge_unique_docs]

# ── Input/Output schemas (hide internal state from callers) ──
class InputState(TypedDict):
    query: str

class OutputState(TypedDict):
    answer: str

class InternalState(InputState, OutputState):
    documents: list
    iteration: int
    grade: str

graph = StateGraph(InternalState, input=InputState, output=OutputState)
# The caller passes only `query` and receives only `answer`.
```

---

# Part 3: Nodes and Edges

```python
# ── A node: takes state, returns a PARTIAL state update ──
def retrieve(state: AgentState) -> dict:
    docs = retriever.invoke(state["query"])
    return {"documents": docs, "visited": ["retrieve"]}
    # Return ONLY the keys you're changing. LangGraph merges them via reducers.

async def generate(state: AgentState) -> dict:
    context = format_docs(state["documents"])
    response = await llm.ainvoke(prompt.format(context=context, question=state["query"]))
    return {"answer": response.content, "messages": [response]}

# ── Building the graph ──
builder = StateGraph(AgentState)
builder.add_node("retrieve", retrieve)
builder.add_node("generate", generate)

builder.add_edge(START, "retrieve")      # entry point
builder.add_edge("retrieve", "generate") # fixed edge
builder.add_edge("generate", END)

graph = builder.compile()
result = graph.invoke({"query": "What was Q3 revenue?"})
```

## Conditional Edges (Branching and Looping)

```python
def grade_documents(state: AgentState) -> dict:
    """Grade retrieved docs for relevance."""
    relevant = [d for d in state["documents"] if is_relevant(state["query"], d)]
    return {"documents": relevant}

def decide_next(state: AgentState) -> Literal["generate", "rewrite", "give_up"]:
    """Routing function — returns the NAME of the next node."""
    if state["documents"]:
        return "generate"
    if state["iteration"] >= 3:
        return "give_up"              # ALWAYS have a termination path
    return "rewrite"

builder.add_conditional_edges(
    "grade",                          # source node
    decide_next,                      # routing function
    {                                 # map return value → node name
        "generate": "generate",
        "rewrite": "rewrite_query",
        "give_up": END,
    },
)

builder.add_edge("rewrite_query", "retrieve")    # ← THE LOOP
```

```
⚠️ EVERY CYCLE NEEDS A TERMINATION CONDITION.
   Without the iteration counter above, a query that never retrieves
   relevant documents loops until you hit the recursion limit — burning
   money on every iteration.

   Belt and braces: also set a hard cap at invoke time.
       graph.invoke(input, config={"recursion_limit": 25})
```

## Parallel Execution (Fan-Out / Fan-In)

```python
# Multiple edges from one node run those nodes IN PARALLEL
builder.add_edge("start_research", "search_web")
builder.add_edge("start_research", "search_docs")
builder.add_edge("start_research", "search_sql")

# All three converge — this node waits for ALL of them
builder.add_edge("search_web", "synthesize")
builder.add_edge("search_docs", "synthesize")
builder.add_edge("search_sql", "synthesize")

# ⚠️ Parallel nodes writing the SAME state key MUST have a reducer,
#    or you get a concurrent-update error. This is where operator.add earns its keep.
```

## Send API (Dynamic Fan-Out / Map-Reduce)

```python
from langgraph.types import Send

def fan_out_subqueries(state: AgentState) -> list[Send]:
    """Spawn one parallel node execution per sub-query — count known only at runtime."""
    return [
        Send("research_one", {"subquery": q, "parent_query": state["query"]})
        for q in state["subqueries"]
    ]

builder.add_conditional_edges("plan", fan_out_subqueries, ["research_one"])
builder.add_edge("research_one", "aggregate")

# Each `research_one` invocation gets its OWN state slice.
# Results merge back into the parent state via reducers. This is map-reduce for agents.
```

---

# Part 4: Persistence and Checkpointing

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.checkpoint.sqlite import SqliteSaver

# Development
checkpointer = MemorySaver()

# Production
DB_URI = "postgresql://user:pass@localhost:5432/langgraph"
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup()                      # creates the required tables
    graph = builder.compile(checkpointer=checkpointer)

# Every invocation needs a thread_id — this is the conversation identity
config = {"configurable": {"thread_id": "user-123-session-456"}}

graph.invoke({"query": "What is RAG?"}, config)
graph.invoke({"query": "How do I implement it?"}, config)
# The second call sees the full state from the first — durable conversation memory.
```

```
WHAT CHECKPOINTING GIVES YOU:
  • Conversation memory across requests, processes, and restarts
  • Resume after a crash from the last completed node (no re-running expensive steps)
  • Time travel — inspect or rewind to any prior state
  • Human-in-the-loop — pause indefinitely, resume when a human responds
  • Full audit trail of every state transition
```

```python
# ── Inspecting and manipulating state ──
snapshot = graph.get_state(config)
snapshot.values          # current state dict
snapshot.next            # which node(s) run next (empty tuple = finished)
snapshot.config          # includes checkpoint_id

history = list(graph.get_state_history(config))   # newest first

# ── Time travel: rewind and re-run from an earlier checkpoint ──
past = history[3]
graph.invoke(None, past.config)     # replays from that exact checkpoint

# ── Manually edit state (fork the conversation) ──
graph.update_state(config, {"documents": corrected_docs}, as_node="retrieve")
graph.invoke(None, config)          # continue with the corrected state
```

---

# Part 5: Human-in-the-Loop

```python
from langgraph.types import interrupt, Command

def request_approval(state: AgentState) -> dict:
    """Pause execution and surface a decision to a human."""
    decision = interrupt({
        "action": "send_email",
        "recipient": state["recipient"],
        "draft": state["draft"],
        "question": "Approve sending this email?",
    })
    # Execution STOPS here. The value passed to Command(resume=...) becomes `decision`.
    if decision["approved"]:
        return {"status": "approved", "final_draft": decision.get("edited", state["draft"])}
    return {"status": "rejected", "reason": decision.get("reason")}

builder.add_node("approval", request_approval)
graph = builder.compile(checkpointer=checkpointer)

# ── Run until the interrupt ──
result = graph.invoke({"draft": "..."}, config)
state = graph.get_state(config)
print(state.next)          # ('approval',) — paused here
print(state.tasks[0].interrupts[0].value)   # the payload shown to the human

# ── Hours or days later, resume with the human's decision ──
graph.invoke(
    Command(resume={"approved": True, "edited": "Revised email text"}),
    config,
)
```

```python
# ── Static interrupts (pause before/after specific nodes) ──
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["execute_payment", "delete_records"],
    interrupt_after=["generate_plan"],
)

# ── Command: update state AND route in one return ──
def router_node(state) -> Command[Literal["a", "b"]]:
    return Command(
        update={"decision": "path_a"},
        goto="a",
    )
```

```
WHEN HUMAN-IN-THE-LOOP IS MANDATORY:
  • Irreversible actions: payments, deletions, sending external communications
  • Low-confidence outputs where the cost of being wrong is high
  • Regulated domains requiring an audit trail with human sign-off
  • Agent-generated code or SQL before execution
```

---

# Part 6: Streaming

```python
# ── stream_mode="values": full state after each node ──
for state in graph.stream(inputs, config, stream_mode="values"):
    print(state["messages"][-1])

# ── stream_mode="updates": only what each node CHANGED (lighter) ──
for update in graph.stream(inputs, config, stream_mode="updates"):
    for node_name, changes in update.items():
        print(f"{node_name} → {changes}")

# ── stream_mode="messages": token-by-token LLM output ──
for token, metadata in graph.stream(inputs, config, stream_mode="messages"):
    if metadata["langgraph_node"] == "generate":
        print(token.content, end="", flush=True)

# ── stream_mode="custom": emit your own progress events from inside nodes ──
from langgraph.config import get_stream_writer

def long_node(state):
    writer = get_stream_writer()
    writer({"progress": "Searching 12 sources..."})
    results = search()
    writer({"progress": f"Found {len(results)} results, synthesizing..."})
    return {"documents": results}

# ── Multiple modes at once ──
for mode, chunk in graph.stream(inputs, config, stream_mode=["updates", "messages"]):
    ...
```

```python
# ── Production SSE endpoint with step visibility ──
@app.post("/agent/stream")
async def agent_stream(req: AgentRequest):
    config = {"configurable": {"thread_id": req.thread_id}, "recursion_limit": 25}

    async def events():
        try:
            async for mode, chunk in graph.astream(
                {"query": req.query}, config, stream_mode=["updates", "messages"]
            ):
                if mode == "updates":
                    for node, _ in chunk.items():
                        yield f"data: {json.dumps({'type':'step','node':node})}\n\n"
                elif mode == "messages":
                    token, meta = chunk
                    if token.content and meta.get("langgraph_node") == "generate":
                        yield f"data: {json.dumps({'type':'token','content':token.content})}\n\n"

            state = await graph.aget_state(config)
            if state.next:      # graph paused on an interrupt
                yield f"data: {json.dumps({'type':'interrupt','payload':state.tasks[0].interrupts[0].value})}\n\n"
            else:
                yield f"data: {json.dumps({'type':'done'})}\n\n"
        except Exception as e:
            yield f"data: {json.dumps({'type':'error','message':str(e)})}\n\n"

    return StreamingResponse(events(), media_type="text/event-stream",
                             headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"})
```

---

# Part 7: Production Patterns

## Pattern 1 — Corrective RAG (Self-Correcting Retrieval)

```python
class CRAGState(TypedDict):
    question: str
    documents: list
    generation: str
    iteration: int
    web_search_needed: bool

def retrieve(state):
    return {"documents": retriever.invoke(state["question"])}

def grade_documents(state):
    """Keep only relevant docs; flag if we need to fall back to web search."""
    kept = []
    for d in state["documents"]:
        verdict = grader.invoke({"question": state["question"], "document": d.page_content})
        if verdict.binary_score == "yes":
            kept.append(d)
    return {"documents": kept, "web_search_needed": len(kept) == 0}

def rewrite_query(state):
    better = rewriter.invoke({"question": state["question"]})
    return {"question": better.content, "iteration": state["iteration"] + 1}

def web_search(state):
    results = tavily.invoke(state["question"])
    return {"documents": state["documents"] + results}

def generate(state):
    return {"generation": rag_chain.invoke({
        "context": format_docs(state["documents"]), "question": state["question"]
    })}

def route_after_grading(state) -> Literal["generate", "rewrite", "web_search"]:
    if state["documents"]:
        return "generate"
    if state["iteration"] < 2:
        return "rewrite"
    return "web_search"          # exhausted rewrites — go external

builder = StateGraph(CRAGState)
for name, fn in [("retrieve", retrieve), ("grade", grade_documents),
                 ("rewrite", rewrite_query), ("web_search", web_search),
                 ("generate", generate)]:
    builder.add_node(name, fn)

builder.add_edge(START, "retrieve")
builder.add_edge("retrieve", "grade")
builder.add_conditional_edges("grade", route_after_grading,
    {"generate": "generate", "rewrite": "rewrite", "web_search": "web_search"})
builder.add_edge("rewrite", "retrieve")       # loop back
builder.add_edge("web_search", "generate")
builder.add_edge("generate", END)

crag = builder.compile(checkpointer=checkpointer)
```

## Pattern 2 — ReAct Agent with Tools

```python
from langgraph.prebuilt import create_react_agent, ToolNode
from langgraph.graph import MessagesState

# ── Prebuilt (fastest path to a working agent) ──
agent = create_react_agent(
    model=ChatOpenAI(model="gpt-4o", temperature=0),
    tools=[search_db, get_weather, calculate],
    prompt="You are a support agent. Use tools; never guess.",
    checkpointer=checkpointer,
)

# ── Custom (when you need control over the loop) ──
tools = [search_db, get_weather, calculate]
llm_with_tools = llm.bind_tools(tools)

def call_model(state: MessagesState):
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

def should_continue(state: MessagesState) -> Literal["tools", END]:
    last = state["messages"][-1]
    return "tools" if last.tool_calls else END

builder = StateGraph(MessagesState)
builder.add_node("agent", call_model)
builder.add_node("tools", ToolNode(tools))
builder.add_edge(START, "agent")
builder.add_conditional_edges("agent", should_continue, {"tools": "tools", END: END})
builder.add_edge("tools", "agent")            # ← the ReAct loop
agent = builder.compile(checkpointer=checkpointer)
```

## Pattern 3 — Supervisor (Multi-Agent Orchestration)

```python
class SupervisorState(TypedDict):
    messages: Annotated[list, add_messages]
    next_agent: str
    task_complete: bool

class Route(BaseModel):
    next: Literal["researcher", "coder", "writer", "FINISH"]
    reason: str

def supervisor(state: SupervisorState) -> dict:
    decision = llm.with_structured_output(Route).invoke([
        SystemMessage("""You manage three specialists:
        - researcher: gathers information from documents and the web
        - coder: writes and executes code
        - writer: produces the final polished response
        Choose who acts next, or FINISH when the task is complete."""),
        *state["messages"],
    ])
    return {"next_agent": decision.next, "task_complete": decision.next == "FINISH"}

def route_to_agent(state) -> str:
    return END if state["task_complete"] else state["next_agent"]

builder = StateGraph(SupervisorState)
builder.add_node("supervisor", supervisor)
builder.add_node("researcher", researcher_agent)
builder.add_node("coder", coder_agent)
builder.add_node("writer", writer_agent)

builder.add_edge(START, "supervisor")
builder.add_conditional_edges("supervisor", route_to_agent,
    {"researcher": "researcher", "coder": "coder", "writer": "writer", END: END})
for worker in ["researcher", "coder", "writer"]:
    builder.add_edge(worker, "supervisor")     # always report back
```

## Pattern 4 — Subgraphs (Composition)

```python
# Build a reusable RAG subgraph
rag_builder = StateGraph(RAGState)
rag_builder.add_node("retrieve", retrieve)
rag_builder.add_node("generate", generate)
rag_builder.add_edge(START, "retrieve")
rag_builder.add_edge("retrieve", "generate")
rag_builder.add_edge("generate", END)
rag_subgraph = rag_builder.compile()

# Embed it inside a parent graph as a single node
parent = StateGraph(ParentState)
parent.add_node("classify", classify_intent)
parent.add_node("rag", rag_subgraph)          # a compiled graph IS a valid node
parent.add_node("sql", sql_subgraph)
parent.add_conditional_edges("classify", route, {"rag": "rag", "sql": "sql"})

# Shared state keys flow automatically. If schemas differ, wrap with a transform function.
```

---

# Part 8: Error Handling and Reliability

```python
# ── Node-level retry policy ──
from langgraph.pregel import RetryPolicy

builder.add_node(
    "call_api",
    unreliable_node,
    retry=RetryPolicy(max_attempts=3, initial_interval=0.5, backoff_factor=2.0),
)

# ── Explicit error state instead of crashing the graph ──
class State(TypedDict):
    errors: Annotated[list[str], operator.add]
    status: str

def safe_node(state):
    try:
        return {"result": risky_operation(state), "status": "ok"}
    except ExternalAPIError as e:
        logger.error("node_failed", node="safe_node", error=str(e))
        return {"errors": [f"safe_node: {e}"], "status": "degraded"}

def route_on_error(state) -> Literal["continue", "fallback", "fail"]:
    if state["status"] == "ok":
        return "continue"
    return "fallback" if len(state["errors"]) < 3 else "fail"

# ── Guard against runaway loops ──
graph.invoke(inputs, config={"recursion_limit": 25})
# Raises GraphRecursionError instead of looping forever.
# Combine with an explicit iteration counter in state for graceful degradation.
```

---

# Part 9: Testing and Debugging

```python
# ── Test nodes in isolation (they're just functions) ──
def test_grade_documents_filters_irrelevant():
    state = {"question": "What is RAG?", "documents": [relevant_doc, irrelevant_doc]}
    result = grade_documents(state)
    assert len(result["documents"]) == 1
    assert result["web_search_needed"] is False

# ── Test routing logic ──
def test_routes_to_web_search_after_two_rewrites():
    assert route_after_grading({"documents": [], "iteration": 2}) == "web_search"
    assert route_after_grading({"documents": [], "iteration": 0}) == "rewrite"
    assert route_after_grading({"documents": [doc], "iteration": 0}) == "generate"

# ── Assert on the execution PATH, not the generated prose ──
def test_crag_loops_on_bad_retrieval():
    path = [node for update in graph.stream(inputs, config, stream_mode="updates")
                 for node in update.keys()]
    assert path == ["retrieve", "grade", "rewrite", "retrieve", "grade", "generate"]

# ── Visualize the graph ──
print(graph.get_graph().draw_mermaid())
graph.get_graph().draw_mermaid_png(output_file_path="graph.png")

# ── Inspect state at each step during debugging ──
for state in graph.stream(inputs, config, stream_mode="values"):
    print(f"iteration={state.get('iteration')} docs={len(state.get('documents', []))}")
```

---

# Part 10: 🧩 Interview Q&A

**Q: When would you use LangGraph instead of an LCEL chain?**
A: LCEL chains are acyclic — data flows forward and the sequence is fixed at design time. That covers most RAG and is cheaper, faster, and easier to test, so it should be the default. LangGraph is warranted when you need cycles (retry retrieval with a rewritten query), conditional branching where the path depends on intermediate results, shared mutable state across steps, human-in-the-loop pauses, or durable execution that survives a process restart. If I can express the workflow as a fixed pipeline, I use LCEL; if the control flow genuinely depends on runtime results, I reach for LangGraph.

**Q: Explain state reducers and why they matter.**
A: Each key in the state schema can carry an annotated reducer that defines how a node's returned value merges with the existing value. Without one, the default is replacement — a node returning `{"messages": [new_msg]}` wipes the entire conversation. With `operator.add` the lists concatenate; with `add_messages` you get append plus deduplication by message ID plus in-place update when the same ID is returned, which is what makes conversation state correct under retries and parallel branches. Reducers also become mandatory when parallel nodes write the same key — without one, LangGraph raises a concurrent-update error rather than silently losing data.

**Q: How does checkpointing work and what does it enable?**
A: A checkpointer persists the full graph state after every node execution, keyed by a `thread_id`. This turns four hard problems into configuration: conversation memory works across requests and process restarts because the next invocation loads the prior state; crash recovery resumes from the last completed node instead of re-running expensive LLM calls; time travel lets you fetch the state history and re-invoke from any past checkpoint, which is invaluable for debugging non-deterministic agents; and human-in-the-loop becomes possible because the graph can pause indefinitely and resume days later when a human responds. In production I use the Postgres checkpointer, since MemorySaver loses everything on restart.

**Q: How do you prevent an agent from looping forever?**
A: Three layers. First, an explicit iteration counter in the state with the routing function checking it — after N attempts, route to a graceful fallback such as web search or an honest "I couldn't find this" response rather than looping. Second, `recursion_limit` at invoke time as a hard backstop that raises rather than burning budget silently. Third, make the loop condition genuinely progressive — if the rewrite step produces the same query every time, the loop is pointless, so I track attempted queries in state and force a different strategy when they repeat. Cost monitoring per thread catches whatever slips through.

**Q: How do you implement human-in-the-loop approval?**
A: Call `interrupt()` inside a node with a payload describing the pending decision. Execution halts and the state is checkpointed, so the pause can last indefinitely across process restarts. The API surfaces the interrupt payload to the frontend by reading `graph.get_state(config).tasks[0].interrupts[0].value`. When the human responds — possibly days later — the application calls `graph.invoke(Command(resume=decision), config)`, and the value passed to `resume` becomes the return value of the original `interrupt()` call, letting the node continue from exactly where it stopped. This requires a checkpointer; without persistence the paused state would be lost.

**Q: How would you architect a multi-agent system in LangGraph?**
A: The supervisor pattern is the most maintainable starting point. A supervisor node uses structured output to choose which specialist runs next, each specialist is a node or subgraph with its own tools, and every specialist edges back to the supervisor so control is always re-centralized. This keeps routing logic in one testable place rather than scattered across peer-to-peer handoffs. For parallelizable work I use the Send API to fan out dynamically — one node execution per subtask, with results merging back through reducers. Subgraphs let me develop and test each specialist independently, then compose them, since a compiled graph is itself a valid node.

**Q: How do you test a LangGraph application?**
A: Nodes are plain functions, so they unit test directly with a state dict in and a partial update out — no LLM required if you inject the model. Routing functions are pure and test exhaustively across every branch, which is where most logic bugs live. For the graph as a whole, I assert on the execution _path_ by collecting node names from `stream_mode="updates"` — asserting that a bad-retrieval input produces retrieve → grade → rewrite → retrieve is stable and meaningful, whereas asserting on generated prose is not. Output quality goes through a golden dataset scored by an LLM judge or RAGAS, gated in CI so a prompt change that regresses faithfulness fails the build.
