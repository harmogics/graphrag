# Machine Learning Algorithms in GraphRAG

## Overview

This directory documents the **machine learning algorithms** that power GraphRAG's knowledge extraction, semantic understanding, and structural analysis capabilities. GraphRAG leverages **state-of-the-art pre-trained models** from external providers, focusing on inference and integration rather than custom model training.

## ML Algorithm Classification

```
┌──────────────────────────────────────────────────────────────────────┐
│                    ML ALGORITHM TAXONOMY                              │
└──────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  DEEP LEARNING (Neural Networks)                                    │
│                                                                       │
│  01-LLM ALGORITHMS                                                  │
│  • Transformer Architecture (GPT-4, GPT-3.5-turbo)                 │
│  • Multi-head Self-Attention                                        │
│  • Few-shot Learning (in-context learning)                         │
│  • Abstractive Summarization                                       │
│  • Retrieval-Augmented Generation (RAG)                           │
│                                                                       │
│  03-TEXT EMBEDDINGS (Embedding & NLP)                              │
│  • Encoder-only Transformers (BERT-style)                          │
│  • Contrastive Learning (text-embedding-ada-002)                  │
│  • Self-supervised Pre-training                                    │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  GRAPH ML (Structural Learning)                                     │
│                                                                       │
│  02-GRAPH ML ALGORITHMS                                             │
│  • Hierarchical Leiden (Modularity Optimization)                   │
│  • Node2Vec (Random Walk + Skip-Gram)                              │
│  • Community Detection (Unsupervised Clustering)                   │
│  • Graph Embeddings (Representation Learning)                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  CLASSICAL ML (Statistical & Manifold Learning)                      │
│                                                                       │
│  03-NLP & DIMENSIONALITY REDUCTION                                  │
│  • NLTK Punkt Tokenizer (Unsupervised Statistical Model)           │
│  • UMAP (Manifold Learning, Topological Data Analysis)             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Complete ML Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│                    GRAPHRAG ML ALGORITHM PIPELINE                     │
└──────────────────────────────────────────────────────────────────────┘

INPUT: Raw Documents
   │
   ▼
┌────────────────────────────────────────────┐
│  NLP PREPROCESSING                         │  ← Classical ML
│  - Punkt Sentence Tokenizer (NLTK)        │
│  - Token-based Chunking (tiktoken)        │
│  - Text Normalization                     │
└────────────────────────────────────────────┘
   │
   ▼ text_units = [{id, text, source_docs}]
   │
   ├────────────────┬──────────────────────────┐
   │                │                          │
   ▼                ▼                          ▼
┌──────────┐  ┌──────────┐           ┌──────────────┐
│ 01-LLM   │  │ 03-TEXT  │           │ 01-LLM       │
│ Entity   │  │ EMBED    │           │ Summaries    │
│ Extract  │  │          │           │              │
└──────────┘  └──────────┘           └──────────────┘
   │                │                          │
   │ Entities       │ 1536-dim vectors        │ Descriptions
   │ Relations      │                          │
   │                │                          │
   └────────┬───────┴──────────────────────────┘
            │
            ▼
   ┌────────────────────────────────────────┐
   │  KNOWLEDGE GRAPH                       │
   │  • Entities (with embeddings)          │
   │  • Relationships (with weights)        │
   │  • Text Units (with embeddings)        │
   └────────────────────────────────────────┘
            │
            ├───────────────┬────────────────────┬──────────────────┐
            │               │                    │                  │
            ▼               ▼                    ▼                  ▼
   ┌─────────────┐  ┌─────────────┐   ┌──────────────┐   ┌─────────────┐
   │ 02-GRAPH ML │  │ 03-TEXT     │   │ 01-LLM       │   │ 03-UMAP     │
   │ Community   │  │ Vector      │   │ Answer Gen   │   │ Viz 2D      │
   │ Detection   │  │ Search      │   │ (RAG)        │   │ Projection  │
   │ (Leiden)    │  │ (Cosine)    │   │              │   │             │
   └─────────────┘  └─────────────┘   └──────────────┘   └─────────────┘
            │               │                    │                  │
            │ Communities   │ Retrieved Context  │ Answers          │ 2D Coords
            │               │                    │                  │
            └───────────────┴────────────────────┴──────────────────┘
                                      │
                                      ▼
                            ┌──────────────────┐
                            │  02-GRAPH ML     │
                            │  Graph Embed     │
                            │  (Node2Vec)      │
                            └──────────────────┘
                                      │
                                      ▼
                            Graph Structure Vectors
```

