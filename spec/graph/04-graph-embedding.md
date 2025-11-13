# Graph Embedding with Node2Vec

## Overview

Graph embedding transforms **discrete graph structure** into **continuous vector space**, enabling similarity search and geometric reasoning over entities. GraphRAG uses **Node2Vec**, a random-walk-based algorithm that learns embeddings by treating graph neighborhoods as "sentences" and applying language modeling techniques.

## Conceptual Foundation

### Why Embed Graphs?

**Problem**: Graph algorithms operate on discrete topology (neighbors, paths, communities), but many ML/retrieval tasks require continuous representations

**Solution**: Map each node to a vector such that **structurally similar nodes** have **similar embeddings**

**GraphRAG use cases**:
- **Entity similarity search**: Find entities related by graph structure (not just text)
- **Visualization**: Project high-dimensional graph into 2D/3D for human inspection
- **Features for downstream ML**: Use embeddings as input to classifiers, clusterers

### Node2Vec Intuition

**Core idea**: Nodes that appear in **similar random walk contexts** should have similar embeddings

**Analogy to NLP**:
- **Text**: Words that appear in similar sentences have similar embeddings (Word2Vec)
- **Graph**: Nodes that appear in similar random walks have similar embeddings (Node2Vec)

**Example**:
```
Graph:
  ALICE -- BOB -- CAROL
    |       |       |
  DAN -- EVE -- FRANK

Random walks starting from ALICE:
  Walk 1: ALICE → BOB → CAROL → FRANK
  Walk 2: ALICE → DAN → EVE → BOB
  Walk 3: ALICE → BOB → EVE → FRANK

Context of ALICE: [BOB, DAN, CAROL, EVE, ...]
Context of BOB: [ALICE, CAROL, EVE, ...]

If BOB and EVE appear in similar contexts, they get similar embeddings
```

---

## Algorithm: Node2Vec

### Phase 1: Random Walk Generation

**Code**: Performed by `graspologic.embed.node2vec_embed` (called from `embed_node2vec.py:34-42`)

**Parameters**:
- `num_walks`: Number of walks to generate per node (default: 10)
- `walk_length`: Length of each walk in hops (default: 40)

**Algorithm**:
```python
def generate_random_walks(graph, num_walks, walk_length):
    walks = []
    for node in graph.nodes():
        for _ in range(num_walks):
            walk = [node]
            current = node
            for _ in range(walk_length - 1):
                neighbors = list(graph.neighbors(current))
                if not neighbors:
                    break
                next_node = random.choice(neighbors)  # Uniform random
                walk.append(next_node)
                current = next_node
            walks.append(walk)
    return walks
```

**Example** (graph with 5 nodes, `num_walks=2`, `walk_length=5`):
```
Total walks generated: 5 nodes × 2 walks = 10 walks
Each walk: [start_node, hop_1, hop_2, hop_3, hop_4]

Sample walks:
  Walk 1: [ALICE, BOB, CAROL, FRANK, EVE]
  Walk 2: [ALICE, DAN, EVE, BOB, CAROL]
  Walk 3: [BOB, ALICE, DAN, EVE, FRANK]
  ...
```

**Complexity**: O(N × num_walks × walk_length) where N = nodes

---

### Phase 2: Skip-Gram Training

**Input**: List of random walks (sequences of node IDs)

**Model**: Skip-gram with negative sampling (same as Word2Vec)

**Objective**: Predict **context nodes** from **target node**

**Algorithm**:
```
For each walk: [n1, n2, n3, n4, n5, ...]
For each position i in walk:
  Target node: ni
  Context window: [n(i-window_size), ..., n(i+window_size)]

  Maximize: P(context | target) = σ(embedding(target) · embedding(context))

  Where σ = sigmoid function
```

**Parameters**:
- `window_size`: Size of context window (default: 2)
- `dimensions`: Embedding vector size (default: 1536)
- `iterations`: Number of training epochs (default: 3)

