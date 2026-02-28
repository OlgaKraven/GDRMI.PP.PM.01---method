# 2. Практика. 4. Реализация backend'а ИС сопровождения игрового продукта (Flask)

## 4.1. Задание к разделу 4 «Backend-логика»

### Необходимо выполнить

1. Реализовать middleware (для Flask — через `before_request/after_request`, декораторы и error handlers):

   * `requestId`
   * `authJwt`
   * `validate`
   * `errorHandler`

2. Реализовать **транзакционную операцию** для "итогового события" (например, `match_finish`) в **одной транзакции**:

   * проверка владения матчем и статуса
   * запись результата в `battle_results`
   * обновление `player_progress`
   * upsert в `leaderboard_scores`
   * upsert в `statistics_daily`
   * commit / rollback

### Артефакт

Код backend (структура папок обязательна):

```text
app/
  routes/
  controllers/
  services/
  repositories/
  middleware/
  config.py      ← конфигурация внутри app/, не в корне
wsgi.py          ← точка входа
```

---

## 4.2. Минимальная рабочая архитектура backend (обязательная логика)

Поток обработки запроса соответствует конвейеру:

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

Принцип распределения ответственности (обязателен для сдачи):

* **middleware**: технический слой (requestId, auth, validate, error), без SQL и бизнес-правил.
* **controller**: принимает вход, вызывает сервис, возвращает JSON. SQL запрещён.
* **service**: бизнес-логика + транзакция + orchestration репозиториев. Именно тут одна TX.
* **repository**: только SQL/операции с БД. Без расчётов наград и правил.

---

## 4.3. Минимальные endpoints для раздела 4 (рекомендуемый набор)

Для сдачи раздела достаточно реализовать один "итоговый" endpoint:

* `POST /api/v1/match/finish`

В данном минимальном примере реализуется: `POST /api/v1/match/finish`.

---

## 4.4. Реализация проекта (Flask) — структура и файлы

### 4.4.1. Структура проекта (Flask)

```text
project-root/
│
├── app/
│   ├── __init__.py       ← create_app(), регистрация blueprint'ов и middleware
│   ├── config.py         ← class Config (читает .env через python-dotenv)
│   ├── extensions.py     ← db, jwt, cors
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
│   │   ├── matches_repo.py
│   │   ├── progress_repo.py
│   │   ├── leaderboard_repo.py
│   │   └── stats_repo.py
│   │
│   └── middleware/
│       ├── request_id.py
│       ├── auth_jwt.py
│       ├── validate_api.py
│       └── error_handler.py
│
├── seed.sql
├── db/
│   └── schema.sql
└── wsgi.py               ← точка входа (python wsgi.py)
```

> **Конфигурация** находится в `app/config.py`. Файлов `config.py` в корне и `run.py` **нет** — используется `wsgi.py`.

---

## 4.4.2. Создание файлов проекта и пример их наполнения (обязательный минимум)

> Цель: получить минимально работоспособный backend, где один endpoint (`POST /api/v1/match/finish`) проходит весь конвейер:
> `requestId → authJwt → validate → controller → service (TX) → repositories (SQL) → JSON response`,
> а ошибки всегда возвращаются единообразно.

---

# A) Конфигурация

## A.1. `app/config.py`

```python
import os
from dotenv import load_dotenv

# Загружаем переменные окружения из .env (если файл есть)
load_dotenv()


class Config:
    # Flask
    SECRET_KEY = os.environ.get("SECRET_KEY", "change-me-in-production")

    # MySQL + SQLAlchemy
    # Пример: mysql+pymysql://root@localhost:3306/strategy_db?charset=utf8mb4
    SQLALCHEMY_DATABASE_URI = os.environ.get(
        "DATABASE_URL",
        os.environ.get("SQLALCHEMY_DATABASE_URI", "sqlite:///dev.db")
    )
    SQLALCHEMY_TRACK_MODIFICATIONS = False

    # JWT
    # В реальном проекте ключ обязательно хранить в .env / секретах
    JWT_SECRET_KEY = os.environ.get("JWT_SECRET_KEY", "change-jwt-secret-in-production")
    JWT_ACCESS_TOKEN_EXPIRES = int(os.environ.get("JWT_ACCESS_TOKEN_EXPIRES", 3600))
```

> Рекомендуется создать `.env` в корне проекта:
>
> ```text
> SQLALCHEMY_DATABASE_URI=mysql+pymysql://root@localhost:3306/strategy_db?charset=utf8mb4
> DATABASE_URL=mysql+pymysql://root@localhost:3306/strategy_db?charset=utf8mb4
> JWT_SECRET_KEY=my-super-secret-jwt-key-change-in-prod
> SECRET_KEY=my-flask-secret-key
> DEBUG=1
> ```

---

## A.2. `wsgi.py`

```python
from app import create_app

# Создание Flask-приложения (регистрация middleware и маршрутов)
app = create_app()

if __name__ == "__main__":
    # Здесь НЕ пишем бизнес-логику и НЕ пишем SQL.
    # Файл отвечает только за запуск сервера.
    app.run(host="0.0.0.0", port=5000, debug=True)
```

---

# B) app/

## B.1. `app/extensions.py`

```python
from flask_sqlalchemy import SQLAlchemy
from flask_jwt_extended import JWTManager
from flask_cors import CORS

# db — единая точка доступа к БД (SQLAlchemy)
db = SQLAlchemy()

# jwt — менеджер JWT (проверка токена, извлечение identity и claims)
jwt = JWTManager()

# cors — поддержка CORS для Unity/Web-клиентов
cors = CORS()
```

---

## B.2. `app/__init__.py`

