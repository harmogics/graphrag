# Indexing Flows Examples

## Обзор

Этот документ содержит примеры информационных потоков для **индексации документов** в GraphRAG, описанных на GFL.

## Базовый Indexing Flow

### Стандартный pipeline

```gfl
FLOW StandardIndexing:
  /*
    Standard GraphRAG indexing pipeline
    Transforms raw documents into indexed knowledge graph
  */

  // ========== PHASE 1: INGESTION ==========
  INGEST documents
    FROM ./input/*.txt
    WITH encoding=utf-8
    YIELD raw_documents

    @cost: low
    @latency: <1s for 1000 docs

  // ========== PHASE 2: CHUNKING (T1) ==========
  CHUNK raw_documents
    INTO text_units
    WITH strategy=tokens,
         size=1200,
         overlap=100
    USING tokenizer=cl100k_base
    YIELD text_units

    @preserves: 95-98%
    @loss: minimal (boundary effects)
    @cost: low (algorithmic)
    @latency: <1s for 1000 docs

  // ========== PHASE 3: ENTITY EXTRACTION (T2 + T3) ==========
  EXTRACT entities, relationships
    FROM text_units
    USING llm=gpt-4
    WITH prompt=EntityExtractionPrompt,
         entity_types=[organization, person, geo, event],
         max_gleanings=1,
         temperature=0.0
    YIELD raw_entities, raw_relationships

    @preserves: 70-85%
    @loss: medium (intentional abstraction)
    @reversible: low
    @cost: high ($96 per 1K docs)
    @latency: 2-5s per chunk
    @quality: {precision: 0.92, recall: 0.85, f1: 0.83}

  // ========== PHASE 4: DESCRIPTION CONSOLIDATION (T4) ==========
  AGGREGATE raw_entities
    BY name
    INTO entity_groups
    YIELD entity_groups

  FOR EACH group IN entity_groups:
    SYNTHESIZE consolidated_description
      FROM group.descriptions
      USING llm=gpt-4
      WITH prompt=SummarizationPrompt,
           max_length=500
      YIELD consolidated_entity

    @preserves: 80-90%
    @loss: low-medium (nuances)
    @cost: medium ($3 per 1K docs)

  COLLECT consolidated_entity INTO consolidated_entities

  // ========== PHASE 5: GRAPH CONSTRUCTION ==========
  BUILD entity_graph
    FROM consolidated_entities, raw_relationships
    YIELD entity_graph

  // ========== PHASE 6: COMMUNITY DETECTION (T5) ==========
  CLUSTER entity_graph
    INTO communities
    USING algorithm=leiden
    WITH resolution=1.0,
         max_cluster_size=10,
         use_lcc=true
    YIELD communities

    @preserves: 90%+
    @loss: minimal (structural organization)
    @cost: low (algorithmic)
    @latency: <10s for 10K entities

  // ========== PHASE 7: COMMUNITY REPORTS (T6) ==========
  FOR EACH community IN communities:
    GENERATE report
      FROM community.entities, community.relationships
      USING llm=gpt-4
      WITH prompt=CommunityReportPrompt,
           max_length=2000,
           format=json
      YIELD community_report

    @preserves: 60-75%
    @loss: medium-high (high-level abstraction)
    @cost: medium ($6 per 1K docs)
    @quality: {factual_accuracy: 0.78, comprehensiveness: 0.82}

    STORE report IN community

  // ========== PHASE 8: EMBEDDING (T7) ==========
  EMBED text_units, consolidated_entities, communities
    INTO vectors
    USING model=text-embedding-3-small
    WITH dimensions=1536,
         batch_size=100
    YIELD embeddings

    @preserves: 85-95% (semantic)
    @loss: low (encoding)
    @reversible: low (approximate)
    @cost: low ($0.96 per 1K docs)

  // ========== PHASE 9: INDEXING ==========
  INDEX embeddings, entity_graph, communities
    INTO vector_store=lancedb, graph_store=memory
    WITH overwrite=true
    YIELD indexed_knowledge_graph

  RETURN indexed_knowledge_graph
```

## Optimized Indexing Flows

### Speed-Optimized Flow

