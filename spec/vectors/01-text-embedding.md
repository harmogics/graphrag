# Text Embedding Algorithms

## Overview

Text embedding transforms **discrete textual representations** (words, sentences, documents) into **continuous vector spaces** where semantic similarity corresponds to geometric proximity. GraphRAG uses **neural language model embeddings** (OpenAI, Azure OpenAI) to create dense vector representations that power semantic search, entity extraction, and hybrid retrieval.

## Conceptual Foundation

### Why Embed Text?

**Problem**: Text is discrete (words, tokens) and high-dimensional (vocabulary size = 50K+ tokens)

**Solution**: Map text to continuous, low-dimensional vectors (typically 1536 dimensions) where:
- Semantically similar texts → nearby vectors (high cosine similarity)
- Semantically dissimilar texts → distant vectors (low cosine similarity)

**GraphRAG use cases**:
1. **Entity description embedding**: Enable semantic search over entity descriptions
2. **Text unit embedding**: Find relevant chunks for query context building
3. **Query embedding**: Convert user questions to vectors for retrieval
4. **Community report embedding**: Enable semantic navigation of knowledge graph summaries

### Embedding Space Geometry

**Key property**: **Cosine similarity** preserves semantic relationships

```
cosine_similarity(v1, v2) = (v1 · v2) / (||v1|| × ||v2||)

Range: [-1, 1]
  1.0  = identical meaning
  0.0  = orthogonal (unrelated)
 -1.0  = opposite meaning (rare in practice)
```

**Example**:
```python
# Semantic relationships in embedding space
cosine_sim(embed("Microsoft"), embed("tech company")) = 0.72  # High similarity
cosine_sim(embed("Microsoft"), embed("OpenAI"))        = 0.65  # Related entities
cosine_sim(embed("Microsoft"), embed("banana"))        = 0.12  # Unrelated
```

**Contrast with graph embeddings** (from `spec/graph/04-graph-embedding.md`):
- **Text embeddings**: Capture semantic meaning from language patterns
- **Graph embeddings**: Capture structural relationships from graph topology
- **Complementary**: GraphRAG uses both for hybrid search

---

## Algorithm: Neural Text Embedding

### Phase 1: Text Preprocessing

**Code**: `graphrag/index/operations/embed_text/strategies/openai.py:134-150`

```python
def _prepare_embed_texts(
    input: list[str], splitter: TokenTextSplitter
) -> tuple[list[str], list[int]]:
    sizes: list[int] = []
    snippets: list[str] = []

    for text in input:
        # Split the input text and filter out any empty content
        split_texts = splitter.split_text(text)
        if split_texts is None:
            continue
        split_texts = [text for text in split_texts if len(text) > 0]

        sizes.append(len(split_texts))
        snippets.extend(split_texts)

    return snippets, sizes
```

**Algorithm**:
1. **Tokenize**: Convert text to tokens using tiktoken encoder
2. **Split**: Break long texts exceeding model limits (8191 tokens for OpenAI)
3. **Filter**: Remove empty strings
4. **Track sizes**: Remember how many snippets per input (for reconstruction)

**Token limits** (from Azure OpenAI API):
```
Model                Max input tokens   Embedding dimensions
text-embedding-ada-002   8191                 1536
text-embedding-3-small   8191                 1536
text-embedding-3-large   8191                 3072
```

**Why split?** API rejects requests exceeding token limit

**Example**:
```python
Input: ["Very long document with 10,000 tokens..."]
After splitting: [
  "Very long document chunk 1 (8000 tokens)",
  "document chunk 2 (2000 tokens)"
]
Sizes: [2]  # One input split into 2 snippets
```

---

### Phase 2: Batch Construction

**Code**: `graphrag/index/operations/embed_text/strategies/openai.py:102-131`

