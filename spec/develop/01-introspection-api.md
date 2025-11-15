# GraphRAG Introspection API

## Обзор

Данный документ описывает архитектуру **Introspection API** — набора REST endpoints, предназначенных для максимального открытия внутренних механизмов системы GraphRAG для внешних пользователей. API позволяет наблюдать и анализировать полную траекторию выполнения запроса, включая промежуточные состояния, цепочки рассуждений LLM, scoring механизмы и процессы построения контекста.

## Философия проектирования

### Принцип максимальной прозрачности

GraphRAG Introspection API следует принципу **"Glass Box"** (стеклянная коробка), где каждый этап обработки запроса доступен для наблюдения и анализа:

1. **Полная трассировка** — каждый шаг выполнения запроса логируется с timestamp, входными/выходными данными
2. **Промежуточные артефакты** — все промежуточные результаты (embeddings, scores, context chunks) доступны через API
3. **Reasoning transparency** — цепочки рассуждений LLM (prompts, responses, token usage) видимы в деталях
4. **Reproducibility** — достаточно данных для воспроизведения результатов запроса

### Целевые use cases

**Для разработчиков**:
- Отладка и оптимизация поисковых запросов
- Понимание причин получения конкретных результатов
- Тюнинг параметров системы (top_k, temperature, context size)
- Анализ производительности и стоимости (token usage, latency)

**Для исследователей**:
- Изучение семантических траекторий в knowledge graph
- Анализ качества контекста и его влияния на ответы
- Исследование attractor patterns и community structure
- Экспериментальная проверка гипотез о reasoning цепочках

**Для продуктовых команд**:
- Объяснение результатов пользователям (explainable AI)
- Мониторинг качества ответов в production
- A/B тестирование различных стратегий поиска
- Аналитика использования knowledge graph

---

## Архитектура Introspection API

### Общая схема

```
┌──────────────────────────────────────────────────────────────────────┐
│                    INTROSPECTION API ARCHITECTURE                     │
└──────────────────────────────────────────────────────────────────────┘

CLIENT REQUEST
   │
   ▼
┌────────────────────────────────────────┐
│  POST /api/v1/search                   │  ← Standard search endpoint
│  ?introspection=full                   │     with introspection flag
└────────────────────────────────────────┘
   │
   ├──────────────── EXECUTION FLOW ────────────────┐
   │                                                 │
   ▼                                                 ▼
┌─────────────────┐                         ┌──────────────┐
│ Query Analysis  │ ───────────────────────▶│ Trace Store  │
│ • Embeddings    │                         │ (Redis/DB)   │
│ • Entity extract│                         └──────────────┘
└─────────────────┘                                 │
   │                                                 │
   ▼                                                 │
┌─────────────────┐                                 │
│ Context Build   │ ───────────────────────────────▶│
│ • Vector search │                                 │
│ • Graph traversal                                 │
└─────────────────┘                                 │
   │                                                 │
   ▼                                                 │
┌─────────────────┐                                 │
│ LLM Reasoning   │ ───────────────────────────────▶│
│ • Map phase     │                                 │
│ • Reduce phase  │                                 │
└─────────────────┘                                 │
   │                                                 │
   ▼                                                 │
┌────────────────────────────────────────┐          │
│  RESPONSE                              │          │
│  {                                     │          │
│    "answer": "...",                   │          │
│    "introspection": {                 │◀─────────┘
│      "trace_id": "uuid",             │
│      "execution_graph": {...},       │
│      "intermediate_results": [...],  │
│      "reasoning_chain": [...],       │
│      "context_metadata": {...}       │
│    }                                  │
│  }                                    │
└────────────────────────────────────────┘
```

### Trace ID и сессии

Каждый запрос с включенной introspection получает **trace_id** (UUID v4), который:
- Связывает все промежуточные артефакты одного запроса
- Позволяет асинхронно запрашивать детали через отдельные endpoints
- Сохраняется в trace store с TTL (по умолчанию 24 часа)

