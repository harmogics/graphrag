# Генерация отчетов о сообществах (Community Reports)

## Обзор этапа

**Community Reports Generation** — это процесс создания человекочитаемых структурированных отчетов о каждом сообществе в графе знаний. Этот этап использует **Community Report LLM Agent** для анализа узлов, ребер и контекста сообщества и генерации comprehensive summary.

Отчеты служат высокоуровневым представлением тематических областей и используются для information retrieval и question answering.

## Архитектура компонента

### Ключевые файлы
- **Workflow**: `graphrag/index/workflows/create_community_reports.py`
- **Summarization**: `graphrag/index/operations/summarize_communities/summarize_communities.py`
- **Context Builder**: `graphrag/index/operations/summarize_communities/graph_context/context_builder.py`
- **Промпт**: `graphrag/prompts/index/community_report.py`
- **Конфигурация**: `graphrag/config/models/community_reports_config.py`

## Процесс генерации отчетов

```
INPUT: communities.parquet + entities.parquet + relationships.parquet
    ↓
┌──────────────────────────────────────────────┐
│  1. Подготовка узлов и ребер                 │
│     - Заполнение пропущенных описаний        │
│     - Форматирование данных                  │
└──────────────────────────────────────────────┘
    ↓
┌──────────────────────────────────────────────┐
│  2. Для каждого сообщества:                  │
│     ┌──────────────────────────────────────┐ │
│     │ a. Построение контекста сообщества  │ │
│     │    - Nodes (entities)                │ │
│     │    - Edges (relationships)           │ │
│     │    - Claims (опционально)            │ │
│     └──────────────────────────────────────┘ │
│     ┌──────────────────────────────────────┐ │
│     │ b. Формирование промпта              │ │
│     │    - Контекст в CSV-подобном формате │ │
│     └──────────────────────────────────────┘ │
│     ┌──────────────────────────────────────┐ │
│     │ c. LLM call (Community Report Agent) │ │
│     └──────────────────────────────────────┘ │
│     ┌──────────────────────────────────────┐ │
│     │ d. Парсинг JSON результата           │ │
│     │    - Title, Summary, Rating          │ │
│     │    - Findings                        │ │
│     └──────────────────────────────────────┘ │
└──────────────────────────────────────────────┘
    ↓
┌──────────────────────────────────────────────┐
│  3. Сохранение отчетов                       │
└──────────────────────────────────────────────┘
    ↓
OUTPUT: community_reports.parquet
```

## Роль Community Report Agent

### Функция агента

**Community Report Agent** — это специализированный LLM-агент, который:
- Анализирует структуру сообщества (entities + relationships)
- Синтезирует ключевые insights
- Создает структурированный отчет в JSON формате
- Оценивает важность сообщества (impact rating)

### Конфигурация агента

```python
class CommunityReportsConfig:
    # Модель LLM для генерации отчетов
    model_id: str = DEFAULT_CHAT_MODEL_ID  # Обычно GPT-4

    # Путь к кастомному промпту (опционально)
    prompt: str | None = None

    # Максимальная длина отчета (в токенах)
    max_report_length: int = 2000

    # Максимальная длина входного контекста
    max_input_length: int = 8000

    # Override стратегии
    strategy: dict | None = None

    # Модель токенизации
    encoding_model: str = "cl100k_base"
```

**Файл**: `graphrag/config/models/community_reports_config.py:17`

### Промпт агента

**Полный промпт** (файл: `graphrag/prompts/index/community_report.py:5`):

