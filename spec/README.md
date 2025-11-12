# GraphRAG Document Processing Pipeline - Техническая спецификация

## О документации

Эта директория содержит полную техническую спецификацию GraphRAG Document Processing Pipeline с подробным описанием всех этапов обработки и индексации документов, включая работу с узлами графа, формирование document chunks с учетом семантики, выделение концептуального слоя и использование языковых моделей.

## Структура документации

### Обязательное чтение

1. **[00-overview.md](00-overview.md)** - Общий обзор pipeline
   - Архитектурная концепция
   - Основные этапы обработки
   - Режимы работы (Standard/Fast)
   - Концептуальный слой

### Этапы Pipeline (в порядке выполнения)

2. **[01-document-ingestion-chunking.md](01-document-ingestion-chunking.md)** - Разбиение документов на chunks
   - Процесс chunking с учетом семантики
   - Стратегии разбиения (tokens, sentence, paragraph)
   - Семантическое перекрытие (overlap)
   - Генерация ID и метаданные text_units

3. **[02-entity-relationship-extraction.md](02-entity-relationship-extraction.md)** - Извлечение сущностей и отношений
   - **Extraction Agent (LLM)**: Извлечение entities и relationships
   - Процесс Gleaning для улучшения результатов
   - **Summarization Agent (LLM)**: Консолидация описаний
   - Парсинг и агрегация результатов

4. **[03-graph-construction.md](03-graph-construction.md)** - Построение графа знаний
   - Создание NetworkX графа из entities и relationships
   - Вычисление метрик узлов (node_degree, node_frequency)
   - Layout: позиционирование узлов для визуализации
   - Graph pruning и оптимизация

5. **[04-community-detection.md](04-community-detection.md)** - Обнаружение тематических сообществ
   - Алгоритм Leiden Clustering
   - Иерархическая структура сообществ (multi-level)
   - Агрегация данных по сообществам
   - Анализ качества кластеризации

6. **[05-community-reports.md](05-community-reports.md)** - Генерация отчетов о сообществах
   - **Community Report Agent (LLM)**: Создание structured reports
   - Построение контекста сообщества
   - Структура отчета (Title, Summary, Findings, Rating)
   - Иерархические отчеты для разных уровней

7. **[06-embeddings-vectorization.md](06-embeddings-vectorization.md)** - Векторизация для семантического поиска
   - Text Embeddings для всех компонентов
   - **Embedding Model**: text-embedding-3-small
   - Graph Embeddings (Node2Vec)
   - Vector Store integration (LanceDB, Azure AI Search)

### Системные аспекты

8. **[07-llm-agents-roles.md](07-llm-agents-roles.md)** - Роли языковых моделей и агентов
   - **Extraction Agent**: Извлечение структурированных данных
   - **Summarization Agent**: Консолидация описаний
   - **Community Report Agent**: Генерация аналитических отчетов
   - **Embedding Model**: Векторизация текстов
   - Оптимизация затрат и производительности

9. **[08-data-flow.md](08-data-flow.md)** - Поток данных через pipeline
   - Схема полного потока от input до output
   - Трансформации данных на каждом этапе
   - Зависимости и параллелизация
   - Статистика типичного запуска

10. **[09-configuration.md](09-configuration.md)** - Конфигурация системы
    - Структура конфигурации
    - Примеры конфигураций (минимальная, production, Azure)
    - Environment variables
    - Оптимальные настройки для разных типов документов

## Ключевые концепции

### Концептуальный слой (Semantic Layer)

GraphRAG создает многоуровневую семантическую структуру:

```
Уровень 1: Text Units (текстовые фрагменты)
    ↓
Уровень 2: Entities (концепты и сущности)
    ↓
Уровень 3: Relationships (связи между концептами)
    ↓
Уровень 4: Communities (тематические кластеры)
    ↓
Уровень 5: Community Reports (высокоуровневые резюме)
```

### Языковые агенты

GraphRAG использует 4 типа LLM-агентов:

1. **Extraction Agent** (70% стоимости)
   - Функция: Извлечение entities и relationships из text_units
   - Модель: GPT-4
   - Промпт: Структурированный extraction template

2. **Summarization Agent** (5-10% стоимости)
   - Функция: Консолидация множественных описаний
   - Модель: GPT-3.5-turbo или GPT-4
   - Применение: Entity descriptions

3. **Community Report Agent** (20-25% стоимости)
   - Функция: Генерация structured reports о сообществах
   - Модель: GPT-4
   - Выход: JSON с title, summary, findings, rating

4. **Embedding Model** (~5% стоимости)
   - Функция: Векторизация текстов для retrieval
   - Модель: text-embedding-3-small
   - Применение: Все текстовые компоненты

### Семантическое разбиение (Semantic Chunking)

```python
# Принципы chunking с учетом семантики:

1. Token-based Strategy (по умолчанию):
   - Размер: 1200 tokens
   - Overlap: 100 tokens
   - Сохранение контекста на границах

2. Overlap для предотвращения потери информации:
   - Сущности на границах chunks сохраняются полностью
   - Отношения между chunks не теряются

3. Метаданные для отслеживания:
   - document_ids: связь с исходными документами
   - n_tokens: количество токенов
   - id: детерминированный SHA512 hash
```

