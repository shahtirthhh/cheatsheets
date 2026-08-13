# Vector Databases — Complete Staff-Level Guide

_Embeddings · ANN Algorithms · Index Tuning · Filtering · Scaling_

---

# Part 1: What a Vector Database Actually Solves

## The Core Problem

```
You have 10 million document chunks, each embedded as a 1536-dimensional vector.
A user asks a question. You embed it. Now find the 5 most similar chunks.

BRUTE FORCE (exact k-NN):
  Compute cosine similarity against ALL 10M vectors.
  10M × 1536 multiply-adds = ~15 billion operations per query.
  Latency: 2-10 seconds. Completely unusable.

APPROXIMATE NEAREST NEIGHBOR (ANN):
  Trade a tiny bit of accuracy for a massive speedup.
  Return ~97% of the true top-5 in 5-20 milliseconds.
  That 3% miss rate is invisible to users; the 500x speedup is not.

A vector database is: ANN index + metadata store + filtering + persistence + scaling.
```

## Why Not Just Use PostgreSQL Without pgvector?

```
A B-tree index works on ORDERED data — you can binary search because
1 < 2 < 3 gives you a total ordering.

Vectors have NO total ordering. "Closeness" in 1536 dimensions isn't
a sortable scalar. You can't binary search a 1536-D space.

This is why ANN algorithms exist: they impose STRUCTURE
(graphs, clusters, hash buckets) that makes proximity search tractable.
```

---

# Part 2: Distance Metrics

```python
import numpy as np

a = np.array([1.0, 2.0, 3.0])
b = np.array([2.0, 4.0, 6.0])     # exactly 2× a — same DIRECTION, different MAGNITUDE

# ── COSINE SIMILARITY: angle between vectors (ignores magnitude) ──
cosine_sim = np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
# = 1.0  (identical direction)
# Range: -1 (opposite) to 1 (identical)
# Cosine DISTANCE = 1 - cosine_similarity  → 0 (identical) to 2 (opposite)

# ── EUCLIDEAN (L2) DISTANCE: straight-line distance ──
l2 = np.linalg.norm(a - b)        # = 3.74  (they ARE far apart in space)
# Range: 0 (identical) to ∞

# ── DOT PRODUCT (inner product) ──
dot = np.dot(a, b)                # = 28
# Unbounded. Grows with magnitude AND alignment.
```

```
WHICH METRIC TO USE:

  COSINE      Text embeddings, semantic search. THE DEFAULT.
              Document length shouldn't affect similarity — a 2-page and a
              20-page document about the same topic should match equally well.
              Magnitude in text embeddings often encodes length/frequency, not meaning.

  L2          Image embeddings, coordinates, physical measurements where
              actual distance in space is meaningful.

  DOT PRODUCT Recommendation systems where magnitude encodes popularity/confidence.
              Also: if vectors are NORMALIZED, dot product == cosine similarity,
              and it's cheaper (no division). This is why embedding APIs
              return normalized vectors — you get cosine for the price of a dot product.

⚠️ THE INDEX AND THE QUERY MUST USE THE SAME METRIC.
   Building an HNSW index with L2 and querying with cosine returns garbage.
```

---

# Part 3: ANN Algorithms (The Part Interviewers Probe)

## Flat (Brute Force) — The Baseline

```
Stores vectors as-is. Compares against every single one.

  Recall:  100% (exact — it IS the ground truth)
  Speed:   O(n × d) per query
  Memory:  n × d × 4 bytes (float32)
  Build:   instant (no index to build)

USE WHEN: fewer than ~50K vectors, or when you need guaranteed exact results.
At 10K vectors this is often FASTER than an index because there's no traversal overhead.
```

## HNSW (Hierarchical Navigable Small World) — The Production Default

```
A multi-layer proximity graph. Think "skip list for vectors."

  Layer 2 (sparse):     A ──────────────── D
                        │                  │
  Layer 1 (medium):     A ──── B ───────── D ──── F
                        │      │           │      │
  Layer 0 (all nodes):  A ─ B ─ C ─ D ─ E ─ F ─ G ─ H

SEARCH ALGORITHM:
  1. Enter at the TOP layer (few nodes, long-range links)
  2. Greedily walk to the neighbor closest to the query
  3. When no neighbor is closer, DROP DOWN one layer
  4. Repeat — each layer refines the search in a smaller neighborhood
  5. At layer 0, collect the ef closest candidates and return the top k

  The top layers give you "coarse navigation" across the whole space.
  The bottom layer gives you "fine-grained local search."
  Result: O(log n) instead of O(n).
```

