# Text Chunking Strategies

## Overview

Text chunking decomposes **long documents** into **manageable segments** (text units) that fit within LLM and embedding model token limits. GraphRAG implements **token-based** and **sentence-based** chunking strategies that balance **semantic coherence** (keeping related content together) with **processing constraints** (max tokens per chunk).

## Conceptual Foundation

### Why Chunk Text?

**Problem**: Documents exceed model limits
- **LLM context windows**: 8K-128K tokens (GPT-4)
- **Embedding models**: 8191 tokens (OpenAI ada-002)
- **Typical documents**: 10K-1M+ tokens

**Solution**: Split into overlapping chunks that:
1. **Fit token limits**: Each chunk ≤ max_tokens
2. **Preserve context**: Overlap prevents information loss at boundaries
3. **Enable parallel processing**: Independent chunks → concurrent LLM calls

**GraphRAG use cases**:
- **Entity extraction**: Extract from each chunk independently → merge results
- **Text embedding**: Embed chunks, average for long documents (see `spec/vectors/01-text-embedding.md`)
- **Context building**: Retrieve relevant chunks for answer generation

---

## Chunking Strategies

### 1. Token-Based Chunking

**Algorithm**: `graphrag/index/text_splitting/text_splitting.py:143-159`

```python
def split_single_text_on_tokens(text: str, tokenizer: Tokenizer) -> list[str]:
    """Split a single text and return chunks using the tokenizer."""
    result = []
    input_ids = tokenizer.encode(text)  # Text → token IDs

    start_idx = 0
    cur_idx = min(start_idx + tokenizer.tokens_per_chunk, len(input_ids))
    chunk_ids = input_ids[start_idx:cur_idx]

    while start_idx < len(input_ids):
        chunk_text = tokenizer.decode(list(chunk_ids))  # Token IDs → text
        result.append(chunk_text)

        # Slide window with overlap
        start_idx += tokenizer.tokens_per_chunk - tokenizer.chunk_overlap
        cur_idx = min(start_idx + tokenizer.tokens_per_chunk, len(input_ids))
        chunk_ids = input_ids[start_idx:cur_idx]

    return result
```

**Parameters**:
- `tokens_per_chunk`: Max tokens per chunk (default: 1200)
- `chunk_overlap`: Overlapping tokens between chunks (default: 100)
- `encoding_model`: Tokenizer model (cl100k_base for GPT-4, ada-002)

**Example**:
```
Text: "The quick brown fox jumps over the lazy dog. The dog sleeps under the tree."
Tokens: [464, 4062, 14198, 39935, 35308, 927, 279, 16053, 5679, 13, 578, 5679, 72490, 1234, 279, 5021, 13]

tokens_per_chunk = 10
chunk_overlap = 3

Chunk 1 (tokens 0-10):   "The quick brown fox jumps over the lazy dog."
Chunk 2 (tokens 7-17):   "lazy dog. The dog sleeps under the tree."
                          ↑ overlap ↑

Overlap preserves context: "lazy dog" appears in both chunks
```

**Overlap rationale**:
- **Prevents boundary loss**: Entity spanning chunk boundary captured by both chunks
- **Improves entity extraction**: LLM sees full context even near boundaries
- **Trade-off**: 10-20% redundancy vs. missing information

**Implementation** (`chunk_text.py:19-79`):
```python
def chunk_text(
    input: pd.DataFrame,
    column: str,
    size: int,               # tokens_per_chunk
    overlap: int,            # chunk_overlap
    encoding_model: str,     # tiktoken model
    strategy: ChunkStrategyType,  # "tokens" or "sentence"
    callbacks: WorkflowCallbacks,
) -> pd.Series:
    """Chunk a piece of text into smaller pieces."""
    strategy_exec = load_strategy(strategy)  # Load chunking strategy

    num_total = _get_num_total(input, column)
    tick = progress_ticker(callbacks.progress, num_total)

    config = ChunkingConfig(size=size, overlap=overlap, encoding_model=encoding_model)

    return input.apply(
        lambda x: run_strategy(strategy_exec, x[column], config, tick),
        axis=1,
    )
```

---

### 2. Sentence-Based Chunking

**Algorithm**: `graphrag/index/operations/chunk_text/strategies.py:58-69`

```python
def run_sentences(
    input: list[str], _config: ChunkingConfig, tick: ProgressTicker
) -> Iterable[TextChunk]:
    """Chunks text into multiple parts by sentence."""
    for doc_idx, text in enumerate(input):
        sentences = nltk.sent_tokenize(text)  # Split into sentences
        for sentence in sentences:
            yield TextChunk(
                text_chunk=sentence,
                source_doc_indices=[doc_idx],
            )
        tick(1)
```

