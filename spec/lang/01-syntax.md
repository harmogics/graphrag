# GFL Syntax Reference

## Грамматика языка

### Базовая структура

```ebnf
Program        ::= (Flow | Component | DataType)*

Flow           ::= "FLOW" Identifier "(" Parameters? ")" ":" Statement*

Component      ::= "COMPONENT" Identifier "(" Parameters ")" ":" Statement*

DataType       ::= "TYPE" Identifier "{" Field* "}"

Statement      ::= Intention | Control | Assignment | Return

Intention      ::= IntentionKeyword Subject Action Object Modifiers?

IntentionKeyword ::= "INGEST" | "CHUNK" | "EXTRACT" | "AGGREGATE"
                   | "CLUSTER" | "EMBED" | "INDEX" | "QUERY"
                   | "RETRIEVE" | "SYNTHESIZE" | "BUILD" | "GENERATE"
                   | "CALL" | "APPLY" | "TRANSFORM"

Modifiers      ::= ("WITH" Parameters)? ("USING" Tool)? ("INTO" Target)?
                   ("FROM" Source)? ("YIELD" Variables)?

Control        ::= If | For | Route | When

Route          ::= "ROUTE" Expression ":" Case+

Case           ::= "WHEN" Condition "THEN" Flow | "DEFAULT" Flow

Parameters     ::= Param ("," Param)*
Param          ::= Identifier "=" Value
```

## Базовые интенции

### 1. INGEST — Загрузка данных

```gfl
INGEST <what> FROM <source> [WITH <params>] [YIELD <output>]

// Примеры:
INGEST documents FROM ./input/*.txt
INGEST documents FROM ./input WITH encoding=utf-8 YIELD raw_documents
INGEST data FROM database WHERE type="article"
```

**Параметры**:
- `encoding`: Кодировка файлов (default: utf-8)
- `file_pattern`: Glob pattern
- `filter`: Условие фильтрации

**Output**: Коллекция Documents

### 2. CHUNK — Разбиение на фрагменты

```gfl
CHUNK <input> INTO <output> WITH <strategy> [YIELD <units>]

// Примеры:
CHUNK documents INTO text_units WITH size=1200, overlap=100
CHUNK text WITH strategy=tokens USING tokenizer=cl100k_base
CHUNK documents INTO units WITH strategy=sentence
```

**Параметры**:
- `strategy`: tokens | sentence | paragraph
- `size`: Размер chunk в tokens (для strategy=tokens)
- `overlap`: Перекрытие в tokens
- `tokenizer`: cl100k_base | gpt2 | ...

**Output**: Коллекция TextUnits

### 3. EXTRACT — Извлечение информации

```gfl
EXTRACT <targets> FROM <source> USING <tool> [WITH <params>]

// Примеры:
EXTRACT entities, relationships FROM text_units USING llm
EXTRACT entities FROM text WITH entity_types=[person, organization]
EXTRACT claims FROM documents USING llm WITH prompt=ClaimsPrompt
```

**Параметры**:
- `entity_types`: Список типов entities
- `prompt`: LLM промпт
- `max_gleanings`: Количество итераций gleaning
- `temperature`: LLM temperature

**Output**: Entities, Relationships, Claims, etc.

### 4. AGGREGATE — Агрегация/группировка

```gfl
AGGREGATE <collection> BY <key> INTO <output> [YIELD <result>]

// Примеры:
AGGREGATE entities BY name INTO entity_groups
AGGREGATE descriptions BY entity_id INTO grouped_descriptions
AGGREGATE answers BY relevance_score DESC INTO ranked_answers
```

**Параметры**:
- `BY <key>`: Ключ группировки
- `INTO <output>`: Имя результата
- Опционально: `WITH reducer=<function>`

**Output**: Коллекция групп

### 5. CLUSTER — Кластеризация

