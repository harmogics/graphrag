# Разбиение документов и создание Text Chunks

## Обзор этапа

Разбиение документов (Document Chunking) — это первый критический этап pipeline, который преобразует входные документы в управляемые текстовые фрагменты (text units). Этот процесс учитывает семантические границы текста, обеспечивает оптимальный размер для LLM-обработки и сохраняет связь с исходными документами.

## Архитектура компонента

### Ключевые файлы
- **Workflow**: `graphrag/index/workflows/create_base_text_units.py`
- **Операция**: `graphrag/index/operations/chunk_text/chunk_text.py`
- **Text Splitting**: `graphrag/index/text_splitting/text_splitting.py`
- **Конфигурация**: `graphrag/config/models/chunking_config.py`

## Процесс разбиения

```
INPUT: Documents (с полями id, text)
    ↓
┌─────────────────────────────────┐
│  1. Загрузка документов         │
└─────────────────────────────────┘
    ↓
┌─────────────────────────────────┐
│  2. Группировка по колонкам     │
│     (по умолчанию: по id)       │
└─────────────────────────────────┘
    ↓
┌─────────────────────────────────┐
│  3. Токенизация текста          │
│     (TokenTextSplitter)         │
└─────────────────────────────────┘
    ↓
┌─────────────────────────────────┐
│  4. Создание chunks с overlap   │
└─────────────────────────────────┘
    ↓
┌─────────────────────────────────┐
│  5. Генерация ID (SHA512 hash)  │
└─────────────────────────────────┘
    ↓
┌─────────────────────────────────┐
│  6. Подсчет токенов             │
└─────────────────────────────────┘
    ↓
┌─────────────────────────────────┐
│  7. Добавление метаданных       │
└─────────────────────────────────┘
    ↓
OUTPUT: text_units.parquet
```

## Конфигурация chunking

### ChunkingConfig параметры

```python
class ChunkingConfig:
    # Размер chunk в токенах
    size: int = 1200

    # Перекрытие между соседними chunks (в токенах)
    overlap: int = 100

    # Колонки для группировки при разбиении
    group_by_columns: list[str] = ["id"]

    # Стратегия разбиения
    strategy: ChunkStrategyType = "tokens"  # tokens | sentence | paragraph

    # Модель токенизации (tiktoken)
    encoding_model: str = "cl100k_base"

    # Добавлять ли метаданные в начало chunk
    prepend_metadata: bool = False

    # Учитывать ли метаданные в размере chunk
    chunk_size_includes_metadata: bool = False
```

**Файл конфигурации**: `graphrag/config/models/chunking_config.py:17`

## Стратегии разбиения

### 1. Token-based Strategy (по умолчанию)

Разбивает текст по границам токенов с использованием библиотеки `tiktoken`.

**Преимущества**:
- Точный контроль размера для LLM (максимальный context window)
- Эффективное использование токенов
- Предсказуемая стоимость API-вызовов

**Механизм**:
```python
# graphrag/index/text_splitting/text_splitting.py
class TokenTextSplitter:
    def __init__(
        self,
        chunk_size: int = 1200,
        chunk_overlap: int = 100,
        encoding_name: str = "cl100k_base"
    ):
        self._encoding = tiktoken.get_encoding(encoding_name)
        self._chunk_size = chunk_size
        self._chunk_overlap = chunk_overlap

    def split_text(self, text: str) -> list[str]:
        # 1. Кодировать текст в токены
        tokens = self._encoding.encode(text)

        # 2. Разбить на chunks с overlap
        chunks = []
        start = 0
        while start < len(tokens):
            end = start + self._chunk_size
            chunk_tokens = tokens[start:end]
            chunk_text = self._encoding.decode(chunk_tokens)
            chunks.append(chunk_text)
            start += self._chunk_size - self._chunk_overlap

        return chunks
```

### 2. Sentence-based Strategy

Разбивает по границам предложений, сохраняя семантическую целостность.

**Преимущества**:
- Сохранение смысловых единиц
- Более естественные границы для NLP
- Лучше для анализа локального контекста

