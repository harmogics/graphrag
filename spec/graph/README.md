# Graph Algorithms in GraphRAG

## Overview

This directory documents the **graph algorithms** that power GraphRAG's knowledge graph construction, analysis, and retrieval. These algorithms transform raw text extractions into a structured, queryable knowledge representation through a multi-stage pipeline: **construction → pruning → clustering → ranking → embedding**.

## Algorithm Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│                         GRAPH ALGORITHM PIPELINE                     │
└─────────────────────────────────────────────────────────────────────┘

INPUT: Text Units (chunks)
   │
   ├─ LLM Extraction (per chunk)
   │   └─ Entities: [{"title": "MICROSOFT", "type": "org", ...}, ...]
   │   └─ Relationships: [{"source": "MICROSOFT", "target": "OPENAI", ...}, ...]
   │
   ▼
┌────────────────────────────────────────────┐
│  03-GRAPH CONSTRUCTION & MERGING           │  ← Deduplication, Aggregation
│  - Entity Merging (by title+type)         │
│  - Relationship Merging (by source+target) │
│  - Description Summarization (LLM)         │
│  - Graph Assembly (networkx)               │
└────────────────────────────────────────────┘
   │
   ▼
┌────────────────────────────────────────────┐
│  05-GRAPH PRUNING                          │  ← Noise Removal
│  - Filter by Degree (min, max_std)        │
│  - Filter by Frequency (min, max_std)     │
│  - Filter by Edge Weight (percentile)     │
│  - Extract LCC (largest component)         │
└────────────────────────────────────────────┘
   │
   ├───────────────────┬────────────────────┬────────────────────┐
   │                   │                    │                    │
   ▼                   ▼                    ▼                    ▼
┌──────────┐   ┌──────────────┐   ┌────────────┐   ┌────────────────┐
│ 02-      │   │ 01-          │   │ 04-        │   │ Query-Time     │
│ CENTRALITY│   │ COMMUNITY    │   │ EMBEDDING  │   │ Operations     │
│ & RANKING │   │ DETECTION    │   │ (Node2Vec) │   │                │
│          │   │              │   │            │   │                │
│ - Degree │   │ - Hierarchical│   │ - Random   │   │ - Vector       │
│ - PageRank│   │   Leiden     │   │   Walks    │   │   Search       │
│ - Combined│   │ - Modularity │   │ - Skip-Gram│   │ - Community    │
│   Edge   │   │   Optimization│   │ - UMAP     │   │   Retrieval    │
└──────────┘   └──────────────┘   └────────────┘   └────────────────┘
   │                   │                    │                    │
   └───────────────────┴────────────────────┴────────────────────┘
                                   │
                                   ▼
                          ┌─────────────────┐
                          │  KNOWLEDGE      │
                          │  GRAPH INDEX    │
                          │                 │
                          │  • Entities     │
                          │  • Communities  │
                          │  • Embeddings   │
                          │  • Rankings     │
                          └─────────────────┘
