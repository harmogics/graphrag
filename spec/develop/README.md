# GraphRAG Introspection & Development API

## Обзор

Данный каталог содержит техническую спецификацию **REST API для максимального открытия внутренних механизмов GraphRAG**. API предоставляет разработчикам, исследователям и продуктовым командам полный доступ к траекториям семантического поиска, графовым и векторным представлениям, аттракторам и концептуальным сетям.

## Философия проектирования

### Glass Box Architecture

API следует принципу **"стеклянной коробки"**, где каждый этап обработки запроса доступен для наблюдения:

```
Традиционный "Black Box":           GraphRAG Introspection API:
┌─────────────────┐                  ┌─────────────────────────────────┐
│                 │                  │  Query Understanding            │
│  Query ──▶ ???  │                  │  ↓                              │
│         ──▶ ???  │                  │  Entity Mapping                 │
│         ──▶ ???  │                  │  ↓                              │
│         ──▶ ???  │                  │  Graph Expansion                │
│                 │                  │  ↓                              │
│  ──▶ Answer     │                  │  Context Building               │
│                 │                  │  ↓                              │
└─────────────────┘                  │  LLM Reasoning                  │
                                     │  ↓                              │
                                     │  Answer Synthesis               │
                                     │                                 │
                                     │  ✓ All stages visible           │
                                     │  ✓ Intermediate results exposed │
                                     │  ✓ Decision points logged       │
                                     │  ✓ Reproducible traces          │
                                     └─────────────────────────────────┘
```

### Ключевые принципы

1. **Максимальная прозрачность** — каждый шаг выполнения доступен через API
2. **Reproducibility** — достаточно данных для воспроизведения результатов
3. **Explainability** — объяснения почему конкретный контекст был выбран
4. **Multi-level access** — от simple summary до full traces
5. **Performance awareness** — метрики производительности на каждом этапе

---

## Структура документации

| Документ | Описание | Основные endpoints |
|----------|----------|-------------------|
| **[01-introspection-api.md](01-introspection-api.md)** | Базовый introspection framework | `POST /api/v1/search`<br/>`GET /api/v1/trace/{id}`<br/>`GET /api/v1/trace/{id}/context/explain` |
| **[02-graph-api.md](02-graph-api.md)** | Доступ к графовому представлению | `GET /api/v1/graph/entities`<br/>`GET /api/v1/graph/relationships`<br/>`POST /api/v1/graph/traverse` |
| **[03-vector-api.md](03-vector-api.md)** | Доступ к векторным embeddings | `POST /api/v1/vectors/search/similar`<br/>`POST /api/v1/vectors/search/hybrid`<br/>`POST /api/v1/vectors/analyze/clusters` |
| **[04-search-trajectory-api.md](04-search-trajectory-api.md)** | Траектории семантического поиска | `POST /api/v1/search/trajectory`<br/>`GET /api/v1/search/trajectory/{id}`<br/>`POST /api/v1/search/trajectory/compare` |
| **[05-attractor-network-api.md](05-attractor-network-api.md)** | Аттракторы и концептуальные сети | `POST /api/v1/attractors/detect`<br/>`POST /api/v1/attractors/expand-concepts`<br/>`POST /api/v1/attractors/related-questions` |
| **[06-implementation-mapping.md](06-implementation-mapping.md)** | Mapping API ↔ код GraphRAG | Детальный mapping к компонентам |

---

## Быстрый старт

### 1. Базовый поиск с introspection

```bash
# Стандартный поиск без introspection
curl -X POST http://localhost:8000/api/v1/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are Microsoft'"'"'s AI partnerships?",
    "search_type": "local"
  }'

# Поиск с полной introspection
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

**Response включает**:
- ✓ Полный ответ на запрос
- ✓ Траекторию выполнения (stages, operations, decisions)
- ✓ Контекст (entities, relationships, text units)
- ✓ LLM prompts и responses
- ✓ Метрики производительности (latency, tokens, cost)

---

### 2. Исследование графа

```bash
# Получить топ-10 сущностей по centrality
curl "http://localhost:8000/api/v1/graph/entities?sort_by=rank&limit=10"

# Детали о Microsoft с соседями
curl "http://localhost:8000/api/v1/graph/entities/MICROSOFT?include_neighbors=true&neighbors_depth=2"