```
You are an AI assistant that helps a human analyst to perform general
information discovery. Information discovery is the process of identifying
and assessing relevant information associated with certain entities within
a network.

# Goal
Write a comprehensive report of a community, given a list of entities that
belong to the community as well as their relationships and optional associated
claims. The report will be used to inform decision-makers about information
associated with the community and their potential impact.

# Report Structure

The report should include the following sections:

- TITLE: community's name that represents its key entities - title should be
  short but specific. When possible, include representative named entities in
  the title.

- SUMMARY: An executive summary of the community's overall structure, how its
  entities are related to each other, and significant information associated
  with its entities.

- IMPACT SEVERITY RATING: a float score between 0-10 that represents the
  severity of IMPACT posed by entities within the community. IMPACT is the
  scored importance of a community.

- RATING EXPLANATION: Give a single sentence explanation of the IMPACT
  severity rating.

- DETAILED FINDINGS: A list of 5-10 key insights about the community. Each
  insight should have a short summary followed by multiple paragraphs of
  explanatory text grounded according to the grounding rules below. Be
  comprehensive.

Return output as a well-formed JSON-formatted string with the following format:
{
    "title": <report_title>,
    "summary": <executive_summary>,
    "rating": <impact_severity_rating>,
    "rating_explanation": <rating_explanation>,
    "findings": [
        {
            "summary": <insight_1_summary>,
            "explanation": <insight_1_explanation>
        },
        ...
    ]
}

# Grounding Rules

Points supported by data should list their data references as follows:

"This is an example sentence supported by multiple data references
[Data: <dataset name> (record ids); <dataset name> (record ids)]."

Do not list more than 5 record ids in a single reference. Instead, list the
top 5 most relevant record ids and add "+more" to indicate that there are more.

For example:
"Person X is the owner of Company Y and subject to many allegations of
wrongdoing [Data: Reports (1), Entities (5, 7); Relationships (23);
Claims (7, 2, 34, 64, 46, +more)]."

Do not include information where the supporting evidence for it is not provided.
```

## Построение контекста сообщества

### Community Context Structure

Контекст сообщества состоит из трех основных компонентов:

```python
class CommunityContext:
    entities: pd.DataFrame      # Сущности в сообществе
    relationships: pd.DataFrame # Отношения внутри сообщества
    claims: pd.DataFrame | None # Claims (опционально)
```

### Процесс построения контекста

```python
def build_community_context(
    community: Community,
    entities: pd.DataFrame,
    relationships: pd.DataFrame,
    claims: pd.DataFrame | None = None
) -> str:
    """
    Строит текстовый контекст для сообщества в CSV-подобном формате

    Returns:
        Форматированная строка с данными сообщества
    """
    context_parts = []

    # 1. Entities section
    context_parts.append("Entities")
    context_parts.append("")
    context_parts.append("id,entity,description")

    community_entities = entities[
        entities["id"].isin(community["entity_ids"])
    ]

    for _, entity in community_entities.iterrows():
        # Очистка description от переносов и кавычек
        description = clean_description(entity["description"])
        context_parts.append(
            f'{entity["human_readable_id"]},{entity["title"]},{description}'
        )

    # 2. Relationships section
    context_parts.append("")
    context_parts.append("Relationships")
    context_parts.append("")
    context_parts.append("id,source,target,description")

    community_relationships = relationships[
        relationships["id"].isin(community["relationship_ids"])
    ]

    for _, rel in community_relationships.iterrows():
        description = clean_description(rel["description"])
        context_parts.append(
            f'{rel["human_readable_id"]},{rel["source"]},{rel["target"]},{description}'
        )

    # 3. Claims section (опционально)
    if claims is not None:
        context_parts.append("")
        context_parts.append("Claims")
        context_parts.append("")
        context_parts.append("id,entity,claim,description")
        # ... добавить claims

    return "\n".join(context_parts)
```

**Файл**: `graphrag/index/operations/summarize_communities/graph_context/context_builder.py:45`

### Пример контекста

```
Entities

id,entity,description
entity_0034,GraphRAG,GraphRAG is a knowledge graph construction framework that uses LLMs to extract entities and relationships from unstructured text
entity_0045,Knowledge Graph,A knowledge graph is a structured representation of information as nodes and edges
entity_0067,Entity Extraction,Entity extraction is the process of identifying named entities in text using NLP or LLM techniques

Relationships

id,source,target,description
rel_0234,GraphRAG,Knowledge Graph,GraphRAG produces knowledge graphs as its primary output
rel_0245,GraphRAG,Entity Extraction,GraphRAG uses entity extraction as a core component of its pipeline
rel_0267,Entity Extraction,Knowledge Graph,Entity extraction is the first step in building a knowledge graph
```

### Управление размером контекста

Если контекст превышает `max_input_length`, применяется truncation:

