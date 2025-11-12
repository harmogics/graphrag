# Семантические преобразования при индексации

## Обзор

Индексация в GraphRAG состоит из **7 основных семантических преобразований**, которые преобразуют неструктурированный текст в многоуровневый граф знаний с векторными представлениями.

## Полная цепочка преобразований

```
┌─────────────────────────────────────────────────────────────┐
│                  INDEXING TRANSFORMATION CHAIN              │
└─────────────────────────────────────────────────────────────┘

Raw Documents (неструктурированный текст)
  ↓ [T1: Semantic Segmentation]
Text Units (семантические фрагменты, ~1200 tokens)
  ↓ [T2: Concept Extraction via LLM]
Entities (концепты: organization, person, geo, event)
  + [T3: Relation Extraction via LLM]
Relationships (связи между entities с описанием и силой)
  ↓ [T4: Description Consolidation via LLM]
Consolidated Entities (единое описание на entity)
  ↓ [T5: Graph Construction + Clustering]
Communities (тематические кластеры, иерархические)
  ↓ [T6: High-level Summarization via LLM]
Community Reports (структурированные отчеты)
  ↓ [T7: Semantic Vectorization]
Embeddings (векторные представления для всех уровней)
  ↓
Queryable Knowledge Graph
```

## T1: Text Chunking (Semantic Segmentation)

### Описание

Разбиение документов на текстовые фрагменты оптимального размера для LLM обработки с сохранением семантической целостности.

### Входные/выходные данные

```
Input:  Document (полный текст, N tokens)
Output: Text Units (chunks, ~1200 tokens each)
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| **Тип преобразования** | Decomposition (разбиение) |
| **Семантическое сохранение** | 95-98% (минимальная потеря контекста) |
| **Гранулярность** | Fine-grained → Fine-grained |
| **Направление абстракции** | Lateral (боковое, без абстракции) |
| **Информационная потеря** | Низкая (5-10% на границах) |
| **Обратимость** | High (можно восстановить документ) |

### Семантические свойства

**Сохранение локального контекста**:
```python
# С overlap=100 tokens
chunk1 = text[0:1200]      # "...context A [boundary] context B..."
chunk2 = text[1100:2300]   # "...context B [boundary] context C..."
                           #      ^^^^^^^^^^^
                           #      preserved overlap
```

**Потеря глобального контекста**:
- Каждый chunk теряет связь с общей структурой документа
- Mitigation: хранение document_ids и metadata

### Параметры преобразования

| Параметр | Значение | Влияние на семантику |
|---|---|---|
| **chunk_size** | 1200 tokens | Баланс контекста/обработки |
| **overlap** | 100 tokens | Сохранение boundary концептов |
| **strategy** | tokens | Семантическая сегментация |
| **encoding** | cl100k_base | Токенизация |

### Алгоритм

```
Algorithm: TokenTextSplitter

Input: document_text, chunk_size=1200, overlap=100
Output: text_units[]

1. Tokenize document → tokens[]
2. current_pos = 0
3. While current_pos < len(tokens):
     chunk_end = current_pos + chunk_size
     chunk_tokens = tokens[current_pos:chunk_end]
     chunk_text = detokenize(chunk_tokens)
     text_units.append(chunk_text)
     current_pos += (chunk_size - overlap)  # Move with overlap
4. Return text_units
```

### Роль в цепочке

**Позиция**: Первое преобразование после загрузки документов

**Назначение**:
- Создание manageable units для LLM
- Сохранение семантической целостности на границах
- Подготовка к extraction

**Связи**:
- **Input from**: Documents
- **Output to**: Entity/Relationship Extraction (T2, T3)
- **Used in query**: Local Search (контекстные фрагменты)

---

## T2: Entity Extraction (Concept Extraction)

### Описание

Извлечение ключевых концептов (entities) из текста с помощью LLM с классификацией по типам и созданием описаний.

### Входные/выходные данные

```
Input:  Text Unit (chunk, ~1200 tokens)
Output: Entities[] (name, type, description)
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| **Тип преобразования** | Abstraction + Extraction |
| **Семантическое сохранение** | 70-85% (concept-level) |
| **Гранулярность** | Fine-grained → Medium-grained |
| **Направление абстракции** | Bottom-up (от текста к концептам) |
| **Информационная потеря** | Средняя (детали → ключевые концепты) |
| **Обратимость** | Low (нельзя восстановить исходный текст) |

### Семантические свойства

**Концептуальная абстракция**:
```
Text: "Microsoft, founded by Bill Gates and Paul Allen in 1975..."

Entities extracted:
1. {name: "Microsoft", type: "organization",
    description: "Technology company founded in 1975"}
2. {name: "Bill Gates", type: "person",
    description: "Co-founder of Microsoft"}
3. {name: "Paul Allen", type: "person",
    description: "Co-founder of Microsoft"}
```

