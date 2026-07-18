# RAG — 0 to Master Cheatsheet
*From "what is RAG" to production pipelines and evaluation*

---

# Part 1: What Is RAG and Why Does It Exist?

## The Problem RAG Solves

Large Language Models (like GPT-4, Claude, LLaMA) are trained on massive internet text. They know a lot — but they have three fatal flaws:

```
Flaw 1: KNOWLEDGE CUTOFF
  "What was our Q3 2024 revenue?"
  LLM: "I don't have access to your company data."

Flaw 2: HALLUCINATION
  "What does section 4.2 of our employee handbook say?"
  LLM: "Section 4.2 states that employees receive 15 vacation days..."
  Reality: Section 4.2 is about parking policy. The LLM made it up confidently.

Flaw 3: NO PRIVATE DATA
  "Summarize the contract we signed with Acme Corp"
  LLM: "I have no access to your contracts."
```

## What RAG Does

**RAG = Retrieval-Augmented Generation.** Instead of asking the LLM to answer from memory, you FIND the relevant documents first, then GIVE them to the LLM as context.

```
WITHOUT RAG:
  User → LLM → Answer (from training data — might be wrong or outdated)

WITH RAG:
  User → Search your documents → Find relevant chunks
       → Stuff them into the prompt → LLM reads them → Answer (grounded in YOUR data)
```

**Real-life analogy:** Imagine an open-book exam. Without RAG, the student answers from memory (might be wrong). With RAG, the student can search through the textbook first, find the relevant pages, then answer based on what they read. Much more accurate.

---

# Part 2: The Complete RAG Pipeline

```
┌────────────────────────────────────────────────────────┐
│                INDEXING (offline, once)                 │
│                                                        │
│  Documents → Parse → Chunk → Embed → Store in Vector DB│
│  (PDF,DOCX)  (text)  (pieces) (numbers)  (searchable) │
└────────────────────────────────────────────────────────┘
                          ↕
┌────────────────────────────────────────────────────────┐
│                QUERYING (real-time, per question)       │
│                                                        │
│  Question → Embed → Search Vector DB → Get top chunks  │
│          → Build prompt with chunks → LLM → Answer     │
└────────────────────────────────────────────────────────┘
```

Every RAG system has these 6 stages. Let's go through each one.

---

# Part 3: Stage 1 — Document Loading

## What Happens Here

You take raw files (PDFs, Word docs, web pages, CSVs) and extract their text content.

```python
# LangChain document loaders
from langchain_community.document_loaders import (
    PyPDFLoader,           # PDF files
    Docx2txtLoader,        # Word documents
    TextLoader,            # Plain text
    CSVLoader,             # CSV files
    WebBaseLoader,         # Web pages
    UnstructuredLoader,    # Auto-detect any format
)

# Load a PDF
loader = PyPDFLoader("annual_report.pdf")
documents = loader.load()
# documents = [
#   Document(page_content="Page 1 text...", metadata={"source": "annual_report.pdf", "page": 0}),
#   Document(page_content="Page 2 text...", metadata={"source": "annual_report.pdf", "page": 1}),
# ]

# Load a web page
loader = WebBaseLoader("https://example.com/blog/ai-trends")
documents = loader.load()

# Load everything in a directory
from langchain_community.document_loaders import DirectoryLoader
loader = DirectoryLoader("./data/", glob="**/*.pdf", loader_cls=PyPDFLoader)
documents = loader.load()
```

**The Document object** is LangChain's standard format:
```python
class Document:
    page_content: str           # the actual text
    metadata: dict              # source, page number, author, etc.
```

---

# Part 4: Stage 2 — Chunking (Text Splitting)

## Why You Can't Send the Whole Document

LLMs have a **context window** — a maximum number of tokens they can process at once. GPT-4o: 128K tokens. Claude: 200K tokens. But even if you COULD fit it all, retrieval works better with small, focused chunks.

