# Semantic Flow Language (SFL)

## Обзор

**Semantic Flow Language (SFL)** — это декларативный метаязык для описания **семантических трансформаций** в системе GraphRAG. В отличие от GFL (GraphRAG Flow Language), который описывает операции с данными, SFL фокусируется на **семантических преобразованиях** информации.

```sfl
// Пример семантического flow
SEMANTIC FLOW DocumentToKnowledge:
  // Семантическое разложение
  DECOMPOSE document INTO semantic_units
    PRESERVING context_continuity=0.95

  // Абстрагирование концептов
  ABSTRACT semantic_units INTO concepts, relations
    PRESERVING semantic_fidelity=0.78

  // Консолидация семантики
  CONSOLIDATE concepts BY semantic_equivalence
    PRESERVING distinctiveness=0.87

  // Организация в структуры
  ORGANIZE concepts INTO semantic_hierarchy
    PRESERVING relationships=0.92

  // Кодирование в векторное пространство
  ENCODE semantic_hierarchy INTO embedding_space
    PRESERVING semantic_similarity=0.91
```

## Философия языка

SFL следует принципам **семантической трансформации**:

1. **Semantic Preservation** — отслеживание сохранения семантики через трансформации
2. **Intentional Abstraction** — фокус на семантических намерениях, не на технической реализации
3. **Information Flow** — описание потока семантической информации
4. **Loss Awareness** — явное указание потерь информации на каждом шаге
5. **Composability** — композиция семантических трансформаций
6. **Traceability** — прослеживаемость семантики от источника до результата

## Базовые семантические интенции

SFL построен на **10 семантических интенциях**:

| Интенция | Семантическое действие | Пример |
|---|---|---|
| **DECOMPOSE** | Разложение на семантические единицы | `DECOMPOSE text INTO semantic_units` |
| **ABSTRACT** | Извлечение концептов и сущностей | `ABSTRACT units INTO concepts` |
| **CONNECT** | Установление семантических связей | `CONNECT concepts BY relationships` |
| **CONSOLIDATE** | Консолидация эквивалентной семантики | `CONSOLIDATE entities BY equivalence` |
| **ORGANIZE** | Организация в структуры | `ORGANIZE concepts INTO hierarchy` |
| **ENCODE** | Кодирование в векторное пространство | `ENCODE concepts INTO embeddings` |
| **INTERPRET** | Интерпретация семантики запроса | `INTERPRET query INTO intent` |
| **NAVIGATE** | Навигация по семантическому пространству | `NAVIGATE space TOWARD intent` |
| **LOCATE** | Локализация релевантной семантики | `LOCATE concepts RELEVANT TO intent` |
| **COMPOSE** | Композиция семантически связного ответа | `COMPOSE answer FROM located_semantics` |

## Семантические метрики

Каждая трансформация аннотируется **семантическими метриками**:

```sfl
ABSTRACT semantic_units INTO concepts:
  @semantic_preservation: 0.78    // Сохранение семантики (0-1)
  @information_density: 0.62      // Плотность информации после трансформации
  @abstraction_level: medium      // Уровень абстракции (low/medium/high)
  @reversibility: 0.15            // Возможность обратной трансформации
  @semantic_loss: 0.22            // Потеря семантической информации
  @precision: 0.92                // Точность извлечения концептов
  @recall: 0.85                   // Полнота извлечения концептов
```

## Структура языка

### Semantic Flows

```sfl
SEMANTIC FLOW <name>:
  <intention> <subject> <semantic_action> <object>
    PRESERVING <semantic_metric>=<value>
    @<annotation>: <value>
```

### Semantic Transformations

```sfl
TRANSFORM semantic_units:
  FROM text_representation
  TO concept_representation
  PRESERVING semantic_fidelity=0.78
  LOSING textual_details=0.22
  GAINING abstraction_power=0.65
```

### Semantic Routing

```sfl
ROUTE BY semantic_intent:
  WHEN intent IS overview THEN broad_semantic_space
  WHEN intent IS specific THEN narrow_semantic_space
  WHEN intent IS exploratory THEN connected_semantic_space
```

## Семантические уровни описания

### Level 1: High-Level Semantic Intent

```sfl
SEMANTIC FLOW BuildSemanticIndex:
  DECOMPOSE raw_text INTO semantic_atoms
  ABSTRACT semantic_atoms INTO knowledge_concepts
  ORGANIZE knowledge_concepts INTO semantic_graph
  ENCODE semantic_graph INTO navigable_space
```

### Level 2: Detailed Semantic Transformations

