# Code-Level Patterns

## Обзор

**Code-Level Patterns** — паттерны на уровне исходного кода Python, используемые в GraphRAG для типобезопасности, читаемости и эффективности.

## Основные Patterns

1. **Dataclass Pattern** - типизированные data models
2. **Protocol Pattern** - structural typing для интерфейсов
3. **Async Pattern** - асинхронное выполнение
4. **Generator Pattern** - ленивое вычисление и streaming
5. **Context Manager Pattern** - управление ресурсами

---

## 1. Dataclass Pattern

### Определение

Использование `@dataclass` для типизированных data-only классов с автоматической генерацией `__init__`, `__repr__`, etc.

### Применение: Data Models

```python
# graphrag/data_model/entity.py

from dataclasses import dataclass, field

@dataclass
class Entity:
    """Entity data model."""

    id: str | int
    name: str
    type: str
    description: str
    text_unit_ids: list[str] = field(default_factory=list)
    graph_embedding: list[float] | None = None
    community_ids: list[str] = field(default_factory=list)

    def __post_init__(self):
        """Validation after initialization."""
        if not self.name:
            raise ValueError("Entity name cannot be empty")

# Usage
entity = Entity(
    id="e1",
    name="GraphRAG",
    type="TECHNOLOGY",
    description="Knowledge graph RAG system",
    text_unit_ids=["tu_1", "tu_2"],
)

print(entity.name)  # "GraphRAG"
print(entity)  # Entity(id='e1', name='GraphRAG', ...)
```

### Dataclass Features

```python
@dataclass
class TextUnit:
    id: str
    text: str
    document_ids: list[str] = field(default_factory=list)
    entity_ids: list[str] = field(default_factory=list)
    relationship_ids: list[str] = field(default_factory=list)
    n_tokens: int | None = None

    # Field metadata
    embedding: list[float] | None = field(default=None, repr=False)  # Hidden from repr

    # Computed property
    @property
    def has_entities(self) -> bool:
        return len(self.entity_ids) > 0

# Frozen dataclass (immutable)
@dataclass(frozen=True)
class SearchResult:
    """Immutable search result."""
    response: str
    context_data: str
    completion_time: float

# result.response = "new value"  # Error: frozen!
```

### Преимущества

✅ Автоматическая генерация boilerplate кода
✅ Type hints для IDE autocomplete
✅ Immutability с `frozen=True`
✅ Default values и factories
✅ Структурное equality (`==`)

---

## 2. Protocol Pattern

### Определение

Использование `typing.Protocol` для structural typing (duck typing с проверкой типов).

### Применение: Callbacks Interface

```python
# graphrag/callbacks/workflow_callbacks.py

from typing import Protocol

class WorkflowCallbacks(Protocol):
    """Protocol for workflow observers."""

    def workflow_start(self, name: str, instance: object) -> None:
        """Called when workflow starts."""
        ...

    def workflow_end(self, name: str, instance: object) -> None:
        """Called when workflow ends."""
        ...

    def error(
        self,
        message: str,
        cause: BaseException | None = None,
    ) -> None:
        """Called on error."""
        ...

# Any class implementing these methods is compatible
class MyCallbacks:
    """No explicit inheritance needed."""

    def workflow_start(self, name: str, instance: object) -> None:
        print(f"Starting {name}")

    def workflow_end(self, name: str, instance: object) -> None:
        print(f"Finished {name}")

    def error(self, message: str, cause: BaseException | None = None) -> None:
        print(f"Error: {message}")

# Type checker accepts this!
callbacks: WorkflowCallbacks = MyCallbacks()
```

### Protocol vs ABC

```python
# ABC approach (nominal typing)
from abc import ABC, abstractmethod

class BaseCallbacks(ABC):
    """Must explicitly inherit."""

    @abstractmethod
    def workflow_start(self, name: str) -> None:
        ...

class MyCallbacks(BaseCallbacks):  # Explicit inheritance required
    def workflow_start(self, name: str) -> None:
        print(f"Starting {name}")

# Protocol approach (structural typing)
from typing import Protocol

class CallbacksProtocol(Protocol):
    """No inheritance needed."""

    def workflow_start(self, name: str) -> None:
        ...

class MyCallbacks:  # No explicit inheritance!
    def workflow_start(self, name: str) -> None:
        print(f"Starting {name}")

# Both work!
callbacks: CallbacksProtocol = MyCallbacks()
```

