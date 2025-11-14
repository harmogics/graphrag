# Vector Search and Similarity

## Overview

Vector search transforms **semantic search** into **geometric nearest neighbor search** in high-dimensional embedding space. GraphRAG uses **cosine similarity** and **approximate nearest neighbor (ANN)** algorithms to retrieve the most semantically relevant entities and text units for a given query.

## Conceptual Foundation

### Similarity as Distance

**Core idea**: Semantic similarity = geometric proximity in embedding space

**Cosine similarity**:
```
similarity(v1, v2) = cos(θ) = (v1 · v2) / (||v1|| × ||v2||)

Range: [-1, 1]
  1.0: identical semantics (θ = 0°)
  0.0: orthogonal (θ = 90°, unrelated)
 -1.0: opposite (θ = 180°, rare)
```

**Why cosine over Euclidean distance?**
- **Scale invariant**: Only angle matters, not magnitude
- **Normalized embeddings**: OpenAI embeddings are unit vectors (||v|| = 1)
- **Intuitive**: cos(θ) directly measures semantic alignment

---

## Algorithms

### 1. Exact Cosine Similarity Search

**Algorithm** (conceptual):
```python
def cosine_similarity_search(query_embedding, corpus_embeddings, k=10):
    """Find top-k most similar vectors."""
    similarities = []
    for doc_id, doc_embedding in corpus_embeddings.items():
        sim = np.dot(query_embedding, doc_embedding)  # Dot product (vectors pre-normalized)
        similarities.append((doc_id, sim))

    # Sort by similarity (descending) and return top-k
    similarities.sort(key=lambda x: x[1], reverse=True)
    return similarities[:k]
```

**Complexity**: O(N × D)
- N = corpus size
- D = embedding dimensions (1536)

**Bottleneck**: Linear scan over entire corpus

**Example** (10K documents, 1536-dim):
```
Computation: 10K × 1536 dot products = 15.36M operations
Time: ~200ms (numpy vectorized)
```

**Limitation**: Doesn't scale to millions of documents

---

### 2. Approximate Nearest Neighbor (ANN)

**Problem**: Exact search is O(N) — too slow for large corpora

**Solution**: **ANN algorithms** trade accuracy for speed
- **HNSW** (Hierarchical Navigable Small World): Graph-based, O(log N)
- **IVF** (Inverted File Index): Clustering-based, O(√N)
- **Flat** (brute-force): Exact O(N), baseline

**Implementation**: Vector databases (LanceDB, Azure AI Search) handle ANN internally

---

### HNSW Algorithm (Azure AI Search)

**From** `azure_ai_search.py:84-99`:

```python
vector_search = VectorSearch(
    algorithms=[
        HnswAlgorithmConfiguration(
            name="HnswAlg",
            parameters=HnswParameters(
                metric=VectorSearchAlgorithmMetric.COSINE  # Cosine similarity
            ),
        )
    ],
    profiles=[
        VectorSearchProfile(
            name="vectorSearchProfile",
            algorithm_configuration_name="HnswAlg",
        )
    ],
)
```

**How HNSW works**:
1. **Build hierarchical graph** of vectors (indexing time)
2. **Navigate graph** from top layer (coarse) to bottom (fine) — query time
3. **Return approximate k-NN** in O(log N) time

**Trade-offs**:
- **Accuracy**: ~95-99% recall @ k=10 (vs. 100% for exact)
- **Speed**: 100-1000x faster than brute-force
- **Memory**: 2-3x embedding size (for graph structure)

**Tuning parameters** (HNSW):
```yaml
m: 16               # Number of connections per node (higher = better accuracy, more memory)
ef_construction: 200  # Build-time search breadth (higher = better index quality)
ef_search: 50       # Query-time search breadth (higher = better accuracy, slower)
```

---

## Integration with GraphRAG

### Entity Extraction via Vector Search

**Code**: `entity_extraction.py:37-79`

```python
def map_query_to_entities(
    query: str,
    text_embedding_vectorstore: BaseVectorStore,
    text_embedder: EmbeddingModel,
    all_entities_dict: dict[str, Entity],
    k: int = 10,
    oversample_scaler: int = 2,
) -> list[Entity]:
    """Extract entities matching query using semantic similarity."""
    if query != "":
        # Embed query
        query_embedding = text_embedder.embed(query)

        # ANN search in vector store
        search_results = text_embedding_vectorstore.similarity_search_by_vector(
            query_embedding,
            k=k * oversample_scaler,  # Retrieve 2× for filtering
        )

        # Map results to Entity objects
        matched_entities = [
            get_entity_by_id(all_entities_dict, result.document.id)
            for result in search_results
        ]
    else:
        # Fallback: rank by degree centrality (graph-based)
        matched_entities = sorted(all_entities, key=lambda x: x.rank, reverse=True)[:k]

    return matched_entities
```

