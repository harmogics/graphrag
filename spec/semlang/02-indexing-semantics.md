# Indexing Semantics

## Обзор

Этот документ описывает **семантику процесса индексации** в GraphRAG, используя язык SFL. Индексация представляет собой серию семантических трансформаций от сырого текста до навигационного семантического пространства.

## Полная семантическая цепочка индексации

```
Raw Documents (text)
  ↓ [DECOMPOSE: 0.95]
Semantic Units (manageable text chunks)
  ↓ [ABSTRACT: 0.78]
Concepts (entities + relationships)
  ↓ [CONSOLIDATE: 0.87]
Consolidated Concepts (unique entities)
  ↓ [ORGANIZE: 0.92]
Semantic Communities (clustered topics)
  ↓ [ABSTRACT: 0.82]
Community Summaries (topic overviews)
  ↓ [ENCODE: 0.91]
Embedding Space (geometric representation)

Overall semantic preservation: 0.95 * 0.78 * 0.87 * 0.92 * 0.82 * 0.91 ≈ 0.44
Gained capabilities: +2.40
Net semantic value: 2.84x original
```

---

## Стандартный семантический поток индексации

```sfl
SEMANTIC FLOW StandardIndexingSemantics:
  /*
    Стандартная семантическая трансформация документов
    в индексированный граф знаний
  */

  // ==================== PHASE 1: SEMANTIC DECOMPOSITION ====================
  DECOMPOSE documents
    FROM raw_text
    INTO semantic_units
    STRATEGY: sliding_window
      WITH size=1200_tokens, overlap=100_tokens
      USING tokenizer=cl100k_base

    PRESERVING:
      context_continuity: 0.95      // Overlap сохраняет контекст
      semantic_coherence: 0.97      // Единицы семантически целостны
      local_context: 0.98           // Локальный контекст сохранен

    LOSING:
      document_structure: 0.20      // Заголовки, разделы
      global_context: 0.15          // Общая структура документа
      boundary_context: 0.05        // Контекст на границах

    METRICS:
      input: 1000 documents (2.5M tokens avg)
      output: 8500 semantic_units (1200 tokens each)
      preservation: 0.95
      latency: <1s
      cost: negligible (algorithmic)

  SEMANTIC_STATE_1:
    representation: Text (linguistic)
    structure: Chunked (manageable units)
    queryability: Low (text search only)
    semantic_value: 0.95

  // ==================== PHASE 2: CONCEPTUAL ABSTRACTION ====================
  ABSTRACT semantic_units
    FROM text_representation
    INTO concept_representation[entities, relationships]
    STRATEGY: llm_guided_extraction
      USING llm=gpt-4, temperature=0.0
      WITH entity_types=[organization, person, geo, event]
      WITH max_gleanings=1  // One refinement iteration

    PRESERVING:
      core_concepts: 0.85           // Main entities identified
      semantic_relationships: 0.78  // Key relationships found
      factual_information: 0.82     // Facts preserved
      concept_precision: 0.92       // Extracted concepts are accurate

    LOSING:
      linguistic_style: 0.90        // Writing style lost
      textual_details: 0.70         // Descriptive details lost
      exact_wording: 0.95           // Exact text lost
      nuanced_expressions: 0.40     // Subtle meanings lost

    GAINING:
      conceptual_clarity: 0.65      // Clear concept definitions
      structural_form: 0.75         // Structured entities/relationships
      queryable_concepts: 0.80      // Can query by entity
      graph_representation: 0.70    // Graph form

    METRICS:
      input: 8500 semantic_units
      output: 15000 entities, 25000 relationships
      precision: 0.92
      recall: 0.85
      f1_score: 0.83
      preservation: 0.78
      latency: 2-5s per unit
      cost: $96 per 1K documents

  SEMANTIC_STATE_2:
    representation: Concepts (structured)
    structure: Graph (entities + edges)
    queryability: Medium (entity queries)
    semantic_value: 0.95 * 0.78 = 0.74

  // ==================== PHASE 3: SEMANTIC CONSOLIDATION ====================
  CONSOLIDATE entities
    BY semantic_equivalence
      CRITERIA: name_similarity AND description_overlap
    STRATEGY: llm_guided_summarization
      USING llm=gpt-4
      WITH max_description_length=500

    PRESERVING:
      semantic_distinctiveness: 0.87  // Unique entities distinct
      unique_information: 0.85        // All unique info kept
      relationship_validity: 0.90     // Relationships preserved
      entity_integrity: 0.88          // Entity meaning intact

    LOSING:
      redundant_descriptions: 0.80    // Duplicates removed
      duplicate_contexts: 0.75        // Repeated contexts merged
      source_diversity: 0.60          // Multiple sources → single view

    GAINING:
      consolidated_understanding: 0.70  // Unified entity view
      coherent_representation: 0.85     // Consistent descriptions
      information_density: 2.1x         // Denser representation
      clarity: 0.75                     // Clearer definitions

    METRICS:
      input: 15000 raw_entities
      output: 5000 consolidated_entities
      consolidation_ratio: 3:1
      preservation: 0.87
      latency: 1-2s per entity group
      cost: $3 per 1K documents

  SEMANTIC_STATE_3:
    representation: Consolidated concepts
    structure: Deduplicated graph
    queryability: High (unique entities)
    semantic_value: 0.74 * 0.87 = 0.64

  // ==================== PHASE 4: STRUCTURAL ORGANIZATION ====================
  ORGANIZE entity_graph
    INTO semantic_communities
    STRATEGY: leiden_clustering
      WITH resolution=1.0
      WITH max_cluster_size=10
      WITH use_lcc=true  // Largest connected component

    PRESERVING:
      relationship_structure: 0.92    // Graph structure intact
      semantic_proximity: 0.88        // Close entities clustered together
      entity_connections: 0.95        // Connections preserved
      community_coherence: 0.85       // Communities are coherent

    DISCOVERING:
      emergent_topics: 150 communities       // Topics emerged from structure
      hierarchical_levels: 3 levels          // Hierarchy discovered
      semantic_modularity: 0.82             // High quality clustering
      cross_community_links: 2500 edges     // Inter-community connections

    GAINING:
      navigability: 0.85              // Can browse by topic
      topic_structure: 0.80           // Topics explicitly identified
      hierarchical_access: 0.75       // Multi-level granularity
      scalability: 0.90               // Can work at different scales

    METRICS:
      input: 5000 entities, 18000 relationships
      output: 150 communities
      avg_community_size: 33 entities
      modularity: 0.82
      preservation: 0.92 (structural)
      latency: <10s
      cost: negligible (algorithmic)

  SEMANTIC_STATE_4:
    representation: Organized concepts
    structure: Hierarchical communities
    queryability: Very high (topic-based)
    semantic_value: 0.64 * 0.92 = 0.59

  // ==================== PHASE 5: HOLISTIC ABSTRACTION ====================
  ABSTRACT communities
    INTO community_summaries
    STRATEGY: llm_guided_summarization
      USING llm=gpt-4
      WITH max_summary_length=2000
      WITH include_key_entities=true

    PRESERVING:
      key_entities: 0.90              // Main entities mentioned
      main_relationships: 0.75        // Important relationships included
      community_theme: 0.82           // Topic theme captured
      factual_accuracy: 0.78          // Facts correct

    LOSING:
      entity_details: 0.60            // Entity details lost
      fine_relationships: 0.50        // Detailed relationships lost
      specific_contexts: 0.65         // Specific contexts generalized

    GAINING:
      holistic_understanding: 0.75    // Big picture view
      navigable_overview: 0.85        // Easy to browse
      topic_summarization: 0.80       // Topic clearly described
      comprehensiveness: 0.78         // Covers community well

    METRICS:
      input: 150 communities (5000 entities, 18000 relationships)
      output: 150 community_summaries (~2000 tokens each)
      preservation: 0.82 (thematic)
      latency: 5-10s per community
      cost: $6 per 1K documents

  SEMANTIC_STATE_5:
    representation: Multi-level (concepts + summaries)
    structure: Hierarchical (entities → communities → summaries)
    queryability: Exceptional (fine + coarse grain)
    semantic_value: 0.59 * 0.82 = 0.48

  // ==================== PHASE 6: GEOMETRIC ENCODING ====================
  ENCODE [semantic_units, entities, community_summaries]
    FROM symbolic_representation
    INTO embedding_space
    STRATEGY: neural_embedding
      USING model=text-embedding-3-small
      WITH dimensions=1536
      WITH batch_size=100

    PRESERVING:
      semantic_similarity: 0.91       // Similar → close in space
      conceptual_relationships: 0.88  // Relationships preserved
      topic_clustering: 0.85          // Topics cluster
      semantic_content: 0.89          // Meaning preserved

    TRANSFORMING:
      symbolic → geometric              // Discrete → continuous
      text → vectors                    // Text → numbers
      exact_match → similarity_search   // Exact → approximate

    LOSING:
      exact_symbolic_form: 0.95       // Exact text/entities lost
      discrete_identity: 0.90         // Discrete form lost

    GAINING:
      geometric_reasoning: 0.85       // Vector operations enabled
      fast_similarity_search: 0.95    // O(log n) search
      continuous_space: 0.90          // Smooth interpolation
      multi_modal_fusion: 0.80        // Can mix modalities

    ENABLING:
      similarity_search: O(log n)     // Sub-linear search
      vector_operations: arithmetic    // Can do math on semantics
      approximate_matching: fuzzy      // Tolerant to variations

    METRICS:
      input: 8500 text_units + 5000 entities + 150 summaries
      output: 13650 embedding_vectors (1536-dim each)
      preservation: 0.91 (similarity)
      nearest_neighbor_accuracy: 0.88
      latency: <1s (batched)
      cost: $0.96 per 1K documents

  SEMANTIC_STATE_6:
    representation: Hybrid (symbolic + geometric)
    structure: Multi-space (graph + vector space)
    queryability: Optimal (all modes)
    semantic_value: 0.48 * 0.91 = 0.44

  // ==================== SEMANTIC BUDGET SUMMARY ====================
  SEMANTIC_BUDGET:
    Initial semantic content: 1.00

    After DECOMPOSE:   0.95  (5% boundary loss)
    After ABSTRACT:    0.74  (26% abstraction loss)
    After CONSOLIDATE: 0.64  (13% redundancy removed)
    After ORGANIZE:    0.59  (8% structural loss)
    After ABSTRACT:    0.48  (18% summarization loss)
    After ENCODE:      0.44  (9% encoding loss)

    TOTAL PRESERVATION: 0.44 (44% direct preservation)

    BUT GAINED CAPABILITIES:
    + Conceptual clarity:      0.65
    + Structural organization: 0.85
    + Queryability:           0.80
    + Geometric reasoning:     0.70
    + Navigability:           0.85
    + Scalability:            0.75

    TOTAL GAINED: 4.60

    NET SEMANTIC VALUE:
      0.44 (preserved) + 4.60 (capabilities) = 5.04

    → System transforms 1 unit of raw text into 5.04 units of
      structured, queryable, navigable semantic knowledge

  // ==================== FINAL INDEXED STATE ====================
  INDEXED_KNOWLEDGE_GRAPH:
    Representations:
      - Textual: 8500 semantic_units (original text preserved)
      - Conceptual: 5000 entities, 18000 relationships (structured)
      - Organizational: 150 communities (hierarchical)
      - Holistic: 150 summaries (overviews)
      - Geometric: 13650 embeddings (searchable)

    Query Capabilities:
      - Text search: Full-text search in semantic_units
      - Entity search: Find by entity name/type
      - Relationship traversal: Graph navigation
      - Topic browsing: Community-based exploration
      - Semantic search: Vector similarity
      - Hybrid queries: Combine multiple modes

    Semantic Properties:
      - Preservation: 0.44 (core semantics)
      - Queryability: 0.95 (exceptional)
      - Navigability: 0.90 (excellent)
      - Scalability: 0.85 (good)
      - Completeness: 0.80 (high coverage)
```

