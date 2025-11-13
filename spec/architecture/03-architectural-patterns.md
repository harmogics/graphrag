# Architectural Patterns

## Обзор

Классические **архитектурные и design паттерны**, используемые в GraphRAG для организации кода, обеспечения расширяемости и поддерживаемости.

## Основные Patterns

1. **Strategy Pattern** - взаимозаменяемые алгоритмы
2. **Template Method Pattern** - каркас алгоритма
3. **Builder Pattern** - пошаговое построение объектов
4. **Factory Pattern** - создание семейств объектов
5. **Observer/Callback Pattern** - уведомления о событиях
6. **Adapter Pattern** - согласование интерфейсов

---

## 1. Strategy Pattern

### Определение

Инкапсуляция семейства алгоритмов, делающая их взаимозаменяемыми.

### Применение: Search Strategies

```python
# graphrag/query/structured_search/base.py

class BaseSearch(ABC, Generic[T]):
    """Strategy interface for search algorithms."""

    @abstractmethod
    async def search(
        self, query: str, **kwargs
    ) -> SearchResult:
        """Execute search strategy."""

# Concrete strategies
class GlobalSearch(BaseSearch[GlobalContextBuilder]):
    """Strategy: Map-Reduce across communities."""
    async def search(self, query: str) -> SearchResult:
        # Map-Reduce implementation
        ...

class LocalSearch(BaseSearch[LocalContextBuilder]):
    """Strategy: Entity-focused detailed search."""
    async def search(self, query: str) -> SearchResult:
        # Local search implementation
        ...

class DRIFTSearch(BaseSearch[DRIFTContextBuilder]):
    """Strategy: Multi-hop graph traversal."""
    async def search(self, query: str) -> SearchResult:
        # DRIFT implementation
        ...

class BasicSearch(BaseSearch[BasicContextBuilder]):
    """Strategy: Simple text similarity."""
    async def search(self, query: str) -> SearchResult:
        # Basic search implementation
        ...
```

### Client Code

```python
# Client не знает о конкретной стратегии
search_engine: BaseSearch = get_search_engine(config, search_type)
result = await search_engine.search(query)
```

### Преимущества

✅ Легко добавить новую стратегию (например, HybridSearch)
✅ Стратегии тестируются независимо
✅ Client code не зависит от конкретной реализации

---

## 2. Template Method Pattern

### Определение

Определение каркаса алгоритма с делегированием отдельных шагов подклассам.

### Применение: BaseSearch

```python
class BaseSearch(ABC):
    """Template for search execution."""

    async def search(self, query: str) -> SearchResult:
        """Template method defining search flow."""
        # Step 1: Build context (delegated to subclass)
        context = await self.build_context(query)

        # Step 2: Generate answer (common)
        answer = await self._generate_answer(query, context)

        # Step 3: Package result (common)
        return self._create_result(answer, context)

    @abstractmethod
    async def build_context(self, query: str) -> Context:
        """Subclasses implement context building."""

    async def _generate_answer(self, query: str, context: Context) -> str:
        """Common answer generation."""
        return await self.model.achat(query, context=context)
```

### Применение: Context Builders

```python
# graphrag/query/context_builder/builders.py

class LocalContextBuilder(ABC):
    """Template for context building."""

    def build_context(self, query: str, **kwargs) -> ContextBuilderResult:
        """Template method."""
        # Step 1: Extract query concepts
        concepts = self._extract_concepts(query)

        # Step 2: Retrieve relevant data (delegated)
        data = self._retrieve_data(concepts, **kwargs)

        # Step 3: Format context (delegated)
        context = self._format_context(data)

        return ContextBuilderResult(context_chunks=context, ...)

    @abstractmethod
    def _retrieve_data(self, concepts, **kwargs):
        """Subclass implements retrieval."""

    @abstractmethod
    def _format_context(self, data):
        """Subclass implements formatting."""
```

---

## 3. Builder Pattern

### Определение

Пошаговое построение сложных объектов с различными представлениями.

### Применение: Context Building

