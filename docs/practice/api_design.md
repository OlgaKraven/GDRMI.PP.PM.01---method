# 2. Практика. 5. API дизайн ИС сопровождения игрового продукта

## 5.0. Смысл раздела

API проектируется так, чтобы Unity/Web/Mobile могли работать с ним **предсказуемо** и **одинаково**:

* stateless REST (без серверных сессий);
* безопасность: JWT + RBAC + валидация;
* единый контракт ответов и ошибок;
* удобство тестирования (Postman / `test_client.html` / Swagger).

Ключевая идея: API — это **конвейер обработки запроса**, где всё разложено по слоям и каждый слой отвечает за своё.

---

## 5.1. Задание к разделу 5 «API дизайн»

### Необходимо выполнить

1. Реализовать MVP endpoints (минимум 5) и обеспечить, что они **реально отвечают**:

   * `POST /api/v1/auth/register`
   * `POST /api/v1/auth/login`
   * `GET /api/v1/profile`
   * `POST /api/v1/events`
   * `GET /api/v1/leaderboard`

   Опционально (если успеваете):

   * `POST /api/v1/match/finish`
   * `GET /health`

2. Реализовать **единый контракт ответов** для всех endpoint'ов:

   * успех: `{ "ok": true, "data": ... }`
   * ошибка: `{ "ok": false, "error": { "code", "message", "requestId" } }`

3. Реализовать обязательные правила REST (ресурсность, stateless, хранение состояния в БД).

4. Реализовать обязательные HTTP-коды (200/201/400/401/403/404/409/429/500) так, чтобы каждый можно было воспроизвести тестом.

### Артефакт

* Работающий API (запускается `python wsgi.py` и отвечает на запросы).
* Для сдачи достаточно Postman-проверки, `test_client.html` или Swagger UI (если подключён).

---

## 5.2. Минимальная рабочая архитектура API (обязательная логика)

API-слой должен обслуживать **несколько ресурсов** (auth, profile, events, leaderboard) единообразно.

Конвейер обработки запросов для всех endpoint'ов:

```text
Request
  → requestId + logging
  → authJwt (там где требуется)
  → validate (body/query)
  → controller
  → service (business / orchestration / при необходимости transaction)
  → repository (SQL)
  → response (JSON)
Error → errorHandler → JSON + logging
```

### 5.2.1. Что где писать (строгое разделение ответственности)