---

## Core Introspection Endpoints

### 1. Search with Introspection

**`POST /api/v1/search`**

Основной endpoint поиска с расширенной introspection информацией.

**Query Parameters**:
```
introspection: enum[none, basic, full, streaming]
  - none: стандартный ответ без introspection (по умолчанию)
  - basic: summary метрики (latency, token counts, score)
  - full: полная трассировка с промежуточными результатами
  - streaming: потоковая передача introspection events

include_context: boolean (default: false)
  - true: включить полный context chunks в ответ

include_reasoning: boolean (default: false)
  - true: включить LLM prompts и responses

include_scores: boolean (default: false)
  - true: включить детальные scores для всех entities/relationships
```

**Request Body**:
```json
{
  "query": "What are Microsoft's AI partnerships?",
  "search_type": "local|global|drift",
  "conversation_history": [
    {"role": "user", "content": "previous question"},
    {"role": "assistant", "content": "previous answer"}
  ],
  "introspection_config": {
    "include_embeddings": false,
    "include_intermediate_answers": true,
    "include_context_selection_process": true,
    "include_llm_reasoning": true,
    "max_trace_depth": 10
  }
}
```

**Response (introspection=full)**:
```json
{
  "answer": "Microsoft has several key AI partnerships including OpenAI (founded in 2015...",

  "metadata": {
    "trace_id": "550e8400-e29b-41d4-a716-446655440000",
    "search_type": "local",
    "total_time_ms": 1247,
    "llm_calls": 2,
    "total_tokens": {
      "prompt": 1523,
      "completion": 342,
      "total": 1865
    },
    "cost_usd": 0.0187
  },

  "introspection": {
    "execution_stages": [
      {
        "stage": "query_analysis",
        "timestamp": "2025-11-14T10:23:45.123Z",
        "duration_ms": 45,
        "details": {
          "query_embedding": {
            "model": "text-embedding-ada-002",
            "dimensions": 1536,
            "vector_preview": [0.123, -0.456, ...], // first 10 dims
            "norm": 1.0
          },
          "detected_entities": [
            {
              "text": "Microsoft",
              "type": "organization",
              "confidence": 0.98,
              "source": "entity_extraction_llm"
            },
            {
              "text": "AI partnerships",
              "type": "concept",
              "confidence": 0.85,
              "source": "keyword_analysis"
            }
          ]
        }
      },

      {
        "stage": "context_building",
        "timestamp": "2025-11-14T10:23:45.168Z",
        "duration_ms": 523,
        "details": {
          "vector_search": {
            "query_vector_preview": [0.123, -0.456, ...],
            "candidates_retrieved": 50,
            "candidates_filtered": 12,
            "filtering_criteria": {
              "min_similarity": 0.7,
              "max_rank_threshold": 100
            },
            "top_matches": [
              {
                "entity_id": "MICROSOFT",
                "entity_title": "MICROSOFT",
                "similarity": 0.94,
                "rank": 1,
                "community_ids": ["community_0", "community_5"],
                "selected": true,
                "selection_reason": "high_similarity_and_rank"
              },
              {
                "entity_id": "OPENAI",
                "entity_title": "OPENAI",
                "similarity": 0.89,
                "rank": 3,
                "community_ids": ["community_5"],
                "selected": true,
                "selection_reason": "related_entity_in_network"
              }
            ]
          },

          "graph_traversal": {
            "seed_entities": ["MICROSOFT"],
            "traversal_strategy": "relationship_expansion",
            "max_hops": 2,
            "relationships_found": 15,
            "relationships_selected": 8,
            "selection_criteria": {
              "ranking_attribute": "weight",
              "top_k_per_entity": 10,
              "mutual_relationships_prioritized": true
            },
            "traversal_graph": {
              "nodes": [
                {"id": "MICROSOFT", "type": "entity", "level": 0},
                {"id": "OPENAI", "type": "entity", "level": 1},
                {"id": "AZURE", "type": "entity", "level": 1}
              ],
              "edges": [
                {
                  "source": "MICROSOFT",
                  "target": "OPENAI",
                  "type": "FOUNDED",
                  "weight": 0.95,
                  "description": "Microsoft founded OpenAI in 2015"
                }
              ]
            }
          },

          "context_assembly": {
            "max_tokens": 8000,
            "actual_tokens": 6543,
            "token_usage_breakdown": {
              "entities": 2134,
              "relationships": 3201,
              "text_units": 1208,
              "covariates": 0
            },
            "context_chunks": [
              {
                "type": "entities",
                "count": 12,
                "token_count": 2134,
                "sample": "id|entity|description|rank\n0|MICROSOFT|..."
              },
              {
                "type": "relationships",
                "count": 8,
                "token_count": 3201,
                "sample": "id|source|target|description|weight\n..."
              }
            ]
          }
        }
      },

      {
        "stage": "llm_reasoning",
        "timestamp": "2025-11-14T10:23:45.691Z",
        "duration_ms": 679,
        "details": {
          "model": "gpt-4-turbo",
          "temperature": 0.0,
          "max_tokens": 1500,
          "system_prompt": {
            "template": "LOCAL_SEARCH_SYSTEM_PROMPT",
            "variables": {
              "context_data": "<full context chunks>",
              "response_type": "multiple paragraphs"
            },
            "rendered_preview": "You are a helpful assistant...\n\n-----Entities-----\nid|entity|description...",
            "token_count": 6789
          },
          "user_prompt": {
            "text": "What are Microsoft's AI partnerships?",
            "token_count": 7
          },
          "response": {
            "content": "Microsoft has several key AI partnerships...",
            "token_count": 342,
            "finish_reason": "stop",
            "model_used": "gpt-4-turbo-2024-04-09"
          },
          "token_usage": {
            "prompt_tokens": 6796,
            "completion_tokens": 342,
            "total_tokens": 7138
          }
        }
      }
    ],

    "context_metadata": {
      "entities": [
        {
          "id": "MICROSOFT",
          "title": "MICROSOFT",
          "type": "organization",
          "description": "Microsoft Corporation is a technology company...",
          "rank": 1,
          "similarity_to_query": 0.94,
          "community_ids": ["community_0", "community_5"],
          "text_unit_ids": ["unit_123", "unit_456"],
          "attributes": {
            "degree": 245,
            "pagerank": 0.0123
          }
        }
      ],
      "relationships": [
        {
          "id": "rel_001",
          "source": "MICROSOFT",
          "target": "OPENAI",
          "description": "Microsoft founded OpenAI in 2015",
          "weight": 0.95,
          "rank": 1,
          "attributes": {
            "type": "FOUNDED",
            "year": 2015
          }
        }
      ],
      "text_units": [
        {
          "id": "unit_123",
          "text": "Microsoft announced a partnership with OpenAI...",
          "similarity_to_query": 0.87,
          "source_document_ids": ["doc_45"],
          "entities_mentioned": ["MICROSOFT", "OPENAI"]
        }
      ]
    },

    "reasoning_chain": {
      "steps": [
        {
          "step": 1,
          "action": "entity_identification",
          "input": "What are Microsoft's AI partnerships?",
          "output": ["MICROSOFT", "AI partnerships concept"],
          "reasoning": "Query explicitly mentions Microsoft, implicit concept of partnerships"
        },
        {
          "step": 2,
          "action": "context_retrieval",
          "input": ["MICROSOFT"],
          "output": ["12 entities", "8 relationships", "5 text units"],
          "reasoning": "Retrieved entities related to Microsoft within 2-hop neighborhood"
        },
        {
          "step": 3,
          "action": "answer_synthesis",
          "input": "Context chunks + Query",
          "output": "Final answer",
          "reasoning": "LLM synthesized answer from provided context focusing on partnership relationships"
        }
      ]
    },

    "performance_breakdown": {
      "query_analysis": 45,
      "vector_search": 234,
      "graph_traversal": 156,
      "context_assembly": 133,
      "llm_call": 679,
      "total": 1247
    }
  },

  "context_data": {
    "entities_used": 12,
    "relationships_used": 8,
    "text_units_used": 5,
    "covariates_used": 0,
    "total_context_tokens": 6543
  }
}
```

