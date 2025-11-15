# Attractor Network API — Аттракторы и концептуальные сети

## Обзор

Attractor Network API предоставляет доступ к анализу аттракторов (центров притяжения) в knowledge graph и концептуальным сетям вокруг запроса пользователя. API позволяет исследовать семантические бассейны, nearest concepts, mutual connections, и related questions.

## Теоретическая основа

### Star-Attractor Patterns

Основано на концепции из **spec/research/02-star-attractor-patterns.md**:

- **Аттрактор** — сущность с высоким graph centrality (degree, PageRank, betweenness)
- **Бассейн аттрактора** — набор сущностей, "притягиваемых" к аттрактору через связи
- **Star topology** — граф где аттрактор в центре, окружен satellite entities
- **Community как бассейн** — community detection выявляет boundaries бассейнов

---

## Core Endpoints

### 1. Attractor Detection

#### POST /api/v1/attractors/detect

Найти аттракторы в графе или подграфе.

**Request Body**:
```json
{
  "scope": "global|community|query_based",

  // For community scope
  "community_ids": ["community_5"],

  // For query_based scope
  "query": "AI research partnerships",
  "context_window": 2,  // k-hop around query results

  "attractor_metrics": {
    "degree_weight": 0.4,
    "pagerank_weight": 0.3,
    "betweenness_weight": 0.2,
    "community_centrality_weight": 0.1
  },

  "min_attractor_score": 0.7,
  "max_attractors": 10
}
```

**Response**:
```json
{
  "scope": "query_based",
  "query": "AI research partnerships",
  "attractors_detected": 5,

  "attractors": [
    {
      "entity": {
        "id": "e1234567...",
        "title": "MICROSOFT",
        "type": "organization",
        "rank": 1
      },
      "attractor_metrics": {
        "degree": 245,
        "degree_percentile": 0.99,
        "pagerank": 0.0123,
        "pagerank_percentile": 0.97,
        "betweenness_centrality": 0.0456,
        "betweenness_percentile": 0.95,
        "community_centrality": 0.89,
        "combined_score": 0.94
      },
      "attractor_properties": {
        "type": "global_hub",  // global_hub, local_hub, bridge
        "influence_radius": 3.2,  // average distance to basin entities
        "basin_size": 67,  // entities in attraction basin
        "basin_diversity": 0.78,  // diversity of entity types in basin
        "star_topology_score": 0.85  // how star-like the local topology
      },
      "basin_entities": [
        {
          "entity_id": "e2345678...",
          "entity_title": "OPENAI",
          "attraction_strength": 0.95,  // relationship weight to attractor
          "distance": 1  // hops from attractor
        },
        {
          "entity_id": "e3456789...",
          "entity_title": "AZURE",
          "attraction_strength": 0.89,
          "distance": 1
        }
      ],
      "communities": ["community_0", "community_5"],
      "role": "Multi-community hub connecting AI research and cloud infrastructure"
    },

    {
      "entity": {
        "id": "e2345678...",
        "title": "OPENAI",
        "type": "organization",
        "rank": 3
      },
      "attractor_metrics": {
        "degree": 187,
        "pagerank": 0.0089,
        "combined_score": 0.87
      },
      "attractor_properties": {
        "type": "local_hub",
        "influence_radius": 2.1,
        "basin_size": 34,
        "basin_diversity": 0.65,
        "star_topology_score": 0.92
      },
      "communities": ["community_5"],
      "role": "Central hub in AI research community"
    }
  ],

  "attractor_network": {
    "total_entities_in_network": 245,
    "entities_in_basins": 198,
    "orphan_entities": 47,
    "average_basin_size": 39.6,
    "basin_overlap": 0.23,  // percentage of entities in multiple basins
    "network_coverage": 0.81  // percentage of entities attracted
  }
}
```

---

### 2. Concept Expansion

#### POST /api/v1/attractors/expand-concepts

Расширить концепты из запроса, найдя nearest concepts и related entities.

