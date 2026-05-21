# Chapter 3: RAG Architecture

## Overview

Retrieval-Augmented Generation (RAG) is the most common pattern for giving LLMs access to your data. Instead of fine-tuning, you retrieve relevant documents and inject them into the prompt. This chapter covers the architecture end-to-end.

## The RAG Pipeline

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  User    │    │  Query   │    │ Retrieve │    │ Augment  │    │ Generate │
│  Query   │───→│ Process  │───→│ Chunks   │───→│  Prompt  │───→│ Response │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
                     │               │
                     ▼               ▼
               ┌──────────┐    ┌──────────┐
               │ Embedding│    │ Vector   │
               │  Model   │    │ Database │
               └──────────┘    └──────────┘
```

## Ingestion Pipeline

Before you can retrieve, you need to ingest your documents:

```
Documents → Parse → Chunk → Embed → Store in Vector DB
```

### Step 1: Parsing

Convert raw documents (PDF, HTML, Markdown, DOCX) into clean text.

| Source | Tool | Notes |
|--------|------|-------|
| PDF | LlamaParse, PyPDF2, Unstructured | LlamaParse handles tables/images best |
| HTML | BeautifulSoup, Trafilatura | Trafilatura extracts article content cleanly |
| Code | Tree-sitter | Parse by function/class, not arbitrary chunks |
| Markdown | Built-in | Easiest to work with |

### Step 2: Chunking

Chunking is the most impactful decision in your RAG pipeline.

**Chunking strategies:**

```
Fixed-size (naive):     [====][====][====][====]
                        Simple but breaks mid-sentence

Sentence-based:         [Sent1. Sent2.][Sent3. Sent4.]
                        Better, respects boundaries

Semantic:               [Topic A paragraph][Topic B paragraph]
                        Best quality, most complex

Recursive:              Split by \n\n → \n → . → " "
                        Good default, handles varied content
```

**Optimal chunk sizes (empirical):**

| Use Case | Chunk Size | Overlap |
|----------|-----------|---------|
| Q&A / Support | 256-512 tokens | 50 tokens |
| Document search | 512-1024 tokens | 100 tokens |
| Code search | Function-level | None |
| Legal / compliance | 1024-2048 tokens | 200 tokens |

**The overlap matters.** Without overlap, you lose context at chunk boundaries.

### Step 3: Embedding

Convert chunks into dense vectors for similarity search.

| Model | Dimensions | Speed | Quality | Cost |
|-------|-----------|-------|---------|------|
| text-embedding-3-small | 1536 | Fast | Good | $0.02/1M tokens |
| text-embedding-3-large | 3072 | Medium | Better | $0.13/1M tokens |
| Cohere embed-v3 | 1024 | Fast | Good | $0.10/1M tokens |
| BGE-M3 (local) | 1024 | Depends | Good | Free |

**Rule of thumb:** Start with `text-embedding-3-small`. Only upgrade if retrieval quality is the bottleneck (measure first).

### Step 4: Vector Storage

Store embeddings in a vector database for fast similarity search.

| Database | Type | Best For |
|----------|------|----------|
| Qdrant | Dedicated | Production, high performance |
| Pinecone | Managed | Easy setup, serverless |
| Chroma | Embedded | Prototyping, small datasets |
| pgvector | Extension | Already using PostgreSQL |
| Weaviate | Dedicated | Multi-modal, hybrid search |

## Retrieval Strategies

### Basic: Vector Similarity

```python
# Pseudocode
query_embedding = embed(user_query)
results = vector_db.search(query_embedding, top_k=5)
```

**Problem:** Pure vector search misses keyword matches. "Error code E1234" might not match semantically.

### Better: Hybrid Search

Combine vector search with keyword search (BM25):

```
Score = α × vector_score + (1-α) × bm25_score
```

Typical α = 0.7 (favor semantic, fall back to keywords).

### Best: Hybrid + Reranking

```
User Query
    │
    ├──→ Vector Search (top 20)──┐
    │                             ├──→ Merge ──→ Reranker (top 5) ──→ LLM
    └──→ BM25 Search (top 20) ──┘