```python
from flask import Flask

from .extensions import db, jwt, cors
from .middleware.request_id import register_request_id
from .middleware.error_handler import register_error_handlers
from .routes.match import match_bp


def create_app(config=None) -> Flask:
    # 1) Создаём Flask-приложение
    app = Flask(__name__)

    # 2) Загружаем конфигурацию из app/config.py
    app.config.from_object(config or "app.config.Config")

    # 3) Инициализируем расширения (db, jwt, cors)
    db.init_app(app)
    jwt.init_app(app)
    cors.init_app(app, resources={r"/*": {"origins": "*"}})

    # 4) Регистрируем middleware и обработчики ошибок
    register_request_id(app)
    register_error_handlers(app)

    # 5) Регистрируем маршруты (Blueprint)
    app.register_blueprint(match_bp)

    return app
```

---

# C) app/middleware/

> В Flask "middleware" реализуется так:
>
> * `@app.before_request` / `@app.after_request` — requestId + логирование
> * декораторы над view-функциями — authJwt, validate
> * `@app.errorhandler(...)` — errorHandler (единый JSON-формат ошибок)

---

## C.1. `app/middleware/request_id.py`

### `requestId` middleware (Flask)

Требования:

* генерирует `requestId` на каждый запрос
* добавляет в ответ заголовок `X-Request-Id`
* логирует `requestId`, метод, путь, статус, duration

Минимальный формат лога:

```text
req_abc123 user=15 POST /api/v1/match/finish 200 18ms
```

#### Реализация (как должно работать)

* В `before_request`:

  * если клиент прислал `X-Request-Id`, использовать его
  * иначе сгенерировать `req_<shortid>`
  * сохранить в `g.request_id`
  * сохранить старт времени `g._start_ms`
* В `after_request`:

  * добавить `X-Request-Id: <g.request_id>`
  * вычислить duration
  * залогировать строку

```python
import time
import uuid
from flask import g, request


def _gen_request_id() -> str:
    # Генерируем короткий requestId для логирования и диагностики
    return "req_" + uuid.uuid4().hex[:8]


def register_request_id(app):
    @app.before_request
    def _before():
        # Если клиент прислал X-Request-Id — используем его (удобно для трассировки цепочки)
        # Иначе генерируем новый id
        g.request_id = request.headers.get("X-Request-Id") or _gen_request_id()

        # Стартовое время для подсчёта длительности запроса
        g._start_ms = int(time.time() * 1000)

    @app.after_request
    def _after(response):
        # Длительность запроса
        duration_ms = int(time.time() * 1000) - getattr(g, "_start_ms", int(time.time() * 1000))

        # Возвращаем requestId клиенту
        response.headers["X-Request-Id"] = getattr(g, "request_id", "-")

        # userId может быть установлен authJwt middleware (через g.user)
        user_id = "-"
        if getattr(g, "user", None) and isinstance(g.user, dict):
            user_id = g.user.get("userId", "-")

        # Лог по требуемому формату:
        # req_abc123 user=15 POST /api/v1/match/finish 200 18ms
        app.logger.info(
            "%s user=%s %s %s %s %sms",
            getattr(g, "request_id", "-"),
            user_id,
            request.method,
            request.path,
            response.status_code,
            duration_ms,
        )
        return response
```

---

## C.2. `app/middleware/error_handler.py`

### `errorHandler` middleware (Flask)

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

#### Реализация (как должно работать)

* Создать класс `AppError(Exception)`:

  * `code`, `http_status`, `message`, `details(optional)`
* `@app.errorhandler(AppError)`:

  * вернуть JSON-ошибку в едином формате
* `@app.errorhandler(Exception)`:

  * вернуть `500 INTERNAL_ERROR`
  * в лог записать stack trace + requestId + userId

```python
from dataclasses import dataclass
from typing import Any, Optional
from flask import jsonify, g


@dataclass
class AppError(Exception):
    # Контролируемая ошибка приложения:
    # code — машинный код для клиента
    # message — человекочитаемое сообщение
    # http_status — HTTP статус ответа
    # details — опциональные детали (например поля валидации)
    code: str
    message: str
    http_status: int = 400
    details: Optional[Any] = None


def _error_response(code: str, message: str, request_id: str, details=None):
    # Единый формат error-ответов по заданию
    body = {
        "ok": False,
        "error": {
            "code": code,
            "message": message,
            "requestId": request_id
        }
    }
    if details is not None:
        body["error"]["details"] = details
    return body


def register_error_handlers(app):
    @app.errorhandler(AppError)
    def handle_app_error(err: AppError):
        rid = getattr(g, "request_id", "-")

        # Логируем контролируемую ошибку (без stacktrace, обычно достаточно кода/сообщения)
        app.logger.warning("AppError %s rid=%s msg=%s", err.code, rid, err.message)

        return jsonify(_error_response(err.code, err.message, rid, err.details)), err.http_status

    @app.errorhandler(Exception)
    def handle_exception(err: Exception):
        rid = getattr(g, "request_id", "-")

        # Логируем неконтролируемую ошибку со stacktrace
        app.logger.exception("Unhandled error rid=%s", rid)

        return jsonify(_error_response("INTERNAL_ERROR", "Unexpected error", rid)), 500
```

---

## C.3. `app/middleware/auth_jwt.py`

### `authJwt` middleware (Flask)

Требования:

* читает `Authorization: Bearer <JWT>`
* валидирует токен
* кладёт в `g.user`:

  * `userId`
  * `roles`

Ошибки:

* `401 Unauthorized` — токен отсутствует/невалиден/истёк
* `403 Forbidden` — роль не подходит (если проверяете RBAC)

#### Реализация (как должно работать)

* Используется декоратор `@auth_required(roles=[...])`
* Декоратор:

  * извлекает Bearer token (это делает `verify_jwt_in_request()` внутри `flask-jwt-extended`)
  * декодирует токен
  * забирает identity (user_id) и roles (из claims)
  * кладёт в `g.user = {"userId": ..., "roles": [...]}`

