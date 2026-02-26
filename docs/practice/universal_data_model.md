# 2. Практика. Универсальная модель данных

## 2.1. Задание к разделу 2 «Универсальная модель данных»

### Необходимо выполнить

1. Подготовить **ядро ER-модели** (достаточно списком сущностей и связей).
2. Подготовить **DDL MySQL 8.0** для ядра (PK/FK/UNIQUE/CHECK, `game_events.payload_json` = `JSON`).
3. Оформить результаты в файлах:

```text
db/schema.sql
docs/02-data-model.md
```

---

### 2.1.1. Ядро ER (обязательные сущности)

Сдать ядро (можно списком):

* `Users`
* `Roles`
* `UserRoles`
* `GameSessions`
* `GameEvents`
* `PlayerProgress`
* `StatisticsDaily`
* `LeaderboardScores`

---

### 2.1.2. DDL MySQL (обязательные требования)

DDL должен содержать:

* PK у каждой таблицы
* FK связи (users → sessions/events/progress/stats/leaderboard)
* UNIQUE ограничения:

  * `users.email`
  * `roles.name`
  * `statistics_daily (user_id, day)`
  * `leaderboard_scores (user_id, board_code, season)`
* CHECK ограничения (минимально: значения не отрицательные, level >= 1, season >= 1)
* `game_events.payload_json` типа `JSON`

---

### 2.1.3. Методическое пояснение (что именно вы сдаёте и как это сделать)

**Что сдаётся “по минимуму” на этом этапе:**

* не “вся база игры”, а **универсальное ядро**, которое подходит для любого жанра;
* ядро должно уметь:

  1. хранить пользователей и роли,
  2. фиксировать сессии,
  3. писать события (telemetry / audit / gameplay),
  4. хранить прогресс,
  5. хранить агрегаты по дням,
  6. хранить рейтинги.

**Как делать ER-ядро быстро:**

1. Выпишите сущности из списка обязательных.
2. Для каждой сущности укажите “зачем она нужна” (1 строка).
3. Укажите связи в формате 1:1 / 1:M / M:N.
4. Отметьте составные уникальности (день, сезон, доска).

**Как делать DDL правильно:**

* начинайте с `roles`, затем `users`, затем таблицы со ссылками;
* для связей используйте `ON DELETE CASCADE` там, где без пользователя данные не имеют смысла;
* `payload_json` делайте `JSON NOT NULL` (это критично для универсальности);
* добавляйте `INDEX` на типовые запросы: (user_id, created_at), (board_code, season, score) и т.д.;
* CHECK в MySQL 8 работает, но важно задавать реалистичные ограничения (неотрицательные, level>=1).

---

### 2.1.4. На что обратить внимание при адаптации под свою игру

**1) Не “ломайте ядро” доменными сущностями.**
Доменные таблицы (например `buildings`, `units`, `quests`, `inventory`) идут позже. Сейчас — только ядро.

**2) События должны покрывать жанровую специфику через JSON.**
В стратегии вы не обязаны делать отдельную таблицу “строительство” на этом этапе — вы можете логировать событие:

* `event_type = 'building_upgrade_started'`
* `payload_json = { buildingId, fromLevel, toLevel, cost, finishAt }`

**3) Прогресс должен быть минимальным, но универсальным.**
Даже если в игре нет “level”, он всё равно может отражать “tier/league/rank” или “stage”.

**4) LeaderboardScores — это не только PvP.**
Подходит и для:

* “макс. волна”, “макс. очки”, “время”, “доход/час”, “эффективность”.

**5) StatisticsDaily — это агрегаты, а не сырые события.**
Сырые события — `game_events`. Агрегаты — заранее подсчитанные итоги (для быстрых отчётов).

---

## 2.2. Требования к проекту (Flask)

### Рекомендуемый стек

* Язык: **Python 3.10+**
* Web-фреймворк: **Flask**
* База данных: **MySQL 8.0**
* ORM (рекомендуется): **SQLAlchemy**
* Миграции (рекомендуется): **Flask-Migrate**
* Аутентификация: **JWT** (например, `flask-jwt-extended`)
* Формат обмена данными: **JSON**
* Протокол: **HTTPS**

---

### Минимальные зависимости (пример `requirements.txt`)

```text
Flask
flask-jwt-extended
flask-sqlalchemy
flask-migrate
pymysql
python-dotenv
```

---

