# Complete End-to-End Examples

## Обзор

Этот документ содержит **полные сценарии end-to-end** от сырого текста до финального ответа пользователю, описанные на GFL.

## Scenario 1: Research Paper Analysis

### Full Pipeline

```gfl
SCENARIO ResearchPaperAnalysis:
  /*
    Complete flow: Ingest research papers → Index → Query → Answer
    Domain: Academic research
    User Question: "What are the main approaches to entity extraction?"
  */

  // ==================== INDEXING PHASE ====================

  FLOW IndexResearchPapers:
    // Input: 100 research papers on NLP
    INGEST papers
      FROM ./research_papers/*.pdf
      WITH parser=scientific_pdf,
           extract_sections=[abstract, introduction, methods, results]
      YIELD 100 documents

    // Chunking: Smaller chunks for technical precision
    CHUNK papers
      INTO text_units
      WITH size=800,  // Smaller for technical content
           overlap=150  // Higher overlap for formulas
      YIELD 8000 text_units

    // Entity Extraction: Domain-specific types
    EXTRACT entities, relationships
      FROM text_units
      USING llm=gpt-4
      WITH entity_types=[
        method,        // "BERT", "Named Entity Recognition"
        metric,        // "F1 score", "Accuracy"
        dataset,       // "CoNLL-2003", "OntoNotes"
        researcher,    // "Devlin et al.", "Manning"
        finding        // "Transformers improve NER"
      ],
      max_gleanings=2  // High recall for academic content
      YIELD 15000 entities, 25000 relationships

    // Example extracted entities:
    // Entity: "BERT" (METHOD)
    //   Description: "Bidirectional transformer model pre-trained on masked language modeling"
    // Entity: "CoNLL-2003" (DATASET)
    //   Description: "Named entity recognition dataset with news articles"
    // Relationship: BERT --[evaluated on]--> CoNLL-2003 (strength: 9)

    // Consolidation
    AGGREGATE entities BY name INTO groups
    SYNTHESIZE groups INTO consolidated_entities
      WITH preserve_technical_details=true
      YIELD 5000 consolidated_entities

    // Graph & Clustering
    BUILD graph FROM consolidated_entities, relationships
    CLUSTER graph INTO communities
      WITH max_cluster_size=8  // Fine-grained topics
      YIELD 150 communities

    // Example community: "Transformer-based NER Methods"
    //   Entities: [BERT, GPT, Transformer, NER, CRF, BiLSTM]
    //   Central theme: Modern neural architectures for NER

    // Community Reports
    FOR EACH community IN communities:
      GENERATE report
        WITH focus=methodology,
             include_metrics=true,
             format=academic_style
      YIELD community.report

    // Example report:
    // Title: "Transformer-based Named Entity Recognition"
    // Summary: "This community focuses on transformer architectures..."
    // Key Findings:
    //   1. BERT achieves 92.8% F1 on CoNLL-2003
    //   2. Pre-training on large corpora improves performance
    //   ...

    // Embedding & Indexing
    EMBED text_units, consolidated_entities, communities
    INDEX ALL INTO knowledge_graph

  // ==================== QUERY PHASE ====================

  FLOW AnswerResearchQuestion:
    /*
      User Question: "What are the main approaches to entity extraction?"
    */

    QUERY "What are the main approaches to entity extraction?"
      YIELD user_query

    // Query Analysis
    ANALYZE user_query
      EXTRACT intent => GLOBAL  // "main approaches" = overview
      EXTRACT concepts => ["entity extraction", "approaches", "methods"]
      CLASSIFY scope => BROAD
      YIELD query_metadata

    // Route to Global Search (overview question)
    ROUTE query_metadata.intent:
      WHEN GLOBAL THEN GlobalSearchFlow(user_query)

    // Global Search Execution
    FLOW GlobalSearchFlow(query):
      // Embed query
      EMBED query INTO query_vector
        YIELD [0.23, -0.15, 0.42, ...]

      // Retrieve relevant community reports
      RETRIEVE community_reports
        MATCHING query_vector
        WITH top_k=8
        YIELD [
          "Transformer-based NER" (similarity: 0.89),
          "Traditional Statistical Methods" (similarity: 0.85),
          "Neural Sequence Labeling" (similarity: 0.83),
          "Feature Engineering Approaches" (similarity: 0.81),
          ...
        ]

      // Map Phase: Analyze each community
      PARALLEL FOR EACH report IN community_reports:
        CALL llm=gpt-4 WITH prompt=GlobalMapPrompt
        CONTEXT report.content
        QUERY "What approaches to entity extraction are described?"

        // Example intermediate answer for "Transformer-based NER":
        YIELD {
          answer: "This community describes transformer-based approaches
                   including BERT, RoBERTa, and ELECTRA. These methods use
                   pre-trained language models fine-tuned for NER tasks.
                   Key advantage: contextual representations. Performance:
                   92-94% F1 on CoNLL-2003. [Data: Reports (15, 23, 42)]",
          score: 95
        }

      COLLECT intermediate_answers

      // Reduce Phase: Synthesize final answer
      AGGREGATE intermediate_answers
        RANKED BY score DESC
        FILTER WHERE score >= 70

      SYNTHESIZE final_answer
        FROM ranked_intermediate_answers
        USING llm=gpt-4
        WITH prompt=GlobalReducePrompt

      YIELD final_answer:
        """
        # Main Approaches to Entity Extraction

        Based on analysis of research communities, entity extraction
        approaches can be categorized into several main paradigms:

        ## 1. Transformer-based Methods (Modern State-of-the-Art)
        - BERT, RoBERTa, ELECTRA: Pre-trained language models fine-tuned
          for sequence labeling
        - Performance: 92-94% F1 on CoNLL-2003 [Data: Reports (15, 23, 42)]
        - Advantage: Contextual word representations
        - Limitation: Computational cost

        ## 2. Neural Sequence Labeling
        - BiLSTM-CRF: Bidirectional LSTM with conditional random fields
        - CNN-based models: Convolutional networks for character-level
          features [Data: Reports (8, 19)]
        - Performance: 88-91% F1
        - Advantage: End-to-end learning

        ## 3. Traditional Statistical Methods
        - CRF (Conditional Random Fields): Feature-based approaches
        - Hidden Markov Models: Probabilistic sequence models
          [Data: Reports (31, 44)]
        - Performance: 80-85% F1
        - Advantage: Interpretability, smaller models

        ## 4. Hybrid Approaches
        - Combining neural and rule-based methods
        - Ensemble models [Data: Reports (12, 27)]

        ## Recent Trends
        - Few-shot learning for low-resource scenarios
        - Cross-lingual entity extraction
        - Domain adaptation techniques

        The field has shifted from feature engineering to pre-trained
        language models, with transformer-based methods now dominating
        benchmarks.
        """

    RETURN final_answer

  // ==================== RESULT ====================
  /*
    Total time: 25 seconds
    Total cost: $0.41 (8 map calls + 1 reduce)
    Quality metrics:
      - Comprehensiveness: 0.92 (covered all major approaches)
      - Factual accuracy: 0.88 (grounded in reports)
      - Citation quality: 0.90 (proper data references)
  */
```

