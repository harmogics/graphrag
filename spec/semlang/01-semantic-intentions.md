# Semantic Intentions

## Обзор

Этот документ описывает **10 базовых семантических интенций** языка SFL. Каждая интенция представляет собой фундаментальное семантическое действие в процессе трансформации информации.

## Классификация интенций

### Indexing Intentions (Семантика индексации)

1. **DECOMPOSE** — разложение на семантические единицы
2. **ABSTRACT** — абстрагирование концептов
3. **CONNECT** — установление семантических связей
4. **CONSOLIDATE** — консолидация эквивалентной семантики
5. **ORGANIZE** — организация в структуры
6. **ENCODE** — кодирование в векторное пространство

### Query Intentions (Семантика запросов)

7. **INTERPRET** — интерпретация семантики запроса
8. **NAVIGATE** — навигация по семантическому пространству
9. **LOCATE** — локализация релевантной семантики
10. **COMPOSE** — композиция семантически связного ответа

---

## 1. DECOMPOSE — Семантическое разложение

### Определение

Разложение непрерывного текста на **семантически когерентные единицы** с сохранением контекстной целостности.

### Синтаксис

```sfl
DECOMPOSE <source>
  INTO <semantic_units>
  STRATEGY: <decomposition_strategy>
  PRESERVING:
    - context_continuity: <value>
    - semantic_coherence: <value>
  LOSING:
    - document_structure: <value>
    - boundary_context: <value>
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| Семантическое действие | Разделение без изменения содержания |
| Уровень абстракции | None (сохраняет текст as-is) |
| Semantic preservation | 0.95-0.98 |
| Information density | ~1.0 (без сжатия) |
| Reversibility | 0.98 (почти полная) |
| Primary loss | Boundary context, document structure |
| Primary gain | Manageable units, parallelizability |

### Примеры

#### Базовое разложение

```sfl
DECOMPOSE documents
  INTO semantic_units
  STRATEGY: sliding_window
    WITH size=1200_tokens, overlap=100_tokens
  PRESERVING:
    - context_continuity: 0.95  // Overlap сохраняет контекст
    - semantic_coherence: 0.97  // Единицы семантически целостны
  LOSING:
    - document_structure: 0.20  // Потеря заголовков, разделов
    - boundary_context: 0.05    // 5% контекста на границах

SEMANTIC_RESULT:
  - 1000 documents → 8500 semantic_units
  - Average unit coherence: 0.96
  - Cross-boundary semantic preservation: 0.95
```

#### Sentence-based разложение

```sfl
DECOMPOSE documents
  INTO semantic_units
  STRATEGY: sentence_boundaries
    WITH min_sentences=5, max_sentences=15
  PRESERVING:
    - sentence_integrity: 1.0   // Предложения целые
    - context_continuity: 0.88  // Меньше overlap
  LOSING:
    - paragraph_boundaries: 0.30
    - cross_sentence_context: 0.12

SEMANTIC_RESULT:
  - Variable size units (200-2000 tokens)
  - Natural semantic boundaries
  - Lower context preservation than overlap
```

### Связь с трансформационными цепочками

- **Соответствует**: T1 (Text Chunking)
- **Входные данные**: Raw documents (text)
- **Выходные данные**: Text units (semantic_units)
- **Следующая трансформация**: ABSTRACT (T2, T3)

### Семантический эффект

```
Input:  Continuous text (high structural context, low manageability)
        ↓ DECOMPOSE
Output: Semantic units (low structural context, high manageability)

Semantic trade-off:
  - Lost: Document structure, some boundary context
  - Gained: Parallelizability, focused processing units
  - Preserved: 95-98% of textual semantics
```

---

## 2. ABSTRACT — Концептуальное абстрагирование

### Определение

Извлечение **концептуальных представлений** (entities, relationships) из текстовых описаний, переход от linguistic representation к conceptual representation.

### Синтаксис

```sfl
ABSTRACT <semantic_units>
  INTO <concepts>[entities, relationships]
  STRATEGY: <abstraction_strategy>
  PRESERVING:
    - core_concepts: <value>
    - semantic_relationships: <value>
    - factual_information: <value>
  LOSING:
    - linguistic_style: <value>
    - textual_details: <value>
    - nuanced_expressions: <value>
  GAINING:
    - conceptual_clarity: <value>
    - structural_representation: <value>
    - queryable_form: <value>
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| Семантическое действие | Linguistic → Conceptual transformation |
| Уровень абстракции | Medium-High |
| Semantic preservation | 0.70-0.85 (concept-level) |
| Information density | 0.60 (концепты плотнее текста) |
| Reversibility | 0.15 (низкая, нельзя восстановить текст) |
| Primary loss | Style, nuances, textual details |
| Primary gain | Structure, queryability, clarity |