**The real reason:** If you search for "Q3 revenue" and return the entire 200-page annual report, the LLM has to find the needle in the haystack. If you return just the 3 paragraphs that mention Q3 revenue, the LLM gives a much better answer.

## Chunking Strategies

### Strategy 1: Recursive Character Splitting (The Default)

Tries to split on natural boundaries: paragraphs first, then sentences, then words.

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,          # max characters per chunk
    chunk_overlap=50,        # overlap between consecutive chunks
    separators=["\n\n", "\n", ". ", " ", ""],   # try in this order
)

chunks = splitter.split_documents(documents)
```

```
How it works:

Original text:
  "Paragraph 1 about revenue.\n\nParagraph 2 about costs.\n\nParagraph 3..."

Step 1: Try splitting on "\n\n" (paragraph breaks)
  Chunk 1: "Paragraph 1 about revenue."
  Chunk 2: "Paragraph 2 about costs."
  Chunk 3: "Paragraph 3..."

If a chunk is STILL too long:
Step 2: Try splitting on "\n" (line breaks)
Step 3: Try splitting on ". " (sentences)
Step 4: Try splitting on " " (words)
Step 5: Split on "" (characters — last resort)
```

### Why Overlap Matters

```
Without overlap (chunk_overlap=0):
  Chunk 1: "The company reported revenue of $4.2 million in"
  Chunk 2: "Q3 2024, representing a 15% increase over Q2."

  If someone asks "What was Q3 revenue?", chunk 2 matches but is incomplete.
  Chunk 1 has the number but doesn't mention Q3.

With overlap (chunk_overlap=50):
  Chunk 1: "The company reported revenue of $4.2 million in Q3 2024, repre..."
  Chunk 2: "...ion in Q3 2024, representing a 15% increase over Q2."

  Both chunks contain the full context. Retrieval finds the complete answer.
```

### Chunk Size Guidelines

```
Chunk Size    Best For                           Trade-off
──────────    ────────                           ─────────
256 tokens    Precise factual Q&A                Very specific matches, might miss context
512 tokens    General Q&A (recommended default)  Good balance of precision and context
1024 tokens   Summarization, analysis            Broad context but less precise matching
2048 tokens   Long-form generation               Lots of context but slower retrieval
```

### Strategy 2: Semantic Chunking (Advanced)

Instead of splitting by character count, split where the MEANING changes.

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

splitter = SemanticChunker(
    embeddings=OpenAIEmbeddings(),
    breakpoint_threshold_type="percentile",
)

# How it works:
# 1. Embed each sentence
# 2. Compare consecutive sentence embeddings (cosine similarity)
# 3. Where similarity drops sharply → that's a topic boundary → split there

# Pro: chunks are semantically coherent
# Con: expensive (embeds every sentence during indexing)
```

---

# Part 5: Stage 3 — Embeddings

## What Is an Embedding?

An embedding converts text into a list of numbers (a vector) that captures its meaning. Similar texts produce similar vectors. This is how the computer "understands" meaning.

```
"How to reset my password"    → [0.12, -0.34, 0.78, ...]   (1536 numbers)
"I forgot my login"           → [0.11, -0.31, 0.76, ...]   (very similar!)
"Today's weather is sunny"    → [-0.45, 0.67, 0.02, ...]   (very different)

Similarity("reset password", "forgot login") = 0.92   (high — same topic)
Similarity("reset password", "weather sunny") = 0.05   (low — different topics)
```

## Embedding Models

```
Model                          Dims    Quality     Cost              Provider
─────                          ────    ───────     ────              ────────
text-embedding-3-small         1536    Very Good   $0.02/1M tokens   OpenAI
text-embedding-3-large         3072    Excellent   $0.13/1M tokens   OpenAI
embed-english-v3               1024    Excellent   Free tier          Cohere
all-MiniLM-L6-v2               384     Good        Free (local)      HuggingFace
nomic-embed-text-v1.5          768     Very Good   Free (local)      Nomic
bge-large-en-v1.5              1024    Excellent   Free (local)      BAAI/HuggingFace
```

