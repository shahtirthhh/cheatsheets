# LangChain — 0 to Master Cheatsheet
*From "what is LangChain" to agents, tools, memory, and LCEL*

---

# Part 1: What Is LangChain?

## The Problem LangChain Solves

Using an LLM directly is simple: send a prompt, get a response. But real applications need more:

```
Simple (no LangChain needed):
  "Translate this to French" → LLM → "Bonjour"

Complex (LangChain helps):
  User asks question → search your database → find relevant docs →
  build a prompt with those docs → call the LLM → parse the response →
  if answer is incomplete, search again → call the LLM again →
  store the conversation in memory → return final answer

  That's 8 steps, 2 LLM calls, 2 database queries, memory management.
  LangChain gives you building blocks to compose this without spaghetti code.
```

## LangChain Is a Toolbox, Not a Framework

LangChain gives you modular components. You pick what you need:

```
Component              What It Does                           Use When
─────────              ────────────                           ────────
Chat Models            Call LLMs (OpenAI, Anthropic, Ollama)  Always
Prompt Templates       Build prompts with variables           Always
Output Parsers         Parse LLM response into structured data  Need JSON/lists
Chains (LCEL)          Compose components into pipelines      Multi-step tasks
Retrievers             Search your documents                  RAG
Tools                  Give the LLM abilities (search, calc)  Agents
Agents                 LLM decides which tools to use         Dynamic workflows
Memory                 Remember past conversation             Chatbots
Callbacks              Log, monitor, stream                   Production
```

## The LangChain Ecosystem

```
langchain-core       — base abstractions (Runnable, prompts, output parsers)
langchain            — chains, agents, retrieval strategies
langchain-openai     — OpenAI models (ChatGPT, embeddings)
langchain-anthropic  — Anthropic models (Claude)
langchain-community  — 600+ integrations (vector stores, loaders, tools)
langchain-ollama     — Ollama (local models)
langchain-huggingface — HuggingFace models
langgraph            — stateful multi-step agent workflows
langserve            — deploy chains as REST APIs
langsmith            — observability and evaluation platform
```

```bash
pip install langchain langchain-openai langchain-community
```

---

# Part 2: Chat Models (Calling LLMs)

## The Basics

```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic
from langchain_ollama import ChatOllama
from langchain_huggingface import ChatHuggingFace

# OpenAI
llm = ChatOpenAI(model="gpt-4o", temperature=0.7, max_tokens=1000)

# Anthropic (Claude)
llm = ChatAnthropic(model="claude-sonnet-4-20250514", temperature=0.7)

# Ollama (local, free)
llm = ChatOllama(model="llama3.1")

# HuggingFace (local or API)
llm = ChatHuggingFace(model_id="meta-llama/Llama-3.1-8B-Instruct")
```

## Message Types

LLMs work with messages, not raw strings. There are 4 message types:

```python
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage, ToolMessage

messages = [
    SystemMessage(content="You are a helpful assistant."),    # system instructions
    HumanMessage(content="What is RAG?"),                     # user's question
    AIMessage(content="RAG stands for..."),                   # LLM's previous response
    HumanMessage(content="Give me an example."),              # follow-up
]

response = llm.invoke(messages)
print(response.content)        # "Here's an example of RAG..."
print(response.usage_metadata)  # {"input_tokens": 45, "output_tokens": 120}
```

## Streaming

```python
# Stream tokens one at a time
for chunk in llm.stream(messages):
    print(chunk.content, end="", flush=True)

# Async streaming
async for chunk in llm.astream(messages):
    print(chunk.content, end="", flush=True)
```

## Model Parameters

```python
llm = ChatOpenAI(
    model="gpt-4o",
    temperature=0.7,      # 0 = deterministic, 1 = creative, 2 = random
    max_tokens=2000,       # max response length
    top_p=0.9,             # nucleus sampling (alternative to temperature)
    frequency_penalty=0,   # penalize repeated tokens
    presence_penalty=0,    # penalize tokens that appeared at all
    seed=42,               # reproducible outputs (OpenAI only)
    timeout=60,            # request timeout in seconds
)

# When to use which temperature:
# 0.0 — factual Q&A, data extraction, classification
# 0.3 — summarization, analysis
# 0.7 — general conversation, explanations (default)
# 1.0 — creative writing, brainstorming
```

---

# Part 3: Prompt Templates

## Why Templates?

You rarely send the same prompt every time. Templates let you define the structure once and fill in variables.

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

# Simple template
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a {role} who speaks {language}."),
    ("human", "{question}"),
])

# Fill in variables
messages = prompt.invoke({
    "role": "helpful assistant",
    "language": "Spanish",
    "question": "What is machine learning?",
})
# Results in:
# SystemMessage: "You are a helpful assistant who speaks Spanish."
# HumanMessage: "What is machine learning?"

