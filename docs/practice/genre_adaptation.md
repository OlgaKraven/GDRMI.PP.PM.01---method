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

## 3.2. Требования к содержанию `docs/03-genre.md`

### 3.2.1. Единица процесса

В файле должно быть указано:

* название единицы процесса (`match/run/episode/cycle`);
* зачем она нужна (результат, метрики, транзакции «итог → награда → прогресс»).

---

### 3.2.2. Жанровые таблицы (3–5)

Для каждой таблицы указать:

* название таблицы;
* ключевые поля (минимум 5–8 полей);
* связи (FK) с `users` и/или единицей процесса.

---

### 3.2.3. События (6 шт.)

Для каждого события указать:

* `event_type` (строка);
* смысл события;
* пример `payload` (JSON-объект).

---

## 3.3. Пример выполнения (Tower Defense / Strategy)

> Этот пример демонстрирует готовую структуру файла `docs/03-genre.md`.
> Студент выбирает другой жанр и делает аналогично.

---

### 3.3.1. Единица процесса: `match`

**Единица процесса:** `match` — матч (забег) в режиме TD.
**Зачем нужна:** матч имеет старт/финиш, результат (win/lose, score, wave_reached) и является основой транзакции «результат → награда → прогресс → рейтинг».

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

## 3.4. Шаблон для файла `docs/03-genre.md`

````markdown
# 3) Жанровая адаптация (раздел 3)

## 3.1. Жанр: <указать жанр>

## 3.2. Единица процесса
- Единица процесса: <match/run/episode/cycle>
- Назначение: <1–2 предложения>

## 3.3. Жанровые таблицы (3–5)
### 1) <table_1>
- Ключевые поля: ...
- Связи (FK): ...

### 2) <table_2>
- Ключевые поля: ...
- Связи (FK): ...

### 3) <table_3>
- Ключевые поля: ...
- Связи (FK): ...

(добавить до 5 таблиц)

## 3.4. События (6 event_type) + payload
1) event_type: `<...>`
```json
{ ... }
````

2. event_type: `<...>`

```json
{ ... }
```

(всего 6 событий)

```

---
```