---

## Algorithm Documents

### [01-llm-algorithms.md](01-llm-algorithms.md)
**Large Language Models** — Transformer-based neural networks for knowledge extraction and generation

**Purpose**: Extract structured knowledge from text, generate summaries, and answer questions

**Key concepts**:
- **Architecture**: Multi-layer Transformer (12-96 layers, 175B+ parameters)
- **Attention mechanism**: Multi-head self-attention for contextual understanding
- **Few-shot learning**: Provide examples in prompt for entity extraction
- **Abstractive summarization**: Consolidate multi-perspective descriptions
- **RAG pattern**: Retrieval-augmented answer generation

**ML Training** (external):
- Pre-training: 300B+ tokens (web, books, code)
- Instruction tuning: RLHF (human feedback)
- Provider: OpenAI (GPT-4, GPT-3.5-turbo)

**Use cases**:
- Entity and relationship extraction from text units
- Entity/relationship description summarization
- Community report generation (hierarchical summaries)
- Query understanding and answer generation
- Claim extraction and validation

**External dependencies**: `openai==1.x`, `fnllm` (Microsoft)

**Complexity**: O(N²) per token (self-attention), O(N) tokens per generation

**Cost**: $10-60 per 1M tokens (varies by model)

---

### [02-graph-ml-algorithms.md](02-graph-ml-algorithms.md)
**Graph Machine Learning** — Structural learning algorithms for community detection and embeddings

**Purpose**: Discover semantic clusters and learn continuous representations of graph structure

**Key concepts**:
- **Hierarchical Leiden**: Multi-scale community detection via modularity optimization
  - Modularity metric: Q = (1/2m) Σ [A_ij - k_i×k_j/(2m)] δ(c_i, c_j)
  - Local moving, refinement, aggregation phases
  - Hierarchical structure (fine → coarse communities)

- **Node2Vec**: Graph embeddings via random walks + Skip-Gram
  - Random walk: Sample graph neighborhoods (num_walks × walk_length)
  - Skip-Gram neural network: Predict context nodes from target
  - Output: 1536-dim continuous vectors (structural similarity)

**ML Training**:
- Leiden: Unsupervised optimization (no training phase)
- Node2Vec: Self-supervised (trained on graph structure)
  - Epochs: 3-10 iterations through walks
  - Optimization: Stochastic gradient descent
  - Loss: Negative sampling objective

**Use cases**:
- Community detection for global search and report generation
- Graph visualization (UMAP projection from embeddings)
- Hybrid retrieval (text + graph embeddings)
- Entity features for ML models

**External dependencies**: `graspologic==3.4.1` (Microsoft Research)

**Parameters**:
- Leiden: `max_cluster_size=10`, `random_seed`
- Node2Vec: `dimensions=1536`, `num_walks=10`, `walk_length=40`

**Complexity**:
- Leiden: O((N+E) × log N) per iteration
- Node2Vec: O(N × num_walks × walk_length × window_size × epochs)

---

### [03-embedding-and-nlp-algorithms.md](03-embedding-and-nlp-algorithms.md)
**Text Embeddings & NLP** — Neural embeddings, sentence tokenization, and dimensionality reduction

**Purpose**: Transform text to semantic vectors, segment sentences, visualize high-dimensional data

**Key concepts**:

**1. Transformer-Based Text Embeddings**:
- **Architecture**: Encoder-only Transformer (BERT-style, 12-24 layers)
- **Models**: text-embedding-ada-002, text-embedding-3-small/large
- **Output**: 1536-dim normalized vectors
- **Training** (external): Contrastive learning on billions of text pairs
- **Use cases**: Entity descriptions, text units, queries, community reports

**2. NLTK Punkt Sentence Tokenizer**:
- **Algorithm**: Unsupervised statistical model
- **Training**: Learns abbreviations, collocations, sentence starters from corpus
- **Type**: Classical ML (probabilistic decision tree)
- **Use cases**: Sentence-based chunking (optional strategy)
- **Accuracy**: ~98% on English text

