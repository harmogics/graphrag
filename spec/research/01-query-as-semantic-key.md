# Conceptual Research: Query as Semantic Key

## Обзор

Этот документ исследует **фундаментальную концепцию вопроса как ключа** в семантическом пространстве GraphRAG, где query служит инструментом для открытия ("unlocking") специфических проявлений концептуального знания.

## Философская основа

### Query-Key-Lock Paradigm

```
Query (Intention)
    ↓ [Transformation]
Semantic Key (Structured Intent + Vector)
    ↓ [Matching]
Semantic Lock (Indexed Knowledge Space)
    ↓ [Unlocking]
Manifestation (Retrieved Knowledge)
```

Вопрос функционирует как **многомерный ключ**, который проходит через серию трансформаций, обретая форму, способную "открыть" релевантные области в семантическом пространстве.

---

## Phase 1: Intent as Primordial Key

### Концепция

Пользовательский вопрос начинается как **нетипизированное намерение** — чистая информационная потребность в естественно-языковой форме.

### Трансформация: INTERPRET

```sfl
INTERPRET query_text
  FROM natural_language
  INTO semantic_intent

  EXTRACTING:
    query_type: {global, local, drift, basic}  // KEY SHAPE
    semantic_scope: {broad, narrow, exploratory}  // KEY RANGE
    key_concepts: [entities, topics]  // KEY TEETH
    expected_answer_type: {summary, explanation, list}  // LOCK TYPE
```

### Код реализации

```python
# graphrag/query/structured_search/base.py
# Intent interpretation определяет "форму ключа"

def interpret_query(query: str) -> SemanticIntent:
    """Transform raw query into structured key."""
    # Pattern matching для определения query type
    if is_overview_question(query):  # "What are...", "List..."
        query_type = "GLOBAL"  # Key для широкого пространства
    elif is_specific_question(query):  # "What is X?", "How does Y?"
        query_type = "LOCAL"  # Key для узкого пространства
    elif is_connection_question(query):  # "How are X and Y related?"
        query_type = "DRIFT"  # Key для навигационного пространства

    # Extraction концептов = формирование "зубцов ключа"
    key_concepts = extract_entities(query)  # ["GraphRAG", "knowledge graph"]

    return SemanticIntent(
        type=query_type,  # Форма ключа
        concepts=key_concepts,  # Зубцы ключа
        scope=determine_scope(query),  # Диапазон ключа
    )
```

### Философский аспект

**Intent Interpretation** — это процесс придания формы чистому намерению. Вопрос "What are the main trends?" обретает структуру:
- **Форма**: GLOBAL (требуется overview)
- **Зубцы**: ["trends", "main"] (ключевые концепты)
- **Диапазон**: BROAD (широкое покрытие)

Эта структура **предопределяет**, какие "замки" в семантическом пространстве могут быть открыты.

---

## Phase 2: Vector as Geometric Key

### Концепция

Структурированное намерение трансформируется в **геометрический ключ** — вектор в высокоразмерном семантическом пространстве.

### Трансформация: ENCODE

```sfl
ENCODE semantic_intent
  FROM symbolic_representation
  INTO query_vector

  TRANSFORMING:
    discrete_symbols → continuous_geometry  // KEY → PHYSICAL FORM
    categorical_intent → metric_space  // INTENTION → MEASURABLE

  PRESERVING:
    semantic_meaning: 0.92  // Essence preserved

  ENABLING:
    geometric_matching: cosine_similarity  // KEY-LOCK FITTING
    fast_navigation: O(log n)  // EFFICIENT SEARCH
```

### Код реализации