### Преимущества Protocol

✅ Structural typing (duck typing with type safety)
✅ No explicit inheritance needed
✅ Better for third-party integrations
✅ More Pythonic

---

## 3. Async Pattern

### Определение

Использование `async`/`await` для асинхронного выполнения I/O-bound операций.

### Async Functions

```python
# graphrag/query/structured_search/local_search/search.py

class LocalSearch:
    async def search(self, query: str) -> SearchResult:
        """Async search execution."""
        start_time = time.time()

        # Async context building (may involve I/O)
        context_result = self.context_builder.build_context(query)

        # Async LLM call
        full_response = ""
        async for chunk in self.model.achat_stream(
            prompt=query,
            history=[{"role": "system", "content": search_prompt}],
            **self.model_params
        ):
            full_response += chunk

        completion_time = time.time() - start_time

        return SearchResult(
            response=full_response,
            completion_time=completion_time,
            ...
        )
```

### Parallel Execution with asyncio.gather

```python
# graphrag/query/structured_search/global_search/search.py

async def parallel_map_calls(
    communities: list[str],
    query: str,
) -> list[str]:
    """Execute map phase in parallel."""

    async def map_task(community: str) -> str:
        """Single map call."""
        response = await model.achat(
            prompt=query,
            history=[{"role": "system", "content": map_prompt.format(community)}],
        )
        return response

    # Create tasks for all communities
    tasks = [map_task(community) for community in communities]

    # Execute in parallel
    results = await asyncio.gather(*tasks)

    return results
```

### Controlled Concurrency

```python
import asyncio
from asyncio import Semaphore

async def controlled_parallel_execution(
    items: list[Any],
    process_fn: callable,
    max_concurrency: int = 10,
) -> list[Any]:
    """Limit concurrent async operations."""
    semaphore = Semaphore(max_concurrency)

    async def limited_task(item: Any) -> Any:
        async with semaphore:  # Acquire semaphore
            return await process_fn(item)

    # Create tasks
    tasks = [limited_task(item) for item in items]

    # Execute with concurrency control
    results = await asyncio.gather(*tasks)

    return results

# Usage: Limit to 10 concurrent LLM calls
results = await controlled_parallel_execution(
    items=texts,
    process_fn=lambda text: extractor.extract(text),
    max_concurrency=10,
)
```

### Async Context Managers

```python
class AsyncLLMClient:
    """Async context manager for LLM client."""

    async def __aenter__(self):
        """Setup on entry."""
        await self.connect()
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        """Cleanup on exit."""
        await self.disconnect()

# Usage
async with AsyncLLMClient() as client:
    response = await client.achat(prompt)
# Automatically cleaned up
```

---

## 4. Generator Pattern

### Определение

Использование generators для ленивого вычисления и memory-efficient обработки.

### Generator Functions

```python
# graphrag/index/operations/chunk_text/strategies.py

def run_sentences(
    input: list[str],
    config: ChunkingConfig,
    tick: ProgressTicker,
) -> Iterable[TextChunk]:
    """Generator for sentence chunks."""
    for doc_idx, text in enumerate(input):
        sentences = nltk.sent_tokenize(text)

        for sentence in sentences:
            # Yield one chunk at a time (lazy)
            yield TextChunk(
                text_chunk=sentence,
                source_doc_indices=[doc_idx],
            )

        tick(1)

# Usage
for chunk in run_sentences(documents, config, tick):
    process(chunk)  # Processed one at a time, not all in memory
```

### Async Generators (Streaming)