**Семантическое обогащение**:
- LLM добавляет контекст к извлеченным entities
- Классификация по типам добавляет семантическую структуру

**Потеря детализации**:
- Пропуск менее важных концептов
- Упрощение описаний

### LLM Промпт

```
-Goal-
Extract all entities from the text.

-Steps-
1. Identify entities (name, type, description)
   Types: organization, person, geo, event
2. Format each as:
   ("entity"<|>NAME<|>TYPE<|>DESCRIPTION)

-Text-
{text_unit}

-Output-
[Structured entities...]
```

**Файл**: `graphrag/prompts/index/extract_graph.py`

### Параметры преобразования

| Параметр | Значение | Влияние на семантику |
|---|---|---|
| **LLM model** | GPT-4 | Качество extraction |
| **entity_types** | [org, person, geo, event] | Семантическая классификация |
| **max_gleanings** | 0-3 | Полнота извлечения (recall) |
| **temperature** | 0 | Детерминированность |

### Роль в цепочке

**Позиция**: После Text Chunking

**Назначение**:
- Извлечение ключевых концептов
- Создание узлов графа
- Семантическая классификация

**Связи**:
- **Input from**: Text Units (T1)
- **Output to**: Description Summarization (T4), Graph Construction
- **Used in query**: Local Search (entity-based retrieval)

---

## T3: Relationship Extraction (Relation Mining)

### Описание

Извлечение семантических связей между entities с описанием характера отношения и оценкой силы связи.

### Входные/выходные данные

```
Input:  Text Unit + Extracted Entities
Output: Relationships[] (source, target, description, strength)
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| **Тип преобразования** | Association + Enrichment |
| **Семантическое сохранение** | 75-85% (relational semantics) |
| **Гранулярность** | Medium-grained (entity-level) |
| **Направление абстракции** | Lateral (связывание, не абстракция) |
| **Информационная потеря** | Средняя (детали связи → описание) |
| **Обратимость** | Low |

### Семантические свойства

**Реляционная семантика**:
```
Text: "Microsoft acquired GitHub in 2018 for $7.5 billion"

Relationship:
{
  source: "Microsoft",
  target: "GitHub",
  description: "Microsoft acquired GitHub in 2018",
  strength: 9  # Strong, definitive relationship
}
```

**Сила связи (Strength)**:
- 9-10: Прямые, очевидные связи (owns, works_for)
- 5-8: Контекстуальные связи (collaborates_with, mentioned_with)
- 1-4: Слабые, косвенные связи (might_be_related_to)

**Семантическое обогащение графа**:
- Добавление структуры (edges)
- Explicit representation неявных связей
- Навигационная информация

### Параметры преобразования

| Параметр | Значение | Влияние на семантику |
|---|---|---|
| **LLM model** | GPT-4 | Качество relation detection |
| **min_strength** | 1 | Порог релевантности связи |
| **description_length** | ~100 tokens | Детальность описания |

### Роль в цепочке

**Позиция**: Параллельно с Entity Extraction

**Назначение**:
- Создание ребер графа
- Связывание концептов
- Навигационная структура

**Связи**:
- **Input from**: Text Units (T1) + Entities (T2)
- **Output to**: Graph Construction
- **Used in query**: Drift Search (graph walk), Local Search (расширение)

---

## T4: Description Summarization (Consolidation)

### Описание

Объединение множественных описаний одной entity из разных text_units в единое краткое описание с помощью LLM.

### Входные/выходные данные

```
Input:  Entity + Multiple Descriptions
Output: Entity + Single Consolidated Description
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| **Тип преобразования** | Aggregation + Abstraction |
| **Семантическое сохранение** | 80-90% (ключевая информация) |
| **Гранулярность** | Medium-grained |
| **Направление абстракции** | Lateral (консолидация без абстракции) |
| **Информационная потеря** | Низкая-средняя (детали → суммаризация) |
| **Обратимость** | Medium (можно восстановить большую часть) |

### Семантические свойства

**Консолидация смысла**:
```
Entity: "GraphRAG"

Description 1 (from chunk 1):
"GraphRAG is a framework for knowledge graph construction"

Description 2 (from chunk 2):
"GraphRAG uses LLMs to extract entities and relationships"

Description 3 (from chunk 3):
"GraphRAG supports hierarchical community detection"

Consolidated Description:
"GraphRAG is a framework for knowledge graph construction that uses
LLMs to extract entities and relationships, and supports hierarchical
community detection for organizing the knowledge graph."
```

**Разрешение противоречий**:
- LLM выбирает наиболее авторитетную информацию
- Или указывает на разногласия, если критично

**Сохранение ключевых аспектов**:
- Все основные характеристики entity
- Удаление дублирования