```gfl
CLUSTER <nodes> INTO <clusters> USING <algorithm> [WITH <params>]

// Примеры:
CLUSTER entities INTO communities USING leiden
CLUSTER entities WITH resolution=1.0, max_cluster_size=10
CLUSTER graph INTO hierarchical_communities WITH levels=3
```

**Параметры**:
- `algorithm`: leiden | louvain | hierarchical
- `resolution`: Resolution parameter (default: 1.0)
- `max_cluster_size`: Максимальный размер кластера
- `levels`: Количество иерархических уровней

**Output**: Communities

### 6. EMBED — Векторизация

```gfl
EMBED <data> INTO <vectors> USING <model> [WITH <params>]

// Примеры:
EMBED text_units INTO vectors
EMBED text_units, entities USING model=text-embedding-3-small
EMBED documents WITH dimensions=1536, batch_size=100
```

**Параметры**:
- `model`: text-embedding-3-small | text-embedding-3-large | ...
- `dimensions`: 1536 | 3072 | ...
- `batch_size`: Размер batch для embedding
- `normalize`: true | false (L2 normalization)

**Output**: Vectors (Embeddings)

### 7. INDEX — Индексация

```gfl
INDEX <data> INTO <store> [WITH <params>]

// Примеры:
INDEX vectors, entities, relationships INTO graph_store
INDEX embeddings INTO vector_store WITH store_type=lancedb
INDEX ALL INTO knowledge_graph
```

**Параметры**:
- `store_type`: lancedb | faiss | azure_search | ...
- `index_type`: hnsw | flat | ivf
- `metric`: cosine | euclidean | dot_product

**Output**: Indexed структуры

### 8. QUERY — Формирование запроса

```gfl
QUERY <query_text> [YIELD <query_object>]

// Примеры:
QUERY "What is GraphRAG?"
QUERY user_input YIELD query
QUERY text WITH expansion=true, rewrite=true
```

**Параметры**:
- `expansion`: Расширение запроса
- `rewrite`: Переформулировка
- `language`: Язык запроса

**Output**: Query object

### 9. RETRIEVE — Поиск данных

```gfl
RETRIEVE <targets> MATCHING <query> [WITH <params>] [YIELD <results>]

// Примеры:
RETRIEVE entities MATCHING query_vector WITH top_k=30
RETRIEVE community_reports MATCHING query WITH similarity_threshold=0.7
RETRIEVE entities, text_units WITH hybrid_search=true
```

**Параметры**:
- `top_k`: Количество результатов
- `similarity_threshold`: Порог similarity
- `hybrid_search`: Комбинация vector + keyword search
- `filters`: Дополнительные фильтры

**Output**: Ranked results

### 10. SYNTHESIZE — Синтез ответа

```gfl
SYNTHESIZE <output> FROM <input> USING <tool> [WITH <params>]

// Примеры:
SYNTHESIZE answer FROM context USING llm
SYNTHESIZE final_answer FROM ranked_answers WITH prompt=ReducePrompt
SYNTHESIZE report FROM community WITH format=json
```

**Параметры**:
- `prompt`: LLM промпт
- `format`: json | markdown | text
- `max_tokens`: Максимальная длина output
- `temperature`: LLM temperature

**Output**: Synthesized результат

## Дополнительные операции

### BUILD — Построение структур

```gfl
BUILD <structure> FROM <components> [YIELD <result>]

// Примеры:
BUILD graph FROM entities, relationships
BUILD context FROM entities, text_units, relationships WITH max_tokens=8000
BUILD knowledge_graph FROM indexed_data
```

### GENERATE — Генерация контента

```gfl
GENERATE <output> FROM <input> [USING <tool>] [WITH <params>]

// Примеры:
GENERATE report FROM community USING llm
GENERATE follow_up_questions FROM answer WITH count=5
GENERATE summary FROM text WITH max_length=500
```

### CALL — Вызов внешних сервисов