```python
def truncate_context(
    context: str,
    max_tokens: int,
    encoding_model: str
) -> str:
    """
    Обрезает контекст до max_tokens, сохраняя целостность записей
    """
    import tiktoken

    encoding = tiktoken.get_encoding(encoding_model)
    tokens = encoding.encode(context)

    if len(tokens) <= max_tokens:
        return context

    # Обрезать по границам записей (не посередине entity/relationship)
    truncated_tokens = tokens[:max_tokens]
    truncated_text = encoding.decode(truncated_tokens)

    # Найти последнюю полную запись
    last_newline = truncated_text.rfind('\n')
    if last_newline > 0:
        truncated_text = truncated_text[:last_newline]

    return truncated_text
```

## Генерация отчета (LLM Call)

### Процесс генерации

```python
async def generate_community_report(
    community_context: str,
    llm_agent: ChatModel,
    prompt_template: str,
    cache: PipelineCache
) -> CommunityReport:
    """
    Генерирует отчет о сообществе с помощью LLM

    Args:
        community_context: Контекст сообщества (entities + relationships)
        llm_agent: LLM модель
        prompt_template: Шаблон промпта
        cache: Кэш для LLM вызовов

    Returns:
        Структурированный отчет
    """
    # Заполнить промпт контекстом
    prompt = prompt_template.format(input_text=community_context)

    # Проверить кэш
    cache_key = hashlib.sha256(prompt.encode()).hexdigest()
    cached_response = await cache.get(cache_key)

    if cached_response:
        response_text = cached_response
    else:
        # LLM call
        response = await llm_agent(prompt)
        response_text = response.content

        # Сохранить в кэш
        await cache.set(cache_key, response_text)

    # Парсинг JSON
    report = parse_community_report(response_text)

    return report
```

**Файл**: `graphrag/index/operations/summarize_communities/summarize_communities.py:85`

## Структура отчета

### CommunityReport Data Model

```python
class CommunityReport:
    # Базовые поля
    id: str                     # ID сообщества
    human_readable_id: str      # "community_0001"
    community: int              # Номер сообщества
    level: int                  # Уровень иерархии
    parent: str | None          # Parent community ID
    children: list[str]         # Child community IDs

    # Генерируемые LLM поля
    title: str                  # Название сообщества
    summary: str                # Executive summary
    rating: float               # Impact severity rating (0-10)
    rating_explanation: str     # Объяснение рейтинга
    findings: list[Finding]     # Ключевые insights

    # Метаданные
    full_content: str           # Полный текст отчета
    full_content_json: str      # JSON representation
    period: str | None          # Период
    size: int                   # Количество сущностей
```

### Finding Structure

```python
class Finding:
    summary: str        # Краткое резюме insight
    explanation: str    # Детальное объяснение с data references
```

### Пример отчета

```json
{
    "id": "community_0012",
    "human_readable_id": "community_0012",
    "level": 0,
    "title": "GraphRAG and Knowledge Graph Construction",
    "summary": "This community focuses on GraphRAG, a framework for constructing knowledge graphs from unstructured text using large language models. The community includes entities related to entity extraction, knowledge graphs, and LLM-based processing.",
    "rating": 8.5,
    "rating_explanation": "This community has high impact due to its focus on cutting-edge AI technology for knowledge representation.",
    "findings": [
        {
            "summary": "GraphRAG as the central entity",
            "explanation": "GraphRAG is the primary entity in this community, serving as a framework for knowledge graph construction. It integrates multiple components including entity extraction, relationship detection, and community detection [Data: Entities (34), Relationships (234, 245, 267)]."
        },
        {
            "summary": "Entity extraction as a core component",
            "explanation": "Entity extraction is a fundamental process within GraphRAG, enabling the identification of key concepts from unstructured text. This process utilizes both NLP techniques and large language models to achieve high accuracy [Data: Entities (67), Relationships (245)]."
        },
        {
            "summary": "Knowledge graphs as output",
            "explanation": "The primary output of GraphRAG is a structured knowledge graph, which represents information as interconnected nodes and edges. This representation enables efficient querying and reasoning over the extracted information [Data: Entities (45), Relationships (234, 267)]."
        }
    ],
    "full_content": "# GraphRAG and Knowledge Graph Construction\n\n...",
    "size": 5
}
```

