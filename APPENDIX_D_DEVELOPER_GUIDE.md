# Приложение D: Руководство разработчика модулей

## Назначение документа
Это **практическое руководство** для разработчиков, создающих собственные модули для системы.

---

## Быстрый старт: Создание первого модуля

### Шаг 1: Создание структуры модуля

```bash
# Создание шаблона модуля
module create my-first-module

# Структура будет создана автоматически:
my-first-module/
├── module.json
├── __init__.py
├── processor.py
├── ui_config.json
├── requirements.txt
├── tests/
│   └── test_processor.py
└── README.md
```

### Шаг 2: Заполнение module.json

```json
{
  "id": "my_first_module",
  "name": "Мой первый модуль",
  "version": "0.1.0",
  "author": {
    "name": "Ваше имя",
    "email": "your@email.com"
  },
  "description": "Описание того, что делает модуль",
  "category": "data_transformation",
  "entry_point": "processor.MyFirstModule",
  "input": {
    "accepted_formats": ["txt", "csv"]
  },
  "output": {
    "formats": ["json", "csv"]
  },
  "dependencies": {
    "python": []
  },
  "license": "MIT"
}
```

### Шаг 3: Реализация логики

```python
# processor.py

from core.module_api import BaseModule, ModuleInput, ModuleOutput

class MyFirstModule(BaseModule):
    """Описание вашего модуля"""

    module_id = "my_first_module"
    version = "0.1.0"

    def validate_input(self, input_data: ModuleInput) -> bool:
        """Проверка входных данных"""
        # Ваша логика валидации
        return True

    def process(self, input_data: ModuleInput) -> ModuleOutput:
        """Основная логика обработки"""

        # 1. Получение данных
        data = input_data.data

        # 2. Обработка
        result = self._do_something(data)

        # 3. Формирование результата
        output = ModuleOutput(
            data=result,
            format='json',
            metadata={},
            metrics={}
        )

        return output

    def _do_something(self, data):
        """Ваша логика обработки"""
        # Реализуйте здесь
        return data
```

### Шаг 4: Тестирование

```python
# tests/test_processor.py

import unittest
from my_first_module.processor import MyFirstModule
from core.module_api import ModuleInput

class TestMyFirstModule(unittest.TestCase):

    def setUp(self):
        self.module = MyFirstModule({})

    def test_basic_processing(self):
        input_data = ModuleInput(
            data="test data",
            format='txt',
            metadata={},
            parameters={}
        )

        result = self.module.process(input_data)

        self.assertIsNotNone(result)
        self.assertEqual(result.format, 'json')
```

```bash
# Запуск тестов
python -m pytest tests/
```

### Шаг 5: Упаковка и установка

```bash
# Валидация модуля
module validate ./my-first-module

# Упаковка
module pack ./my-first-module

# Результат: my-first-module-0.1.0.zip

# Локальная установка
module install ./my-first-module-0.1.0.zip

# Тестирование
module test my_first_module
```

---

## Чеклист разработки модуля

### ✅ Обязательные элементы

- [ ] `module.json` с полными метаданными
- [ ] Класс, наследующий `BaseModule`
- [ ] Реализация метода `process()`
- [ ] Реализация метода `validate_input()`
- [ ] `__init__.py` с экспортом `MODULE_CLASS`
- [ ] `requirements.txt` со всеми зависимостями
- [ ] `README.md` с описанием и примерами

### ✅ Рекомендуемые элементы

- [ ] `ui_config.json` для красивого UI
- [ ] Unit тесты (`tests/`)
- [ ] Обработка ошибок
- [ ] Логирование важных событий
- [ ] Документация API (docstrings)
- [ ] Примеры использования

### ✅ Опциональные элементы

- [ ] Асинхронная версия (`AsyncBaseModule`)
- [ ] Поддержка кэширования
- [ ] Hooks для расширения
- [ ] CLI инструменты
- [ ] Интеграционные тесты

---

## Best Practices

### 1. Именование

**✅ Хорошо:**
```python
class EmailExtractor(BaseModule):
    module_id = "email_extractor"  # snake_case
```

**❌ Плохо:**
```python
class email_extractor(BaseModule):
    module_id = "EmailExtractor"
```

### 2. Обработка ошибок

**✅ Хорошо:**
```python
def process(self, input_data):
    try:
        return self._safe_process(input_data)
    except ValidationError as e:
        self.logger.error(f"Validation failed: {e}")
        raise
    except Exception as e:
        self.logger.exception("Unexpected error")
        raise ModuleError(f"Processing failed: {e}")
```

**❌ Плохо:**
```python
def process(self, input_data):
    # Игнорирование ошибок
    try:
        return self._process(input_data)
    except:
        pass
```

### 3. Валидация

**✅ Хорошо:**
```python
def validate_input(self, input_data):
    if input_data.format not in ['csv', 'json']:
        raise ValueError(f"Unsupported format: {input_data.format}")

    if not input_data.data:
        raise ValueError("Empty data")

    return True
```

**❌ Плохо:**
```python
def validate_input(self, input_data):
    return True  # Нет проверок
```

### 4. Логирование

**✅ Хорошо:**
```python
def process(self, input_data):
    self.logger.info(f"Processing {len(input_data.data)} items")

    result = self._process(input_data)

    self.logger.info(f"Processed successfully: {len(result)} items")
    return result
```

**❌ Плохо:**
```python
def process(self, input_data):
    print("Processing...")  # Использование print
    return self._process(input_data)
```

