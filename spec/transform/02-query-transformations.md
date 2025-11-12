# Семантические преобразования при выполнении запросов

## Обзор

Query execution в GraphRAG состоит из **7 основных семантических преобразований**, которые преобразуют пользовательский запрос в структурированный ответ с использованием индексированного графа знаний.

## Полная цепочка преобразований

```
┌─────────────────────────────────────────────────────────────┐
│                  QUERY TRANSFORMATION CHAIN                 │
└─────────────────────────────────────────────────────────────┘

User Query (естественный язык)
  ↓ [Q1: Query Analysis]
Query Intent + Type (global/local/drift)
  ↓ [Q2: Query Embedding]
Query Vector (1536 dimensions)
  ↓ [Q3: Semantic Retrieval]
Relevant Nodes (communities/entities/text_units)
  ↓ [Q4: Context Building]
Structured Context (consolidated information)
  ↓ [Q5: Answer Generation]
Initial Answer (from LLM)
  ↓ [Q6: Answer Synthesis] (optional, for Global Search)
Final Answer (consolidated response)
  ↓ [Q7: Follow-up Generation] (optional)
Suggested Follow-up Questions
  ↓
Complete Response
```

## Типы запросов и их цепочки

### Global Search Chain

```
Query → [Q1+Q2+Q3] → Communities → [Q4] → Multiple Contexts
  → [Q5: Map Phase] → Intermediate Answers
  → [Q6: Reduce Phase] → Final Answer
  → [Q7] → Follow-ups
```

### Local Search Chain

```
Query → [Q1+Q2+Q3] → Entities + Text Units → [Q4] → Single Context
  → [Q5] → Answer
  → [Q7] → Follow-ups
```

### Drift Search Chain

```
Query → [Q1+Q2+Q3] → Start Entities → [Q4: Graph Walk]
  → Iterative Context Building → [Q5] → Answer
  → [Q7] → Follow-ups (exploration paths)
```

---

## Q1: Query Analysis (Intent Understanding)

### Описание

Анализ пользовательского запроса для определения типа поиска и извлечения ключевых аспектов запроса.

### Входные/выходные данные

```
Input:  User Query (естественный язык, ~10-100 tokens)
Output: Query Intent (type: global/local/drift, entities, aspects)
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| **Тип преобразования** | Classification + Extraction |
| **Семантическое сохранение** | 90-95% (intent preservation) |
| **Гранулярность** | Variable → Structured |
| **Направление абстракции** | Lateral (анализ без изменения) |
| **Информационная потеря** | Минимальная |
| **Обратимость** | High |

### Семантические свойства

**Классификация типа запроса**:
```
Query: "What are the main trends in AI research?"
↓
Intent Analysis:
{
  query_type: "global",  # Broad, overview question
  key_aspects: ["AI research", "trends", "overview"],
  scope: "broad",
  expected_depth: "high-level"
}

Query: "What is GraphRAG and how does it work?"
↓
Intent Analysis:
{
  query_type: "local",  # Specific entity question
  key_aspects: ["GraphRAG", "mechanism", "definition"],
  target_entities: ["GraphRAG"],
  scope: "specific",
  expected_depth: "detailed"
}
```

**Извлечение ключевых аспектов**:
- Определение основных тем запроса
- Извлечение потенциальных entity names
- Определение ожидаемой глубины ответа

**Роль в выборе стратегии**:
- Global Search: для обзорных вопросов о трендах, темах
- Local Search: для конкретных вопросов о entities
- Drift Search: для вопросов о связях и отношениях

### Алгоритм

```
Algorithm: QueryAnalysis

Input: query_text
Output: query_intent

1. Classify query type:
   - Contains "how are X and Y related"? → drift
   - Contains broad terms ("trends", "overview", "main themes")? → global
   - Contains specific entity names? → local
   - Default → local

2. Extract key aspects:
   - Named entities (potential targets)
   - Action verbs (type of information needed)
   - Scope indicators (broad/specific)

3. Determine parameters:
   - Expected answer length
   - Need for sources
   - Complexity level

