# Query Semantics

## Обзор

Этот документ описывает **семантику выполнения запросов** в GraphRAG, используя язык SFL. Query execution представляет собой серию семантических трансформаций от пользовательского запроса до итогового ответа.

## Полная семантическая цепочка запросов

```
User Query (natural language)
  ↓ [INTERPRET: 0.92]
Semantic Intent (structured: type, concepts, scope)
  ↓ [ENCODE: 0.92]
Query Vector (geometric representation)
  ↓ [NAVIGATE: 0.88]
Relevant Semantic Space (candidates)
  ↓ [LOCATE: 0.88]
Precise Semantics (entities, text_units, summaries)
  ↓ [COMPOSE: 0.85]
Final Answer (natural language + citations)

Overall intent preservation: 0.92 * 0.92 * 0.88 * 0.88 * 0.85 ≈ 0.56
Semantic enrichment: +1.05
Net answer quality: 1.61x query intent
```

---

## Query Routing Semantics

```sfl
SEMANTIC FLOW QueryRoutingSemantics:
  /*
    Маршрутизация запроса на основе семантического намерения
  */

  // ==================== PHASE 1: INTENT INTERPRETATION ====================
  INTERPRET query_text
    FROM natural_language
    INTO semantic_intent
    STRATEGY: pattern_analysis + keyword_detection

    EXTRACTING:
      query_type: {global, local, drift, basic}
      semantic_scope: {broad, narrow, exploratory, specific}
      key_concepts: [entity_mentions, topics]
      temporal_aspect: {current, historical, any}
      expected_answer_type: {summary, explanation, list, comparison}

    PRESERVING:
      core_information_need: 0.92     // User's need captured
      key_concepts: 0.90              // Main concepts identified
      intent_specificity: 0.88        // Specificity level preserved

    LOSING:
      exact_wording: 0.90             // Rephrased
      linguistic_style: 0.95          // Style lost

    GAINING:
      structured_intent: 0.85         // Structured representation
      routable_form: 0.90             // Can route to strategy

  // ==================== PHASE 2: SEMANTIC ROUTING ====================
  ROUTE BY semantic_intent.query_type:

    WHEN query_type IS GLOBAL:
      // Overview questions: "What are the main trends?"
      SEMANTIC_CHARACTERISTICS:
        - Scope: BROAD (wants overview)
        - Coverage: HIGH (many topics)
        - Detail: LOW (summary level)
        - Navigation: Community summaries
      ROUTE TO: GlobalSearchSemantics

    WHEN query_type IS LOCAL:
      // Specific questions: "What is X?", "How does Y work?"
      SEMANTIC_CHARACTERISTICS:
        - Scope: NARROW (focused on entity)
        - Coverage: LOW (specific entity + neighbors)
        - Detail: HIGH (detailed explanation)
        - Navigation: Specific entities
      ROUTE TO: LocalSearchSemantics

    WHEN query_type IS DRIFT:
      // Connection questions: "How are X and Y related?"
      SEMANTIC_CHARACTERISTICS:
        - Scope: EXPLORATORY (multi-hop)
        - Coverage: MEDIUM (connected entities)
        - Detail: MEDIUM (relationship paths)
        - Navigation: Graph traversal
      ROUTE TO: DriftSearchSemantics

    DEFAULT:
      ROUTE TO: BasicSearchSemantics
```

---

## Global Search Semantics

