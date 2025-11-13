# Semantic Scenarios

## Обзор

Этот документ содержит **полные end-to-end семантические сценарии**, демонстрирующие применение SFL для реальных use cases. Каждый сценарий включает:
- Полную семантическую цепочку от индексации до query execution
- Детальное отслеживание семантических метрик
- Реальные примеры трансформаций

---

## Scenario 1: Research Paper Analysis

### Use Case Description

**Задача**: Индексировать 100 научных статей по machine learning и ответить на вопрос "What are the main approaches to entity extraction in NLP?"

**Corpus**: 100 research papers (2.5M tokens total)

### Semantic Indexing Flow

```sfl
SEMANTIC SCENARIO ResearchPaperIndexing:
  /*
    Индексация научных статей с сохранением технической семантики
  */

  // ==================== INPUT ====================
  INPUT:
    documents: 100 research papers
    total_tokens: 2.5M tokens
    domain: machine learning / NLP
    technical_level: high

  SEMANTIC_STATE_0:
    representation: Raw text (PDF)
    semantic_value: 1.0 (baseline)
    queryability: 0.0 (not indexed)

  // ==================== PHASE 1: DECOMPOSE ====================
  DECOMPOSE papers
    INTO semantic_units
    STRATEGY: sliding_window
      WITH size=800_tokens, overlap=150_tokens
      REASON: "Smaller chunks for technical precision"

    PRESERVING:
      technical_context: 0.96          // Formulas, terms preserved
      citation_context: 0.92           // Citations in context
      methodological_details: 0.94     // Method details kept

    LOSING:
      paper_structure: 0.25            // Sections lost
      cross_section_context: 0.15      // Inter-section refs

    METRICS:
      input: 100 papers (2.5M tokens)
      output: 3400 semantic_units (800 tokens each)
      preservation: 0.96
      latency: 1.2s

  SEMANTIC_STATE_1:
    semantic_value: 1.0 * 0.96 = 0.96
    representation: Text units (technical)

  SEMANTIC_TRACE:
    "...BERT uses bidirectional Transformers for language understanding.
     The model achieves 92.8 F1 on SQuAD 1.1..." [text_unit_1847]

    Context continuity with overlap:
      Previous unit: "...Transformers architecture introduced in 2017..."
      Current unit: "...BERT uses bidirectional Transformers..."
      Next unit: "...achieving state-of-the-art results across..."

  // ==================== PHASE 2: ABSTRACT ====================
  ABSTRACT semantic_units
    INTO [entities, relationships]
    STRATEGY: llm_guided_extraction
      USING llm=gpt-4
      WITH entity_types=[
        method,        // Algorithms: "BERT", "BiLSTM-CRF"
        metric,        // Metrics: "F1 score", "Accuracy"
        dataset,       // Datasets: "SQuAD", "CoNLL-2003"
        finding,       // Results: "92.8% F1", "State-of-the-art"
        researcher,    // Authors
        organization   // Research labs
      ]

    PRESERVING:
      technical_entities: 0.88         // High precision on methods
      methodological_semantics: 0.82   // Method semantics
      quantitative_facts: 0.92         // Numbers, metrics preserved
      precision: 0.92                  // Accurate extraction
      recall: 0.88                     // Most entities found

    LOSING:
      prose_style: 0.95                // Academic writing style
      detailed_explanations: 0.70      // Detailed prose
      mathematical_notation: 0.60      // Formulas → text

    GAINING:
      structured_methods: 0.85         // Methods as entities
      queryable_metrics: 0.90          // Can query by metric
      comparable_results: 0.88         // Can compare results

    METRICS:
      input: 3400 semantic_units
      output: 15000 entities, 28000 relationships
      precision: 0.92
      recall: 0.88
      f1: 0.90
      latency: 8500s (2.5s per unit)
      cost: $240 (GPT-4)

  SEMANTIC_STATE_2:
    semantic_value: 0.96 * 0.88 = 0.85
    representation: Concepts (methods, metrics, findings)

  SEMANTIC_TRACE:
    Input text:
      "BERT uses bidirectional Transformers achieving 92.8 F1 on SQuAD 1.1"

    Extracted concepts:
      Entity("BERT", METHOD, "Bidirectional Encoder Representations
             from Transformers for language understanding")
      Entity("Transformers", METHOD, "Attention-based neural architecture")
      Entity("SQuAD 1.1", DATASET, "Stanford Question Answering Dataset v1.1")
      Entity("F1 score", METRIC, "Harmonic mean of precision and recall")
      Entity("92.8", FINDING, "F1 score achieved by BERT on SQuAD 1.1")

      Relationship(BERT, Transformers, "uses", strength=9)
      Relationship(BERT, SQuAD 1.1, "evaluated_on", strength=8)
      Relationship(BERT, 92.8, "achieves_score", strength=10)

  // ==================== PHASE 3: CONSOLIDATE ====================
  CONSOLIDATE entities
    BY semantic_equivalence
      EXAMPLES:
        "BERT" == "Bidirectional Encoder Representations" → merge
        "F1 score" == "F1-score" == "F1" → merge
        "SQuAD" == "SQuAD 1.1" → keep separate (different versions)

    PRESERVING:
      technical_distinctiveness: 0.90  // Important to distinguish versions
      quantitative_precision: 0.95     // Numbers must be exact

    METRICS:
      input: 15000 raw_entities
      output: 4200 consolidated_entities
      consolidation_ratio: 3.6:1
      preservation: 0.90
      cost: $15

  SEMANTIC_STATE_3:
    semantic_value: 0.85 * 0.90 = 0.77
    representation: Consolidated concepts

  SEMANTIC_TRACE:
    Before consolidation:
      Entity("BERT", METHOD, "...") [from paper_1.pdf, text_unit_42]
      Entity("BERT model", METHOD, "...") [from paper_3.pdf, text_unit_89]
      Entity("Bidirectional Encoder Representations", METHOD, "...")
            [from paper_7.pdf, text_unit_203]

    After consolidation:
      Entity("BERT", METHOD, "Bidirectional Encoder Representations from
             Transformers. Pretrained bidirectional language model using
             masked language modeling. Introduced by Devlin et al. 2018.
             Achieves state-of-the-art on multiple NLP tasks.")
      [consolidated from 47 mentions across 23 papers]

  // ==================== PHASE 4: ORGANIZE ====================
  ORGANIZE entity_graph
    INTO communities
    STRATEGY: leiden_clustering
      WITH max_cluster_size=8
      REASON: "Fine-grained research topics"

    DISCOVERING:
      research_topics: 180 communities
      topics_examples: [
        "Transformer-based Language Models" (142 entities)
        "Named Entity Recognition Methods" (89 entities)
        "Question Answering Datasets" (67 entities)
        "Evaluation Metrics" (45 entities)
      ]
      hierarchical_structure: 3 levels
      modularity: 0.85

    PRESERVING:
      methodological_relationships: 0.92
      topic_coherence: 0.88

    METRICS:
      input: 4200 entities, 20000 relationships
      output: 180 communities
      avg_community_size: 23 entities
      modularity: 0.85

  SEMANTIC_STATE_4:
    semantic_value: 0.77 * 0.92 = 0.71
    representation: Organized by topics

  SEMANTIC_TRACE:
    Community #42: "Entity Extraction Methods"
      Entities:
        - BiLSTM-CRF (method)
        - BERT for NER (method)
        - Conditional Random Fields (method)
        - CoNLL-2003 (dataset)
        - F1 score (metric)
        - IOB tagging (method)
        ... (89 entities total)

      Relationships: 450 internal edges
      Theme: Methods and evaluation for entity extraction
      Coherence: 0.87

  // ==================== PHASE 5: ABSTRACT COMMUNITIES ====================
  ABSTRACT communities
    INTO community_summaries
    STRATEGY: llm_guided_summarization
      WITH focus=methodology
      WITH include_metrics=true

    PRESERVING:
      methodological_overview: 0.85    // Methods overview kept
      key_findings: 0.82               // Main findings
      quantitative_results: 0.88       // Numbers preserved

    METRICS:
      input: 180 communities
      output: 180 summaries (~2000 tokens each)
      latency: 1800s (10s per community)
      cost: $36

  SEMANTIC_STATE_5:
    semantic_value: 0.71 * 0.85 = 0.60
    representation: Multi-level (concepts + summaries)

  SEMANTIC_TRACE:
    Community #42 Summary:
      "Entity Extraction Methods

      This community covers approaches to named entity recognition (NER)
      in NLP. Main methods include:

      1. Neural Sequence Labeling:
         - BiLSTM-CRF: 88-91% F1 on CoNLL-2003
         - BERT-based models: 92-94% F1 on CoNLL-2003

      2. Traditional Methods:
         - CRF: 80-85% F1
         - Rule-based: 70-75% F1

      Key datasets: CoNLL-2003, OntoNotes, ACE
      Evaluation: F1 score, precision, recall

      Recent trends: Transformer-based models achieving SOTA"

  // ==================== PHASE 6: ENCODE ====================
  ENCODE [semantic_units, entities, community_summaries]
    INTO embedding_space

    METRICS:
      input: 3400 + 4200 + 180 = 7780 objects
      output: 7780 vectors (1536-dim)
      preservation: 0.91
      cost: $2.40

  SEMANTIC_STATE_6 (INDEXED):
    semantic_value: 0.60 * 0.91 = 0.55
    representation: Hybrid (symbolic + geometric)

  // ==================== INDEXING SUMMARY ====================
  INDEXING_RESULT:
    Total semantic preservation: 0.55 (55%)

    BUT GAINED:
    + Technical structure: 0.85 (methods, metrics organized)
    + Queryability: 0.90 (can query by method/metric)
    + Comparative analysis: 0.88 (can compare results)
    + Topic navigation: 0.85 (browse by research topic)

    Total gained: 3.48
    Net semantic value: 0.55 + 3.48 = 4.03x

    Metrics:
      Total time: ~2.5 hours
      Total cost: $293.40
      Entities: 4200
      Communities: 180
      Queryability: Excellent
```

