# Vector Algorithms in GraphRAG

## Overview

This directory documents the **vector algorithms** that enable GraphRAG's semantic retrieval capabilities. These algorithms transform discrete text into continuous embeddings, enabling geometric similarity search that powers entity extraction, context building, and hybrid retrieval.

## Vector Algorithm Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│                      VECTOR ALGORITHM PIPELINE                        │
└──────────────────────────────────────────────────────────────────────┘

INPUT: Raw Documents
   │
   ▼
┌────────────────────────────────────────────┐
│  02-CHUNKING STRATEGIES                    │  ← Text Segmentation
│  - Token-based chunking (tiktoken)         │
│  - Sentence-based chunking (NLTK)          │
│  - Sliding window with overlap             │
│  - Multi-document streaming                │
└────────────────────────────────────────────┘
   │
   ▼ text_units = [{id, text, source_doc_ids}]
   │
┌────────────────────────────────────────────┐
│  01-TEXT EMBEDDING                         │  ← Semantic Encoding
│  - Neural embeddings (OpenAI, Azure)       │
│  - Batch processing (16/batch)             │
│  - Parallel API calls (4 concurrent)       │
│  - Split-average-normalize long texts      │
└────────────────────────────────────────────┘
   │
   ▼ embeddings = [vector_1536dim, ...]
   │
┌────────────────────────────────────────────┐
│  04-VECTOR STORAGE & INDEXING              │  ← Persistent Storage
│  - LanceDB (embedded, Apache Arrow)        │
│  - Azure AI Search (cloud, HNSW)           │
│  - Automatic ANN index construction        │
│  - Metadata storage (graph attributes)     │
└────────────────────────────────────────────┘
   │
   ▼ indexed_embeddings → vector_store
   │
   ├─────────────── QUERY TIME ───────────────┐
   │                                           │
   ▼                                           ▼
┌──────────────────┐                  ┌────────────────┐
│ Query Embedding  │                  │ Graph Context  │
│ (same pipeline)  │                  │ (centrality,   │
└──────────────────┘                  │  communities)  │
   │                                   └────────────────┘
   ▼                                           │
┌────────────────────────────────────────────┐│
│  03-VECTOR SEARCH & SIMILARITY             ││
│  - Cosine similarity (dot product)         ││
│  - ANN search (HNSW: O(log N))            ││
│  - Top-k retrieval (k=10-100)             ││
│  - Hybrid scoring (text + graph)          ││
└────────────────────────────────────────────┘│
   │                                           │
   └────────────── + ─────────────────────────┘
   │
   ▼