```

**Reranking models** (Cohere Rerank, BGE-Reranker) re-score the merged results using cross-attention, which is more accurate than embedding similarity but too slow to run on the full corpus.

## Prompt Assembly

After retrieval, assemble the final prompt:

```
System: You are a helpful assistant. Answer based ONLY on the provided context.
If the context doesn't contain the answer, say "I don't know."

Context:
---
{chunk_1}
---
{chunk_2}
---
{chunk_3}

User: {original_question}
```

**Key design decisions:**
- **How many chunks?** 3-5 is typical. More = more context but higher cost and potential confusion
- **Ordering?** Most relevant first (models pay more attention to the beginning)
- **Citation?** Include source metadata so the model can cite: `[Source: docs/api.md, line 42]`

## Architecture Diagram: Production RAG

```
┌─────────────────────────────────────────────────────────┐
│                    Ingestion Pipeline                     │
│  ┌─────────┐  ┌────────┐  ┌────────┐  ┌─────────────┐  │
│  │ Document │→│ Parser │→│Chunker │→│  Embedder   │  │
│  │  Store   │  │        │  │        │  │             │  │
│  └─────────┘  └────────┘  └────────┘  └──────┬──────┘  │
│                                               │          │
│                                        ┌──────▼──────┐  │
│                                        │  Vector DB  │  │
│                                        │  + BM25     │  │
│                                        └──────┬──────┘  │
└───────────────────────────────────────────────┼─────────┘
                                                │
┌───────────────────────────────────────────────┼─────────┐
│                    Query Pipeline              │          │
│                                                │          │
│  User Query → Query Rewrite → ┌────────────────┤          │
│                               │ Hybrid Search  │          │
│                               └───────┬────────┘          │
│                                       │                   │
│                               ┌───────▼────────┐         │
│                               │   Reranker     │         │
│                               └───────┬────────┘         │
│                                       │                   │
│                               ┌───────▼────────┐         │
│                               │ Prompt Assembly│         │
│                               └───────┬────────┘         │
│                                       │                   │
│                               ┌───────▼────────┐         │
│                               │     LLM        │         │
│                               └───────┬────────┘         │
│                                       │                   │
│                               ┌───────▼────────┐         │
│                               │  Guardrails    │         │
│                               └───────┬────────┘         │
│                                       ▼                   │
│                                   Response                │
└───────────────────────────────────────────────────────────┘
```

## Common Pitfalls

1. **Chunking too large.** 2000-token chunks dilute relevance. Smaller chunks (256-512) with overlap usually perform better.
2. **No evaluation.** You can't improve what you don't measure. Track retrieval precision@k and answer correctness.
3. **Ignoring query rewriting.** User queries are often vague. Rewriting "how do I fix it?" to "how to fix authentication error in the login endpoint" dramatically improves retrieval.
4. **Stuffing too many chunks.** More context ≠ better answers. The "lost in the middle" problem means models ignore middle chunks. Use 3-5, not 20.
5. **Not handling "I don't know."** Without explicit instructions, LLMs will hallucinate answers from chunks that are tangentially related.

## Practice Problem

**Design a RAG system for a company with 10M support tickets.** Requirements:
- Agents should find relevant past tickets when handling new ones
- Response time < 2 seconds
- Must cite source tickets
- New tickets should be searchable within 5 minutes of creation

Consider: chunking strategy, embedding model, vector DB choice, indexing pipeline, query flow, cost at scale.

## Further Reading

- [Pinecone's RAG guide](https://www.pinecone.io/learn/retrieval-augmented-generation/)
- [LangChain RAG tutorial](https://python.langchain.com/docs/tutorials/rag/)
- [RAGAS evaluation framework](https://github.com/explodinggradients/ragas)
