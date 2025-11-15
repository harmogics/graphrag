# Search Trajectory API — Траектории семантического поиска

## Обзор

Search Trajectory API предоставляет детальную информацию о траектории выполнения поискового запроса, включая последовательность шагов, промежуточные результаты, и цепочку рассуждений системы.

## Концепция Search Trajectory

**Search Trajectory** — это упорядоченная последовательность операций и решений, которые система GraphRAG выполняет при обработке запроса:

1. **Query Understanding** — анализ запроса, извлечение ключевых концептов
2. **Entity Mapping** — отображение концептов на сущности в графе
3. **Context Selection** — выбор релевантного контекста (entities, relationships, text units)
4. **Reasoning Steps** — промежуточные шаги рассуждений LLM
5. **Answer Synthesis** — финальная генерация ответа

```
Query → Embedding → Entity Match → Graph Expansion → Context → LLM → Answer
  ↓         ↓            ↓              ↓            ↓       ↓       ↓
  t0       t1           t2             t3           t4      t5      t6
```

---

## Core Endpoints

### 1. Trajectory Recording

#### POST /api/v1/search/trajectory

Выполнить поиск с полной записью траектории.

**Request Body**:
```json
{
  "query": "What are Microsoft's AI partnerships?",
  "search_type": "local|global|drift",
  "record_options": {
    "record_embeddings": true,
    "record_scores": true,
    "record_reasoning": true,
    "record_context_selection": true,
    "record_llm_prompts": true,
    "granularity": "fine|medium|coarse"
  }
}
```