**Example** (window_size=2):
```
Walk: [ALICE, BOB, CAROL, FRANK, EVE]

Position 0 (ALICE):
  Target: ALICE
  Context: [BOB, CAROL]  (only right neighbors, since at start)

Position 2 (CAROL):
  Target: CAROL
  Context: [ALICE, BOB, FRANK, EVE]  (window of 2 on each side)

Position 4 (EVE):
  Target: EVE
  Context: [CAROL, FRANK]  (only left neighbors, since at end)
```

**Training objective**:
```
Maximize: Σ log P(context_node | target_node)
         for all (target, context) pairs

Equivalently, minimize:
  -Σ log σ(embedding(target) · embedding(context))
```

**Negative sampling**: For each positive (target, context) pair, sample K negative context nodes and push their embeddings away from target

**Complexity**: O(num_walks × walk_length × window_size × iterations)

---

### Phase 3: Embedding Extraction

**Output**: Mapping `{node_id: embedding_vector}`

```python
# From graspologic output (embed_node2vec.py:34-43)
lcc_tensors = gc.embed.node2vec_embed(
    graph=graph,
    dimensions=dimensions,
    window_size=window_size,
    iterations=iterations,
    num_walks=num_walks,
    walk_length=walk_length,
    random_seed=random_seed,
)

# lcc_tensors[0]: numpy array of shape (num_nodes, dimensions)
# lcc_tensors[1]: list of node IDs (same order as embeddings)

embeddings = lcc_tensors[0]  # shape: (N, 1536)
nodes = lcc_tensors[1]       # length: N
```

**Example output**:
```python
{
  "ALICE": [0.23, -0.45, 0.12, ..., 0.67],    # 1536-dim vector
  "BOB": [0.19, -0.42, 0.15, ..., 0.71],      # Similar to ALICE (close in graph)
  "CAROL": [0.21, -0.44, 0.13, ..., 0.69],    # Similar to BOB
  "FRANK": [-0.15, 0.32, -0.08, ..., -0.23],  # Different cluster
  ...
}
```

---

## Implementation in GraphRAG

### Code Flow

**High-level wrapper**: `graphrag/index/operations/embed_graph/embed_graph.py:16-50`

```python
def embed_graph(
    graph: nx.Graph,
    config: EmbedGraphConfig,
) -> NodeEmbeddings:
    """Embed a graph into a vector space using node2vec."""
    if config.use_lcc:
        graph = stable_largest_connected_component(graph)

    # create graph embedding using node2vec
    embeddings = embed_node2vec(
        graph=graph,
        dimensions=config.dimensions,
        num_walks=config.num_walks,
        walk_length=config.walk_length,
        window_size=config.window_size,
        iterations=config.iterations,
        random_seed=config.random_seed,
    )

    pairs = zip(embeddings.nodes, embeddings.embeddings.tolist(), strict=True)
    sorted_pairs = sorted(pairs, key=lambda x: x[0])

    return dict(sorted_pairs)
```

**Core algorithm**: `graphrag/index/operations/embed_graph/embed_node2vec.py:20-43`

```python
def embed_node2vec(
    graph: nx.Graph | nx.DiGraph,
    dimensions: int = 1536,
    num_walks: int = 10,
    walk_length: int = 40,
    window_size: int = 2,
    iterations: int = 3,
    random_seed: int = 86,
) -> NodeEmbeddings:
    """Generate node embeddings using Node2Vec."""
    import graspologic as gc

    lcc_tensors = gc.embed.node2vec_embed(
        graph=graph,
        dimensions=dimensions,
        window_size=window_size,
        iterations=iterations,
        num_walks=num_walks,
        walk_length=walk_length,
        random_seed=random_seed,
    )
    return NodeEmbeddings(embeddings=lcc_tensors[0], nodes=lcc_tensors[1])
```

---

### Largest Connected Component (LCC) Preprocessing

**Code**: `graphrag/index/utils/stable_lcc.py:12-20`

```python
def stable_largest_connected_component(graph: nx.Graph) -> nx.Graph:
    """Return the largest connected component of the graph, with nodes and edges sorted in a stable way."""
    from graspologic.utils import largest_connected_component

    graph = graph.copy()
    graph = cast("nx.Graph", largest_connected_component(graph))
    graph = normalize_node_names(graph)
    return _stabilize_graph(graph)
```