4. Return structured intent
```

### Параметры преобразования

| Параметр | Значение | Влияние на семантику |
|---|---|---|
| **default_search** | local | Стратегия по умолчанию |
| **classification_threshold** | 0.7 | Уверенность классификации |
| **entity_extraction** | enabled | Выделение entities |

### Роль в цепочке

**Позиция**: Первое преобразование, определяет стратегию

**Назначение**:
- Выбор типа поиска
- Извлечение ключевых аспектов
- Определение параметров выполнения

**Связи**:
- **Input from**: User (raw query)
- **Output to**: Query Embedding (Q2), Retrieval (Q3)
- **Controls**: Вся последующая цепочка

---

## Q2: Query Embedding (Semantic Vectorization)

### Описание

Преобразование текста запроса в dense векторное представление для семантического поиска.

### Входные/выходные данные

```
Input:  Query Text (natural language)
Output: Query Vector (1536 dimensions)
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| **Тип преобразования** | Vectorization (encoding) |
| **Семантическое сохранение** | 90-95% (semantic meaning) |
| **Гранулярность** | Text → Fixed dimensions |
| **Направление абстракции** | Encoding (не абстракция) |
| **Информационная потеря** | Низкая |
| **Обратимость** | Low (approximate) |

### Семантические свойства

**Семантическая эквивалентность**:
```python
query1 = "How does GraphRAG extract entities?"
query2 = "What is the entity extraction process in GraphRAG?"

emb1 = embed(query1)  # [0.15, -0.23, 0.41, ...]
emb2 = embed(query2)  # [0.17, -0.21, 0.39, ...]

cosine_similarity(emb1, emb2) = 0.92  # Very high similarity
```

**Сохранение семантических намерений**:
- Похожие по смыслу запросы → близкие векторы
- Различные намерения → далекие векторы
- Синонимы и парафразы сохраняют близость

**Независимость от формулировки**:
- "What is X?" ≈ "Define X" ≈ "Explain X"
- Инвариантность к порядку слов (частично)

### Embedding Model

```
Model: text-embedding-3-small
Dimensions: 1536
Same model as for indexing embeddings

Properties:
  - Consistent with indexed embeddings
  - Fast inference (~50ms)
  - Semantic similarity preservation
```

### Параметры преобразования

| Параметр | Значение | Влияние на семантику |
|---|---|---|
| **model** | text-embedding-3-small | Качество векторов |
| **dimensions** | 1536 | Информационная емкость |
| **normalization** | L2 | Cosine similarity optimization |

### Роль в цепочке

**Позиция**: После Query Analysis

**Назначение**:
- Подготовка к семантическому поиску
- Обеспечение совместимости с indexed embeddings
- Similarity matching

**Связи**:
- **Input from**: Query Analysis (Q1)
- **Output to**: Semantic Retrieval (Q3)
- **Shared space**: With all indexed embeddings (T7)

---

## Q3: Semantic Retrieval (Embedding → Nodes)

### Описание

Поиск наиболее релевантных узлов графа на основе семантической близости query embedding.

### Входные/выходные данные

```
Input:  Query Vector + Node Type (communities/entities/text_units)
Output: Ranked Nodes (top-k most similar)
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| **Тип преобразования** | Retrieval + Ranking |
| **Семантическое сохранение** | 85-90% (relevance matching) |
| **Гранулярность** | Vector → Nodes |
| **Направление абстракции** | Materialization (vector → objects) |
| **Информационная потеря** | Средняя (отбор top-k) |
| **Обратимость** | Medium |

### Семантические свойства

**Семантическая релевантность**:
```python
Query: "How does GraphRAG use LLMs for entity extraction?"
Query Embedding: [0.15, -0.23, 0.41, ...]

Vector Store Search:
┌────────────────────────────────────────────────────────┐
│ Top Entities (cosine similarity):                      │
│  1. "GraphRAG" (0.89)                                  │
│  2. "Entity Extraction" (0.87)                         │
│  3. "LLM" (0.85)                                       │
│  4. "Knowledge Graph" (0.78)                           │
│  5. "GPT-4" (0.76)                                     │
└────────────────────────────────────────────────────────┘
```

**Стратегии поиска по типу запроса**:

**Global Search**:
```
Query → Embedding → Search Community Reports
  ↓
Top 10 Community Reports (by similarity)
  ↓
Filter by relevance threshold (>0.7)
```

**Local Search**:
```
Query → Embedding → Search Entities + Text Units
  ↓
Top 30 Entities + Top 20 Text Units
  ↓