```python
# ── HNSW parameters ──
# m                 (16 default)  Max connections per node per layer.
#                                 Higher = better recall, more memory, slower build.
#                                 Typical: 16 (general), 32-64 (high recall needed)
#
# ef_construction   (64-200)      Size of the candidate list during BUILD.
#                                 Higher = better graph quality, much slower build.
#                                 Typical: 128-200 for production.
#
# ef_search         (k to 500)    Size of the candidate list during SEARCH.
#                                 THE runtime recall/latency dial.
#                                 Must be >= k. Higher = better recall, slower query.
#                                 Tune this per-query without rebuilding the index.

# Memory estimate:  n × (d × 4 bytes + m × 2 × 4 bytes)
# 1M vectors, 1536 dims, m=16:  1M × (6144 + 128) ≈ 6.3 GB
```

```
HNSW PROPERTIES:
  Recall:   95-99% (tunable via ef_search)
  Speed:    O(log n) — sub-millisecond at millions of vectors
  Memory:   HIGH — the entire graph must be in RAM
  Build:    slow (minutes to hours for millions of vectors)
  Updates:  supports incremental insert; deletes are "soft" (tombstoned)

  ⚠️ Deletion weakness: HNSW can't cleanly remove a node without degrading the graph.
     Most implementations tombstone and require periodic rebuilds.
     High-churn workloads suffer.
```

## IVF (Inverted File Index) — Cluster-Based

```
1. BUILD: run k-means to partition vectors into `nlist` clusters.
   Each cluster has a centroid.

        ● ● ●          ▲ ▲ ▲
       ● (C1) ●       ▲ (C2) ▲        C = centroid
        ● ● ●          ▲ ▲ ▲
                 ■ ■ ■
                ■ (C3) ■
                 ■ ■ ■

2. SEARCH: compare the query to the `nlist` centroids only (cheap).
   Pick the `nprobe` closest clusters.
   Search exhaustively ONLY inside those clusters.

   nlist=1000, nprobe=10 → you search 1% of the data. 100x speedup.
```

```python
# nlist   Number of clusters. Rule of thumb: sqrt(n) to 4*sqrt(n)
#         1M vectors → nlist ≈ 1000-4000
# nprobe  Clusters to search at query time. THE recall/speed dial.
#         nprobe=1 → fast, ~70% recall
#         nprobe=10 → balanced, ~95% recall
#         nprobe=nlist → exact search (no speedup)

# ⚠️ IVF REQUIRES TRAINING. You must fit k-means on a representative sample
#    BEFORE inserting. Inserting 10 vectors then training gives terrible clusters.
#    Train on at least 30 × nlist vectors.

# ⚠️ THE EDGE PROBLEM: if the true nearest neighbor sits just across a cluster
#    boundary and that cluster isn't probed, you miss it. This is IVF's main
#    recall failure mode. Raising nprobe mitigates it.
```

```
IVF PROPERTIES:
  Recall:   90-98% (tunable via nprobe)
  Speed:    fast, though slower than HNSW at equal recall
  Memory:   LOWER than HNSW (no graph edges to store)
  Build:    fast (k-means is quick)
  Updates:  new vectors go to the nearest existing cluster — quality drifts over time,
            requiring periodic retraining
```

## Product Quantization (PQ) — Compression

```
The problem: 1 billion vectors × 1536 dims × 4 bytes = 6.1 TB of RAM. Impossible.

PQ SOLUTION — compress each vector by 32-64x:

  1. Split the 1536-D vector into m sub-vectors (e.g. m=96, each 16-D)
  2. For each sub-space, run k-means to learn 256 centroids (a "codebook")
  3. Replace each sub-vector with the ID of its nearest centroid (1 byte, since 256 = 2^8)

  Original:   1536 dims × 4 bytes = 6144 bytes
  Compressed: 96 sub-vectors × 1 byte = 96 bytes
  → 64x compression

  Distance is computed on the compressed codes using precomputed lookup tables,
  which is also FASTER than full-precision arithmetic.

  COST: recall drops to 80-95% depending on m and the data distribution.
```