**3. UMAP Dimensionality Reduction**:
- **Algorithm**: Manifold learning (topological data analysis)
- **Training**: Unsupervised (on data, not pre-trained)
- **Process**: k-NN graph → fuzzy topological sets → SGD optimization
- **Output**: 2D/3D coordinates from 1536-dim embeddings
- **Use cases**: Graph visualization, cluster exploration

**External dependencies**:
- `openai==1.x` — Text embeddings (API)
- `nltk==3.8.1` — Punkt tokenizer (pre-trained models)
- `umap-learn==0.5.4` — Manifold learning

**Complexity**:
- Text embedding: O(N² × L) where L = sequence length (attention)
- Punkt: O(N) linear scan
- UMAP: O(N × log N) for k-NN, O(N × epochs) for optimization

**Performance**:
- Text embedding: ~500ms per batch (16 texts), API limited
- Punkt: < 1ms per document
- UMAP: ~1 minute for 10K points (1536-dim → 2D)

---

## ML Algorithm Interaction Map

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ALGORITHM DEPENDENCIES                            │
└─────────────────────────────────────────────────────────────────────┘

TEXT PREPROCESSING (Classical ML)
   ├─ Punkt Sentence Tokenizer (NLTK)
   └─ Token-based Chunking (tiktoken)
         │
         └──▶ text_units (chunks)
               │
               ├──────────────────┬────────────────────┐
               │                  │                    │
               ▼                  ▼                    ▼
         ┌──────────┐      ┌─────────────┐     ┌──────────┐
         │ LLM      │      │ TEXT        │     │ LLM      │
         │ Extract  │      │ EMBEDDING   │     │ Summary  │
         └──────────┘      └─────────────┘     └──────────┘
               │                  │                    │
               │ Entities         │ 1536-dim          │ Descriptions
               │ Relations        │ vectors            │
               │                  │                    │
               └──────────┬───────┴────────────────────┘
                          │
                          ▼
                   KNOWLEDGE GRAPH
                 (entities + relations)
                          │
                          ├─────────────────┬──────────────────┐
                          │                 │                  │
                          ▼                 ▼                  ▼
                   ┌──────────┐      ┌──────────┐      ┌──────────┐
                   │ LEIDEN   │      │ NODE2VEC │      │ VECTOR   │
                   │ Community│      │ Graph    │      │ SEARCH   │
                   │ Detection│      │ Embedding│      │ Cosine   │
                   └──────────┘      └──────────┘      └──────────┘
                          │                 │                  │
                          │                 │                  │
                          │                 ▼                  │
                          │           ┌──────────┐            │
                          │           │ UMAP     │            │
                          │           │ 2D Viz   │            │
                          │           └──────────┘            │
                          │                                   │
                          └─────────────┬─────────────────────┘
                                        │
                                        ▼
                                  ┌──────────┐
                                  │ LLM      │
                                  │ Answer   │
                                  │ Generate │
                                  └──────────┘
```

**Critical dependencies**:
1. **LLM + Text Embedding** → Run in parallel during indexing (independent)
2. **Punkt/Chunking** → Must precede all other algorithms (creates text units)
3. **Graph ML** → Depends on graph construction (entity extraction via LLM)
4. **UMAP** → Depends on Node2Vec (visualizes graph embeddings)
5. **Vector Search** → Depends on text embeddings (retrieval at query time)

---

## External Dependencies vs. Internal Implementation

### All ML Algorithms Use External Pre-Trained Models

```
┌──────────────────────────────────────────────────────────────────────┐
│            EXTERNAL (Third-Party Packages)                            │
└──────────────────────────────────────────────────────────────────────┘

| ML Algorithm         | Package           | Provider          | Training  |
|----------------------|-------------------|-------------------|-----------|
| LLMs (GPT-4, GPT-3.5)| openai==1.x       | OpenAI            | External  |
| Text Embeddings      | openai==1.x       | OpenAI            | External  |
| Leiden Community     | graspologic==3.4.1| Microsoft Research| N/A (opt) |
| Node2Vec Embeddings  | graspologic==3.4.1| Microsoft Research| On data   |
| Punkt Tokenizer      | nltk==3.8.1       | NLTK Project      | External  |
| UMAP Reduction       | umap-learn==0.5.4 | McInnes et al.    | On data   |

