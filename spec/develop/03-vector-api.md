# Vector API — Доступ к векторным представлениям

## Обзор

Vector API предоставляет доступ к векторным embeddings сущностей и text units, а также к операциям similarity search и vector space analysis.

## Ключевые возможности

1. **Embeddings Access** — получение векторных представлений (text-embedding-ada-002, Node2Vec)
2. **Similarity Search** — поиск похожих сущностей/текстов в векторном пространстве
3. **Vector Space Analytics** — кластеризация, проекции, аномалии
4. **Hybrid Search** — комбинированный поиск (vector + graph centrality)

---

## Core Endpoints

### 1. Embeddings Retrieval

#### GET /api/v1/vectors/entities/{entity_id}/embeddings

Получить embeddings конкретной сущности.

**Query Parameters**:
```
embedding_type: enum[description, name, graph] (default: description)
  - description: text embedding description
  - name: text embedding name/title
  - graph: Node2Vec structural embedding
format: enum[array, base64] (default: array)
```

**Response**:
```json
{
  "entity_id": "e1234567-89ab-cdef-0123-456789abcdef",
  "entity_title": "MICROSOFT",
  "embeddings": {
    "description": {
      "model": "text-embedding-ada-002",
      "dimensions": 1536,
      "vector": [0.123, -0.456, 0.789, ...],
      "norm": 1.0,
      "created_at": "2025-11-14T10:23:45Z"
    },
    "name": {
      "model": "text-embedding-ada-002",
      "dimensions": 1536,
      "vector": [0.234, -0.567, 0.890, ...],
      "norm": 1.0
    },
    "graph": {
      "model": "Node2Vec",
      "dimensions": 1536,
      "vector": [0.345, -0.678, 0.901, ...],
      "norm": 1.0,
      "training_params": {
        "num_walks": 10,
        "walk_length": 40,
        "window_size": 2
      }
    }
  }
}
```

---

### 2. Similarity Search

#### POST /api/v1/vectors/search/similar

Найти похожие сущности/тексты по векторному сходству.

**Request Body**:
```json
{
  "query": "AI partnerships and research collaborations",
  "search_type": "entity|text_unit|both",
  "embedding_type": "description|name|graph",
  "top_k": 10,
  "similarity_metric": "cosine|euclidean|dot_product",
  "min_similarity": 0.7,
  "filters": {
    "entity_types": ["organization"],
    "communities": ["community_5"],
    "min_rank": 10
  },
  "include_scores": true,
  "include_embeddings": false
}
```

**Response**:
```json
{
  "query": "AI partnerships and research collaborations",
  "query_embedding": {
    "model": "text-embedding-ada-002",
    "dimensions": 1536,
    "vector_preview": [0.123, -0.456, ...]  // first 10 dims
  },
  "search_type": "entity",
  "top_k": 10,
  "similarity_metric": "cosine",
  "total_candidates": 50234,
  "filtered_candidates": 1234,

  "results": [
    {
      "entity": {
        "id": "e1234567-89ab-cdef-0123-456789abcdef",
        "title": "MICROSOFT",
        "type": "organization",
        "description": "Microsoft Corporation...",
        "rank": 1
      },
      "similarity": 0.94,
      "distance": 0.06,
      "embedding_type": "description",
      "matched_attributes": {
        "keywords": ["AI", "partnerships", "research"],
        "entities_mentioned": ["OpenAI", "Meta AI"],
        "concepts": ["collaboration", "innovation"]
      }
    },
    {
      "entity": {
        "id": "e2345678-89ab-cdef-0123-456789abcdef",
        "title": "OPENAI",
        "type": "organization",
        "description": "OpenAI is an AI research laboratory...",
        "rank": 3
      },
      "similarity": 0.89,
      "distance": 0.11,
      "embedding_type": "description"
    }
  ],

  "search_metadata": {
    "search_time_ms": 45,
    "ann_algorithm": "HNSW",
    "ef_search": 100,
    "recall_estimate": 0.98
  }
}
```

---

### 3. Vector Space Analytics

#### POST /api/v1/vectors/analyze/clusters

Кластеризация в векторном пространстве.

**Request Body**:
```json
{
  "embedding_type": "description|graph",
  "clustering_algorithm": "kmeans|hdbscan|spectral",
  "n_clusters": 10,  // for kmeans
  "min_cluster_size": 5,  // for hdbscan
  "filters": {
    "entity_types": ["organization", "person"],
    "min_rank": 5
  },
  "include_visualization": true,
  "projection_method": "umap|tsne|pca"
}
```

