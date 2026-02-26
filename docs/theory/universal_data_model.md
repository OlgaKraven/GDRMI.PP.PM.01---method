# 1. Теория. 2. Универсальная модель данных (адаптируемая под жанр)

Универсальная модель данных — это **ядро** ИС сопровождения игрового продукта. Она отвечает на 5 ключевых вопросов:

1. **Кто** играет? (Users, Roles)
2. **Когда и откуда** он играет? (GameSessions)
3. **Что** он делает? (GameEvents)
4. **Какой итог** это даёт “прямо сейчас”? (PlayerProgress)
5. **Как это превращается** в аналитику и сравнение? (Statistics, Leaderboard)

Почему модель жанронезависимая: любой жанр (RPG/TD/VN/Survival) всё равно порождает одинаковый конвейер данных:

**аккаунт → сессии → события → состояние → агрегаты → рейтинг**

---

## 2.1. Архитектурный контекст и роль БД

```
Game Client (Unity / Web / Mobile)
        ↓ JSON / HTTPS
REST API (Stateless + JWT)
        ↓
Backend (Business Logic)
        ↓
Relational DB (MySQL 8.0+)
```

### Почему именно такая схема (а не “клиент всё считает сам”)?

* **Клиент нельзя считать источником истины**: он может быть модифицирован, “накручен”, может работать офлайн или с ошибками.
* Сервер — место, где применяются правила: **экономика, прогресс, рейтинги, античит**.
* База данных — “память проекта”: без неё нельзя доказать, что произошло, и восстановить состояние.

### Ключевой смысл контура (логика данных)

1. Клиент отправляет **событие**: “построил башню”, “закончил матч”, “получил награду”.
2. API проверяет: JSON валиден? JWT валиден? игрок не забанен? частота запросов нормальная?
3. Backend:

   * записывает **факт** (event),
   * обновляет **состояние** (progress),
   * обновляет **агрегаты** (stats),
   * обновляет **сравнение** (leaderboard).
4. **БД — источник истины**: сервер решает, что принять, а что отклонить.

---

## 2.2. Концепция универсальной модели: 8 обязательных блоков ядра

### 2.2.1. Users (аккаунт)

Храним:

* идентификатор,
* email (уникальный логин),
* password_hash (не пароль),
* nickname,
* created_at/last_login_at,
* is_banned.

**Почему обязательно `is_banned` в users:**
бан — это “серверная политика”, которая должна мгновенно отключать доступ независимо от клиента.

---

### 2.2.2. Roles (права доступа)

Роли нужны, потому что одна ИС обслуживает:

* игроков,
* модераторов (проверка логов/жалоб),
* админов (управление конфигами/сезонами/досками).

**Почему роли — отдельная сущность, а не поле `users.role`:**
потому что:

* ролей может быть несколько (M:N),
* права могут меняться,
* админка/модерация развивается отдельно от игры.

---

### 2.2.3. GameSessions (сессии)

Сессия — факт запуска игры и временной интервал.

Зачем сессии нужны *в реальных задачах*:

* retention D1/D7/D30 (по факту входов),
* playtime (время в игре),
* расследование “что происходило перед багом/баном”.

**Почему сессия отдельно от событий:**
событий много, а сессия — “контейнер”, который упрощает аналитику и аудит.

---

### 2.2.4. GameEvents (поток событий)

Events — главный “сырой источник” поведения.

**Почему events нужны, даже если есть progress/stats:**

* progress показывает “что сейчас”, но не объясняет “как к этому пришли”.
* events позволяют:

  * восстановить историю,
  * расследовать,
  * строить аналитику баланса,
  * видеть аномалии (античит).

---

### 2.2.5. PlayerProgress (состояние игрока сейчас)

Progress — агрегированное “сальдо”:

* level, xp,
* soft/hard currency,
* updated_at.

**Почему progress нельзя вычислять каждый раз из events:**
из-за объёма и скорости:

* events быстро растут до миллионов строк,
* пересчёт прогресса “на лету” убивает производительность,
* UI профиля должен грузиться быстро.

---

### 2.2.6. Achievements (справочник) + UserAchievements (факты получения)

Разделение:

* achievements: “что существует”,
* user_achievements: “кто получил”.

**Почему это важно:**
если хранить название достижения внутри `user_achievements`, любое изменение текста потребует обновлять тысячи строк.