---

## Оптимизированные семантические потоки

### Speed-Optimized Semantics

```sfl
SEMANTIC FLOW FastIndexingSemantics:
  /*
    Оптимизация для скорости с минимальными потерями качества
  */

  DECOMPOSE documents WITH size=1500, overlap=50
    PRESERVING: context_continuity: 0.90  // Ниже из-за меньшего overlap
    REASON: "Fewer chunks = fewer LLM calls"

  ABSTRACT semantic_units
    USING llm=gpt-3.5-turbo  // Faster, cheaper
    WITH max_gleanings=0      // No refinement
    PRESERVING: core_concepts: 0.75  // Lower quality
    REASON: "Trade quality for speed"

  // SKIP CONSOLIDATE PHASE
  // Work with raw entities directly
  REASON: "Save consolidation time"

  ORGANIZE entity_graph
    WITH max_cluster_size=15  // Larger clusters = fewer
    PRESERVING: community_coherence: 0.78  // Lower quality

  // SKIP COMMUNITY SUMMARIES
  // Work with entities directly
  REASON: "Save summarization time"

  ENCODE ALL WITH batch_size=500  // Larger batches
    REASON: "Maximize throughput"

  SEMANTIC_BUDGET:
    Preservation: 0.90 * 0.75 * 0.92 * 0.91 = 0.56  // vs 0.44
    But gained: Speed 4x faster
    Lost: Some quality (F1: 0.70 vs 0.83)
```

