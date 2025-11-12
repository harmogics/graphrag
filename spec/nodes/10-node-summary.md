# Сводная карта типов узлов и их взаимодействий

## Быстрый обзор

Этот документ предоставляет **визуальную сводку** всех типов узлов GraphRAG, их связей и влияния на поиск.

## Полная иерархия узлов

```
┌─────────────────────────────────────────────────────────────────┐
│                   GRAPHRAG NODE HIERARCHY                       │
└─────────────────────────────────────────────────────────────────┘

LEVEL 0: SOURCE
┌────────────────────────────┐
│     DOCUMENT NODES         │  ← Исходные документы
│  • id, title, text         │
│  • Роль: Источник данных   │
└──────────────┬─────────────┘
               │ has_chunks (1:N)
               ↓
LEVEL 1: CHUNKS
┌────────────────────────────┐
│    TEXT UNIT NODES         │  ← Текстовые фрагменты
│  • id, text, n_tokens      │
│  • Роль: Единица обработки │
└──────────────┬─────────────┘
               │ contains_entities (M:N)
               ↓
LEVEL 2: CONCEPTS
┌────────────────────────────┐
│     ENTITY NODES           │  ← Ключевые концепты
│  • id, title, type         │
│  • node_degree, frequency  │
│  • Роль: Навигация         │
└──────────────┬─────────────┘
               │ related_to (M:N)
               │
               ├─────────────────────────┐
               │                         │
               ↓                         ↓
┌────────────────────────────┐  ┌──────────────────────┐
│  RELATIONSHIP EDGES        │  │  COVARIATE NODES     │
│  • source, target, weight  │  │  • Claims/Facts      │
│  • Роль: Связи             │  │  • Роль: Assertions  │
└──────────────┬─────────────┘  └──────────────────────┘
               │ forms_graph
               ↓
LEVEL 3: CLUSTERS
┌────────────────────────────┐
│    COMMUNITY NODES         │  ← Тематические кластеры
│  • id, level, parent       │
│  • entity_ids              │
│  • Роль: Тематическая      │
│    организация             │
└──────────────┬─────────────┘
               │ has_report (1:1)
               ↓
LEVEL 4: SUMMARIES
┌────────────────────────────┐
│  COMMUNITY REPORTS         │  ← Структурированные отчеты
│  • title, summary          │
│  • findings, rating        │
│  • Роль: Высокоуровневые   │
│    ответы                  │
└────────────────────────────┘

AUXILIARY: EMBEDDINGS
┌────────────────────────────┐
│   EMBEDDING VECTORS        │  ← Векторные представления
│  • Для всех типов узлов    │
│  • Роль: Семантический     │
│    поиск                   │
└────────────────────────────┘
```

## Матрица использования узлов в Query Types

```
┌──────────────────┬──────────────┬──────────────┬──────────────┐
│  Node Type       │ Global Search│ Local Search │ Drift Search │
├──────────────────┼──────────────┼──────────────┼──────────────┤
│ Documents        │      -       │     ○        │      -       │
│ Text Units       │      -       │    ★★★       │     ★★       │
│ Entities         │      ○       │    ★★★       │    ★★★       │
│ Relationships    │      -       │     ★        │    ★★★       │
│ Communities      │     ★★       │      -       │      ○       │
│ Comm. Reports    │    ★★★       │      -       │      -       │
│ Covariates       │      -       │     ○        │      ○       │
└──────────────────┴──────────────┴──────────────┴──────────────┘

Legend:
  ★★★ = Primary (критичны для работы)
  ★★  = Secondary (важны для качества)
  ★   = Tertiary (используются для расширения)
  ○   = Optional (опциональны)
  -   = Not used (не используются)
```

## Поток данных через узлы

### От документа до ответа