### Минимальная структура проекта (доработка) 

```text
project-root/
│
├── db/
│   └── schema.sql
│
├── docs/
│   └── 02-data-model.md
│
├── app/               (может быть пустым на этом этапе)
└── README.md
```

---

### Как создать файлы

1. Создать папку `db`
2. Создать файл `db/schema.sql`
3. Создать папку `docs`
4. Создать файл `docs/02-data-model.md`
5. Вставить в файлы: перечень сущностей/связей и DDL

---

## 2.3. Пример выполнения (ядро универсальной модели)

### 2.3.1. Пример содержимого `docs/02-data-model.md`

```markdown
# 2) Универсальная модель данных (ядро)

## Сущности ядра
- users — аккаунты пользователей
- roles — роли доступа (admin/moderator/player)
- user_roles — связь users и roles (M:N)
- game_sessions — сессии пользователя
- game_events — события пользователя (payload в JSON)
- player_progress — текущее состояние пользователя
- statistics_daily — дневные агрегаты
- leaderboard_scores — рейтинги по доскам и сезонам

## Связи
- users 1:M user_roles, roles 1:M user_roles
- users 1:M game_sessions
- users 1:M game_events
- game_sessions 1:M game_events (через session_id, может быть NULL)
- users 1:1 player_progress
- users 1:M statistics_daily (UNIQUE(user_id, day))
- users 1:M leaderboard_scores (UNIQUE(user_id, board_code, season))
```

---

### 2.3.2. Пример содержимого `db/schema.sql` (MySQL 8.0)

```sql
-- MySQL 8.0+
-- Рекомендуемо:
-- ENGINE=InnoDB, CHARSET=utf8mb4, COLLATION=utf8mb4_0900_ai_ci

CREATE TABLE roles (
  id   INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(32) NOT NULL UNIQUE
) ENGINE=InnoDB;

CREATE TABLE users (
  id            INT AUTO_INCREMENT PRIMARY KEY,
  email         VARCHAR(255) NOT NULL UNIQUE,
  password_hash VARCHAR(255) NOT NULL,
  nickname      VARCHAR(64)  NOT NULL,
  created_at    DATETIME(3)  NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  last_login_at DATETIME(3)  NULL,
  is_banned     TINYINT(1)   NOT NULL DEFAULT 0
) ENGINE=InnoDB;

CREATE TABLE user_roles (
  user_id     INT NOT NULL,
  role_id     INT NOT NULL,
  assigned_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (user_id, role_id),
  CONSTRAINT fk_user_roles_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  CONSTRAINT fk_user_roles_role FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE RESTRICT,
  INDEX idx_user_roles_role_id (role_id)
) ENGINE=InnoDB;

CREATE TABLE game_sessions (
  id              INT AUTO_INCREMENT PRIMARY KEY,
  user_id         INT NOT NULL,
  client_platform ENUM('unity','web','mobile') NOT NULL,
  client_version  VARCHAR(32) NULL,
  ip              VARCHAR(45) NULL,
  started_at      DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  ended_at        DATETIME(3) NULL,
  CONSTRAINT fk_sessions_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  INDEX idx_sessions_user_started (user_id, started_at)
) ENGINE=InnoDB;

CREATE TABLE game_events (
  id           BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id      INT NOT NULL,
  session_id   INT NULL,
  event_type   VARCHAR(64) NOT NULL,
  payload_json JSON NOT NULL,
  created_at   DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),

  CONSTRAINT fk_events_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  CONSTRAINT fk_events_session FOREIGN KEY (session_id) REFERENCES game_sessions(id) ON DELETE SET NULL,

  INDEX idx_events_user_time (user_id, created_at),
  INDEX idx_events_type_time (event_type, created_at)
) ENGINE=InnoDB;

CREATE TABLE player_progress (
  user_id        INT PRIMARY KEY,
  level          INT NOT NULL DEFAULT 1,
  xp             INT NOT NULL DEFAULT 0,
  soft_currency  INT NOT NULL DEFAULT 0,
  hard_currency  INT NOT NULL DEFAULT 0,
  updated_at     DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),

  CONSTRAINT fk_progress_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  CONSTRAINT chk_level CHECK (level >= 1),
  CONSTRAINT chk_xp CHECK (xp >= 0),
  CONSTRAINT chk_soft CHECK (soft_currency >= 0),
  CONSTRAINT chk_hard CHECK (hard_currency >= 0)
) ENGINE=InnoDB;

CREATE TABLE statistics_daily (
  id               INT AUTO_INCREMENT PRIMARY KEY,
  user_id          INT NOT NULL,
  day              DATE NOT NULL,
  sessions_count   INT NOT NULL DEFAULT 0,
  events_count     INT NOT NULL DEFAULT 0,
  playtime_seconds INT NOT NULL DEFAULT 0,
  wins             INT NOT NULL DEFAULT 0,
  losses           INT NOT NULL DEFAULT 0,
  score_sum        INT NOT NULL DEFAULT 0,

  CONSTRAINT fk_stats_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  UNIQUE KEY uq_stats_user_day (user_id, day),
  INDEX idx_stats_day (day),
  INDEX idx_stats_user_day (user_id, day),

  CONSTRAINT chk_sessions_count CHECK (sessions_count >= 0),
  CONSTRAINT chk_events_count CHECK (events_count >= 0),
  CONSTRAINT chk_playtime CHECK (playtime_seconds >= 0),
  CONSTRAINT chk_wins CHECK (wins >= 0),
  CONSTRAINT chk_losses CHECK (losses >= 0),
  CONSTRAINT chk_score_sum CHECK (score_sum >= 0)
) ENGINE=InnoDB;

CREATE TABLE leaderboard_scores (
  id         INT AUTO_INCREMENT PRIMARY KEY,
  user_id    INT NOT NULL,
  board_code VARCHAR(64) NOT NULL,
  season     INT NOT NULL DEFAULT 1,
  score      INT NOT NULL DEFAULT 0,
  updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),

  CONSTRAINT fk_lb_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  UNIQUE KEY uq_lb_user_board_season (user_id, board_code, season),
  INDEX idx_lb_board_season_score (board_code, season, score),
  INDEX idx_lb_user (user_id),

  CONSTRAINT chk_season CHECK (season >= 1),
  CONSTRAINT chk_score CHECK (score >= 0)
) ENGINE=InnoDB;
```