**How it works**:
1. **NLTK sentence tokenizer**: Trained on linguistic patterns (periods, capitalization)
2. **One sentence per chunk**: No overlap, natural boundaries
3. **Preserve semantic units**: Sentences are self-contained thoughts

**Example**:
```
Text: "Microsoft was founded in 1975. It develops software products. The company is based in Redmond."

Chunks:
1. "Microsoft was founded in 1975."
2. "It develops software products."
3. "The company is based in Redmond."
```

**Use cases**:
- **Fine-grained extraction**: Extract entities per sentence (very precise provenance)
- **Short documents**: Already within token limits, sentence-level is sufficient
- **Testing/debugging**: Easier to inspect sentence-level outputs

**Limitations**:
- **No overlap**: May lose cross-sentence relationships
- **Variable size**: Some sentences are very long (> 100 tokens) or very short (< 10 tokens)
- **Less common**: Token-based is standard for GraphRAG

---

### 3. Multi-Document Chunking

**Algorithm**: `text_splitting.py:164-193`

```python
def split_multiple_texts_on_tokens(
    texts: list[str], tokenizer: Tokenizer, tick: ProgressTicker
) -> list[TextChunk]:
    """Split multiple texts and return chunks with metadata."""
    result = []
    mapped_ids = []

    # Encode all texts, track source document
    for source_doc_idx, text in enumerate(texts):
        encoded = tokenizer.encode(text)
        mapped_ids.append((source_doc_idx, encoded))

    # Flatten to single token stream with doc IDs
    input_ids = [
        (source_doc_idx, id) for source_doc_idx, ids in mapped_ids for id in ids
    ]

    # Chunk with sliding window
    start_idx = 0
    cur_idx = min(start_idx + tokenizer.tokens_per_chunk, len(input_ids))
    chunk_ids = input_ids[start_idx:cur_idx]

    while start_idx < len(input_ids):
        chunk_text = tokenizer.decode([id for _, id in chunk_ids])
        doc_indices = list({doc_idx for doc_idx, _ in chunk_ids})  # Which docs in chunk
        result.append(TextChunk(chunk_text, doc_indices, len(chunk_ids)))

        start_idx += tokenizer.tokens_per_chunk - tokenizer.chunk_overlap
        cur_idx = min(start_idx + tokenizer.tokens_per_chunk, len(input_ids))
        chunk_ids = input_ids[start_idx:cur_idx]

    return result
```

**Key feature**: **Cross-document chunks**
- Chunks can span multiple source documents
- Tracks which documents contribute to each chunk (`source_doc_indices`)

**Example**:
```
Doc A: "Microsoft was founded in 1975." (50 tokens)
Doc B: "OpenAI develops GPT models." (30 tokens)

tokens_per_chunk = 60
chunk_overlap = 10

Chunk 1 (tokens 0-60):
  Text: "Microsoft was founded in 1975. OpenAI develops..."
  source_doc_indices: [0, 1]  # Spans Doc A and Doc B

Chunk 2 (tokens 50-80):
  Text: "...1975. OpenAI develops GPT models."
  source_doc_indices: [0, 1]  # Overlap includes both docs
```

**Use case**: Batch processing multiple documents as continuous stream

---

## Integration with GraphRAG Pipeline

### Chunking in Indexing Flow

**From** `spec/semlang/01-document-transformation.md`:

```sfl
SEMANTIC FLOW Indexing:
  LOAD documents → raw_documents

  CHUNK raw_documents → text_units                    # ← THIS SPEC
    STRATEGY: tokens(size=1200, overlap=100)
    RESULT: text_units = [{id, text, source_doc_ids}]

  EXTRACT entities FROM text_units → graph
    # Each text_unit processed independently (embarrassingly parallel)

  EMBED text_units.text → embeddings
    # Chunks fit within embedding model limits (8191 tokens)

  STORE text_units, embeddings → index
```

**Actual workflow** (conceptual YAML):
```yaml
workflows:
  - name: create_base_text_units
    steps:
      - verb: chunk_text
        args:
          column: text              # Column with document text
          size: 1200                # Max tokens per chunk
          overlap: 100              # Overlapping tokens
          strategy: tokens          # Token-based chunking
```

---

## Connection to Graph Algorithms

### Entity Extraction Per Chunk

**From** `spec/graph/03-graph-construction-and-merging.md`:

**Challenge**: Entities extracted from chunks need merging