```sfl
SEMANTIC FLOW GlobalSearchSemantics:
  /*
    Семантика Global Search (Map-Reduce)
    Для overview questions, broad topics
  */

  // ==================== PHASE 1: QUERY ENCODING ====================
  ENCODE query_intent
    FROM symbolic_intent
    INTO query_vector
    STRATEGY: neural_embedding
      USING model=text-embedding-3-small

    PRESERVING:
      semantic_meaning: 0.92          // Query meaning preserved
      conceptual_relationships: 0.88  // Concept relationships kept

    TRANSFORMING:
      text → vector                   // Symbolic → geometric
      discrete → continuous           // Discrete → continuous space

    ENABLING:
      geometric_similarity: cosine_distance
      fast_search: O(log n)

    METRICS:
      input: semantic_intent (text)
      output: query_vector (1536-dim)
      preservation: 0.92
      latency: <100ms

  SEMANTIC_STATE_1:
    representation: Geometric (vector)
    queryability: Geometric search enabled
    semantic_value: 0.92

  // ==================== PHASE 2: SEMANTIC NAVIGATION ====================
  NAVIGATE embedding_space
    TOWARD query_vector
    STRATEGY: community_summary_retrieval
      FOR query_type=GLOBAL

    LOCATING:
      community_summaries: top_k=10         // Broad coverage
      semantic_relevance: threshold=0.7     // Relevance filter
      diversity: maximize_topic_coverage    // Diverse perspectives

    PRESERVING:
      relevance_precision: 0.85             // Relevant summaries
      topic_diversity: 0.82                 // Diverse topics

    METRICS:
      input: query_vector
      output: 10 community_summaries
      coverage: ~1200 entities (indirect)
      avg_relevance: 0.82
      latency: <500ms

  SEMANTIC_STATE_2:
    representation: Community summaries (text + structure)
    coverage: Broad (24% of knowledge graph)
    semantic_value: 0.92 * 0.85 = 0.78

  // ==================== PHASE 3: MAP PHASE ====================
  FOR EACH community_summary IN located_summaries:
    COMPOSE intermediate_answer
      FROM community_summary
      STRATEGY: llm_guided
        USING llm=gpt-4
        WITH prompt=GlobalMapPrompt

      PRESERVING:
        community_perspective: 0.85         // Perspective captured
        factual_accuracy: 0.82              // Facts correct
        relevance_to_query: 0.80            // Relevant to query

      GENERATING:
        intermediate_perspective: summary_view
        relevance_score: 0-100
        key_points: [point1, point2, ...]

      METRICS:
        input: 1 community_summary (~2000 tokens)
        output: intermediate_answer (~300 tokens)
        latency: 2-3s per community
        cost: ~$0.04 per community

  COLLECT intermediate_answers
    YIELD: 10 perspectives

  SEMANTIC_STATE_3:
    representation: Multiple perspectives
    coverage: 10 community views
    semantic_value: 0.78 * 0.85 = 0.66

  // ==================== PHASE 4: REDUCE PHASE ====================
  COMPOSE final_answer
    FROM intermediate_answers
    STRATEGY: llm_guided_synthesis
      USING llm=gpt-4
      WITH prompt=GlobalReducePrompt

    INTEGRATING:
      multiple_perspectives: 10 communities     // Multi-view
      diverse_viewpoints: cross_community       // Diverse
      ranked_by_relevance: score_descending     // Best first

    PRESERVING:
      comprehensiveness: 0.85                   // Covers multiple aspects
      factual_accuracy: 0.78                    // Facts from sources
      semantic_coherence: 0.88                  // Coherent narrative
      query_relevance: 0.82                     // Answers query

    LOSING:
      source_verbatim: 0.90                     // Not exact quotes
      fine_details: 0.70                        // High-level only

    GAINING:
      integrated_view: 0.80                     // Synthesized view
      comprehensive_coverage: 0.85              // Broad coverage
      multi_perspective: 0.75                   // Multiple views

    GENERATING:
      comprehensive_answer: natural_language
      grounded_claims: with_citations
      structured_overview: organized

    METRICS:
      input: 10 intermediate_answers (~3000 tokens total)
      output: final_answer (~800 tokens)
      latency: 3-5s
      cost: ~$0.05

  SEMANTIC_STATE_4:
    representation: Synthesized answer
    semantic_value: 0.66 * 0.85 = 0.56

  // ==================== SEMANTIC BUDGET ====================
  SEMANTIC_BUDGET:
    Query intent captured: 0.92
    Encoded to vector: 0.92 * 0.92 = 0.85
    Navigated to communities: 0.85 * 0.85 = 0.72
    Map phase perspectives: 0.72 * 0.85 = 0.61
    Reduce phase synthesis: 0.61 * 0.85 = 0.52

    TOTAL PRESERVATION: 0.52

    BUT GAINED:
    + Comprehensive coverage: 0.85  // Broad view
    + Multi-perspective: 0.75       // Multiple views
    + Contextual enrichment: 0.40   // Added context
    + Synthesis quality: 0.50       // Integrated narrative

    TOTAL GAINED: 2.50

    NET ANSWER QUALITY:
      0.52 (intent) + 2.50 (enrichment) = 3.02

    METRICS:
      Total latency: ~25s (2-3s * 10 map + 3-5s reduce)
      Total cost: ~$0.45 (10 map + 1 reduce)
      Comprehensiveness: 0.85
      Factual accuracy: 0.78
```