**Flow**:
```
Query: "Microsoft AI partnerships"
   ↓
Embed → [0.23, -0.12, ..., 0.67]
   ↓
Vector search (k=10)
   ↓
Top results:
  1. Entity "OpenAI" (score=0.91): "AI research lab partnered with Microsoft..."
  2. Entity "Azure OpenAI" (score=0.88): "Microsoft's AI service offering..."
  3. Entity "GitHub Copilot" (score=0.85): "AI-powered code assistant..."
   ↓
Return entities for context building
```

**Oversample rationale**: Retrieve k=20, filter to k=10 after exclusions

---

### Text Unit Retrieval (Local Search)

**From** `local_search/search.py`:

```python
context_result = self.context_builder.build_context(
    query=query,
    conversation_history=conversation_history,
    **kwargs,
)

# context_builder internally:
# 1. Embed query
# 2. Search text_unit_embeddings (vector store)
# 3. Retrieve top-k text units
# 4. Concatenate as context for LLM
```

**Use case**: Find text chunks relevant to user question

---

## Vector Store Implementations

### BaseVectorStore Interface

**From** `base.py:40-86`:

```python
class BaseVectorStore(ABC):
    """Base class for vector storage."""

    @abstractmethod
    def similarity_search_by_vector(
        self, query_embedding: list[float], k: int = 10, **kwargs: Any
    ) -> list[VectorStoreSearchResult]:
        """Perform ANN search by vector."""

    @abstractmethod
    def similarity_search_by_text(
        self, text: str, text_embedder: TextEmbedder, k: int = 10, **kwargs: Any
    ) -> list[VectorStoreSearchResult]:
        """Perform ANN search by text (embed internally)."""

    @abstractmethod
    def filter_by_id(self, include_ids: list[str] | list[int]) -> Any:
        """Build query filter to filter documents by ID."""
```

**Implementations**:
1. **LanceDB** (`lancedb.py`): Embedded vector database (local files)
2. **Azure AI Search** (`azure_ai_search.py`): Cloud vector search service
3. **CosmosDB** (`cosmosdb.py`): Azure Cosmos DB with vector search

---

### LanceDB Search

**Code**: `lancedb.py:96-128`

```python
def similarity_search_by_vector(
    self, query_embedding: list[float], k: int = 10, **kwargs: Any
) -> list[VectorStoreSearchResult]:
    """Perform vector-based similarity search."""
    if self.query_filter:
        # Filtered search (e.g., specific document IDs)
        docs = (
            self.document_collection.search(
                query=query_embedding, vector_column_name="vector"
            )
            .where(self.query_filter, prefilter=True)  # Apply filter before ANN
            .limit(k)
            .to_list()
        )
    else:
        # Unfiltered search
        docs = (
            self.document_collection.search(query=query_embedding, vector_column_name="vector")
            .limit(k)
            .to_list()
        )

    # Convert distance to similarity score
    return [
        VectorStoreSearchResult(
            document=VectorStoreDocument(...),
            score=1 - abs(float(doc["_distance"])),  # Distance → similarity
        )
        for doc in docs
    ]
```

**Distance metric**: LanceDB returns `_distance` (lower = more similar)
- **L2 distance**: Euclidean distance in embedding space
- **Conversion**: `similarity = 1 - distance` (for cosine-like interpretation)

---

## Connection to Graph Algorithms

### Hybrid Retrieval: Text Embeddings + Graph Embeddings

**Concept**: Combine semantic similarity (text) with structural similarity (graph)

**From** `spec/graph/04-graph-embedding.md`:

```python
# Text embedding similarity
query_text_emb = embed_text("Microsoft partnerships")
entity_text_emb = embed_text(entity.description)
text_sim = cosine_similarity(query_text_emb, entity_text_emb)  # 0.85

# Graph embedding similarity (Node2Vec)
query_graph_emb = node2vec("MICROSOFT")  # Structural neighborhood
entity_graph_emb = node2vec(entity.id)
graph_sim = cosine_similarity(query_graph_emb, entity_graph_emb)  # 0.72

# Weighted combination
final_score = 0.7 * text_sim + 0.3 * graph_sim
            = 0.7 * 0.85 + 0.3 * 0.72
            = 0.595 + 0.216 = 0.811
```

**Use case**: Retrieve entities that are:
- Semantically related to query (text similarity)
- Structurally central/related in graph (graph similarity)

---

### Centrality-Weighted Ranking

**From** `spec/graph/02-centrality-and-ranking.md`:

**Combine vector search with graph centrality**:
```python
# Vector search results
vector_results = vector_store.similarity_search_by_vector(query_emb, k=100)

# Re-rank by combined score
for result in vector_results:
    entity = get_entity(result.document.id)
    text_score = result.score  # Cosine similarity
    graph_score = entity.degree / max_degree  # Normalized degree centrality

    result.combined_score = 0.6 * text_score + 0.4 * graph_score

# Return top-k by combined score
return sorted(vector_results, key=lambda x: x.combined_score, reverse=True)[:k]
```