### Quality-Optimized Semantics

```sfl
SEMANTIC FLOW HighQualitySemantics:
  /*
    Максимальное сохранение семантики
  */

  DECOMPOSE documents WITH size=800, overlap=150
    PRESERVING: context_continuity: 0.98  // Высокий overlap
    REASON: "Preserve all context"

  ABSTRACT semantic_units
    USING llm=gpt-4
    WITH max_gleanings=3  // Multiple refinement iterations
    PRESERVING: core_concepts: 0.90  // Higher recall
    PRESERVING: factual_accuracy: 0.95
    REASON: "Maximize semantic extraction"

  CONSOLIDATE entities
    STRATEGY: careful_llm_synthesis
    PRESERVING: unique_information: 0.92  // Higher preservation
    REASON: "Preserve all nuances"

  ORGANIZE entity_graph
    WITH max_cluster_size=8  // Smaller, more coherent
    PRESERVING: community_coherence: 0.90

  ABSTRACT communities
    WITH max_summary_length=3000  // Longer, more detailed
    PRESERVING: entity_details: 0.85  // More details
    REASON: "Comprehensive summaries"

  ENCODE ALL WITH dimensions=1536

  SEMANTIC_BUDGET:
    Preservation: 0.98 * 0.90 * 0.92 * 0.90 * 0.85 * 0.91 = 0.58
    Quality: F1: 0.90 vs 0.83 standard
    Cost: +30% cost, +50% time
```

