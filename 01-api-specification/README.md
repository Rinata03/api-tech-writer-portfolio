> 🚀 **[Интерактивная спецификация в Swagger UI (Live Demo)](https://rinata03.github.io/api-tech-writer-portfolio/)**

---

# Кейс 1: Спецификация REST API и межсервисная интеграция

## 📌 Бизнес-контекст
Сервис электронного документооборота банковской платформы. Смежным системам (CRM, фронтенд кредитного конвейера) требовался единый метод для первичной валидации документов, сохранения метаданных и постановки задач в очередь на факсимильное подписание.

## 🎯 Проблема
* Разработчики тратили до 5 часов в неделю на консультации смежных команд по структуре запросов.
* Документация в Confluence не отражала реальные коды ошибок, возникали сбои из-за превышения лимитов размера payload (Base64).

## 🛠 Что сделано
1. **Разработана спецификация [openapi.yaml](./openapi.yaml)** по стандарту OpenAPI 3.0 с полной типизацией и примерами.
2. **Внедрен стандарт RFC 7807 (Problem Details)** для унификации сообщений об ошибках (`400`, `401`, `422`, `429`, `504`).
3. **Задокументирован механизм идемпотентности** с использованием заголовка `X-Idempotency-Key` (UUIDv4) для защиты от повторных списаний/созданий при сетевых сбоях.
4. **Спроектирована Sequence-диаграмма** взаимодействия микросервисов с обработкой исключений и таймаутов.

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
    
    GW->>GW: Проверка JWT токена и Rate Limit
    alt Невалидный токен или превышен лимит вызовов
        GW-->>Client: 401 Unauthorized / 429 Too Many Requests (RFC 7807)
    else Авторизация успешна
        GW->>DocSvc: Проксирование запроса
        activate DocSvc

        DocSvc->>DB: Поиск X-Idempotency-Key
        alt Ключ найден в кэше (Повторный запрос)
            DB-->>DocSvc: Данные созданного документа
            DocSvc-->>Client: 201 Created (X-Cache-Lookup: HIT)
        else Новый запрос
            DocSvc->>DocSvc: Валидация payload (PDF, Base64, до 15МБ)
            alt Ошибка бизнес-валидации схемы
                DocSvc-->>Client: 422 Unprocessable Entity (RFC 7807)
            else Валидация успешна
                DocSvc->>DB: Запись в БД (Статус: PROCESSING)
                
                DocSvc->>SignSvc: POST /v1/signatures (Внешний вызов)
                activate SignSvc
                
                alt Сбой внешнего сервиса / Таймаут 5 сек
                    SignSvc--x DocSvc: Timeout / Connection Drop
                    DocSvc->>DB: Обновление статуса (FAILED)
                    DocSvc-->>GW: 504 Gateway Timeout
                    GW-->>Client: 504 Gateway Timeout (RFC 7807, Retryable)
                else Успешная постановка в очередь
                    SignSvc-->>DocSvc: 202 Accepted (ID заявки на подпись)
                    deactivate SignSvc
                    DocSvc->>DB: Фиксация Sign ID
                    DocSvc-->>GW: 201 Created (documentId, Location header)
                    GW-->>Client: 201 Created (documentId, Location header)
                end
            end
        end
        deactivate DocSvc
    end
    deactivate GW
```

---

## 💡 Ключевые решения в контракте
* **Разделение валидации:** Техническая ошибка парсинга структуры возвращает `400 Bad Request`, а превышение лимита файла или неизвестный `docType` — `422 Unprocessable Entity` с массивом проблемных полей `invalidParams`.
* **Отказоустойчивость:** При сбое внешнего сервиса подписания (`504 Gateway Timeout`) клиенту гарантируется сохранность состояния и возможность безопасного повтора запроса с тем же `X-Idempotency-Key`.
