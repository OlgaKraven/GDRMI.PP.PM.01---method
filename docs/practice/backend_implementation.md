# 2. Практика. 4. Реализация backend’а ИС сопровождения игрового продукта (Flask)

## 4.1. Задание к разделу 4 «Backend-логика»

### Необходимо выполнить

1. Реализовать middleware (для Flask — через `before_request/after_request`, декораторы и error handlers):

   * `requestId`
   * `authJwt`
   * `validate`
   * `errorHandler`

2. Реализовать **транзакционную операцию** для “итогового события” (например, `match_finish`) в **одной транзакции**:

   * запись события в `game_events`
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
  models/        (опционально, если используете ORM)
run.py
config.py
requirements.txt
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

Для сдачи раздела достаточно реализовать один “итоговый” endpoint:

* `POST /match/finish` **или**
* `POST /event` с `eventType = match_finish`

В данном минимальном примере реализуется: `POST /match/finish`.

---

## 4.4. Реализация проекта (Flask) — структура и файлы

### 4.4.1. Минимальная структура проекта (Flask)

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

## 4.4.2. Создание файлов проекта и пример их наполнения (обязательный минимум)

> Цель: получить минимально работоспособный backend, где один endpoint (`POST /match/finish`) проходит весь конвейер:
> `requestId → authJwt → validate → controller → service (TX) → repositories (SQL) → JSON response`,
> а ошибки всегда возвращаются единообразно.

---

# A) project-root

## A.1. `requirements.txt`

```text
Flask
flask-jwt-extended
flask-sqlalchemy
pymysql
python-dotenv
Flask-Migrate
```

> `Flask-Migrate` можно убрать, если миграции не используете. Но для практики обычно оставляют.

---

## A.2. `config.py`

```python
import os
from dotenv import load_dotenv

# Загружаем переменные окружения из .env (если файл есть)
load_dotenv()


class Config:
    # Flask
    # DEBUG=1 включает расширенные логи и удобнее для разработки
    DEBUG = os.getenv("DEBUG", "1") == "1"

    # MySQL + SQLAlchemy
    # Пример: mysql+pymysql://user:pass@localhost:3306/game_db?charset=utf8mb4
    SQLALCHEMY_DATABASE_URI = os.getenv("SQLALCHEMY_DATABASE_URI", "")
    SQLALCHEMY_TRACK_MODIFICATIONS = False

    # JWT
    # В реальном проекте ключ обязательно хранить в .env / секретах
    JWT_SECRET_KEY = os.getenv("JWT_SECRET_KEY", "change-me")

    # JSON
    JSON_SORT_KEYS = False
```

> Рекомендуется создать `.env` (не обязателен для сдачи, но удобно):
>
> * `SQLALCHEMY_DATABASE_URI=...`
> * `JWT_SECRET_KEY=...`
> * `DEBUG=1`

---

## A.3. `run.py`

```python
from app import create_app

# Создание Flask-приложения (регистрация middleware и маршрутов)
app = create_app()

if __name__ == "__main__":
    # Здесь НЕ пишем бизнес-логику и НЕ пишем SQL.
    # Файл отвечает только за запуск сервера.
    app.run(host="0.0.0.0", port=5000)
```

---

# B) app/

## B.1. `app/extensions.py`

```python
from flask_sqlalchemy import SQLAlchemy
from flask_jwt_extended import JWTManager

# db — единая точка доступа к БД (SQLAlchemy)
db = SQLAlchemy()

# jwt — менеджер JWT (проверка токена, извлечение identity и claims)
jwt = JWTManager()
```

---

## B.2. `app/__init__.py`

```python
from flask import Flask

from .extensions import db, jwt
from .middleware.request_id import register_request_id
from .middleware.error_handler import register_error_handlers
from .routes.match import match_bp
from config import Config


def create_app() -> Flask:
    # 1) Создаём Flask-приложение
    app = Flask(__name__)

    # 2) Загружаем конфигурацию
    app.config.from_object(Config)

    # 3) Инициализируем расширения (db, jwt)
    db.init_app(app)
    jwt.init_app(app)

    # 4) Регистрируем middleware и обработчики ошибок
    register_request_id(app)
    register_error_handlers(app)

    # 5) Регистрируем маршруты (Blueprint)
    app.register_blueprint(match_bp)

    return app
```

---

# C) app/middleware/