---

## 2.4. Проверка, что схема реально работает (минимальная)

### 2.4.1. Добавить роли

```sql
INSERT INTO roles (name) VALUES ('player'), ('moderator'), ('admin');
```

### 2.4.2. Создать пользователя

```sql
INSERT INTO users (email, password_hash, nickname)
VALUES ('p1@example.com', 'hash_here', 'PlayerOne');
```

### 2.4.3. Назначить роль игроку

```sql
INSERT INTO user_roles (user_id, role_id)
SELECT u.id, r.id
FROM users u, roles r
WHERE u.email='p1@example.com' AND r.name='player';
```

### 2.4.4. Создать progress

```sql
INSERT INTO player_progress (user_id)
SELECT id FROM users WHERE email='p1@example.com';
```

### 2.4.5. Создать сессию

```sql
INSERT INTO game_sessions (user_id, client_platform, client_version, ip)
SELECT id, 'unity', '1.0.0', '127.0.0.1'
FROM users WHERE email='p1@example.com';
```

### 2.4.6. Записать событие

```sql
INSERT INTO game_events (user_id, session_id, event_type, payload_json)
SELECT
  u.id,
  s.id,
  'match_finish',
  JSON_OBJECT('score', 1200, 'durationSeconds', 300, 'isWin', true)
FROM users u
JOIN game_sessions s ON s.user_id = u.id
WHERE u.email='p1@example.com'
ORDER BY s.id DESC
LIMIT 1;
```

---

## 2.5. Пример адаптации под другую предметную область (ServiceDesk)

### Что остаётся без изменений (ядро)

* `users`, `roles`, `user_roles`
* `game_sessions` (трактуется как web-сессии)
* `game_events` (события в домене заявок)
* `statistics_daily` (агрегаты активности)
* `leaderboard_scores` (рейтинги/эффективность)

### Как выглядит доменное событие в `payload_json`

`event_type = 'ticket_status_changed'`

```json
{
  "ticketId": 501,
  "fromStatus": "new",
  "toStatus": "in_progress",
  "priority": "high"
}
```

---
 

## 2.6. Пустой шаблон `docs/02-data-model.md` для заполнения студентом

