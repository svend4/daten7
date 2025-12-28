# Приложение B: API для разработки модулей

## Назначение документа
Это **техническое расширение** документации с подробным описанием API для создания собственных модулей обработки данных.

---

## Базовый интерфейс модуля

### Абстрактный класс BaseModule

Все модули наследуются от базового класса `BaseModule`:

```python
from abc import ABC, abstractmethod
from typing import Dict, Any, List
from dataclasses import dataclass

@dataclass
class ModuleInput:
    """Входные данные для модуля"""
    data: Any
    format: str
    metadata: Dict[str, Any]
    parameters: Dict[str, Any]

@dataclass
class ModuleOutput:
    """Выходные данные модуля"""
    data: Any
    format: str
    metadata: Dict[str, Any]
    metrics: Dict[str, Any]

class BaseModule(ABC):
    """Базовый класс для всех модулей"""

    def __init__(self, config: Dict[str, Any]):
        self.config = config
        self.logger = self._setup_logger()
        self.cache = self._setup_cache()

    @abstractmethod
    def process(self, input_data: ModuleInput) -> ModuleOutput:
        """
        Основной метод обработки данных

        Args:
            input_data: Входные данные

        Returns:
            ModuleOutput: Обработанные данные
        """
        pass

    @abstractmethod
    def validate_input(self, input_data: ModuleInput) -> bool:
        """Валидация входных данных"""
        pass

    def setup(self):
        """Инициализация модуля (опционально)"""
        pass

    def teardown(self):
        """Очистка ресурсов (опционально)"""
        pass

    def get_metadata(self) -> Dict[str, Any]:
        """Получение метаданных модуля"""
        return {
            "id": self.module_id,
            "version": self.version,
            "status": "ready"
        }
```

---

## Создание простого модуля

### Пример: Модуль извлечения email

```python
# email_extractor/processor.py

import re
from typing import List, Set
from core.module_api import BaseModule, ModuleInput, ModuleOutput

class EmailExtractor(BaseModule):
    """Модуль извлечения email-адресов из текста"""

    # Метаданные модуля
    module_id = "email_extractor"
    version = "1.0.0"

    # Regex паттерн для email
    EMAIL_PATTERN = r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}'

    def __init__(self, config):
        super().__init__(config)
        self.case_sensitive = config.get('case_sensitive', False)
        self.unique_only = config.get('unique_only', True)
        self.validate_mx = config.get('validate_mx', False)

    def validate_input(self, input_data: ModuleInput) -> bool:
        """Проверка входных данных"""
        # Проверка формата
        if input_data.format not in ['txt', 'md', 'html', 'csv', 'json']:
            raise ValueError(f"Unsupported format: {input_data.format}")

        # Проверка размера
        if len(str(input_data.data)) > 50 * 1024 * 1024:  # 50MB
            raise ValueError("File too large")

        return True

    def process(self, input_data: ModuleInput) -> ModuleOutput:
        """Извлечение email-адресов"""

        # Валидация
        self.validate_input(input_data)

        # Извлечение текста
        text = self._extract_text(input_data.data, input_data.format)

        # Поиск email
        emails = self._find_emails(text)

        # Валидация (если включена)
        if self.validate_mx:
            emails = self._validate_emails(emails)

        # Формирование результата
        output = ModuleOutput(
            data=list(emails),
            format='json',
            metadata={
                'source_format': input_data.format,
                'processing_time': self._get_processing_time()
            },
            metrics={
                'total_found': len(emails),
                'unique_count': len(set(emails))
            }
        )

        return output

    def _extract_text(self, data: Any, format: str) -> str:
        """Извлечение текста из различных форматов"""
        if format == 'txt' or format == 'md':
            return str(data)
        elif format == 'html':
            return self._strip_html(data)
        elif format == 'json':
            return self._extract_from_json(data)
        elif format == 'csv':
            return self._extract_from_csv(data)
        return str(data)

    def _find_emails(self, text: str) -> List[str] or Set[str]:
        """Поиск email адресов"""

        # Применение regex
        emails = re.findall(self.EMAIL_PATTERN, text)

        # Нормализация
        if not self.case_sensitive:
            emails = [email.lower() for email in emails]

        # Удаление дубликатов
        if self.unique_only:
            emails = set(emails)

        return emails

    def _validate_emails(self, emails: List[str]) -> List[str]:
        """Валидация email с проверкой MX записей"""
        import dns.resolver

        valid_emails = []
        for email in emails:
            domain = email.split('@')[1]
            try:
                # Проверка MX записей
                dns.resolver.resolve(domain, 'MX')
                valid_emails.append(email)
            except:
                # Домен не имеет MX записей
                pass

        return valid_emails
```

---

## Регистрация модуля

### Файл __init__.py

```python
# email_extractor/__init__.py

from .processor import EmailExtractor

# Экспорт класса модуля
__all__ = ['EmailExtractor']

# Метаданные (используются при загрузке)
MODULE_CLASS = EmailExtractor
```

---

## Конфигурация UI