### Cost-Optimized Semantics

```sfl
SEMANTIC FLOW CostOptimizedSemantics:
  /*
    Минимизация стоимости при разумном качестве
  */

  DECOMPOSE documents WITH size=1200, overlap=100
    // Standard chunking

  ABSTRACT semantic_units
    USING llm=gpt-3.5-turbo  // 90% cheaper than GPT-4
    WITH max_gleanings=0
    PRESERVING: core_concepts: 0.75
    REASON: "GPT-3.5 is much cheaper"

  CONSOLIDATE entities
    USING llm=gpt-3.5-turbo  // Cheaper consolidation
    WITH max_description_length=300  // Shorter

  ORGANIZE entity_graph
    // Same (algorithmic, free)

  ABSTRACT communities
    USING llm=gpt-3.5-turbo  // Cheaper
    WITH max_summary_length=1500  // Shorter

  ENCODE ALL
    // Same (cheap already)

  SEMANTIC_BUDGET:
    Preservation: 0.95 * 0.75 * 0.85 * 0.92 * 0.78 * 0.91 = 0.42
    Cost: $21 per 1K docs (vs $105 with GPT-4)
    Savings: 80%
    Quality: F1: 0.75 vs 0.83 standard
```

---

## Доменно-специфичные семантические потоки