```gfl
CALL <service> WITH <params> [YIELD <result>]

// Примеры:
CALL llm WITH prompt=EntityExtractionPrompt, context=text_unit
CALL embedding_service WITH text=documents, model=text-embedding-3-small
CALL rate_limiter WITH tokens_per_minute=50000
```

### TRANSFORM — Общее преобразование

```gfl
TRANSFORM <input> INTO <output> [USING <method>] [WITH <params>]

// Примеры:
TRANSFORM documents INTO knowledge_graph
TRANSFORM text_units USING EntityExtractionPipeline
TRANSFORM entities WITH consolidation=true
```

### APPLY — Применение функции/компонента

```gfl
APPLY <function> TO <data> [WITH <params>] [YIELD <result>]

// Примеры:
APPLY EntityExtraction TO text_units
APPLY Deduplication TO entities WITH threshold=0.95
APPLY Filtering TO results WITH min_score=0.7
```

## Управляющие конструкции

### IF — Условное выполнение

```gfl
IF <condition> THEN:
  <statements>
[ELIF <condition> THEN:
  <statements>]*
[ELSE:
  <statements>]

// Пример:
IF entity_count > 1000 THEN:
  APPLY Sampling WITH ratio=0.5
ELIF entity_count > 500 THEN:
  APPLY Filtering WITH threshold=0.8
ELSE:
  PROCESS ALL entities
```

### FOR — Итерация

```gfl
FOR EACH <item> IN <collection>:
  <statements>

// Примеры:
FOR EACH text_unit IN text_units:
  EXTRACT entities FROM text_unit
  STORE entities WITH provenance=text_unit.id

FOR EACH community IN communities:
  GENERATE report FROM community
  YIELD community.report
```

### WHILE — Цикл с условием

```gfl
WHILE <condition>:
  <statements>

// Пример:
WHILE relevance_score < threshold AND iteration < max_iterations:
  RETRIEVE additional_context
  REFINE answer
  EVALUATE relevance_score
```

### ROUTE — Маршрутизация

```gfl
ROUTE <expression>:
  WHEN <condition> THEN <flow>
  WHEN <condition> THEN <flow>
  DEFAULT <flow>

// Примеры:
ROUTE query_type:
  WHEN "global" THEN GlobalSearchFlow
  WHEN "local" THEN LocalSearchFlow
  WHEN "drift" THEN DriftSearchFlow
  DEFAULT LocalSearchFlow

ROUTE query:
  WHEN query CONTAINS "overview", "trends" THEN GlobalSearch
  WHEN query CONTAINS "specific", "how does" THEN LocalSearch
  WHEN query CONTAINS "related", "connection" THEN DriftSearch
  DEFAULT LocalSearch
```

### MATCH — Pattern matching

```gfl
MATCH <value>:
  CASE <pattern> => <action>
  CASE <pattern> => <action>
  DEFAULT => <action>

// Пример:
MATCH entity.type:
  CASE "organization" => APPLY OrganizationEnrichment
  CASE "person" => APPLY PersonEnrichment
  CASE "geo" => APPLY GeoEnrichment
  DEFAULT => APPLY GenericEnrichment
```

## Модификаторы и клаузы

### WITH — Параметры

```gfl
<statement> WITH <param1>=<value1>, <param2>=<value2>, ...

// Примеры:
CHUNK documents WITH size=1200, overlap=100, strategy=tokens
EXTRACT entities WITH entity_types=[person, org], max_gleanings=2
EMBED text WITH model=text-embedding-3-small, batch_size=100
```

### USING — Инструмент/модель

```gfl
<statement> USING <tool>

// Примеры:
EXTRACT entities USING llm=gpt-4
EMBED text USING model=text-embedding-3-small
CLUSTER entities USING algorithm=leiden
CHUNK text USING tokenizer=cl100k_base
```

### FROM — Источник данных

```gfl
<statement> FROM <source>

// Примеры:
INGEST documents FROM ./input/*.txt
EXTRACT entities FROM text_units
SYNTHESIZE answer FROM context
BUILD graph FROM entities, relationships
```