### Примеры

#### Entity extraction

```sfl
ABSTRACT text_units
  INTO entities
  STRATEGY: llm_guided_extraction
    WITH entity_types=[organization, person, geo, event]
    WITH max_gleanings=1  // Iterative refinement
  PRESERVING:
    - core_entities: 0.85       // Main entities extracted
    - entity_attributes: 0.78   // Descriptions captured
    - factual_accuracy: 0.92    // Facts preserved
  LOSING:
    - linguistic_style: 0.90    // Writing style lost
    - textual_details: 0.70     // Descriptive details lost
    - contextual_nuances: 0.40  // Subtle meanings lost
  GAINING:
    - conceptual_clarity: 0.65  // Clear entity definitions
    - queryable_form: 0.80      // Can query by entity
    - structured_data: 0.75     // Structured representation

SEMANTIC_RESULT:
  - 8500 text_units → 15000 raw_entities
  - Precision: 0.92 (entities are accurate)
  - Recall: 0.85 (most entities found)
  - F1 score: 0.83
```

#### Relationship extraction

```sfl
ABSTRACT text_units
  INTO relationships
  STRATEGY: llm_guided_extraction
    WITH relationship_types=[related_to, part_of, leads, works_for]
  PRESERVING:
    - primary_relationships: 0.78  // Main connections
    - relationship_semantics: 0.75 // Relationship meaning
  LOSING:
    - relationship_nuances: 0.60   // Subtle connections
    - temporal_context: 0.50       // Time information
  GAINING:
    - graph_structure: 0.85        // Explicit graph
    - traversable_links: 0.90      // Can navigate

SEMANTIC_RESULT:
  - 8500 text_units → 25000 relationships
  - Coverage: 78% of significant relationships
  - Strength scoring: 1-10 scale
```

#### Combined entity+relationship extraction

```sfl
ABSTRACT text_units
  INTO [entities, relationships]
  STRATEGY: joint_extraction
    USING llm=gpt-4
    WITH temperature=0.0  // Deterministic
  PRESERVING:
    - semantic_graph: 0.78        // Overall graph semantics
    - factual_information: 0.82   // Facts preserved
  LOSING:
    - text_representation: 0.85   // Text lost
    - linguistic_features: 0.90   // Grammar, style lost
  GAINING:
    - graph_representation: 0.80  // Structured graph
    - multi_perspective: 0.70     // Multiple text units → single graph

SEMANTIC_RESULT:
  - Text form → Graph form
  - 15000 entities + 25000 relationships
  - Semantic fidelity: 0.78 (text meaning → graph meaning)
```

### Связь с трансформационными цепочками

- **Соответствует**: T2 (Entity Extraction), T3 (Relationship Extraction)
- **Входные данные**: Text units (semantic_units)
- **Выходные данные**: Entities, Relationships (concepts)
- **Следующая трансформация**: CONNECT, CONSOLIDATE

### Семантический эффект

```
Input:  Text units (linguistic representation, high verbosity)
        ↓ ABSTRACT
Output: Concepts (conceptual representation, high density)

Semantic transformation:
  "Microsoft Research developed GraphRAG in 2023..."
    ↓
  Entity(Microsoft Research, ORGANIZATION, "Research division of Microsoft")
  Entity(GraphRAG, TECHNOLOGY, "Knowledge graph RAG system")
  Entity(2023, DATE, "Year of development")
  Relationship(GraphRAG, Microsoft Research, "developed_by", strength=9)

Lost: Writing style, sentence structure, contextual prose
Gained: Queryable entities, traversable graph, structured data
Preserved: Core semantic facts (who, what, when, relationships)
```

---

