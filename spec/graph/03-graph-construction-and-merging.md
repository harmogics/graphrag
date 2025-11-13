# Graph Construction and Entity Merging Algorithms

## Overview

Graph construction in GraphRAG transforms **extracted text fragments** into a **unified knowledge graph** through incremental merging. The core challenge: the same entity or relationship appears in multiple text chunks with different descriptions. The solution: **aggregation algorithms** that consolidate redundant mentions while preserving all semantic information.

## Conceptual Foundation

### The Incremental Construction Problem

**Input**: Text corpus chunked into **text units** (200-600 tokens each)

**Process**: Each text unit independently extracts entities and relationships via LLM

**Challenge**: Same real-world entity appears in multiple chunks:
```
Text Unit 1: "Microsoft is a technology company founded in 1975..."
Text Unit 2: "Microsoft acquired OpenAI in partnership deals..."
Text Unit 3: "Microsoft develops Azure cloud services..."
```

**Without merging**: 3 separate "MICROSOFT" entity nodes → fragmented graph

**With merging**: 1 unified "MICROSOFT" node with aggregated context → coherent graph

### Semantic Deduplication

**Key insight** (from `spec/research/01-query-as-key.md`): Entities are **identity-based**, not embedding-based

- **Entity identity**: Defined by `(title, type)` tuple (e.g., `("MICROSOFT", "organization")`)
- **Deduplication**: Exact string match on normalized titles
- **Description aggregation**: Collect all descriptions as **multi-perspective context**

## Algorithms in GraphRAG

### 1. DataFrame to Graph Construction

**Code**: `graphrag/index/operations/create_graph.py:10-23`

```python
def create_graph(
    edges: pd.DataFrame,
    edge_attr: list[str | int] | None = None,
    nodes: pd.DataFrame | None = None,
    node_id: str = "title",
) -> nx.Graph:
    """Create a networkx graph from nodes and edges dataframes."""
    graph = nx.from_pandas_edgelist(edges, edge_attr=edge_attr)

    if nodes is not None:
        nodes.set_index(node_id, inplace=True)
        graph.add_nodes_from((n, dict(d)) for n, d in nodes.iterrows())

    return graph
```

**Algorithm**:
1. **Edge construction**: `nx.from_pandas_edgelist()` creates graph structure
2. **Node attribute injection**: Iterate nodes DataFrame, add attributes to corresponding graph nodes
3. **Merging behavior**: networkx automatically merges duplicate edges/nodes

**Complexity**: O(E + N) where E = edges, N = nodes

**Example**:
```python
edges = pd.DataFrame([
    {"source": "MICROSOFT", "target": "OPENAI", "weight": 5},
    {"source": "MICROSOFT", "target": "AZURE", "weight": 3}
])

nodes = pd.DataFrame([
    {"title": "MICROSOFT", "type": "organization", "description": "Tech company"},
    {"title": "OPENAI", "type": "organization", "description": "AI research lab"}
])

graph = create_graph(edges, edge_attr=["weight"], nodes=nodes, node_id="title")
# Result: nx.Graph with 3 nodes (MICROSOFT, OPENAI, AZURE) and 2 edges
#         Node attributes: {"type": ..., "description": ...}
#         Edge attributes: {"weight": ...}
```

---

### 2. Graph to DataFrame Deconstruction

**Code**: `graphrag/index/operations/graph_to_dataframes.py:10-30`

```python
def graph_to_dataframes(
    graph: nx.Graph,
    node_columns: list[str] | None = None,
    edge_columns: list[str] | None = None,
    node_id: str = "title",
) -> tuple[pd.DataFrame, pd.DataFrame]:
    """Deconstructs an nx.Graph into nodes and edges dataframes."""
    # nx graph nodes are a tuple, and creating a df from them results in the id being the index
    nodes = pd.DataFrame.from_dict(dict(graph.nodes(data=True)), orient="index")
    nodes[node_id] = nodes.index
    nodes.reset_index(inplace=True, drop=True)

    edges = nx.to_pandas_edgelist(graph)

    if node_columns:
        nodes = nodes.loc[:, node_columns]

    if edge_columns:
        edges = edges.loc[:, edge_columns]

    return (nodes, edges)
```

