# Конфигурация GraphRAG

## Обзор системы конфигурации

GraphRAG использует **иерархическую систему конфигурации** на основе Pydantic моделей. Конфигурация может быть задана через:
- YAML файл (`settings.yaml`)
- Environment variables
- JSON файл
- Программный API

## Структура конфигурации

### GraphRagConfig (корневая конфигурация)

```python
# graphrag/config/models/graph_rag_config.py

class GraphRagConfig:
    # Root directory для всех путей
    root_dir: str = "."

    # Reporting
    reporting: ReportingConfig

    # Storage
    storage: StorageConfig

    # Cache
    cache: CacheConfig

    # Input
    input: InputConfig

    # Chunking
    chunking: ChunkingConfig

    # Extraction
    extract_graph: ExtractGraphConfig

    # Summarization
    summarize_descriptions: SummarizeDescriptionsConfig

    # Graph
    cluster_graph: ClusterGraphConfig
    embed_graph: EmbedGraphConfig

    # Community Reports
    community_reports: CommunityReportsConfig

    # Embeddings
    embed_text: TextEmbeddingConfig

    # Claims (опционально)
    extract_claims: ExtractClaimsConfig | None

    # LLM Models
    models: dict[str, LanguageModelConfig]

    # Vector Store
    vector_store: VectorStoreConfig | None

    # Encoding
    encoding_model: str = "cl100k_base"
```

**Файл**: `graphrag/config/models/graph_rag_config.py:25`

## Основные секции конфигурации

### 1. Input Configuration

```yaml
# settings.yaml

input:
  type: file  # file | blob | memory
  base_dir: "input"  # Директория с входными документами
  file_pattern: ".*\\.txt$"  # Regex pattern для файлов
  file_type: text  # text | csv | json
  encoding: utf-8

  # Для CSV
  source_column: "text"
  title_column: "title"
  timestamp_column: null

  # Для text files
  text_column: "text"
```

**Файл конфигурации**: `graphrag/config/models/input_config.py`

### 2. Chunking Configuration

```yaml
chunking:
  size: 1200          # Размер chunk в токенах
  overlap: 100        # Перекрытие между chunks
  group_by_columns:   # Группировка при chunking
    - id
  strategy: tokens    # tokens | sentence | paragraph
  encoding_model: cl100k_base
```

**Параметры**:
- `size`: 300-3000 tokens (оптимально: 1000-1500)
- `overlap`: 50-200 tokens (оптимально: 100)
- `strategy`: tokens (рекомендуется для LLM)

### 3. Extract Graph Configuration

```yaml
extract_graph:
  model_id: gpt-4                    # LLM модель для extraction
  entity_types:                      # Типы извлекаемых сущностей
    - organization
    - person
    - geo
    - event
  max_gleanings: 0                   # Итерации gleaning (0-3)
  prompt: null                       # Путь к кастомному промпту
  strategy:
    type: graph_intelligence
    llm:
      type: openai_chat
      model: gpt-4
      temperature: 0
      max_tokens: 4000
```

**Параметры**:
- `model_id`: gpt-4 (рекомендуется), gpt-4-turbo, gpt-3.5-turbo
- `max_gleanings`: 0 (по умолчанию), 1-2 (для лучшего recall)
- `entity_types`: Можно добавить кастомные типы

### 4. Cluster Graph Configuration

```yaml
cluster_graph:
  max_cluster_size: 10   # Максимальный размер сообщества
  use_lcc: true          # Использовать только LCC
  seed: null             # Random seed для воспроизводимости
```

**Параметры**:
- `max_cluster_size`: 5-20 (оптимально: 10)
- `use_lcc`: true (рекомендуется)

### 5. Community Reports Configuration

```yaml
community_reports:
  model_id: gpt-4                    # LLM для отчетов
  max_report_length: 2000            # Макс длина отчета (tokens)
  max_input_length: 8000             # Макс входной контекст
  prompt: null                       # Кастомный промпт
  strategy:
    type: graph_intelligence
    llm:
      type: openai_chat
      model: gpt-4
      temperature: 0
      max_tokens: 2000
```