```python
# OpenAI embeddings
from langchain_openai import OpenAIEmbeddings
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# Embed a single query
vector = embeddings.embed_query("What is RAG?")
# Returns: [0.012, -0.034, 0.078, ...] (1536 floats)

# Embed multiple documents (batch — faster and cheaper)
vectors = embeddings.embed_documents(["doc 1 text", "doc 2 text", "doc 3 text"])
# Returns: [[...], [...], [...]] (3 vectors)

# HuggingFace embeddings (free, runs locally)
from langchain_huggingface import HuggingFaceEmbeddings
embeddings = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2")

# Ollama embeddings (free, runs locally)
from langchain_ollama import OllamaEmbeddings
embeddings = OllamaEmbeddings(model="nomic-embed-text")
```

### Critical Rule

**You MUST use the same embedding model for indexing AND querying.** If you embed documents with `text-embedding-3-small`, you must embed queries with `text-embedding-3-small`. Vectors from different models live in incompatible mathematical spaces.

---

# Part 6: Stage 4 — Vector Store (Database)

## What Is a Vector Store?

A database optimized for storing and searching vectors. You give it a query vector, it returns the K most similar stored vectors. This is called "similarity search" or "nearest neighbor search."

```
Your documents (after embedding):
  "Q3 revenue was $4.2M"     → [0.12, -0.34, 0.78, ...]
  "Parking policy updated"   → [-0.45, 0.67, 0.02, ...]
  "New hire onboarding"      → [0.23, 0.15, -0.44, ...]
  ... (thousands more)

Query: "What was the revenue?"  → [0.11, -0.31, 0.76, ...]

Vector store finds: "Q3 revenue was $4.2M" is the most similar!
(because [0.11, -0.31, 0.76] is closest to [0.12, -0.34, 0.78])
```

### Vector Store Options

```python
# ChromaDB (local, easy, great for prototyping)
from langchain_chroma import Chroma

vectorstore = Chroma.from_documents(
    documents=chunks,              # your chunked documents
    embedding=embeddings,           # your embedding model
    persist_directory="./chroma_db" # save to disk
)

# FAISS (local, very fast, from Meta)
from langchain_community.vectorstores import FAISS

vectorstore = FAISS.from_documents(chunks, embeddings)
vectorstore.save_local("faiss_index")                    # save
vectorstore = FAISS.load_local("faiss_index", embeddings) # load

# Pinecone (cloud, managed, production-ready)
from langchain_pinecone import PineconeVectorStore

vectorstore = PineconeVectorStore.from_documents(
    chunks, embeddings, index_name="my-index"
)

# pgvector (PostgreSQL extension — use your existing DB)
from langchain_postgres import PGVector

vectorstore = PGVector.from_documents(
    chunks, embeddings, connection=DATABASE_URL, collection_name="documents"
)
```

### Searching the Vector Store

```python
# Basic similarity search — return top 5 most similar chunks
results = vectorstore.similarity_search("What was Q3 revenue?", k=5)
# Returns: [Document(page_content="Q3 revenue was...", metadata={...}), ...]

# With relevance scores
results = vectorstore.similarity_search_with_score("What was Q3 revenue?", k=5)
# Returns: [(Document(...), 0.92), (Document(...), 0.87), ...]

# As a retriever (LangChain integration)
retriever = vectorstore.as_retriever(
    search_type="similarity",        # or "mmr" (Maximum Marginal Relevance)
    search_kwargs={"k": 5},
)
docs = retriever.invoke("What was Q3 revenue?")
```

### MMR vs Similarity Search

```
Similarity search: return the 5 MOST similar documents
  Problem: all 5 might say the same thing (redundant)

MMR (Maximum Marginal Relevance): return 5 documents that are similar
  to the query but DIVERSE from each other
  Better coverage of different aspects of the topic
```

---

# Part 7: Stage 5 — Retrieval Strategies

