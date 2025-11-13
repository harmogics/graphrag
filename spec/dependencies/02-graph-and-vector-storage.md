# Graph Processing and Vector Storage Dependencies

## Overview

This document describes dependencies used for graph construction, analysis, community detection, and vector similarity search in GraphRAG. These libraries form the **structural organization layer**, enabling the system to represent knowledge as interconnected graphs and perform efficient semantic search through high-dimensional vector spaces.

## Graph Processing Libraries

### networkx

**Version**: `^3.4.2`

**Purpose**: Python graph library for creating, manipulating, and studying complex networks

**Role in GraphRAG**:
- **Entity graph construction**: Build graph from extracted entities and relationships
- **Graph algorithms**: Compute centrality, shortest paths, connected components
- **Graph manipulation**: Add/remove nodes/edges, merge graphs
- **Graph export**: Convert to GraphML, JSON, other formats
- **Visualization support**: Prepare data for graph visualization

**Architecture Integration**:
```
Entity Extraction → networkx Graph → Community Detection → Query Navigation
```

**Key Features Used**:

**1. Graph Construction**:
```python
import networkx as nx

# Create undirected graph (entities connected bidirectionally)
G = nx.Graph()

# Add entity nodes
G.add_node(
    "ALICE",
    type="PERSON",
    description="Software engineer at Microsoft",
    source_id="doc_1"
)

# Add relationship edges
G.add_edge(
    "ALICE",
    "MICROSOFT",
    weight=5.0,  # Relationship strength
    description="Alice works at Microsoft",
    source_id="doc_1"
)
```

**2. Graph Analysis**:
```python
# Degree centrality (how connected each entity is)
degree_centrality = nx.degree_centrality(G)
# {'ALICE': 0.75, 'MICROSOFT': 0.85, ...}

# PageRank (importance based on connections)
pagerank = nx.pagerank(G)
# {'MICROSOFT': 0.35, 'ALICE': 0.15, ...}

# Connected components (isolated subgraphs)
components = list(nx.connected_components(G))
# [{'ALICE', 'MICROSOFT', 'AZURE'}, {'GOOGLE', 'TENSORFLOW'}]

# Shortest path (entity navigation)
path = nx.shortest_path(G, source="ALICE", target="AZURE")
# ['ALICE', 'MICROSOFT', 'AZURE']
```

**3. Graph Merging** (from multiple documents):
```python
def merge_entity_graphs(graphs: list[nx.Graph]) -> nx.Graph:
    """Merge multiple graphs into unified knowledge graph."""
    merged = nx.Graph()

    for graph in graphs:
        for node, attrs in graph.nodes(data=True):
            if node in merged.nodes:
                # Node exists: merge descriptions
                merged.nodes[node]['description'] += '\n' + attrs['description']
                merged.nodes[node]['source_id'] += ', ' + attrs['source_id']
            else:
                # New node: add to merged graph
                merged.add_node(node, **attrs)

        for u, v, attrs in graph.edges(data=True):
            if merged.has_edge(u, v):
                # Edge exists: sum weights
                merged[u][v]['weight'] += attrs['weight']
            else:
                # New edge
                merged.add_edge(u, v, **attrs)

    return merged
```

**Code Locations**:
- `graphrag/index/operations/create_graph.py` - Graph construction from entities
- `graphrag/index/operations/cluster_graph.py` - Uses networkx as input to Leiden
- `graphrag/index/operations/snapshot_graphml.py` - Export graph to GraphML format
- `graphrag/query/structured_search/drift_search/state.py` - DRIFT navigation graph

**Performance Characteristics**:
- **Node capacity**: Scales to ~1M nodes efficiently
- **Edge capacity**: Scales to ~10M edges
- **Algorithm speed**: O(N log N) to O(N²) depending on algorithm
- **Memory**: ~100 bytes per node + 50 bytes per edge

**Why networkx**:
- **Pure Python**: Easy to integrate, debug, extend
- **Rich algorithms**: 100+ graph algorithms built-in
- **Flexible data model**: Arbitrary attributes on nodes/edges
- **Community support**: Well-documented, widely used

**Alternatives Considered**:
- **igraph**: Faster but C-based (harder to debug)
- **graph-tool**: Most performant but complex installation
- **Neo4j**: Overkill for in-memory graphs, requires separate DB

---

