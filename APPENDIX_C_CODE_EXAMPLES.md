# Приложение C: Примеры программного кода модулей

## Назначение документа
Это **практическое приложение** с готовыми примерами кода для различных типов модулей.

---

## ПРИМЕР 1: Простой модуль фильтрации

### Структура модуля

```
csv_filter/
├── module.json
├── __init__.py
├── processor.py
├── ui_config.json
├── requirements.txt
└── README.md
```

### module.json

```json
{
  "id": "csv_filter",
  "name": "CSV Фильтр",
  "version": "1.0.0",
  "author": {
    "name": "Data Pipeline Team",
    "email": "dev@datapipeline.io"
  },
  "description": "Фильтрация CSV данных по условиям",
  "category": "data_transformation",
  "tags": ["csv", "filter", "data"],
  "compatibility": {
    "core_version": ">=1.0.0",
    "python_version": ">=3.8"
  },
  "entry_point": "processor.CSVFilter",
  "input": {
    "accepted_formats": ["csv"],
    "max_file_size": "100MB"
  },
  "output": {
    "formats": ["csv", "json"]
  },
  "dependencies": {
    "python": ["pandas>=1.3.0"]
  },
  "license": "MIT"
}
```

### processor.py

```python
"""CSV Filter Module - Фильтрация CSV данных"""

import pandas as pd
from typing import Any, Dict, List
from core.module_api import BaseModule, ModuleInput, ModuleOutput

class CSVFilter(BaseModule):
    """Модуль фильтрации CSV данных"""

    module_id = "csv_filter"
    version = "1.0.0"

    def __init__(self, config: Dict[str, Any]):
        super().__init__(config)

        # Параметры фильтрации
        self.filter_column = config.get('filter_column')
        self.operator = config.get('operator', '==')
        self.value = config.get('value')

    def validate_input(self, input_data: ModuleInput) -> bool:
        """Валидация входных данных"""
        if input_data.format != 'csv':
            raise ValueError("Only CSV format is supported")

        if not self.filter_column:
            raise ValueError("Filter column is required")

        return True

    def process(self, input_data: ModuleInput) -> ModuleOutput:
        """Фильтрация CSV данных"""

        # Валидация
        self.validate_input(input_data)

        # Загрузка CSV
        df = pd.read_csv(input_data.data)

        # Проверка наличия колонки
        if self.filter_column not in df.columns:
            raise ValueError(f"Column '{self.filter_column}' not found")

        # Применение фильтра
        filtered_df = self._apply_filter(df)

        # Формирование результата
        output = ModuleOutput(
            data=filtered_df.to_dict('records'),
            format='json',
            metadata={
                'original_rows': len(df),
                'filtered_rows': len(filtered_df),
                'columns': list(df.columns)
            },
            metrics={
                'rows_removed': len(df) - len(filtered_df),
                'retention_rate': len(filtered_df) / len(df) * 100
            }
        )

        return output

    def _apply_filter(self, df: pd.DataFrame) -> pd.DataFrame:
        """Применение фильтра к DataFrame"""

        column = df[self.filter_column]

        # Применение оператора
        if self.operator == '==':
            mask = column == self.value
        elif self.operator == '!=':
            mask = column != self.value
        elif self.operator == '>':
            mask = column > float(self.value)
        elif self.operator == '<':
            mask = column < float(self.value)
        elif self.operator == '>=':
            mask = column >= float(self.value)
        elif self.operator == '<=':
            mask = column <= float(self.value)
        elif self.operator == 'contains':
            mask = column.str.contains(str(self.value), na=False)
        elif self.operator == 'starts_with':
            mask = column.str.startswith(str(self.value), na=False)
        elif self.operator == 'ends_with':
            mask = column.str.endswith(str(self.value), na=False)
        else:
            raise ValueError(f"Unknown operator: {self.operator}")

        return df[mask]

    def setup(self):
        """Инициализация модуля"""
        self.logger.info(f"CSVFilter v{self.version} initialized")

    def teardown(self):
        """Очистка ресурсов"""
        self.logger.info("CSVFilter teardown")
```

### __init__.py

```python
"""CSV Filter Module"""

from .processor import CSVFilter

__all__ = ['CSVFilter']
MODULE_CLASS = CSVFilter
```

### requirements.txt