## Парсинг LLM результата

### JSON Parsing

```python
def parse_community_report(response_text: str) -> dict:
    """
    Парсит JSON ответ от LLM в структурированный отчет

    Args:
        response_text: Raw text response от LLM

    Returns:
        Словарь с полями отчета
    """
    import json
    import re

    # Извлечь JSON из markdown code block, если есть
    json_match = re.search(r'```json\s*(\{.*?\})\s*```', response_text, re.DOTALL)
    if json_match:
        json_text = json_match.group(1)
    else:
        # Попробовать найти первый JSON объект
        json_match = re.search(r'\{.*\}', response_text, re.DOTALL)
        if json_match:
            json_text = json_match.group(0)
        else:
            raise ValueError("No JSON found in LLM response")

    # Парсинг JSON
    try:
        report_data = json.loads(json_text)
    except json.JSONDecodeError as e:
        # Попробовать исправить распространенные ошибки
        json_text = fix_json_formatting(json_text)
        report_data = json.loads(json_text)

    # Валидация структуры
    required_fields = ["title", "summary", "rating", "rating_explanation", "findings"]
    for field in required_fields:
        if field not in report_data:
            raise ValueError(f"Missing required field: {field}")

    return report_data
```

### Обработка ошибок парсинга

```python
def fix_json_formatting(json_text: str) -> str:
    """
    Исправляет распространенные ошибки форматирования JSON от LLM
    """
    # Удалить trailing commas
    json_text = re.sub(r',\s*}', '}', json_text)
    json_text = re.sub(r',\s*]', ']', json_text)

    # Экранировать неэкранированные кавычки в строках
    # (упрощенная версия, в реальности сложнее)

    # Заменить одинарные кавычки на двойные (если нужно)
    # json_text = json_text.replace("'", '"')

    return json_text
```

## Иерархические отчеты (Multi-level)

### Level-based Context

Для сообществ высоких уровней контекст строится рекурсивно:

```python
async def build_level_context(
    community: Community,
    level: int,
    child_reports: list[CommunityReport]
) -> str:
    """
    Строит контекст для сообщества высокого уровня на основе
    дочерних отчетов

    Args:
        community: Сообщество высокого уровня
        level: Уровень иерархии
        child_reports: Отчеты дочерних сообществ

    Returns:
        Агрегированный контекст
    """
    context_parts = []

    context_parts.append(f"# Level {level} Community")
    context_parts.append(f"Size: {community['size']} entities")
    context_parts.append("")

    # Агрегировать insights из дочерних отчетов
    context_parts.append("## Key Themes from Sub-communities")
    context_parts.append("")

    for child_report in child_reports:
        context_parts.append(f"### {child_report['title']}")
        context_parts.append(child_report['summary'])
        context_parts.append("")

        # Топ findings
        for finding in child_report['findings'][:3]:  # Первые 3
            context_parts.append(f"- {finding['summary']}")

        context_parts.append("")

    return "\n".join(context_parts)
```

### Стратегия для разных уровней

```python
# Level 0: Детальный анализ entities и relationships
level_0_prompt = COMMUNITY_REPORT_PROMPT  # Полный промпт

# Level 1+: Synthesis из дочерних отчетов
level_high_prompt = """
You are synthesizing a high-level report based on sub-community reports.

# Goal
Create a comprehensive overview that captures the main themes and insights
from the sub-communities, identifying common patterns and overarching trends.

# Input
{input_text}

# Output
Generate a JSON report with the same structure as before, but focus on:
- Identifying common themes across sub-communities
- Highlighting connections between different sub-communities
- Providing a higher-level perspective
"""
```

## Финализация отчетов

### Создание full_content

```python
def create_full_content(report: dict) -> str:
    """
    Создает full_content текст из структурированного отчета

    Returns:
        Markdown-форматированный текст отчета
    """
    content_parts = []

    # Title
    content_parts.append(f"# {report['title']}")
    content_parts.append("")

    # Summary
    content_parts.append("## Summary")
    content_parts.append(report['summary'])
    content_parts.append("")

    # Rating
    content_parts.append("## Impact Rating")
    content_parts.append(f"**Rating**: {report['rating']}/10")
    content_parts.append(f"**Explanation**: {report['rating_explanation']}")
    content_parts.append("")

    # Findings
    content_parts.append("## Key Findings")
    content_parts.append("")

    for i, finding in enumerate(report['findings'], 1):
        content_parts.append(f"### {i}. {finding['summary']}")
        content_parts.append(finding['explanation'])
        content_parts.append("")

    return "\n".join(content_parts)
```