**Ограничения**:
- Менее предсказуемый размер chunks
- Может быть избыточным или недостаточным для LLM

### 3. Paragraph-based Strategy

Разбивает по абзацам (границы `\n\n`).

**Преимущества**:
- Максимальное сохранение контекста
- Идеально для структурированного текста

**Ограничения**:
- Большой разброс в размерах chunks
- Требует хорошо структурированный исходный текст

## Семантическое перекрытие (Overlap)

Перекрытие между chunks критически важно для сохранения контекста на границах.

### Принцип работы

```
Document: "... токен1 токен2 токен3 токен4 токен5 токен6 ..."

Chunk 1: [токен1 токен2 токен3 токен4]
                            ↓ overlap ↓
Chunk 2:             [токен3 токен4 токен5 токен6]
```

### Зачем нужен overlap?

1. **Предотвращение потери контекста**: Если важная сущность или отношение находится на границе chunk, overlap гарантирует её полное представление в одном из chunks.

2. **Улучшение quality extraction**: LLM может видеть полный контекст для сущностей, упомянутых в конце одного chunk и развиваемых в начале следующего.

3. **Связность графа**: Сущности, упомянутые в overlap-области, будут извлечены из обоих chunks, создавая более плотный и связный граф.

### Рекомендуемые значения

- **Standard documents**: overlap = 100 tokens (~8-12% от chunk_size)
- **Technical texts**: overlap = 150-200 tokens (больше для сохранения терминологии)
- **Narrative texts**: overlap = 50-100 tokens (меньше для избежания избыточности)

## Генерация ID и метаданные

### ID generation

Каждый text_unit получает уникальный ID на основе SHA512 hash содержимого:

```python
def create_chunk_id(text: str, document_id: str, chunk_index: int) -> str:
    """Создает детерминированный ID для chunk"""
    content = f"{document_id}_{chunk_index}_{text}"
    return hashlib.sha512(content.encode()).hexdigest()[:16]
```

**Преимущества**:
- Детерминированность: один и тот же chunk всегда получает один ID
- Идемпотентность: повторное выполнение не создает дубликаты
- Уникальность: коллизии практически невозможны

### Метаданные text_unit

```python
TextUnit = {
    "id": str,                    # SHA512 hash
    "text": str,                  # Текст chunk
    "n_tokens": int,              # Количество токенов
    "document_ids": list[str],    # Исходные документы
    "metadata": dict | None,      # Дополнительные метаданные

    # Добавляется на более поздних этапах:
    "entity_ids": list[str],           # Извлеченные сущности
    "relationship_ids": list[str],     # Извлеченные отношения
    "covariate_ids": list[str]         # Claims (если enabled)
}
```

## Workflow: create_base_text_units

### Код workflow

```python
# graphrag/index/workflows/create_base_text_units.py

async def create_base_text_units(
    config: GraphRagConfig,
    context: PipelineRunContext
) -> WorkflowFunctionOutput:
    """
    Создает базовые text units из документов

    INPUT: documents.parquet (id, title, text)
    OUTPUT: text_units.parquet (id, text, n_tokens, document_ids)
    """

    # 1. Загрузить documents
    documents = await context.storage.get("documents")

    # 2. Получить конфигурацию chunking
    chunk_config = config.chunking

    # 3. Выполнить chunking
    text_units = chunk_text(
        documents,
        column="text",
        chunk_size=chunk_config.size,
        chunk_overlap=chunk_config.overlap,
        encoding_model=chunk_config.encoding_model
    )

    # 4. Сохранить результат
    await context.storage.set("text_units", text_units)

    return WorkflowFunctionOutput(
        outputs={"text_units": text_units}
    )
```

**Файл**: `graphrag/index/workflows/create_base_text_units.py:25`

## Оптимизация размера chunks

### Факторы, влияющие на выбор размера

1. **Context window LLM**: Размер chunk должен умещаться в контекст extraction-промпта
   - GPT-4: 8k tokens → chunks до 1500 tokens безопасны
   - GPT-4-32k: 32k tokens → можно использовать chunks до 4000-5000 tokens