```python
class ContextBuilder:
    """Builder for complex context objects."""

    def __init__(self):
        self._entities = []
        self._relationships = []
        self._text_units = []
        self._metadata = {}

    def add_entities(self, entities: list[Entity]) -> 'ContextBuilder':
        """Add entities to context."""
        self._entities.extend(entities)
        return self  # Method chaining

    def add_relationships(self, relationships: list[Relationship]) -> 'ContextBuilder':
        """Add relationships to context."""
        self._relationships.extend(relationships)
        return self

    def add_text_units(self, text_units: list[TextUnit]) -> 'ContextBuilder':
        """Add text units."""
        self._text_units.extend(text_units)
        return self

    def set_metadata(self, **metadata) -> 'ContextBuilder':
        """Set metadata."""
        self._metadata.update(metadata)
        return self

    def build(self, max_tokens: int = 8000) -> str:
        """Build final context string."""
        context_parts = []

        # Format entities
        if self._entities:
            context_parts.append("# Entities")
            for entity in self._entities[:30]:  # Limit
                context_parts.append(f"- {entity.name}: {entity.description}")

        # Format relationships
        if self._relationships:
            context_parts.append("\n# Relationships")
            for rel in self._relationships[:50]:
                context_parts.append(
                    f"- {rel.source} {rel.type} {rel.target}"
                )

        # Format text units
        if self._text_units:
            context_parts.append("\n# Text Units")
            for idx, unit in enumerate(self._text_units[:20]):
                context_parts.append(f"[{idx}] {unit.text}")

        # Join and truncate to token limit
        full_context = "\n".join(context_parts)
        return self._truncate_to_tokens(full_context, max_tokens)

# Usage
context = (ContextBuilder()
    .add_entities(retrieved_entities)
    .add_relationships(retrieved_relationships)
    .add_text_units(retrieved_text_units)
    .set_metadata(query=query, retrieval_time=time.time())
    .build(max_tokens=8000))
```

### Преимущества

✅ Пошаговое построение сложного объекта
✅ Fluent interface (method chaining)
✅ Различные представления из одних данных

---

## 4. Factory Pattern

### Определение

Создание семейств связанных объектов без указания конкретных классов.

### Применение: Search Engine Factory

```python
# graphrag/query/factory.py

def get_local_search_engine(
    config: GraphRagConfig,
    entities: list[Entity],
    relationships: list[Relationship],
    ...
) -> LocalSearch:
    """Factory for LocalSearch with all dependencies."""

    # Create chat model
    model_settings = config.get_language_model_config(...)
    chat_model = ModelManager().get_or_create_chat_model(
        name="local_search_chat",
        model_type=model_settings.type,
        config=model_settings,
    )

    # Create embedding model
    embedding_model = ModelManager().get_or_create_embedding_model(
        name="local_search_embedding",
        ...
    )

    # Create context builder
    context_builder = LocalSearchMixedContext(
        entities=entities,
        relationships=relationships,
        entity_text_embeddings=description_embedding_store,
        text_embedder=embedding_model,
        ...
    )

    # Create search engine
    return LocalSearch(
        model=chat_model,
        context_builder=context_builder,
        token_encoder=tiktoken.get_encoding(...),
        ...
    )

# Similar factories for other search types
def get_global_search_engine(...) -> GlobalSearch: ...
def get_drift_search_engine(...) -> DRIFTSearch: ...
def get_basic_search_engine(...) -> BasicSearch: ...
```

### Abstract Factory: Vector Store

```python
# graphrag/vector_stores/factory.py

class VectorStoreFactory:
    """Factory for creating vector stores."""

    @staticmethod
    def create(
        store_type: str,
        config: VectorStoreConfig,
    ) -> BaseVectorStore:
        """Create vector store based on type."""
        if store_type == "lancedb":
            return LanceDBVectorStore(config)
        elif store_type == "azure_ai_search":
            return AzureAISearchVectorStore(config)
        elif store_type == "cosmosdb":
            return CosmosDBVectorStore(config)
        else:
            raise ValueError(f"Unknown vector store type: {store_type}")

# Usage
vector_store = VectorStoreFactory.create(
    store_type=config.vector_store.type,
    config=config.vector_store,
)
```

### Преимущества

✅ Централизованное создание объектов
✅ Скрытие сложности инициализации
✅ Легко добавить новые типы
✅ Dependency injection

---