---

## Local Search Semantics

```sfl
SEMANTIC FLOW LocalSearchSemantics:
  /*
    Семантика Local Search
    Для specific questions, detailed explanations
  */

  // ==================== PHASE 1: QUERY ENCODING ====================
  ENCODE query_intent
    INTO query_vector
    // Same as Global

    METRICS:
      preservation: 0.92

  // ==================== PHASE 2: SEMANTIC NAVIGATION ====================
  NAVIGATE embedding_space
    TOWARD query_vector
    STRATEGY: entity_retrieval + text_unit_retrieval
      FOR query_type=LOCAL

    LOCATING:
      specific_entities: top_k=30             // Focused entities
      supporting_text_units: top_k=20         // Original context
      relationships: connected_to_entities    // Graph connections

    PRESERVING:
      relevance_precision: 0.88               // High precision
      context_completeness: 0.85              // Complete context

    METRICS:
      input: query_vector
      output: 30 entities + 20 text_units
      precision@30: 0.85
      recall: 0.80
      latency: <1s

  SEMANTIC_STATE_2:
    representation: Entities + text_units + relationships
    coverage: Narrow but deep
    semantic_value: 0.92 * 0.88 = 0.81

  // ==================== PHASE 3: CONTEXT BUILDING ====================
  LOCATE relevant_semantics
    IN navigated_entities
    STRATEGY: expand_with_relationships

    EXPANDING:
      primary_entities: 30 entities           // Direct matches
      related_entities: +50 via relationships // Graph expansion
      text_units: 20 units                    // Original text
      relationships: 100 edges                // Connections

    PRESERVING:
      entity_semantics: 0.95                  // Entities intact
      relationship_semantics: 0.92            // Relationships intact
      textual_context: 0.90                   // Text preserved

    BUILDING context_structure:
      primary_focus: query_entities
      supporting_context: related_entities
      textual_grounding: text_units
      structural_context: relationships

    METRICS:
      input: 30 entities, 20 text_units
      output: 80 entities, 20 text_units, 100 relationships
      context_size: ~8000 tokens
      completeness: 0.85

  SEMANTIC_STATE_3:
    representation: Rich context (entities + text + graph)
    depth: High (detailed)
    semantic_value: 0.81 * 0.95 = 0.77

  // ==================== PHASE 4: ANSWER COMPOSITION ====================
  COMPOSE answer
    FROM context_structure
    STRATEGY: context_based_generation
      USING llm=gpt-4
      WITH prompt=LocalSearchPrompt

    INTEGRATING:
      entity_descriptions: detailed           // Full descriptions
      textual_evidence: text_units            // Original text
      graph_structure: relationships          // Connections
      contextual_entities: related            // Surrounding context

    PRESERVING:
      factual_accuracy: 0.88                  // Grounded in sources
      contextual_relevance: 0.85              // Relevant context
      semantic_coherence: 0.90                // Coherent explanation
      detail_level: 0.85                      // Detailed answer

    GENERATING:
      detailed_explanation: natural_language
      citations: text_unit_references         // Grounded
      relationship_context: graph_info        // Structural info

    METRICS:
      input: ~8000 tokens context
      output: ~600 tokens answer
      latency: 3-4s
      cost: ~$0.09

  SEMANTIC_STATE_4:
    representation: Detailed answer
    semantic_value: 0.77 * 0.90 = 0.69

  // ==================== SEMANTIC BUDGET ====================
  SEMANTIC_BUDGET:
    Query intent: 0.92
    Encoded: 0.92 * 0.92 = 0.85
    Navigated: 0.85 * 0.88 = 0.75
    Context built: 0.75 * 0.95 = 0.71
    Answer composed: 0.71 * 0.90 = 0.64

    TOTAL PRESERVATION: 0.64

    BUT GAINED:
    + Detailed context: 0.85        // Rich context
    + Textual grounding: 0.90       // Original text
    + Structural insights: 0.75     // Graph structure
    + Citation support: 0.85        // Grounded claims

    TOTAL GAINED: 3.35

    NET ANSWER QUALITY:
      0.64 (intent) + 3.35 (enrichment) = 3.99

    METRICS:
      Total latency: ~4s
      Total cost: ~$0.09
      Factual accuracy: 0.88
      Detail level: 0.85
```