┌──────────────────────────────────────────────────────────────────────┐
│            INTERNAL (GraphRAG Implementation)                         │
└──────────────────────────────────────────────────────────────────────┘

GraphRAG implements:
• Pipeline orchestration (chunking → embedding → extraction)
• Prompt engineering (few-shot examples for entity extraction)
• Map-reduce patterns (parallel LLM calls, result merging)
• Caching and batching (optimize API calls)
• Integration glue (connect LLMs to graph/vector stores)

GraphRAG does NOT implement:
• Neural network architectures
• Model training or fine-tuning
• Core ML algorithms (Transformer, Leiden, Skip-Gram, UMAP)
```

**Key insight**: GraphRAG is an **inference and integration system**, not an ML training platform. All ML models are pre-trained and consumed via APIs or libraries.

---

## Cross-References to Other Specs

### To `spec/graph/` (Graph Algorithms)

**[spec/graph/03-graph-construction-and-merging.md](../graph/03-graph-construction-and-merging.md)**:
- **LLM entity extraction** outputs raw extractions → graph construction merges them
- **LLM description summarization** consolidates multi-perspective entity descriptions

**[spec/graph/01-community-detection.md](../graph/01-community-detection.md)**:
- **Hierarchical Leiden** (Graph ML) enables global search and community reports
- **LLM community summarization** generates natural language summaries of clusters

**[spec/graph/04-graph-embedding.md](../graph/04-graph-embedding.md)**:
- **Node2Vec** (Graph ML) produces structural embeddings
- **UMAP** (Classical ML) projects embeddings to 2D for visualization

---

### To `spec/vectors/` (Vector Algorithms)

**[spec/vectors/01-text-embedding.md](../vectors/01-text-embedding.md)**:
- **Transformer text embeddings** documented here (ML algorithm)
- Vector processing pipeline documented there (chunking, batching, storage)

**[spec/vectors/02-chunking-strategies.md](../vectors/02-chunking-strategies.md)**:
- **Punkt sentence tokenizer** (Classical ML) enables sentence-based chunking
- **tiktoken** (not ML) enables token-based chunking

**[spec/vectors/03-vector-search-and-similarity.md](../vectors/03-vector-search-and-similarity.md)**:
- **Cosine similarity** (not ML, geometric) uses embeddings from Transformer models
- **ANN algorithms** (HNSW, IVF) are search algorithms, not ML training

---

### To `spec/dependencies/` (Implementation Details)

**[spec/dependencies/01-llm-and-language-processing.md](../dependencies/01-llm-and-language-processing.md)**:
- Complete list of LLM packages: `openai`, `fnllm`, `tiktoken`
- LLM configuration and usage patterns

**[spec/dependencies/02-graph-and-vector-storage.md](../dependencies/02-graph-and-vector-storage.md)**:
- `graspologic` package details (Leiden, Node2Vec implementation)
- `lancedb`, `azure-search` for vector storage (not ML algorithms)

---

### To `spec/research/` (Conceptual Foundations)

**[spec/research/01-query-as-key.md](../research/01-query-as-key.md)**:
- **Query embeddings** (Transformer) serve as "keys" unlocking relevant context
- **Vector similarity** realizes the query-as-key paradigm geometrically

**[spec/research/02-star-attractor-patterns.md](../research/02-star-attractor-patterns.md)**:
- **Community detection** (Leiden) identifies attractor basins
- **Graph embeddings** (Node2Vec) encode attractor strength in vector space

---

### To `spec/architecture/` (System Patterns)

**[spec/architecture/01-agent-patterns.md](../architecture/01-agent-patterns.md)**:
- **Map-reduce**: Parallel LLM entity extraction (map) → merging (reduce)
- **Parallel community reports**: Independent LLM summarization per community
- **Async batching**: Optimize LLM and embedding API calls

---

### To `spec/semlang/` (Semantic Flow Language)

**[spec/semlang/01-document-transformation.md](../semlang/01-document-transformation.md)**:
- LLM extraction is a **TRANSFORM** operation (text → entities)
- Text embedding is a **TRANSFORM** operation (text → vectors)
- Graph ML is a **TRANSFORM** operation (graph → communities/embeddings)

---

## Decision Trees for Practitioners

### Which ML Algorithms Do I Need?

```
START: Define your use case
   │
   ├─ Need basic entity search (keyword-like)?
   │  │
   │  └─▶ Minimal:
   │        • LLM: Entity extraction
   │        • Text Embedding: Entity/text unit embeddings
   │        • Vector Search: Retrieval
   │      Skip: Community detection, graph embeddings, UMAP
   │
   ├─ Need "summarize the corpus" queries (global search)?
   │  │
   │  └─▶ Add:
   │        • Leiden Community Detection (graph ML)
   │        • LLM: Community report generation
   │
   ├─ Need graph visualization?
   │  │
   │  └─▶ Add:
   │        • Node2Vec (graph ML): Structure embeddings
   │        • UMAP (classical ML): 2D projection
   │
   ├─ Need hybrid search (text + graph importance)?
   │  │
   │  └─▶ Add:
   │        • Node2Vec (optional): Structural similarity
   │        • Centrality metrics: Degree, PageRank (not ML)
   │
   └─ Research/exploration?
      │
      └─▶ Enable all algorithms for full capabilities