### INTO — Целевой объект

```gfl
<statement> INTO <target>

// Примеры:
CHUNK documents INTO text_units
AGGREGATE entities INTO groups
INDEX vectors INTO vector_store
CLUSTER entities INTO communities
```

### YIELD — Выходные данные

```gfl
<statement> YIELD <output_vars>

// Примеры:
INGEST documents YIELD raw_documents
EXTRACT entities, relationships FROM text YIELD extracted_data
RETRIEVE entities MATCHING query YIELD relevant_entities
```

### WHERE — Фильтрация

```gfl
<statement> WHERE <condition>

// Примеры:
RETRIEVE entities WHERE type="organization"
INGEST documents FROM source WHERE date > "2024-01-01"
FILTER results WHERE score > 0.8
```

## Аннотации и метаданные

### @-Аннотации

```gfl
<statement>
  @preserves: <percentage>        // Semantic preservation
  @loss: <level>                   // Information loss
  @reversible: <level>             // Reversibility
  @cost: <level>                   // Computational cost
  @latency: <duration>             // Expected latency
  @quality: <metrics>              // Quality metrics

// Пример:
EXTRACT entities FROM text
  @preserves: 70-85%
  @loss: medium
  @reversible: low
  @cost: high
  @latency: 2-5s
  @quality: {precision: 0.92, recall: 0.85}
```

### ENSURE — Ограничения качества

```gfl
<statement>
  ENSURE <constraint>

// Примеры:
EXTRACT entities
  ENSURE precision >= 0.90
  ENSURE recall >= 0.75
  ENSURE hallucination_rate < 0.10

RETRIEVE entities
  ENSURE similarity_score > 0.7
  ENSURE result_count >= 10
```

### WITHIN — Ограничения производительности

```gfl
<statement>
  WITHIN <constraint>

// Примеры:
CALL llm
  WITHIN latency < 5s
  WITHIN cost < $0.01 per call
  WITHIN memory < 1GB

INDEX vectors
  WITHIN build_time < 5min
  WITHIN index_size < 10GB
```

### REASON — Объяснение

```gfl
<statement>
  REASON "<explanation>"

// Пример:
CHUNK documents WITH size=800, overlap=150
  REASON "Smaller chunks for precision in technical content"

CLUSTER entities WITH max_cluster_size=8
  REASON "Smaller communities for fine-grained topics"
```

## Типы данных

### Примитивные типы

```gfl
String      // "text"
Integer     // 42
Float       // 3.14
Boolean     // true, false
List[T]     // [item1, item2, ...]
Map[K,V]    // {key1: value1, key2: value2}
Vector      // [0.1, 0.2, ..., 0.n]
```

### Составные типы

```gfl
TYPE Document {
  text: String
  metadata: Map[String, Any]
  source: URI
  id: String
}

TYPE TextUnit {
  text: String
  n_tokens: Integer
  source_doc: Reference<Document>
  chunk_index: Integer
}

TYPE Entity {
  name: String
  type: EntityType
  description: String
  embeddings: Vector
  source_chunks: List[Reference<TextUnit>]
}

TYPE Relationship {
  source: Reference<Entity>
  target: Reference<Entity>
  description: String
  strength: Float
}

TYPE Community {
  id: String
  entities: List[Reference<Entity>]
  level: Integer
  report: CommunityReport?
}

TYPE Query {
  text: String
  intent: QueryIntent
  vector: Vector
  metadata: Map[String, Any]
}
```

### Enum типы

```gfl
ENUM EntityType {
  ORGANIZATION,
  PERSON,
  GEO,
  EVENT,
  TECHNOLOGY,
  OTHER
}

ENUM QueryIntent {
  GLOBAL,    // Overview, trends
  LOCAL,     // Specific details
  DRIFT,     // Explore connections
  BASIC      // Simple lookup
}

ENUM ChunkStrategy {
  TOKENS,
  SENTENCE,
  PARAGRAPH
}
```

