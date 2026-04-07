# Calculator Infrastructure

Инфраструктура для запуска микросервисного калькулятора.

## Архитектура

Проект состоит из двух микросервисов и вспомогательных сервисов:

| Сервис | Описание | Порт |
|--------|----------|------|
| **calculator-api** | HTTP Gateway. Принимает REST запросы, публикует в NATS, вызывает gRPC | 8080 |
| **calculator-storage** | Worker + gRPC сервер. Обрабатывает сообщения из NATS, хранит данные в PostgreSQL | 50051 |
| **NATS JetStream** | Message broker с гарантированной доставкой | 4222 (client), 8222 (monitor) |
| **PostgreSQL** | База данных для хранения вычислений | 5433 (external) |

### Взаимодействие сервисов

POST /calculate → calculator-api → NATS → calculator-storage → PostgreSQL\
GET /calculations → calculator-api → gRPC → calculator-storage → PostgreSQL


## Быстрый старт

### 1. Клонирование репозиториев
```bash
# Создайте папку проекта
mkdir calculator-project
cd calculator-project

# Клонируйте все репозитории
git clone https://github.com/AnatolyKoltun/calculator-infrastructure.git
git clone https://github.com/AnatolyKoltun/calculator-api.git
git clone https://github.com/AnatolyKoltun/calculator-storage.git
```
### 2. Настройка окружения
```bash
cd calculator-infrastructure
cp .env.example .env
```
### 3. Запуск
```bash
docker-compose up -d --build
```
### 4. Проверка работы
```bash
# Проверить статус контейнеров
docker-compose ps

# Отправить выражение на вычисление
POST:

$body = @{
    argument1 = 332
    argument2 = 5
    operator = "/"
} | ConvertTo-Json

Invoke-WebRequest -Uri "http://localhost:8080/calculate" `
    -Method POST `
    -Headers @{"Content-Type"="application/json"} `
    -Body $body

# Получить список вычислений
GET:

Invoke-WebRequest -Uri "http://localhost:8080/calculations" -Method GET         
Invoke-WebRequest -Uri "http://localhost:8080/calculations?date_from=2026-03-19" -Method GET
```
### 5. Переменные окружения (.env)

| Переменная | Описание | Значение по умолчанию |
|------------|----------|----------------------|
| `DB_USER` | Пользователь PostgreSQL | postgres |
| `DB_PASSWORD` | Пароль PostgreSQL | 258456 |
| `DB_NAME` | Имя базы данных | calculator |
| `DB_HOST` | Хост PostgreSQL (внутренний) | postgres |
| `DB_PORT` | Порт PostgreSQL (внутренний) | 5432 |
| `POSTGRES_EXTERNAL_PORT` | Внешний порт PostgreSQL | 5433 |
| `NATS_CLIENT_PORT` | Порт для клиентов NATS | 4222 |
| `NATS_MONITOR_PORT` | Порт веб-интерфейса NATS | 8222 |
| `NATS_INTERNAL_URL` | Внутренний URL NATS | nats://nats:4222 |
| `API_PORT` | Порт HTTP API | 8080 |
| `STORAGE_GRPC_PORT` | Порт gRPC сервера | 50051 |
| `STORAGE_SERVICE_URL` | Внутренний URL storage | calculator-storage:50051 |

### 6. Управление сервисами
```bash
# Запустить все сервисы
docker-compose up -d

# Пересобрать и запустить
docker-compose up -d --build

# Остановить все сервисы
docker-compose down

# Остановить и удалить тома (очистить БД)
docker-compose down -v

# Посмотреть логи всех сервисов
docker-compose logs -f

# Посмотреть логи конкретного сервиса
docker-compose logs -f calculator-api
docker-compose logs -f calculator-storage

# Пересборка отдельных сервисов
# Только API
docker-compose build calculator-api
docker-compose up -d calculator-api

# Только Storage
docker-compose build calculator-storage
docker-compose up -d calculator-storage

# Подключаемся к PostgreSQL
docker exec -it calculator-infrastructure-postgres-1 psql -U postgres -d calculator

-- Проверяем, есть ли таблицы
\dt

-- Создаём таблицу
CREATE TABLE IF NOT EXISTS calculations (
    id SERIAL PRIMARY KEY,
    argument1 DECIMAL,
    argument2 DECIMAL,
    operator VARCHAR(10),
    result DECIMAL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Проверяем, что таблица создалась
\dt

-- Выход
\q
```
### 7. Мониторинг
NATS веб-интерфейс	http://localhost:8222\
Healthcheck PostgreSQL	docker-compose logs postgres
### 7. Структура проекта
```bash
calculator-infrastructure/
├── docker-compose.yml      # Оркестрация сервисов
├── .env                    # Переменные окружения
├── .env.example            # Пример переменных
└── README.md               # Документация

calculator-api/             # HTTP Gateway (отдельный репозиторий)
calculator-storage/         # Worker + gRPC (отдельный репозиторий)
```
