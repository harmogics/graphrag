# GraphRAG Flow Language (GFL)

## Обзор

**GraphRAG Flow Language (GFL)** — это декларативный метаязык для описания информационных потоков в системе GraphRAG. Язык семантически близок к английскому и использует небольшое число базовых интенций для навигации по трансформациям данных.

```gfl
// Пример простого flow
FLOW IndexDocument:
  INGEST document FROM source
  CHUNK document INTO text_units WITH size=1200, overlap=100
  EXTRACT entities, relationships FROM text_units USING llm
  AGGREGATE entities BY name INTO consolidated_entities
  EMBED text_units, entities INTO vectors
  INDEX vectors, entities, relationships INTO graph
```

## Философия языка

GFL следует принципам:

1. **Semantic Clarity** — каждая инструкция читается как естественный английский
2. **Intentional Design** — фокус на намерениях (WHAT), не на реализации (HOW)
3. **Flow-Oriented** — описание потоков данных, не императивных команд
4. **Composable** — возможность композиции flows
5. **Traceable** — явная прослеживаемость трансформаций

## Базовые интенции

GFL построен на **10 базовых интенциях**:

| Интенция | Назначение | Пример |
|---|---|---|
| **INGEST** | Загрузка данных | `INGEST documents FROM ./input` |
| **CHUNK** | Разбиение на фрагменты | `CHUNK text INTO units WITH size=1200` |
| **EXTRACT** | Извлечение информации | `EXTRACT entities FROM text USING llm` |
| **AGGREGATE** | Агрегация/группировка | `AGGREGATE descriptions BY entity` |
| **CLUSTER** | Кластеризация | `CLUSTER entities INTO communities` |
| **EMBED** | Векторизация | `EMBED text INTO vectors` |
| **INDEX** | Индексация | `INDEX entities INTO graph` |
| **QUERY** | Формирование запроса | `QUERY "What is GraphRAG?"` |
| **RETRIEVE** | Поиск данных | `RETRIEVE entities MATCHING query` |
| **SYNTHESIZE** | Синтез ответа | `SYNTHESIZE answer FROM context` |

## Структура языка

### Flows (Потоки)

```gfl
FLOW <name>:
  <intention> <subject> <action> <object> [WITH <params>]
  ...
```

### Conditional Routing

```gfl
ROUTE query:
  WHEN query CONTAINS "overview", "trends" THEN GlobalSearch
  WHEN query CONTAINS "specific", "detail" THEN LocalSearch
  DEFAULT DriftSearch
```

### Transformations

```gfl
TRANSFORM text_units:
  APPLY EntityExtraction WITH types=[organization, person, geo]
  YIELD entities, relationships
  PRESERVE provenance
```

### Data Objects

```gfl
Document {
  text: String
  metadata: Map
  source: URI
}

TextUnit {
  text: String
  n_tokens: Integer
  source_doc: Reference<Document>
}

Entity {
  name: String
  type: EntityType
  description: String
  embeddings: Vector
}
```

## Семантические уровни

GFL описывает потоки на трех уровнях:

### Level 1: High-Level Intent

```gfl
FLOW BuildKnowledgeGraph:
  INGEST documents
  TRANSFORM INTO knowledge_graph
  ENABLE semantic_search
```

### Level 2: Transformation Pipeline

```gfl
FLOW Indexing:
  INGEST documents FROM source
  CHUNK INTO text_units
  EXTRACT entities, relationships
  CLUSTER INTO communities
  GENERATE reports
  EMBED ALL
  INDEX INTO graph
```

### Level 3: Detailed Operations

```gfl
FLOW EntityExtractionDetailed:
  FOR EACH text_unit IN text_units:
    CALL llm WITH prompt=EntityExtractionPrompt,
                   params={entity_types, temperature=0.0}
    PARSE response INTO entities, relationships
    IF max_gleanings > 0:
      REFINE entities WITH gleaning_iterations=max_gleanings
    STORE entities, relationships WITH provenance=text_unit.id
```

## Пример: Полный Indexing Flow

