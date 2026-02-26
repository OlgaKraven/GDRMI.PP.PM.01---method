# 1. Теория. 7. Тестирование интеграции (клиент ↔ REST API ↔ MySQL)

## 7.1. Уровни тестирования (что именно проверяем)

### 7.1.1. API-level (Postman / curl)

Проверяем контракт HTTP/JSON без клиента:

* статусы (200/201/400/401/403/409/429/500)
* формат `ok/data` и `ok/error`
* корректность JWT

### 7.1.2. Client-level (Unity / Web)

Проверяем:

* добавляется ли `Authorization: Bearer`
* корректно ли сериализуется JSON
* корректно ли обрабатываются ошибки/таймауты
* работает ли очередь событий и backoff

### 7.1.3. DB-level (MySQL)

Проверяем:

* записи появились в нужных таблицах
* FK/UNIQUE/CHECK не нарушены
* нет дублей событий при повторе запросов

---

## 7.2. Базовый набор тест-данных (чтобы тесты были воспроизводимы)

В отчёте студент фиксирует:

* тестовый пользователь: `player1@example.com`
* пароль: `P@ssw0rd!` (только для стенда)
* boardCode: `td_score`, season: `1`
* sessionId: `555`
* eventId: фиксированный UUID для дедуп-теста

Почему фиксируем: иначе студент “каждый раз тестирует по-разному”, и это невозможно проверять преподавателю.

---

## 7.3. Позитивные сценарии (обязательные)

Ниже — минимальный чек-лист. Студент должен выполнить **в таком порядке**, чтобы видно было зависимость шагов.

### P1. Регистрация → пользователь создан

**Шаги**

1. `POST /auth/register`
2. убедиться `201` и `ok=true`

**Ожидание**

* `users.email` создан
* назначена роль `player` (в `user_roles`)

**DB-проверка**

```sql
SELECT id, email, created_at, is_banned FROM users WHERE email='player1@example.com';
SELECT ur.user_id, r.name FROM user_roles ur JOIN roles r ON r.id=ur.role_id
WHERE ur.user_id = (SELECT id FROM users WHERE email='player1@example.com');
```

---

### P2. Логин → выдан JWT

**Шаги**

1. `POST /auth/login`
2. `200`, `accessToken` получен

**Ожидание**

* `last_login_at` обновился
* JWT работает в защищённых методах

**DB-проверка**

```sql
SELECT last_login_at FROM users WHERE email='player1@example.com';
```

---

### P3. Профиль → витрина (users + player_progress)

**Шаги**

1. `GET /profile` с Bearer токеном
2. `200`, `ok=true`, есть `progress`

**Ожидание**

* один запрос возвращает всё для UI профиля
* значения прогресса правдоподобны (level>=1, currency>=0)

---

### P4. Отправка события session_start → запись в events

**Шаги**

1. `POST /events` (`eventType=session_start`, `eventId=UUID`)
2. `200`, `accepted=true`

**Ожидание**

* событие есть в `game_events`
* `user_id` в событии соответствует JWT, а не body

**DB-проверка**

```sql
SELECT event_type, created_at
FROM game_events
WHERE user_id = (SELECT id FROM users WHERE email='player1@example.com')
ORDER BY created_at DESC
LIMIT 5;
```

---

### P5. Лидерборд читается и сортируется

**Шаги**

1. `GET /leaderboard?boardCode=td_score&season=1&limit=10`
2. `200`, items отсортированы

**Ожидание**

* сортировка по score DESC
* rank формирует сервер

---

## 7.4. Негативные сценарии (обязательные)

### N1. Невалидный JSON (валидация) → 400 VALIDATION_ERROR

**Пример**

* отправить `POST /progress` с `level=0` или `softCurrency=-1`

**Ожидание**

* `400`
* `ok=false`, `error.code=VALIDATION_ERROR`
* есть `requestId`

**Почему проверяем**
это показывает, что студент умеет строить API с “защитой от мусора”, а не только happy-path.

---

### N2. Нет токена → 401 UNAUTHORIZED

**Шаг**

* `GET /profile` без Authorization

**Ожидание**

* `401`
* единый JSON ошибки

---

### N3. Просроченный/битый токен → 401 UNAUTHORIZED

**Шаг**

* подменить 1 символ в JWT и вызвать `GET /profile`

**Ожидание**

* `401`
* клиент должен инициировать logout

---

### N4. Забаненный пользователь → 403 USER_BANNED