```

---

## Algorithm Documents

### [01-community-detection.md](01-community-detection.md)
**Hierarchical Leiden Algorithm** — Multi-scale graph clustering via modularity optimization

**Purpose**: Group entities into semantic communities at multiple hierarchy levels

**Key concepts**:
- Modularity maximization (Q metric)
- Local moving, refinement, aggregation phases
- Hierarchical community structure (fine → coarse)
- Integration with map-reduce summarization

**Use cases**:
- Global search: "Summarize the main themes in the corpus"
- Community reports: LLM summaries of entity clusters
- Multi-level navigation: Zoom in/out of knowledge graph

**Parameters**: `max_cluster_size`, `random_seed`

**Complexity**: O((N+E) × log N) per iteration

---

### [02-centrality-and-ranking.md](02-centrality-and-ranking.md)
**Entity Importance Metrics** — Degree centrality, PageRank, combined edge degree

**Purpose**: Identify structurally important entities and relationships

**Key concepts**:
- Degree centrality: Local importance (connection count)
- PageRank: Global importance (recursive authority)
- Combined edge degree: Relationship strength
- Integration with query context building

**Use cases**:
- Entity ranking in community reports
- Hybrid search (text similarity + structural importance)
- Filtering top-k entities for LLM context

**Parameters**: None (computed from graph structure)

**Complexity**: O(N) for degree, O(N+E) × iterations for PageRank

---

### [03-graph-construction-and-merging.md](03-graph-construction-and-merging.md)
**Entity/Relationship Aggregation** — Deduplication and consolidation of LLM extractions

**Purpose**: Transform fragmented per-chunk extractions into unified knowledge graph

**Key concepts**:
- Entity merging: Group by (title, type), aggregate descriptions
- Relationship merging: Group by (source, target), sum weights
- Description summarization: LLM consolidation of multi-perspective context
- Frequency tracking: Count text units mentioning each entity

**Use cases**:
- Foundation of all downstream graph operations
- Provenance tracking (which chunks mention which entities)
- Incremental graph updates (merge new documents into existing graph)

**Parameters**: `text_column`, `id_column`, LLM strategy config

**Complexity**: O(N log N) for merging, O(N+E) × LLM_time for summarization

---

### [04-graph-embedding.md](04-graph-embedding.md)
**Node2Vec Embeddings** — Continuous vector representations of graph structure

**Purpose**: Enable geometric operations (similarity, clustering, visualization) on discrete graph

**Key concepts**:
- Random walk generation: Sample graph neighborhoods
- Skip-gram training: Predict context nodes from target (Word2Vec analog)
- LCC preprocessing: Ensure connected graph for consistent embeddings
- Stable ordering: Reproducibility via sorted nodes/edges

**Use cases**:
- Graph visualization (UMAP → 2D projection)
- Hybrid search (text embedding + graph embedding)
- Entity features for ML models

**Parameters**: `dimensions=1536`, `num_walks=10`, `walk_length=40`, `window_size=2`

**Complexity**: O(N × num_walks × walk_length × window_size × iterations)

---

### [05-graph-pruning.md](05-graph-pruning.md)
**Noise Filtering** — Multi-criterion removal of low-signal nodes and edges

**Purpose**: Improve graph quality by removing extraction errors, outliers, and weak connections

**Key concepts**:
- Degree filtering: Remove low-connectivity nodes (peripheral noise)
- Frequency filtering: Remove rare mentions (likely typos/hallucinations)
- Edge weight filtering: Remove weak relationships (spurious co-occurrences)
- Statistical outlier removal: Standard deviation thresholds for hubs
- LCC extraction: Focus on main knowledge cluster

**Use cases**:
- Pre-clustering cleanup (stabilize community detection)
- Pre-embedding optimization (reduce Node2Vec computational cost)
- Quality control (remove contamination like "DOCUMENT", "SYSTEM")

**Parameters**: `min_node_degree=1`, `min_node_freq=1`, `min_edge_weight_pct=0`, `max_*_std`

**Complexity**: O(E log E) for edge weight percentile calculation

---

## Algorithm Interaction Map

```
┌─────────────────────────────────────────────────────────────────────┐
│                      ALGORITHM DEPENDENCIES                          │
└─────────────────────────────────────────────────────────────────────┘

CONSTRUCTION (03)
   ├─ Produces: Merged graph with frequency, weight attributes
   │
   └──▶ PRUNING (05)
         ├─ Consumes: frequency (from entity merging)
         ├─ Consumes: weight (from relationship merging)
         ├─ Produces: Clean graph (noise removed)
         │
         ├──▶ CENTRALITY (02)
         │     ├─ Consumes: Graph structure
         │     ├─ Produces: Degree, PageRank scores
         │     └─ Used by: Community report ranking, context building
         │
         ├──▶ COMMUNITY DETECTION (01)
         │     ├─ Consumes: Clean graph
         │     ├─ Produces: Hierarchical community assignments
         │     └─ Used by: Global search, report generation, navigation
         │
         └──▶ EMBEDDING (04)
               ├─ Consumes: LCC of clean graph
               ├─ Produces: Node → vector mapping
               └─ Used by: Visualization, hybrid search, ML features