### Файл ui_config.json

```json
{
  "ui": {
    "collapsed": {
      "template": "▶ {name} | Ввод: {input_formats} → Вывод: {output_formats}",
      "variables": {
        "name": "Извлечение email-адресов",
        "input_formats": "TXT, MD, HTML, CSV, JSON",
        "output_formats": "JSON, CSV, TXT"
      }
    },

    "expanded": {
      "sections": [
        {
          "id": "input",
          "title": "📥 ВВОД",
          "fields": [
            {
              "type": "file_upload",
              "label": "Загрузить файл",
              "accept": [".txt", ".md", ".html", ".csv", ".json"],
              "max_size": "50MB"
            },
            {
              "type": "textarea",
              "label": "Или вставить текст",
              "placeholder": "Вставьте текст для обработки...",
              "rows": 5
            }
          ]
        },

        {
          "id": "settings",
          "title": "⚙️ Настройки",
          "fields": [
            {
              "type": "checkbox",
              "name": "unique_only",
              "label": "Только уникальные адреса",
              "default": true
            },
            {
              "type": "checkbox",
              "name": "case_sensitive",
              "label": "Учитывать регистр",
              "default": false
            },
            {
              "type": "checkbox",
              "name": "validate_mx",
              "label": "Проверять MX записи",
              "default": false,
              "help": "Проверка существования почтового сервера (медленнее)"
            }
          ]
        },

        {
          "id": "processing",
          "title": "⚡ ОБРАБОТКА",
          "fields": [
            {
              "type": "button",
              "label": "Запустить обработку",
              "action": "process",
              "style": "primary"
            },
            {
              "type": "progress_bar",
              "id": "progress",
              "hidden": true
            }
          ]
        },

        {
          "id": "output",
          "title": "📤 ВЫВОД",
          "fields": [
            {
              "type": "stats",
              "template": "Найдено email: {total} | Уникальных: {unique}",
              "variables": ["total", "unique"]
            },
            {
              "type": "button_group",
              "label": "Экспорт:",
              "buttons": [
                {"label": "JSON", "action": "export_json"},
                {"label": "CSV", "action": "export_csv"},
                {"label": "TXT", "action": "export_txt"}
              ]
            },
            {
              "type": "button",
              "label": "↓ Скачать",
              "action": "download"
            },
            {
              "type": "button",
              "label": "→ Передать в модуль",
              "action": "pipe_to_next"
            }
          ]
        }
      ]
    }
  }
}
```

---

## Продвинутые возможности

### 1. Асинхронная обработка

```python
from core.module_api import AsyncBaseModule
import asyncio

class AsyncEmailExtractor(AsyncBaseModule):
    """Асинхронная версия модуля"""

    async def process(self, input_data: ModuleInput) -> ModuleOutput:
        """Асинхронная обработка"""

        # Параллельная обработка нескольких файлов
        tasks = [
            self._process_file(file)
            for file in input_data.data
        ]

        results = await asyncio.gather(*tasks)

        return ModuleOutput(
            data=results,
            format='json',
            metadata={},
            metrics={}
        )

    async def _process_file(self, file):
        """Обработка одного файла"""
        # ... логика обработки
        pass
```

### 2. Потоковая обработка (Streaming)

```python
from typing import Generator

class StreamingModule(BaseModule):
    """Модуль с потоковой обработкой"""

    def process_stream(self,
                      input_stream: Generator) -> Generator:
        """Обработка данных по частям"""

        for chunk in input_stream:
            # Обработка части данных
            processed_chunk = self._process_chunk(chunk)

            # Отправка результата
            yield processed_chunk
```

### 3. Кэширование результатов

```python
from functools import lru_cache
import hashlib

class CachedModule(BaseModule):
    """Модуль с кэшированием"""

    def process(self, input_data: ModuleInput) -> ModuleOutput:
        # Генерация ключа кэша
        cache_key = self._generate_cache_key(input_data)

        # Проверка кэша
        if cached := self.cache.get(cache_key):
            return cached

        # Обработка данных
        result = self._do_process(input_data)

        # Сохранение в кэш
        self.cache.set(cache_key, result, ttl=3600)

        return result

    def _generate_cache_key(self, input_data: ModuleInput) -> str:
        """Генерация уникального ключа"""
        content = str(input_data.data) + str(input_data.parameters)
        return hashlib.md5(content.encode()).hexdigest()
```

### 4. Прогресс обработки

```python
class ProgressModule(BaseModule):
    """Модуль с отчётом о прогрессе"""

    def process(self, input_data: ModuleInput) -> ModuleOutput:
        total_steps = 5

        # Шаг 1
        self.emit_progress(1, total_steps, "Загрузка данных...")
        data = self._load_data(input_data)

        # Шаг 2
        self.emit_progress(2, total_steps, "Валидация...")
        self.validate_input(input_data)

        # Шаг 3
        self.emit_progress(3, total_steps, "Обработка...")
        result = self._process(data)

        # Шаг 4
        self.emit_progress(4, total_steps, "Форматирование...")
        formatted = self._format_output(result)

        # Шаг 5
        self.emit_progress(5, total_steps, "Завершено!")

        return formatted
```

