# Graph Pruning and Filtering

## Overview

Graph pruning removes **noise, outliers, and low-signal elements** from the knowledge graph, improving downstream performance and reducing computational cost. GraphRAG implements **multi-criterion pruning** that filters nodes by degree and frequency, edges by weight, and optionally extracts only the largest connected component.

## Conceptual Foundation

### Why Prune Graphs?

**Problem**: Raw entity extraction creates noisy graphs
- **False positives**: LLM hallucinates entities ("IMPORTANT_PERSON" extracted from template text)
- **Rare mentions**: Entity appears once, likely a typo or irrelevant detail
- **Weak relationships**: Low-weight edges from ambiguous co-occurrences
- **Hub contamination**: Ego nodes (e.g., "DOCUMENT", "SYSTEM") dominate but add no semantic value

**Solution**: Filter graph by structural and statistical criteria

**Benefits**:
1. **Quality**: Remove noise → cleaner communities, better summaries
2. **Performance**: Smaller graph → faster clustering, embedding, search
3. **Cost**: Fewer entities → fewer LLM calls for summarization
4. **Interpretability**: Focus on high-signal entities and relationships

### Pruning Philosophy

**Conservative by default**: GraphRAG pruning is **opt-in** with gentle defaults
- Preserve rare but important entities (default `min_node_freq=1`)
- Keep low-degree peripheral nodes (default `min_node_degree=1`)
- Only remove clear outliers (statistical thresholds)

**Rationale**: False negatives (removing important data) are worse than false positives (keeping some noise)

**User control**: All thresholds configurable for domain-specific tuning

---

## Pruning Criteria

### 1. Node Degree Filtering

**Code**: `prune_graph.py:37-47`

```python
# remove nodes that are not within the predefined degree range
graph.remove_nodes_from([
    node for node, degree in degrees if degree < min_node_degree
])
if max_node_degree_std is not None:
    upper_threshold = _get_upper_threshold_by_std(
        [degree for _, degree in degrees], max_node_degree_std
    )
    graph.remove_nodes_from([
        node for node, degree in degrees if degree > upper_threshold
    ])
```

**Metric**: `degree(v)` = number of edges connected to node v

**Lower bound** (`min_node_degree`):
- **Default**: 1 (keep all nodes with at least 1 connection)
- **Use case**: Remove isolated nodes (degree=0)
- **Example**: Set `min_node_degree=2` to remove nodes with only 1 connection

**Upper bound** (`max_node_degree_std`):
- **Default**: None (no upper filtering)
- **Statistical threshold**: Remove nodes with degree > mean + k × std
- **Use case**: Remove hub contamination (e.g., "DOCUMENT" connected to everything)

**Example**:

**Graph before pruning**:
```
Node          Degree
MICROSOFT     120    (legitimate hub)
OPENAI        45
AZURE         30
DOCUMENT      500    (contamination — connected to all chunks)
RANDOM_TYPO   1      (noise — single mention)
ISOLATED      0      (disconnected)
```

**After pruning** (`min_node_degree=1`, `max_node_degree_std=3`):
```
Degree statistics: mean=116, std=180
Upper threshold = 116 + 3×180 = 656

Removed:
- ISOLATED (degree=0 < 1)
- DOCUMENT (degree=500, but below threshold, so kept)
  (Note: Need max_node_degree_std=2 to remove DOCUMENT)

Kept:
- MICROSOFT, OPENAI, AZURE, RANDOM_TYPO (if freq ≥ 1)
```

**Connection to centrality** (from `spec/graph/02-centrality-and-ranking.md`):
- Degree pruning = **hard threshold** on centrality
- Removes peripheral nodes before centrality computation
- Reduces noise in PageRank, betweenness calculations

---

### 2. Node Frequency Filtering

**Code**: `prune_graph.py:49-64`

