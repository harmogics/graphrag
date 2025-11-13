# Query Flows Examples

## Обзор

Этот документ содержит примеры информационных потоков для **выполнения запросов** в GraphRAG, описанных на GFL.

## Query Execution Pipeline

### Complete Query Flow with Routing

```gfl
FLOW QueryExecution(user_query):
  /*
    Main entry point for query execution
    Analyzes query and routes to appropriate search strategy
  */

  // ========== PHASE 1: QUERY ANALYSIS (Q1) ==========
  QUERY user_query YIELD query_object

  ANALYZE query_object
    EXTRACT intent, key_concepts, scope, complexity
    CLASSIFY query_type IN [global, local, drift, basic]
    DETECT language
    YIELD query_metadata

    @cost: negligible (rule-based)
    @latency: <100ms

  // ========== PHASE 2: ROUTE TO STRATEGY ==========
  ROUTE query_metadata.query_type:
    WHEN "global" THEN GlobalSearchFlow(query_object, query_metadata)
    WHEN "local" THEN LocalSearchFlow(query_object, query_metadata)
    WHEN "drift" THEN DriftSearchFlow(query_object, query_metadata)
    WHEN "basic" THEN BasicSearchFlow(query_object, query_metadata)
    DEFAULT LocalSearchFlow(query_object, query_metadata)

  RETURN answer_with_metadata
```

### Query Analysis Details

```gfl
FLOW QueryAnalysisDetailed(user_query):
  /*
    Detailed query analysis and intent classification
  */

  PARSE user_query
    EXTRACT tokens, phrases, named_entities
    YIELD parsed_query

  // Intent classification
  CLASSIFY intent:
    WHEN query CONTAINS ["overview", "trends", "summary", "main themes"]
      THEN intent = GLOBAL
      REASON "Broad overview questions"

    WHEN query CONTAINS ["what is", "who is", "define", "explain", "how does"]
      THEN intent = LOCAL
      REASON "Specific factual questions"

    WHEN query CONTAINS ["related", "connection", "relationship", "how are", "link"]
      THEN intent = DRIFT
      REASON "Relationship exploration"

    WHEN query IS simple_lookup
      THEN intent = BASIC
      REASON "Direct entity lookup"

    DEFAULT intent = LOCAL

  // Extract key concepts
  EXTRACT key_concepts
    FROM parsed_query
    WITH method=noun_phrase_extraction
    YIELD concepts

  // Determine scope
  EVALUATE scope:
    IF LENGTH(concepts) > 5 THEN scope = BROAD
    ELIF LENGTH(concepts) >= 2 THEN scope = MODERATE
    ELSE scope = NARROW

  // Assess complexity
  EVALUATE complexity:
    factors = [
      query_length,
      num_concepts,
      has_conditions,
      requires_comparison,
      requires_aggregation
    ]
    complexity_score = COMPUTE(factors)
    YIELD complexity_score

  RETURN QueryMetadata {
    intent: intent,
    concepts: concepts,
    scope: scope,
    complexity: complexity_score,
    original_query: user_query
  }
```

## Local Search Flow

### Standard Local Search