### LLM Промпт

```
-Goal-
Consolidate multiple descriptions of the same entity into a single
comprehensive description.

-Entity-
{entity_name}

-Descriptions-
1. {description_1}
2. {description_2}
...

-Output-
Single, comprehensive description in third person.
```

**Файл**: `graphrag/prompts/index/summarize_descriptions.py`

### Роль в цепочке

**Позиция**: После Entity Extraction

**Назначение**:
- Устранение дублирования
- Создание единого представления entity
- Улучшение качества embeddings

**Связи**:
- **Input from**: Entities (T2, raw)
- **Output to**: Graph Construction, Embeddings (T7)
- **Used in query**: Entity descriptions в контексте

---

## T5: Community Detection (Semantic Clustering)

### Описание

Кластеризация entities в тематические сообщества с использованием алгоритма Leiden, создавая иерархическую структуру.

### Входные/выходные данные

```
Input:  Knowledge Graph (Entities + Relationships)
Output: Communities (hierarchical clusters)
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| **Тип преобразования** | Clustering + Structuring |
| **Семантическое сохранение** | 90%+ (структурная организация) |
| **Гранулярность** | Medium → Coarse-grained |
| **Направление абстракции** | Bottom-up (entities → themes) |
| **Информационная потеря** | Минимальная (добавление структуры) |
| **Обратимость** | High (можно развернуть в entities) |

### Семантические свойства

**Тематическая организация**:
```
Level 0 Communities (детальные):
- Community 0: AI Technology (12 entities)
  Entities: [GPT-4, LLM, Transformer, ...]

- Community 1: Graph Methods (8 entities)
  Entities: [Knowledge Graph, Graph Neural Networks, ...]

Level 1 Communities (абстрактные):
- Community 10: AI and Graphs (20 entities)
  Children: [Community 0, Community 1]
```

**Семантическая близость**:
- Entities с сильными relationships группируются
- Тематическая coherence внутри сообществ

**Иерархическая абстракция**:
- Level 0: конкретные подтемы
- Level 1+: общие темы

### Алгоритм

```
Algorithm: Hierarchical Leiden Clustering

Input: Graph G(V, E), max_cluster_size
Output: Communities (hierarchical)

1. Apply Leiden to G → level_0_communities
2. current_level = 0
3. While len(current_communities) > 1:
     For each community in current_communities:
       If size > max_cluster_size:
         Split into sub-communities
     Build community_graph (communities as nodes)
     Apply Leiden to community_graph → next_communities
     Set parent-child relationships
     current_level++
4. Return all communities (all levels)
```

### Параметры преобразования

| Параметр | Значение | Влияние на семантику |
|---|---|---|
| **max_cluster_size** | 10 | Гранулярность кластеров |
| **use_lcc** | true | Фокус на связном графе |
| **resolution** | auto | Уровень детализации |

### Роль в цепочке

**Позиция**: После Graph Construction

**Назначение**:
- Тематическая организация
- Иерархическая структура
- Подготовка к суммаризации

**Связи**:
- **Input from**: Knowledge Graph (T2, T3, T4)
- **Output to**: Community Report Generation (T6)
- **Used in query**: Global Search (community-level retrieval)

---

## T6: Community Report Generation (High-level Summarization)

### Описание

Генерация структурированных отчетов о каждом сообществе с использованием LLM, включая title, summary, findings, и rating.

### Входные/выходные данные

```
Input:  Community (entity_ids, relationship_ids) + Context
Output: Community Report (title, summary, findings, rating)
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| **Тип преобразования** | Abstraction + Synthesis |
| **Семантическое сохранение** | 60-75% (high-level insights) |
| **Гранулярность** | Medium → Coarse-grained |
| **Направление абстракции** | Bottom-up (entities → themes) |
| **Информационная потеря** | Средняя-высокая (детали → insights) |
| **Обратимость** | Low (нельзя восстановить все entities) |

### Семантические свойства

**Абстракция к insights**:
```
Community Input:
- Entities: GraphRAG, LLM, Entity Extraction, Knowledge Graph (20 total)
- Relationships: 35 connections
- Text Units: 50 mentions

Community Report Output:
{
  "title": "GraphRAG and Knowledge Graph Construction",
  "summary": "This community focuses on GraphRAG framework...",
  "findings": [
    {"summary": "GraphRAG uses LLMs for entity extraction",
     "explanation": "GraphRAG leverages large language models..."},
    ...
  ],
  "rating": 8.5
}
```

**Семантическое обогащение**:
- Выявление паттернов
- Создание insights
- Оценка важности

**Потеря конкретики**:
- Переход от фактов к интерпретации
- Отбор наиболее важной информации

### LLM Промпт