## Basic Retrieval

```python
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})
docs = retriever.invoke("What was Q3 revenue?")
```

## Hybrid Search (Vector + Keyword)

Combines semantic search (meaning) with keyword search (exact terms). Catches cases where the exact term matters.

```
Query: "Error code XJ-4092"

Vector search finds: "System errors can occur during authentication..."  (semantic match, wrong doc)
Keyword search finds: "Error XJ-4092: Database connection timeout"       (exact match, right doc!)

Hybrid combines both → much better recall.
```

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

# BM25 = keyword-based (like Google before AI)
bm25 = BM25Retriever.from_documents(chunks, k=5)

# Vector = semantic
vector_retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

# Combine with weights
hybrid = EnsembleRetriever(
    retrievers=[bm25, vector_retriever],
    weights=[0.3, 0.7],     # 70% semantic, 30% keyword
)
```

## Multi-Query Retrieval

The user's query might not match the document's wording. Generate multiple phrasings, retrieve for each, and combine.

```python
from langchain.retrievers import MultiQueryRetriever
from langchain_openai import ChatOpenAI

retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(),
    llm=ChatOpenAI(temperature=0),
)

# User asks: "What was Q3 revenue?"
# LLM generates:
#   "Q3 quarterly revenue figures"
#   "Third quarter financial results 2024"
#   "Revenue report for July August September"
# Retrieves for ALL of them, deduplicates results
```

## Re-Ranking

Retrieve a large set (20), then use a smarter model to re-score and keep the best 5.

```python
# Retrieve 20 candidates
docs = vectorstore.similarity_search(query, k=20)

# Re-rank with Cohere
import cohere
co = cohere.Client(api_key="...")
results = co.rerank(
    query=query,
    documents=[d.page_content for d in docs],
    top_n=5,
    model="rerank-english-v3.0",
)
final_docs = [docs[r.index] for r in results.results]
```

---

# Part 8: Stage 6 — Prompt Construction and Generation

## Building the Prompt

```python
# The RAG prompt template
RAG_PROMPT = """You are a helpful assistant that answers questions based on the provided context.

Rules:
1. Answer ONLY based on the context below. Do not use prior knowledge.
2. If the context doesn't contain the answer, say "I don't have enough information."
3. Cite which source each piece of information comes from.

Context:
{context}

Question: {question}

Answer:"""
```

```python
# Full RAG chain
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

# Components
llm = ChatOpenAI(model="gpt-4o", temperature=0)
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})
prompt = ChatPromptTemplate.from_template(RAG_PROMPT)

# Format retrieved documents into a single string
def format_docs(docs):
    return "\n\n".join(
        f"[Source: {d.metadata.get('source', 'unknown')}]\n{d.page_content}"
        for d in docs
    )

# Build the chain (LCEL — LangChain Expression Language)
rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

# Use it
answer = rag_chain.invoke("What was Q3 revenue?")
print(answer)
# "According to the annual report, Q3 revenue was $4.2 million,
#  representing a 15% increase over Q2. [Source: annual_report.pdf]"
```

## Streaming the Response

```python
# Stream tokens as they're generated
async for chunk in rag_chain.astream("What was Q3 revenue?"):
    print(chunk, end="", flush=True)
```

---

# Part 9: Advanced RAG Patterns

## Corrective RAG (CRAG)

After retrieval, check if the documents are actually relevant. If not, fall back to web search.

```
Query → Retrieve → Grade documents (relevant? ambiguous? irrelevant?)
                      ↓                              ↓
                   Relevant                      Irrelevant
                      ↓                              ↓
                   Generate answer              Web search → Generate answer
```

## Self-RAG

The LLM evaluates its OWN output: "Is my answer actually supported by the context?"

```
1. Retrieve → Generate answer
2. Ask LLM: "Is this answer supported by the retrieved documents?"
3. If no → retrieve again with different query → regenerate
4. If yes → return answer
```

## Agentic RAG

Instead of a fixed pipeline, an AI agent DECIDES how to retrieve:

```
User: "Compare Product A and Product B revenues in 2024"