Retrieved Context → LLM Answer Generation
```

---

## Algorithm Documents

### [01-text-embedding.md](01-text-embedding.md)
**Neural Text Embeddings** — Transform text to 1536-dim semantic vectors

**Purpose**: Map text to continuous space where similarity = geometric proximity

**Key concepts**:
- OpenAI/Azure text-embedding-ada-002 models
- Preprocessing: tokenization, splitting (8191 token limit)
- Batching: 16 texts per API request
- Parallelization: 4 concurrent requests
- Reconstruction: average + normalize for long texts
- Caching: MD5-keyed cache for cost optimization

**Use cases**:
- Entity description embedding (semantic entity search)
- Text unit embedding (context retrieval)
- Query embedding (question → vector)
- Community report embedding (navigate summaries)

**Parameters**: `batch_size=16`, `batch_max_tokens=8191`, `num_threads=4`

**Complexity**: O(B × T_api) where B = batches, T_api = API latency (~500ms)

---

### [02-chunking-strategies.md](02-chunking-strategies.md)
**Text Chunking** — Decompose long documents into token-constrained segments

**Purpose**: Enable parallel LLM processing and fit embedding model limits

**Key concepts**:
- Token-based: Sliding window (1200 tokens, 100 overlap)
- Sentence-based: NLTK sentence tokenization
- tiktoken: OpenAI-compatible tokenizer
- Overlap strategy: Preserve context at boundaries (reduce entity loss)
- Multi-document: Track source document IDs

**Use cases**:
- Pre-processing for entity extraction
- Text unit creation for retrieval
- Context window management

**Parameters**: `size=1200`, `overlap=100`, `encoding_model=cl100k_base`

**Complexity**: O(N × L) where N = texts, L = avg length

**Trade-offs**: Chunk size (200-1200 tokens), overlap (0-20%)

---

### [03-vector-search-and-similarity.md](03-vector-search-and-similarity.md)
**Similarity Search** — Geometric nearest neighbor search in embedding space

**Purpose**: Retrieve semantically relevant entities/texts for queries

**Key concepts**:
- Cosine similarity: `cos(θ) = (v1 · v2) / (||v1|| × ||v2||)`
- ANN algorithms: HNSW (O(log N)), IVF (O(√N))
- Exact search: O(N × D) brute-force baseline
- Oversample and filter: Retrieve k×2, filter to k
- Hybrid retrieval: Combine text similarity + graph centrality

**Use cases**:
- Entity extraction (map query to entities)
- Text unit retrieval (find relevant chunks)
- Hybrid scoring (text + graph signals)

**Complexity**: O(D × log N) for HNSW, O(N × D) for exact

**Performance**: 100-1000x speedup via ANN (95-99% recall)

---

### [04-vector-storage-and-indexing.md](04-vector-storage-and-indexing.md)
**Vector Databases** — Persistent storage with fast similarity search

**Purpose**: Scalable, efficient storage for millions of embeddings

**Key concepts**:
- LanceDB: Embedded, Apache Arrow, local files
- Azure AI Search: Cloud, HNSW, managed service
- VectorStoreDocument: `{id, text, vector, attributes}`
- HNSW indexing: Multi-layer navigable graph
- Batched ingestion: 500 docs/batch

**Use cases**:
- Index storage (entities, text units, communities)
- Fast k-NN queries (sub-millisecond latency)
- Metadata storage (graph attributes: degree, community_id)

**Storage**: 61.4 MB per 10K vectors (1536-dim), ~2x with HNSW index

**Performance**: 2K docs/sec ingestion, 5-50ms search latency

---

## Algorithm Interaction Map

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ALGORITHM DEPENDENCIES                            │
└─────────────────────────────────────────────────────────────────────┘

CHUNKING (02)
   ├─ Input: Raw documents
   ├─ Output: text_units = [{text, source_doc_ids}]
   │
   └──▶ EMBEDDING (01)
         ├─ Input: text_units.text
         ├─ Process: OpenAI API → embeddings
         ├─ Output: vectors (1536-dim)
         │
         └──▶ STORAGE (04)
               ├─ Input: vectors + metadata
               ├─ Process: HNSW index construction
               ├─ Output: queryable vector store
               │
               └──▶ SEARCH (03)
                     ├─ Input: query_vector
                     ├─ Process: ANN search (HNSW)
                     └─ Output: top-k similar documents
```

**Critical path**:
1. **Chunking** → Must happen first (create processable units)
2. **Embedding** → API bottleneck (500ms per batch)
3. **Storage** → Async (write while embedding)
4. **Search** → Query-time only (fast, < 50ms)

---

## Cross-References to Other Specs

### To `spec/graph/` (Graph Algorithms)

**[spec/graph/03-graph-construction-and-merging.md](../graph/03-graph-construction-and-merging.md)**:
- **Chunking enables map-reduce**: Extract entities per chunk (map) → merge (reduce)
- **Text units as provenance**: Track which chunks contain each entity

**[spec/graph/04-graph-embedding.md](../graph/04-graph-embedding.md)**:
- **Two embedding spaces**: Text (semantic) vs. Graph (structural)
- **Hybrid retrieval**: Combine cosine similarity from both spaces
- **Complementary signals**: Text = meaning, Graph = importance

**[spec/graph/02-centrality-and-ranking.md](../graph/02-centrality-and-ranking.md)**:
- **Hybrid scoring**: `α × text_similarity + β × degree_centrality`
- **Entity ranking**: Combine semantic match with structural importance

---

### To `spec/research/` (Conceptual Foundations)

**[spec/research/01-query-as-key.md](../research/01-query-as-key.md)**:
- **Query embedding = key**: Unlocks relevant context via similarity search
- **Retrieval as geometric search**: Nearest neighbors in embedding space

**[spec/research/02-star-attractor-patterns.md](../research/02-star-attractor-patterns.md)**:
- **Attractor embedding**: High-centrality entities cluster in embedding space
- **Semantic basins**: Related entities form dense regions

---

### To `spec/dependencies/` (Implementation)

**[spec/dependencies/01-llm-and-language-processing.md](../dependencies/01-llm-and-language-processing.md)**:
- **fnllm**: Unified LLM interface (OpenAI, Azure)
- **tiktoken**: Fast tokenization for chunking and token counting
- **OpenAI API**: text-embedding-ada-002 model