```
COMBINED INDEXES (what large-scale systems actually use):

  IVF_PQ      Cluster first, then compress. Billion-scale on commodity hardware.
  IVF_FLAT    Cluster, keep full precision. Good balance under ~100M.
  HNSW_PQ     Graph navigation + compressed vectors. Lower memory than pure HNSW.
  HNSW_SQ     Scalar quantization (float32 → int8). 4x compression, ~1% recall loss.
              Often the best default: nearly free memory savings.
```

## Algorithm Selection Table

```
Vectors      Recommended Index       Reasoning
───────      ─────────────────       ─────────
< 10K        Flat                    Index overhead exceeds the benefit
10K - 100K   Flat or HNSW            HNSW if latency matters
100K - 10M   HNSW                    Best recall/latency; RAM is affordable
10M - 100M   HNSW_SQ or IVF_FLAT     Memory becomes the constraint
100M - 1B+   IVF_PQ / DiskANN        Compression is mandatory
High churn   IVF                     HNSW deletion degrades the graph
Exact needed Flat                    No approximation acceptable
```

---

# Part 4: The Vector Databases

```
                Type        Filtering   Hybrid   Scale     Ops Burden   Best For
                ────        ─────────   ──────   ─────     ──────────   ────────
Chroma          Embedded    Good        Basic    <1M       None         Prototyping
FAISS           Library     Manual      No       <10M      Low          Speed, research
pgvector        Extension   Excellent   Via FTS  <50M      None extra   Already on Postgres ★
Qdrant          Dedicated   Excellent   Yes      100M+     Medium       Production default ★
Weaviate        Dedicated   Excellent   Native   100M+     Medium       Built-in hybrid
Milvus          Dedicated   Good        Yes      10B+      High         Massive scale
Pinecone        Managed     Good        Yes      1B+       None         Zero ops, paid
Elasticsearch   Search eng  Excellent   Native   1B+       High         Already using ES
```

## pgvector (Recommended If You Run PostgreSQL)

```sql
CREATE EXTENSION vector;

CREATE TABLE documents (
    id          BIGSERIAL PRIMARY KEY,
    tenant_id   BIGINT NOT NULL,
    content     TEXT NOT NULL,
    metadata    JSONB DEFAULT '{}',
    embedding   VECTOR(1536),
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- ── HNSW index (recommended) ──
CREATE INDEX idx_docs_embedding_hnsw ON documents
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 128);

-- Query-time recall dial
SET hnsw.ef_search = 100;

-- ── IVFFlat index (faster build, needs data present first) ──
CREATE INDEX idx_docs_embedding_ivf ON documents
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 1000);
SET ivfflat.probes = 10;

-- Operator classes must match your metric:
--   vector_cosine_ops  → <=>  cosine distance
--   vector_l2_ops      → <->  L2 distance
--   vector_ip_ops      → <#>  negative inner product

-- ── THE KILLER QUERY: vector search + SQL filters + JOIN ──
SELECT d.id, d.content, u.name AS author,
       1 - (d.embedding <=> $1::vector) AS similarity
FROM documents d
JOIN users u ON u.id = (d.metadata->>'author_id')::bigint
WHERE d.tenant_id = $2                                  -- hard multi-tenant isolation
  AND d.metadata->>'department' = 'finance'
  AND d.created_at > NOW() - INTERVAL '90 days'
  AND 1 - (d.embedding <=> $1::vector) > 0.7            -- similarity floor
ORDER BY d.embedding <=> $1::vector
LIMIT 10;

-- Pinecone cannot JOIN. Chroma cannot JOIN. This is pgvector's decisive advantage.

-- Supporting indexes for the filters (critical — otherwise the filter is a seq scan)
CREATE INDEX idx_docs_tenant ON documents(tenant_id);
CREATE INDEX idx_docs_metadata ON documents USING GIN(metadata);
CREATE INDEX idx_docs_created ON documents(created_at DESC);
```

## Qdrant (Production Standalone)