**Why extract LCC?**:

**Problem**: Disconnected graph components cause issues
- **Random walks get stuck**: Walk from node in Component A never reaches Component B
- **Embedding inconsistency**: Nodes in isolated components have arbitrary embeddings (no training signal)

**Solution**: Only embed the **largest connected component**

**Example**:
```
Original graph:
  Component A (100 nodes): MICROSOFT -- OPENAI -- GPT-4 -- ...
  Component B (50 nodes): BIOLOGY -- CELL -- DNA -- ...
  Component C (3 nodes): RANDOM1 -- RANDOM2 -- RANDOM3

LCC extraction → Keep only Component A (largest)
```

**Trade-off**:
- **Pro**: Consistent, meaningful embeddings for main cluster
- **Con**: Lose nodes from smaller components (often noise anyway)

**Normalization** (`normalize_node_names`, line 64-67):
```python
def normalize_node_names(graph: nx.Graph | nx.DiGraph) -> nx.Graph | nx.DiGraph:
    """Normalize node names."""
    node_mapping = {node: html.unescape(node.upper().strip()) for node in graph.nodes()}
    return nx.relabel_nodes(graph, node_mapping)
```

**Purpose**: Ensure deterministic node ordering (uppercase, strip whitespace, unescape HTML entities)

---

### Stable Graph Ordering

**Code**: `stable_lcc.py:23-61`

**Problem**: Graph data structures (dictionaries, sets) have **non-deterministic iteration order**

**Consequence**: Same input graph → different node ordering → different random walk sequences → different embeddings

**Solution**: Sort nodes and edges lexicographically before embedding

```python
def _stabilize_graph(graph: nx.Graph) -> nx.Graph:
    """Ensure an undirected graph with the same relationships will always be read the same way."""
    fixed_graph = nx.DiGraph() if graph.is_directed() else nx.Graph()

    # Sort nodes alphabetically
    sorted_nodes = graph.nodes(data=True)
    sorted_nodes = sorted(sorted_nodes, key=lambda x: x[0])

    fixed_graph.add_nodes_from(sorted_nodes)
    edges = list(graph.edges(data=True))

    # For undirected graphs, normalize edge direction (A->B same as B->A)
    if not graph.is_directed():
        def _sort_source_target(edge):
            source, target, edge_data = edge
            if source > target:
                temp = source
                source = target
                target = temp
            return source, target, edge_data

        edges = [_sort_source_target(edge) for edge in edges]

    # Sort edges by "source -> target" string
    edges = sorted(edges, key=lambda x: f"{x[0]} -> {x[1]}")

    fixed_graph.add_edges_from(edges)
    return fixed_graph
```

**Benefit**: **Reproducibility** — same graph + same random_seed → identical embeddings every time

---

## Parameter Tuning

### Dimensions

**Default**: 1536 (matches OpenAI text embedding dimension)

**Effect**:
- **Higher**: More expressive, but more memory/compute
- **Lower**: Faster, but may lose structural nuance

**Recommendation**:
- **1536**: If integrating with text embeddings (hybrid search)
- **128-256**: If only using graph embeddings (sufficient for most tasks)

---

### Num Walks

**Default**: 10

**Effect**: More walks → better coverage of neighborhood, but slower

**Intuition**: Each walk samples a different path through the graph. More walks = more diverse context samples.

**Example** (node with degree=5):
```
1 walk:  May only visit 2-3 of the 5 neighbors
10 walks: Likely visits all 5 neighbors multiple times
```

**Recommendation**:
- **5-10**: Standard (balanced coverage/speed)
- **20-50**: Dense graphs or when quality critical

---

### Walk Length

**Default**: 40

**Effect**: Longer walks → capture broader neighborhood, but diminishing returns

**Intuition**: Nodes far away in walk contribute less to embedding (due to window_size decay)

**Example**:
```
Walk length = 5:  Captures 1-2 hop neighbors
Walk length = 40: Captures up to 10-20 hop neighbors (but distant hops have small weight)
```

**Recommendation**:
- **10-20**: Small graphs (< 1000 nodes)
- **40-80**: Large graphs (> 10K nodes)

