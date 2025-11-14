# Embedding and NLP Algorithms

## Overview

GraphRAG uses **neural text embeddings** and **classical NLP algorithms** to transform text into semantic representations and extract linguistic structures. These ML components enable semantic search, sentence segmentation, and dimensionality reduction for visualization.

## ML Algorithm Classification

**Type**: Mixed
- **Text Embeddings**: Deep learning (Transformer neural networks)
- **NLP**: Classical ML (unsupervised probabilistic models)
- **Dimensionality Reduction**: Manifold learning

**Implementation**: External (third-party packages)

---

## 1. Text Embedding Algorithms

### 1.1 Transformer-Based Text Embeddings

**Purpose**: Convert text to dense semantic vectors

**ML Algorithm**: Encoder-only Transformer (BERT-style)

**Models used**:
- `text-embedding-ada-002` (OpenAI)
- `text-embedding-3-small` (OpenAI)
- `text-embedding-3-large` (OpenAI)

**Code**: `graphrag/index/operations/embed_text/strategies/openai.py`

**Detailed explanation**: See `spec/vectors/01-text-embedding.md`

---

#### Neural Architecture

**Base architecture**: Transformer Encoder (similar to BERT)

```
Input: Tokenized text
   ↓
┌─────────────────────────────────┐
│  Token Embedding Layer          │
│  + Positional Encoding          │
└─────────────────────────────────┘
   ↓
┌─────────────────────────────────┐
│  Transformer Encoder Layer 1    │
│  - Multi-Head Self-Attention    │
│  - Layer Norm                   │
│  - Feed-Forward Network         │
│  - Layer Norm                   │
└─────────────────────────────────┘
   ↓
   ... (12-24 layers)
   ↓
┌─────────────────────────────────┐
│  Pooling Layer                  │
│  (mean pooling or [CLS] token)  │
└─────────────────────────────────┘
   ↓
Output: 1536-dim vector (normalized)
```

**Key components**:

1. **Self-Attention**:
   ```
   Attention(Q, K, V) = softmax(QK^T / sqrt(d_k)) V
   ```
   **Learns**: Contextual relationships between tokens

2. **Multi-Head Attention**:
   ```
   MultiHead(Q, K, V) = Concat(head_1, ..., head_h) W^O
   ```
   **Learns**: Different semantic aspects (syntax, semantics, etc.)

3. **Feed-Forward Network**:
   ```
   FFN(x) = max(0, xW_1 + b_1) W_2 + b_2
   ```
   **Learns**: Non-linear transformations

4. **Layer Normalization**:
   ```
   LayerNorm(x) = γ (x - μ) / σ + β
   ```
   **Stabilizes**: Deep network training

---

#### ML Training Details

**Training type**: Self-supervised contrastive learning

**Objective** (Contrastive loss - simplified):
```
Minimize: -log(exp(sim(text, positive) / τ) /
               Σ exp(sim(text, negative_i) / τ))

Where:
- sim(a, b) = cosine_similarity(embed(a), embed(b))
- τ = temperature parameter
- positive = semantically similar text
- negative_i = dissimilar texts
```

**Training data** (OpenAI models):
- Billions of (text, similar_text) pairs
- Mined from web, books, code

**Training is external** - GraphRAG uses pre-trained models via API

---

#### External Dependencies

**Package**: `openai==1.x`

**API call**:
```python
from openai import OpenAI

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

response = client.embeddings.create(
    model="text-embedding-ada-002",
    input=["Text to embed"],
    encoding_format="float"
)

embedding = response.data[0].embedding  # 1536-dim list
```

**Parameters**:
- **model**: `text-embedding-ada-002` (default), `text-embedding-3-small/large`
- **input**: Text or list of texts (max 8191 tokens per text)
- **dimensions**: Optional (for text-embedding-3-*, e.g., 512, 1536, 3072)

---

#### Performance Characteristics

**Latency** (per API call):
- Single text: ~100-200ms
- Batch (16 texts): ~500ms

