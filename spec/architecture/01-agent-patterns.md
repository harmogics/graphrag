# Agent Patterns

## Обзор

**Agent Patterns** — это архитектурные паттерны, связанные с использованием LLM (Large Language Models) в качестве агентов для выполнения когнитивных задач. В GraphRAG агенты используются для извлечения, анализа и генерации семантической информации.

## Основные Agent Patterns

1. **LLM Agent Pattern** - базовый паттерн взаимодействия с LLM
2. **Gleaning Pattern** - итеративное уточнение результатов
3. **Map-Reduce Agent Pattern** - распределенная обработка
4. **Multi-hop Agent Pattern** - многоступенчатая навигация
5. **Extraction Agent Pattern** - структурированное извлечение данных

---

## 1. LLM Agent Pattern

### Определение

**LLM Agent** — это объект, который инкапсулирует взаимодействие с language model для выполнения специфической задачи с четко определенным промптом.

### Структура

```python
class LLMAgent:
    """Base pattern for LLM-based agents."""

    def __init__(
        self,
        model: ChatModel,           # LLM для вызова
        prompt_template: str,       # Шаблон промпта
        parser: ResponseParser,     # Парсер ответа
        error_handler: ErrorHandler # Обработка ошибок
    ):
        self._model = model
        self._prompt_template = prompt_template
        self._parser = parser
        self._error_handler = error_handler

    async def execute(self, input_data: Any) -> Output:
        # 1. Format prompt
        prompt = self._prompt_template.format(**input_data)

        # 2. Call LLM
        try:
            response = await self._model.achat(prompt)
        except Exception as e:
            return self._error_handler(e, input_data)

        # 3. Parse response
        output = self._parser.parse(response)

        return output
```

### Реализация в GraphRAG: GraphExtractor

```python
# graphrag/index/operations/extract_graph/graph_extractor.py

class GraphExtractor:
    """LLM Agent for extracting entities and relationships from text."""

    def __init__(
        self,
        model_invoker: ChatModel,
        prompt: str | None = None,
        max_gleanings: int = 1,
        on_error: ErrorHandlerFn | None = None,
    ):
        self._model = model_invoker
        self._extraction_prompt = prompt or GRAPH_EXTRACTION_PROMPT
        self._max_gleanings = max_gleanings
        self._on_error = on_error or (lambda _e, _s, _d: None)

    async def __call__(
        self, texts: list[str], prompt_variables: dict[str, Any]
    ) -> GraphExtractionResult:
        """Execute extraction agent."""
        all_records: dict[int, str] = {}

        for doc_index, text in enumerate(texts):
            try:
                # Format prompt with text
                prompt = self._extraction_prompt.format(
                    input_text=text,
                    entity_types=prompt_variables["entity_types"],
                    tuple_delimiter=prompt_variables["tuple_delimiter"],
                    # ...
                )

                # Call LLM
                response = await self._model.achat(
                    prompt=prompt,
                    temperature=0.0,
                )

                # Parse response
                records = self._parse_extraction_response(response)
                all_records[doc_index] = records

            except Exception as e:
                self._on_error(e, traceback.format_exc(), None)
                continue

        # Convert records to graph
        graph = self._build_graph_from_records(all_records)

        return GraphExtractionResult(output=graph, source_docs=...)
```

### Ключевые компоненты

| Компонент | Назначение | Пример в GraphExtractor |
|---|---|---|
| **Model** | LLM для вызова | `ChatModel` (GPT-4, GPT-3.5, etc.) |
| **Prompt Template** | Инструкция для LLM | `GRAPH_EXTRACTION_PROMPT` |
| **Input Formatter** | Подготовка входных данных | `prompt.format(input_text=text, ...)` |
| **Response Parser** | Парсинг ответа LLM | `_parse_extraction_response()` |
| **Error Handler** | Обработка ошибок | `on_error` callback |

### Преимущества

✅ Инкапсуляция сложности взаимодействия с LLM
✅ Переиспользуемость промптов и парсеров
✅ Централизованная обработка ошибок
✅ Легко тестировать с mock LLM

---

## 2. Gleaning Pattern

### Определение

**Gleaning** — итеративное уточнение результатов LLM путем повторных запросов для извлечения дополнительной информации, которая могла быть пропущена в первом проходе.