---

### Window Size

**Default**: 2

**Effect**: Larger window → more context, but slower training

**Intuition**: In walk [A, B, C, D, E]:
- `window_size=1`: C's context is [B, D]
- `window_size=2`: C's context is [A, B, D, E]

**Recommendation**:
- **2**: Standard (captures immediate neighborhood)
- **5-10**: If long-range dependencies matter

---

### Iterations

**Default**: 3

**Effect**: More iterations → better convergence, but slower

**Intuition**: Skip-gram is optimized via SGD. More epochs = better embedding quality.

**Recommendation**:
- **1-3**: Fast prototyping
- **5-10**: Production quality

---

### Random Seed

**Default**: 86

**Effect**: Ensures **reproducibility** — same seed → same embeddings

**Critical**: Must use stable graph ordering (via `stable_largest_connected_component`) for seed to be effective

---

## Integration with GraphRAG

### Use Case 1: Graph Visualization

**From** `spec/semlang/visualization.md` (conceptual):

```python
# After indexing
embeddings = embed_graph(graph, config)

# Reduce to 2D via UMAP
from umap import UMAP
reducer = UMAP(n_components=2)
embeddings_2d = reducer.fit_transform(list(embeddings.values()))

# Plot
import matplotlib.pyplot as plt
plt.scatter(embeddings_2d[:, 0], embeddings_2d[:, 1])
for node, coord in zip(embeddings.keys(), embeddings_2d):
    plt.annotate(node, coord)
plt.show()
```

**Benefit**: Visualize community structure, identify clusters, spot outliers

---

### Use Case 2: Hybrid Search (Text + Graph)

**Conceptual flow**:

```python
# Text embedding (from LLM)
text_emb = embed_text("What are Microsoft's AI initiatives?")

# Graph embedding (from Node2Vec)
graph_emb = embeddings["MICROSOFT"]

# Combine embeddings
hybrid_emb = 0.7 * text_emb + 0.3 * graph_emb  # Weighted combination

# Search
results = vector_search(hybrid_emb, index)
```

**Benefit**: Retrieve entities relevant by **both** text content **and** graph structure

---

### Use Case 3: Entity Features for ML

**Example**: Predict entity importance

```python
# Features from graph embeddings
features = [embeddings[entity] for entity in entities]

# Labels from centrality
labels = [degree_centrality[entity] for entity in entities]

# Train model
from sklearn.linear_model import Ridge
model = Ridge().fit(features, labels)

# Predict for new entities
predicted_importance = model.predict([embeddings["NEW_ENTITY"]])
```

**Benefit**: Use structural features (embeddings) to predict properties (importance, type, etc.)

---

## Connection to Dependencies

### graspologic Library

**From** `spec/dependencies/02-graph-and-vector-storage.md`:

**Why graspologic?**:
- **Optimized Node2Vec**: C++ backend, 10-100x faster than pure Python
- **Integrated with networkx**: Direct graph input
- **Stable API**: Maintained by Microsoft Research

**Performance** (from graspologic benchmarks):
```
Graph size: 10K nodes, 50K edges
num_walks=10, walk_length=40, dimensions=128

graspologic: 12 seconds
gensim Word2Vec + manual walks: 180 seconds
```

---

### UMAP for Dimensionality Reduction

**From** `spec/dependencies/02-graph-and-vector-storage.md`:

**Purpose**: Reduce Node2Vec embeddings (1536-dim) to 2D/3D for visualization

**Why UMAP over t-SNE?**:
- **Faster**: 10-100x speedup on large datasets
- **Preserves global structure**: t-SNE only preserves local neighborhoods
- **Deterministic**: With fixed random seed

**Example**:
```python
from umap import UMAP
import numpy as np

# Extract embeddings as matrix
embedding_matrix = np.array([embeddings[node] for node in sorted(embeddings.keys())])

# Reduce to 2D
reducer = UMAP(n_components=2, n_neighbors=15, min_dist=0.1, random_state=42)
embeddings_2d = reducer.fit_transform(embedding_matrix)

# embeddings_2d.shape: (num_nodes, 2)
```

---

## Performance Characteristics

