# Implementation Mapping — API to Code Components

## Обзор

Данный документ предоставляет детальный mapping между предложенными REST API endpoints и существующими компонентами кода GraphRAG, а также описывает необходимые изменения для реализации.

---

## 1. Introspection API Mapping

### POST /api/v1/search (with introspection)

**Existing Components**:
- `graphrag/query/structured_search/local_search/search.py:LocalSearch.search()`
- `graphrag/query/structured_search/global_search/search.py:GlobalSearch.search()`
- `graphrag/query/structured_search/drift_search/search.py:DRIFTSearch.search()`

**Modifications Needed**:

```python
# CURRENT CODE (graphrag/query/structured_search/local_search/search.py:58-125)
async def search(
    self,
    query: str,
    conversation_history: ConversationHistory | None = None,
    **kwargs,
) -> SearchResult:
    start_time = time.time()
    context_result = self.context_builder.build_context(
        query=query,
        conversation_history=conversation_history,
        **kwargs,
        **self.context_builder_params,
    )
    # ... LLM call ...
    return SearchResult(
        response=full_response,
        context_data=context_result.context_records,
        # ...
    )
```

**PROPOSED MODIFICATIONS**:

```python
# NEW CODE with introspection support
async def search(
    self,
    query: str,
    conversation_history: ConversationHistory | None = None,
    introspection: IntrospectionLevel = IntrospectionLevel.NONE,
    **kwargs,
) -> SearchResult:
    start_time = time.time()

    # NEW: Initialize trace if introspection enabled
    trace = None
    if introspection != IntrospectionLevel.NONE:
        from graphrag.query.introspection.trace import Trace
        trace = Trace(
            query=query,
            search_type="local",
            introspection_level=introspection
        )
        trace.start_stage("query_analysis")

    # NEW: Query embedding with tracing
    if trace and introspection.value >= IntrospectionLevel.FULL.value:
        query_embedding = await self._embed_query_with_trace(query, trace)
        detected_entities = await self._extract_entities_with_trace(query, trace)
        trace.end_stage(outputs={
            "query_embedding": query_embedding,
            "detected_entities": detected_entities
        })

        trace.start_stage("context_building")

    # EXISTING: Context building (now with trace parameter)
    context_result = self.context_builder.build_context(
        query=query,
        conversation_history=conversation_history,
        trace=trace,  # NEW parameter
        **kwargs,
        **self.context_builder_params,
    )

    # NEW: End context building stage
    if trace:
        trace.end_stage(outputs={
            "entities_selected": len(context_result.context_records.get("entities", [])),
            "relationships_selected": len(context_result.context_records.get("relationships", [])),
            "context_tokens": context_result.context_tokens
        })
        trace.start_stage("llm_reasoning")

    # EXISTING: LLM call
    search_prompt = self.system_prompt.format(...)
    async for response in self.model.achat_stream(...):
        full_response += response

    # NEW: Record LLM stage
    if trace:
        trace.record_llm_call(
            model=self.model.model_name,
            system_prompt=search_prompt,
            user_prompt=query,
            response=full_response,
            token_usage={...}
        )
        trace.end_stage(outputs={"answer": full_response})

    # EXISTING: Return result
    result = SearchResult(
        response=full_response,
        context_data=context_result.context_records,
        context_text=context_result.context_chunks,
        completion_time=time.time() - start_time,
        # NEW fields
        introspection=trace.serialize() if trace else None
    )

    # NEW: Store trace for later retrieval
    if trace:
        from graphrag.query.introspection.trace_store import TraceStore
        trace_store = TraceStore()
        await trace_store.save(trace)

    return result
```

**New Files to Create**:
1. `graphrag/query/introspection/__init__.py`
2. `graphrag/query/introspection/trace.py` — Trace class
3. `graphrag/query/introspection/trace_store.py` — TraceStore (Redis/memory backend)
4. `graphrag/query/introspection/levels.py` — IntrospectionLevel enum

**Estimated LOC**: ~500 lines

---

## 2. Graph API Mapping

### GET /api/v1/graph/entities

**Existing Components**:
- `graphrag/query/input/loaders/dfs.py:read_entities()`
- `graphrag/query/input/retrieval/entities.py:get_entity_by_id()`, `get_entity_by_key()`, `to_entity_dataframe()`
- `graphrag/data_model/entity.py:Entity`

**Implementation**:

```python
# graphrag/api/graph/entities.py (NEW FILE)

from graphrag.query.input.loaders.dfs import read_entities
from graphrag.query.input.retrieval.entities import (
    get_entity_by_id,
    to_entity_dataframe
)
from graphrag.data_model.entity import Entity

class EntitiesAPI:
    """REST API for entity operations."""

    def __init__(self, entities_path: str):
        # USES: graphrag/query/input/loaders/dfs.py:read_entities()
        self.entities: list[Entity] = read_entities(entities_path)
        self.entities_by_id = {e.id: e for e in self.entities}
        self.entities_by_title = {e.title: e for e in self.entities}

    def get_entities(
        self,
        type: str | None = None,
        community_id: str | None = None,
        min_rank: int | None = None,
        limit: int = 100,
        offset: int = 0,
        **filters
    ) -> dict:
        """
        GET /api/v1/graph/entities endpoint implementation.

        Uses existing Entity model from graphrag/data_model/entity.py
        """
        filtered = self.entities

        # Filter by type (Entity.type field line 16)
        if type:
            filtered = [e for e in filtered if e.type == type]

        # Filter by community (Entity.community_ids field line 29)
        if community_id:
            filtered = [
                e for e in filtered
                if e.community_ids and community_id in e.community_ids
            ]

        # Filter by rank (Entity.rank field line 34)
        if min_rank:
            filtered = [e for e in filtered if (e.rank or 0) >= min_rank]

        # Sort and paginate
        filtered.sort(key=lambda e: e.rank or 0, reverse=True)
        total = len(filtered)
        paginated = filtered[offset:offset + limit]

        return {
            "total": total,
            "limit": limit,
            "offset": offset,
            "entities": [self._entity_to_dict(e) for e in paginated]
        }

    def get_entity_by_id(self, entity_id: str, include_neighbors: bool = False) -> dict | None:
        """
        GET /api/v1/graph/entities/{id} endpoint implementation.

        Uses: graphrag/query/input/retrieval/entities.py:get_entity_by_id()
        """
        # USES: graphrag/query/input/retrieval/entities.py:15
        entity = get_entity_by_id(self.entities_by_id, entity_id)
        if not entity:
            return None

        result = {"entity": self._entity_to_dict(entity)}

        if include_neighbors:
            result["neighbors"] = self._get_neighbors(entity)

        return result

    def _entity_to_dict(self, entity: Entity) -> dict:
        """Convert Entity to dict using all fields."""
        return {
            # From graphrag/data_model/entity.py:13-38
            "id": entity.id,
            "short_id": entity.short_id,
            "title": entity.title,
            "type": entity.type,
            "description": entity.description,
            "description_embedding": entity.description_embedding,
            "name_embedding": entity.name_embedding,
            "community_ids": entity.community_ids,
            "text_unit_ids": entity.text_unit_ids,
            "rank": entity.rank,
            "attributes": entity.attributes
        }
```

**New Files to Create**:
1. `graphrag/api/graph/__init__.py`
2. `graphrag/api/graph/entities.py`
3. `graphrag/api/graph/relationships.py`
4. `graphrag/api/graph/communities.py`
5. `graphrag/api/graph/traversal.py`

**Estimated LOC**: ~1000 lines

---

### POST /api/v1/graph/traverse

**Existing Components**:
- `graphrag/query/context_builder/local_context.py:_filter_relationships()` (lines 228-313)
- `graphrag/query/input/retrieval/relationships.py:get_in_network_relationships()`, `get_out_network_relationships()`

**Implementation**:

```python
# graphrag/api/graph/traversal.py (NEW FILE)

from graphrag.query.input.retrieval.relationships import (
    get_in_network_relationships,
    get_out_network_relationships,
    get_candidate_relationships
)

class GraphTraversalAPI:
    """Graph traversal operations."""

    def __init__(self, entities: list[Entity], relationships: list[Relationship]):
        self.entities = entities
        self.relationships = relationships
        self.entities_by_title = {e.title: e for e in entities}

    def traverse(
        self,
        seed_entities: list[str],
        strategy: str = "bfs",
        max_depth: int = 2,
        max_nodes: int = 100,
        filters: dict | None = None
    ) -> dict:
        """
        POST /api/v1/graph/traverse endpoint implementation.

        Uses BFS/DFS on relationship network.
        """
        # Get seed entities
        seeds = [self.entities_by_title.get(title) for title in seed_entities]
        seeds = [s for s in seeds if s is not None]

        if not seeds:
            return {"error": "No valid seed entities found"}

        # BFS traversal
        visited = set()
        queue = [(seed, 0) for seed in seeds]  # (entity, depth)
        nodes = []
        edges = []

        while queue and len(nodes) < max_nodes:
            entity, depth = queue.pop(0)

            if entity.id in visited or depth > max_depth:
                continue

            visited.add(entity.id)
            nodes.append(self._node_to_dict(entity, depth))

            if depth < max_depth:
                # USES: graphrag/query/input/retrieval/relationships.py
                neighbors = self._get_neighbors(entity, filters)

                for neighbor_entity, relationship in neighbors:
                    if neighbor_entity.id not in visited:
                        queue.append((neighbor_entity, depth + 1))
                        edges.append(self._edge_to_dict(relationship))

        return {
            "nodes_discovered": len(nodes),
            "edges_discovered": len(edges),
            "graph": {"nodes": nodes, "edges": edges}
        }
```