```python
from qdrant_client import QdrantClient
from qdrant_client.models import (
    Distance, VectorParams, PointStruct, Filter,
    FieldCondition, MatchValue, Range, HnswConfigDiff,
    OptimizersConfigDiff, ScalarQuantization, ScalarQuantizationConfig, ScalarType,
)

client = QdrantClient(url="http://localhost:6333", api_key=os.getenv("QDRANT_API_KEY"))

client.create_collection(
    collection_name="documents",
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
    hnsw_config=HnswConfigDiff(m=16, ef_construct=128),
    quantization_config=ScalarQuantization(
        scalar=ScalarQuantizationConfig(type=ScalarType.INT8, quantile=0.99, always_ram=True)
    ),   # 4x memory reduction, ~1% recall loss
    optimizers_config=OptimizersConfigDiff(indexing_threshold=20000),
)

# Payload indexes make filtering fast (without these, filters scan everything)
client.create_payload_index("documents", "tenant_id", field_schema="integer")
client.create_payload_index("documents", "department", field_schema="keyword")

client.upsert(
    collection_name="documents",
    points=[
        PointStruct(id=1, vector=embedding, payload={
            "tenant_id": 42, "department": "finance", "content": "...", "year": 2025,
        })
    ],
)

results = client.query_points(
    collection_name="documents",
    query=query_vector,
    query_filter=Filter(must=[
        FieldCondition(key="tenant_id", match=MatchValue(value=42)),
        FieldCondition(key="year", range=Range(gte=2024)),
    ]),
    limit=10,
    score_threshold=0.7,
    search_params={"hnsw_ef": 128},
).points
```

---

# Part 5: Filtering (The Hardest Real Problem)

```
Naive assumption: "just filter after searching."
Reality: this breaks badly.

PRE-FILTER (filter first, then search):
  Find all docs where tenant_id=42 → 500 docs → brute-force search those 500.
  ✅ Correct results, always returns k items if k exist
  ❌ Can't use the ANN index; degenerates to brute force on large filtered sets

POST-FILTER (search first, then filter):
  ANN search returns top 100 → filter to tenant_id=42 → maybe 3 survive.
  ✅ Fast, uses the index
  ❌ You asked for 10 and got 3. Worse: if tenant 42 is 0.1% of the data,
     you may get ZERO results despite thousands of matching documents existing.

FILTERED HNSW (what good vector DBs actually do):
  Apply the filter DURING graph traversal — only visit nodes passing the filter.
  Maintains index efficiency AND correctness.
  Qdrant, Weaviate, and pgvector (with proper indexes) do this.
  This is a real differentiator between vector DBs.
```

```python
# Practical rule for selective filters:
#   If the filter matches < ~1% of the corpus, PRE-FILTER (brute force the subset).
#   If it matches > ~10%, use FILTERED ANN.
#   In between, benchmark both.

# Multi-tenancy: NEVER post-filter tenant isolation.
# Post-filtering leaks timing information and can return empty results
# for legitimate queries. Always enforce tenant_id at the index level.
```

---

# Part 6: Hybrid Search

```python
# Dense (semantic) + Sparse (keyword) fused by rank.

from qdrant_client.models import SparseVector, Prefetch, FusionQuery, Fusion

results = client.query_points(
    collection_name="documents",
    prefetch=[
        Prefetch(query=dense_vector, using="dense", limit=50),
        Prefetch(query=SparseVector(indices=idx, values=vals), using="sparse", limit=50),
    ],
    query=FusionQuery(fusion=Fusion.RRF),      # reciprocal rank fusion
    limit=10,
).points
```

```python
# Manual RRF — works across any set of retrievers
def reciprocal_rank_fusion(result_lists, k=60, top_n=10):
    scores = {}
    for results in result_lists:
        for rank, doc in enumerate(results):
            key = doc.id
            scores.setdefault(key, {"doc": doc, "score": 0.0})
            scores[key]["score"] += 1.0 / (k + rank + 1)
    return [v["doc"] for v in sorted(scores.values(), key=lambda x: -x["score"])[:top_n]]

# WHY RRF BEATS WEIGHTED SCORE FUSION:
#   BM25 scores are unbounded (0 to ~30). Cosine is bounded (-1 to 1).
#   Normalizing them to combine is fragile and dataset-dependent.
#   RRF uses only RANK POSITION, so no normalization is needed and it's
#   robust to outlier scores. k=60 is the standard constant from the original paper.
```