```python
from functools import wraps
from flask import g
from flask_jwt_extended import verify_jwt_in_request, get_jwt, get_jwt_identity

from .error_handler import AppError


def auth_required(roles=None):
    # roles — список ролей, которые допускаются к endpoint
    roles = roles or []

    def decorator(fn):
        @wraps(fn)
        def wrapper(*args, **kwargs):
            try:
                # Проверяем токен (валидность, срок жизни)
                verify_jwt_in_request()
            except Exception:
                # Любая ошибка JWT → 401
                raise AppError(code="UNAUTHORIZED", message="JWT is missing/invalid/expired", http_status=401)

            # user_id берём из identity токена
            user_id = get_jwt_identity()

            # Дополнительные данные токена (claims)
            claims = get_jwt() or {}
            user_roles = claims.get("roles", [])

            # Если endpoint требует роли — проверяем RBAC
            if roles:
                ok = any(r in user_roles for r in roles)
                if not ok:
                    raise AppError(code="FORBIDDEN", message="Insufficient role", http_status=403)

            # Кладём пользователя в g (используется сервисами/логированием)
            g.user = {"userId": user_id, "roles": user_roles}

            return fn(*args, **kwargs)

        return wrapper

    return decorator
```

> Для работы этого декоратора в JWT должны быть:
>
> * identity = user_id
> * claim `roles` = список ролей
>
> Это формируется в `auth_service.py` при логине через `create_access_token(identity=user["id"], additional_claims={"roles": roles})`.

---

## C.4. `app/middleware/validate_api.py`

### `validate` middleware (Flask)

Требования:

* проверяет тело запроса до БД
* при ошибке возвращает `400` в едином формате

Минимальные проверки для `match_finish`:

* `matchId` — число
* `result.isWin` — boolean
* `result.score` — число >= 0
* `result.durationSeconds` — число >= 0
* `result.powerDelta` — число или null (опционально)

#### Реализация (как должно работать)

* Используется декоратор `@validate_match_finish`
* Декоратор:

  * читает `request.get_json(silent=True)`
  * проверяет наличие ключей/типов
  * при ошибке выбрасывает `AppError(code='VALIDATION_ERROR', http_status=400, ...)`

```python
from functools import wraps
from flask import request
from .error_handler import AppError


def _bad_body(details):
    raise AppError(code="VALIDATION_ERROR", message="Invalid request body", http_status=400, details=details)


def validate_match_finish(fn):
    # Валидация запроса выполняется ДО контроллера/сервиса, чтобы не трогать БД при мусорном вводе

    @wraps(fn)
    def wrapper(*args, **kwargs):
        data = request.get_json(silent=True)

        if not isinstance(data, dict):
            _bad_body({"body": "JSON object expected"})

        match_id = data.get("matchId")
        result = data.get("result")

        if not isinstance(match_id, int):
            _bad_body({"matchId": "int required"})

        if not isinstance(result, dict):
            _bad_body({"result": "object required"})

        is_win = result.get("isWin")
        score = result.get("score")
        duration = result.get("durationSeconds")
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

# D) app/routes/

## D.1. `app/routes/match.py`

Файл маршрутов связывает URL с контроллером и подключает middleware-декораторы.

```python
from flask import Blueprint
from app.middleware.auth_jwt import auth_required
from app.middleware.validate_api import validate_match_finish
from app.controllers.match_controller import finish_match_controller

# Группа маршрутов /api/v1/match
match_bp = Blueprint("match", __name__, url_prefix="/api/v1/match")


@match_bp.post("/finish")
# 1) authJwt: проверяем токен и роль игрока
@auth_required(roles=["player"])
# 2) validate: проверяем тело запроса
@validate_match_finish
def finish_match():
    # 3) controller: извлекает данные и вызывает сервис
    return finish_match_controller()
```

---

# E) app/controllers/

## E.1. `app/controllers/match_controller.py`

Контроллер: только HTTP-обвязка.
SQL и транзакций тут быть не должно.

```python
from flask import g, request, jsonify
from app.services.match_service import finish_match_service


def finish_match_controller():
    # userId берём только из JWT (g.user заполняется middleware authJwt)
    user_id = g.user["userId"]

    # JSON тела запроса (уже провалидирован middleware validate)
    payload = request.get_json()

    # Сервис содержит бизнес-логику и транзакцию
    result = finish_match_service(user_id=user_id, payload=payload)

    # Единый формат успешного ответа
    return jsonify({
        "ok": True,
        "data": result
    }), 200
```

---

# F) app/services/

## F.1. `app/services/match_service.py`

Сервис: бизнес-логика + транзакция + orchestration репозиториев.

Важное правило: одна транзакция на весь итог.

```python
from app.extensions import db
from app.repositories.matches_repo import ensure_match_ownership_started, finish_match_row
from app.repositories.progress_repo import ensure_progress_row, lock_progress_row, add_progress_rewards
from app.repositories.stats_repo import upsert_daily_stats
from app.repositories.leaderboard_repo import upsert_leaderboard_score

_XP_WIN   = 100
_XP_LOSS  = 30
_SOFT_WIN = 50
_SOFT_LOSS = 10
_BOARD_CODE = "default"
_SEASON = 1