## Операторы

### Сравнение

```gfl
==    // Равно
!=    // Не равно
<     // Меньше
>     // Больше
<=    // Меньше или равно
>=    // Больше или равно
```

### Логические

```gfl
AND   // Логическое И
OR    // Логическое ИЛИ
NOT   // Логическое НЕ
```

### Строковые

```gfl
CONTAINS     // Содержит подстроку
MATCHES      // Соответствует regex
STARTS_WITH  // Начинается с
ENDS_WITH    // Заканчивается на
```

### Коллекции

```gfl
IN           // Присутствует в коллекции
NOT IN       // Отсутствует в коллекции
EMPTY        // Коллекция пуста
SIZE         // Размер коллекции
```

## Комментарии

```gfl
// Однострочный комментарий

/*
  Многострочный
  комментарий
*/

FLOW Example:
  // This is a comment
  INGEST documents  /* inline comment */ FROM source
```

## Именование

### Conventions

```gfl
// Flows: PascalCase
FLOW EntityExtractionPipeline:
  ...

// Components: PascalCase
COMPONENT EmbeddingService:
  ...

// Variables: snake_case
text_units
consolidated_entities
query_vector

// Constants: UPPER_SNAKE_CASE
MAX_CHUNK_SIZE = 1200
DEFAULT_OVERLAP = 100

// Types: PascalCase
TYPE Entity { ... }
```

## Примеры синтаксиса

### Простой flow

```gfl
FLOW SimpleIndexing:
  INGEST documents FROM ./input
  CHUNK documents INTO text_units WITH size=1200
  EXTRACT entities FROM text_units USING llm
  INDEX entities INTO graph
```

### Flow с параметрами

```gfl
FLOW ParameterizedIndexing(chunk_size, entity_types):
  INGEST documents
  CHUNK WITH size=chunk_size
  EXTRACT entities WITH entity_types=entity_types
  INDEX ALL
```

### Flow с условиями

```gfl
FLOW AdaptiveIndexing:
  INGEST documents YIELD docs

  IF SIZE(docs) > 1000 THEN:
    APPLY Sampling WITH ratio=0.5
  ELSE:
    PROCESS ALL

  CHUNK INTO text_units
  EXTRACT entities
  INDEX ALL
```

### Flow с циклами

```gfl
FLOW IterativeExtraction:
  CHUNK documents INTO text_units

  FOR EACH unit IN text_units:
    EXTRACT entities FROM unit YIELD extracted
    STORE extracted WITH provenance=unit.id

  AGGREGATE ALL entities BY name
  INDEX INTO graph
```

### Композиция flows

```gfl
COMPONENT EntityPipeline(text):
  EXTRACT entities FROM text
  AGGREGATE BY name
  YIELD consolidated_entities

FLOW MainFlow:
  CHUNK documents
  APPLY EntityPipeline TO text_units
  INDEX entities
```

## Связь с GraphRAG

GFL синтаксис напрямую мапится на GraphRAG концепции:

| GFL | GraphRAG Concept |
|---|---|
| `CHUNK` | T1: Text Chunking |
| `EXTRACT entities` | T2: Entity Extraction |
| `EXTRACT relationships` | T3: Relationship Extraction |
| `AGGREGATE BY name` | T4: Description Summarization |
| `CLUSTER` | T5: Community Detection |
| `GENERATE report` | T6: Community Report |
| `EMBED` | T7: Text Embedding |
| `ANALYZE query` | Q1: Query Analysis |
| `EMBED query` | Q2: Query Embedding |
| `RETRIEVE` | Q3: Semantic Retrieval |
| `BUILD context` | Q4: Context Building |
| `SYNTHESIZE answer` | Q5: Answer Generation / Q6: Synthesis |

---

**Next**: [Indexing Flows Examples](02-indexing-flows.md)