## 5. Observer/Callback Pattern

### Определение

Определение зависимости один-ко-многим, где изменение объекта уведомляет всех наблюдателей.

### Применение: Workflow Callbacks

```python
# graphrag/callbacks/workflow_callbacks.py

class WorkflowCallbacks(Protocol):
    """Observer interface for workflow events."""

    def workflow_start(self, name: str, instance: object) -> None:
        """Notified when workflow starts."""

    def workflow_end(self, name: str, instance: object) -> None:
        """Notified when workflow ends."""

    def error(
        self,
        message: str,
        cause: BaseException | None = None,
        ...
    ) -> None:
        """Notified on errors."""

    def progress(self, progress: Progress) -> None:
        """Notified on progress updates."""

# Concrete observers
class ConsoleWorkflowCallbacks:
    """Print to console."""
    def workflow_start(self, name: str, instance: object):
        print(f"Starting {name}...")

class FileWorkflowCallbacks:
    """Write to file."""
    def workflow_start(self, name: str, instance: object):
        self.log_file.write(f"[START] {name}\n")

class ProgressWorkflowCallbacks:
    """Update progress bar."""
    def progress(self, progress: Progress):
        self.progress_bar.update(progress.completed_items)
```

### Subject (Observable)

```python
class WorkflowExecutor:
    """Subject that notifies observers."""

    def __init__(self, callbacks: list[WorkflowCallbacks]):
        self._callbacks = callbacks

    async def execute_workflow(self, workflow: Workflow):
        """Execute workflow with notifications."""

        # Notify start
        for callback in self._callbacks:
            callback.workflow_start(workflow.name, workflow)

        try:
            # Execute workflow
            result = await workflow.run()

            # Notify end
            for callback in self._callbacks:
                callback.workflow_end(workflow.name, workflow)

            return result

        except Exception as e:
            # Notify error
            for callback in self._callbacks:
                callback.error(f"Workflow {workflow.name} failed", cause=e)
            raise
```

### Usage

```python
# Attach multiple observers
callbacks = [
    ConsoleWorkflowCallbacks(),
    FileWorkflowCallbacks(log_path="./logs/workflow.log"),
    ProgressWorkflowCallbacks(),
]

executor = WorkflowExecutor(callbacks=callbacks)
await executor.execute_workflow(indexing_workflow)
```

---

## 6. Adapter Pattern

### Определение

Преобразование интерфейса класса в другой интерфейс, ожидаемый клиентом.

### Применение: Indexer Adapters

```python
# graphrag/query/indexer_adapters.py

class IndexerAdapter:
    """Adapt indexer output to query input format."""

    @staticmethod
    def adapt_entities(
        indexer_entities: pd.DataFrame
    ) -> list[Entity]:
        """Adapt entity format."""
        return [
            Entity(
                id=row["id"],
                name=row["title"],  # Adapt: title → name
                type=row["type"],
                description=row["description"],
                text_unit_ids=row.get("text_unit_ids", []),
            )
            for _, row in indexer_entities.iterrows()
        ]

    @staticmethod
    def adapt_relationships(
        indexer_relationships: pd.DataFrame
    ) -> list[Relationship]:
        """Adapt relationship format."""
        return [
            Relationship(
                source=row["source"],
                target=row["target"],
                type=row["relationship"],  # Adapt: relationship → type
                description=row.get("description", ""),
                weight=row.get("weight", 1.0),
            )
            for _, row in indexer_relationships.iterrows()
        ]
```

### Storage Adapter

```python
# Adapter между различными storage backends

class StorageAdapter:
    """Adapt different storage backends to common interface."""

    def __init__(self, backend: str, config: dict):
        if backend == "blob":
            self._storage = BlobPipelineStorage(config)
        elif backend == "cosmosdb":
            self._storage = CosmosDBPipelineStorage(config)
        elif backend == "file":
            self._storage = FilePipelineStorage(config)

    async def read(self, key: str) -> Any:
        """Common read interface."""
        return await self._storage.get(key)

    async def write(self, key: str, data: Any) -> None:
        """Common write interface."""
        await self._storage.set(key, data)
```

---

## Pattern Combinations

### Example 1: Strategy + Factory