```gfl
FLOW LocalSearchFlow(query, metadata):
  /*
    Local search for specific, detailed questions
    Uses entities + text units + relationships
  */

  // ========== PHASE 1: QUERY EMBEDDING (Q2) ==========
  EMBED query
    INTO query_vector
    USING model=text-embedding-3-small
    WITH dimensions=1536
    YIELD query_vector

    @cost: negligible (<$0.0001)
    @latency: ~50ms

  // ========== PHASE 2: SEMANTIC RETRIEVAL (Q3) ==========
  // Retrieve entities
  RETRIEVE entities
    FROM entity_embeddings
    MATCHING query_vector
    WITH top_k=30,
         similarity_threshold=0.7,
         hybrid_alpha=0.7
    YIELD relevant_entities

    @preserves: 85-90%
    @cost: negligible (vector search)
    @latency: <100ms

  // Retrieve text units
  RETRIEVE text_units
    FROM text_unit_embeddings
    MATCHING query_vector
    WITH top_k=20,
         similarity_threshold=0.7
    YIELD relevant_text_units

  // Expand with relationships
  FOR EACH entity IN relevant_entities:
    RETRIEVE relationships
      WHERE source=entity OR target=entity
    YIELD entity_relationships

  COLLECT entity_relationships INTO all_relationships

  // ========== PHASE 3: CONTEXT BUILDING (Q4) ==========
  BUILD context
    FROM relevant_entities, relevant_text_units, all_relationships
    WITH max_tokens=8000,
         format=structured_tables,
         deduplicate=true,
         sort_by=relevance
    YIELD structured_context

    @preserves: 95%+
    @cost: negligible
    @latency: <500ms

  // ========== PHASE 4: ANSWER GENERATION (Q5) ==========
  SYNTHESIZE answer
    FROM structured_context
    USING llm=gpt-4
    WITH prompt=LocalSearchPrompt,
         query=query,
         response_type="detailed with citations",
         temperature=0.0,
         max_tokens=2000
    YIELD answer_with_sources

    @preserves: 80-90%
    @cost: ~$0.09 per query
    @latency: 2-4s
    @quality: {correctness: 0.85, citation_accuracy: 0.90}

  // ========== PHASE 5: POST-PROCESSING ==========
  VALIDATE answer
    ENSURE has_citations
    ENSURE grounded_in_context
    CHECK hallucination_rate < 0.10

  RETURN answer_with_sources
```

### Enhanced Local Search with Re-ranking

```gfl
FLOW EnhancedLocalSearch(query, metadata):
  /*
    Local search with semantic re-ranking
  */

  EMBED query INTO query_vector

  // Initial retrieval (cast wide net)
  RETRIEVE entities
    MATCHING query_vector
    WITH top_k=100
    YIELD candidate_entities

  // Re-rank using LLM
  RERANK candidate_entities
    USING llm=gpt-3.5-turbo
    WITH query=query,
         method=cross_encoder
    YIELD reranked_entities

  // Take top after re-ranking
  SELECT TOP 30 FROM reranked_entities
    YIELD final_entities

  RETRIEVE text_units MATCHING query_vector WITH top_k=20
  EXPAND final_entities WITH relationships

  BUILD context
    FROM final_entities, text_units, relationships
    WITH max_tokens=10000

  SYNTHESIZE answer FROM context USING llm=gpt-4

  RETURN answer
```

## Global Search Flow

### Standard Global Search (Map-Reduce)

```gfl
FLOW GlobalSearchFlow(query, metadata):
  /*
    Global search for overview questions
    Uses community reports with map-reduce pattern
  */

  // ========== PHASE 1: QUERY EMBEDDING (Q2) ==========
  EMBED query INTO query_vector

  // ========== PHASE 2: RETRIEVE COMMUNITIES (Q3) ==========
  RETRIEVE community_reports
    FROM community_embeddings
    MATCHING query_vector
    WITH top_k=10,
         similarity_threshold=0.7,
         filter_by_level=0  // Use lowest level communities
    YIELD relevant_reports

    @cost: negligible
    @latency: <100ms

  // ========== PHASE 3: MAP PHASE (Q5) ==========
  /*
    Generate intermediate answers from each community report
    Execute in parallel for speed
  */

  PARALLEL FOR EACH report IN relevant_reports:
    CALL llm=gpt-4
      WITH prompt=GlobalMapPrompt,
           context=report,
           query=query,
           temperature=0.0,
           max_tokens=1000
      YIELD intermediate_answer, relevance_score

      @cost: ~$0.03 per report
      @latency: 2-3s per report (parallel)

  COLLECT intermediate_answer INTO map_responses

  // ========== PHASE 4: REDUCE PHASE (Q6) ==========
  /*
    Synthesize final answer from intermediate answers
  */

  AGGREGATE map_responses
    RANKED BY relevance_score DESC
    FILTER WHERE relevance_score >= 7.0
    YIELD ranked_responses

  SYNTHESIZE final_answer
    FROM ranked_responses
    USING llm=gpt-4
    WITH prompt=GlobalReducePrompt,
         response_type="comprehensive summary with sections",
         temperature=0.0,
         max_tokens=2000
    YIELD synthesized_answer

    @preserves: 75-85%
    @cost: ~$0.09
    @latency: 3-5s
    @quality: {completeness: 0.92, synthesis_quality: 0.85}

  // ========== PHASE 5: POST-PROCESSING ==========
  FORMAT synthesized_answer
    AS markdown
    WITH sections=true,
         bullet_points=true,
         preserve_citations=true

  RETURN synthesized_answer
```