**Estimated LOC**: ~300 lines

---

## 3. Vector API Mapping

### POST /api/v1/vectors/search/similar

**Existing Components**:
- `graphrag/vector_stores/base.py:VectorStore.similarity_search()`
- `graphrag/vector_stores/lancedb.py:LanceDBVectorStore`
- `graphrag/query/context_builder/entity_extraction.py:EntityVectorStoreKey`

**Implementation**:

```python
# graphrag/api/vectors/search.py (NEW FILE)

from graphrag.vector_stores.base import VectorStore
from graphrag.data_model.entity import Entity

class VectorSearchAPI:
    """Vector similarity search operations."""

    def __init__(self, vector_store: VectorStore, entities: list[Entity]):
        # USES: graphrag/vector_stores/lancedb.py:LanceDBVectorStore
        self.vector_store = vector_store
        self.entities_by_id = {e.id: e for e in entities}

    async def search_similar(
        self,
        query: str,
        top_k: int = 10,
        filters: dict | None = None
    ) -> dict:
        """
        POST /api/v1/vectors/search/similar endpoint implementation.

        Uses: graphrag/vector_stores/base.py:VectorStore.similarity_search()
        """
        # Embed query
        query_embedding = await self._embed_query(query)

        # USES: graphrag/vector_stores/base.py (abstract method)
        # Implemented in: graphrag/vector_stores/lancedb.py:similarity_search()
        results = await self.vector_store.similarity_search(
            query_embedding=query_embedding,
            k=top_k,
            filter=self._build_vector_store_filter(filters)
        )

        # Hydrate with entity data
        return {
            "query": query,
            "results": [
                {
                    "entity": self.entities_by_id.get(r.id),
                    "similarity": r.score,
                    "distance": 1 - r.score
                }
                for r in results
                if r.id in self.entities_by_id
            ]
        }

    async def _embed_query(self, query: str) -> list[float]:
        """
        Embed query using same model as entities.

        Could use: graphrag/index/operations/embed_text/strategies/openai.py
        """
        # Implementation here
        pass
```

**Files Referenced**:
- `graphrag/vector_stores/base.py` (lines 1-100)
- `graphrag/vector_stores/lancedb.py` (lines 1-200)

**New Files to Create**:
1. `graphrag/api/vectors/__init__.py`
2. `graphrag/api/vectors/search.py`
3. `graphrag/api/vectors/analytics.py`

**Estimated LOC**: ~400 lines

---

## 4. Search Trajectory API Mapping

### POST /api/v1/search/trajectory

**Existing Components**:
- All search classes: `LocalSearch`, `GlobalSearch`, `DRIFTSearch`
- Context builders: `LocalContextBuilder`, `GlobalContextBuilder`

**Implementation**:

```python
# graphrag/query/introspection/trace.py (NEW FILE)

class Trace:
    """Records execution trace for search trajectory."""

    def __init__(self, query: str, search_type: str):
        self.trace_id = str(uuid.uuid4())
        self.query = query
        self.search_type = search_type
        self.stages = []
        self.current_stage = None
        self.created_at = datetime.utcnow()

    def start_stage(self, name: str, inputs: dict | None = None):
        """Start recording a new stage."""
        self.current_stage = {
            "stage_id": len(self.stages) + 1,
            "stage_name": name,
            "timestamp": datetime.utcnow().isoformat(),
            "start_time": time.time(),
            "inputs": inputs or {},
            "operations": [],
            "outputs": {},
            "decision": {}
        }

    def record_operation(self, operation: str, **kwargs):
        """Record an operation within current stage."""
        if not self.current_stage:
            raise ValueError("No active stage to record operation")

        self.current_stage["operations"].append({
            "operation": operation,
            "timestamp": datetime.utcnow().isoformat(),
            **kwargs
        })

    def end_stage(self, outputs: dict, decision: dict | None = None):
        """End current stage and add to trace."""
        if not self.current_stage:
            raise ValueError("No active stage to end")

        self.current_stage["duration_ms"] = int(
            (time.time() - self.current_stage.pop("start_time")) * 1000
        )
        self.current_stage["outputs"] = outputs
        self.current_stage["decision"] = decision or {}

        self.stages.append(self.current_stage)
        self.current_stage = None

    def serialize(self) -> dict:
        """Serialize trace to dict."""
        return {
            "trace_id": self.trace_id,
            "query": self.query,
            "search_type": self.search_type,
            "created_at": self.created_at.isoformat(),
            "stages": self.stages,
            "trajectory_summary": {
                "total_stages": len(self.stages),
                "total_duration_ms": sum(s["duration_ms"] for s in self.stages)
            }
        }
```