```python
# Factory создает стратегию
def create_search_strategy(
    search_type: str,
    config: GraphRagConfig,
    ...
) -> BaseSearch:
    """Factory for creating search strategies."""
    if search_type == "local":
        return get_local_search_engine(config, ...)
    elif search_type == "global":
        return get_global_search_engine(config, ...)
    elif search_type == "drift":
        return get_drift_search_engine(config, ...)
    else:
        return get_basic_search_engine(config, ...)

# Client использует strategy через единый интерфейс
strategy = create_search_strategy("local", config, ...)
result = await strategy.search(query)
```

### Example 2: Builder + Template Method

```python
class ContextBuilder(ABC):
    """Template Method + Builder combination."""

    def build_context(self, query: str) -> str:
        """Template method."""
        builder = StringBuilder()

        # Step 1: Add entities (delegated)
        entities = self._get_entities(query)
        builder.add_section("Entities", self._format_entities(entities))

        # Step 2: Add relationships (delegated)
        relationships = self._get_relationships(entities)
        builder.add_section("Relationships", self._format_relationships(relationships))

        # Step 3: Build (builder pattern)
        return builder.build(max_tokens=8000)

    @abstractmethod
    def _get_entities(self, query: str) -> list[Entity]:
        """Subclass implements."""

    @abstractmethod
    def _get_relationships(self, entities: list[Entity]) -> list[Relationship]:
        """Subclass implements."""
```

### Example 3: Observer + Factory

```python
# Factory создает observers
def create_callbacks(config: GraphRagConfig) -> list[WorkflowCallbacks]:
    """Factory for creating callback observers."""
    callbacks = []

    if config.reporting.type == "console":
        callbacks.append(ConsoleWorkflowCallbacks())

    if config.reporting.type == "file":
        callbacks.append(FileWorkflowCallbacks(
            log_path=config.reporting.base_dir
        ))

    if config.reporting.type == "blob":
        callbacks.append(BlobWorkflowCallbacks(
            connection_string=config.reporting.connection_string
        ))

    return callbacks

# Usage
callbacks = create_callbacks(config)
executor = WorkflowExecutor(callbacks=callbacks)
```

---

## Design Principles

### 1. Open/Closed Principle

```python
# ✅ GOOD: Open for extension, closed for modification
class BaseSearch(ABC):
    @abstractmethod
    async def search(self, query: str) -> SearchResult:
        """Extend by creating new subclass."""

class MyCustomSearch(BaseSearch):
    """New search type без изменения BaseSearch."""
    async def search(self, query: str) -> SearchResult:
        # Custom implementation
        ...

# ❌ BAD: Modifying base class
class Search:
    async def search(self, query: str, search_type: str):
        if search_type == "local":
            ...
        elif search_type == "global":
            ...
        elif search_type == "my_custom":  # Modification!
            ...
```

### 2. Dependency Inversion

```python
# ✅ GOOD: Depend on abstractions
class LocalSearch:
    def __init__(
        self,
        model: ChatModel,  # Interface, not concrete class
        context_builder: LocalContextBuilder,  # Interface
    ):
        self.model = model
        self.context_builder = context_builder

# ❌ BAD: Depend on concrete classes
class LocalSearch:
    def __init__(self):
        self.model = OpenAIChatModel(...)  # Concrete!
        self.context_builder = LocalSearchMixedContext(...)  # Concrete!
```

### 3. Single Responsibility

```python
# ✅ GOOD: Each class has one responsibility
class GraphExtractor:
    """Responsibility: Extract entities from text."""

class GraphConsolidator:
    """Responsibility: Consolidate duplicate entities."""

class GraphIndexer:
    """Responsibility: Index graph to storage."""

# ❌ BAD: Multiple responsibilities
class GraphProcessor:
    """Does extraction, consolidation, and indexing."""
    def process(self, text):
        entities = self.extract(text)  # Responsibility 1
        consolidated = self.consolidate(entities)  # Responsibility 2
        self.index(consolidated)  # Responsibility 3
```

---

**Related**:
- [Agent Patterns](01-agent-patterns.md)
- [Semantic Patterns](02-semantic-patterns.md)
- [Code-Level Patterns](04-code-level-patterns.md)