### Scientific Papers Semantics

```sfl
SEMANTIC FLOW ScientificPapersSemantics:
  /*
    Семантика для научных статей
    Фокус: методы, метрики, датасеты, findings
  */

  DECOMPOSE papers
    WITH size=800, overlap=150  // Smaller for precision
    PRESERVING:
      technical_context: 0.96          // Preserve formulas, terms
      citation_context: 0.90           // Keep citations
    REASON: "Scientific content requires precision"

  ABSTRACT semantic_units
    WITH entity_types=[
      method,        // Algorithms, techniques
      metric,        // Evaluation metrics
      dataset,       // Data sources
      finding,       // Research findings
      researcher,    // Authors
      organization   // Institutions
    ]
    PRESERVING:
      technical_entities: 0.88         // High precision needed
      methodological_relationships: 0.82
    REASON: "Domain-specific entity types"

  CONSOLIDATE entities
    WITH preserve_technical_details=true
    PRESERVING:
      technical_accuracy: 0.90         // Critical for science

  ORGANIZE entity_graph
    WITH max_cluster_size=8  // Fine-grained topics
    DISCOVERING:
      research_topics: detailed        // Specific research areas
    REASON: "Scientific fields are narrow"

  ABSTRACT communities
    WITH focus=methodology
    WITH include_metrics=true
    PRESERVING:
      methodological_details: 0.85     // Keep method details

  ENCODE ALL

  SEMANTIC_RESULT:
    Specialized for scientific domain
    Preserves technical semantics: 0.85
    Enables method/metric queries
```

### Legal Documents Semantics

```sfl
SEMANTIC FLOW LegalDocumentsSemantics:
  /*
    Семантика для юридических документов
    Фокус: citations, precedents, statutes
  */

  DECOMPOSE documents
    WITH size=1000, overlap=200  // High overlap for citations
    WITH boundary_aware=true
    PRESERVING:
      citation_context: 0.95           // Citations critical
      legal_context: 0.92              // Legal context important
    REASON: "Legal citations span boundaries"

  ABSTRACT semantic_units
    WITH entity_types=[
      case,          // Legal cases
      statute,       // Laws
      precedent,     // Precedents
      party,         // Legal parties
      court,         // Courts
      judge          // Judges
    ]
    PRESERVING:
      legal_entities: 0.90             // High precision
      precedent_relationships: 0.88    // Critical

  EXTRACT citations
    WITH pattern=legal_citation_regex
    PRESERVING:
      citation_accuracy: 0.95          // Must be exact

  CONSOLIDATE entities
    WITH preserve_case_details=true

  ORGANIZE entity_graph
    WITH max_cluster_size=5  // Small, distinct
    WITH resolution=0.5      // Less clustering
    REASON: "Each case should remain distinct"

  ABSTRACT communities
    WITH focus=precedents
    WITH include_citations=true
    WITH format=legal_brief

  ENCODE ALL

  SEMANTIC_RESULT:
    Specialized for legal domain
    Preserves legal semantics: 0.88
    Citation tracking: 0.95
```

---

## Incremental Indexing Semantics

### Update Existing Index