**Response**:
```json
{
  "answer": "Microsoft has several key AI partnerships...",
  "trajectory_id": "traj_550e8400-e29b-41d4-a716-446655440000",
  "trajectory": {
    "stages": [
      {
        "stage_id": 1,
        "stage_name": "query_understanding",
        "timestamp": "2025-11-15T10:23:45.123Z",
        "duration_ms": 45,
        "inputs": {
          "query_text": "What are Microsoft's AI partnerships?"
        },
        "operations": [
          {
            "operation": "query_embedding",
            "model": "text-embedding-ada-002",
            "input": "What are Microsoft's AI partnerships?",
            "output": {
              "embedding": [0.123, -0.456, ...],
              "dimensions": 1536
            },
            "metadata": {
              "tokens": 7,
              "latency_ms": 42
            }
          },
          {
            "operation": "entity_extraction",
            "model": "gpt-4-turbo",
            "input": "What are Microsoft's AI partnerships?",
            "output": {
              "entities": ["Microsoft", "AI", "partnerships"],
              "entity_types": {
                "Microsoft": "organization",
                "AI": "concept",
                "partnerships": "relationship_type"
              }
            },
            "reasoning": "Identified 'Microsoft' as main subject, 'partnerships' as relationship focus"
          }
        ],
        "outputs": {
          "query_embedding": [0.123, -0.456, ...],
          "detected_entities": ["Microsoft"],
          "query_intent": "find_relationships",
          "relationship_focus": "partnerships"
        },
        "decision": {
          "next_stage": "entity_mapping",
          "reason": "Entities detected, proceed to graph mapping"
        }
      },

      {
        "stage_id": 2,
        "stage_name": "entity_mapping",
        "timestamp": "2025-11-15T10:23:45.168Z",
        "duration_ms": 234,
        "inputs": {
          "query_embedding": [0.123, -0.456, ...],
          "detected_entities": ["Microsoft"]
        },
        "operations": [
          {
            "operation": "vector_search",
            "vector_store": "lancedb",
            "query": [0.123, -0.456, ...],
            "top_k": 50,
            "output": {
              "candidates": [
                {
                  "entity_id": "e1234567...",
                  "entity_title": "MICROSOFT",
                  "similarity": 0.94,
                  "rank": 1
                },
                {
                  "entity_id": "e2345678...",
                  "entity_title": "MICROSOFT AZURE",
                  "similarity": 0.76,
                  "rank": 12
                }
              ]
            },
            "metadata": {
              "total_candidates": 50,
              "ann_recall": 0.98,
              "search_time_ms": 12
            }
          },
          {
            "operation": "entity_filtering",
            "criteria": {
              "min_similarity": 0.7,
              "max_rank": 100
            },
            "input_count": 50,
            "output_count": 12,
            "filtered_entities": ["e1234567...", "e2345678...", ...]
          }
        ],
        "outputs": {
          "selected_entities": [
            {"id": "e1234567...", "title": "MICROSOFT", "selection_score": 0.94}
          ],
          "rejected_entities": [
            {"id": "e9999999...", "title": "MICROSOFT OFFICE", "rejection_reason": "low_relevance"}
          ]
        },
        "decision": {
          "next_stage": "graph_expansion",
          "reason": "Found seed entities, expand to find relationships"
        }
      },

      {
        "stage_id": 3,
        "stage_name": "graph_expansion",
        "timestamp": "2025-11-15T10:23:45.402Z",
        "duration_ms": 156,
        "inputs": {
          "seed_entities": ["MICROSOFT"]
        },
        "operations": [
          {
            "operation": "relationship_retrieval",
            "seed_entity": "MICROSOFT",
            "expansion_strategy": "outgoing_relationships",
            "filters": {
              "relationship_types": null,
              "min_weight": 0.5
            },
            "output": {
              "relationships_found": 15,
              "relationships": [
                {
                  "id": "r1234567...",
                  "source": "MICROSOFT",
                  "target": "OPENAI",
                  "type": "FOUNDED",
                  "weight": 0.95,
                  "relevance_score": 0.89
                }
              ]
            }
          },
          {
            "operation": "relationship_ranking",
            "input_count": 15,
            "ranking_criteria": {
              "weight": 0.6,
              "mutual_connections": 0.3,
              "entity_rank": 0.1
            },
            "output": {
              "top_relationships": [
                {"id": "r1234567...", "combined_score": 0.92}
              ]
            }
          }
        ],
        "outputs": {
          "expanded_entities": ["OPENAI", "AZURE", "META AI"],
          "selected_relationships": 8,
          "graph_metrics": {
            "subgraph_size": 12,
            "avg_relationship_weight": 0.78
          }
        },
        "decision": {
          "next_stage": "context_building",
          "reason": "Sufficient relationships found, proceed to context assembly"
        }
      },

      {
        "stage_id": 4,
        "stage_name": "context_building",
        "timestamp": "2025-11-15T10:23:45.558Z",
        "duration_ms": 133,
        "inputs": {
          "entities": 12,
          "relationships": 8,
          "text_units": 5
        },
        "operations": [
          {
            "operation": "context_assembly",
            "max_tokens": 8000,
            "priority_order": ["entities", "relationships", "text_units"],
            "output": {
              "context_chunks": {
                "entities": {
                  "count": 12,
                  "tokens": 2134,
                  "content_preview": "id|entity|description|rank\n0|MICROSOFT|..."
                },
                "relationships": {
                  "count": 8,
                  "tokens": 3201,
                  "content_preview": "id|source|target|description\n..."
                },
                "text_units": {
                  "count": 5,
                  "tokens": 1208,
                  "content_preview": "Microsoft announced partnership..."
                }
              },
              "total_tokens": 6543
            }
          }
        ],
        "outputs": {
          "context_text": "-----Entities-----\nid|entity|...",
          "context_tokens": 6543,
          "context_completeness": 0.87
        },
        "decision": {
          "next_stage": "llm_reasoning",
          "reason": "Context assembled within token limits"
        }
      },

      {
        "stage_id": 5,
        "stage_name": "llm_reasoning",
        "timestamp": "2025-11-15T10:23:45.691Z",
        "duration_ms": 679,
        "inputs": {
          "query": "What are Microsoft's AI partnerships?",
          "context": "-----Entities-----\n..."
        },
        "operations": [
          {
            "operation": "llm_call",
            "model": "gpt-4-turbo",
            "temperature": 0.0,
            "system_prompt": {
              "template": "LOCAL_SEARCH_SYSTEM_PROMPT",
              "rendered": "You are a helpful assistant...\n\n-----Entities-----\n...",
              "tokens": 6789
            },
            "user_prompt": {
              "text": "What are Microsoft's AI partnerships?",
              "tokens": 7
            },
            "output": {
              "response": "Microsoft has several key AI partnerships...",
              "tokens": 342,
              "finish_reason": "stop"
            },
            "reasoning_trace": {
              "key_facts_identified": [
                "Microsoft founded OpenAI in 2015",
                "Partnership with Meta on OPT model",
                "Azure hosts OpenAI models"
              ],
              "synthesis_strategy": "chronological_then_thematic",
              "confidence": 0.92
            }
          }
        ],
        "outputs": {
          "answer": "Microsoft has several key AI partnerships...",
          "answer_tokens": 342,
          "sources_used": ["MICROSOFT", "OPENAI", "relationship:FOUNDED"]
        },
        "decision": {
          "next_stage": "complete",
          "reason": "Answer generated successfully"
        }
      }
    ],

    "trajectory_summary": {
      "total_stages": 5,
      "total_duration_ms": 1247,
      "operations_count": 9,
      "llm_calls": 2,
      "vector_searches": 1,
      "graph_traversals": 1,
      "decision_points": 5
    },

    "semantic_path": {
      "query_concepts": ["Microsoft", "AI", "partnerships"],
      "activated_entities": ["MICROSOFT", "OPENAI", "AZURE", "META AI"],
      "activated_relationships": [
        "MICROSOFT → FOUNDED → OPENAI",
        "MICROSOFT → PARTNERED_WITH → META"
      ],
      "activated_communities": ["community_5"],
      "concept_drift": 0.12,
      "path_coherence": 0.89
    }
  }
}
```

---

### 2. Trajectory Retrieval

#### GET /api/v1/search/trajectory/{trajectory_id}

Получить сохраненную траекторию поиска.