```gfl
FLOW FastIndexing:
  /*
    Optimized for speed at the cost of some quality
    Use case: Large-scale indexing with time constraints
  */

  INGEST documents FROM source

  // Larger chunks = fewer chunks to process
  CHUNK documents
    WITH size=1500, overlap=50
    REASON "Fewer chunks reduce LLM calls"

  // No gleaning = faster but lower recall
  EXTRACT entities, relationships
    USING llm=gpt-3.5-turbo  // Faster than GPT-4
    WITH max_gleanings=0      // No refinement iterations
    REASON "Trade recall for speed"

  // Skip description consolidation for speed
  // Use raw entity descriptions directly

  BUILD graph FROM entities, relationships

  CLUSTER graph
    WITH max_cluster_size=15  // Larger clusters = fewer
    REASON "Reduce community report generation time"

  GENERATE reports FOR communities

  EMBED text_units, entities
    WITH batch_size=500  // Larger batches
    REASON "Maximize throughput"

  INDEX ALL

  /*
    Performance impact:
    - Latency: -60% (4x faster)
    - Cost: -40%
    - Quality: -15% (F1 score)
  */
```

### Quality-Optimized Flow

```gfl
FLOW HighQualityIndexing:
  /*
    Optimized for maximum quality
    Use case: High-value content requiring precision
  */

  INGEST documents FROM source

  // Smaller chunks = better precision
  CHUNK documents
    WITH size=800, overlap=150
    REASON "Preserve technical context and citations"

  // Maximum gleaning for best recall
  EXTRACT entities, relationships
    USING llm=gpt-4
    WITH max_gleanings=3,
         temperature=0.0
    REASON "Maximize entity extraction recall"

    ENSURE recall >= 0.90
    ENSURE precision >= 0.92

  // Careful consolidation with GPT-4
  AGGREGATE entities BY name
  SYNTHESIZE descriptions
    USING llm=gpt-4
    WITH max_length=500,
         preserve_details=true

  BUILD graph FROM consolidated_entities, relationships

  // Fine-grained clustering
  CLUSTER graph
    WITH max_cluster_size=8,
         resolution=0.8
    REASON "Smaller communities for detailed topics"

  // Longer, more detailed reports
  GENERATE reports
    WITH max_length=3000,
         detail_level=high

  EMBED ALL
    WITH dimensions=1536

  INDEX ALL

  /*
    Performance impact:
    - Latency: +50% (slower)
    - Cost: +30%
    - Quality: +20% (F1 score)
  */
```

### Cost-Optimized Flow

```gfl
FLOW CostOptimizedIndexing:
  /*
    Optimized for minimum cost
    Use case: Budget-constrained projects
  */

  INGEST documents

  // Standard chunking
  CHUNK documents WITH size=1200, overlap=100

  // Use GPT-3.5 instead of GPT-4
  EXTRACT entities, relationships
    USING llm=gpt-3.5-turbo
    WITH max_gleanings=0
    REASON "GPT-3.5 is 90% cheaper than GPT-4"

  // Minimal consolidation
  AGGREGATE entities BY name
  SYNTHESIZE descriptions
    USING llm=gpt-3.5-turbo
    WITH max_length=300

  BUILD graph
  CLUSTER graph

  // Shorter reports with GPT-3.5
  GENERATE reports
    USING llm=gpt-3.5-turbo
    WITH max_length=1500

  EMBED ALL

  INDEX ALL

  /*
    Cost impact:
    - Indexing: $21 per 1K docs (vs $105 with GPT-4)
    - Savings: 80%
    - Quality: -10-15%
  */
```

## Domain-Specific Flows

### Scientific Papers Indexing

```gfl
FLOW ScientificPapersIndexing:
  /*
    Specialized flow for scientific papers
    Focus: Methods, metrics, datasets, findings
  */

  INGEST papers
    FROM ./papers/*.pdf
    WITH parser=scientific_pdf
    YIELD papers

  // Smaller chunks for technical precision
  CHUNK papers
    WITH size=800, overlap=150
    REASON "Preserve mathematical formulas and technical terms"

  // Domain-specific entity types
  EXTRACT entities, relationships
    WITH entity_types=[
      method,        // Algorithms, techniques
      metric,        // Evaluation metrics
      dataset,       // Data sources
      finding,       // Research findings
      researcher,    // Authors, scientists
      organization   // Research institutions
    ]
    REASON "Scientific domain entities"

  // Include few-shot examples for scientific domain
  APPLY DomainAdaptation
    WITH examples=scientific_entity_examples

  AGGREGATE entities BY name

  BUILD graph FROM entities, relationships

  // Fine-grained clustering for research topics
  CLUSTER graph
    WITH max_cluster_size=8
    REASON "Detailed research topics"

  GENERATE reports
    WITH focus=methodology,
         include_metrics=true

  EMBED ALL

  INDEX INTO knowledge_graph
```

### Legal Documents Indexing