**Параметры**:
- `model_id`: gpt-4 (обязательно для качества)
- `max_report_length`: 1500-3000 tokens
- `max_input_length`: 6000-10000 tokens

### 6. Text Embedding Configuration

```yaml
embed_text:
  model_id: text-embedding-3-small   # Embedding модель
  batch_size: 500                    # Размер батча
  batch_max_tokens: 10000            # Макс токены в батче
  target: all                        # all | required | selected | none
  names: []                          # Список конкретных embeddings
  vector_store_id: default           # Vector store для загрузки
  strategy:
    type: openai_embedding
    llm:
      type: openai_embedding
      model: text-embedding-3-small
```

**Параметры**:
- `model_id`: text-embedding-3-small (рекомендуется), text-embedding-3-large
- `batch_size`: 100-1000 (оптимально: 500)
- `target`: all (все embeddings), required (только для query)

### 7. Storage Configuration

```yaml
storage:
  type: file                         # file | blob | memory
  base_dir: "output"                 # Директория для выхода
  connection_string: null            # Для blob storage
  container_name: null
```

**Типы storage**:
- `file`: Локальная файловая система (по умолчанию)
- `blob`: Azure Blob Storage
- `memory`: In-memory (для тестирования)

### 8. Cache Configuration

```yaml
cache:
  type: file                         # file | blob | memory | none
  base_dir: "cache"                  # Директория кэша
  connection_string: null
  container_name: null
```

**Важность кэша**:
- Кэширование всех LLM-вызовов для идемпотентности
- Значительное снижение затрат при повторных запусках
- Восстановление после сбоев

### 9. Language Model Configuration

```yaml
models:
  default:                           # Модель по умолчанию
    type: openai_chat
    model: gpt-4
    api_key: ${OPENAI_API_KEY}       # Environment variable
    api_base: null
    api_version: null
    temperature: 0
    max_tokens: 4000
    concurrent_requests: 3           # Параллельные запросы
    encoding_model: cl100k_base

  extraction:                        # Специфичная модель для extraction
    type: openai_chat
    model: gpt-4-turbo
    temperature: 0
    max_tokens: 4000

  embedding:                         # Модель для embeddings
    type: openai_embedding
    model: text-embedding-3-small
    api_key: ${OPENAI_API_KEY}
```

**Поддерживаемые типы моделей**:
- `openai_chat`: OpenAI GPT models
- `azure_openai_chat`: Azure OpenAI
- `openai_embedding`: OpenAI Embeddings
- `azure_openai_embedding`: Azure OpenAI Embeddings

### 10. Vector Store Configuration

```yaml
vector_store:
  type: lancedb                      # lancedb | azure_ai_search | chromadb
  container_name: embeddings
  overwrite: true                    # Перезаписать при повторном запуске

  # Для Azure AI Search
  azure_search_endpoint: null
  azure_search_key: null

  # Для других
  connection_string: null
```

## Примеры конфигураций

### Минимальная конфигурация

```yaml
# settings.yaml

input:
  type: file
  base_dir: "input"

models:
  default:
    type: openai_chat
    model: gpt-4
    api_key: ${OPENAI_API_KEY}

storage:
  type: file
  base_dir: "output"

cache:
  type: file
  base_dir: "cache"
```

### Конфигурация для production