# Chain it with the LLM
chain = prompt | llm
response = chain.invoke({"role": "teacher", "language": "English", "question": "Explain RAG"})
```

## Chat History in Templates

```python
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    MessagesPlaceholder(variable_name="history"),    # past conversation goes here
    ("human", "{question}"),
])

response = chain.invoke({
    "history": [
        HumanMessage(content="My name is Alice"),
        AIMessage(content="Hello Alice! How can I help?"),
    ],
    "question": "What's my name?",
})
# LLM knows: "Your name is Alice." (because history is in the prompt)
```

## Few-Shot Prompting

```python
from langchain_core.prompts import FewShotChatMessagePromptTemplate

examples = [
    {"input": "happy", "output": "sad"},
    {"input": "tall", "output": "short"},
    {"input": "hot", "output": "cold"},
]

example_prompt = ChatPromptTemplate.from_messages([
    ("human", "{input}"),
    ("ai", "{output}"),
])

few_shot = FewShotChatMessagePromptTemplate(
    example_prompt=example_prompt,
    examples=examples,
)

final_prompt = ChatPromptTemplate.from_messages([
    ("system", "Give the opposite of each word."),
    few_shot,                                        # injects examples
    ("human", "{input}"),
])

chain = final_prompt | llm
response = chain.invoke({"input": "bright"})
# "dark"
```

---

# Part 4: Output Parsers

## Getting Structured Output from LLMs

LLMs return strings. But you often need JSON, lists, or typed objects.

```python
# Method 1: Pydantic output (structured, validated)
from langchain_core.output_parsers import PydanticOutputParser
from pydantic import BaseModel, Field

class MovieReview(BaseModel):
    title: str = Field(description="The movie title")
    rating: float = Field(ge=0, le=10, description="Rating out of 10")
    summary: str = Field(description="One sentence summary")
    sentiment: str = Field(description="positive, negative, or neutral")

parser = PydanticOutputParser(pydantic_object=MovieReview)

prompt = ChatPromptTemplate.from_messages([
    ("system", "Analyze the movie review and extract structured data.\n{format_instructions}"),
    ("human", "{review}"),
])

chain = prompt | llm | parser

result = chain.invoke({
    "review": "Inception was mind-blowing! The visuals were stunning and the plot kept me guessing. 9/10.",
    "format_instructions": parser.get_format_instructions(),
})
# result = MovieReview(title="Inception", rating=9.0, summary="...", sentiment="positive")
# It's a typed Python object, not a string!

# Method 2: with_structured_output (simpler, OpenAI function calling)
structured_llm = llm.with_structured_output(MovieReview)
result = structured_llm.invoke("Review: Inception was amazing! 9/10")
# Same typed MovieReview object
```

```python
# Method 3: Simple string/JSON parsing
from langchain_core.output_parsers import StrOutputParser, JsonOutputParser

chain = prompt | llm | StrOutputParser()      # returns plain string
chain = prompt | llm | JsonOutputParser()     # returns parsed dict
```

---

# Part 5: LCEL — LangChain Expression Language

## What Is LCEL?

LCEL is the pipe (`|`) syntax for composing LangChain components. Think of it like Unix pipes: `cat file | grep pattern | sort`. Each component receives input from the previous one.

```python
# The pipe: prompt → LLM → parser
chain = prompt | llm | StrOutputParser()

# This is equivalent to:
messages = prompt.invoke({"question": "What is AI?"})
response = llm.invoke(messages)
text = StrOutputParser().invoke(response)
```

## Every Component Is a Runnable

All LangChain components implement the Runnable interface:

```python
runnable.invoke(input)          # single input → single output
runnable.batch([input1, input2]) # multiple inputs → multiple outputs (parallel)
runnable.stream(input)          # single input → stream of outputs
await runnable.ainvoke(input)   # async version
await runnable.abatch([...])    # async batch
async for chunk in runnable.astream(input):  # async stream
```

## Composing Chains

### Sequential Chain (Pipe)

```python
# Each step feeds into the next
chain = step1 | step2 | step3
```

### Parallel Chain (RunnableParallel)

```python
from langchain_core.runnables import RunnableParallel, RunnablePassthrough

# Run multiple things at once, combine results
chain = RunnableParallel(
    summary=summary_chain,
    sentiment=sentiment_chain,
    keywords=keywords_chain,
) | combine_results

# RunnablePassthrough passes input through unchanged
chain = RunnableParallel(
    context=retriever | format_docs,
    question=RunnablePassthrough(),
) | prompt | llm
```

### Branching (RunnableBranch)

```python
from langchain_core.runnables import RunnableBranch