```
INPUT: "How does GraphRAG work?"
    ↓
┌─────────────────────────────────────────────┐
│  1. DOCUMENT NODES                          │
│     document.txt → загружен                 │
└──────────────────┬──────────────────────────┘
                   ↓ chunking
┌─────────────────────────────────────────────┐
│  2. TEXT UNIT NODES                         │
│     5 chunks созданы (size=1200, overlap=100)│
└──────────────────┬──────────────────────────┘
                   ↓ LLM extraction
┌─────────────────────────────────────────────┐
│  3. ENTITY NODES                            │
│     20 entities извлечено:                  │
│     • "GraphRAG" (organization)             │
│     • "Knowledge Graph" (concept)           │
│     • "LLM" (technology)                    │
│     • ...                                   │
└──────────────────┬──────────────────────────┘
                   ↓ relationships
┌─────────────────────────────────────────────┐
│  4. RELATIONSHIP EDGES                      │
│     35 relationships:                       │
│     • GraphRAG → uses → LLM                 │
│     • GraphRAG → produces → Knowledge Graph │
│     • ...                                   │
└──────────────────┬──────────────────────────┘
                   ↓ clustering
┌─────────────────────────────────────────────┐
│  5. COMMUNITY NODES                         │
│     3 communities (hierarchical):           │
│     • Level 0: AI Technology (12 entities)  │
│     • Level 0: Graph Methods (8 entities)   │
│     • Level 1: GraphRAG Ecosystem (20)      │
└──────────────────┬──────────────────────────┘
                   ↓ LLM summarization
┌─────────────────────────────────────────────┐
│  6. COMMUNITY REPORTS                       │
│     3 reports созданы:                      │
│     • "AI Technology Community"             │
│       Title, Summary, 5 Findings            │
│     • "Graph Methods Community"             │
│     • "GraphRAG Ecosystem"                  │
└──────────────────┬──────────────────────────┘
                   ↓ query execution
┌─────────────────────────────────────────────┐
│  7. ANSWER GENERATION                       │
│                                             │
│  LOCAL SEARCH выбран (детальный вопрос)     │
│  ↓                                          │
│  • Find entities: "GraphRAG", "LLM"         │
│  • Get text_units: chunks 1, 2, 4           │
│  • Build context                            │
│  • LLM generates answer                     │
│                                             │
│  Result: "GraphRAG uses LLMs to extract..." │
└─────────────────────────────────────────────┘
```

## Метрики важности узлов

### Entity Importance

```
High Importance (Hub Entities)
┌─────────────────────────────────────────────┐
│  • node_degree > 20                         │
│  • node_frequency > 50                      │
│  • Примеры: "AI", "GraphRAG", "LLM"         │
│  • Используются: В большинстве queries      │
└─────────────────────────────────────────────┘

Medium Importance
┌─────────────────────────────────────────────┐
│  • node_degree 5-20                         │
│  • node_frequency 10-50                     │
│  • Примеры: Specific методы, технологии     │
│  • Используются: В специфичных queries      │
└─────────────────────────────────────────────┘

Low Importance (Leaf Entities)
┌─────────────────────────────────────────────┐
│  • node_degree 1-4                          │
│  • node_frequency 1-9                       │
│  • Примеры: Даты, малоизвестные персоны     │
│  • Используются: Редко, для детализации     │
└─────────────────────────────────────────────┘
```

### Relationship Strength

```
Strong Relationships (weight < 0.2)
┌─────────────────────────────────────────────┐
│  • strength = 9-10 (от LLM)                 │
│  • Прямые, очевидные связи                  │
│  • Примеры: "is_part_of", "works_on"        │
└─────────────────────────────────────────────┘

Medium Relationships (weight 0.2-0.5)
┌─────────────────────────────────────────────┐
│  • strength = 5-8                           │
│  • Связи с контекстом                       │
│  • Примеры: "related_to", "mentions"        │
└─────────────────────────────────────────────┘

Weak Relationships (weight > 0.5)
┌─────────────────────────────────────────────┐
│  • strength = 1-4                           │
│  • Косвенные связи                          │
│  • Используются: Только при расширении      │
└─────────────────────────────────────────────┘
```

## Decision Tree: Выбор типа поиска

```
                    QUERY
                      │
                      ↓
         ┌────────────┴────────────┐
         │                         │
    Широкий вопрос?           Специфичный?
    "Каковы тренды?"          "Что такое X?"
         │                         │
         ↓                         ↓
   ┌─────────────┐           ┌─────────────┐
   │   GLOBAL    │           │    LOCAL    │
   │   SEARCH    │           │   SEARCH    │
   └─────────────┘           └─────────────┘
         │                         │
         ↓                         ↓
   Community                  Entity
   Reports                    Text Units
         │                         │
         ↓                         ↓
   Map-Reduce                Single LLM
   (Много LLM)               call
         │                         │
         └──────────┬──────────────┘
                    ↓
              Исследовать связи?
              "Как связаны X и Y?"
                    │
                    ↓
              ┌─────────────┐
              │    DRIFT    │
              │   SEARCH    │
              └─────────────┘
                    │
                    ↓
              Graph Walk
              Relationships
                    │
                    ↓
              Iterative
              (Много hops)
```

## Оптимизация по use case

### Use Case 1: FAQ System

```yaml
Требование: Быстрые ответы на частые вопросы

Оптимизация:
  - Primary: Entity Nodes (pre-indexed)
  - Search: Local Search (single LLM call)
  - Caching: Aggressive (cache all queries)

Настройки:
  top_entities: 10
  top_text_units: 5
  enable_caching: true
```

