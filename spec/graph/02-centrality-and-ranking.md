# Centrality and Entity Ranking Algorithms

## Overview

Centrality algorithms identify **important entities** in the knowledge graph by measuring different aspects of structural influence. In GraphRAG, centrality guides which entities to prioritize in community reports, context building, and answer generation.

## Conceptual Foundation

### What is Centrality?

**Definition**: A measure of an entity's **structural importance** in the graph

**Semantic interpretation**: High centrality = entity is a **hub** or **bridge** in semantic space, connecting many concepts or topics

**Attractor interpretation** (from `spec/research/02-star-attractor-patterns.md`):
High-centrality entities are **strong attractors** with large basins—they influence and connect to many other entities

## Algorithms in GraphRAG

### 1. Degree Centrality

**Code**: `graphrag/index/operations/compute_degree.py`

```python
def compute_degree(graph: nx.Graph) -> pd.DataFrame:
    """Compute degree for each node."""
    return pd.DataFrame([
        {"title": node, "degree": int(degree)}
        for node, degree in graph.degree
    ])
```

**Formula**: `C_D(v) = degree(v) = number of edges incident to v`

**Interpretation**:
- **High degree**: Entity appears in many relationships
- **Physical analogy**: "Popular" node with many connections
- **Use in GraphRAG**: Filter entities for context (top-k by degree)

**Example**:
```
Entity          Degree
"MICROSOFT"     45      (appears in 45 relationships)
"AI"            38      (mentioned with 38 different entities)
"OPENAI"        25
```

**Complexity**: O(N) where N = nodes

---

### 2. Combined Edge Degree

**Code**: `graphrag/index/operations/compute_edge_combined_degree.py`

```python
def compute_edge_combined_degree(edge_df, node_degree_df, ...):
    """Compute combined degree for each edge."""
    # For edge (u, v): combined_degree = degree(u) + degree(v)
    output_df["combined_degree"] = (
        output_df[source_degree] + output_df[target_degree]
    )
    return output_df["combined_degree"]
```

**Formula**: `C_E(u,v) = degree(u) + degree(v)`

**Interpretation**:
- Measures "importance" of an edge by importance of its endpoints
- High combined degree = edge connects two central entities

**Use in GraphRAG**: Prioritize relationships in community reports

**Example**:
```
Relationship                    Combined Degree
("MICROSOFT", "OPENAI")         45 + 25 = 70
("GPT-4", "CHATGPT")            30 + 28 = 58
```

---

### 3. PageRank (Implicit)

**Not explicitly computed in current code, but used conceptually**

**Algorithm** (from networkx):
```python
pagerank = nx.pagerank(graph, alpha=0.85)
# Returns: {node: score, ...}
```

**Formula** (iterative):
```
PR(v) = (1-d)/N + d * Σ(PR(u) / out_degree(u))
        for all u linking to v

Where:
- d = damping factor (0.85 typical)
- N = total nodes
```

**Interpretation**:
- **Random surfer model**: Probability of landing on node v by random walk
- **High PageRank**: Entity is "authoritative"—linked by other important entities
- **Recursive importance**: Importance flows through graph structure

**Difference from degree**:
- **Degree**: Raw connection count
- **PageRank**: Weighted by importance of neighbors

**Example**:
```
Entity          Degree   PageRank
"MICROSOFT"     45       0.025     (many connections)
"AI"            38       0.035     (fewer connections but to important entities)
"OPENAI"        25       0.018
```

---

## Integration with GraphRAG

### Community Report Generation

**Conceptual flow** (from `spec/architecture/01-agent-patterns.md`):