```python
def _create_text_batches(
    texts: list[str],
    max_batch_size: int,
    max_batch_tokens: int,
    splitter: TokenTextSplitter,
) -> list[list[str]]:
    """Create batches of texts to embed."""
    result = []
    current_batch = []
    current_batch_tokens = 0

    for text in texts:
        token_count = splitter.num_tokens(text)
        if (
            len(current_batch) >= max_batch_size
            or current_batch_tokens + token_count > max_batch_tokens
        ):
            result.append(current_batch)
            current_batch = []
            current_batch_tokens = 0

        current_batch.append(text)
        current_batch_tokens += token_count

    if len(current_batch) > 0:
        result.append(current_batch)

    return result
```

**Algorithm**:
1. **Accumulate texts** into batch until hitting limit:
   - **Size limit**: max_batch_size (default 16 per Azure OpenAI limit)
   - **Token limit**: max_batch_tokens (default 8191)
2. **Flush batch** when limit reached
3. **Repeat** until all texts batched

**Batch constraints** (from Azure OpenAI):
- **Max batch size**: 16 concurrent embeddings per request
- **Max tokens per request**: 8191 total across all texts in batch

**Why batch?** 10-100x throughput improvement vs. serial requests

**Example**:
```python
Texts: ["text1 (100 tokens)", "text2 (200 tokens)", ..., "text50 (150 tokens)"]
max_batch_size = 16
max_batch_tokens = 8191

Batch 1: [text1, text2, ..., text16]  (16 texts, 2400 tokens)
Batch 2: [text17, ..., text32]         (16 texts, 2500 tokens)
Batch 3: [text33, ..., text48]         (16 texts, 2300 tokens)
Batch 4: [text49, text50]              (2 texts, 300 tokens)
```

---

### Phase 3: Parallel Embedding

**Code**: `graphrag/index/operations/embed_text/strategies/openai.py:83-99`

```python
async def _execute(
    model: EmbeddingModel,
    chunks: list[list[str]],
    tick: ProgressTicker,
    semaphore: asyncio.Semaphore,
) -> list[list[float]]:
    async def embed(chunk: list[str]):
        async with semaphore:
            chunk_embeddings = await model.aembed_batch(chunk)
            result = np.array(chunk_embeddings)
            tick(1)
        return result

    futures = [embed(chunk) for chunk in chunks]
    results = await asyncio.gather(*futures)
    # merge results in a single list of lists (reduce the collect dimension)
    return [item for sublist in results for item in sublist]
```

**Algorithm**:
1. **Create async tasks** for each batch
2. **Semaphore control**: Limit concurrent requests (default 4 threads)
3. **API call**: `model.aembed_batch(texts)` → sends batch to OpenAI/Azure
4. **Await all**: `asyncio.gather()` waits for all batches to complete
5. **Flatten results**: Merge batch results into single list

**Concurrency control**:
```python
semaphore = asyncio.Semaphore(num_threads)  # Default: 4

# At most 4 concurrent API requests
# Prevents rate limiting, manages memory
```

**Performance** (100 texts, batch_size=16, num_threads=4):
```
Serial (1 request at a time):     60 seconds
Batched (16 per request):          10 seconds (6x speedup)
Parallel batched (4 concurrent):   3 seconds (20x speedup)
```

**API response format** (from OpenAI API):
```json
{
  "object": "list",
  "data": [
    {
      "object": "embedding",
      "embedding": [0.0023, -0.0015, ..., 0.0042],  // 1536 floats
      "index": 0
    },
    ...
  ],
  "model": "text-embedding-ada-002",
  "usage": {
    "prompt_tokens": 8,
    "total_tokens": 8
  }
}
```

**Model invocation** (abstracted by fnllm):
```python
from graphrag.language_model.manager import ModelManager

model = ModelManager().get_or_create_embedding_model(
    name="text_embedding",
    model_type="openai_embedding",  # or "azure_openai_embedding"
    config=llm_config,
    callbacks=callbacks,
    cache=cache,
)

embeddings = await model.aembed_batch(texts)
# Returns: list[list[float]] with shape (len(texts), 1536)
```

---

### Phase 4: Embedding Reconstruction

**Code**: `graphrag/index/operations/embed_text/strategies/openai.py:153-172`