def finish_match_service(user_id: int, payload: dict) -> dict:
    match_id = payload["matchId"]
    result   = payload["result"]
    is_win   = result["isWin"]
    score    = result["score"]
    duration = result["durationSeconds"]

    # Сервер рассчитывает награды сам — клиент их не присылает
    xp_gained   = _XP_WIN   if is_win else _XP_LOSS
    soft_gained = _SOFT_WIN if is_win else _SOFT_LOSS

    try:
        # Важно: одна транзакция на весь итог (атомарность)
        # 0) Проверка владения матчем и статуса (без блокировки)
        ensure_match_ownership_started(db.session, match_id=match_id, user_id=user_id)

        # 1) Закрытие матча + запись battle_results
        finish_match_row(db.session, match_id=match_id, is_win=is_win, score=score, duration_seconds=duration)

        # 2) Гарантируем наличие строки прогресса
        ensure_progress_row(db.session, user_id=user_id)

        # 3) Блокируем строку прогресса (защита от гонок)
        lock_progress_row(db.session, user_id=user_id)

        # 4) Начисляем награды
        add_progress_rewards(db.session, user_id=user_id, xp=xp_gained, soft_currency=soft_gained)

        # 5) Upsert статистики
        upsert_daily_stats(db.session, user_id=user_id, playtime_seconds=duration, is_win=is_win, score=score)

        # 6) Upsert лидерборда
        upsert_leaderboard_score(db.session, user_id=user_id, board_code=_BOARD_CODE, season=_SEASON, score=score)

        # Commit — всё или ничего
        db.session.commit()

    except Exception:
        db.session.rollback()
        raise

    return {
        "matchId": match_id,
        "isWin": is_win,
        "xpGained": xp_gained,
        "softCurrencyGained": soft_gained,
    }
```

---

# G) app/repositories/

> Репозитории: только SQL, никаких расчётов наград, никаких правил "как считать".

## G.1. `app/repositories/matches_repo.py`

```python
from sqlalchemy import text
from app.middleware.error_handler import AppError


def ensure_match_ownership_started(session, match_id: int, user_id: int):
    # Проверяет, что матч существует, принадлежит пользователю и имеет статус started
    sql = text("SELECT id, user_id, status FROM matches WHERE id = :match_id")
    row = session.execute(sql, {"match_id": match_id}).mappings().first()
    if row is None:
        raise AppError(code="NOT_FOUND", message="Match not found", http_status=404)
    if row["user_id"] != user_id:
        raise AppError(code="FORBIDDEN", message="Match belongs to another user", http_status=403)
    if row["status"] != "started":
        raise AppError(code="CONFLICT", message="Match is not in started state", http_status=409)


def finish_match_row(session, match_id: int, is_win: bool, score: int, duration_seconds: int):
    # Закрытие матча
    upd = text("UPDATE matches SET status='finished', ended_at=NOW(3) WHERE id=:match_id")
    session.execute(upd, {"match_id": match_id})

    # Запись результата (upsert на случай retry)
    ins = text("""
        INSERT INTO battle_results (match_id, is_win, score, duration_seconds)
        VALUES (:match_id, :is_win, :score, :duration)
        ON DUPLICATE KEY UPDATE
          is_win=VALUES(is_win), score=VALUES(score), duration_seconds=VALUES(duration_seconds)
    """)
    session.execute(ins, {"match_id": match_id, "is_win": int(is_win), "score": score, "duration": duration_seconds})
```

---

## G.2. `app/repositories/progress_repo.py`

```python
from sqlalchemy import text
from app.middleware.error_handler import AppError


def ensure_progress_row(session, user_id: int):
    # INSERT IGNORE — создаёт строку прогресса, если её нет
    sql = text("INSERT IGNORE INTO player_progress (user_id) VALUES (:user_id)")
    session.execute(sql, {"user_id": user_id})


def lock_progress_row(session, user_id: int):
    # SELECT ... FOR UPDATE — блокировка строки (защита от гонок)
    sql = text("""
        SELECT user_id, xp, soft_currency
        FROM player_progress
        WHERE user_id = :user_id
        FOR UPDATE
    """)
    row = session.execute(sql, {"user_id": user_id}).fetchone()
    if row is None:
        raise AppError(code="PROGRESS_NOT_FOUND", message="player_progress row not found", http_status=404)


def add_progress_rewards(session, user_id: int, xp: int, soft_currency: int):
    # Обновление прогресса игрока
    sql = text("""
        UPDATE player_progress
        SET xp = xp + :xp,
            soft_currency = soft_currency + :soft
        WHERE user_id = :user_id
    """)
    session.execute(sql, {"xp": xp, "soft": soft_currency, "user_id": user_id})
```

---

## G.3. `app/repositories/stats_repo.py`

```python
from sqlalchemy import text


def upsert_daily_stats(session, user_id: int, playtime_seconds: int, is_win: bool, score: int):
    wins   = 1 if is_win else 0
    losses = 0 if is_win else 1

    # UPSERT статистики по дню
    sql = text("""
        INSERT INTO statistics_daily
          (user_id, day, sessions_count, events_count, playtime_seconds, wins, losses, score_sum)
        VALUES (:user_id, CURDATE(), 0, 1, :playtime, :wins, :losses, :score)
        ON DUPLICATE KEY UPDATE
          events_count     = events_count + 1,
          playtime_seconds = playtime_seconds + VALUES(playtime_seconds),
          wins             = wins + VALUES(wins),
          losses           = losses + VALUES(losses),
          score_sum        = score_sum + VALUES(score_sum)
    """)
    session.execute(sql, {
        "user_id": user_id,
        "playtime": playtime_seconds,
        "wins": wins,
        "losses": losses,
        "score": score
    })
```

---

## G.4. `app/repositories/leaderboard_repo.py`

```python
from sqlalchemy import text


def upsert_leaderboard_score(session, user_id: int, board_code: str, season: int, score: int):
    # UPSERT лидерборда (берём лучший score)
    sql = text("""
        INSERT INTO leaderboard_scores (user_id, board_code, season, score)
        VALUES (:user_id, :board_code, :season, :score)
        ON DUPLICATE KEY UPDATE
          score      = GREATEST(score, VALUES(score)),
          updated_at = CURRENT_TIMESTAMP(3)
    """)
    session.execute(sql, {
        "user_id": user_id,
        "board_code": board_code,
        "season": season,
        "score": score
    })