**Request Body**:
```json
{
  "query": "AI partnerships",
  "seed_concepts": ["AI", "partnerships"],  // optional, auto-extracted if not provided
  "expansion_strategy": "semantic|graph|hybrid",
  "expansion_depth": 2,
  "max_concepts": 20,
  "filters": {
    "entity_types": ["organization", "concept"],
    "min_relevance": 0.6
  }
}
```

**Response**:
```json
{
  "query": "AI partnerships",
  "seed_concepts": [
    {
      "concept": "AI",
      "type": "technology",
      "mapped_entities": ["ARTIFICIAL INTELLIGENCE", "MACHINE LEARNING", "GPT-4"]
    },
    {
      "concept": "partnerships",
      "type": "relationship_pattern",
      "mapped_entities": []  // abstract concept
    }
  ],

  "expanded_concepts": [
    {
      "concept": "research collaboration",
      "distance_from_seed": 1,
      "expansion_path": ["partnerships", "research collaboration"],
      "relevance_score": 0.89,
      "type": "activity",
      "related_entities": [
        {
          "entity_id": "e1234567...",
          "entity_title": "MICROSOFT RESEARCH",
          "connection_strength": 0.92
        }
      ],
      "evidence": {
        "semantic_similarity": 0.87,
        "graph_proximity": 0.91,
        "co_occurrence_frequency": 45
      }
    },

    {
      "concept": "open source AI",
      "distance_from_seed": 2,
      "expansion_path": ["AI", "research collaboration", "open source AI"],
      "relevance_score": 0.76,
      "type": "technology_approach",
      "related_entities": [
        {"entity_title": "META AI", "connection_strength": 0.88},
        {"entity_title": "HUGGING FACE", "connection_strength": 0.85}
      ]
    },

    {
      "concept": "large language models",
      "distance_from_seed": 1,
      "expansion_path": ["AI", "large language models"],
      "relevance_score": 0.94,
      "type": "technology",
      "related_entities": [
        {"entity_title": "GPT-4", "connection_strength": 0.96},
        {"entity_title": "LLAMA", "connection_strength": 0.89}
      ]
    }
  ],

  "concept_network": {
    "nodes": [
      {"id": "c1", "label": "AI", "type": "seed", "size": 10},
      {"id": "c2", "label": "partnerships", "type": "seed", "size": 10},
      {"id": "c3", "label": "research collaboration", "type": "expanded", "size": 8},
      {"id": "c4", "label": "large language models", "type": "expanded", "size": 9}
    ],
    "edges": [
      {"source": "c1", "target": "c4", "weight": 0.94, "type": "semantic"},
      {"source": "c2", "target": "c3", "weight": 0.89, "type": "semantic"}
    ]
  }
}
```

---

### 3. Mutual Connections Analysis

#### POST /api/v1/attractors/mutual-connections

Найти взаимные связи между сущностями (shared neighbors, bridges).

**Request Body**:
```json
{
  "entity_ids": ["MICROSOFT", "GOOGLE", "META"],
  "connection_types": [
    "shared_neighbors",
    "bridging_entities",
    "common_communities",
    "mutual_relationships"
  ],
  "min_connection_strength": 0.5
}
```