## Scenario 2: Product Documentation Search

### Full Pipeline

```gfl
SCENARIO ProductDocumentationSearch:
  /*
    Complete flow: Index product docs → Specific query → Detailed answer
    Domain: Software documentation
    User Question: "How do I configure rate limiting in the API?"
  */

  // ==================== INDEXING PHASE ====================

  FLOW IndexProductDocs:
    // Input: Product documentation (API docs, tutorials, guides)
    INGEST documents
      FROM ./docs/**/*.{md,rst,html}
      WITH parser=markdown,
           preserve_code_blocks=true
      YIELD 250 documents

    // Chunking: Preserve code blocks and sections
    CHUNK documents
      INTO text_units
      WITH size=1000,
           overlap=100,
           boundary_aware=true,  // Don't split code blocks
           respect_headers=true   // Keep sections intact
      YIELD 2500 text_units

    // Entity Extraction: Documentation-specific types
    EXTRACT entities, relationships
      WITH entity_types=[
        api_endpoint,     // "/api/v1/users"
        configuration,    // "rate_limit_config"
        parameter,        // "max_requests_per_minute"
        code_example,     // Code snippets
        error_code       // "429 Too Many Requests"
      ]
      YIELD 8000 entities, 12000 relationships

    // Example entities:
    // Entity: "rate_limit_config" (CONFIGURATION)
    //   Description: "Configuration object for API rate limiting"
    // Entity: "max_requests_per_minute" (PARAMETER)
    //   Description: "Maximum number of requests allowed per minute"

    AGGREGATE entities BY name
    SYNTHESIZE INTO consolidated_entities

    BUILD graph FROM entities, relationships
    CLUSTER graph INTO communities BY feature_area
      YIELD 50 communities  // e.g., "Authentication", "Rate Limiting", "Webhooks"

    GENERATE reports FOR communities
      WITH format=documentation_style,
           include_code_examples=true

    EMBED text_units, entities
    INDEX ALL

  // ==================== QUERY PHASE ====================

  FLOW AnswerSpecificQuestion:
    /*
      User Question: "How do I configure rate limiting in the API?"
    */

    QUERY "How do I configure rate limiting in the API?"
      YIELD user_query

    // Query Analysis
    ANALYZE user_query
      EXTRACT intent => LOCAL  // "How do I" = specific instructions
      EXTRACT concepts => ["configure", "rate limiting", "API"]
      DETECT action => CONFIGURATION
      YIELD query_metadata

    // Route to Local Search (specific question)
    ROUTE query_metadata.intent:
      WHEN LOCAL THEN LocalSearchFlow(user_query)

    // Local Search Execution
    FLOW LocalSearchFlow(query):
      // Embed query
      EMBED query INTO query_vector

      // Retrieve specific entities
      RETRIEVE entities
        MATCHING query_vector
        WITH top_k=20,
             filter_by_type=[configuration, api_endpoint, parameter]
        YIELD relevant_entities:
          [
            "rate_limit_config" (score: 0.92),
            "max_requests_per_minute" (score: 0.88),
            "/api/v1/rate-limit" (score: 0.85),
            "rate_limiter" (score: 0.83),
            ...
          ]

      // Retrieve documentation text units
      RETRIEVE text_units
        MATCHING query_vector
        WITH top_k=15,
             prefer_code_blocks=true  // Prioritize units with code
        YIELD relevant_text_units

      // Expand with relationships
      EXPAND relevant_entities
        WITH relationships
        YIELD expanded_context

      // Build structured context
      BUILD context
        FROM relevant_entities, relevant_text_units, expanded_context
        WITH format=structured_tables,
             max_tokens=8000,
             prioritize=code_examples
        YIELD structured_context:
          """
          Entities:
          | Name | Type | Description |
          |------|------|-------------|
          | rate_limit_config | CONFIGURATION | Configuration for rate limiting |
          | max_requests_per_minute | PARAMETER | Max requests per minute |

          Relationships:
          | Source | Relationship | Target |
          |--------|--------------|--------|
          | rate_limit_config | has_parameter | max_requests_per_minute |
          | /api/v1/rate-limit | configured_by | rate_limit_config |

          Sources (Text Units):
          [1] "To configure rate limiting, add the rate_limit_config...
               ```python
               rate_limit_config = {
                   'max_requests_per_minute': 100,
                   'burst_size': 20
               }
               ```"

          [2] "The rate limiter can be customized per endpoint..."
          ...
          """

      // Generate answer
      SYNTHESIZE answer
        FROM structured_context
        USING llm=gpt-4
        WITH prompt=LocalSearchPrompt,
             include_code_examples=true,
             include_citations=true

      YIELD answer:
        """
        # Configuring API Rate Limiting

        To configure rate limiting in the API, you need to set up the
        `rate_limit_config` configuration object [Data: Entities (5)].

        ## Configuration Parameters

        The main parameters are:

        - **max_requests_per_minute**: Maximum number of requests allowed
          per minute (default: 100) [Data: Entities (7)]
        - **burst_size**: Number of requests allowed in a burst
          (default: 20) [Data: Entities (12)]

        ## Example Configuration

        ```python
        from api import RateLimiter

        # Configure rate limiter
        rate_limit_config = {
            'max_requests_per_minute': 100,
            'burst_size': 20,
            'enabled': True
        }

        # Apply to API
        limiter = RateLimiter(rate_limit_config)
        app.add_middleware(limiter)
        ```

        [Data: Sources (15, 23)]

        ## Endpoint-Specific Configuration

        You can also configure rate limiting per endpoint:

        ```python
        @app.route('/api/v1/data')
        @rate_limit(max_requests=50, window='1m')
        def get_data():
            ...
        ```

        [Data: Sources (42)]

        ## Testing Rate Limits

        To test your configuration, use the `/api/v1/rate-limit/status`
        endpoint [Data: Entities (18)].

        For more details, see the Rate Limiting documentation.
        """

    RETURN answer

  // ==================== RESULT ====================
  /*
    Total time: 4 seconds
    Total cost: $0.09 (1 LLM call)
    Quality metrics:
      - Correctness: 0.92 (accurate configuration)
      - Code accuracy: 0.95 (working code examples)
      - Citation accuracy: 0.93 (proper references)
  */
```