### 5. Обработка ошибок

```python
from core.exceptions import ModuleError, ValidationError

class RobustModule(BaseModule):
    """Модуль с обработкой ошибок"""

    def process(self, input_data: ModuleInput) -> ModuleOutput:
        try:
            return self._safe_process(input_data)

        except ValidationError as e:
            self.logger.error(f"Validation failed: {e}")
            raise

        except Exception as e:
            # Логирование ошибки
            self.logger.exception("Processing failed")

            # Отправка в систему мониторинга
            self.report_error(e)

            # Попытка восстановления
            if self.config.get('retry_on_error'):
                return self._retry_process(input_data)

            raise ModuleError(f"Processing failed: {e}")

    def _retry_process(self, input_data, max_retries=3):
        """Повторная попытка обработки"""
        for attempt in range(max_retries):
            try:
                return self._safe_process(input_data)
            except Exception as e:
                if attempt == max_retries - 1:
                    raise
                self.logger.warning(f"Retry {attempt + 1}/{max_retries}")
                time.sleep(2 ** attempt)  # Exponential backoff
```

---

## Hooks (Перехватчики)

Модули могут использовать hooks для расширения функциональности:

```python
class HookableModule(BaseModule):
    """Модуль с поддержкой hooks"""

    def process(self, input_data: ModuleInput) -> ModuleOutput:
        # Before hook
        self.trigger_hook('before_process', input_data)

        # Обработка
        result = self._do_process(input_data)

        # After hook
        result = self.trigger_hook('after_process', result)

        return result

    def register_hook(self, event: str, callback):
        """Регистрация hook"""
        if not hasattr(self, '_hooks'):
            self._hooks = {}

        if event not in self._hooks:
            self._hooks[event] = []

        self._hooks[event].append(callback)

    def trigger_hook(self, event: str, data):
        """Вызов hook"""
        if hasattr(self, '_hooks') and event in self._hooks:
            for callback in self._hooks[event]:
                data = callback(data) or data
        return data
```

**Использование:**

```python
# Регистрация hook
module = HookableModule(config)

def log_before_process(input_data):
    print(f"Processing started with {len(input_data.data)} items")

module.register_hook('before_process', log_before_process)
```

---

## Тестирование модулей

### Unit тесты

```python
# tests/test_email_extractor.py

import unittest
from email_extractor import EmailExtractor
from core.module_api import ModuleInput

class TestEmailExtractor(unittest.TestCase):

    def setUp(self):
        """Инициализация перед каждым тестом"""
        self.module = EmailExtractor({
            'unique_only': True,
            'case_sensitive': False
        })

    def test_extract_emails_from_text(self):
        """Тест извлечения email из текста"""
        input_data = ModuleInput(
            data="Contact: john@example.com or jane@test.org",
            format='txt',
            metadata={},
            parameters={}
        )

        result = self.module.process(input_data)

        self.assertEqual(len(result.data), 2)
        self.assertIn('john@example.com', result.data)
        self.assertIn('jane@test.org', result.data)

    def test_unique_emails(self):
        """Тест удаления дубликатов"""
        input_data = ModuleInput(
            data="john@example.com, john@example.com, jane@test.org",
            format='txt',
            metadata={},
            parameters={}
        )

        result = self.module.process(input_data)

        self.assertEqual(len(result.data), 2)

    def test_invalid_format(self):
        """Тест обработки неподдерживаемого формата"""
        input_data = ModuleInput(
            data="test",
            format='pdf',  # Неподдерживаемый формат
            metadata={},
            parameters={}
        )

        with self.assertRaises(ValueError):
            self.module.process(input_data)
```

---

## Документация модуля

### Файл README.md в модуле

```markdown
# Email Extractor Module

## Описание
Модуль для извлечения email-адресов из текстовых данных.

## Установка
\`\`\`bash
module install email-extractor
\`\`\`

## Использование

### Через UI
1. Загрузите текстовый файл или вставьте текст
2. Настройте параметры извлечения
3. Нажмите "Обработать"
4. Экспортируйте результаты

### Через API
\`\`\`python
from email_extractor import EmailExtractor

module = EmailExtractor({
    'unique_only': True,
    'case_sensitive': False
})

result = module.process(input_data)
print(result.data)  # ['email1@example.com', 'email2@test.org']
\`\`\`

## Параметры

- `unique_only` (bool): Удалять дубликаты (по умолчанию: true)
- `case_sensitive` (bool): Учитывать регистр (по умолчанию: false)
- `validate_mx` (bool): Проверять MX записи (по умолчанию: false)

## Поддерживаемые форматы

**Ввод:** TXT, MD, HTML, CSV, JSON
**Вывод:** JSON, CSV, TXT

## Лицензия
MIT
```

---

**Версия**: 1.0
**Дата**: 2025-12-28
**Тип**: Техническое приложение