---

## Drift Search Semantics

```sfl
SEMANTIC FLOW DriftSearchSemantics:
  /*
    Семантика Drift Search (Graph Traversal)
    Для connection questions, exploratory queries
  */

  // ==================== PHASE 1: QUERY ENCODING ====================
  ENCODE query_intent
    INTO query_vector
    EXTRACTING source_concept, target_concept
    // Same encoding as Local

  // ==================== PHASE 2: ANCHOR IDENTIFICATION ====================
  NAVIGATE embedding_space
    TOWARD query_vector
    STRATEGY: dual_anchor_identification

    LOCATING:
      source_entities: entities_for(source_concept)  // Start points
      target_entities: entities_for(target_concept)  // End points

    PRESERVING:
      anchor_precision: 0.90                         // Correct anchors

    METRICS:
      input: query with 2 concepts
      output: 5 source entities, 5 target entities
      anchor_accuracy: 0.90

  SEMANTIC_STATE_2:
    representation: Anchor entities
    semantic_value: 0.92 * 0.90 = 0.83

  // ==================== PHASE 3: GRAPH TRAVERSAL ====================
  NAVIGATE entity_graph
    FROM source_entities
    TOWARD target_entities
    STRATEGY: multi_hop_traversal
      WITH max_hops=3
      WITH path_ranking=semantic_relevance

    DISCOVERING:
      connection_paths: 25 paths                     // Paths found
      intermediate_entities: 150 entities            // Discovered
      semantic_bridges: key_connecting_entities      // Important nodes

    PRESERVING:
      path_semantics: 0.88                           // Path meaning kept
      relationship_semantics: 0.85                   // Relationships kept

    GAINING:
      discovered_connections: 0.80                   // Found connections
      multi_hop_insights: 0.75                       // Multi-hop understanding

    METRICS:
      input: 5 source, 5 target entities
      output: 25 paths, 150 entities
      avg_path_length: 2.3 hops
      path_diversity: 0.82

  SEMANTIC_STATE_3:
    representation: Connection paths + entities
    coverage: Exploratory (connected space)
    semantic_value: 0.83 * 0.88 = 0.73

  // ==================== PHASE 4: PATH ANALYSIS ====================
  LOCATE significant_paths
    IN discovered_paths
    RANKED BY [semantic_relevance, path_strength, uniqueness]

    IDENTIFYING:
      primary_paths: top 10 most relevant           // Key connections
      key_intermediaries: high_betweenness          // Important bridges
      semantic_clusters: path_grouping              // Grouped paths

    PRESERVING:
      connection_semantics: 0.85                    // Connection meaning
      path_diversity: 0.80                          // Diverse paths

  // ==================== PHASE 5: ANSWER COMPOSITION ====================
  COMPOSE answer
    FROM significant_paths
    STRATEGY: path_narrative_generation
      USING llm=gpt-4
      WITH prompt=DriftSearchPrompt

    INTEGRATING:
      connection_paths: multi_hop                   // Paths
      intermediate_entities: bridge_concepts        // Bridges
      relationship_types: edge_semantics            // Relationships
      path_strengths: weighted                      // Strength scores

    PRESERVING:
      factual_accuracy: 0.82                        // Facts correct
      connection_validity: 0.85                     // Connections valid
      semantic_coherence: 0.88                      // Coherent narrative

    GENERATING:
      connection_narrative: natural_language        // Story
      key_intermediaries: highlighted               // Key bridges
      multiple_pathways: diverse_perspectives       // Alternative paths

    METRICS:
      input: 10 paths, 150 entities
      output: ~700 tokens answer
      latency: 4-5s per path analysis + 5-7s composition
      cost: ~$0.30

  SEMANTIC_STATE_5:
    representation: Connection narrative
    semantic_value: 0.73 * 0.85 = 0.62

  // ==================== SEMANTIC BUDGET ====================
  SEMANTIC_BUDGET:
    Query intent: 0.92
    Encoded: 0.92 * 0.92 = 0.85
    Anchors located: 0.85 * 0.90 = 0.77
    Graph traversed: 0.77 * 0.88 = 0.68
    Paths analyzed: 0.68 * 0.85 = 0.58

    TOTAL PRESERVATION: 0.58

    BUT GAINED:
    + Discovered connections: 0.80  // Found relationships
    + Multi-hop insights: 0.75      // Indirect connections
    + Path diversity: 0.80          // Multiple routes
    + Structural understanding: 0.70 // Graph structure

    TOTAL GAINED: 3.05

    NET ANSWER QUALITY:
      0.58 (intent) + 3.05 (discovery) = 3.63

    METRICS:
      Total latency: ~42s (traversal + analysis + composition)
      Total cost: ~$0.30
      Connection validity: 0.85
      Discovery value: 0.80 (new connections found)
```