**Response**:
```json
{
  "entities": ["MICROSOFT", "GOOGLE", "META"],

  "shared_neighbors": [
    {
      "neighbor_entity": {
        "id": "e7777777...",
        "title": "ARTIFICIAL INTELLIGENCE",
        "type": "concept"
      },
      "connections": [
        {
          "from_entity": "MICROSOFT",
          "relationship": "RESEARCHES",
          "weight": 0.89
        },
        {
          "from_entity": "GOOGLE",
          "relationship": "DEVELOPS",
          "weight": 0.92
        },
        {
          "from_entity": "META",
          "relationship": "INVESTS_IN",
          "weight": 0.85
        }
      ],
      "connectivity_strength": 0.89,
      "role": "common_interest"
    }
  ],

  "bridging_entities": [
    {
      "bridge_entity": {
        "id": "e8888888...",
        "title": "PYTORCH",
        "type": "technology"
      },
      "bridges_between": ["META", "MICROSOFT"],
      "path": [
        {"from": "META", "to": "PYTORCH", "relation": "CREATED"},
        {"from": "PYTORCH", "to": "MICROSOFT", "relation": "USED_BY"}
      ],
      "bridge_strength": 0.87,
      "betweenness_centrality": 0.045
    }
  ],

  "common_communities": [
    {
      "community_id": "community_5",
      "community_title": "AI Research and Development",
      "entities_in_community": ["MICROSOFT", "GOOGLE", "META"],
      "community_role": {
        "MICROSOFT": "hub",
        "GOOGLE": "central",
        "META": "bridge"
      }
    }
  ],

  "mutual_relationships": [
    {
      "entity_pair": ["MICROSOFT", "META"],
      "relationship_type": "PARTNERED_WITH",
      "relationships": [
        {
          "id": "r9999999...",
          "source": "MICROSOFT",
          "target": "META",
          "description": "Partnership on PyTorch development",
          "weight": 0.78
        },
        {
          "id": "r0000000...",
          "source": "META",
          "target": "MICROSOFT",
          "description": "Collaboration on open-source AI",
          "weight": 0.76
        }
      ],
      "reciprocity": 0.97,  // strength of mutual connection
      "type": "bidirectional"
    }
  ],

  "connection_graph": {
    "nodes": [
      {"id": "MICROSOFT", "type": "query_entity"},
      {"id": "GOOGLE", "type": "query_entity"},
      {"id": "META", "type": "query_entity"},
      {"id": "ARTIFICIAL INTELLIGENCE", "type": "shared_neighbor"},
      {"id": "PYTORCH", "type": "bridge"}
    ],
    "edges": [
      {"source": "MICROSOFT", "target": "ARTIFICIAL INTELLIGENCE", "type": "direct"},
      {"source": "META", "target": "PYTORCH", "type": "created"},
      {"source": "PYTORCH", "target": "MICROSOFT", "type": "used_by"}
    ]
  }
}
```

---

### 4. Related Questions Generation

#### POST /api/v1/attractors/related-questions

Генерировать связанные вопросы на основе текущего запроса и context.

**Request Body**:
```json
{
  "query": "What are Microsoft's AI partnerships?",
  "search_type": "local",
  "num_questions": 5,
  "question_types": [
    "clarification",
    "deeper_dive",
    "related_entities",
    "temporal",
    "comparative"
  ],
  "context": {
    "entities_in_context": ["MICROSOFT", "OPENAI", "META"],
    "communities_in_context": ["community_5"],
    "concepts_identified": ["AI partnerships", "research collaboration"]
  }
}
```

**Response**:
```json
{
  "original_query": "What are Microsoft's AI partnerships?",
  "related_questions": [
    {
      "question": "What specific AI technologies has Microsoft developed through its partnership with OpenAI?",
      "type": "deeper_dive",
      "relevance_score": 0.92,
      "reasoning": "User asked about partnerships, natural follow-up to explore specific outcomes",
      "expected_entities": ["GPT-4", "AZURE OPENAI SERVICE", "COPILOT"],
      "expected_search_type": "local",
      "question_embedding": [0.234, -0.567, ...]
    },

    {
      "question": "How does Microsoft's AI partnership strategy compare to Google's?",
      "type": "comparative",
      "relevance_score": 0.87,
      "reasoning": "Microsoft and Google are competitors in AI space, comparison provides broader context",
      "expected_entities": ["GOOGLE", "DEEPMIND", "BARD"],
      "expected_search_type": "global"
    },

    {
      "question": "When did Microsoft first invest in OpenAI?",
      "type": "temporal",
      "relevance_score": 0.78,
      "reasoning": "Timeline provides historical context for partnership evolution",
      "expected_entities": ["MICROSOFT", "OPENAI"],
      "expected_search_type": "local"
    },

    {
      "question": "What are the key goals of Microsoft's AI research partnerships?",
      "type": "clarification",
      "relevance_score": 0.85,
      "reasoning": "Clarifies strategic intent behind partnerships mentioned",
      "expected_entities": ["MICROSOFT RESEARCH", "AGI"],
      "expected_search_type": "global"
    },

    {
      "question": "Which other companies partner with OpenAI besides Microsoft?",
      "type": "related_entities",
      "relevance_score": 0.81,
      "reasoning": "Explores broader ecosystem around OpenAI",
      "expected_entities": ["OPENAI", "STRIPE", "SHOPIFY"],
      "expected_search_type": "local"
    }
  ],

  "question_graph": {
    "nodes": [
      {"id": "q0", "label": "Original query", "type": "seed"},
      {"id": "q1", "label": "Deeper dive", "type": "related"},
      {"id": "q2", "label": "Comparative", "type": "related"}
    ],
    "edges": [
      {"source": "q0", "target": "q1", "relation": "explores_outcome"},
      {"source": "q0", "target": "q2", "relation": "broadens_context"}
    ]
  }
}
```

