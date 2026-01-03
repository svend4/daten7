# Приложение A: Динамическая Plug-in архитектура модулей

## Назначение документа
Этот документ является **расширением** основной документации и описывает систему динамического управления модулями - установку, удаление, обновление и создание собственных модулей.

---

## Концепция динамических модулей

### Ключевые принципы

**1. Plug-and-Play**
Модули можно устанавливать и удалять без перезапуска системы

**2. Независимость**
Каждый модуль - автономный компонент с собственными зависимостями

**3. Версионирование**
Поддержка множественных версий одного модуля

**4. Репозиторий**
Центральное хранилище модулей с возможностью публикации своих

**5. Горячая замена**
Обновление модулей без остановки системы

---

## Архитектура системы модулей

### Структура каталогов

```
/app
├── core/                    # Ядро системы (неизменяемое)
│   ├── module_loader.py     # Загрузчик модулей
│   ├── module_registry.py   # Реестр модулей
│   └── module_api.py        # API для модулей
│
├── modules/                 # Установленные модули
│   ├── installed/
│   │   ├── email_extractor@1.0.0/
│   │   │   ├── module.json       # Манифест модуля
│   │   │   ├── __init__.py
│   │   │   ├── processor.py      # Логика обработки
│   │   │   ├── ui_config.json    # Конфигурация UI
│   │   │   ├── requirements.txt  # Зависимости
│   │   │   └── README.md
│   │   │
│   │   ├── csv_merge@2.1.0/
│   │   └── pdf_parser@1.5.3/
│   │
│   ├── disabled/            # Отключенные модули
│   └── cache/               # Кэш результатов
│
├── module_repository/       # Локальный репозиторий
│   ├── index.json          # Каталог модулей
│   └── packages/           # Скачанные пакеты
│
└── config/
    └── modules.yaml        # Конфигурация модулей
```

---

## Манифест модуля (module.json)

Каждый модуль содержит файл `module.json` с метаданными:

```json
{
  "id": "email_extractor",
  "name": "Извлечение email-адресов",
  "version": "1.0.0",
  "author": {
    "name": "John Doe",
    "email": "john@example.com",
    "url": "https://github.com/johndoe/email-extractor"
  },
  "description": "Извлекает email-адреса из текстовых данных",
  "category": "text_processing",
  "tags": ["email", "extraction", "text", "regex"],

  "compatibility": {
    "core_version": ">=1.0.0",
    "python_version": ">=3.8"
  },

  "entry_point": "processor.EmailExtractor",

  "input": {
    "accepted_formats": ["txt", "md", "html", "csv", "json"],
    "max_file_size": "50MB"
  },

  "output": {
    "formats": ["json", "csv", "txt"]
  },

  "parameters": [
    {
      "name": "case_sensitive",
      "type": "boolean",
      "default": false,
      "description": "Учитывать регистр"
    },
    {
      "name": "unique_only",
      "type": "boolean",
      "default": true,
      "description": "Только уникальные адреса"
    }
  ],

  "dependencies": {
    "python": ["validators>=0.20.0", "email-validator>=1.1.0"],
    "system": [],
    "modules": []
  },

  "license": "MIT",
  "repository": "https://github.com/johndoe/email-extractor",
  "homepage": "https://email-extractor.example.com",
  "documentation": "https://docs.email-extractor.example.com",

  "resources": {
    "memory": "256MB",
    "cpu": "low"
  },

  "ui": {
    "icon": "envelope",
    "color": "#3498db",
    "config_file": "ui_config.json"
  }
}
```

---

## Жизненный цикл модуля

### 1. Установка модуля

**Способ 1: Из URL**
```bash
module install https://modules.example.com/email-extractor@1.0.0.zip
```

**Способ 2: Из репозитория**
```bash
module install email-extractor
module install email-extractor@1.0.0  # Конкретная версия
```

**Способ 3: Из локального файла**
```bash
module install ./email-extractor-1.0.0.zip
```

**Процесс установки:**
```
1. Скачивание пакета
2. Валидация module.json
3. Проверка зависимостей
4. Установка Python зависимостей
5. Регистрация в реестре
6. Применение конфигурации
7. Активация модуля
```

### 2. Обновление модуля

```bash
module update email-extractor          # До последней версии
module update email-extractor@2.0.0    # До конкретной версии
module update --all                    # Обновить все модули
```