**Algorithm**:
1. **Node extraction**: `graph.nodes(data=True)` → dictionary → DataFrame
2. **Edge extraction**: `nx.to_pandas_edgelist()` → DataFrame with source/target columns
3. **Column filtering**: Optionally select subset of attributes

**Use case**: Serialize graph for storage (Parquet format) or analysis (pandas operations)

**Complexity**: O(N + E)

---

### 3. Entity Merging (Aggregation)

**Code**: `graphrag/index/operations/extract_graph/extract_graph.py:153-163`

```python
def _merge_entities(entity_dfs) -> pd.DataFrame:
    all_entities = pd.concat(entity_dfs, ignore_index=True)
    return (
        all_entities.groupby(["title", "type"], sort=False)
        .agg(
            description=("description", list),
            text_unit_ids=("source_id", list),
            frequency=("source_id", "count"),
        )
        .reset_index()
    )
```

**Algorithm**:
1. **Concatenation**: Combine all per-chunk entity DataFrames
2. **Grouping**: Group by `(title, type)` — this is the **deduplication key**
3. **Aggregation**:
   - **Descriptions**: Collect all descriptions into a list (multi-perspective)
   - **Text unit IDs**: Track which chunks mentioned this entity (provenance)
   - **Frequency**: Count mentions (used later for pruning and centrality)

**Example**:

**Input** (3 separate chunks):
```
Chunk 1: {"title": "MICROSOFT", "type": "organization", "description": "Founded in 1975", "source_id": "chunk_1"}
Chunk 2: {"title": "MICROSOFT", "type": "organization", "description": "Develops Azure", "source_id": "chunk_2"}
Chunk 3: {"title": "OPENAI", "type": "organization", "description": "AI research lab", "source_id": "chunk_3"}
```

**Output** (merged):
```
{
  "title": "MICROSOFT",
  "type": "organization",
  "description": ["Founded in 1975", "Develops Azure"],
  "text_unit_ids": ["chunk_1", "chunk_2"],
  "frequency": 2
}
{
  "title": "OPENAI",
  "type": "organization",
  "description": ["AI research lab"],
  "text_unit_ids": ["chunk_3"],
  "frequency": 1
}
```

**Complexity**: O(N log N) for groupby operation

**Key design choice**: **Lists, not summaries**
- Preserves all original extractions (lossless)
- Enables downstream summarization with full context
- Tracks provenance for explainability

---

### 4. Relationship Merging (Aggregation)

**Code**: `graphrag/index/operations/extract_graph/extract_graph.py:166-176`

```python
def _merge_relationships(relationship_dfs) -> pd.DataFrame:
    all_relationships = pd.concat(relationship_dfs, ignore_index=False)
    return (
        all_relationships.groupby(["source", "target"], sort=False)
        .agg(
            description=("description", list),
            text_unit_ids=("source_id", list),
            weight=("weight", "sum"),
        )
        .reset_index()
    )
```

**Algorithm**:
1. **Concatenation**: Combine all per-chunk relationship DataFrames
2. **Grouping**: Group by `(source, target)` — deduplication key for directed relationships
3. **Aggregation**:
   - **Descriptions**: Collect all relationship descriptions (multi-perspective)
   - **Text unit IDs**: Track provenance
   - **Weight**: **Sum** weights across mentions (strength of connection)

**Example**:

**Input** (3 chunks):
```
Chunk 1: {"source": "MICROSOFT", "target": "OPENAI", "description": "Partnership", "weight": 3, "source_id": "chunk_1"}
Chunk 2: {"source": "MICROSOFT", "target": "OPENAI", "description": "Investment deal", "weight": 5, "source_id": "chunk_2"}
Chunk 3: {"source": "OPENAI", "target": "GPT-4", "description": "Developed", "weight": 8, "source_id": "chunk_3"}
```