```sfl
SEMANTIC FLOW IndexingSemantics:
  // Семантическое разложение (T1)
  DECOMPOSE documents INTO text_units
    PRESERVING context_continuity=0.95
    USING overlap_strategy
    @loss: boundary_effects=0.05

  // Абстрагирование (T2, T3)
  ABSTRACT text_units INTO entities, relationships
    PRESERVING semantic_fidelity=0.78
    USING llm_abstraction
    @loss: textual_details=0.22
    @gain: conceptual_clarity=0.65

  // Консолидация (T4)
  CONSOLIDATE entities BY semantic_equivalence
    PRESERVING distinctiveness=0.87
    LOSING redundancy=0.13

  // Организация (T5)
  ORGANIZE entities INTO communities
    PRESERVING relationships=0.92
    DISCOVERING emergent_structure

  // Кодирование (T7)
  ENCODE ALL INTO embedding_space
    PRESERVING semantic_similarity=0.91
    ENABLING geometric_reasoning
```

### Level 3: Semantic Preservation Tracking

```sfl
SEMANTIC FLOW WithPreservationTracking:
  START WITH semantic_content=1.0

  DECOMPOSE documents:
    semantic_content *= 0.95  // 5% loss at boundaries

  ABSTRACT entities:
    semantic_content *= 0.78  // 22% loss in abstraction
    conceptual_content = 0.65 // gained abstraction

  CONSOLIDATE:
    semantic_content *= 0.87  // 13% redundancy removed
    distinctiveness += 0.12   // gained clarity

  ORGANIZE:
    structural_semantics = 0.92  // relationships preserved
    emergent_semantics += 0.15    // discovered patterns

  FINAL semantic_preservation = semantic_content
       + conceptual_content
       + emergent_semantics
       ≈ 0.95 (overall semantic value preserved/enhanced)
```

## Пример: Полный семантический поток индексации

```sfl
SEMANTIC FLOW DocumentIndexingSemantics:
  /*
    Полное описание семантических трансформаций
    от сырого текста до индексированного семантического пространства
  */

  // ========== PHASE 1: SEMANTIC DECOMPOSITION ==========
  DECOMPOSE raw_documents
    INTO semantic_units
    STRATEGY: sliding_window WITH overlap
    PRESERVING:
      - context_continuity: 0.95
      - semantic_coherence: 0.97
    LOSING:
      - document_structure: 0.20
      - boundary_context: 0.05

    SEMANTIC_RESULT:
      - 1000 documents → 8500 semantic_units
      - Average coherence: 0.96
      - Overlap preserves 95% of cross-boundary semantics

  // ========== PHASE 2: CONCEPTUAL ABSTRACTION ==========
  ABSTRACT semantic_units
    INTO concepts[entities, relationships]
    STRATEGY: llm_guided_extraction
    PRESERVING:
      - core_concepts: 0.85
      - semantic_relationships: 0.78
      - factual_information: 0.82
    LOSING:
      - linguistic_style: 0.90
      - textual_details: 0.70
      - nuanced_expressions: 0.40
    GAINING:
      - conceptual_clarity: 0.65
      - structural_representation: 0.75
      - queryable_form: 0.80

    SEMANTIC_RESULT:
      - 8500 units → 15000 entities, 25000 relationships
      - Precision: 0.92, Recall: 0.85
      - Semantic fidelity: 0.78 (text→concepts)

  // ========== PHASE 3: SEMANTIC CONSOLIDATION ==========
  CONSOLIDATE concepts
    BY semantic_equivalence
    STRATEGY: name_matching + description_synthesis
    PRESERVING:
      - semantic_distinctiveness: 0.87
      - unique_information: 0.85
      - relationship_validity: 0.90
    LOSING:
      - redundant_descriptions: 0.80
      - duplicate_contexts: 0.75
    GAINING:
      - consolidated_understanding: 0.70
      - coherent_representation: 0.85

    SEMANTIC_RESULT:
      - 15000 entities → 5000 consolidated_entities
      - Redundancy removed: 67%
      - Information density increased: 2.1x
      - Semantic distinctiveness: 0.87

  // ========== PHASE 4: STRUCTURAL ORGANIZATION ==========
  ORGANIZE concepts
    INTO semantic_hierarchy[communities]
    STRATEGY: graph_clustering
    PRESERVING:
      - relationship_structure: 0.92
      - semantic_proximity: 0.88
      - community_coherence: 0.85
    DISCOVERING:
      - emergent_topics: 150 communities
      - hierarchical_structure: 3 levels
      - semantic_clusters: high modularity

    SEMANTIC_RESULT:
      - 5000 entities → 150 communities
      - Modularity: 0.82
      - Community coherence: 0.85
      - Cross-community connections preserved

  // ========== PHASE 5: SEMANTIC SUMMARIZATION ==========
  ABSTRACT communities
    INTO community_summaries
    STRATEGY: llm_guided_summarization
    PRESERVING:
      - key_entities: 0.90
      - main_relationships: 0.75
      - community_theme: 0.82
    LOSING:
      - entity_details: 0.60
      - fine_relationships: 0.50
    GAINING:
      - holistic_understanding: 0.75
      - navigable_overview: 0.85

    SEMANTIC_RESULT:
      - 150 communities → 150 summaries
      - Coverage of key entities: 90%
      - Thematic accuracy: 0.82
      - Comprehensiveness: 0.78

  // ========== PHASE 6: GEOMETRIC ENCODING ==========
  ENCODE concepts, summaries
    INTO embedding_space
    STRATEGY: neural_embedding
    PRESERVING:
      - semantic_similarity: 0.91
      - conceptual_relationships: 0.88
      - topic_clustering: 0.85
    TRANSFORMING:
      - symbolic → geometric representation
      - discrete → continuous space
      - exact → approximate similarity
    ENABLING:
      - fast_similarity_search: O(log n)
      - geometric_reasoning: vector_operations
      - multi_modal_fusion: unified_space

    SEMANTIC_RESULT:
      - 5000 entities + 8500 text_units + 150 communities
      - Embedding dimension: 1536
      - Semantic similarity preservation: 0.91
      - Nearest neighbor accuracy: 0.88

  // ========== OVERALL SEMANTIC BUDGET ==========
  SEMANTIC_BUDGET:
    Initial semantic information: 1.00

    After decomposition:  0.95  (5% boundary loss)
    After abstraction:    0.74  (0.95 * 0.78)
    After consolidation:  0.64  (0.74 * 0.87)
    After organization:   0.59  (0.64 * 0.92)
    After summarization:  0.48  (0.59 * 0.82)
    After encoding:       0.44  (0.48 * 0.91)

    BUT GAINED:
    + Conceptual clarity:      0.65
    + Structural understanding: 0.85
    + Queryability:            0.80
    + Geometric reasoning:     0.70

    Effective semantic value: 0.44 + 0.65 + 0.85 + 0.80 + 0.70 = 3.44

    → System transforms 1 unit of raw text into 3.44 units of
      queryable, structured, navigable semantic knowledge
```

