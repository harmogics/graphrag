# Query Execution: Влияние типов узлов на поиск и формулировку ответов

## Обзор

GraphRAG поддерживает три основных типа query execution, каждый из которых использует различные комбинации типов узлов для формирования ответа. Понимание того, как каждый тип узлов влияет на поиск, критично для эффективного использования системы.

## Типы поиска

```
┌────────────────────────────────────────────────────────────┐
│                    QUERY TYPES                             │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  1. GLOBAL SEARCH                                          │
│     • Широкие, обзорные вопросы                            │
│     • Использует: Community Reports (primary)              │
│     • Пример: "Каковы основные тренды в AI?"               │
│                                                            │
│  2. LOCAL SEARCH                                           │
│     • Детальные, специфичные вопросы                       │
│     • Использует: Entities, Text Units (primary)           │
│     • Пример: "Как GraphRAG извлекает сущности?"           │
│                                                            │
│  3. DRIFT SEARCH                                           │
│     • Исследовательский, многошаговый поиск                │
│     • Использует: Entities, Relationships (primary)        │
│     • Пример: "Исследуй связи между AI и этикой"           │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

## Global Search (Глобальный поиск)

### Концепция

Global Search отвечает на **широкие вопросы** путем анализа community reports — структурированных резюме тематических кластеров.

### Алгоритм

```
Query: "Каковы основные применения GraphRAG?"
    ↓
┌─────────────────────────────────────────────┐
│  STEP 1: Найти релевантные сообщества      │
│  • Semantic search по community embeddings  │
│  • Top-K communities (обычно K=5-10)        │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│  STEP 2: Извлечь Community Reports          │
│  • Получить full_content каждого отчета     │
│  • Title, Summary, Findings                 │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│  STEP 3: Map Phase (параллельно)            │
│  • Для каждого отчета:                      │
│    LLM отвечает на вопрос используя отчет   │
│  • Получаем N частичных ответов             │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│  STEP 4: Reduce Phase                       │
│  • LLM синтезирует финальный ответ          │
│  • Объединяет insights из всех отчетов      │
└──────────────────┬──────────────────────────┘
                   ↓
                Answer
```

### Используемые узлы

**Primary**: Community Reports
**Secondary**: Communities (для метаданных)

### Реализация

```python
async def global_search(
    query: str,
    communities: pd.DataFrame,
    community_reports: pd.DataFrame,
    community_embeddings: pd.DataFrame,
    llm: ChatModel
) -> dict:
    """
    Global Search implementation
    """
    # 1. Найти релевантные сообщества
    query_embedding = create_embedding(query)

    similarities = community_embeddings["embedding"].apply(
        lambda emb: cosine_similarity(query_embedding, emb)
    )

    top_community_ids = community_embeddings.loc[
        similarities.nlargest(5).index,
        "id"
    ]

    # 2. Извлечь отчеты
    relevant_reports = community_reports[
        community_reports["id"].isin(top_community_ids)
    ]

    # 3. Map Phase: LLM для каждого отчета
    map_responses = []

    map_prompt_template = """
    Answer the following question using ONLY the information in the report below.

    Question: {query}

    Report:
    {report}

    Answer:
    """

    for _, report in relevant_reports.iterrows():
        map_prompt = map_prompt_template.format(
            query=query,
            report=report["full_content"]
        )

        response = await llm(map_prompt)
        map_responses.append({
            "community": report["title"],
            "answer": response,
            "rating": report["rating"]
        })

    # 4. Reduce Phase: синтез финального ответа
    reduce_prompt = f"""
    Synthesize a comprehensive answer to the question based on the following
    partial answers from different community reports.

    Question: {query}

    Partial Answers:
    """

    for i, resp in enumerate(map_responses, 1):
        reduce_prompt += f"\n\n{i}. From '{resp['community']}':\n{resp['answer']}"

    reduce_prompt += "\n\nFinal Comprehensive Answer:"

    final_answer = await llm(reduce_prompt)

    return {
        "answer": final_answer,
        "sources": [r["community"] for r in map_responses],
        "num_communities": len(map_responses)
    }