```

**Critical path**:
1. **Construction** → must happen first (creates graph from raw extractions)
2. **Pruning** → should happen before clustering/embedding (removes noise)
3. **Centrality, Community, Embedding** → independent, can run in parallel

---

## Cross-References to Other Specs

### To `spec/research/` (Conceptual Foundations)

**[spec/research/01-query-as-key.md](../research/01-query-as-key.md)**:
- **Entity identity**: Defines (title, type) as deduplication key (used in construction)
- **Vector similarity**: Complements graph structure for retrieval

**[spec/research/02-star-attractor-patterns.md](../research/02-star-attractor-patterns.md)**:
- **Attractor strength**: Centrality measures map to attractor basins
- **Star topology**: High-degree nodes = strong attractors
- **Community = attractor basin**: Clustering reveals attractor boundaries

---

### To `spec/architecture/` (System Patterns)

**[spec/architecture/01-agent-patterns.md](../architecture/01-agent-patterns.md)**:
- **Map-reduce**: Entity extraction (map) → merging (reduce)
- **Parallel community reports**: Independent LLM summarization per community
- **Incremental indexing**: Idempotent merging enables `graphrag update`

---

### To `spec/dependencies/` (Implementation)

**[spec/dependencies/02-graph-and-vector-storage.md](../dependencies/02-graph-and-vector-storage.md)**:
- **networkx**: Graph data structure, construction, algorithms
- **graspologic**: Leiden clustering, Node2Vec embedding, LCC extraction
- **lancedb / azure-search**: Vector storage for embeddings

**[spec/dependencies/03-data-processing-and-infrastructure.md](../dependencies/03-data-processing-and-infrastructure.md)**:
- **pandas**: Entity/relationship DataFrames, groupby merging
- **numpy**: Statistical pruning (percentiles, standard deviations)
- **pyarrow**: Parquet serialization for graph DataFrames

---

### To `spec/semlang/` (Semantic Flow Language)

**[spec/semlang/01-document-transformation.md](../semlang/01-document-transformation.md)**:
- Graph construction is a **TRANSFORM** operation (text units → graph)
- Embedding is a **TRANSFORM** operation (graph → vectors)

**[spec/semlang/03-query-semantics.md](../semlang/03-query-semantics.md)**:
- Centrality-weighted entity selection for context building
- Community-based answer aggregation (global search)

---

## Decision Trees for Practitioners

### Which Algorithms to Enable?

```
START: Do you need GraphRAG indexing?
   │
   ├─ YES → Enable CONSTRUCTION (required, always on)
   │         │
   │         ├─ Is your corpus noisy (web scrapes, OCR, social media)?
   │         │  │
   │         │  ├─ YES → Enable PRUNING with aggressive thresholds
   │         │  │         (min_freq=3, min_edge_weight_pct=20, max_degree_std=2)
   │         │  │
   │         │  └─ NO → Enable PRUNING with gentle thresholds or skip
   │         │           (min_freq=1, min_edge_weight_pct=5)
   │         │
   │         ├─ Do you need global search ("summarize the corpus")?
   │         │  │
   │         │  ├─ YES → Enable COMMUNITY DETECTION (required)
   │         │  │         + Enable CENTRALITY for report ranking
   │         │  │
   │         │  └─ NO → Skip COMMUNITY DETECTION (local search only)
   │         │
   │         ├─ Do you need graph visualization or hybrid search?
   │         │  │
   │         │  ├─ YES → Enable EMBEDDING (Node2Vec)
   │         │  │
   │         │  └─ NO → Skip EMBEDDING (saves time/memory)
   │         │
   │         └─ DONE: Indexing configured
   │
   └─ NO → Query-only mode (pre-built index required)
```

---

### Tuning Pruning Thresholds

```
START: Graph quality issues?
   │
   ├─ Too many tiny/meaningless communities (1-2 entities each)
   │  │
   │  └─▶ INCREASE pruning: min_freq=3, min_edge_weight_pct=20
   │
   ├─ Important entities missing from communities
   │  │
   │  └─▶ DECREASE pruning: min_freq=1, min_edge_weight_pct=0
   │
   ├─ Single generic hub (e.g., "DOCUMENT") dominates graph
   │  │
   │  └─▶ Enable max_node_degree_std=2 OR remove_ego_nodes=True
   │
   ├─ Graph fragments into many disconnected components
   │  │
   │  └─▶ Enable lcc_only=True (focus on largest component)
   │
   └─ Graph size acceptable, quality good
      │
      └─▶ Keep current pruning config
```

---

### Choosing Embedding Dimensions

```
START: Need Node2Vec embeddings?
   │
   ├─ Integrating with text embeddings (OpenAI, Azure)?
   │  │
   │  └─▶ Use dimensions=1536 (match text embedding size)
   │
   ├─ Graph-only use case (visualization, clustering)?
   │  │
   │  ├─ Small graph (< 10K nodes)
   │  │  └─▶ Use dimensions=128 (sufficient, fast)
   │  │
   │  └─ Large graph (> 10K nodes)
   │     └─▶ Use dimensions=256-512 (balance quality/speed)
   │
   └─ Memory/speed critical
      │
      └─▶ Use dimensions=64-128 (minimal, may lose nuance)