---

# Part 7: Production Operations

## Capacity Planning

```
MEMORY (HNSW, float32):
  n × (d × 4 + m × 2 × 4) bytes

  1M   vectors, 1536-D, m=16  →  ~6.3 GB
  10M  vectors, 1536-D, m=16  →  ~63 GB
  100M vectors, 1536-D, m=16  →  ~630 GB   ← must compress

WITH INT8 SCALAR QUANTIZATION (4x):
  100M vectors → ~160 GB     ← now feasible on a large instance

WITH PQ (64x):
  1B vectors → ~100 GB       ← billion-scale on one machine

REDUCE DIMENSIONS FIRST (cheapest win):
  Matryoshka models let you truncate 3072 → 512 dims with ~2% quality loss
  and 6x memory savings. Always evaluate this before reaching for PQ.
```

## Index Tuning Workflow

```python
# You cannot tune what you don't measure. Build a ground-truth set first.

def build_ground_truth(queries, vectors, k=10):
    """Exact k-NN via brute force — the recall denominator."""
    truth = {}
    for qid, q in queries.items():
        sims = vectors @ q / (np.linalg.norm(vectors, axis=1) * np.linalg.norm(q))
        truth[qid] = set(np.argsort(-sims)[:k].tolist())
    return truth

def measure_recall(index, queries, truth, k=10, **search_params):
    total = 0.0
    for qid, q in queries.items():
        got = set(index.search(q, k=k, **search_params))
        total += len(got & truth[qid]) / k
    return total / len(queries)

# Sweep ef_search and record the recall/latency curve
for ef in [16, 32, 64, 128, 256, 512]:
    recall = measure_recall(index, queries, truth, k=10, ef_search=ef)
    latency = benchmark_latency(index, queries, ef_search=ef)
    print(f"ef={ef:4d}  recall@10={recall:.4f}  p95={latency:.2f}ms")

# Pick the smallest ef that meets your recall SLA. Typical target: recall@10 >= 0.95.
```

## Re-indexing Without Downtime

```python
# Embedding model upgrades require FULL re-indexing — vectors from different
# models are not comparable. Use blue-green collections.

def reindex_blue_green(client, old="documents_v1", new="documents_v2", alias="documents"):
    create_collection(client, new)                      # 1. build the new collection
    for batch in iter_documents(batch_size=500):        # 2. re-embed and populate
        vectors = new_embedding_model.embed_documents([d.content for d in batch])
        client.upsert(new, points=to_points(batch, vectors))

    assert measure_recall_on(new) >= 0.95               # 3. validate before switching
    client.update_collection_aliases(                    # 4. atomic alias switch
        change_aliases_operations=[
            {"delete_alias": {"alias_name": alias}},
            {"create_alias": {"collection_name": new, "alias_name": alias}},
        ]
    )
    # 5. keep the old collection for a rollback window, then drop it

# ALWAYS store the embedding model name and dimension in collection metadata
# so a mismatch is detected loudly rather than silently returning nonsense.
```

## Monitoring

```
METRIC                    WHY IT MATTERS                  ALERT ON
──────                    ──────────────                  ────────
p50/p95/p99 query latency User-facing performance         p95 > SLA
recall@k (sampled)        Silent quality degradation      recall < 0.90
Index memory usage        OOM risk                        > 80% of available RAM
Insert throughput         Ingestion keeping up            backlog growing
Filtered-query ratio      Detects post-filter starvation  empty results rising
Collection size drift     Unexpected growth/deletion      sudden change
Tombstone ratio (HNSW)    Graph degradation from deletes  > 20% → rebuild
```

---

# Part 8: 🧩 Interview Q&A