### Semantic Query Flow

```sfl
SEMANTIC SCENARIO ResearchPaperQuery:
  /*
    Query: "What are the main approaches to entity extraction in NLP?"
  */

  // ==================== PHASE 1: INTERPRET ====================
  INTERPRET "What are the main approaches to entity extraction in NLP?"

    EXTRACTING:
      query_type: GLOBAL              // "main approaches" = overview
      semantic_scope: BROAD           // Multiple methods
      key_concepts: [entity extraction, NLP, approaches]
      expected_answer: SUMMARY        // Overview of methods

    PRESERVING:
      information_need: 0.95          // Clear need
      key_concepts: 0.92              // Concepts identified

  SEMANTIC_TRACE:
    Query text: "What are the main approaches to entity extraction in NLP?"

    Parsed intent:
      Type: GLOBAL (keywords: "main", "approaches" → overview)
      Domain: NLP + entity extraction
      Expected: List of methods with descriptions
      Scope: Broad (multiple approaches)

  // ==================== PHASE 2: ROUTE ====================
  ROUTE TO GlobalSearchSemantics
    REASON: "Overview question requires broad coverage"

  // ==================== PHASE 3: ENCODE ====================
  ENCODE query_intent INTO query_vector

    METRICS:
      preservation: 0.92

  // ==================== PHASE 4: NAVIGATE ====================
  NAVIGATE embedding_space
    TOWARD query_vector
    STRATEGY: community_summary_retrieval

    LOCATING:
      community_summaries: [
        Community #42: "Entity Extraction Methods" (relevance: 0.95)
        Community #67: "Neural Sequence Labeling" (relevance: 0.88)
        Community #103: "CRF-based Methods" (relevance: 0.82)
        Community #128: "Evaluation Metrics for NER" (relevance: 0.78)
        Community #145: "Entity Recognition Datasets" (relevance: 0.75)
        ... (8 communities total)
      ]

    METRICS:
      retrieved: 8 communities
      avg_relevance: 0.84
      coverage: ~600 entities (indirect)

  SEMANTIC_TRACE:
    Query vector: [0.123, -0.456, 0.789, ..., 0.234] (1536-dim)

    Nearest communities in embedding space:
      1. "Entity Extraction Methods" (cosine=0.95)
      2. "Neural Sequence Labeling" (cosine=0.88)
      3. ...

  // ==================== PHASE 5: MAP PHASE ====================
  FOR EACH community IN [top_8_communities]:
    COMPOSE intermediate_answer
      FROM community.summary
      USING llm=gpt-4
      WITH prompt=GlobalMapPrompt

  SEMANTIC_TRACE:
    Community #42 intermediate answer:
      "Entity extraction in NLP uses several approaches:

       1. Transformer-based: BERT-based models achieve 92-94% F1 on
          CoNLL-2003, representing current state-of-the-art.

       2. Neural Sequence Labeling: BiLSTM-CRF achieves 88-91% F1,
          effective for sequence tagging tasks.

       3. Traditional: CRF models achieve 80-85% F1, computationally
          efficient but lower accuracy.

       [Relevance: 95/100]"

    Community #67 intermediate answer:
      "Neural sequence labeling for entity extraction includes:

       1. BiLSTM-CRF: Captures bidirectional context, 88-91% F1
       2. LSTM-CRF: Unidirectional, 85-88% F1
       3. Advantage: Learns patterns from data, no manual rules

       [Relevance: 88/100]"

  METRICS:
    input: 8 communities
    output: 8 intermediate_answers (~250 tokens each)
    latency: 18s (8 * 2.25s)
    cost: $0.32

  // ==================== PHASE 6: REDUCE PHASE ====================
  COMPOSE final_answer
    FROM intermediate_answers
    USING llm=gpt-4
    WITH prompt=GlobalReducePrompt

    INTEGRATING:
      perspectives: 8 communities
      ranked_by: relevance_score

    PRESERVING:
      comprehensiveness: 0.88         // Covers main approaches
      factual_accuracy: 0.85          // Facts from sources
      coherence: 0.92                 // Well-structured

  SEMANTIC_TRACE:
    Final answer:
      "# Main Approaches to Entity Extraction in NLP

      Entity extraction in NLP has evolved through several approaches:

      ## 1. Transformer-based Methods (Current SOTA)
      - **BERT-based models**: Achieve 92-94% F1 on CoNLL-2003
      - **Key advantage**: Bidirectional context, pretrained representations
      - **Examples**: BERT for NER, RoBERTa-NER
      [Data: Communities (42, 67)]

      ## 2. Neural Sequence Labeling
      - **BiLSTM-CRF**: 88-91% F1 on CoNLL-2003
      - **LSTM-CRF**: 85-88% F1
      - **Key advantage**: Learns patterns from data, no manual features
      [Data: Communities (67, 103)]

      ## 3. Traditional Statistical Methods
      - **Conditional Random Fields (CRF)**: 80-85% F1
      - **Key advantage**: Computationally efficient, interpretable
      - **Limitation**: Requires feature engineering
      [Data: Communities (103, 128)]

      ## 4. Rule-based and Hybrid
      - **Rule-based**: 70-75% F1, high precision in specific domains
      - **Hybrid**: Combines neural + rules for domain adaptation
      [Data: Communities (145)]

      The field has shifted from traditional statistical methods to
      neural approaches, with transformer-based models currently
      achieving state-of-the-art results. Evaluation is standardized
      using F1 score on benchmarks like CoNLL-2003 and OntoNotes."

  METRICS:
    input: 8 intermediate_answers
    output: final_answer (~650 tokens)
    latency: 4s
    cost: $0.06

  // ==================== QUERY RESULT ====================
  QUERY_RESULT:
    Total latency: 25s (navigation + map + reduce)
    Total cost: $0.38

    Semantic quality:
      Comprehensiveness: 0.88 (covers main approaches)
      Factual accuracy: 0.85 (grounded in papers)
      Coherence: 0.92 (well-structured)
      Citation coverage: 90% (most claims cited)

    Intent preservation: 0.58
    Enrichment: +2.80 (comprehensive multi-source answer)
    Net answer quality: 3.38x query intent
```

