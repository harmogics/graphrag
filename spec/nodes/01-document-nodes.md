# Document Nodes (Узлы документов)

## Обзор

**Document Nodes** — это корневой уровень в иерархии GraphRAG, представляющий исходные документы, загруженные в систему. Каждый документ является отправной точкой для всех последующих преобразований и извлечений.

## Структура Document Node

### Атрибуты

```python
class Document:
    # Идентификация
    id: str                         # Уникальный ID документа
    human_readable_id: str          # Человекочитаемый ID ("doc_0001")

    # Основное содержимое
    title: str                      # Название документа
    text: str                       # Полный текст документа

    # Связи
    text_unit_ids: list[str]        # IDs всех text_units из этого документа

    # Метаданные
    metadata: dict | None           # Дополнительные метаданные
    creation_date: str | None       # Дата создания

    # Источник
    source: str | None              # Путь к исходному файлу
```

**Файл данных**: `output/documents.parquet`

## Роль в архитектуре

### Позиция в иерархии

```
┌─────────────────────────────────────────────┐
│          INPUT FILES                        │
│  doc1.txt, doc2.pdf, doc3.md                │
└──────────────────┬──────────────────────────┘
                   │ Loading
                   ↓
┌─────────────────────────────────────────────┐
│       DOCUMENT NODES (Вы здесь)             │
│  • Нормализованное представление            │
│  • Единая структура для всех форматов       │
└──────────────────┬──────────────────────────┘
                   │ Chunking
                   ↓
┌─────────────────────────────────────────────┐
│          TEXT UNIT NODES                    │
│  Фрагменты для обработки                   │
└─────────────────────────────────────────────┘
```

### Связи с другими узлами

```
Document
   ├─► has_chunks → Text Unit (1:N)
   │
   └─► referenced_by → Entity (indirect, через Text Units)
```

## Создание Document Nodes

### Workflow: create_base_documents

```python
# graphrag/index/workflows/create_base_documents.py

async def create_base_documents(
    config: GraphRagConfig,
    context: PipelineRunContext
) -> pd.DataFrame:
    """
    Загружает документы из input source и создает Document nodes

    Process:
    1. Загрузить файлы из input directory
    2. Парсить каждый файл в зависимости от типа (txt, csv, json, pdf)
    3. Нормализовать в единую структуру
    4. Генерировать ID для каждого документа
    5. Сохранить в documents.parquet
    """

    # 1. Загрузка файлов
    files = load_input_files(config.input)

    documents = []
    for file in files:
        # 2. Парсинг
        content = parse_file(file)

        # 3. Создание Document node
        doc = Document(
            id=generate_document_id(file),
            title=extract_title(content, file),
            text=content,
            source=file.path,
            metadata=extract_metadata(file)
        )

        documents.append(doc)

    # 4. Сохранение
    df = pd.DataFrame(documents)
    await context.storage.set("documents", df)

    return df
```

**Файл**: `graphrag/index/workflows/create_base_documents.py`

## Типы источников документов

### 1. Text Files (.txt, .md)

```python
# Простейший случай - прямое чтение
with open("document.txt", "r", encoding="utf-8") as f:
    text = f.read()

document = Document(
    id=generate_id("document.txt"),
    title="document.txt",
    text=text
)
```

### 2. CSV Files

```python
# Чтение CSV с указанием колонок
df = pd.read_csv("documents.csv")

documents = []
for _, row in df.iterrows():
    document = Document(
        id=row["id"],
        title=row.get("title", ""),
        text=row["text"],
        metadata={
            col: row[col]
            for col in df.columns
            if col not in ["id", "title", "text"]
        }
    )
    documents.append(document)
```

**Конфигурация**:
```yaml
input:
  type: file
  file_type: csv
  base_dir: "input"
  source_column: "text"
  title_column: "title"
  timestamp_column: "date"
```

### 3. JSON Files

```python
# Чтение JSON документов
with open("documents.json", "r") as f:
    data = json.load(f)

documents = []
for item in data:
    document = Document(
        id=item["id"],
        title=item["title"],
        text=item["content"],
        metadata=item.get("metadata", {})
    )
    documents.append(document)
```