---

## Basic Search Semantics

```sfl
SEMANTIC FLOW BasicSearchSemantics:
  /*
    Семантика Basic Search (Simple lookup)
    Для simple factual queries
  */

  ENCODE query_intent INTO query_vector

  NAVIGATE embedding_space
    STRATEGY: simple_retrieval
    LOCATING:
      best_matches: top_k=5
      match_type: [entity, text_unit]

  LOCATE exact_match
    PREFER entity_exact_match IF available

  COMPOSE answer
    FROM located_match
    STRATEGY: direct_extraction
      WITHOUT llm  // Direct return

    PRESERVING:
      factual_accuracy: 0.95         // Exact match
      source_fidelity: 0.98          // Direct from source

  SEMANTIC_BUDGET:
    Very high preservation: 0.92 * 0.95 * 0.98 = 0.86
    Minimal enrichment: +0.10
    Net quality: 0.96

    METRICS:
      Latency: <1s (no LLM)
      Cost: ~$0 (no LLM)
      Accuracy: 0.95 (exact matches)
```

---

## Семантическое сравнение стратегий поиска

| Strategy | Semantic Focus | Preservation | Enrichment | Net Quality | Latency | Cost |
|---|---|---|---|---|---|---|
| **Global** | Comprehensive overview | 0.52 | 2.50 | 3.02 | 25s | $0.45 |
| **Local** | Detailed explanation | 0.64 | 3.35 | 3.99 | 4s | $0.09 |
| **Drift** | Connection discovery | 0.58 | 3.05 | 3.63 | 42s | $0.30 |
| **Basic** | Exact factual lookup | 0.86 | 0.10 | 0.96 | 1s | $0 |

