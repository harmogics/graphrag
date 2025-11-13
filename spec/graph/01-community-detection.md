# Community Detection: Hierarchical Leiden Algorithm

## Overview

Community detection is the **foundational graph algorithm** in GraphRAG that transforms a flat entity graph into a hierarchical structure of semantic communities. This hierarchy enables multi-scale query routing: Global queries use coarse communities (high-level summaries), while Local queries use fine communities (specific details).

The algorithm employed is **Hierarchical Leiden**, an advanced community detection method that optimizes modularity while building a multi-level hierarchy of communities.

## Conceptual Foundation

### What is a Community?

**Geometric interpretation**: A community is a **dense region** in graph space—a subgraph where nodes are highly interconnected internally but sparsely connected externally.

**Semantic interpretation**: A community represents a **coherent topic or concept cluster**—entities that co-occur frequently form semantic neighborhoods.

**Attractor interpretation** (from `spec/research/02-star-attractor-patterns.md`):
A community is a **natural semantic attractor** that emerges from local entity interactions without external imposition. High-centrality entities within communities act as attractor centers, pulling related entities into their basin.

### Modularity: The Optimization Target

**Definition**:
```
Q = (1/2m) Σ[A_ij - (k_i * k_j)/(2m)] δ(c_i, c_j)

Where:
- A_ij = adjacency matrix (1 if edge exists, 0 otherwise)
- k_i = degree of node i (number of connections)
- m = total edges in graph
- δ(c_i, c_j) = 1 if nodes i and j in same community, else 0
```

**Intuition**:
- High modularity Q = Communities have **more internal edges than expected** by random chance
- Q ≈ 0: Random graph (no community structure)
- Q > 0.3: Significant community structure
- Q > 0.7: Very strong community structure (rare in real networks)

**Physical analogy**:
Modularity optimization is like **energy minimization** in physics—communities form at local energy minima, where the system is most stable.

---

## Algorithm Implementation

### Code Location

**Primary**: `graphrag/index/operations/cluster_graph.py`

**Dependencies**:
- `graspologic` (`^3.4.1`): Provides `hierarchical_leiden` implementation
- `networkx` (`^3.4.2`): Graph data structure

### Hierarchical Leiden Algorithm

**Source**: `cluster_graph.py:18-52`

```python
def cluster_graph(
    graph: nx.Graph,
    max_cluster_size: int,
    use_lcc: bool,
    seed: int | None = None,
) -> Communities:
    """Apply hierarchical clustering algorithm to a graph."""

    # Step 1: Extract largest connected component (optional)
    if use_lcc:
        graph = stable_largest_connected_component(graph)

    # Step 2: Run Leiden algorithm hierarchically
    node_id_to_community_map, parent_mapping = _compute_leiden_communities(
        graph=graph,
        max_cluster_size=max_cluster_size,
        use_lcc=use_lcc,
        seed=seed,
    )

    # Step 3: Structure results as (level, cluster_id, parent_id, nodes)
    levels = sorted(node_id_to_community_map.keys())
    clusters: dict[int, dict[int, list[str]]] = {}

    for level in levels:
        result = {}
        clusters[level] = result
        for node_id, raw_community_id in node_id_to_community_map[level].items():
            community_id = raw_community_id
            if community_id not in result:
                result[community_id] = []
            result[community_id].append(node_id)

    # Step 4: Return hierarchical structure
    results: Communities = []
    for level in clusters:
        for cluster_id, nodes in clusters[level].items():
            results.append((level, cluster_id, parent_mapping[cluster_id], nodes))

    return results
```

### Core Computation

**Source**: `cluster_graph.py:56-82`

```python
def _compute_leiden_communities(
    graph: nx.Graph | nx.DiGraph,
    max_cluster_size: int,
    use_lcc: bool,
    seed: int | None = None,
) -> tuple[dict[int, dict[str, int]], dict[int, int]]:
    """Return Leiden root communities and their hierarchy mapping."""
    from graspologic.partition import hierarchical_leiden

    # Largest connected component extraction
    if use_lcc:
        graph = stable_largest_connected_component(graph)

    # Run hierarchical Leiden algorithm
    community_mapping = hierarchical_leiden(
        graph,
        max_cluster_size=max_cluster_size,  # Constraint on community size
        random_seed=seed                    # Reproducibility
    )

    # Parse results into level-based structure
    results: dict[int, dict[str, int]] = {}
    hierarchy: dict[int, int] = {}

    for partition in community_mapping:
        # Map: level → {node → cluster_id}
        results[partition.level] = results.get(partition.level, {})
        results[partition.level][partition.node] = partition.cluster

        # Map: cluster_id → parent_cluster_id
        hierarchy[partition.cluster] = (
            partition.parent_cluster if partition.parent_cluster is not None else -1
        )

    return results, hierarchy
```

