# Установка

## Что понадобится

Для работы стартера нужен PHP `^8.5`. 
Также потребуются расширения `pdo_pgsql` и `redis`.

Стартер можно запустить как через Docker, так и без контейнеров.

В Docker уже подготовлены `PHP-FPM`, `PostgreSQL 17`, `Redis`, `Nginx` и `Mailpit`. 
Для запуска достаточно установить Docker с Compose и PHP, который понадобится для первоначальной настройки.

При запуске без Docker потребуется самостоятельно установить и настроить `Composer`, `PostgreSQL` и `Redis`.

Фронтенд находится в каталоге `frontend/`. 
Для него нужен `Node.js` версии ^22.18.0 || >=24.12.0.
Можно использовать локально установленный Node.js с npm или готовый контейнер.

## Подготовка окружения

```bash
./run setup
```

Скрипт `bin/setup` создает `.env` из `.env.example`, генерирует отсутствующие секреты и готовит каталоги `var/`.
Уже заполненные значения скрипт оставляет как есть.

Перед запуском задайте собственный DB_PASSWORD
## Запуск backend в Docker

После подготовки окружения можно запустить контейнеры и установить зависимости:
```bash
./run up -d --build
./run composer install
./run migrate --no-interaction
```

`run` передает обычные команды Docker Compose, а для Composer, миграций и консоли выполняет команды в сервисе `app`.

Backend доступен на `http://localhost:8080`, Swagger UI - на `http://localhost:8081`, письма Mailpit - на `http://localhost:8025`.

Проверка доступности API:

```bash
curl http://localhost:8080/health/live
```

`PostgreSQL` и `Redis` опубликованы на `127.0.0.1:5432` и `127.0.0.1:6379`.
Перед запуском стоит убедиться, что необходимые порты свободны.

## Запуск интерфейса

Фронтенд можно запустить локально через Vite или в Docker.

Для локального запуска требуется установить зависимости и запустить сервер разработки:

```bash
# frontend/
npm ci
npm run dev -- --host 127.0.0.1 --port 5173 --strictPort
```

Для запуска через Docker:

```bash
docker compose run --rm frontend npm ci
docker compose --profile frontend up -d frontend
```

После запуска интерфейс будет доступен по адресу http://localhost:5173.
`frontend/vite.config.ts` пересылает `/api` в backend и удаляет этот префикс: `/api/auth/login` становится `/auth/login`.
В контейнере цель прокси - `http://nginx`, на хосте - `http://127.0.0.1:8080`.

## Backend без контейнера

Если PostgreSQL и Redis уже установлены локально, приложение можно запустить без контейнеров.

Для начала необходимо настроить подключения в .env. В частности, указать локальный адрес базы данных и Redis:

```dotenv
DB_HOST=127.0.0.1
REDIS_URL=redis://127.0.0.1:6379
```

Также нужно указать порт PostgreSQL и создать базу данных с пользователем, соответствующими `DB_NAME`, `DB_USER` и `DB_PASSWORD`. 
У пользователя должны быть права на работу со схемой базы данных.

Отдельно нужно проверить `MAILER_DSN`. 
Адрес mailpit работает внутри сети Docker, поэтому при локальном запуске потребуется указать доступный SMTP-сервер.

После настройки окружения установите зависимости, выполните миграции и запустите приложение:

```bash
composer install
php bin/doctrine migrations:migrate --no-interaction
php -S 127.0.0.1:8080 -t public public/index.php
```

## Данные и фоновые задачи

После миграций можно отдельно добавить демонстрационных пользователей и включить worker:

```bash
./run console db:seed
./run --profile queue up -d worker
```

Команда seed работает по умолчанию только в `local` и `testing`.
Она создает `admin@example.test`, `member@example.test` и `pending@example.test` с паролем `ChangeMe12345!`.