```

---

### Choosing LLM Models

```
START: Which LLM model?
   │
   ├─ Quality most important (cost not constrained)?
   │  │
   │  └─▶ gpt-4-turbo or gpt-4
   │        • Best entity extraction accuracy
   │        • Best summarization quality
   │        • $10-60 per 1M tokens
   │
   ├─ Speed important (large corpus)?
   │  │
   │  └─▶ gpt-3.5-turbo
   │        • 10x faster than GPT-4
   │        • 5-10x cheaper
   │        • Acceptable quality for most use cases
   │
   ├─ Privacy/data residency requirements?
   │  │
   │  └─▶ Azure OpenAI (same models, Azure cloud)
   │        OR local models (LLaMA, Mistral via fnllm)
   │
   └─ Budget constrained?
      │
      └─▶ gpt-3.5-turbo with reduced parallelism
          • Disable description summarization
          • Use smaller batch sizes
```

---

### Optimizing ML Algorithm Performance

```
START: Performance bottleneck?
   │
   ├─ LLM calls taking too long?
   │  │
   │  ├─ Increase parallelism:
   │  │    num_threads: 50 (watch API rate limits)
   │  │
   │  ├─ Disable summarization:
   │  │    summarize_descriptions: false (10-100x speedup)
   │  │
   │  └─ Use faster model:
   │       gpt-3.5-turbo instead of gpt-4
   │
   ├─ Text embedding API slow?
   │  │
   │  ├─ Increase batch_size: 16 (max)
   │  │
   │  ├─ Increase num_threads: 4-8 (concurrent requests)
   │  │
   │  └─ Check rate limits:
   │       Azure OpenAI has higher quotas
   │
   ├─ Node2Vec embedding OOM?
   │  │
   │  ├─ Reduce walk parameters:
   │  │    num_walks: 5 (from 10)
   │  │    walk_length: 20 (from 40)
   │  │
   │  ├─ Reduce dimensions:
   │  │    dimensions: 256 (from 1536)
   │  │
   │  └─ Prune graph first:
   │       Remove low-frequency nodes before embedding
   │
   └─ Leiden community detection slow?
      │
      └─▶ Already optimized (graspologic efficient)
          If still slow: Graph is very large (> 100K nodes)
          Solution: More aggressive pruning