**Response**:
```json
{
  "clustering_algorithm": "kmeans",
  "n_clusters": 10,
  "total_entities": 1234,
  "silhouette_score": 0.67,

  "clusters": [
    {
      "cluster_id": 0,
      "size": 123,
      "centroid": [0.123, -0.456, ...],
      "top_entities": [
        {"title": "MICROSOFT", "distance_to_centroid": 0.12},
        {"title": "GOOGLE", "distance_to_centroid": 0.15}
      ],
      "cluster_keywords": ["technology", "AI", "cloud"],
      "coherence_score": 0.78
    }
  ],

  "visualization": {
    "projection_method": "umap",
    "dimensions": 2,
    "points": [
      {
        "entity_id": "e1234567...",
        "entity_title": "MICROSOFT",
        "cluster_id": 0,
        "x": 0.45,
        "y": 0.67,
        "color": "#FF6B6B"
      }
    ]
  }
}
```

---

### 4. Hybrid Search

#### POST /api/v1/vectors/search/hybrid

Комбинированный поиск (vector similarity + graph metrics).

**Request Body**:
```json
{
  "query": "What are Microsoft's AI partnerships?",
  "weights": {
    "vector_similarity": 0.6,
    "graph_centrality": 0.3,
    "community_relevance": 0.1
  },
  "top_k": 20,
  "rerank": true,
  "filters": {
    "entity_types": ["organization"],
    "min_rank": 5
  }
}
```

**Response**:
```json
{
  "query": "What are Microsoft's AI partnerships?",
  "hybrid_results": [
    {
      "entity": {
        "id": "e1234567...",
        "title": "MICROSOFT",
        "type": "organization"
      },
      "scores": {
        "vector_similarity": 0.94,
        "graph_centrality": 0.89,
        "community_relevance": 0.85,
        "combined_score": 0.91
      },
      "score_breakdown": {
        "vector_contribution": 0.564,
        "graph_contribution": 0.267,
        "community_contribution": 0.085
      },
      "rank": 1,
      "explanation": "High vector similarity (0.94) combined with top graph centrality"
    }
  ]
}
```

---

## Реализация — Mapping к компонентам

| API Endpoint | Python Component | Key Files |
|--------------|------------------|-----------|
| `GET /api/v1/vectors/entities/{id}/embeddings` | Entity data model | `graphrag/data_model/entity.py:22-26` (description_embedding, name_embedding) |
| `POST /api/v1/vectors/search/similar` | Vector search | `graphrag/vector_stores/base.py`<br/>`graphrag/vector_stores/lancedb.py`<br/>`graphrag/query/context_builder/entity_extraction.py` |
| `POST /api/v1/vectors/analyze/clusters` | UMAP projection | `graphrag/index/operations/layout_graph/umap.py` |
| `POST /api/v1/vectors/search/hybrid` | Local context builder | `graphrag/query/context_builder/local_context.py` |

### Предлагаемый код

```python
# graphrag/api/vectors/search.py

from graphrag.vector_stores.base import VectorStore
from graphrag.data_model.entity import Entity

class VectorSearchAPI:
    """API for vector similarity search."""

    def __init__(self, vector_store: VectorStore, entities: list[Entity]):
        self.vector_store = vector_store
        self.entities_by_id = {e.id: e for e in entities}

    async def search_similar(
        self,
        query: str,
        embedding_type: str = "description",
        top_k: int = 10,
        filters: dict | None = None
    ) -> dict:
        """Perform similarity search."""
        # Embed query
        query_embedding = await self._embed_query(query)

        # Vector search
        results = await self.vector_store.similarity_search(
            query_embedding=query_embedding,
            k=top_k,
            filter=filters
        )

        # Hydrate with entity data
        return {
            "query": query,
            "results": [
                {
                    "entity": self._entity_to_dict(self.entities_by_id[r.id]),
                    "similarity": r.score,
                    "distance": 1 - r.score
                }
                for r in results
            ]
        }
```

---

## Примеры использования

### 1. Получить embeddings для Microsoft

```bash
curl "http://localhost:8000/api/v1/vectors/entities/MICROSOFT/embeddings?embedding_type=description"
```

### 2. Найти похожие сущности

```bash
curl -X POST http://localhost:8000/api/v1/vectors/search/similar \
  -H "Content-Type: application/json" \
  -d '{
    "query": "AI research partnerships",
    "search_type": "entity",
    "top_k": 10,
    "min_similarity": 0.7
  }'
```

### 3. Hybrid search

```bash
curl -X POST http://localhost:8000/api/v1/vectors/search/hybrid \
  -H "Content-Type: application/json" \
  -d '{
    "query": "Microsoft AI partnerships",
    "weights": {
      "vector_similarity": 0.6,
      "graph_centrality": 0.4
    },
    "top_k": 20
  }'
```