### Dynamic Global Search (Adaptive)

```gfl
FLOW DynamicGlobalSearch(query, metadata):
  /*
    Adaptive global search that adjusts based on intermediate results
  */

  EMBED query INTO query_vector

  // Initial retrieval
  RETRIEVE community_reports
    WITH top_k=5
    YIELD initial_reports

  // Map phase 1
  FOR EACH report IN initial_reports:
    GENERATE intermediate_answer WITH llm
    EVALUATE coverage_score
    YIELD answer, score

  // Assess coverage
  AGGREGATE coverage_scores INTO overall_coverage

  // Adaptive expansion
  IF overall_coverage < 0.8 THEN:
    // Retrieve more communities
    RETRIEVE additional_reports
      WITH top_k=5, skip=5
      REASON "Initial coverage insufficient"

    // Map phase 2
    FOR EACH report IN additional_reports:
      GENERATE intermediate_answer
      YIELD additional_answer

    COLLECT additional_answer INTO map_responses

  // Reduce all responses
  SYNTHESIZE final_answer FROM all_map_responses

  RETURN final_answer
```

## Drift Search Flow

### Standard Drift Search (Iterative Exploration)

```gfl
FLOW DriftSearchFlow(query, metadata):
  /*
    Drift search for exploratory questions about connections
    Uses iterative graph walk with follow-up questions
  */

  // ========== PHASE 1: PRIMER ==========
  /*
    Get initial answer and seed follow-up questions
    from community reports
  */

  EMBED query INTO query_vector

  RETRIEVE community_reports
    MATCHING query_vector
    WITH top_k=3
    YIELD seed_reports

  CALL llm=gpt-4
    WITH prompt=DriftPrimerPrompt,
         community_reports=seed_reports,
         query=query
    YIELD initial_answer, seed_follow_ups, initial_score

    @cost: ~$0.05
    @latency: 3-5s

  // ========== PHASE 2: ITERATIVE HOPS ==========
  /*
    Execute follow-up questions iteratively
    Each hop explores deeper into the graph
  */

  SET max_hops = 3
  SET current_hop = 1
  SET accumulated_answers = [initial_answer]

  WHILE current_hop <= max_hops AND initial_score < 90:
    // Select next follow-up question
    SELECT follow_up
      FROM seed_follow_ups
      ORDER BY relevance DESC
      LIMIT 1
      YIELD next_question

    // Execute local search for follow-up
    EMBED next_question INTO follow_up_vector

    RETRIEVE entities, text_units
      MATCHING follow_up_vector
      WITH top_k_entities=15,
           top_k_text_units=10
      YIELD hop_context

    // Generate answer for this hop
    SYNTHESIZE hop_answer
      FROM hop_context
      USING llm=gpt-4
      WITH prompt=DriftLocalPrompt,
           query=next_question,
           global_query=query
      YIELD hop_answer, hop_score, new_follow_ups

      @cost: ~$0.06 per hop
      @latency: 3-5s per hop

    // Update state
    APPEND hop_answer TO accumulated_answers
    UPDATE seed_follow_ups WITH new_follow_ups
    INCREMENT current_hop

    // Early stop if good enough
    IF hop_score >= 90 THEN BREAK

  // ========== PHASE 3: SYNTHESIS ==========
  /*
    Combine all hop answers into coherent final answer
  */

  SYNTHESIZE final_answer
    FROM accumulated_answers
    USING llm=gpt-4
    WITH prompt=DriftReducePrompt,
         query=query,
         format=markdown
    YIELD comprehensive_answer

    @cost: ~$0.04
    @latency: 2-3s

  // ========== PHASE 4: FOLLOW-UP GENERATION ==========
  GENERATE follow_up_questions
    FROM final_answer
    WITH count=5,
         focus=unexplored_connections
    YIELD suggested_follow_ups

  RETURN DriftSearchResult {
    answer: comprehensive_answer,
    follow_ups: suggested_follow_ups,
    hops_executed: current_hop,
    total_cost: COMPUTE_COST(accumulated_answers)
  }
```