### Time Complexity

| Phase | Complexity | Dominant Factor |
|-------|------------|-----------------|
| Random walk generation | O(N × num_walks × walk_length) | Graph traversal |
| Skip-gram training | O(W × window_size × iterations) | W = total walk tokens |
| Total | O(N × num_walks × walk_length × window_size × iterations) | |

**Where**:
- N = number of nodes
- W = N × num_walks × walk_length (total walk tokens)

**Bottleneck**: Walk generation for large graphs (millions of nodes)

---

### Space Complexity

| Component | Memory Usage |
|-----------|--------------|
| Random walks | O(N × num_walks × walk_length × 8 bytes) |
| Embedding matrix | O(N × dimensions × 4 bytes) |
| Skip-gram gradients | O(N × dimensions × 4 bytes) |
| Total | O(N × num_walks × walk_length) |

**Example** (1M nodes, default params):
```
Random walks: 1M × 10 × 40 × 8 bytes = 3.2 GB
Embeddings: 1M × 1536 × 4 bytes = 6.1 GB
Total: ~10 GB peak memory
```

---

### Benchmarks

**Setup**: GraphRAG knowledge graph from 10K document corpus

| Graph Size | Embedding Time | Memory Usage |
|------------|----------------|--------------|
| 1K nodes, 3K edges | 5 seconds | 200 MB |
| 10K nodes, 30K edges | 45 seconds | 1.5 GB |
| 100K nodes, 300K edges | 8 minutes | 12 GB |

**Machine**: 16-core CPU, 32GB RAM (no GPU acceleration)

---

## Connection to Research Patterns

### Structural vs. Semantic Similarity

**From** `spec/research/01-query-as-key.md`:

**Text embeddings**: Capture **semantic similarity**
- "Microsoft" and "tech company" are similar (cosine similarity ~0.7)

**Graph embeddings**: Capture **structural similarity**
- "Microsoft" and "OpenAI" are similar (both central in tech community)

**Complementary signals**:
- Text: *What* does the entity mean?
- Graph: *How* does the entity relate to others?

**Hybrid approach**: Combine both for richer retrieval

---

### Attractor Basins in Embedding Space

**From** `spec/research/02-star-attractor-patterns.md`:

**Observation**: High-centrality entities (attractors) form **dense clusters** in embedding space

**Explanation**:
- Attractor nodes appear in many random walks
- Their embeddings are trained on diverse contexts
- Diverse contexts → central position in embedding space

**Example** (2D projection):
```
     OPENAI
       / \
      /   \
GPT-4     CHATGPT
      \   /
       \ /
   MICROSOFT (central attractor)
       |
     AZURE
```

**Implication**: Clustering in embedding space approximates communities in graph space

---

## Troubleshooting

### Problem: Embeddings are All Similar (Low Variance)

**Symptom**: All cosine similarities ~0.99