### Концепция

```
Initial Extraction
    ↓
Check: "Are there more entities?"
    ↓ (Yes)
Additional Extraction (gleaning)
    ↓
Merge Results
    ↓
Check: "Are there more entities?"
    ↓ (No)
Final Result
```

### Алгоритм

```python
async def gleaning_extraction(
    text: str,
    max_gleanings: int = 1
) -> ExtractionResult:
    # Initial extraction
    result = await extract_entities(text)

    for gleaning_round in range(max_gleanings):
        # Ask LLM: "Are there more entities?"
        has_more = await check_for_more_entities(
            text=text,
            already_extracted=result.entities
        )

        if not has_more:
            break  # No more entities

        # Extract additional entities
        additional = await extract_additional_entities(
            text=text,
            already_extracted=result.entities
        )

        # Merge with previous results
        result = merge_results(result, additional)

    return result
```

### Реализация в GraphRAG

```python
# graphrag/index/operations/extract_graph/graph_extractor.py

class GraphExtractor:
    async def __call__(self, texts: list[str], ...) -> GraphExtractionResult:
        all_records = {}

        for doc_index, text in enumerate(texts):
            # ===== INITIAL EXTRACTION =====
            response = await self._model.achat(
                prompt=self._extraction_prompt.format(input_text=text, ...)
            )
            records = response

            # ===== GLEANING ITERATIONS =====
            for gleaning in range(self._max_gleanings):
                # Ask: "Did you miss any entities?"
                gleaning_response = await self._model.achat(
                    prompt=CONTINUE_PROMPT,  # "MANY entities were missed. Add them:"
                    history=[...previous messages...]
                )

                if not gleaning_response or len(gleaning_response) == 0:
                    break

                # Check if LLM thinks there are more
                check_response = await self._model.achat(
                    prompt=LOOP_PROMPT,  # "Answer Y or N if there are still entities"
                    **self._loop_args  # Logit bias for Y/N tokens
                )

                if check_response.strip().lower() != 'y':
                    break  # No more entities

                # Merge gleaned entities with original
                records += gleaning_response

            all_records[doc_index] = records

        return GraphExtractionResult(...)
```

### Prompts для Gleaning

```python
# graphrag/prompts/index/extract_graph.py

CONTINUE_PROMPT = """
MANY entities were missing in your last extraction.
Add the entities you missed using the same format:
"""

LOOP_PROMPT = """
It appears some entities may have still been missed.
Answer Y or N: Are there still entities that need to be added?
"""
```

### Метрики Gleaning

| Метрика | Без Gleaning | С Gleaning (1 round) | Improvement |
|---|---|---|---|
| Precision | 0.92 | 0.92 | 0% (stable) |
| Recall | 0.70 | 0.85 | +15% |
| F1 Score | 0.79 | 0.88 | +9% |
| Latency | 2.5s | 5.0s | +100% (trade-off) |
| Cost | $0.02 | $0.04 | +100% (trade-off) |

### Когда использовать Gleaning

✅ **Используйте**:
- Когда recall критичен (нужно найти ВСЕ сущности)
- Когда допустима большая latency
- Для сложных документов с множеством сущностей

❌ **Не используйте**:
- Когда latency критична
- Когда бюджет LLM calls ограничен
- Для простых документов с очевидными сущностями

---

## 3. Map-Reduce Agent Pattern

### Определение

**Map-Reduce Pattern** — распределение задачи по множеству независимых агентов (MAP phase), с последующим агрегированием результатов (REDUCE phase).

### Архитектура

```
Query
  ↓
[MAP PHASE] Parallel processing
  ├─> Agent 1 → Intermediate Result 1
  ├─> Agent 2 → Intermediate Result 2
  ├─> Agent 3 → Intermediate Result 3
  └─> Agent N → Intermediate Result N
  ↓
[REDUCE PHASE] Aggregation
  Reduce Agent → Final Result
```

### Реализация в GraphRAG: Global Search

