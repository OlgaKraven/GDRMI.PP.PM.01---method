# 2. Практика. 1. Архитектура

## 1.1. Задание к разделу «Архитектура»

### 1.1.1. Необходимо выполнить

1. Создать архитектурную схему ИС сопровождения игрового продукта.
2. Описать один серверный сценарий прохождения запроса через систему.
3. Оформить всё в файле:

```text
docs/01-architecture.md
```

---

## 1.2. Архитектурная схема

### 1.2.1. Требования к схеме

Схема должна быть выполнена в формате **Mermaid flowchart** и содержать строго 5 блоков:

* **Client**
* **API**
* **Auth (JWT + RBAC)**
* **Business Logic**
* **Data Layer (MySQL + optional Redis/MQ)**

Обязательно:

* указать направление запроса (Client → API → Auth → Business Logic → DB);
* указать протокол передачи данных (HTTPS + JSON);
* показать последовательность прохождения запроса;
* не добавлять лишние блоки (например, CDN, Nginx, Docker — если они не входят в минимальный контур).

---

### 1.2.2. Методическое пояснение (как адаптировать под свою игру)

При адаптации схемы под свою игру необходимо:

* заменить абстрактный Client на конкретный:

  * Unity (2D/3D),
  * Web (React/Vue),
  * Mobile (Android/iOS);

* указать назначение Business Logic (что именно делает сервер в вашей игре), например:

  * Match Service (матчи / сессии),
  * Economy Engine (валюта / ресурсы / покупки),
  * Achievement Engine (достижения),
  * City Service (строительство / улучшения),
  * Quest Service (квесты / задания);

* при необходимости показать optional-компоненты:

  * Redis — кэш (профиль, лидерборд, сессии),
  * MQ — обработка событий (лог событий, аналитика, античит, уведомления).

⚠ Важно:  
Схема должна отражать **архитектуру сопровождения игрового продукта**, а не просто “сайт + БД”.  
То есть обязательно присутствует игровой сценарий: матч/бой, строительство, прогресс, экономика, события, рейтинги.

---

## 1.3. Описание одного серверного сценария

### 1.3.1. Выбор endpoint

Выбрать один endpoint:

* `POST /auth/login`
* `POST /match/finish`
* `GET /profile`

---

### 1.3.2. Описание должно включать

Описание должно включать:

* пример HTTP-запроса;
* проверки Auth;
* валидацию;
* действия Business Logic;
* SQL-запросы;
* JSON-ответ;
* минимум 2 возможные ошибки.

---

### 1.3.3. Методическое пояснение (как выбрать сценарий под жанр)

При адаптации под свою игру:

* Если это **RPG** → можно описать `POST /battle/complete`
* Если это **Tower Defense** → `POST /match/finish`
* Если это **Shooter** → `POST /session/end`
* Если это **Idle / Simulator** → `POST /economy/collect`
* Если это **Strategy** → `POST /city/upgrade`

Главное — показать:

1. Что расчёты выполняются **на сервере**, а не доверяются клиенту.
2. Что используется транзакция.
3. Что обновляются связанные таблицы (progress/resources/leaderboard/rewards).
4. Что предусмотрены ошибки конкурентного доступа (race condition, повторная отправка).
5. Что входные данные валидируются (типы, диапазоны, принадлежность объекта пользователю).

---

## 1.4. Требования к проекту (Flask)

### 1.4.1. Рекомендуемый стек

* Язык: **Python 3.10+**
* Web-фреймворк: **Flask**
* База данных: **MySQL 8.0**
* ORM (рекомендуется): **SQLAlchemy**
* Миграции (рекомендуется): **Flask-Migrate**
* Аутентификация: **JWT** (например, `flask-jwt-extended`)
* Формат обмена данными: **JSON**
* Протокол: **HTTPS**

---

### 1.4.2. Минимальные зависимости (пример `requirements.txt`)

```text
Flask
flask-jwt-extended
flask-sqlalchemy
flask-migrate
pymysql
python-dotenv
```

---