## 3. CONNECT — Установление семантических связей

### Определение

Установление **явных семантических связей** между концептами, обогащение графа отношениями.

### Синтаксис

```sfl
CONNECT <concepts>
  BY <relationship_criteria>
  DISCOVERING:
    - <relationship_type>: <value>
  PRESERVING:
    - <semantic_property>: <value>
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| Семантическое действие | Relationship discovery & linking |
| Уровень абстракции | Structural |
| Semantic preservation | 0.90+ (relationships) |
| Information density | Increases (adds links) |
| Reversibility | N/A (creates new info) |
| Primary loss | Minimal |
| Primary gain | Graph connectivity, navigability |

### Примеры

```sfl
CONNECT entities
  BY extracted_relationships
  DISCOVERING:
    - direct_relationships: 25000 edges
    - relationship_types: 15 types
    - weighted_connections: strength 1-10
  PRESERVING:
    - relationship_semantics: 0.92
    - entity_integrity: 1.0

SEMANTIC_RESULT:
  - 15000 entities + 25000 relationships
  - Graph density: 0.11 (11% of possible edges)
  - Average node degree: 3.3
  - Connected components: 1 (fully connected)
```

### Связь с трансформационными цепочками

- **Соответствует**: T3 (part of relationship extraction)
- **Входные данные**: Entities, Relationships
- **Выходные данные**: Connected graph
- **Следующая трансформация**: ORGANIZE

---

## 4. CONSOLIDATE — Семантическая консолидация

### Определение

Консолидация **семантически эквивалентных** элементов, удаление избыточности при сохранении уникальной информации.

### Синтаксис

```sfl
CONSOLIDATE <concepts>
  BY <equivalence_criteria>
  STRATEGY: <consolidation_strategy>
  PRESERVING:
    - semantic_distinctiveness: <value>
    - unique_information: <value>
  LOSING:
    - redundancy: <value>
  GAINING:
    - consolidated_understanding: <value>
    - coherence: <value>
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| Семантическое действие | Merge equivalent semantics |
| Уровень абстракции | Same as input |
| Semantic preservation | 0.85-0.90 (distinctiveness) |
| Information density | Increases (removes redundancy) |
| Reversibility | 0.60 (can't recover all descriptions) |
| Primary loss | Redundant descriptions, duplicate contexts |
| Primary gain | Coherence, clarity, reduced noise |

### Примеры

#### Entity consolidation

```sfl
CONSOLIDATE entities
  BY semantic_equivalence
    CRITERIA: name_similarity + description_overlap
  STRATEGY: llm_guided_summarization
  PRESERVING:
    - semantic_distinctiveness: 0.87  // Unique entities remain
    - unique_information: 0.85        // All unique facts kept
    - relationship_validity: 0.90     // Relationships preserved
  LOSING:
    - redundant_descriptions: 0.80    // Duplicate descriptions merged
    - duplicate_contexts: 0.75        // Repeated contexts removed
  GAINING:
    - consolidated_understanding: 0.70  // Unified view
    - coherent_representation: 0.85     // Consistent representation
    - information_density: 2.1x         // Denser information

SEMANTIC_RESULT:
  - 15000 raw_entities → 5000 consolidated_entities
  - Redundancy removed: 67%
  - Information density: 2.1x higher
  - Semantic distinctiveness: 0.87

EXAMPLE:
  Before:
    Entity("Microsoft", "ORGANIZATION", "Tech company") [from text_unit_1]
    Entity("Microsoft Corp", "ORGANIZATION", "Software company") [from text_unit_2]
    Entity("Microsoft", "ORGANIZATION", "Cloud services provider") [from text_unit_3]

  After:
    Entity("Microsoft", "ORGANIZATION",
           "Technology company specializing in software and cloud services")
    [consolidated from 3 descriptions]
```

#### Relationship consolidation

```sfl
CONSOLIDATE relationships
  BY [source, target, type]
  STRATEGY: strength_aggregation
  PRESERVING:
    - unique_relationships: 1.0       // All unique edges kept
    - aggregate_strength: 0.95        // Strengths combined
  LOSING:
    - duplicate_edges: 0.90           // Removed
  GAINING:
    - relationship_confidence: 0.80   // More confident from multiple sources

SEMANTIC_RESULT:
  - 25000 raw_relationships → 18000 consolidated_relationships
  - Average strength increased (multiple sources)
  - Duplicate removal: 28%
```

### Связь с трансформационными цепочками

- **Соответствует**: T4 (Description Summarization)
- **Входные данные**: Raw entities (multiple descriptions per entity)
- **Выходные данные**: Consolidated entities (single coherent description)
- **Следующая трансформация**: ORGANIZE

### Семантический эффект

```
Input:  15000 raw entities (many duplicates, fragmented descriptions)
        ↓ CONSOLIDATE
Output: 5000 consolidated entities (unique, coherent descriptions)

Semantic transformation:
  Multiple fragmented views → Single consolidated view
  Redundant information → Unique information
  Scattered semantics → Coherent semantics

Example:
  "Microsoft" (from 12 different text units, 12 descriptions)
    ↓ CONSOLIDATE
  "Microsoft" (1 entity, synthesized description from all 12)
```

---

## 5. ORGANIZE — Семантическая организация

### Определение

Организация концептов в **семантические структуры** (иерархии, кластеры, communities), обнаружение паттернов и emergent topics.

### Синтаксис

```sfl
ORGANIZE <concepts>
  INTO <semantic_structures>
  STRATEGY: <organization_strategy>
  PRESERVING:
    - <structural_property>: <value>
  DISCOVERING:
    - <emergent_pattern>: <description>
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| Семантическое действие | Structural organization |
| Уровень абстракции | Structural meta-level |
| Semantic preservation | 0.90+ (relationships) |
| Information density | Same |
| Reversibility | 1.0 (non-destructive) |
| Primary loss | None (adds structure) |
| Primary gain | Emergent topics, hierarchies, navigability |

### Примеры

#### Community detection

```sfl
ORGANIZE entities
  INTO communities
  STRATEGY: leiden_clustering
    WITH resolution=1.0, max_cluster_size=10
  PRESERVING:
    - relationship_structure: 0.92    // Graph structure intact
    - semantic_proximity: 0.88        // Close entities clustered
    - community_coherence: 0.85       // Coherent topics
  DISCOVERING:
    - emergent_topics: 150 communities  // Found topics
    - hierarchical_levels: 3 levels     // Hierarchy discovered
    - semantic_clusters: modularity=0.82 // High modularity

SEMANTIC_RESULT:
  - 5000 entities → 150 communities
  - Average community size: 33 entities
  - Modularity score: 0.82 (high quality clustering)
  - Topics discovered: AI, Healthcare, Finance, etc.

EXAMPLE:
  Community #42: "Machine Learning Research"
    - Entities: BERT, GPT, Transformer, Attention Mechanism, ...
    - Relationships: 450 internal edges
    - Theme: Neural language models
    - Coherence: 0.87
```

#### Hierarchical organization

```sfl
ORGANIZE communities
  INTO hierarchy
  STRATEGY: hierarchical_leiden
    WITH levels=3
  PRESERVING:
    - parent_child_semantics: 0.90
    - cross_level_relationships: 0.85
  DISCOVERING:
    - top_level_themes: 15 macro-topics
    - mid_level_topics: 60 topics
    - detailed_communities: 150 communities

SEMANTIC_RESULT:
  Level 0 (Macro): "Technology" (2000 entities)
    Level 1: "AI & ML" (800 entities)
      Level 2: "Language Models" (150 entities)
      Level 2: "Computer Vision" (180 entities)
    Level 1: "Cloud Computing" (600 entities)
```

### Связь с трансформационными цепочками

- **Соответствует**: T5 (Community Detection)
- **Входные данные**: Entity graph (consolidated_entities + relationships)
- **Выходные данные**: Communities (semantic clusters)
- **Следующая трансформация**: ABSTRACT (community summaries), ENCODE

### Семантический эффект

```
Input:  Flat entity graph (5000 entities, no organization)
        ↓ ORGANIZE
Output: Hierarchical communities (150 communities, 3 levels)

Semantic transformation:
  Unstructured → Structured
  Implicit topics → Explicit communities
  Flat → Hierarchical

Gained:
  - Navigability: Can browse by topic
  - Discoverability: Topics explicitly identified
  - Scalability: Can work at different granularities
```

---

## 6. ENCODE — Семантическое кодирование

### Определение

Кодирование семантики в **геометрическое векторное пространство**, где семантическое сходство ≈ геометрическая близость.

### Синтаксис

```sfl
ENCODE <semantic_objects>
  INTO <embedding_space>
  STRATEGY: <encoding_strategy>
  PRESERVING:
    - semantic_similarity: <value>
    - conceptual_relationships: <value>
  TRANSFORMING:
    - symbolic → geometric
    - discrete → continuous
    - exact → approximate
  ENABLING:
    - <capability>: <description>
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| Семантическое действие | Symbolic → Geometric transformation |
| Уровень абстракции | Representation change |
| Semantic preservation | 0.88-0.93 (similarity) |
| Information density | Compressed to fixed dimension |
| Reversibility | 0.05 (very low, lossy encoding) |
| Primary loss | Exact symbolic information |
| Primary gain | Geometric reasoning, fast search |

### Примеры

#### Entity encoding

```sfl
ENCODE entities
  INTO embedding_space
  STRATEGY: neural_embedding
    USING model=text-embedding-3-small
    WITH dimensions=1536
  PRESERVING:
    - semantic_similarity: 0.91       // Similar entities → close vectors
    - conceptual_relationships: 0.88  // Relationships preserved
    - topic_clustering: 0.85          // Topics cluster
  TRANSFORMING:
    - symbolic_names → dense_vectors
    - discrete_space → continuous_space
    - exact_match → similarity_search
  ENABLING:
    - fast_similarity_search: O(log n)  // Sub-linear search
    - geometric_reasoning: vector_math   // Can do vector operations
    - multi_modal_fusion: shared_space   // Can mix text, images, etc.

SEMANTIC_RESULT:
  - 5000 entities → 5000 vectors (1536-dim)
  - Nearest neighbor accuracy: 0.88
  - Semantic similarity preservation: 0.91
  - Search latency: <10ms for 5000 entities
```

#### Text unit encoding

```sfl
ENCODE text_units
  INTO embedding_space
  STRATEGY: neural_embedding
    USING model=text-embedding-3-small
  PRESERVING:
    - semantic_content: 0.92          // Meaning preserved
    - contextual_relationships: 0.87  // Context preserved
  LOSING:
    - exact_wording: 0.95             // Exact text lost
    - linguistic_features: 0.90       // Grammar lost
  ENABLING:
    - semantic_search: similarity_based

SEMANTIC_RESULT:
  - 8500 text_units → 8500 vectors
  - Paraphrase detection: 0.89
  - Semantic search precision@10: 0.85
```

#### Community encoding

```sfl
ENCODE community_summaries
  INTO embedding_space
  STRATEGY: neural_embedding
  PRESERVING:
    - topic_semantics: 0.89           // Topic meaning preserved
    - community_distinctiveness: 0.86 // Communities separated

SEMANTIC_RESULT:
  - 150 communities → 150 vectors
  - Community separation: 0.86
  - Topic retrieval accuracy: 0.88
```

### Связь с трансформационными цепочками

- **Соответствует**: T7 (Text Embedding)
- **Входные данные**: Text units, Entities, Communities (symbolic)
- **Выходные данные**: Embeddings (geometric)
- **Следующая трансформация**: NAVIGATE, LOCATE (during queries)

### Семантический эффект

```
Input:  Symbolic representation (text, entities)
        ↓ ENCODE
Output: Geometric representation (vectors in R^1536)

Semantic transformation:
  "Microsoft develops AI technologies"
    ↓
  [0.023, -0.145, 0.891, ..., 0.034] (1536 dimensions)

Key property: Semantic similarity → Geometric proximity
  similar("Microsoft", "Google") = high
    → distance(vec(Microsoft), vec(Google)) = small

Enables:
  - Fast similarity search
  - Geometric operations (analogy: king - man + woman ≈ queen)
  - Multi-modal fusion (text + images in same space)
```

---

## 7. INTERPRET — Семантическая интерпретация запроса

### Определение

Интерпретация пользовательского запроса для извлечения **семантического намерения** (intent, scope, key concepts).

### Синтаксис

```sfl
INTERPRET <query_text>
  INTO <semantic_intent>[type, concepts, scope]
  EXTRACTING:
    - query_type: <type>
    - semantic_scope: <scope>
    - key_concepts: <concepts>
  PRESERVING:
    - <semantic_property>: <value>
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| Семантическое действие | Text → Structured intent |
| Уровень абстракции | Intent extraction |
| Semantic preservation | 0.90-0.95 (intent) |
| Information density | Increases (structure added) |
| Reversibility | Low (one-way interpretation) |
| Primary loss | Exact wording |
| Primary gain | Structured queryable intent |

### Примеры

#### Global query interpretation

```sfl
INTERPRET "What are the main trends in AI research?"
  INTO semantic_intent
  EXTRACTING:
    - query_type: GLOBAL              // Overview question
    - semantic_scope: BROAD           // Wide coverage
    - key_concepts: [AI, trends, research]
    - temporal_aspect: RECENT         // Current trends
    - expected_answer: SUMMARY        // Wants overview
  PRESERVING:
    - core_information_need: 0.92     // User's need captured
    - key_concepts: 0.90              // Concepts identified

SEMANTIC_RESULT:
  - Query type → Global (route to community summaries)
  - Scope → Broad (retrieve many communities)
  - Concepts → AI, trends, research
```

#### Local query interpretation

```sfl
INTERPRET "How does BERT work?"
  INTO semantic_intent
  EXTRACTING:
    - query_type: LOCAL               // Specific question
    - semantic_scope: NARROW          // Focused on one entity
    - key_concepts: [BERT, mechanism]
    - expected_answer: EXPLANATION    // Wants details
  PRESERVING:
    - specificity: 0.95               // Specific entity identified
    - information_need: 0.92

SEMANTIC_RESULT:
  - Query type → Local (route to specific entities)
  - Scope → Narrow (retrieve BERT + related)
  - Concepts → BERT, mechanism
```

#### Drift query interpretation

```sfl
INTERPRET "How are transformers related to language models?"
  INTO semantic_intent
  EXTRACTING:
    - query_type: DRIFT               // Connection question
    - semantic_scope: EXPLORATORY     // Multi-hop exploration
    - key_concepts: [transformers, language_models]
    - relationship_focus: CONNECTION  // Wants relationship
  PRESERVING:
    - relational_intent: 0.93         // Connection intent clear

SEMANTIC_RESULT:
  - Query type → Drift (route to graph traversal)
  - Scope → Exploratory (multi-hop search)
  - Concepts → transformers, language_models
```

### Связь с трансформационными цепочками

- **Соответствует**: Q1 (Query Analysis)
- **Входные данные**: Query text (string)
- **Выходные данные**: Semantic intent (structured)
- **Следующая трансформация**: ENCODE (query embedding), NAVIGATE

---

## 8. NAVIGATE — Семантическая навигация

### Определение

Навигация по **семантическому пространству** к релевантным областям на основе query intent.

### Синтаксис

```sfl
NAVIGATE <semantic_space>
  TOWARD <query_vector>
  STRATEGY: <determined_by_intent_type>
  LOCATING:
    - <target_type>: <description>
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| Семантическое действие | Space traversal |
| Уровень абстракции | Navigation |
| Semantic preservation | 0.85-0.92 (relevance) |
| Information density | N/A (selection operation) |
| Reversibility | N/A (deterministic search) |
| Primary loss | Non-relevant areas |
| Primary gain | Focused relevant space |

### Примеры

#### Navigate to communities (Global)

```sfl
NAVIGATE embedding_space
  TOWARD query_vector
  STRATEGY: community_summary_search
    FOR query_type=GLOBAL
  LOCATING:
    - community_summaries: top_k=10
    - broad_coverage: diverse topics
    - semantic_relevance: >0.7

SEMANTIC_RESULT:
  - Navigated to: 10 community summaries
  - Coverage: 1200 entities (indirect)
  - Relevance: 0.82 average
```

#### Navigate to entities (Local)

```sfl
NAVIGATE embedding_space
  TOWARD query_vector
  STRATEGY: entity_search
    FOR query_type=LOCAL
  LOCATING:
    - specific_entities: top_k=30
    - related_text_units: top_k=20
    - focused_context: narrow scope

SEMANTIC_RESULT:
  - Navigated to: 30 entities + 20 text_units
  - Precision: 0.88
  - Specific focus on query entities
```

#### Navigate through graph (Drift)

```sfl
NAVIGATE entity_graph
  THROUGH relationships
  STRATEGY: graph_traversal
    FOR query_type=DRIFT
    WITH max_hops=3
  LOCATING:
    - connected_entities: multi-hop paths
    - relationship_paths: semantic connections
    - exploratory_space: expanding context

SEMANTIC_RESULT:
  - Traversed: 3 hops
  - Discovered: 150 connected entities
  - Paths found: 25 semantic paths
```

### Связь с трансформационными цепочками

- **Соответствует**: Q3 (Semantic Retrieval) - начало
- **Входные данные**: Query vector, Intent type
- **Выходные данные**: Located space (candidates)
- **Следующая трансформация**: LOCATE (refine)

---

## 9. LOCATE — Семантическая локализация

### Определение

Точная **локализация релевантной семантики** из navigated space с ранжированием по релевантности.

### Синтаксис

```sfl
LOCATE <target_semantics>
  IN <navigated_space>
  RANKED BY <relevance_criteria>
  PRESERVING:
    - relevance_precision: <value>
    - semantic_diversity: <value>
  RETRIEVING:
    - <semantic_object_type>: <count>
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| Семантическое действие | Precision retrieval |
| Уровень абстракции | Selection |
| Semantic preservation | 0.85-0.90 (relevance) |
| Information density | N/A (filtering) |
| Reversibility | N/A (deterministic) |
| Primary loss | Irrelevant candidates |
| Primary gain | High-precision relevant set |

### Примеры

#### Locate for Global query

```sfl
LOCATE community_summaries
  IN community_space
  RANKED BY [semantic_similarity, coverage]
  PRESERVING:
    - relevance_precision: 0.88       // High precision
    - semantic_diversity: 0.82        // Diverse perspectives
    - coverage: 0.85                  // Broad coverage
  RETRIEVING:
    - community_summaries: 10
    - covering_entities: 1200 (indirect)

SEMANTIC_RESULT:
  - Located: 10 communities
  - Relevance precision: 0.88
  - Coverage: 24% of knowledge graph
  - Diversity: Communities from different topics
```

#### Locate for Local query

```sfl
LOCATE relevant_context
  IN entity_space
  RANKED BY [similarity, specificity]
  PRESERVING:
    - relevance_precision: 0.85
    - context_completeness: 0.80
  RETRIEVING:
    - specific_entities: 30
    - supporting_text_units: 20
    - related_relationships: 50

SEMANTIC_RESULT:
  - Located: 30 entities + 20 text_units
  - Precision@30: 0.85
  - Recall: 0.80 (most relevant found)
  - Context completeness: 0.80
```

### Связь с трансформационными цепочками

- **Соответствует**: Q3 (Semantic Retrieval) - завершение
- **Входные данные**: Navigated space (candidates)
- **Выходные данные**: Located semantics (precise relevant set)
- **Следующая трансформация**: COMPOSE

---

## 10. COMPOSE — Семантическая композиция

### Определение

Композиция **семантически связного ответа** из located semantics с интеграцией multiple perspectives.

### Синтаксис

```sfl
COMPOSE <answer>
  FROM <located_semantics>
  STRATEGY: <composition_strategy>
  INTEGRATING:
    - <source_type>: <description>
  PRESERVING:
    - <quality_metric>: <value>
  GENERATING:
    - <output_type>: <description>
```

### Семантические характеристики

| Характеристика | Значение |
|---|---|
| Семантическое действие | Multi-source synthesis |
| Уровень абстракции | Answer generation |
| Semantic preservation | 0.80-0.88 (coherence) |
| Information density | High (synthesized) |
| Reversibility | Low (synthesis is creative) |
| Primary loss | Source verbatim text |
| Primary gain | Coherent narrative, multi-perspective |

### Примеры

#### Compose Global answer (Map-Reduce)

```sfl
// MAP PHASE
FOR EACH community_summary IN located_summaries:
  COMPOSE intermediate_answer
    FROM community_summary
    STRATEGY: llm_guided
      WITH prompt=GlobalMapPrompt
    PRESERVING:
      - factual_accuracy: 0.85
      - relevance: 0.82
    GENERATING:
      - perspective: community_view
      - claims: grounded_statements

// REDUCE PHASE
COMPOSE final_answer
  FROM intermediate_answers
  STRATEGY: llm_guided_synthesis
    WITH prompt=GlobalReducePrompt
  INTEGRATING:
    - multiple_perspectives: 10 communities
    - diverse_viewpoints: cross-community
  PRESERVING:
    - comprehensiveness: 0.85        // Covers multiple aspects
    - factual_accuracy: 0.82         // Facts correct
    - coherence: 0.88                // Narrative flows
  GENERATING:
    - comprehensive_summary: natural_language
    - grounded_claims: with_citations
    - multi_perspective: integrated_view

SEMANTIC_RESULT:
  - Integrated: 10 community perspectives
  - Comprehensiveness: 0.85
  - Coherence: 0.88
  - Citations: Grounded in source communities
```

#### Compose Local answer

```sfl
COMPOSE answer
  FROM [entities, text_units, relationships]
  STRATEGY: context_based_generation
    WITH prompt=LocalSearchPrompt
  INTEGRATING:
    - specific_entities: detailed_descriptions
    - supporting_text: original_context
    - structural_relationships: graph_connections
  PRESERVING:
    - factual_accuracy: 0.88         // Grounded in sources
    - contextual_relevance: 0.85     // Relevant to query
    - semantic_coherence: 0.90       // Coherent answer
  GENERATING:
    - detailed_explanation: natural_language
    - citations: text_unit_references
    - relationship_context: graph_info

SEMANTIC_RESULT:
  - Synthesized from: 30 entities + 20 text_units
  - Factual grounding: 0.88
  - Coherence: 0.90
  - Citation coverage: 85% of claims cited
```

### Связь с трансформационными цепочками

- **Соответствует**: Q5 (Answer Generation), Q6 (Answer Synthesis)
- **Входные данные**: Located semantics (entities, text_units, summaries)
- **Выходные данные**: Final answer (natural language)
- **Следующая трансформация**: None (final output)

---

## Композиция интенций

### Indexing Flow

```sfl
DECOMPOSE → ABSTRACT → CONSOLIDATE → ORGANIZE → ENCODE

Semantic transformation:
  Raw text (linguistic)
    → Semantic units (manageable)
    → Concepts (structured)
    → Consolidated concepts (coherent)
    → Communities (organized)
    → Embeddings (geometric)

Overall semantic preservation:
  0.95 * 0.78 * 0.87 * 0.92 * 0.91 = 0.56

Net semantic value:
  0.56 (preserved) + 2.40 (gained capabilities) = 2.96x
```

### Query Flow

```sfl
INTERPRET → ENCODE → NAVIGATE → LOCATE → COMPOSE

Semantic transformation:
  Query text (user intent)
    → Semantic intent (structured)
    → Query vector (geometric)
    → Navigated space (relevant area)
    → Located semantics (precise set)
    → Answer (synthesized)

Overall semantic accuracy:
  0.92 * 0.92 * 0.88 * 0.88 * 0.85 = 0.56

Net answer quality:
  0.56 (intent preserved) + 1.05 (enrichment) = 1.61x
```

---

## Семантические метрики

### Preservation Metrics

| Метрика | Описание | Диапазон |
|---|---|---|
| `semantic_preservation` | Сохранение core семантики | 0.0 - 1.0 |
| `information_density` | Плотность информации | 0.0 - ∞ |
| `reversibility` | Возможность обратной трансформации | 0.0 - 1.0 |
| `precision` | Точность извлечения | 0.0 - 1.0 |
| `recall` | Полнота извлечения | 0.0 - 1.0 |
| `coherence` | Семантическая связность | 0.0 - 1.0 |

### Loss/Gain Metrics

| Метрика | Описание |
|---|---|
| `LOSING: <aspect>: <value>` | Что теряется |
| `GAINING: <capability>: <value>` | Что приобретается |
| `PRESERVING: <property>: <value>` | Что сохраняется |

---

**Next**: [Indexing Semantics](02-indexing-semantics.md)