```python
# graphrag/query/structured_search/global_search/search.py

class GlobalSearch(BaseSearch[GlobalContextBuilder]):
    """Map-Reduce search across community summaries."""

    async def search(self, query: str, **kwargs) -> GlobalSearchResult:
        start_time = time.time()

        # ===== RETRIEVE COMMUNITIES (preparation) =====
        context_result = await self.context_builder.build_context(
            query=query, **self.context_builder_params
        )
        # context_result contains ~10 community summaries

        # ===== MAP PHASE: Parallel LLM calls =====
        map_responses = []

        # Create tasks for parallel execution
        async def map_task(community_summary: str) -> SearchResult:
            # Each community gets its own LLM call
            map_prompt = self.map_system_prompt.format(
                context_data=community_summary,
                response_type=self.response_type,
            )

            response = await self.model.achat(
                prompt=query,
                history=[{"role": "system", "content": map_prompt}],
                **self.map_llm_params
            )

            return SearchResult(
                response=response,
                context_data=community_summary,
                ...
            )

        # Execute MAP phase in parallel
        map_tasks = [
            map_task(community) for community in context_result.communities
        ]
        map_responses = await asyncio.gather(*map_tasks)  # Parallel!

        # ===== REDUCE PHASE: Aggregate results =====
        # Collect all intermediate answers
        intermediate_answers = [
            r.response for r in map_responses
        ]

        # Reduce prompt
        reduce_prompt = self.reduce_system_prompt.format(
            report_data="\n\n".join(intermediate_answers),
            response_type=self.response_type,
        )

        # Single LLM call to synthesize final answer
        final_response = await self.model.achat(
            prompt=query,
            history=[{"role": "system", "content": reduce_prompt}],
            **self.reduce_llm_params
        )

        completion_time = time.time() - start_time

        return GlobalSearchResult(
            response=final_response,
            map_responses=map_responses,  # Intermediate results
            completion_time=completion_time,
            llm_calls=len(map_responses) + 1,  # N map + 1 reduce
            ...
        )
```

### Map Prompt Example

```
SYSTEM: You are analyzing the following community report:

{community_report}

Based on this report, provide points relevant to: "{user_query}"

Format as JSON:
{
  "points": ["point 1", "point 2", ...],
  "rating": 0-100
}
```

### Reduce Prompt Example

```
SYSTEM: You have received the following intermediate responses:

Response 1: {map_response_1}
Response 2: {map_response_2}
...
Response 10: {map_response_10}

Synthesize a comprehensive answer to: "{user_query}"
covering all important points from the responses.
```

### Performance Characteristics

| Aspect | Value | Note |
|---|---|---|
| Map LLM Calls | N (10 typically) | Parallel execution |
| Reduce LLM Calls | 1 | Sequential after map |
| Total Latency | max(map_times) + reduce_time | ~25s typical |
| Total Cost | N * map_cost + reduce_cost | ~$0.45 per query |
| Coverage | High (multiple communities) | 24% of graph |
| Parallelization | High | Limited by concurrency |

### Concurrency Control

```python
# Control parallel execution
concurrent_coroutines = 32  # Max parallel LLM calls

# Batch processing
for batch in chunks(map_tasks, concurrent_coroutines):
    batch_results = await asyncio.gather(*batch)
    map_responses.extend(batch_results)
```

---

## 4. Multi-hop Agent Pattern

### Определение

**Multi-hop Pattern** — агент итеративно навигирует по графу знаний, выполняя несколько "прыжков" для исследования связей между концептами.

### Концепция (Drift Search)

```
Query: "How are consensus algorithms related to blockchain?"

Hop 1: "consensus algorithms" → [Paxos, Raft, PBFT]
  ↓
Hop 2: [Paxos, Raft, PBFT] → [State Machine Replication, Byzantine Fault Tolerance]
  ↓
Hop 3: [Byzantine Fault Tolerance] → [Blockchain, Distributed Ledger]
  ↓
Found connection!
```

### Алгоритм

```python
async def multi_hop_search(
    query: str,
    start_entities: list[Entity],
    target_entities: list[Entity],
    max_hops: int = 3
) -> list[Path]:
    # Initialize frontier with start entities
    frontier = [(entity, [entity]) for entity in start_entities]
    discovered_paths = []

    for hop in range(max_hops):
        new_frontier = []

        for current_entity, path in frontier:
            # Get neighbors
            neighbors = get_connected_entities(current_entity)

            for neighbor in neighbors:
                new_path = path + [neighbor]

                # Check if reached target
                if neighbor in target_entities:
                    discovered_paths.append(new_path)
                    continue

                # Add to new frontier for next hop
                if len(new_path) < max_hops:
                    new_frontier.append((neighbor, new_path))

        frontier = new_frontier

        if not frontier:
            break  # No more nodes to explore

    return discovered_paths
```