```
pandas>=1.3.0
numpy>=1.21.0
```

---

## ПРИМЕР 2: Модуль с асинхронной обработкой

### processor.py

```python
"""Async Web Scraper Module"""

import aiohttp
import asyncio
from typing import List, Dict, Any
from bs4 import BeautifulSoup
from core.module_api import AsyncBaseModule, ModuleInput, ModuleOutput

class AsyncWebScraper(AsyncBaseModule):
    """Асинхронный модуль веб-скрейпинга"""

    module_id = "async_web_scraper"
    version = "1.0.0"

    def __init__(self, config: Dict[str, Any]):
        super().__init__(config)

        self.max_concurrent = config.get('max_concurrent', 5)
        self.timeout = config.get('timeout', 30)
        self.user_agent = config.get('user_agent', 'DataPipeline/1.0')

    def validate_input(self, input_data: ModuleInput) -> bool:
        """Валидация URLs"""
        if not isinstance(input_data.data, list):
            raise ValueError("Input must be a list of URLs")

        for url in input_data.data:
            if not url.startswith(('http://', 'https://')):
                raise ValueError(f"Invalid URL: {url}")

        return True

    async def process(self, input_data: ModuleInput) -> ModuleOutput:
        """Асинхронная загрузка и парсинг веб-страниц"""

        # Валидация
        self.validate_input(input_data)

        urls = input_data.data

        # Создание семафора для ограничения concurrent запросов
        semaphore = asyncio.Semaphore(self.max_concurrent)

        # Асинхронная обработка всех URLs
        async with aiohttp.ClientSession() as session:
            tasks = [
                self._scrape_url(session, url, semaphore)
                for url in urls
            ]

            results = await asyncio.gather(*tasks, return_exceptions=True)

        # Обработка результатов
        successful = [r for r in results if not isinstance(r, Exception)]
        failed = [r for r in results if isinstance(r, Exception)]

        output = ModuleOutput(
            data=successful,
            format='json',
            metadata={
                'total_urls': len(urls),
                'successful': len(successful),
                'failed': len(failed)
            },
            metrics={
                'success_rate': len(successful) / len(urls) * 100
            }
        )

        return output

    async def _scrape_url(self,
                         session: aiohttp.ClientSession,
                         url: str,
                         semaphore: asyncio.Semaphore) -> Dict[str, Any]:
        """Загрузка и парсинг одного URL"""

        async with semaphore:
            try:
                headers = {'User-Agent': self.user_agent}

                async with session.get(url,
                                      headers=headers,
                                      timeout=self.timeout) as response:

                    html = await response.text()
                    soup = BeautifulSoup(html, 'html.parser')

                    # Извлечение данных
                    result = {
                        'url': url,
                        'status': response.status,
                        'title': soup.title.string if soup.title else None,
                        'links': [a['href'] for a in soup.find_all('a', href=True)],
                        'text': soup.get_text(strip=True)[:1000]
                    }

                    self.logger.info(f"Scraped: {url}")
                    return result

            except Exception as e:
                self.logger.error(f"Failed to scrape {url}: {e}")
                raise
```

---

## ПРИМЕР 3: Модуль с ML моделью

### processor.py

