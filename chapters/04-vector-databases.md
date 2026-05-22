# Chapter 4: Vector Databases

## Overview

Vector databases are purpose-built for storing and searching embeddings — the dense numerical representations that power semantic search, RAG, and recommendation systems. Choosing the right one and understanding the indexing algorithms is critical for performance at scale.

## How Vector Search Works

```
Query: "How do I reset my password?"
                │
                ▼
        ┌──────────────┐
        │   Embedding   │
        │    Model      │
        └──────┬───────┘
               │
               ▼
        [0.12, -0.45, 0.89, ..., 0.23]   ← 1536-dim vector
               │
               ▼
        ┌──────────────┐
        │  Vector DB    │   Find nearest neighbors
        │  (ANN search) │   in high-dimensional space
        └──────┬───────┘
               │
               ▼
        Top-K most similar documents
```

**Key insight:** Vector search finds *semantically similar* content, not keyword matches. "How to reset my password" matches "Steps to change your login credentials" even though they share no words.

## Indexing Algorithms

This is the core of vector DB performance. You're trading accuracy for speed.

### Flat Index (Brute Force)

```
Query vector vs EVERY stored vector → exact nearest neighbors
```

- **Accuracy:** 100% (exact)
- **Speed:** O(n) — checks every vector
- **When to use:** < 50k vectors, or as accuracy baseline

### IVF (Inverted File Index)

```
┌─────────────────────────────────────┐
│           Vector Space              │
│                                     │
│    ┌──────┐  ┌──────┐  ┌──────┐   │
│    │Cell 1│  │Cell 2│  │Cell 3│   │
│    │ ●●●  │  │ ●●   │  │●●●●  │   │
│    │  ●●  │  │●●●●  │  │ ●●   │   │
│    └──────┘  └──────┘  └──────┘   │
│    ┌──────┐  ┌──────┐              │
│    │Cell 4│  │Cell 5│              │
│    │ ●●●● │  │ ●●●  │  ★ query   │
│    │  ●   │  │●●    │              │
│    └──────┘  └──────┘              │
└─────────────────────────────────────┘

1. Cluster vectors into cells (Voronoi partitions)
2. At query time, only search nprobe nearest cells
```

- **Speed:** O(n/k) where k = number of cells
- **Tuning:** `nlist` (number of cells), `nprobe` (cells to search)
- **Trade-off:** Higher nprobe = more accurate but slower

### HNSW (Hierarchical Navigable Small World)

```
Layer 2 (sparse):    A ─────────────────── D
                     │                     │
Layer 1 (medium):    A ──── B ──── C ──── D
                     │      │      │      │
Layer 0 (dense):     A ─ B ─ C ─ D ─ E ─ F ─ G ─ H
```

- **How it works:** Multi-layer graph. Start at top (sparse) layer, greedily descend to denser layers, narrowing search at each level.
- **Speed:** O(log n) — logarithmic, very fast
- **Memory:** High — stores the graph structure
- **Best for:** Low-latency production search. Most popular algorithm.

### PQ (Product Quantization)

```
Original vector (1536 dims):
[0.12, -0.45, 0.89, ..., 0.23, -0.67, 0.34]

Split into subvectors:
[0.12, -0.45, ...] [0.89, ..., 0.23] [-0.67, 0.34, ...]
      │                    │                  │
      ▼                    ▼                  ▼
   Code: 42            Code: 187          Code: 5

Compressed: [42, 187, 5]   ← way smaller than 1536 floats
```

- **Purpose:** Compress vectors to reduce memory usage
- **Trade-off:** Lossy compression — reduces accuracy
- **When to use:** Massive datasets (100M+ vectors) where memory is the bottleneck

### Algorithm Comparison

| Algorithm | Speed | Memory | Accuracy | Best For |
|-----------|-------|--------|----------|----------|
| Flat | Slow | Low | 100% | < 50k vectors, baseline |
| IVF | Medium | Low | ~95% | Medium datasets, memory-constrained |
| HNSW | Fast | High | ~99% | Production, low-latency |
| IVF+PQ | Fast | Very Low | ~90% | 100M+ vectors |
| HNSW+PQ | Fast | Medium | ~95% | Large scale + low latency |

## Vector Database Comparison

| Database | Type | Algorithm | Filtering | Unique Strength |
|----------|------|-----------|-----------|-----------------|
| **Qdrant** | Dedicated | HNSW | Advanced | Best filtering + performance |
| **Pinecone** | Managed | Proprietary | Good | Zero-ops, serverless |
| **Weaviate** | Dedicated | HNSW | Excellent | Multi-modal, GraphQL API |
| **Milvus** | Dedicated | IVF/HNSW | Good | Massive scale (billions) |
| **Chroma** | Embedded | HNSW | Basic | Simplest for prototyping |
| **pgvector** | Extension | IVF/HNSW | Full SQL | Already using PostgreSQL |
| **Redis** | Extension | HNSW | Basic | Already using Redis |

