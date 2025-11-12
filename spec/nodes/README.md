# Типы узлов и их взаимосвязи в GraphRAG

## Обзор

GraphRAG строит многоуровневую сеть узлов различных типов, каждый из которых играет специфическую роль в процессе индексации и поиска. Понимание типов узлов, их связей и влияния на query execution критически важно для эффективного использования системы.

## Иерархия типов узлов

```
┌─────────────────────────────────────────────────────────────┐
│                    DOCUMENT NODES                           │
│  • Исходные документы                                       │
│  • Корневой уровень данных                                  │
└──────────────────────┬──────────────────────────────────────┘
                       │ has_chunks
                       ↓
┌─────────────────────────────────────────────────────────────┐
│                  TEXT UNIT NODES                            │
│  • Текстовые фрагменты (chunks)                             │
│  • Единица обработки для LLM                                │
└──────────────────────┬──────────────────────────────────────┘
                       │ contains_entities
                       ↓
┌─────────────────────────────────────────────────────────────┐
│                    ENTITY NODES                             │
│  • Концепты, персоны, организации, места, события          │
│  • Ключевые объекты в графе знаний                         │
└──────────────────────┬──────────────────────────────────────┘
                       │ related_to (RELATIONSHIPS)
                       ↓
┌─────────────────────────────────────────────────────────────┐
│                  RELATIONSHIP EDGES                         │
│  • Связи между entities                                     │
│  • Описание отношений с весами                              │
└──────────────────────┬──────────────────────────────────────┘
                       │ forms_graph
                       ↓
┌─────────────────────────────────────────────────────────────┐
│                  COMMUNITY NODES                            │
│  • Тематические кластеры entities                           │
│  • Иерархическая структура (levels)                         │
└──────────────────────┬──────────────────────────────────────┘
                       │ has_report
                       ↓
┌─────────────────────────────────────────────────────────────┐
│                 COMMUNITY REPORTS                           │
│  • Структурированные отчеты о сообществах                   │
│  • Используются для global search                           │
└─────────────────────────────────────────────────────────────┘

                   ДОПОЛНИТЕЛЬНО (опционально)
┌─────────────────────────────────────────────────────────────┐
│                   COVARIATE NODES                           │
│  • Claims (утверждения/факты)                               │
│  • Извлеченные из текста assertion                          │
└─────────────────────────────────────────────────────────────┘
```

## Типы узлов

### 1. Document Nodes
- **Описание**: Исходные документы, загруженные в систему
- **Ключевые атрибуты**: `id`, `title`, `text`, `metadata`
- **Роль в поиске**: Источник информации, связь с исходными данными
- **Подробнее**: [01-document-nodes.md](01-document-nodes.md)

### 2. Text Unit Nodes
- **Описание**: Текстовые фрагменты после chunking
- **Ключевые атрибуты**: `id`, `text`, `n_tokens`, `document_ids`
- **Роль в поиске**: Контекстные фрагменты для Local Search
- **Подробнее**: [02-text-unit-nodes.md](02-text-unit-nodes.md)

### 3. Entity Nodes
- **Описание**: Извлеченные сущности (концепты, персоны, организации, места)
- **Ключевые атрибуты**: `id`, `title`, `type`, `description`, `node_degree`
- **Роль в поиске**: Ключевые объекты для навигации и retrieval
- **Подробнее**: [03-entity-nodes.md](03-entity-nodes.md)

### 4. Relationship Edges
- **Описание**: Связи между entities
- **Ключевые атрибуты**: `source`, `target`, `description`, `weight`, `combined_degree`
- **Роль в поиске**: Навигация по графу, выявление связей
- **Подробнее**: [04-relationship-edges.md](04-relationship-edges.md)

### 5. Community Nodes
- **Описание**: Тематические кластеры связанных entities
- **Ключевые атрибуты**: `id`, `level`, `entity_ids`, `parent`, `children`
- **Роль в поиске**: Тематическая организация для Global Search
- **Подробнее**: [05-community-nodes.md](05-community-nodes.md)

### 6. Community Reports
- **Описание**: Структурированные отчеты о сообществах
- **Ключевые атрибуты**: `title`, `summary`, `findings`, `rating`
- **Роль в поиске**: Высокоуровневые ответы на вопросы
- **Подробнее**: [06-community-reports.md](06-community-reports.md)

### 7. Covariate Nodes (Claims)
- **Описание**: Извлеченные утверждения и факты
- **Ключевые атрибуты**: `subject`, `object`, `type`, `description`
- **Роль в поиске**: Fact-checking и детальная информация
- **Подробнее**: [07-covariate-nodes.md](07-covariate-nodes.md)

## Связи между типами узлов

### Иерархические связи (Containment)

```
Document
  └─► has_chunks → Text Unit
      └─► mentions_entity → Entity
          └─► member_of_community → Community
              └─► has_report → Community Report
```

### Горизонтальные связи (Relations)

```
Entity ←──► related_to ←──► Entity
   │                           │
   └─► mentions_claim → Claim ←┘
```

### Вспомогательные связи

```
Text Unit ─► references_document → Document
Entity ─► appears_in_text_unit → Text Unit
Relationship ─► mentioned_in_text_unit → Text Unit
```