```sfl
SEMANTIC FLOW IncrementalIndexingSemantics:
  /*
    Обновление существующего индекса с минимальным пересчетом
  */

  LOAD existing_semantic_graph
    FROM ./output/knowledge_graph
    PRESERVING: all_existing_semantics: 1.0

  DECOMPOSE new_documents INTO new_semantic_units
    // Process only new docs

  ABSTRACT new_semantic_units
    INTO new_entities, new_relationships

  CONSOLIDATE new_entities
    WITH existing_entities
    STRATEGY: semantic_merge
      CRITERIA: equivalence_with_existing
    PRESERVING:
      existing_semantics: 0.98         // Don't disrupt existing
      new_semantics: 0.87              // Integrate new

  IDENTIFY affected_communities
    WHERE contains_new_entities=true
    RESULT: subset of communities

  REORGANIZE affected_communities
    PRESERVING:
      unaffected_structure: 1.0        // Don't touch unaffected
      affected_coherence: 0.90         // Maintain quality

  ABSTRACT affected_communities
    INTO updated_summaries

  ENCODE new_content INTO embedding_space
    MERGE WITH existing_embeddings

  SEMANTIC_RESULT:
    Incremental update preserves: 0.95 of existing semantics
    New content integrated: 0.87 semantic quality
    Computation: Only affected parts recomputed (~10-20%)
```

---

## Semantic Preservation Tracking

### Detailed Tracking Example

```sfl
SEMANTIC FLOW WithDetailedTracking:
  /*
    Отслеживание семантики через весь pipeline
  */

  START WITH:
    semantic_content = {
      textual: 1.00,
      conceptual: 0.00,
      structural: 0.00,
      navigational: 0.00
    }

  DECOMPOSE documents:
    semantic_content.textual *= 0.95    // 5% boundary loss
    TRACK: "Textual semantics: 0.95"

  ABSTRACT entities:
    semantic_content.textual *= 0.78    // Abstraction loss
    semantic_content.conceptual = 0.65  // Gained
    TRACK: "Textual: 0.74, Conceptual: 0.65"

  CONSOLIDATE:
    semantic_content.conceptual *= 0.87
    TRACK: "Conceptual: 0.57"

  ORGANIZE:
    semantic_content.structural = 0.85  // Discovered
    TRACK: "Structural: 0.85"

  ABSTRACT communities:
    semantic_content.conceptual *= 0.82

  ENCODE:
    semantic_content.navigational = 0.91  // Enabled
    TRACK: "Navigational: 0.91"

  FINAL semantic_content = {
    textual: 0.74,        // Preserved in text_units
    conceptual: 0.47,     // Entities and summaries
    structural: 0.85,     // Communities
    navigational: 0.91    // Embeddings
  }

  TOTAL_SEMANTIC_VALUE:
    Σ = 0.74 + 0.47 + 0.85 + 0.91 = 2.97x original
```

---

## Семантические метрики по фазам

| Phase | Preservation | Loss | Gain | Net Value |
|---|---|---|---|---|
| DECOMPOSE | 0.95 | 0.05 (boundaries) | 0.20 (manageability) | 1.10 |
| ABSTRACT | 0.78 | 0.22 (details) | 0.65 (structure) | 1.21 |
| CONSOLIDATE | 0.87 | 0.13 (redundancy) | 0.15 (clarity) | 0.89 |
| ORGANIZE | 0.92 | 0.08 (flatness) | 0.85 (hierarchy) | 1.69 |
| ABSTRACT | 0.82 | 0.18 (details) | 0.75 (overview) | 1.39 |
| ENCODE | 0.91 | 0.09 (symbolic) | 0.90 (geometric) | 1.72 |
| **OVERALL** | **0.44** | **0.56** | **3.50** | **3.94** |

---

## Связь с трансформационными цепочками

| SFL Intention | GraphRAG Transform | Semantic Preservation |
|---|---|---|
| DECOMPOSE | T1 (Text Chunking) | 0.95-0.98 |
| ABSTRACT | T2 (Entity Extraction) | 0.70-0.85 |
| ABSTRACT | T3 (Relationship Extraction) | 0.75-0.85 |
| CONSOLIDATE | T4 (Description Summarization) | 0.85-0.90 |
| ORGANIZE | T5 (Community Detection) | 0.90+ |
| ABSTRACT | T6 (Community Reports) | 0.60-0.75 |
| ENCODE | T7 (Text Embedding) | 0.85-0.95 |

---

**Next**: [Query Semantics](03-query-semantics.md)