```

---

## 4.5. Middleware (итог)

В этом разделе выполнены middleware:

* `requestId` — `before_request/after_request`
* `authJwt` — декоратор `@auth_required(...)`
* `validate` — декоратор `@validate_match_finish`
* `errorHandler` — `@app.errorhandler(AppError)` и `@app.errorhandler(Exception)`

---

## 4.6. Транзакционная операция: `match_finish` (обязательная)

### 4.6.1. Что должно происходить внутри одной транзакции

Внутри одной TX выполняются:

1. проверка матча (`ensure_match_ownership_started`)
2. закрытие матча + запись `battle_results` (`finish_match_row`)
3. блокировка прогресса (`lock_progress_row`)
4. начисление наград (`add_progress_rewards`)
5. upsert `statistics_daily`
6. upsert `leaderboard_scores`
7. commit

---

### 4.6.2. Что обязательно добавить (чего обычно не хватает)

1. **Idempotency — защита от повторного `match_finish`**:
   Реализована через статус матча: `ensure_match_ownership_started()` выбрасывает `409 CONFLICT` если `status != 'started'`. Дополнительно в `events_service.py` используется `processed_events` с `UNIQUE(user_id, event_id)`.

2. **Anti-cheat / ownership checks**:
   * `matchId` должен принадлежать `userId` из JWT — проверяется в `ensure_match_ownership_started()`
   * статус матча должен быть `started` — проверяется там же

3. **Блокировки для защиты от гонок**:
   * `SELECT ... FOR UPDATE` на `player_progress` в `lock_progress_row()`

4. **Атомарность**:
   * любой сбой → `db.session.rollback()` в блоке `except Exception`

---

### 4.6.3. Пример входного запроса (Strategy)

```http
POST /api/v1/match/finish
Authorization: Bearer <JWT>
Content-Type: application/json
```

```json
{
  "matchId": 9,
  "result": {
    "isWin": true,
    "score": 1850,
    "durationSeconds": 1200,
    "powerDelta": 50
  }
}
```

> Матч `id=9` из `seed.sql` принадлежит `alice` (user_id=1), статус `started`.

---

### 4.6.4. Бизнес-логика начислений

Награды рассчитываются сервером в `match_service.py`:

```text
xp_gained   = 100  (isWin = true)  / 30  (isWin = false)
soft_gained = 50   (isWin = true)  / 10  (isWin = false)
```

---

### 4.6.5. SQL внутри транзакции (MySQL 8.0)

Начало транзакции:

```sql
START TRANSACTION;
```

#### 0) Проверка матча

```sql
SELECT id, user_id, status FROM matches WHERE id = 9;
```

#### 1) Закрытие матча

```sql
UPDATE matches SET status='finished', ended_at=NOW(3) WHERE id = 9;
```

#### 2) Запись результата

```sql
INSERT INTO battle_results (match_id, is_win, score, duration_seconds)
VALUES (9, 1, 1850, 1200)
ON DUPLICATE KEY UPDATE
  is_win=VALUES(is_win), score=VALUES(score), duration_seconds=VALUES(duration_seconds);
```

#### 3) Блокировка прогресса (защита от гонок)

```sql
SELECT user_id, xp, soft_currency
FROM player_progress
WHERE user_id = 1
FOR UPDATE;
```

#### 4) Обновление прогресса

```sql
UPDATE player_progress
SET xp = xp + 100,
    soft_currency = soft_currency + 50
WHERE user_id = 1;
```

#### 5) Обновление дневной статистики (upsert)

```sql
INSERT INTO statistics_daily
  (user_id, day, sessions_count, events_count, playtime_seconds, wins, losses, score_sum)
VALUES (1, CURDATE(), 0, 1, 1200, 1, 0, 1850)
ON DUPLICATE KEY UPDATE
  events_count     = events_count + 1,
  playtime_seconds = playtime_seconds + VALUES(playtime_seconds),
  wins             = wins + VALUES(wins),
  losses           = losses + VALUES(losses),
  score_sum        = score_sum + VALUES(score_sum);
```

#### 6) Upsert лидерборда

```sql
INSERT INTO leaderboard_scores (user_id, board_code, season, score)
VALUES (1, 'default', 1, 1850)
ON DUPLICATE KEY UPDATE
  score      = GREATEST(score, VALUES(score)),
  updated_at = CURRENT_TIMESTAMP(3);
```

Завершение транзакции:

```sql
COMMIT;
```

---

## 4.7. Единый формат успешного ответа

```json
{
  "ok": true,
  "data": {
    "matchId": 9,
    "isWin": true,
    "xpGained": 100,
    "softCurrencyGained": 50
  }
}
```

---

## 4.4.3. Проверка работоспособности (минимальная)

### 1) Запуск

```bash
python wsgi.py
```

### 2) Запрос (пример)

```http
POST /api/v1/match/finish
Authorization: Bearer <JWT>
Content-Type: application/json
```

```json
{
  "matchId": 9,
  "result": {
    "isWin": true,
    "score": 1850,
    "durationSeconds": 1200
  }
}
```

Ожидается:

* статус `200`
* заголовок `X-Request-Id`
* JSON `ok: true`
* записи в:

  * `matches` (status = finished)
  * `battle_results`
  * `player_progress`
  * `statistics_daily`
  * `leaderboard_scores`

### 3) Тестирование через test_client.html

В корне проекта находится `test_client.html` — HTML-страница для ручного тестирования всех endpoint'ов через браузер без Postman.

---

## 4.8. Установка библиотек (зависимости проекта)

### 4.8.1. Что и зачем устанавливается

Проект `strategy-support-is` использует следующие библиотеки Python:

| Библиотека | Зачем нужна |
|---|---|
| `Flask` | Web-фреймворк, маршруты, Blueprint'ы, `before/after_request` |
| `flask-jwt-extended` | JWT-аутентификация: `create_access_token`, `verify_jwt_in_request`, `get_jwt_identity` |
| `flask-sqlalchemy` | ORM-обёртка над SQLAlchemy, управление сессиями БД (`db.session`) |
| `flask-cors` | Заголовки CORS для Unity/Web/Mobile клиентов (`Access-Control-Allow-Origin`) |
| `pymysql` | MySQL-драйвер для Python (SQLAlchemy использует его под капотом) |
| `python-dotenv` | Загрузка переменных из `.env` файла в `os.environ` |
| `bcrypt` | Хэширование паролей при регистрации и проверка при логине |

---

### 4.8.2. Установка (шаг за шагом)

#### 1. Убедиться что Python установлен

```bash
python --version
# Должно быть Python 3.10 или новее