### Semantic Trade-offs

**Global Search**:
- ✅ Comprehensive coverage (0.85)
- ✅ Multi-perspective (0.75)
- ❌ Lower factual accuracy (0.78)
- ❌ High latency (25s)

**Local Search**:
- ✅ High factual accuracy (0.88)
- ✅ Detailed context (0.85)
- ✅ Fast (4s)
- ❌ Narrow coverage

**Drift Search**:
- ✅ Discovery of connections (0.80)
- ✅ Multi-hop insights (0.75)
- ❌ Highest latency (42s)
- ❌ Moderate accuracy (0.82)

**Basic Search**:
- ✅ Highest accuracy (0.95)
- ✅ Fastest (<1s)
- ✅ Free
- ❌ No enrichment, no context

---

## Advanced Query Patterns

### Conversational Query Semantics

```sfl
SEMANTIC FLOW ConversationalQuerySemantics:
  /*
    Семантика conversational queries с контекстом
  */

  INTERPRET current_query
    WITH conversation_history
    RESOLVING:
      pronouns: from_history             // "it", "they" → entities
      implicit_context: from_history     // Unstated context
      follow_up_intent: continuation     // Follow-up or new?

    PRESERVING:
      conversational_coherence: 0.90     // Context maintained
      intent_continuity: 0.88            // Continuous intent

  // Rest of the flow based on resolved intent
  ROUTE TO appropriate_strategy

  SEMANTIC_RESULT:
    Context-aware query processing
    Coherence: 0.90
```

### Aggregation Query Semantics

```sfl
SEMANTIC FLOW AggregationQuerySemantics:
  /*
    Семантика aggregation queries: "List all X", "Count Y"
  */

  INTERPRET query
    EXTRACTING:
      aggregation_type: {count, list, sum, average}
      entity_type: target_entity_type
      filters: conditions

  LOCATE ALL matching_entities
    WHERE meets_conditions

  COMPOSE aggregated_answer
    FROM matching_entities
    STRATEGY: aggregation_formatting
      WITH format=determined_by_type

  SEMANTIC_CHARACTERISTICS:
    Completeness focus: Recall > Precision
    Preservation: 0.85 (completeness)
    Enrichment: Minimal (factual)
```

---

## Semantic Metrics Across Query Types

### Preservation Metrics

| Query Type | Intent Preserved | Context Preserved | Accuracy | Coherence |
|---|---|---|---|---|
| Global | 0.52 | 0.70 | 0.78 | 0.88 |
| Local | 0.64 | 0.85 | 0.88 | 0.90 |
| Drift | 0.58 | 0.75 | 0.82 | 0.88 |
| Basic | 0.86 | 0.95 | 0.95 | N/A |

### Enrichment Metrics

| Query Type | Coverage | Detail | Multi-perspective | Discovery |
|---|---|---|---|---|
| Global | 0.85 | 0.60 | 0.75 | 0.40 |
| Local | 0.40 | 0.85 | 0.30 | 0.20 |
| Drift | 0.60 | 0.70 | 0.60 | 0.80 |
| Basic | 0.10 | 0.95 | 0.00 | 0.00 |

---

## Связь с трансформационными цепочками

| SFL Intention | GraphRAG Transform | Semantic Operation |
|---|---|---|
| INTERPRET | Q1 (Query Analysis) | Text → Structured intent |
| ENCODE | Q2 (Query Embedding) | Intent → Vector |
| NAVIGATE | Q3 (Semantic Retrieval) - start | Vector → Candidates |
| LOCATE | Q3 (Semantic Retrieval) - end | Candidates → Precise set |
| COMPOSE (intermediate) | Q5 (Answer Generation) | Context → Intermediate answer |
| COMPOSE (final) | Q6 (Answer Synthesis) | Intermediate → Final answer |

---

**Next**: [Semantic Scenarios](04-semantic-scenarios.md)