```gfl
FLOW DocumentIndexing:
  // Phase 1: Ingestion
  INGEST documents FROM ./input/*.txt
    WITH encoding=utf-8
    YIELD raw_documents

  // Phase 2: Chunking
  CHUNK raw_documents
    INTO text_units
    WITH strategy=tokens, size=1200, overlap=100
    USING tokenizer=cl100k_base
    YIELD text_units

  // Phase 3: Entity Extraction
  EXTRACT entities, relationships
    FROM text_units
    USING llm=gpt-4
    WITH prompt=EntityExtractionPrompt,
         entity_types=[organization, person, geo, event],
         max_gleanings=1,
         temperature=0.0
    YIELD raw_entities, raw_relationships

  // Phase 4: Description Consolidation
  AGGREGATE raw_entities
    BY name
    INTO entity_groups
    YIELD entity_groups

  SYNTHESIZE entity_groups
    INTO consolidated_entities
    USING llm=gpt-4
    WITH prompt=SummarizationPrompt,
         max_length=500
    YIELD consolidated_entities

  // Phase 5: Community Detection
  BUILD graph FROM consolidated_entities, raw_relationships
    YIELD entity_graph

  CLUSTER entity_graph
    INTO communities
    USING algorithm=leiden,
          resolution=1.0
    YIELD communities

  // Phase 6: Community Reports
  FOR EACH community IN communities:
    GENERATE report
      FROM community.entities, community.relationships
      USING llm=gpt-4
      WITH prompt=CommunityReportPrompt,
           max_length=2000
      YIELD community.report

  // Phase 7: Embedding
  EMBED text_units, consolidated_entities, communities
    INTO vectors
    USING model=text-embedding-3-small,
          dimensions=1536
    YIELD embeddings

  // Phase 8: Indexing
  INDEX embeddings, entity_graph, communities
    INTO vector_store, graph_store
    WITH store_type=lancedb
    YIELD indexed_knowledge_graph

  RETURN indexed_knowledge_graph
```

## Пример: Query Execution Flow

```gfl
FLOW QueryExecution:
  // Phase 1: Query Analysis
  QUERY user_input="What are the main trends in AI?"
    YIELD query

  ANALYZE query
    EXTRACT intent, key_concepts, scope
    CLASSIFY query_type IN [global, local, drift]
    YIELD query_metadata

  // Phase 2: Route to Search Strategy
  ROUTE query_metadata.query_type:
    WHEN "global" THEN GlobalSearchFlow
    WHEN "local" THEN LocalSearchFlow
    WHEN "drift" THEN DriftSearchFlow

FLOW GlobalSearchFlow(query):
  // Map Phase
  EMBED query INTO query_vector

  RETRIEVE community_reports
    MATCHING query_vector
    WITH top_k=10, similarity_threshold=0.7
    YIELD relevant_reports

  FOR EACH report IN relevant_reports:
    CALL llm WITH prompt=GlobalMapPrompt,
                   context=report,
                   query=query
    YIELD intermediate_answer, relevance_score
    STORE intermediate_answer

  // Reduce Phase
  AGGREGATE intermediate_answers
    RANKED BY relevance_score DESC
    YIELD ranked_answers

  SYNTHESIZE final_answer
    FROM ranked_answers
    USING llm=gpt-4
    WITH prompt=GlobalReducePrompt,
         response_type="comprehensive summary"
    YIELD final_answer

  RETURN final_answer

FLOW LocalSearchFlow(query):
  EMBED query INTO query_vector

  RETRIEVE entities, text_units
    MATCHING query_vector
    WITH top_k_entities=30,
         top_k_text_units=20,
         hybrid_search=true
    YIELD relevant_entities, relevant_text_units

  EXPAND relevant_entities
    WITH relationships, connected_entities
    YIELD expanded_context

  BUILD context
    FROM relevant_entities, relevant_text_units, expanded_context
    WITH max_tokens=8000
    YIELD structured_context

  SYNTHESIZE answer
    FROM structured_context
    USING llm=gpt-4
    WITH prompt=LocalSearchPrompt,
         query=query,
         include_citations=true
    YIELD answer_with_sources

  RETURN answer_with_sources
```

## Композиция и переиспользование