2. **Качество extraction**:
   - Маленькие chunks (300-600 tokens): высокая precision, низкий recall
   - Средние chunks (1000-1500 tokens): баланс precision/recall
   - Большие chunks (2000-3000 tokens): низкая precision, высокий recall

3. **Стоимость обработки**:
   - Больше chunks = больше LLM-вызовов = выше стоимость
   - Но: маленькие chunks могут требовать больше overlap → тоже увеличение стоимости

### Рекомендации по настройке

```python
# Для научных/технических текстов
ChunkingConfig(
    size=1500,           # Больше для сохранения технического контекста
    overlap=200,         # Больше overlap для терминологии
    strategy="tokens"
)

# Для новостей/блогов
ChunkingConfig(
    size=1000,           # Стандартный размер
    overlap=100,         # Стандартный overlap
    strategy="tokens"
)

# Для диалогов/чатов
ChunkingConfig(
    size=800,            # Меньше для коротких реплик
    overlap=50,          # Минимальный overlap
    strategy="sentence"  # По предложениям для сохранения реплик
)
```

## Примеры разбиения

### Пример 1: Technical Document

**Исходный текст** (фрагмент):
```
GraphRAG is a knowledge graph construction framework. It uses Large Language
Models (LLMs) to extract entities and relationships from unstructured text.
The extraction process involves multiple steps: chunking, entity extraction,
relationship extraction, and community detection.
```

**Chunking с size=30, overlap=10**:

Chunk 1:
```
GraphRAG is a knowledge graph construction framework. It uses Large Language
Models (LLMs) to extract entities and
```

Chunk 2 (с overlap):
```
to extract entities and relationships from unstructured text.
The extraction process involves multiple steps:
```

Chunk 3 (с overlap):
```
multiple steps: chunking, entity extraction,
relationship extraction, and community detection.
```

### Пример 2: Document metadata preservation

```python
# Если prepend_metadata = True:

# Исходный document с metadata
document = {
    "id": "doc_123",
    "title": "GraphRAG Architecture",
    "date": "2024-01-15",
    "text": "GraphRAG is a..."
}

# Chunk с prepended metadata:
chunk_text = """
---
Document: GraphRAG Architecture
Date: 2024-01-15
Source: doc_123
---

GraphRAG is a...
"""
```

## Обработка edge cases

### Очень длинные документы

Для документов, превышающих разумный размер:

```python
# Рекомендация: pre-split по разделам
if document_length > 100_000 tokens:
    # 1. Разбить документ по структурным элементам (chapters, sections)
    # 2. Создать sub-documents
    # 3. Применить chunking к каждому sub-document
```

### Очень короткие документы

```python
# Если document < chunk_size:
# → Создается один chunk = весь документ
# → Overlap не применяется
```

### Специальные символы и форматирование

```python
# Сохранение code blocks, формул, таблиц:
# → Tokenizer автоматически обрабатывает Unicode
# → Рекомендуется preprocessing для сохранения структуры

def preprocess_document(text: str) -> str:
    # Защита code blocks
    text = protect_code_blocks(text)
    # Защита формул (LaTeX)
    text = protect_formulas(text)
    return text
```

## Выходные данные

### Схема text_units.parquet

```python
COLUMNS = [
    "id",              # str: SHA512 hash chunk
    "text",            # str: текст chunk
    "n_tokens",        # int: количество токенов
    "document_ids",    # list[str]: ID исходных документов
    "metadata",        # dict: дополнительные метаданные (опционально)
]
```

**Файл схемы**: `graphrag/data_model/schemas.py`

### Статистика разбиения

После выполнения workflow доступна статистика:

```python
stats = {
    "total_documents": 150,
    "total_text_units": 3420,
    "avg_chunks_per_document": 22.8,
    "avg_tokens_per_chunk": 1187,
    "total_tokens": 4_060_000
}
```

## Следующий этап

После создания text_units следующий workflow:
- **create_final_documents**: Финализация документов с добавлением связей к text_units
- **extract_graph**: Извлечение сущностей и отношений из каждого text_unit

→ [Переход к извлечению сущностей и отношений](02-entity-relationship-extraction.md)