pip --version
# pip 23.x или новее
```

Если Python не установлен: скачать с [https://www.python.org/downloads/](https://www.python.org/downloads/)  
При установке обязательно отметить галку **«Add Python to PATH»**.

---

#### 2. Перейти в папку проекта

```bash
cd C:\Users\**ваш пользователь**\Desktop\project\strategy-support-is
```

Или на Linux/Mac:
```bash
cd ~/project/strategy-support-is
```

---

#### 3. Создать виртуальное окружение (рекомендуется)

```bash
python -m venv venv
```

Активировать:

```bash
# Windows
venv\Scripts\activate

# Linux / Mac
source venv/bin/activate
```

После активации в начале строки терминала появится `(venv)`.

---

#### 4. Установить все зависимости одной командой

```bash
pip install Flask flask-jwt-extended flask-sqlalchemy flask-cors pymysql python-dotenv bcrypt
```

Проверить что всё установилось:

```bash
pip list
```

Должны присутствовать все перечисленные пакеты.

---

#### 5. Создать файл `.env` в корне проекта

Создайте файл `.env` рядом с `wsgi.py`:

```text
SQLALCHEMY_DATABASE_URI=mysql+pymysql://root:yourpassword@localhost:3306/strategy_db?charset=utf8mb4
DATABASE_URL=mysql+pymysql://root:yourpassword@localhost:3306/strategy_db?charset=utf8mb4
JWT_SECRET_KEY=my-super-secret-jwt-key-change-in-prod
SECRET_KEY=my-flask-secret-key
```

Замените `root`, `yourpassword`, `strategy_db` на ваши реальные данные MySQL.

> **Важно:** файл `.env` не коммитится в Git (должен быть в `.gitignore`).

---

#### Шаг 6. Подготовить базу данных MySQL

Подключиться к MySQL и выполнить:

```sql
CREATE DATABASE strategy_db CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
```

Затем применить схему и тестовые данные:

```bash
mysql -u root -p strategy_db < db/schema.sql
mysql -u root -p strategy_db < seed.sql
```

---

####  7. Запустить сервер

```bash
python wsgi.py
```

Ожидаемый вывод:

```text
 * Running on http://127.0.0.1:5000
 * Debug mode: on
```

---

### 4.8.3. Проверка установки (быстрая)

Открыть в браузере:

```
http://127.0.0.1:5000/health
```

Ожидаемый ответ:

```json
{ "ok": true, "data": { "status": "healthy" } }
```

---

### 4.8.4. Частые ошибки при установке

| Ошибка | Причина | Решение |
|---|---|---|
| `ModuleNotFoundError: No module named 'flask'` | Окружение не активировано | Выполнить `venv\Scripts\activate` |
| `Access denied for user 'root'@'localhost'` | Неверный пароль MySQL | Проверить `.env` → `DATABASE_URL` |
| `Unknown database 'strategy_db'` | База не создана | `CREATE DATABASE strategy_db ...` |
| `Address already in use: 5000` | Порт занят | Завершить другой процесс или сменить порт в `wsgi.py` |
| `bcrypt not found` | Не установлен | `pip install bcrypt` |
| `cryptography` warning от PyMySQL | Не критично | `pip install cryptography` по желанию |

---

## 4.9. Клиентская сторона: `test_client.html`

### 4.9.1. Что такое `test_client.html` и зачем он нужен

`test_client.html` — это однофайловое веб-приложение, которое живёт прямо в корне проекта (`strategy-support-is/test_client.html`). Оно позволяет тестировать все API endpoint'ы **прямо в браузере** без Postman, curl и дополнительных инструментов.

**Что умеет клиент:**
- Отправлять все 6 типов запросов к API
- Автоматически сохранять JWT после логина и подставлять его в защищённые запросы
- Подсвечивать JSON-ответы (ключи, строки, числа, булевы значения)
- Вести лог всех запросов с статусами и временем выполнения
- Автозаполнять тела запросов готовыми тестовыми данными (Quick Fill)
- Настраивать Base URL для работы с любым сервером

---

### 4.9.2. Как запустить

1. Убедиться, что Flask-сервер запущен (`python wsgi.py`).
2. Открыть файл в браузере:

   ```
   Двойной клик → test_client.html
   ```

   Или перетащить файл в браузер. Никакого веб-сервера для самого клиента не нужно — он работает как локальный HTML-файл.

3. В поле **Base URL** (верхняя панель) проверить адрес — по умолчанию `http://localhost:5000`.

---

### 4.9.3. Дизайн и структура интерфейса

Интерфейс выдержан в стиле **военного командного терминала** — тёмная цветовая схема с тёмно-зелёными и золотыми акцентами, моноширинные шрифты для технических данных.