```

---

## Performance Benchmarks

### Typical GraphRAG Corpus (10K Documents, 100K Text Units)

| ML Algorithm | Input | Output | Time | Memory | Cost (USD) |
|--------------|-------|--------|------|--------|------------|
| **LLM: Entity Extraction** | 100K chunks | 50K entities | 30 min | 2 GB | $50-150 |
| **LLM: Description Summary** | 50K entities | 50K summaries | 25 min | 1 GB | $20-60 |
| **Text Embedding** | 100K text units | 100K vectors | 8 min | 1 GB | $1-5 |
| **Leiden Community** | 50K nodes, 150K edges | 500 communities | 3 min | 2 GB | $0 (local) |
| **Node2Vec Embedding** | 50K nodes | 50K vectors | 15 min | 8 GB | $0 (local) |
| **UMAP Visualization** | 50K vectors (1536-dim) | 50K coords (2D) | 5 min | 4 GB | $0 (local) |

**Total indexing time**: ~90 minutes (LLM calls dominate)

**Total cost**: ~$75-220 (varies by model and API provider)

**Bottleneck**: LLM API calls (can be parallelized up to rate limits)

---

### Scaling Characteristics

| Corpus Size | LLM Extract | Text Embed | Leiden | Node2Vec | Total Time |
|-------------|-------------|------------|--------|----------|------------|
| 1K docs | 3 min | 1 min | 5 sec | 30 sec | 5 min |
| 10K docs | 30 min | 8 min | 3 min | 15 min | 60 min |
| 100K docs | 5 hours | 80 min | 30 min | 2 hours | 9 hours |
| 1M docs | 50 hours* | 800 min* | 4 hours | 18 hours* | 4 days* |

\* Parallelizable with distributed processing and higher API quotas

**Scaling laws**:
- LLM extraction: O(N) calls (limited by rate limits, parallelizable)
- Text embedding: O(N) API calls (parallelizable)
- Leiden: O((N+E) log N) iterations (fast, local compute)
- Node2Vec: O(N × walks × length) (slow, parallelizable)

---

## Common Issues and Solutions

### Issue: LLM Extraction Quality Poor

**Symptom**: Missing entities, incorrect relationships, hallucinations

**Diagnosis**:
- Prompt not tuned for domain
- Insufficient few-shot examples
- Chunks too small or too large

**Solution**:
1. **Tune entity_extraction prompt**:
   ```yaml
   entity_extraction:
     prompt: |
       Extract entities of types: {entity_types}
       Focus on: [domain-specific guidance]
   ```

2. **Add domain-specific few-shot examples**:
   ```yaml
   few_shot_examples:
     - input: "[example from your domain]"
       output: "[expected extraction]"
   ```

3. **Adjust chunk size**:
   ```yaml
   chunk_size: 1500  # Increase for more context
   ```

4. **Use GPT-4** instead of GPT-3.5 (higher accuracy)

---

### Issue: Embedding API Rate Limits

**Symptom**: `RateLimitError` from OpenAI/Azure

**Solution**:
1. **Reduce parallelism**:
   ```yaml
   num_threads: 2  # Down from 4
   ```

2. **Add exponential backoff**:
   ```yaml
   max_retries: 10
   ```

3. **Increase API quota** (Azure OpenAI supports higher limits)

4. **Use local embedding models** (sentence-transformers):
   ```python
   from sentence_transformers import SentenceTransformer
   model = SentenceTransformer('all-MiniLM-L6-v2')
   ```

---

### Issue: Community Detection Produces Too Many Small Communities

**Symptom**: 1000s of tiny communities (1-2 entities each)

**Diagnosis**: Noisy graph (no pruning applied)

**Solution**:
1. **Enable graph pruning** before Leiden:
   ```yaml
   pruning:
     enabled: true
     min_node_freq: 3
     min_edge_weight_pct: 20
   ```

2. **Increase max_cluster_size**:
   ```yaml
   max_cluster_size: 50  # Allow larger communities
   ```

3. **Inspect removed nodes** to verify important entities not pruned

---

### Issue: Node2Vec Runs Out of Memory

**Symptom**: OOM during random walk generation

**Solution**:
1. **Reduce walk parameters**:
   ```yaml
   num_walks: 5       # From 10
   walk_length: 20    # From 40
   ```

2. **Reduce dimensions**:
   ```yaml
   dimensions: 256    # From 1536
   ```

3. **Extract largest connected component only**:
   ```yaml
   use_lcc: true
   ```

4. **Prune graph more aggressively** before embedding

---

## Future Enhancements

### 1. Local LLM Support (Privacy & Cost)

**Current**: OpenAI/Azure API (cloud, paid)

**Proposed**: Local LLMs (LLaMA, Mistral, Phi)
```python
from transformers import AutoModelForCausalLM

model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-3-8B")
# Run entity extraction locally (no API cost, full data privacy)
```

**Benefit**: $0 cost, data privacy, offline operation

---

### 2. Fine-Tuned Embedding Models

**Current**: Generic OpenAI embeddings

**Proposed**: Domain-specific fine-tuning
```python
# Collect positive/negative pairs from your corpus
pairs = [
    ("Microsoft founded OpenAI", "OpenAI research lab", 1.0),  # Similar
    ("Microsoft founded OpenAI", "Apple iPhone released", 0.1), # Dissimilar
]