```python
# graphrag/query/context_builder/entity_extraction.py

async def map_query_to_entities(
    query: str,
    text_embedding_vectorstore: BaseVectorStore,
    text_embedder: EmbeddingModel,
    k: int = 10,
) -> list[Entity]:
    """Use query vector as geometric key to unlock entities."""

    # Encode query into geometric key
    query_vector = await text_embedder.embed(query)  # Shape: (1536,)
    # query_vector = [0.123, -0.456, 0.789, ..., 0.234]
    # This IS the geometric key

    # Search for matching "locks" in vector space
    search_results = text_embedding_vectorstore.similarity_search_by_vector(
        query_vector=query_vector,
        k=k,  # Top k "locks" that fit the key
    )

    # Cosine similarity measures "key-lock fit"
    # similarity = cosine(query_vector, entity_vector)
    # High similarity = Key fits the lock

    return [result.entity for result in search_results]
```

### Геометрическая интерпретация

```
Semantic Space (1536-dimensional)

Query Vector (Key): Q = [q₁, q₂, ..., q₁₅₃₆]
Entity Vectors (Locks): E₁ = [e₁₁, e₁₂, ..., e₁₁₅₃₆]
                        E₂ = [e₂₁, e₂₂, ..., e₂₁₅₃₆]
                        ...

Matching (Key-Lock Fit):
    similarity(Q, Eᵢ) = cos(θ) = Q·Eᵢ / (||Q|| ||Eᵢ||)

If similarity > threshold:
    KEY FITS LOCK → Entity unlocked
Else:
    KEY DOESN'T FIT → Entity remains locked
```

### Философский аспект

Vectorization превращает **символическое намерение в геометрический ключ**. В этом пространстве:
- **Близость = Семантическое родство**
- **Направление = Смысловой вектор**
- **Расстояние = Концептуальная дистанция**

Query vector "ищет" области пространства, где его "форма" максимально соответствует форме "замков" (entity vectors).

---

## Phase 3: Routing as Key-Lock Type Matching

### Концепция

Различные типы вопросов требуют различных типов "замков". **Routing** определяет, какой тип семантического пространства должен быть открыт.

### Трансформация: ROUTE

```sfl
ROUTE BY semantic_intent.query_type:

  WHEN query_type IS GLOBAL:
    // KEY TYPE: Broad overview key
    // LOCK TYPE: Community summaries (aggregated knowledge)
    NAVIGATE TO: community_semantic_space
    CHARACTERISTICS:
      - Scope: BROAD (many communities)
      - Granularity: COARSE (summary level)
      - Strategy: Map-Reduce (parallel unlocking)

  WHEN query_type IS LOCAL:
    // KEY TYPE: Specific entity key
    // LOCK TYPE: Detailed entity + text units
    NAVIGATE TO: entity_semantic_space
    CHARACTERISTICS:
      - Scope: NARROW (specific entities)
      - Granularity: FINE (detailed level)
      - Strategy: Direct unlocking (semantic search)

  WHEN query_type IS DRIFT:
    // KEY TYPE: Connection exploration key
    // LOCK TYPE: Graph paths
    NAVIGATE TO: relationship_semantic_space
    CHARACTERISTICS:
      - Scope: EXPLORATORY (multi-hop)
      - Granularity: RELATIONAL (path level)
      - Strategy: Sequential unlocking (traversal)
```

### Код реализации

```python
# graphrag/query/factory.py

def get_search_engine(query_intent: SemanticIntent, config: Config):
    """Route to appropriate search engine based on key type."""

    if query_intent.type == "GLOBAL":
        # Use broad key to unlock community summaries
        return GlobalSearch(
            context_builder=GlobalContextBuilder(
                # Communities are "locks" for broad keys
                community_reports=reports,
                ...
            ),
            ...
        )

    elif query_intent.type == "LOCAL":
        # Use specific key to unlock entity details
        return LocalSearch(
            context_builder=LocalContextBuilder(
                # Entities + text units are "locks" for specific keys
                entities=entities,
                text_units=text_units,
                ...
            ),
            ...
        )

    elif query_intent.type == "DRIFT":
        # Use exploration key to unlock graph paths
        return DRIFTSearch(
            context_builder=DRIFTContextBuilder(
                # Relationships are "locks" for exploration keys
                relationships=relationships,
                ...
            ),
            ...
        )
```