```yaml
# settings.yaml

root_dir: "."

input:
  type: file
  base_dir: "input"
  file_pattern: ".*\\.(txt|md|pdf)$"
  encoding: utf-8

chunking:
  size: 1200
  overlap: 100
  strategy: tokens

extract_graph:
  model_id: gpt-4-turbo
  entity_types:
    - organization
    - person
    - geo
    - event
    - product
    - technology
  max_gleanings: 0

cluster_graph:
  max_cluster_size: 10
  use_lcc: true

community_reports:
  model_id: gpt-4
  max_report_length: 2000
  max_input_length: 8000

embed_text:
  model_id: text-embedding-3-small
  batch_size: 500
  target: all

models:
  default:
    type: openai_chat
    model: gpt-4-turbo
    api_key: ${OPENAI_API_KEY}
    temperature: 0
    max_tokens: 4000
    concurrent_requests: 5

  summarization:
    type: openai_chat
    model: gpt-3.5-turbo  # Экономия
    temperature: 0
    max_tokens: 500

  embedding:
    type: openai_embedding
    model: text-embedding-3-small
    api_key: ${OPENAI_API_KEY}

storage:
  type: file
  base_dir: "output"

cache:
  type: file
  base_dir: "cache"

vector_store:
  type: lancedb
  container_name: embeddings
  overwrite: false

encoding_model: cl100k_base
```

### Azure OpenAI конфигурация

```yaml
models:
  default:
    type: azure_openai_chat
    model: gpt-4
    api_base: https://YOUR_RESOURCE.openai.azure.com/
    api_version: "2024-02-15-preview"
    deployment_name: gpt-4
    api_key: ${AZURE_OPENAI_API_KEY}
    # Или использовать managed identity:
    # auth_type: azure_managed_identity

  embedding:
    type: azure_openai_embedding
    model: text-embedding-3-small
    api_base: https://YOUR_RESOURCE.openai.azure.com/
    api_version: "2024-02-15-preview"
    deployment_name: text-embedding-3-small
    api_key: ${AZURE_OPENAI_API_KEY}
```

### Fast Mode конфигурация (экономия)

```yaml
# Использует NLP вместо LLM где возможно

extract_graph:
  strategy:
    type: nltk  # Или spacy - NLP-based extraction

community_reports:
  strategy:
    type: text_units  # Text-based вместо LLM

embed_text:
  target: required  # Только обязательные embeddings

models:
  default:
    model: gpt-3.5-turbo  # Более дешевая модель
```

## Environment Variables

### Основные переменные

```bash
# OpenAI
export OPENAI_API_KEY="sk-..."
export OPENAI_ORG_ID="org-..."  # Опционально

# Azure OpenAI
export AZURE_OPENAI_API_KEY="..."
export AZURE_OPENAI_ENDPOINT="https://..."

# Azure Storage (если используется)
export AZURE_STORAGE_CONNECTION_STRING="..."

# GraphRAG specific
export GRAPHRAG_CONFIG_PATH="./settings.yaml"
export GRAPHRAG_ROOT_DIR="."
```

### Использование в конфигурации

```yaml
models:
  default:
    api_key: ${OPENAI_API_KEY}        # Из environment
    # Или явно:
    # api_key: "sk-..."
```

## Programmatic Configuration

### Python API

```python
from graphrag.config.models.graph_rag_config import GraphRagConfig
from graphrag.config.models.language_model_config import LanguageModelConfig

# Создать конфигурацию программно
config = GraphRagConfig(
    root_dir=".",
    models={
        "default": LanguageModelConfig(
            type="openai_chat",
            model="gpt-4",
            api_key="sk-...",
            temperature=0,
            max_tokens=4000
        )
    },
    chunking=ChunkingConfig(
        size=1200,
        overlap=100
    ),
    extract_graph=ExtractGraphConfig(
        model_id="default",
        entity_types=["organization", "person", "geo"]
    )
    # ... и т.д.
)

# Сохранить в YAML
config.save("settings.yaml")

# Загрузить из YAML
config = GraphRagConfig.load("settings.yaml")
```

## Defaults и рекомендации

### Для научных/технических документов

```yaml
chunking:
  size: 1500          # Больше для технического контекста
  overlap: 200        # Больше для терминологии

extract_graph:
  entity_types:
    - organization
    - person
    - technology
    - concept
    - method
  max_gleanings: 1    # Лучший recall для технических терминов
```

### Для новостей/блогов

```yaml
chunking:
  size: 1000          # Стандартный размер
  overlap: 100

extract_graph:
  entity_types:
    - organization
    - person
    - geo
    - event
  max_gleanings: 0    # Обычно достаточно одной итерации
```