* **routes/** — только URL-таблица (маршруты), подключение декораторов и вызов controller.
  Назначение: связать HTTP-метод + путь с конкретным контроллером и middleware.
  Ограничения: никаких SQL, минимум логики.

* **controllers/** — слой HTTP-адаптера: извлечь входные данные (body/query/g.user), вызвать service, вернуть `{ok:true,data:...}`.
  Назначение: унифицировать формат ответа и разделить транспорт (HTTP) и бизнес-логику.
  Ограничения: никаких SQL, никаких транзакций.

* **services/** — бизнес-логика, проверки, orchestration нескольких репозиториев, правила обработки событий.
  Назначение: точка, где реализуются бизнес-решения (например, "если eventType=match_finish — выполнить итоговую обработку").
  Ограничения: транзакции допускаются только здесь (если нужны); контроллеры и репозитории транзакции не открывают.

* **repositories/** — доступ к данным: только SQL (`SELECT/INSERT/UPDATE/UPSERT`).
  Назначение: изолировать работу с БД и сделать SQL переиспользуемым.
  Ограничения: никакой бизнес-логики, никаких "правил игры", никаких расчётов наград.

* **middleware/** — инфраструктура: requestId/auth/validate/errorHandler.
  Назначение: единые правила для всех endpoint'ов (логирование, безопасность, валидация, ошибки).
  Ограничения: никакого SQL.

---

## 5.3. Обязательные правила проектирования REST

### 5.3.1. Ресурсный подход

* плохо: `/doLogin`, `/sendEventCommand`
* хорошо: `/api/v1/auth/login`, `/api/v1/events`, `/api/v1/profile`

Допустимое "действие" в игровых API — только как подресурс:

* `POST /api/v1/match/finish`
* `POST /api/v1/events`

---

### 5.3.2. Stateless

Каждый запрос несёт всё необходимое:

* `Authorization: Bearer <JWT>`
* JSON body / query params

Состояние игрока хранится в БД (`player_progress`, `matches`, `leaderboard_scores`, `statistics_daily`), а не в памяти процесса.

---

### 5.3.3. Единый JSON-контракт (обязателен для всех endpoint'ов)

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

## 5.4. HTTP коды (минимальный обязательный набор)

Должны реально использоваться:

* `200 OK` — успешно
* `201 Created` — создан ресурс
* `400 Bad Request` — ошибки валидации/формата
* `401 Unauthorized` — нет/невалиден JWT
* `403 Forbidden` — нет прав / пользователь забанен
* `404 Not Found` — ресурс не найден
* `409 Conflict` — конфликт состояния/уникальности / дублирующееся событие
* `429 Too Many Requests` — rate limit
* `500 Internal Server Error` — непредвиденная ошибка

---

# 5.5. Расширение структуры проекта под API v1

Если ранее присутствовал только один endpoint, для данного раздела требуется расширить проект по единой схеме слоёв: routes → controllers → services → repositories.

Структура проекта после расширения:

```text
app/
  config.py          ← конфигурация здесь, не в корне
  __init__.py        ← create_app() регистрирует все blueprint'ы
  extensions.py      ← db, jwt, cors
  routes/
    auth.py
    match.py
    profile.py
    events.py
    leaderboard.py
    health.py        ← GET / и GET /health
  controllers/
    auth_controller.py
    match_controller.py
    profile_controller.py
    events_controller.py
    leaderboard_controller.py
  services/
    __init__.py
    auth_service.py
    match_service.py
    profile_service.py
    events_service.py
    leaderboard_service.py
  repositories/
    events_repo.py
    progress_repo.py
    leaderboard_repo.py
    stats_repo.py
    users_repo.py
    roles_repo.py
    matches_repo.py
    sessions_repo.py ← работа с game_sessions
    dedupe_repo.py
  middleware/
    request_id.py
    auth_jwt.py
    validate_api.py
    error_handler.py
wsgi.py              ← точка входа
.env                 ← переменные окружения (не коммитить в Git!)
db/
  schema.sql         ← DDL: CREATE TABLE для всех таблиц
seed.sql             ← тестовые данные (10 пользователей, матчи, прогресс)
test_client.html     ← HTML-клиент для ручного тестирования в браузере
```

Важно: middleware остаётся общим для всех endpoint'ов (requestId/errorHandler), а `auth_required` и валидаторы подключаются там, где требуется.

---

# 5.6. Создание файлов API v1 и пример их наполнения (обязательный минимум)

Цель: чтобы все 5 MVP endpoint'ов были реализованы архитектурно корректно (routes/controllers/services/repositories) и выдавали единый контракт.

---

## 5.6.1. Регистрация blueprint'ов в `app/__init__.py`

Назначение: подключить все группы маршрутов к приложению, чтобы endpoints реально появились.

```python
from flask import Flask
from .extensions import db, jwt, cors
from .middleware.error_handler import register_error_handlers
from .middleware.request_id import register_request_id
from .routes.auth import auth_bp
from .routes.events import events_bp
from .routes.health import health_bp
from .routes.leaderboard import leaderboard_bp
from .routes.profile import profile_bp
from .routes.match import match_bp


def create_app(config=None):
    app = Flask(__name__)
    app.config.from_object(config or "app.config.Config")

    # Инициализация расширений (порядок важен: db → jwt → cors)
    db.init_app(app)
    jwt.init_app(app)
    cors.init_app(app, resources={r"/*": {"origins": "*"}})

    # Регистрация глобального middleware
    register_error_handlers(app)   # AppError + HTTPException + Exception → единый JSON
    register_request_id(app)       # X-Request-Id заголовок + логирование всех запросов

    # Регистрируем blueprint'ы, чтобы Flask начал обслуживать маршруты
    app.register_blueprint(auth_bp)
    app.register_blueprint(events_bp)
    app.register_blueprint(health_bp)
    app.register_blueprint(leaderboard_bp)
    app.register_blueprint(profile_bp)
    app.register_blueprint(match_bp)

    return app
```

Комментарий: если blueprint не зарегистрирован — соответствующий endpoint отсутствует в приложении и не может быть протестирован.

---

## 5.6.2. Routes (обязательные 5 endpoints)

Назначение файлов routes: определить URL, HTTP-метод и подключить нужные декораторы (валидация, JWT).

### `app/routes/auth.py`

```python
from flask import Blueprint
from app.controllers.auth_controller import register_controller, login_controller
from app.middleware.validate_api import validate_register, validate_login

# Blueprint для /api/v1/auth/*
# Назначение: логически сгруппировать маршруты регистрации и логина
auth_bp = Blueprint("auth", __name__, url_prefix="/api/v1/auth")


@auth_bp.post("/register")
@validate_register  # проверка тела запроса до попадания в сервис
def register():
    # Маршрут не содержит логики: делегирование в controller
    return register_controller()


@auth_bp.post("/login")
@validate_login  # проверка тела запроса до попадания в сервис
def login():
    return login_controller()
```

---

### `app/routes/profile.py`

```python
from flask import Blueprint
from app.controllers.profile_controller import profile_controller
from app.middleware.auth_jwt import auth_required

# Blueprint для /api/v1/*
# Назначение: вынести профиль в отдельный модуль маршрутов
profile_bp = Blueprint("profile", __name__, url_prefix="/api/v1")


@profile_bp.get("/profile")
@auth_required(roles=["player"])  # доступ только с JWT и ролью player
def profile():
    return profile_controller()
```

---

### `app/routes/events.py`

```python
from flask import Blueprint
from app.controllers.events_controller import post_event_controller
from app.middleware.auth_jwt import auth_required
from app.middleware.validate_api import validate_event

# Назначение: приём игровых событий единым endpoint'ом
events_bp = Blueprint("events", __name__, url_prefix="/api/v1")


@events_bp.post("/events")
@auth_required(roles=["player"])  # событие пишется только от аутентифицированного игрока
@validate_event                   # проверка структуры eventId/eventType/sessionId/clientTime/payload
def post_event():
    return post_event_controller()
```

---

### `app/routes/leaderboard.py`

```python
from flask import Blueprint
from app.controllers.leaderboard_controller import leaderboard_controller
from app.middleware.validate_api import validate_leaderboard_query

# Назначение: выдача таблицы лидеров с фильтрацией через query-параметры
# Примечание: leaderboard — публичный endpoint, auth_required не требуется
leaderboard_bp = Blueprint("leaderboard", __name__, url_prefix="/api/v1")


@leaderboard_bp.get("/leaderboard")
@validate_leaderboard_query             # проверка query: boardCode/season/limit
def get_leaderboard():
    return leaderboard_controller()
```

---

## 5.6.3. Controllers (HTTP слой, без SQL)

Назначение контроллеров: унифицировать ответы (`ok/data`) и передать управление сервисам.

### `app/controllers/auth_controller.py`

```python
from flask import request, jsonify
from app.services.auth_service import register_service, login_service

# Контроллеры не содержат SQL и не содержат бизнес-правил.
# Они принимают HTTP-вход, вызывают сервис и возвращают единый контракт.


def register_controller():
    # JSON тела запроса должен быть уже провалидирован validate_register
    payload = request.get_json()

    # Вся логика создания пользователя (bcrypt + роли + progress) находится в сервисе
    result = register_service(payload)

    # 201 Created — ресурс создан
    return jsonify({"ok": True, "data": result}), 201


def login_controller():
    payload = request.get_json()

    # Вся логика проверки bcrypt-пароля/выдачи токена находится в сервисе
    result = login_service(payload)

    return jsonify({"ok": True, "data": result}), 200
```

---

### `app/controllers/profile_controller.py`

```python
from flask import g, jsonify
from app.services.profile_service import profile_service

# Назначение: отдать витрину профиля.
# userId берётся только из JWT, извлечение реализует auth_required.


def profile_controller():
    user_id = g.user["userId"]

    result = profile_service(user_id)

    return jsonify({"ok": True, "data": result}), 200
```

---

### `app/controllers/events_controller.py`

```python
from flask import g, request, jsonify
from app.services.events_service import accept_event_service

# Назначение: принять событие и делегировать обработку сервису.
# SQL и бизнес-решения в контроллере запрещены.


def post_event_controller():
    user_id = g.user["userId"]
    payload = request.get_json()

    result = accept_event_service(user_id, payload)

    return jsonify({"ok": True, "data": result}), 200
```

---

### `app/controllers/leaderboard_controller.py`

```python
from flask import request, jsonify
from app.services.leaderboard_service import leaderboard_service

# Назначение: получить query-параметры и вызвать сервис.
# validate_leaderboard_query гарантирует корректность типов и диапазонов.


def leaderboard_controller():
    board_code = request.args.get("boardCode")
    season     = int(request.args.get("season", 0))
    limit      = int(request.args.get("limit", 50))

    result = leaderboard_service(board_code, season, limit)

    return jsonify({"ok": True, "data": result}), 200
```

---

## 5.6.4. Services (бизнес-логика + orchestration)

Назначение сервисов: реализовать правила поведения системы и управлять репозиториями.

### `app/services/auth_service.py`

```python
import bcrypt
from flask_jwt_extended import create_access_token
from app.extensions import db
from app.middleware.error_handler import AppError
from app.repositories.users_repo import find_user_by_email, create_user, get_user_roles
from app.repositories.roles_repo import assign_role_to_user
from app.repositories.progress_repo import ensure_progress_row

# Назначение:
# - register_service: создать пользователя (bcrypt-хэш), назначить роль player, создать progress
# - login_service: проверить bcrypt-пароль и выдать JWT с claims.roles


def register_service(payload: dict) -> dict:
    email    = payload["email"]
    password = payload["password"]
    nickname = payload["nickname"]

    # Проверка уникальности email
    if find_user_by_email(email) is not None:
        raise AppError(code="CONFLICT", message="Email already registered", http_status=409)

    # Хэширование пароля bcrypt
    hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt()).decode()

    try:
        user_id = create_user(db.session, email=email, password=hashed, nickname=nickname)
        assign_role_to_user(db.session, user_id=user_id, role_name="player")
        ensure_progress_row(db.session, user_id=user_id)
        db.session.commit()
    except Exception:
        db.session.rollback()
        raise

    return {"userId": user_id, "email": email, "nickname": nickname}


def login_service(payload: dict) -> dict:
    email    = payload["email"]
    password = payload["password"]

    user = find_user_by_email(email)
    if user is None:
        raise AppError(code="UNAUTHORIZED", message="Invalid credentials", http_status=401)

    # Проверка bcrypt-хэша
    if not bcrypt.checkpw(password.encode(), user["password"].encode()):
        raise AppError(code="UNAUTHORIZED", message="Invalid credentials", http_status=401)

    if user["is_banned"]:
        raise AppError(code="FORBIDDEN", message="Account is banned", http_status=403)

    # JWT claims: roles (для RBAC)
    roles = get_user_roles(user["id"])
    access_token = create_access_token(
        identity=user["id"],
        additional_claims={"roles": roles}
    )

    return {
        "accessToken": access_token,
        "userId": user["id"],
        "nickname": user["nickname"]
    }
```

---

### `app/services/profile_service.py`

```python
from app.middleware.error_handler import AppError
from app.repositories.users_repo import get_user_by_id, get_user_roles
from app.repositories.progress_repo import get_progress_by_user

# Назначение: собрать витрину профиля (user + roles + progress) из БД.
# Примечание: is_banned не проверяется здесь повторно — auth_jwt уже не пропустит
# заблокированного пользователя (если is_banned проверяется при логине).


def profile_service(user_id: int) -> dict:
    user = get_user_by_id(user_id)
    if user is None:
        raise AppError(code="NOT_FOUND", message="User not found", http_status=404)

    roles    = get_user_roles(user_id)
    progress = get_progress_by_user(user_id)

    return {
        "userId":   user["id"],
        "email":    user["email"],
        "nickname": user["nickname"],
        "roles":    roles,
        "progress": progress,
    }
```

---

### `app/services/events_service.py`

```python
from app.extensions import db
from app.repositories.dedupe_repo import register_event_or_conflict
from app.repositories.events_repo import insert_game_event

# Назначение: принять событие, зарегистрировать в processed_events (дедупликация) и записать в game_events.
# Требование: повторный eventId → 409 EVENT_REJECTED через UNIQUE(user_id, event_id) в processed_events.


def accept_event_service(user_id: int, payload: dict) -> dict:
    event_id    = payload["eventId"]
    event_type  = payload["eventType"]
    session_id  = payload.get("sessionId")
    event_payload = payload.get("payload") or {}

    try:
        # Дедупликация: UNIQUE(user_id, event_id) — повторный event_id → IntegrityError → 409
        register_event_or_conflict(db.session, user_id=user_id, event_id=event_id, event_type=event_type)
        # Запись события в game_events
        insert_game_event(db.session, user_id=user_id, session_id=session_id, event_type=event_type, payload=event_payload)
        db.session.commit()
    except Exception:
        db.session.rollback()
        raise

    return {"eventId": event_id, "status": "accepted"}
```

---

### `app/services/leaderboard_service.py`

```python
from app.repositories.leaderboard_repo import get_leaderboard

# Назначение: получить данные таблицы лидеров и привести к контракту API.


def leaderboard_service(board_code: str, season: int, limit: int) -> dict:
    items = get_leaderboard(board_code, season, limit)
    return {
        "boardCode": board_code,
        "season":    season,
        "items":     items
    }
```

---

## 5.6.5. Repositories (SQL слой, без бизнес-логики)

Назначение репозиториев: изолировать SQL и сделать операции с БД повторно используемыми.

### `app/repositories/users_repo.py`

```python
from sqlalchemy import text
from app.extensions import db

# Назначение:
# - find_user_by_email: получить пользователя по email (включая password для bcrypt-проверки)
# - create_user: создать пользователя
# - get_user_by_id: получить пользователя по id
# - get_user_roles: получить список ролей пользователя
# Ограничение: только SQL, никакой бизнес-логики.


def find_user_by_email(email: str):
    sql = text("SELECT id, email, password, nickname, is_banned FROM users WHERE email = :email")
    row = db.session.execute(sql, {"email": email}).mappings().first()
    if row is None:
        return None
    return {
        "id":        row["id"],
        "email":     row["email"],
        "password":  row["password"],
        "nickname":  row["nickname"],
        "is_banned": bool(row["is_banned"]),
    }


def create_user(session, email: str, password: str, nickname: str) -> int:
    sql = text("""
        INSERT INTO users (email, password, nickname, created_at, is_banned)
        VALUES (:email, :password, :nickname, NOW(3), 0)
    """)
    res = session.execute(sql, {"email": email, "password": password, "nickname": nickname})
    return res.lastrowid


def get_user_by_id(user_id: int):
    sql = text("SELECT id, email, nickname, is_banned FROM users WHERE id = :id")
    row = db.session.execute(sql, {"id": user_id}).mappings().first()
    if row is None:
        return None
    return {"id": row["id"], "email": row["email"], "nickname": row["nickname"], "is_banned": bool(row["is_banned"])}


def get_user_roles(user_id: int):
    sql = text("""
        SELECT r.name AS role_name
        FROM user_roles ur
        JOIN roles r ON r.id = ur.role_id
        WHERE ur.user_id = :user_id
    """)
    rows = db.session.execute(sql, {"user_id": user_id}).mappings().all()
    return [r["role_name"] for r in rows] or ["player"]
```

---

### `app/repositories/roles_repo.py`

```python
from sqlalchemy import text
from app.extensions import db

# Назначение: назначить пользователю роль (например, player).
# Ограничение: только SQL.


def assign_role_to_user(session, user_id: int, role_name: str):
    # Получение role_id по имени
    role_sql = text("SELECT id FROM roles WHERE name = :name")
    role = session.execute(role_sql, {"name": role_name}).mappings().first()

    if role is None:
        # Создание роли, если её нет
        ins_role = text("INSERT INTO roles (name) VALUES (:name)")
        res = session.execute(ins_role, {"name": role_name})
        role_id = res.lastrowid
    else:
        role_id = role["id"]

    # Привязка роли к пользователю (повторная привязка не должна ломать систему)
    link_sql = text("""
        INSERT INTO user_roles (user_id, role_id, assigned_at)
        VALUES (:user_id, :role_id, NOW())
        ON DUPLICATE KEY UPDATE assigned_at = assigned_at
    """)
    session.execute(link_sql, {"user_id": user_id, "role_id": role_id})
```

---

### `app/repositories/progress_repo.py` (дополнение для профиля)

```python
from sqlalchemy import text
from app.extensions import db

# Назначение: получить прогресс игрока для вывода в профиле.
# Ограничение: только SQL.


def get_progress_by_user(user_id: int) -> dict:
    sql = text("""
        SELECT level, xp, soft_currency, hard_currency
        FROM player_progress
        WHERE user_id = :user_id
    """)
    row = db.session.execute(sql, {"user_id": user_id}).mappings().first()
    if row is None:
        return {"level": 1, "xp": 0, "softCurrency": 0, "hardCurrency": 0}

    return {
        "level":        row["level"],
        "xp":           row["xp"],
        "softCurrency": row["soft_currency"],
        "hardCurrency": row["hard_currency"]
    }
```

---

### `app/repositories/events_repo.py`

```python
import json
from sqlalchemy import text
from app.extensions import db

# Назначение: записать событие в game_events.
# Ограничение: только SQL.


def insert_game_event(session, user_id: int, session_id, event_type: str, payload: dict):
    sql = text("""
        INSERT INTO game_events (user_id, session_id, event_type, payload_json)
        VALUES (:user_id, :session_id, :event_type, CAST(:payload AS JSON))
    """)
    session.execute(sql, {
        "user_id":    user_id,
        "session_id": session_id,
        "event_type": event_type,
        "payload":    json.dumps(payload, ensure_ascii=False)
    })
```

---

### `app/repositories/dedupe_repo.py`

```python
from sqlalchemy import text
from app.middleware.error_handler import AppError

# Назначение: регистрировать eventId в processed_events для защиты от дублей.
# UNIQUE(user_id, event_id) → IntegrityError при повторной отправке → 409 EVENT_REJECTED


def register_event_or_conflict(session, user_id: int, event_id: str, event_type: str):
    sql = text("""
        INSERT INTO processed_events (user_id, event_id, event_type, processed_at)
        VALUES (:user_id, :event_id, :event_type, NOW(3))
    """)
    try:
        session.execute(sql, {"user_id": user_id, "event_id": event_id, "event_type": event_type})
    except Exception:
        raise AppError(code="EVENT_REJECTED", message="Duplicate eventId", http_status=409)
```

---

### `app/repositories/leaderboard_repo.py` (дополнение для GET)

```python
from sqlalchemy import text
from app.extensions import db

# Назначение: выбрать топ игроков по board_code/season с ограничением limit.
# Ограничение: только SQL.


def get_leaderboard(board_code: str, season: int, limit: int):
    sql = text("""
        SELECT ls.user_id AS userId, u.nickname AS nickname, ls.score AS score
        FROM leaderboard_scores ls
        JOIN users u ON u.id = ls.user_id
        WHERE ls.board_code = :board_code AND ls.season = :season
        ORDER BY ls.score DESC
        LIMIT :limit
    """)
    rows = db.session.execute(sql, {"board_code": board_code, "season": season, "limit": limit}).mappings().all()

    items = []
    rank = 1
    for r in rows:
        items.append({"rank": rank, "userId": r["userId"], "nickname": r["nickname"], "score": r["score"]})
        rank += 1

    return items
```

---

## 5.6.6. Валидация для API endpoints (единый подход)

Файл валидаторов API: `app/middleware/validate_api.py`

Назначение: централизовать правила проверки входных данных до обращения к сервисам и БД.

Декораторы:

* `validate_register`
* `validate_login`
* `validate_event` — дополнительно проверяет `eventId` (UUID-строка, обязательный для дедупликации)
* `validate_leaderboard_query`
* `validate_match_finish`

```python
from functools import wraps
from flask import request
from app.middleware.error_handler import AppError


def _bad_body(details):
    raise AppError(code="VALIDATION_ERROR", message="Invalid request body", http_status=400, details=details)

def _bad_query(details):
    raise AppError(code="VALIDATION_ERROR", message="Invalid query params", http_status=400, details=details)


def validate_register(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        data = request.get_json(silent=True)
        if not isinstance(data, dict):
            _bad_body({"body": "JSON object expected"})

        email    = data.get("email")
        password = data.get("password")
        nickname = data.get("nickname")

        if not isinstance(email, str) or "@" not in email:
            _bad_body({"email": "valid email required"})
        if not isinstance(password, str) or len(password) < 6:
            _bad_body({"password": "string length >= 6 required"})
        if not isinstance(nickname, str) or len(nickname) < 2:
            _bad_body({"nickname": "string length >= 2 required"})

        return fn(*args, **kwargs)
    return wrapper


def validate_login(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        data = request.get_json(silent=True)
        if not isinstance(data, dict):
            _bad_body({"body": "JSON object expected"})

        email    = data.get("email")
        password = data.get("password")

        if not isinstance(email, str) or "@" not in email:
            _bad_body({"email": "valid email required"})
        if not isinstance(password, str) or len(password) < 1:
            _bad_body({"password": "string required"})

        return fn(*args, **kwargs)
    return wrapper


def validate_event(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        data = request.get_json(silent=True)
        if not isinstance(data, dict):
            _bad_body({"body": "JSON object expected"})

        event_id   = data.get("eventId")
        event_type = data.get("eventType")
        session_id = data.get("sessionId")
        client_time = data.get("clientTime")
        payload    = data.get("payload")

        # eventId обязателен — используется для дедупликации через processed_events
        if not isinstance(event_id, str) or len(event_id) < 8:
            _bad_body({"eventId": "uuid string required"})
        if not isinstance(event_type, str) or len(event_type) < 1:
            _bad_body({"eventType": "string required"})
        if session_id is not None and not isinstance(session_id, int):
            _bad_body({"sessionId": "int or null required"})
        if client_time is not None and not isinstance(client_time, str):
            _bad_body({"clientTime": "string or null required"})
        if payload is not None and not isinstance(payload, dict):
            _bad_body({"payload": "object or null required"})

        return fn(*args, **kwargs)
    return wrapper


def validate_leaderboard_query(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        board_code = request.args.get("boardCode")
        season     = request.args.get("season")
        limit      = request.args.get("limit")

        if not isinstance(board_code, str) or len(board_code) < 1:
            _bad_query({"boardCode": "string required"})
        if season is None or not season.isdigit():
            _bad_query({"season": "int required"})
        if limit is None or not limit.isdigit():
            _bad_query({"limit": "int required"})

        lim = int(limit)
        if lim < 1 or lim > 1000:
            _bad_query({"limit": "1..1000 required"})

        return fn(*args, **kwargs)
    return wrapper


def validate_match_finish(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        data = request.get_json(silent=True)
        if not isinstance(data, dict):
            _bad_body({"body": "JSON object expected"})

        match_id = data.get("matchId")
        result   = data.get("result")

        if not isinstance(match_id, int):
            _bad_body({"matchId": "int required"})
        if not isinstance(result, dict):
            _bad_body({"result": "object required"})

        is_win      = result.get("isWin")
        score       = result.get("score")
        duration    = result.get("durationSeconds")
        power_delta = result.get("powerDelta")

        if not isinstance(is_win, bool):
            _bad_body({"result.isWin": "bool required"})
        if not isinstance(score, int) or score < 0:
            _bad_body({"result.score": "int >= 0 required"})
        if not isinstance(duration, int) or duration < 0:
            _bad_body({"result.durationSeconds": "int >= 0 required"})
        if power_delta is not None and not isinstance(power_delta, int):
            _bad_body({"result.powerDelta": "int or null required"})

        return fn(*args, **kwargs)
    return wrapper
```

---

# 5.7. MVP endpoints — спецификация + примеры

## 5.7.1. `POST /api/v1/auth/register`

**Назначение:** создать аккаунт, хэшировать пароль bcrypt, назначить роль `player`, создать строку `player_progress`.

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
  "data": { "userId": 101, "email": "player1@example.com", "nickname": "PlayerOne" }
}
```

Ошибки:

* `400 VALIDATION_ERROR`
* `409 CONFLICT` — email уже зарегистрирован

---

## 5.7.2. `POST /api/v1/auth/login`

**Назначение:** проверить bcrypt-пароль и выдать JWT с claims.roles.

**Request**

```http
POST /api/v1/auth/login
Content-Type: application/json
```

```json
{
  "email": "alice@example.com",
  "password": "password123"
}
```

**Response (200)**

```json
{
  "ok": true,
  "data": {
    "accessToken": "<JWT>",
    "userId": 1,
    "nickname": "Commander_Alice"
  }
}
```

Ошибки:

* `401 UNAUTHORIZED` — неверный пароль или пользователь не найден
* `403 FORBIDDEN` — пользователь заблокирован (`is_banned = 1`)

---

## 5.7.3. `GET /api/v1/profile`

**Назначение:** отдать \"витрину\" профиля: userId, email, nickname, roles, progress.

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
    "userId":   1,
    "email":    "alice@example.com",
    "nickname": "Commander_Alice",
    "roles":    ["player", "admin"],
    "progress": { "level": 42, "xp": 85400, "softCurrency": 12500, "hardCurrency": 250 }
  }
}
```

Ошибки:

* `401 UNAUTHORIZED`
* `404 NOT_FOUND` — пользователь удалён из БД после выдачи токена

---

## 5.7.4. `POST /api/v1/events`

**Назначение:** записать событие в `game_events` с дедупликацией через `processed_events` по `eventId`.

**Request**

```http
POST /api/v1/events
Authorization: Bearer <JWT>
Content-Type: application/json
```

```json
{
  "eventId": "evt-a1b2c3d4-e5f6-7890",
  "eventType": "unit_deploy",
  "sessionId": 1,
  "clientTime": "2026-02-27T12:20:10.000Z",
  "payload": {
    "matchId": 9,
    "unitCode": "cavalry",
    "amount": 40,
    "atSecond": 15
  }
}
```

**Response (200)**

```json
{
  "ok": true,
  "data": { "eventId": "evt-a1b2c3d4-e5f6-7890", "status": "accepted" }
}
```

Ошибки:

* `400 VALIDATION_ERROR`
* `409 EVENT_REJECTED` — повторный `eventId`

---

## 5.7.5. `GET /api/v1/leaderboard`

**Назначение:** вернуть топ игроков по заданной доске и сезону. Endpoint публичный — JWT не требуется.

**Request**

```http
GET /api/v1/leaderboard?boardCode=default&season=1&limit=10
```

**Response (200)**

```json
{
  "ok": true,
  "data": {
    "boardCode": "default",
    "season": 1,
    "items": [
      { "rank": 1, "userId": 5, "nickname": "Siege_Eve",    "score": 9850 },
      { "rank": 2, "userId": 3, "nickname": "WarLord_Carol", "score": 8720 }
    ]
  }
}
```

Ошибки:

* `400 VALIDATION_ERROR` (невалидные query-параметры: отсутствует boardCode, season или limit выходит за диапазон 1–1000)

---

# 5.8. Требование к воспроизводимости проверок (Postman / test_client.html)

> Пошаговые инструкции по Postman — в разделе **5.12**. Инструкция по `test_client.html` — в разделе **4.9** документа `backend_implementation.md`.

Должно быть возможно проверить:

* `201` при регистрации
* `409` при повторной регистрации того же email
* `200` при логине и получение `accessToken`
* `401` при логине с неверным паролем
* `403` при логине заблокированного пользователя (`banned@example.com` из `seed.sql`)
* `401` при доступе к /profile без JWT
* `400` при неверных параметрах /leaderboard или неверном JSON /events
* `409` при повторной отправке события с тем же `eventId`
* `500` при искусственной ошибке сервера

> Для ручного тестирования в браузере используйте `test_client.html` из корня проекта.

---

# 5.9. Требования к обработке события и match_finish (обязательные усиления)

Событие `match_finish` обрабатывается через `POST /api/v1/match/finish`:

1. **Идемпотентность**:
   * статус матча `started` — гарантирует, что повторный вызов вернёт `409 CONFLICT`
   * дополнительно: `processed_events` с `UNIQUE(user_id, event_id)` защищает `/events` от дублей

2. **Ownership/anti-cheat**:
   * `matchId` должен принадлежать `userId` из JWT — проверяется в `ensure_match_ownership_started()`
   * статус матча должен быть `started`

3. **Блокировки**:
   * `SELECT ... FOR UPDATE` на `player_progress` в `lock_progress_row()` (обязательно)

4. **Атомарность**:
   * сбой на любом шаге → `db.session.rollback()`

---

# 5.10. Swagger / OpenAPI (если подключаете)

## 5.10.1. Что должно быть описано

* все MVP endpoints
* схемы запросов/ответов (включая `ErrorResponse`)
* security scheme JWT bearer
* ответы `401/403/409` там, где они возможны

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

# 5.11. ⚠️ Напоминание: заполнение базы данных перед работой

> **Без выполнения этого шага API не будет работать корректно.** Таблицы окажутся пустыми, логин с тестовыми данными не пройдёт, лидерборд вернёт пустой список, матч с id=9 не будет найден.

Порядок заполнения БД обязателен и выполняется **один раз** после создания базы данных.

## 5.11.1. Что нужно выполнить и в каком порядке

### Шаг 1. Создать базу данных (если ещё не создана)

Подключиться к MySQL и выполнить:

```sql
CREATE DATABASE IF NOT EXISTS strategy_db
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;
```

> Имя базы, хост, пользователь и пароль должны совпадать с тем, что прописано в `.env`.

---

### Шаг 2. Применить схему (`db/schema.sql`)

`db/schema.sql` содержит все `CREATE TABLE` — структуру всех таблиц. Без этого файла БД пустая и Python-код упадёт с ошибкой «Table not found».

```bash
# Из корня проекта:
mysql -u root -p strategy_db < db/schema.sql
```

**Что создаётся этим файлом:**

| Таблица | Назначение |
|---|---|
| `roles` | Роли: player, moderator, admin |
| `users` | Аккаунты игроков (email, bcrypt-пароль, nickname) |
| `user_roles` | Связь пользователь ↔ роль |
| `game_sessions` | Игровые сессии (платформа, версия клиента) |
| `game_events` | Поток игровых событий с JSON-payload |
| `player_progress` | Уровень, XP, валюта каждого игрока |
| `statistics_daily` | Агрегированная статистика по дням |
| `leaderboard_scores` | Рейтинговые очки (board_code, season) |
| `matches` | Матчи (pve/pvp, карта, статус) |
| `battle_results` | Итоги матча 1:1 с matches |
| `army_units` | Армия пользователя (тип юнита, количество) |
| `city_buildings` | Здания города (тип, уровень, статус) |
| `processed_events` | Реестр обработанных eventId (дедупликация) |

---

### Шаг 3. Залить тестовые данные (`seed.sql`)

`seed.sql` содержит тестовых пользователей, матчи, прогресс, лидерборд — всё что нужно для тестирования API прямо после запуска.

```bash
# Из корня проекта:
mysql -u root -p strategy_db < seed.sql
```

**Что создаётся тестовыми данными:**

| Данные | Подробности |
|---|---|
| **10 пользователей** | alice, bob, carol, dave, eve, frank, grace, hank, ivan, banned |
| **Роли** | alice — player+admin, bob — player+moderator, остальные — player, banned — player (is_banned=1) |
| **Прогресс** | Уровень, XP, soft/hard валюта для всех 10 пользователей |
| **Лидерборд** | 10 записей на `board_code=default, season=1`; 5 записей на `board_code=pvp, season=1` |
| **Матчи** | 10 матчей: 8 завершённых, 1 в статусе `started` (id=9, owner=alice), 1 заброшенный |
| **Результаты матчей** | `battle_results` для 9 завершённых матчей |
| **Игровые сессии** | 6 сессий от разных пользователей |
| **Армия и здания** | `army_units` и `city_buildings` для 5 игроков |
| **Статистика** | `statistics_daily` за последние 7 дней для топ-игроков |
| **Processed events** | 5 записей дедупликации в `processed_events` |

> **Пароль у всех тестовых пользователей:** `password123`
>
> **Матч id=9** — в статусе `started`, принадлежит alice (user_id=1). Именно его используют для тестирования `POST /api/v1/match/finish`.

---

### Шаг 4. Проверить, что данные залиты

```sql
USE strategy_db;
SELECT COUNT(*) FROM users;           -- должно быть 10
SELECT COUNT(*) FROM player_progress; -- должно быть 10
SELECT COUNT(*) FROM leaderboard_scores; -- должно быть 15
SELECT id, user_id, status FROM matches WHERE id = 9; -- должна быть строка status=started
```

---

### Шаг 5. Файл `.env` — подключение к БД

В корне проекта должен быть файл `.env` со строкой подключения к MySQL. Без него Flask не знает, к какой БД подключаться.

```text
SQLALCHEMY_DATABASE_URI=mysql+pymysql://root@localhost:3306/strategy_db?charset=utf8mb4
DATABASE_URL=mysql+pymysql://root@localhost:3306/strategy_db?charset=utf8mb4
JWT_SECRET_KEY=change-jwt-secret-in-production
SECRET_KEY=change-me-in-production
```

> Если у вашего MySQL есть пароль, формат: `mysql+pymysql://root:ПАРОЛЬ@localhost:3306/strategy_db?charset=utf8mb4`

---

### Полный порядок развёртывания (сводка)

```bash
# 1. Перейти в папку проекта
cd C:\Users\gvadoskr\Desktop\project\strategy-support-is

# 2. Активировать виртуальное окружение
venv\Scripts\activate

# 3. Установить зависимости (если не установлены)
pip install Flask flask-jwt-extended flask-sqlalchemy flask-cors pymysql python-dotenv bcrypt

# 4. Проверить / создать .env файл с параметрами БД

# 5. Применить схему БД
mysql -u root -p strategy_db < db/schema.sql

# 6. Залить тестовые данные
mysql -u root -p strategy_db < seed.sql

# 7. Запустить сервер
python wsgi.py
```

После этого сервер доступен по адресу `http://127.0.0.1:5000`.

---

# 5.12. Установка Postman и проверка API

## 5.12.1. Что такое Postman и зачем он нужен

**Postman** — это инструмент для тестирования HTTP API. Позволяет отправлять запросы к серверу, просматривать ответы, сохранять запросы в коллекции и автоматизировать проверки. В отличие от `test_client.html`, Postman предоставляет расширенные возможности: переменные среды, цепочки запросов, скрипты авто-тестов.

## 5.12.2. Установка Postman

### Шаг 1. Скачать Postman

Перейти на официальный сайт:

```
https://www.postman.com/downloads/
```

Выбрать версию для Windows и нажать **Download**.

### Шаг 2. Установить

Запустить скачанный установщик `Postman-win64-Setup.exe`. Установка стандартная — Next → Install → Finish.

### Шаг 3. Запустить и пропустить регистрацию

При первом запуске появится окно регистрации. Внизу есть ссылка **«Skip and go to the app»** — нажать её. Регистрация не обязательна для базового использования.

---

## 5.12.3. Настройка переменной `base_url`

Чтобы не вводить адрес сервера в каждый запрос вручную, настроим переменную окружения.

### Создать Environment

1. В левом сайдбаре нажать **Environments** (иконка глаза или раздел в меню).
2. Нажать **+** (Add Environment).
3. Назвать среду: `strategy-local`.
4. Добавить переменную:
   - **Variable:** `base_url`
   - **Initial value:** `http://127.0.0.1:5000`
   - **Current value:** `http://127.0.0.1:5000`
5. Нажать **Save**.
6. В правом верхнем углу выбрать среду `strategy-local` из выпадающего списка.

Теперь в запросах можно писать `{{base_url}}/api/v1/auth/login` вместо полного адреса.

---

## 5.12.4. Создание коллекции запросов

### Создать коллекцию

1. В левом сайдбаре нажать **Collections → +** (New Collection).
2. Назвать: `Strategy IS API`.
3. Нажать **Create**.

### Добавить папки по группам

Внутри коллекции создать папки (`Add a folder`):
- `Auth`
- `Profile`
- `Leaderboard`
- `Events`
- `Match`
- `System`

---

## 5.12.5. Проверка каждого endpoint'а в Postman

### Запрос 1: Health Check

**Цель:** убедиться, что сервер запущен и отвечает.

1. В папке `System` нажать **Add a request**.
2. Назвать: `Health Check`.
3. Метод: `GET`.
4. URL: `{{base_url}}/health`
5. Нажать **Send**.

**Ожидаемый ответ:**
```json
{ "ok": true, "data": { "status": "ok" } }
```
Статус: `200 OK`.

---

### Запрос 2: Регистрация

**Цель:** создать нового пользователя.

1. В папке `Auth` создать запрос `Register`.
2. Метод: `POST`.
3. URL: `{{base_url}}/api/v1/auth/register`
4. Вкладка **Body** → выбрать `raw` → выбрать `JSON`.
5. Тело запроса:

```json
{
  "email":    "testplayer@example.com",
  "password": "password123",
  "nickname": "TestPlayer"
}
```

6. Нажать **Send**.

**Ожидаемый ответ:**
```json
{
  "ok": true,
  "data": { "userId": 11, "email": "testplayer@example.com", "nickname": "TestPlayer" }
}
```
Статус: `201 Created`.

**Проверка ошибки 409:** отправить тот же запрос ещё раз — придёт `409 CONFLICT` (email уже занят).

---

### Запрос 3: Логин

**Цель:** получить JWT-токен для защищённых запросов.

1. В папке `Auth` создать запрос `Login`.
2. Метод: `POST`.
3. URL: `{{base_url}}/api/v1/auth/login`
4. Body → raw → JSON:

```json
{
  "email":    "alice@example.com",
  "password": "password123"
}
```

5. Нажать **Send**.

**Ожидаемый ответ:**
```json
{
  "ok": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "userId":      1,
    "nickname":    "Commander_Alice"
  }
}
```
Статус: `200 OK`.

### Сохранить токен автоматически (скрипт)

Чтобы токен автоматически сохранялся в переменную и подставлялся в следующие запросы:

1. На вкладке `Login` перейти на вкладку **Scripts** (ранее — Tests).
2. Вставить скрипт:

```javascript
const resp = pm.response.json();
if (resp.ok && resp.data.accessToken) {
    pm.environment.set("jwt_token", resp.data.accessToken);
    console.log("Token saved:", resp.data.accessToken.slice(0, 30) + "...");
}
```

3. Нажать **Save**.

Теперь после каждого логина токен автоматически записывается в переменную `{{jwt_token}}`.

**Проверка ошибки 401:** отправить с неверным паролем — придёт `401 UNAUTHORIZED`.

**Проверка ошибки 403:** заменить email на `banned@example.com` — придёт `403 FORBIDDEN`.

---

### Запрос 4: Профиль

**Цель:** получить данные профиля авторизованного пользователя.

1. В папке `Profile` создать запрос `My Profile`.
2. Метод: `GET`.
3. URL: `{{base_url}}/api/v1/profile`
4. Вкладка **Auth** → тип `Bearer Token` → в поле Token: `{{jwt_token}}`.
5. Нажать **Send**.

**Ожидаемый ответ:**
```json
{
  "ok": true,
  "data": {
    "userId":   1,
    "email":    "alice@example.com",
    "nickname": "Commander_Alice",
    "roles":    ["player", "admin"],
    "progress": {
      "level":        42,
      "xp":           85400,
      "softCurrency": 12500,
      "hardCurrency": 250
    }
  }
}
```
Статус: `200 OK`.

**Проверка ошибки 401:** убрать токен (вкладка Auth → No Auth) — придёт `401 UNAUTHORIZED`.

---

### Запрос 5: Лидерборд

**Цель:** получить топ игроков. Endpoint публичный, JWT не требуется.

1. В папке `Leaderboard` создать запрос `Get Leaderboard`.
2. Метод: `GET`.
3. URL: `{{base_url}}/api/v1/leaderboard`
4. Вкладка **Params** добавить параметры:

| Key | Value |
|---|---|
| boardCode | default |
| season | 1 |
| limit | 10 |

5. Нажать **Send**.

**Ожидаемый ответ:**
```json
{
  "ok": true,
  "data": {
    "boardCode": "default",
    "season": 1,
    "items": [
      { "rank": 1, "userId": 5, "nickname": "Siege_Eve",     "score": 9850 },
      { "rank": 2, "userId": 3, "nickname": "WarLord_Carol", "score": 8720 }
    ]
  }
}
```
Статус: `200 OK`.

**Проверка `boardCode=pvp`:** заменить значение параметра — вернётся другая доска.

**Проверка ошибки 400:** удалить параметр `boardCode` — придёт `400 VALIDATION_ERROR`.

---

### Запрос 6: Отправка события

**Цель:** записать игровое событие с дедупликацией по `eventId`.

1. В папке `Events` создать запрос `Post Event`.
2. Метод: `POST`.
3. URL: `{{base_url}}/api/v1/events`
4. Auth → Bearer Token → `{{jwt_token}}`.
5. Body → raw → JSON:

```json
{
  "eventId":   "evt-test-001-unique",
  "eventType": "unit_deploy",
  "sessionId": null,
  "clientTime": "2026-02-27T12:00:00.000Z",
  "payload": {
    "matchId":  9,
    "unitCode": "cavalry",
    "amount":   40,
    "atSecond": 15
  }
}
```

6. Нажать **Send**.

**Ожидаемый ответ:**
```json
{
  "ok": true,
  "data": { "eventId": "evt-test-001-unique", "status": "accepted" }
}
```
Статус: `200 OK`.

**Проверка дедупликации 409:** нажать **Send** ещё раз с тем же `eventId` — придёт `409 EVENT_REJECTED`. Для нового успешного запроса нужно изменить `eventId` (например, на `evt-test-002-unique`).

**Проверка ошибки 400:** удалить поле `eventId` из тела — придёт `400 VALIDATION_ERROR`.

---

### Запрос 7: Завершение матча

**Цель:** завершить матч id=9 (принадлежит alice, статус `started`) и получить начисленные награды.

> ⚠️ Матч id=9 имеет статус `started` только один раз — после первого успешного вызова он перейдёт в `finished`. Для повторного тестирования нужно вернуть статус вручную через MySQL: `UPDATE matches SET status='started', ended_at=NULL WHERE id=9;`

1. В папке `Match` создать запрос `Finish Match`.
2. Метод: `POST`.
3. URL: `{{base_url}}/api/v1/match/finish`
4. Auth → Bearer Token → `{{jwt_token}}`.
5. Body → raw → JSON:

```json
{
  "matchId": 9,
  "result": {
    "isWin":           true,
    "score":           1850,
    "durationSeconds": 1200
  }
}
```

6. Нажать **Send**.

**Ожидаемый ответ:**
```json
{
  "ok": true,
  "data": {
    "matchId":             9,
    "isWin":               true,
    "xpGained":            100,
    "softCurrencyGained":  50
  }
}
```
Статус: `200 OK`.

**Проверка ошибки 409:** нажать **Send** ещё раз — придёт `409 CONFLICT` (матч уже в статусе `finished`).

**Проверка ошибки 403:** залогиниться как bob и попробовать завершить матч id=9 — придёт `403 FORBIDDEN` (матч принадлежит alice).

---

## 5.12.6. Сводная таблица запросов для коллекции Postman

| Папка | Имя запроса | Метод | URL | Auth | Тело |
|---|---|---|---|---|---|
| System | Health Check | GET | `{{base_url}}/health` | — | — |
| Auth | Register | POST | `{{base_url}}/api/v1/auth/register` | — | JSON |
| Auth | Login | POST | `{{base_url}}/api/v1/auth/login` | — | JSON |
| Profile | My Profile | GET | `{{base_url}}/api/v1/profile` | Bearer `{{jwt_token}}` | — |
| Leaderboard | Get Leaderboard | GET | `{{base_url}}/api/v1/leaderboard` | — | Query params |
| Events | Post Event | POST | `{{base_url}}/api/v1/events` | Bearer `{{jwt_token}}` | JSON |
| Match | Finish Match | POST | `{{base_url}}/api/v1/match/finish` | Bearer `{{jwt_token}}` | JSON |

---

## 5.12.7. Экспорт и импорт коллекции

Чтобы передать коллекцию другому разработчику или сохранить резервную копию:

**Экспорт:**
1. В сайдбаре найти коллекцию `Strategy IS API`.
2. Нажать `...` → **Export**.
3. Выбрать формат `Collection v2.1`.
4. Сохранить файл `strategy-is-api.postman_collection.json`.

**Импорт:**
1. В Postman нажать **Import** (кнопка рядом с New).
2. Перетащить JSON-файл или выбрать через диалог.
3. Коллекция появится в сайдбаре.

---

## 5.12.8. Альтернатива Postman: `test_client.html`

Если Postman устанавливать не хочется, в корне проекта есть готовый HTML-клиент:

```
strategy-support-is/test_client.html
```

Открыть двойным кликом в браузере (Chrome, Firefox, Edge). Не требует установки — работает как локальный HTML-файл.

Возможности:
- Все те же 7 endpoint'ов в одном интерфейсе
- Автоматическое сохранение JWT после логина
- Quick Fill кнопки с готовыми тестовыми данными
- Лог всех запросов с HTTP-статусами и временем выполнения
- Подсветка JSON-ответов

Подробное описание интерфейса и каждого элемента — в разделе **4.9** документа `backend_implementation.md`.
