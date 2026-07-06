# Bellboy — микросервисы расписания

Система для получения расписания, парсинга PDF и доставки уведомлений. Собрана вокруг событий в **RabbitMQ** и общего инфраструктурного слоя (**PostgreSQL / Redis / MailHog**) для локальной разработки.

## ▶️ Описание проекта

Проект состоит из 4 частей:

- **`composer`**: поднимает инфраструктуру (Postgres, Redis, RabbitMQ, MailHog)
- **`schedule-parser` (API: 8000)**: находит расписание/ссылки на PDF, публикует задания в RabbitMQ
- **`pdf-parser` (API: 8001)**: потребляет задания из RabbitMQ, скачивает/парсит PDF, публикует результат
- **`notification` (API: 8002)**: потребляет готовое расписание и отправляет уведомления (Telegram)

---

## 🔷 Архитектура

### Общая схема системы

```text
┌───────────────────────────────────────────────────────────────────────────────┐
│                               MICROSERVICES ECOSYSTEM                         │
└───────────────────────────────────────────────────────────────────────────────┘

                         Docker network: tasks (shared)
                                  │
                                  ▼
┌──────────────────────────┐   publish (DIRECT)    ┌──────────────────────────┐
│     schedule-parser       │──────────────────────▶│        RabbitMQ          │
│       (API: 8000)         │  exchange: schedule   │         (5672)           │
│                            │  rk/queue: schedule.pdf                       │
│  • HTTP API                │                         ┌──────────────────────┐
│  • APScheduler (hourly)    │                         │  RabbitMQ UI: 15672  │
└──────────┬────────────────┘                         └──────────────────────┘
           │                                                     │
           │ consume schedule.pdf                                 │ consume schedule.pdf
           ▼                                                     ▼
┌──────────────────────────┐   publish result (DIRECT)  ┌──────────────────────────┐
│        pdf-parser         │────────────────────────────▶│       notification       │
│       (API: 8001)         │ exchange: schedule          │        (API: 8002)       │
│                            │ rk/queue: schedule.pdf.result │                        │
│  • Rabbit consumer         │                              │  • Rabbit subscriber    │
│  • PDF download + parse    │                              │  • Telegram bot polling │
└──────────────────────────┘                              └──────────────────────────┘

┌───────────────────────────────────────────────────────────────────────────────┐
│                                 INFRASTRUCTURE                                │
└───────────────────────────────────────────────────────────────────────────────┘

PostgreSQL (5432)  Redis (6379)  MailHog SMTP (1025) + UI (8025)
```

### Компоненты системы

#### 1. **`composer`** (инфраструктура)

**Ответственность:**

- Поднять общие зависимости для разработки
- Единая Docker-сеть `tasks` для всех сервисов

**Технологии:**

- Docker Compose
- PostgreSQL 16, Redis 7, RabbitMQ 3.13, MailHog

#### 2. **`schedule-parser`** (порт 8000)

**Ответственность:**

- Получить расписание/ссылки на PDF из источника (`SCHEDULE_INDEX_URL`)
- По расписанию (APScheduler) или вручную через API сформировать задачу
- Публиковать задачу в RabbitMQ: `exchange=schedule`, `routing_key=schedule.pdf`

**Технологии:**

- FastAPI, Dishka, APScheduler, FastStream (Rabbit)

#### 3. **`pdf-parser`** (порт 8001)

**Ответственность:**

- Подписаться на задания из RabbitMQ (`schedule.pdf`)
- Скачать PDF по ссылкам и распарсить расписание
- Опубликовать результат в RabbitMQ: `routing_key=schedule.pdf.result`

**Технологии:**

- FastAPI, Dishka, FastStream (Rabbit)
- Внешний экстрактор (DocStrange) по API (ключ в env)

#### 4. **`notification`** (порт 8002)

**Ответственность:**

- Подписаться на готовое расписание из RabbitMQ
- Отправить уведомления пользователям (Telegram)
- Поднять HTTP API и (опционально) Telegram long polling

