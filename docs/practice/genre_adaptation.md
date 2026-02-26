

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
   Пример: `POST /match/start`, `POST /match/finish`, `POST /city/upgrade`.

3. **Как события (game_events) связаны с жанровыми таблицами**:
   в каждом payload должен быть **идентификатор процесса** (`matchId/runId/episodeId/cycleId`) + необходимые ключи домена.

4. **Какие поля являются “истиной на сервере”** (anti-cheat / доверие данным):

* клиент не присылает xp/награды как “готовые”;
* клиент присылает факты (например waveReached, buildingId), а сервер считает награды/стоимость.

5. **Индексы и уникальности** для жанровых таблиц (минимально):

* FK + индекс по FK;
* уникальности там, где нужно (например `UNIQUE(user_id, season, match_number)`).

6. **Транзакция итогов**:
   при закрытии процесса обязательно указать, что происходит атомарно:
   `result → rewards → progress → leaderboard → events`.

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
  * “минимально необходимые факты” для сервера;
* запрет: не включать “награду как истину”, если это выдаёт сервер
  (награда может быть в `reward_granted`, но как результат расчёта BL).

---

## 3.3. Пример выполнения (Tower Defense / Strategy)

> Этот пример демонстрирует готовую структуру файла `docs/03-genre.md`.
> Студент выбирает другой жанр и делает аналогично.

---

### 3.3.1. Единица процесса: `match`

**Единица процесса:** `match` — матч (забег) в режиме TD.
**Зачем нужна:** матч имеет старт/финиш, результат (win/lose, score, wave_reached) и является основой транзакции «результат → награда → прогресс → рейтинг».

**Связь с архитектурой (шаг 1):**

* Client отправляет `POST /match/start` → сервер создаёт запись `matches`.
* Во время матча Client отправляет события → сервер пишет в `game_events`.
* Client отправляет `POST /match/finish` → сервер фиксирует итог, считает награды, обновляет `player_progress` и `leaderboard_scores`.

---

### 3.3.2. Жанровые таблицы (минимум 5)

#### 1) `matches` — матчи (единица процесса)

Ключевые поля:

* `id` BIGINT PK
* `user_id` INT FK → users.id
* `mode` ENUM('td','pve','pvp')
* `map_code` VARCHAR(64)
* `season` INT
* `status` ENUM('started','finished','abandoned')
* `started_at` DATETIME(3)
* `ended_at` DATETIME(3)

Связи:

* users 1:M matches

Индексы/ограничения (чтобы не забыли):

* INDEX `(user_id, started_at)`
* CHECK `season >= 1`

---

#### 2) `match_waves` — волны матча

Ключевые поля:

* `id` BIGINT PK
* `match_id` BIGINT FK → matches.id
* `wave_number` INT
* `enemies_spawned` INT
* `enemies_killed` INT
* `base_hp_left` INT

Связи:

* matches 1:M match_waves

Индексы/ограничения:

* UNIQUE `(match_id, wave_number)`
* CHECK `wave_number >= 1`, `enemies_spawned >= 0`, `enemies_killed >= 0`

---

#### 3) `towers` — справочник башен

Ключевые поля:

* `id` INT PK
* `code` VARCHAR(64) UNIQUE
* `title` VARCHAR(128)
* `base_damage` INT
* `base_rate` INT
* `base_cost` INT

Связи:

* используется в build/upgrade событиях (через code)

---

#### 4) `match_build_actions` — журнал действий (build/upgrade/sell)

Ключевые поля:

* `id` BIGINT PK
* `match_id` BIGINT FK → matches.id
* `action_type` ENUM('build','upgrade','sell')
* `tower_code` VARCHAR(64)
* `at_second` INT
* `cost` INT
* `cell_x` INT
* `cell_y` INT

Связи:

* matches 1:M match_build_actions

Индексы/ограничения:

* INDEX `(match_id, at_second)`
* CHECK `at_second >= 0`, `cost >= 0`

---

#### 5) `battle_results` — итог матча

Ключевые поля:

* `match_id` BIGINT PK, FK → matches.id
* `is_win` TINYINT(1)
* `score` INT
* `duration_seconds` INT
* `wave_reached` INT
* `damage_dealt` INT
* `damage_taken` INT

Связи:

* matches 1:1 battle_results

Индексы/ограничения:

* CHECK `score >= 0`, `duration_seconds >= 0`, `wave_reached >= 0`

---

### 3.3.3. События жанра (6 шт.) + payload

> Эти события записываются в `game_events` (event_type + payload_json).
> `user_id` всегда берётся из JWT, не из клиента.

#### 1) `match_start`

```json
{
  "matchId": 987654321,
  "mode": "td",
  "mapCode": "forest_01",
  "season": 1
}
```

#### 2) `build`

```json
{
  "matchId": 987654321,
  "towerCode": "tower_arrow",
  "cell": { "x": 5, "y": 7 },
  "cost": 100,
  "atSecond": 12
}
```

#### 3) `upgrade`

```json
{
  "matchId": 987654321,
  "towerCode": "tower_arrow",
  "newLevel": 2,
  "cost": 150,
  "atSecond": 45
}
```

#### 4) `wave_end`

```json
{
  "matchId": 987654321,
  "waveNumber": 3,
  "enemiesSpawned": 20,
  "enemiesKilled": 18,
  "baseHpLeft": 85,
  "atSecond": 73
}
```

#### 5) `match_finish`

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

#### 6) `reward_granted`

```json
{
  "matchId": 987654321,
  "rewards": {
    "xp": 170,
    "softCurrency": 340
  },
  "reason": "match_finish"
}
```

---

## 3.4. Дополнение: связка с таблицами ядра 

В `docs/03-genre.md` обязательно указать, какие таблицы ядра задействуются:

* **game_events** — лог всех ключевых действий (match_start/build/…)
* **player_progress** — итог: xp/валюта/уровень (обновляется на finish)
* **leaderboard_scores** — обновление рейтинга по доске (например `td_score`)
* **statistics_daily** — обновление агрегатов (sessions_count, wins, losses, score_sum)

Это обеспечивает согласованность со 2-й работой (универсальная модель данных) и с 1-й (архитектурный контур).

---

## 3.5. Шаблон для файла `docs/03-genre.md`  

````markdown
# 3) Жанровая адаптация (раздел 3)

## 3.1. Жанр: <указать жанр>

## 3.2. Единица процесса
- Единица процесса: <match/run/episode/cycle>
- Назначение: <1–2 предложения>
- Ключевые endpoint(ы): <минимум 2–3>
- Что является истиной на сервере: <что клиент НЕ должен присылать как “готовое”>

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

## 3.6. Транзакция “итог → награда → прогресс → рейтинг”
Опишите 3–6 строками, что должно происходить атомарно на сервере при закрытии процесса.
````
 