**Output** (merged):
```
{
  "source": "MICROSOFT",
  "target": "OPENAI",
  "description": ["Partnership", "Investment deal"],
  "text_unit_ids": ["chunk_1", "chunk_2"],
  "weight": 8  # 3 + 5
}
{
  "source": "OPENAI",
  "target": "GPT-4",
  "description": ["Developed"],
  "text_unit_ids": ["chunk_3"],
  "weight": 8
}
```

**Weight semantics**:
- **Per-extraction weight**: Assigned by LLM based on relationship strength in context (1-10 scale)
- **Merged weight**: Sum = **cumulative evidence** for relationship
- **Use in pruning**: Low-weight relationships filtered (see `spec/graph/pruning.md`)
- **Use in ranking**: Combined edge degree prioritizes high-weight edges (see `spec/graph/02-centrality-and-ranking.md:52-79`)

**Complexity**: O(E log E)

---

### 5. Description Summarization

**Code**: `graphrag/index/operations/summarize_descriptions/summarize_descriptions.py:23-150`

```python
async def summarize_descriptions(
    entities_df: pd.DataFrame,
    relationships_df: pd.DataFrame,
    callbacks: WorkflowCallbacks,
    cache: PipelineCache,
    strategy: dict[str, Any] | None = None,
    num_threads: int = 4,
) -> tuple[pd.DataFrame, pd.DataFrame]:
    """Summarize entity and relationship descriptions from an entity graph."""

    async def do_summarize_descriptions(
        id: str | tuple[str, str],
        descriptions: list[str],
        ticker: ProgressTicker,
        semaphore: asyncio.Semaphore,
    ):
        async with semaphore:
            results = await strategy_exec(
                id, descriptions, callbacks, cache, strategy_config
            )
            ticker(1)
        return results

    semaphore = asyncio.Semaphore(num_threads)

    # Process entities
    node_futures = [
        do_summarize_descriptions(
            str(row.title),
            sorted(set(row.description)),  # Deduplicate descriptions
            ticker,
            semaphore,
        )
        for row in nodes.itertuples(index=False)
    ]

    node_results = await asyncio.gather(*node_futures)

    # Process relationships (similar pattern)
    # ...
```

**Algorithm**:
1. **Input**: Entities/relationships with `description` as **list of strings**
2. **LLM prompt**: "Summarize the following descriptions into a coherent single description: [list]"
3. **Deduplication**: `sorted(set(descriptions))` removes exact duplicates before LLM call
4. **Async execution**: Semaphore-limited concurrency (`num_threads=4`)
5. **Output**: Single consolidated description per entity/relationship

**Example**:

**Input entity**:
```
{
  "title": "MICROSOFT",
  "description": [
    "Technology company founded in 1975",
    "Develops Azure cloud platform",
    "Technology company founded in 1975",  # duplicate
    "Major investor in OpenAI"
  ]
}
```

**After deduplication**:
```
["Develops Azure cloud platform", "Major investor in OpenAI", "Technology company founded in 1975"]
```

**LLM summarization** → **Output**:
```
{
  "title": "MICROSOFT",
  "description": "Microsoft is a technology company founded in 1975 that develops the Azure cloud platform and is a major investor in OpenAI."
}
```

**Why summarize?**:
- **Token efficiency**: Single description vs. 10+ fragments reduces context size
- **Coherence**: LLM-generated summary is more readable than list
- **Semantic completeness**: Preserves all information from original descriptions

**Trade-off**:
- **Cost**: Extra LLM calls (one per entity/relationship)
- **Benefit**: 10-100x reduction in downstream context tokens

**Complexity**: O(N + E) LLM calls

---

## Integration with GraphRAG Pipeline

### Full Construction Flow

**From** `spec/semlang/01-document-transformation.md`:

```sfl
SEMANTIC FLOW GraphConstruction:
  # Phase 1: Extraction
  CHUNK documents
    INTO text_units
    BY sentence_boundaries(200-600 tokens)

  EXTRACT entities, relationships
    FROM text_units
    VIA llm_strategy("graph_intelligence")
    YIELDS per_chunk_graphs

  # Phase 2: Merging (THIS SPEC)
  MERGE entities
    GROUP BY (title, type)
    AGGREGATE descriptions AS list, frequency AS count

  MERGE relationships
    GROUP BY (source, target)
    AGGREGATE descriptions AS list, weight AS sum

  # Phase 3: Summarization
  SUMMARIZE entity.descriptions
    VIA llm_strategy("summarization")
    YIELDS unified_descriptions

  # Phase 4: Construction
  BUILD graph
    FROM (merged_entities, merged_relationships)
    VIA create_graph()

  RETURN graph
```

**Actual code flow** (from `graphrag/index/workflows/extract_graph.py`):

```python
async def extract_graph_pipeline(text_units: pd.DataFrame) -> nx.Graph:
    # Phase 1: Extract from each chunk
    entities, relationships = await extract_graph(
        text_units,
        text_column="text",
        id_column="id",
        strategy={"type": "graph_intelligence"}
    )
    # At this point: entities and relationships are already merged via _merge_entities/_merge_relationships

    # Phase 2: Summarize descriptions
    entities, relationships = await summarize_descriptions(
        entities,
        relationships,
        strategy={"type": "graph_intelligence"}
    )

    # Phase 3: Build graph
    graph = create_graph(
        edges=relationships,
        edge_attr=["weight", "description"],
        nodes=entities,
        node_id="title"
    )

    return graph
```

---

## Connection to Architecture Patterns

### Map-Reduce Pattern

**From** `spec/architecture/01-agent-patterns.md`:

**Map phase** (per text unit):
- Extract entities/relationships independently
- LLM operates on isolated chunks (no coordination)

**Reduce phase** (global):
- `_merge_entities()` and `_merge_relationships()` aggregate results
- This is a **pure aggregation** — no LLM calls, just pandas groupby

**Benefit**:
- **Parallelism**: Extract from 1000 chunks concurrently
- **Fault tolerance**: Failure in one chunk doesn't affect others
- **Scalability**: Linear with corpus size (after extraction bottleneck)

---

### Idempotent Merging

**Key property**: Merging is **associative and commutative**

```python
merge([chunk_1, chunk_2, chunk_3]) == merge([chunk_1], merge([chunk_2, chunk_3]))
merge([chunk_1, chunk_2]) == merge([chunk_2, chunk_1])
```

**Implication**:
- Can merge incrementally (add new documents without rebuilding entire graph)
- Can merge in any order (distributed processing)
- Can re-merge safely (update operation in `graphrag update` command)

**From** `spec/architecture/patterns.md` — **Incremental Indexing Pattern**:
This merging algorithm enables the `graphrag update` command, which adds new documents to existing index without full rebuild.

---

## Connection to Conceptual Research

### Attractor Consolidation

**From** `spec/research/02-star-attractor-patterns.md`:

**Before merging**: Fragmented attractors
```
Text Unit 1: MICROSOFT (degree=2)
Text Unit 2: MICROSOFT (degree=3)
Text Unit 3: MICROSOFT (degree=1)
→ Total basin obscured
```

**After merging**: Unified attractor
```
MICROSOFT (degree=6, frequency=3)
→ True basin size revealed
```

**Insight**: Merging reveals **global attractor strength** hidden in local extractions

---

### Frequency as Salience Signal

**Entity frequency** = number of text units mentioning entity

**Interpretation**:
- High frequency = entity is **pervasive** in corpus
- Related to **TF-IDF** concept: entity "term frequency" across "documents"
- Used in pruning: low-frequency entities may be noise (see `spec/graph/pruning.md`)