**Технологии:**

- FastAPI, FastStream (Rabbit), Redis, PostgreSQL, aiogram

---

## 🔹 Сервисы

### `composer` (infra)

Файл: `composer/docker-compose.yml`

**Контейнеры и порты:**

- `postgres` — `5432:5432`
- `redis` — `6379:6379`
- `rabbitmq` — `5672:5672`, `15672:15672`
- `mailhog` — `1025:1025`, `8025:8025`

**UI:**

- RabbitMQ Management: `http://localhost:15672` (guest/guest)
- MailHog UI: `http://localhost:8025`

### `schedule-parser` (8000)

- Папка: `schedule-parser/`
- Docker: `schedule-parser/docker-compose.yml`
- Env: `schedule-parser/env/.env` (есть `.env.example`)

### `pdf-parser` (8001)

- Папка: `pdf-parser/`
- Docker: `pdf-parser/docker-compose.yml`
- Env:
  - `pdf-parser/src/app/env/parser.env`
  - `pdf-parser/src/app/env/rabbitmq.env`

### `notification` (8002)

- Папка: `notification/`
- Docker: `notification/Dockerfile` + entrypoint `notification/src/start.sh`
- Env: прокидывается при запуске контейнера (compose файла в репе нет)

---

## 🔀 Поток данных (E2E)

### 1. Публикация задания на парсинг (schedule-parser → RabbitMQ)

1. `schedule-parser` получает актуальную страницу расписаний (`SCHEDULE_INDEX_URL`)
2. Формирует payload (дата + список PDF URL)
3. Публикует в RabbitMQ:
   - exchange: `schedule`
   - routing key: `schedule.pdf`
   - queue binding: `schedule.pdf`

### 2. Парсинг PDF (pdf-parser)

1. `pdf-parser` получает сообщение из `schedule.pdf`
2. Скачивает PDF
3. Парсит в `ScheduleByDateResponse`
4. Публикует результат:
   - exchange: `schedule`
   - routing key: `schedule.pdf.result`

### 3. Уведомления (notification)

1. `notification` получает готовое расписание из RabbitMQ
2. Находит подписчиков/получателей (Postgres)
3. Отправляет уведомления (Telegram)

---

## ⚡ Запуск

### Предварительные требования

- Docker + Docker Compose v2
- Свободные порты: `5432`, `6379`, `5672`, `15672`, `1025`, `8025`, `8000`, `8001`, `8002`

### 1. Создать Docker-сеть

`composer` использует внешнюю сеть `tasks`:

```bash
docker network create tasks
```

### 2. Поднять инфраструктуру

```bash
cd composer
docker compose up -d
docker compose ps
```

### 3. Поднять `schedule-parser`

```bash
cd ..\schedule-parser
docker compose up --build
```

API: `http://localhost:8000`

### 4. Поднять `pdf-parser`

```bash
cd ..\pdf-parser
docker compose up --build
```

API: `http://localhost:8001`  
Health: `GET /health`

### 5. Поднять `notification`

В репозитории нет `docker-compose.yml` для `notification`, поэтому запуск через `docker run`:

```bash
cd ..\notification
docker build -t bellboy-notification .

docker run --rm -it ^
  --name bellboy-notification ^
  --network tasks ^
  -p 8002:8002 ^
  -e PORT=8002 ^
  -e RABBITMQ_HOST=rabbitmq ^
  -e RABBITMQ_PORT=5672 ^
  -e RABBITMQ_USER=guest ^
  -e RABBITMQ_PASSWORD=guest ^
  -e RABBITMQ_EXCHANGE=schedule ^
  -e RABBITMQ_QUEUE=schedule.pdf.result ^
  -e RABBITMQ_ROUTING_KEY=schedule.pdf.result ^
  -e REDIS_HOST=redis ^
  -e REDIS_PORT=6379 ^
  -e POSTGRES_HOST=postgres ^
  -e POSTGRES_PORT=5432 ^
  -e POSTGRES_DB=notification ^
  -e POSTGRES_USER=postgres ^
  -e POSTGRES_PASSWORD=postgres ^
  -e TELEGRAM_BOT_TOKEN="<PUT_YOUR_TOKEN_HERE>" ^
  bellboy-notification
```