**Подробнее**: [08-node-interactions.md](08-node-interactions.md)

## Влияние на Query Execution

GraphRAG поддерживает несколько типов поиска, каждый использующий разные комбинации узлов:

### Global Search (глобальные вопросы)

```
Query
  ↓
[1] Найти релевантные Communities (через embeddings)
  ↓
[2] Извлечь Community Reports
  ↓
[3] Map: LLM отвечает на вопрос по каждому отчету
  ↓
[4] Reduce: LLM синтезирует финальный ответ
  ↓
Answer
```

**Используемые узлы**: Community Reports (primary), Communities

### Local Search (детальные вопросы)

```
Query
  ↓
[1] Найти релевантные Entities (через embeddings)
  ↓
[2] Найти связанные Text Units
  ↓
[3] Найти связанные Relationships
  ↓
[4] Построить локальный контекст
  ↓
[5] LLM отвечает на вопрос с контекстом
  ↓
Answer
```

**Используемые узлы**: Entities (primary), Text Units, Relationships

### Drift Search (исследовательский поиск)

```
Query
  ↓
[1] Начать с релевантных Entities
  ↓
[2] Следовать по Relationships (graph walk)
  ↓
[3] Собрать контекст из Text Units
  ↓
[4] LLM генерирует вопросы для углубления
  ↓
[5] Повторить с новыми вопросами
  ↓
Answer (с follow-up вопросами)
```

**Используемые узлы**: Entities, Relationships (primary), Text Units

**Подробнее**: [09-query-execution.md](09-query-execution.md)

## Embeddings и семантический поиск

Каждый тип узлов имеет векторное представление (embedding) для семантического поиска:

```
Document → document_text_embedding
Text Unit → text_unit_text_embedding
Entity → entity_description_embedding
Relationship → relationship_description_embedding
Community Report → community_full_content_embedding
```

**Подробнее**: [10-node-embeddings.md](10-node-embeddings.md)

## Метрики узлов

### Централизация (Centrality)

- **node_degree**: Количество связей у entity
- **betweenness_centrality**: Важность entity как "моста"
- **pagerank**: Важность entity в графе

### Частота (Frequency)

- **node_frequency**: Количество упоминаний entity в text_units
- **document_frequency**: В скольких документах упоминается

### Вес (Weight)

- **relationship_weight**: Сила связи между entities
- **combined_degree**: Сумма степеней узлов в relationship

## Примеры использования

### Пример 1: Поиск экспертов по теме

```python
# Query: "Кто основные эксперты по GraphRAG?"

# 1. Найти Entity "GraphRAG"
entity = find_entity("GraphRAG")

# 2. Найти связанные Entities типа "person"
experts = find_related_entities(
    entity,
    entity_type="person",
    relationship_types=["works_on", "contributes_to"]
)

# 3. Ранжировать по node_degree (влиятельность)
experts = sorted(experts, key=lambda e: e.node_degree, reverse=True)

# 4. Получить контекст из Text Units
context = get_text_units_for_entities(experts[:5])

# 5. LLM генерирует ответ
answer = llm_generate(query, context)
```

### Пример 2: Анализ темы

```python
# Query: "Что такое GraphRAG и как он работает?"

# 1. Найти релевантные Communities по embeddings
communities = semantic_search(
    query_embedding,
    community_embeddings,
    top_k=5
)

# 2. Извлечь Community Reports
reports = [get_community_report(c.id) for c in communities]

# 3. Синтезировать ответ из отчетов
answer = synthesize_from_reports(query, reports)
```

## Навигация по документации

1. **[Document Nodes](01-document-nodes.md)** - Исходные документы
2. **[Text Unit Nodes](02-text-unit-nodes.md)** - Текстовые фрагменты
3. **[Entity Nodes](03-entity-nodes.md)** - Извлеченные сущности
4. **[Relationship Edges](04-relationship-edges.md)** - Связи между сущностями
5. **[Community Nodes](05-community-nodes.md)** - Тематические кластеры
6. **[Community Reports](06-community-reports.md)** - Отчеты о сообществах
7. **[Covariate Nodes](07-covariate-nodes.md)** - Claims и утверждения
8. **[Node Interactions](08-node-interactions.md)** - Взаимодействия узлов
9. **[Query Execution](09-query-execution.md)** - Влияние на поиск и ответы
10. **[Node Embeddings](10-node-embeddings.md)** - Векторные представления

## Визуальные схемы

Все документы содержат ASCII-диаграммы и схемы для визуализации концепций. Для интерактивной визуализации графа используйте:

```bash
# Экспорт графа в формат для визуализации
graphrag visualize --output graph.html
```

## Ключевые концепции

- **Многоуровневая структура**: От документов до абстрактных тем
- **Семантическая связность**: Все узлы связаны через embeddings
- **Иерархия сообществ**: От детальных до глобальных кластеров
- **Метрики важности**: node_degree, node_frequency, combined_degree
- **Типы поиска**: Global (по сообществам), Local (по entities), Drift (исследовательский)

## Дальнейшее чтение

- [Основная документация pipeline](../README.md)
- [Роли языковых агентов](../07-llm-agents-roles.md)
- [Поток данных](../08-data-flow.md)