**Подготовка**

```sql
UPDATE users SET is_banned=1 WHERE email='player1@example.com';
```

**Шаг**

* `GET /profile` с валидным JWT

**Ожидание**

* `403` (политика запрета)
* клиент показывает “доступ запрещён”, останавливает очередь событий

---

### N5. Conflict (состояние) → 409

**Пример**

* повторно завершить матч / повторно закрыть сессию / конфликт optimistic lock

**Ожидание**

* `409`
* `error.code` типа `EVENT_REJECTED` / `MATCH_ALREADY_FINISHED` / `PROGRESS_CONFLICT`

---

## 7.5. Проверка авторизации (JWT + RBAC)

### A1. Роль player не может вызывать admin endpoint

**Шаг**

* вызвать условный `/admin/ban` с токеном игрока

**Ожидание**

* `403 FORBIDDEN`

### A2. Роль moderator может

**Подготовка**

```sql
INSERT IGNORE INTO roles(name) VALUES ('moderator');
INSERT INTO user_roles(user_id, role_id)
SELECT u.id, r.id
FROM users u, roles r
WHERE u.email='player1@example.com' AND r.name='moderator';
```

**Шаг**

* перелогиниться (JWT должен включать роль)
* вызвать endpoint

**Ожидание**

* доступ разрешён

**Почему важно**
RBAC нельзя “проверять на клиенте”, только на сервере.

---

## 7.6. Проверка 429 Too Many Requests (rate limit) + backoff на клиенте

### R1. Нагрузить /events или /login

**Шаг**

* отправить 30–100 запросов `/events` подряд за короткое время (скриптом/Postman Runner)

**Ожидание**

* начиная с некоторого момента сервер отвечает `429 RATE_LIMITED`

### R2. Поведение клиента

**Ожидание на клиенте**

* не продолжать спам
* применять backoff (2s → 4s → 8s)
* события не теряются (остаются в очереди)

**Критерий зачёта**
видео/лог, где видно: пришёл `429`, клиент подождал, затем продолжил.

---

## 7.7. Дедупликация событий по eventId (критичный тест)

### D1. Два одинаковых запроса с одним eventId

**Шаг**

* отправить `POST /events` два раза с **одинаковым** `eventId`

**Ожидание (правильное)**

* в БД ровно **одна** запись (или вторая помечена как duplicate без побочных эффектов)
* ответ может быть `200 accepted=true` оба раза, но сервер не дублирует данные

**DB-проверка (если вы храните event_id)**

```sql
SELECT COUNT(*) 
FROM game_events
WHERE payload_json->>'$.eventId' = 'c2a4f7be-0f0c-4e69-9f20-93f8c4f50a4d';
```

Если eventId храните отдельной колонкой — ещё лучше:

```sql
SELECT COUNT(*) FROM game_events WHERE event_id='...';
```

**Почему это обязательный тест**
иначе любая нестабильная сеть “накрутит” валюту/очки/прогресс.

---

## 7.8. Проверка целостности данных (FK/UNIQUE/CHECK)

### I1. FK: нельзя записать событие для несуществующего user_id

**Ожидание**

* server layer должен брать userId из JWT → такой тест “невозможен снаружи”
* но можно проверить, что вы **не принимаете** userId из body

**Как проверить**

* отправьте `POST /events` с `payload.userId=999999` → в БД всё равно должен быть userId из JWT.

---

### I2. UNIQUE: нельзя создать два users с одним email

**Шаг**

* два раза `POST /register` с одним email

**Ожидание**

* первый раз `201`, второй раз `409 EMAIL_ALREADY_EXISTS`

---

### I3. CHECK: нельзя level<1, currency<0

**Шаг**

* `POST /progress` с отрицательными значениями

**Ожидание**

* сервер вернёт 400 (валидация до БД)
* в БД не появится “абсурд”

---

## 7.9. Базовое нагрузочное тестирование

### 7.9.1. Цель нагрузки

Не “сломать сервер”, а проверить:

* стабильность контрактов
* деградацию времени ответа
* отсутствие дублей/гонок

### 7.9.2. Минимальный сценарий

* 1000 событий `POST /events` (например, `match_event`)
* параллельность 5–20 (в зависимости от стенда)

**Метрики, которые фиксирует студент**

* p50/p95 latency
* процент 5xx
* процент 429
* число записей в БД (совпадает с числом принятых событий)

---