---

### 2. Trace Details Endpoint

**`GET /api/v1/trace/{trace_id}`**

Получить полную трассировку выполнения запроса по trace_id.

**Response**:
```json
{
  "trace_id": "550e8400-e29b-41d4-a716-446655440000",
  "created_at": "2025-11-14T10:23:45.123Z",
  "ttl_expires_at": "2025-11-15T10:23:45.123Z",
  "query": {
    "text": "What are Microsoft's AI partnerships?",
    "search_type": "local",
    "parameters": {...}
  },
  "execution_graph": {
    // Directed acyclic graph (DAG) of execution steps
    "nodes": [
      {
        "id": "step_1",
        "type": "query_embedding",
        "status": "completed",
        "start_time": "2025-11-14T10:23:45.123Z",
        "end_time": "2025-11-14T10:23:45.168Z",
        "duration_ms": 45,
        "inputs": ["query_text"],
        "outputs": ["query_embedding"],
        "metadata": {...}
      },
      {
        "id": "step_2",
        "type": "vector_search",
        "status": "completed",
        "start_time": "2025-11-14T10:23:45.168Z",
        "end_time": "2025-11-14T10:23:45.402Z",
        "duration_ms": 234,
        "inputs": ["query_embedding"],
        "outputs": ["candidate_entities"],
        "metadata": {...}
      }
    ],
    "edges": [
      {"from": "step_1", "to": "step_2", "data_flow": "query_embedding"}
    ]
  },
  "artifacts": {
    "query_embedding": {
      "artifact_id": "art_001",
      "type": "embedding",
      "size_bytes": 6144,
      "preview": [0.123, -0.456, ...],
      "download_url": "/api/v1/trace/{trace_id}/artifact/art_001"
    },
    "context_chunks": {
      "artifact_id": "art_002",
      "type": "text",
      "size_bytes": 25600,
      "preview": "-----Entities-----\nid|entity|...",
      "download_url": "/api/v1/trace/{trace_id}/artifact/art_002"
    }
  },
  "statistics": {
    "total_duration_ms": 1247,
    "llm_calls": 2,
    "vector_searches": 1,
    "graph_traversals": 1,
    "total_tokens": 7138,
    "total_cost_usd": 0.0187
  }
}
```