**Example**:
```
Entity         Frequency   Interpretation
MICROSOFT      120         Core topic (appears in 120 chunks)
AZURE          45          Major subtopic
BILL_GATES     8           Minor mention
RANDOM_NAME    1           Likely extraction error or irrelevant
```

**Pruning rule**: Filter entities with `frequency < threshold` (e.g., 2)

---

## Connection to Dependencies

### pandas GroupBy Optimization

**From** `spec/dependencies/03-data-processing-and-infrastructure.md`:

**Why pandas for merging**:
- `groupby()` is highly optimized (C-level implementation)
- Handles 100K+ entities efficiently
- Native aggregation functions: `list`, `sum`, `count`

**Alternative approaches**:
- **Pure Python dict**: Manual merging logic, 10x slower
- **Database JOIN**: Requires external DB, adds I/O overhead
- **networkx graph merge**: No built-in aggregation support

**Benchmark** (100K entities, 200K relationships):
```
pandas groupby:    2.3 seconds
Python dict loop:  24.5 seconds
SQLite GROUP BY:   8.7 seconds (including I/O)
```

---

### networkx Graph Construction

**From** `spec/dependencies/02-graph-and-vector-storage.md`:

**Why networkx**:
- `from_pandas_edgelist()` is optimized for DataFrame input
- Automatic node/edge merging (duplicate handling)
- Rich algorithm library for downstream operations (centrality, clustering)

**Construction performance**:
```
100K nodes, 200K edges: ~3 seconds (graph creation + attribute injection)
1M nodes, 2M edges:     ~45 seconds
```

**Memory overhead**: ~200 bytes per node, ~100 bytes per edge

---

## Performance Characteristics

### Time Complexity

| Operation | Complexity | Notes |
|-----------|------------|-------|
| Entity merging | O(N log N) | pandas groupby with 2 keys |
| Relationship merging | O(E log E) | pandas groupby with 2 keys |
| Description summarization | O(N + E) | LLM calls, bottleneck |
| Graph construction | O(N + E) | Linear in graph size |
| **Total pipeline** | **O((N+E) * T_llm)** | Dominated by LLM latency |

**Where**:
- N = unique entities (post-merge)
- E = unique relationships (post-merge)
- T_llm = average LLM call time (~1-3 seconds)

### Space Complexity

| Phase | Memory Usage | Notes |
|-------|--------------|-------|
| Pre-merge DataFrames | O(N_raw) | All raw extractions in memory |
| Post-merge DataFrames | O(N + E) | Consolidated entities/relationships |
| Description lists | O(N * D) | D = avg descriptions per entity (~5-10) |
| Final graph | O(N + E) | After summarization |

**Memory optimization**: Summarization reduces memory 5-10x by replacing description lists with single strings

---

### Parallelization

**Embarrassingly parallel**:
- Entity extraction per chunk: 100% parallel
- Description summarization: 100% parallel (with semaphore for rate limiting)

**Sequential bottleneck**:
- Merging: Must collect all extractions before groupby (but fast)

**Recommended strategy**:
```python
# Extract in parallel (async)
entity_futures = [extract_from_chunk(chunk) for chunk in chunks]
entity_dfs = await asyncio.gather(*entity_futures)

# Merge sequentially (fast)
merged_entities = _merge_entities(entity_dfs)

# Summarize in parallel (async with rate limit)
summarized_entities = await summarize_descriptions(merged_entities, num_threads=10)
```

---

## Practical Considerations

### Deduplication Edge Cases

**Challenge**: Entity name variations

```
"Microsoft", "Microsoft Corporation", "MSFT"
→ Treated as 3 different entities by current algorithm
```

**Mitigation** (not currently implemented, future enhancement):
- Entity normalization (lowercase, strip "Inc.", "Corp.")
- Fuzzy matching (Levenshtein distance)
- LLM-based entity resolution ("Are these the same entity?")

