# Graph API — Доступ к графовому представлению

## Обзор

Graph API предоставляет прямой доступ к knowledge graph GraphRAG, позволяя внешним пользователям исследовать граф сущностей, связей и communities, а также выполнять graph traversal операции.

## Архитектура Graph API

```
┌──────────────────────────────────────────────────────────────────────┐
│                         GRAPH API LAYERS                              │
└──────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  REST API Layer                                                      │
│  • GET /api/v1/graph/entities                                       │
│  • GET /api/v1/graph/relationships                                  │
│  • GET /api/v1/graph/communities                                    │
│  • POST /api/v1/graph/traverse                                      │
│  • POST /api/v1/graph/subgraph                                      │
└─────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Graph Query Engine                                                  │
│  • Entity filtering (type, rank, community)                         │
│  • Relationship filtering (weight, type)                            │
│  • Graph traversal algorithms (BFS, DFS, shortest path)             │
│  • Subgraph extraction (k-hop, community-based)                     │
└─────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Data Model (graphrag/data_model/)                                  │
│  • Entity (graphrag/data_model/entity.py)                           │
│  • Relationship (graphrag/data_model/relationship.py)               │
│  • Community (graphrag/data_model/community.py)                     │
│  • CommunityReport (graphrag/data_model/community_report.py)        │
└─────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Storage Layer (Parquet files / Vector stores)                      │
│  • output/entities.parquet                                          │
│  • output/relationships.parquet                                     │
│  • output/communities.parquet                                       │
│  • output/community_reports.parquet                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Core Endpoints

### 1. Entities API

#### GET /api/v1/graph/entities

Получить список сущностей с фильтрацией и пагинацией.

**Query Parameters**:
```
type: string (optional) - фильтр по типу сущности (organization, person, concept, etc.)
community_id: string (optional) - фильтр по community
min_rank: integer (optional) - минимальный rank (centrality)
max_rank: integer (optional) - максимальный rank
search: string (optional) - поиск по title и description (полнотекстовый)
limit: integer (default: 100, max: 1000)
offset: integer (default: 0)
sort_by: enum[rank, title, degree] (default: rank)
sort_order: enum[asc, desc] (default: desc)
include_embeddings: boolean (default: false)
include_attributes: boolean (default: true)
```

**Response**:
```json
{
  "total": 50234,
  "limit": 100,
  "offset": 0,
  "entities": [
    {
      "id": "e1234567-89ab-cdef-0123-456789abcdef",
      "short_id": "0",
      "title": "MICROSOFT",
      "type": "organization",
      "description": "Microsoft Corporation is a multinational technology company...",
      "rank": 1,
      "degree": 245,
      "community_ids": ["community_0", "community_5", "community_12"],
      "text_unit_ids": ["unit_123", "unit_456", "unit_789"],
      "attributes": {
        "founded": 1975,
        "headquarters": "Redmond, WA",
        "industry": "Technology",
        "pagerank": 0.0123
      },
      "description_embedding": null,  // if include_embeddings=true, returns [0.123, -0.456, ...]
      "name_embedding": null,
      "graph_position": {
        "x": 0.45,
        "y": 0.67,
        "z": 0.23
      }  // if Node2Vec embeddings available
    },
    {
      "id": "e2345678-89ab-cdef-0123-456789abcdef",
      "short_id": "1",
      "title": "OPENAI",
      "type": "organization",
      "description": "OpenAI is an AI research laboratory...",
      "rank": 3,
      "degree": 187,
      "community_ids": ["community_5"],
      "text_unit_ids": ["unit_234", "unit_567"],
      "attributes": {
        "founded": 2015,
        "pagerank": 0.0089
      },
      "description_embedding": null,
      "name_embedding": null,
      "graph_position": {
        "x": 0.48,
        "y": 0.69,
        "z": 0.25
      }
    }
  ],
  "metadata": {
    "total_entities_in_graph": 50234,
    "filtered_entities": 50234,
    "types_distribution": {
      "organization": 1234,
      "person": 2345,
      "concept": 3456,
      "location": 890,
      "event": 567
    },
    "rank_statistics": {
      "min": 1,
      "max": 245,
      "median": 15,
      "mean": 23.4
    }
  }
}
```

---

#### GET /api/v1/graph/entities/{entity_id}

Получить детальную информацию о конкретной сущности.

**Query Parameters**:
```
include_embeddings: boolean (default: false)
include_neighbors: boolean (default: true) - включить соседние сущности
neighbors_depth: integer (default: 1, max: 3) - глубина соседства
include_text_units: boolean (default: false) - включить text units упоминающие сущность
include_communities: boolean (default: true) - включить community информацию
```

**Response**:
```json
{
  "entity": {
    "id": "e1234567-89ab-cdef-0123-456789abcdef",
    "short_id": "0",
    "title": "MICROSOFT",
    "type": "organization",
    "description": "Microsoft Corporation is a multinational technology company...",
    "rank": 1,
    "degree": 245,
    "community_ids": ["community_0", "community_5", "community_12"],
    "text_unit_ids": ["unit_123", "unit_456"],
    "attributes": {
      "founded": 1975,
      "pagerank": 0.0123,
      "betweenness_centrality": 0.0456,
      "closeness_centrality": 0.0234
    },
    "description_embedding": [0.123, -0.456, ...],  // if requested
    "name_embedding": [0.234, -0.567, ...]
  },

  "neighbors": {
    "outgoing": [
      {
        "entity": {
          "id": "e2345678-89ab-cdef-0123-456789abcdef",
          "title": "OPENAI",
          "type": "organization"
        },
        "relationship": {
          "id": "r1234567-89ab-cdef-0123-456789abcdef",
          "type": "FOUNDED",
          "description": "Microsoft founded OpenAI in 2015",
          "weight": 0.95,
          "rank": 1
        },
        "distance": 1
      }
    ],
    "incoming": [
      {
        "entity": {
          "id": "e3456789-89ab-cdef-0123-456789abcdef",
          "title": "SATYA NADELLA",
          "type": "person"
        },
        "relationship": {
          "id": "r2345678-89ab-cdef-0123-456789abcdef",
          "type": "CEO_OF",
          "description": "Satya Nadella is CEO of Microsoft",
          "weight": 0.98,
          "rank": 1
        },
        "distance": 1
      }
    ],
    "mutual": [
      {
        "entity": {
          "id": "e4567890-89ab-cdef-0123-456789abcdef",
          "title": "AZURE",
          "type": "product"
        },
        "relationships": [
          {
            "id": "r3456789-89ab-cdef-0123-456789abcdef",
            "source": "MICROSOFT",
            "target": "AZURE",
            "type": "DEVELOPS",
            "weight": 0.92
          },
          {
            "id": "r4567890-89ab-cdef-0123-456789abcdef",
            "source": "AZURE",
            "target": "MICROSOFT",
            "type": "PRODUCT_OF",
            "weight": 0.89
          }
        ],
        "distance": 1
      }
    ]
  },

  "communities": [
    {
      "id": "community_5",
      "level": 0,
      "title": "AI Research and Development",
      "size": 45,
      "entity_role": "central",  // central, peripheral, bridge
      "summary": "This community focuses on AI research organizations and their partnerships..."
    }
  ],

  "text_units": [
    {
      "id": "unit_123",
      "text": "Microsoft announced a major partnership with OpenAI, investing $1 billion...",
      "source_document_ids": ["doc_45"],
      "entities_mentioned": ["MICROSOFT", "OPENAI"],
      "position_in_document": 0.23
    }
  ],

  "graph_metrics": {
    "degree": 245,
    "in_degree": 123,
    "out_degree": 122,
    "pagerank": 0.0123,
    "betweenness_centrality": 0.0456,
    "closeness_centrality": 0.0234,
    "clustering_coefficient": 0.67,
    "triangles": 34
  }
}
```

---

### 2. Relationships API

#### GET /api/v1/graph/relationships

Получить список связей между сущностями.

**Query Parameters**:
```
source_entity_id: string (optional) - фильтр по source entity
target_entity_id: string (optional) - фильтр по target entity
entity_id: string (optional) - фильтр по любой entity (source or target)
type: string (optional) - тип связи
min_weight: float (optional) - минимальный вес
min_rank: integer (optional) - минимальный rank
limit: integer (default: 100)
offset: integer (default: 0)
sort_by: enum[weight, rank] (default: weight)
sort_order: enum[asc, desc] (default: desc)
```

**Response**:
```json
{
  "total": 150456,
  "limit": 100,
  "offset": 0,
  "relationships": [
    {
      "id": "r1234567-89ab-cdef-0123-456789abcdef",
      "short_id": "0",
      "source": "MICROSOFT",
      "source_id": "e1234567-89ab-cdef-0123-456789abcdef",
      "target": "OPENAI",
      "target_id": "e2345678-89ab-cdef-0123-456789abcdef",
      "type": "FOUNDED",
      "description": "Microsoft founded OpenAI in 2015 as part of its AI strategy",
      "weight": 0.95,
      "rank": 1,
      "text_unit_ids": ["unit_123", "unit_234"],
      "attributes": {
        "year": 2015,
        "investment_amount": "$1B",
        "relationship_strength": "strong"
      }
    }
  ],
  "metadata": {
    "total_relationships_in_graph": 150456,
    "filtered_relationships": 150456,
    "types_distribution": {
      "FOUNDED": 234,
      "PARTNERED_WITH": 456,
      "ACQUIRED": 123,
      "CEO_OF": 89,
      "DEVELOPED": 567
    },
    "weight_statistics": {
      "min": 0.01,
      "max": 0.98,
      "median": 0.45,
      "mean": 0.52
    }
  }
}
```

---

### 3. Communities API

#### GET /api/v1/graph/communities

Получить список communities (кластеров сущностей).

**Query Parameters**:
```
level: integer (optional) - иерархический уровень (0 = fine-grained, higher = coarser)
min_size: integer (optional) - минимальный размер community
max_size: integer (optional) - максимальный размер community
search: string (optional) - поиск по названию и summary
limit: integer (default: 100)
offset: integer (default: 0)
sort_by: enum[size, level, title] (default: size)
sort_order: enum[asc, desc] (default: desc)
include_entities: boolean (default: false)
include_report: boolean (default: true)
```

**Response**:
```json
{
  "total": 523,
  "limit": 100,
  "offset": 0,
  "communities": [
    {
      "id": "community_5",
      "short_id": "5",
      "level": 0,
      "title": "AI Research and Development",
      "size": 45,
      "entity_ids": ["e1234567...", "e2345678...", ...],  // if include_entities=true
      "relationship_count": 123,
      "report": {
        "id": "report_5",
        "title": "AI Research and Development Community Report",
        "summary": "This community represents the ecosystem of AI research organizations...",
        "full_content": "## Overview\n\nThe AI Research and Development community...",
        "rank": 8.5,
        "rank_explanation": "High importance due to central role in AI ecosystem",
        "findings": [
          "Microsoft and OpenAI have a strong partnership focused on AGI",
          "Google DeepMind is a key competitor in AI research",
          "Meta AI Research contributes to open-source AI models"
        ],
        "rating": 8.5,
        "rating_explanation": "High coherence and importance"
      },
      "parent_community_id": "community_102",  // if hierarchical
      "child_community_ids": ["community_45", "community_67"],
      "metrics": {
        "modularity": 0.78,
        "density": 0.34,
        "average_degree": 12.3,
        "diameter": 4
      }
    }
  ],
  "metadata": {
    "total_communities_in_graph": 523,
    "hierarchy_levels": 3,
    "size_distribution": {
      "small (1-10)": 234,
      "medium (11-50)": 189,
      "large (51-100)": 78,
      "very_large (100+)": 22
    }
  }
}
```

---

#### GET /api/v1/graph/communities/{community_id}

Детальная информация о community.

**Query Parameters**:
```
include_entities: boolean (default: true)
include_relationships: boolean (default: true)
include_subgraph: boolean (default: false) - full subgraph structure
include_report: boolean (default: true)
```

**Response**:
```json
{
  "community": {
    "id": "community_5",
    "short_id": "5",
    "level": 0,
    "title": "AI Research and Development",
    "size": 45,
    "relationship_count": 123,
    "parent_community_id": "community_102",
    "child_community_ids": [],
    "metrics": {
      "modularity": 0.78,
      "density": 0.34,
      "average_degree": 12.3,
      "diameter": 4,
      "clustering_coefficient": 0.67
    }
  },

  "entities": [
    {
      "id": "e1234567-89ab-cdef-0123-456789abcdef",
      "title": "MICROSOFT",
      "type": "organization",
      "rank": 1,
      "role_in_community": "hub",  // hub, bridge, peripheral
      "local_centrality": 0.89
    },
    {
      "id": "e2345678-89ab-cdef-0123-456789abcdef",
      "title": "OPENAI",
      "type": "organization",
      "rank": 3,
      "role_in_community": "central",
      "local_centrality": 0.78
    }
  ],

  "relationships": [
    {
      "id": "r1234567-89ab-cdef-0123-456789abcdef",
      "source": "MICROSOFT",
      "target": "OPENAI",
      "type": "FOUNDED",
      "weight": 0.95,
      "is_internal": true  // both entities in same community
    }
  ],

  "subgraph": {
    "nodes": [...],  // if include_subgraph=true
    "edges": [...]
  },

  "report": {
    "id": "report_5",
    "title": "AI Research and Development Community Report",
    "summary": "This community represents...",
    "full_content": "## Overview\n\n...",
    "rank": 8.5,
    "findings": [...],
    "rating": 8.5
  },

  "neighboring_communities": [
    {
      "community_id": "community_12",
      "title": "Cloud Computing Infrastructure",
      "shared_entities": 5,
      "cross_community_relationships": 12,
      "similarity": 0.45
    }
  ]
}
```

---

### 4. Graph Traversal API

#### POST /api/v1/graph/traverse

Выполнить graph traversal от заданных seed entities.

**Request Body**:
```json
{
  "seed_entities": ["MICROSOFT", "OPENAI"],  // or entity IDs
  "traversal_strategy": "bfs|dfs|shortest_path|relationship_expansion",
  "max_depth": 2,
  "max_nodes": 100,
  "filters": {
    "entity_types": ["organization", "person"],
    "relationship_types": ["FOUNDED", "PARTNERED_WITH"],
    "min_relationship_weight": 0.5,
    "min_entity_rank": 10,
    "communities": ["community_5"]
  },
  "include_paths": true,
  "include_metrics": true
}
```

**Response**:
```json
{
  "traversal_id": "trav_550e8400-e29b-41d4-a716-446655440000",
  "seed_entities": ["MICROSOFT", "OPENAI"],
  "strategy": "bfs",
  "max_depth": 2,
  "nodes_discovered": 87,
  "edges_discovered": 156,

  "graph": {
    "nodes": [
      {
        "id": "e1234567-89ab-cdef-0123-456789abcdef",
        "title": "MICROSOFT",
        "type": "organization",
        "depth": 0,
        "discovery_order": 1,
        "parent_node": null,
        "rank": 1
      },
      {
        "id": "e2345678-89ab-cdef-0123-456789abcdef",
        "title": "OPENAI",
        "type": "organization",
        "depth": 0,
        "discovery_order": 2,
        "parent_node": null,
        "rank": 3
      },
      {
        "id": "e3456789-89ab-cdef-0123-456789abcdef",
        "title": "AZURE",
        "type": "product",
        "depth": 1,
        "discovery_order": 3,
        "parent_node": "MICROSOFT",
        "rank": 5
      }
    ],
    "edges": [
      {
        "id": "r1234567-89ab-cdef-0123-456789abcdef",
        "source": "MICROSOFT",
        "target": "OPENAI",
        "type": "FOUNDED",
        "weight": 0.95,
        "traversal_step": 1
      }
    ]
  },

  "paths": [
    {
      "from": "MICROSOFT",
      "to": "GPT-4",
      "path": ["MICROSOFT", "OPENAI", "GPT-4"],
      "length": 2,
      "total_weight": 1.87,
      "relationships": [
        {"type": "FOUNDED", "weight": 0.95},
        {"type": "DEVELOPED", "weight": 0.92}
      ]
    }
  ],

  "metrics": {
    "traversal_time_ms": 234,
    "nodes_visited": 87,
    "nodes_filtered": 13,
    "edges_traversed": 156,
    "max_depth_reached": 2,
    "average_branching_factor": 5.4
  },

  "statistics": {
    "entity_types_distribution": {
      "organization": 23,
      "person": 12,
      "product": 34,
      "concept": 18
    },
    "relationship_types_distribution": {
      "FOUNDED": 5,
      "PARTNERED_WITH": 12,
      "DEVELOPED": 34,
      "CEO_OF": 3
    },
    "depth_distribution": {
      "0": 2,
      "1": 23,
      "2": 62
    }
  }
}
```

---

### 5. Subgraph Extraction API

#### POST /api/v1/graph/subgraph

Извлечь подграф по заданным критериям.

**Request Body**:
```json
{
  "extraction_strategy": "k_hop|community|entities_list|query_based",

  // For k_hop strategy
  "seed_entities": ["MICROSOFT"],
  "k": 2,

  // For community strategy
  "community_ids": ["community_5", "community_12"],

  // For entities_list strategy
  "entity_ids": ["e1234567...", "e2345678...", ...],

  // For query_based strategy
  "query": "AI partnerships",
  "max_entities": 50,

  // Common filters
  "filters": {
    "entity_types": ["organization"],
    "min_entity_rank": 5,
    "relationship_types": ["FOUNDED", "PARTNERED_WITH"],
    "min_relationship_weight": 0.5
  },

  "include_internal_relationships_only": false,
  "include_isolated_nodes": false,
  "output_format": "json|graphml|gexf|cytoscape"
}
```

**Response (JSON format)**:
```json
{
  "subgraph_id": "sg_550e8400-e29b-41d4-a716-446655440000",
  "extraction_strategy": "k_hop",
  "seed_entities": ["MICROSOFT"],
  "k": 2,
  "timestamp": "2025-11-15T10:23:45.123Z",

  "graph": {
    "directed": true,
    "multigraph": false,
    "nodes": [
      {
        "id": "e1234567-89ab-cdef-0123-456789abcdef",
        "label": "MICROSOFT",
        "type": "organization",
        "rank": 1,
        "attributes": {
          "founded": 1975,
          "pagerank": 0.0123
        },
        "position": {
          "x": 0.45,
          "y": 0.67
        }
      }
    ],
    "edges": [
      {
        "id": "r1234567-89ab-cdef-0123-456789abcdef",
        "source": "e1234567-89ab-cdef-0123-456789abcdef",
        "target": "e2345678-89ab-cdef-0123-456789abcdef",
        "label": "FOUNDED",
        "weight": 0.95,
        "attributes": {
          "year": 2015
        }
      }
    ]
  },

  "statistics": {
    "total_nodes": 45,
    "total_edges": 123,
    "density": 0.12,
    "average_degree": 5.5,
    "diameter": 4,
    "average_clustering_coefficient": 0.34,
    "connected_components": 1,
    "largest_component_size": 45
  },

  "export_urls": {
    "graphml": "/api/v1/graph/subgraph/sg_550e8400.../export?format=graphml",
    "gexf": "/api/v1/graph/subgraph/sg_550e8400.../export?format=gexf",
    "cytoscape": "/api/v1/graph/subgraph/sg_550e8400.../export?format=cytoscape",
    "json": "/api/v1/graph/subgraph/sg_550e8400.../export?format=json"
  }
}
```

---

### 6. Shortest Path API

#### POST /api/v1/graph/shortest-path

Найти кратчайший путь между двумя сущностями.

**Request Body**:
```json
{
  "source_entity": "MICROSOFT",  // or entity ID
  "target_entity": "GPT-4",
  "algorithm": "dijkstra|bfs|bidirectional",
  "weight_attribute": "weight",  // or null for unweighted
  "max_path_length": 5,
  "find_all_paths": false,  // if true, find all shortest paths
  "filters": {
    "relationship_types": null,
    "min_relationship_weight": 0.3
  }
}
```

**Response**:
```json
{
  "source": {
    "id": "e1234567-89ab-cdef-0123-456789abcdef",
    "title": "MICROSOFT"
  },
  "target": {
    "id": "e9876543-21ab-cdef-0123-456789abcdef",
    "title": "GPT-4"
  },
  "path_exists": true,
  "shortest_path": {
    "length": 2,
    "total_weight": 1.87,
    "nodes": [
      {"id": "e1234567...", "title": "MICROSOFT"},
      {"id": "e2345678...", "title": "OPENAI"},
      {"id": "e9876543...", "title": "GPT-4"}
    ],
    "edges": [
      {
        "id": "r1234567...",
        "source": "MICROSOFT",
        "target": "OPENAI",
        "type": "FOUNDED",
        "weight": 0.95
      },
      {
        "id": "r2345678...",
        "source": "OPENAI",
        "target": "GPT-4",
        "type": "DEVELOPED",
        "weight": 0.92
      }
    ]
  },
  "alternative_paths": [
    {
      "length": 3,
      "total_weight": 2.34,
      "nodes": ["MICROSOFT", "AZURE", "OPENAI", "GPT-4"],
      "edges": [...]
    }
  ],
  "algorithm_used": "dijkstra",
  "computation_time_ms": 45
}
```

---

## Реализация на основе текущей архитектуры

### Mapping к компонентам кода

| API Endpoint | Python Component | Key Files |
|--------------|------------------|-----------|
| `GET /api/v1/graph/entities` | Data loaders | `graphrag/query/input/loaders/dfs.py`<br/>`graphrag/query/input/retrieval/entities.py` |
| `GET /api/v1/graph/entities/{id}` | Entity retrieval | `graphrag/query/input/retrieval/entities.py:15-42` (get_entity_by_id, get_entity_by_key) |
| `GET /api/v1/graph/relationships` | Relationship retrieval | `graphrag/query/input/retrieval/relationships.py` |
| `GET /api/v1/graph/communities` | Community loaders | `graphrag/data_model/community.py`<br/>`graphrag/data_model/community_report.py` |
| `POST /api/v1/graph/traverse` | Graph traversal logic | `graphrag/query/context_builder/local_context.py:228-313` (_filter_relationships)<br/>Custom BFS/DFS implementation |
| `POST /api/v1/graph/subgraph` | Subgraph extraction | `graphrag/query/context_builder/local_context.py:316-353` (get_candidate_context) |

### Предлагаемая структура кода

```python
# graphrag/api/graph/__init__.py

