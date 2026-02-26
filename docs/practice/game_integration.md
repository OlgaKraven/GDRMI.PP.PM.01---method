# 2. Практика 6. Руководство по сборке Unity-клиента для strategy-support-is

## Что это такое и зачем

Данный документ — пошаговая инструкция по созданию Unity-приложения, которое подключается к Flask-бэкенду `strategy-support-is` и обменивается с ним данными. Уровень реализации — учебный MVP: авторизация, профиль, события, завершение матча, лидерборд.

После выполнения всех шагов у вас будет Unity-приложение с тремя сценами, которое:
- Регистрирует и авторизует пользователей через Flask JWT
- Отображает профиль и прогресс игрока
- Отправляет игровые события с UUID-дедупликацией
- Завершает матч и получает начисленные сервером награды
- Показывает таблицу лидеров

---

## Часть 1. Теория: как Unity общается с сервером

### 6.1. Что такое REST API и почему это важно для Unity

REST API — это набор URL-адресов (endpoint'ов) на сервере, к которым клиент обращается по протоколу HTTP. Unity выступает клиентом: она формирует запрос, отправляет его серверу и получает ответ в формате JSON.

Главное правило: **сервер — источник истины**. Unity не хранит постоянные данные о прогрессе, очках и наградах в себе. Всё это живёт в базе данных на сервере и запрашивается по API.

### 6.2. HTTP-методы

| Метод  | Назначение                    | Пример                        |
|--------|-------------------------------|-------------------------------|
| GET    | Получить данные               | `GET /api/v1/profile`         |
| POST   | Создать / выполнить действие  | `POST /api/v1/match/finish`   |

### 6.3. JSON — формат обмена данными

Все запросы и ответы используют JSON. Unity сериализует классы C# в JSON с помощью `JsonUtility.ToJson()` и десериализует ответы обратно через `JsonUtility.FromJson<T>()`.

```json
// Пример запроса к /api/v1/auth/login
{ "email": "alice@example.com", "password": "password123" }

// Пример успешного ответа
{ "ok": true, "data": { "accessToken": "eyJ...", "userId": 1, "nickname": "Commander_Alice" } }

// Пример ответа с ошибкой
{ "ok": false, "error": { "code": "UNAUTHORIZED", "message": "Invalid credentials", "requestId": "req_abc123" } }
```

### 6.4. JWT — как работает авторизация

**JWT (JSON Web Token)** — это зашифрованный токен, который сервер выдаёт после успешного входа. Все последующие запросы к защищённым endpoint'ам отправляются с заголовком:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

Токен хранится в памяти приложения (в классе `AuthSession`) и не сохраняется на диск (в учебном проекте). При перезапуске приложения нужно войти заново.

### 6.5. UnityWebRequest — HTTP в Unity

Unity не может использовать `HttpClient` из .NET напрямую в корутинах. Для работы с HTTP используется `UnityWebRequest` — встроенный класс Unity.

Все вызовы API выполняются как **корутины** (IEnumerator + StartCoroutine). Это значит: HTTP-запрос не блокирует основной поток, а выполняется асинхронно в фоне.

```csharp
// Паттерн корутины
StartCoroutine(api.Login("email", "pass",
    onSuccess: resp => Debug.Log("OK: " + resp.nickname),
    onError:   err  => Debug.LogError("Fail: " + err)));
```

### 6.6. Контракт всех ответов сервера

Все ответы сервера имеют единую структуру:

```
{ "ok": true/false, "data": {...} }   ← успех
{ "ok": false, "error": {...} }       ← ошибка
```

В Unity это отражено классом `ApiResponse<T>`:

```csharp
[Serializable]
public class ApiResponse<T>
{
    public bool     ok;
    public T        data;
    public ApiError error;
}
```

---

## Часть 2. Структура скриптов

```
Assets/
  Scripts/
    API/
      ApiConfig.cs        ← адрес сервера и все URL
      ApiModels.cs        ← все DTO (классы запросов/ответов)
      AuthSession.cs      ← хранит JWT-токен и UserId
      ApiClient.cs        ← низкоуровневый HTTP (Get/Post)
      GameApiService.cs   ← высокоуровневый сервис (Login/Profile/...)
    UI/
      AuthController.cs   ← экран входа/регистрации
      ProfileController.cs← экран профиля
      LeaderboardController.cs ← таблица лидеров
    Game/
      MatchController.cs  ← управление матчем
      EventSender.cs      ← отправка произвольных событий
```

### Зависимости между слоями

```
UI / Game
    ↓  вызывают
GameApiService
    ↓  использует
ApiClient  +  AuthSession
    ↓  использует
UnityWebRequest  (встроен в Unity)
```

---

## Часть 3. Endpoint'ы сервера

### 3.1. Таблица всех endpoint'ов

| Метод | URL                         | Авторизация | Описание                   |
|-------|-----------------------------|-------------|----------------------------|
| POST  | `/api/v1/auth/register`     | Нет         | Регистрация                |
| POST  | `/api/v1/auth/login`        | Нет         | Вход, получить JWT         |
| GET   | `/api/v1/profile`           | Bearer JWT  | Профиль + прогресс         |
| POST  | `/api/v1/events`            | Bearer JWT  | Отправить игровое событие  |
| POST  | `/api/v1/match/finish`      | Bearer JWT  | Завершить матч             |
| GET   | `/api/v1/leaderboard`       | Нет         | Таблица лидеров            |

### 3.2. Описание каждого endpoint'а

#### POST /api/v1/auth/register

**Запрос:**
```json
{
  "email":    "player@example.com",
  "password": "mypassword",
  "nickname": "Commander"
}
```

**Ответ 201:**
```json
{ "ok": true, "data": { "userId": 42, "email": "...", "nickname": "Commander" } }
```

**Ошибки:**
- `400` — не заполнены поля / пароль < 6 символов
- `409` — email уже зарегистрирован

---

#### POST /api/v1/auth/login

**Запрос:**
```json
{ "email": "alice@example.com", "password": "password123" }
```

**Ответ 200:**
```json
{
  "ok": true,
  "data": {
    "accessToken": "eyJ...",
    "userId":      1,
    "nickname":    "Commander_Alice"
  }
}
```

**Ошибки:**
- `401` — неверный пароль или email
- `403` — аккаунт заблокирован

> Данные из `seed.sql`: `alice@example.com` / `password123`

---

#### GET /api/v1/profile

**Заголовок:** `Authorization: Bearer <token>`

**Ответ 200:**
```json
{
  "ok": true,
  "data": {
    "userId":   1,
    "email":    "alice@example.com",
    "nickname": "Commander_Alice",
    "roles":    ["player"],
    "progress": {
      "level":        42,
      "xp":           85400,
      "softCurrency": 12500,
      "hardCurrency": 250
    }
  }
}
```

**Ошибки:**
- `401` — токен отсутствует или устарел
- `403` — аккаунт заблокирован

---

#### POST /api/v1/events

**Заголовок:** `Authorization: Bearer <token>`

**Запрос:**
```json
{
  "eventId":   "550e8400-e29b-41d4-a716-446655440000",
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

**Ответ 200:**
```json
{ "ok": true, "data": { "eventId": "550e...", "status": "accepted" } }
```

**Важно:** `eventId` — уникальный UUID на каждое событие. Сервер отклонит повторный запрос с тем же `eventId` с кодом `409`.

**Ошибки:**
- `400` — неверная структура
- `409` — дублирующийся `eventId`

---

#### POST /api/v1/match/finish

**Заголовок:** `Authorization: Bearer <token>`

**Запрос:**
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

**Ответ 200:**
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

**Важно:** XP и монеты **рассчитывает сервер**, клиент их не присылает.

**Ошибки:**
- `404` — матч не найден
- `403` — матч принадлежит другому игроку
- `409` — матч уже завершён (status ≠ started)

---

#### GET /api/v1/leaderboard

**Параметры запроса:** `?boardCode=default&season=1&limit=10`

**Ответ 200:**
```json
{
  "ok": true,
  "data": {
    "boardCode": "default",
    "season":    1,
    "items": [
      { "rank": 1, "userId": 5, "nickname": "Siege_Eve",     "score": 9850 },
      { "rank": 2, "userId": 3, "nickname": "WarLord_Carol", "score": 8720 }
    ]
  }
}
```

---

## Часть 4. Пошаговая сборка проекта Unity

### Шаг 1. Создать проект

1. Открыть **Unity Hub**.
2. Нажать **New Project** → выбрать шаблон **2D** (или **3D**) → назвать `StrategyClient`.
3. Нажать **Create project**.

---

### Шаг 2. Скопировать скрипты

1. В папке проекта откройте `Assets`.
2. Создайте структуру папок:
   ```
   Assets/Scripts/API/
   Assets/Scripts/UI/
   Assets/Scripts/Game/
   ```
3. Скопируйте все `.cs`-файлы из пакета в соответствующие папки:
   - `API/` → `ApiConfig.cs`, `ApiModels.cs`, `AuthSession.cs`, `ApiClient.cs`, `GameApiService.cs`
   - `UI/`  → `AuthController.cs`, `ProfileController.cs`, `LeaderboardController.cs`
   - `Game/` → `MatchController.cs`, `EventSender.cs`

---

### Шаг 3. Настроить адрес сервера

Откройте `Assets/Scripts/API/ApiConfig.cs`. По умолчанию:

```csharp
public const string BaseUrl = "http://127.0.0.1:5000";
```

Если Flask запущен на другом порту или машине — замените адрес здесь. Менять нужно только эту одну строчку.

---

### Шаг 4. Запустить сервер

Перед тестированием в Unity убедитесь, что Flask-сервер запущен:

```bash
cd strategy-support-is
python wsgi.py
```

Сервер должен выводить:
```
* Running on http://127.0.0.1:5000
```

---

### Шаг 5. Сцена AuthScene — экран входа

#### 5.1. Создать сцену

`File → New Scene → Save As → Scenes/AuthScene`.

#### 5.2. Создать Canvas

1. `GameObject → UI → Canvas`.
2. В Canvas создайте два панели: `LoginPanel` и `RegisterPanel`.

#### 5.3. Наполнить LoginPanel

Добавьте в **LoginPanel**:

| Объект                 | Компонент        | Назначение            |
|------------------------|------------------|-----------------------|
| `EmailInput`           | TMP_InputField   | Поле email            |
| `PasswordInput`        | TMP_InputField   | Поле пароля (Password)|
| `LoginButton`          | Button + TMP_Text| Кнопка "Войти"        |
| `StatusText`           | TMP_Text         | Статус / ошибка       |
| `GoToRegisterButton`   | Button           | Переключить панель     |

#### 5.4. Наполнить RegisterPanel (аналогично)

Добавьте поля email, password, nickname, кнопку регистрации и статус.

#### 5.5. Создать менеджер

1. `GameObject → Create Empty` → назвать `AuthManager`.
2. Добавьте компоненты: `AuthController` и `GameApiService`.
3. Привяжите поля в Inspector:
   - `loginPanel` → объект `LoginPanel`
   - `registerPanel` → объект `RegisterPanel`
   - `loginEmail` → `EmailInput`
   - `loginPassword` → `PasswordInput`
   - `loginButton` → `LoginButton`
   - `regEmail`, `regPassword`, `regNickname` → соответствующие поля
   - `registerButton` → кнопка регистрации
   - `statusText` → `StatusText`
   - `sceneAfterLogin` → `"MainMenu"` (имя следующей сцены)

4. На `GoToRegisterButton` в Inspector → `Button.OnClick`:
   - Добавить → выбрать `AuthManager` → метод `AuthController.ShowRegister`.

---

### Шаг 6. Сцена MainMenu — профиль и лидерборд

#### 6.1. Создать сцену

`File → New Scene → Save As → Scenes/MainMenu`.

#### 6.2. Панель профиля

1. Создайте `GameObject → Create Empty` → `ProfileManager`.
2. Добавьте `ProfileController` и `GameApiService`.
3. В Canvas создайте панель `ProfilePanel` с TMP_Text-полями:
   - `NicknameText`, `EmailText`, `LevelText`, `XpText`, `SoftText`, `HardText`, `StatusText`
4. Привяжите все поля в Inspector у `ProfileController`.
5. Кнопка "Обновить" → `ProfileController.Refresh()`.

#### 6.3. Панель лидерборда

1. Создайте `GameObject → Create Empty` → `LeaderboardManager`.
2. Добавьте `LeaderboardController` и `GameApiService`.
3. В Canvas создайте `ScrollView` для списка.
4. Создайте Prefab строки:
   - `GameObject → Create Empty` → `LeaderboardRow`.
   - Добавьте три `TMP_Text`: `RankText`, `NicknameText`, `ScoreText`.
   - Сохраните в `Assets/Prefabs/LeaderboardRow.prefab`.
5. В Inspector `LeaderboardController`:
   - `rowContainer` → `ScrollView/Viewport/Content`
   - `rowPrefab` → `LeaderboardRow.prefab`

---

### Шаг 7. Сцена GameScene — матч

#### 7.1. Создать сцену

`File → New Scene → Save As → Scenes/GameScene`.

#### 7.2. Создать менеджер матча

1. `GameObject → Create Empty` → `MatchManager`.
2. Добавьте `MatchController` и `GameApiService`.
3. В Inspector:
   - `matchId` → `9` (матч из seed.sql, принадлежащий alice)
   - Привяжите: `timerText`, `statusText`, `rewardText`, `resultPanel`

#### 7.3. Кнопки тестирования

Добавьте два Button:

- `WinButton` → в OnClick: `MatchController.EndMatchWin()` или напишите обёртку:

```csharp
// Добавьте в MatchController для привязки кнопок в Inspector:
public void EndMatchWin()  => EndMatch(isWin: true,  score: 1850);
public void EndMatchLose() => EndMatch(isWin: false, score:  300);
```

---

### Шаг 8. Добавить сцены в Build Settings

1. `File → Build Settings`.
2. Нажать **Add Open Scenes** для каждой сцены: `AuthScene`, `MainMenu`, `GameScene`.
3. Убедиться, что `AuthScene` стоит первой (index 0).

---

### Шаг 9. Проверить работу

**Порядок тестирования:**

1. Запустить Flask: `python wsgi.py`.
2. Нажать **Play** в Unity.
3. Войти с данными из `seed.sql`: `alice@example.com` / `password123`.
4. После входа автоматически откроется `MainMenu`.
5. Проверить профиль — должны появиться данные уровня/XP.
6. Проверить лидерборд — должны появиться строки.
7. Перейти в `GameScene`, нажать "Победа" — должны появиться награды.

---

### Шаг 10. Отладка

#### Консоль Unity

Все запросы и ответы логируются через `Debug.Log`. При ошибке будет:
```
[Match] FinishMatch error: [409] Match is not in started state
```

#### Частые проблемы

| Проблема | Причина | Решение |
|---|---|---|
| `Connection refused` | Flask не запущен | `python wsgi.py` |
| `401 UNAUTHORIZED` | Не выполнен вход | Вызвать Login() перед другими запросами |
| `409 CONFLICT` при finish | Матч уже закрыт | Создать новый матч в БД или использовать другой `matchId` |
| `400 VALIDATION_ERROR` | Неверный тип поля | Проверить DTO в `ApiModels.cs` |
| JSON parse error | Ответ сервера не совпадает с моделью | Проверить поля класса в `ApiModels.cs` |

---

## Часть 5. Тестовые учётные данные (из seed.sql)

| Пользователь | Email                  | Пароль        | Матч id=9 |
|--------------|------------------------|---------------|-----------|
| alice        | `alice@example.com`    | `password123` | ✅ owner  |
| bob          | `bob@example.com`      | `password123` | —         |
| banned       | `banned@example.com`   | `password123` | заблокирован |

---

## Часть 6. Описание каждого скрипта

### ApiConfig.cs

Единственное место где хранится `BaseUrl` и все строки URL. При смене адреса сервера меняется только здесь.

```Csharp
namespace StrategyGame.API
{
    /// <summary>
    /// Единая точка настройки адреса сервера.
    /// Меняйте BaseUrl под своё окружение:
    ///   Локально:   http://127.0.0.1:5000
    ///   Production: https://your-server.com
    /// </summary>
    public static class ApiConfig
    {
        public const string BaseUrl = "http://127.0.0.1:5000";

        // Endpoints
        public const string Register       = BaseUrl + "/api/v1/auth/register";
        public const string Login          = BaseUrl + "/api/v1/auth/login";
        public const string Profile        = BaseUrl + "/api/v1/profile";
        public const string Events         = BaseUrl + "/api/v1/events";
        public const string MatchFinish    = BaseUrl + "/api/v1/match/finish";
        public const string Leaderboard    = BaseUrl + "/api/v1/leaderboard";

        // Leaderboard defaults
        public const string DefaultBoard   = "default";
        public const int    DefaultSeason  = 1;
        public const int    DefaultLimit   = 10;
    }
}
```

---

### ApiModels.cs

Все классы-модели данных (`[Serializable]`). Каждый класс точно соответствует полю `data` в ответе сервера. Нельзя переименовывать поля — JsonUtility сопоставляет их по имени.

```Csharp
using System;
using System.Collections.Generic;

namespace StrategyGame.API
{
    // ─────────────────────────────────────────
    // Общая обёртка ответов сервера
    // { "ok": true/false, "data": {...}, "error": {...} }
    // ─────────────────────────────────────────

    [Serializable]
    public class ApiResponse<T>
    {
        public bool ok;
        public T    data;
        public ApiError error;
    }

    [Serializable]
    public class ApiError
    {
        public string code;
        public string message;
        public string requestId;
    }

    // ─────────────────────────────────────────
    // POST /api/v1/auth/register
    // ─────────────────────────────────────────

    [Serializable]
    public class RegisterRequest
    {
        public string email;
        public string password;
        public string nickname;
    }

    [Serializable]
    public class RegisterResponse
    {
        public int    userId;
        public string email;
        public string nickname;
    }

    // ─────────────────────────────────────────
    // POST /api/v1/auth/login
    // ─────────────────────────────────────────

    [Serializable]
    public class LoginRequest
    {
        public string email;
        public string password;
    }

    [Serializable]
    public class LoginResponse
    {
        public string accessToken;
        public int    userId;
        public string nickname;
    }

    // ─────────────────────────────────────────
    // GET /api/v1/profile
    // ─────────────────────────────────────────

    [Serializable]
    public class ProfileUser
    {
        public int    id;
        public string email;
        public string nickname;
        public List<string> roles;
    }

    [Serializable]
    public class ProfileProgress
    {
        public int level;
        public int xp;
        public int softCurrency;
        public int hardCurrency;
    }

    [Serializable]
    public class ProfileResponse
    {
        public int              userId;
        public string           email;
        public string           nickname;
        public List<string>     roles;
        public ProfileProgress  progress;
    }

    // ─────────────────────────────────────────
    // POST /api/v1/events
    // ─────────────────────────────────────────

    [Serializable]
    public class EventRequest
    {
        public string eventId;      // UUID — обязателен для дедупликации
        public string eventType;
        public int?   sessionId;    // nullable
        public string clientTime;   // ISO 8601
        public object payload;      // произвольный JSON-объект
    }

    [Serializable]
    public class EventResponse
    {
        public string eventId;
        public string status;       // "accepted"
    }

    // ─────────────────────────────────────────
    // POST /api/v1/match/finish
    // ─────────────────────────────────────────

    [Serializable]
    public class MatchResult
    {
        public bool   isWin;
        public int    score;
        public int    durationSeconds;
        public int?   powerDelta;    // опционально
    }

    [Serializable]
    public class MatchFinishRequest
    {
        public int         matchId;
        public MatchResult result;
    }

    [Serializable]
    public class MatchFinishResponse
    {
        public int  matchId;
        public bool isWin;
        public int  xpGained;
        public int  softCurrencyGained;
    }

    // ─────────────────────────────────────────
    // GET /api/v1/leaderboard
    // ─────────────────────────────────────────

    [Serializable]
    public class LeaderboardItem
    {
        public int    rank;
        public int    userId;
        public string nickname;
        public int    score;
    }

    [Serializable]
    public class LeaderboardResponse
    {
        public string                boardCode;
        public int                   season;
        public List<LeaderboardItem> items;
    }
}

```

---



### AuthSession.cs

Статический класс-синглтон. Хранит `AccessToken`, `UserId`, `Nickname` в памяти. Все API-клиенты читают токен из него автоматически.

```Csharp
namespace StrategyGame.API
{
    /// <summary>
    /// Хранит JWT-токен и данные текущего пользователя.
    /// Живёт в памяти на протяжении игровой сессии.
    /// Используется всеми API-клиентами для подстановки Bearer-заголовка.
    /// </summary>
    public static class AuthSession
    {
        public static string AccessToken { get; private set; }
        public static int    UserId      { get; private set; }
        public static string Nickname    { get; private set; }
        public static bool   IsLoggedIn  => !string.IsNullOrEmpty(AccessToken);

        public static void Save(LoginResponse resp)
        {
            AccessToken = resp.accessToken;
            UserId      = resp.userId;
            Nickname    = resp.nickname;
        }

        public static void Clear()
        {
            AccessToken = null;
            UserId      = 0;
            Nickname    = null;
        }
    }
}

```

---



### ApiClient.cs

Низкоуровневый слой. Умеет выполнять GET и POST через `UnityWebRequest`. Разбирает ответ в `ApiResponse<T>` и вызывает `onSuccess` или `onError`. Напрямую из UI не вызывается — только через `GameApiService`.

```Csharp
using System;
using System.Collections;
using System.Text;
using UnityEngine;
using UnityEngine.Networking;

namespace StrategyGame.API
{
    /// <summary>
    /// Низкоуровневый HTTP-клиент.
    /// Умеет отправлять GET и POST с JSON-телом.
    /// При наличии AccessToken автоматически добавляет заголовок Authorization.
    /// Все методы — корутины: вызывайте через StartCoroutine().
    /// </summary>
    public static class ApiClient
    {
        // ─── GET ────────────────────────────────────────────────

        /// <summary>
        /// Отправить GET-запрос и получить JSON-ответ.
        /// </summary>
        public static IEnumerator Get<TResponse>(
            string url,
            Action<TResponse> onSuccess,
            Action<string>    onError,
            bool              withAuth = true)
        {
            using var req = UnityWebRequest.Get(url);

            SetHeaders(req, withAuth);

            yield return req.SendWebRequest();

            HandleResponse(req, onSuccess, onError);
        }

        // ─── POST ───────────────────────────────────────────────

        /// <summary>
        /// Отправить POST-запрос с JSON-телом и получить JSON-ответ.
        /// </summary>
        public static IEnumerator Post<TRequest, TResponse>(
            string   url,
            TRequest body,
            Action<TResponse> onSuccess,
            Action<string>    onError,
            bool              withAuth = true)
        {
            string json = JsonUtility.ToJson(body);
            byte[] bytes = Encoding.UTF8.GetBytes(json);

            using var req = new UnityWebRequest(url, "POST");
            req.uploadHandler   = new UploadHandlerRaw(bytes);
            req.downloadHandler = new DownloadHandlerBuffer();

            SetHeaders(req, withAuth);

            yield return req.SendWebRequest();

            HandleResponse<TResponse>(req, onSuccess, onError);
        }

        // ─── Helpers ────────────────────────────────────────────

        private static void SetHeaders(UnityWebRequest req, bool withAuth)
        {
            req.SetRequestHeader("Content-Type",  "application/json");
            req.SetRequestHeader("Accept",        "application/json");

            if (withAuth && AuthSession.IsLoggedIn)
                req.SetRequestHeader("Authorization", "Bearer " + AuthSession.AccessToken);
        }

        private static void HandleResponse<TResponse>(
            UnityWebRequest   req,
            Action<TResponse> onSuccess,
            Action<string>    onError)
        {
            if (req.result != UnityWebRequest.Result.Success)
            {
                // Попытаться распарсить тело ошибки
                string raw = req.downloadHandler?.text ?? "";
                try
                {
                    var errWrapper = JsonUtility.FromJson<ApiResponse<TResponse>>(raw);
                    string msg = errWrapper?.error?.message ?? req.error;
                    onError?.Invoke($"[{req.responseCode}] {msg}");
                }
                catch
                {
                    onError?.Invoke($"[{req.responseCode}] {req.error}");
                }
                return;
            }

            try
            {
                var wrapper = JsonUtility.FromJson<ApiResponse<TResponse>>(req.downloadHandler.text);
                if (wrapper.ok)
                    onSuccess?.Invoke(wrapper.data);
                else
                    onError?.Invoke(wrapper.error?.message ?? "Unknown server error");
            }
            catch (Exception ex)
            {
                onError?.Invoke("Parse error: " + ex.Message);
            }
        }
    }
}

```

---

### GameApiService.cs

`MonoBehaviour`-компонент. Одна точка входа для всех вызовов. Каждый метод — корутина. Добавляется на GameObject в сцене.

```Csharp
using System;
using System.Collections;
using UnityEngine;

namespace StrategyGame.API
{
    /// <summary>
    /// Высокоуровневый сервис — одна точка входа для всех вызовов API.
    /// Использует ApiClient для HTTP и AuthSession для хранения токена.
    ///
    /// Использование:
    ///   GameApiService svc = GetComponent<GameApiService>();
    ///   StartCoroutine(svc.Login("alice@example.com", "password123", onOk, onErr));
    /// </summary>
    public class GameApiService : MonoBehaviour
    {
        // ─── Auth ────────────────────────────────────────────────

        /// <summary>POST /api/v1/auth/register</summary>
        public IEnumerator Register(
            string email, string password, string nickname,
            Action<RegisterResponse> onSuccess,
            Action<string>           onError)
        {
            var body = new RegisterRequest
            {
                email    = email,
                password = password,
                nickname = nickname
            };
            yield return StartCoroutine(
                ApiClient.Post<RegisterRequest, RegisterResponse>(
                    ApiConfig.Register, body, onSuccess, onError, withAuth: false));
        }

        /// <summary>POST /api/v1/auth/login — сохраняет токен в AuthSession</summary>
        public IEnumerator Login(
            string email, string password,
            Action<LoginResponse> onSuccess,
            Action<string>        onError)
        {
            var body = new LoginRequest { email = email, password = password };
            yield return StartCoroutine(
                ApiClient.Post<LoginRequest, LoginResponse>(
                    ApiConfig.Login, body,
                    resp =>
                    {
                        AuthSession.Save(resp);
                        onSuccess?.Invoke(resp);
                    },
                    onError,
                    withAuth: false));
        }

        public void Logout() => AuthSession.Clear();

        // ─── Profile ─────────────────────────────────────────────

        /// <summary>GET /api/v1/profile — требует авторизации</summary>
        public IEnumerator GetProfile(
            Action<ProfileResponse> onSuccess,
            Action<string>          onError)
        {
            yield return StartCoroutine(
                ApiClient.Get<ProfileResponse>(
                    ApiConfig.Profile, onSuccess, onError));
        }

        // ─── Events ──────────────────────────────────────────────

        /// <summary>POST /api/v1/events — отправить игровое событие</summary>
        public IEnumerator SendEvent(
            string eventType,
            object payload,
            Action<EventResponse> onSuccess,
            Action<string>        onError,
            int?  sessionId  = null,
            string clientTime = null)
        {
            var body = new EventRequest
            {
                eventId    = Guid.NewGuid().ToString(),   // уникальный UUID для дедупликации
                eventType  = eventType,
                sessionId  = sessionId,
                clientTime = clientTime ?? DateTime.UtcNow.ToString("o"),
                payload    = payload
            };
            yield return StartCoroutine(
                ApiClient.Post<EventRequest, EventResponse>(
                    ApiConfig.Events, body, onSuccess, onError));
        }

        // ─── Match ───────────────────────────────────────────────

        /// <summary>POST /api/v1/match/finish — завершить матч</summary>
        public IEnumerator FinishMatch(
            int  matchId,
            bool isWin,
            int  score,
            int  durationSeconds,
            Action<MatchFinishResponse> onSuccess,
            Action<string>              onError,
            int? powerDelta = null)
        {
            var body = new MatchFinishRequest
            {
                matchId = matchId,
                result  = new MatchResult
                {
                    isWin           = isWin,
                    score           = score,
                    durationSeconds = durationSeconds,
                    powerDelta      = powerDelta
                }
            };
            yield return StartCoroutine(
                ApiClient.Post<MatchFinishRequest, MatchFinishResponse>(
                    ApiConfig.MatchFinish, body, onSuccess, onError));
        }

        // ─── Leaderboard ─────────────────────────────────────────

        /// <summary>GET /api/v1/leaderboard?boardCode=...&season=...&limit=...</summary>
        public IEnumerator GetLeaderboard(
            Action<LeaderboardResponse> onSuccess,
            Action<string>              onError,
            string boardCode = ApiConfig.DefaultBoard,
            int    season    = ApiConfig.DefaultSeason,
            int    limit     = ApiConfig.DefaultLimit)
        {
            string url = $"{ApiConfig.Leaderboard}?boardCode={boardCode}&season={season}&limit={limit}";
            yield return StartCoroutine(
                ApiClient.Get<LeaderboardResponse>(url, onSuccess, onError));
        }
    }
}

```

---

### AuthController.cs

Управляет UI-логикой экрана входа/регистрации. После успешного входа загружает следующую сцену через `SceneManager.LoadScene()`.

```Csharp
using UnityEngine;
using UnityEngine.UI;
using TMPro;
using StrategyGame.API;

namespace StrategyGame.UI
{
    /// <summary>
    /// Контроллер экрана авторизации.
    ///
    /// Как подключить в сцене:
    ///   1. Создайте пустой GameObject "AuthManager".
    ///   2. Добавьте компоненты AuthController и GameApiService.
    ///   3. Привяжите UI-поля в Inspector.
    ///   4. Укажите onLoginSuccess: имя сцены которую надо загрузить после входа.
    /// </summary>
    public class AuthController : MonoBehaviour
    {
        [Header("Panels")]
        [SerializeField] private GameObject loginPanel;
        [SerializeField] private GameObject registerPanel;

        [Header("Login Fields")]
        [SerializeField] private TMP_InputField loginEmail;
        [SerializeField] private TMP_InputField loginPassword;
        [SerializeField] private Button         loginButton;

        [Header("Register Fields")]
        [SerializeField] private TMP_InputField regEmail;
        [SerializeField] private TMP_InputField regPassword;
        [SerializeField] private TMP_InputField regNickname;
        [SerializeField] private Button         registerButton;

        [Header("Status")]
        [SerializeField] private TMP_Text statusText;

        [Header("Navigation")]
        [SerializeField] private string sceneAfterLogin = "MainMenu";

        private GameApiService _api;

        private void Awake()
        {
            _api = GetComponent<GameApiService>();
            if (_api == null)
                _api = gameObject.AddComponent<GameApiService>();

            loginButton   .onClick.AddListener(OnLoginClick);
            registerButton.onClick.AddListener(OnRegisterClick);

            ShowLogin();
        }

        // ─── Переключение панелей ────────────────────────────────

        public void ShowLogin()
        {
            loginPanel   .SetActive(true);
            registerPanel.SetActive(false);
            SetStatus("");
        }

        public void ShowRegister()
        {
            loginPanel   .SetActive(false);
            registerPanel.SetActive(true);
            SetStatus("");
        }

        // ─── Логин ──────────────────────────────────────────────

        private void OnLoginClick()
        {
            string email = loginEmail.text.Trim();
            string pass  = loginPassword.text;

            if (string.IsNullOrEmpty(email) || string.IsNullOrEmpty(pass))
            {
                SetStatus("Заполните email и пароль", error: true);
                return;
            }

            SetStatus("Вход...");
            SetInteractable(false);

            StartCoroutine(_api.Login(email, pass,
                onSuccess: resp =>
                {
                    SetStatus($"Добро пожаловать, {resp.nickname}!");
                    SetInteractable(true);
                    // Переход в главное меню
                    UnityEngine.SceneManagement.SceneManager.LoadScene(sceneAfterLogin);
                },
                onError: err =>
                {
                    SetStatus(err, error: true);
                    SetInteractable(true);
                }));
        }

        // ─── Регистрация ─────────────────────────────────────────

        private void OnRegisterClick()
        {
            string email    = regEmail.text.Trim();
            string pass     = regPassword.text;
            string nickname = regNickname.text.Trim();

            if (string.IsNullOrEmpty(email) || string.IsNullOrEmpty(pass) || string.IsNullOrEmpty(nickname))
            {
                SetStatus("Заполните все поля", error: true);
                return;
            }

            SetStatus("Регистрация...");
            SetInteractable(false);

            StartCoroutine(_api.Register(email, pass, nickname,
                onSuccess: resp =>
                {
                    SetStatus("Аккаунт создан! Войдите.");
                    SetInteractable(true);
                    ShowLogin();
                },
                onError: err =>
                {
                    SetStatus(err, error: true);
                    SetInteractable(true);
                }));
        }

        // ─── Helpers ─────────────────────────────────────────────

        private void SetStatus(string msg, bool error = false)
        {
            if (statusText == null) return;
            statusText.text  = msg;
            statusText.color = error ? Color.red : Color.white;
        }

        private void SetInteractable(bool value)
        {
            loginButton   .interactable = value;
            registerButton.interactable = value;
        }
    }
}


```

---

### ProfileController.cs

Запрашивает профиль при `Start()` и отображает данные в TMP_Text-полях. Кнопка `Refresh` вызывает повторный запрос.

```Csharp
using UnityEngine;
using TMPro;
using StrategyGame.API;

namespace StrategyGame.UI
{
    /// <summary>
    /// Отображает профиль текущего игрока.
    ///
    /// Как подключить:
    ///   1. GameObject "ProfilePanel" → добавить ProfileController + GameApiService.
    ///   2. Привязать TMP-поля в Inspector.
    ///   3. Вызывается автоматически при Start().
    /// </summary>
    public class ProfileController : MonoBehaviour
    {
        [Header("UI Labels")]
        [SerializeField] private TMP_Text nicknameText;
        [SerializeField] private TMP_Text emailText;
        [SerializeField] private TMP_Text levelText;
        [SerializeField] private TMP_Text xpText;
        [SerializeField] private TMP_Text softCurrencyText;
        [SerializeField] private TMP_Text hardCurrencyText;
        [SerializeField] private TMP_Text statusText;

        private GameApiService _api;

        private void Awake()
        {
            _api = GetComponent<GameApiService>();
            if (_api == null)
                _api = gameObject.AddComponent<GameApiService>();
        }

        private void Start()
        {
            if (!AuthSession.IsLoggedIn)
            {
                SetStatus("Не авторизован", error: true);
                return;
            }
            Refresh();
        }

        /// <summary>Перезагрузить данные профиля с сервера.</summary>
        public void Refresh()
        {
            SetStatus("Загрузка...");
            StartCoroutine(_api.GetProfile(
                onSuccess: data =>
                {
                    ApplyProfile(data);
                    SetStatus("");
                },
                onError: err => SetStatus(err, error: true)));
        }

        // ─── Применение данных ────────────────────────────────────

        private void ApplyProfile(ProfileResponse data)
        {
            if (nicknameText)     nicknameText.text     = data.nickname;
            if (emailText)        emailText.text        = data.email;
            if (levelText)        levelText.text        = $"Уровень {data.progress?.level}";
            if (xpText)           xpText.text           = $"XP: {data.progress?.xp}";
            if (softCurrencyText) softCurrencyText.text = $"Монеты: {data.progress?.softCurrency}";
            if (hardCurrencyText) hardCurrencyText.text = $"Гемы: {data.progress?.hardCurrency}";
        }

        private void SetStatus(string msg, bool error = false)
        {
            if (statusText == null) return;
            statusText.text  = msg;
            statusText.color = error ? Color.red : Color.gray;
        }
    }
}

```

---

### LeaderboardController.cs

При `Start()` запрашивает таблицу лидеров и создаёт экземпляры `rowPrefab` для каждой строки.

```Csharp
using UnityEngine;
using UnityEngine.UI;
using TMPro;
using StrategyGame.API;

namespace StrategyGame.UI
{
    /// <summary>
    /// Загружает и отображает таблицу лидеров.
    ///
    /// Как подключить:
    ///   1. GameObject "LeaderboardPanel" → добавить LeaderboardController + GameApiService.
    ///   2. Создайте Prefab строки: TMP_Text rank | nickname | score.
    ///      Привяжите как rowPrefab.
    ///   3. Привяжите ScrollView.Content как rowContainer.
    /// </summary>
    public class LeaderboardController : MonoBehaviour
    {
        [Header("List")]
        [SerializeField] private Transform rowContainer;   // ScrollView > Viewport > Content
        [SerializeField] private GameObject rowPrefab;     // Prefab строки таблицы

        [Header("Status")]
        [SerializeField] private TMP_Text statusText;

        [Header("Settings")]
        [SerializeField] private string boardCode = ApiConfig.DefaultBoard;
        [SerializeField] private int    season    = ApiConfig.DefaultSeason;
        [SerializeField] private int    limit     = ApiConfig.DefaultLimit;

        private GameApiService _api;

        private void Awake()
        {
            _api = GetComponent<GameApiService>();
            if (_api == null)
                _api = gameObject.AddComponent<GameApiService>();
        }

        private void Start() => Refresh();

        /// <summary>Перезагрузить таблицу с сервера.</summary>
        public void Refresh()
        {
            SetStatus("Загрузка...");
            StartCoroutine(_api.GetLeaderboard(
                onSuccess: data =>
                {
                    Render(data);
                    SetStatus("");
                },
                onError: err => SetStatus(err, error: true),
                boardCode: boardCode,
                season:    season,
                limit:     limit));
        }

        // ─── Рендер строк ─────────────────────────────────────────

        private void Render(LeaderboardResponse data)
        {
            // Очистить старые строки
            foreach (Transform child in rowContainer)
                Destroy(child.gameObject);

            if (data.items == null || data.items.Count == 0)
            {
                SetStatus("Таблица пуста");
                return;
            }

            foreach (var item in data.items)
            {
                var row = Instantiate(rowPrefab, rowContainer);

                // Ищем TMP-поля в prefab по тегам или именам объектов
                var labels = row.GetComponentsInChildren<TMP_Text>();
                if (labels.Length >= 3)
                {
                    labels[0].text = item.rank.ToString();
                    labels[1].text = item.nickname;
                    labels[2].text = item.score.ToString();
                }
            }
        }

        private void SetStatus(string msg, bool error = false)
        {
            if (statusText == null) return;
            statusText.text  = msg;
            statusText.color = error ? Color.red : Color.gray;
        }
    }
}

```

---

### MatchController.cs

При `Start()` запускает таймер и отправляет `match_start` событие. Метод `EndMatch(isWin, score)` останавливает таймер и отправляет `POST /api/v1/match/finish`.

```Csharp
using System.Collections;
using UnityEngine;
using TMPro;
using StrategyGame.API;

namespace StrategyGame.Game
{
    /// <summary>
    /// Управляет жизненным циклом матча в Unity:
    ///   • Отправляет событие match_start
    ///   • Считает время матча
    ///   • При вызове EndMatch() — отправляет POST /api/v1/match/finish
    ///   • Показывает результат (награды) в UI
    ///
    /// Как подключить:
    ///   1. Создайте GameObject "MatchManager".
    ///   2. Добавьте MatchController + GameApiService.
    ///   3. Привяжите UI-поля в Inspector.
    ///   4. Назначьте matchId в Inspector (или задайте через код перед стартом матча).
    ///   5. Вызывайте EndMatch(isWin, score) из игровой логики.
    /// </summary>
    public class MatchController : MonoBehaviour
    {
        [Header("Match Settings")]
        [SerializeField] private int matchId = 9;    // id матча из БД (seed.sql → id=9)

        [Header("UI")]
        [SerializeField] private TMP_Text timerText;
        [SerializeField] private TMP_Text statusText;
        [SerializeField] private TMP_Text rewardText;
        [SerializeField] private GameObject resultPanel;

        private GameApiService _api;
        private float          _elapsed;
        private bool           _running;

        // ─── Lifecycle ───────────────────────────────────────────

        private void Awake()
        {
            _api = GetComponent<GameApiService>();
            if (_api == null)
                _api = gameObject.AddComponent<GameApiService>();

            if (resultPanel) resultPanel.SetActive(false);
        }

        private void Start()
        {
            StartMatch();
        }

        private void Update()
        {
            if (!_running) return;
            _elapsed += Time.deltaTime;
            if (timerText)
                timerText.text = FormatTime(_elapsed);
        }

        // ─── Старт матча ─────────────────────────────────────────

        private void StartMatch()
        {
            _elapsed = 0f;
            _running = true;

            SetStatus("Матч начался!");

            // Отправляем событие match_start
            var payload = new MatchStartPayload
            {
                matchId  = matchId,
                mode     = "pve",
                mapCode  = "coastal_siege",
                season   = 1
            };

            StartCoroutine(_api.SendEvent(
                eventType: "match_start",
                payload:   payload,
                onSuccess: _ => Debug.Log("[Match] match_start принят"),
                onError:   err => Debug.LogWarning("[Match] match_start ошибка: " + err)));
        }

        // ─── Конец матча (вызывать из игровой логики) ────────────

        /// <summary>
        /// Завершить матч. Вызывайте когда игра выиграна или проиграна.
        /// </summary>
        public void EndMatch(bool isWin, int score)
        {
            if (!_running) return;
            _running = false;

            int duration = Mathf.RoundToInt(_elapsed);
            SetStatus(isWin ? "Победа! Отправляем результат..." : "Поражение... Отправляем результат...");

            StartCoroutine(SendMatchFinish(isWin, score, duration));
        }

        private IEnumerator SendMatchFinish(bool isWin, int score, int duration)
        {
            yield return StartCoroutine(_api.FinishMatch(
                matchId:         matchId,
                isWin:           isWin,
                score:           score,
                durationSeconds: duration,
                onSuccess: data =>
                {
                    ShowResult(data);
                },
                onError: err =>
                {
                    SetStatus("Ошибка: " + err, error: true);
                    Debug.LogError("[Match] FinishMatch error: " + err);
                }));
        }

        // ─── Отображение результата ──────────────────────────────

        private void ShowResult(MatchFinishResponse data)
        {
            if (resultPanel) resultPanel.SetActive(true);

            string result = data.isWin ? "🏆 ПОБЕДА!" : "💀 ПОРАЖЕНИЕ";
            SetStatus(result);

            if (rewardText)
                rewardText.text = $"+{data.xpGained} XP\n+{data.softCurrencyGained} монет";

            Debug.Log($"[Match] Завершён. isWin={data.isWin} xp={data.xpGained} soft={data.softCurrencyGained}");
        }

        // ─── Helpers ─────────────────────────────────────────────

        private void SetStatus(string msg, bool error = false)
        {
            if (statusText == null) return;
            statusText.text  = msg;
            statusText.color = error ? Color.red : Color.white;
        }

        private static string FormatTime(float seconds)
        {
            int m = (int)seconds / 60;
            int s = (int)seconds % 60;
            return $"{m:00}:{s:00}";
        }
    }

    // ─── Payload для match_start ──────────────────────────────────

    [System.Serializable]
    public class MatchStartPayload
    {
        public int    matchId;
        public string mode;
        public string mapCode;
        public int    season;
    }
}

```

---

### EventSender.cs

Статический вспомогательный класс. Метод `EventSender.Send(this, "eventType", payload)` ищет `GameApiService` в сцене и отправляет произвольное событие.

```Csharp
using System;
using UnityEngine;
using StrategyGame.API;

namespace StrategyGame.Game
{
    /// <summary>
    /// Вспомогательный компонент для отправки произвольных игровых событий.
    ///
    /// Использование из любого MonoBehaviour:
    ///   EventSender.Send(this, "unit_deploy", new { matchId=9, unitCode="cavalry", amount=40 });
    /// </summary>
    public static class EventSender
    {
        /// <summary>
        /// Отправить событие через GameApiService, найденный в сцене.
        /// </summary>
        public static void Send(MonoBehaviour caller, string eventType, object payload,
            Action onSuccess = null, Action<string> onError = null)
        {
            var svc = UnityEngine.Object.FindObjectOfType<GameApiService>();
            if (svc == null)
            {
                Debug.LogError("[EventSender] GameApiService не найден в сцене!");
                return;
            }

            caller.StartCoroutine(svc.SendEvent(
                eventType: eventType,
                payload:   payload,
                onSuccess: resp =>
                {
                    Debug.Log($"[Event] {eventType} принят: {resp.eventId}");
                    onSuccess?.Invoke();
                },
                onError: err =>
                {
                    Debug.LogWarning($"[Event] {eventType} ошибка: {err}");
                    onError?.Invoke(err);
                }));
        }
    }

    // ─── Готовые payload-классы для жанровых событий ─────────────

    [Serializable]
    public class UnitDeployPayload
    {
        public int    matchId;
        public string unitCode;   // infantry / cavalry / archer / siege
        public int    amount;
        public int    atSecond;
    }

    [Serializable]
    public class UnitRetreatPayload
    {
        public int    matchId;
        public string unitCode;
        public int    lost;
        public int    atSecond;
    }

    [Serializable]
    public class BuildingUpgradePayload
    {
        public string buildingCode;  // barracks / farm / forge / walls
        public int    fromLevel;
        public int    toLevel;
        public int    goldCost;
    }
}

```

---

---

## Часть 7. Расширение проекта

### Добавить новое событие

1. В `EventSender.cs` добавить новый payload-класс:
```csharp
[Serializable]
public class MyEventPayload { public int matchId; public string data; }
```

2. Отправить из любого MonoBehaviour:
```csharp
EventSender.Send(this, "my_event", new MyEventPayload { matchId = 9, data = "test" });
```

### Сохранить токен между сессиями (PlayerPrefs)

```csharp
// При сохранении:
PlayerPrefs.SetString("token",    AuthSession.AccessToken);
PlayerPrefs.SetInt   ("userId",   AuthSession.UserId);
PlayerPrefs.SetString("nickname", AuthSession.Nickname);

// При загрузке:
if (PlayerPrefs.HasKey("token"))
{
    var fakeResp = new LoginResponse {
        accessToken = PlayerPrefs.GetString("token"),
        userId      = PlayerPrefs.GetInt("userId"),
        nickname    = PlayerPrefs.GetString("nickname")
    };
    AuthSession.Save(fakeResp);
}
```

### Смена сервера (dev / prod)

Измените `ApiConfig.BaseUrl`:
```csharp
// Локально:    "http://127.0.0.1:5000"
// По сети:     "http://192.168.1.100:5000"
// Production:  "https://your-server.com"
```

---

