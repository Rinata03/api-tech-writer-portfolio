# Кейс 2: Руководство по интеграции (Quick Start) и Postman-коллекция

## 📌 О кейсе

Когда сторонние команды или внешние B2B-партнеры подключаются к нашему API, им нужен не только голый Swagger, но и понятный сценарий первого запуска («Quick Start») без лишней теории.

В рамках проекта были созданы:
1. Интеграционный гайд с примерами рабочих запросов (cURL), политикой отказоустойчивости и маппингом ошибок.
2. Готовая коллекция Postman с предустановленными переменными окружения, генерацией заголовков на лету и автотестами ответов.

---

## 🚀 Quick Start Guide: Документооборот API

Перед началом интеграции убедитесь, что вам выдан `Client-ID` и закрытый ключ для тестового контура (**Stage**).

### 1. Базовые параметры

* **Base URL (Stage):** `https://api-stage.bank.internal/v1`
* **Формат данных:** `application/json; charset=utf-8`
* **Авторизация:** Заголовок `Authorization: Bearer <JWT_TOKEN>`

### 2. Обязательные заголовки

| Заголовок | Тип | Описание |
| :--- | :--- | :--- |
| `Authorization` | String | Bearer-токен сервисной учетной записи |
| `X-Idempotency-Key` | UUIDv4 | Уникальный ключ запроса для исключения дублирования |
| `Content-Type` | String | Всегда `application/json` |

---

### 3. Пример запроса (cURL)

```bash
curl --location 'https://api-stage.bank.internal/v1/documents/process' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsIn...' \
--header 'X-Idempotency-Key: c9b2c8a1-6e3e-4b77-8f55-123456789abc' \
--data '{
    "externalId": "DOC-REQ-77881",
    "docType": "LOAN_AGREEMENT",
    "payloadBase64": "JVBERi0xLjQKJcOkw7zDtsOf..."
}'