# Route to different chains based on input
chain = RunnableBranch(
    (lambda x: "code" in x["question"].lower(), code_chain),
    (lambda x: "math" in x["question"].lower(), math_chain),
    default_chain,    # fallback
)
```

### RunnableLambda (Custom Functions)

```python
from langchain_core.runnables import RunnableLambda

def custom_transform(data):
    return {"processed": data["text"].upper()}

chain = RunnableLambda(custom_transform) | next_step
```

## Full RAG Chain in LCEL

```python
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

# All pieces
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})
prompt = ChatPromptTemplate.from_template(
    "Answer based on context:\n{context}\n\nQuestion: {question}"
)
llm = ChatOpenAI(model="gpt-4o", temperature=0)

def format_docs(docs):
    return "\n\n".join(d.page_content for d in docs)

# Compose
rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

# Use
answer = rag_chain.invoke("What was Q3 revenue?")

# Stream
for chunk in rag_chain.stream("What was Q3 revenue?"):
    print(chunk, end="")
```

---

# Part 6: Tools and Agents

## What Is a Tool?

A tool gives the LLM the ability to DO things — search the web, run calculations, query a database, call an API. Without tools, an LLM can only generate text. With tools, it can take actions.

```python
from langchain_core.tools import tool

@tool
def search_database(query: str) -> str:
    """Search the company database for information. Use for factual questions about company data."""
    results = db.search(query)
    return str(results)

@tool
def calculate(expression: str) -> str:
    """Calculate a mathematical expression. Use for any math operations."""
    return str(eval(expression))    # use safe eval in production

@tool
def get_weather(city: str) -> str:
    """Get current weather for a city."""
    data = weather_api.get(city)
    return f"{city}: {data['temp']}°C, {data['condition']}"
```

## What Is an Agent?

A chain has a fixed sequence: step1 → step2 → step3. An agent DECIDES what to do at each step. The LLM sees the available tools, decides which one to call (or none), reads the result, and decides what to do next.

```
Chain (fixed):     Retrieve → Generate → Done
Agent (dynamic):   Think → "I need weather data" → Call weather tool
                   → Read result → "Now I can answer" → Generate → Done

                   OR

                   Think → "I need to search AND calculate"
                   → Call search → Read result → Call calculator → Read result
                   → Think → "Now I have everything" → Generate → Done
```

```python
# Create an agent with tools
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(
    model=ChatOpenAI(model="gpt-4o"),
    tools=[search_database, calculate, get_weather],
)

# The agent decides which tools to use
response = agent.invoke({
    "messages": [HumanMessage(content="What's the total revenue if we add Q3 ($4.2M) and Q4 ($5.1M)?")]
})

# Agent's thought process:
# 1. "I need to calculate 4.2 + 5.1"
# 2. Calls calculate("4.2 + 5.1") → "9.3"
# 3. "The total revenue is $9.3M"
```

## Built-in Tools

```python
# Web search
from langchain_community.tools import DuckDuckGoSearchRun
search = DuckDuckGoSearchRun()

# Wikipedia
from langchain_community.tools import WikipediaQueryRun
from langchain_community.utilities import WikipediaAPIWrapper
wiki = WikipediaQueryRun(api_wrapper=WikipediaAPIWrapper())

# Python REPL (execute Python code)
from langchain_experimental.tools import PythonREPLTool
python_repl = PythonREPLTool()

# Requests (call APIs)
from langchain_community.tools import RequestsGetTool
requests_tool = RequestsGetTool()

agent = create_react_agent(
    model=llm,
    tools=[search, wiki, python_repl],
)
```

---

# Part 7: Memory (Conversation History)

## Why Memory Matters

LLMs are stateless — they don't remember previous messages unless you send them again. Memory stores the conversation history and includes it in each new prompt.

```python
# Without memory:
llm.invoke("My name is Alice")     → "Hello Alice!"
llm.invoke("What's my name?")     → "I don't know your name."

# With memory:
# Message history is maintained and sent with each new message
```

## Message History

```python
from langchain_community.chat_message_histories import ChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory

# In-memory store (use Redis/PostgreSQL in production)
store = {}

def get_session_history(session_id: str):
    if session_id not in store:
        store[session_id] = ChatMessageHistory()
    return store[session_id]

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    MessagesPlaceholder(variable_name="history"),
    ("human", "{input}"),
])

chain = prompt | llm

# Wrap with memory
chain_with_memory = RunnableWithMessageHistory(
    chain,
    get_session_history,
    input_messages_key="input",
    history_messages_key="history",
)

# Use with session ID
config = {"configurable": {"session_id": "user_123"}}

response1 = chain_with_memory.invoke(
    {"input": "My name is Alice"},
    config=config,
)
# "Hello Alice!"

response2 = chain_with_memory.invoke(
    {"input": "What's my name?"},
    config=config,
)
# "Your name is Alice!" — it remembers because history is included
```

## Summary Memory (For Long Conversations)

After many messages, the history gets too long. Summarize older messages to save tokens.

```python
from langchain.memory import ConversationSummaryMemory