```python
"""Text Classifier Module - ML модуль классификации текста"""

import joblib
import numpy as np
from typing import Dict, Any, List
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from core.module_api import BaseModule, ModuleInput, ModuleOutput

class TextClassifier(BaseModule):
    """Модуль классификации текста с использованием ML"""

    module_id = "text_classifier"
    version = "1.0.0"

    def __init__(self, config: Dict[str, Any]):
        super().__init__(config)

        self.model_path = config.get('model_path')
        self.vectorizer_path = config.get('vectorizer_path')
        self.categories = config.get('categories', [])

        # Загрузка модели и векторизатора
        self.model = None
        self.vectorizer = None

    def setup(self):
        """Загрузка предобученной модели"""
        if self.model_path:
            self.model = joblib.load(self.model_path)
            self.logger.info(f"Model loaded from {self.model_path}")

        if self.vectorizer_path:
            self.vectorizer = joblib.load(self.vectorizer_path)
            self.logger.info(f"Vectorizer loaded from {self.vectorizer_path}")

    def validate_input(self, input_data: ModuleInput) -> bool:
        """Валидация входных данных"""
        if not self.model or not self.vectorizer:
            raise ValueError("Model not loaded. Call setup() first.")

        return True

    def process(self, input_data: ModuleInput) -> ModuleOutput:
        """Классификация текста"""

        self.validate_input(input_data)

        # Получение текстов
        if isinstance(input_data.data, str):
            texts = [input_data.data]
        elif isinstance(input_data.data, list):
            texts = input_data.data
        else:
            raise ValueError("Input must be string or list of strings")

        # Векторизация
        X = self.vectorizer.transform(texts)

        # Предсказание
        predictions = self.model.predict(X)
        probabilities = self.model.predict_proba(X)

        # Формирование результата
        results = []
        for i, text in enumerate(texts):
            result = {
                'text': text[:100] + '...' if len(text) > 100 else text,
                'predicted_category': predictions[i],
                'confidence': float(max(probabilities[i])),
                'probabilities': {
                    cat: float(prob)
                    for cat, prob in zip(self.categories, probabilities[i])
                }
            }
            results.append(result)

        output = ModuleOutput(
            data=results,
            format='json',
            metadata={
                'model': self.model.__class__.__name__,
                'categories': self.categories
            },
            metrics={
                'avg_confidence': float(np.mean([r['confidence'] for r in results]))
            }
        )

        return output

    def train(self, texts: List[str], labels: List[str]):
        """Обучение модели (опционально)"""

        # Векторизация
        self.vectorizer = TfidfVectorizer(max_features=5000)
        X = self.vectorizer.fit_transform(texts)

        # Обучение модели
        self.model = MultinomialNB()
        self.model.fit(X, labels)

        self.categories = sorted(set(labels))

        self.logger.info(f"Model trained on {len(texts)} samples")

    def save_model(self, model_path: str, vectorizer_path: str):
        """Сохранение модели"""
        joblib.dump(self.model, model_path)
        joblib.dump(self.vectorizer, vectorizer_path)
        self.logger.info(f"Model saved to {model_path}")
```

---

## ПРИМЕР 4: Модуль с Pipeline поддержкой

### processor.py

```python
"""Data Transformer - Модуль с поддержкой pipeline"""

from typing import Dict, Any, Optional
from core.module_api import BaseModule, ModuleInput, ModuleOutput

class DataTransformer(BaseModule):
    """Модуль трансформации данных с поддержкой pipeline"""

    module_id = "data_transformer"
    version = "1.0.0"

    def __init__(self, config: Dict[str, Any]):
        super().__init__(config)

        # Опции pipeline
        self.chain_to = config.get('chain_to')  # ID следующего модуля
        self.parallel_to = config.get('parallel_to', [])  # Параллельные модули

    def process(self, input_data: ModuleInput) -> ModuleOutput:
        """Трансформация данных"""

        # Основная обработка
        transformed_data = self._transform(input_data.data)

        # Формирование вывода
        output = ModuleOutput(
            data=transformed_data,
            format=input_data.format,
            metadata={
                'source_module': self.module_id,
                'transformations_applied': self._get_transformations()
            },
            metrics={}
        )

        # Pipeline routing
        if self.chain_to:
            output.metadata['next_module'] = self.chain_to

        if self.parallel_to:
            output.metadata['parallel_modules'] = self.parallel_to

        return output

    def _transform(self, data: Any) -> Any:
        """Применение трансформаций"""

        # Пример: цепочка трансформаций
        transformations = self.config.get('transformations', [])

        for transform in transformations:
            transform_type = transform.get('type')

            if transform_type == 'uppercase':
                data = self._uppercase_transform(data)
            elif transform_type == 'remove_duplicates':
                data = self._remove_duplicates(data)
            elif transform_type == 'filter':
                data = self._filter_transform(data, transform.get('criteria'))
            # ... другие трансформации

        return data

    def can_chain_with(self, module_id: str) -> bool:
        """Проверка совместимости с другим модулем"""
        # Логика проверки совместимости форматов
        compatible_modules = self.config.get('compatible_modules', [])
        return module_id in compatible_modules
```

---

## ПРИМЕР 5: Модуль с кэшированием

### processor.py