---

### 3. Context Explainability Endpoint

**`GET /api/v1/trace/{trace_id}/context/explain`**

Детальное объяснение того, почему конкретные сущности/связи были включены в контекст.

**Query Parameters**:
```
entity_id: string (optional) - explain specific entity selection
relationship_id: string (optional) - explain specific relationship selection
```

**Response**:
```json
{
  "trace_id": "550e8400-e29b-41d4-a716-446655440000",
  "context_selection_explanation": {
    "entities": [
      {
        "entity_id": "MICROSOFT",
        "entity_title": "MICROSOFT",
        "selected": true,
        "selection_reasoning": {
          "primary_reason": "high_similarity_to_query",
          "contributing_factors": [
            {
              "factor": "vector_similarity",
              "value": 0.94,
              "weight": 0.6,
              "contribution": 0.564
            },
            {
              "factor": "entity_rank",
              "value": 1,
              "normalized": 1.0,
              "weight": 0.3,
              "contribution": 0.3
            },
            {
              "factor": "community_relevance",
              "value": 0.85,
              "weight": 0.1,
              "contribution": 0.085
            }
          ],
          "total_score": 0.949,
          "selection_threshold": 0.7,
          "decision": "SELECTED"
        },
        "alternative_entities_considered": [
          {
            "entity_id": "GOOGLE",
            "entity_title": "GOOGLE",
            "score": 0.67,
            "not_selected_reason": "below_threshold"
          }
        ]
      }
    ],
    "relationships": [
      {
        "relationship_id": "rel_001",
        "source": "MICROSOFT",
        "target": "OPENAI",
        "selected": true,
        "selection_reasoning": {
          "primary_reason": "in_network_relationship",
          "contributing_factors": [
            {
              "factor": "both_entities_selected",
              "value": true,
              "priority": "highest"
            },
            {
              "factor": "relationship_weight",
              "value": 0.95,
              "rank": 1
            },
            {
              "factor": "relationship_type",
              "value": "FOUNDED",
              "relevance": "high"
            }
          ],
          "decision": "SELECTED",
          "priority": "in_network"
        }
      },
      {
        "relationship_id": "rel_045",
        "source": "MICROSOFT",
        "target": "APPLE",
        "selected": false,
        "selection_reasoning": {
          "primary_reason": "context_token_limit_exceeded",
          "contributing_factors": [
            {
              "factor": "relationship_weight",
              "value": 0.42,
              "rank": 15
            },
            {
              "factor": "available_tokens",
              "value": 0,
              "limit_reached": true
            }
          ],
          "decision": "NOT_SELECTED"
        }
      }
    ],
    "text_units": [
      {
        "text_unit_id": "unit_123",
        "selected": true,
        "selection_reasoning": {
          "primary_reason": "contains_selected_entities",
          "entities_mentioned": ["MICROSOFT", "OPENAI"],
          "similarity_to_query": 0.87,
          "decision": "SELECTED"
        }
      }
    ]
  },
  "context_constraints": {
    "max_tokens": 8000,
    "actual_tokens_used": 6543,
    "remaining_tokens": 1457,
    "entities_limit": null,
    "relationships_limit": null,
    "text_units_limit": null
  }
}
```