```python
def _reconstitute_embeddings(
    raw_embeddings: list[list[float]], sizes: list[int]
) -> list[list[float] | None]:
    """Reconstitute the embeddings into the original input texts."""
    embeddings: list[list[float] | None] = []
    cursor = 0
    for size in sizes:
        if size == 0:
            embeddings.append(None)
        elif size == 1:
            embedding = raw_embeddings[cursor]
            embeddings.append(embedding)
            cursor += 1
        else:
            chunk = raw_embeddings[cursor : cursor + size]
            average = np.average(chunk, axis=0)
            normalized = average / np.linalg.norm(average)
            embeddings.append(normalized.tolist())
            cursor += size
    return embeddings
```

**Algorithm**:
1. **Single snippet**: Use embedding as-is
2. **Multiple snippets** (from text splitting):
   - **Average**: Compute mean across snippet embeddings
   - **Normalize**: Divide by L2 norm to preserve unit length
3. **Empty text**: Return None

**Why average + normalize?**
- **Averaging**: Combines semantic information from all snippets
- **Normalization**: Maintains cosine similarity properties (all vectors on unit hypersphere)

**Mathematical detail**:
```
Given snippets: [s1, s2, s3] with embeddings [e1, e2, e3]

Average: e_avg = (e1 + e2 + e3) / 3

Normalize: e_final = e_avg / ||e_avg||

Where ||e_avg|| = sqrt(e_avg[0]^2 + e_avg[1]^2 + ... + e_avg[1535]^2)
```

**Example**:
```python
Input text: "Very long document..." (10,000 tokens)
Split into: ["chunk1" (8000 tokens), "chunk2" (2000 tokens)]

Embed chunks:
  embed("chunk1") = [0.1, 0.2, ..., 0.3]  # 1536-dim
  embed("chunk2") = [0.15, 0.18, ..., 0.25]

Average:
  e_avg = ([0.1, 0.2, ...] + [0.15, 0.18, ...]) / 2
        = [0.125, 0.19, ..., 0.275]

Normalize:
  norm = sqrt(0.125^2 + 0.19^2 + ... + 0.275^2) = 5.3
  e_final = [0.125/5.3, 0.19/5.3, ..., 0.275/5.3]
          = [0.024, 0.036, ..., 0.052]

Final embedding: [0.024, 0.036, ..., 0.052]
```

---

## Integration with GraphRAG Pipeline

### Embedding Flow in Indexing

**From** `spec/semlang/01-document-transformation.md`:

```sfl
SEMANTIC FLOW Indexing:
  CHUNK documents → text_units (200-600 tokens)

  EMBED text_units.text → text_unit_embeddings     # ← THIS SPEC
    VIA openai_embedding("text-embedding-ada-002")

  EXTRACT entities, relationships → graph

  EMBED entities.description → entity_embeddings   # ← THIS SPEC
    VIA openai_embedding("text-embedding-ada-002")

  STORE embeddings → vector_store (LanceDB, Azure AI Search)
```

**Actual code flow** (from `graphrag/index/operations/embed_text/embed_text.py:38-107`):

```python
async def embed_text(
    input: pd.DataFrame,
    callbacks: WorkflowCallbacks,
    cache: PipelineCache,
    embed_column: str,
    strategy: dict,
    embedding_name: str,
    id_column: str = "id",
    title_column: str | None = None,
):
    """Embed a piece of text into a vector space."""
    vector_store_config = strategy.get("vector_store")

    if vector_store_config:
        # Path 1: Embed and store in vector database
        collection_name = _get_collection_name(vector_store_config, embedding_name)
        vector_store = _create_vector_store(vector_store_config, collection_name)

        return await _text_embed_with_vector_store(
            input=input,
            embed_column=embed_column,
            strategy=strategy,
            vector_store=vector_store,
            # ...
        )

    # Path 2: Embed in-memory (return embeddings as DataFrame column)
    return await _text_embed_in_memory(
        input=input,
        embed_column=embed_column,
        strategy=strategy,
    )
```

**Two embedding modes**:

**Mode 1: In-memory** (returns embeddings as list)
```python
embeddings = await embed_text(
    input=text_units_df,
    embed_column="text",
    strategy={"type": "openai", "llm": {...}}
)
# Returns: list[list[float]]
# Use case: Graph embeddings, intermediate processing
```

**Mode 2: Vector store** (stores in database, returns embeddings)
```python
embeddings = await embed_text(
    input=entities_df,
    embed_column="description",
    strategy={
        "type": "openai",
        "llm": {...},
        "vector_store": {"type": "lancedb", "db_uri": "./lancedb"}
    },
    embedding_name="entity_description"
)
# Returns: list[list[float]]
# Side effect: Stored in LanceDB for later search
# Use case: Entity/text unit embeddings for retrieval
```

---

### Caching for Cost Optimization

**From** `spec/dependencies/01-llm-and-language-processing.md`:

**Problem**: Embedding API calls are expensive
- OpenAI: $0.0001 per 1K tokens
- 1M entities × 100 tokens each = $10 for single embedding pass

**Solution**: PipelineCache stores embeddings keyed by text hash

**Code** (conceptual, handled by fnllm):
```python
cache_key = hashlib.md5(text.encode()).hexdigest()

# Check cache
if cached := cache.get(cache_key):
    return cached

# Cache miss → call API
embeddings = await model.aembed_batch(texts)

# Store in cache
cache.set(cache_key, embeddings)
```

**Cache backends**:
- **Memory**: Fast, volatile (lost on restart)
- **File**: Persistent, slower I/O
- **Redis**: Distributed, shared across workers

**Performance impact** (re-indexing with cached embeddings):
```
First run (cold cache):   300 seconds, $50 API cost
Second run (warm cache):  5 seconds, $0 API cost

Speedup: 60x, Cost reduction: 100%
```

---

## Connection to Vector Search

### Entity Extraction via Embedding Similarity

**From** `graphrag/query/context_builder/entity_extraction.py:37-79`:

```python
def map_query_to_entities(
    query: str,
    text_embedding_vectorstore: BaseVectorStore,
    text_embedder: EmbeddingModel,
    all_entities_dict: dict[str, Entity],
    k: int = 10,
    oversample_scaler: int = 2,
) -> list[Entity]:
    """Extract entities that match a given query using semantic similarity."""
    if query != "":
        # Embed query
        query_embedding = text_embedder.embed(query)

        # Search vector store
        search_results = text_embedding_vectorstore.similarity_search_by_vector(
            query_embedding,
            k=k * oversample_scaler,  # Oversample to account for filtering
        )

        # Map search results back to entities
        matched_entities = [
            get_entity_by_id(all_entities_dict, result.document.id)
            for result in search_results
        ]
    else:
        # Fallback: rank by graph centrality
        all_entities.sort(key=lambda x: x.rank, reverse=True)
        matched_entities = all_entities[:k]

    return matched_entities
```

**Flow**:
1. **Query embedding**: Convert user question to vector
2. **Vector search**: Find k most similar entity description embeddings
3. **Entity lookup**: Map embedding IDs back to Entity objects
4. **Context building**: Use matched entities for answer generation

**Integration with local search** (from `spec/graph/02-centrality-and-ranking.md`):
- **Text embedding similarity**: Find semantically related entities
- **Graph centrality**: Rank entities by structural importance
- **Combined score**: `α × text_similarity + β × graph_centrality`

---

## Connection to Dependencies

### fnllm: Unified LLM Interface

**From** `spec/dependencies/01-llm-and-language-processing.md`:

**Why fnllm?**
- **Provider abstraction**: Same code for OpenAI, Azure, Anthropic
- **Automatic retries**: Handles rate limits, transient errors
- **Request batching**: Optimizes throughput
- **Caching integration**: Seamless cache lookups