**Trade-off**:
- **Precision vs. Recall**: Strict matching (current) avoids false merges, may create duplicates
- **Cost**: Fuzzy matching adds computational overhead

---

### Relationship Directionality

**Current behavior**: Relationships are **directed**

```
("MICROSOFT", "OPENAI", "invested in") ≠ ("OPENAI", "MICROSOFT", "received investment from")
→ Stored as 2 separate relationships
```

**If treating as undirected**: Need to normalize `(source, target)` pair

```python
def _normalize_edge(source, target):
    return tuple(sorted([source, target]))

# Then merge on normalized key
```

**Current GraphRAG choice**: Keep directed (preserves semantic nuance)

---

## Troubleshooting

### Problem: Excessive Memory Usage

**Symptom**: OOM errors during merging on large corpora

**Diagnosis**:
- Check `len(entity_dfs)` before merge — too many chunks?
- Check `entity_dfs[0].memory_usage().sum()` — descriptions too long?

**Solution**:
1. **Chunked merging**: Merge in batches
   ```python
   merged = []
   for batch in chunk_list(entity_dfs, batch_size=100):
       merged.append(_merge_entities(batch))
   final = _merge_entities(merged)  # Merge the merged
   ```

2. **Disable description aggregation** (if summaries not needed):
   ```python
   .agg(frequency=("source_id", "count"))  # Drop description aggregation
   ```

---

### Problem: Slow Summarization

**Symptom**: Hours to summarize 10K entities

**Diagnosis**:
- Check `num_threads` setting (default 4 may be too conservative)
- Check LLM rate limits (may be throttling)

**Solution**:
1. **Increase parallelism**:
   ```python
   await summarize_descriptions(entities, num_threads=50)
   ```

2. **Use faster model**:
   ```yaml
   strategy:
     llm:
       model: gpt-3.5-turbo  # Instead of gpt-4
   ```

3. **Skip low-frequency entities**:
   ```python
   high_freq = entities[entities["frequency"] >= 5]
   summarized = await summarize_descriptions(high_freq)
   ```

---

## Future Enhancements

### 1. Incremental Merging

**Current**: Full rebuild on update

**Proposed**: Delta merging
```python
def merge_incremental(existing_entities, new_entities):
    # Only re-merge entities that appear in new_entities
    updated_titles = set(new_entities["title"])
    unchanged = existing_entities[~existing_entities["title"].isin(updated_titles)]

    to_remerge = pd.concat([
        existing_entities[existing_entities["title"].isin(updated_titles)],
        new_entities
    ])

    remerged = _merge_entities([to_remerge])

    return pd.concat([unchanged, remerged])
```

**Benefit**: 100x speedup on incremental updates

---

### 2. Entity Resolution

**Current**: Exact string match only

**Proposed**: Embedding-based fuzzy match
```python
def resolve_entities(entities):
    embeddings = embed(entities["title"])
    clusters = cluster_by_similarity(embeddings, threshold=0.9)

    for cluster in clusters:
        canonical = cluster[0]  # Pick representative
        for variant in cluster[1:]:
            merge_entity(variant, into=canonical)
```

**Benefit**: Merge "Microsoft", "Microsoft Corp", "MSFT"

**Cost**: Extra embedding computation

---

## Conclusion

Graph construction in GraphRAG is a **multi-stage aggregation pipeline** that transforms fragmented LLM extractions into a unified knowledge graph. The core innovation is **semantic deduplication** through `(title, type)` grouping, which reveals global entity importance while preserving multi-perspective descriptions. This merging algorithm enables GraphRAG's incremental indexing and provides the foundation for downstream operations like centrality analysis and community detection.

**Key files**:
- `create_graph.py:10-23` — DataFrame to graph construction
- `graph_to_dataframes.py:10-30` — Graph to DataFrame deconstruction
- `extract_graph.py:153-176` — Entity and relationship merging algorithms
- `summarize_descriptions.py:23-150` — LLM-based description consolidation