**Стратегии обновления:**
- **Hot-swap**: Замена без остановки (для совместимых версий)
- **Graceful**: Завершение текущих задач, затем обновление
- **Scheduled**: Обновление по расписанию

### 3. Удаление модуля

```bash
module uninstall email-extractor
module uninstall email-extractor@1.0.0  # Конкретная версия
```

**Процесс удаления:**
```
1. Проверка зависимостей (используется ли другими модулями)
2. Остановка активных задач модуля
3. Удаление из реестра
4. Очистка кэша
5. Удаление файлов
```

### 4. Включение/Отключение

```bash
module enable email-extractor   # Включить
module disable email-extractor  # Отключить (не удаляя)
```

---

## Управление модулями через UI

### Интерфейс менеджера модулей

```
┌────────────────────────────────────────────────────────┐
│ 🔌 МЕНЕДЖЕР МОДУЛЕЙ                                    │
├────────────────────────────────────────────────────────┤
│                                                        │
│ [Установленные] [Доступные] [Обновления] [Настройки] │
│                                                        │
├────────────────────────────────────────────────────────┤
│ УСТАНОВЛЕННЫЕ МОДУЛИ (15)                              │
│                                                        │
│ ┌──────────────────────────────────────────────────┐ │
│ │ ✅ Извлечение email-адресов         v1.0.0      │ │
│ │    Автор: John Doe                              │ │
│ │    Категория: Текстовая обработка               │ │
│ │    [⚙️ Настроить] [🔄 Обновить] [🗑️ Удалить]    │ │
│ └──────────────────────────────────────────────────┘ │
│                                                        │
│ ┌──────────────────────────────────────────────────┐ │
│ │ ✅ Слияние CSV файлов               v2.1.0      │ │
│ │    Автор: Jane Smith                            │ │
│ │    Категория: Трансформация данных              │ │
│ │    🆕 Доступно обновление: v2.2.0               │ │
│ │    [⚙️ Настроить] [🔄 Обновить] [🗑️ Удалить]    │ │
│ └──────────────────────────────────────────────────┘ │
│                                                        │
│ ┌──────────────────────────────────────────────────┐ │
│ │ ⏸️ PDF парсер                       v1.5.3      │ │
│ │    (Отключен)                                   │ │
│ │    [▶️ Включить] [🗑️ Удалить]                   │ │
│ └──────────────────────────────────────────────────┘ │
│                                                        │
├────────────────────────────────────────────────────────┤
│ [+ Установить из URL] [📦 Установить из файла]       │
└────────────────────────────────────────────────────────┘
```

### Установка из репозитория

```
┌────────────────────────────────────────────────────────┐
│ 📦 РЕПОЗИТОРИЙ МОДУЛЕЙ                                 │
├────────────────────────────────────────────────────────┤
│ Поиск: [text processing____________] [🔍]             │
│                                                        │
│ Категории: [Все ▼] Сортировка: [Популярные ▼]        │
│                                                        │
├────────────────────────────────────────────────────────┤
│                                                        │
│ ┌──────────────────────────────────────────────────┐ │
│ │ 📧 Извлечение email-адресов         v1.2.0      │ │
│ │    ⭐⭐⭐⭐⭐ (4.8) • 1,245 загрузок              │ │
│ │    Извлекает email, URL, телефоны из текста     │ │
│ │    [ℹ️ Подробнее] [📥 Установить]               │ │
│ └──────────────────────────────────────────────────┘ │
│                                                        │
│ ┌──────────────────────────────────────────────────┐ │
│ │ 🤖 NLP Анализ                       v2.0.1      │ │
│ │    ⭐⭐⭐⭐☆ (4.2) • 823 загрузки                │ │
│ │    Токенизация, NER, sentiment analysis         │ │
│ │    [ℹ️ Подробнее] [📥 Установить]               │ │
│ └──────────────────────────────────────────────────┘ │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## Репозиторий модулей

### Публичный репозиторий

**URL:** `https://modules.datapipeline.io`

**Структура index.json:**
```json
{
  "version": "1.0.0",
  "updated": "2025-12-28T10:00:00Z",
  "modules": [
    {
      "id": "email_extractor",
      "name": "Извлечение email-адресов",
      "latest_version": "1.2.0",
      "category": "text_processing",
      "downloads": 1245,
      "rating": 4.8,
      "author": "John Doe",
      "license": "MIT",
      "package_url": "https://modules.datapipeline.io/packages/email-extractor-1.2.0.zip",
      "checksum": "sha256:abc123...",
      "versions": [
        {
          "version": "1.2.0",
          "release_date": "2025-12-15",
          "changes": "Добавлена поддержка международных доменов",
          "package_url": "...",
          "checksum": "sha256:..."
        },
        {
          "version": "1.1.0",
          "release_date": "2025-11-10",
          "package_url": "...",
          "checksum": "sha256:..."
        }
      ]
    }
  ]
}
```