### Философский аспект

**Routing** — это метапроцесс выбора правильной **категории замков**. Не все замки открываются одним ключом. Вопросы разных типов требуют доступа к разным слоям семантической реальности:
- **Global**: Доступ к коллективному знанию (communities)
- **Local**: Доступ к индивидуальному знанию (entities)
- **Drift**: Доступ к реляционному знанию (paths)

---

## Phase 4: Navigation as Key Movement

### Концепция

Geometric key (query vector) **навигирует** через семантическое пространство, приближаясь к областям с высокой плотностью релевантных "замков".

### Трансформация: NAVIGATE

```sfl
NAVIGATE embedding_space
  TOWARD query_vector
  STRATEGY: determined_by_query_type

  MOVEMENT_PATTERN:
    - Start: Query vector position in space
    - Direction: Gradient of semantic similarity
    - Destination: High-density regions of relevant locks

  RESULT:
    - Located: Candidate locks (entities, communities, paths)
    - Filtered: By relevance threshold
    - Ranked: By key-lock fit (similarity score)
```

### Код реализации

```python
# graphrag/query/structured_search/local_search/search.py

class LocalSearch:
    async def search(self, query: str) -> SearchResult:
        # Build context = Navigate to relevant locks
        context_result = self.context_builder.build_context(query)

        # Internal navigation:
        # 1. Embed query → geometric key
        query_vector = await self.embedder.embed(query)

        # 2. Navigate to nearest entities (locks)
        similar_entities = await self.vector_store.similarity_search(
            query_vector, k=30
        )  # Top 30 locks that "fit" the key

        # 3. Expand navigation via graph
        related_relationships = get_relationships(similar_entities)
        expanded_entities = expand_via_relationships(
            similar_entities, related_relationships
        )

        # Navigation complete: Found relevant locks
        return context_result
```

### Визуализация навигации

```
Semantic Space (conceptual visualization)

    Community C1 ●
                  \
                   \
    Query Q -------→ Entity E1 ● ← Found (similarity: 0.89)
          |          |
          |          |
          |          Entity E2 ● ← Found (similarity: 0.85)
          |
          └--------→ Community C2 ● ← Found (similarity: 0.82)

Navigation path:
1. Query vector Q enters space
2. Measures similarity to all entities/communities
3. Selects top k with highest similarity (best key-lock fit)
4. Expands via relationships (graph navigation)
```

### Философский аспект

**Navigation** — это процесс **движения ключа через поле замков**. Query vector не просто "сопоставляется" со всеми возможными замками; он **перемещается** по градиенту семантического сходства, естественным образом приближаясь к областям с высокой релевантностью.

Это аналогично гравитационному притяжению: ключ "притягивается" к замкам с высоким семантическим сродством.

---

## Phase 5: Unlocking as Manifestation

### Концепция

Когда ключ находит подходящие замки (высокая semantic similarity), происходит **unlocking** — извлечение конкретных проявлений концептуального знания.

### Трансформация: LOCATE

```sfl
LOCATE relevant_semantics
  IN navigated_space
  RANKED BY relevance_score

  UNLOCKING_PROCESS:
    - Measure: Key-lock fit (cosine similarity)
    - Filter: Locks with fit > threshold
    - Extract: Contents behind locks (entities, text, summaries)
    - Return: Manifested knowledge

  PRESERVING:
    relevance_precision: 0.85-0.90  // Unlocked correct locks

  RESULT:
    concept → manifestation binding established
```

### Код реализации