## 1.5. Минимальная структура проекта (Flask)

```text
project-root/
│
├── docs/
│   └── 01-architecture.md
│
├── app/
│   ├── __init__.py
│   ├── extensions.py
│   │
│   ├── routes/
│   │   ├── auth.py
│   │   ├── match.py
│   │   └── profile.py
│   │
│   ├── models/
│   │   ├── user.py
│   │   ├── match.py
│   │   └── progress.py
│   │
│   └── services/
│       └── match_service.py
│
├── config.py
├── run.py
└── requirements.txt
```

---

## 1.6. Как создать файл `docs/01-architecture.md`

1. Создать папку `docs`
2. Создать файл `01-architecture.md`
3. Вставить в файл:

   * Mermaid-схему
   * описание одного сценария
   * SQL
   * JSON

---

## 1.7. Пример выполнения (Tower Defense)

### 1.7.1. Архитектурная схема

```mermaid
flowchart LR
  C[Client\Unity TD Game] -->|HTTPS + JSON| A[API\nREST]
  A --> AU[Auth\nJWT + RBAC]
  AU --> BL[Business Logic\nMatch / Rewards]
  BL --> D[Data Layer\nMySQL 8.0]

  BL -.cache.-> R[(Redis optional)]
  BL -.events.-> Q[(MQ optional)]
```

---

### 1.7.2. Сценарий: `POST /match/finish`

#### 1.7.2.1. Клиентский запрос

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

#### 1.7.2.2. Auth (JWT + RBAC)

Проверяется:

* JWT валиден
* `userId` извлечён (например `15`)
* роль = `player`

Ошибка:

```http
401 Unauthorized
```

---

#### 1.7.2.3. Validation

Проверяется:

* матч существует
* матч принадлежит `userId`
* статус матча = `started`
* матч ещё не завершён

Ошибка:

```http
409 Conflict
```

---

#### 1.7.2.4. Business Logic

Расчёт наград (сервер рассчитывает сам):

```text
xp = 50 + (waveReached * 10)
softCurrency = 100 + (waveReached * 20)
```

Для `waveReached = 12`:

```text
xp = 170
softCurrency = 340
```

---

#### 1.7.2.5. Data Layer (MySQL, транзакция)

```text
START TRANSACTION;
```

Закрытие матча:

```sql
UPDATE matches
SET status = 'finished',
    ended_at = CURRENT_TIMESTAMP(3)
WHERE id = 987654321;
```

Запись результата:

```sql
INSERT INTO battle_results
(match_id, is_win, score, duration_seconds, wave_reached)
VALUES
(987654321, 1, 12450, 410, 12);
```

Обновление прогресса:

```sql
UPDATE player_progress
SET xp = xp + 170,
    soft_currency = soft_currency + 340
WHERE user_id = 15;
```

Обновление лидерборда:

```sql
INSERT INTO leaderboard_scores (user_id, board_code, season, score)
VALUES (15, 'td_score', 1, 12450)
ON DUPLICATE KEY UPDATE
  score = GREATEST(score, VALUES(score));
```

```text
COMMIT;
```

---

#### 1.7.2.6. Ответ сервера

```json
{
  "matchId": 987654321,
  "status": "finished",
  "rewards": {
    "xp": 170,
    "softCurrency": 340
  }
}
```

---

#### 1.7.2.7. Возможные ошибки (минимум 2)

* `401 Unauthorized` — JWT отсутствует/просрочен/невалиден.
* `409 Conflict` — матч уже завершён или не принадлежит пользователю.
* `404 Not Found` — matchId не существует (дополнительно, если хотите).

---

## 1.8. На что обратить внимание при адаптации под свою игру

### 1.8.1. Сервер — источник истины

* Нельзя доверять клиентским расчётам наград.
* Нельзя принимать XP и валюту “как есть”.
* Все расчёты должны быть воспроизводимы сервером.

---

### 1.8.2. Целостность данных

* Использовать транзакции.
* Предусмотреть повторный запрос (idempotency).
* Защититься от двойной отправки.

---