> В Flask “middleware” реализуется так:
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
req_abc123 user=15 POST /match/finish 200 18ms
```

#### Реализация (как должно работать)

* В `before_request`:

  * если клиент прислал `X-Request-Id`, использовать его
  * иначе сгенерировать `req_<shortid>`
  * сохранить в `g.request_id`
  * сохранить старт времени `g.started_at_ms`
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
        # req_abc123 user=15 POST /match/finish 200 18ms
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

---

## C.4. `app/middleware/validate.py`

### `validate` middleware (Flask)

Требования:

* проверяет тело запроса до БД
* при ошибке возвращает `400` в едином формате

Минимальные проверки для `match_finish`:

* `matchId` — число
* `result.isWin` — boolean
* `result.score` — число >= 0
* `result.durationSeconds` — число >= 0
* `result.waveReached` — число >= 0

#### Реализация (как должно работать)

* Используется декоратор `@validate_match_finish`
* Декоратор:

  * читает `request.get_json(silent=True)`
  * проверяет наличие ключей/типов
  * при ошибке выбрасывает `AppError(code='VALIDATION_ERROR', http_status=400, ...)`

```python
from flask import request
from .error_handler import AppError


def validate_match_finish(fn):
    # Валидация запроса выполняется ДО контроллера/сервиса, чтобы не трогать БД при мусорном вводе

    def _bad(details):
        # Единая форма ошибки валидации
        raise AppError(code="VALIDATION_ERROR", message="Invalid request body", http_status=400, details=details)

    def wrapper(*args, **kwargs):
        data = request.get_json(silent=True)

        if not isinstance(data, dict):
            _bad({"body": "JSON object expected"})

        match_id = data.get("matchId")
        result = data.get("result")

        if not isinstance(match_id, int):
            _bad({"matchId": "int required"})

        if not isinstance(result, dict):
            _bad({"result": "object required"})

        is_win = result.get("isWin")
        score = result.get("score")
        duration = result.get("durationSeconds")
        wave = result.get("waveReached")

        if not isinstance(is_win, bool):
            _bad({"result.isWin": "bool required"})
        if not isinstance(score, int) or score < 0:
            _bad({"result.score": "int >= 0 required"})
        if not isinstance(duration, int) or duration < 0:
            _bad({"result.durationSeconds": "int >= 0 required"})
        if not isinstance(wave, int) or wave < 0:
            _bad({"result.waveReached": "int >= 0 required"})

        return fn(*args, **kwargs)

    # Сохраняем имя для Flask/debug
    wrapper.__name__ = fn.__name__
    return wrapper
```

---

# D) app/routes/

## D.1. `app/routes/match.py`

Файл маршрутов связывает URL с контроллером и подключает middleware-декораторы.

```python
from flask import Blueprint
from app.middleware.auth_jwt import auth_required
from app.middleware.validate import validate_match_finish
from app.controllers.match_controller import finish_match_controller

# Группа маршрутов /match
match_bp = Blueprint("match", __name__, url_prefix="/match")


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
from datetime import datetime, timezone
from flask import g
from app.extensions import db
from app.middleware.error_handler import AppError

from app.repositories.events_repo import insert_game_event
from app.repositories.progress_repo import lock_progress_row, add_progress_rewards
from app.repositories.stats_repo import upsert_daily_stats
from app.repositories.leaderboard_repo import upsert_leaderboard_score


def _calc_rewards(wave_reached: int) -> dict:
    # Минимальная бизнес-логика начислений (пример)
    xp = 50 + wave_reached * 10
    soft = 100 + wave_reached * 20
    return {"xp": xp, "softCurrency": soft}


def finish_match_service(user_id: int, payload: dict) -> dict:
    # requestId нужен для идемпотентности/трассировки
    request_id = getattr(g, "request_id", "req_unknown")

    match_id = payload["matchId"]
    result = payload["result"]
    is_win = result["isWin"]
    score = result["score"]
    duration = result["durationSeconds"]
    wave = result["waveReached"]

    rewards = _calc_rewards(wave)

    # Для минимума фиксируем board_code и season
    board_code = "td_score"
    season = 1

    try:
        # Важно: одна транзакция на весь итог (атомарность)
        with db.session.begin():
            # 0) блокируем строку прогресса (защита от гонок)
            lock_progress_row(db.session, user_id)

            # 1) пишем событие (game_events)
            insert_game_event(
                db.session,
                user_id=user_id,
                session_id=None,
                event_type="match_finish",
                payload={
                    "requestId": request_id,
                    "matchId": match_id,
                    "result": {
                        "isWin": is_win,
                        "score": score,
                        "durationSeconds": duration,
                        "waveReached": wave
                    }
                }
            )

            # 2) обновляем прогресс (player_progress)
            add_progress_rewards(db.session, user_id, xp=rewards["xp"], soft_currency=rewards["softCurrency"])

            # 3) upsert statistics_daily
            upsert_daily_stats(
                db.session,
                user_id=user_id,
                playtime_seconds=duration,
                is_win=is_win,
                score=score
            )

            # 4) upsert leaderboard_scores
            upsert_leaderboard_score(
                db.session,
                user_id=user_id,
                board_code=board_code,
                season=season,
                score=score
            )

        # commit выполняется автоматически при выходе из begin()

        return {
            "matchId": match_id,
            "rewards": rewards,
            "leaderboard": {
                "boardCode": board_code,
                "season": season,
                "score": score
            },
            "requestId": request_id,
            "serverTime": datetime.now(timezone.utc).isoformat()
        }

    except AppError:
        # Контролируемые ошибки (валидация/доступ/логика) — пробрасываем наверх
        raise
    except Exception:
        # Любая другая ошибка должна привести к rollback и единому формату ответа
        raise AppError(code="TX_FAILED", message="Transaction failed", http_status=500)