```
┌─────────────────────────────────────────────────────────┐
│  ⚔ Strategy IS    Base URL: [http://localhost:5000]  ●  │  ← Топ-бар
├──────────────┬──────────────────────────┬────────────────┤
│              │  Register User           │  Request Log   │
│  AUTH        │  POST /api/v1/auth/reg   │                │
│  › Register  │  ┌────────────────────┐  │  POST /auth/r  │
│  › Login     │  │ { "email": ...     │  │  GET  /profile │
│              │  └────────────────────┘  │  POST /match/f │
│  PROFILE     │  [▶ Send]                │                │
│  › My Profile│  ─────────────────────  │                │
│              │  RESPONSE: 201 Created   │                │
│  LEADERBOARD │  {"ok":true,"data":{...}}│                │
│  › Leaderboard                          │                │
│              │                          │                │
│  EVENTS      │                          │                │
│  › Post Event│                          │                │
│              │                          │                │
│  MATCH       │                          │                │
│  › Finish    │                          │                │
└──────────────┴──────────────────────────┴────────────────┘
```

**Три колонки:**
- **Левая (240px)** — сайдбар с группами endpoint'ов
- **Центральная** — рабочая область: редактор запроса + ответ
- **Правая (320px)** — лог запросов

---

### 4.9.4. Описание всех элементов интерфейса

#### Топ-бар

| Элемент | Назначение |
|---|---|
| Лого `⚔ Strategy IS` | Идентификатор приложения |
| Поле **Base URL** | Адрес сервера. Менять при развёртывании на другом хосте |
| Индикатор `●` | Зелёный = JWT есть, серый = не авторизован. Показывает `user <id>` |

#### Сайдбар (левая панель)

Endpoint'ы сгруппированы по разделам:

| Группа | Endpoint'ы |
|---|---|
| **Auth** | Register, Login |
| **Profile** | My Profile |
| **Leaderboard** | Leaderboard |
| **Events** | Post Event |
| **Match** | Finish Match |
| **System** | Health Check |

Активный endpoint подсвечен: правая граница тёмно-зелёная + фон.

Для каждого endpoint'а в сайдбаре видны:
- Метка метода (`GET` — синяя, `POST` — зелёная)
- Название endpoint'а
- Путь URL

#### Центральная область — верх (Request Area)

| Элемент | Назначение |
|---|---|
| Заголовок + путь | Название и полный путь выбранного endpoint'а |
| **Info Box** (золотой фон) | Подсказка — что делает endpoint, что нужно сделать сначала |
| **Query Params** | Поля параметров строки запроса (только для GET с параметрами, например `/leaderboard`) |
| **Quick Fill кнопки** | Готовые тестовые данные — нажать чтобы заполнить тело одним кликом |
| **Request Body** | Редактор JSON. Можно редактировать вручную |
| Кнопка `▶ Send` | Отправить запрос |

#### Центральная область — низ (Response Area)

| Элемент | Назначение |
|---|---|
| **Status Pill** | HTTP-статус ответа. Зелёный = 2xx, красный = ошибка |
| **Время** | Время выполнения запроса в миллисекундах |
| **Response Body** | JSON ответа с подсветкой синтаксиса |

**Цветовая схема подсветки JSON:**

| Цвет | Тип значения |
|---|---|
| Бирюзовый | Ключи объекта |
| Золотой | Строки |
| Голубой | Числа |
| Розовый | Boolean (true/false) |
| Серый | null |

#### Правая панель — Request Log

Показывает историю всех отправленных запросов в сессии:
- Метод (цвет по методу)
- Путь endpoint'а
- HTTP статус (зелёный/красный)
- Время запроса

Кнопка **Clear** — очистить лог.

---

### 4.9.5. Сценарий полного тестирования (шаг за шагом)

#### Шаг 1. Health Check

1. Нажать **System → Health Check** в сайдбаре.
2. Нажать `▶ Send`.
3. Ожидаемый результат: `200 OK`, ответ `{"ok": true, "data": {"status": "healthy"}}`.

Если появляется ошибка CORS или сеть недоступна — сервер не запущен.

---

#### Шаг 2. Регистрация нового пользователя

1. Нажать **Auth → Register**.
2. Нажать Quick Fill **«New User»** — тело заполнится автоматически.
3. Можно изменить email/nickname вручную.
4. Нажать `▶ Send`.
5. Ожидаемый результат: `201 Created`, в ответе `userId`.

---

#### Шаг 3. Логин

1. Нажать **Auth → Login**.
2. Нажать Quick Fill **«alice (admin)»**.
3. Нажать `▶ Send`.
4. Ожидаемый результат: `200 OK`. Индикатор `●` в топ-баре станет зелёным — JWT сохранён.

После этого все защищённые endpoint'ы будут работать автоматически.

> Тестовые пользователи (пароль у всех: `password123`):
> - `alice@example.com` — роли player + admin
> - `bob@example.com` — роли player + moderator
> - `carol@example.com` — player
> - `eve@example.com` — player (высокий рейтинг)
> - `banned@example.com` — заблокирован (вернёт 403)

---

#### Шаг 4. Просмотр профиля

1. Нажать **Profile → My Profile**.
2. Нажать `▶ Send`.
3. Ожидаемый результат: `200 OK`, данные alice: уровень, XP, валюта, роли.

Если вернулся `401 UNAUTHORIZED` — вернуться к шагу 3 (логин).

---

#### Шаг 5. Таблица лидеров

1. Нажать **Leaderboard → Leaderboard**.
2. В Query Params проверить: `boardCode=default`, `season=1`, `limit=10`.
3. Нажать `▶ Send`.
4. Ожидаемый результат: список игроков с рангами и очками.

Попробовать `boardCode=pvp` — другая доска.

---

#### Шаг 6. Отправка события