## Пример: Семантический поток запросов

```sfl
SEMANTIC FLOW QuerySemantics:
  /*
    Семантические трансформации при выполнении запроса
  */

  // ========== PHASE 1: INTENT INTERPRETATION ==========
  INTERPRET query_text
    INTO semantic_intent[type, concepts, scope]
    PRESERVING:
      - core_information_need: 0.92
      - key_concepts: 0.90
    EXTRACTING:
      - query_type: {global, local, drift}
      - semantic_scope: {broad, narrow, exploratory}
      - key_concepts: entities + relationships

    SEMANTIC_RESULT:
      - "What are main AI trends?" →
        type: global
        scope: broad
        concepts: [AI, trends, overview]

  // ========== PHASE 2: SEMANTIC ENCODING ==========
  ENCODE semantic_intent
    INTO query_vector
    PRESERVING:
      - semantic_meaning: 0.92
      - conceptual_relationships: 0.88
    ENABLING:
      - geometric_similarity: cosine_distance
      - fast_retrieval: O(log n)

  // ========== PHASE 3: SEMANTIC NAVIGATION ==========
  NAVIGATE embedding_space
    TOWARD query_vector
    STRATEGY: determined_by_intent_type
    LOCATING:
      - semantically_similar: top_k candidates
      - structurally_relevant: connected entities
      - contextually_appropriate: scope matching

    FOR global_intent:
      NAVIGATE TO community_summaries
      LOCATE broad_semantic_coverage

    FOR local_intent:
      NAVIGATE TO specific_entities
      LOCATE detailed_semantic_context

    FOR drift_intent:
      NAVIGATE THROUGH relationship_paths
      LOCATE connected_semantic_spaces

  // ========== PHASE 4: SEMANTIC LOCALIZATION ==========
  LOCATE relevant_semantics
    IN indexed_space
    PRESERVING:
      - relevance_ranking: 0.88
      - semantic_diversity: 0.82
    RETRIEVING:
      - primary_concepts: entities
      - supporting_context: text_units
      - structural_context: relationships
      - holistic_context: community_summaries

    SEMANTIC_RESULT:
      - Retrieved: 30 entities, 20 text_units, 10 summaries
      - Relevance precision: 0.85
      - Coverage recall: 0.80

  // ========== PHASE 5: SEMANTIC COMPOSITION ==========
  COMPOSE answer
    FROM located_semantics
    STRATEGY: llm_guided_synthesis
    PRESERVING:
      - factual_accuracy: 0.82
      - semantic_coherence: 0.88
      - contextual_relevance: 0.85
    INTEGRATING:
      - multiple_perspectives: community_summaries
      - specific_details: entities + text_units
      - structural_relationships: graph_context
    GENERATING:
      - coherent_narrative: natural_language
      - grounded_statements: citations
      - comprehensive_coverage: multi-source

    SEMANTIC_RESULT:
      - Answer comprehensiveness: 0.85
      - Factual grounding: 0.82
      - Coherence: 0.88

  // ========== SEMANTIC ACCURACY TRACKING ==========
  SEMANTIC_ACCURACY:
    Query intent captured:        0.92
    Encoded to vector:           0.92 * 0.92 = 0.85
    Navigated to relevant space: 0.85 * 0.88 = 0.75
    Located semantics:           0.75 * 0.88 = 0.66
    Composed answer:             0.66 * 0.85 = 0.56

    → 56% of original query intent directly preserved

    BUT ENHANCED WITH:
    + Contextual enrichment:  0.40
    + Structural insights:    0.35
    + Multi-source synthesis: 0.30

    Effective semantic answer quality: 0.56 + 0.40 + 0.35 + 0.30 = 1.61

    → System transforms 1 unit of query intent into 1.61 units of
      enriched, contextualized, multi-perspective answer
```