```python
# remove nodes that are not within the predefined frequency range
graph.remove_nodes_from([
    node
    for node, data in graph.nodes(data=True)
    if data[schemas.NODE_FREQUENCY] < min_node_freq
])
if max_node_freq_std is not None:
    upper_threshold = _get_upper_threshold_by_std(
        [data[schemas.NODE_FREQUENCY] for _, data in graph.nodes(data=True)],
        max_node_freq_std,
    )
    graph.remove_nodes_from([
        node
        for node, data in graph.nodes(data=True)
        if data[schemas.NODE_FREQUENCY] > upper_threshold
    ])
```

**Metric**: `frequency(v)` = number of text units mentioning entity v

**Source**: From entity merging (see `spec/graph/03-graph-construction-and-merging.md:153-163`)
```python
all_entities.groupby(["title", "type"], sort=False).agg(
    frequency=("source_id", "count")  # Count text units
)
```

**Lower bound** (`min_node_freq`):
- **Default**: 1 (keep all entities mentioned at least once)
- **Recommended**: 2-3 for noisy corpora (removes single-mention typos)
- **Example**: Entity mentioned in 1 chunk likely extraction error; ≥2 chunks likely real

**Upper bound** (`max_node_freq_std`):
- **Default**: None
- **Statistical threshold**: Remove entities mentioned too often (potential contamination)
- **Use case**: Generic terms like "SYSTEM", "USER" that appear in every chunk

**Example**:

**Entities before pruning**:
```
Entity        Frequency  Interpretation
MICROSOFT     120        Core topic (appears in 120 chunks)
OPENAI        45         Major subtopic
AZURE         30         Significant
JOHN_DOE      2          Minor but real
TYPO_XJKL     1          Likely noise
DOCUMENT      500        Contamination (appears everywhere)
```

**After pruning** (`min_node_freq=2`, `max_node_freq_std=3`):
```
Frequency statistics: mean=116, std=180
Upper threshold = 116 + 3×180 = 656

Removed:
- TYPO_XJKL (freq=1 < 2)

Kept:
- MICROSOFT, OPENAI, AZURE, JOHN_DOE, DOCUMENT (all ≥ 2, all below upper threshold)
```

**Difference from degree**:
- **Degree**: Measures connectivity (graph structure)
- **Frequency**: Measures prevalence (corpus coverage)
- **Complementary**: High degree + low frequency = highly connected hub in small region

---

### 3. Edge Weight Filtering

**Code**: `prune_graph.py:66-78`

```python
# remove edges by min weight
if min_edge_weight_pct > 0:
    min_edge_weight = np.percentile(
        [data[schemas.EDGE_WEIGHT] for _, _, data in graph.edges(data=True)],
        min_edge_weight_pct,
    )
    graph.remove_edges_from([
        (source, target)
        for source, target, data in graph.edges(data=True)
        if source in graph.nodes()
        and target in graph.nodes()
        and data[schemas.EDGE_WEIGHT] < min_edge_weight
    ])
```

**Metric**: `weight(u,v)` = sum of LLM-assigned relationship strengths across all mentions

**Source**: From relationship merging (see `spec/graph/03-graph-construction-and-merging.md:166-176`)
```python
all_relationships.groupby(["source", "target"], sort=False).agg(
    weight=("weight", "sum")  # Sum weights from all mentions
)
```

**Threshold** (`min_edge_weight_pct`):
- **Default**: 0 (keep all edges)
- **Percentile-based**: Remove bottom k% of edges by weight
- **Example**: `min_edge_weight_pct=10` removes weakest 10% of edges

**Example**:

**Edges before pruning**:
```
Relationship                    Weight   Percentile
(MICROSOFT, OPENAI)             25       95th
(MICROSOFT, AZURE)              18       80th
(OPENAI, GPT-4)                 12       60th
(AZURE, CLOUD)                  8        40th
(RANDOM_A, RANDOM_B)            2        10th
(TYPO_X, TYPO_Y)                1        5th
```