---

## Scenario 2: Product Documentation Search

### Use Case Description

**Задача**: Индексировать 250 страниц продуктовой документации и ответить на вопрос "How do I configure authentication in the API?"

**Corpus**: 250 markdown documents (800K tokens)

### Semantic Indexing Flow

```sfl
SEMANTIC SCENARIO ProductDocsIndexing:
  /*
    Индексация продуктовой документации для точных ответов
  */

  INPUT:
    documents: 250 markdown files
    total_tokens: 800K
    domain: API documentation
    structure: Hierarchical (guides, references, tutorials)

  // ==================== OPTIMIZED FOR PRECISION ====================
  DECOMPOSE documents
    WITH size=1000, overlap=150
    PRESERVING:
      code_examples: 0.98            // Critical: code must be intact
      step_sequences: 0.95           // Steps must be in order

  ABSTRACT semantic_units
    WITH entity_types=[
      api_endpoint,    // "/auth/login"
      parameter,       // "api_key", "timeout"
      configuration,   // "auth_config"
      code_example,    // Code snippets
      prerequisite     // Requirements
    ]
    PRESERVING:
      technical_accuracy: 0.95       // Must be exact
      procedural_steps: 0.92         // Steps preserved

  // Fewer consolidation (preserve exact details)
  CONSOLIDATE entities
    WITH preserve_variants=true
    PRESERVING:
      technical_precision: 0.92

  ORGANIZE entity_graph
    WITH max_cluster_size=12
    DISCOVERING:
      documentation_topics: [
        "Authentication & Authorization" (85 entities)
        "API Endpoints" (120 entities)
        "Configuration" (95 entities)
        "Error Handling" (60 entities)
      ]

  ENCODE ALL

  INDEXING_RESULT:
    Preservation: 0.62 (higher than average due to precision focus)
    Queryability: 0.95 (excellent for specific questions)
    Time: 45 minutes
    Cost: $78
```