**Q: Explain how HNSW works and what its parameters control.**
A: HNSW builds a multi-layer proximity graph, conceptually a skip list for vectors. Upper layers are sparse with long-range links; the bottom layer contains every node with short-range links. A search enters at the top, greedily walks toward the query, drops a layer when no neighbor improves, and repeats — giving logarithmic rather than linear complexity. Three parameters matter: `m` is the max connections per node, controlling graph density and memory; `ef_construction` is the candidate list size at build time, controlling graph quality; `ef_search` is the candidate list size at query time, which is the runtime recall-versus-latency dial you can tune without rebuilding. Its main weakness is deletion — nodes are tombstoned rather than removed, so high-churn workloads degrade and need periodic rebuilds.

**Q: HNSW versus IVF — when do you pick each?**
A: HNSW gives better recall at a given latency and needs no training, but it's memory-hungry because the whole graph must be resident, slow to build, and handles deletions poorly. IVF partitions vectors into k-means clusters and searches only the `nprobe` nearest ones; it uses less memory, builds fast, and tolerates updates better, but requires a training step on representative data and suffers the edge problem where a true neighbor just across an unprobed cluster boundary is missed. I'd default to HNSW under about 10 million vectors where RAM is affordable, and move to IVF or IVF_PQ when memory becomes the binding constraint or the workload has heavy churn.

**Q: Why is cosine similarity the default for text embeddings rather than Euclidean distance?**
A: In text embeddings, vector magnitude often correlates with document length or token frequency rather than meaning. A two-paragraph and a twenty-page document about the same topic should be equally similar to a query about that topic, but their L2 distance would differ substantially. Cosine measures only the angle, so it isolates semantic direction from magnitude. Worth noting: if vectors are L2-normalized — which most embedding APIs do — then cosine similarity is mathematically equivalent to the dot product, and the dot product is cheaper since it skips the division. That's why normalized vectors plus inner product is a common production configuration.

**Q: How do you handle filtering in vector search, and why is it hard?**
A: There are three approaches. Post-filtering searches first then filters the results, which is fast but can return far fewer than k items — if a tenant owns 0.1% of the corpus, a top-100 ANN search might yield zero of their documents even though thousands exist. Pre-filtering restricts the candidate set first then does exact search, which is correct but degenerates to brute force on large subsets. The right answer for production is filtered ANN, where the filter is applied during graph traversal so only qualifying nodes are visited — this preserves both index efficiency and correctness. Qdrant, Weaviate, and pgvector with proper supporting indexes do this. For multi-tenancy specifically, tenant isolation must be enforced at the index level, never post-hoc.

**Q: How would you scale a vector search system to a billion vectors?**
A: Memory is the binding constraint — a billion 1536-dimensional float32 vectors is over 6 TB. First, reduce dimensions using Matryoshka-capable embedding models, since truncating 3072 to 512 costs roughly 2% quality for 6x savings. Second, apply quantization: int8 scalar quantization gives 4x for about 1% recall loss, and product quantization gives 32-64x for a larger but often acceptable loss. Third, use IVF_PQ or DiskANN so the index doesn't need to be fully resident. Fourth, shard by tenant or domain so each query searches a smaller subspace via pre-filtering. Fifth, use a two-stage pipeline: cheap approximate retrieval of 50-100 candidates followed by a cross-encoder reranker, which recovers most of the accuracy lost to compression. Finally, cache aggressively at both the embedding and answer level.

**Q: How do you validate that your ANN index is actually returning good results?**
A: Build a ground-truth set by running exact brute-force k-NN over a representative query sample, then measure recall@k as the overlap between the ANN results and the exact results. Sweep the runtime parameter — `ef_search` for HNSW or `nprobe` for IVF — and record the recall-versus-latency curve, then pick the smallest value meeting your recall SLA, typically 0.95 at k=10. In production, sample a small percentage of live queries, compute exact results offline, and track recall as a continuous metric — silent recall degradation from index drift or deletions is otherwise invisible, since the system returns results happily either way.

**Q: What happens when you change embedding models?**
A: Everything must be re-indexed. Vectors from different models occupy different, incompatible spaces — similarity scores between them are meaningless noise, and the failure is silent rather than an error, which makes it dangerous. The safe pattern is blue-green: build a new collection with the new model, re-embed the entire corpus, validate recall against a ground-truth set, then atomically switch a collection alias, keeping the old collection for a rollback window. Store the embedding model name and dimension in collection metadata so a mismatch surfaces loudly at startup rather than degrading quality invisibly.