**Cost** (as of 2024):
```
text-embedding-ada-002:  $0.10 per 1M tokens
text-embedding-3-small:  $0.02 per 1M tokens (5x cheaper)
text-embedding-3-large:  $0.13 per 1M tokens (better quality)
```

**Throughput** (GraphRAG with parallel batching):
- ~2000 embeddings/minute (limited by API rate limits)

---

### 1.2 Comparison: Text vs. Graph Embeddings

| Aspect | Text Embeddings | Graph Embeddings (Node2Vec) |
|--------|----------------|------------------------------|
| **Input** | Text (tokens) | Graph structure (edges) |
| **Algorithm** | Transformer | Random walk + Skip-Gram |
| **Dimensions** | 1536 (fixed) | 1536 (configurable) |
| **Training** | Supervised contrastive | Self-supervised (unsupervised) |
| **Captures** | Semantic meaning | Structural relationships |
| **Use case** | Semantic search | Structural similarity |

**Complementary**: GraphRAG combines both for hybrid retrieval

---

## 2. NLP Algorithms

### 2.1 Punkt Sentence Tokenizer (NLTK)

**Purpose**: Segment text into sentences

**ML Algorithm**: Unsupervised statistical model

**Type**: Classical ML (decision tree-like)

**Code**: Used in sentence-based chunking strategy

**From**: `graphrag/index/operations/chunk_text/strategies.py:58-69`

```python
import nltk

sentences = nltk.sent_tokenize(text)
# Input: "Dr. Smith works at MIT. He studies AI."
# Output: ["Dr. Smith works at MIT.", "He studies AI."]
```

---

#### ML Algorithm Details

**Punkt** = **Unsupervised sentence boundary detection**

**Training** (done by NLTK, pre-trained models):

1. **Abbreviation detection**:
   ```
   Learn: "Dr.", "Inc.", "Prof." are abbreviations (not sentence ends)
   Features: Token length, capitalization patterns, context
   ```

2. **Collocation detection**:
   ```
   Learn: "Mr. Smith" is a collocation (period is abbreviation)
   Features: Token co-occurrence statistics
   ```

3. **Sentence starter detection**:
   ```
   Learn: Capitalized words after period → likely sentence start
   Features: Orthographic features
   ```

**Decision process**:
```python
for period in text.periods():
    is_abbreviation = check_abbreviation_list(token_before_period)
    is_collocation = check_collocation(token_before, token_after)
    is_sentence_starter = is_capitalized(token_after)

    if not is_abbreviation and not is_collocation and is_sentence_starter:
        mark_sentence_boundary(period)
```

---

#### ML Model Type

**Type**: Probabilistic decision tree

**Training**: Unsupervised (no labeled data)

**Algorithm**:
1. Extract collocation candidates (frequent bigrams)
2. Compute log-likelihood ratio for abbreviations
3. Build decision rules based on statistics

**Pre-trained models** (NLTK provides):
- English: `nltk_data/tokenizers/punkt/english.pickle`
- German, French, Spanish, etc.: Also available

**Model size**: ~150KB (compact statistical model)

---

#### External Dependencies

**Package**: `nltk==3.8.1`

**Installation**:
```bash
pip install nltk

# Download Punkt model
python -c "import nltk; nltk.download('punkt')"
```

**Usage**:
```python
import nltk

# Ensure Punkt model is downloaded
nltk.download('punkt')

# Tokenize
text = "Dr. Smith works at MIT. He studies AI in Cambridge, Mass."
sentences = nltk.sent_tokenize(text)

# Output:
# [
#   "Dr. Smith works at MIT.",
#   "He studies AI in Cambridge, Mass."
# ]
```

**Why NLTK?**
- **Accurate**: Handles abbreviations, honorifics correctly
- **Fast**: Rule-based (not deep learning, instant inference)
- **Multi-language**: Pre-trained for 17 languages
- **Lightweight**: No GPU required

---

#### Performance

**Latency**: < 1ms per document (very fast)

**Accuracy** (English):
- ~98% precision on standard benchmarks
- Handles edge cases: "Dr.", "U.S.A.", "Inc."