### Semantic Query Flow

```sfl
SEMANTIC SCENARIO ProductDocsQuery:
  /*
    Query: "How do I configure authentication in the API?"
  */

  INTERPRET query
    EXTRACTING:
      query_type: LOCAL              // "How do I" = specific procedure
      semantic_scope: NARROW         // Specific topic
      key_concepts: [configure, authentication, API]
      expected_answer: PROCEDURE     // Step-by-step guide

  ROUTE TO LocalSearchSemantics
    REASON: "Specific procedural question"

  NAVIGATE + LOCATE:
    specific_entities: [
      Entity("API Authentication", CONFIGURATION, ...)
      Entity("auth_config", PARAMETER, ...)
      Entity("/auth/login", API_ENDPOINT, ...)
    ]
    text_units: [
      "To configure authentication, edit config.yaml..."
      "Set auth_provider to 'oauth' or 'apikey'..."
      "Example: auth_config: { provider: 'oauth', ... }"
    ]
    code_examples: 3 relevant snippets

  COMPOSE answer
    FROM [entities, text_units, code_examples]
    PRESERVING:
      procedural_accuracy: 0.92      // Steps correct
      code_correctness: 0.98         // Code works
      completeness: 0.88             // All steps included

  SEMANTIC_TRACE:
    Final answer:
      "# Configuring API Authentication

      Follow these steps to configure authentication:

      ## 1. Edit Configuration File
      Open `config.yaml` and add the auth section:
      ```yaml
      auth_config:
        provider: 'oauth'  # or 'apikey'
        token_expiry: 3600
      ```
      [Data: Textual (unit_4523)]

      ## 2. Set Environment Variables
      ```bash
      export API_KEY='your-key-here'
      ```
      [Data: Textual (unit_4524)]

      ## 3. Initialize Auth Client
      ```python
      from api import AuthClient
      client = AuthClient(config='config.yaml')
      ```
      [Data: Textual (unit_4525)]

      ## 4. Test Authentication
      ```python
      response = client.authenticate()
      if response.success:
          print('Authenticated!')
      ```
      [Data: Entities (API Authentication, auth_config)]"

  QUERY_RESULT:
    Latency: 4s
    Cost: $0.09

    Quality:
      Factual accuracy: 0.92
      Procedural completeness: 0.88
      Code correctness: 0.98
      Actionability: 0.95 (user can follow steps)

    Net answer quality: 4.2x query intent
```