Expand with relationships
```

**Drift Search**:
```
Query → Embedding → Search Entities (start points)
  ↓
Top 5 Start Entities
  ↓
Graph walk from these entities
```

**Гибридное ранжирование**:
- Semantic score (cosine similarity)
- Structural score (node_degree, frequency)
- Combined: `final_score = α * semantic + (1-α) * structural`

### Алгоритм

```
Algorithm: SemanticRetrieval

Input: query_embedding, node_type, top_k
Output: ranked_nodes[]

1. Vector search in appropriate store:
   IF node_type == "community_reports":
     candidates = vector_search(community_embeddings, query_embedding)
   ELIF node_type == "entities":
     candidates = vector_search(entity_embeddings, query_embedding)
   ELIF node_type == "text_units":
     candidates = vector_search(text_unit_embeddings, query_embedding)

2. Calculate scores:
   FOR each candidate:
     semantic_score = cosine_similarity(candidate.embedding, query_embedding)
     structural_score = normalize(candidate.node_degree, candidate.frequency)
     final_score = 0.7 * semantic_score + 0.3 * structural_score

3. Rank by final_score

4. Select top_k

5. Return ranked_nodes
```

### Параметры преобразования

| Параметр | Значение | Влияние на семантику |
|---|---|---|
| **top_k** | 10-30 | Полнота retrieval |
| **similarity_threshold** | 0.7 | Фильтр релевантности |
| **hybrid_alpha** | 0.7 | Баланс semantic/structural |
| **expansion** | enabled (Local) | Расширение контекста |

### Роль в цепочке

**Позиция**: После Query Embedding

**Назначение**:
- Отбор релевантных узлов
- Ранжирование по релевантности
- Подготовка к context building

**Связи**:
- **Input from**: Query Embedding (Q2), Indexed Embeddings (T7)
- **Output to**: Context Building (Q4)
- **Used by**: Все типы search

---

## Q4: Context Building (Nodes → Context)

### Описание

Сборка структурированного контекста из извлеченных узлов для передачи LLM на генерацию ответа.

### Входные/выходные данные

```
Input:  Ranked Nodes (entities, text_units, reports)
Output: Structured Context (formatted text for LLM)
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| **Тип преобразования** | Aggregation + Formatting |
| **Семантическое сохранение** | 95%+ (assembly without loss) |
| **Гранулярность** | Nodes → Unified text |
| **Направление абстракции** | Lateral (сборка без абстракции) |
| **Информационная потеря** | Минимальная |
| **Обратимость** | High |

### Семантические свойства

**Структурированная сборка контекста**:

**Global Search Context**:
```
Context structure:
┌─────────────────────────────────────────────┐
│ COMMUNITY REPORTS (10 items)               │
│                                             │
│ Report 1: AI Technology Community           │
│   Title: "Advances in AI and LLMs"         │
│   Summary: [2-3 sentences]                  │
│   Findings:                                 │
│     - Finding 1: [description]              │
│     - Finding 2: [description]              │
│     ...                                     │
│   Rating: 8.5                               │
│                                             │
│ Report 2: Graph Methods Community           │
│   ...                                       │
└─────────────────────────────────────────────┘

Total tokens: ~15,000-20,000
```

**Local Search Context**:
```
Context structure:
┌─────────────────────────────────────────────┐
│ ENTITIES (30 items)                         │
│                                             │
│ Entity: GraphRAG                            │
│   Type: organization                        │
│   Description: [consolidated description]   │
│   Degree: 45, Frequency: 120               │
│                                             │
│ Entity: LLM                                 │
│   ...                                       │
│                                             │
├─────────────────────────────────────────────┤
│ RELATIONSHIPS (50 items)                    │
│                                             │
│ GraphRAG → uses → LLM                       │
│   Description: [relationship description]   │
│   Strength: 9                               │
│   ...                                       │
│                                             │
├─────────────────────────────────────────────┤
│ TEXT UNITS (20 items)                       │
│                                             │
│ Text Unit 1:                                │
│   [original text chunk, ~1200 tokens]       │
│                                             │
│ Text Unit 2:                                │
│   ...                                       │
└─────────────────────────────────────────────┘

Total tokens: ~8,000-12,000
```