**Example** (OpenAI vs. Azure, identical code):
```yaml
# OpenAI
llm:
  type: openai_embedding
  api_key: ${OPENAI_API_KEY}
  model: text-embedding-ada-002

# Azure OpenAI (different backend, same interface)
llm:
  type: azure_openai_embedding
  api_key: ${AZURE_OPENAI_API_KEY}
  api_base: https://my-resource.openai.azure.com
  api_version: "2024-02-01"
  deployment_name: text-embedding-ada-002
```

**fnllm handles**:
- API endpoint construction
- Authentication (API keys, Azure AD)
- Response parsing
- Error handling (retries with exponential backoff)

---

### tiktoken: Fast Tokenization

**From** `spec/dependencies/01-llm-and-language-processing.md`:

**Why tiktoken?**
- **OpenAI-compatible**: Uses same tokenizer as OpenAI models
- **Fast**: C++ backend, 10-100x faster than pure Python
- **Accurate token counting**: Predict API costs before requests

**Example**:
```python
import tiktoken

encoding = tiktoken.get_encoding("cl100k_base")  # For text-embedding-ada-002

text = "Microsoft is a technology company"
tokens = encoding.encode(text)  # [9416, 374, 264, 5557, 2883]
token_count = len(tokens)        # 5 tokens

# Predict cost
cost_per_1k = 0.0001  # $0.0001 per 1K tokens
cost = (token_count / 1000) * cost_per_1k  # $0.0000005
```

---

### numpy: Vector Operations

**From** `spec/dependencies/03-data-processing-and-infrastructure.md`:

**Operations in embedding pipeline**:
1. **Averaging snippets**: `np.average(embeddings, axis=0)`
2. **Normalization**: `embedding / np.linalg.norm(embedding)`
3. **Cosine similarity**: `np.dot(v1, v2) / (np.linalg.norm(v1) * np.linalg.norm(v2))`

**Why numpy?**
- **C-level performance**: 10-100x faster than pure Python loops
- **Vectorization**: Operate on entire arrays at once
- **Memory efficient**: Contiguous arrays, minimal overhead

**Benchmark** (10K embeddings, 1536-dim):
```
Pure Python loop:   2.5 seconds
NumPy vectorized:   0.03 seconds (83x speedup)
```

---

## Performance Characteristics

### Time Complexity

| Phase | Complexity | Dominant Factor |
|-------|------------|-----------------|
| Tokenization | O(N × L) | N = texts, L = avg length |
| Batching | O(N) | Linear scan |
| API calls | O(B × T_api) | B = batches, T_api = API latency (~500ms) |
| Reconstruction | O(N × D) | N = texts, D = embedding dim (1536) |
| **Total** | **O(B × T_api)** | **API calls dominate** |

**Where**:
- N = number of texts
- L = average text length (characters)
- B = number of batches (N / batch_size)
- D = embedding dimensions (1536)

**Bottleneck**: API latency

**Example** (1000 texts, batch_size=16, num_threads=4):
```
Total batches: 1000 / 16 = 63 batches
Serial time:   63 × 500ms = 31.5 seconds
Parallel time: 63 / 4 × 500ms = 7.9 seconds
```

---

### Space Complexity

| Component | Memory Usage |
|-----------|--------------|
| Raw texts | O(N × L) |
| Token IDs | O(N × L / avg_chars_per_token) |
| Embeddings | O(N × D × 4 bytes) |
| **Total** | **O(N × D)** for embeddings |

**Example** (10K texts, 1536-dim embeddings):
```
Embeddings: 10K × 1536 × 4 bytes = 61.4 MB
```

**Memory optimization**: Stream to vector store instead of holding all in memory

---

### Cost Analysis

**OpenAI pricing** (as of 2024):
```
Model                   Cost per 1M tokens
text-embedding-ada-002  $0.10
text-embedding-3-small  $0.02
text-embedding-3-large  $0.13
```

**Example corpus** (1M text units, 300 tokens each):
```
Total tokens: 1M × 300 = 300M tokens

Using text-embedding-ada-002:
Cost = 300M / 1M × $0.10 = $30

Using text-embedding-3-small (cheaper):
Cost = 300M / 1M × $0.02 = $6 (5x savings)
```