**Comparison with alternatives**:
```
SpaCy sentence tokenizer:    99% accuracy, 10x slower
NLTK Punkt:                  98% accuracy, fast
Regex-based (naive):         80% accuracy, fast
```

**GraphRAG uses NLTK** for sentence-based chunking (optional strategy)

---

## 3. Dimensionality Reduction (UMAP)

### 3.1 UMAP Algorithm

**Purpose**: Reduce high-dimensional embeddings to 2D/3D for visualization

**ML Algorithm**: Manifold learning (topological data analysis)

**Type**: Unsupervised dimensionality reduction

**Code**: `graphrag/index/operations/layout_graph/umap.py`

```python
from umap import UMAP

reducer = UMAP(
    n_components=2,      # Output dimensions
    n_neighbors=15,      # Local neighborhood size
    min_dist=0.1,        # Minimum distance in low-dim space
    metric='cosine',     # Distance metric
    random_state=42
)

# Input: 1536-dim embeddings
# Output: 2D coordinates for visualization
coords_2d = reducer.fit_transform(embeddings)
```

---

#### ML Algorithm Details

**UMAP** = **Uniform Manifold Approximation and Projection**

**Goal**: Preserve both local and global structure in low dimensions

**Mathematical formulation** (simplified):

1. **Build k-NN graph** in high-dimensional space:
   ```
   For each point x_i:
       Find k nearest neighbors
       Create fuzzy simplicial set (topological structure)
   ```

2. **Optimize low-dimensional layout**:
   ```
   Minimize: Cross-entropy between high-dim and low-dim fuzzy sets

   Loss = Σ [log(1 - v_ij) - log(1 - w_ij)]
          edges

   Where:
   - v_ij: High-dim edge probability
   - w_ij: Low-dim edge probability
   ```

3. **Stochastic gradient descent**:
   ```
   Update positions to minimize loss
   Iterations: ~200-500 epochs
   ```

---

#### Algorithm Steps

```python
# 1. Construct high-dimensional graph
knn_graph = construct_knn_graph(
    data,
    n_neighbors=15,
    metric='cosine'
)

# 2. Convert to fuzzy topological representation
fuzzy_simplicial_set = compute_membership_strengths(knn_graph)

# 3. Initialize low-dimensional embedding
embedding_2d = initialize_embedding(
    n_samples,
    n_components=2,
    random_state=42
)

# 4. Optimize via SGD
for epoch in range(n_epochs):
    for edge in fuzzy_simplicial_set:
        gradient = compute_gradient(edge, embedding_2d)
        embedding_2d -= learning_rate * gradient

return embedding_2d
```

---

#### ML Training Details

**Training type**: Unsupervised

**Optimization**:
- **Method**: Stochastic gradient descent
- **Objective**: Preserve topological structure
- **Complexity**: O(N × log N) for k-NN, O(N × epochs) for SGD

**Hyperparameters**:
```python
n_neighbors: int = 15    # Larger = preserve global structure
min_dist: float = 0.1    # Smaller = tight clusters
metric: str = 'cosine'   # For embeddings (unit vectors)
n_epochs: int = 200      # More = better convergence
```

---

#### External Dependencies

**Package**: `umap-learn==0.5.4`

**Installation**:
```bash
pip install umap-learn
```

**Usage in GraphRAG**:
```python
from umap import UMAP
import numpy as np

# Node2Vec embeddings (1536-dim)
embeddings = node2vec(graph, dimensions=1536)
embedding_matrix = np.array([embeddings[node] for node in sorted(embeddings.keys())])

# Reduce to 2D
reducer = UMAP(n_components=2, metric='cosine', random_state=42)
coords_2d = reducer.fit_transform(embedding_matrix)

# Visualize
import matplotlib.pyplot as plt
plt.scatter(coords_2d[:, 0], coords_2d[:, 1], alpha=0.5)
for i, node in enumerate(sorted(embeddings.keys())):
    plt.annotate(node, coords_2d[i])
plt.show()
```

**Why UMAP?**
- **Better than t-SNE**: Preserves global structure (t-SNE only local)
- **Faster**: 10-100x speedup vs. t-SNE on large datasets
- **Deterministic**: Fixed random seed → reproducible results
- **Scalable**: Handles millions of points