```python
# graphrag/query/structured_search/base.py

class BaseSearch(ABC):
    @abstractmethod
    async def stream_search(
        self, query: str
    ) -> AsyncGenerator[str, None]:
        """Stream search results."""
        yield ""  # Makes it async generator

# Implementation
class LocalSearch(BaseSearch):
    async def stream_search(self, query: str) -> AsyncGenerator[str, None]:
        """Stream answer token by token."""
        context = self.context_builder.build_context(query)

        search_prompt = self.system_prompt.format(context_data=context)

        # Stream from LLM
        async for token in self.model.achat_stream(
            prompt=query,
            history=[{"role": "system", "content": search_prompt}],
            **self.model_params
        ):
            yield token  # Stream to client immediately

# Usage
async for token in search_engine.stream_search(query):
    print(token, end="", flush=True)  # Real-time streaming
```

### Generator Expressions

```python
# Memory-efficient processing
entity_names = (entity.name for entity in entities)  # Generator expression
entity_names_list = [entity.name for entity in entities]  # List (all in memory)

# Chain generators
def process_pipeline(documents):
    """Pipeline of generators."""
    chunks = chunk_documents(documents)  # Generator
    entities = extract_entities(chunks)  # Generator
    filtered = filter_entities(entities)  # Generator
    return filtered  # Nothing computed yet!

# Only when consumed:
for entity in process_pipeline(docs):
    save(entity)  # Computed on-demand
```

---

## 5. Context Manager Pattern

### Определение

Использование context managers для автоматического управления ресурсами (setup/teardown).

### File Context Manager

```python
# Built-in context manager
with open("data.txt", "r") as f:
    content = f.read()
# File automatically closed
```

### Custom Context Manager

```python
from contextlib import contextmanager

@contextmanager
def timer(name: str):
    """Context manager for timing."""
    start = time.time()
    try:
        yield
    finally:
        end = time.time()
        print(f"{name} took {end - start:.2f}s")

# Usage
with timer("Entity extraction"):
    entities = await extractor.extract(text)
# Automatically prints timing
```

### Async Context Manager

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def llm_session(model: ChatModel):
    """Async context manager for LLM session."""
    # Setup
    await model.connect()
    try:
        yield model
    finally:
        # Teardown
        await model.disconnect()

# Usage
async with llm_session(model) as llm:
    response = await llm.achat(prompt)
# Automatically disconnected
```

---

## Additional Patterns

### Type Hints

```python
from typing import List, Dict, Optional, Union, Any, TypeVar, Generic

def process_entities(
    entities: list[Entity],
    config: dict[str, Any],
    callback: Optional[callable] = None,
) -> list[Entity]:
    """Type hints for better IDE support."""
    processed: list[Entity] = []

    for entity in entities:
        if callback:
            callback(entity)
        processed.append(transform(entity))

    return processed
```

### Generic Types

```python
from typing import TypeVar, Generic

T = TypeVar("T", GlobalContextBuilder, LocalContextBuilder, DRIFTContextBuilder)

class BaseSearch(ABC, Generic[T]):
    """Generic base class."""

    def __init__(self, context_builder: T):
        self.context_builder: T = context_builder

# Type-safe usage
local_search: BaseSearch[LocalContextBuilder] = LocalSearch(local_builder)
```

### Enum Pattern

```python
from enum import Enum

class SearchType(str, Enum):
    """Enum for search types."""
    LOCAL = "local"
    GLOBAL = "global"
    DRIFT = "drift"
    BASIC = "basic"

# Usage
def create_search(search_type: SearchType) -> BaseSearch:
    if search_type == SearchType.LOCAL:
        return LocalSearch(...)
    elif search_type == SearchType.GLOBAL:
        return GlobalSearch(...)
    # Type-safe, autocomplete works

# Client
search = create_search(SearchType.LOCAL)  # Type-safe
```

### Property Pattern

```python
class Entity:
    def __init__(self, name: str, description: str):
        self._name = name
        self._description = description
        self._embedding = None

    @property
    def name(self) -> str:
        """Read-only property."""
        return self._name

    @property
    def embedding(self) -> list[float] | None:
        """Lazy-loaded property."""
        if self._embedding is None:
            self._embedding = self._compute_embedding()
        return self._embedding

    @embedding.setter
    def embedding(self, value: list[float]) -> None:
        """Setter for embedding."""
        self._embedding = value

# Usage
entity = Entity("GraphRAG", "Knowledge graph system")
print(entity.name)  # Property access