### Граф знаний

```python
# Структура графа:

G = (V, E)

V (Vertices) = Entities
  - Properties: title, type, description
  - Metrics: node_degree, node_frequency
  - Position: node_x, node_y (для визуализации)

E (Edges) = Relationships
  - Properties: source, target, description, weight
  - Metrics: combined_degree (source_degree + target_degree)
```

## Быстрый старт

### 1. Минимальная конфигурация

Создайте `settings.yaml`:

```yaml
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

### 2. Запуск pipeline

```bash
# Установить environment variable
export OPENAI_API_KEY="sk-..."

# Запустить indexing
graphrag index --config settings.yaml
```

### 3. Выходные данные

После выполнения в директории `output/` будут созданы:

```
output/
├── documents.parquet
├── text_units.parquet
├── entities.parquet
├── relationships.parquet
├── communities.parquet
├── community_reports.parquet
└── embeddings/
    ├── document_text_embedding.parquet
    ├── text_unit_text_embedding.parquet
    ├── entity_description_embedding.parquet
    └── community_full_content_embedding.parquet
```

## Типичная производительность

Для 100 документов (~500K tokens):

```
Этап                    Время       Стоимость   LLM Calls
─────────────────────────────────────────────────────────
Chunking                10 сек      -           -
Entity Extraction       28 мин      $35         420
Summarization           2 мин       $2          850
Graph Construction      5 сек       -           -
Community Detection     8 сек       -           -
Community Reports       5 мин       $18         79
Embeddings              45 сек      $1.50       4849
─────────────────────────────────────────────────────────
TOTAL                   ~35 мин     ~$56.50     6198
```

## Оптимизация

### Для production

```yaml
# Оптимизированная конфигурация

extract_graph:
  model_id: gpt-4-turbo      # Быстрее и дешевле
  max_gleanings: 0           # Без gleaning

models:
  summarization:
    model: gpt-3.5-turbo     # 10x дешевле для summarization

embed_text:
  model_id: text-embedding-3-small  # Оптимальное качество/цена
  batch_size: 500
```

### Fast Mode

Для снижения стоимости на 60-70%:

```yaml
# Использует NLP вместо LLM где возможно

extract_graph:
  strategy:
    type: nltk  # NLP-based extraction

community_reports:
  strategy:
    type: text_units  # Упрощенная суммаризация
```

## Схема Pipeline

```
INPUT DOCUMENTS
    ↓
[1. Chunking] → text_units.parquet
    ↓
[2. Entity Extraction (LLM)] → entities.parquet + relationships.parquet
    ↓
[3. Graph Construction] → NetworkX Graph
    ↓
[4. Community Detection (Leiden)] → communities.parquet
    ↓
[5. Community Reports (LLM)] → community_reports.parquet
    ↓
[6. Embeddings] → embeddings/*.parquet
    ↓
[7. Vector Store] → LanceDB / Azure AI Search
    ↓
OUTPUT: Queryable Knowledge Base
```

## Ключевые файлы кодовой базы

Основные компоненты реализации:

```
graphrag/
├── index/
│   ├── workflows/                    # Pipeline workflows
│   │   ├── create_base_text_units.py
│   │   ├── extract_graph.py
│   │   ├── create_communities.py
│   │   └── create_community_reports.py
│   ├── operations/                   # Core operations
│   │   ├── chunk_text/
│   │   ├── extract_graph/
│   │   ├── cluster_graph.py
│   │   └── embed_text/
│   └── run/
│       └── run_pipeline.py           # Pipeline runner
├── prompts/
│   └── index/
│       ├── extract_graph.py          # Extraction prompts
│       ├── summarize_descriptions.py
│       └── community_report.py       # Report prompts
├── config/
│   └── models/                       # Configuration models
│       ├── graph_rag_config.py
│       ├── chunking_config.py
│       └── extract_graph_config.py
└── data_model/                       # Data schemas
    ├── entity.py
    ├── relationship.py
    └── community.py
```

## Дополнительные ресурсы

- **GraphRAG GitHub**: https://github.com/microsoft/graphrag
- **Документация**: https://microsoft.github.io/graphrag/
- **Paper**: "From Local to Global: A Graph RAG Approach to Query-Focused Summarization"

## Глоссарий

- **Text Unit**: Текстовый фрагмент (chunk) после разбиения документа
- **Entity**: Сущность (концепт, объект, персона, организация)
- **Relationship**: Отношение между двумя сущностями
- **Community**: Тематический кластер связанных сущностей
- **Community Report**: Структурированный отчет о сообществе
- **Node Degree**: Степень узла в графе (количество связей)
- **Node Frequency**: Частота упоминаний сущности в документах
- **Leiden Clustering**: Алгоритм иерархической кластеризации графов
- **Embedding**: Векторное представление текста
- **LLM Agent**: Специализированный агент на основе языковой модели
- **Gleaning**: Итеративное улучшение результатов extraction
- **Vector Store**: Хранилище векторных представлений для поиска

## Контакты и вопросы

Эта документация создана на основе анализа кодовой базы GraphRAG.

Для вопросов по использованию GraphRAG обращайтесь к официальной документации проекта.

---

**Версия документации**: 1.0
**Дата создания**: 2025-11-12
**Основано на**: GraphRAG 2.x