---

### 2.2.7. Statistics (агрегаты)

Статистика хранится по дням/неделям/сезонам.

**Почему агрегаты вынесены отдельно от events:**
потому что аналитические запросы типа “сколько игроков играли вчера” или “сколько времени провёл игрок за неделю” не должны сканировать всю таблицу events.

---

### 2.2.8. Leaderboard (рейтинги)

Лидерборд вынесен отдельно, потому что:

* досок много,
* сезоны меняются,
* правила рейтинга разные.

---

### 2.2.9. Конвейер обработки: “сырьё → продукт”

> **Events** — сырьё
> **Progress / Stats / Leaderboard** — продукты переработки

##### Пример конвейера (match_finish)

1. приходит event: `match_finish` (score, duration, isWin)
2. сервер:

* записывает событие,
* начисляет награды (xp/currency),
* обновляет progress,
* увеличивает статистику дня,
* делает upsert в leaderboard.

---

## 2.3. Логическая ER-модель (универсальное ядро)

```mermaid id="z2u6kq"
erDiagram
  USERS {
    int id PK
    string email UNIQUE
    string password_hash
    string nickname
    datetime created_at
    datetime last_login_at
    bool is_banned
  }

  ROLES {
    int id PK
    string name UNIQUE
  }

  USER_ROLES {
    int user_id PK, FK
    int role_id PK, FK
    datetime assigned_at
  }

  GAME_SESSIONS {
    int id PK
    int user_id FK
    string client_platform
    string client_version
    string ip
    datetime started_at
    datetime ended_at
  }

  GAME_EVENTS {
    bigint id PK
    int user_id FK
    int session_id FK
    string event_type
    json payload_json
    datetime created_at
  }

  PLAYER_PROGRESS {
    int user_id PK, FK
    int level
    int xp
    int soft_currency
    int hard_currency
    datetime updated_at
  }

  ACHIEVEMENTS {
    int id PK
    string code UNIQUE
    string title
    string description
    int points
  }

  USER_ACHIEVEMENTS {
    int user_id PK, FK
    int achievement_id PK, FK
    datetime unlocked_at
  }

  STATISTICS_DAILY {
    int id PK
    int user_id FK
    date day
    int sessions_count
    int events_count
    int playtime_seconds
    int wins
    int losses
    int score_sum
  }

  LEADERBOARD_SCORES {
    int id PK
    int user_id FK
    string board_code
    int season
    int score
    datetime updated_at
  }

  USERS ||--o{ USER_ROLES : has
  ROLES ||--o{ USER_ROLES : includes

  USERS ||--o{ GAME_SESSIONS : starts
  GAME_SESSIONS ||--o{ GAME_EVENTS : contains
  USERS ||--o{ GAME_EVENTS : produces

  USERS ||--|| PLAYER_PROGRESS : owns

  USERS ||--o{ USER_ACHIEVEMENTS : unlocks
  ACHIEVEMENTS ||--o{ USER_ACHIEVEMENTS : granted

  USERS ||--o{ STATISTICS_DAILY : aggregates
  USERS ||--o{ LEADERBOARD_SCORES : ranks
```

---

## 2.4. Логика связей: что они дают системе (с примерами сценариев)

### 2.4.1. Users ↔ Roles (M:N)

**Сценарий:** “модератор заблокировал игрока”

* модератор имеет роль `moderator`,
* сервер даёт доступ к `/admin/ban`,
* игрок после бана не проходит auth-check.

**Почему M:N:** один человек может быть и `player`, и `moderator` (например, тестировщик).

---

### 2.4.2. Users → GameSessions (1:M)

**Сценарий:** “посчитать D7 retention”

* берём пользователей, у которых есть session в день регистрации + session через 7 дней.

**Почему сессии нужны отдельно:** события могут не фиксировать “вход”, а сессия — фиксирует.

---

### 2.4.3. GameSessions → GameEvents (1:M)

**Сценарий:** “игра вылетает — какие события были перед вылетом?”

* берём последнюю сессию,
* берём последние 20 событий этой сессии,
* видим последовательность действий.

**Почему payload_json:** разные жанры требуют разные поля, а события — универсальная шина.

---

### 2.4.4. Users → PlayerProgress (1:1)

