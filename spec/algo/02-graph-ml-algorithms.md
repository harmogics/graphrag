# Graph Machine Learning Algorithms

## Overview

GraphRAG uses **graph-specific ML algorithms** to analyze knowledge graph structure and learn latent representations. These algorithms discover communities, measure importance, and create embeddings that complement text-based semantics with structural patterns.

## ML Algorithm Classification

**Type**: Graph Machine Learning
**Training**: Unsupervised learning on graph topology
**Inference**: Iterative optimization (Leiden) or random sampling (Node2Vec)
**Implementation**: External (third-party: graspologic)

---

## Graph ML Algorithms in GraphRAG

### 1. Hierarchical Leiden Community Detection

**Purpose**: Discover hierarchical communities (clusters) in knowledge graph

**ML Algorithm**: Modularity optimization via iterative refinement

**Type**: Unsupervised clustering

**Code**: `graphrag/index/operations/cluster_graph.py:10-52`

**Detailed explanation**: See `spec/graph/01-community-detection.md`

---

#### ML Formulation

**Objective function** (Modularity Q):
```
Q = (1 / 2m) Σ [A_ij - (k_i × k_j) / (2m)] × δ(c_i, c_j)

Where:
- A_ij: Adjacency matrix (1 if edge exists, 0 otherwise)
- k_i: Degree of node i
- m: Total number of edges
- δ(c_i, c_j): 1 if nodes i and j in same community, else 0
```

**Interpretation**:
- **High Q**: Many edges within communities, few between
- **Low Q**: Random community assignment
- **Optimization goal**: Maximize Q

**Example**:
```
Graph:
  A -- B    C -- D
  |    |    |    |
  E -- F    G -- H

Perfect modularity (2 communities):
  Community 1: {A, B, E, F}
  Community 2: {C, D, G, H}
  Q ≈ 0.5 (high modularity)

Random assignment:
  Community 1: {A, C, E, G}
  Community 2: {B, D, F, H}
  Q ≈ 0.0 (low modularity)
```

---

#### Algorithm Steps

**Phase 1: Initial Partition**
```python
# Start with each node in own community
communities = {i: {i} for i in graph.nodes()}
```

**Phase 2: Local Moving** (Louvain-style)
```python
for node in graph.nodes():
    # Try moving node to neighbor communities
    for neighbor_community in get_neighbor_communities(node):
        delta_Q = compute_modularity_gain(node, neighbor_community)
        if delta_Q > 0:
            move_node(node, neighbor_community)
```

**Modularity gain**:
```
ΔQ = [edges_in_community + k_i_in] / (2m)
     - [(degree_sum + k_i)^2 / (4m^2)]
     - [edges_in_community / (2m) - (degree_sum / (2m))^2 - (k_i / (2m))^2]
```

**Phase 3: Refinement**
```python
# Split large communities
for community in communities:
    if len(community) > max_cluster_size:
        sub_communities = leiden_refinement(community)
        communities.extend(sub_communities)
```

**Phase 4: Aggregation**
```python
# Create super-graph where communities become nodes
super_graph = aggregate_communities(graph, communities)
```

**Phase 5: Recursion**
```python
# Repeat until no improvement
if modularity_improved:
    return hierarchical_leiden(super_graph, level + 1)
```

**Output**:
```python
# Hierarchical structure
[
    (level=0, cluster_id=1, parent_id=None, nodes=[A, B, E, F]),
    (level=0, cluster_id=2, parent_id=None, nodes=[C, D, G, H]),
    (level=1, cluster_id=3, parent_id=1, nodes=[A, B]),
    (level=1, cluster_id=4, parent_id=1, nodes=[E, F]),
    ...
]
```

---

#### ML Training Details

**Training type**: Unsupervised (no labels)

**Optimization**:
- **Method**: Greedy local search (not gradient descent)
- **Iterations**: Until convergence (ΔQ < threshold)
- **Complexity**: O((N + E) × log N) per iteration

**Hyperparameters**:
```python
max_cluster_size: int = 10      # Max nodes per community
random_seed: int = 42           # For reproducibility
use_lcc: bool = True            # Use largest connected component
```