```python
# graphrag/query/context_builder/local_context.py

def build_context(self, query: str, **kwargs) -> ContextBuilderResult:
    """Unlock knowledge using query as key."""

    # Phase 1: Locate locks (entities)
    query_embedding = self.text_embedder.embed(query)  # Key
    matched_entities = self.entity_embeddings.similarity_search(
        query_embedding, k=30
    )  # Locks that fit

    # Phase 2: Unlock entities (extract content)
    unlocked_entities = []
    for entity_match in matched_entities:
        # "Unlock" = retrieve entity data
        entity = get_entity_by_id(entity_match.id)
        unlocked_entities.append(entity)

    # Phase 3: Unlock related content
    text_units = []
    for entity in unlocked_entities:
        # Each entity "unlocks" its text unit manifestations
        for text_unit_id in entity.text_unit_ids:
            text_unit = get_text_unit(text_unit_id)
            text_units.append(text_unit)  # Concrete manifestation

    # Phase 4: Unlock relationships
    relationships = get_relationships(unlocked_entities)

    return ContextBuilderResult(
        entities=unlocked_entities,  # Abstract concepts
        text_units=text_units,  # Concrete manifestations
        relationships=relationships,  # Connections
    )
```

### Concept-Manifestation Binding

```
Abstract Concept (Entity)
    ↓ [UNLOCKED BY QUERY KEY]
Concrete Manifestations (Text Units)

Example:
Query: "What is GraphRAG?"
    ↓ [Encode to vector]
Key: [0.123, -0.456, ..., 0.789]
    ↓ [Navigate & Match]
UNLOCKED: Entity("GraphRAG", type="TECHNOLOGY")
    ↓ [Retrieve manifestations]
MANIFESTATIONS:
    - Text Unit 1: "GraphRAG is a knowledge graph approach..."
    - Text Unit 2: "The system uses community detection..."
    - Text Unit 3: "GraphRAG enables semantic search..."
```

### Философский аспект

**Unlocking** — это момент **связывания концепции с ее проявлениями**. Entity "GraphRAG" существует как абстрактная концепция, но query-key открывает доступ к конкретным текстовым проявлениям этой концепции.

Это философски соответствует платоновскому различию между Идеей (Entity) и ее чувственными проявлениями (Text Units). Query служит инструментом, позволяющим "видеть" конкретное через призму абстрактного.

---

## Phase 6: Composition as Answer Synthesis

### Концепция

Unlocked knowledge (manifestations) **компонуется** в связный ответ — синтез множественных проявлений в единое семантическое целое.

### Трансформация: COMPOSE

```sfl
COMPOSE answer
  FROM [entities, text_units, relationships]
  STRATEGY: llm_guided_synthesis

  SYNTHESIS_PROCESS:
    - Input: Multiple manifestations (unlocked knowledge)
    - Integration: Weave manifestations into narrative
    - Grounding: Cite sources (which locks were opened)
    - Output: Coherent answer (unified manifestation)

  PRESERVING:
    factual_accuracy: 0.82-0.88  // Truth preserved
    semantic_coherence: 0.88-0.92  // Unity achieved

  RESULT:
    Many → One (multiplicity → unity)
```

### Код реализации

```python
# graphrag/query/structured_search/local_search/search.py

async def generate_answer(
    query: str,
    context: ContextBuilderResult,
) -> str:
    """Compose answer from unlocked manifestations."""

    # Format unlocked knowledge
    context_text = format_context(
        entities=context.entities,  # Concepts unlocked
        text_units=context.text_units,  # Manifestations unlocked
        relationships=context.relationships,  # Connections unlocked
    )

    # LLM synthesizes multiplicity into unity
    prompt = f"""
    Context (unlocked knowledge):
    {context_text}

    Question (original key):
    {query}

    Synthesize a coherent answer from the unlocked knowledge.
    """

    answer = await llm.achat(prompt)

    return answer  # Unified manifestation
```

### Философский аспект

**Composition** — это **синтез множественности в единство**. Query-key открывает множество проявлений (entities, text units), но ответ требует их интеграции в единое семантическое целое.

