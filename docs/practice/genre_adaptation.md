# 2. Практика 3. Жанровая адаптация ИС сопровождения игрового продукта

## 3.1. Задание к разделу 3 «Жанровая адаптация»

### Необходимо выполнить

Выбрать **один жанр** (например: TD/Strategy, RPG, VN, Economic, Shooter, Survival, Idle) и сделать:

1. Определить **единицу процесса**: `match` / `run` / `episode` / `cycle`.
2. Добавить **3–5 жанровых таблиц** (названия + ключевые поля), привязанных к `users` и к единице процесса.
3. Определить **6 событий** (`event_type`) для жанра + по **1 примеру `payload`** на каждое событие.

Оформить всё в файле:

```text
docs/03-genre.md
```

---

### 3.1.1. Правила жанровой адаптации

* **Ядро не меняется.** Таблицы ядра (Users/Sessions/Events/Progress/Stats/Leaderboard) остаются как есть.
* Жанр добавляет **свои таблицы**, которые:

  * связаны с `users`;
  * связаны с центральной сущностью процесса (`match/run/episode/cycle`).
* Ключевые действия жанра фиксируются через `game_events` (универсальная шина событий) с `event_type` и `payload_json`.

---

### 3.1.2. Необходимо для фиксации

Чтобы шаг 3 реально был **продолжением архитектуры и ядра**, в `docs/03-genre.md` нужно явно зафиксировать:

1. **Как процесс проходит по контуру Client → API → Auth → BL → DB** для выбранной единицы процесса.
   Не код, а 3–5 строк: где создаётся процесс, где закрывается, где выдаются награды.

2. **Какие эндпоинты будут работать с процессом** (минимум 2–3, в списке).
   Пример: `POST /api/v1/events` (с `eventType='match_start'`), `POST /api/v1/match/finish`, `GET /api/v1/profile`.

3. **Как события (game_events) связаны с жанровыми таблицами**:
   в каждом payload должен быть **идентификатор процесса** (`matchId/runId/episodeId/cycleId`) + необходимые ключи домена.

4. **Какие поля являются "истиной на сервере"** (anti-cheat / доверие данным):

* клиент не присылает xp/награды как "готовые";
* клиент присылает факты (например score, isWin, durationSeconds), а сервер считает награды по константам.

5. **Индексы и уникальности** для жанровых таблиц (минимально):

* FK + индекс по FK;
* уникальности там, где нужно (например `UNIQUE(user_id, unit_code)`, `UNIQUE(user_id, building_code)`).

6. **Транзакция итогов**:
   при закрытии процесса обязательно указать, что происходит атомарно:
   `result → rewards → progress → leaderboard → stats`.

---

## 3.2. Требования к содержанию `docs/03-genre.md`

### 3.2.1. Единица процесса

В файле должно быть указано:

* название единицы процесса (`match/run/episode/cycle`);
* зачем она нужна (результат, метрики, транзакции «итог → награда → прогресс»).

**Добавить обязательно (с учётом шагов 1–2):**

* какие endpoint(ы) создают и закрывают единицу процесса;
* какие поля у процесса сервер считает критичными (истина на сервере);
* какие таблицы ядра затрагиваются при закрытии процесса (минимум: `player_progress`, опционально `leaderboard_scores`, `statistics_daily`, `game_events`).

---

### 3.2.2. Жанровые таблицы (3–5)

Для каждой таблицы указать:

* название таблицы;
* ключевые поля (минимум 5–8 полей);
* связи (FK) с `users` и/или единицей процесса.

**Добавить обязательно (с учётом шага 2):**

* PK тип (INT/BIGINT) и назначение;
* FK-ограничения (ON DELETE);
* 1–2 индекса (по FK/времени/поисковому полю);
* 1 CHECK или правило диапазона (например `season >= 1`, `duration_seconds >= 0`).

---

### 3.2.3. События (6 шт.)

Для каждого события указать:

* `event_type` (строка);
* смысл события;
* пример `payload` (JSON-объект).

**Добавить обязательно (с учётом шага 2 / game_events):**

* payload должен включать:

  * идентификатор процесса (`matchId`/`runId`/`episodeId`/`cycleId`);
  * "минимально необходимые факты" для сервера;
* запрет: не включать "награду как истину", если это выдаёт сервер
  (награда может быть в `reward_granted`, но как результат расчёта BL).

---

## 3.3. Пример выполнения (Strategy)