# Найти кратчайший путь Microsoft → GPT-4
curl -X POST http://localhost:8000/api/v1/graph/shortest-path \
  -H "Content-Type: application/json" \
  -d '{
    "source_entity": "MICROSOFT",
    "target_entity": "GPT-4",
    "algorithm": "dijkstra"
  }'
```

---

### 3. Векторный поиск

```bash
# Найти похожие сущности
curl -X POST http://localhost:8000/api/v1/vectors/search/similar \
  -H "Content-Type: application/json" \
  -d '{
    "query": "AI research partnerships",
    "search_type": "entity",
    "top_k": 10,
    "min_similarity": 0.7
  }'

# Hybrid search (vector + graph)
curl -X POST http://localhost:8000/api/v1/vectors/search/hybrid \
  -H "Content-Type: application/json" \
  -d '{
    "query": "Microsoft AI partnerships",
    "weights": {
      "vector_similarity": 0.6,
      "graph_centrality": 0.4
    }
  }'
```

---

### 4. Анализ траектории

```bash
# Выполнить поиск с записью траектории
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

# Получить сохраненную траекторию
curl "http://localhost:8000/api/v1/search/trajectory/traj_550e8400?include_full_prompts=true"

# Визуализировать траекторию (Mermaid diagram)
curl "http://localhost:8000/api/v1/search/trajectory/traj_550e8400/visualize?format=mermaid"
```

---

### 5. Аттракторы и концептуальные сети

```bash
# Найти аттракторы в AI research community
curl -X POST http://localhost:8000/api/v1/attractors/detect \
  -H "Content-Type: application/json" \
  -d '{
    "scope": "query_based",
    "query": "AI research partnerships",
    "max_attractors": 5
  }'

# Расширить концепты из запроса
curl -X POST http://localhost:8000/api/v1/attractors/expand-concepts \
  -H "Content-Type: application/json" \
  -d '{
    "query": "AI partnerships",
    "expansion_depth": 2,
    "max_concepts": 10
  }'

# Генерировать связанные вопросы
curl -X POST http://localhost:8000/api/v1/attractors/related-questions \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are Microsoft'"'"'s AI partnerships?",
    "num_questions": 5
  }'
```

---

## Архитектура API

### Полная схема endpoints

```
/api/v1/
├── search
│   ├── POST /search                        # Основной поиск с introspection
│   └── POST /trajectory                    # Поиск с записью траектории
│
├── trace/{trace_id}
│   ├── GET /                              # Получить полную трассировку
│   ├── GET /context/explain               # Объяснение выбора контекста
│   ├── GET /reasoning                     # Цепочка рассуждений LLM
│   ├── GET /execution-graph               # Граф выполнения
│   └── GET /visualize                     # Визуализация траектории
│
├── graph
│   ├── entities
│   │   ├── GET /                          # Список сущностей
│   │   └── GET /{entity_id}               # Детали сущности
│   ├── relationships
│   │   └── GET /                          # Список связей
│   ├── communities
│   │   ├── GET /                          # Список communities
│   │   └── GET /{community_id}            # Детали community
│   ├── POST /traverse                     # Graph traversal (BFS/DFS)
│   ├── POST /subgraph                     # Извлечение подграфа
│   └── POST /shortest-path                # Кратчайший путь
│
├── vectors
│   ├── GET /entities/{id}/embeddings      # Embeddings сущности
│   ├── search
│   │   ├── POST /similar                  # Vector similarity search
│   │   └── POST /hybrid                   # Hybrid (vector + graph)
│   └── analyze
│       └── POST /clusters                 # Кластеризация
│
├── trajectory
│   ├── POST /search                       # Поиск с траекторией
│   ├── GET /{trajectory_id}               # Получить траекторию
│   ├── POST /compare                      # Сравнить траектории
│   └── GET /{id}/visualize                # Визуализация
│
└── attractors
    ├── POST /detect                       # Найти аттракторы
    ├── POST /expand-concepts              # Расширить концепты
    ├── POST /mutual-connections           # Взаимные связи
    └── POST /related-questions            # Генерация вопросов
```

---

## Use Cases

### Для разработчиков

**Отладка поисковых запросов**:
```python
# Выполнить поиск с introspection
response = await client.post("/api/v1/search", json={
    "query": "complex query",
    "introspection": "full"
})

# Анализировать траекторию
trajectory = response["introspection"]
for stage in trajectory["stages"]:
    print(f"Stage: {stage['stage_name']}")
    print(f"Duration: {stage['duration_ms']}ms")
    print(f"Operations: {len(stage['operations'])}")