### Targeted Drift Search

```gfl
FLOW TargetedDriftSearch(query, start_entities, end_entities):
  /*
    Drift search with known start and end points
    Find connection paths between specific entities
  */

  // Validate input entities exist
  VERIFY start_entities, end_entities IN knowledge_graph

  // Initialize breadth-first search
  SET visited = []
  SET queue = [(start_entities, path=[], depth=0)]
  SET max_depth = 5

  WHILE queue NOT EMPTY AND depth < max_depth:
    DEQUEUE (current_entities, path, depth)

    // Check if reached target
    IF current_entities INTERSECTS end_entities THEN:
      RECORD path AS connection_path
      BREAK

    // Explore neighbors
    FOR EACH entity IN current_entities:
      IF entity NOT IN visited THEN:
        RETRIEVE connected_entities
          FROM entity.relationships
          WHERE strength >= 6
          YIELD neighbors

        FOR EACH neighbor IN neighbors:
          ENQUEUE (neighbor, path + [entity], depth + 1)

        ADD entity TO visited

  // Build narrative from path
  IF connection_path FOUND THEN:
    SYNTHESIZE narrative
      FROM connection_path
      USING llm=gpt-4
      WITH prompt="Explain the connection: {start} → ... → {end}"
      YIELD connection_narrative

    RETURN connection_narrative
  ELSE:
    RETURN "No direct connection found within {max_depth} hops"
```

## Basic Search Flow

### Simple Entity Lookup

```gfl
FLOW BasicSearchFlow(query, metadata):
  /*
    Simple search for direct entity lookup
    Fast path for simple questions
  */

  // Extract entity name from query
  PARSE query
    EXTRACT entity_name
    YIELD target_entity

  // Direct lookup
  RETRIEVE entity
    FROM knowledge_graph.entities
    WHERE name = target_entity
    YIELD found_entity

  IF found_entity EXISTS THEN:
    // Get entity description and relationships
    RETRIEVE relationships
      WHERE source = found_entity OR target = found_entity
      LIMIT 10
      YIELD entity_relationships

    // Simple formatting
    FORMAT response AS:
      """
      {found_entity.name} ({found_entity.type})

      Description: {found_entity.description}

      Key Relationships:
      {FORMAT entity_relationships AS bullets}
      """

    RETURN formatted_response
  ELSE:
    RETURN "Entity not found in knowledge graph"
```

## Hybrid Search Flows

### Multi-Strategy Search

```gfl
FLOW HybridSearch(query, metadata):
  /*
    Combine multiple search strategies
    Use for complex multi-part questions
  */

  PARSE query INTO sub_queries
    YIELD overview_part, specific_part

  // Execute global search for overview
  PARALLEL:
    global_result = GlobalSearchFlow(overview_part, metadata)
    local_result = LocalSearchFlow(specific_part, metadata)

  // Merge results
  SYNTHESIZE combined_answer
    FROM global_result, local_result
    USING llm=gpt-4
    WITH prompt="Combine overview and specific details coherently"
    YIELD final_answer

  RETURN final_answer
```

### Comparison Query

```gfl
FLOW ComparisonQuery(query, metadata):
  /*
    Handle comparison questions: "Compare X and Y"
  */

  PARSE query
    EXTRACT entities_to_compare
    YIELD entity_A, entity_B

  // Retrieve information for each entity
  PARALLEL:
    info_A = LocalSearchFlow("Tell me about {entity_A}", metadata)
    info_B = LocalSearchFlow("Tell me about {entity_B}", metadata)

  // Structured comparison
  SYNTHESIZE comparison
    FROM info_A, info_B
    USING llm=gpt-4
    WITH prompt=ComparisonPrompt,
         format=comparison_table
    YIELD comparison_result

  RETURN comparison_result
```

## Conversational Query Flow

### Multi-Turn Conversation