---

### 4. Reasoning Chain Endpoint

**`GET /api/v1/trace/{trace_id}/reasoning`**

Получить детальную цепочку рассуждений LLM, включая все промежуточные prompts и responses.

**Response**:
```json
{
  "trace_id": "550e8400-e29b-41d4-a716-446655440000",
  "search_type": "global",
  "reasoning_phases": [
    {
      "phase": "map",
      "description": "Parallel LLM calls on community summaries",
      "calls": [
        {
          "call_id": "map_call_1",
          "community_id": "community_5",
          "model": "gpt-4-turbo",
          "temperature": 0.0,
          "system_prompt": {
            "template": "MAP_SYSTEM_PROMPT",
            "rendered": "You are an analyst reviewing data about community 5...\n\n-----Reports-----\nCommunity Summary: This community contains...",
            "token_count": 1234
          },
          "user_prompt": {
            "text": "What are Microsoft's AI partnerships?",
            "token_count": 7
          },
          "response": {
            "content": "{\"points\": [{\"description\": \"Microsoft founded OpenAI in 2015\", \"score\": 85}, {\"description\": \"Partnership with Meta on OPT model\", \"score\": 70}]}",
            "parsed": {
              "points": [
                {
                  "description": "Microsoft founded OpenAI in 2015",
                  "score": 85,
                  "analyst": 1
                },
                {
                  "description": "Partnership with Meta on OPT model",
                  "score": 70,
                  "analyst": 1
                }
              ]
            },
            "token_count": 67
          },
          "token_usage": {
            "prompt_tokens": 1241,
            "completion_tokens": 67,
            "total_tokens": 1308
          },
          "latency_ms": 456
        },
        {
          "call_id": "map_call_2",
          "community_id": "community_12",
          "model": "gpt-4-turbo",
          // ... similar structure
        }
      ],
      "aggregated_results": {
        "total_points": 15,
        "total_calls": 8,
        "filtered_points": 12,
        "filter_criteria": "score > 0",
        "sorted_by": "score_desc"
      }
    },
    {
      "phase": "reduce",
      "description": "Combine map responses into final answer",
      "calls": [
        {
          "call_id": "reduce_call_1",
          "model": "gpt-4-turbo",
          "temperature": 0.0,
          "system_prompt": {
            "template": "REDUCE_SYSTEM_PROMPT",
            "variables": {
              "report_data": "----Analyst 1----\nImportance Score: 85\nMicrosoft founded OpenAI...",
              "response_type": "multiple paragraphs"
            },
            "rendered": "You are a helpful assistant synthesizing information...",
            "token_count": 3456
          },
          "user_prompt": {
            "text": "What are Microsoft's AI partnerships?",
            "token_count": 7
          },
          "response": {
            "content": "Microsoft has established several significant AI partnerships...",
            "token_count": 342
          },
          "token_usage": {
            "prompt_tokens": 3463,
            "completion_tokens": 342,
            "total_tokens": 3805
          },
          "latency_ms": 679
        }
      ]
    }
  ],
  "total_llm_statistics": {
    "total_calls": 9,
    "map_calls": 8,
    "reduce_calls": 1,
    "total_tokens": 12456,
    "total_cost_usd": 0.0342,
    "total_latency_ms": 4123,
    "average_latency_per_call_ms": 458
  }
}
```

