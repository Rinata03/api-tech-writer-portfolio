# Кейс 2: Руководство по интеграции (Quick Start) и Postman-коллекция

## 📌 О кейсе
Когда сторонние команды или внешние B2B-партнеры подключаются к нашему API, им нужен не только голый Swagger, но и понятный сценарий первого запуска («Quick Start») без лишней теории.

В рамках проекта были созданы:
1. Интеграционный гайд с примерами рабочих запросов (cURL) и маппингом ошибок.
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
```

---

## 🧪 Архитектура коллекции Postman

Для команд интеграции подготовлена готовая коллекция с настроенной автоматизацией.

### 1. Переменные окружения (Environments)
Коллекция параметризована и переключается между окружениями в 1 клик:
* `{{baseUrl}}` — хост окружения (`dev`, `stage`, `prod`).
* `{{jwtToken}}` — токен авторизации.
* `{{lastCreatedDocId}}` — глобальная переменная, куда сохраняется ID созданного документа для использования в следующих вызовах.

### 2. Pre-request Script (Автогенерация ключа идемпотентности)
Разработчикам не нужно генерировать UUID вручную — скрипт генерирует его автоматически перед отправкой запроса:

```javascript
// Автоматическая генерация UUIDv4 для заголовка идемпотентности
pm.request.headers.add({
    key: 'X-Idempotency-Key',
    value: pm.variables.replaceIn('{{$guid}}')
});
```

### 3. Tests (Автоматические проверки контракта)
Каждый запрос в коллекции валидирует не только код ответа, но и структуру возвращаемого объекта:

```javascript
// Проверка успешного создания ресурса
pm.test("Status code is 201 Created", function () {
    pm.response.to.have.status(201);
});

// Проверка наличия обязательных полей
pm.test("Response contains documentId and status", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData).to.have.property('documentId');
    pm.expect(jsonData.status).to.eql("PROCESSING");
    
    // Сохраняем ID для следующих тестов (проверки статуса)
    pm.environment.set("lastCreatedDocId", jsonData.documentId);
});
```

---

## 📈 Результаты для бизнеса и команды
* **Онбординг:** Время первой успешной отправки тестового запроса новой командой сократилось с 3 дней до **40 минут**.
* **Качество:** Использование Postman-тестов перед релизами исключило появление регрессионных ошибок при изменении схем ответов.