```gfl
// Определение переиспользуемых компонентов
COMPONENT EmbeddingPipeline(data):
  EMBED data
    USING model=text-embedding-3-small
    WITH batch_size=100
  YIELD vectors

COMPONENT LLMCall(prompt, context, params):
  CALL llm=gpt-4
    WITH prompt=prompt,
         context=context,
         temperature=params.temperature,
         max_tokens=params.max_tokens
  WITH rate_limit=50000 tokens/min
  WITH retry_strategy=exponential_backoff
  YIELD response

// Использование компонентов
FLOW IndexWithComponents:
  CHUNK documents INTO text_units
  EXTRACT entities FROM text_units
    USING LLMCall(EntityExtractionPrompt, text_units, {temperature: 0.0})
  EMBED entities
    USING EmbeddingPipeline(entities)
  INDEX ALL
```

## Семантические аннотации

```gfl
FLOW EntityExtraction:
  EXTRACT entities FROM text_units
    USING llm

    // Semantic annotations
    @preserves: 70-85%  // Semantic preservation
    @loss: medium       // Information loss level
    @reversible: low    // Can't reconstruct input
    @cost: high         // LLM calls expensive

    WITH prompt=EntityExtractionPrompt,
         entity_types=[organization, person, geo, event]

    // Quality constraints
    ENSURE precision >= 0.90
    ENSURE recall >= 0.75

    // Performance constraints
    WITHIN latency < 5s
    WITHIN cost < $0.01 per chunk

  YIELD entities, relationships
```

## Документация

Полная документация языка:

1. **[Syntax Reference](01-syntax.md)** — синтаксис и грамматика
2. **[Indexing Flows](02-indexing-flows.md)** — примеры индексации
3. **[Query Flows](03-query-flows.md)** — примеры query execution
4. **[Semantics](04-semantics.md)** — семантика выполнения
5. **[Examples](05-examples.md)** — полные сценарии
6. **[Extensions](06-extensions.md)** — расширения языка

## Связь с GraphRAG

GFL является **формальным описанием** информационных потоков, описанных в:

- **[Transformation Chains](../transform/)** — T1-T7, Q1-Q7
- **[Prompts](../prompts/)** — LLM промпты
- **[Chunking Strategies](../chunking-strategies.md)** — стратегии chunking
- **[Node Types](../nodes/)** — типы узлов графа

## Применение

GFL может использоваться для:

1. **Документирования** существующих flows
2. **Проектирования** новых flows
3. **Анализа** производительности
4. **Оптимизации** трансформаций
5. **Генерации кода** (потенциально)
6. **Визуализации** pipeline

## Пример использования

```gfl
// Define custom indexing flow
FLOW ScientificPapersIndexing:
  INGEST papers FROM ./papers/*.pdf
    WITH parser=pdf_scientific

  CHUNK papers
    WITH strategy=tokens, size=800, overlap=150
    REASON "Smaller chunks for precision in technical content"

  EXTRACT entities
    WITH entity_types=[method, metric, dataset, researcher, finding]
    REASON "Domain-specific entity types for scientific papers"

  CLUSTER entities
    WITH max_cluster_size=8
    REASON "Smaller communities for fine-grained topics"

  EMBED ALL
  INDEX INTO knowledge_graph

// Execute query with custom routing
QUERY "Compare BERT and GPT performance on SQuAD":
  ANALYZE intent => comparison_query
  ROUTE TO HybridSearch  // Custom routing logic
  SYNTHESIZE WITH format=comparison_table
```

## Преимущества GFL

✅ **Declarative** — описание WHAT, не HOW
✅ **Readable** — близко к естественному языку
✅ **Composable** — переиспользование компонентов
✅ **Traceable** — явная прослеживаемость
✅ **Semantic** — сохранение семантики трансформаций
✅ **Optimizable** — возможность анализа и оптимизации
✅ **Extensible** — легкое добавление новых интенций

## Следующие шаги

1. Изучить [синтаксис](01-syntax.md) языка
2. Просмотреть [примеры flows](05-examples.md)
3. Понять [семантику выполнения](04-semantics.md)

---

**Версия**: GFL 1.0
**Статус**: Specification
**Авторы**: Based on GraphRAG conceptual documentation