**[spec/dependencies/02-graph-and-vector-storage.md](../dependencies/02-graph-and-vector-storage.md)**:
- **lancedb**: Embedded vector database
- **Azure AI Search**: Cloud vector search service

**[spec/dependencies/03-data-processing-and-infrastructure.md](../dependencies/03-data-processing-and-infrastructure.md)**:
- **numpy**: Vector operations (dot product, normalization)
- **pandas**: DataFrame processing for batch embedding
- **pyarrow**: Columnar storage for LanceDB

---

### To `spec/semlang/` (Semantic Flow Language)

**[spec/semlang/01-document-transformation.md](../semlang/01-document-transformation.md)**:
- Chunking is a **TRANSFORM** operation (documents → text_units)
- Embedding is a **TRANSFORM** operation (text → vectors)

---

## Decision Trees for Practitioners

### Which Chunking Strategy?

```
START: Choose chunking strategy
   │
   ├─ Is corpus already short texts (tweets, sentences)?
   │  │
   │  └─▶ NO chunking needed (process directly)
   │
   ├─ Do you need precise entity provenance (which sentence)?
   │  │
   │  └─▶ YES: Use sentence-based chunking (NLTK)
   │
   ├─ Standard use case (long documents)?
   │  │
   │  └─▶ YES: Use token-based chunking (default)
   │         Parameters: size=1200, overlap=100
   │
   └─ Entity extraction quality issues?
      │
      └─▶ Increase overlap to 200 (13% overlap)
          OR increase size to 1500 tokens
```

---

### Which Vector Store?

```
START: Choose vector storage
   │
   ├─ Development/prototyping?
   │  │
   │  └─▶ YES: LanceDB (embedded, no setup)
   │
   ├─ Corpus size < 10M vectors?
   │  │
   │  └─▶ YES: LanceDB (fits in memory)
   │
   ├─ Production deployment with > 10M vectors?
   │  │
   │  └─▶ YES: Azure AI Search (managed, scalable)
   │
   ├─ Need global distribution?
   │  │
   │  └─▶ YES: Azure AI Search (multi-region)
   │
   └─ Hybrid document + vector storage?
      │
      └─▶ YES: Cosmos DB (NoSQL + vector)
```

---

### Optimizing Search Performance

```
START: Search latency issues?
   │
   ├─ Latency > 100ms for small corpus (< 100K)?
   │  │
   │  └─▶ Check: Is ANN index enabled? (should be automatic)
   │         Solution: Verify vector_column_name in search query
   │
   ├─ Latency > 50ms for large corpus (> 1M)?
   │  │
   │  └─▶ Reduce ef_search parameter:
   │         ef_search: 30 (faster, 90% recall)
   │         vs. ef_search: 100 (slower, 98% recall)
   │
   ├─ Accuracy issues (irrelevant results)?
   │  │
   │  └─▶ Increase ef_search: 100
   │         OR increase k and oversample: k=20 → filter to 10
   │
   └─ Memory issues?
      │
      └─▶ Switch from LanceDB to cloud (Azure AI Search)
          OR shard data into multiple tables
```

---

## Performance Benchmarks

### Typical GraphRAG Corpus (10K Documents, 100K Text Units)

| Algorithm | Input | Output | Time | Memory | Bottleneck |
|-----------|-------|--------|------|--------|------------|
| Chunking | 10K docs | 100K chunks | 30 sec | 500 MB | Tokenization |
| Embedding | 100K chunks | 100K vectors | 8 min | 1 GB | OpenAI API |
| Storage (LanceDB) | 100K vectors | Index | 15 sec | 650 MB | Disk I/O |
| Search (k=10) | Query | Results | 5 ms | 650 MB | ANN traversal |

**Total indexing time**: ~9 minutes (embedding dominates)

**Query-time performance**: < 10ms per search

---

### Scaling Characteristics

| Corpus Size | Chunking | Embedding | Storage | Search (HNSW) |
|-------------|----------|-----------|---------|---------------|
| 1K docs | 3 sec | 1 min | 2 sec | 2 ms |
| 10K docs | 30 sec | 8 min | 15 sec | 5 ms |
| 100K docs | 5 min | 80 min | 2 min | 10 ms |
| 1M docs | 50 min | 800 min* | 20 min | 20 ms |

\* Parallelizable with more API quota

---

## Common Issues and Solutions

### Issue: Embedding API Rate Limits

**Symptom**: `RateLimitError` from OpenAI/Azure