```

---

# G) app/repositories/

> Репозитории: только SQL, никаких расчётов наград, никаких правил “как считать”.

## G.1. `app/repositories/events_repo.py`

```python
import json
from sqlalchemy import text


def insert_game_event(session, user_id: int, session_id, event_type: str, payload: dict):
    # Репозиторий выполняет только SQL (вставка события)
    sql = text("""
        INSERT INTO game_events (user_id, session_id, event_type, payload_json)
        VALUES (:user_id, :session_id, :event_type, CAST(:payload AS JSON))
    """)
    session.execute(sql, {
        "user_id": user_id,
        "session_id": session_id,
        "event_type": event_type,
        "payload": json.dumps(payload, ensure_ascii=False)
    })
```

---

## G.2. `app/repositories/progress_repo.py`

```python
from sqlalchemy import text
from app.middleware.error_handler import AppError


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
        # Если строки прогресса нет — ошибка инициализации данных пользователя
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
    # Минимальная агрегация по победам/поражениям
    wins = 1 if is_win else 0
    losses = 0 if is_win else 1

    # UPSERT статистики по дню
    sql = text("""
        INSERT INTO statistics_daily (user_id, day, sessions_count, events_count, playtime_seconds, wins, losses, score_sum)
        VALUES (:user_id, CURDATE(), 0, 1, :playtime, :wins, :losses, :score)
        ON DUPLICATE KEY UPDATE
          events_count = events_count + 1,
          playtime_seconds = playtime_seconds + VALUES(playtime_seconds),
          wins = wins + VALUES(wins),
          losses = losses + VALUES(losses),
          score_sum = score_sum + VALUES(score_sum);
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
          score = GREATEST(score, VALUES(score)),
          updated_at = CURRENT_TIMESTAMP(3);
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

1. запись события в `game_events` (`event_type='match_finish'`)
2. обновление `player_progress` (начисления)
3. upsert `statistics_daily`
4. upsert `leaderboard_scores`
5. commit

---

### 4.6.2. Что обязательно добавить (чего обычно не хватает)

1. **Идемпотентность**: защита от повторного `match_finish`
   Минимально: проверка, что матч ещё не завершён (если есть `matches`).
   Универсально: хранить `requestId` в payload и проверять дубликаты на уровне приложения.

2. **Anti-cheat / ownership checks**:

   * `matchId` должен принадлежать `userId` из JWT (если есть таблица matches)
   * статус матча должен быть `started`

3. **Блокировки для защиты от гонок**:

   * `SELECT ... FOR UPDATE` на `player_progress` и (при необходимости) на `leaderboard_scores`

4. **Атомарность**:

   * любой сбой → `ROLLBACK` и единый формат ошибки

---

### 4.6.3. Пример входного запроса (Tower Defense)

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

### 4.6.4. Минимальная бизнес-логика (пример начислений)

Награды рассчитываются сервером:

```text
xp = 50 + waveReached * 10
softCurrency = 100 + waveReached * 20
```

---

### 4.6.5. SQL внутри транзакции (MySQL 8.0)

Начало транзакции:

```sql
START TRANSACTION;
```

#### 0) Блокировка прогресса игрока (защита от гонок)

```sql
SELECT user_id, xp, soft_currency
FROM player_progress
WHERE user_id = ?
FOR UPDATE;
```

#### 1) Запись события в `game_events`

```sql
INSERT INTO game_events (user_id, session_id, event_type, payload_json)
VALUES (
  ?, 
  NULL,
  'match_finish',
  JSON_OBJECT(
    'requestId', ?,
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

#### 2) Обновление прогресса

```sql
UPDATE player_progress
SET xp = xp + ?,
    soft_currency = soft_currency + ?
WHERE user_id = ?;
```

#### 3) Обновление дневной статистики (upsert)

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

#### 4) Upsert лидерборда

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

## 4.7. Единый формат успешного ответа

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
    },
    "requestId": "req_abc123"
  }
}
```

---

## 4.4.3. Проверка работоспособности (минимальная)

### 1) Запуск

```bash
python run.py
```

### 2) Запрос (пример)

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

Ожидается:

* статус `200`
* заголовок `X-Request-Id`
* JSON `ok: true`
* записи в:

  * `game_events`
  * `player_progress`
  * `statistics_daily`
  * `leaderboard_scores`

---
 