---

## Scenario 3: Exploratory Research

### Use Case Description

**Задача**: Индексировать 500 документов по distributed systems и найти связи между "consensus algorithms" и "blockchain"

**Corpus**: 500 technical documents (1.8M tokens)

### Semantic Indexing Flow

```sfl
SEMANTIC SCENARIO ExploratoryResearchIndexing:
  /*
    Индексация для exploratory queries
  */

  INPUT:
    documents: 500 technical docs
    total_tokens: 1.8M
    domain: Distributed systems
    focus: Relationships and connections

  DECOMPOSE documents
    WITH size=1200, overlap=100
    // Standard chunking

  ABSTRACT semantic_units
    WITH entity_types=[
      algorithm,       // "Paxos", "Raft"
      concept,         // "Consensus", "Byzantine fault"
      technology,      // "Blockchain", "Distributed ledger"
      property,        // "Safety", "Liveness"
      application      // "Cryptocurrency", "Database"
    ]
    WITH focus=relationships
    PRESERVING:
      conceptual_relationships: 0.82   // Focus on connections

  CONSOLIDATE entities
    // Standard

  ORGANIZE entity_graph
    WITH preserve_cross_cluster_edges=true
    REASON: "Exploratory queries need connections"
    DISCOVERING:
      research_topics: 220 communities
      cross_community_connections: 3200 edges (important!)

  ENCODE ALL

  INDEXING_RESULT:
    Preservation: 0.58
    Graph connectivity: 0.92 (high - good for drift)
    Cross-community edges: Well preserved
    Time: 3.5 hours
    Cost: $425
```