```gfl
FLOW LegalDocumentsIndexing:
  /*
    Specialized flow for legal documents
    Focus: Citations, precedents, statutes
  */

  INGEST documents
    FROM ./legal/*.pdf
    WITH parser=legal_pdf,
         preserve_structure=true

  // Preserve legal citations and references
  CHUNK documents
    WITH size=1000, overlap=200,
         boundary_aware=true
    REASON "High overlap preserves citations"

  EXTRACT entities, relationships
    WITH entity_types=[
      case,          // Legal cases
      statute,       // Laws and statutes
      precedent,     // Precedents
      party,         // Legal parties
      court,         // Courts
      judge          // Judges
    ]

  // Extract citations separately
  EXTRACT citations
    FROM text_units
    WITH pattern=legal_citation_regex
    YIELD citations

  AGGREGATE entities, citations

  // Don't cluster legal documents heavily
  // Each case should remain distinct
  CLUSTER graph
    WITH max_cluster_size=5,
         resolution=0.5
    REASON "Legal documents need separation"

  GENERATE reports
    WITH focus=precedents,
         include_citations=true,
         format=legal_brief

  EMBED ALL

  INDEX INTO legal_knowledge_graph
```

### Code Documentation Indexing

```gfl
FLOW CodeDocumentationIndexing:
  /*
    Specialized flow for code + documentation
    Focus: Functions, classes, APIs
  */

  INGEST code_files
    FROM ./src/**/*.{py,js,java}
    WITH parser=code_parser

  // Parse code structure
  EXTRACT code_elements
    FROM code_files
    WITH types=[function, class, method, module]
    YIELD code_elements

  // Chunk documentation separately
  CHUNK documentation
    FROM code_files.docstrings
    WITH size=800, overlap=100

  // Extract entities from documentation
  EXTRACT entities
    FROM documentation
    WITH entity_types=[
      api,           // API endpoints
      parameter,     // Function parameters
      return_type,   // Return types
      exception,     // Exceptions
      dependency     // Dependencies
    ]

  // Link code elements to entities
  LINK code_elements WITH entities
    ON name_match
    YIELD linked_graph

  BUILD graph FROM linked_graph

  CLUSTER graph
    BY module
    REASON "Group by code modules"

  GENERATE API_documentation
    FROM clusters
    WITH format=markdown,
         include_examples=true

  EMBED documentation, code_elements

  INDEX INTO code_knowledge_graph
```

## Incremental Indexing

### Update Existing Index

```gfl
FLOW IncrementalIndexing:
  /*
    Update existing knowledge graph with new documents
    Avoids re-indexing entire corpus
  */

  // Load existing index
  LOAD existing_graph FROM ./output/knowledge_graph

  // Ingest only new documents
  INGEST new_documents
    FROM ./input/new/*.txt
    WHERE created_date > existing_graph.last_update

  // Process new documents
  CHUNK new_documents INTO new_text_units
  EXTRACT entities, relationships FROM new_text_units

  // Merge with existing entities
  MERGE entities
    INTO existing_graph.entities
    WITH strategy=consolidate,
         conflict_resolution=latest_wins

  // Rebuild affected communities only
  IDENTIFY affected_communities
    WHERE contains_new_entities=true

  RECOMPUTE communities FOR affected_communities

  // Regenerate reports for affected communities
  FOR EACH community IN affected_communities:
    GENERATE report FROM community

  // Embed new content
  EMBED new_text_units, new_entities

  // Update index
  UPDATE vector_store WITH new_embeddings
  UPDATE graph_store WITH merged_graph

  SAVE updated_graph WITH timestamp=now()
```

### Delete and Reindex

```gfl
FLOW DeleteAndReindex:
  /*
    Remove documents and rebuild affected parts
  */

  LOAD existing_graph FROM ./output

  // Identify documents to remove
  IDENTIFY documents_to_remove
    WHERE source_id IN deleted_document_ids

  // Find all derived entities and relationships
  TRACE dependencies FROM documents_to_remove
    YIELD affected_entities, affected_relationships

  // Remove from graph
  DELETE affected_entities FROM existing_graph.entities
  DELETE affected_relationships FROM existing_graph.relationships

  // Rebuild affected communities
  IDENTIFY affected_communities
    WHERE overlaps_with(affected_entities)

  RECLUSTER affected_communities

  // Regenerate reports
  GENERATE reports FOR affected_communities

  // Update embeddings
  DELETE embeddings WHERE source IN documents_to_remove
  REBUILD vector_store

  SAVE updated_graph
```

## Parallel Processing

### Multi-Stage Pipeline with Parallelism