from graphrag.api.graph.entities import EntitiesAPI
from graphrag.api.graph.relationships import RelationshipsAPI
from graphrag.api.graph.communities import CommunitiesAPI
from graphrag.api.graph.traversal import GraphTraversalAPI
from graphrag.api.graph.subgraph import SubgraphAPI

class GraphAPI:
    """Unified Graph API facade."""

    def __init__(self, storage_config: dict):
        self.entities = EntitiesAPI(storage_config)
        self.relationships = RelationshipsAPI(storage_config)
        self.communities = CommunitiesAPI(storage_config)
        self.traversal = GraphTraversalAPI(storage_config)
        self.subgraph = SubgraphAPI(storage_config)
```

```python
# graphrag/api/graph/entities.py

from graphrag.data_model.entity import Entity
from graphrag.query.input.loaders.dfs import read_entities
from graphrag.query.input.retrieval.entities import (
    get_entity_by_id,
    get_entity_by_key,
    get_entity_by_name,
    to_entity_dataframe
)

class EntitiesAPI:
    """API for entity operations."""

    def __init__(self, storage_config: dict):
        self.entities = self._load_entities(storage_config)
        self.entities_by_id = {e.id: e for e in self.entities}
        self.entities_by_title = {e.title: e for e in self.entities}

    def _load_entities(self, config: dict) -> list[Entity]:
        """Load entities from parquet files."""
        return read_entities(config["entities_path"])

    def get_entities(
        self,
        type: str | None = None,
        community_id: str | None = None,
        min_rank: int | None = None,
        search: str | None = None,
        limit: int = 100,
        offset: int = 0,
        sort_by: str = "rank",
        sort_order: str = "desc"
    ) -> dict:
        """Get entities with filtering and pagination."""
        filtered = self.entities

        # Apply filters
        if type:
            filtered = [e for e in filtered if e.type == type]
        if community_id:
            filtered = [e for e in filtered if community_id in (e.community_ids or [])]
        if min_rank:
            filtered = [e for e in filtered if (e.rank or 0) >= min_rank]
        if search:
            search_lower = search.lower()
            filtered = [
                e for e in filtered
                if search_lower in e.title.lower()
                or (e.description and search_lower in e.description.lower())
            ]

        # Sort
        reverse = (sort_order == "desc")
        if sort_by == "rank":
            filtered.sort(key=lambda e: e.rank or 0, reverse=reverse)
        elif sort_by == "title":
            filtered.sort(key=lambda e: e.title, reverse=reverse)
        elif sort_by == "degree":
            filtered.sort(key=lambda e: e.attributes.get("degree", 0) if e.attributes else 0, reverse=reverse)

        # Paginate
        total = len(filtered)
        paginated = filtered[offset:offset + limit]

        return {
            "total": total,
            "limit": limit,
            "offset": offset,
            "entities": [self._entity_to_dict(e) for e in paginated],
            "metadata": self._compute_metadata(filtered)
        }

    def get_entity_by_id(self, entity_id: str, **options) -> dict | None:
        """Get entity by ID with optional neighbors, communities, etc."""
        entity = get_entity_by_id(self.entities_by_id, entity_id)
        if not entity:
            return None

        result = {"entity": self._entity_to_dict(entity)}

        if options.get("include_neighbors"):
            result["neighbors"] = self._get_neighbors(
                entity,
                depth=options.get("neighbors_depth", 1)
            )

        if options.get("include_communities"):
            result["communities"] = self._get_communities(entity)

        return result

    def _entity_to_dict(self, entity: Entity) -> dict:
        """Convert Entity to dict."""
        return {
            "id": entity.id,
            "short_id": entity.short_id,
            "title": entity.title,
            "type": entity.type,
            "description": entity.description,
            "rank": entity.rank,
            "community_ids": entity.community_ids,
            "text_unit_ids": entity.text_unit_ids,
            "attributes": entity.attributes,
            "description_embedding": entity.description_embedding,
            "name_embedding": entity.name_embedding
        }