### 4. PDF Files

```python
# Парсинг PDF
import PyPDF2

with open("document.pdf", "rb") as f:
    pdf_reader = PyPDF2.PdfReader(f)

    # Извлечь текст со всех страниц
    text = ""
    for page in pdf_reader.pages:
        text += page.extract_text()

    document = Document(
        id=generate_id("document.pdf"),
        title=extract_pdf_title(pdf_reader),
        text=text,
        metadata={
            "num_pages": len(pdf_reader.pages),
            "author": pdf_reader.metadata.get("/Author", ""),
        }
    )
```

## Метаданные документов

### Стандартные метаданные

```python
metadata = {
    # Временные метки
    "creation_date": "2024-01-15T10:30:00Z",
    "modification_date": "2024-01-20T15:45:00Z",

    # Источник
    "source": "input/docs/document.txt",
    "source_type": "file",

    # Авторство
    "author": "John Doe",
    "organization": "Acme Corp",

    # Классификация
    "category": "technical",
    "tags": ["AI", "GraphRAG", "Knowledge Graphs"],

    # Технические
    "file_size": 15420,  # bytes
    "encoding": "utf-8",
    "language": "en"
}
```

### Использование метаданных в поиске

```python
# Фильтрация документов по метаданным
def filter_documents_by_metadata(
    documents: pd.DataFrame,
    filters: dict
) -> pd.DataFrame:
    """
    Фильтрует документы по метаданным

    Example:
        filtered = filter_documents_by_metadata(
            documents,
            {"category": "technical", "author": "John Doe"}
        )
    """
    for key, value in filters.items():
        documents = documents[
            documents["metadata"].apply(
                lambda m: m.get(key) == value if m else False
            )
        ]

    return documents
```

## Связь с Text Units

### От Document к Text Units

```python
# Получить все text_units документа
def get_text_units_for_document(
    document_id: str,
    text_units: pd.DataFrame
) -> pd.DataFrame:
    """
    Находит все text_units, принадлежащие документу
    """
    return text_units[
        text_units["document_ids"].apply(
            lambda ids: document_id in ids
        )
    ]

# Пример
document_id = "doc_0001"
units = get_text_units_for_document(document_id, text_units)

print(f"Document {document_id} has {len(units)} text units")
```

### От Text Unit к Document

```python
# Обратная связь: найти документ по text_unit
def get_documents_for_text_unit(
    text_unit_id: str,
    text_units: pd.DataFrame,
    documents: pd.DataFrame
) -> pd.DataFrame:
    """
    Находит документы, которым принадлежит text_unit
    """
    unit = text_units[text_units["id"] == text_unit_id].iloc[0]
    document_ids = unit["document_ids"]

    return documents[documents["id"].isin(document_ids)]
```

## Использование в Query Execution

### Роль в Local Search

Document nodes предоставляют контекст источника:

```python
# Local Search с указанием источника
def local_search_with_source(
    query: str,
    entities: pd.DataFrame,
    text_units: pd.DataFrame,
    documents: pd.DataFrame
) -> dict:
    """
    Local search с информацией об источниках
    """
    # 1. Найти релевантные entities
    relevant_entities = find_relevant_entities(query, entities)

    # 2. Найти text_units с этими entities
    relevant_units = find_text_units_for_entities(relevant_entities, text_units)

    # 3. Для каждого text_unit, получить документ-источник
    sources = []
    for unit_id in relevant_units["id"]:
        docs = get_documents_for_text_unit(unit_id, text_units, documents)
        sources.extend(docs["title"].tolist())

    # 4. Построить контекст с источниками
    context = build_context_with_sources(relevant_units, documents)

    # 5. LLM генерирует ответ
    answer = llm_generate(query, context)

    return {
        "answer": answer,
        "sources": list(set(sources)),  # Уникальные источники
        "num_documents": len(set(sources))
    }
```

### Роль в Global Search