```gfl
FLOW ConversationalQuery(conversation_history, new_query):
  /*
    Handle follow-up questions in context of conversation
  */

  // Contextualize query with history
  REWRITE new_query
    USING conversation_history
    WITH max_turns=5
    YIELD contextualized_query

  // Standard query execution
  ROUTE contextualized_query TO appropriate_search

  GET answer FROM search_result

  // Track conversation state
  UPDATE conversation_history
    ADD (query: new_query, answer: answer)

  // Generate contextual follow-ups
  GENERATE follow_ups
    FROM conversation_history
    WITH focus=next_logical_questions
    YIELD suggested_questions

  RETURN ConversationalResult {
    answer: answer,
    follow_ups: suggested_questions,
    conversation_id: conversation_history.id
  }
```

## Advanced Query Patterns

### Aggregation Query

```gfl
FLOW AggregationQuery(query, metadata):
  /*
    Handle aggregation: "How many X?", "List all Y"
  */

  PARSE query
    EXTRACT aggregation_type, target_entity_type
    YIELD agg_type, entity_type

  ROUTE agg_type:
    WHEN "count" THEN:
      RETRIEVE entities WHERE type = entity_type
      COUNT entities YIELD count
      RETURN "There are {count} {entity_type} entities"

    WHEN "list" THEN:
      RETRIEVE entities WHERE type = entity_type
      FORMAT AS bulleted_list
      RETURN entity_list

    WHEN "summary" THEN:
      RETRIEVE entities WHERE type = entity_type
      AGGREGATE entities BY common_attributes
      SYNTHESIZE summary USING llm
      RETURN aggregated_summary
```

### Temporal Query

```gfl
FLOW TemporalQuery(query, metadata):
  /*
    Handle time-based queries: "What happened in 2020?"
  */

  PARSE query
    EXTRACT time_range, event_type
    YIELD start_date, end_date, event_filter

  RETRIEVE events
    FROM knowledge_graph
    WHERE timestamp BETWEEN start_date AND end_date
      AND type MATCHES event_filter
    ORDER BY timestamp ASC
    YIELD timeline_events

  SYNTHESIZE timeline_narrative
    FROM timeline_events
    USING llm=gpt-4
    WITH format=chronological_narrative
    YIELD narrative

  RETURN narrative
```

## Quality Assurance in Query Flow

### Query Flow with Validation

```gfl
FLOW ValidatedQueryFlow(query, metadata):
  /*
    Query execution with comprehensive validation
  */

  ROUTE query TO search_strategy
  GET answer FROM search_strategy

  // Validation checks
  VALIDATE answer:
    CHECK has_grounding
    CHECK citation_accuracy > 0.90
    CHECK hallucination_rate < 0.10
    CHECK relevance_to_query > 0.80
    YIELD validation_results

  IF validation_results.passed THEN:
    RETURN answer
  ELSE:
    // Attempt refinement
    REFINE answer
      WITH stricter_grounding=true,
           increased_context=true
      YIELD refined_answer

    VALIDATE refined_answer
    IF still_invalid THEN:
      RETURN "Unable to generate reliable answer"
    ELSE:
      RETURN refined_answer
```

## Performance Monitoring

### Instrumented Query Flow

```gfl
FLOW InstrumentedQueryFlow(query, metadata):
  /*
    Query flow with detailed performance tracking
  */

  METRICS:
    query_count: Counter
    query_latency: Histogram
    query_cost: Counter
    quality_scores: Gauge
    cache_hit_rate: Gauge

  START_TIMER total_query_time

  // Check cache first
  CHECK cache FOR query
  IF cache_hit THEN:
    RECORD INTO METRICS.cache_hit_rate
    RETURN cached_answer

  // Execute query
  ROUTE query TO search_strategy
    RECORD latency INTO METRICS.query_latency
    RECORD cost INTO METRICS.query_cost

  GET answer FROM search_strategy

  // Quality evaluation
  EVALUATE answer_quality
    YIELD correctness, completeness, citation_accuracy
  RECORD quality INTO METRICS.quality_scores

  // Cache result
  CACHE answer WITH ttl=1hour

  STOP_TIMER total_query_time

  LOG "Query completed: latency={total_query_time}, cost=${cost}, quality={quality}"

  RETURN answer
```

---

**Next**: [Semantics and Execution Model](04-semantics.md)