**Сценарий:** “открыть профиль за 100 мс”

* один SELECT users + один SELECT progress,
* не нужно анализировать events.

**Почему прогресс 1:1:** это текущее агрегированное состояние, оно одно.

---

### 2.4.5. Users ↔ Achievements (M:N)

**Сценарий:** “дать достижение за 100 побед”

* статистика/ивенты подтверждают условие,
* запись в `user_achievements` фиксирует факт навсегда.

**Почему справочник достижений отдельно:** меняем описание и не трогаем историю.

---

### 2.4.6. Users → StatisticsDaily (1:M)

**Сценарий:** “аналитика: сколько времени играют по дням”

* агрегаты дают быстрый ответ без событий.

**Почему дневная статистика:** самый практичный базовый слой (можно дополнить weekly/monthly).

---

### 2.4.7. Users → LeaderboardScores (1:M)

**Сценарий:** “лидерборд TD за сезон 3”

* выбираем по board_code+season,
* сортируем score DESC,
* выдаём top-N.

**Почему leaderboard отдельный продукт:** лидерборд — это не “свойство пользователя”, а отдельная бизнес-функция.

---

## 2.5. Нормализация до 3NF — примеры “как правильно” и “как неправильно”

### Неправильно (типичный студенческий анти-паттерн)

`users`:

* email
* password_hash
* role_name
* level
* xp
* soft_currency
* best_score
* last_10_events (текстом)

Почему плохо:

* при изменении роли или формата событий переписываете users;
* дублирование и несогласованность данных;
* невозможно масштабировать.

### Правильно (3NF)

* роль живёт в roles/user_roles,
* прогресс — в player_progress,
* лучший результат — в leaderboard_scores,
* события — в game_events.

---

## 2.6. Индексация — показываем, какие запросы “поддерживает” индекс

### 2.6.1. Login

Запрос:

```sql
SELECT id, password_hash, is_banned
FROM users
WHERE email = ?;
```

Индекс: `UNIQUE(users.email)`.

---

### 2.6.2. История сессий игрока

```sql
SELECT *
FROM game_sessions
WHERE user_id = ?
ORDER BY started_at DESC
LIMIT 20;
```

Индекс: `(user_id, started_at)`.

---

### 2.6.3. Последние события игрока

```sql
SELECT *
FROM game_events
WHERE user_id = ?
ORDER BY created_at DESC
LIMIT 50;
```

Индекс: `(user_id, created_at)`.

---

### 2.6.4. Аналитика по типу события

```sql
SELECT COUNT(*)
FROM game_events
WHERE event_type = 'purchase'
  AND created_at >= NOW() - INTERVAL 7 DAY;
```

Индекс: `(event_type, created_at)`.

---

### 2.6.5. Лидерборд

```sql
SELECT user_id, score
FROM leaderboard_scores
WHERE board_code = ? AND season = ?
ORDER BY score DESC
LIMIT 100;
```

Индекс: `(board_code, season, score)`.

---

## 2.7. Ограничения целостности — почему “БД должна защищать систему”

### FK защищают от “висячих” данных

Например: событие без пользователя — мусор и ошибка аналитики.

### UNIQUE защищают от дублей

Например:

* stats на (user_id, day) должны быть **одни**, иначе агрегаты будут неверны.

### CHECK защищают от абсурда

Например:

* level = -5 не должен существовать даже по ошибке.

---

## 2.8. MySQL-специфика: что важно именно здесь

* `utf8mb4` — никнеймы/эмодзи/локализация.
* `datetime(3)` — важна точность при плотных событиях.
* `JSON` — удобно для payload, но **не заменяет** индексацию по event_type/time.
* `ON DUPLICATE KEY UPDATE` — must-have для leaderboard/stats (upsert).

---

## 2.9. Жанровая адаптация — принцип расширения (подчёркнуто практично)

**Правило архитектора:**

* ядро остаётся неизменным;
* жанр добавляет “центральную сущность процесса”:

  * match/run/episode/cycle,
* плюс таблицы вокруг неё.

Пример:

* TD: `matches`, `waves`, `battle_results`
* VN: `story_nodes`, `user_choices`
* Idle: `user_generators`, `offline_income`

И всё это всё равно привязано к `users` и логируется через `game_events`.

---