Document nodes используются опосредованно через communities:

```python
# Трассировка от Community Report до Documents
def trace_community_to_documents(
    community_id: str,
    communities: pd.DataFrame,
    entities: pd.DataFrame,
    text_units: pd.DataFrame,
    documents: pd.DataFrame
) -> list[str]:
    """
    Находит все документы, связанные с сообществом
    """
    # 1. Получить entity_ids сообщества
    community = communities[communities["id"] == community_id].iloc[0]
    entity_ids = community["entity_ids"]

    # 2. Получить text_unit_ids для этих entities
    community_entities = entities[entities["id"].isin(entity_ids)]
    text_unit_ids = set()
    for entity_text_units in community_entities["text_unit_ids"]:
        text_unit_ids.update(entity_text_units)

    # 3. Получить document_ids для text_units
    community_units = text_units[text_units["id"].isin(text_unit_ids)]
    document_ids = set()
    for unit_doc_ids in community_units["document_ids"]:
        document_ids.update(unit_doc_ids)

    # 4. Получить названия документов
    community_docs = documents[documents["id"].isin(document_ids)]
    return community_docs["title"].tolist()
```

## Статистика документов

### Анализ коллекции

```python
def analyze_document_collection(documents: pd.DataFrame) -> dict:
    """
    Анализирует коллекцию документов
    """
    import tiktoken
    encoding = tiktoken.get_encoding("cl100k_base")

    stats = {
        "total_documents": len(documents),
        "total_chars": documents["text"].apply(len).sum(),
        "total_tokens": documents["text"].apply(
            lambda t: len(encoding.encode(t))
        ).sum(),
        "avg_tokens_per_doc": 0,
        "min_tokens": 0,
        "max_tokens": 0,
        "documents_by_size": {}
    }

    token_counts = documents["text"].apply(lambda t: len(encoding.encode(t)))

    stats["avg_tokens_per_doc"] = token_counts.mean()
    stats["min_tokens"] = token_counts.min()
    stats["max_tokens"] = token_counts.max()

    # Распределение по размеру
    stats["documents_by_size"] = {
        "small (<1000 tokens)": len(token_counts[token_counts < 1000]),
        "medium (1000-5000)": len(token_counts[
            (token_counts >= 1000) & (token_counts < 5000)
        ]),
        "large (5000-20000)": len(token_counts[
            (token_counts >= 5000) & (token_counts < 20000)
        ]),
        "very_large (>20000)": len(token_counts[token_counts >= 20000])
    }

    return stats

# Пример использования
stats = analyze_document_collection(documents)
print(f"Total documents: {stats['total_documents']}")
print(f"Average tokens per document: {stats['avg_tokens_per_doc']:.0f}")
print(f"Distribution: {stats['documents_by_size']}")
```

## Document Embeddings

### Создание embeddings

```python
# Document embeddings используются для document-level retrieval
document_embeddings = create_embeddings(
    documents["text"].tolist(),
    model="text-embedding-3-small"
)

# Сохранить
embeddings_df = pd.DataFrame({
    "id": documents["id"],
    "embedding": document_embeddings
})

embeddings_df.to_parquet("embeddings.document_text_embedding.parquet")
```

### Семантический поиск по документам

```python
def search_documents_semantic(
    query: str,
    documents: pd.DataFrame,
    document_embeddings: pd.DataFrame,
    top_k: int = 5
) -> pd.DataFrame:
    """
    Семантический поиск по документам
    """
    # 1. Создать embedding для query
    query_embedding = create_embedding(query)

    # 2. Вычислить similarity со всеми документами
    similarities = document_embeddings["embedding"].apply(
        lambda emb: cosine_similarity(query_embedding, emb)
    )

    # 3. Топ-K документов
    top_indices = similarities.nlargest(top_k).index
    top_doc_ids = document_embeddings.loc[top_indices, "id"]

    # 4. Вернуть документы с scores
    results = documents[documents["id"].isin(top_doc_ids)].copy()
    results["similarity"] = results["id"].map(
        dict(zip(document_embeddings.loc[top_indices, "id"],
                 similarities.loc[top_indices]))
    )

    return results.sort_values("similarity", ascending=False)
```