Это аналогично герменевтическому кругу: понимание целого через части (unlocked manifestations) и частей через целое (synthesized answer).

---

## Полный цикл: Query-Key-Lock-Unlock-Manifest

```
User Question: "What is GraphRAG?"
    ↓ [Phase 1: INTERPRET]
Semantic Intent:
    type: LOCAL
    concepts: ["GraphRAG"]
    scope: NARROW
    ↓ [Phase 2: ENCODE]
Query Vector (Key):
    [0.123, -0.456, 0.789, ..., 0.234]  (1536-dim)
    ↓ [Phase 3: ROUTE]
Search Strategy: LocalSearch
Target Space: Entity + Text Unit space
    ↓ [Phase 4: NAVIGATE]
Navigate to similar entities:
    - Entity("GraphRAG") - similarity: 0.92  ← High fit!
    - Entity("RAG") - similarity: 0.85
    - Entity("Knowledge Graph") - similarity: 0.83
    ↓ [Phase 5: LOCATE & UNLOCK]
Unlocked Entities:
    - GraphRAG (concept)
Unlocked Manifestations:
    - "GraphRAG is a knowledge graph approach to RAG..." (text)
    - "The system uses community detection..." (text)
    - "Developed by Microsoft Research..." (text)
Unlocked Relationships:
    - GraphRAG --implements--> RAG
    - GraphRAG --uses--> Community Detection
    ↓ [Phase 6: COMPOSE]
Synthesized Answer:
    "GraphRAG is a knowledge graph-based approach to Retrieval-
     Augmented Generation developed by Microsoft Research. The
     system uses community detection to organize entities into
     hierarchical topics, enabling semantic search across documents.
     [Citations: Entity(GraphRAG), TextUnit(1847, 2451)]"
```

---

## Концептуальные выводы

### 1. Query as Multi-Dimensional Key

Query — это **многомерный ключ** с несколькими "гранями":
- **Symbolic dimension**: Intent type, concepts (форма ключа)
- **Geometric dimension**: Vector representation (физическая форма)
- **Navigational dimension**: Strategy, routing (тип замка)

### 2. Semantic Space as Lock Field

Индексированное знание — это **поле замков** различных типов:
- **Entity locks**: Концептуальные замки (abstract)
- **Text unit locks**: Текстовые замки (concrete)
- **Community locks**: Агрегированные замки (holistic)
- **Relationship locks**: Реляционные замки (connective)

### 3. Similarity as Key-Lock Fit

Cosine similarity — это мера **соответствия ключа замку**:
- High similarity (>0.8): Key fits perfectly → Full unlock
- Medium similarity (0.6-0.8): Partial fit → Related unlock
- Low similarity (<0.6): No fit → Lock remains closed

### 4. Unlocking as Concept-Manifestation Binding

Процесс unlocking устанавливает связь между:
- **Концепцией** (Entity) — абстрактное, идеальное
- **Проявлением** (Text Unit) — конкретное, материальное

Query-key служит инструментом **трансцендентации** — перехода от вопроса (intention) к ответу (manifestation) через семантическое пространство (field of possibilities).

---

## Философские параллели

### Платонизм
- **Идеи** (Entities) vs **Чувственные проявления** (Text Units)
- Query как инструмент **анамнезиса** (припоминания знания)

### Феноменология (Husserl)
- **Интенциональность**: Query как направленность сознания
- **Ноэма** (смысл) vs **Ноэзис** (акт придания смысла)

### Герменевтика (Gadamer)
- **Герменевтический круг**: Понимание через итерацию
- **Слияние горизонтов**: Query (вопрошающий) + Knowledge (вопрошаемое)

---

**Related**:
- [Star-Attractor Patterns](02-star-attractor-patterns.md)
- [LLM Awareness Patterns](03-llm-awareness-patterns.md)
- [Concept-Manifestation Duality](04-concept-manifestation-duality.md)
