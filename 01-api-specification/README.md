# Кейс 1: Спецификация REST API и межсервисная интеграция

## 📌 Бизнес-контекст
Сервис электронного документооборота банковской платформы. Смежным системам (CRM, фронтенд кредитного конвейера) требовался единый метод для первичной валидации документов, сохранения метаданных и постановки задач в очередь на факсимильное подписание.

## 🎯 Проблема
* Разработчики тратили до 5 часов в неделю на консультации смежных команд по структуре запросов.
* Документация в Confluence не отражала реальные коды ошибок, возникали сбои из-за превышения лимитов размера payload (Base64).

## 🛠 Что сделано
1. **Разработана спецификация [openapi.yaml](./openapi.yaml)** по стандарту OpenAPI 3.0.
2. **Внедрен стандарт RFC 7807 (Problem Details)** для унификации сообщений об ошибках (`400`, `401`, `422`).
3. **Задокументирован механизм идемпотентности** с использованием заголовка `X-Idempotency-Key` (UUIDv4) для предотвращения дублирования договоров при сетевых таймаутах (`504 Gateway Timeout`).
4. **Спроектирована Sequence-диаграмма** взаимодействия сервисов при обработке запроса.

---

## 🔄 Архитектура вызова (Sequence Diagram)

```mermaid
sequenceDiagram
    autonumber
    actor Client as Клиент / CRM
    participant GW as API Gateway
    participant DocSvc as Document Service
    participant DB as PostgreSQL
    participant SignSvc as Внешний сервис подписания

    Client->>GW: POST /documents/process (с X-Idempotency-Key)
    activate GW
    
    GW->>GW: Валидация JWT токена
    alt Невалидный токен
        GW-->>Client: 401 Unauthorized (RFC 7807)
    end

    GW->>DocSvc: Проксирование запроса
    activate DocSvc

    DocSvc->>DB: Проверка X-Idempotency-Key
    alt Запрос с таким ключом уже выполнялся
        DB-->>DocSvc: Данные существующего документа
        DocSvc-->>Client: 200 OK (Кэшированный ответ)
    end

    DocSvc->>DocSvc: Валидация payload (Base64, лимит 15МБ)
    alt Ошибка валидации
        DocSvc-->>Client: 422 Unprocessable Entity
    end

    DocSvc->>DB: Запись в БД (Статус: PENDING)
    DocSvc->>SignSvc: POST /v1/signatures (Внешний вызов)
    SignSvc-->>DocSvc: 202 Accepted (ID заявки)
    
    DocSvc->>DB: Обновление статуса (PROCESSING)
    DocSvc-->>Client: 201 Created (documentId, Location header)
    
    deactivate DocSvc
    deactivate GW
```

---

## 💡 Ключевые решения в контракте
* **Разделение валидации:** Техническая ошибка парсинга структуры возвращает `400 Bad Request`, а превышение лимита файла или неизвестный `docType` — `422 Unprocessable Entity` с массивом проблемных полей `invalidParams`.
* **Защита от сетевых сбоев:** Клиенту гарантируется безопасность повторного вызова с тем же UUID при получении `5xx` ошибок от сети.