**Solution**:
1. Reduce `num_threads` (concurrent requests): `num_threads: 2`
2. Increase `batch_size` (fewer total requests): `batch_size: 16`
3. Add `max_retries` with backoff: `max_retries: 10`

---

### Issue: Poor Retrieval Quality

**Symptom**: Irrelevant entities/texts returned

**Diagnosis**:
- Embeddings not normalized
- Chunk size too small (< 200 tokens)
- ANN recall too low

**Solution**:
1. Verify normalization: `np.linalg.norm(embedding) ≈ 1.0`
2. Increase chunk size: `size: 1500`
3. Increase ef_search: `ef_search: 100`
4. Use hybrid scoring: Combine text + graph centrality

---

### Issue: High Memory Usage

**Symptom**: OOM during indexing or search

**Solution**:
1. Stream to disk (don't accumulate in memory):
   ```python
   for batch in chunks:
       embed_and_store(batch)  # Process and discard
   ```

2. Switch to cloud storage (Azure AI Search)

3. Reduce embedding dimensions (if model supports):
   ```yaml
   dimensions: 512  # Down from 1536
   ```

---

## Future Enhancements

### 1. Matryoshka Embeddings (Adaptive Dimensions)

**Current**: Fixed 1536-dim

**Proposed**: Variable dimensions based on precision needs
```python
# High-precision search
embed_text(text, dimensions=1536)

# Fast approximate search (6x memory savings)
embed_text(text, dimensions=256)
```

---

### 2. Semantic Chunking

**Current**: Fixed token boundaries

**Proposed**: Chunk at semantic boundaries (paragraphs, topics)
- Preserve semantic coherence
- Improve entity extraction quality
- Reduce boundary loss

---

### 3. Multi-Vector Search

**Current**: Single query vector

**Proposed**: Multiple query aspects
```python
aspects = [
    embed_text("Microsoft partnerships"),
    embed_text("AI products"),
]
results = multi_aspect_search(aspects, k=10)
```

---

### 4. Domain-Specific Fine-Tuning

**Current**: Generic OpenAI embeddings

**Proposed**: Fine-tune on corpus-specific data
- Collect positive/negative pairs
- Fine-tune embedding model
- 10-30% accuracy improvement

---

## Getting Started

### Minimal Pipeline (Indexing)

```yaml
workflows:
  - name: create_embeddings
    steps:
      # 1. Chunk documents
      - verb: chunk_text
        args:
          size: 1200
          overlap: 100
          strategy: tokens

      # 2. Embed chunks
      - verb: embed_text
        args:
          strategy:
            type: openai
            llm:
              model: text-embedding-ada-002
              batch_size: 16
            vector_store:
              type: lancedb
              db_uri: ./lancedb
```

**Result**: Embeddings stored in LanceDB, ready for search

---

### Query Pipeline (Retrieval)

```python
from graphrag.query import LocalSearch

# Initialize search
search = LocalSearch(
    model=llm,
    context_builder=context_builder,  # Uses vector store internally
)

# Query
result = await search.search("What are Microsoft's AI partnerships?")

# result.response: LLM-generated answer
# result.context: Retrieved text units + entities (via vector search)
```

---

## Conclusion

GraphRAG's vector algorithms transform discrete text into continuous semantic representations, enabling:
- **Semantic retrieval**: Find relevant content by meaning, not keywords
- **Scalable search**: O(log N) queries via ANN (1000x faster than brute-force)
- **Hybrid intelligence**: Combine text semantics with graph structure
- **Production-ready**: Embedded (LanceDB) or cloud (Azure) deployment options

**Key algorithms**:
- **Chunking**: Decompose documents → token-constrained segments
- **Embedding**: Text → 1536-dim vectors via OpenAI models
- **Storage**: Persistent vector DB with HNSW indexes
- **Search**: Cosine similarity + ANN for sub-millisecond retrieval

**Integration**:
- Works with graph algorithms for hybrid retrieval
- Enables query-as-key paradigm for context building
- Powers local search, entity extraction, community navigation

**Key files in this directory**:
- `01-text-embedding.md` — Neural embeddings via OpenAI/Azure
- `02-chunking-strategies.md` — Token and sentence-based segmentation
- `03-vector-search-and-similarity.md` — Cosine similarity and ANN
- `04-vector-storage-and-indexing.md` — LanceDB, Azure AI Search

**Next steps**:
- Read individual algorithm docs for deep dives
- Consult decision trees for tuning guidance
- Cross-reference `spec/graph`, `spec/research`, `spec/dependencies` for system context