**Integration with LocalSearch**:

```python
# Modifications to graphrag/query/context_builder/local_context.py

def build_entity_context(
    selected_entities: list[Entity],
    token_encoder: tiktoken.Encoding | None = None,
    max_tokens: int = 8000,
    trace: Trace | None = None,  # NEW PARAMETER
    **kwargs
) -> tuple[str, pd.DataFrame]:
    """Prepare entity data table as context data for system prompt."""

    # NEW: Record entity selection if trace enabled
    if trace:
        trace.record_operation(
            operation="entity_selection",
            total_entities_considered=len(selected_entities),
            max_tokens=max_tokens
        )

    # EXISTING LOGIC ...

    if len(selected_entities) == 0:
        return "", pd.DataFrame()

    # ... rest of existing code ...

    # NEW: Record final selection
    if trace:
        trace.record_operation(
            operation="entity_context_built",
            entities_included=len(all_context_records) - 1,
            tokens_used=current_tokens,
            truncated=current_tokens + new_tokens > max_tokens
        )

    return current_context_text, record_df
```

**New Files to Create**:
1. `graphrag/query/introspection/trace.py`
2. `graphrag/query/introspection/trace_store.py`
3. `graphrag/api/trajectory/__init__.py`
4. `graphrag/api/trajectory/endpoints.py`

**Estimated LOC**: ~600 lines

---

## 5. Attractor Network API Mapping

### POST /api/v1/attractors/detect

**Existing Components**:
- Entity rank field: `graphrag/data_model/entity.py:34` (`rank`)
- Graph metrics could use NetworkX

**Implementation**:

```python
# graphrag/api/attractors/detector.py (NEW FILE)

import networkx as nx
from graphrag.data_model.entity import Entity
from graphrag.data_model.relationship import Relationship

class AttractorDetector:
    """Detect attractor entities in knowledge graph."""

    def __init__(self, entities: list[Entity], relationships: list[Relationship]):
        self.entities = entities
        self.relationships = relationships
        self.entities_by_id = {e.id: e for e in entities}

        # Build NetworkX graph for centrality calculations
        self.graph = nx.DiGraph()
        for entity in entities:
            # USES: graphrag/data_model/entity.py:34 (rank field)
            self.graph.add_node(entity.id, rank=entity.rank or 0)

        for rel in relationships:
            # USES: graphrag/data_model/relationship.py
            self.graph.add_edge(
                rel.source_id,
                rel.target_id,
                weight=rel.weight or 1.0
            )

    def detect_attractors(
        self,
        metric_weights: dict,
        min_score: float = 0.7,
        max_attractors: int = 10
    ) -> list[dict]:
        """
        POST /api/v1/attractors/detect endpoint implementation.

        Computes centrality metrics using NetworkX.
        """
        # Compute centrality metrics
        degree_centrality = nx.degree_centrality(self.graph)
        pagerank = nx.pagerank(self.graph, weight="weight")
        betweenness = nx.betweenness_centrality(self.graph, weight="weight")

        # Combine metrics
        scores = {}
        for entity_id in self.graph.nodes():
            scores[entity_id] = (
                metric_weights.get("degree_weight", 0.4) * degree_centrality[entity_id] +
                metric_weights.get("pagerank_weight", 0.4) * pagerank[entity_id] +
                metric_weights.get("betweenness_weight", 0.2) * betweenness[entity_id]
            )

        # Filter and rank
        attractors = [
            (entity_id, score)
            for entity_id, score in scores.items()
            if score >= min_score
        ]
        attractors.sort(key=lambda x: x[1], reverse=True)

        # Build response
        return [
            {
                "entity": self.entities_by_id[entity_id],
                "attractor_score": score,
                "metrics": {
                    "degree_centrality": degree_centrality[entity_id],
                    "pagerank": pagerank[entity_id],
                    "betweenness": betweenness[entity_id]
                },
                "basin": self._compute_basin(entity_id)
            }
            for entity_id, score in attractors[:max_attractors]
        ]

    def _compute_basin(self, attractor_id: str) -> dict:
        """Compute attraction basin for attractor."""
        # Get all nodes within k hops
        basin_nodes = nx.single_source_shortest_path_length(
            self.graph,
            attractor_id,
            cutoff=2
        )

        return {
            "size": len(basin_nodes),
            "entities": [
                self.entities_by_id[node_id].title
                for node_id in basin_nodes
                if node_id in self.entities_by_id
            ]
        }
```