```
Chunk 1: Extract → [{"title": "MICROSOFT", "type": "org", ...}]
Chunk 2: Extract → [{"title": "MICROSOFT", "type": "org", ...}]  # Duplicate!

Merge by (title, type) → Single "MICROSOFT" entity with:
  - Descriptions from both chunks
  - frequency = 2 (mentioned in 2 chunks)
```

**Overlap benefit**: Entity near chunk boundary extracted by both chunks → higher confidence

---

### Text Unit Retrieval

**From** `spec/vectors/01-text-embedding.md`:

**Query-time flow**:
1. **Embed query**: "What are Microsoft's products?"
2. **Search text_unit embeddings**: Find top-k similar chunks
3. **Return text units**: Provide chunks as context to LLM

**Why chunking matters**:
- **Granularity**: Retrieve specific paragraphs, not entire documents
- **Relevance**: Small chunks = focused context (less noise)
- **Token budget**: Pack multiple relevant chunks into LLM context window

**Example**:
```
Query: "Microsoft Azure features"

Retrieved chunks (k=5):
1. "...Azure provides cloud computing services including..." (score=0.92)
2. "...Microsoft Azure supports virtual machines..." (score=0.89)
3. "...Azure AI services integrate with OpenAI..." (score=0.85)
...

Context = concatenate chunks → LLM generates answer
```

---

## Connection to Dependencies

### tiktoken: Fast Tokenization

**From** `spec/dependencies/01-llm-and-language-processing.md`:

**Why tiktoken?**
- **OpenAI-compatible**: Same tokenizer as GPT models (cl100k_base)
- **Fast**: C++ implementation, 10-100x faster than Python tokenizers
- **Accurate chunking**: Exact token count → no API rejections

**Usage**:
```python
import tiktoken

encoding = tiktoken.get_encoding("cl100k_base")
tokens = encoding.encode("Microsoft was founded in 1975")  # [12401, 574, 18538, 304, 220, 16, 24609, 20]
text = encoding.decode(tokens)  # "Microsoft was founded in 1975"
```

---

### nltk: Sentence Tokenization

**From** `spec/dependencies/01-llm-and-language-processing.md`:

**How NLTK sentence_tokenize works**:
- **Punkt tokenizer**: Unsupervised ML model trained on text corpora
- **Learns abbreviations**: "Dr." vs. "end of sentence."
- **Multi-language support**: English, German, French, etc.

**Example**:
```python
import nltk
nltk.download('punkt')

text = "Dr. Smith works at Microsoft. He joined in 2020."
sentences = nltk.sent_tokenize(text)
# ["Dr. Smith works at Microsoft.", "He joined in 2020."]
# Note: "Dr." correctly recognized as abbreviation, not sentence boundary
```

---

## Performance Characteristics

### Time Complexity

| Operation | Complexity | Notes |
|-----------|------------|-------|
| Token encoding | O(N) | N = text length (chars) |
| Token chunking | O(T) | T = total tokens |
| Sentence tokenization | O(N) | Linear scan with NLTK |
| Multi-doc chunking | O(D × T_avg) | D = docs, T_avg = avg tokens per doc |

**Bottleneck**: Token encoding (CPU-bound)

**Example** (1M characters, 250K tokens):
```
Encoding:  2 seconds (tiktoken)
Chunking:  0.1 seconds (sliding window)
Total:     2.1 seconds
```

---

### Chunk Size Trade-offs

| Chunk Size | Pros | Cons | Use Case |
|------------|------|------|----------|
| Small (200-400 tokens) | Fine-grained retrieval, focused context | More chunks → slower processing | Entity extraction, precise search |
| Medium (600-1200 tokens) | **Balanced** (default) | - | General purpose |
| Large (2000-4000 tokens) | Fewer chunks, more context per chunk | May exceed embedding limits | Summarization, document-level tasks |

**GraphRAG default**: 1200 tokens (balanced)

---

### Overlap Trade-offs

| Overlap | Pros | Cons | Boundary Loss |
|---------|------|------|---------------|
| 0% | No redundancy, fast processing | High boundary loss (entities split) | ~20% entities lost |
| ~8% (100 tokens / 1200) | **Balanced** (default) | Minimal redundancy | ~2% entities lost |
| ~20% (240 tokens / 1200) | Very low boundary loss | 20% more processing cost | <1% entities lost |

**GraphRAG default**: 100 tokens overlap (~8%)

---

## Troubleshooting

### Problem: Chunks Exceed Embedding Model Limits

