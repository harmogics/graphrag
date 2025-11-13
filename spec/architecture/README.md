# GraphRAG Architecture Patterns

## Обзор

Этот раздел описывает **основные архитектурные, агентские и семантические паттерны**, используемые в реализации GraphRAG. Документация включает как высокоуровневые архитектурные решения, так и конкретные примеры кода.

## Структура документации

### [01. Agent Patterns](01-agent-patterns.md)
Паттерны, связанные с агентами и LLM-based операциями:
- **LLM Agent Pattern** - базовый паттерн для взаимодействия с LLM
- **Gleaning Pattern** - итеративное уточнение результатов извлечения
- **Map-Reduce Agent Pattern** - распределенная обработка для Global Search
- **Multi-hop Agent Pattern** - многоступенчатая навигация для Drift Search
- **Extraction Agent Pattern** - извлечение структурированных данных из текста

### [02. Semantic Patterns](02-semantic-patterns.md)
Паттерны семантической обработки информации:
- **Chunking Strategy Pattern** - стратегии разбиения текста
- **Embedding Pattern** - преобразование текста в векторное пространство
- **Graph Construction Pattern** - построение графа знаний из текста
- **Context Building Pattern** - формирование семантического контекста
- **Semantic Retrieval Pattern** - поиск по семантическому сходству

### [03. Architectural Patterns](03-architectural-patterns.md)
Классические архитектурные паттерны:
- **Strategy Pattern** - различные стратегии поиска
- **Template Method Pattern** - базовые классы с шаблонными методами
- **Builder Pattern** - построение сложных объектов
- **Factory Pattern** - создание семейств связанных объектов
- **Observer/Callback Pattern** - мониторинг выполнения
- **Adapter Pattern** - адаптация интерфейсов

### [04. Code-Level Patterns](04-code-level-patterns.md)
Паттерны на уровне исходного кода:
- **Dataclass Pattern** - типизированные модели данных
- **Protocol Pattern** - structural typing для интерфейсов
- **Async Pattern** - асинхронное выполнение
- **Generator Pattern** - ленивое вычисление и streaming
- **Dependency Injection** - внедрение зависимостей

## Иерархия паттернов

```
Architectural Level
├── Agent Patterns (LLM-based operations)
│   ├── LLM Agent Pattern
│   ├── Gleaning Pattern
│   ├── Map-Reduce Pattern
│   └── Multi-hop Pattern
│
├── Semantic Patterns (Information processing)
│   ├── Chunking Strategy
│   ├── Embedding Pattern
│   ├── Graph Construction
│   └── Context Building
│
├── Architectural Patterns (Design patterns)
│   ├── Strategy Pattern
│   ├── Template Method
│   ├── Builder Pattern
│   ├── Factory Pattern
│   └── Observer/Callback
│
└── Code-Level Patterns (Implementation details)
    ├── Dataclass Pattern
    ├── Protocol Pattern
    ├── Async Pattern
    └── Generator Pattern
```

## Ключевые принципы архитектуры

### 1. Separation of Concerns

```python
# Query execution отделен от context building
class LocalSearch(BaseSearch[LocalContextBuilder]):
    def __init__(self, model: ChatModel, context_builder: LocalContextBuilder):
        self.model = model
        self.context_builder = context_builder

    async def search(self, query: str) -> SearchResult:
        # Context building - отдельный concern
        context = self.context_builder.build_context(query)

        # Answer generation - отдельный concern
        answer = await self.model.achat(context)

        return SearchResult(...)
```

### 2. Strategy Pattern for Extensibility

```python
# Различные стратегии search легко заменяемы
class BaseSearch(ABC, Generic[T]):
    @abstractmethod
    async def search(self, query: str) -> SearchResult:
        """Each strategy implements search differently."""

# Implementations:
# - GlobalSearch: Map-Reduce across communities
# - LocalSearch: Entity-focused detailed search
# - DRIFTSearch: Multi-hop graph traversal
# - BasicSearch: Simple text similarity
```

### 3. Builder Pattern for Complexity Management