## Scenario 3: Exploratory Research

### Full Pipeline

```gfl
SCENARIO ExploratoryResearch:
  /*
    Complete flow: Index knowledge base → Drift search → Discovery
    Domain: Technology research
    User Question: "How are knowledge graphs related to LLMs?"
  */

  // ==================== INDEXING PHASE ====================

  FLOW IndexTechKnowledgeBase:
    // Input: Technology articles, blog posts, papers
    INGEST documents
      FROM ./tech_content/**/*.txt
      YIELD 500 documents

    CHUNK documents WITH size=1200, overlap=100
      YIELD 5000 text_units

    // Extract tech entities
    EXTRACT entities, relationships
      WITH entity_types=[
        technology,      // "Knowledge Graph", "LLM"
        organization,    // "Google", "OpenAI"
        method,          // "Graph Neural Networks"
        application     // "Question Answering"
      ]
      YIELD 12000 entities, 18000 relationships

    // Build rich relationship graph
    AGGREGATE entities
    BUILD entity_graph FROM entities, relationships
      WITH preserve_relationship_strength=true

    CLUSTER graph INTO communities
    GENERATE reports FOR communities

    EMBED ALL
    INDEX INTO knowledge_graph

  // ==================== QUERY PHASE ====================

  FLOW ExploreConnections:
    /*
      User Question: "How are knowledge graphs related to LLMs?"
    */

    QUERY "How are knowledge graphs related to LLMs?"
      YIELD user_query

    // Query Analysis
    ANALYZE user_query
      EXTRACT intent => DRIFT  // "related" = explore connections
      EXTRACT start_concepts => ["knowledge graphs"]
      EXTRACT end_concepts => ["LLMs"]
      YIELD query_metadata

    // Route to Drift Search
    ROUTE query_metadata.intent:
      WHEN DRIFT THEN DriftSearchFlow(user_query)

    // Drift Search Execution
    FLOW DriftSearchFlow(query):
      // Phase 1: Primer (get initial understanding)
      EMBED query INTO query_vector

      RETRIEVE community_reports
        MATCHING query_vector
        WITH top_k=3
        YIELD seed_reports:
          [
            "Knowledge Graph Technologies",
            "Large Language Models",
            "Graph-Enhanced NLP"
          ]

      CALL llm=gpt-4 WITH prompt=DriftPrimerPrompt
      YIELD initial_answer:
        """
        Knowledge graphs and LLMs intersect in several ways:
        1. LLMs can be used to construct knowledge graphs (entity extraction)
        2. Knowledge graphs can enhance LLM capabilities (grounding, factuality)
        3. Hybrid approaches combine both (GraphRAG, KG-augmented generation)
        """

      GENERATE follow_up_questions:
        [
          "How do LLMs extract entities for knowledge graphs?",
          "How do knowledge graphs improve LLM factuality?",
          "What are examples of graph-enhanced language models?",
          "How does GraphRAG work?",
          "What are the performance comparisons?"
        ]

      // Phase 2: Hop 1 - Explore entity extraction
      SELECT follow_up = "How do LLMs extract entities for knowledge graphs?"

      EMBED follow_up INTO hop1_vector

      RETRIEVE entities, text_units
        MATCHING hop1_vector
        WITH top_k_entities=15, top_k_text_units=10
        YIELD hop1_context

      SYNTHESIZE hop1_answer
        FROM hop1_context
        WITH prompt=DriftLocalPrompt
        YIELD hop1_answer:
          """
          LLMs like GPT-4 extract entities through:
          1. **Prompting**: Structured prompts guide entity identification
          2. **Few-shot learning**: Examples improve accuracy
          3. **Iterative refinement**: Gleaning passes catch missed entities

          GraphRAG uses this approach, achieving 92% precision, 85% recall
          [Data: Sources (15, 23, 42)]
          """

      GENERATE new_follow_ups:
        [
          "What is gleaning in entity extraction?",
          "How does GraphRAG compare to traditional methods?",
          ...
        ]

      // Phase 3: Hop 2 - Explore graph enhancement of LLMs
      SELECT follow_up = "How do knowledge graphs improve LLM factuality?"

      RETRIEVE entities, text_units FOR follow_up
      SYNTHESIZE hop2_answer:
        """
        Knowledge graphs improve LLM factuality through:
        1. **Grounding**: Graph facts verify LLM outputs
        2. **Retrieval augmentation**: Graph provides factual context
        3. **Structured reasoning**: Graph constraints guide generation

        Studies show 30-50% reduction in hallucinations when using
        graph-augmented generation [Data: Sources (67, 89)]
        """

      // Phase 4: Hop 3 - Specific implementations
      SELECT follow_up = "How does GraphRAG work?"

      RETRIEVE entities, text_units FOR follow_up
      SYNTHESIZE hop3_answer:
        """
        GraphRAG combines knowledge graphs and LLMs:

        **Indexing Phase**:
        1. LLMs extract entities/relationships from text
        2. Graph clustering creates thematic communities
        3. LLMs generate community summaries

        **Query Phase**:
        1. Query routes to global (communities) or local (entities) search
        2. Retrieved context feeds to LLM
        3. LLM generates grounded answer with citations

        This enables both broad (global) and specific (local) questions
        [Data: Sources (101, 112, 125)]
        """

      // Phase 5: Synthesis
      SYNTHESIZE final_answer
        FROM [initial_answer, hop1_answer, hop2_answer, hop3_answer]
        WITH prompt=DriftReducePrompt

      YIELD comprehensive_answer:
        """
        # Knowledge Graphs and LLMs: A Symbiotic Relationship

        Knowledge graphs and large language models have a bidirectional
        relationship where each enhances the other.

        ## LLMs Building Knowledge Graphs

        LLMs excel at extracting structured information from unstructured
        text. Through techniques like:
        - **Prompted entity extraction**: GPT-4 can identify entities with
          92% precision [Data: Sources (15, 23)]
        - **Relationship mining**: Detecting semantic connections
        - **Iterative refinement**: Gleaning passes improve recall to 85%

        GraphRAG exemplifies this approach, using LLMs to construct
        multi-level knowledge graphs from documents [Data: Sources (101)].

        ## Knowledge Graphs Enhancing LLMs

        Conversely, knowledge graphs address key LLM limitations:
        - **Factual grounding**: Graph facts verify and constrain generation,
          reducing hallucinations by 30-50% [Data: Sources (67, 89)]
        - **Structured reasoning**: Graph topology guides logical inference
        - **Explainability**: Graph paths provide interpretable reasoning

        ## Hybrid Approaches

        Modern systems like GraphRAG, KG-BERT, and KAPING combine both:

        1. **Indexing**: LLMs extract graph from text
        2. **Querying**: Graph retrieves relevant context
        3. **Generation**: LLM synthesizes answer from graph context

        This enables:
        - **Global search**: Community-level overviews
        - **Local search**: Entity-specific details
        - **Drift search**: Connection exploration

        ## Performance Benefits

        Compared to pure LLM approaches:
        - 40% better factual accuracy [Data: Sources (125)]
        - 60% reduction in irrelevant information
        - Full source attribution and explainability

        ## Future Directions

        - **Graph neural networks + LLMs**: Combining representational strengths
        - **Dynamic knowledge graphs**: LLMs maintaining living knowledge bases
        - **Multi-modal graphs**: Integrating text, images, code

        The synergy between knowledge graphs and LLMs represents a powerful
        paradigm for knowledge-intensive NLP tasks.
        """

      GENERATE exploration_follow_ups:
        [
          "What are specific benchmarks for graph-augmented LLMs?",
          "How do different graph types (RDF vs property graphs) work with LLMs?",
          "What are the computational costs of graph-augmented generation?",
          "How does graph quality affect LLM performance?",
          "What are open-source implementations of graph-augmented LLMs?"
        ]

    RETURN DriftResult {
      answer: comprehensive_answer,
      hops_explored: 3,
      follow_ups: exploration_follow_ups,
      discovered_connections: [
        "LLM → Entity Extraction → Knowledge Graph",
        "Knowledge Graph → Factual Grounding → LLM Enhancement",
        "GraphRAG → Hybrid System"
      ]
    }

  // ==================== RESULT ====================
  /*
    Total time: 42 seconds
    Total cost: $0.28 (1 primer + 3 hops + 1 reduce)
    Discovery metrics:
      - Connections found: 8 major pathways
      - Non-obvious insights: 3 (gleaning, hallucination reduction, hybrid architectures)
      - Follow-up quality: 0.88 (relevant exploration paths)
  */
```