**Drift Search Context (Iterative)**:
```
Context per hop:
┌─────────────────────────────────────────────┐
│ HOP 1:                                      │
│   Start Entity: GraphRAG                    │
│   Connected Entities: [LLM, Knowledge Graph]│
│   Relationships: [uses, produces]           │
│   Text Units: [relevant chunks]             │
│                                             │
│ HOP 2:                                      │
│   From: LLM                                 │
│   Connected: [GPT-4, Entity Extraction]     │
│   ...                                       │
└─────────────────────────────────────────────┘

Total tokens: ~5,000-10,000 per iteration
```

**Оптимизация контекста**:
- Дедупликация entities
- Сортировка по relevance
- Truncation при превышении лимита токенов
- Приоритизация high-degree entities

### Алгоритм

```
Algorithm: ContextBuilding

Input: ranked_nodes, query, search_type
Output: structured_context

1. Initialize context sections:
   context = {entities: [], relationships: [], text_units: [], reports: []}

2. Add nodes by type:
   FOR each node in ranked_nodes:
     IF search_type == "global":
       context.reports.append(format_report(node))
     ELIF search_type == "local":
       IF node.type == "entity":
         context.entities.append(format_entity(node))
         # Expand relationships
         context.relationships.extend(get_relationships(node))
         # Get text units mentioning entity
         context.text_units.extend(get_text_units(node))

3. Deduplicate:
   context.entities = unique(context.entities)
   context.text_units = unique(context.text_units)

4. Sort by relevance:
   context.entities = sort_by_degree(context.entities)

5. Truncate if needed:
   total_tokens = count_tokens(context)
   IF total_tokens > max_context_tokens:
     context = truncate_lowest_priority(context, max_context_tokens)

6. Format for LLM:
   formatted_context = """
   Entities:
   {format_entities(context.entities)}

   Relationships:
   {format_relationships(context.relationships)}

   Sources:
   {format_text_units(context.text_units)}
   """

7. Return formatted_context
```

### Параметры преобразования

| Параметр | Значение | Влияние на семантику |
|---|---|---|
| **max_context_tokens** | 8000-20000 | Полнота контекста |
| **deduplication** | enabled | Устранение повторов |
| **prioritization** | by_degree | Порядок важности |
| **include_sources** | true (Local) | Детальность ответа |

### Роль в цепочке

**Позиция**: После Semantic Retrieval

**Назначение**:
- Сборка всей релевантной информации
- Форматирование для LLM
- Оптимизация размера контекста

**Связи**:
- **Input from**: Semantic Retrieval (Q3)
- **Output to**: Answer Generation (Q5)
- **Critical for**: Качество ответа

---

## Q5: Answer Generation (Context → Answer)

### Описание

Генерация ответа на запрос с использованием LLM и собранного контекста.

### Входные/выходные данные

```
Input:  Query + Structured Context
Output: Answer (natural language response)
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| **Тип преобразования** | Synthesis + Generation |
| **Семантическое сохранение** | 80-90% (contextualized answer) |
| **Гранулярность** | Structured → Natural language |
| **Направление абстракции** | Synthesis (объединение + формулировка) |
| **Информационная потеря** | Средняя (фокус на запросе) |
| **Обратимость** | Medium |

### Семантические свойства

**Синтез информации**:
```
Query: "How does GraphRAG use LLMs?"

Context (abbreviated):
  Entities: GraphRAG, LLM, GPT-4, Entity Extraction
  Relationships: GraphRAG→uses→LLM, LLM→performs→Entity Extraction
  Text Units: [3 chunks describing the process]

LLM Answer Generation:
↓
Answer:
"GraphRAG leverages Large Language Models (LLMs), specifically GPT-4,
for several key tasks in its pipeline:

1. **Entity Extraction**: LLMs analyze text chunks to identify and
   extract entities such as organizations, people, and concepts.

2. **Relationship Extraction**: The models detect and describe semantic
   relationships between entities.

3. **Description Summarization**: LLMs consolidate multiple descriptions
   of the same entity into coherent summaries.

4. **Community Report Generation**: They create structured reports
   summarizing thematic communities in the knowledge graph.

