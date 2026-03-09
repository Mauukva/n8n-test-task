# n8n News Autoposter

Автоматизированная система для создания новостных постов из YouTube видео с использованием n8n, Whisper AI и LLM.

## Возможности

- Скачивание аудио из YouTube видео (yt-dlp)
- Транскрибация аудио в текст (Groq Whisper API)
- Генерация новостных постов с помощью LLM (OpenRouter)
- Сохранение результатов в PostgreSQL
- Отправка постов в Telegram
- Обработка ошибок с логированием
- Автоматическая очистка временных файлов

## Технологический стек

- **n8n** 1.112.5 - платформа автоматизации workflow
- **PostgreSQL** 16 - база данных
- **yt-dlp** - скачивание аудио из YouTube
- **Groq Whisper API** - транскрибация речи
- **OpenRouter API** - генерация текста (GPT-4o-mini)
- **Telegram Bot API** - публикация постов
- **Docker & Docker Compose** - контейнеризация

## Требования

- Docker
- Docker Compose
- API ключи:
  - OpenRouter API key
  - Groq API key
  - Telegram Bot token

## Установка и запуск

### 1. Клонируйте репозиторий
```bash
git clone <repository-url>
cd n8n-test-task
```

### 2. Настройте переменные окружения
```bash
cp .env.example .env
```

Отредактируйте `.env` и заполните:
- `OPENROUTER_API_KEY` - ключ OpenRouter
- `GROQ_API_KEY` - ключ Groq API
- `TELEGRAM_BOT_TOKEN` - токен Telegram бота
- `TELEGRAM_CHAT_ID` - ID чата для отправки постов
- `N8N_ENCRYPTION_KEY` - сгенерируйте: `openssl rand -hex 16`

### 3. Запустите Docker Compose
```bash
docker compose up -d
```

### 4. Импортируйте workflow

1. Откройте n8n: http://localhost:5678
2. Перейдите в Workflows → Import from File
3. Загрузите `workflows/Workflow.json`
4. Настройте credentials:
   - **PostgreSQL**: host=postgres, port=5432, database=n8n, user/password из .env
   - **Telegram**: Bot token из .env
   - **Groq API**: Header Auth с `Authorization: Bearer <GROQ_API_KEY>`
   - **OpenRouter API**: Header Auth с `Authorization: Bearer <OPENROUTER_API_KEY>`

### 5. Активируйте workflow

В n8n интерфейсе активируйте импортированный workflow.

## Использование

Отправьте POST запрос на webhook:
```bash
curl -X POST http://localhost:5678/webhook/news-autoposter \
  -H "Content-Type: application/json" \
  -d '{"url": "https://www.youtube.com/watch?v=VIDEO_ID"}'
```

## Архитектура workflow
```
Webhook → Validate Input → Cleanup /tmp → Download Audio (yt-dlp) →
Read Binary Files → Transcribe (Whisper) → Generate Post (LLM) →
Prepare DB Data (Code) → Save to PostgreSQL → Send to Telegram →
Cleanup Files → Respond Success
```

### Обработка ошибок

При возникновении ошибок на любом этапе:
1. Ошибка логируется в таблицу `error_logs`
2. Отправляется уведомление в Telegram
3. Retry механизм (до 3 попыток для Whisper/LLM, 2 для Telegram)

## Структура базы данных

### Таблица `news_posts`
- `id` - PRIMARY KEY (SERIAL)
- `source_url` - URL исходного видео
- `transcript` - текст транскрибации
- `post_text` - сгенерированный пост
- `status` - статус ('draft', 'published')
- `created_at` - время создания

### Таблица `error_logs`
- `id` - PRIMARY KEY (SERIAL)
- `node_name` - имя ноды где произошла ошибка
- `error_message` - текст ошибки
- `input_data` - входные данные (JSONB)
- `created_at` - время ошибки

## Исправленные ошибки в исходном задании

1. `DB_POSTGRESDB_HSOT` → `DB_POSTGRESDB_HOST`
2. Добавлен volume `n8n_storage:/home/node/.n8n`
3. Удалены лишние порты PostgreSQL (используется внутренняя Docker сеть)
4. Удалена устаревшая директива `version` в docker-compose.yml
5. Исправлен `.env.example` (удалён trailing slash, добавлены инструкции)

## Команды управления
```bash
# Запуск
docker compose up -d

# Остановка
docker compose down

# Логи n8n
docker compose logs -f n8n

# Логи PostgreSQL
docker compose logs -f postgres

# Статус контейнеров
docker compose ps

# Перезапуск
docker compose restart
```

## Дополнительные заметки

- Workflow использует `/tmp` для временного хранения аудиофайлов
- Автоматическая очистка `/tmp/*.mp3` перед и после обработки
- Поддержка русского языка в Whisper (`language: ru`)
- Temperature=0.7 для баланса креативности и точности в LLM