### Semantic Query Flow

```sfl
SEMANTIC SCENARIO ExploratoryQuery:
  /*
    Query: "How are consensus algorithms related to blockchain?"
  */

  INTERPRET query
    EXTRACTING:
      query_type: DRIFT              // "How are X related to Y"
      semantic_scope: EXPLORATORY    // Connection exploration
      key_concepts: [consensus algorithms, blockchain]
      expected_answer: CONNECTION    // Relationship explanation

  ROUTE TO DriftSearchSemantics
    REASON: "Connection question requires graph traversal"

  // ==================== ANCHOR IDENTIFICATION ====================
  NAVIGATE embedding_space
    STRATEGY: dual_anchor_identification

    LOCATING:
      source_entities: [
        Entity("Consensus Algorithms", CONCEPT)
        Entity("Paxos", ALGORITHM)
        Entity("Raft", ALGORITHM)
        Entity("PBFT", ALGORITHM)
      ]
      target_entities: [
        Entity("Blockchain", TECHNOLOGY)
        Entity("Bitcoin", APPLICATION)
        Entity("Ethereum", APPLICATION)
      ]

  SEMANTIC_TRACE:
    Source anchor: "Consensus Algorithms" + instances
    Target anchor: "Blockchain" + instances
    Distance in graph: ~2-3 hops

  // ==================== GRAPH TRAVERSAL ====================
  NAVIGATE entity_graph
    FROM source_entities
    TOWARD target_entities
    WITH max_hops=3

    DISCOVERING:
      connection_paths: [
        Path 1:
          Consensus Algorithms
          → Byzantine Fault Tolerance
          → Blockchain Consensus
          → Blockchain
          (strength: 0.92, hops: 3)

        Path 2:
          Paxos
          → State Machine Replication
          → Distributed Ledger
          → Blockchain
          (strength: 0.88, hops: 3)

        Path 3:
          PBFT
          → Byzantine Agreement
          → Proof of Stake
          → Ethereum
          → Blockchain
          (strength: 0.85, hops: 4)

        ... (18 paths total)
      ]

      key_intermediaries: [
        "Byzantine Fault Tolerance" (appears in 8 paths)
        "State Machine Replication" (appears in 5 paths)
        "Distributed Ledger" (appears in 6 paths)
      ]

  SEMANTIC_TRACE:
    Path 1 details:
      Consensus Algorithms --[enables]--> Byzantine Fault Tolerance
        Edge strength: 9/10
        Source: "Consensus algorithms like PBFT provide Byzantine
                 fault tolerance..." [text_unit_8923]

      Byzantine Fault Tolerance --[required_for]--> Blockchain Consensus
        Edge strength: 8/10
        Source: "Blockchain consensus requires Byzantine fault
                 tolerance to handle malicious nodes..." [text_unit_12445]

      Blockchain Consensus --[part_of]--> Blockchain
        Edge strength: 10/10
        Source: "Blockchain systems use consensus mechanisms
                 to agree on ledger state..." [text_unit_15678]

  // ==================== PATH ANALYSIS ====================
  LOCATE significant_paths
    RANKED BY [semantic_relevance, path_strength, uniqueness]

    IDENTIFYING:
      primary_connection: Byzantine Fault Tolerance (central bridge)
      alternative_paths: State Machine Replication, Distributed Ledger
      strength_distribution: [0.92, 0.88, 0.85, 0.82, ...]

  // ==================== ANSWER COMPOSITION ====================
  COMPOSE answer
    FROM [paths, intermediaries, relationships]
    STRATEGY: connection_narrative

    INTEGRATING:
      multiple_paths: 5 strongest paths
      key_bridges: 3 main intermediaries
      technical_details: from text_units

    PRESERVING:
      connection_validity: 0.88        // Paths are valid
      technical_accuracy: 0.85         // Technical facts correct
      relationship_semantics: 0.90     // Relationships correct

  SEMANTIC_TRACE:
    Final answer:
      "# Connection Between Consensus Algorithms and Blockchain

      Consensus algorithms are fundamentally connected to blockchain
      through several paths:

      ## Primary Connection: Byzantine Fault Tolerance

      The strongest connection (strength: 0.92) flows through Byzantine
      Fault Tolerance:

      1. **Consensus Algorithms → BFT**: Classical consensus algorithms
         like Paxos and PBFT provide Byzantine fault tolerance, allowing
         systems to reach agreement even with malicious nodes.
         [Data: Textual (8923)]

      2. **BFT → Blockchain**: Blockchain consensus mechanisms (Proof of
         Work, Proof of Stake) are specialized forms of Byzantine fault
         tolerant consensus designed for decentralized networks.
         [Data: Textual (12445); Entities (Byzantine Fault Tolerance)]

      ## Alternative Paths:

      ### Via State Machine Replication (strength: 0.88)
      Consensus algorithms enable replicated state machines, which form
      the conceptual foundation for distributed ledgers in blockchain.
      [Data: Entities (Paxos, State Machine Replication, Distributed Ledger)]

      ### Via Distributed Ledger (strength: 0.85)
      Consensus mechanisms ensure all nodes agree on the ledger state,
      which is the core function of blockchain systems.
      [Data: Entities (Distributed Ledger, Blockchain)]

      ## Key Insight:
      Blockchain systems are essentially distributed consensus systems
      designed for open, permissionless networks. They inherit core
      concepts from classical consensus algorithms but adapt them for
      decentralized trust.

      [Discovered: 18 connection paths, 3 key intermediaries]"

  QUERY_RESULT:
    Latency: 42s (traversal + analysis + composition)
    Cost: $0.28

    Quality:
      Connection validity: 0.88 (paths are real)
      Discovery value: 0.85 (found non-obvious connections)
      Technical accuracy: 0.85
      Multi-path coverage: 0.90 (multiple perspectives)

    Semantic value:
      Intent preserved: 0.58
      Discovery enrichment: +3.15 (found connections, intermediaries)
      Net quality: 3.73x query intent
```

