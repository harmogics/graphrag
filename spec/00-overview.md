# GraphRAG Document Processing Pipeline - Общий обзор

## Введение

GraphRAG (Graph Retrieval-Augmented Generation) — это система для построения графов знаний из неструктурированного текста с использованием больших языковых моделей (LLM). Система преобразует коллекцию документов в многоуровневый граф знаний с семантическими embeddings для эффективного поиска и генерации ответов.

## Архитектурная концепция

GraphRAG основан на **асинхронном модульном pipeline**, состоящем из последовательности взаимосвязанных workflows. Каждый workflow выполняет специфическую задачу обработки данных и передает результаты следующему этапу.

```
INPUT DOCUMENTS (текст)
        ↓
╔═══════════════════════════════════════╗
║     GRAPHRAG INDEXING PIPELINE        ║
╠═══════════════════════════════════════╣
║ 1. Document Chunking                  ║
║ 2. Entity & Relationship Extraction   ║
║ 3. Graph Construction                 ║
║ 4. Community Detection                ║
║ 5. Community Summarization            ║
║ 6. Embedding Generation               ║
╚═══════════════════════════════════════╝
        ↓
OUTPUT: Knowledge Graph + Embeddings
```

## Основные этапы Pipeline

### Этап 1: Разбиение документов (Document Chunking)
- **Цель**: Разделить большие документы на управляемые текстовые фрагменты (chunks)
- **Метод**: Токенизация с учетом семантических границ и перекрытий
- **Выход**: Коллекция text_units с метаданными

### Этап 2: Извлечение сущностей и отношений (Entity & Relationship Extraction)
- **Цель**: Идентифицировать ключевые концепты и их связи в тексте
- **Метод**: LLM-based extraction с использованием специализированных промптов
- **Агенты**: Extraction Agent, Summarization Agent
- **Выход**: Entities (узлы графа) и Relationships (ребра графа)

### Этап 3: Построение графа знаний (Graph Construction)
- **Цель**: Создать структурированный граф из извлеченных данных
- **Метод**: NetworkX граф с взвешенными ребрами
- **Выход**: Граф G = (V, E), где V - сущности, E - отношения

### Этап 4: Обнаружение сообществ (Community Detection)
- **Цель**: Выявить тематические кластеры в графе знаний
- **Метод**: Иерархический алгоритм Leiden clustering
- **Выход**: Многоуровневая иерархия сообществ (communities)

### Этап 5: Суммаризация сообществ (Community Summarization)
- **Цель**: Создать человекочитаемые описания каждого сообщества
- **Метод**: LLM-based summarization с контекстом узлов и ребер
- **Агент**: Community Report Agent
- **Выход**: Структурированные отчеты о сообществах

### Этап 6: Генерация embeddings (Embedding Generation)
- **Цель**: Создать векторные представления для семантического поиска
- **Метод**: Text embeddings (OpenAI API) и Graph embeddings (Node2Vec)
- **Выход**: Векторные представления для всех основных объектов

## Режимы работы Pipeline

### Standard Mode (Полный режим)
Использует LLM для всех этапов extraction и summarization. Обеспечивает максимальное качество за счет более длительного времени обработки.

**Workflows**:
1. create_base_text_units
2. create_final_documents
3. extract_graph (LLM-based)
4. finalize_graph
5. extract_covariates (опционально)
6. create_communities
7. create_final_text_units
8. create_community_reports (LLM-based)
9. generate_text_embeddings

### Fast Mode (Быстрый режим)
Использует NLP-методы вместо LLM там, где возможно. Обеспечивает более быстрое выполнение с приемлемым качеством.

**Workflows**:
1. create_base_text_units
2. create_final_documents
3. extract_graph_nlp (NLP-based, без LLM)
4. prune_graph (обрезка слабых связей)
5. finalize_graph
6. create_communities
7. create_final_text_units
8. create_community_reports_text (упрощенная суммаризация)
9. generate_text_embeddings

## Роль языковых моделей

GraphRAG использует LLM на нескольких критических этапах:

### 1. Extraction LLM (Агент извлечения)
- **Функция**: Извлечение сущностей и отношений из текста
- **Промпт**: Структурированный template с примерами
- **Выход**: Списки entities и relationships в специальном формате

### 2. Summarization LLM (Агент суммаризации)
- **Функция**: Создание кратких описаний сущностей и отношений
- **Применение**: Consolidation множественных описаний одной сущности

### 3. Community Report LLM (Агент отчетов)
- **Функция**: Генерация структурированных отчетов о тематических кластерах
- **Промпт**: Контекст сообщества (узлы + ребра) → структурированный отчет
- **Выход**: Title, Summary, Findings, Rating

### 4. Embedding LLM (Модель векторизации)
- **Функция**: Преобразование текста в векторные представления
- **Модели**: text-embedding-3-small/large, ada-002
- **Применение**: Семантический поиск и retrieval

## Концептуальный слой (Semantic Layer)

GraphRAG создает многоуровневую семантическую структуру:

### Уровень 1: Текстовые фрагменты (Text Units)
- Базовые chunks с токенами и метаданными
- Прямая связь с исходными документами

### Уровень 2: Концепты (Entities)
- Извлеченные сущности с типами и описаниями
- Представляют ключевые концепты в предметной области

### Уровень 3: Связи (Relationships)
- Отношения между концептами
- Имеют веса (strength) и описания

### Уровень 4: Сообщества (Communities)
- Тематические кластеры связанных концептов
- Иерархическая структура (multi-level)

### Уровень 5: Отчеты (Community Reports)
- Высокоуровневые резюме тематических областей
- Findings с рейтингами важности

## Поток данных

```
Documents.txt
    ↓
[Chunking] → text_units.parquet
    ↓
[LLM Extraction] → entities.parquet + relationships.parquet
    ↓
[Graph Building] → NetworkX Graph
    ↓
[Leiden Clustering] → communities.parquet (multi-level)
    ↓
[LLM Summarization] → community_reports.parquet
    ↓
[Embedding] → embeddings/*.parquet
    ↓
[Vector Store] → Searchable Knowledge Base
```

## Ключевые особенности

### Идемпотентность
- Все LLM-вызовы кэшируются по (prompt + parameters)
- Возможность восстановления после сбоев
- Повторное выполнение не создает дубликаты

### Масштабируемость
- Асинхронная обработка с настраиваемым параллелизмом
- Батчирование для LLM-запросов
- Поддержка больших документов

### Гибкость
- Настраиваемые промпты для extraction
- Выбор LLM-провайдеров (OpenAI, Azure, etc.)
- Конфигурируемые параметры на каждом этапе

### Иерархичность
- Многоуровневые сообщества для разных уровней абстракции
- From low-level entities → high-level themes

## Основные файлы

- **Pipeline Factory**: `graphrag/index/workflows/factory.py`
- **Pipeline Runner**: `graphrag/index/run/run_pipeline.py`
- **Configuration**: `graphrag/config/models/graph_rag_config.py`
- **Data Models**: `graphrag/data_model/*.py`

## Следующие разделы

1. [Разбиение документов и создание chunks](01-document-ingestion-chunking.md)
2. [Извлечение сущностей и отношений](02-entity-relationship-extraction.md)
3. [Построение графа знаний](03-graph-construction.md)
4. [Обнаружение сообществ](04-community-detection.md)
5. [Генерация отчетов о сообществах](05-community-reports.md)
6. [Embeddings и векторизация](06-embeddings-vectorization.md)
7. [Роли языковых агентов](07-llm-agents-roles.md)
8. [Поток данных через pipeline](08-data-flow.md)
9. [Конфигурация системы](09-configuration.md)