```python
def generate_community_report(community, graph):
    # Step 1: Extract community subgraph
    subgraph = graph.subgraph(community.nodes)

    # Step 2: Compute centrality within community
    degrees = {node: deg for node, deg in subgraph.degree()}
    pageranks = nx.pagerank(subgraph)

    # Step 3: Identify key entities (top-k by centrality)
    top_entities = sorted(
        pageranks.items(),
        key=lambda x: x[1],
        reverse=True
    )[:5]

    # Step 4: Build LLM prompt with key entities emphasized
    prompt = f"""
    Community: {community.id}
    Key entities: {[e for e, _ in top_entities]}
    All entities: {community.nodes}
    Relationships: {list(subgraph.edges())}

    Generate summary emphasizing the key entities.
    """

    return await llm.achat(prompt)
```

**Why centrality matters**:
- **Focus**: LLM attention on structurally important entities
- **Coherence**: Reports center around hubs/bridges
- **Efficiency**: Mention top-k entities explicitly, others implicitly

---

### Context Building for Queries

**From** `spec/semlang/03-query-semantics.md`:

```sfl
SEMANTIC FLOW EntityRanking:
  LOCATE relevant_entities
    VIA vector_similarity(query, entity_embeddings)

  ORGANIZE entities
    BY combined_metric:
      - similarity_score * 0.6
      - degree_centrality * 0.3
      - pagerank * 0.1

  SELECT top_k entities
    WHERE k = context_budget / avg_entity_size
```

**Implementation** (conceptual):

```python
def build_query_context(query, entities, graph, k=10):
    # Similarity scores from vector search
    similarities = vector_search(query, entities)

    # Centrality scores
    degrees = compute_degree(graph)
    degree_map = {e.title: e.degree for e in degrees}

    # Combined scoring
    scored_entities = []
    for entity, sim in similarities.items():
        degree = degree_map.get(entity, 0)
        score = 0.6 * sim + 0.4 * (degree / max_degree)
        scored_entities.append((entity, score))

    # Return top-k
    return sorted(scored_entities, key=lambda x: x[1], reverse=True)[:k]
```

**Trade-off**:
- **Pure similarity**: May retrieve niche, low-connectivity entities
- **Pure centrality**: May retrieve important but irrelevant entities
- **Hybrid**: Balance relevance (similarity) and importance (centrality)

---

## Connection to Research Patterns

### Attractor Strength

**From** `spec/research/02-star-attractor-patterns.md`:

Centrality measures **attractor strength**:

- **Degree centrality** = Local attractor strength (direct connections)
- **PageRank** = Global attractor strength (indirect influence)
- **Betweenness centrality** = Bridge attractor (controls information flow)

**Example**: Community with star topology

```
        ALICE (degree=4, PageRank=0.40)
       /  |  |  \
      /   |  |   \
   BOB   CAR  DAN  EVE
  (deg=1, PR=0.15 each)
```

ALICE is central attractor—high degree and PageRank. Others orbit around ALICE.

---

## Performance Characteristics

| Algorithm | Time | Space | Use Case |
|-----------|------|-------|----------|
| Degree | O(N) | O(N) | Fast filtering, initial ranking |
| Combined Edge Degree | O(E) | O(E) | Relationship prioritization |
| PageRank | O(N+E) iterations | O(N) | Authoritative entity identification |
| Betweenness | O(N²E) | O(N²) | Bridge detection (expensive) |

**Recommendation**: Use degree for real-time, PageRank for batch processing

---

## Future Enhancements

### Query-Personalized PageRank

```python
def personalized_pagerank(graph, query_entities, alpha=0.85):
    """PageRank biased toward query-relevant entities."""
    personalization = {
        entity: 1.0 if entity in query_entities else 0.0
        for entity in graph.nodes()
    }

    return nx.pagerank(graph, alpha=alpha, personalization=personalization)
```

**Benefit**: Entities important **in context of query**, not globally

---

## Conclusion

Centrality algorithms provide the **structural lens** for understanding entity importance. Combined with semantic similarity (vector search), they enable GraphRAG to identify not just *relevant* entities, but *important and relevant* entities—the key to high-quality answers.