---

### 5. Execution Graph Endpoint

**`GET /api/v1/trace/{trace_id}/execution-graph`**

Получить граф выполнения запроса в формате, пригодном для визуализации.

**Query Parameters**:
```
format: enum[json, dot, mermaid]
  - json: структурированный JSON (по умолчанию)
  - dot: GraphViz DOT format
  - mermaid: Mermaid diagram syntax
```

**Response (format=json)**:
```json
{
  "trace_id": "550e8400-e29b-41d4-a716-446655440000",
  "graph_type": "execution_dag",
  "nodes": [
    {
      "id": "start",
      "type": "entry_point",
      "label": "Query Received",
      "metadata": {
        "query": "What are Microsoft's AI partnerships?",
        "timestamp": "2025-11-14T10:23:45.123Z"
      }
    },
    {
      "id": "embed_query",
      "type": "transform",
      "label": "Embed Query",
      "metadata": {
        "model": "text-embedding-ada-002",
        "duration_ms": 45,
        "output_dimensions": 1536
      }
    },
    {
      "id": "vector_search",
      "type": "retrieval",
      "label": "Vector Search (Entities)",
      "metadata": {
        "candidates": 50,
        "selected": 12,
        "duration_ms": 234
      }
    },
    {
      "id": "graph_traversal",
      "type": "retrieval",
      "label": "Graph Traversal",
      "metadata": {
        "seed_entities": 12,
        "relationships_found": 15,
        "duration_ms": 156
      }
    },
    {
      "id": "build_context",
      "type": "transform",
      "label": "Build Context",
      "metadata": {
        "context_tokens": 6543,
        "duration_ms": 133
      }
    },
    {
      "id": "llm_call",
      "type": "llm",
      "label": "LLM Answer Generation",
      "metadata": {
        "model": "gpt-4-turbo",
        "tokens": 7138,
        "duration_ms": 679
      }
    },
    {
      "id": "end",
      "type": "exit_point",
      "label": "Response Returned",
      "metadata": {
        "total_duration_ms": 1247
      }
    }
  ],
  "edges": [
    {"from": "start", "to": "embed_query", "label": "query_text"},
    {"from": "embed_query", "to": "vector_search", "label": "query_embedding"},
    {"from": "vector_search", "to": "graph_traversal", "label": "seed_entities"},
    {"from": "graph_traversal", "to": "build_context", "label": "entities+relationships"},
    {"from": "build_context", "to": "llm_call", "label": "context_chunks"},
    {"from": "llm_call", "to": "end", "label": "answer"}
  ],
  "metadata": {
    "total_nodes": 7,
    "total_edges": 6,
    "critical_path": ["start", "embed_query", "vector_search", "graph_traversal", "build_context", "llm_call", "end"],
    "critical_path_duration_ms": 1247
  }
}
```