### 5. Конфигурация

**✅ Хорошо:**
```python
def __init__(self, config):
    super().__init__(config)

    # Использование get() с defaults
    self.timeout = config.get('timeout', 30)
    self.max_retries = config.get('max_retries', 3)
```

**❌ Плохо:**
```python
def __init__(self, config):
    super().__init__(config)

    # Прямой доступ без проверки
    self.timeout = config['timeout']  # KeyError если отсутствует
```

---

## Распространённые паттерны

### Паттерн 1: Batch Processing

```python
def process(self, input_data):
    """Обработка пакетами для больших данных"""

    items = input_data.data
    batch_size = self.config.get('batch_size', 100)

    results = []
    for i in range(0, len(items), batch_size):
        batch = items[i:i + batch_size]
        batch_result = self._process_batch(batch)
        results.extend(batch_result)

        # Прогресс
        self.emit_progress(i + len(batch), len(items))

    return ModuleOutput(data=results, ...)
```

### Паттерн 2: Retry Logic

```python
def process(self, input_data):
    """Обработка с повторными попытками"""

    max_retries = self.config.get('max_retries', 3)

    for attempt in range(max_retries):
        try:
            return self._process_once(input_data)

        except TemporaryError as e:
            if attempt == max_retries - 1:
                raise

            wait_time = 2 ** attempt  # Exponential backoff
            self.logger.warning(f"Retry {attempt + 1} after {wait_time}s")
            time.sleep(wait_time)
```

### Паттерн 3: Resource Cleanup

```python
def __init__(self, config):
    super().__init__(config)
    self.connection = None

def setup(self):
    """Инициализация ресурсов"""
    self.connection = create_connection(self.config)

def process(self, input_data):
    """Обработка"""
    return self._use_connection(input_data)

def teardown(self):
    """Очистка ресурсов"""
    if self.connection:
        self.connection.close()
        self.connection = None
```

### Паттерн 4: Конфигурируемая обработка

```python
def process(self, input_data):
    """Обработка с конфигурируемыми шагами"""

    data = input_data.data

    # Конфигурируемая цепочка обработки
    steps = self.config.get('processing_steps', [
        'normalize',
        'validate',
        'transform'
    ])

    for step in steps:
        method = getattr(self, f'_step_{step}', None)
        if method:
            data = method(data)

    return ModuleOutput(data=data, ...)
```

---

## Отладка модулей

### Включение debug логов

```python
# В конфигурации модуля
config = {
    'debug': True,
    'log_level': 'DEBUG'
}

module = MyModule(config)
```

### Использование breakpoints

```python
def process(self, input_data):
    # Точка останова для отладки
    import pdb; pdb.set_trace()

    result = self._process(input_data)
    return result
```

### Профилирование производительности

```python
import cProfile
import pstats

def profile_process(self, input_data):
    """Профилирование обработки"""

    profiler = cProfile.Profile()
    profiler.enable()

    result = self.process(input_data)

    profiler.disable()
    stats = pstats.Stats(profiler)
    stats.sort_stats('cumulative')
    stats.print_stats(10)  # Top 10 функций

    return result
```

---

## Публикация модуля

### 1. Подготовка к публикации

```bash
# Проверка кода
module lint ./my-module

# Запуск всех тестов
module test ./my-module --coverage

# Проверка безопасности
module scan ./my-module
```

### 2. Создание релиза

```bash
# Обновление версии в module.json
# 0.1.0 -> 1.0.0

# Создание changelog
echo "## Version 1.0.0
- Initial release
- Feature A
- Feature B" > CHANGELOG.md

# Упаковка
module pack ./my-module
```

### 3. Публикация в репозиторий

```bash
# Подпись модуля
module sign my-module-1.0.0.zip --key private.key

# Публикация
module publish my-module-1.0.0.zip \
    --repository official \
    --api-key $REPO_API_KEY
```

### 4. Создание документации

```markdown
# README.md

# My Module

## Установка
\`\`\`bash
module install my-module
\`\`\`

## Использование
[Примеры использования]

## API
[Документация API]

## Лицензия
MIT
```

---

## Полезные ресурсы

### Официальная документация
- Core API Reference: `/docs/api`
- Module Development Guide: `/docs/modules`
- Best Practices: `/docs/best-practices`

### Примеры модулей
- Official Modules Repository: `https://github.com/datapipeline/modules`
- Community Modules: `https://github.com/datapipeline/community-modules`

### Инструменты
- Module CLI: `module --help`
- Module Testing Framework: `/docs/testing`
- Module Validator: `/docs/validation`

---

## FAQ

**Q: Как обновить существующий модуль?**
A: Обновите версию в `module.json` и запустите `module pack`. Используйте semantic versioning.

**Q: Можно ли использовать внешние библиотеки?**
A: Да, укажите их в `requirements.txt`. Система автоматически установит зависимости.

**Q: Как сделать модуль асинхронным?**
A: Наследуйте от `AsyncBaseModule` вместо `BaseModule` и используйте `async def process()`.

**Q: Как добавить custom UI?**
A: Создайте `ui_config.json` с описанием UI компонентов. См. APPENDIX_B_MODULE_API.md.

**Q: Как тестировать взаимодействие с другими модулями?**
A: Используйте интеграционные тесты с `ModulePipeline` классом.

---

**Версия**: 1.0
**Дата**: 2025-12-28
**Тип**: Практическое руководство для разработчиков
