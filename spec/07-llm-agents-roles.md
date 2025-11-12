# Роли языковых моделей и агентов в GraphRAG

## Обзор

GraphRAG использует **Large Language Models (LLM)** в качестве интеллектуальных агентов на нескольких критических этапах pipeline. Каждый агент выполняет специфическую функцию, используя специализированные промпты и конфигурации.

Система использует **4 основных типа LLM-агентов** и **1 embedding модель**.

## Архитектура LLM-агентов

```
                    ┌─────────────────────────────┐
                    │   Language Model Config     │
                    │   (OpenAI, Azure, etc.)     │
                    └──────────────┬──────────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        │                          │                          │
        ▼                          ▼                          ▼
┌───────────────┐          ┌──────────────┐         ┌─────────────────┐
│  Extraction   │          │ Summarization│         │ Community Report│
│     Agent     │          │    Agent     │         │     Agent       │
└───────────────┘          └──────────────┘         └─────────────────┘
        │                          │                          │
        │                          │                          │
        ▼                          ▼                          ▼
   text_units              entities            community_reports
        │               relationships                 │
        │                          │                  │
        └──────────────────────────┴──────────────────┘
                                   │
                                   ▼
                           ┌──────────────┐
                           │  Embedding   │
                           │    Model     │
                           └──────────────┘
                                   │
                                   ▼
                            embeddings/*.parquet
```

## 1. Extraction Agent (Агент извлечения)

### Назначение

**Extraction Agent** — это основной агент, который анализирует текстовые фрагменты и извлекает структурированную информацию о сущностях и их отношениях.

### Функции

1. **Идентификация сущностей** (Entities)
   - Распознавание named entities в тексте
   - Классификация по типам (organization, person, geo, event)
   - Создание описаний сущностей

2. **Извлечение отношений** (Relationships)
   - Определение связей между сущностями
   - Описание характера отношений
   - Оценка силы связи (strength 1-10)

3. **Gleaning** (опционально)
   - Итеративное улучшение результатов
   - Поиск пропущенных сущностей и отношений

### Конфигурация

```python
class ExtractGraphConfig:
    model_id: str = "gpt-4"                    # Рекомендуется GPT-4
    entity_types: list[str] = [
        "organization",
        "person",
        "geo",
        "event"
    ]
    max_gleanings: int = 0                     # Итерации gleaning
    prompt: str | None = None                  # Кастомный промпт
    encoding_model: str = "cl100k_base"
```

### Промпт (упрощенная структура)

```
-Goal-
Extract all entities and relationships from the text.

-Steps-
1. Identify all entities (name, type, description)
2. Identify relationships (source, target, description, strength)
3. Return formatted output

-Format-
("entity"<|>NAME<|>TYPE<|>DESCRIPTION)
("relationship"<|>SOURCE<|>TARGET<|>DESCRIPTION<|>STRENGTH)
```

**Полный промпт**: `graphrag/prompts/index/extract_graph.py`

### Входные/выходные данные

```python
INPUT:  text_units.parquet["text"]
OUTPUT: entities.parquet + relationships.parquet

# Количество вызовов: len(text_units)
# Стоимость: ~70% от общих затрат на LLM
```

### Характеристики модели

```python
RECOMMENDED_MODEL = "gpt-4"  # или "gpt-4-turbo"

PARAMETERS = {
    "temperature": 0,          # Детерминированность
    "max_tokens": 4000,        # Достаточно для extraction
    "top_p": 1.0
}

AVERAGE_TOKENS_PER_CALL = {
    "input": 1500,   # text_unit + prompt
    "output": 800    # entities + relationships
}
```

### Производительность

```python
# Для 1000 text_units:
PERFORMANCE = {
    "total_calls": 1000,
    "parallel_requests": 3,              # Конфигурируемо
    "avg_time_per_call": 4.5,            # секунды (GPT-4)
    "total_time": "~25 минут",           # С параллелизмом
    "estimated_cost": "$40-60"           # Зависит от pricing
}
```

## 2. Summarization Agent (Агент суммаризации)

### Назначение

**Summarization Agent** объединяет множественные описания одной и той же сущности или отношения в единое краткое описание.

### Функции

1. **Consolidation описаний сущностей**
   - Объединение описаний из разных text_units
   - Разрешение противоречий
   - Создание comprehensive description