# Найти узкие места
bottlenecks = [
    s for s in trajectory["stages"]
    if s["duration_ms"] > 500
]
```

**Оптимизация параметров**:
```python
# Сравнить разные настройки
traj1 = await search(query, top_k=10)
traj2 = await search(query, top_k=20)

comparison = await client.post("/api/v1/search/trajectory/compare", json={
    "trajectory_id_1": traj1["trace_id"],
    "trajectory_id_2": traj2["trace_id"]
})

# Анализ: какой top_k лучше?
print(f"Quality difference: {comparison['quality_delta']}")
print(f"Cost difference: ${comparison['cost_difference_usd']}")
```

---

### Для исследователей

**Изучение семантических траекторий**:
```python
# Записать траекторию поиска
trajectory = await search_with_trajectory("AI partnerships")

# Анализ semantic path
semantic_path = trajectory["semantic_path"]
print(f"Query concepts: {semantic_path['query_concepts']}")
print(f"Activated entities: {semantic_path['activated_entities']}")
print(f"Concept drift: {semantic_path['concept_drift']}")
print(f"Path coherence: {semantic_path['path_coherence']}")
```

**Анализ аттракторов**:
```python
# Найти аттракторы
attractors = await client.post("/api/v1/attractors/detect", json={
    "scope": "global",
    "max_attractors": 10
})

# Изучить бассейны аттракторов
for attractor in attractors["attractors"]:
    print(f"Attractor: {attractor['entity']['title']}")
    print(f"Basin size: {attractor['basin']['size']}")
    print(f"Star topology score: {attractor['star_topology_score']}")
```

---

### Для продуктовых команд

**Explainable AI для пользователей**:
```python
# Получить объяснение результатов
trace_id = search_result["metadata"]["trace_id"]
explanation = await client.get(f"/api/v1/trace/{trace_id}/context/explain")

# Показать пользователю:
# "Я выбрал эти источники потому что:"
for entity in explanation["entities"]:
    if entity["selected"]:
        reason = entity["selection_reasoning"]["primary_reason"]
        score = entity["selection_reasoning"]["total_score"]
        print(f"✓ {entity['entity_title']}: {reason} (score: {score})")
```

**A/B тестирование стратегий поиска**:
```python
# Вариант A: local search
result_a = await search(query, search_type="local", introspection="basic")

# Вариант B: global search
result_b = await search(query, search_type="global", introspection="basic")

# Метрики для сравнения
metrics = {
    "latency_a": result_a["metadata"]["total_time_ms"],
    "latency_b": result_b["metadata"]["total_time_ms"],
    "cost_a": result_a["metadata"]["cost_usd"],
    "cost_b": result_b["metadata"]["cost_usd"],
    "tokens_a": result_a["metadata"]["total_tokens"]["total"],
    "tokens_b": result_b["metadata"]["total_tokens"]["total"]
}
```

---

## Реализация

### Технический стек

**Backend**:
- **FastAPI** — REST API framework
- **Pydantic** — Request/response validation
- **Redis** (optional) — Trace storage с TTL
- **NetworkX** — Graph algorithms (centrality, shortest path)

**Existing GraphRAG Components**:
- `graphrag/query/structured_search/` — Search engines
- `graphrag/query/context_builder/` — Context building
- `graphrag/data_model/` — Entity, Relationship, Community models
- `graphrag/vector_stores/` — Vector storage and search

### Оценка усилий

| Component | New LOC | Estimated Time | Priority |
|-----------|---------|----------------|----------|
| Introspection API | ~500 | 2 weeks | HIGH |
| Graph API | ~1000 | 2 weeks | HIGH |
| Vector API | ~400 | 1 week | MEDIUM |
| Trajectory API | ~600 | 2 weeks | HIGH |
| Attractor API | ~800 | 2 weeks | MEDIUM |
| Testing | ~1000 | 1 week | HIGH |
| **TOTAL** | **~4300** | **10 weeks** | - |

### Поэтапный план

**Phase 1: Core Introspection (Weeks 1-2)**
- ✓ Implement `Trace` and `TraceStore`
- ✓ Modify `LocalSearch` для introspection
- ✓ Add `/api/v1/search` endpoint

**Phase 2: Graph API (Weeks 3-4)**
- ✓ Entities, Relationships, Communities endpoints
- ✓ Graph traversal и subgraph extraction

**Phase 3: Vector & Trajectory (Weeks 5-7)**
- ✓ Vector similarity search
- ✓ Trajectory recording в context builders
- ✓ Visualization endpoints

**Phase 4: Attractors & Polish (Weeks 8-10)**
- ✓ Attractor detection
- ✓ Concept expansion
- ✓ Related questions
- ✓ Performance optimization
- ✓ Documentation

---

## Performance Considerations

### Introspection Overhead

| Introspection Level | Latency Overhead | Storage per Query | Use Case |
|---------------------|------------------|-------------------|----------|
| `none` | 0ms | 0 KB | Production (default) |
| `basic` | +5-10ms | 1-2 KB | Monitoring |
| `full` | +20-50ms | 50-200 KB | Development |
| `streaming` | +30-70ms | 100-500 KB | Research |

### Рекомендации для production

1. **Default introspection=none** — минимальный overhead
2. **Selective sampling** — включать introspection для 1-5% запросов
3. **TTL для traces** — 24 часа (configurable)
4. **Async trace writes** — не блокировать основной response
5. **Separate trace store** — Redis cluster, не основная DB

---

## Безопасность и приватность

### Соображения безопасности

1. **Sensitive data in traces** — LLM prompts могут содержать user data
   - Solution: Опциональная редакция PII в traces
   - Flag: `redact_pii=true`

2. **API authentication** — защита endpoints
   - JWT tokens для аутентификации
   - Rate limiting (100 req/min per user)

3. **Trace access control** — кто может видеть traces?
   - User-scoped traces (только свои traces)
   - Admin access для debugging

4. **Cost control** — introspection увеличивает storage
   - Квоты на trace storage per user
   - Auto-cleanup старых traces

---

## Примеры интеграции

### Python SDK

```python
from graphrag_client import GraphRAGClient

