# Семантические преобразования в GraphRAG

## Обзор

GraphRAG выполняет серию **семантических преобразований** данных на двух основных этапах:
1. **Индексация (Indexing)** - преобразование неструктурированного текста в граф знаний
2. **Query Execution** - преобразование вопроса в структурированный ответ

Каждое преобразование изменяет **семантическое представление** данных, сохраняя или извлекая смысл.

## Типы семантических преобразований

```
┌─────────────────────────────────────────────────────────────┐
│              INDEXING TRANSFORMATIONS                       │
├─────────────────────────────────────────────────────────────┤
│  1. Text → Chunks           (Semantic Segmentation)         │
│  2. Chunks → Entities       (Concept Extraction)            │
│  3. Chunks → Relationships  (Relation Extraction)           │
│  4. Entities → Summary      (Description Consolidation)     │
│  5. Graph → Communities     (Semantic Clustering)           │
│  6. Communities → Reports   (High-level Summarization)      │
│  7. Text → Embeddings       (Semantic Vectorization)        │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              QUERY TRANSFORMATIONS                          │
├─────────────────────────────────────────────────────────────┤
│  1. Query → Intent          (Query Understanding)           │
│  2. Query → Embedding       (Semantic Vectorization)        │
│  3. Embedding → Nodes       (Semantic Retrieval)            │
│  4. Nodes → Context         (Context Assembly)              │
│  5. Context → Answer        (Answer Generation)             │
│  6. Answers → Synthesis     (Multi-source Integration)      │
│  7. Answer → Follow-ups     (Question Generation)           │
└─────────────────────────────────────────────────────────────┘
```

## Характеристики преобразований

### По направлению семантического изменения

**Детализация (Decomposition)**:
- Text → Chunks: разбиение на фрагменты
- Chunks → Entities: извлечение концептов

**Абстракция (Abstraction)**:
- Entities → Summary: обобщение описаний
- Communities → Reports: высокоуровневое резюме

**Связывание (Association)**:
- Chunks → Relationships: выявление связей
- Graph → Communities: группировка по близости

**Векторизация (Vectorization)**:
- Text → Embeddings: преобразование в числовое представление

### По типу агента

**LLM-based** (используют языковые модели):
- Entity Extraction
- Relationship Extraction
- Description Summarization
- Community Report Generation
- Answer Generation
- Answer Synthesis

**Algorithmic** (алгоритмические):
- Text Chunking
- Community Detection (Leiden)
- Semantic Search (cosine similarity)
- Context Assembly

**Hybrid** (комбинированные):
- Query Understanding (правила + LLM)
- Semantic Retrieval (embeddings + ранжирование)

### По изменению информации

**Lossy** (с потерей информации):
- Text → Chunks (потеря общего контекста)
- Description Summarization (потеря деталей)
- Community Reports (агрегация → обобщение)

**Lossless** (без потери):
- Text → Embeddings (обратимо через approximation)
- Entity Extraction (извлечение без потери исходного текста)

**Enriching** (обогащение):
- Relationship Extraction (добавление связей)
- Community Detection (добавление структуры)
- Answer Generation (добавление контекста)

## Семантические свойства

### Сохранение смысла (Semantic Preservation)

```
┌──────────────────────┬─────────────────┬─────────────────┐
│ Transformation       │ Preservation    │ Information     │
├──────────────────────┼─────────────────┼─────────────────┤
│ Text → Chunks        │ High (95%+)     │ Segmentation    │
│ Chunks → Entities    │ Extractive      │ Concept-level   │
│ Entities → Summary   │ Medium (80%)    │ Consolidation   │
│ Communities → Reports│ Low-Medium (70%)│ Abstraction     │
│ Text → Embeddings    │ Semantic equiv. │ Dense vector    │
│ Context → Answer     │ Context-dep.    │ Generation      │
└──────────────────────┴─────────────────┴─────────────────┘
```

### Семантическая гранулярность

```
Fine-grained (детальные)
  ↑
  │ Text Units (tokens)
  │ Entities (concepts)
  │ Relationships (connections)
  │ Communities (clusters)
  │ Reports (summaries)
  ↓
Coarse-grained (обобщенные)
```