### 1.8.3. Архитектурная чистота

Разделение слоёв обязательно:

* routes → только HTTP
* services → бизнес-логика
* models → ORM
* DB → хранение

---

### 1.8.4. Жанровая специфика

| Жанр      | Что обязательно отразить     |
| --------- | ---------------------------- |
| RPG       | инвентарь, квесты, XP        |
| Shooter   | матчи, K/D, рейтинг          |
| Strategy  | ресурсы, строительство       |
| Simulator | экономика, таймеры           |
| Idle      | начисление дохода по времени |

---

## 1.9. Пример выполнения (Strategy — Unity Strategy Game)

### 1.9.1. Архитектурная схема

```mermaid
flowchart LR
  C[Client\Unity Strategy Game] -->|HTTPS + JSON| A[REST API]
  A --> AU[Auth\nJWT + RBAC]
  AU --> BL[Business Logic\nCity / Economy / Battle]
  BL --> D[Data Layer\nMySQL 8.0]

  BL -.cache.-> R[(Redis optional)]
  BL -.events.-> Q[(Message Queue optional)]
```

---

### 1.9.2. Сценарий: `POST /city/upgrade`

#### 1.9.2.1. Клиентский запрос

```http
POST /city/upgrade
Authorization: Bearer <JWT>
Content-Type: application/json
```

```json
{
  "buildingId": 12,
  "targetLevel": 3
}
```

---

#### 1.9.2.2. Auth (JWT + RBAC)

Проверяется:

- JWT валиден
- извлечён `userId`
- роль пользователя = `player`
- пользователь не заблокирован

Ошибка:

```http
401 Unauthorized
```

---

#### 1.9.2.3. Validation

Проверяется:

- здание существует
- здание принадлежит пользователю
- текущий уровень = 2
- targetLevel = 3 (последовательное улучшение)
- хватает ресурсов
- нет активного строительства этого здания

Ошибка:

```http
409 Conflict
```

---

#### 1.9.2.4. Business Logic

Для уровня 3 сервер рассчитывает:

```text
goldCost  = 500
woodCost  = 300
foodCost  = 200
buildTime = 3600 seconds
```

Сервер:

1. Проверяет ресурсы.
2. Списывает ресурсы.
3. Создаёт запись строительства.
4. Обновляет состояние здания (status = upgrading).

---

#### 1.9.2.5. Data Layer (MySQL, транзакция)

```text
START TRANSACTION;
```

Проверка баланса:

```sql
SELECT gold, wood, food
FROM player_resources
WHERE user_id = 15
FOR UPDATE;
```

Списание ресурсов:

```sql
UPDATE player_resources
SET gold = gold - 500,
    wood = wood - 300,
    food = food - 200
WHERE user_id = 15;
```

Создание записи строительства:

```sql
INSERT INTO building_upgrades
(user_id, building_id, target_level, started_at, finish_at)
VALUES
(15, 12, 3, NOW(), DATE_ADD(NOW(), INTERVAL 3600 SECOND));
```

Обновление статуса здания:

```sql
UPDATE city_buildings
SET status = 'upgrading'
WHERE id = 12 AND user_id = 15;
```

```text
COMMIT;
```

---

#### 1.9.2.6. Ответ сервера

```json
{
  "buildingId": 12,
  "status": "upgrading",
  "targetLevel": 3,
  "finishAt": "2026-02-26T18:45:00Z",
  "remainingResources": {
    "gold": 1200,
    "wood": 850,
    "food": 600
  }
}
```

---

#### 1.9.2.7. Возможные ошибки (минимум 2)

* `401 Unauthorized` — токен отсутствует/невалиден/просрочен.
* `409 Conflict` — недостаточно ресурсов / здание уже улучшается / нарушена последовательность уровней.
* `404 Not Found` — здание не принадлежит пользователю (дополнительно).

---

## 1.10. Шаблон для файла `docs/01-architecture.md`