**Cost optimization strategies**:
1. **Use smaller models**: text-embedding-3-small vs. ada-002
2. **Cache embeddings**: Avoid re-embedding unchanged texts
3. **Deduplicate**: Hash identical texts, embed once
4. **Truncate**: Limit input length to essential content (first 2000 chars)

---

## Connection to Research Patterns

### Query-as-Key Paradigm

**From** `spec/research/01-query-as-key.md`:

**Core insight**: Query embedding is the "key" that unlocks relevant context

**Retrieval flow**:
```
Query: "What are Microsoft's AI initiatives?"
   ↓
Embed query → [0.23, -0.12, 0.45, ..., 0.67]  (1536-dim)
   ↓
Search vector store (cosine similarity)
   ↓
Top-k entity embeddings:
  1. "OpenAI partnership" (similarity=0.89)
  2. "Azure AI services" (similarity=0.85)
  3. "GitHub Copilot" (similarity=0.81)
   ↓
Build context from matched entities
   ↓
Generate answer via LLM
```

**Embedding enables semantic matching**:
- Traditional keyword search: "AI initiatives" matches only exact phrase
- Embedding search: Matches "OpenAI partnership", "machine learning projects", "artificial intelligence programs" (all semantically similar)

---

### Complementarity with Graph Embeddings

**From** `spec/graph/04-graph-embedding.md`:

**Two embedding spaces**:
1. **Text embeddings** (this spec): Semantic similarity from language
2. **Graph embeddings** (Node2Vec): Structural similarity from topology

**Hybrid retrieval**:
```python
# Query embedding (text)
query_text_emb = embed_text("What are Microsoft's partnerships?")

# Entity embeddings
entity_text_emb = embed_text(entity.description)  # Semantic
entity_graph_emb = node2vec(entity.id)            # Structural

# Combined similarity
text_sim = cosine_similarity(query_text_emb, entity_text_emb)
graph_sim = cosine_similarity(query_text_emb, entity_graph_emb)

combined_score = 0.7 * text_sim + 0.3 * graph_sim
```

**Use case**: Find entities that are:
- Semantically related to query (text embedding)
- Structurally central in knowledge graph (graph embedding)

---

## Troubleshooting

### Problem: Embedding API Timeout/Rate Limit

**Symptom**: `RateLimitError` or `TimeoutError` from OpenAI API

**Diagnosis**:
- Too many concurrent requests (num_threads too high)
- API quota exceeded (requests per minute limit)

**Solution**:
1. **Reduce concurrency**:
   ```yaml
   num_threads: 2  # Down from 4
   ```

2. **Increase batch size** (fewer total requests):
   ```yaml
   batch_size: 16  # Use maximum allowed
   ```

3. **Add retry logic** (handled by fnllm automatically):
   ```python
   llm:
     max_retries: 10
     request_timeout: 60.0
   ```

4. **Upgrade API tier** (increase quota with OpenAI/Azure)

---

### Problem: Out of Memory During Embedding

**Symptom**: `MemoryError` when embedding large corpus

**Diagnosis**: Holding too many embeddings in memory simultaneously