---

## Примеры использования

### Проверить, что RabbitMQ жив

Открыть RabbitMQ UI: `http://localhost:15672` → Queues → убедиться, что появляются `schedule.pdf` / `schedule.pdf.result`.

### Проверить, что MailHog жив

Открыть MailHog UI: `http://localhost:8025`.

### Проверить `pdf-parser`

```bash
curl http://localhost:8001/health
```

---

## 🔧 Технологии

- **Python** (разные версии в Dockerfile’ах)
- **FastAPI** — HTTP API
- **FastStream (Rabbit)** — подписчики/публикаторы RabbitMQ
- **RabbitMQ** — транспорт событий между сервисами
- **PostgreSQL** — данные подписок/состояния (в `notification`)
- **Redis** — rate limiting / pubsub / вспомогательное хранилище (в `notification`)
- **APScheduler** — планировщик в `schedule-parser`
- **Dishka** — DI
- **aiogram** — Telegram bot

---

## 📂 Структура проекта

```text
bellboy/
├── composer/               # Инфраструктура: Postgres/Redis/RabbitMQ/MailHog
│   ├── docker-compose.yml
│   └── README.md           # ← единая документация (этот файл)
│
├── schedule-parser/        # API 8000: поиск расписания + публикация задач
│   ├── docker-compose.yml
│   ├── env/
│   │   ├── .env
│   │   └── .env.example
│   └── src/
│
├── pdf-parser/             # API 8001: consumer schedule.pdf → publish schedule.pdf.result
│   ├── docker-compose.yml
│   ├── Dockerfile
│   └── src/
│
└── notification/           # API 8002: consumer результата → Telegram уведомления
    ├── Dockerfile
    ├── requirements.txt
    └── src/
```

---

## 🏗️ Архитектурные решения

### Event-driven через RabbitMQ (DIRECT exchange)

Сервисная связка построена на RabbitMQ, чтобы:

- `schedule-parser` не зависел напрямую от `pdf-parser`
- `pdf-parser` не зависел напрямую от `notification`
- можно было масштабировать консьюмеры независимо

### Планировщик только в `schedule-parser`

`schedule-parser` сам решает когда формировать “задачу дня” и публиковать её, остальные сервисы просто реагируют на события.

---

## 🔍 Troubleshooting

### `network tasks not found`

```bash
docker network create tasks
```

### Сервисы в Docker не видят RabbitMQ/Postgres по именам

Если сервис запущен в Docker и подключён к сети `tasks`, то host’ы должны быть:

- `rabbitmq`, `postgres`, `redis`, `mailhog`

Если сервис запускается на хосте (вне Docker) — используйте `localhost`.

### Очереди не появляются / notification ничего не получает

Проверь:

- `pdf-parser` публикует в `routing_key=schedule.pdf.result`
- `notification` подписан на **тот же** `RABBITMQ_QUEUE` и `RABBITMQ_ROUTING_KEY`

---

## ⚙️ Полезные команды

```bash
# Инфра
cd composer
docker compose ps
docker compose logs -f

# Логи конкретных контейнеров
docker logs rabbitmq --tail 200
docker logs postgres --tail 200
docker logs redis --tail 200
docker logs mailhog --tail 200
```

---

## 📈 Метрики

В текущем составе репозитория отдельный мониторинг (Prometheus/Grafana) не описан в `composer/docker-compose.yml`.  
Если добавим метрики — этот раздел расширим под конкретные endpoint’ы (`/metrics`) и дашборды.

---

## 📄 Лицензия

Не задана.

## 👨‍💻 Контрибьюторы

Внутренний проект/учебный проект команды.