> Этот пример демонстрирует готовую структуру файла `docs/03-genre.md` для жанра Strategy.
> Все данные соответствуют реальному проекту `strategy-support-is`.
> Студент другого жанра делает аналогично.

---

### 3.3.1. Единица процесса: `match`

**Единица процесса:** `match` — одно сражение в режиме pve или pvp.  
**Зачем нужна:** матч имеет старт/финиш, результат (win/lose, score, duration) и является основой транзакции «результат → награда → прогресс → рейтинг».

**Связь с архитектурой:**

* Client отправляет `POST /api/v1/events` с `eventType='match_start'` → событие пишется в `game_events`.
* Во время матча Client отправляет события (deploy, retreat) → сервер пишет в `game_events`.
* Client отправляет `POST /api/v1/match/finish` → сервер фиксирует итог в `matches` и `battle_results`, считает награды, обновляет `player_progress`, `leaderboard_scores`, `statistics_daily`.

**Что является истиной на сервере:**

Клиент **не присылает** `xpGained` и `softCurrencyGained`. Сервер рассчитывает их в `match_service.py` по константам:

```text
_XP_WIN   = 100   _XP_LOSS   = 30
_SOFT_WIN = 50    _SOFT_LOSS = 10
```

---

### 3.3.2. Жанровые таблицы (5)

#### 1) `matches` — матчи (единица процесса)

Ключевые поля:

* `id` BIGINT PK AUTO_INCREMENT
* `user_id` INT FK → users.id
* `mode` ENUM('pve','pvp')
* `map_code` VARCHAR(64) — код карты (`forest_raid`, `desert_clash`, `mountain_pass`, `coastal_siege`)
* `season` INT DEFAULT 1
* `status` ENUM('started','finished','abandoned')
* `started_at` DATETIME(3)
* `ended_at` DATETIME(3)

Связи:

* users 1:M matches

Индексы/ограничения:

* INDEX `(user_id, started_at)`
* CHECK `season >= 1`

---

#### 2) `battle_results` — итог матча (1:1 с matches)

Ключевые поля:

* `match_id` BIGINT PK FK → matches.id
* `is_win` TINYINT(1)
* `score` INT
* `duration_seconds` INT
* `units_lost` INT
* `units_killed` INT

Связи:

* matches 1:1 battle_results

Индексы/ограничения:

* FK ON DELETE CASCADE
* CHECK `score >= 0`, `duration_seconds >= 0`, `units_lost >= 0`, `units_killed >= 0`

---

#### 3) `army_units` — армия пользователя

Ключевые поля:

* `id` INT PK AUTO_INCREMENT
* `user_id` INT FK → users.id
* `unit_code` VARCHAR(64) — тип юнита (`infantry`, `cavalry`, `archer`, `siege`)
* `amount` INT — количество
* `power` INT — суммарная боевая сила

Связи:

* users 1:M army_units

Индексы/ограничения:

* UNIQUE `(user_id, unit_code)`
* INDEX `(user_id)`
* CHECK `amount >= 0`, `power >= 0`

---

#### 4) `city_buildings` — здания города

Ключевые поля:

* `id` INT PK AUTO_INCREMENT
* `user_id` INT FK → users.id
* `building_code` VARCHAR(64) — тип здания (`barracks`, `farm`, `forge`, `walls`)
* `level` INT DEFAULT 1
* `status` ENUM('idle','upgrading')

Связи:

* users 1:M city_buildings

Индексы/ограничения:

* UNIQUE `(user_id, building_code)`
* INDEX `(user_id)`
* CHECK `level >= 1`

---

#### 5) `processed_events` — реестр дедупликации

Ключевые поля:

* `id` BIGINT PK AUTO_INCREMENT
* `user_id` INT FK → users.id
* `event_id` VARCHAR(128) — UUID события от клиента
* `event_type` VARCHAR(64)
* `processed_at` DATETIME(3)

Связи:

* users 1:M processed_events

Индексы/ограничения:

* UNIQUE `(user_id, event_id)` — повторный `event_id` → IntegrityError → `409 EVENT_REJECTED`
* INDEX `(user_id)`

---

### 3.3.3. События жанра (6 шт.) + payload

> События записываются в `game_events` через `insert_game_event()` в `events_repo.py`.  
> `user_id` всегда берётся из JWT, не из тела запроса.  
> Каждое событие дополнительно регистрируется в `processed_events` через `register_event_or_conflict()` для защиты от дублей.