```python
# Context building - сложный процесс построения контекста
class LocalContextBuilder(ABC):
    @abstractmethod
    def build_context(self, query: str, **kwargs) -> ContextBuilderResult:
        """Build context step by step."""

class LocalSearchMixedContext(LocalContextBuilder):
    def build_context(self, query: str, **kwargs) -> ContextBuilderResult:
        # Step 1: Extract entities from query
        entities = self._extract_entities(query)

        # Step 2: Retrieve related context
        text_units = self._get_text_units(entities)
        relationships = self._get_relationships(entities)

        # Step 3: Build final context
        context = self._build_context_string(entities, text_units, relationships)

        return ContextBuilderResult(context_chunks=context, ...)
```

### 4. Callback Pattern for Observability

```python
# Наблюдение за выполнением pipeline
class WorkflowCallbacks(Protocol):
    def workflow_start(self, name: str) -> None: ...
    def workflow_end(self, name: str) -> None: ...
    def error(self, message: str, cause: BaseException) -> None: ...

# Usage
callbacks.workflow_start("extract_graph")
try:
    result = await extractor(texts)
    callbacks.workflow_end("extract_graph")
except Exception as e:
    callbacks.error("Extraction failed", cause=e)
```

## Примеры применения паттернов

### Пример 1: Query Execution (Strategy + Factory)

```python
# Factory создает нужную стратегию search
def get_local_search_engine(
    config: GraphRagConfig,
    entities: list[Entity],
    ...
) -> LocalSearch:
    # Factory создает все зависимости
    chat_model = ModelManager().get_or_create_chat_model(...)
    embedding_model = ModelManager().get_or_create_embedding_model(...)

    # Builder создает context builder
    context_builder = LocalSearchMixedContext(
        entities=entities,
        text_embedder=embedding_model,
        ...
    )

    # Strategy pattern - создается конкретная стратегия
    return LocalSearch(
        model=chat_model,
        context_builder=context_builder,
        ...
    )

# Client code - использует strategy через единый интерфейс
search_engine = get_local_search_engine(...)
result = await search_engine.search("What is GraphRAG?")
```

### Пример 2: Entity Extraction (Agent + Gleaning)

```python
# LLM Agent с gleaning pattern
class GraphExtractor:
    def __init__(self, model: ChatModel, max_gleanings: int = 1):
        self._model = model
        self._max_gleanings = max_gleanings

    async def __call__(self, texts: list[str]) -> GraphExtractionResult:
        # Initial extraction
        result = await self._extract_entities(texts)

        # Gleaning - iterative refinement
        for i in range(self._max_gleanings):
            # Ask LLM if more entities exist
            has_more = await self._check_for_more_entities(result)
            if not has_more:
                break

            # Extract additional entities
            additional = await self._extract_additional_entities(result)
            result = self._merge_results(result, additional)

        return result
```

### Пример 3: Context Building (Builder + Semantic Patterns)

```python
# Builder с семантическими паттернами
class LocalSearchMixedContext(LocalContextBuilder):
    def build_context(self, query: str, **kwargs) -> ContextBuilderResult:
        # Semantic Pattern 1: Entity Extraction
        entities = self._extract_entities_from_query(query)

        # Semantic Pattern 2: Semantic Retrieval via Embeddings
        query_embedding = self.text_embedder.embed(query)
        similar_entities = self.entity_text_embeddings.similarity_search(
            query_embedding, k=30
        )

        # Semantic Pattern 3: Graph Navigation
        relationships = self._get_entity_relationships(similar_entities)

        # Semantic Pattern 4: Context Construction
        context = self._format_context(
            entities=similar_entities,
            relationships=relationships,
            text_units=self._get_text_units(similar_entities)
        )

        return ContextBuilderResult(context_chunks=context, ...)
```

## Влияние паттернов на качество системы

### Maintainability

- **Strategy Pattern** → легко добавить новый тип search (например, Hybrid Search)
- **Builder Pattern** → легко модифицировать процесс построения контекста
- **Factory Pattern** → централизованное создание объектов

### Extensibility

- **Template Method** → легко расширить базовое поведение search
- **Protocol Pattern** → легко добавить новые реализации интерфейсов
- **Callback Pattern** → легко добавить новые обработчики событий

### Testability