### Создание собственного репозитория

```yaml
# config/repositories.yaml
repositories:
  - name: official
    url: https://modules.datapipeline.io
    priority: 1
    enabled: true

  - name: company-internal
    url: https://modules.mycompany.com
    priority: 2
    enabled: true
    auth:
      type: token
      token: ${REPO_TOKEN}

  - name: local
    type: local
    path: /opt/module-repository
    priority: 3
    enabled: true
```

---

## Безопасность модулей

### Подпись модулей

```bash
# Подписать модуль при публикации
module sign email-extractor-1.0.0.zip --key private.key

# Проверка подписи при установке
module install email-extractor --verify-signature
```

### Sandboxing

Модули выполняются в изолированной среде с ограничениями:

```json
{
  "sandbox": {
    "filesystem": {
      "allowed_paths": ["/tmp", "/data/uploads"],
      "readonly_paths": ["/app/config"],
      "max_storage": "1GB"
    },
    "network": {
      "allowed_domains": ["api.example.com"],
      "allow_all": false
    },
    "resources": {
      "max_memory": "512MB",
      "max_cpu_percent": 50,
      "max_execution_time": "5m"
    }
  }
}
```

### Проверка безопасности

```bash
# Сканирование модуля на уязвимости
module scan email-extractor-1.0.0.zip

# Аудит установленных модулей
module audit --all
```

---

## CLI команды

### Список всех команд

```bash
# Информация о модулях
module list                    # Список установленных
module list --all             # Включая отключенные
module search <query>         # Поиск в репозитории
module info <module_id>       # Информация о модуле

# Управление
module install <module_id>    # Установить
module uninstall <module_id>  # Удалить
module update <module_id>     # Обновить
module enable <module_id>     # Включить
module disable <module_id>    # Отключить

# Разработка
module create <module_name>   # Создать шаблон модуля
module validate <path>        # Валидация module.json
module pack <path>            # Упаковать модуль
module publish <package>      # Опубликовать в репозиторий

# Обслуживание
module cache clear            # Очистить кэш
module logs <module_id>       # Логи модуля
module stats                  # Статистика использования
```

---

## Конфигурация системы

### Файл modules.yaml

```yaml
system:
  auto_update: true
  check_updates_interval: 24h
  allow_beta_versions: false
  telemetry: true

repositories:
  - official
  - company-internal

modules:
  email_extractor:
    enabled: true
    auto_update: true
    config:
      case_sensitive: false
      unique_only: true

  csv_merge:
    enabled: true
    auto_update: false
    version: "2.1.0"  # Закрепить версию

cache:
  enabled: true
  max_size: 5GB
  ttl: 7d

logging:
  level: INFO
  file: /var/log/modules.log
  max_size: 100MB
```

---

## Версионирование модулей

### Semantic Versioning

Модули используют semver: `MAJOR.MINOR.PATCH`

**Примеры:**
- `1.0.0` → `1.0.1` - исправление ошибок (совместимо)
- `1.0.0` → `1.1.0` - новая функциональность (совместимо)
- `1.0.0` → `2.0.0` - breaking changes (несовместимо)

### Одновременные версии

Система может использовать несколько версий одного модуля:

```
modules/installed/
├── email_extractor@1.0.0/  # Используется в старых pipeline
├── email_extractor@2.0.0/  # Используется в новых pipeline
└── email_extractor@latest -> email_extractor@2.0.0/
```

---

## Совместимость модулей

### Матрица совместимости

```json
{
  "compatibility_matrix": {
    "core_version": "1.5.0",
    "compatible_modules": {
      "email_extractor": ["1.0.0", "1.1.0", "1.2.0"],
      "csv_merge": ["2.0.0+"],
      "pdf_parser": ["1.5.0-1.5.9"]
    }
  }
}
```

### Проверка перед установкой

```bash
module check-compatibility email-extractor@3.0.0
# Output: ⚠️  Требуется core version >= 2.0.0 (установлено: 1.5.0)
```

---

**Версия**: 1.0
**Дата**: 2025-12-28
**Тип**: Приложение к основной документации