```markdown
# 2. Практика. 1. Архитектура

**ФИО:** Иванов Иван Иванович
**Группа:** ДКИП-401гдрми
**Проект:** <название проекта>  
**Жанр:** <жанр игры>  
**Клиент:** <Unity / Web / Mobile>  
**Backend:** Flask + MySQL  
**Дата:** <дата>

---

## 1.1. Задание к разделу «Архитектура»

### 1.1.1. Цель

Кратко описать цель разработки архитектуры ИС сопровождения игрового продукта.

---

## 1.2. Архитектурная схема

### 1.2.1. Схема (Mermaid)

```mermaid
flowchart LR
  C[Client] -->|HTTPS + JSON| A[API]
  A --> AU[Auth\nJWT + RBAC]
  AU --> BL[Business Logic]
  BL --> D[Data Layer\nMySQL 8.0]

  BL -.cache(optional).-> R[(Redis)]
  BL -.events(optional).-> Q[(Message Queue)]
```

---

### 1.2.2. Пояснение к архитектуре

Опишите:

- какую роль выполняет Client;
- что делает API;
- что проверяет Auth;
- какие задачи решает Business Logic;
- какие данные хранит Data Layer;
- используются ли Redis / MQ и зачем.

---

## 1.3. Описание серверного сценария

### 1.3.1. Выбранный endpoint

`<указать endpoint>`

Примеры:

- `POST /auth/login`
- `POST /match/finish`
- `GET /profile`
- `POST /city/upgrade`
- `POST /battle/complete`

---

## 1.4. HTTP-запрос

```http
<пример HTTP-запроса>
```

```json
<пример JSON-тела запроса>
```

---

## 1.5. Auth (JWT + RBAC)

Указать:

- как проверяется JWT;
- какие данные извлекаются из токена;
- какие роли допускаются;
- какие ошибки возможны.

**Возможная ошибка:**

```http
401 Unauthorized
```

```json
{
  "error": "...",
  "message": "..."
}
```

---

## 1.6. Validation

Опишите проверки:

- существование объекта;
- принадлежность пользователю;
- корректность параметров;
- бизнес-ограничения (уровни, ресурсы, статус и т.д.).

**Возможная ошибка:**

```http
409 Conflict
```

```json
{
  "error": "...",
  "message": "..."
}
```

---

## 1.7. Business Logic

Опишите:

- какие расчёты выполняет сервер;
- какие данные обновляются;
- какие сущности участвуют;
- почему клиент не является источником истины.

Если есть формулы — привести:

```text
<пример расчёта>
```

---

## 1.8. Data Layer (MySQL, транзакция)

### 1.8.1. Начало транзакции

```text
START TRANSACTION;
```

---

### 1.8.2. SQL-запросы

```sql
-- пример SELECT
<SQL-запрос>
```

```sql
-- пример UPDATE
<SQL-запрос>
```

```sql
-- пример INSERT
<SQL-запрос>
```

---

### 1.8.3. Завершение транзакции

```text
COMMIT;
```

---

## 1.9. JSON-ответ сервера

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "<ключ>": "<значение>"
}
```

---

## 1.10. Возможные ошибки (минимум 2)

### 1.10.1. 401 Unauthorized

Причина: <описание>

---

### 1.10.2. 409 Conflict

Причина: <описание>

---

### 1.10.3. (Опционально) 404 Not Found

Причина: <описание>

---

## 1.11. Архитектурные принципы проекта

Перечислить:

1. Сервер — источник истины.
2. Использование транзакций.
3. Stateless REST.
4. Разделение слоёв (routes / services / models / DB).
5. Защита от race condition.

---

## 1.12. Адаптация под жанр

| Жанр      | Какие элементы необходимо учесть |
|-----------|----------------------------------|
| RPG       |                                  |
| Shooter   |                                  |
| Strategy  |                                  |
| Simulator |                                  |
| Idle      |                                  |

---

## 1.13. Итог

Краткий вывод:

- соответствует ли архитектура модели  
  Client → API → Auth → Business Logic → DB;
- обеспечивается ли серверная валидация;
- обеспечивается ли целостность данных;
- предусмотрена ли масштабируемость.

```