```python
"""Cached Data Processor - Модуль с кэшированием"""

import hashlib
import pickle
from typing import Dict, Any, Optional
from datetime import datetime, timedelta
from core.module_api import BaseModule, ModuleInput, ModuleOutput

class CachedProcessor(BaseModule):
    """Модуль с поддержкой кэширования результатов"""

    module_id = "cached_processor"
    version = "1.0.0"

    def __init__(self, config: Dict[str, Any]):
        super().__init__(config)

        # Настройки кэша
        self.cache_enabled = config.get('cache_enabled', True)
        self.cache_ttl = config.get('cache_ttl', 3600)  # seconds
        self.cache_max_size = config.get('cache_max_size', 1000)  # items

    def process(self, input_data: ModuleInput) -> ModuleOutput:
        """Обработка с кэшированием"""

        # Генерация ключа кэша
        cache_key = self._generate_cache_key(input_data)

        # Проверка кэша
        if self.cache_enabled:
            cached_result = self._get_from_cache(cache_key)
            if cached_result:
                self.logger.info(f"Cache hit: {cache_key}")
                return cached_result

        # Обработка данных
        result = self._do_process(input_data)

        # Сохранение в кэш
        if self.cache_enabled:
            self._save_to_cache(cache_key, result)

        return result

    def _generate_cache_key(self, input_data: ModuleInput) -> str:
        """Генерация уникального ключа кэша"""

        # Создание строки для хеширования
        cache_string = (
            str(input_data.data) +
            str(input_data.format) +
            str(input_data.parameters)
        )

        # MD5 хеш
        return hashlib.md5(cache_string.encode()).hexdigest()

    def _get_from_cache(self, key: str) -> Optional[ModuleOutput]:
        """Получение из кэша"""
        try:
            cache_entry = self.cache.get(key)
            if not cache_entry:
                return None

            # Проверка TTL
            if datetime.now() > cache_entry['expires_at']:
                self.cache.delete(key)
                return None

            return cache_entry['data']

        except Exception as e:
            self.logger.warning(f"Cache read error: {e}")
            return None

    def _save_to_cache(self, key: str, data: ModuleOutput):
        """Сохранение в кэш"""
        try:
            cache_entry = {
                'data': data,
                'created_at': datetime.now(),
                'expires_at': datetime.now() + timedelta(seconds=self.cache_ttl)
            }

            self.cache.set(key, cache_entry)

        except Exception as e:
            self.logger.warning(f"Cache write error: {e}")

    def clear_cache(self):
        """Очистка кэша"""
        self.cache.clear()
        self.logger.info("Cache cleared")
```

---

## ПРИМЕР 6: Модуль с webhooks

### processor.py

```python
"""Webhook Notifier - Модуль отправки уведомлений"""

import requests
from typing import Dict, Any
from core.module_api import BaseModule, ModuleInput, ModuleOutput

class WebhookNotifier(BaseModule):
    """Модуль отправки webhook уведомлений"""

    module_id = "webhook_notifier"
    version = "1.0.0"

    def __init__(self, config: Dict[str, Any]):
        super().__init__(config)

        self.webhook_url = config.get('webhook_url')
        self.events = config.get('events', ['on_success', 'on_error'])
        self.headers = config.get('headers', {})

    def process(self, input_data: ModuleInput) -> ModuleOutput:
        """Отправка данных через webhook"""

        # Основная обработка
        result = self._process_data(input_data)

        # Отправка webhook при успехе
        if 'on_success' in self.events:
            self._send_webhook('success', result)

        return result

    def on_error(self, error: Exception):
        """Обработка ошибки"""
        if 'on_error' in self.events:
            self._send_webhook('error', {
                'error': str(error),
                'type': error.__class__.__name__
            })

    def _send_webhook(self, event_type: str, data: Any):
        """Отправка webhook"""
        try:
            payload = {
                'event': event_type,
                'module': self.module_id,
                'timestamp': datetime.now().isoformat(),
                'data': data
            }

            response = requests.post(
                self.webhook_url,
                json=payload,
                headers=self.headers,
                timeout=10
            )

            response.raise_for_status()
            self.logger.info(f"Webhook sent: {event_type}")

        except Exception as e:
            self.logger.error(f"Webhook failed: {e}")
```

---

**Версия**: 1.0
**Дата**: 2025-12-28
**Тип**: Практическое приложение с примерами кода