```

### Влияние типов узлов

**Community Reports (критичны)**:
- Качество отчетов прямо влияет на качество ответа
- Title и Summary определяют релевантность
- Findings предоставляют детальную информацию

**Communities (метаданные)**:
- Level определяет уровень абстракции ответа
- Size влияет на полноту покрытия темы

### Оптимизация

```python
def optimize_global_search(
    query: str,
    communities: pd.DataFrame,
    community_reports: pd.DataFrame,
    strategy: str = "balanced"
) -> dict:
    """
    Оптимизированный global search с разными стратегиями
    """
    if strategy == "fast":
        # Использовать только level 1+ сообщества (более абстрактные)
        high_level_communities = communities[communities["level"] >= 1]
        num_communities = 3

    elif strategy == "comprehensive":
        # Использовать все уровни
        high_level_communities = communities
        num_communities = 10

    else:  # balanced
        # Микс level 0 и level 1
        high_level_communities = communities[communities["level"] <= 1]
        num_communities = 5

    # Выполнить search...
```

## Local Search (Локальный поиск)

### Концепция

Local Search отвечает на **специфичные вопросы** путем поиска релевантных entities и использования их text_units как контекста.

### Алгоритм

```
Query: "Как GraphRAG извлекает сущности из текста?"
    ↓
┌─────────────────────────────────────────────┐
│  STEP 1: Найти релевантные Entities         │
│  • Semantic search по entity embeddings     │
│  • Top-K entities (обычно K=20)             │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│  STEP 2: Расширить через Relationships      │
│  • Найти связанные entities (1-2 hops)      │
│  • Собрать relationships                    │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│  STEP 3: Собрать Text Units                 │
│  • Получить text_unit_ids из entities       │
│  • Извлечь text_units                       │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│  STEP 4: Ранжировать Text Units             │
│  • Semantic similarity к query              │
│  • Важность entities в text_unit            │
│  • Комбинированный score                    │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│  STEP 5: Построить контекст                 │
│  • Top-N text_units (обычно N=10)           │
│  • Добавить entity/relationship информацию  │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│  STEP 6: LLM генерирует ответ               │
│  • Single LLM call с контекстом             │
└──────────────────┬──────────────────────────┘
                   ↓
                Answer
```

### Используемые узлы

**Primary**: Entities, Text Units
**Secondary**: Relationships (для расширения)
**Tertiary**: Documents (для source attribution)

### Реализация

```python
async def local_search(
    query: str,
    entities: pd.DataFrame,
    relationships: pd.DataFrame,
    text_units: pd.DataFrame,
    entity_embeddings: pd.DataFrame,
    text_unit_embeddings: pd.DataFrame,
    llm: ChatModel
) -> dict:
    """
    Local Search implementation
    """
    # 1. Найти релевантные entities
    query_embedding = create_embedding(query)

    entity_similarities = entity_embeddings["embedding"].apply(
        lambda emb: cosine_similarity(query_embedding, emb)
    )

    top_entity_ids = entity_embeddings.loc[
        entity_similarities.nlargest(20).index,
        "id"
    ]

    relevant_entities = entities[entities["id"].isin(top_entity_ids)]

    # 2. Расширить через relationships
    expanded_entities = expand_via_relationships(
        relevant_entities,
        relationships,
        entities,
        max_hops=1
    )

    # 3. Собрать text_unit_ids
    text_unit_ids = set()
    for _, entity in expanded_entities.iterrows():
        text_unit_ids.update(entity["text_unit_ids"])

    context_text_units = text_units[text_units["id"].isin(text_unit_ids)]

    # 4. Ранжировать text_units
    text_unit_scores = {}

    for _, unit in context_text_units.iterrows():
        # Semantic score
        unit_embedding = text_unit_embeddings[
            text_unit_embeddings["id"] == unit["id"]
        ].iloc[0]["embedding"]

        semantic_score = cosine_similarity(query_embedding, unit_embedding)

        # Entity importance score
        unit_entities = entities[entities["id"].isin(unit["entity_ids"])]
        entity_score = unit_entities["node_degree"].mean() if len(unit_entities) > 0 else 0

        # Комбинированный score
        text_unit_scores[unit["id"]] = (
            0.7 * semantic_score +
            0.3 * (entity_score / entities["node_degree"].max())
        )

    # Сортировать
    sorted_unit_ids = sorted(
        text_unit_scores.keys(),
        key=lambda x: text_unit_scores[x],
        reverse=True
    )

    top_units = context_text_units[
        context_text_units["id"].isin(sorted_unit_ids[:10])
    ]

    # 5. Построить контекст
    context_parts = []

    context_parts.append("# Relevant Entities:")
    for _, entity in relevant_entities.head(5).iterrows():
        context_parts.append(
            f"- **{entity['title']}** ({entity['type']}): {entity['description']}"
        )

    context_parts.append("\n# Relevant Relationships:")
    relevant_rels = relationships[
        (relationships["source"].isin(relevant_entities["title"])) |
        (relationships["target"].isin(relevant_entities["title"]))
    ]

    for _, rel in relevant_rels.head(5).iterrows():
        context_parts.append(
            f"- {rel['source']} → {rel['target']}: {rel['description']}"
        )

    context_parts.append("\n# Relevant Text:")
    for i, (_, unit) in enumerate(top_units.iterrows(), 1):
        context_parts.append(f"\n[Excerpt {i}]\n{unit['text']}")

    context = "\n".join(context_parts)

    # 6. LLM генерирует ответ
    prompt = f"""
    Answer the following question using the provided context.

    Question: {query}

    Context:
    {context}

    Answer:
    """

    answer = await llm(prompt)

    return {
        "answer": answer,
        "num_entities": len(relevant_entities),
        "num_text_units": len(top_units),
        "sources": top_units["document_ids"].explode().unique().tolist()
    }