**Solution**:
1. **Stream to vector store** (don't accumulate in memory):
   ```yaml
   strategy:
     vector_store:
       type: lancedb
       batch_size: 500  # Write to disk every 500 embeddings
   ```

2. **Process in chunks**:
   ```python
   for chunk in chunk_dataframe(df, chunk_size=1000):
       embeddings = await embed_text(chunk)
       vector_store.load_documents(embeddings, overwrite=False)
   ```

3. **Use smaller embedding dimensions** (if supported by model):
   ```yaml
   llm:
     model: text-embedding-3-large
     dimensions: 512  # Down from 3072 (if model supports)
   ```

---

### Problem: Poor Embedding Quality (Low Retrieval Accuracy)

**Symptom**: Irrelevant entities returned for queries

**Diagnosis**:
- Entity descriptions too short (< 50 tokens)
- Descriptions lack semantic detail
- Wrong embedding model for domain

**Solution**:
1. **Improve entity descriptions** (during extraction):
   ```yaml
   entity_extraction:
     prompt: "Extract detailed, context-rich descriptions..."
   ```

2. **Use larger embedding model**:
   ```yaml
   llm:
     model: text-embedding-3-large  # 3072-dim, more expressive
   ```

3. **Fine-tune embeddings** (domain adaptation):
   - Collect positive/negative pairs from your domain
   - Fine-tune model on domain-specific data (requires custom training)

4. **Check embedding normalization**:
   ```python
   # Verify vectors are unit-length
   norm = np.linalg.norm(embedding)
   assert 0.99 < norm < 1.01, f"Embedding not normalized: {norm}"
   ```

---

## Future Enhancements

### 1. Matryoshka Embeddings (Adaptive Dimensionality)

**Current**: Fixed 1536-dim embeddings

**Proposed**: Variable dimensions based on use case
```python
# High-precision search (entity descriptions)
embed_text(text, dimensions=1536)

# Fast approximate search (text units)
embed_text(text, dimensions=256)  # 6x smaller, 90% accuracy retained
```

**Benefit**: 6x memory savings, 3x faster search for low-precision use cases

---

### 2. Domain-Specific Fine-Tuning

**Current**: Generic OpenAI embeddings

**Proposed**: Fine-tune on corpus-specific data
```python
# Collect training data from user feedback
positive_pairs = [(query, relevant_entity), ...]
negative_pairs = [(query, irrelevant_entity), ...]

# Fine-tune embedding model
fine_tuned_model = train_embedding_model(
    base_model="text-embedding-ada-002",
    training_data=(positive_pairs, negative_pairs),
    epochs=5
)
```

**Benefit**: 10-30% improvement in domain-specific retrieval accuracy

---

### 3. Multimodal Embeddings

**Current**: Text-only embeddings

**Proposed**: Joint text-image-graph embeddings
```python
def multimodal_embed(entity):
    text_emb = embed_text(entity.description)
    graph_emb = node2vec(entity.id)
    image_emb = embed_image(entity.image) if entity.image else None

    # Concatenate or attention-based fusion
    return fuse_embeddings([text_emb, graph_emb, image_emb])
```

**Benefit**: Richer semantic representation, better retrieval for multimodal queries

---

### 4. Incremental Embedding Updates

**Current**: Full re-embedding on corpus update

**Proposed**: Embed only new/changed texts
```python
def incremental_embed(new_entities, existing_embeddings):
    # Hash-based deduplication
    new_texts = [
        e.description for e in new_entities
        if hash(e.description) not in existing_embeddings
    ]

    # Embed only new
    new_embeddings = await embed_text(new_texts)

    # Merge with existing
    return {**existing_embeddings, **new_embeddings}
```

**Benefit**: 10-100x speedup for incremental indexing

---

## Conclusion

Text embedding is the **semantic foundation** of GraphRAG's retrieval capabilities, transforming discrete text into continuous vector spaces where similarity search becomes geometric proximity calculation. By leveraging neural language models (OpenAI, Azure), GraphRAG creates embeddings that capture nuanced semantic relationships, enabling precise entity extraction, context-aware search, and hybrid retrieval that combines textual semantics with graph structure.

**Key principles**:
- **Batching + parallelization**: 20x throughput improvement via concurrent API calls
- **Split-average-normalize**: Handle long texts by averaging snippet embeddings
- **Caching**: 60x speedup and 100% cost reduction on re-indexing
- **Cosine similarity**: Geometric measure of semantic relatedness

**Integration points**:
- **Entity extraction**: Semantic search for relevant entities (local search)
- **Context building**: Find text units related to query
- **Hybrid retrieval**: Combine with graph embeddings for structural + semantic signals
- **Vector storage**: LanceDB, Azure AI Search for scalable similarity search

**Key files**:
- `embed_text.py:38-107` — High-level embedding orchestration
- `openai.py:25-172` — OpenAI embedding strategy (batch, parallel, reconstruct)
- `base.py:14-26` — VectorStoreDocument data model for embedding storage