## Цепочки преобразований

### Indexing Pipeline Chain

```
Raw Text
  ↓ [Segmentation]
Text Chunks
  ↓ [LLM Extraction]
Entities + Relationships
  ↓ [LLM Summarization]
Consolidated Entities
  ↓ [Graph Construction]
Knowledge Graph
  ↓ [Leiden Clustering]
Communities (hierarchical)
  ↓ [LLM Summarization]
Community Reports
  ↓ [Embedding]
Semantic Vectors
```

### Query Execution Chains

**Global Search Chain**:
```
Query
  ↓ [Embedding]
Query Vector
  ↓ [Semantic Search]
Relevant Communities
  ↓ [Report Retrieval]
Community Reports
  ↓ [Map: LLM per report]
Partial Answers
  ↓ [Reduce: LLM synthesis]
Final Answer
```

**Local Search Chain**:
```
Query
  ↓ [Embedding + Intent Analysis]
Query Vector + Intent
  ↓ [Semantic Search]
Relevant Entities
  ↓ [Graph Expansion]
Extended Entity Set
  ↓ [Context Assembly]
Text Units + Entities + Relationships
  ↓ [LLM Generation]
Final Answer
```

**Drift Search Chain**:
```
Query
  ↓ [Initial Search]
Starting Entities
  ↓ [Graph Walk + Context Gathering]
Hop 1: Entities + Relationships
  ↓ [LLM Analysis + Question Generation]
Follow-up Questions
  ↓ [Iterative Graph Walk]
Hop 2-N: Extended exploration
  ↓ [Final LLM Synthesis]
Comprehensive Answer + Suggestions
```

## Документация

### Детальные описания

1. **[Преобразования при индексации](01-indexing-transformations.md)**
   - Text Chunking (Semantic Segmentation)
   - Entity Extraction (Concept Extraction)
   - Relationship Extraction (Relation Mining)
   - Description Summarization (Consolidation)
   - Community Detection (Clustering)
   - Community Report Generation (High-level Summary)
   - Text Embedding (Vectorization)

2. **[Преобразования при Query Execution](02-query-transformations.md)**
   - Query Understanding (Intent Analysis)
   - Semantic Search (Vector Retrieval)
   - Context Building (Assembly)
   - Answer Generation (LLM Generation)
   - Answer Synthesis (Multi-source Integration)
   - Follow-up Generation (Question Creation)

3. **[Цепочки преобразований](03-transformation-chains.md)**
   - Indexing pipeline chain
   - Global Search chain
   - Local Search chain
   - Drift Search chain
   - Hybrid chains

4. **[Сравнительные таблицы](04-comparison-tables.md)**
   - Характеристики по типам
   - Семантические свойства
   - Computational complexity
   - Quality metrics

## Ключевые концепции

### Семантическая эквивалентность

Преобразование сохраняет семантику, если:
```
semantic_similarity(input, output) ≥ threshold
```

Пример:
```
Input:  "GraphRAG uses LLMs to extract entities from text"
Entity: {name: "GraphRAG", type: "technology",
         description: "Uses LLMs for entity extraction"}
Semantic preservation: ~85%
```

### Информационная плотность

```
Information Density = Information Content / Representation Size

High density:  Embeddings (1536 dims vs thousands of tokens)
Medium:        Entities (concepts vs full text)
Low:           Community Reports (summaries of clusters)
```

### Композиция преобразований

Преобразования могут быть скомпонованы:
```
f: Text → Chunks
g: Chunks → Entities
h: Entities → Embeddings

h ∘ g ∘ f: Text → Embeddings (композиция)
```

Свойства композиции:
- Семантическая деградация накапливается
- Каждый этап добавляет latency
- Кэширование промежуточных результатов критично

## Роли агентов в преобразованиях

### Extraction Agent
**Роль**: Преобразование текста в структурированные концепты
**Преобразования**:
- Chunks → Entities
- Chunks → Relationships
**Семантика**: Извлечение ключевых концептов и связей