**Response (format=mermaid)**:
```
graph TD
    start[Query Received] --> embed_query[Embed Query<br/>45ms]
    embed_query --> vector_search[Vector Search<br/>234ms<br/>50→12 entities]
    vector_search --> graph_traversal[Graph Traversal<br/>156ms<br/>15 relationships]
    graph_traversal --> build_context[Build Context<br/>133ms<br/>6543 tokens]
    build_context --> llm_call[LLM Call<br/>679ms<br/>gpt-4-turbo]
    llm_call --> end[Response<br/>Total: 1247ms]
```

---

## Реализация на основе текущей архитектуры

### Точки интеграции в код

#### 1. LocalSearch (graphrag/query/structured_search/local_search/search.py)

**Текущий код**:
```python:graphrag/query/structured_search/local_search/search.py
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
        context_text=context_result.context_chunks,
        completion_time=time.time() - start_time,
        # ...
    )
```

**Предлагаемые изменения**:
```python
async def search(
    self,
    query: str,
    conversation_history: ConversationHistory | None = None,
    introspection: IntrospectionLevel = IntrospectionLevel.NONE,
    **kwargs,
) -> SearchResult:
    # Initialize trace if introspection enabled
    trace = Trace(query=query) if introspection != IntrospectionLevel.NONE else None

    start_time = time.time()

    # Trace: Query analysis
    if trace:
        query_embedding = await self._embed_query(query)
        trace.add_stage(
            stage="query_analysis",
            duration_ms=...,
            details={
                "query_embedding": query_embedding,
                "detected_entities": ...
            }
        )

    # Trace: Context building
    context_result = self.context_builder.build_context(
        query=query,
        conversation_history=conversation_history,
        trace=trace,  # Pass trace down
        **kwargs,
        **self.context_builder_params,
    )

    # Trace: LLM reasoning
    if trace:
        trace.add_stage(
            stage="llm_reasoning",
            duration_ms=...,
            details={
                "system_prompt": search_prompt,
                "user_prompt": query,
                "response": full_response,
                "token_usage": {...}
            }
        )

    result = SearchResult(
        response=full_response,
        context_data=context_result.context_records,
        context_text=context_result.context_chunks,
        completion_time=time.time() - start_time,
        introspection=trace.serialize() if trace else None
    )

    # Store trace for later retrieval
    if trace:
        await self.trace_store.save(trace)

    return result
```

#### 2. LocalContextBuilder (graphrag/query/context_builder/local_context.py)

**Добавить трассировку в build_entity_context**:
```python
def build_entity_context(
    selected_entities: list[Entity],
    token_encoder: tiktoken.Encoding | None = None,
    max_tokens: int = 8000,
    trace: Trace | None = None,  # NEW
    **kwargs
) -> tuple[str, pd.DataFrame]:
    if trace:
        trace.add_event(
            event="entity_selection",
            details={
                "total_entities_considered": len(selected_entities),
                "entities_selected": [...],
                "selection_criteria": {...},
                "token_budget": max_tokens,
                "tokens_used": current_tokens
            }
        )

    # ... existing logic ...
```

#### 3. Trace Storage