## 2.10. Минимально жизнеспособное ядро (MVP) — почему этого достаточно

Для запуска MVP, где есть:

* аккаунты,
* авторизация,
* прогресс,
* события,
* лидерборд,

достаточно:

* `users`
* `roles`, `user_roles`
* `game_sessions`
* `game_events`
* `player_progress`
* `leaderboard_scores`

### Почему можно отложить achievements/statistics

* достижения и статистика — это “надстройки” над событиями и прогрессом;
* их можно добавить итерацией №2, не ломая ядро.

---

## 2.11. Набор обязательных `event_type` + примеры `payload_json` (универсальный словарь событий)

Цель словаря событий — задать **единый контракт телеметрии** между клиентом и сервером.
Почему это важно:

* **единая аналитика**: одинаковые события в разных жанрах дают сопоставимые метрики;
* **расследование проблем**: по последовательности событий видно, что делал игрок;
* **античит-валидация**: сервер проверяет допустимость параметров события;
* **расширяемость под жанры**: новые события добавляются без ломки ядра.

### 2.11.1. Общая структура события (как должен выглядеть payload)

Минимальный шаблон, который рекомендуется для всех событий:

```json
{
  "eventType": "string",
  "sessionId": 123,
  "clientTime": "2026-02-26T12:10:05.120Z",
  "clientVersion": "1.0.3",
  "payload": {}
}
```

На сервере это превращается в запись `game_events`:

* `user_id` — из JWT (не из клиента!)
* `session_id` — из body (можно null)
* `event_type` — строка
* `payload_json` — объект
* `created_at` — серверное время

---

### 2.11.2. Категории обязательных событий (ядро)

Ниже — **минимальный обязательный набор** событий, достаточный для любой игры.

#### A) Системные события (жизненный цикл)

1. `session_start` — начало игровой сессии
2. `session_end` — завершение сессии (или heartbeat)

**session_start**

```json
{
  "platform": "unity",
  "device": { "os": "Android", "model": "Pixel 7" },
  "locale": "ru-RU",
  "tzOffsetMin": 180
}
```

**session_end**

```json
{
  "reason": "user_exit",
  "playtimeSeconds": 842
}
```

> Почему эти события нужны, если есть `game_sessions`:
> `game_sessions` фиксирует интервал на сервере, но `session_end` помогает понять *почему* завершилось (exit/crash/network).

---

#### B) События аккаунта и авторизации

3. `register` — регистрация (опционально логировать)
4. `login_success` / `login_failed`

**login_failed**

```json
{
  "reason": "invalid_password",
  "emailHash": "sha256:...."
}
```

> Важно: **не хранить email в чистом виде** в событиях, лучше hash/маска.

---

#### C) Прогресс и экономика (универсально)

5. `progress_update` — изменение прогресса (если обновляете прогресс)
6. `currency_change` — изменение валюты
7. `item_grant` — выдача предмета/награды
8. `item_spend` — трата ресурса/предмета

**progress_update**

```json
{
  "levelBefore": 5,
  "levelAfter": 6,
  "xpBefore": 2300,
  "xpAfter": 2450,
  "reason": "match_finish"
}
```

**currency_change**

```json
{
  "currency": "soft",
  "delta": 300,
  "balanceAfter": 1200,
  "reason": "reward"
}
```

> Почему currency_change полезен даже если есть `player_progress`:
> прогресс показывает “сколько стало”, а currency_change объясняет **почему изменилось** (аудит экономики).

---

#### D) События контента (что игрок делает в игре)

9. `level_start` / `level_complete` (или `stage_start`/`stage_complete`)
10. `quest_start` / `quest_complete` (если RPG/квесты)
11. `tutorial_step` (если есть обучение)

**level_complete**

```json
{
  "levelCode": "level_03",
  "result": "win",
  "durationSeconds": 410,
  "score": 12450
}
```

**tutorial_step**

```json
{
  "step": 3,
  "status": "completed",
  "durationSeconds": 18
}
```

---

#### E) Соревновательные/матчевые события (универсально для match-based)

12. `match_start`
13. `match_event` (для ключевых действий в матче)
14. `match_finish`

**match_start**

```json
{
  "mode": "td",
  "mapCode": "forest_01",
  "season": 1
}
```

**match_event** (пример для TD build)

