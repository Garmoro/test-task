# Conversation Summarizer

Учебный монорепозиторий для автоматического резюмирования диалогов пользователей с ботами и операторами.

## Стек и архитектура

Основная связка:

~~~text
Next.js -> NestJS -> Prisma/PostgreSQL -> Go summarizer -> Redis + Ollama
~~~

- apps/web — Next.js-клиент.
- apps/api — NestJS API/BFF.
- apps/api/prisma — Prisma schema и миграции.
- conversation-summarizer — Go-микросервис из технического задания.
- PostgreSQL — постоянное хранение диалогов, сообщений и версий summary.
- Redis — кэш последнего summary.
- Ollama — локальная OpenAI-compatible LLM.
- docker-compose.yml — полный стек.

Поток данных:

~~~text
Browser
  -> Next.js :3000
  -> NestJS :4000
  -> Prisma/PostgreSQL
  -> Go summarizer :8091
  -> Redis
  -> Ollama :11434
~~~

PostgreSQL является источником истины. Redis хранит временную копию последнего summary.

## Требования

- Node.js 20+;
- npm;
- Go 1.23+;
- Podman Desktop;
- запущенная Podman machine;
- podman compose.

Проверка:

~~~powershell
node --version
npm --version
go version
podman --version
podman machine list
podman compose version
~~~

Docker Desktop не обязателен. Compose-файл совместим с Docker Compose, но основной сценарий для Windows использует Podman Desktop.

## Запуск через Podman

~~~powershell
cd "C:\Users\user\Documents\Codex\2026-09-10\x20-git-github-x20"
podman machine start podman-machine-default
podman compose up --build
~~~

Для фонового запуска:

~~~powershell
podman compose up -d --build
~~~

Первый запуск скачивает модель qwen2.5:0.5b и может занимать время.

Проверка:

~~~powershell
podman ps
podman ps -a
curl.exe http://localhost:8091/health
curl.exe http://localhost:4000/health
curl.exe http://localhost:3000
~~~

Адреса:

~~~text
Next.js:        http://localhost:3000
NestJS API:     http://localhost:4000
NestJS Swagger: http://localhost:4000/docs
Go health:      http://localhost:8091/health
Go Swagger:     http://localhost:8091/swagger/index.html
Ollama:         http://localhost:11434
~~~

Контейнер ollama-model после успешной загрузки может иметь статус Exited (0). Это нормальный одноразовый контейнер.

## Остановка и логи

~~~powershell
podman compose down
~~~

Команда выше останавливает контейнеры без удаления данных. Команда podman compose down -v удаляет volumes PostgreSQL и Ollama вместе с данными и моделью.

~~~powershell
podman compose logs --tail=100 api
podman compose logs --tail=100 summarizer
podman compose logs --tail=100 ollama
podman compose logs -f summarizer ollama
~~~

## API NestJS

| Метод | Путь | Назначение |
|---|---|---|
| GET | /health | PostgreSQL и Go health |
| GET | /api/v1/conversations | список диалогов |
| POST | /api/v1/conversations | создать диалог |
| GET | /api/v1/conversations/{id} | диалог и сообщения |
| POST | /api/v1/conversations/{id}/messages | добавить сообщение |
| POST | /api/v1/conversations/{id}/summarize | создать summary |
| GET | /api/v1/conversations/{id}/summary | последнее summary |
| GET | /api/v1/conversations/{id}/summary/history | история summary |
| POST | /api/v1/conversations/{id}/summary/regenerate | новая версия без чтения кэша |

Swagger: http://localhost:4000/docs

## API Go summarizer

| Метод | Путь | Назначение |
|---|---|---|
| GET | /health | PostgreSQL и Redis health |
| POST | /api/v1/summaries | создать summary |
| GET | /api/v1/summaries/{conversation_id} | последнее summary |
| GET | /api/v1/summaries/{conversation_id}/history | история версий |
| POST | /api/v1/summaries/{conversation_id}/regenerate | игнорировать Redis |
| GET | /swagger/index.html | Swagger UI |

Примеры находятся в conversation-summarizer/examples/requests.http.

## Жизненный цикл summary

~~~text
Next.js
  -> NestJS
  -> Prisma/PostgreSQL: получить сообщения
  -> Go: проверить и ограничить вход
  -> Redis: cache hit/miss
  -> LLM через OpenAI-compatible API
  -> декодировать и нормализовать JSON
  -> проверить status/sentiment/priority/topics
  -> PostgreSQL: сохранить версию
  -> Redis: обновить кэш
  -> вернуть ответ браузеру