memory = ConversationSummaryMemory(llm=llm)

# After 20+ messages, instead of storing all 20, it stores:
# "The user introduced themselves as Alice, asked about RAG,
#  discussed vector databases, and is now asking about deployment."
```

---

# Part 8: Callbacks (Logging, Monitoring, Streaming)

```python
from langchain_core.callbacks import BaseCallbackHandler

class MyCallback(BaseCallbackHandler):
    def on_llm_start(self, serialized, prompts, **kwargs):
        print(f"LLM called with {len(prompts)} prompts")

    def on_llm_end(self, response, **kwargs):
        print(f"LLM responded with {len(response.generations)} outputs")
        # Log token usage, cost, latency

    def on_llm_error(self, error, **kwargs):
        print(f"LLM error: {error}")
        # Alert monitoring system

    def on_chain_start(self, serialized, inputs, **kwargs):
        print(f"Chain started: {serialized.get('name', 'unknown')}")

    def on_tool_start(self, serialized, input_str, **kwargs):
        print(f"Tool called: {serialized.get('name', 'unknown')}")

# Use callbacks
llm = ChatOpenAI(callbacks=[MyCallback()])
# Or pass to invoke: chain.invoke(input, config={"callbacks": [MyCallback()]})
```

---

# Part 9: LangGraph (Brief — For Multi-Step Agent Workflows)

LangGraph extends LangChain for stateful, multi-step workflows with branching, loops, and human-in-the-loop.

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
from langgraph.graph import add_messages

# Define state
class State(TypedDict):
    messages: Annotated[list, add_messages]
    next_step: str

# Define nodes (each is a function)
async def classify(state):
    # Classify the user's intent
    response = await llm.ainvoke(state["messages"])
    return {"next_step": "rag" if "question" in response.content.lower() else "chat"}

async def rag_node(state):
    # Run RAG pipeline
    docs = retriever.invoke(state["messages"][-1].content)
    answer = await rag_chain.ainvoke(state["messages"][-1].content)
    return {"messages": [AIMessage(content=answer)]}

async def chat_node(state):
    response = await llm.ainvoke(state["messages"])
    return {"messages": [response]}

# Build graph
graph = StateGraph(State)
graph.add_node("classify", classify)
graph.add_node("rag", rag_node)
graph.add_node("chat", chat_node)

graph.set_entry_point("classify")
graph.add_conditional_edges("classify", lambda s: s["next_step"], {
    "rag": "rag",
    "chat": "chat",
})
graph.add_edge("rag", END)
graph.add_edge("chat", END)

app = graph.compile()
```

---

# Part 10: 🧩 Interview Q&A

**Q: What is LangChain in one sentence?**
A: LangChain is a framework that provides modular building blocks (models, prompts, chains, tools, memory, retrievers) to compose LLM-powered applications without writing glue code.

**Q: What is LCEL?**
A: LangChain Expression Language — the pipe (`|`) syntax for composing Runnables. Every component (prompt, LLM, parser, retriever) is a Runnable with `.invoke()`, `.stream()`, `.batch()`. You chain them with `|` like Unix pipes.

**Q: Chain vs Agent — what's the difference?**
A: A chain has a fixed sequence of steps (always the same pipeline). An agent has an LLM that DECIDES which tools to call and how many times, based on the input. Chains are predictable; agents are flexible.

**Q: When would you NOT use LangChain?**
A: For simple one-shot LLM calls (just use the OpenAI SDK directly). For very custom pipelines where LangChain abstractions add overhead without value. For latency-critical applications where you need full control.

**Q: What's the difference between LangChain and LangGraph?**
A: LangChain is for linear chains and simple agents. LangGraph is for complex stateful workflows with branching, loops, human-in-the-loop interrupts, and persistent state. Think of LangGraph as a state machine engine for AI agents.

**Q: How does LangChain handle different LLM providers?**
A: Every provider (OpenAI, Anthropic, Ollama, HuggingFace) implements the same `BaseChatModel` interface. Your chain code stays the same — just swap `ChatOpenAI` for `ChatAnthropic` and it works.

**Q: What is a Runnable?**
A: The base interface in LangChain. Any component that takes input and produces output. Has methods: `.invoke()` (single), `.batch()` (multiple), `.stream()` (streaming), and async versions of each. Prompts, LLMs, parsers, retrievers — all are Runnables.

**Q: How would you build a production RAG system with LangChain?**
A: Document loaders → text splitter → embedding model → vector store (indexing pipeline). At query time: retriever → prompt template with retrieved context → LLM → output parser. Add RunnableWithMessageHistory for conversation memory, callbacks for logging, and LangSmith for observability.