```

### Влияние типов узлов

**Entities (критичны)**:
- node_degree определяет важность
- type помогает в фильтрации
- description предоставляет контекст

**Text Units (критичны)**:
- Качество chunks влияет на качество контекста
- Overlap обеспечивает полноту информации

**Relationships (расширение)**:
- Помогают найти связанные концепты
- weight определяет силу связи

### Сравнение стратегий

```python
def compare_local_search_strategies(
    query: str,
    data: dict
) -> dict:
    """
    Сравнивает разные стратегии local search
    """
    results = {}

    # Стратегия 1: Entity-first
    results["entity_first"] = local_search(
        query,
        top_entities=20,
        expand_relationships=True,
        max_text_units=10
    )

    # Стратегия 2: Text-first
    results["text_first"] = local_search_text_based(
        query,
        top_text_units=20,
        extract_entities=True
    )

    # Стратегия 3: Hybrid
    results["hybrid"] = local_search_hybrid(
        query,
        entity_weight=0.5,
        text_weight=0.5
    )

    return results
```

## Drift Search (Исследовательский поиск)

### Концепция

Drift Search выполняет **исследовательский поиск** путем навигации по графу relationships и генерации follow-up вопросов.

### Алгоритм

```
Query: "Исследуй применения GraphRAG в медицине"
    ↓
┌─────────────────────────────────────────────┐
│  STEP 1: Найти стартовые Entities           │
│  • "GraphRAG", "medicine"                   │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│  STEP 2: Graph Walk (hop 1)                 │
│  • Следовать Relationships                  │
│  • Собрать связанные Entities               │
│  • Собрать Text Units                       │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│  STEP 3: LLM анализ + генерация вопросов    │
│  • Проанализировать найденную информацию    │
│  • Сгенерировать follow-up вопросы          │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│  STEP 4: Graph Walk (hop 2)                 │
│  • Исследовать новые направления            │
│  • Углубить поиск                           │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│  STEP 5: Финальный синтез                   │
│  • Объединить все findings                  │
│  • Предложить дальнейшие исследования       │
└──────────────────┬──────────────────────────┘
                   ↓
        Answer + Follow-up Questions