- **Dependency Injection** → легко тестировать с mock dependencies
- **Strategy Pattern** → можно тестировать каждую стратегию независимо
- **Builder Pattern** → можно тестировать построение контекста пошагово

### Performance

- **Async Pattern** → эффективная обработка I/O-bound операций (LLM calls)
- **Generator Pattern** → ленивое вычисление и streaming responses
- **Map-Reduce Pattern** → параллельная обработка в Global Search

## Semantic vs Technical Patterns

| Aspect | Semantic Patterns | Technical Patterns |
|---|---|---|
| **Focus** | Meaning transformation | Code organization |
| **Examples** | Chunking, Embedding, Graph Construction | Strategy, Builder, Factory |
| **Goal** | Preserve/transform semantics | Maintainable code |
| **Domain** | Information processing | Software engineering |
| **Metrics** | Semantic preservation, accuracy | Code complexity, coupling |

## Связь с трансформациями

| Transformation | Architectural Pattern | Semantic Pattern | Agent Pattern |
|---|---|---|---|
| **T1: Chunking** | Strategy (chunking strategies) | Chunking Strategy | - |
| **T2: Entity Extraction** | Template Method (extractors) | Graph Construction | LLM Agent + Gleaning |
| **T3: Relationship Extraction** | Template Method | Graph Construction | LLM Agent + Gleaning |
| **T4: Consolidation** | Builder (consolidation) | Semantic Merging | LLM Agent |
| **T5: Community Detection** | Strategy (clustering algorithms) | Graph Organization | - |
| **T6: Community Reports** | Template Method | Semantic Summarization | LLM Agent |
| **T7: Embedding** | Strategy (embedding models) | Embedding Pattern | - |
| **Q1-Q7: Query Execution** | Strategy + Factory | Context Building + Retrieval | Map-Reduce, Multi-hop |

## Best Practices

### 1. Используйте правильный паттерн для задачи

```python
# ✅ GOOD: Strategy для разных алгоритмов
class BaseSearch(ABC):
    @abstractmethod
    async def search(self, query: str) -> SearchResult: ...

# ❌ BAD: Условия внутри одного класса
class Search:
    async def search(self, query: str, search_type: str):
        if search_type == "local":
            return await self._local_search(query)
        elif search_type == "global":
            return await self._global_search(query)
        # ... много условий
```

### 2. Разделяйте concerns

```python
# ✅ GOOD: Context building отдельно от search
class LocalSearch:
    def __init__(self, model: ChatModel, context_builder: LocalContextBuilder):
        self.model = model
        self.context_builder = context_builder

# ❌ BAD: Все в одном классе
class LocalSearch:
    def search(self, query: str):
        # Building context inline
        entities = self._extract_entities(query)
        text_units = self._get_text_units(entities)
        # Generating answer inline
        answer = self._call_llm(entities, text_units)
```

### 3. Используйте dependency injection

```python
# ✅ GOOD: Dependencies injected
class GraphExtractor:
    def __init__(self, model: ChatModel, on_error: ErrorHandlerFn):
        self._model = model
        self._on_error = on_error

# ❌ BAD: Hard-coded dependencies
class GraphExtractor:
    def __init__(self):
        self._model = OpenAIChatModel(...)  # Hard-coded
```

### 4. Применяйте async для I/O-bound операций

```python
# ✅ GOOD: Async для LLM calls
async def extract_entities(texts: list[str]) -> list[Entity]:
    tasks = [extract_from_text(text) for text in texts]
    results = await asyncio.gather(*tasks)  # Parallel
    return results

# ❌ BAD: Sequential sync calls
def extract_entities(texts: list[str]) -> list[Entity]:
    results = []
    for text in texts:
        result = extract_from_text(text)  # Sequential
        results.append(result)
    return results
```

## Документация

1. **[Agent Patterns](01-agent-patterns.md)** - LLM-based агентские паттерны
2. **[Semantic Patterns](02-semantic-patterns.md)** - Семантические паттерны обработки
3. **[Architectural Patterns](03-architectural-patterns.md)** - Классические design patterns
4. **[Code-Level Patterns](04-code-level-patterns.md)** - Паттерны на уровне кода

---

**Версия**: 1.0
**Статус**: Specification
**Основано на**: GraphRAG codebase analysis