**Benefit**: Surface entities that are both relevant AND important

---

## Performance Characteristics

### Search Complexity

| Algorithm | Time Complexity | Accuracy | Use Case |
|-----------|----------------|----------|----------|
| Brute-force (exact) | O(N × D) | 100% | Small corpora (< 100K) |
| HNSW | O(D × log N) | 95-99% | Large corpora (> 1M) |
| IVF | O(D × √N) | 90-95% | Medium corpora |

**Where**:
- N = corpus size
- D = embedding dimensions (1536)

**Example** (1M documents, 1536-dim, k=10):
```
Brute-force:  1M × 1536 × 4 bytes = 6.1 GB scan, ~3 seconds
HNSW:         log(1M) ≈ 20 hops, ~5ms
Speedup:      600x
```

---

### Recall Trade-offs

**Recall @ k**: Fraction of true top-k results returned by ANN

| ef_search | Recall @ 10 | Query Time | Memory |
|-----------|-------------|------------|--------|
| 10 | 85% | 2ms | 1x |
| 50 | 95% | 5ms | 1x |
| 100 | 98% | 10ms | 1x |
| 500 (exact) | 100% | 50ms | 1x |

**Recommendation**: ef_search=50 (95% recall, 5ms latency)

---

## Connection to Research Patterns

### Query-as-Key Paradigm

**From** `spec/research/01-query-as-key.md`:

**Core insight**: Query embedding is the "key" unlocking relevant knowledge

**Retrieval as geometric search**:
```
Query vector (key): [0.23, -0.12, ..., 0.67]
    ↓
Search embedding space (lock)
    ↓
Top-k nearest neighbors (opened doors):
  - Entity 1 (0.91 similarity)
  - Entity 2 (0.88 similarity)
  - ...
    ↓
Context for answer generation
```

**Vector search realizes this paradigm** through ANN algorithms

---

## Troubleshooting

### Problem: Low Search Accuracy

**Symptom**: Irrelevant results returned for queries

**Diagnosis**:
- ANN recall too low (ef_search too small)
- Embeddings not normalized
- Wrong similarity metric

**Solution**:
1. **Increase ef_search**:
   ```yaml
   vector_store:
     ef_search: 100  # Up from 50
   ```

2. **Verify embedding normalization**:
   ```python
   assert 0.99 < np.linalg.norm(embedding) < 1.01
   ```

3. **Use cosine metric** (not L2 for normalized vectors)

---

### Problem: Slow Search Performance

**Symptom**: Query latency > 100ms

**Diagnosis**: Large corpus, suboptimal indexing

**Solution**:
1. **Reduce ef_search** (trade accuracy for speed):
   ```yaml
   ef_search: 30  # Down from 50
   ```

2. **Pre-filter** (search subset):
   ```python
   vector_store.filter_by_id(relevant_entity_ids)  # Search only filtered subset
   vector_store.similarity_search_by_vector(query_emb, k=10)
   ```

3. **Use GPU-accelerated vector stores** (e.g., Faiss with CUDA)

---

## Future Enhancements

### 1. Multi-Vector Search

**Current**: Single query embedding

**Proposed**: Multiple query aspects
```python
query = "Microsoft's AI partnerships and products"
aspects = [
    embed_text("Microsoft partnerships"),    # Aspect 1
    embed_text("AI products"),               # Aspect 2
]

results = []
for aspect_emb in aspects:
    results.extend(vector_store.search(aspect_emb, k=5))

# De-duplicate and merge
final_results = merge_and_deduplicate(results, k=10)
```

**Benefit**: Capture multi-faceted queries

---

### 2. Learned Similarity Metrics

**Current**: Cosine similarity (fixed)

**Proposed**: Trainable similarity
```python
def learned_similarity(query_emb, doc_emb, weights):
    """Learned weighted similarity."""
    return np.dot(query_emb * weights, doc_emb)

# Train weights on user feedback (click-through data)
```

**Benefit**: Domain-specific relevance

---

## Conclusion

Vector search transforms semantic retrieval into geometric proximity calculation in embedding space. By using **cosine similarity** and **ANN algorithms** (HNSW, IVF), GraphRAG achieves:
- **Semantic precision**: Retrieve entities/texts by meaning, not keywords
- **Scalability**: O(log N) search via HNSW (1000x faster than brute-force)
- **Hybrid retrieval**: Combine text similarity with graph centrality

**Key principles**:
- **Cosine similarity**: Measure semantic alignment via vector angle
- **ANN for scale**: Trade 1-5% accuracy for 100-1000x speedup
- **Oversample and filter**: Retrieve k× results, filter to k
- **Integration with graph**: Combine vector similarity with structural importance

**Key files**:
- `base.py:40-86` — VectorStore interface
- `lancedb.py:96-128` — LanceDB ANN search implementation
- `entity_extraction.py:37-79` — Entity retrieval via vector search