```

### Используемые узлы

**Primary**: Entities, Relationships
**Secondary**: Text Units (для контекста)
**Tertiary**: Communities (для тематической группировки)

### Реализация

```python
async def drift_search(
    query: str,
    entities: pd.DataFrame,
    relationships: pd.DataFrame,
    text_units: pd.DataFrame,
    llm: ChatModel,
    max_hops: int = 3
) -> dict:
    """
    Drift Search implementation
    """
    visited_entities = set()
    collected_info = {
        "entities": [],
        "relationships": [],
        "text_units": [],
        "insights": []
    }

    # 1. Найти стартовые entities
    current_entities = find_relevant_entities(query, entities, top_k=3)

    for hop in range(max_hops):
        # 2. Собрать информацию о текущих entities
        for entity in current_entities:
            if entity["id"] in visited_entities:
                continue

            visited_entities.add(entity["id"])
            collected_info["entities"].append(entity)

            # Получить text_units
            entity_units = text_units[
                text_units["entity_ids"].apply(
                    lambda ids: entity["id"] in ids
                )
            ]
            collected_info["text_units"].extend(
                entity_units.to_dict('records')
            )

            # Получить relationships
            entity_rels = relationships[
                (relationships["source"] == entity["title"]) |
                (relationships["target"] == entity["title"])
            ]
            collected_info["relationships"].extend(
                entity_rels.to_dict('records')
            )

        # 3. LLM анализ + генерация follow-up вопросов
        analysis_context = build_drift_context(collected_info)

        analysis_prompt = f"""
        Based on the information gathered so far about: {query}

        Information:
        {analysis_context}

        Provide:
        1. Key insights discovered
        2. 3 follow-up questions to explore further

        Format:
        Insights: [list]
        Questions: [list]
        """

        analysis = await llm(analysis_prompt)
        parsed_analysis = parse_drift_analysis(analysis)

        collected_info["insights"].append(parsed_analysis["insights"])

        # 4. Использовать follow-up вопросы для следующего hop
        if hop < max_hops - 1:
            next_query = parsed_analysis["questions"][0]  # Первый вопрос
            current_entities = find_relevant_entities(
                next_query,
                entities,
                top_k=3
            )

    # 5. Финальный синтез
    final_prompt = f"""
    Synthesize a comprehensive answer to: {query}

    Based on the exploration through the knowledge graph:

    Insights discovered:
    {format_insights(collected_info["insights"])}

    Key entities: {[e["title"] for e in collected_info["entities"][:10]]}
    Key relationships: {len(collected_info["relationships"])}

    Provide:
    1. Comprehensive answer
    2. 3 suggestions for further research
    """

    final_answer = await llm(final_prompt)

    return {
        "answer": final_answer,
        "num_hops": max_hops,
        "entities_visited": len(visited_entities),
        "relationships_explored": len(collected_info["relationships"]),
        "follow_up_questions": parsed_analysis["questions"]
    }
```

### Влияние типов узлов

**Entities (навигация)**:
- node_degree определяет "проходимость" графа
- type помогает направлять exploration

**Relationships (критичны)**:
- description предоставляет контекст связи
- weight определяет приоритет исследования

**Text Units (контекст)**:
- Предоставляют детали для каждого hop

## Сравнение типов поиска

```
┌──────────────┬─────────────┬─────────────┬─────────────────┐
│ Характеристика│ Global     │ Local       │ Drift           │
├──────────────┼─────────────┼─────────────┼─────────────────┤
│ Тип вопроса  │ Широкий     │ Специфичный │ Исследовательский│
│ Основные узлы│ Communities │ Entities    │ Relationships   │
│ LLM calls    │ Много (M+R) │ Один        │ Много (итеративно)│
│ Скорость     │ Средняя     │ Быстрая     │ Медленная       │
│ Полнота      │ Высокая     │ Средняя     │ Очень высокая   │
│ Глубина      │ Обзорная    │ Детальная   │ Исследовательская│
└──────────────┴─────────────┴─────────────┴─────────────────┘
```

### Когда использовать каждый тип

```python
def select_search_type(query: str) -> str:
    """
    Автоматически выбирает тип поиска на основе query
    """
    # Широкие вопросы → Global
    global_patterns = [
        r"какие основные",
        r"общий обзор",
        r"главные тренды",
        r"в целом",
        r"overall",
        r"main",
        r"general"
    ]

    # Специфичные вопросы → Local
    local_patterns = [
        r"как именно",
        r"что такое",
        r"определение",
        r"explain",
        r"describe",
        r"what is"
    ]

    # Исследовательские → Drift
    drift_patterns = [
        r"исследуй",
        r"изучи связи",
        r"как связаны",
        r"explore",
        r"investigate",
        r"how are.*related"
    ]

    for pattern in global_patterns:
        if re.search(pattern, query, re.IGNORECASE):
            return "global"

    for pattern in local_patterns:
        if re.search(pattern, query, re.IGNORECASE):
            return "local"

    for pattern in drift_patterns:
        if re.search(pattern, query, re.IGNORECASE):
            return "drift"

    # По умолчанию - local (наиболее универсальный)
    return "local"