### Финальная схема community_reports.parquet

```python
COMMUNITY_REPORTS_FINAL_COLUMNS = [
    "id",                   # str: ID сообщества
    "human_readable_id",    # str: "community_0001"
    "community",            # int: номер сообщества
    "level",                # int: уровень иерархии
    "parent",               # str | None: parent ID
    "children",             # list[str]: child IDs
    "title",                # str: название сообщества (LLM-generated)
    "summary",              # str: executive summary (LLM-generated)
    "full_content",         # str: полный markdown текст
    "rank",                 # float: ранжирование для поиска
    "rating_explanation",   # str: объяснение рейтинга
    "findings",             # list[dict]: ключевые insights
    "full_content_json",    # str: JSON representation
    "period",               # str | None: период
    "size",                 # int: количество сущностей
]
```

**Файл схемы**: `graphrag/data_model/schemas.py:105`

## Использование отчетов в Query

### Global Search

Отчеты используются для ответов на глобальные вопросы:

```python
async def global_search(
    query: str,
    community_reports: pd.DataFrame,
    embeddings: pd.DataFrame
) -> str:
    """
    Отвечает на вопрос, используя community reports
    """
    # 1. Найти релевантные сообщества по embeddings
    relevant_communities = find_relevant_communities(
        query,
        embeddings
    )

    # 2. Извлечь отчеты этих сообществ
    reports = community_reports[
        community_reports["id"].isin(relevant_communities)
    ]

    # 3. Map: Для каждого отчета, ответить на вопрос
    partial_answers = []
    for _, report in reports.iterrows():
        context = report["full_content"]
        answer = await llm_call(f"Question: {query}\nContext: {context}")
        partial_answers.append(answer)

    # 4. Reduce: Синтезировать финальный ответ
    final_answer = await llm_call(
        f"Synthesize these answers into one: {partial_answers}"
    )

    return final_answer
```

### Local Search

Отчеты обеспечивают контекст для детальных вопросов:

```python
async def local_search(
    query: str,
    entities: list[str],
    community_reports: pd.DataFrame
) -> str:
    """
    Детальный поиск с использованием community context
    """
    # Найти сообщества, содержащие entities
    relevant_communities = community_reports[
        community_reports["entity_ids"].apply(
            lambda x: any(e in x for e in entities)
        )
    ]

    # Использовать отчеты как дополнительный контекст
    community_context = "\n\n".join(relevant_communities["summary"])

    # LLM query с контекстом
    answer = await llm_call(f"""
    Question: {query}

    Community Context:
    {community_context}

    Answer:
    """)

    return answer
```

## Оптимизация и Best Practices

### Балансировка детальности

```python
# Для level 0: max_report_length = 2000 (детальные отчеты)
# Для level 1+: max_report_length = 1500 (более высокоуровневые)

config = CommunityReportsConfig(
    max_report_length=lambda level: 2000 if level == 0 else 1500
)
```

### Кэширование отчетов

Все LLM-вызовы кэшируются, поэтому:
- Повторное выполнение не создает расходов
- Изменения в данных (новые entities) требуют invalidation кэша

### Качество отчетов

```python
def evaluate_report_quality(report: dict) -> dict:
    """
    Оценивает качество отчета
    """
    return {
        "has_title": len(report.get("title", "")) > 0,
        "has_summary": len(report.get("summary", "")) > 50,
        "num_findings": len(report.get("findings", [])),
        "has_data_references": check_data_references(report),
        "rating_in_range": 0 <= report.get("rating", -1) <= 10
    }
```

## Следующий этап

После генерации community reports:
- **generate_text_embeddings**: Создание векторных представлений для всех компонентов, включая отчеты

→ [Переход к embeddings и векторизации](06-embeddings-vectorization.md)