## Семантическая композиция

```sfl
// Композиция семантических трансформаций
COMPOSE SemanticPipeline:
  FROM raw_text
  TO queryable_knowledge

  VIA:
    DECOMPOSE    // 0.95 preservation
    → ABSTRACT   // 0.78 preservation
    → CONSOLIDATE // 0.87 preservation
    → ORGANIZE   // 0.92 preservation
    → ENCODE     // 0.91 preservation

  OVERALL_PRESERVATION:
    0.95 * 0.78 * 0.87 * 0.92 * 0.91 = 0.56

  OVERALL_GAIN:
    + Queryability: 0.80
    + Structure: 0.85
    + Navigability: 0.75
    = 2.40 effective gain

  NET_SEMANTIC_VALUE:
    0.56 (preserved) + 2.40 (gained) = 2.96x original value
```

## Отличия от GFL

| Аспект | GFL | SFL |
|---|---|---|
| **Фокус** | Операции с данными | Семантические трансформации |
| **Интенции** | INGEST, CHUNK, INDEX | DECOMPOSE, ABSTRACT, ORGANIZE |
| **Метрики** | Latency, cost, throughput | Semantic preservation, information density |
| **Описание** | Что делать с данными | Что происходит с семантикой |
| **Уровень** | Технический pipeline | Концептуальные трансформации |
| **Цель** | Описать execution flow | Описать semantic transformations |

## Применение

SFL используется для:

1. **Анализа семантических трансформаций** — понимание потерь/усиления информации
2. **Оптимизации pipeline** — минимизация потерь критической семантики
3. **Документирования** — объяснение семантики трансформаций
4. **Валидации** — проверка сохранения требуемой семантики
5. **Трассировки** — отслеживание семантики от источника до результата
6. **Проектирования** — планирование новых семантических flow

## Документация

1. **[Semantic Intentions](01-semantic-intentions.md)** — базовые семантические интенции
2. **[Indexing Semantics](02-indexing-semantics.md)** — семантика индексации
3. **[Query Semantics](03-query-semantics.md)** — семантика запросов
4. **[Semantic Scenarios](04-semantic-scenarios.md)** — полные сценарии

## Преимущества SFL

✅ **Semantic-First** — фокус на семантике, не на технологии
✅ **Traceable** — явная трассировка семантики
✅ **Quantified** — количественные метрики сохранения семантики
✅ **Loss-Aware** — явное указание потерь информации
✅ **Gain-Aware** — явное указание усиления возможностей
✅ **Composable** — композиция семантических трансформаций
✅ **Verifiable** — возможность проверки семантических свойств

---

**Версия**: SFL 1.0
**Статус**: Specification
**Основано на**: GraphRAG semantic transformation chains (T1-T7, Q1-Q7)