### Для диалогов/чатов

```yaml
chunking:
  size: 800           # Меньше для коротких сообщений
  overlap: 50
  strategy: sentence  # По предложениям для сохранения реплик

extract_graph:
  entity_types:
    - person
    - organization
    - event
```

## Валидация конфигурации

### Автоматическая валидация

```python
# Pydantic автоматически валидирует:
# - Типы данных
# - Required fields
# - Допустимые значения (через validators)

try:
    config = GraphRagConfig.load("settings.yaml")
except ValidationError as e:
    print(f"Configuration error: {e}")
```

### Custom валидация

```python
def validate_config(config: GraphRagConfig) -> list[str]:
    """Проверяет конфигурацию на типичные проблемы"""
    issues = []

    # Проверка chunk size vs LLM context
    if config.chunking.size > 3000:
        issues.append("Chunk size too large for most LLMs")

    # Проверка API keys
    if not config.models["default"].api_key:
        issues.append("Missing API key for default model")

    # Проверка overlap
    if config.chunking.overlap >= config.chunking.size:
        issues.append("Overlap must be smaller than chunk size")

    return issues
```

## Migration и версионирование

### Обратная совместимость

```python
# GraphRAG поддерживает миграцию старых конфигураций

# v1.0 config
old_config = {
    "llm_model": "gpt-4",
    "chunk_size": 1200
}

# Автоматически мигрируется в v2.0 format
new_config = migrate_config(old_config)
```

### Версия конфигурации

```yaml
version: "2.0"  # Версия schema конфигурации

# Остальная конфигурация...
```

## Troubleshooting

### Частые проблемы

```python
COMMON_ISSUES = {
    "API key not found": {
        "error": "Missing or invalid API key",
        "solution": "Set OPENAI_API_KEY environment variable or in config"
    },
    "Rate limit exceeded": {
        "error": "429 Too Many Requests",
        "solution": "Reduce concurrent_requests in models config"
    },
    "Out of memory": {
        "error": "MemoryError during processing",
        "solution": "Reduce batch_size for embeddings or chunking.size"
    },
    "Cache corruption": {
        "error": "Invalid cache data",
        "solution": "Clear cache directory and rerun"
    }
}
```

## Резюме: Оптимальная конфигурация

```yaml
# Рекомендуемая конфигурация для большинства случаев

input:
  type: file
  base_dir: "input"

chunking:
  size: 1200
  overlap: 100
  strategy: tokens

extract_graph:
  model_id: gpt-4-turbo
  max_gleanings: 0
  entity_types:
    - organization
    - person
    - geo
    - event

cluster_graph:
  max_cluster_size: 10
  use_lcc: true

community_reports:
  model_id: gpt-4
  max_report_length: 2000

embed_text:
  model_id: text-embedding-3-small
  batch_size: 500
  target: all

models:
  default:
    type: openai_chat
    model: gpt-4-turbo
    temperature: 0
    max_tokens: 4000
    concurrent_requests: 3

  summarization:
    model: gpt-3.5-turbo  # Экономия

  embedding:
    type: openai_embedding
    model: text-embedding-3-small

storage:
  type: file
  base_dir: "output"

cache:
  type: file
  base_dir: "cache"

vector_store:
  type: lancedb
  container_name: embeddings
```

## Заключение

Документация по GraphRAG pipeline завершена. Все основные аспекты системы описаны:

1. [Общий обзор](00-overview.md)
2. [Разбиение документов](01-document-ingestion-chunking.md)
3. [Извлечение сущностей и отношений](02-entity-relationship-extraction.md)
4. [Построение графа](03-graph-construction.md)
5. [Обнаружение сообществ](04-community-detection.md)
6. [Генерация отчетов](05-community-reports.md)
7. [Embeddings](06-embeddings-vectorization.md)
8. [Роли LLM агентов](07-llm-agents-roles.md)
9. [Поток данных](08-data-flow.md)
10. [Конфигурация](09-configuration.md) ← Вы здесь