~~~

Ключ Redis:

~~~text
conversation-summary:{conversation_id}
~~~

Обычный GET:

~~~text
Redis hit  -> response
Redis miss -> PostgreSQL -> Redis.Set -> response
~~~

TTL задаётся REDIS_TTL_SECONDS. При regenerate чтение Redis пропускается, а новая версия сохраняется в PostgreSQL и записывается в Redis.

## LLM

Go обращается к POST http://ollama:11434/v1/chat/completions.

Модель по умолчанию в текущем Compose: qwen2.5:0.5b.

Ожидаемый результат:

~~~json
{
  "summary": "Краткое содержание.",
  "problem": "Основная проблема.",
  "actions_taken": ["Повторный вход"],
  "status": "unresolved",
  "sentiment": "negative",
  "priority": "high",
  "topics": ["payment"]
}
~~~

Допустимые значения:

~~~text
status: resolved, unresolved, waiting_user, waiting_operator, unknown
sentiment: positive, neutral, negative, angry
priority: low, medium, high, critical
~~~

История ограничивается MAX_MESSAGES и MAX_CONTENT_LENGTH. Retry применяется к сетевым ошибкам, timeout, HTTP 429 и HTTP 5xx.

Маленькая локальная модель иногда возвращает строку вместо массива или русские enum. Go нормализует безопасные варианты:

~~~text
actions_taken: "Повторный вход" -> ["Повторный вход"]
Ожидаемый -> unknown
Высокий -> high
Негативный -> negative
~~~

## PostgreSQL и миграции

Prisma хранит conversations и messages. Go-миграция создаёт conversation_summaries, индекс истории и ограничения status, sentiment, priority.

В summary сохраняются:

~~~text
id, conversation_id, summary, problem, actions_taken,
status, sentiment, priority, topics, model,
prompt_tokens, completion_tokens, llm_duration_ms,
created_at, updated_at
~~~

Исходные приватные сообщения Go не сохраняет: он получает их от NestJS для текущей генерации. Поэтому regenerate снова получает сообщения из Prisma.

## Логирование и приватность

Go использует slog. Логируются запуск, HTTP method/path/duration, conversation_id, cache hit/miss, retry, модель, token usage, длительность LLM, ошибки и success/failure.

Не логируются API-ключи, пароли, Authorization headers, access tokens, полный prompt и содержимое приватных сообщений.

LLM-метрики сейчас сохраняются в PostgreSQL и логируются через slog. Отдельный Prometheus endpoint /metrics пока не реализован.

## Тесты

~~~powershell
npm test
npm --prefix apps/api install
npm --prefix apps/api run prisma:generate
npm --prefix apps/api test
npm --prefix apps/api run build
npm --prefix apps/web install
npm --prefix apps/web run build
cd conversation-summarizer
go mod download
go test ./...
cd ..
podman compose --profile test run --rm test
~~~

Go unit-тесты используют mock LLM, mock repository и mock Redis. Проверяются validation, enum, cache hit/miss, сохранение, regenerate, неправильный JSON, пустой ответ, retry после 5xx и timeout.

## CI/CD

GitHub workflow находится в .github/workflows/ci.yml. Он проверяет Node.js, Go, Prisma generate, NestJS test/build и Next.js build.

Gitea workflow находится в conversation-summarizer/.gitea/workflows/ci-cd.yml. Он выполняет тесты, сборку NestJS/Next.js, Docker image build и ручной deploy через workflow_dispatch. Для deploy нужны self-hosted runner, Docker и DEPLOY_PATH.

## Команды

~~~powershell
npm test
npm run api:install
npm run api:test
npm run api:build
npm run web:install
npm run web:build
podman compose up --build
podman compose down
~~~

## Предыдущий учебный этап: Task Board

В корне сохранён первый учебный проект на чистом Node.js: server.js, public/, test/server.test.js и data/tasks.json.

Он демонстрирует схему:

~~~text
браузер -> fetch -> Node.js HTTP-сервер -> JSON-файл -> JSON-ответ
~~~

Запуск:

~~~powershell
npm start
~~~

Task Board сохранён как предыдущий этап и не является основной реализацией Conversation Summarizer.