2. **Summarization описаний отношений**
   - Аналогично для relationships

### Конфигурация

```python
class SummarizeDescriptionsConfig:
    model_id: str = "gpt-4"                # Или GPT-3.5 для экономии
    prompt: str | None = None
    max_input_length: int = 8000           # Для множественных описаний
```

### Промпт (упрощенная структура)

```
-Goal-
Given multiple descriptions of the same entity, create a single comprehensive
description that captures all key information.

-Entity-
{entity_name}

-Descriptions-
1. {description_1}
2. {description_2}
...

-Output-
Single consolidated description in third person.
```

**Полный промпт**: `graphrag/prompts/index/summarize_descriptions.py`

### Входные/выходные данные

```python
INPUT:  Множественные описания сущностей из разных text_units
OUTPUT: Единое описание для каждой уникальной сущности

# Количество вызовов: количество уникальных сущностей (меньше, чем text_units)
# Стоимость: ~5-10% от общих затрат
```

### Характеристики модели

```python
RECOMMENDED_MODEL = "gpt-4" или "gpt-3.5-turbo"

PARAMETERS = {
    "temperature": 0,
    "max_tokens": 500,  # Короткое описание
}

AVERAGE_TOKENS_PER_CALL = {
    "input": 2000,   # Множественные описания
    "output": 200    # Краткое описание
}
```

### Оптимизация

```python
# Можно использовать более дешевую модель:
COST_OPTIMIZATION = {
    "gpt-4": "$0.06 per call",
    "gpt-3.5-turbo": "$0.006 per call",  # 10x дешевле
    "quality_difference": "Minimal"       # Для summarization
}

# Рекомендация: GPT-3.5-turbo достаточно для summarization
```

## 3. Community Report Agent (Агент отчетов о сообществах)

### Назначение

**Community Report Agent** — это аналитический агент, который создает comprehensive отчеты о тематических сообществах в графе знаний.

### Функции

1. **Анализ структуры сообщества**
   - Изучение узлов (entities) и ребер (relationships)
   - Выявление ключевых паттернов

2. **Генерация insights**
   - Создание 5-10 ключевых findings
   - Обоснование каждого insight с data references

3. **Оценка важности**
   - Impact severity rating (0-10)
   - Объяснение рейтинга

4. **Структурирование отчета**
   - Title, Summary, Findings
   - JSON-форматированный вывод

### Конфигурация

```python
class CommunityReportsConfig:
    model_id: str = "gpt-4"                # Требуется GPT-4 для качества
    max_report_length: int = 2000          # Токены для отчета
    max_input_length: int = 8000           # Контекст сообщества
    prompt: str | None = None
```

### Промпт (упрощенная структура)

```
You are an AI assistant that helps perform information discovery.

-Goal-
Write a comprehensive report of a community, given its entities and relationships.

-Report Structure-
- TITLE: Short but specific name
- SUMMARY: Executive summary
- IMPACT SEVERITY RATING: 0-10
- RATING EXPLANATION: One sentence
- DETAILED FINDINGS: 5-10 key insights with explanations

-Output Format-
JSON with fields: title, summary, rating, rating_explanation, findings

-Grounding Rules-
Support statements with data references:
[Data: Entities (1, 5); Relationships (23); ...]
```

**Полный промпт**: `graphrag/prompts/index/community_report.py`

### Входные/выходные данные

```python
INPUT:  communities.parquet + контекст (entities + relationships)
OUTPUT: community_reports.parquet с отчетами

# Количество вызовов: количество сообществ на всех уровнях
# Стоимость: ~20-25% от общих затрат
```

### Характеристики модели

```python
RECOMMENDED_MODEL = "gpt-4"  # GPT-3.5 не справляется с качеством

PARAMETERS = {
    "temperature": 0,
    "max_tokens": 2000,  # Для полного отчета
}

AVERAGE_TOKENS_PER_CALL = {
    "input": 5000,   # Контекст сообщества
    "output": 1500   # Структурированный отчет
}
```

### Производительность

```python
# Для графа с 150 сообществами (level 0):
PERFORMANCE = {
    "total_calls": 150,
    "parallel_requests": 3,
    "avg_time_per_call": 8.0,           # секунды (GPT-4, длинный вывод)
    "total_time": "~7 минут",
    "estimated_cost": "$25-35"
}
```