---

#### Performance

**Latency** (10K points, 1536-dim → 2D):
```
t-SNE:  ~10 minutes
UMAP:   ~1 minute (10x faster)
PCA:    ~1 second (fast but loses structure)
```

**Memory**: O(N × D) where D = dimensions

**Use case in GraphRAG**:
- Visualize entity graph (nodes as 2D points)
- Explore community structure visually
- Identify outliers and clusters

---

## Algorithm Comparison

| Algorithm | Type | Training | Package | Use Case |
|-----------|------|----------|---------|----------|
| **Text Embedding** | Deep Learning (Transformer) | Supervised contrastive (external) | `openai` | Semantic search |
| **Punkt Tokenizer** | Classical ML (probabilistic) | Unsupervised (pre-trained) | `nltk` | Sentence segmentation |
| **UMAP** | Manifold learning | Unsupervised (on data) | `umap-learn` | Visualization |

**Common trait**: All are **external implementations** (not custom GraphRAG code)

---

## Integration with GraphRAG

### Indexing Flow

```
Documents
   ↓
[Punkt: Sentence segmentation] (optional, for sentence-based chunking)
   ↓
Text Units
   ↓
[Text Embedding: Transformer] → 1536-dim vectors
   ↓
Vector Store
```

### Visualization Flow

```
Graph Embeddings (Node2Vec)
   ↓
[UMAP: Dimensionality reduction] → 2D coordinates
   ↓
Graph Visualization (Matplotlib, D3.js)
```

---

## External Dependency Summary

| Package | Version | Purpose | ML Algorithm |
|---------|---------|---------|--------------|
| `openai` | 1.x | Text embeddings | Transformer encoder |
| `nltk` | 3.8.1 | Sentence tokenization | Punkt (unsupervised) |
| `umap-learn` | 0.5.4 | Dimensionality reduction | Manifold learning |

**All packages are third-party** - GraphRAG does not implement these algorithms internally.

---

## Future Enhancements

### 1. Local Embedding Models

**Current**: OpenAI API (cloud)

**Proposed**: Sentence-BERT, all-MiniLM-L6-v2 (local)

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('all-MiniLM-L6-v2')
embeddings = model.encode(texts)  # Run locally (no API cost)
```

**Benefits**: No API cost, data privacy, offline operation

---

### 2. Advanced Tokenization

**Current**: NLTK Punkt (sentence-level)

**Proposed**: SpaCy with NER

```python
import spacy

nlp = spacy.load("en_core_web_sm")
doc = nlp(text)

for ent in doc.ents:
    print(ent.text, ent.label_)  # Named entity recognition
```

**Benefits**: Entity recognition, part-of-speech tagging, dependency parsing

---

### 3. Dimensionality Reduction Alternatives

**Current**: UMAP

**Proposed**: t-SNE, PaCMAP, TriMap

**Trade-offs**:
- **t-SNE**: Better local structure, slower
- **PaCMAP**: Balanced local/global, newer
- **TriMap**: Faster than UMAP, similar quality

---

## Conclusion

Embedding and NLP algorithms provide **semantic and linguistic intelligence** to GraphRAG:

**Text Embeddings** (Transformer):
- **Purpose**: Semantic vector representation
- **Algorithm**: Encoder-only Transformer (BERT-style)
- **Package**: `openai` (external API)
- **Training**: Supervised contrastive (pre-trained by OpenAI)

**Punkt Tokenizer** (Classical ML):
- **Purpose**: Sentence segmentation
- **Algorithm**: Unsupervised statistical model
- **Package**: `nltk` (pre-trained models)
- **Training**: Unsupervised (abbreviation/collocation detection)

**UMAP** (Manifold Learning):
- **Purpose**: Visualization (1536D → 2D)
- **Algorithm**: Topological data analysis + SGD
- **Package**: `umap-learn`
- **Training**: Unsupervised (on data)

**Key insight**: GraphRAG leverages **state-of-the-art pre-trained models** from external packages, focusing integration effort on pipeline orchestration rather than ML model development.