The use of LLMs enables GraphRAG to understand semantic nuances and
extract structured knowledge from unstructured text."
```

**Семантические операции**:
- **Aggregation**: Объединение информации из разных источников
- **Inference**: Логические выводы на основе контекста
- **Formulation**: Преобразование структурированных данных в естественный язык
- **Citation**: Ссылки на источники (опционально)

**Контекстуализация**:
- Ответ фокусируется на запросе
- Нерелевантная информация из контекста игнорируется
- Добавление объяснений и примеров

### LLM Промпт

**Local Search Prompt**:
```
---Role---
You are a helpful assistant responding to questions about data in the
knowledge graph.

---Goal---
Generate a response to the user's question using only the provided context.

---Context---
{context_data}

---Question---
{query}

---Instructions---
- Answer the question using ONLY the information from the context
- If the context doesn't contain enough information, say so
- Cite sources when possible
- Be concise and direct
- Use bullet points for clarity when appropriate

---Response---
```

**Global Search Map Prompt** (per community):
```
---Role---
You are a helpful assistant analyzing a community report to answer a question.

---Community Report---
{community_report}

---Question---
{query}

---Instructions---
- Identify information in this report relevant to the question
- Provide specific points that help answer the question
- Rate the relevance of this report to the question (0-10)
- Be specific and cite findings from the report

---Response---
```

**Файлы**:
- `graphrag/prompts/query/local_search.py`
- `graphrag/prompts/query/global_search.py`

### Параметры преобразования

| Параметр | Значение | Влияние на семантику |
|---|---|---|
| **LLM model** | GPT-4 | Качество synthesis |
| **temperature** | 0 | Детерминированность |
| **max_tokens** | 1500 | Длина ответа |
| **include_citations** | true | Верифицируемость |

### Роль в цепочке

**Позиция**: После Context Building

**Назначение**:
- Генерация ответа на запрос
- Синтез информации из контекста
- Формулировка в естественном языке

**Связи**:
- **Input from**: Context Building (Q4)
- **Output to**: Answer Synthesis (Q6) или Final Response
- **Core operation**: Основная генерация ответа

---

## Q6: Answer Synthesis (Multiple Answers → Final Answer)

### Описание

Объединение множественных промежуточных ответов в единый финальный ответ (используется в Global Search с Map-Reduce).

### Входные/выходные данные

```
Input:  Multiple Intermediate Answers (from Map phase)
Output: Final Synthesized Answer
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| **Тип преобразования** | Consolidation + Abstraction |
| **Семантическое сохранение** | 75-85% (key points) |
| **Гранулярность** | Multiple → Single response |
| **Направление абстракции** | Bottom-up (детали → общее) |
| **Информационная потеря** | Средняя (консолидация) |
| **Обратимость** | Medium |

### Семантические свойства

**Map-Reduce для Global Search**:

```
Query: "What are the main trends in AI research?"

MAP PHASE (Q5):
┌─────────────────────────────────────────────┐
│ Community 1 (AI Technology):                │
│ "Recent trends include transformer models,  │
│  multimodal learning, and efficiency..."    │
│  Relevance: 9/10                            │
├─────────────────────────────────────────────┤
│ Community 2 (Machine Learning):             │
│ "Key developments are in reinforcement      │
│  learning and few-shot learning..."         │
│  Relevance: 8/10                            │
├─────────────────────────────────────────────┤
│ ... (8 more community responses)            │
└─────────────────────────────────────────────┘

REDUCE PHASE (Q6):
↓ Synthesis
┌─────────────────────────────────────────────┐
│ Final Answer:                               │
│                                             │
│ "Based on analysis of 10 research areas,    │
│  the main trends in AI research include:    │
│                                             │
│  1. **Transformer Architectures**: ...      │
│  2. **Multimodal Learning**: ...            │
│  3. **Efficiency & Compression**: ...       │
│  4. **Reinforcement Learning**: ...         │
│  5. **Few-shot & Zero-shot Learning**: ...  │
│                                             │
│  These trends reflect a focus on scaling,   │
│  efficiency, and generalization."           │
└─────────────────────────────────────────────┘
```

**Семантические операции**:
- **Deduplication**: Удаление дублирующихся insights
- **Prioritization**: Фокус на высокорелевантных ответах
- **Integration**: Логическое объединение разных аспектов
- **Abstraction**: Выделение общих паттернов
- **Structuring**: Организация в coherent narrative