### Decision Framework

```
Prototyping / < 100k vectors?
  → Chroma (embedded, zero config)

Already have PostgreSQL?
  → pgvector (no new infra)

Need managed / zero-ops?
  → Pinecone (serverless)

Need best performance + filtering?
  → Qdrant or Weaviate

Billions of vectors?
  → Milvus

Need hybrid search (vector + keyword)?
  → Qdrant, Weaviate, or Elasticsearch
```

## Architecture: Production Vector Search

```
┌─────────────────────────────────────────────────────────┐
│                  Write Path                              │
│                                                         │
│  Document ──→ Chunk ──→ Embed ──→ Upsert to Vector DB  │
│                                        │                 │
│                                   ┌────▼────┐           │
│                                   │ Vector  │           │
│                                   │  Index  │           │
│                                   │ (HNSW)  │           │
│                                   └────┬────┘           │
│                                        │                 │
│                                   ┌────▼────┐           │
│                                   │Metadata │           │
│                                   │  Store  │           │
│                                   │(filter) │           │
│                                   └─────────┘           │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                  Read Path                               │
│                                                         │
│  Query ──→ Embed ──→ ANN Search ──→ Filter ──→ Top-K   │
│                          │                               │
│                     Pre-filter vs Post-filter:           │
│                     Pre: filter THEN search (faster)     │
│                     Post: search THEN filter (accurate)  │
└─────────────────────────────────────────────────────────┘
```

## Metadata Filtering

Real-world search always combines semantic similarity with filters.

```python
# "Find similar support tickets from the billing team in the last 30 days"
results = vector_db.search(
    vector=embed("customer charged twice"),
    top_k=10,
    filter={
        "team": "billing",
        "created_at": {"$gte": "2026-04-21"},
        "status": {"$in": ["open", "in_progress"]}
    }
)
```

**Pre-filtering vs post-filtering:**

| Strategy | How | Trade-off |
|----------|-----|-----------|
| Pre-filter | Filter first, then ANN search on subset | Fast, but fewer candidates may reduce quality |
| Post-filter | ANN search first, then filter results | Better quality, but may return < top_k results |

Most production systems use pre-filtering with over-fetching (search top_k * 3, filter, return top_k).

## Scaling Patterns

### Sharding

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Shard 1  │  │ Shard 2  │  │ Shard 3  │
│ Users A-F│  │ Users G-N│  │ Users O-Z│
│ 10M vecs │  │ 10M vecs │  │ 10M vecs │
└──────────┘  └──────────┘  └──────────┘
      │              │              │
      └──────────────┼──────────────┘
                     │
              Merge top-K from each shard
```

### Replication

```
┌──────────┐
│ Primary  │───write───→ Vector Index
│          │
└────┬─────┘
     │ replicate
     ├──────────┐
┌────▼─────┐ ┌──▼───────┐
│ Replica 1│ │ Replica 2│  ← read traffic
└──────────┘ └──────────┘
```

## Common Pitfalls

1. **Wrong distance metric.** Cosine similarity for normalized embeddings, L2 for raw embeddings. Using the wrong one silently degrades results.
2. **Not indexing metadata.** Filtering on unindexed fields is a full scan. Index fields you filter on frequently.
3. **Embedding model mismatch.** Query and document embeddings MUST use the same model. Mixing models = garbage results.
4. **Over-engineering early.** Start with Chroma or pgvector. Move to dedicated vector DB when you hit performance limits, not before.
5. **Ignoring index build time.** HNSW index builds are slow for millions of vectors. Plan for reindexing windows.

## Practice Problem

**Design a vector search system for an e-commerce product catalog.** Requirements:
- 5M products, each with title, description, images, and 50+ attributes
- Search by text ("red running shoes under $100") and by image (upload a photo)
- Filters: category, price range, brand, availability, rating
- P99 latency < 100ms
- New products searchable within 1 minute of listing

Consider: embedding strategy (text vs multi-modal), index type, filtering approach, sharding, cache layer.

## Further Reading

- [Pinecone's Guide to Vector Databases](https://www.pinecone.io/learn/vector-database/)
- [HNSW Algorithm Explained](https://www.pinecone.io/learn/series/faiss/hnsw/)
- [Qdrant Benchmarks](https://qdrant.tech/benchmarks/)
- [pgvector Performance Guide](https://neon.tech/blog/pgvector-performance)