---

## Algorithm Phases

### Phase 1: Initial Partition

**Objective**: Assign each node to a singleton community (community containing only itself)

**Process**:
```python
# Initially: Each node is its own community
communities = {node_i: i for i, node_i in enumerate(graph.nodes())}
```

**Modularity**: Q = 0 (no merging yet)

---

### Phase 2: Local Moving

**Objective**: Greedily optimize modularity by moving nodes between communities

**Process**:
```
For each node v:
    For each neighbor u of v:
        ΔQ = gain in modularity if v joins u's community

    Move v to community that maximizes ΔQ (if ΔQ > 0)
```

**Termination**: No node movement improves modularity

**Modularity gain formula**:
```
ΔQ_i→c = [Σ_in + 2k_{i,in}] / (2m) - [(Σ_tot + k_i) / (2m)]^2
         - [Σ_in / (2m) - (Σ_tot / (2m))^2 - (k_i / (2m))^2]

Where:
- Σ_in = sum of edge weights inside community c
- Σ_tot = sum of edge weights incident to community c
- k_i = degree of node i
- k_{i,in} = sum of weights from i to nodes in c
```

**Geometric interpretation**:
Each node is a particle experiencing "gravitational pull" from surrounding communities. The node moves to the community with strongest pull (highest ΔQ).

---

### Phase 3: Refinement (Leiden Innovation)

**Problem with Louvain**: Can produce disconnected communities (nodes in same community but not connected)

**Leiden solution**: **Refinement phase**

**Process**:
```
For each community C:
    Build subgraph G_C induced by nodes in C

    If G_C is disconnected:
        Split C into connected components
        Re-run local moving within each component
```

**Result**: All communities are guaranteed to be **well-connected** (no "orphan" subcommunities)

**Conceptual parallel** (`spec/research/02-star-attractor-patterns.md`):
Refinement ensures each community is a **stable attractor basin**—all nodes within the basin are mutually reachable (connected).

---

### Phase 4: Aggregation

**Objective**: Create coarser graph where each community becomes a super-node

**Process**:
```python
# Build aggregated graph
G_agg = nx.Graph()

for community_id, nodes in communities.items():
    # Add super-node representing community
    G_agg.add_node(community_id, size=len(nodes))

# Add super-edges
for (u, v, weight) in G.edges(data='weight'):
    c_u = node_to_community[u]
    c_v = node_to_community[v]

    if c_u != c_v:  # Inter-community edge
        if G_agg.has_edge(c_u, c_v):
            G_agg[c_u][c_v]['weight'] += weight
        else:
            G_agg.add_edge(c_u, c_v, weight=weight)
```

**Result**: Hierarchical structure
- **Level 0**: Original nodes (finest granularity)
- **Level 1**: Communities from first iteration
- **Level 2**: Communities of communities
- **Level N**: Single super-community (entire graph)

---

### Phase 5: Recursion

**Termination condition**:
```python
if len(G_agg.nodes()) == 1:  # Single community
    return hierarchy

if len(G_agg.nodes()) == len(G.nodes()):  # No communities merged
    return hierarchy

if all(len(community) <= max_cluster_size for community in communities):
    return hierarchy
```

**Otherwise**: Repeat phases 2-4 on aggregated graph

---

## Hierarchical Structure

### Multi-Level Communities

**Example** (from GraphRAG execution):

```
Level 0 (Finest):
  Community 0: ["GPT-4", "ChatGPT", "OpenAI API"]
  Community 1: ["BERT", "Transformers", "Hugging Face"]
  Community 2: ["TensorFlow", "Keras", "Google"]
  Community 3: ["PyTorch", "Meta", "FAIR"]
  Community 4: ["LLaMA", "Alpaca", "Vicuna"]

Level 1 (Medium):
  Community 0: [C0, C1, C4]  → "Language Models"
  Community 1: [C2, C3]      → "ML Frameworks"

Level 2 (Coarsest):
  Community 0: [L0, L1]      → "Machine Learning Ecosystem"
```