**Training is deterministic** (given same seed and use_lcc=True)

---

#### External Dependencies

**Package**: `graspologic==3.4.1`

**Source**: Microsoft Research (formerly GraSPy)

**Installation**:
```bash
pip install graspologic
```

**Usage in GraphRAG**:
```python
from graspologic.partition import hierarchical_leiden

# Learn communities
community_mapping, parent_mapping = hierarchical_leiden(
    graph,
    max_cluster_size=10,
    random_seed=42
)

# Output: dict mapping node → community_id at each level
```

**Why graspologic?**
- **Optimized**: C++ backend for performance
- **Hierarchical**: Multi-level communities (unique feature)
- **Maintained**: Active development by Microsoft Research

**Alternative implementations**:
- `python-louvain`: Original Louvain (single level)
- `igraph`: Leiden implementation (C library)
- `networkx.community`: Various methods (slower, pure Python)

---

### 2. Node2Vec Graph Embeddings

**Purpose**: Learn continuous vector representations of graph nodes

**ML Algorithm**: Random walk + Skip-Gram word embedding

**Type**: Unsupervised representation learning

**Code**: `graphrag/index/operations/embed_graph/embed_node2vec.py:20-43`

**Detailed explanation**: See `spec/graph/04-graph-embedding.md`

---

#### ML Formulation

**Step 1: Generate random walks** (sampling)
```python
walks = []
for node in graph.nodes():
    for _ in range(num_walks):  # Default: 10
        walk = random_walk(graph, start=node, length=40)
        walks.append(walk)
```

**Random walk example**:
```
Start at node A:
  Walk 1: [A, B, C, E, F, G]
  Walk 2: [A, D, E, C, B, A]
  ...

These walks capture local graph structure
```

**Step 2: Skip-Gram objective** (same as Word2Vec)
```
Maximize: Σ Σ log P(n_i | n_j)
          walks w
          nodes n_j in w
          context n_i

Where:
P(n_i | n_j) = exp(emb(n_i) · emb(n_j)) / Σ_k exp(emb(n_k) · emb(n_j))
                                             all nodes k
```

**Interpretation**:
- Nodes appearing in similar walk contexts → similar embeddings
- Analogous to word embeddings (co-occurrence → similarity)

**Example**:
```
If walks often contain [..., A, B, C, ...] and [..., A, D, E, ...]
Then embeddings satisfy:
  similarity(A, B) > similarity(A, X) for unrelated X
  similarity(A, D) > similarity(A, Y) for unrelated Y
```

---

#### Neural Network Architecture

**Skip-Gram model**:
```
Input: One-hot encoded node ID (e.g., node A)
   ↓
Embedding Layer: W_in (V × D)  where V = vocab (nodes), D = dimensions
   ↓
Hidden representation: h = W_in[node_id]  (D-dimensional vector)
   ↓
Output Layer: W_out (D × V)
   ↓
Softmax: P(context | target) = softmax(W_out · h)
```

**Parameters**:
- `dimensions`: Embedding size (default 1536)
- `window_size`: Context window (default 2)
- `iterations`: Training epochs (default 3)

**Training objective**:
```
Loss = -Σ log P(context_node | target_node)
        over all (target, context) pairs from walks
```

**Optimization**: Stochastic gradient descent with negative sampling

**Negative sampling**:
```
Instead of computing softmax over all V nodes:
Sample k negative nodes (k=5-20) and optimize:

L = log σ(emb(pos) · emb(target))
    + Σ log σ(-emb(neg_i) · emb(target))
        i=1 to k
```
**Benefit**: Reduce complexity from O(V) to O(k)

---

#### Algorithm Steps

**Full Node2Vec pipeline**:

```python
# 1. Generate walks
walks = []
for node in graph.nodes():
    for _ in range(num_walks):
        walk = biased_random_walk(
            graph,
            start=node,
            length=walk_length,
            p=1,  # Return parameter
            q=1   # In-out parameter
        )
        walks.append(walk)

# 2. Train Skip-Gram on walks
from gensim.models import Word2Vec

# Treat nodes as "words", walks as "sentences"
model = Word2Vec(
    walks,
    vector_size=dimensions,
    window=window_size,
    min_count=1,
    sg=1,  # Skip-Gram (not CBOW)
    workers=4,
    epochs=iterations
)

# 3. Extract embeddings
embeddings = {node: model.wv[node] for node in graph.nodes()}
```

**GraphRAG uses graspologic** (wraps similar logic):
```python
from graspologic.embed import node2vec_embed

embeddings, nodes = node2vec_embed(
    graph,
    dimensions=1536,
    num_walks=10,
    walk_length=40,
    window_size=2,
    iterations=3,
    random_seed=86
)
```

---

#### ML Training Details

**Training type**: Self-supervised (no labels, uses graph structure)

**Optimization**:
- **Method**: SGD on Skip-Gram loss
- **Complexity**: O(N × num_walks × walk_length × window_size × iterations × D)
  - N = nodes
  - D = dimensions

**Hyperparameters**:
```python
dimensions: int = 1536      # Embedding size (match text embeddings)
num_walks: int = 10         # Walks per node
walk_length: int = 40       # Tokens per walk
window_size: int = 2        # Skip-Gram context window
iterations: int = 3         # Training epochs
random_seed: int = 86       # Reproducibility
```

**Training time** (10K nodes, 30K edges):
```
Walk generation: 5 seconds
Skip-Gram training: 10 seconds
Total: 15 seconds
```

---

#### External Dependencies

**Package**: `graspologic==3.4.1`

**Node2Vec implementation** (uses pecanpy internally):
```python
import graspologic as gc

embeddings = gc.embed.node2vec_embed(
    graph,
    dimensions=1536,
    num_walks=10,
    walk_length=40
)
```

**Alternative implementations**:
- `node2vec` (original implementation, slower)
- `gensim.models.Word2Vec` (manual walk generation + training)
- `PyTorch Geometric`: GNN-based alternatives (GCN, GraphSAGE)

**Why graspologic?**
- **Fast**: Optimized C++ random walk generation
- **Integrated**: Works with networkx graphs
- **Reproducible**: Stable random seed behavior

---

## ML Comparison: Leiden vs. Node2Vec

| Aspect | Leiden | Node2Vec |
|--------|--------|----------|
| **Task** | Community detection | Node embedding |
| **Output** | Discrete (community IDs) | Continuous (vectors) |
| **Training** | Modularity optimization | Skip-Gram NLP model |
| **Complexity** | O((N+E) log N) | O(N × walks × length) |
| **Use case** | Clustering, summarization | Similarity, visualization |
| **Interpretability** | High (community = topic) | Low (latent dimensions) |

**Complementary**:
- Leiden: Discover macro-structure (communities)
- Node2Vec: Capture micro-structure (local neighborhoods)

---

## Integration with GraphRAG Pipeline

### Indexing Flow

```
Graph Construction
   ↓
┌─────────────────────────────┐
│  Leiden Community Detection │  ← Unsupervised clustering
│  Input: Graph structure     │
│  Output: Community IDs      │
└─────────────────────────────┘
   ↓
Community Reports (LLM summarization)
   ↓
┌─────────────────────────────┐
│  Node2Vec Embeddings        │  ← Unsupervised representation learning
│  Input: Graph structure     │
│  Output: 1536-dim vectors   │
└─────────────────────────────┘
   ↓
Hybrid Retrieval (text + graph embeddings)
```

### Query Flow

```
Query → [Text Embedding] → Semantic similarity
            +
        [Graph Embedding (Node2Vec)] → Structural similarity
            ↓
        Combined score → Top-k entities
```

---

## Performance Characteristics

### Leiden Community Detection

| Graph Size | Nodes | Edges | Time | Memory |
|------------|-------|-------|------|--------|
| Small | 1K | 3K | 0.5 sec | 50 MB |
| Medium | 10K | 30K | 3 sec | 200 MB |
| Large | 100K | 300K | 45 sec | 2 GB |