---

## Semantic Metrics Comparison

### Indexing Metrics

| Scenario | Corpus Size | Preservation | Queryability | Time | Cost |
|---|---|---|---|---|---|
| Research Papers | 100 papers, 2.5M tokens | 0.55 | 0.90 | 2.5h | $293 |
| Product Docs | 250 docs, 800K tokens | 0.62 | 0.95 | 45m | $78 |
| Exploratory | 500 docs, 1.8M tokens | 0.58 | 0.88 | 3.5h | $425 |

### Query Metrics

| Scenario | Query Type | Intent Preserved | Enrichment | Net Quality | Latency | Cost |
|---|---|---|---|---|---|---|
| Research Papers | Global | 0.58 | 2.80 | 3.38 | 25s | $0.38 |
| Product Docs | Local | 0.64 | 3.56 | 4.20 | 4s | $0.09 |
| Exploratory | Drift | 0.58 | 3.15 | 3.73 | 42s | $0.28 |

### Semantic Trade-offs

**Research Papers (Global Search)**:
- ✅ Comprehensive overview (0.88)
- ✅ Multi-paper synthesis
- ⚠️ Moderate accuracy (0.85)
- 📊 Best for: Literature review, trend analysis

**Product Docs (Local Search)**:
- ✅ Highest accuracy (0.92)
- ✅ Actionable steps (0.95)
- ✅ Fast response (4s)
- 📊 Best for: How-to questions, debugging