## Обновление и версионирование

### Инкрементальное обновление

```python
def incremental_update_documents(
    existing_documents: pd.DataFrame,
    new_files: list[str]
) -> pd.DataFrame:
    """
    Добавляет новые документы без переобработки существующих
    """
    # 1. Загрузить только новые файлы
    new_documents = []

    for file_path in new_files:
        # Проверить, не обработан ли уже
        if is_document_processed(file_path, existing_documents):
            continue

        # Загрузить и создать Document node
        doc = load_and_create_document(file_path)
        new_documents.append(doc)

    # 2. Объединить с существующими
    if new_documents:
        new_df = pd.DataFrame(new_documents)
        updated_documents = pd.concat([existing_documents, new_df])
        return updated_documents

    return existing_documents
```

### Версионирование документов

```python
class DocumentVersion:
    """Версионированный документ"""
    id: str
    version: int
    text: str
    previous_version: str | None
    changes: dict
    timestamp: str

# При обновлении документа
def update_document_with_versioning(
    document_id: str,
    new_text: str,
    documents: pd.DataFrame
) -> Document:
    """
    Обновляет документ с сохранением версии
    """
    old_doc = documents[documents["id"] == document_id].iloc[0]

    new_version = DocumentVersion(
        id=f"{document_id}_v{old_doc.get('version', 0) + 1}",
        version=old_doc.get('version', 0) + 1,
        text=new_text,
        previous_version=document_id,
        changes=compute_diff(old_doc["text"], new_text),
        timestamp=datetime.now().isoformat()
    )

    return new_version
```

## Best Practices

### 1. Нормализация текста

```python
def normalize_document_text(text: str) -> str:
    """
    Нормализует текст документа
    """
    # Удалить лишние пробелы
    text = re.sub(r'\s+', ' ', text)

    # Удалить невидимые символы
    text = ''.join(char for char in text if char.isprintable() or char.isspace())

    # Нормализовать переносы строк
    text = text.replace('\r\n', '\n')

    return text.strip()
```

### 2. Извлечение метаданных

```python
def extract_rich_metadata(file_path: str, text: str) -> dict:
    """
    Извлекает расширенные метаданные
    """
    import magic
    from datetime import datetime

    metadata = {
        # Файловая система
        "file_path": file_path,
        "file_name": os.path.basename(file_path),
        "file_size": os.path.getsize(file_path),
        "file_type": magic.from_file(file_path, mime=True),
        "created_at": datetime.fromtimestamp(
            os.path.getctime(file_path)
        ).isoformat(),
        "modified_at": datetime.fromtimestamp(
            os.path.getmtime(file_path)
        ).isoformat(),

        # Содержимое
        "char_count": len(text),
        "word_count": len(text.split()),
        "line_count": text.count('\n') + 1,

        # Язык (опционально, требует langdetect)
        # "language": detect_language(text),
    }

    return metadata
```

### 3. Обработка ошибок

```python
def safe_load_document(file_path: str) -> Document | None:
    """
    Безопасная загрузка документа с обработкой ошибок
    """
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            text = f.read()

        return Document(
            id=generate_id(file_path),
            title=os.path.basename(file_path),
            text=normalize_document_text(text),
            source=file_path
        )

    except UnicodeDecodeError:
        # Попробовать другие кодировки
        try:
            with open(file_path, 'r', encoding='latin-1') as f:
                text = f.read()
            return Document(...)
        except Exception as e:
            log.error(f"Failed to load {file_path}: {e}")
            return None

    except Exception as e:
        log.error(f"Error loading {file_path}: {e}")
        return None
```

## Следующие разделы

- **[Text Unit Nodes →](02-text-unit-nodes.md)**: Фрагменты документов
- **[Entity Nodes →](03-entity-nodes.md)**: Извлеченные сущности
- **[Node Interactions →](08-node-interactions.md)**: Связи между узлами