## Comparison Matrix

| Scenario | Query Type | Search Strategy | Entities | Communities | Hops | Latency | Cost | Use Case |
|---|---|---|---|---|---|---|---|---|
| Research Papers | Overview | Global | 5K | 150 | 1 (map-reduce) | 25s | $0.41 | Academic research |
| Product Docs | Specific | Local | 8K | 50 | 1 | 4s | $0.09 | Technical support |
| Exploratory | Connection | Drift | 12K | 80 | 3 | 42s | $0.28 | Discovery research |

## Key Insights

1. **Query Type Determines Strategy**:
   - Overview questions → Global search (communities)
   - Specific questions → Local search (entities + text units)
   - Connection questions → Drift search (graph walk)

2. **Indexing Adapts to Domain**:
   - Research: Smaller chunks, domain-specific entities
   - Docs: Preserve code blocks, API-specific types
   - General: Standard configuration

3. **Cost-Latency Trade-offs**:
   - Global: Higher cost ($0.41), moderate latency (25s), breadth
   - Local: Lower cost ($0.09), fast (4s), depth
   - Drift: Medium cost ($0.28), slow (42s), discovery

4. **Quality Through Grounding**:
   - All answers cite sources [Data: ...]
   - Graph structure ensures factual accuracy
   - Multi-hop exploration discovers non-obvious connections

---

**Язык GFL позволяет описывать полные информационные потоки от сырых данных до финальных ответов, сохраняя читаемость и точность.**