---

## Реализация — Mapping к компонентам

| API Feature | Python Component | Key Files |
|-------------|------------------|-----------|
| Attractor Detection | Graph centrality metrics | `graphrag/index/operations/graph_metrics/` (custom implementation)<br/>networkx algorithms |
| Concept Expansion | Entity extraction + Vector search | `graphrag/query/context_builder/entity_extraction.py`<br/>`graphrag/query/context_builder/local_context.py` |
| Mutual Connections | Relationship filtering | `graphrag/query/context_builder/local_context.py:228-313` (_filter_relationships) |
| Related Questions | Question generation LLM | `graphrag/query/question_gen/local_gen.py` |

### Предлагаемый код

```python
# graphrag/api/attractors/detector.py

class AttractorDetector:
    """Detect attractor entities in knowledge graph."""

    def __init__(self, entities: list[Entity], relationships: list[Relationship]):
        self.entities = entities
        self.relationships = relationships
        self.graph = self._build_networkx_graph()

    def detect_attractors(
        self,
        metric_weights: dict,
        min_score: float = 0.7,
        max_attractors: int = 10
    ) -> list[dict]:
        """Detect attractor entities based on centrality metrics."""
        # Compute centrality metrics
        degree_centrality = nx.degree_centrality(self.graph)
        pagerank = nx.pagerank(self.graph)
        betweenness = nx.betweenness_centrality(self.graph)

        # Combine metrics
        scores = {}
        for entity_id in self.graph.nodes():
            scores[entity_id] = (
                metric_weights["degree"] * degree_centrality[entity_id] +
                metric_weights["pagerank"] * pagerank[entity_id] +
                metric_weights["betweenness"] * betweenness[entity_id]
            )

        # Filter and sort
        attractors = [
            (entity_id, score)
            for entity_id, score in scores.items()
            if score >= min_score
        ]
        attractors.sort(key=lambda x: x[1], reverse=True)

        # Return top attractors with basin analysis
        return [
            {
                "entity": self._get_entity(entity_id),
                "attractor_score": score,
                "basin": self._compute_basin(entity_id)
            }
            for entity_id, score in attractors[:max_attractors]
        ]

    def _compute_basin(self, attractor_id: str) -> dict:
        """Compute attraction basin for attractor."""
        # Get neighbors within k hops
        basin_entities = nx.single_source_shortest_path_length(
            self.graph,
            attractor_id,
            cutoff=2
        )

        return {
            "size": len(basin_entities),
            "entities": list(basin_entities.keys())
        }
```

---

## Примеры использования

### 1. Найти аттракторы в AI research community

```bash
curl -X POST http://localhost:8000/api/v1/attractors/detect \
  -H "Content-Type: application/json" \
  -d '{
    "scope": "query_based",
    "query": "AI research partnerships",
    "attractor_metrics": {
      "degree_weight": 0.4,
      "pagerank_weight": 0.4,
      "betweenness_weight": 0.2
    },
    "max_attractors": 5
  }'
```

### 2. Расширить концепты

```bash
curl -X POST http://localhost:8000/api/v1/attractors/expand-concepts \
  -H "Content-Type: application/json" \
  -d '{
    "query": "AI partnerships",
    "expansion_strategy": "hybrid",
    "expansion_depth": 2,
    "max_concepts": 10
  }'
```

### 3. Сгенерировать связанные вопросы

```bash
curl -X POST http://localhost:8000/api/v1/attractors/related-questions \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are Microsoft'"'"'s AI partnerships?",
    "num_questions": 5
  }'
```