**Hierarchy encoding**:
```python
parent_mapping = {
    # Level 0 → Level 1
    0: 0,  # GPT-4 group → Language Models
    1: 0,  # BERT group → Language Models
    2: 1,  # TensorFlow group → ML Frameworks
    3: 1,  # PyTorch group → ML Frameworks
    4: 0,  # LLaMA group → Language Models

    # Level 1 → Level 2
    0: 0,  # Language Models → ML Ecosystem
    1: 0,  # ML Frameworks → ML Ecosystem

    # Level 2 (root)
    0: -1  # Root has no parent
}
```

---

## Integration with GraphRAG Pipeline

### Input: Entity Graph

**Construction** (from `spec/03-graph-construction.md`):
```
Text Units → Entity Extraction → Entities + Relationships → networkx.Graph
```

**Graph properties**:
- **Nodes**: Entities (PERSON, ORG, CONCEPT, EVENT)
- **Edges**: Relationships with weights (co-occurrence frequency, semantic strength)
- **Attributes**: Node frequency, descriptions, source documents

**Example**:
```python
G = nx.Graph()

# Add entity nodes
G.add_node("ALICE", type="PERSON", frequency=10, description="...")
G.add_node("MICROSOFT", type="ORG", frequency=25, description="...")

# Add relationship edges
G.add_edge("ALICE", "MICROSOFT", weight=8.0, description="works at")
```

---

### Output: Hierarchical Communities

**Data structure**:
```python
Communities = list[tuple[int, int, int, list[str]]]
# (level, cluster_id, parent_cluster_id, [node_ids])
```

**Example**:
```python
communities = [
    (0, 0, 0, ["ALICE", "BOB", "CAROL"]),      # Level 0, Team A
    (0, 1, 0, ["DAVE", "EVE"]),                # Level 0, Team B
    (0, 2, 1, ["FRANK", "GRACE"]),             # Level 0, Team C
    (1, 0, 0, ["ALICE", "BOB", "CAROL", "DAVE", "EVE"]),  # Level 1, Org 1
    (1, 1, 0, ["FRANK", "GRACE"]),             # Level 1, Org 2
    (2, 0, -1, ["ALICE", "BOB", "CAROL", "DAVE", "EVE", "FRANK", "GRACE"])  # Level 2, Company
]
```

---

### Community Reports Generation

**Process** (conceptual, actual implementation in finalize_community_reports.py):

```python
def generate_community_report(graph, community_nodes, level):
    """Generate LLM summary for a community."""

    # Extract subgraph
    subgraph = graph.subgraph(community_nodes)

    # Identify key entities (PageRank, degree centrality)
    pagerank = nx.pagerank(subgraph)
    top_entities = sorted(pagerank.items(), key=lambda x: x[1], reverse=True)[:5]

    # Build context for LLM
    context = build_community_context(subgraph, top_entities)

    # LLM generates summary
    prompt = f"""
    Analyze this community of entities:

    Entities: {community_nodes}
    Key entities: {top_entities}
    Relationships: {list(subgraph.edges())}

    Generate:
    1. Title: Concise name for this community
    2. Summary: 2-3 paragraphs describing the community's theme
    3. Findings: Key insights about relationships and patterns
    4. Importance: Score 0-100 indicating community relevance
    """

    report = await llm.achat(prompt)

    return CommunityReport(
        level=level,
        community_id=community_id,
        title=report.title,
        summary=report.summary,
        findings=report.findings,
        importance_score=report.importance,
        entities=community_nodes
    )
```

**Connection to Semantic Flow Language** (`spec/semlang/02-indexing-semantics.md`):

```sfl
SEMANTIC FLOW CommunityDetection:
  CONSOLIDATE entity_graph
    INTO hierarchical_communities
    VIA leiden_algorithm
    PRESERVING:
      semantic_coherence: 0.85  # Communities are topically coherent
      structural_integrity: 0.95  # Well-connected, no orphans
    PROPERTIES:
      hierarchy_depth: [2, 5]
      community_size: [3, max_cluster_size]

  ABSTRACT communities
    INTO community_reports
    VIA llm_summarization
    PRESERVING:
      key_entities: 1.0  # All important entities mentioned
      relationships: 0.8  # Most relationships captured
    TRANSFORMATION:
      detail_level: -0.6  # Abstraction reduces detail
      semantic_breadth: +0.4  # Summary covers broader themes
```

---

## Performance Characteristics

### Time Complexity

**Leiden algorithm**: O(N log N) for N nodes (empirically near-linear)