### Summarization Agent
**Роль**: Консолидация множественных описаний
**Преобразования**:
- Multiple Descriptions → Single Description
**Семантика**: Сохранение смысла при уменьшении объема

### Community Report Agent
**Роль**: Создание высокоуровневых резюме
**Преобразования**:
- Community (Entities + Relationships) → Report
**Семантика**: Абстракция от деталей к общей картине

### Embedding Model
**Роль**: Векторизация для семантического поиска
**Преобразования**:
- Text → Dense Vector
**Семантика**: Сохранение семантической близости в векторном пространстве

### Answer Generation Agent
**Роль**: Синтез ответа из контекста
**Преобразования**:
- Query + Context → Answer
**Семантика**: Генерация релевантного ответа, сохраняющего смысл источников

## Метрики качества преобразований

### Семантическая точность

```python
def semantic_accuracy(
    input_text: str,
    output_representation: Any,
    ground_truth: Any
) -> float:
    """
    Измеряет точность семантического преобразования

    Returns: score [0, 1]
    """
    # Для entity extraction
    if type(output_representation) == Entity:
        # Precision: правильно извлеченные / все извлеченные
        # Recall: правильно извлеченные / все в ground truth
        return f1_score(output_representation, ground_truth)

    # Для summarization
    elif type(output_representation) == str:
        # Semantic similarity
        return cosine_similarity(
            embed(output_representation),
            embed(ground_truth)
        )
```

### Информационная полнота

```python
def information_completeness(
    input_text: str,
    output_representation: Any
) -> float:
    """
    Измеряет полноту сохранения информации

    Returns: score [0, 1]
    """
    # Извлечь ключевые концепты из input
    input_concepts = extract_concepts(input_text)

    # Извлечь концепты из output
    output_concepts = extract_concepts_from_representation(
        output_representation
    )

    # Покрытие
    coverage = len(input_concepts & output_concepts) / len(input_concepts)

    return coverage
```

### Вычислительная эффективность

```python
TRANSFORMATION_COMPLEXITY = {
    "text_chunking": "O(n)",           # n = text length
    "entity_extraction": "O(m * k)",   # m = chunks, k = LLM time
    "community_detection": "O(V + E)", # V = nodes, E = edges
    "semantic_search": "O(n * d)",     # n = vectors, d = dimensions
    "answer_generation": "O(k)",       # k = LLM time
}
```

## Оптимизация преобразований

### Параллелизация

Преобразования, которые можно параллелить:
- Entity Extraction (по chunks)
- Text Embedding (по batches)
- Map phase в Global Search (по reports)

### Кэширование

Кэшировать результаты преобразований:
- LLM outputs (extraction, summarization)
- Embeddings (векторы неизменны)
- Intermediate results (entities, relationships)

### Batching

Группировать для эффективности:
- Embedding generation (batch_size=500)
- LLM calls (где возможно)

## Сравнение с другими подходами

### GraphRAG vs Traditional RAG

```
Traditional RAG:
Query → Embedding → Semantic Search → Chunks → LLM → Answer

GraphRAG:
Query → Embedding → Semantic Search → Entities/Communities →
  → Graph Context → LLM → Answer

Дополнительные преобразования в GraphRAG:
+ Entity Extraction (semantic enrichment)
+ Community Detection (thematic organization)
+ Graph-based Context Building (relational context)
```

## Дальнейшее чтение

- **[Основная документация pipeline](../README.md)**
- **[Роли LLM агентов](../07-llm-agents-roles.md)**
- **[Типы узлов](../nodes/README.md)**
- **[Поток данных](../08-data-flow.md)**

## Глоссарий

- **Семантическое преобразование**: Изменение представления данных с сохранением смысла
- **Семантическая эквивалентность**: Равенство смысла при разных представлениях
- **Информационная плотность**: Количество информации на единицу представления
- **Lossy transformation**: Преобразование с потерей части информации
- **Lossless transformation**: Преобразование без потери информации
- **Composition**: Последовательное применение преобразований
- **Semantic preservation**: Сохранение смысла при преобразовании