## 4. Embedding Model (Модель векторизации)

### Назначение

**Embedding Model** преобразует текстовые данные в векторные представления для семантического поиска.

### Функции

1. **Векторизация текстов**
   - Documents, text_units, entities, relationships, community reports
   - Создание dense vectors (1536 или 3072 dimensions)

2. **Семантическое представление**
   - Похожие по смыслу тексты → близкие векторы
   - Используется для retrieval

### Конфигурация

```python
class TextEmbeddingConfig:
    model_id: str = "text-embedding-3-small"  # Рекомендуется
    batch_size: int = 500                     # Батчирование для эффективности
    batch_max_tokens: int = 10000
    target: str = "all"                       # Какие embeddings создавать
```

### Поддерживаемые модели

```python
MODELS = {
    "text-embedding-3-small": {
        "dimensions": 1536,
        "cost_per_1M_tokens": 0.02,      # $
        "max_input": 8191,               # tokens
        "quality": "Good",
        "speed": "Fast"
    },
    "text-embedding-3-large": {
        "dimensions": 3072,
        "cost_per_1M_tokens": 0.13,
        "max_input": 8191,
        "quality": "Excellent",
        "speed": "Medium"
    },
    "text-embedding-ada-002": {
        "dimensions": 1536,
        "cost_per_1M_tokens": 0.10,
        "max_input": 8191,
        "quality": "Good (legacy)",
        "speed": "Fast"
    }
}

# Рекомендация: text-embedding-3-small (оптимальное качество/цена)
```

### Входные/выходные данные

```python
INPUT:  Все текстовые поля из всех таблиц
OUTPUT: embeddings/*.parquet

EMBEDDED_FIELDS = [
    "document_text_embedding",
    "text_unit_text_embedding",
    "entity_title_embedding",
    "entity_description_embedding",
    "relationship_description_embedding",
    "community_title_embedding",
    "community_summary_embedding",
    "community_full_content_embedding"
]

# Количество вызовов: зависит от количества объектов
# Стоимость: ~5% от общих затрат
```

### Производительность

```python
# Для 10,000 текстов:
PERFORMANCE = {
    "total_texts": 10000,
    "batch_size": 500,
    "total_batches": 20,
    "avg_time_per_batch": 1.5,          # секунды
    "total_time": "~30 секунд",
    "estimated_cost": "$0.50-1.00"      # Очень дешево
}
```

## Сравнение агентов

### Сводная таблица

```
┌──────────────────┬─────────────┬───────────────┬──────────────┬─────────────┐
│ Agent            │ Model       │ Cost Share    │ Calls        │ Critical    │
├──────────────────┼─────────────┼───────────────┼──────────────┼─────────────┤
│ Extraction       │ GPT-4       │ 70%           │ High         │ ✓✓✓         │
│ Summarization    │ GPT-3.5/4   │ 5-10%         │ Medium       │ ✓           │
│ Community Report │ GPT-4       │ 20-25%        │ Medium       │ ✓✓          │
│ Embedding        │ ada-3-small │ ~5%           │ Very High    │ ✓✓          │
└──────────────────┴─────────────┴───────────────┴──────────────┴─────────────┘
```

### Критичность для качества

```python
QUALITY_IMPACT = {
    "Extraction Agent": "CRITICAL",        # Низкое качество → плохой граф
    "Community Report Agent": "HIGH",      # Важно для query quality
    "Summarization Agent": "MEDIUM",       # Улучшает читаемость
    "Embedding Model": "HIGH"              # Важно для retrieval
}
```

## Оптимизация затрат

### Стратегии снижения стоимости

```python
# 1. Использовать GPT-3.5-turbo для Summarization
OPTIMIZATION_1 = {
    "agent": "Summarization",
    "change": "gpt-4 → gpt-3.5-turbo",
    "cost_reduction": "90%",
    "quality_impact": "Minimal"
}

# 2. Уменьшить max_gleanings для Extraction
OPTIMIZATION_2 = {
    "agent": "Extraction",
    "change": "max_gleanings: 1 → 0",
    "cost_reduction": "50% calls",
    "quality_impact": "Moderate (10-15% recall)"
}

# 3. Использовать text-embedding-3-small вместо large
OPTIMIZATION_3 = {
    "agent": "Embedding",
    "change": "3-large → 3-small",
    "cost_reduction": "85%",
    "quality_impact": "Small (3-5% retrieval accuracy)"
}

# 4. Fast Mode вместо Standard
OPTIMIZATION_4 = {
    "pipeline": "Overall",
    "change": "Standard → Fast mode",
    "cost_reduction": "60-70%",
    "quality_impact": "Moderate (NLP extraction вместо LLM)"
}
```