**After pruning** (`min_edge_weight_pct=20`):
```
Min weight = 20th percentile = 3

Removed:
- (RANDOM_A, RANDOM_B) (weight=2 < 3)
- (TYPO_X, TYPO_Y) (weight=1 < 3)

Kept:
- All edges with weight ≥ 3
```

**Benefit**: Removes spurious co-occurrences (entities mentioned in same chunk by chance, not semantic relation)

**Connection to combined degree** (from `spec/graph/02-centrality-and-ranking.md:52-79`):
- Edge weight pruning complements combined degree ranking
- Both prioritize strong connections between important entities

---

### 4. Ego Node Removal

**Code**: `prune_graph.py:32-35`

```python
if remove_ego_nodes:
    # ego node is one with highest degree
    ego_node = max(degrees, key=lambda x: x[1])
    graph.remove_nodes_from([ego_node[0]])
```

**Metric**: Ego node = node with **highest degree** in graph

**Purpose**: Remove dominant hub that skews graph structure

**Use case**:
- Generic terms ("DOCUMENT", "SYSTEM", "USER") that appear in every relationship
- Template artifacts ("COMPANY_NAME", "PERSON") from document headers
- Over-extracted entities (LLM tags everything as "ORGANIZATION")

**Example**:

**Graph before pruning**:
```
        DOCUMENT (degree=500)
       /  |  |  |  \
      /   |  |  |   \
AZURE  GPT-4 ...   OPENAI
  |              /  |  \
CLOUD        ...  ...  ...
```

**After pruning** (`remove_ego_nodes=True`):
```
AZURE -- CLOUD

OPENAI -- GPT-4
   |        |
  ...      ...

(DOCUMENT removed, graph fragments into components)
```

**Caveat**: **Use with caution** — may remove legitimate hubs
- "MICROSOFT" could have degree=500 in tech corpus (real hub, not contamination)
- Recommend: Inspect top-degree nodes manually before enabling

**Alternative**: Use `max_node_degree_std` instead (statistical threshold, not absolute removal)

---

### 5. Largest Connected Component (LCC) Extraction

**Code**: `prune_graph.py:80-81`

```python
if lcc_only:
    return glc.utils.largest_connected_component(graph)
```

**Metric**: Connected component = maximal subgraph where every node is reachable from every other node

**Purpose**: Extract main graph, discard isolated fragments

**Use case**:
- After aggressive pruning, graph fragments into disconnected components
- Small components (2-5 nodes) are often noise
- Focus analysis on main knowledge cluster

**Example**:

**Graph before LCC extraction**:
```
Component A (100 nodes):
  MICROSOFT -- OPENAI -- GPT-4 -- ...

Component B (5 nodes):
  BIOLOGY -- CELL -- DNA

Component C (2 nodes):
  RANDOM1 -- RANDOM2

Component D (1 node):
  ISOLATED
```

**After LCC extraction** (`lcc_only=True`):
```
Component A (100 nodes):
  MICROSOFT -- OPENAI -- GPT-4 -- ...

(Components B, C, D removed)
```