```markdown
# 2) Универсальная модель данных (ядро)

## 1. Сущности ядра (описать назначение каждой)
- users —
- roles —
- user_roles —
- game_sessions —
- game_events —
- player_progress —
- statistics_daily —
- leaderboard_scores —

## 2. Связи (кардинальности)
- users 1:M ...
- roles 1:M ...
- users 1:1 ...
- users 1:M ...
- game_sessions 1:M ...

## 3. Уникальности (UNIQUE)
- users(email)
- roles(name)
- statistics_daily(user_id, day)
- leaderboard_scores(user_id, board_code, season)

## 4. CHECK ограничения (минимально)
- level >= 1
- season >= 1
- все счётчики/валюты >= 0

## 5. Примеры событий для вашего жанра (event_type + payload_json)
1) event_type =
   payload_json =
2) event_type =
   payload_json =
3) event_type =
   payload_json =
```

---

## 2.7. Пустой шаблон `db/schema.sql` (каркас, без конкретных полей)

> Внимание: это каркас для заполнения. При сдаче он должен содержать PK/FK/UNIQUE/CHECK и JSON.

```sql
-- MySQL 8.0+
-- ENGINE=InnoDB, CHARSET=utf8mb4, COLLATION=utf8mb4_0900_ai_ci

CREATE TABLE roles (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(32) NOT NULL UNIQUE
) ENGINE=InnoDB;

CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,
  password_hash VARCHAR(255) NOT NULL,
  nickname VARCHAR(64) NOT NULL,
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  last_login_at DATETIME(3) NULL,
  is_banned TINYINT(1) NOT NULL DEFAULT 0
) ENGINE=InnoDB;

CREATE TABLE user_roles (
  user_id INT NOT NULL,
  role_id INT NOT NULL,
  assigned_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (user_id, role_id),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE RESTRICT
) ENGINE=InnoDB;

CREATE TABLE game_sessions (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT NOT NULL,
  client_platform ENUM('unity','web','mobile') NOT NULL,
  client_version VARCHAR(32) NULL,
  ip VARCHAR(45) NULL,
  started_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  ended_at DATETIME(3) NULL,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB;

CREATE TABLE game_events (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id INT NOT NULL,
  session_id INT NULL,
  event_type VARCHAR(64) NOT NULL,
  payload_json JSON NOT NULL,
  created_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  FOREIGN KEY (session_id) REFERENCES game_sessions(id) ON DELETE SET NULL
) ENGINE=InnoDB;

CREATE TABLE player_progress (
  user_id INT PRIMARY KEY,
  level INT NOT NULL DEFAULT 1,
  xp INT NOT NULL DEFAULT 0,
  soft_currency INT NOT NULL DEFAULT 0,
  hard_currency INT NOT NULL DEFAULT 0,
  updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  CONSTRAINT chk_level CHECK (level >= 1),
  CONSTRAINT chk_xp CHECK (xp >= 0),
  CONSTRAINT chk_soft CHECK (soft_currency >= 0),
  CONSTRAINT chk_hard CHECK (hard_currency >= 0)
) ENGINE=InnoDB;

CREATE TABLE statistics_daily (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT NOT NULL,
  day DATE NOT NULL,
  sessions_count INT NOT NULL DEFAULT 0,
  events_count INT NOT NULL DEFAULT 0,
  playtime_seconds INT NOT NULL DEFAULT 0,
  wins INT NOT NULL DEFAULT 0,
  losses INT NOT NULL DEFAULT 0,
  score_sum INT NOT NULL DEFAULT 0,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  UNIQUE KEY uq_stats_user_day (user_id, day),
  CONSTRAINT chk_sessions_count CHECK (sessions_count >= 0),
  CONSTRAINT chk_events_count CHECK (events_count >= 0),
  CONSTRAINT chk_playtime CHECK (playtime_seconds >= 0),
  CONSTRAINT chk_wins CHECK (wins >= 0),
  CONSTRAINT chk_losses CHECK (losses >= 0),
  CONSTRAINT chk_score_sum CHECK (score_sum >= 0)
) ENGINE=InnoDB;

CREATE TABLE leaderboard_scores (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT NOT NULL,
  board_code VARCHAR(64) NOT NULL,
  season INT NOT NULL DEFAULT 1,
  score INT NOT NULL DEFAULT 0,
  updated_at DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  UNIQUE KEY uq_lb_user_board_season (user_id, board_code, season),
  CONSTRAINT chk_season CHECK (season >= 1),
  CONSTRAINT chk_score CHECK (score >= 0)
) ENGINE=InnoDB;
```

--- 