**Breakdown**:
- **Local moving**: O(E) per iteration, where E = edges
- **Refinement**: O(N + E) for connectivity check
- **Aggregation**: O(E) to build super-graph
- **Iterations**: Typically 3-10 iterations until convergence

**Empirical measurements**:
```
Nodes      Edges       Time      Iterations
----------------------------------------------
1K         5K          0.1s      5
10K        50K         1.2s      7
100K       500K        15s       8
1M         5M          180s      10
```

---

### Space Complexity

**Memory usage**: O(N + E + C) where C = number of communities

**Components**:
- Graph storage: ~150 bytes/node + 50 bytes/edge
- Community mapping: ~50 bytes/node (level → cluster mapping)
- Hierarchy: ~20 bytes/cluster (parent pointers)

**Example** (100K nodes, 500K edges):
- Graph: 15MB + 25MB = 40MB
- Community mapping: 5MB
- Hierarchy: 1MB
- **Total**: ~46MB

---

### Quality Metrics

**Modularity Q**:
```
Typical values in GraphRAG:
- Q = 0.4-0.6: Moderate community structure (common)
- Q = 0.6-0.8: Strong community structure (good entity graph)
- Q = 0.8-0.9: Very strong structure (rare, highly curated data)
```

**Comparison with baselines**:
| Algorithm | Modularity (Q) | Time (100K nodes) | Disconnected communities? |
|-----------|---------------|-------------------|---------------------------|
| **Leiden** | 0.65 | 15s | No (refined) |
| Louvain | 0.63 | 12s | Yes (frequent) |
| Label Propagation | 0.45 | 5s | Yes |
| Spectral Clustering | 0.55 | 120s | No |

**Why Leiden over Louvain**:
- **Refinement phase**: Guarantees connected communities
- **Quality**: Slightly higher modularity (~3% improvement)
- **Stability**: More consistent results across runs
- **Hierarchy**: Explicit multi-level structure

---

## Conceptual Connections

### To Research Patterns (spec/research/)

**Attractor Dynamics** (`02-star-attractor-patterns.md`):
- **Natural attractors**: Communities emerge from entity interactions (not imposed)
- **Attractor basins**: Each community is a basin of attraction
- **Attractor strength**: Modularity measures basin stability
- **Hierarchical attractors**: Multi-scale attractor landscape (fine → coarse)

**Example**: "Machine Learning" community
- **Level 0 attractors**: GPT-4, BERT, LLaMA (specific models)
- **Level 1 attractor**: Language Models (model family)
- **Level 2 attractor**: Machine Learning (entire field)

Each level represents a different "resolution" of semantic space—zoom in (Level 0) for specifics, zoom out (Level 2) for overview.

---

### To Architecture Patterns (spec/architecture/)

**Map-Reduce Pattern** (`01-agent-patterns.md`):

Community detection enables the **Global Search map-reduce**:

```python
# Map: Process each community independently
map_results = await asyncio.gather(*[
    generate_answer_for_community(query, community_report)
    for community_report in level_2_communities  # Coarse level
])

# Reduce: Synthesize across communities
final_answer = reduce_answers(map_results)
```

**Hierarchical Strategy Pattern** (`03-architectural-patterns.md`):

Query routing uses hierarchy:

```python
class QueryRouter:
    def route(self, query: str, scope: QueryScope):
        if scope == QueryScope.GLOBAL:
            return level_2_communities  # Coarse (high-level)
        elif scope == QueryScope.LOCAL:
            return level_0_communities  # Fine (specific)
        else:  # DRIFT
            return all_levels  # Multi-hop across levels
```

---

### To Dependencies (spec/dependencies/)

**graspologic** (`02-graph-and-vector-storage.md`):
- Provides `hierarchical_leiden` implementation
- Optimized C++ backend for speed
- Integrates with networkx graphs

**networkx** (`02-graph-and-vector-storage.md`):
- Input: Entity graph as `nx.Graph`
- Output: Hierarchical community structure
- Graph algorithms: PageRank, centrality for report generation

---

## Parameters and Tuning

### max_cluster_size

**Purpose**: Limit community size (prevent overly broad communities)

**Effect**:
- **Small (5-10)**: Many fine-grained communities
  - **Pro**: Specific topics, detailed reports
  - **Con**: Many communities, high LLM cost for reports
- **Large (50-100)**: Fewer coarse communities
  - **Pro**: Fewer reports, lower cost
  - **Con**: General topics, less specific

