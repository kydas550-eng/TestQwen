# 🗄️ Архиватор идей: Telegram-бот для сбора артефактов

Telegram-бот, который агрегирует пересланные посты из каналов, нормализует их с помощью локальной LLM в структурированные карточки данных и сохраняет в базу знаний для последующего семантического поиска.

## 📋 Оглавление

- [Возможности](#возможности)
- [Архитектура](#архитектура)
- [Требования](#требования)
- [Быстрый старт](#быстрый-старт)
- [Настройка окружения](#настройка-окружения)
- [Запуск через Docker](#запуск-через-docker)
- [Локальная разработка](#локальная-разработка)
- [Использование бота](#использование-бота)
- [API и поиск](#api-и-поиск)
- [Тестирование](#тестирование)
- [Troubleshooting](#troubleshooting)
- [Лицензия](#лицензия)

## ✨ Возможности

- **Прием пересланных сообщений**: Автоматический парсинг текста, ссылок и метаданных из пересланных постов.
- **LLM-структурирование**: Локальная модель (Llama 3.2 через Ollama) преобразует сырой текст в структурированные карточки:
  - Сущность (инструмент, новость, метод)
  - Описание (краткая суть)
  - Ссылки (исходные URL)
  - Источник (название канала)
  - Теги (автоматическая генерация)
- **Семантический поиск**: Векторные эмбеддинги (nomic-embed-text) + pgvector для быстрого поиска по смыслу.
- **Асинхронная обработка**: Очередь задач на Celery + Redis предотвращает зависание бота при инференсе модели.
- **Масштабируемость**: Полная контейнеризация через Docker Compose.

## 🏗️ Архитектура

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│   Telegram  │────▶│   Aiogram    │────▶│   Celery    │
│    Bot API  │     │    (Bot)     │     │   Worker    │
└─────────────┘     └──────────────┘     └──────┬──────┘
                                                │
                    ┌──────────────┐            ▼
                    │    Ollama    │◀────┌─────────────┐
                    │ (Llama 3.2)  │     │   Redis     │
                    └──────────────┘     │  (Queue)    │
                                         └─────────────┘
                                                │
                    ┌──────────────┐            ▼
                    │   PostgreSQL │◀────┌─────────────┐
                    │  + pgvector  │     │  Embedding  │
                    └──────────────┘     │   Model     │
                                         └─────────────┘
```

**Компоненты:**
- `bot`: Telegram-бот на aiogram 3.x
- `worker`: Celery worker для обработки задач LLM
- `ollama`: Сервис с локальной LLM (Llama 3.2) и embedding-моделью
- `redis`: Брокер сообщений для очереди задач
- `postgres`: База данных с расширением pgvector
- `adminer`: Веб-интерфейс для просмотра БД (опционально)

## 📦 Требования

### Минимальные
- **CPU**: 4+ ядра
- **RAM**: 16 GB (для моделей 7B на CPU)
- **Disk**: 20 GB свободного места
- **Docker**: 20.10+ и Docker Compose 2.0+

### Рекомендуемые (с GPU)
- **GPU**: NVIDIA с 8+ GB VRAM (CUDA 11.8+)
- **RAM**: 32 GB
- **Docker**: С поддержкой NVIDIA Container Toolkit

### Программные зависимости
- Python 3.10+
- Git
- Make (опционально, для удобства)

## 🚀 Быстрый старт

### 1. Клонирование репозитория

```bash
git clone https://github.com/kydas550-eng/TestQwen.git
cd TestQwen
```

### 2. Настройка переменных окружения

Скопируйте шаблон файла окружения:

```bash
cp .env.example .env
```

Отредактируйте `.env` и укажите свои значения:

```ini
# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token_here

# Ollama Settings
OLLAMA_HOST=http://ollama:11434
OLLAMA_MODEL=llama3.2
OLLAMA_EMBED_MODEL=nomic-embed-text

# Database
POSTGRES_USER=archivator
POSTGRES_PASSWORD=secure_password_change_me
POSTGRES_DB=ideas_archive
DATABASE_URL=postgresql://archivator:secure_password_change_me@postgres:5432/ideas_archive

# Redis
REDIS_URL=redis://redis:6379/0

# Celery
CELERY_BROKER_URL=redis://redis:6379/0
CELERY_RESULT_BACKEND=redis://redis:6379/0

# Logging
LOG_LEVEL=INFO
```

> ⚠️ **Важно**: Замените `your_bot_token_here` на токен вашего бота от [@BotFather](https://t.me/BotFather) и измените пароль базы данных!

### 3. Запуск через Docker Compose

#### Без GPU (CPU-версия)

```bash
docker compose up -d
```

#### С GPU (NVIDIA)

Убедитесь, что установлен [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html), затем:

```bash
docker compose --profile gpu up -d
```

### 4. Проверка статуса

```bash
# Просмотр логов всех сервисов
docker compose logs -f

# Проверка состояния контейнеров
docker compose ps
```

Первый запуск может занять 5-10 минут (скачивание образов, инициализация БД, загрузка моделей).

## ⚙️ Настройка окружения

### Получение токена Telegram бота

1. Откройте [@BotFather](https://t.me/BotFather) в Telegram
2. Отправьте команду `/newbot`
3. Следуйте инструкциям: укажите имя и юзернейм бота
4. Скопируйте полученный токен (выглядит как `123456789:ABCdefGHIjklMNOpqrsTUVwxyz`)
5. Вставьте токен в `.env` файл в переменную `TELEGRAM_BOT_TOKEN`

### Добавление бота в каналы

Чтобы бот мог получать пересланные сообщения из каналов:
1. Добавьте бота как участника в нужные каналы (через настройки канала → Добавить участника)
2. Или используйте пересылку от личного аккаунта (бот получит сообщение как "пересланное от")

### Настройка LLM моделей

По умолчанию используются:
- **Основная модель**: `llama3.2` (3B параметров, быстрая, подходит для CPU)
- **Embedding модель**: `nomic-embed-text` (для векторного поиска)

Для изменения моделей отредактируйте `.env`:

```ini
OLLAMA_MODEL=mistral  # или qwen, llama3, etc.
OLLAMA_EMBED_MODEL=all-minilm
```

Модели автоматически загружаются при первом запуске Ollama.

## 🐳 Запуск через Docker

### Основные команды

```bash
# Запуск всех сервисов
docker compose up -d

# Остановка всех сервисов
docker compose down

# Перезапуск конкретного сервиса
docker compose restart bot

# Просмотр логов
docker compose logs -f bot
docker compose logs -f worker
docker compose logs -f ollama

# Масштабирование воркеров (для нагрузки)
docker compose up -d --scale worker=3
```

### Профили запуска

- **default**: CPU-версия (все сервисы кроме GPU-оптимизированного ollama)
- **gpu**: Версия с поддержкой NVIDIA GPU

```bash
# Запуск с GPU
docker compose --profile gpu up -d

# Запуск без админки (экономия ресурсов)
docker compose up -d --no-deps adminer
```

### Управление базой данных

```bash
# Подключение к PostgreSQL
docker compose exec postgres psql -U archivator -d ideas_archive

# Создание миграций (если изменилась схема)
docker compose exec bot alembic revision --autogenerate -m "Description"
docker compose exec bot alembic upgrade head

# Бэкап базы данных
docker compose exec postgres pg_dump -U archivator ideas_archive > backup.sql

# Восстановление из бэкапа
cat backup.sql | docker compose exec -T postgres psql -U archivator -d ideas_archive
```

## 💻 Локальная разработка

### Установка зависимостей

```bash
# Создание виртуального окружения
python -m venv venv
source venv/bin/activate  # Linux/Mac
# или
venv\Scripts\activate  # Windows

# Установка пакетов
pip install -r requirements.txt
pip install -r requirements-dev.txt  # для разработки и тестов
```

### Запуск сервисов по отдельности

```bash
# Только БД и Redis
docker compose up -d postgres redis ollama

# Бот в режиме разработки (с авто-релоадом)
docker compose up -d bot-dev

# Или локально
python -m src.bot.main
```

### Pre-commit хуки

```bash
# Установка pre-commit
pre-commit install

# Запуск всех проверок вручную
pre-commit run --all-files
```

## 🤖 Использование бота

### Основные команды

| Команда | Описание |
|---------|----------|
| `/start` | Приветствие и инструкция |
| `/help` | Подробная справка по использованию |
| `/search <запрос>` | Семантический поиск по базе знаний |
| `/stats` | Статистика сохраненных артефактов |
| `/export [format]` | Экспорт данных (JSON, CSV) |

### Как сохранить артефакт

1. **Перешлите сообщение** из любого канала боту
2. Бот подтвердит получение: *"📥 Сообщение получено, обрабатываю..."*
3. Через несколько секунд (зависит от мощности сервера) бот пришлет структурированную карточку:
   ```
   ✅ Артефакт сохранен!

   📌 Сущность: Инструмент
   📝 Описание: Краткая суть...
   🔗 Ссылки: https://...
   📢 Источник: @channel_name
   🏷️ Теги: #тег1 #тег2
   ```

### Поиск по базе

Отправьте команду:
```
/search как настроить асинхронную обработку
```

Бот вернет топ-3 наиболее релевантных артефактов с кратким описанием и ссылками.

### Примеры использования

```
# Поиск по тегу
/search #machine_learning

# Поиск по источнику
/search источник:@tech_channel

# Комбинированный запрос
/search нейросети локальные инструменты
```

## 🔌 API и поиск

### REST API

Бот предоставляет HTTP API для внешней интеграции (например, для RAG-пайплайнов):

```bash
# Поиск артефактов
curl -X POST http://localhost:8000/api/v1/search \
  -H "Content-Type: application/json" \
  -d '{"query": "асинхронная обработка", "limit": 5}'

# Добавление артефакта вручную
curl -X POST http://localhost:8000/api/v1/artifacts \
  -H "Content-Type: application/json" \
  -d '{
    "entity": "Метод",
    "description": "...",
    "links": ["https://..."],
    "source": "Manual",
    "tags": ["manual"]
  }'

# Получение статистики
curl http://localhost:8000/api/v1/stats
```

**Документация API**: Откройте `http://localhost:8000/docs` (Swagger UI) после запуска.

### Интеграция с RAG

Пример использования в внешнем приложении:

```python
import requests

def search_knowledge_base(query: str, limit: int = 3):
    response = requests.post(
        "http://localhost:8000/api/v1/search",
        json={"query": query, "limit": limit}
    )
    return response.json()["results"]

# Использование
results = search_knowledge_base("как развернуть LLM локально")
for artifact in results:
    print(f"{artifact['entity']}: {artifact['description']}")
```

## 🧪 Тестирование

### Запуск тестов

```bash
# Все тесты
pytest

# С покрытием
pytest --cov=src --cov-report=html

# Конкретный модуль
pytest tests/test_bot.py -v

# Тесты с логированием
pytest -vv -s
```

### Покрытие кода

После запуска тестов с флагом `--cov` откройте `htmlcov/index.html` в браузере для детального отчета.

Текущее покрытие: **>85%**

### Типы тестов

- **Unit-тесты**: Проверка отдельных функций и классов
- **Integration-тесты**: Взаимодействие между компонентами (бот → Celery → LLM → БД)
- **E2E-тесты**: Полные сценарии использования бота

### Моки и фикстуры

Тесты используют моки для внешних сервисов (Ollama, Telegram API). Пример:

```python
@pytest.fixture
def mock_ollama():
    with patch("src.llm.client.OllamaClient") as mock:
        mock.return.generate.return_value = {"entity": "Test", ...}
        yield mock
```

## 🔧 Troubleshooting

### Бот не отвечает на команды

1. Проверьте логи: `docker compose logs bot`
2. Убедитесь, что токен правильный в `.env`
3. Проверьте подключение к Telegram: `curl https://api.telegram.org/bot<YOUR_TOKEN>/getMe`

### Ollama медленно отвечает

- **На CPU**: Нормально для моделей 7B (1-3 минуты на запрос). Используйте `llama3.2` (3B) для скорости.
- **На GPU**: Убедитесь, что запущен профиль `gpu` и установлен NVIDIA Container Toolkit.
- Проверьте загрузку: `docker compose exec ollama nvidia-smi` (для GPU)

### Ошибки подключения к базе данных

```bash
# Проверьте, что БД запущена
docker compose ps postgres

# Проверьте логи БД
docker compose logs postgres

# Пересоздайте БД (осторожно: удалит все данные!)
docker compose down -v
docker compose up -d postgres
docker compose exec bot alembic upgrade head
```

### Celery worker не обрабатывает задачи

1. Проверьте очередь Redis: `docker compose exec redis redis-cli llen celery`
2. Перезапустите воркер: `docker compose restart worker`
3. Убедитесь, что `REDIS_URL` правильный в `.env`

### Модели не загружаются

```bash
# Проверьте доступность Ollama
docker compose exec ollama curl http://localhost:11434/api/tags

# Загрузите модель вручную
docker compose exec ollama ollama pull llama3.2
docker compose exec ollama ollama pull nomic-embed-text
```

### Недостаточно памяти

- Уменьшите размер модели: используйте `llama3.2:1b` или `phi3`
- Добавьте swap-файл на сервере
- Отключите неиспользуемые сервисы (например, `adminer`)

## 📊 Мониторинг

### Метрики

- Количество сохраненных артефактов: `/stats` в боте или `GET /api/v1/stats`
- Время обработки задачи: логи Celery worker
- Использование ресурсов: `docker stats`

### Логирование

Уровень логирования настраивается в `.env`:

```ini
LOG_LEVEL=DEBUG  # INFO, WARNING, ERROR, CRITICAL
```

Логи пишутся в стандартный вывод и собираются через `docker compose logs`.

## 🛡️ Безопасность

- **Токены**: Никогда не коммитьте `.env` в гит! Файл добавлен в `.gitignore`.
- **Пароли БД**: Измените пароль по умолчанию перед продакшеном.
- **HTTPS**: Для продакшена настройте reverse proxy (nginx) с SSL.
- **Rate limiting**: Встроена защита от спама в обработчиках бота.

## 🤝 Вклад в проект

1. Форкните репозиторий
2. Создайте ветку (`git checkout -b feature/amazing-feature`)
3. Закоммитьте изменения (`git commit -m 'Add amazing feature'`)
4. Отправьте в ветку (`git push origin feature/amazing-feature`)
5. Откройте Pull Request

### Требования к коду

- Следуйте PEP 8
- Покрывайте новый код тестами (>85% покрытие)
- Используйте type hints
- Добавляйте docstrings

## 📄 Лицензия

MIT License - см. файл [LICENSE](LICENSE) для деталей.

## 📞 Контакты

- **Issues**: [GitHub Issues](https://github.com/kydas550-eng/TestQwen/issues)
- **Discussions**: [GitHub Discussions](https://github.com/kydas550-eng/TestQwen/discussions)

---

**Made with ❤️ using AI and Python**

*Версия: 1.0.0 | Последнее обновление: 2024*