#### 1) `match_start`

```json
{
  "matchId": 9,
  "mode": "pve",
  "mapCode": "coastal_siege",
  "season": 1
}
```

#### 2) `unit_deploy`

```json
{
  "matchId": 9,
  "unitCode": "cavalry",
  "amount": 40,
  "atSecond": 15
}
```

#### 3) `unit_retreat`

```json
{
  "matchId": 9,
  "unitCode": "infantry",
  "lost": 25,
  "atSecond": 180
}
```

#### 4) `match_finish`

```json
{
  "matchId": 9,
  "isWin": true,
  "score": 1850,
  "durationSeconds": 1200
}
```

> Клиент не присылает `xpGained` и `softCurrencyGained` — сервер считает сам.

#### 5) `building_upgrade_started`

```json
{
  "buildingCode": "barracks",
  "fromLevel": 4,
  "toLevel": 5,
  "goldCost": 500
}
```

#### 6) `reward_granted`

```json
{
  "matchId": 9,
  "xp": 100,
  "softCurrency": 50,
  "reason": "match_finish"
}
```

> Это событие пишется **сервером** как результат расчёта в `match_service.py`, не клиентом.

---

## 3.4. Дополнение: связка с таблицами ядра

В `docs/03-genre.md` обязательно указать, какие таблицы ядра задействуются:

* **game_events** — лог всех ключевых действий; пишется через `events_repo.py → insert_game_event()`
* **processed_events** — дедупликация по `(user_id, event_id)` через `dedupe_repo.py → register_event_or_conflict()`
* **player_progress** — итог: xp/валюта (обновляется на match_finish через `add_progress_rewards()`)
* **leaderboard_scores** — upsert с `GREATEST(score)` по доске `default`, сезон `1` (через `upsert_leaderboard_score()`)
* **statistics_daily** — upsert агрегатов (events_count, playtime_seconds, wins/losses, score_sum) через `upsert_daily_stats()`

Это обеспечивает согласованность с разделом 2 (универсальная модель данных) и разделом 1 (архитектурный контур).

---

## 3.5. Шаблон для файла `docs/03-genre.md`

````markdown
# 3) Жанровая адаптация (раздел 3)

## 3.1. Жанр: <указать жанр>

## 3.2. Единица процесса
- Единица процесса: <match/run/episode/cycle>
- Назначение: <1–2 предложения>
- Ключевые endpoint(ы): <минимум 2–3>
- Что является истиной на сервере: <что клиент НЕ должен присылать как "готовое">

## 3.3. Жанровые таблицы (3–5)

### 1) <table_1> (единица процесса)
- PK: <тип + поле>
- Ключевые поля (5–8): ...
- FK: ...
- Индексы/уникальности: ...
- CHECK/правила: ...

### 2) <table_2>
- PK: ...
- Ключевые поля (5–8): ...
- FK: ...
- Индексы/уникальности: ...
- CHECK/правила: ...

### 3) <table_3>
- PK: ...
- Ключевые поля (5–8): ...
- FK: ...
- Индексы/уникальности: ...
- CHECK/правила: ...

(добавить до 5 таблиц)

## 3.4. События (6 event_type) + payload
> События пишутся в game_events (event_type + payload_json).
> В payload должен быть идентификатор процесса (matchId/runId/episodeId/cycleId).

1) event_type: `<...>` — <смысл>
```json
{ ... }
```

2) event_type: `<...>` — <смысл>
```json
{ ... }
```

3) event_type: `<...>` — <смысл>
```json
{ ... }
```

4) event_type: `<...>` — <смысл>
```json
{ ... }
```

5) event_type: `<...>` — <смысл>
```json
{ ... }
```

6) event_type: `<...>` — <смысл>
```json
{ ... }
```

## 3.5. Связка с ядром (обязательно)
- Какие данные пишутся в player_progress при завершении процесса:
  - xp: ...
  - soft_currency: ...
  - hard_currency: ...
- Как обновляется leaderboard_scores:
  - board_code: ...
  - season: ...
  - score: ...
- Какие поля обновляются в statistics_daily:
  - sessions_count/events_count/playtime_seconds/wins/losses/score_sum: ...

## 3.6. Транзакция "итог → награда → прогресс → рейтинг"
Опишите 3–6 строками, что должно происходить атомарно на сервере при закрытии процесса.
````