**Dependencies**:
- `networkx` (already in dependencies for graph operations)

**New Files to Create**:
1. `graphrag/api/attractors/__init__.py`
2. `graphrag/api/attractors/detector.py`
3. `graphrag/api/attractors/concept_expander.py`
4. `graphrag/api/attractors/question_gen.py`

**Estimated LOC**: ~800 lines

---

## Summary — Total Implementation Effort

| Component | New Files | Modified Files | Estimated LOC | Priority |
|-----------|-----------|----------------|---------------|----------|
| Introspection API | 4 | 3 (search classes) | ~500 | HIGH |
| Graph API | 5 | 0 | ~1000 | HIGH |
| Vector API | 3 | 0 | ~400 | MEDIUM |
| Trajectory API | 4 | 2 (context builders) | ~600 | HIGH |
| Attractor API | 4 | 0 | ~800 | MEDIUM |
| **TOTAL** | **20** | **5** | **~3300** | - |

---

## Dependencies

### External Libraries (already in project)
- `networkx` — Graph algorithms (centrality, shortest path)
- `pandas` — Data manipulation
- `tiktoken` — Token counting
- `redis` (optional) — Trace storage

### New Internal Dependencies
- None (uses existing GraphRAG components)

---

## API Server Framework

### Recommended: FastAPI

```python
# graphrag/api/server.py (NEW FILE)

from fastapi import FastAPI, HTTPException
from graphrag.api.graph import EntitiesAPI, RelationshipsAPI
from graphrag.api.vectors import VectorSearchAPI
from graphrag.api.trajectory import TrajectoryAPI
from graphrag.api.attractors import AttractorDetector

app = FastAPI(title="GraphRAG Introspection API")

# Initialize components
entities_api = EntitiesAPI(entities_path="./output/entities.parquet")
# ... other components ...

@app.post("/api/v1/search")
async def search_with_introspection(request: SearchRequest):
    """Search endpoint with introspection support."""
    # Uses modified LocalSearch/GlobalSearch classes
    pass

@app.get("/api/v1/graph/entities")
async def get_entities(
    type: str | None = None,
    limit: int = 100,
    offset: int = 0
):
    """Get entities with filtering."""
    return entities_api.get_entities(type=type, limit=limit, offset=offset)

# ... other endpoints ...
```

---

## Testing Strategy

### Unit Tests
- Test each API class in isolation
- Mock external dependencies (LLM, vector store)
- Focus on logic correctness

### Integration Tests
- Test full API flow with real GraphRAG index
- Validate response schemas
- Performance benchmarks

### Example Test

```python
# tests/api/test_graph_api.py

import pytest
from graphrag.api.graph import EntitiesAPI

def test_get_entities_with_type_filter():
    """Test entity filtering by type."""
    api = EntitiesAPI(entities_path="tests/fixtures/entities.parquet")

    result = api.get_entities(type="organization", limit=10)

    assert result["total"] >= 0
    assert len(result["entities"]) <= 10
    assert all(e["type"] == "organization" for e in result["entities"])
```

---

## Migration Path

### Phase 1: Core Introspection (Week 1-2)
1. Implement `Trace` and `TraceStore`
2. Modify `LocalSearch` to support introspection
3. Add `/api/v1/search` endpoint with `introspection` parameter
4. Unit tests

### Phase 2: Graph API (Week 3-4)
1. Implement `EntitiesAPI`, `RelationshipsAPI`
2. Add graph endpoints
3. Integration tests with real index

### Phase 3: Vector & Trajectory (Week 5-6)
1. Implement `VectorSearchAPI`
2. Complete trajectory recording in context builders
3. Add visualization endpoints

### Phase 4: Attractors & Polish (Week 7-8)
1. Implement `AttractorDetector`
2. Add related questions generation
3. Performance optimization
4. Documentation

**Total Estimated Time**: 8 weeks (1 developer)