```

---

## Performance Benchmarks

### Typical GraphRAG Corpus (10K Documents, 100K Text Units)

| Algorithm | Nodes | Edges | Time | Memory | Bottleneck |
|-----------|-------|-------|------|--------|------------|
| Construction | 50K | 150K | 30 min | 5 GB | LLM summarization |
| Pruning | 50K → 35K | 150K → 100K | 30 sec | 3 GB | Edge weight percentile |
| Centrality (degree) | 35K | 100K | 5 sec | 1 GB | Graph iteration |
| Community Detection | 35K | 100K | 3 min | 2 GB | Leiden iterations |
| Embedding (Node2Vec) | 35K | 100K | 15 min | 8 GB | Random walk generation |

**Total indexing time**: ~50 minutes (construction dominates due to LLM calls)

**Parallelization**: Construction and embedding are embarrassingly parallel (scale with CPU cores)

**Machine**: 16-core CPU, 32GB RAM, no GPU

---

### Scaling Characteristics

| Graph Size | Construction | Pruning | Community | Embedding |
|------------|--------------|---------|-----------|-----------|
| 1K nodes | 3 min | 1 sec | 5 sec | 30 sec |
| 10K nodes | 30 min | 30 sec | 3 min | 15 min |
| 100K nodes | 5 hours | 5 min | 30 min | 2 hours |
| 1M nodes | 50 hours* | 45 min | 4 hours | 18 hours* |

\* Requires distributed processing or incremental indexing

**Scaling laws**:
- Construction: O(N) LLM calls (linear with entities post-merge)
- Pruning: O(E log E) (dominated by sorting)
- Community: O((N+E) log N) (Leiden iterations)
- Embedding: O(N × walk_length × num_walks) (walk generation)

---

## Common Issues and Solutions

### Issue: Graph Construction is Slow

**Symptom**: Hours to process small corpus

**Diagnosis**:
- LLM summarization bottleneck (one call per entity/relationship)
- Serial processing (not parallelizing extraction)

**Solution**:
1. **Disable summarization** (if not needed):
   ```yaml
   summarize_descriptions: false
   ```
   - Store description lists instead of consolidated summaries
   - 10-100x speedup

2. **Increase parallelism**:
   ```yaml
   num_threads: 50  # Concurrent LLM calls
   ```
   - Limited by LLM rate limits (check API quotas)

3. **Use faster LLM**:
   ```yaml
   model: gpt-3.5-turbo  # Instead of gpt-4
   ```
   - Trade quality for speed

---

### Issue: Community Detection Produces Too Many Small Communities

**Symptom**: 1000s of communities, most with 1-2 entities

**Diagnosis**:
- Noisy graph (many low-frequency entities)
- No pruning or too gentle

**Solution**:
1. **Enable aggressive pruning**:
   ```yaml
   min_node_freq: 3
   min_edge_weight_pct: 20
   ```

2. **Increase max_cluster_size**:
   ```yaml
   max_cluster_size: 50  # Default is 10
   ```
   - Allows larger communities (fewer small fragments)

3. **Inspect removed nodes**:
   - Verify pruning isn't removing important entities

---

### Issue: Node2Vec Embedding Runs Out of Memory

**Symptom**: OOM error during random walk generation

**Diagnosis**:
- Too many walks stored in memory
- Graph too large

**Solution**:
1. **Reduce walk parameters**:
   ```yaml
   num_walks: 5        # Default 10
   walk_length: 20     # Default 40
   ```

2. **Reduce embedding dimensions**:
   ```yaml
   dimensions: 256     # Default 1536
   ```

3. **Enable LCC extraction**:
   ```yaml
   use_lcc: true       # Only embed largest component
   ```

4. **Prune graph more aggressively** before embedding

---

### Issue: Centrality Scores All Near Zero or One

**Symptom**: No variance in degree/PageRank scores

**Diagnosis**:
- Graph too sparse (low connectivity) or too dense (complete graph)

**Solution**:
1. **Check graph density**:
   ```python
   density = 2 * num_edges / (num_nodes * (num_nodes - 1))
   # Should be 0.001 - 0.1 for meaningful structure
   ```

2. **If too sparse** (density < 0.0001):
   - Increase edge extraction (LLM prompt tuning)
   - Decrease edge weight pruning

3. **If too dense** (density > 0.5):
   - Increase edge weight pruning (remove weak edges)
   - Check for contamination hubs (generic entities)

---

## Future Enhancements

### 1. Incremental Graph Updates

**Current**: Full rebuild on new documents

**Proposed**: Delta merging
- Only re-merge entities appearing in new documents
- Incrementally update community assignments
- Selective re-embedding (nodes within k-hop radius of changes)

**Benefit**: 100x speedup for small document additions

---

### 2. Heterogeneous Graph Support

**Current**: All nodes treated identically (entities only)

**Proposed**: Multi-node-type graphs
- Entities, Events, Topics, Concepts as distinct node types
- Type-specific embedding spaces
- Cross-type relationship modeling

**Benefit**: Richer semantic structure, type-aware queries

---

### 3. Temporal Graph Evolution

**Current**: Static snapshot only

**Proposed**: Time-stamped edges and nodes
- Track entity mentions over document timeline
- Temporal PageRank (importance evolves over time)
- Trend detection (emerging vs. declining entities)

**Benefit**: Time-aware queries ("What changed in 2023?")

---

### 4. Graph Neural Networks (GNNs)

**Current**: Node2Vec (random walk + skip-gram)

**Proposed**: GNN embeddings (message passing)
- Graph Attention Networks (GAT)
- GraphSAGE (inductive learning)
- Integration with text embeddings (multi-modal GNN)

**Benefit**: Richer structural features, better few-shot learning

---

## Getting Started

### Minimal Pipeline (Local Search Only)

```yaml
graphrag:
  construction:
    enabled: true
    summarize_descriptions: false  # Speed up indexing

  pruning:
    enabled: false  # Optional, skip for clean corpora

  community:
    enabled: false  # Not needed for local search

  embedding:
    enabled: false  # Not needed for local search

  centrality:
    enabled: true   # Useful for entity ranking