### Реализация в GraphRAG: DRIFT Search

```python
# graphrag/query/structured_search/drift_search/search.py

class DRIFTSearch(BaseSearch[DRIFTContextBuilder]):
    """Multi-hop exploration search."""

    async def search(self, query: str, **kwargs) -> SearchResult:
        # ===== PHASE 1: IDENTIFY ANCHORS =====
        # Extract source and target concepts from query
        query_entities = extract_entities_from_query(query)
        # e.g., ["consensus algorithms", "blockchain"]

        # ===== PHASE 2: MULTI-HOP TRAVERSAL =====
        paths = []
        max_depth = 3  # Max hops

        # BFS-style traversal
        for depth in range(max_depth):
            # At each depth, find neighbors
            current_frontier = get_entities_at_depth(depth)

            for entity in current_frontier:
                # Get connected entities
                neighbors = get_relationships(entity)

                # For each neighbor, ask LLM: "Is this relevant?"
                for neighbor in neighbors:
                    relevance = await self._assess_relevance(
                        query=query,
                        entity=neighbor,
                        path_so_far=get_path_to(neighbor)
                    )

                    if relevance > threshold:
                        paths.append(get_path_to(neighbor))

        # ===== PHASE 3: PATH RANKING =====
        ranked_paths = rank_paths_by_relevance(paths, query)

        # ===== PHASE 4: ANSWER GENERATION =====
        # Generate narrative from top paths
        answer = await self._generate_connection_narrative(
            query=query,
            paths=ranked_paths[:10]  # Top 10 paths
        )

        return SearchResult(response=answer, ...)
```

### LLM Relevance Assessment

```python
async def _assess_relevance(
    self,
    query: str,
    entity: Entity,
    path_so_far: list[Entity]
) -> float:
    """Use LLM to assess if entity is relevant to query."""
    prompt = f"""
    Original query: {query}
    Path so far: {' → '.join([e.name for e in path_so_far])}
    Current entity: {entity.name} - {entity.description}

    On a scale of 0-100, how relevant is this entity to the query?
    Answer with just a number.
    """

    response = await self.model.achat(prompt)
    score = int(response.strip())
    return score / 100.0
```

### Performance Characteristics

| Aspect | Value |
|---|---|
| Hops | 2-3 typical |
| Paths discovered | 15-25 typical |
| LLM calls per hop | 10-20 (relevance assessment) |
| Total latency | 35-50s |
| Total cost | $0.25-$0.35 |
| Discovery value | High (finds non-obvious connections) |

---

## 5. Extraction Agent Pattern

### Определение

**Extraction Agent** — специализированный LLM agent для извлечения структурированных данных из неструктурированного текста с использованием четко определенного выходного формата.

### Структура

```python
class ExtractionAgent:
    """Pattern for structured data extraction."""

    def __init__(
        self,
        model: ChatModel,
        extraction_schema: OutputSchema,  # Desired output structure
        few_shot_examples: list[Example],  # Examples for LLM
    ):
        self._model = model
        self._schema = extraction_schema
        self._examples = few_shot_examples

    async def extract(self, text: str) -> StructuredOutput:
        # Build prompt with schema and examples
        prompt = self._build_extraction_prompt(
            text=text,
            schema=self._schema,
            examples=self._examples
        )

        # Call LLM
        response = await self._model.achat(prompt)

        # Parse and validate against schema
        structured_output = self._parse_and_validate(
            response, self._schema
        )

        return structured_output
```

### Пример: Entity Extraction Schema