# Lazy loading
emb = entity.embedding  # Computed on first access
emb2 = entity.embedding  # Cached, not recomputed
```

### Decorator Pattern

```python
from functools import wraps
import logging

def log_execution(func):
    """Decorator for logging function execution."""
    @wraps(func)
    async def wrapper(*args, **kwargs):
        logging.info(f"Executing {func.__name__}")
        try:
            result = await func(*args, **kwargs)
            logging.info(f"Completed {func.__name__}")
            return result
        except Exception as e:
            logging.error(f"Error in {func.__name__}: {e}")
            raise
    return wrapper

# Usage
@log_execution
async def extract_entities(text: str) -> list[Entity]:
    """Function is automatically logged."""
    return await extractor.extract(text)
```

### Cache Pattern

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def get_encoding(model_name: str):
    """Cache expensive encoding creation."""
    return tiktoken.encoding_for_model(model_name)

# First call: creates encoding
enc1 = get_encoding("gpt-4")

# Second call: returns cached
enc2 = get_encoding("gpt-4")  # Same object
```

---

## Best Practices

### 1. Type Hints Everywhere

```python
# ✅ GOOD: Clear types
async def search(
    query: str,
    top_k: int = 10,
    filters: dict[str, Any] | None = None,
) -> list[SearchResult]:
    ...

# ❌ BAD: No types
async def search(query, top_k=10, filters=None):
    ...
```

### 2. Dataclasses for Data

```python
# ✅ GOOD: Dataclass
@dataclass
class SearchResult:
    response: str
    context: str
    latency: float

# ❌ BAD: Dict
result = {
    "response": "...",
    "context": "...",
    "latency": 1.5,
}  # No type safety, no IDE support
```

### 3. Async for I/O

```python
# ✅ GOOD: Async for I/O-bound
async def fetch_data(url: str) -> str:
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.text()

# ❌ BAD: Sync blocking
def fetch_data(url: str) -> str:
    response = requests.get(url)  # Blocks event loop!
    return response.text
```

### 4. Generators for Large Data

```python
# ✅ GOOD: Generator (lazy)
def process_documents(docs: list[str]) -> Iterable[Entity]:
    for doc in docs:
        entities = extract_entities(doc)
        for entity in entities:
            yield entity  # One at a time

# ❌ BAD: Load all in memory
def process_documents(docs: list[str]) -> list[Entity]:
    all_entities = []
    for doc in docs:
        entities = extract_entities(doc)
        all_entities.extend(entities)  # All in memory!
    return all_entities
```

### 5. Context Managers for Resources

```python
# ✅ GOOD: Context manager
with open("data.txt") as f:
    data = f.read()
# Automatically closed

# ❌ BAD: Manual cleanup
f = open("data.txt")
data = f.read()
f.close()  # Easy to forget!
```

---

## Performance Patterns

### Batching

```python
async def process_in_batches(
    items: list[Any],
    batch_size: int = 100,
) -> list[Result]:
    """Process large lists in batches."""
    results = []

    for i in range(0, len(items), batch_size):
        batch = items[i:i+batch_size]
        batch_results = await process_batch(batch)
        results.extend(batch_results)

    return results
```

### Caching

```python
from functools import lru_cache

class EntityRetriever:
    @lru_cache(maxsize=1000)
    def get_entity(self, entity_id: str) -> Entity:
        """Cache frequently accessed entities."""
        return self._fetch_from_db(entity_id)
```

### Lazy Evaluation

```python
class LazyResult:
    """Lazy evaluation pattern."""

    def __init__(self, compute_fn: callable):
        self._compute_fn = compute_fn
        self._result = None
        self._computed = False

    @property
    def value(self):
        """Compute only when accessed."""
        if not self._computed:
            self._result = self._compute_fn()
            self._computed = True
        return self._result

# Usage
lazy = LazyResult(lambda: expensive_computation())
# Nothing computed yet

result = lazy.value  # Computed now
result2 = lazy.value  # Cached
```

---

**Related**:
- [Agent Patterns](01-agent-patterns.md)
- [Semantic Patterns](02-semantic-patterns.md)
- [Architectural Patterns](03-architectural-patterns.md)