```

## Hybrid Search

### Комбинирование подходов

```python
async def hybrid_search(
    query: str,
    all_data: dict,
    llm: ChatModel
) -> dict:
    """
    Hybrid search: комбинирует Global и Local
    """
    # 1. Выполнить оба поиска параллельно
    global_result, local_result = await asyncio.gather(
        global_search(query, all_data, llm),
        local_search(query, all_data, llm)
    )

    # 2. LLM синтезирует комбинированный ответ
    synthesis_prompt = f"""
    Synthesize a comprehensive answer combining:

    Global perspective (high-level):
    {global_result["answer"]}

    Local details (specific):
    {local_result["answer"]}

    Question: {query}

    Provide a unified answer that combines both perspectives.
    """

    combined_answer = await llm(synthesis_prompt)

    return {
        "answer": combined_answer,
        "global_sources": global_result["sources"],
        "local_sources": local_result["sources"],
        "search_types": ["global", "local"]
    }
```

## Оптимизация Query Execution

### Кэширование results

```python
class QueryCache:
    """Кэш для query results"""

    def __init__(self):
        self.cache = {}

    def get_cache_key(self, query: str, search_type: str) -> str:
        """Создает cache key"""
        return hashlib.sha256(
            f"{query}_{search_type}".encode()
        ).hexdigest()

    async def get_or_compute(
        self,
        query: str,
        search_type: str,
        compute_fn
    ):
        """Получает из кэша или вычисляет"""
        cache_key = self.get_cache_key(query, search_type)

        if cache_key in self.cache:
            return self.cache[cache_key]

        result = await compute_fn()
        self.cache[cache_key] = result

        return result
```

### Batch Query Processing

```python
async def process_queries_batch(
    queries: list[str],
    all_data: dict,
    llm: ChatModel
) -> list[dict]:
    """
    Обрабатывает несколько queries параллельно
    """
    # Группировать по типу поиска
    global_queries = []
    local_queries = []

    for query in queries:
        search_type = select_search_type(query)
        if search_type == "global":
            global_queries.append(query)
        else:
            local_queries.append(query)

    # Параллельно обработать каждую группу
    global_tasks = [
        global_search(q, all_data, llm)
        for q in global_queries
    ]

    local_tasks = [
        local_search(q, all_data, llm)
        for q in local_queries
    ]

    results = await asyncio.gather(*global_tasks, *local_tasks)

    return results
```

## Best Practices

### 1. Выбор правильного типа поиска

```python
# Рекомендации:
SEARCH_TYPE_GUIDE = {
    "Обзорные вопросы": "global",
    "Детальные вопросы": "local",
    "Исследование связей": "drift",
    "Факт-чекинг": "local",
    "Тематический анализ": "global",
    "Навигация по графу": "drift"
}
```

### 2. Настройка параметров

```python
# Для разных use cases
SEARCH_PARAMETERS = {
    "fast": {
        "top_entities": 10,
        "top_text_units": 5,
        "max_hops": 1
    },
    "balanced": {
        "top_entities": 20,
        "top_text_units": 10,
        "max_hops": 2
    },
    "comprehensive": {
        "top_entities": 50,
        "top_text_units": 20,
        "max_hops": 3
    }
}
```

### 3. Мониторинг quality

```python
def evaluate_search_quality(
    query: str,
    result: dict,
    ground_truth: str | None = None
) -> dict:
    """
    Оценивает качество результата поиска
    """
    metrics = {
        "answer_length": len(result["answer"]),
        "num_sources": len(result.get("sources", [])),
        "response_time": result.get("time", 0)
    }

    if ground_truth:
        # Semantic similarity с ground truth
        metrics["accuracy"] = compute_similarity(
            result["answer"],
            ground_truth
        )

    return metrics
```

## Заключение

Типы узлов в GraphRAG организованы в иерархическую структуру, где каждый уровень служит специфической цели в query execution:

- **Documents**: Источник информации
- **Text Units**: Контекстные фрагменты для Local Search
- **Entities**: Ключевые объекты для навигации
- **Relationships**: Связи для graph walk
- **Communities**: Тематические кластеры для Global Search
- **Community Reports**: Высокоуровневые резюме для обзорных вопросов

Понимание роли каждого типа узлов позволяет:
1. Выбрать правильный тип поиска для вопроса
2. Оптимизировать производительность
3. Настроить качество ответов
4. Эффективно использовать граф знаний

## Следующие разделы

- **[← Entity Nodes](03-entity-nodes.md)**
- **[Node Embeddings →](10-node-embeddings.md)**
- **[Node Interactions →](08-node-interactions.md)**