```python
# Expected output format
ENTITY_SCHEMA = """
Output Format:
("entity"|<entity_name>|<entity_type>|<entity_description>)##
("relationship"|<source_entity>|<target_entity>|<description>|<strength>)##
<|COMPLETE|>

Entity Types: {entity_types}
Delimiters:
  - Record delimiter: ##
  - Tuple delimiter: |
  - Completion: <|COMPLETE|>
"""

# Few-shot examples
FEW_SHOT_EXAMPLES = [
    Example(
        input="Microsoft Research developed GraphRAG in 2023.",
        output='''
("entity"|MICROSOFT RESEARCH|ORGANIZATION|Research division of Microsoft)##
("entity"|GRAPHRAG|TECHNOLOGY|Knowledge graph RAG system)##
("entity"|2023|DATE|Year of development)##
("relationship"|GRAPHRAG|MICROSOFT RESEARCH|developed by|9)##
<|COMPLETE|>
'''
    ),
    # ... more examples
]
```

### Реализация в GraphRAG

```python
# graphrag/index/operations/extract_graph/graph_extractor.py

class GraphExtractor:
    """Extraction agent for entities and relationships."""

    async def __call__(self, texts: list[str], prompt_variables: dict) -> GraphExtractionResult:
        all_records = {}

        for doc_index, text in enumerate(texts):
            # ===== BUILD EXTRACTION PROMPT =====
            prompt = GRAPH_EXTRACTION_PROMPT.format(
                input_text=text,
                entity_types=prompt_variables["entity_types"],
                tuple_delimiter=DEFAULT_TUPLE_DELIMITER,
                record_delimiter=DEFAULT_RECORD_DELIMITER,
                completion_delimiter=DEFAULT_COMPLETION_DELIMITER,
            )

            # ===== CALL LLM =====
            response = await self._model.achat(
                prompt=prompt,
                temperature=0.0,  # Deterministic
            )

            # ===== PARSE STRUCTURED OUTPUT =====
            records = self._parse_response(
                response,
                tuple_delimiter=DEFAULT_TUPLE_DELIMITER,
                record_delimiter=DEFAULT_RECORD_DELIMITER,
            )

            all_records[doc_index] = records

        # ===== BUILD GRAPH FROM RECORDS =====
        graph = self._records_to_graph(all_records)

        return GraphExtractionResult(output=graph, source_docs=...)

    def _parse_response(
        self, response: str, tuple_delimiter: str, record_delimiter: str
    ) -> list[tuple]:
        """Parse LLM response into structured records."""
        records = []

        # Split by record delimiter
        raw_records = response.split(record_delimiter)

        for raw_record in raw_records:
            if not raw_record.strip():
                continue

            # Parse tuple: ("entity"|name|type|description)
            match = re.match(
                r'\("(\w+)"\|([^|]+)\|([^|]+)\|([^|]+)\)',
                raw_record
            )

            if match:
                record_type = match.group(1)  # "entity" or "relationship"
                fields = [match.group(i) for i in range(2, 5)]
                records.append((record_type, *fields))

        return records
```

### Best Practices для Extraction

1. **Четкая схема**: Определите точный формат вывода
2. **Few-shot examples**: 2-3 примера значительно улучшают качество
3. **Delimiters**: Используйте уникальные разделители (`##`, `<|>`)
4. **Validation**: Валидируйте выходные данные против схемы
5. **Error handling**: Обработка невалидных ответов LLM

---

## Сравнение Agent Patterns

| Pattern | Use Case | LLM Calls | Latency | Cost | Accuracy |
|---|---|---|---|---|---|
| **LLM Agent** | Basic task | 1 | Low | Low | Medium |
| **Gleaning** | High recall extraction | 2-3 | Medium | Medium | High |
| **Map-Reduce** | Broad coverage | N+1 | High | High | High |
| **Multi-hop** | Connection discovery | 15-30 | Very High | Medium | Medium |
| **Extraction** | Structured data | 1 | Low | Low | High (with examples) |

## Комбинирование Patterns

### Gleaning + Extraction

```python
# High-quality entity extraction
extractor = GraphExtractor(
    model=model,
    max_gleanings=1,  # Gleaning for recall
    # + Extraction schema for structure
)
```

### Map-Reduce + Extraction

```python
# Each map call uses extraction
async def map_with_extraction(community: str):
    # Extract structured points using schema
    points = await extraction_agent.extract(community)
    return points

# Reduce aggregates structured outputs
final = await reduce_agent.aggregate(all_points)
```

---

**Related**:
- [Semantic Patterns](02-semantic-patterns.md)
- [Architectural Patterns](03-architectural-patterns.md)
- [Code-Level Patterns](04-code-level-patterns.md)