# Fine-tune embedding model
fine_tuned_model = finetune_embeddings(base_model, pairs)
```

**Benefit**: 10-30% accuracy improvement on domain-specific retrieval

---

### 3. Graph Neural Networks (GNNs)

**Current**: Node2Vec (random walk + skip-gram)

**Proposed**: GNN embeddings (message passing)
- Graph Attention Networks (GAT)
- GraphSAGE (inductive learning)
- Multi-modal GNN (combine text + structure)

**Benefit**: Richer structural features, better few-shot learning

---

### 4. Incremental ML Updates

**Current**: Full re-indexing on new documents

**Proposed**: Incremental updates
- Re-extract entities only from new documents
- Incrementally update community assignments (hierarchical merging)
- Selective re-embedding (only changed nodes)

**Benefit**: 100x speedup for small document additions

---

## Getting Started

### Minimal Configuration (Local Search Only)

```yaml
graphrag:
  llm:
    model: gpt-3.5-turbo  # Fast, cheap
    api_key: ${OPENAI_API_KEY}

  entity_extraction:
    enabled: true
    summarize_descriptions: false  # Speed up indexing

  text_embedding:
    model: text-embedding-ada-002
    batch_size: 16

  community_detection:
    enabled: false  # Not needed for local search

  graph_embedding:
    enabled: false  # Not needed for local search
```

**Result**: Entity graph + text embeddings, local search supported

**Time**: ~30 min for 10K docs

**Cost**: ~$50

---

### Full Configuration (Global + Local Search + Visualization)

```yaml
graphrag:
  llm:
    model: gpt-4-turbo  # Best quality
    api_key: ${OPENAI_API_KEY}

  entity_extraction:
    enabled: true
    summarize_descriptions: true  # Consolidate descriptions

  text_embedding:
    model: text-embedding-3-large
    dimensions: 1536
    batch_size: 16

  community_detection:
    enabled: true
    algorithm: leiden
    max_cluster_size: 10

  graph_embedding:
    enabled: true
    algorithm: node2vec
    dimensions: 1536
    num_walks: 10
    walk_length: 40

  visualization:
    enabled: true
    algorithm: umap
    n_components: 2
```

**Result**: Full GraphRAG capabilities (global search, visualization)

**Time**: ~90 min for 10K docs

**Cost**: ~$150-200

---

## Conclusion

GraphRAG's ML algorithms provide **intelligence at multiple levels**:

**Deep Learning (Transformers)**:
- **LLMs**: Extract knowledge, generate summaries, answer questions
- **Text Embeddings**: Semantic understanding and retrieval

**Graph ML**:
- **Leiden**: Multi-scale community detection
- **Node2Vec**: Structural graph embeddings

**Classical ML**:
- **Punkt**: Sentence segmentation (preprocessing)
- **UMAP**: Dimensionality reduction (visualization)

**Key principles**:
- **External pre-trained models**: GraphRAG focuses on inference, not training
- **Hybrid intelligence**: Combine semantic (text) and structural (graph) signals
- **Scalable pipelines**: Parallel API calls, efficient graph algorithms
- **Production-ready**: API-based (OpenAI, Azure) and local (graspologic, UMAP)

**Integration**:
- LLMs extract structured knowledge from unstructured text
- Text embeddings enable semantic retrieval
- Graph ML discovers patterns in entity relationships
- Classical ML handles preprocessing and visualization

**Trade-offs**:
- **Quality vs. Cost**: GPT-4 (best) vs. GPT-3.5 (cheapest)
- **Speed vs. Accuracy**: Parallel API calls vs. rate limits
- **Memory vs. Quality**: Node2Vec dimensions (256 vs. 1536)

**Key files in this directory**:
- `01-llm-algorithms.md` — Transformer architecture, entity extraction, summarization
- `02-graph-ml-algorithms.md` — Leiden community detection, Node2Vec embeddings
- `03-embedding-and-nlp-algorithms.md` — Text embeddings, Punkt tokenizer, UMAP

**Next steps**:
- Read individual algorithm docs for deep dives into ML theory
- Consult decision trees for configuration guidance
- Cross-reference `spec/graph`, `spec/vectors`, `spec/dependencies` for system integration
- Review benchmarks for performance expectations