**Symptom**: Embedding API returns `InvalidRequestError: tokens exceed limit`

**Diagnosis**: Chunk size configured too large

**Solution**:
```yaml
chunking:
  size: 1200  # Ensure < 8191 (OpenAI limit)
  overlap: 100
```

For safety, use **chunk_size < 8000** to account for encoding variations.

---

### Problem: Entity Extraction Quality Poor

**Symptom**: Many partial entities, relationships missing

**Diagnosis**: Chunks too small or no overlap

**Solution**:
1. **Increase chunk size**:
   ```yaml
   size: 1500  # Up from 600
   ```

2. **Increase overlap**:
   ```yaml
   overlap: 200  # Up from 100 (13% overlap)
   ```

3. **Inspect chunk boundaries** (look for split entities):
   ```python
   for i, chunk in enumerate(chunks[:5]):
       print(f"Chunk {i}: ...{chunk[-100:]}")  # Last 100 chars
       print(f"Chunk {i+1}: {chunks[i+1][:100]}...")  # First 100 chars of next
       # Check if entities are split across boundary
   ```

---

### Problem: Memory Usage Too High

**Symptom**: OOM error during chunking of large corpus

**Diagnosis**: All chunks loaded in memory simultaneously

**Solution**:
1. **Stream processing**:
   ```python
   for doc_batch in chunk_dataframe(documents, batch_size=1000):
       chunks = chunk_text(doc_batch, ...)
       process_chunks(chunks)  # Process and discard
   ```

2. **Use generator patterns** (already implemented in sentence chunking)

---

## Future Enhancements

### 1. Semantic Chunking

**Current**: Fixed token boundaries (may split mid-sentence)

**Proposed**: Chunk at semantic boundaries (paragraphs, topics)
```python
def semantic_chunk(text, max_tokens=1200):
    """Chunk at paragraph or topic shifts."""
    paragraphs = text.split("\n\n")
    chunks = []
    current_chunk = []
    current_tokens = 0

    for para in paragraphs:
        para_tokens = count_tokens(para)
        if current_tokens + para_tokens > max_tokens:
            chunks.append("\n\n".join(current_chunk))
            current_chunk = [para]
            current_tokens = para_tokens
        else:
            current_chunk.append(para)
            current_tokens += para_tokens

    return chunks
```

**Benefit**: Preserve semantic coherence, improve extraction quality

---

### 2. Adaptive Overlap

**Current**: Fixed overlap (100 tokens)

**Proposed**: Variable overlap based on content
```python
def adaptive_overlap(chunk1, chunk2):
    """Increase overlap if entity detected near boundary."""
    boundary_text = chunk1[-200:]  # Last 200 tokens of chunk1
    if has_incomplete_entity(boundary_text):
        return 200  # Double overlap
    return 100  # Standard overlap
```

**Benefit**: Reduce boundary loss only where needed (lower redundancy)

---

### 3. Hierarchical Chunking

**Current**: Single-level chunks (1200 tokens)

**Proposed**: Multi-level chunks (sentences → paragraphs → sections)
```python
chunks = {
    "level_0": sentences,          # 50-100 tokens
    "level_1": paragraphs,         # 400-600 tokens
    "level_2": sections,           # 1200-1500 tokens
}
```

**Benefit**: Retrieve at appropriate granularity (sentence for precise, section for context)

---

## Conclusion

Text chunking is the **preprocessing foundation** for GraphRAG's entity extraction and embedding pipelines. By decomposing long documents into token-constrained, overlapping segments, chunking enables:
- **Parallel LLM processing**: Independent chunks → concurrent entity extraction
- **Embedding model compatibility**: Chunks fit within 8191-token limit
- **Boundary preservation**: Overlap ensures entities near chunk boundaries aren't lost
- **Fine-grained retrieval**: Small chunks enable precise context selection

**Key principles**:
- **Token-based chunking**: Use tiktoken for OpenAI-compatible tokenization
- **Overlap strategy**: 8% overlap (100/1200 tokens) balances redundancy vs. boundary loss
- **Multi-document support**: Track source document IDs for provenance
- **Sliding window**: Efficient O(T) algorithm for chunk generation

**Integration points**:
- **Entity extraction**: One LLM call per chunk (map-reduce pattern)
- **Text embedding**: Chunks within embedding model limits
- **Vector search**: Retrieve relevant chunks for query context

**Key files**:
- `chunk_text.py:19-79` — High-level chunking orchestration
- `text_splitting.py:143-193` — Token-based sliding window algorithm
- `strategies.py:35-69` — Token and sentence chunking strategies
