# Practice Problem: Design a RAG Pipeline for 10M Documents

> Difficulty: ⭐⭐⭐ | Key Concepts: Vector DBs, chunking strategies, reranking

## Problem Statement

Design a RAG system for a legal firm with 10M documents (contracts, case law, internal memos). Lawyers need to search and get AI-synthesized answers with citations.

## Requirements

### Functional
- Natural language search across all documents
- AI-generated answers with exact citations (document name, page, paragraph)
- Support document types: PDF, DOCX, scanned images (OCR needed)
- Filter by: date range, document type, client, practice area
- New documents searchable within 5 minutes of upload
- Highlight relevant passages in source documents

### Non-Functional
- Search latency < 2 seconds (P99)
- Retrieval accuracy > 85% (relevant docs in top 5)
- Handle 1,000 concurrent users
- 99.9% uptime (legal deadlines are non-negotiable)
- Cost < $50K/month total infrastructure

## Constraints
- Documents contain confidential client information — no external APIs for embedding (must self-host)
- Some documents are 500+ pages
- Scanned PDFs need OCR before processing
- Must comply with legal data retention (7 years minimum)

## Hints

1. How do you chunk a 500-page contract? Fixed-size won't work.
2. Self-hosted embeddings — which model? How many GPUs?
3. 10M documents × 10 chunks each = 100M vectors. Which vector DB handles this?
4. How do you handle OCR quality issues (bad scans)?

## Solution Walkthrough

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                 Ingestion Pipeline                       │
│                                                         │
│  Upload ──→ OCR ──→ Parse ──→ Chunk ──→ Embed ──→ Store │
│   │         (if     │         │         │          │     │
│   │        scanned) │     Semantic   Self-hosted  Milvus│
│   │                 │     chunking   BGE-M3      +Redis │
│   │            LlamaParse  by section             cache │
│   │                                                     │
│   └──→ Metadata extraction (date, type, client, area)   │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                  Query Pipeline                          │
│                                                         │
│  Query ──→ Rewrite ──→ Hybrid Search ──→ Rerank ──→ LLM│
│             │           │                │           │   │
│          Expand      Vector + BM25    BGE-Reranker  │   │
│          legal         top 50          top 5      Sonnet│
│          terms                                   + cite │
└─────────────────────────────────────────────────────────┘
```

### Key Decisions

**Chunking:** Semantic chunking by document section (headers, clauses). Legal documents have clear structure — exploit it. Chunk size: 512-1024 tokens with 100-token overlap. Store parent-child relationships (clause → section → document).

**Embedding:** Self-hosted BGE-M3 (1024 dims, multilingual). 4× A100 GPUs for serving. Batch embed during ingestion, real-time for queries.

**Vector DB:** Milvus — handles 100M+ vectors, supports metadata filtering, sharding built-in. 3-node cluster with replication.

**Hybrid Search:** Vector search (semantic) + BM25 (keyword) with α=0.6. Legal queries often contain exact phrases ("Force Majeure clause") that need keyword matching.

**Reranking:** BGE-Reranker-v2 on top 50 merged results → top 5. Cross-attention reranking is 10x more accurate than embedding similarity for legal text.

**Cost Estimate:**
```
Milvus cluster (3 nodes): ~$8K/month
GPU servers (4× A100):    ~$15K/month
LLM calls (Sonnet):       ~$12K/month (200K queries × $0.06)
OCR processing:           ~$3K/month
Storage (10TB):           ~$2K/month
Total:                    ~$40K/month ✓
```