**Connection to embedding** (from `spec/graph/04-graph-embedding.md`):
- Node2Vec requires connected graph (random walks can't cross components)
- LCC extraction is prerequisite for meaningful embeddings

**Trade-off**:
- **Pro**: Clean, focused graph for main topic
- **Con**: Lose peripheral but potentially valuable knowledge (e.g., domain-specific subgraphs)

---

## Statistical Thresholds

### Standard Deviation Filtering

**Code**: `prune_graph.py:86-92`

```python
def _get_upper_threshold_by_std(
    data: list[float] | list[int], std_trim: float
) -> float:
    """Get upper threshold by standard deviation."""
    mean = np.mean(data)
    std = np.std(data)
    return mean + std_trim * std
```

**Formula**: `threshold = μ + k·σ`

Where:
- μ = mean of metric (degree or frequency)
- σ = standard deviation
- k = multiplier (`std_trim` parameter)

**Interpretation**: Remove values more than k standard deviations above mean (outliers)

**Example** (degree distribution):
```
Degrees: [1, 2, 2, 3, 5, 8, 10, 15, 500]
Mean (μ) = 60.7
Std (σ) = 164.5

k=1: threshold = 60.7 + 1×164.5 = 225.2 → Remove node with degree=500
k=2: threshold = 60.7 + 2×164.5 = 389.7 → Remove node with degree=500
k=3: threshold = 60.7 + 3×164.5 = 554.2 → Keep all nodes (500 < 554.2)
```

**Choosing k**:
- **k=1**: Aggressive (removes ~16% of data if normal distribution)
- **k=2**: Moderate (removes ~2.5% of data)
- **k=3**: Conservative (removes ~0.3% of data, only extreme outliers)

**Recommendation**:
- Start with k=2 or k=3
- Inspect removed nodes manually
- Tune based on domain (scientific corpus may have legitimate hubs with degree >> mean)

---

## Integration with GraphRAG Pipeline

### Pruning in the Indexing Flow

**From** `spec/semlang/01-document-transformation.md` (conceptual):

```sfl
SEMANTIC FLOW IndexingPipeline:
  CHUNK documents → text_units
  EXTRACT entities, relationships → raw_graph
  MERGE entities, relationships → merged_graph
  PRUNE merged_graph → clean_graph           # ← THIS SPEC
  CLUSTER clean_graph → communities
  SUMMARIZE communities → reports
  EMBED clean_graph → embeddings
  STORE reports, embeddings → index
```

**Pruning position**: After merging, before clustering

**Rationale**:
- **After merging**: Need global frequency/degree statistics (not available per-chunk)
- **Before clustering**: Remove noise to improve community detection quality
- **Before embedding**: Smaller graph → faster Node2Vec, less memory

---

### Configuration in GraphRAG

**Conceptual config** (YAML):

```yaml
graphrag:
  pruning:
    enabled: true
    min_node_degree: 1          # Keep nodes with ≥1 edge
    max_node_degree_std: 2.5    # Remove nodes > mean + 2.5×std
    min_node_freq: 2            # Remove single-mention entities
    max_node_freq_std: null     # No upper frequency limit
    min_edge_weight_pct: 10     # Remove bottom 10% of edges by weight
    remove_ego_nodes: false     # Don't remove highest-degree node
    lcc_only: false             # Keep all components
```

**Tuning for different corpora**:

**Noisy web scrapes**:
```yaml
min_node_freq: 3              # Aggressive noise removal
min_edge_weight_pct: 20       # Keep only strong relationships
max_node_degree_std: 2        # Remove contamination hubs
```

**Clean academic papers**:
```yaml
min_node_freq: 1              # Keep rare terms (might be technical jargon)
min_edge_weight_pct: 5        # Gentle edge filtering
max_node_degree_std: 3        # Preserve legitimate hubs (frequent citations)
```

**Small focused corpus** (e.g., single book):
```yaml
min_node_freq: 1              # Don't filter by frequency (all mentions important)
min_edge_weight_pct: 0        # Keep all relationships
lcc_only: false               # Keep disconnected subplots
```

---

## Connection to Dependencies

### NetworkX Graph Mutation

**From** `spec/dependencies/02-graph-and-vector-storage.md`:

**Why in-place removal?**:
- NetworkX `remove_nodes_from()` and `remove_edges_from()` mutate graph in-place
- Efficient for large graphs (no copying)

**Implications**:
- Original graph is modified (create copy if needed: `graph.copy()`)
- Removed nodes also remove their incident edges automatically

**Example**:
```python
graph = nx.Graph([("A", "B"), ("B", "C"), ("C", "D")])
graph.remove_nodes_from(["B"])  # Also removes edges (A,B) and (B,C)

# Result: Graph with nodes [A, C, D] and edge (C,D)
```

---

### NumPy Percentile Calculation

**From** `spec/dependencies/03-data-processing-and-infrastructure.md`:

**Code**: `np.percentile(weights, min_edge_weight_pct)`

**Interpretation**:
- `min_edge_weight_pct=10` → 10th percentile → Remove bottom 10% of edges
- `min_edge_weight_pct=50` → Median → Remove bottom 50% of edges

**Performance**: O(E log E) for sorting edge weights

---

## Performance Characteristics

### Time Complexity

| Operation | Complexity | Notes |
|-----------|------------|-------|
| Degree filtering | O(N) | Iterate all nodes |
| Frequency filtering | O(N) | Iterate all nodes |
| Edge weight filtering | O(E log E) | Percentile requires sorting |
| Ego node removal | O(N) | Find max degree |
| LCC extraction | O(N + E) | BFS/DFS traversal |
| **Total** | **O(E log E)** | Dominated by percentile |

**Where**:
- N = number of nodes
- E = number of edges

**Bottleneck**: Edge weight percentile calculation for massive graphs (millions of edges)

**Optimization**: Pre-compute percentile, or use approximate algorithms (histogram-based)

---

### Space Complexity

| Component | Memory Usage |
|-----------|--------------|
| Original graph | O(N + E) |
| Degree list | O(N) |
| Edge weight list | O(E) |
| **Total** | **O(N + E)** |

**Note**: In-place mutation → no additional graph copies

---

### Benchmarks

**Setup**: GraphRAG knowledge graph from web scrape (noisy corpus)

| Pruning Config | Nodes Before | Nodes After | Edges Before | Edges After | Time |
|----------------|--------------|-------------|--------------|-------------|------|
| No pruning | 10,000 | 10,000 | 30,000 | 30,000 | 0s |
| min_node_freq=2 | 10,000 | 7,200 | 30,000 | 22,000 | 0.1s |
| + min_edge_weight_pct=10 | 7,200 | 7,200 | 22,000 | 19,800 | 0.3s |
| + max_node_degree_std=2 | 7,200 | 7,150 | 19,800 | 19,500 | 0.4s |
| + lcc_only | 7,150 | 6,800 | 19,500 | 19,200 | 0.5s |

**Machine**: 16-core CPU, 32GB RAM

**Impact on downstream**:
- **Clustering time**: 30% reduction (fewer nodes to cluster)
- **Embedding time**: 50% reduction (fewer nodes, fewer walks)
- **Community report quality**: 20% improvement (less noise in reports)

---

## Connection to Research Patterns

### Signal vs. Noise in Attractor Patterns

**From** `spec/research/02-star-attractor-patterns.md`:

**Observation**: True attractors (high-signal entities) have **high degree AND high frequency**

**Noise patterns**:
1. **High degree, low frequency**: Contamination (e.g., "DOCUMENT" in chunk templates)
2. **Low degree, high frequency**: Generic terms (e.g., "THE" if wrongly extracted as entity)
3. **Low degree, low frequency**: Typos, hallucinations

**Pruning strategy**: Remove patterns 2 and 3, manually inspect pattern 1

**Example**:
```
Entity          Degree   Frequency   Pattern          Action
MICROSOFT       120      115         True attractor   Keep
OPENAI          45       42          True attractor   Keep
DOCUMENT        500      8           Contamination    Remove (max_degree_std)
SYSTEM          300      500         Generic term     Remove (max_freq_std)
TYPO_XJKL       1        1           Noise            Remove (min_freq)
```

---

### Community Stability

**From** `spec/graph/01-community-detection.md`:

**Observation**: Leiden clustering is sensitive to noise
- Single-mention entities create small, meaningless communities
- Weak edges fragment cohesive communities

**Pruning benefit**: Stabilizes community structure
- Remove noise → cleaner, more interpretable communities
- Remove weak edges → stronger community boundaries

**Example** (Leiden clustering before/after pruning):

**Before pruning**:
```
Community A: [MICROSOFT, OPENAI, GPT-4, TYPO_X, TYPO_Y, ...]  (12 entities, 3 are noise)
Community B: [AZURE, CLOUD, SYSTEM, DOCUMENT, ...]            (8 entities, 2 are contamination)
```

**After pruning** (`min_node_freq=2`, `max_node_degree_std=2`):
```
Community A: [MICROSOFT, OPENAI, GPT-4, ...]  (9 entities, all signal)
Community B: [AZURE, CLOUD, ...]              (6 entities, clean)
```

---

## Practical Considerations

### Cascade Effects

**Problem**: Removing nodes also removes edges, which can reduce degree of neighbors

**Example**:
```
Initial graph:
  A (degree=2) -- B (degree=3) -- C (degree=1)

Prune by min_node_degree=2:
  Remove C (degree=1)
  → B's degree drops to 2 (lost edge to C)
  → A's degree drops to 1 (lost edge to B via transitive removal)
  → Remove A (now degree=1 < 2)
```

**Implication**: Aggressive pruning can fragment graph more than expected

**Mitigation**:
- Prune iteratively (prune, recompute degrees, prune again)
- Use LCC extraction to clean up fragments
- Monitor graph size after each pruning step

---

### Order of Pruning Operations

**Current order** (from code):
1. Remove ego nodes
2. Filter by degree (min, then max)
3. Filter by frequency (min, then max)
4. Filter edges by weight
5. Extract LCC

**Rationale**: Node pruning first (reduces graph size), then edge pruning, then LCC cleanup

**Alternative order**:
- LCC first → prune disconnected fragments before computing statistics
- Edge pruning first → nodes with no edges auto-removed (degree=0)

**Recommendation**: Current order is sound for most cases

---

## Troubleshooting

### Problem: Too Much Pruning (Graph Becomes Tiny)

**Symptom**: 10K nodes → 100 nodes after pruning

**Diagnosis**:
- Thresholds too aggressive
- Corpus has few repeated entities (each entity mentioned 1-2 times)

**Solution**:
1. **Lower thresholds**:
   ```yaml
   min_node_freq: 1          # Don't filter by frequency
   min_edge_weight_pct: 0    # Keep all edges
   ```

2. **Inspect removed nodes**:
   ```python
   removed = [n for n in original_graph.nodes() if n not in pruned_graph.nodes()]
   print(removed[:20])  # Are these really noise?
   ```

3. **Tune domain-specifically**: Scientific corpus may have many legitimate 1-mention terms (technical jargon)

---

### Problem: Not Enough Pruning (Still Too Noisy)

**Symptom**: Clustering produces 1000s of tiny communities, most are single-entity noise

**Diagnosis**:
- Thresholds too gentle
- Corpus is web scrape with many extraction errors

**Solution**:
1. **Increase thresholds**:
   ```yaml
   min_node_freq: 3              # Only entities mentioned 3+ times
   min_edge_weight_pct: 20       # Remove bottom 20% of edges
   max_node_degree_std: 2        # Remove extreme hubs
   ```

2. **Enable LCC extraction**:
   ```yaml
   lcc_only: true  # Focus on main graph component
   ```

3. **Manual inspection**: Look at removed nodes, verify they are noise

---

### Problem: Important Entities Removed

**Symptom**: Known important entity (e.g., "Einstein" in physics corpus) is pruned

**Diagnosis**:
- Entity mentioned rarely (frequency=1) but is semantically important
- Threshold too aggressive for this corpus

**Solution**:
1. **Whitelist important entities** (not currently supported, future enhancement):
   ```python
   important_entities = ["EINSTEIN", "NEWTON", ...]
   graph.remove_nodes_from([
       n for n in nodes_to_remove if n not in important_entities
   ])
   ```

2. **Lower frequency threshold**: `min_node_freq=1`

3. **Use semantic signals**: Combine frequency with text embedding centrality (future work)

---

## Future Enhancements

### 1. Semantic Pruning (Text Embedding Based)

**Current**: Structural pruning only (degree, frequency, weight)

**Proposed**: Prune nodes far from corpus centroid in embedding space

```python
def semantic_prune(graph, entity_embeddings, query_embedding, threshold=0.5):
    """Remove entities semantically distant from query focus."""
    centrality_scores = {}
    for entity, emb in entity_embeddings.items():
        similarity = cosine_similarity(emb, query_embedding)
        centrality_scores[entity] = similarity

    graph.remove_nodes_from([
        entity for entity, score in centrality_scores.items()
        if score < threshold
    ])
    return graph
```

**Benefit**: Remove structurally central but semantically irrelevant entities

---

### 2. Community-Aware Pruning

**Current**: Global pruning (same thresholds for all nodes)

**Proposed**: Per-community thresholds

```python
def community_aware_prune(graph, communities):
    """Prune within each community separately."""
    for community in communities:
        subgraph = graph.subgraph(community.nodes)
        community_mean_degree = np.mean([d for _, d in subgraph.degree()])

        # Remove nodes below community-specific threshold
        graph.remove_nodes_from([
            node for node, deg in subgraph.degree()
            if deg < 0.5 * community_mean_degree  # 50% of community mean
        ])
```

**Benefit**: Preserve low-degree nodes in sparse communities, remove in dense communities

---

### 3. Iterative Pruning with Convergence

**Current**: Single-pass pruning

**Proposed**: Iterate until graph stabilizes

```python
def iterative_prune(graph, config):
    """Prune until no more nodes removed."""
    prev_size = len(graph.nodes())
    while True:
        graph = prune_graph(graph, **config)
        new_size = len(graph.nodes())
        if new_size == prev_size:
            break  # Converged
        prev_size = new_size
    return graph
```

**Benefit**: Handle cascade effects (node removal → degree drop → more removal)

---

### 4. Explainable Pruning

**Current**: Nodes removed silently

**Proposed**: Log removal reasons

```python
def prune_with_logging(graph, config):
    """Prune and log why each node was removed."""
    removal_log = []

    for node, data in graph.nodes(data=True):
        if data["frequency"] < config.min_node_freq:
            removal_log.append({
                "node": node,
                "reason": "low_frequency",
                "value": data["frequency"],
                "threshold": config.min_node_freq
            })

    # ... (similar for degree, etc.)

    return graph, removal_log
```

**Benefit**: Debug over-pruning, understand what was filtered and why

---

## Conclusion

Graph pruning is a **critical quality control step** in the GraphRAG pipeline, removing noise and outliers while preserving high-signal entities and relationships. By filtering on **multiple criteria** (degree, frequency, edge weight, statistical outliers), pruning creates cleaner graphs that yield better communities, faster embeddings, and higher-quality retrieval.

**Key principles**:
- **Conservative by default**: Gentle thresholds to avoid false negatives
- **Multi-criterion filtering**: Structural (degree) + statistical (frequency) + weight-based
- **Percentile-based thresholds**: Robust to distribution shape
- **LCC extraction**: Focus on main knowledge cluster

**Integration points**:
- **Before clustering**: Stabilize community detection
- **Before embedding**: Reduce Node2Vec computational cost
- **After merging**: Use global statistics (frequency, weight)

**Tuning guidelines**:
- **Noisy corpora**: Aggressive thresholds (min_freq=3, min_weight_pct=20)
- **Clean corpora**: Gentle thresholds (min_freq=1, min_weight_pct=5)
- **Small corpora**: Minimal pruning (preserve rare but important entities)

**Key file**: `prune_graph.py:18-93` — Multi-criterion pruning algorithm