```gfl
FLOW ParallelIndexing:
  /*
    Pipeline with parallel processing stages
  */

  INGEST documents YIELD docs

  // Stage 1: Chunk (sequential)
  CHUNK docs INTO text_units

  // Stage 2: Extract (parallel per chunk)
  PARALLEL FOR EACH unit IN text_units:
    EXTRACT entities, relationships FROM unit
    WITH concurrency=25  // 25 concurrent LLM calls
    YIELD extracted_data

  COLLECT extracted_data INTO all_entities, all_relationships

  // Stage 3: Aggregate (sequential)
  AGGREGATE all_entities BY name INTO consolidated_entities

  // Stage 4: Community detection (sequential)
  BUILD graph FROM consolidated_entities, all_relationships
  CLUSTER graph INTO communities

  // Stage 5: Generate reports (parallel per community)
  PARALLEL FOR EACH community IN communities:
    GENERATE report FROM community
    WITH concurrency=10
    YIELD community_report

  // Stage 6: Embed (parallel batches)
  PARALLEL:
    EMBED text_units WITH batch_size=100
    EMBED consolidated_entities WITH batch_size=50
    EMBED communities WITH batch_size=20

  COLLECT embeddings

  // Stage 7: Index (sequential)
  INDEX ALL INTO knowledge_graph
```

## Error Handling and Retry

### Robust Indexing with Error Handling

```gfl
FLOW RobustIndexing:
  /*
    Indexing flow with comprehensive error handling
  */

  TRY:
    INGEST documents FROM source
  CATCH FileNotFoundError:
    LOG "Source directory not found"
    RETURN error

  CHUNK documents INTO text_units

  // Retry logic for LLM calls
  FOR EACH unit IN text_units:
    TRY:
      EXTRACT entities FROM unit
        WITH retry_strategy=exponential_backoff,
             max_retries=3,
             retry_delay=[2s, 4s, 8s]
    CATCH RateLimitError AS e:
      WAIT e.retry_after
      RETRY
    CATCH LLMError AS e:
      LOG "Failed to extract from unit {unit.id}: {e}"
      CONTINUE  // Skip this unit, continue with others

  // Validate extracted data
  VALIDATE entities:
    ENSURE NOT EMPTY
    ENSURE ALL entity.name NOT NULL
    ENSURE ALL entity.type IN EntityType

  IF validation_failed THEN:
    LOG "Validation failed"
    APPLY DataCleaning TO entities

  BUILD graph FROM entities, relationships

  TRY:
    CLUSTER graph INTO communities
  CATCH ClusteringError:
    LOG "Clustering failed, using fallback"
    APPLY FallbackClustering

  GENERATE reports FOR communities

  EMBED ALL

  // Atomic index update
  BEGIN TRANSACTION:
    INDEX embeddings INTO vector_store
    INDEX graph INTO graph_store
  COMMIT

  RETURN success
```

## Monitoring and Metrics

### Instrumented Indexing Flow

```gfl
FLOW InstrumentedIndexing:
  /*
    Flow with comprehensive monitoring
  */

  METRICS:
    document_count: Counter
    chunk_count: Counter
    entity_count: Counter
    extraction_latency: Histogram
    extraction_cost: Counter
    quality_scores: Gauge

  START_TIMER total_time

  INGEST documents
    COUNT INTO METRICS.document_count

  CHUNK documents INTO text_units
    COUNT INTO METRICS.chunk_count

  FOR EACH unit IN text_units:
    START_TIMER extraction_time

    EXTRACT entities FROM unit
      RECORD cost INTO METRICS.extraction_cost
      RECORD latency INTO METRICS.extraction_latency

    STOP_TIMER extraction_time

    COUNT entities INTO METRICS.entity_count

    // Quality checks
    EVALUATE extraction_quality
      YIELD precision, recall
    RECORD precision, recall INTO METRICS.quality_scores

  BUILD graph
  CLUSTER graph
  GENERATE reports
  EMBED ALL
  INDEX ALL

  STOP_TIMER total_time

  // Log final metrics
  LOG "Indexing completed"
  LOG "Documents: {METRICS.document_count}"
  LOG "Text units: {METRICS.chunk_count}"
  LOG "Entities: {METRICS.entity_count}"
  LOG "Total cost: ${METRICS.extraction_cost}"
  LOG "Total time: {total_time}"
  LOG "Avg quality: P={METRICS.quality_scores.precision_avg}, R={METRICS.quality_scores.recall_avg}"
```

---

**Next**: [Query Flows Examples](03-query-flows.md)