Agent thinks:
  Step 1: Search for "Product A revenue 2024" → got results
  Step 2: Search for "Product B revenue 2024" → got results
  Step 3: Both results are incomplete, search "product comparison 2024" → more results
  Step 4: Now I have enough context → generate comprehensive comparison
```

## Parent-Child Chunking

Small chunks for precise retrieval, big chunks for context:

```
Document split into large "parent" chunks (2000 tokens)
Each parent split into small "child" chunks (256 tokens)

At query time:
  1. Search child chunks (precise matching)
  2. Return the PARENT chunk (broader context)
  3. LLM gets rich context even though matching was precise
```

---

# Part 10: Evaluation

## How to Know If Your RAG Is Working

```
Metric              What It Measures                    Good Score
──────              ────────────────                    ──────────
Faithfulness        Does the answer only use the        > 0.9
                    provided context? (no hallucination)
Answer Relevancy    Does the answer actually address    > 0.85
                    the question?
Context Precision   Are the retrieved chunks relevant   > 0.8
                    to the question?
Context Recall      Did we retrieve ALL the chunks      > 0.8
                    needed to answer?
```

```python
# RAGAS evaluation framework
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision

result = evaluate(
    dataset={
        "question": ["What was Q3 revenue?"],
        "answer": ["Q3 revenue was $4.2M..."],
        "contexts": [["The company reported $4.2M in Q3..."]],
        "ground_truth": ["Q3 revenue was $4.2 million"],
    },
    metrics=[faithfulness, answer_relevancy, context_precision],
)
```

---

# Part 11: Anti-Hallucination Strategies

```
Strategy                     How It Works
────────                     ────────────
Low temperature              Set temperature=0 to 0.3 for factual Q&A
Grounding instructions       "Answer ONLY based on the provided context"
Score threshold              If no chunk scores above 0.7, say "I don't know"
Citation requirements        Force LLM to cite [Source: filename.pdf]
LLM-as-judge                 Second LLM call checks: "Is this answer supported?"
Confidence scoring           Ask LLM to rate its confidence 1-5
```

---

# Part 12: 🧩 Interview Q&A

**Q: What is RAG in one sentence?**
A: RAG retrieves relevant documents from your data and injects them into the LLM prompt so the answer is grounded in facts rather than the model's training data.

**Q: Why not just put all documents in the prompt?**
A: Context windows have limits (even 200K tokens is only ~150 pages). Retrieval finds the RELEVANT 5 chunks out of millions, giving the LLM focused context for better answers.

**Q: What's the most important stage to get right?**
A: Chunking. Bad chunks destroy retrieval quality regardless of how good your embeddings or LLM are. If a chunk splits a sentence about "Q3 revenue" across two chunks, neither chunk is retrievable.

**Q: Why use a vector database instead of regular search?**
A: Regular search (keyword/BM25) matches exact words. Vector search matches MEANING. "How to reset my password" matches "forgot login credentials" even though they share zero words. For semantic questions over documents, vector search is essential.

**Q: How do you prevent hallucination in RAG?**
A: Low temperature, explicit grounding instructions in the prompt, relevance score thresholds (say "I don't know" if retrieval quality is low), citation requirements, and optionally a second LLM call to verify the answer is supported.

**Q: What's the difference between Naive RAG, Advanced RAG, and Agentic RAG?**
A: Naive RAG is the basic pipeline (retrieve → generate). Advanced RAG adds query rewriting, hybrid search, re-ranking, and answer grading. Agentic RAG uses an AI agent that dynamically decides how many retrieval steps to take and which strategies to use.

**Q: Vector search vs keyword search — when to use which?**
A: Vector for semantic questions ("What's our refund policy?"). Keyword for specific terms ("Error code XJ-4092"). Hybrid (both combined) for production systems.