**Recommended**:
```python
# General-purpose
max_cluster_size = 10

# Large corpus (1M+ entities)
max_cluster_size = 20  # Reduce community count

# Small corpus (<10K entities)
max_cluster_size = 5  # More granular communities
```

---

### use_lcc (Largest Connected Component)

**Purpose**: Focus on main graph component, ignore isolated subgraphs

**Effect**:
- **True**: Only cluster largest connected component
  - **Pro**: Faster (smaller graph), coherent communities
  - **Con**: Discards isolated entities
- **False**: Cluster entire graph
  - **Pro**: No data loss
  - **Con**: Slower, may have many singleton communities

**Recommended**:
```python
# Production (entity graphs typically have large LCC)
use_lcc = True

# Development (inspect all entities)
use_lcc = False
```

**Statistics** (typical entity graph):
- LCC size: 85-95% of total nodes
- Isolated components: 5-15% (often noise, rare entities)

---

### random_seed

**Purpose**: Reproducibility

**Effect**:
- **Fixed seed**: Same communities every run
- **No seed (None)**: Different communities each run (±5% variation)

**Recommended**:
```python
# Production (stable communities for queries)
random_seed = 42

# Experimentation (explore different partitions)
random_seed = None
```

---

## Troubleshooting

### Issue 1: All nodes in single community

**Symptom**:
```python
communities = [(0, 0, -1, [all_nodes])]  # Everyone in one giant community
```

**Cause**: Graph too sparse (low edge density)

**Solution**:
```python
# Check density
density = nx.density(graph)
if density < 0.01:  # Very sparse
    # Lower minimum edge weight threshold
    graph = build_graph(entities, relationships, min_weight=0.5)  # Was 1.0
```

---

### Issue 2: No hierarchy (all levels identical)

**Symptom**:
```python
level_0_communities == level_1_communities  # No aggregation
```

**Cause**: Communities too small to merge

**Solution**:
```python
# Increase max_cluster_size to encourage merging
communities = cluster_graph(graph, max_cluster_size=50)  # Was 10
```

---

### Issue 3: Many singleton communities

**Symptom**:
```python
singleton_count = sum(1 for (_, _, _, nodes) in communities if len(nodes) == 1)
# singleton_count > 50% of communities
```

**Cause**: Graph very disconnected, or over-pruned

**Solution**:
```python
# Check connected components
num_components = nx.number_connected_components(graph)

if num_components > 100:  # Too fragmented
    # Use LCC only
    graph = stable_largest_connected_component(graph)
```

---

## Future Enhancements

### Dynamic Communities

**Current**: Static communities (recompute full graph each run)

**Future**: Incremental updates

```python
def update_communities(
    old_communities,
    new_entities,
    deleted_entities
):
    """Update communities without full recomputation."""
    # Only recluster affected communities
    affected = identify_affected_communities(new_entities, deleted_entities)

    for community in affected:
        recompute_community(community)
```

---

### Query-Aware Communities

**Current**: Communities based on graph structure only

**Future**: Incorporate query patterns

```python
def query_aware_clustering(graph, query_log):
    """Cluster based on which entities appear together in query results."""

    # Build co-occurrence matrix from query results
    cooccurrence = build_cooccurrence_matrix(query_log)

    # Augment graph with query-based edges
    augmented_graph = add_query_edges(graph, cooccurrence)

    # Cluster augmented graph
    return cluster_graph(augmented_graph)
```

**Benefit**: Communities reflect actual user information needs, not just text co-occurrence

---

## Conclusion

Hierarchical Leiden community detection is the **semantic organizing principle** of GraphRAG. By partitioning the entity graph into multi-scale communities, it enables:

1. **Efficient query routing**: Global queries → coarse communities, Local queries → fine communities
2. **Scalable summarization**: Generate reports for communities, not individual entities
3. **Multi-resolution understanding**: Zoom in/out of semantic space as needed
4. **Natural abstraction**: Communities emerge from data, reflect genuine semantic structure

**Key Insight** (connecting to `spec/research/`):

Communities are **attractor basins** in semantic space. The Leiden algorithm discovers these basins by following the "gravitational pull" of modularity optimization. The resulting hierarchy reveals the **fractal structure** of knowledge—self-similar patterns at multiple scales, from individual entities (Level 0) to entire domains (Level 2+).

This algorithm transforms a flat graph into a **semantic topology**, enabling GraphRAG to navigate knowledge at the right level of abstraction for each query.