```json
{
  "matchId": 987654321,
  "event": "build",
  "atSecond": 12,
  "payload": {
    "buildingCode": "tower_arrow",
    "cell": { "x": 5, "y": 7 },
    "cost": 100
  }
}
```

**match_finish**

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

> Почему выделяем match_start/finish отдельно:
> эти события почти всегда запускают транзакцию “результат → награды → статистика → лидерборд”.

---

#### F) Достижения

15. `achievement_unlocked`

```json
{
  "code": "FIRST_WIN",
  "points": 10,
  "context": { "mode": "td", "season": 1 }
}
```

---

#### G) Ошибки клиента (крайне рекомендовано)

16. `client_error`
17. `crash_report` (если интеграция есть)

**client_error**

```json
{
  "type": "network",
  "message": "timeout",
  "endpoint": "/event",
  "retryCount": 2
}
```

---

### 2.11.3. Правила проектирования event_type (чтобы не было хаоса)

**1) Именование:**

* lower_snake_case: `match_finish`, `currency_change`
* стабильные имена (нельзя менять без миграций аналитики)

**2) Payload должен быть “самодостаточным”:**

* причина изменения (`reason`)
* ключевые параметры результата (`score`, `durationSeconds`)

**3) Серверная истина:**

* `user_id` только из JWT
* `created_at` только серверное время
* награды лучше вычислять сервером, а не принимать “как есть”

---

## 2.12. Таблица: какой endpoint какие таблицы читает/пишет

Цель таблицы — дать студенту **карту данных**: что трогает каждый endpoint, где нужны транзакции, какие индексы критичны.

> Обозначения:
> **R** — Read (читает)
> **W** — Write (пишет)
> **TX** — транзакция рекомендуется/обязательна

| Endpoint                     | Таблицы (R)                                                                  | Таблицы (W)                                                                   | TX       | Зачем / примечания                                                       |
| ---------------------------- | ---------------------------------------------------------------------------- | ----------------------------------------------------------------------------- | -------- | ------------------------------------------------------------------------ |
| **POST /register**           | `users` (проверка email), `roles`                                            | `users`, `user_roles`, (опц.) `player_progress`                               | да       | Создать аккаунт, назначить роль `player`, опционально создать прогресс   |
| **POST /login**              | `users`, `user_roles`, `roles`                                               | `users` (обновить last_login_at), (опц.) `game_sessions`                      | нет/опц. | Проверка пароля, банов, формирование JWT claims                          |
| **GET /profile**             | `users`, `player_progress`, (опц.) `user_achievements`, `leaderboard_scores` | —                                                                             | нет      | Витрина профиля: состояние + краткие достижения/позиции                  |
| **POST /progress**           | `player_progress` (текущее)                                                  | `player_progress`, `game_events` (progress_update), (опц.) `statistics_daily` | да       | Обновление прогресса должно быть атомарным с логом события               |
| **POST /event**              | `game_sessions` (опц. проверка), (опц.) `player_progress`                    | `game_events`, (опц.) `statistics_daily`, (опц.) `leaderboard_scores`         | иногда   | Простые события — без TX, “итоговые” (match_finish) — лучше TX           |
| **GET /leaderboard**         | `leaderboard_scores`, `users`                                                | —                                                                             | нет      | Топ-N по board_code/season, требуется индекс (board_code, season, score) |
| **GET /stats**               | `statistics_daily`                                                           | —                                                                             | нет      | Диапазоны дат, сводка за 7/30 дней                                       |
| *(опц.) POST /session/start* | —                                                                            | `game_sessions`, `game_events` (session_start)                                | нет      | Если сессии создаёте отдельным endpoint                                  |
| *(опц.) POST /session/end*   | `game_sessions`                                                              | `game_sessions`, `game_events` (session_end)                                  | да       | Закрыть сессию + зафиксировать причину завершения                        |

### 2.12.1. Где точно нужна транзакция (типовой список)

Транзакция обязательна, если endpoint:

* обновляет `player_progress` **и** пишет в `leaderboard_scores`/`statistics_daily`;
* завершает матч (`match_finish`);
* выдаёт награды (item_grant/currency_change) и меняет баланс.

Иначе появляются “полу-записанные” состояния:

* прогресс обновился, а лидерборд нет;
* событие записалось, а награда нет;
* статистика посчиталась дважды.