```

---

## Примеры использования

### 1. Получить топ-10 сущностей по centrality

```bash
curl "http://localhost:8000/api/v1/graph/entities?sort_by=rank&sort_order=desc&limit=10"
```

### 2. Найти все организации в community_5

```bash
curl "http://localhost:8000/api/v1/graph/entities?type=organization&community_id=community_5"
```

### 3. Получить детали о Microsoft

```bash
curl "http://localhost:8000/api/v1/graph/entities/MICROSOFT?include_neighbors=true&neighbors_depth=2&include_communities=true"
```

### 4. Найти кратчайший путь Microsoft → GPT-4

```bash
curl -X POST http://localhost:8000/api/v1/graph/shortest-path \
  -H "Content-Type: application/json" \
  -d '{
    "source_entity": "MICROSOFT",
    "target_entity": "GPT-4",
    "algorithm": "dijkstra"
  }'
```

### 5. Извлечь 2-hop subgraph вокруг Microsoft

```bash
curl -X POST http://localhost:8000/api/v1/graph/subgraph \
  -H "Content-Type: application/json" \
  -d '{
    "extraction_strategy": "k_hop",
    "seed_entities": ["MICROSOFT"],
    "k": 2,
    "filters": {
      "min_relationship_weight": 0.5
    },
    "output_format": "json"
  }'
```

---

## Следующие шаги

Следующие документы дополняют Graph API:

- **03-vector-api.md** — API для векторных представлений и similarity search
- **04-search-trajectory-api.md** — API траекторий семантического поиска
- **05-attractor-network-api.md** — API аттракторов и концептуальных сетей