```
-Goal-
Write a comprehensive report of a community given its entities
and relationships.

-Report Structure-
- TITLE: community's name
- SUMMARY: executive summary
- IMPACT SEVERITY RATING: 0-10
- DETAILED FINDINGS: 5-10 key insights

-Context-
Entities: [entity details...]
Relationships: [relationship details...]

-Output-
JSON formatted report
```

**Файл**: `graphrag/prompts/index/community_report.py`

### Параметры преобразования

| Параметр | Значение | Влияние на семантику |
|---|---|---|
| **LLM model** | GPT-4 | Качество insights |
| **max_report_length** | 2000 tokens | Детальность отчета |
| **max_input_length** | 8000 tokens | Контекст для анализа |

### Роль в цепочке

**Позиция**: После Community Detection

**Назначение**:
- Высокоуровневые резюме тем
- Insights для Global Search
- Человекочитаемые описания кластеров

**Связи**:
- **Input from**: Communities (T5)
- **Output to**: Embeddings (T7)
- **Used in query**: Global Search (primary source)

---

## T7: Text Embedding (Semantic Vectorization)

### Описание

Преобразование всех текстовых представлений в dense векторы для семантического поиска.

### Входные/выходные данные

```
Input:  Text (documents, text_units, entities, reports)
Output: Dense Vectors (embeddings, 1536 dimensions)
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| **Тип преобразования** | Vectorization (encoding) |
| **Семантическое сохранение** | 85-95% (semantic equivalence) |
| **Гранулярность** | Variable → Fixed dimensions |
| **Направление абстракции** | Encoding (не абстракция) |
| **Информационная потеря** | Средняя (dense representation) |
| **Обратимость** | Low (approximate reconstruction) |

### Семантические свойства

**Семантическая эквивалентность в векторном пространстве**:
```python
text1 = "GraphRAG extracts entities from text"
text2 = "GraphRAG identifies concepts in documents"

emb1 = embed(text1)  # [0.23, -0.15, 0.42, ...]
emb2 = embed(text2)  # [0.25, -0.13, 0.40, ...]

cosine_similarity(emb1, emb2) = 0.89  # High similarity
```

**Сохранение семантической близости**:
- Похожие по смыслу тексты → близкие векторы
- Различные по смыслу → далекие векторы

**Информационная компрессия**:
- Текст (тысячи токенов) → 1536 чисел
- Lossy, но сохраняет семантику

### Embedding Model

```
Model: text-embedding-3-small
Dimensions: 1536
Training: Contrastive learning on large corpus
Properties:
  - Semantic similarity preservation
  - Fast inference
  - Stable representations
```

### Параметры преобразования

| Параметр | Значение | Влияние на семантику |
|---|---|---|
| **model** | text-embedding-3-small | Качество embeddings |
| **dimensions** | 1536 | Информационная емкость |
| **batch_size** | 500 | Эффективность |

### Роль в цепочке

**Позиция**: Финальное преобразование (параллельно для всех уровней)

**Назначение**:
- Семантический поиск
- Similarity matching
- Retrieval

**Связи**:
- **Input from**: All text representations (T1-T6)
- **Output to**: Vector Store
- **Used in query**: Все типы search (semantic retrieval)

---

## Сравнительная таблица преобразований

| Преобразование | Тип | Semantic Preservation | Information Loss | Reversibility | LLM Required |
|---|---|---|---|---|---|
| T1: Text Chunking | Decomposition | 95-98% | Low | High | No |
| T2: Entity Extraction | Abstraction | 70-85% | Medium | Low | Yes |
| T3: Relationship Extraction | Association | 75-85% | Medium | Low | Yes |
| T4: Description Summarization | Aggregation | 80-90% | Low-Medium | Medium | Yes |
| T5: Community Detection | Clustering | 90%+ | Minimal | High | No |
| T6: Community Reports | Abstraction | 60-75% | Medium-High | Low | Yes |
| T7: Text Embedding | Vectorization | 85-95% | Medium | Low | No |

## Композиция преобразований

### Последовательная композиция

```
Documents
  →[T1]→ Text Units
  →[T2]→ Entities
  →[T4]→ Consolidated Entities
  →[T5]→ Communities
  →[T6]→ Reports

Overall semantic preservation:
0.96 * 0.77 * 0.85 * 0.92 * 0.67 ≈ 0.41 (41%)
```

**Вывод**: Композиция преобразований значительно снижает детальность, но сохраняет высокоуровневую семантику.

### Параллельная композиция

```
Text Units →[T2]→ Entities ↘
                            →[Graph]→ Knowledge Graph
Text Units →[T3]→ Relationships ↗
```

Параллельные преобразования обогащают друг друга.

## Следующие разделы

- **[Query Transformations →](02-query-transformations.md)**
- **[Transformation Chains →](03-transformation-chains.md)**
- **[Comparison Tables →](04-comparison-tables.md)**
