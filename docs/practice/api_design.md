# 2. Практика. 5. API дизайн ИС сопровождения игрового продукта 

### 5.0. Смысл раздела

API проектируется так, чтобы Unity/Web/Mobile могли работать с ним **предсказуемо** и **одинаково**:

* stateless REST (без серверных сессий);
* безопасность: JWT + RBAC + валидация;
* единый контракт ответов и ошибок;
* удобство тестирования (Postman / Swagger).

---

## 5.1. Задание к разделу 5 «API дизайн»

### Необходимо выполнить

1. Реализовать MVP endpoints (минимум 5):

   * `POST /api/v1/auth/register`
   * `POST /api/v1/auth/login`
   * `GET /api/v1/profile`
   * `POST /api/v1/events`
   * `GET /api/v1/leaderboard`

   Опционально (если успеваете):

   * `POST /api/v1/progress`
   * `GET /api/v1/stats`

2. Реализовать **единый контракт ответов** для всех endpoint’ов:

   * успех: `{ "ok": true, "data": ... }`
   * ошибка: `{ "ok": false, "error": { "code", "message", "requestId" } }`

### Артефакт

* Работающий API (запускается локально и отвечает на запросы).
* Для сдачи достаточно Postman-проверки или Swagger UI (если подключён).

---

## 5.2. Обязательные правила проектирования REST

### 5.2.1. Ресурсный подход

* плохо: `/doLogin`, `/sendEventCommand`
* хорошо: `/auth/login`, `/events`, `/profile`

Допустимое “действие” в играх — только как подресурс:

* `POST /matches/{id}/finish`
* `POST /matches/{id}/events`

---

### 5.2.2. Stateless

Каждый запрос несёт всё необходимое:

* `Authorization: Bearer <JWT>`
* JSON body / query params

Состояние игрока хранится в БД (`player_progress`, `matches`, `leaderboard_scores`), а не в памяти процесса.

---

### 5.2.3. Единый JSON-контракт

Успех:

```json
{ "ok": true, "data": {} }
```

Ошибка:

```json
{
  "ok": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Field level must be >= 1",
    "requestId": "req_abc123"
  }
}
```

---

## 5.3. HTTP коды (минимальный обязательный набор)

* `200 OK` — успешно
* `201 Created` — создан ресурс
* `400 Bad Request` — ошибки валидации/формата
* `401 Unauthorized` — нет/невалиден JWT
* `403 Forbidden` — нет прав / пользователь забанен
* `404 Not Found` — ресурс не найден
* `409 Conflict` — конфликт состояния/уникальности
* `429 Too Many Requests` — rate limit
* `500 Internal Server Error` — непредвиденная ошибка

---

# 5.4. MVP endpoints — спецификация + примеры

## 5.4.1. `POST /api/v1/auth/register`

**Назначение:** создать аккаунт, назначить роль `player`.

**Request**

```http
POST /api/v1/auth/register
Content-Type: application/json
```

```json
{
  "email": "player1@example.com",
  "password": "P@ssw0rd!",
  "nickname": "PlayerOne"
}
```

**Response (201)**

```json
{
  "ok": true,
  "data": { "userId": 101 }
}
```

Ошибки:

* `400 VALIDATION_ERROR`
* `409 EMAIL_ALREADY_EXISTS`

---

## 5.4.2. `POST /api/v1/auth/login`

**Назначение:** проверить пароль и выдать JWT.

**Request**

```http
POST /api/v1/auth/login
Content-Type: application/json
```

```json
{
  "email": "player1@example.com",
  "password": "P@ssw0rd!"
}
```

**Response (200)**

```json
{
  "ok": true,
  "data": {
    "accessToken": "<JWT>",
    "tokenType": "Bearer",
    "expiresInSeconds": 3600
  }
}
```

Ошибки:

* `401 INVALID_CREDENTIALS`
* `403 USER_BANNED`

---

## 5.4.3. `GET /api/v1/profile`

**Назначение:** отдать “витрину” профиля: user + progress.

**Request**

```http
GET /api/v1/profile
Authorization: Bearer <JWT>
```

**Response (200)**

```json
{
  "ok": true,
  "data": {
    "user": { "id": 101, "email": "player1@example.com", "nickname": "PlayerOne" },
    "progress": { "level": 6, "xp": 2450, "softCurrency": 1200, "hardCurrency": 10 }
  }
}
```

Ошибки:

* `401 UNAUTHORIZED`
* `403 USER_BANNED`

---

## 5.4.4. `POST /api/v1/events`

**Назначение:** записать событие (и по необходимости запустить обработку).

**Request**

```http
POST /api/v1/events
Authorization: Bearer <JWT>
Content-Type: application/json
```

```json
{
  "eventType": "match_finish",
  "sessionId": 555,
  "clientTime": "2026-02-26T12:20:10.000Z",
  "payload": {
    "mode": "td",
    "mapCode": "forest_01",
    "season": 1,
    "isWin": true,
    "score": 12450,
    "durationSeconds": 410,
    "waveReached": 12
  }
}
```

**Response (200)**

```json
{
  "ok": true,
  "data": { "accepted": true }
}
```

Ошибки:

* `400 VALIDATION_ERROR`
* `429 RATE_LIMITED`
* `409 EVENT_REJECTED`

---

## 5.4.5. `GET /api/v1/leaderboard`

**Request**

```http
GET /api/v1/leaderboard?boardCode=td_score&season=1&limit=100
Authorization: Bearer <JWT>
```

**Response (200)**

```json
{
  "ok": true,
  "data": {
    "boardCode": "td_score",
    "season": 1,
    "items": [
      { "rank": 1, "userId": 10, "nickname": "Max", "score": 50000 },
      { "rank": 2, "userId": 15, "nickname": "Lina", "score": 45000 }
    ]
  }
}
```

Ошибки:

* `400 VALIDATION_ERROR` (невалидные query-параметры)
* `401 UNAUTHORIZED`

---

# 5.5. Swagger / OpenAPI (если подключаете)

## 5.5.1. Что должно быть описано

* все MVP endpoints
* схемы запросов/ответов (включая `ErrorResponse`)
* security scheme JWT bearer
* ответы `401/403/409/429` там, где они возможны

Минимальный фрагмент:

```yaml
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
```

---