**Exploratory (Drift Search)**:
- ✅ Discovery of connections (0.85)
- ✅ Multi-path insights (0.90)
- ⚠️ Longer latency (42s)
- 📊 Best for: Research exploration, concept connections

---

## Semantic Insights

### Preservation Patterns

1. **Technical domains require higher precision**:
   - Product docs: 0.62 preservation (vs 0.55 research papers)
   - Reason: Code and procedures must be exact

2. **Exploratory queries benefit from graph connectivity**:
   - Cross-community edges critical
   - Path diversity enables discovery

3. **Global queries trade accuracy for comprehensiveness**:
   - Lower factual accuracy (0.78-0.85)
   - But higher coverage (0.85-0.90)

### Optimization Recommendations

**For Research Papers**:
- Smaller chunks (800 tokens) for technical precision
- Domain-specific entity types (method, metric, finding)
- Fine-grained communities (max_cluster_size=8)

**For Product Documentation**:
- Preserve code examples intact
- Focus on procedural steps
- High precision over recall

**For Exploratory Research**:
- Preserve cross-community connections
- Enable multi-hop traversal
- Focus on relationship extraction quality

---

**Related**:
- [Semantic Intentions](01-semantic-intentions.md)
- [Indexing Semantics](02-indexing-semantics.md)
- [Query Semantics](03-query-semantics.md)