**Обработка противоречий**:
- Идентификация расхождений
- Указание на различные perspectives
- Приоритет более авторитетным источникам

### LLM Промпт (Reduce Phase)

```
---Role---
You are a helpful assistant synthesizing multiple perspectives to answer
a user's question.

---Question---
{query}

---Map Responses---
{map_responses}

---Instructions---
- Synthesize the information from all map responses
- Prioritize responses with higher relevance ratings
- Identify common themes and key insights
- Remove redundancy
- Create a coherent, well-structured answer
- Use specific examples from the responses
- If responses contradict, note different perspectives

---Final Answer---
```

**Файл**: `graphrag/prompts/query/global_search.py`

### Параметры преобразования

| Параметр | Значение | Влияние на семантику |
|---|---|---|
| **min_relevance** | 7.0 | Фильтр промежуточных ответов |
| **max_intermediate** | 10 | Сколько ответов синтезировать |
| **deduplication** | enabled | Устранение повторов |

### Роль в цепочке

**Позиция**: После Answer Generation (Q5 Map phase)

**Назначение**:
- Объединение множественных perspectives
- Создание coherent финального ответа
- Устранение дублирования

**Связи**:
- **Input from**: Answer Generation (Q5, multiple calls)
- **Output to**: Final Response
- **Used in**: Global Search only

---

## Q7: Follow-up Generation (Answer → Questions)

### Описание

Генерация рекомендованных follow-up вопросов для дальнейшего исследования темы (опциональное преобразование).

### Входные/выходные данные

```
Input:  Original Query + Generated Answer
Output: Follow-up Questions[] (3-5 suggested questions)
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| **Тип преобразования** | Extrapolation + Generation |
| **Семантическое сохранение** | 70-80% (context extension) |
| **Гранулярность** | Answer → Questions |
| **Направление абстракции** | Lateral (расширение, не абстракция) |
| **Информационная потеря** | N/A (создание новой информации) |
| **Обратимость** | N/A |

### Семантические свойства

**Генерация контекстуальных вопросов**:
```
Original Query: "How does GraphRAG use LLMs?"

Answer: [detailed explanation of LLM usage]

Follow-up Questions Generated:
1. "What specific LLM models does GraphRAG support?"
2. "How does entity extraction quality compare to rule-based methods?"
3. "Can GraphRAG work with open-source LLMs instead of GPT-4?"
4. "What is the cost impact of using LLMs in the GraphRAG pipeline?"
5. "How are LLM prompts optimized for entity extraction?"
```

**Типы follow-up вопросов**:

**Deepening Questions** (углубление):
- Детализация аспектов из ответа
- Механизмы и внутреннее устройство
- Примеры: "How exactly does X work?"

**Broadening Questions** (расширение):
- Смежные темы
- Альтернативные подходы
- Примеры: "What about Y related to X?"

**Comparative Questions** (сравнение):
- Сравнение с альтернативами
- Trade-offs
- Примеры: "How does X compare to Y?"

**Practical Questions** (практические):
- Use cases
- Конфигурация и настройка
- Примеры: "How do I configure X?"

### LLM Промпт

```
---Role---
You are a helpful assistant generating follow-up questions to guide
further exploration.

---Original Question---
{query}

---Answer Provided---
{answer}

---Instructions---
Generate 3-5 follow-up questions that:
- Deepen understanding of topics mentioned in the answer
- Explore related areas not fully covered
- Are specific and actionable
- Progress logically from the current answer
- Vary in type (deepening, broadening, comparative, practical)