### graspologic

**Version**: `^3.4.1`

**Purpose**: Graph statistics and machine learning library (formerly Microsoft Research's graspy)

**Role in GraphRAG**:
- **Community detection**: Hierarchical Leiden algorithm for clustering entities
- **Graph embeddings**: Node2vec, spectral embeddings
- **Graph matching**: Align graphs across different time periods
- **Visualization**: t-SNE, UMAP projections of graph structure

**Architecture Integration**:
```
Entity Graph → graspologic.hierarchical_leiden → Hierarchical Communities → Community Reports
```

**Key Features Used**:

**1. Hierarchical Leiden Clustering**:

The core algorithm for detecting semantic communities in GraphRAG:

```python
from graspologic.partition import hierarchical_leiden

# Input: networkx graph with weighted edges
G = nx.Graph()
# ... populate with entities and relationships ...

# Run hierarchical clustering
community_mapping = hierarchical_leiden(
    G,
    max_cluster_size=10,  # Maximum entities per community
    random_seed=42       # Reproducibility
)

# Output: Hierarchical structure
# [
#   Partition(node='ALICE', level=0, cluster=0, parent_cluster=0),
#   Partition(node='ALICE', level=1, cluster=0, parent_cluster=-1),
#   Partition(node='BOB', level=0, cluster=0, parent_cluster=0),
#   Partition(node='CHARLIE', level=0, cluster=1, parent_cluster=0),
#   ...
# ]
```

**2. Multi-Level Community Structure**:

```python
def extract_hierarchical_communities(community_mapping):
    """Extract communities at each hierarchical level."""
    communities_by_level = {}

    for partition in community_mapping:
        level = partition.level
        if level not in communities_by_level:
            communities_by_level[level] = {}

        cluster_id = partition.cluster
        if cluster_id not in communities_by_level[level]:
            communities_by_level[level][cluster_id] = []

        communities_by_level[level][cluster_id].append(partition.node)

    return communities_by_level

# Result:
# Level 0 (fine-grained):
#   Cluster 0: ['ALICE', 'BOB', 'CAROL']  # Team A
#   Cluster 1: ['DAVE', 'EVE']            # Team B
#   Cluster 2: ['FRANK', 'GRACE', 'HEIDI'] # Team C
#
# Level 1 (coarse-grained):
#   Cluster 0: ['ALICE', 'BOB', 'CAROL', 'DAVE', 'EVE']  # Org 1
#   Cluster 1: ['FRANK', 'GRACE', 'HEIDI']                # Org 2
```

**3. Community Report Generation**:

```python
def generate_community_report(G, community_nodes):
    """Create summary report for a community."""
    subgraph = G.subgraph(community_nodes)

    report = {
        'title': f"Community {community_id}",
        'size': len(community_nodes),
        'entities': list(community_nodes),

        # Central entities (high PageRank within community)
        'key_entities': get_top_k_by_pagerank(subgraph, k=5),

        # Summary of community's focus
        'summary': generate_llm_summary(subgraph),

        # Inter-community connections
        'external_links': count_external_edges(G, community_nodes)
    }

    return report
```

**Code Locations**:
- `graphrag/index/operations/cluster_graph.py` - Main clustering logic
- `graphrag/index/utils/stable_lcc.py` - Largest connected component extraction

**Algorithm Details**:

**Leiden Algorithm**:
- **Goal**: Maximize modularity (Q) - measure of community structure quality
- **Method**: Iterative optimization with refinement phase
- **Modularity**: `Q = (edges within communities - expected edges) / total edges`
- **Hierarchical**: Recursively cluster communities to form hierarchy

**Why Hierarchical Leiden**:
- **Multi-scale**: Different query scopes need different granularity
  - Global query → Use coarse communities (level 2)
  - Local query → Use fine communities (level 0)
- **Quality**: Better than Louvain (resolves pathological cases)
- **Stability**: Produces consistent communities across runs

**Performance Characteristics**:
- **Speed**: O(N log N) for N nodes (near-linear)
- **Quality**: Q typically 0.5-0.9 (higher = better structure)
- **Scalability**: Handles 100K+ node graphs efficiently
- **Determinism**: Fixed seed → reproducible communities

---

## Vector Storage and Similarity Search

### lancedb

**Version**: `^0.17.0`

**Purpose**: Open-source vector database built on Apache Arrow and Lance format

**Role in GraphRAG**:
- **Vector storage**: Persist text unit and entity embeddings
- **Similarity search**: Fast k-NN search for query-to-entity matching
- **Scalability**: Handle millions of embeddings efficiently
- **Persistence**: Durable storage of embedding vectors

**Architecture Integration**:
```
Text Units → Embeddings (1536D) → LanceDB → Similarity Search → Relevant Entities
```

**Key Features Used**:

**1. Vector Table Creation**:
```python
import lancedb

# Connect to database
db = lancedb.connect("./lancedb")

# Create table with schema
table = db.create_table(
    "text_unit_embeddings",
    data=[
        {
            "id": "unit_001",
            "text": "GraphRAG is a knowledge extraction system...",
            "vector": [0.123, -0.456, ..., 0.789],  # 1536D
            "entity_ids": ["GRAPHRAG", "KNOWLEDGE_EXTRACTION"],
            "source_doc": "doc_1"
        },
        # ... more rows
    ]
)
```

**2. Similarity Search**:
```python
# Query: Find top-10 most similar text units
query_vector = await embed_text("How does GraphRAG work?")

results = (
    table.search(query_vector)
    .limit(10)
    .to_list()
)

# Results: [
#   {'id': 'unit_042', 'text': '...', '_distance': 0.12},  # Closest
#   {'id': 'unit_137', 'text': '...', '_distance': 0.18},
#   ...
# ]
```

**3. Filtered Search**:
```python
# Find similar text units from specific document
results = (
    table.search(query_vector)
    .where("source_doc = 'doc_1'")
    .limit(10)
    .to_list()
)
```

**4. Hybrid Search** (vector + metadata):
```python
# Find entities related to query, filter by entity type
results = (
    table.search(query_vector)
    .where("entity_type = 'PERSON'")
    .select(["id", "text", "entity_ids"])
    .limit(10)
    .to_list()
)
```

**Code Locations**:
- `graphrag/vector_stores/lancedb.py` - LanceDB adapter implementation
- `graphrag/query/context_builder/entity_extraction.py` - Uses vector search

**Why LanceDB**:
- **Embedded**: Runs in-process (no separate server)
- **Fast**: GPU-accelerated similarity search
- **Arrow-native**: Efficient columnar storage
- **Lance format**: Versioned, transactional vector storage
- **Open-source**: MIT licensed, no vendor lock-in

**Performance Characteristics**:
```
Benchmark (1M vectors, 1536D, k=10):
- Index build: ~2 minutes
- Query latency: <10ms (99th percentile)
- Throughput: ~10K queries/second
- Memory: ~6GB (for 1M x 1536D float32)
- Disk: ~6GB (Lance format compression)
```

**Alternatives in GraphRAG**:
- **Azure AI Search**: Cloud-based, scalable
- **Custom in-memory**: Simpler but not persistent

---

### azure-search-documents

**Version**: `^11.5.2`

**Purpose**: Azure Cognitive Search SDK for Python

**Role in GraphRAG**:
- **Cloud vector storage**: Alternative to LanceDB for production
- **Hybrid search**: Combine vector similarity + keyword search + filters
- **Scalability**: Auto-scaling for high query throughput
- **Enterprise features**: Security, compliance, SLA guarantees

**Architecture Integration**:
```
Embeddings → Azure AI Search Index → Hybrid Search (vector + keyword) → Results
```

**Key Features Used**:

**1. Index Creation**:
```python
from azure.search.documents.indexes import SearchIndexClient
from azure.search.documents.indexes.models import (
    SearchIndex,
    SearchField,
    SearchFieldDataType,
    VectorSearch,
    VectorSearchProfile
)

# Define index schema
index = SearchIndex(
    name="graphrag-text-units",
    fields=[
        SearchField(name="id", type=SearchFieldDataType.String, key=True),
        SearchField(name="text", type=SearchFieldDataType.String, searchable=True),
        SearchField(
            name="vector",
            type=SearchFieldDataType.Collection(SearchFieldDataType.Single),
            vector_search_dimensions=1536,
            vector_search_profile_name="default"
        ),
        SearchField(name="entity_ids", type=SearchFieldDataType.Collection(SearchFieldDataType.String)),
    ],
    vector_search=VectorSearch(
        profiles=[VectorSearchProfile(name="default", algorithm_configuration_name="hnsw")]
    )
)

# Create index
index_client.create_index(index)
```

**2. Vector Upload**:
```python
from azure.search.documents import SearchClient

search_client = SearchClient(endpoint=endpoint, index_name="graphrag-text-units", credential=credential)

# Batch upload embeddings
documents = [
    {
        "id": "unit_001",
        "text": "GraphRAG extracts knowledge...",
        "vector": embedding_vector,  # 1536D list
        "entity_ids": ["GRAPHRAG", "KNOWLEDGE"]
    },
    # ... up to 1000 docs per batch
]

search_client.upload_documents(documents)
```

**3. Hybrid Search**:
```python
# Combined vector similarity + keyword matching
results = search_client.search(
    search_text="knowledge extraction",  # Keyword search
    vector_queries=[{
        "vector": query_embedding,
        "k_nearest_neighbors": 10,
        "fields": "vector"
    }],
    select=["id", "text", "entity_ids"],
    top=10
)

for result in results:
    print(f"Score: {result['@search.score']}, Text: {result['text']}")
```

**4. Filtered Vector Search**:
```python
# Vector search with metadata filters
results = search_client.search(
    vector_queries=[{
        "vector": query_embedding,
        "k_nearest_neighbors": 10,
        "fields": "vector"
    }],
    filter="source_doc eq 'doc_1' and entity_type eq 'PERSON'",
    top=10
)
```

**Code Locations**:
- `graphrag/vector_stores/azure_ai_search.py` - Azure Search adapter

**Why Azure AI Search**:
- **Production-ready**: High availability, SLA guarantees
- **Hybrid search**: Better than pure vector search for many queries
- **Geo-replication**: Deploy globally for low latency
- **Security**: RBAC, encryption at rest/in transit, VNet integration

**Performance Characteristics**:
```
Azure AI Search (Standard S1 tier):
- Max documents: 15M
- Max index size: 25GB
- Query latency: ~50ms (p95)
- Throughput: ~1K QPS (scales with tier)
- Cost: ~$250/month (Standard S1)
```

**Trade-offs**:
- **LanceDB**: Free, embedded, local - Good for development/small deployments
- **Azure AI Search**: Paid, cloud, scalable - Good for production/large deployments

---

## Data Science Libraries (Graph-Related)

### umap-learn

**Version**: `^0.5.6`

**Purpose**: Uniform Manifold Approximation and Projection for dimension reduction

**Role in GraphRAG**:
- **Graph visualization**: Project high-D graph embeddings to 2D/3D for visualization
- **Cluster visualization**: Show community structure in 2D space
- **Quality assessment**: Visualize entity similarity patterns

**Architecture Integration**:
```
Graph Embeddings (128D) → UMAP → 2D Coordinates → Visualization
```

**Key Features Used**:

**1. Graph Embedding Reduction**:
```python
import umap

# Node embeddings from graph (e.g., Node2vec)
node_embeddings = np.array([
    [0.1, -0.3, ..., 0.5],  # ALICE (128D)
    [0.2, -0.2, ..., 0.4],  # BOB
    # ... all nodes
])

# Reduce to 2D for visualization
reducer = umap.UMAP(
    n_components=2,
    n_neighbors=15,
    min_dist=0.1,
    metric='cosine'
)

coords_2d = reducer.fit_transform(node_embeddings)

# Result: [[x1, y1], [x2, y2], ...] for plotting
```

**2. Community Visualization**:
```python
import matplotlib.pyplot as plt

# Color by community
colors = [community_id_for_node[node] for node in nodes]

plt.scatter(
    coords_2d[:, 0],
    coords_2d[:, 1],
    c=colors,
    cmap='tab20',
    alpha=0.6
)

# Add labels for key entities
for i, node in enumerate(nodes):
    if is_key_entity(node):
        plt.annotate(node, coords_2d[i])

plt.title("Entity Graph - Community Structure")
plt.show()
```

**Code Locations**:
- `graphrag/index/operations/layout_graph/umap.py` - UMAP-based graph layout
- `graphrag/index/operations/layout_graph/layout_graph.py` - Graph visualization pipeline

**Why UMAP**:
- **Preserves global structure**: Unlike t-SNE, maintains large-scale patterns
- **Fast**: Handles 100K+ points efficiently
- **Flexible**: Works with any distance metric (cosine, euclidean, etc.)
- **Parameter control**: Adjustable preservation of local vs global structure

**Parameters**:
- `n_neighbors=15`: How many neighbors define local structure (higher = more global)
- `min_dist=0.1`: Minimum distance between points (lower = tighter clusters)
- `metric='cosine'`: Use cosine distance for semantic embeddings

---

## Dependency Interaction Diagram

```
                    Entity Graph (networkx)
                            |
                            |
        +-------------------+-------------------+
        |                                       |
        v                                       v
graspologic.hierarchical_leiden           Graph Algorithms
        |                                  (centrality, paths)
        v                                       |
Hierarchical Communities                       |
        |                                       |
        v                                       |
 Community Reports                              |
        |                                       |
        +-------------------+-------------------+
                            |
                            v
                    Text Units + Entities
                            |
                            v
                Text Embeddings (OpenAI)
                     [1536D vectors]
                            |
            +---------------+---------------+
            |                               |
            v                               v
    LanceDB (local)              Azure AI Search (cloud)
            |                               |
            v                               v
    Similarity Search            Hybrid Search (vector + keyword)
            |                               |
            +---------------+---------------+
                            |
                            v
                    Query Results
                            |
                            v
        (Optional: UMAP visualization of results)
```

---

## Storage Patterns

### Vector Index Types

**1. Flat Index** (Brute Force):
```python
# No index, scan all vectors
# O(N) search time
# 100% recall (exact search)
# Use for: N < 10K vectors
```

**2. HNSW Index** (Hierarchical Navigable Small World):
```python
# Used by Azure AI Search and LanceDB
# O(log N) search time
# ~95-99% recall (approximate)
# Use for: N > 10K vectors

lancedb_table.create_index(
    metric="cosine",
    index_type="IVF_PQ",  # LanceDB's optimized index
    num_partitions=256,
    num_sub_vectors=96
)
```

**3. IVF (Inverted File Index)**:
```python
# Partition space into cells
# Search only nearest cells
# O(sqrt(N)) search time
# Use for: Very large datasets (1M+ vectors)
```

### Persistence Strategies

**Development**:
```python
# In-memory vector store (fast, no persistence)
vector_store = InMemoryVectorStore(embeddings)
```

**Production (Local)**:
```python
# LanceDB (persistent, local file system)
db = lancedb.connect("./data/lancedb")
table = db.create_table("embeddings", data=embeddings)
```

**Production (Cloud)**:
```python
# Azure AI Search (persistent, cloud, scalable)
search_client = SearchClient(
    endpoint=azure_endpoint,
    index_name="graphrag-embeddings",
    credential=credential
)
```

---

## Performance Optimization

### Graph Processing

**1. Largest Connected Component**:
```python
# Many graphs have small disconnected components
# Focus on main component for efficiency
from graphrag.index.utils.stable_lcc import stable_largest_connected_component

G = nx.Graph()
# ... populate graph ...

G_main = stable_largest_connected_component(G)
# Typically removes 5-10% of nodes (isolated entities)
# Speeds up clustering by 20-30%
```

**2. Community Size Constraints**:
```python
# Limit community size for manageable reports
hierarchical_leiden(
    G,
    max_cluster_size=10,  # Smaller = more communities, finer granularity
    random_seed=42
)

# Trade-off:
# - Small communities (size 5-10): Specific, many reports, expensive to process
# - Large communities (size 50-100): General, fewer reports, cheaper
```

**3. Graph Pruning**:
```python
# Remove low-weight edges (weak relationships)
def prune_graph(G, weight_threshold=1.0):
    edges_to_remove = [
        (u, v) for u, v, w in G.edges(data='weight')
        if w < weight_threshold
    ]
    G.remove_edges_from(edges_to_remove)
    return G

# Reduces graph size by 30-50%
# Speeds up clustering, improves community quality
```

### Vector Search

**1. Batch Embeddings**:
```python
# Good: Batch embed for efficiency
texts = [unit.text for unit in text_units]
embeddings = await embed_texts_batch(texts, batch_size=100)

# Bad: Individual embeds
embeddings = [await embed_text(text) for text in texts]
```

**2. Index Optimization**:
```python
# LanceDB: Optimize index for query pattern
table.create_index(
    metric="cosine",
    num_partitions=256,  # More partitions = faster search, slower build
    num_sub_vectors=96   # Compression (1536/96 = 16x)
)

# Trade-off:
# - More partitions: Faster queries, longer index build, more memory
# - Fewer partitions: Slower queries, faster index build, less memory
```

**3. Pre-filter Before Vector Search**:
```python
# Bad: Vector search entire dataset, then filter
results = table.search(query_vector).limit(1000).to_list()
filtered = [r for r in results if r['entity_type'] == 'PERSON']

# Good: Filter first, then vector search
results = (
    table.search(query_vector)
    .where("entity_type = 'PERSON'")  # Reduces search space
    .limit(10)
    .to_list()
)
```

---

## Troubleshooting

### Graph Processing Issues

**Issue 1: Memory Error During Clustering**
```
MemoryError: Unable to allocate array
```

**Solution**: Use largest connected component + pruning:
```python
G = stable_largest_connected_component(G)
G = prune_graph(G, weight_threshold=2.0)
# Reduces memory by 50-70%
```

**Issue 2: No Communities Detected**
```
Warning: All nodes in single community
```

**Solution**: Graph too sparse or disconnected:
```python
# Check connectivity
num_components = nx.number_connected_components(G)
if num_components > 1:
    # Focus on largest component
    G = stable_largest_connected_component(G)

# Check density
density = nx.density(G)
if density < 0.01:
    # Graph too sparse, lower edge weight threshold
    G = build_graph(entities, relationships, min_weight=0.5)
```

### Vector Search Issues

**Issue 1: Slow Queries**
```
Query latency: 5 seconds (expected <100ms)
```

**Solution**: Create index:
```python
# LanceDB: Build ANN index
table.create_index(metric="cosine")

# Azure AI Search: Use higher tier
# Standard S1 → Standard S2 (2x throughput)
```

**Issue 2: Low Recall**
```
Expected entities not in top-10 results
```

**Solution**: Check embedding quality:
```python
# Verify embedding model consistency
query_emb = await embed_text(query)
text_emb = await embed_text(text)

# Should use SAME model for both
assert query_emb.shape == text_emb.shape  # (1536,)

# Check similarity
sim = cosine_similarity(query_emb, text_emb)
# Should be > 0.5 for relevant matches
```

---

## Future Considerations

### Graph Scaling

**1. Distributed Graph Processing**:
- **GraphX** (Spark): For 10M+ node graphs
- **Dask-NetworkX**: Parallel processing on cluster
- **Neo4j**: Native graph database with Cypher queries

**2. Incremental Updates**:
- **Add entities**: Append to graph, recluster affected communities
- **Update relationships**: Adjust edge weights, propagate changes
- **Delete entities**: Remove node, rebalance communities

### Vector Search Scaling

**1. Sharding**:
```python
# Partition vectors by document/topic
shard_1 = lancedb.connect("./shard_1")  # Documents 1-10K
shard_2 = lancedb.connect("./shard_2")  # Documents 10K-20K

# Query all shards, merge results
results = merge_results([
    shard_1.search(query_vector).limit(10),
    shard_2.search(query_vector).limit(10)
])
```

**2. Quantization**:
```python
# Reduce precision: float32 → int8
# 4x smaller, 4x faster, ~1% recall loss
table.create_index(
    metric="cosine",
    index_type="IVF_PQ",
    num_sub_vectors=96,  # Product quantization
    nbits=8               # 8-bit quantization
)
```

---

## Conclusion

Graph processing and vector storage dependencies provide GraphRAG's **structural foundation**:

- **networkx**: Flexible graph representation and algorithms
- **graspologic**: Advanced community detection (hierarchical Leiden)
- **lancedb**: Embedded vector database for development/small deployments
- **azure-search-documents**: Cloud vector search for production/large deployments
- **umap-learn**: Dimensionality reduction for visualization

Together, these libraries enable GraphRAG to:
1. **Organize knowledge** as interconnected entity graphs
2. **Discover structure** through hierarchical community detection
3. **Navigate semantics** via vector similarity search
4. **Scale efficiently** from thousands to millions of entities

**Key Takeaway**: The graph layer provides semantic structure, while the vector layer provides semantic search. Their integration—entities with embeddings in a clustered graph—creates a powerful knowledge representation that supports both exploratory (global) and targeted (local) queries.