**Новый компонент: TraceStore**:
```python
# graphrag/query/introspection/trace_store.py

class TraceStore:
    """Storage for execution traces."""

    def __init__(self, backend: str = "redis"):
        if backend == "redis":
            self.client = redis.Redis(...)
        elif backend == "memory":
            self.client = {}  # In-memory dict
        # ...

    async def save(self, trace: Trace, ttl: int = 86400) -> str:
        """Save trace with TTL (default 24 hours)."""
        trace_id = trace.trace_id
        serialized = trace.serialize()
        await self.client.setex(
            f"trace:{trace_id}",
            ttl,
            json.dumps(serialized)
        )
        return trace_id

    async def get(self, trace_id: str) -> Trace | None:
        """Retrieve trace by ID."""
        data = await self.client.get(f"trace:{trace_id}")
        if data:
            return Trace.deserialize(json.loads(data))
        return None
```

---

## API Implementation Mapping

| API Endpoint | Python Component | Key Methods |
|--------------|------------------|-------------|
| `POST /api/v1/search` | `LocalSearch.search()`<br/>`GlobalSearch.search()`<br/>`DRIFTSearch.search()` | • `search(introspection=...)`<br/>• `Trace.add_stage()`<br/>• `TraceStore.save()` |
| `GET /api/v1/trace/{id}` | `TraceStore` | • `TraceStore.get(trace_id)`<br/>• `Trace.serialize()` |
| `GET /api/v1/trace/{id}/context/explain` | `LocalContextBuilder`<br/>`EntityRetrieval` | • `explain_entity_selection()`<br/>• `explain_relationship_selection()` |
| `GET /api/v1/trace/{id}/reasoning` | `GlobalSearch._map_response()`<br/>`GlobalSearch._reduce_response()` | • `Trace.get_llm_calls()`<br/>• `LLMCall.serialize()` |
| `GET /api/v1/trace/{id}/execution-graph` | `Trace` | • `Trace.to_execution_graph()`<br/>• `Trace.to_mermaid()` |

---

## Пример использования

### 1. Поиск с introspection

```bash
curl -X POST http://localhost:8000/api/v1/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are Microsoft'"'"'s AI partnerships?",
    "search_type": "local",
    "introspection": "full",
    "include_context": true,
    "include_reasoning": true
  }'
```

### 2. Получение трассировки

```bash
curl http://localhost:8000/api/v1/trace/550e8400-e29b-41d4-a716-446655440000
```

### 3. Объяснение выбора контекста

```bash
curl "http://localhost:8000/api/v1/trace/550e8400-e29b-41d4-a716-446655440000/context/explain?entity_id=MICROSOFT"
```

### 4. Цепочка рассуждений

```bash
curl http://localhost:8000/api/v1/trace/550e8400-e29b-41d4-a716-446655440000/reasoning
```

### 5. Граф выполнения (Mermaid)

```bash
curl "http://localhost:8000/api/v1/trace/550e8400-e29b-41d4-a716-446655440000/execution-graph?format=mermaid"
```

---

## Производительность и масштабирование

### Overhead introspection

| Introspection Level | Latency Overhead | Storage per Query | Use Case |
|---------------------|------------------|-------------------|----------|
| `none` | 0ms | 0 KB | Production (standard) |
| `basic` | +5-10ms | 1-2 KB | Production monitoring |
| `full` | +20-50ms | 50-200 KB | Development, debugging |
| `streaming` | +30-70ms | 100-500 KB | Research, analysis |

### Рекомендации по production deployment

1. **Default introspection=none** в production
2. **Selective introspection** для небольшого % запросов (sampling)
3. **TTL для traces** — 24 часа (configurable)
4. **Separate trace store** — Redis cluster с репликацией
5. **Async trace writes** — не блокировать основной ответ

---

## Следующие шаги

Данный документ является первой частью полного предложения по Introspection API. Следующие документы описывают:

- **02-graph-api.md** — API для работы с графовым представлением
- **03-vector-api.md** — API для работы с векторными представлениями
- **04-search-trajectory-api.md** — API траекторий семантического поиска
- **05-attractor-network-api.md** — API для аттракторов и концептуальных сетей
- **06-implementation-mapping.md** — Детальный mapping между API и компонентами кода