**Query Parameters**:
```
include_embeddings: boolean (default: false)
include_full_prompts: boolean (default: true)
format: enum[json, visualization] (default: json)
```

---

### 3. Trajectory Comparison

#### POST /api/v1/search/trajectory/compare

Сравнить две траектории поиска.

**Request Body**:
```json
{
  "trajectory_id_1": "traj_550e8400...",
  "trajectory_id_2": "traj_660f9511...",
  "comparison_aspects": [
    "stages_sequence",
    "entity_selection",
    "context_composition",
    "reasoning_paths",
    "performance"
  ]
}
```

**Response**:
```json
{
  "comparison": {
    "stages_comparison": {
      "common_stages": ["query_understanding", "entity_mapping", "llm_reasoning"],
      "unique_to_traj1": ["graph_expansion"],
      "unique_to_traj2": ["community_selection"],
      "stage_order_similarity": 0.78
    },
    "entity_selection_comparison": {
      "common_entities": ["MICROSOFT", "OPENAI"],
      "unique_to_traj1": ["AZURE"],
      "unique_to_traj2": ["GOOGLE"],
      "jaccard_similarity": 0.67
    },
    "performance_comparison": {
      "traj1_duration_ms": 1247,
      "traj2_duration_ms": 1589,
      "speedup_factor": 1.27,
      "traj1_tokens": 7138,
      "traj2_tokens": 8945,
      "cost_difference_usd": 0.0045
    }
  }
}
```

---

### 4. Trajectory Visualization

#### GET /api/v1/search/trajectory/{trajectory_id}/visualize

Получить визуализацию траектории в формате Mermaid/D3.

**Response (Mermaid)**:
```
graph TD
    A[Query: 'Microsoft AI partnerships'] --> B[Query Embedding<br/>45ms]
    B --> C[Vector Search<br/>50 candidates]
    C --> D{Entity Filter<br/>12 selected}
    D --> E[Graph Expansion<br/>15 relationships]
    E --> F[Context Building<br/>6543 tokens]
    F --> G[LLM Reasoning<br/>gpt-4-turbo]
    G --> H[Answer Generated]

    style A fill:#e1f5ff
    style D fill:#fff4e6
    style G fill:#ffe7e7
    style H fill:#e7ffe7
```

---

## Реализация — Mapping к компонентам

| Feature | Python Component | Implementation |
|---------|------------------|----------------|
| Trajectory Recording | `LocalSearch.search()` | Add `TrajectoryRecorder` middleware |
| Stage Tracking | `LocalContextBuilder.build_context()` | Emit stage events to recorder |
| Decision Logging | Context filtering logic | Log filter criteria and results |
| LLM Reasoning Trace | `model.achat()` | Capture prompts, responses, metadata |

### Предлагаемый код

```python
# graphrag/query/trajectory/recorder.py

class TrajectoryRecorder:
    """Records search trajectory for introspection."""

    def __init__(self):
        self.stages = []
        self.current_stage = None

    def start_stage(self, name: str, inputs: dict):
        """Start a new stage."""
        self.current_stage = {
            "stage_id": len(self.stages) + 1,
            "stage_name": name,
            "timestamp": datetime.utcnow().isoformat(),
            "start_time": time.time(),
            "inputs": inputs,
            "operations": [],
            "outputs": {},
            "decision": {}
        }

    def record_operation(self, operation: str, **kwargs):
        """Record an operation within current stage."""
        if not self.current_stage:
            raise ValueError("No active stage")

        self.current_stage["operations"].append({
            "operation": operation,
            **kwargs
        })

    def end_stage(self, outputs: dict, decision: dict):
        """End current stage."""
        if not self.current_stage:
            raise ValueError("No active stage")

        self.current_stage["duration_ms"] = int((time.time() - self.current_stage["start_time"]) * 1000)
        self.current_stage["outputs"] = outputs
        self.current_stage["decision"] = decision

        self.stages.append(self.current_stage)
        self.current_stage = None

    def get_trajectory(self) -> dict:
        """Get full trajectory."""
        return {
            "stages": self.stages,
            "trajectory_summary": self._compute_summary()
        }

    def _compute_summary(self) -> dict:
        """Compute trajectory summary statistics."""
        return {
            "total_stages": len(self.stages),
            "total_duration_ms": sum(s["duration_ms"] for s in self.stages),
            "operations_count": sum(len(s["operations"]) for s in self.stages)
        }
```

---

## Примеры использования

### 1. Выполнить поиск с траекторией

```bash
curl -X POST http://localhost:8000/api/v1/search/trajectory \
  -H "Content-Type: application/json" \
  -d '{
    "query": "Microsoft AI partnerships",
    "search_type": "local",
    "record_options": {
      "granularity": "fine",
      "record_reasoning": true
    }
  }'
```

### 2. Получить траекторию

```bash
curl "http://localhost:8000/api/v1/search/trajectory/traj_550e8400?include_full_prompts=true"
```

### 3. Визуализировать траекторию

```bash
curl "http://localhost:8000/api/v1/search/trajectory/traj_550e8400/visualize?format=mermaid"
```