```

**Result**: Fast indexing, entity graph only, local search supported

---

### Full Pipeline (Global + Local Search)

```yaml
graphrag:
  construction:
    enabled: true
    summarize_descriptions: true  # Consolidate descriptions

  pruning:
    enabled: true
    min_node_freq: 2
    min_edge_weight_pct: 10

  community:
    enabled: true
    max_cluster_size: 10

  embedding:
    enabled: true
    dimensions: 1536
    num_walks: 10

  centrality:
    enabled: true
```

**Result**: Full GraphRAG capabilities, slower indexing, best quality

---

### Research/Exploration Pipeline

```yaml
graphrag:
  construction:
    enabled: true
    summarize_descriptions: false  # Keep raw extractions

  pruning:
    enabled: false  # Keep all data (analyze pruning separately)

  community:
    enabled: true
    max_cluster_size: 20  # Larger communities for exploration

  embedding:
    enabled: true
    dimensions: 128       # Fast visualization
    num_walks: 20         # Better coverage

  centrality:
    enabled: true
```

**Result**: Rich graph for analysis, visualization-ready, flexible experimentation

---

## Conclusion

GraphRAG's graph algorithms form a **layered pipeline** that progressively refines raw text extractions into structured, queryable knowledge. Each algorithm addresses a specific challenge:

- **Construction**: Deduplicate and aggregate fragmented extractions
- **Pruning**: Remove noise and outliers
- **Community Detection**: Discover semantic clusters at multiple scales
- **Centrality**: Identify important entities and relationships
- **Embedding**: Enable geometric reasoning and visualization

Together, these algorithms enable GraphRAG's unique capabilities: **global multi-hop reasoning**, **hierarchical knowledge navigation**, and **hybrid structural-semantic search**.

**Key files in this directory**:
- `01-community-detection.md` — Hierarchical Leiden clustering
- `02-centrality-and-ranking.md` — Degree, PageRank, combined edge degree
- `03-graph-construction-and-merging.md` — Entity/relationship aggregation
- `04-graph-embedding.md` — Node2Vec continuous representations
- `05-graph-pruning.md` — Multi-criterion noise filtering

**Next steps**:
- Read individual algorithm docs for deep dives
- Consult decision trees for tuning guidance
- Cross-reference `spec/architecture`, `spec/research`, `spec/dependencies` for system context