---Follow-up Questions---
```

### Параметры преобразования

| Параметр | Значение | Влияние на семантику |
|---|---|---|
| **num_questions** | 3-5 | Количество вариантов |
| **question_types** | mixed | Разнообразие типов |
| **specificity** | high | Конкретность вопросов |

### Роль в цепочке

**Позиция**: Финальное опциональное преобразование

**Назначение**:
- Guided exploration
- Улучшение UX
- Обнаружение смежных тем

**Связи**:
- **Input from**: Answer Generation (Q5) или Answer Synthesis (Q6)
- **Output to**: User (displayed with answer)
- **Used in**: All search types (опционально)

---

## Сравнительная таблица преобразований

| Преобразование | Тип | Semantic Preservation | Information Loss | Reversibility | LLM Required |
|---|---|---|---|---|---|
| Q1: Query Analysis | Classification | 90-95% | Minimal | High | No |
| Q2: Query Embedding | Vectorization | 90-95% | Low | Low | No |
| Q3: Semantic Retrieval | Retrieval | 85-90% | Medium | Medium | No |
| Q4: Context Building | Aggregation | 95%+ | Minimal | High | No |
| Q5: Answer Generation | Synthesis | 80-90% | Medium | Medium | Yes |
| Q6: Answer Synthesis | Consolidation | 75-85% | Medium | Medium | Yes |
| Q7: Follow-up Generation | Extrapolation | 70-80% | N/A | N/A | Yes |

## Композиция преобразований

### Global Search Chain

```
User Query
  →[Q1: 95%]→ Intent
  →[Q2: 92%]→ Vector
  →[Q3: 88%]→ Community Reports
  →[Q4: 98%]→ Contexts (10x)
  →[Q5 Map: 85%]→ Intermediate Answers (10x)
  →[Q6 Reduce: 80%]→ Final Answer

Overall preservation: 0.95 * 0.92 * 0.88 * 0.98 * 0.85 * 0.80 ≈ 0.55 (55%)
```

**Интерпретация**: 55% сохранение означает, что ответ содержит высокоуровневые insights, но теряет конкретные детали (что ожидаемо для обзорных вопросов).

### Local Search Chain

```
User Query
  →[Q1: 95%]→ Intent
  →[Q2: 92%]→ Vector
  →[Q3: 88%]→ Entities + Text Units
  →[Q4: 98%]→ Detailed Context
  →[Q5: 85%]→ Answer

Overall preservation: 0.95 * 0.92 * 0.88 * 0.98 * 0.85 ≈ 0.66 (66%)
```

**Интерпретация**: 66% сохранение выше, чем у Global, так как используются оригинальные text_units с конкретными деталями.

### Drift Search Chain (per iteration)

```
User Query
  →[Q1: 95%]→ Intent
  →[Q2: 92%]→ Vector
  →[Q3: 88%]→ Start Entities
  →[Q4: Graph Walk]→ Expanded Context
  →[Q5: 85%]→ Intermediate Answer
  (iterate 3-5 times)
  →[Q6: Optional Synthesis]→ Final Answer

Per hop preservation: ~65%
After 3 hops: 0.65³ ≈ 0.27 (27%)
```

**Интерпретация**: Низкое сохранение из-за многократных итераций, но это компенсируется обнаружением неочевидных связей.

## Критические точки информационной потери

### Q3: Semantic Retrieval
**Потеря**: ~12% при отборе top-k узлов
- Митигация: Увеличение top_k, hybrid ranking

### Q5: Answer Generation
**Потеря**: ~15% при синтезе ответа
- Митигация: Увеличение max_context_tokens, лучшие промпты

### Q6: Answer Synthesis
**Потеря**: ~20% при reduce phase
- Митигация: Приоритизация high-relevance ответов

## Оптимизация цепочек

### Для быстрых ответов (Local Search)

```yaml
Оптимизация:
  - Кэширование query embeddings
  - Ограничение top_k до 15
  - Single LLM call
  - Минимальный контекст

Результат:
  - Время: 3-5 сек
  - Preservation: 65%
  - Стоимость: низкая
```

### Для глубоких ответов (Global Search)

```yaml
Оптимизация:
  - Параллельные LLM calls (Map phase)
  - Увеличение top_k до 15
  - Больше контекста (20k tokens)
  - Тщательный Reduce

Результат:
  - Время: 15-25 сек
  - Preservation: 55% (но высокоуровневые insights)
  - Стоимость: высокая
```

### Для exploratory исследований (Drift Search)

```yaml
Оптимизация:
  - Адаптивный max_hops (останавливаться при низкой relevance)
  - Pruning слабых relationships
  - Кэширование промежуточных hops
  - Follow-up generation для guided exploration

Результат:
  - Время: 30-60 сек
  - Preservation: 27% (но обнаружение связей)
  - Стоимость: средняя-высокая
```

## Следующие разделы

- **[Transformation Chains →](03-transformation-chains.md)**
- **[Comparison Tables →](04-comparison-tables.md)**
- **[← Indexing Transformations](01-indexing-transformations.md)**