### Use Case 2: Research Assistant

```yaml
Требование: Глубокий анализ с источниками

Оптимизация:
  - Primary: Community Reports + Text Units
  - Search: Hybrid (Global + Local)
  - Caching: Moderate

Настройки:
  top_communities: 10
  top_entities: 30
  max_text_units: 20
  include_sources: true
```

### Use Case 3: Knowledge Explorer

```yaml
Требование: Исследование связей

Оптимизация:
  - Primary: Relationships + Entities
  - Search: Drift Search
  - Caching: Minimal (exploratory)

Настройки:
  max_hops: 3
  min_relationship_strength: 5
  generate_followups: true
```

## Performance Benchmarks

### Типичные показатели

```
┌──────────────┬────────────┬─────────────┬──────────────┐
│ Search Type  │ Avg Time   │ LLM Calls   │ Tokens Used  │
├──────────────┼────────────┼─────────────┼──────────────┤
│ Global       │ 15-30 sec  │ 6-11        │ 15,000-30,000│
│ Local        │ 3-8 sec    │ 1           │ 3,000-6,000  │
│ Drift        │ 30-60 sec  │ 5-15        │ 20,000-40,000│
│ Hybrid       │ 20-40 sec  │ 7-12        │ 18,000-35,000│
└──────────────┴────────────┴─────────────┴──────────────┘

Условия: 100 документов, 2000 entities, GPT-4
```

## Ключевые принципы

### 1. Иерархия абстракции

```
Documents (конкретные)
    ↓
Text Units (фрагменты)
    ↓
Entities (концепты)
    ↓
Communities (темы)
    ↓
Reports (резюме - абстрактные)
```

### 2. Семантическая связность

Все узлы связаны через embeddings:
- Одинаковый embedding space
- Cosine similarity для поиска
- Гибридное ранжирование (semantic + structural)

### 3. Многоуровневый поиск

Разные уровни для разных вопросов:
- **Level 0 (Data)**: Documents, Text Units → детали
- **Level 1 (Concepts)**: Entities, Relationships → факты
- **Level 2 (Themes)**: Communities → темы
- **Level 3 (Summaries)**: Reports → обзор

### 4. Гибкость и расширяемость

Можно добавить:
- Новые типы entities
- Кастомные relationships
- Дополнительные метрики
- Альтернативные алгоритмы clustering

## Визуальная шпаргалка

```
ВОПРОС: Как выбрать узлы для ответа?

┌─────────────────────────────────────────────────────────┐
│  Нужен обзор темы?                                      │
│  → Community Reports                                    │
│  → Global Search                                        │
│                                                         │
│  Нужны конкретные факты?                                │
│  → Entities + Text Units                                │
│  → Local Search                                         │
│                                                         │
│  Нужно исследовать связи?                               │
│  → Relationships + Entities                             │
│  → Drift Search                                         │
│                                                         │
│  Нужны источники?                                       │
│  → Documents через Text Units                           │
│  → Source attribution                                   │
│                                                         │
│  Нужна быстрая проверка факта?                          │
│  → Covariates (Claims)                                  │
│  → Fact-checking                                        │
└─────────────────────────────────────────────────────────┘
```

## Дополнительные ресурсы

### Детальная документация

1. **[README](README.md)** - Полный обзор всех типов узлов
2. **[Document Nodes](01-document-nodes.md)** - Исходные данные
3. **[Text Unit Nodes](02-text-unit-nodes.md)** - Текстовые фрагменты
4. **[Entity Nodes](03-entity-nodes.md)** - Ключевые концепты
5. **[Query Execution](09-query-execution.md)** - Влияние на поиск

### Основная документация pipeline

- **[../README.md](../README.md)** - Главная документация
- **[../07-llm-agents-roles.md](../07-llm-agents-roles.md)** - Роли LLM агентов
- **[../08-data-flow.md](../08-data-flow.md)** - Поток данных

## Практические рекомендации

### Для быстрого старта

1. Начните с **Local Search** - самый универсальный
2. Используйте **Global Search** для обзорных вопросов
3. Экспериментируйте с **Drift Search** для exploration

### Для production

1. Кэшируйте результаты queries
2. Настройте параметры под ваш domain
3. Мониторьте метрики quality и performance
4. Регулярно обновляйте граф с новыми документами

### Для отладки

1. Визуализируйте граф для понимания структуры
2. Проверяйте entity extraction quality
3. Анализируйте community reports
4. Тестируйте разные типы queries

---

**Это краткая сводка. Для детальной информации обратитесь к соответствующим разделам документации.**
