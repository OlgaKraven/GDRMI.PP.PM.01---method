# 2. Практика. 4. Реализация backend’а ИС сопровождения игрового продукта 

## 4.1. Задание к разделу 4 «Backend-логика»

### Необходимо выполнить

1. Реализовать middleware:

   * `requestId`
   * `authJwt`
   * `validate`
   * `errorHandler`

2. Реализовать **транзакционную операцию** для “итогового события” (например, `match_finish`):

   * запись события в `game_events`
   * обновление `player_progress`
   * upsert в `leaderboard_scores`
   * обновление `statistics_daily`
   * всё в **одной транзакции**

### Артефакт

Код backend (структура папок обязательна):

```text
app/
  routes/
  controllers/
  services/
  repositories/
  middleware/
  models/        (опционально, если используете ORM)
run.py
config.py
requirements.txt
```

---

## 4.2. Минимальная рабочая архитектура backend (обязательная логика)

Поток обработки запроса должен соответствовать конвейеру:

```text
Request
  → requestId + logging
  → authJwt
  → validate
  → controller
  → service (business + transaction)
  → repository (SQL)
  → response (JSON)
Error → errorHandler → JSON + logging
```

---

## 4.3. Минимальные endpoints для раздела 4 (рекомендуемый набор)

Для сдачи раздела достаточно реализовать один “итоговый” endpoint:

* `POST /match/finish` **или**
* `POST /event` с `eventType = match_finish`

---

# 4.4. Реализация проекта (Flask) — структура и файлы

## 4.4.1. Минимальная структура проекта (Flask)

```text
project-root/
│
├── app/
│   ├── __init__.py
│   ├── extensions.py
│   │
│   ├── routes/
│   │   └── match.py
│   │
│   ├── controllers/
│   │   └── match_controller.py
│   │
│   ├── services/
│   │   └── match_service.py
│   │
│   ├── repositories/
│   │   ├── events_repo.py
│   │   ├── progress_repo.py
│   │   ├── leaderboard_repo.py
│   │   └── stats_repo.py
│   │
│   └── middleware/
│       ├── request_id.py
│       ├── auth_jwt.py
│       ├── validate.py
│       └── error_handler.py
│
├── config.py
├── run.py
└── requirements.txt
```

---

# 4.5. Middleware (обязательные)

## 4.5.1. `requestId` middleware

Требования:

* генерирует `requestId` на каждый запрос
* добавляет в ответ заголовок: `X-Request-Id`
* логирует `requestId`, метод, путь, статус, duration

Минимальный формат лога:

```text
req_abc123 user=15 POST /match/finish 200 18ms
```

---

## 4.5.2. `authJwt` middleware

Требования:

* читает `Authorization: Bearer <JWT>`
* валидирует токен
* помещает в `g.user` или `request.user`:

  * `userId`
  * `roles`

Ошибки:

* `401 Unauthorized` — токен отсутствует/невалиден/истёк
* `403 Forbidden` — роль не подходит (если проверяете RBAC)

---

## 4.5.3. `validate` middleware

Требования:

* проверяет тело запроса до БД
* при ошибке возвращает `400` в едином формате

Минимальные проверки для `match_finish`:

* `matchId` — число
* `result.isWin` — boolean
* `result.score` — число >= 0
* `result.durationSeconds` — число >= 0
* `result.waveReached` — число >= 0

---

## 4.5.4. `errorHandler` middleware

Требования:

* перехватывает любые исключения
* возвращает единый формат ошибки
* логирует ошибку вместе с `requestId` и `userId` (если есть)

Единый формат ошибок:

```json
{
  "ok": false,
  "error": {
    "code": "INTERNAL_ERROR",
    "message": "Unexpected error",
    "requestId": "req_abc123"
  }
}
```

---

# 4.6. Транзакционная операция: `match_finish` (обязательная)

## 4.6.1. Что должно происходить внутри одной транзакции

Внутри одной TX выполняются:

1. запись события в `game_events` (`event_type='match_finish'`)
2. обновление `player_progress` (начисления)
3. обновление `statistics_daily`
4. upsert в `leaderboard_scores`
5. commit

---

## 4.6.2. Пример входного запроса (Tower Defense)

```http
POST /match/finish
Authorization: Bearer <JWT>
Content-Type: application/json
```

```json
{
  "matchId": 987654321,
  "result": {
    "isWin": true,
    "score": 12450,
    "durationSeconds": 410,
    "waveReached": 12
  }
}
```

---

## 4.6.3. Минимальная бизнес-логика (пример начислений)

Награды рассчитываются сервером:

```text
xp = 50 + waveReached * 10
softCurrency = 100 + waveReached * 20
```

Для `waveReached = 12`:

```text
xp = 170
softCurrency = 340
```

---

## 4.6.4. SQL внутри транзакции (MySQL 8.0)

Начало транзакции:

```sql
START TRANSACTION;
```

### 1) Запись события в `game_events`

```sql
INSERT INTO game_events (user_id, session_id, event_type, payload_json)
VALUES (
  ?, 
  NULL,
  'match_finish',
  JSON_OBJECT(
    'matchId', ?,
    'result', JSON_OBJECT(
      'isWin', ?,
      'score', ?,
      'durationSeconds', ?,
      'waveReached', ?
    )
  )
);
```

### 2) Обновление прогресса

```sql
UPDATE player_progress
SET xp = xp + ?,
    soft_currency = soft_currency + ?
WHERE user_id = ?;
```

### 3) Обновление дневной статистики (upsert)

```sql
INSERT INTO statistics_daily (user_id, day, sessions_count, events_count, playtime_seconds, wins, losses, score_sum)
VALUES (?, CURDATE(), 0, 1, ?, ?, ?, ?)
ON DUPLICATE KEY UPDATE
  events_count = events_count + 1,
  playtime_seconds = playtime_seconds + VALUES(playtime_seconds),
  wins = wins + VALUES(wins),
  losses = losses + VALUES(losses),
  score_sum = score_sum + VALUES(score_sum);
```

Где:

* `playtime_seconds` = `durationSeconds`
* `wins` = 1 если `isWin=true`, иначе 0
* `losses` = 1 если `isWin=false`, иначе 0
* `score_sum` = `score`

### 4) Upsert лидерборда

```sql
INSERT INTO leaderboard_scores (user_id, board_code, season, score)
VALUES (?, 'td_score', 1, ?)
ON DUPLICATE KEY UPDATE
  score = GREATEST(score, VALUES(score)),
  updated_at = CURRENT_TIMESTAMP(3);
```

Завершение транзакции:

```sql
COMMIT;
```

---

## 4.6.5. Ответ сервера (единый формат)

```json
{
  "ok": true,
  "data": {
    "matchId": 987654321,
    "rewards": {
      "xp": 170,
      "softCurrency": 340
    },
    "leaderboard": {
      "boardCode": "td_score",
      "season": 1,
      "score": 12450
    }
  }
}
```
 