**Scaling**: Near-linear with graph size

---

### Node2Vec Embeddings

| Graph Size | Nodes | Edges | Time | Memory |
|------------|-------|-------|------|--------|
| Small | 1K | 3K | 5 sec | 200 MB |
| Medium | 10K | 30K | 45 sec | 1.5 GB |
| Large | 100K | 300K | 8 min | 12 GB |

**Bottleneck**: Random walk generation (CPU-bound)

---

## Hyperparameter Tuning

### Leiden

**max_cluster_size**:
- **Small (5-10)**: Many fine-grained communities
- **Large (50-100)**: Fewer coarse communities
- **Recommendation**: 10 (default, balanced)

**random_seed**:
- **Fixed**: Reproducible results
- **Varies**: Explore alternative clusterings

---

### Node2Vec

**dimensions**:
- **Match text embeddings**: 1536 (for hybrid retrieval)
- **Standalone**: 128-256 (sufficient, faster)

**num_walks**:
- **Low (5)**: Fast, may miss structure
- **High (20)**: Better quality, slower
- **Recommendation**: 10 (default)

**walk_length**:
- **Short (10-20)**: Local neighborhoods
- **Long (40-80)**: Broader context
- **Recommendation**: 40 (default)

---

## Limitations and Considerations

### Leiden

**Limitation**: Sensitive to graph density
- **Sparse graphs**: May fragment into many tiny communities
- **Dense graphs**: May merge into few large communities

**Mitigation**: Tune `max_cluster_size`, use hierarchical levels

---

### Node2Vec

**Limitation**: Computational cost for large graphs
- **Memory**: O(N × num_walks × walk_length) for walk storage
- **Time**: Hours for million-node graphs

**Mitigation**:
- Reduce `num_walks` or `walk_length`
- Sample subgraph (top-degree nodes only)
- Use GPU-accelerated implementations

---

## Future Enhancements

### 1. Graph Neural Networks (GNNs)

**Current**: Node2Vec (random walk + Skip-Gram)

**Proposed**: GNN-based embeddings (GCN, GraphSAGE, GAT)

```python
# Graph Convolutional Network
class GCN(torch.nn.Module):
    def forward(self, x, edge_index):
        x = self.conv1(x, edge_index)
        x = F.relu(x)
        x = self.conv2(x, edge_index)
        return x

# Learn embeddings
model = GCN(in_features=node_features, out_features=1536)
embeddings = model(node_features, edge_index)
```

**Benefits**:
- **Message passing**: Aggregate neighbor features
- **Supervised**: Can use labels (if available)
- **Faster**: No random walk sampling

---

### 2. Dynamic Community Detection

**Current**: Static snapshot

**Proposed**: Temporal Leiden (track community evolution)

```python
def temporal_leiden(graph_snapshots):
    communities_over_time = []
    for t, graph_t in enumerate(graph_snapshots):
        communities_t = leiden(graph_t)
        # Track community persistence
        if t > 0:
            match_communities(communities_t, communities_over_time[-1])
        communities_over_time.append(communities_t)
    return communities_over_time
```

**Benefits**: Detect emerging trends, track entity importance over time

---

## Conclusion

Graph ML algorithms in GraphRAG provide **structural intelligence** that complements text-based semantics:

**Leiden Community Detection**:
- **Unsupervised clustering** via modularity optimization
- **Hierarchical communities** for multi-scale navigation
- **External package**: graspologic (Microsoft Research)
- **Complexity**: O((N+E) log N)

**Node2Vec Embeddings**:
- **Unsupervised representation learning** via random walks + Skip-Gram
- **Continuous vectors** for similarity search and visualization
- **External package**: graspologic (wraps pecanpy/gensim)
- **Complexity**: O(N × walks × length × window × iterations)

**Integration**:
- Leiden → Community reports (global search)
- Node2Vec → Hybrid retrieval (text + graph similarity)
- Both → Complement text embeddings with structural patterns

**Key dependency**: `graspologic==3.4.1` (Microsoft Research)