1. Нажать **Events → Post Event**.
2. Нажать Quick Fill **«Match Start»** или **«Unit Deploy»**.
3. Обратить внимание: `eventId` генерируется автоматически (уникальный UUID).
4. Нажать `▶ Send`.
5. Ожидаемый результат: `200 OK`, `{"status": "accepted"}`.

Попробовать нажать `▶ Send` ещё раз с тем же `eventId` — вернётся `409 EVENT_REJECTED` (дедупликация).

---

#### Шаг 7. Завершение матча

1. Нажать **Match → Finish Match**.
2. Нажать Quick Fill **«Win (match 9)»**.

   > Матч с `id=9` создан в `seed.sql` и принадлежит alice. Нужно быть залогиненным как alice.

3. Нажать `▶ Send`.
4. Ожидаемый результат: `200 OK`, в ответе `xpGained: 100`, `softCurrencyGained: 50`.

Попробовать нажать `▶ Send` повторно — вернётся `409 CONFLICT` (матч уже завершён).

---

### 4.9.6. Технические детали реализации клиента

Клиент написан на чистом HTML + CSS + JavaScript, без фреймворков.

#### Архитектура JS

```javascript
// Конфигурация всех endpoint'ов
const endpoints = {
  register:     { method: 'POST', path: '/api/v1/auth/register', auth: false, ... },
  login:        { method: 'POST', path: '/api/v1/auth/login',    auth: false, ... },
  profile:      { method: 'GET',  path: '/api/v1/profile',       auth: true,  ... },
  leaderboard:  { method: 'GET',  path: '/api/v1/leaderboard',   auth: false, params: [...] },
  event:        { method: 'POST', path: '/api/v1/events',        auth: true,  ... },
  match_finish: { method: 'POST', path: '/api/v1/match/finish',  auth: true,  ... },
  health:       { method: 'GET',  path: '/health',               auth: false  },
};
```

#### Хранение JWT

```javascript
let token = null;  // хранится в памяти вкладки

// При логине — автоматически извлекается из ответа:
if (currentEndpoint === 'login' && resp.ok && data?.data?.accessToken) {
    token = data.data.accessToken;
    updateTokenUI();
}

// При запросе — автоматически подставляется:
if (ep.auth && token)
    opts.headers['Authorization'] = 'Bearer ' + token;
```

#### Quick Fill для событий — автогенерация `eventId`

```javascript
// При выборе Quick Fill для /events — eventId генерируется каждый раз новый:
if (currentEndpoint === 'event' && body.eventId === '') {
    body.eventId = 'evt-' + Math.random().toString(36).slice(2,10) + Date.now().toString(36);
}
```

Это гарантирует уникальность при повторных нажатиях Quick Fill.

#### Подсветка JSON

```javascript
function syntaxHighlight(json) {
  return json.replace(/(ключи|строки|числа|булевые|null)/g, match => {
    // каждый тип оборачивается в <span class="json-key|str|num|bool|null">
  });
}
```

---

### 4.9.7. Дизайн: цветовая палитра и CSS-переменные

Все цвета объявлены как CSS-переменные в `:root`:

```css
:root {
  --bg:      #080c10;   /* фон — почти чёрный */
  --panel:   #0d1318;   /* панели */
  --panel2:  #111920;   /* вторичные панели */
  --border:  #1a2830;   /* границы */
  --border2: #1f3040;   /* акцентные границы */
  --gold:    #c8982a;   /* золото — Quick Fill, Info Box */
  --gold2:   #e8b84b;   /* золото светлое — лого, JSON-строки */
  --teal:    #1db8a0;   /* бирюза — активные элементы, ссылки */
  --teal2:   #25d4ba;   /* бирюза светлая — JSON-ключи, inputs */
  --red:     #e05050;   /* ошибки */
  --green:   #3dba7a;   /* успех, POST-метки */
  --blue:    #3a8fd4;   /* GET-метки, JSON-числа */
  --text:    #d4e0e8;   /* основной текст */
  --muted:   #4a6070;   /* вторичный текст */
  --bright:  #eef4f8;   /* заголовки */
}
```

**Сетка страницы** — CSS Grid 3 колонки:

```css
.shell {
  display: grid;
  grid-template-columns: 240px 1fr 320px;  /* сайдбар | контент | лог */
  grid-template-rows: 56px 1fr;            /* топ-бар | тело */
  height: 100vh;
}
```

**Анимированный фон** — сетка из тонких линий через псевдоэлемент:

```css
body::before {
  background-image:
    linear-gradient(rgba(29,184,160,0.03) 1px, transparent 1px),
    linear-gradient(90deg, rgba(29,184,160,0.03) 1px, transparent 1px);
  background-size: 40px 40px;
}
```

**Шрифты** (подключены через Google Fonts):
- `Rajdhani` — основной UI (навигация, кнопки, заголовки)
- `Share Tech Mono` — технические данные (URL, JSON, метки методов)

---

### 4.9.8. Как адаптировать клиент под свой проект

Если вы добавили новый endpoint в Flask — добавьте его в `test_client.html`:

```javascript
// В объект endpoints добавьте новую запись:
my_endpoint: {
  title:       'My New Endpoint',
  method:      'POST',
  path:        '/api/v1/my-endpoint',
  auth:        true,
  info:        'Описание что делает endpoint',
  quickFills: [
    { label: 'Пример', body: { field1: 'value1', field2: 42 } }
  ],
  defaultBody: { field1: 'value1', field2: 42 }
},
```

Добавьте кнопку в HTML-сайдбар:

```html
<div class="sidebar-section">
  <div class="sidebar-section-title">My Group</div>
  <button class="endpoint-btn" onclick="selectEndpoint('my_endpoint')">
    <div><span class="method-tag POST">POST</span></div>
    <div>
      <div class="ep-name">My Endpoint</div>
      <div class="ep-path">/api/v1/my-endpoint</div>
    </div>
  </button>
</div>
```