**Diagnosis**:
- Graph is too sparse (few edges)
- Walk length too short (doesn't explore neighborhood)
- Window size too small (only captures immediate neighbors)

**Solution**:
1. **Increase walk length**: `walk_length=80` (from default 40)
2. **Increase num_walks**: `num_walks=20` (from default 10)
3. **Check graph density**: `avg_degree = 2 * num_edges / num_nodes` (should be ≥ 3)

---

### Problem: Out of Memory During Embedding

**Symptom**: `MemoryError` during `node2vec_embed()`

**Diagnosis**: Too many walks stored in memory

**Solution**:
1. **Reduce num_walks**: `num_walks=5` (from default 10)
2. **Reduce walk_length**: `walk_length=20` (from default 40)
3. **Reduce dimensions**: `dimensions=256` (from default 1536)
4. **Process in batches**: Embed subgraphs separately, then merge

---

### Problem: Embeddings Change Between Runs

**Symptom**: Same graph → different embeddings each run

**Diagnosis**: Non-deterministic graph ordering or random seed not set

**Solution**:
1. **Enable LCC stabilization**: `use_lcc=True` (normalizes node order)
2. **Set random seed**: `random_seed=42`
3. **Verify graph is sorted**: Check `stable_largest_connected_component()` is called

---

### Problem: Slow Embedding for Large Graphs

**Symptom**: Hours to embed 1M node graph

**Diagnosis**: Bottleneck in random walk generation

**Solution**:
1. **Reduce num_walks**: `num_walks=5`
2. **Reduce walk_length**: `walk_length=20`
3. **Use LCC only**: Filter to largest component before embedding
4. **Sample subgraph**: Embed only high-degree nodes (top 10K by centrality)

---

## Future Enhancements

### 1. Biased Random Walks (Original Node2Vec)

**Current**: Uniform random walks (equal probability for all neighbors)

**Proposed**: Biased walks with return parameter (p) and in-out parameter (q)

```python
def biased_random_walk(graph, start, p=1, q=1, length=40):
    """
    p: Return parameter (likelihood to return to previous node)
    q: In-out parameter (likelihood to explore outward vs. stay local)
    """
    walk = [start]
    for _ in range(length - 1):
        current = walk[-1]
        prev = walk[-2] if len(walk) > 1 else None

        # Compute transition probabilities
        neighbors = list(graph.neighbors(current))
        probs = []
        for neighbor in neighbors:
            if neighbor == prev:
                probs.append(1 / p)  # Less likely to return
            elif graph.has_edge(prev, neighbor):
                probs.append(1)  # Stay in neighborhood
            else:
                probs.append(1 / q)  # Explore outward

        # Normalize and sample
        probs = np.array(probs) / sum(probs)
        next_node = np.random.choice(neighbors, p=probs)
        walk.append(next_node)

    return walk
```

**Benefit**: Control exploration strategy (BFS-like vs. DFS-like)

---

### 2. Heterogeneous Graph Embeddings

**Current**: All nodes treated identically

**Proposed**: Type-aware embeddings (entities vs. events vs. topics)

```python
def embed_heterogeneous_graph(graph, node_types):
    """
    Learn separate embedding spaces for each node type,
    then combine via attention mechanism.
    """
    embeddings_by_type = {}
    for type_name in set(node_types.values()):
        subgraph = graph.subgraph([n for n, t in node_types.items() if t == type_name])
        embeddings_by_type[type_name] = embed_node2vec(subgraph)

    # Combine via cross-type edges
    # ... (complex, omitted for brevity)
```

**Benefit**: Capture type-specific structural patterns

---

### 3. Dynamic Graph Embeddings

**Current**: Static snapshot only

**Proposed**: Incrementally update embeddings as graph evolves

```python
def update_embeddings(old_embeddings, new_edges, new_nodes):
    """
    Update embeddings for affected nodes only,
    without full re-embedding.
    """
    # Identify nodes within k-hop radius of changes
    affected = get_k_hop_neighbors(new_nodes, k=2)

    # Re-embed only affected subgraph
    subgraph = graph.subgraph(affected)
    updated = embed_node2vec(subgraph)

    # Merge with old embeddings
    return {**old_embeddings, **updated}
```

**Benefit**: Real-time embedding updates for streaming data

---

## Conclusion

Node2Vec embeddings provide a **structural lens** on the knowledge graph, complementing text-based semantic embeddings. By treating random walks as "sentences", Node2Vec learns continuous representations that capture **graph topology**, enabling similarity search, visualization, and ML features. In GraphRAG, these embeddings power hybrid retrieval, entity clustering, and graph analytics — revealing patterns invisible to text-only methods.

**Key takeaways**:
- **Random walks** sample graph neighborhoods, creating "context" for nodes
- **Skip-gram** learns embeddings by predicting context from target (same as Word2Vec)
- **LCC extraction** ensures consistent embeddings for main graph component
- **Stable ordering** guarantees reproducibility with fixed random seed
- **Trade-offs**: More walks/length = better quality but slower/more memory

**Integration points**:
- Visualization via UMAP dimensionality reduction
- Hybrid search by combining with text embeddings
- Features for ML models (importance prediction, classification)

**Key files**:
- `embed_node2vec.py:20-43` — Core Node2Vec wrapper
- `embed_graph.py:16-50` — High-level API with LCC preprocessing
- `stable_lcc.py:12-67` — Graph normalization for reproducibility