### Рекомендуемая конфигурация для production

```python
PRODUCTION_CONFIG = {
    "extraction": {
        "model_id": "gpt-4-turbo",        # Баланс качества/скорости
        "max_gleanings": 0,               # Без gleaning
    },
    "summarization": {
        "model_id": "gpt-3.5-turbo",      # Достаточно
    },
    "community_reports": {
        "model_id": "gpt-4-turbo",        # Требуется качество
    },
    "embeddings": {
        "model_id": "text-embedding-3-small",  # Оптимально
    }
}

ESTIMATED_COST = "$50-80 per 1000 documents"  # Зависит от размера документов
```

## Кэширование и идемпотентность

### Механизм кэширования

Все LLM-вызовы кэшируются для обеспечения идемпотентности:

```python
class PipelineCache:
    def get_cache_key(self, prompt: str, config: dict) -> str:
        """Создает уникальный ключ для кэша"""
        cache_data = {
            "prompt": prompt,
            "model": config["model"],
            "temperature": config["temperature"],
            "max_tokens": config["max_tokens"]
        }
        return hashlib.sha256(
            json.dumps(cache_data).encode()
        ).hexdigest()

    async def get(self, cache_key: str) -> str | None:
        """Получает кэшированный результат"""
        # Чтение из storage (file, db, etc.)

    async def set(self, cache_key: str, result: str):
        """Сохраняет результат в кэш"""
        # Запись в storage
```

**Файл**: `graphrag/cache/pipeline_cache.py`

### Преимущества кэширования

```python
BENEFITS = {
    "cost_savings": "100% на повторных запусках",
    "reproducibility": "Детерминированные результаты",
    "fault_tolerance": "Восстановление после сбоев без потерь",
    "debugging": "Легко проверить LLM outputs"
}
```

## Мониторинг и логирование

### Отслеживание LLM-вызовов

```python
class LLMCallbacks:
    async def on_llm_call_start(self, prompt: str, config: dict):
        """Вызывается перед LLM call"""
        log.info(f"LLM call started: {config['model']}")

    async def on_llm_call_end(
        self,
        prompt: str,
        response: str,
        tokens_used: dict
    ):
        """Вызывается после LLM call"""
        log.info(f"Tokens: {tokens_used['input']} + {tokens_used['output']}")

    async def on_llm_call_error(self, error: Exception):
        """Вызывается при ошибке"""
        log.error(f"LLM call failed: {error}")
```

### Метрики производительности

```python
METRICS = {
    "total_llm_calls": int,
    "total_tokens_used": int,
    "estimated_cost": float,
    "avg_response_time": float,
    "cache_hit_rate": float,
    "error_rate": float
}
```

## Обработка ошибок

### Retry logic

```python
async def llm_call_with_retry(
    prompt: str,
    config: dict,
    max_retries: int = 3,
    backoff_factor: int = 2
) -> str:
    """LLM вызов с повторными попытками"""
    for attempt in range(max_retries):
        try:
            response = await llm_call(prompt, config)
            return response
        except RateLimitError as e:
            if attempt == max_retries - 1:
                raise
            wait_time = backoff_factor ** attempt
            await asyncio.sleep(wait_time)
        except Exception as e:
            log.error(f"LLM call failed: {e}")
            if attempt == max_retries - 1:
                raise
```

### Graceful degradation

```python
# При сбое агента:
# 1. Extraction → Вернуть пустой граф или использовать NLP fallback
# 2. Summarization → Использовать первое описание без суммаризации
# 3. Community Report → Создать базовый template-based отчет
# 4. Embedding → Пропустить, использовать keyword search
```

## Следующие разделы

→ [Поток данных через pipeline](08-data-flow.md)
→ [Конфигурация системы](09-configuration.md)