client = GraphRAGClient(base_url="http://localhost:8000")

# Поиск с introspection
result = await client.search(
    query="What are Microsoft's AI partnerships?",
    introspection="full"
)

# Анализ траектории
if result.trace_id:
    trajectory = await client.get_trajectory(result.trace_id)
    print(f"Total stages: {len(trajectory.stages)}")
    print(f"Total time: {trajectory.total_duration_ms}ms")

# Graph exploration
microsoft = await client.graph.get_entity("MICROSOFT", include_neighbors=True)
print(f"Neighbors: {len(microsoft.neighbors['outgoing'])}")

# Attractor analysis
attractors = await client.attractors.detect(
    scope="query_based",
    query="AI research"
)
for a in attractors:
    print(f"{a.entity.title}: score={a.attractor_score:.2f}")
```

### JavaScript/TypeScript SDK

```typescript
import { GraphRAGClient } from '@graphrag/client';

const client = new GraphRAGClient({ baseUrl: 'http://localhost:8000' });

// Search with introspection
const result = await client.search({
  query: "Microsoft AI partnerships",
  introspection: "full"
});

// Visualize trajectory
if (result.trace_id) {
  const diagram = await client.trajectory.visualize(result.trace_id, {
    format: "mermaid"
  });
  console.log(diagram);
}
```

---

## FAQ

**Q: Какой overhead добавляет introspection?**
A: Зависит от уровня: `basic` — +5-10ms, `full` — +20-50ms. Production должен использовать `introspection=none` по умолчанию.

**Q: Как долго хранятся traces?**
A: По умолчанию 24 часа (TTL). Configurable через API.

**Q: Можно ли использовать introspection в production?**
A: Да, но выборочно (1-5% запросов) для мониторинга качества.

**Q: Поддерживает ли API real-time streaming?**
A: Да, `introspection=streaming` отправляет events по мере выполнения (SSE/WebSocket).

**Q: Совместим ли API с существующим GraphRAG CLI?**
A: Да, использует те же data models и компоненты.

---

## Дальнейшие шаги

1. **Review документации** — прочитать все 6 документов
2. **Prototype** — создать минимальный proof-of-concept
3. **Feedback** — собрать отзывы от потенциальных пользователей
4. **Implementation** — начать с Phase 1 (Core Introspection)
5. **Beta testing** — тестирование с ранними adopters
6. **Production release** — full API launch

---

## Контакты и поддержка

- **GitHub Issues**: https://github.com/microsoft/graphrag/issues
- **Documentation**: https://microsoft.github.io/graphrag/
- **Community**: GraphRAG Slack channel

---

## Лицензия

Данная спецификация является частью проекта GraphRAG (Microsoft) и распространяется под MIT License.
