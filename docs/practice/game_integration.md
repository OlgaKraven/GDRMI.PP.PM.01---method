
## 6. Интеграция с игрой (Unity / Web / Mobile) — подробная теория и задание (обновлено)

Раздел интеграции проверяет, что клиент связывается с backend **как инженерная система**, а не как набор разрозненных запросов: есть единый сетевой слой, токены живут централизованно, ответы парсятся по контракту, ошибки обрабатываются одинаково, события не теряются, повторная отправка не ломает данные, а `requestId` позволяет быстро связать запрос клиента с логами сервера.

[![REST API Tutorial](https://tse2.mm.bing.net/th/id/OIP.D4eit5gVSm8KYGcCIa7TeQHaEK?pid=Api)](https://www.javaguides.net/p/rest-api-tutorial.html?utm_source=chatgpt.com)

---

# 6.1. Что интегрируем: MVP-сценарии и почему они обязательны

Минимальный набор сценариев (MVP) должен закрывать полный цикл данных:

1. **Регистрация/логин** → клиент получает JWT → появляется авторизованный контекст
2. **Профиль** → клиент получает витрину состояния игрока (`user + progress`)
3. **События (телеметрия)** → клиент отправляет факты (`session_start`, `match_finish`)
4. **Лидерборд** → клиент читает агрегированные витрины (`/leaderboard`)
5. **Ошибки/лимиты/бан** → клиент корректно живёт при 400/401/403/409/429/500
6. **Дедупликация** → повторная отправка события не даёт двойных начислений

Почему это считается архитектурным минимумом:

* JWT показывает понимание stateless REST и того, что аутентификация идёт через заголовки.
* Profile проверяет чтение витрин и работу с DTO.
* Events проверяет журналирование фактов и устойчивость к сетевым сбоям.
* Leaderboard проверяет понимание “сырьё” vs “агрегаты”.
* Ошибки проверяют эксплуатацию: большая часть проблем в проде — это обработка краёв.

---

# 6.2. Архитектура сетевого слоя клиента: как должно быть устроено

## 6.2.1. Почему нельзя делать запросы “в кнопке” или “в MonoBehaviour напрямую”

Если запросы размазаны по UI/сценам:

* токен забудут добавить в часть запросов;
* обработка ошибок станет разной в разных местах;
* невозможно централизованно включить таймауты, ретраи, логирование;
* сложно тестировать и воспроизводить баги.

Требование: показать **минимальную архитектуру клиентского networking слоя**.

## 6.2.2. Минимальные компоненты (обязательные)

### 1) ApiClient (Networking Gateway)

Ответственность ApiClient:

* сбор URL (`baseUrl + path`)
* заголовки (`Authorization`, `Content-Type`, `X-Request-Id`)
* таймаут
* отправка запроса
* парсинг ответа в единый формат `ApiEnvelope<T>`
* возврат результата в бизнес-слой

Запрещено в ApiClient:

* UI-логика
* расчёт наград/геймплей
* хранение профиля игрока (кроме токена через TokenStore)

---

### 2) TokenStore (JWT storage + lifecycle)

Ответственность TokenStore:

* сохранить токен после login
* загрузить токен при старте
* очистить токен при logout
* (опционально) проверить срок жизни / refresh flow

Хранение (минимально допустимо для учебного MVP):

* Unity: `PlayerPrefs` (с пометкой “в проде нужен secure storage”)
* Web: `localStorage` (с пометкой “в проде предпочтительнее memory + refresh”)

---

### 3) EventQueue (offline-first)

Ответственность EventQueue:

* сформировать событие (`eventId` = UUID) и добавить в очередь **до отправки**
* отправить событие (по одному или пачкой)
* удалить из очереди только при подтверждённом успехе (`ok:true`)
* сохранить очередь (Unity: файл/PlayerPrefs; Web: localStorage; лучше: SQLite)
* реализовать политику retry/backoff, а также остановку на 401/403

---

### 4) ErrorMapper (policy layer)

Ответственность ErrorMapper:

* перевести `HTTP status` и `error.code` в действие клиента:

  * logout
  * retry
  * backoff
  * refresh profile
  * показать сообщение
* обеспечить единый UX ошибок для всех запросов

---

# 6.3. Контракт HTTP/JSON: что обязан соблюдать клиент и сервер

## 6.3.1. Единый формат ответа

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
    "requestId": "req_01HF..."
  }
}
```

### Почему `requestId` обязателен

`requestId` — связка клиент ↔ лог сервера:

* клиент фиксирует `requestId` в консоли/логе;
* по `requestId` быстро ищется запись на сервере;
* воспроизведение багов ускоряется.

Требование: клиент логирует `requestId` на каждый запрос.

---

## 6.3.2. События и прогресс — разные сущности

* `/events` — журнал фактов: “что произошло”
* `/profile` / `/progress` — витрина состояния: “что сейчас”

Если смешать:

* “прогресс как событие” → придётся пересчитывать всё из логов;
* “события как прогресс” → теряется история и аудит.

---

## 6.3.3. Дедупликация: `eventId` как защита от повторов

Повторная отправка в мобильной сети неизбежна. Поэтому:

* каждое событие должно иметь `eventId` (UUID)
* при повторной отправке `eventId` не меняется
* сервер делает дедупликацию по `eventId` (уникальность на стороне БД или отдельный реестр обработанных requestId/eventId)

Это защищает от:

* двойного начисления валюты
* двойных наград
* разъезжающихся статистик

---

# 6.4. Unity интеграция: инженерный минимум (с примерами)

## 6.4.1. DTO (обязательные классы)

```csharp
[Serializable]
public class ApiEnvelope<T>
{
    // Единый контракт API: ok + data/error
    public bool ok;
    public T data;
    public ApiError error;
}

[Serializable]
public class ApiError
{
    public string code;
    public string message;
    public string requestId;
}

[Serializable]
public class LoginRequest
{
    public string email;
    public string password;
}

[Serializable]
public class LoginData
{
    public string accessToken;
    public string tokenType;
    public int expiresInSeconds;
}

[Serializable]
public class ProfileData
{
    public UserDto user;
    public ProgressDto progress;
}

[Serializable]
public class UserDto
{
    public int id;
    public string email;
    public string nickname;
}

[Serializable]
public class ProgressDto
{
    public int level;
    public int xp;
    public int softCurrency;
    public int hardCurrency;
}
```

Комментарий: DTO разрывают связь UI ↔ JSON и обеспечивают типизацию.

---

## 6.4.2. TokenStore (пример)

```csharp
public interface ITokenStore
{
    string GetToken();
    void SaveToken(string token);
    void Clear();
}

public class PlayerPrefsTokenStore : ITokenStore
{
    private const string Key = "jwt_token";

    public string GetToken()
    {
        return PlayerPrefs.GetString(Key, "");
    }

    public void SaveToken(string token)
    {
        PlayerPrefs.SetString(Key, token ?? "");
        PlayerPrefs.Save();
    }

    public void Clear()
    {
        PlayerPrefs.DeleteKey(Key);
        PlayerPrefs.Save();
    }
}
```

---

## 6.4.3. ApiClient (UnityWebRequest, единый вход, логирование, requestId)

```csharp
using System;
using System.Text;
using System.Threading.Tasks;
using UnityEngine;
using UnityEngine.Networking;
using Newtonsoft.Json;

public class ApiClient
{
    private readonly string _baseUrl;
    private readonly ITokenStore _tokenStore;

    public ApiClient(string baseUrl, ITokenStore tokenStore)
    {
        _baseUrl = baseUrl.TrimEnd('/');
        _tokenStore = tokenStore;
    }

    // Универсальный метод: отправить запрос и получить ApiEnvelope<T>
    public async Task<ApiEnvelope<T>> SendAsync<T>(string method, string path, object body = null, int timeoutSeconds = 10)
    {
        var url = _baseUrl + path;
        var requestId = "req_" + Guid.NewGuid().ToString("N")[..8];

        using var req = new UnityWebRequest(url, method);
        req.timeout = timeoutSeconds;

        // Заголовки
        req.SetRequestHeader("Content-Type", "application/json");
        req.SetRequestHeader("X-Request-Id", requestId);

        var token = _tokenStore.GetToken();
        if (!string.IsNullOrEmpty(token))
            req.SetRequestHeader("Authorization", "Bearer " + token);

        // Тело запроса
        if (body != null)
        {
            var json = JsonConvert.SerializeObject(body);
            var bytes = Encoding.UTF8.GetBytes(json);
            req.uploadHandler = new UploadHandlerRaw(bytes);
        }

        req.downloadHandler = new DownloadHandlerBuffer();

        var start = Time.realtimeSinceStartup;
        var op = req.SendWebRequest();
        while (!op.isDone) await Task.Yield();
        var durationMs = (int)((Time.realtimeSinceStartup - start) * 1000);

        // requestId из ответа сервера (если сервер вернул)
        var serverRid = req.GetResponseHeader("X-Request-Id") ?? requestId;

        // Логирование сетевого слоя (обязательно)
        Debug.Log($"{method} {path} status={(int)req.responseCode} {durationMs}ms rid={serverRid}");

        // Если тело не JSON — это ошибка интеграции
        var text = req.downloadHandler.text;

        ApiEnvelope<T> envelope;
        try
        {
            envelope = JsonConvert.DeserializeObject<ApiEnvelope<T>>(text);
        }
        catch
        {
            return new ApiEnvelope<T>
            {
                ok = false,
                error = new ApiError
                {
                    code = "CLIENT_PARSE_ERROR",
                    message = "Response is not valid ApiEnvelope JSON",
                    requestId = serverRid
                }
            };
        }

        // Даже при HTTP 200 envelope.ok может быть false (контракт)
        // HTTP-статус дополнительно анализируется ErrorMapper-ом на уровне выше.
        if (envelope != null && envelope.error != null && string.IsNullOrEmpty(envelope.error.requestId))
            envelope.error.requestId = serverRid;

        return envelope;
    }
}
```

---

## 6.4.4. EventQueue (пример минимальной политики)

```csharp
[Serializable]
public class QueuedEvent
{
    public string eventId;        // UUID клиента (дедупликация)
    public string eventType;      // session_start / match_finish / ...
    public int? sessionId;
    public string clientTime;     // ISO string
    public object payload;        // конкретный payload
    public int attempt;           // попытки отправки
}

public class EventQueue
{
    private readonly ApiClient _api;
    private readonly ITokenStore _tokenStore;
    private readonly List<QueuedEvent> _queue = new();

    public EventQueue(ApiClient api, ITokenStore tokenStore)
    {
        _api = api;
        _tokenStore = tokenStore;
    }

    public void Enqueue(string eventType, int? sessionId, object payload)
    {
        _queue.Add(new QueuedEvent
        {
            eventId = Guid.NewGuid().ToString(),
            eventType = eventType,
            sessionId = sessionId,
            clientTime = DateTime.UtcNow.ToString("o"),
            payload = payload,
            attempt = 0
        });

        // В учебном MVP допустимо хранить очередь в памяти,
        // но для устойчивости лучше сохранять в файл/PlayerPrefs/SQLite.
    }

    public async Task DrainAsync()
    {
        for (int i = 0; i < _queue.Count; i++)
        {
            var ev = _queue[i];
            ev.attempt++;

            // Обязательное правило: при повторе eventId не меняем
            var body = new
            {
                eventId = ev.eventId,
                eventType = ev.eventType,
                sessionId = ev.sessionId,
                clientTime = ev.clientTime,
                payload = ev.payload
            };

            var res = await _api.SendAsync<object>("POST", "/api/v1/events", body);

            // Успех: удалить из очереди
            if (res.ok)
            {
                _queue.RemoveAt(i);
                i--;
                continue;
            }

            // Политики по ошибкам
            var code = res.error?.code ?? "UNKNOWN";
            if (code == "UNAUTHORIZED" || code == "USER_BANNED")
            {
                // 401/403: остановить очередь, очистить токен
                _tokenStore.Clear();
                return;
            }

            if (code == "RATE_LIMITED")
            {
                // 429: backoff (упрощённо)
                await Task.Delay(Math.Min(30000, (int)Math.Pow(2, ev.attempt) * 1000));
                i--;
                continue;
            }

            // 409 (duplicate / rejected): событие не удалять автоматически без политики,
            // обычно нужно запросить профиль для синхронизации и принять решение.
            // network/500: оставить в очереди
        }
    }
}
```

---

# 6.5. Web интеграция: инженерный минимум (с примерами)

## 6.5.1. Timeout (обязателен)

```js
export async function apiFetch(path, { method = "GET", body = null, token = null, timeoutMs = 8000 } = {}) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);

  const requestId = `req_${crypto.randomUUID().slice(0, 8)}`;
  const headers = { "Content-Type": "application/json", "X-Request-Id": requestId };
  if (token) headers["Authorization"] = `Bearer ${token}`;

  const start = performance.now();
  try {
    const res = await fetch(BASE_URL + path, {
      method,
      headers,
      body: body ? JSON.stringify(body) : null,
      signal: controller.signal,
    });

    const durationMs = Math.round(performance.now() - start);
    const serverRid = res.headers.get("X-Request-Id") || requestId;

    const json = await res.json();

    console.log(`${method} ${path} status=${res.status} ${durationMs}ms rid=${serverRid}`);

    if (!res.ok || !json.ok) {
      const err = json?.error || { code: "HTTP_ERROR", message: "Request failed", requestId: serverRid };
      if (!err.requestId) err.requestId = serverRid;
      throw { httpStatus: res.status, ...err };
    }

    return json.data;
  } finally {
    clearTimeout(timer);
  }
}
```

---

# 6.6. Mobile-специфика: обязательные знания

## 6.6.1. Offline-first для событий

Правило: события не теряются.

Если нет сети:

* событие кладётся в очередь
* отправляется при восстановлении
* удаляется только после подтверждённого `ok:true`

## 6.6.2. Backoff на 429

При `429 RATE_LIMITED`:

* запрещено спамить повтором
* включается экспоненциальная задержка 1 → 2 → 4 → 8 → … до 30 секунд

---

# 6.7. EventQueue: минимальная реализация логики (что должен уметь студент)

Очередь обязана реализовать политику:

1. сформировать событие (`eventId` UUID + payload)
2. положить в очередь
3. попытаться отправить
4. при успехе удалить
5. при 401/403 остановить очередь + logout
6. при 429 backoff
7. при network error оставить в очереди

---

# 6.8. Таблица “ошибка → действие клиента” (обязательна в отчёте)

| Ситуация           | HTTP / code                        | Действие клиента                                                   |
| ------------------ | ---------------------------------- | ------------------------------------------------------------------ |
| Данные невалидны   | 400 / VALIDATION_ERROR             | показать сообщение, логировать requestId                           |
| Нет токена/истёк   | 401 / UNAUTHORIZED                 | logout + экран login                                               |
| Бан пользователя   | 403 / USER_BANNED                  | показать “доступ запрещён”, остановить очередь                     |
| Конфликт состояния | 409 / EVENT_REJECTED или duplicate | обновить профиль (GET /profile), синхронизировать состояние        |
| Слишком часто      | 429 / RATE_LIMITED                 | backoff, блокировать повтор                                        |
| Ошибка сервера     | 500 / INTERNAL_ERROR               | показать “позже”, оставить событие в очереди, логировать requestId |

---

# Практическое задание (подробно)

## Вариант A: Unity (рекомендуется для геймдева)

Сделать:

1. Экран Login (email+password)
2. `POST /api/v1/auth/login` → сохранить JWT в TokenStore
3. Экран Profile (nickname + level + currency)
4. `GET /api/v1/profile` с JWT
5. Кнопка “Send session_start” → `POST /api/v1/events` (eventId UUID)
6. Кнопка “Send match_finish” → `POST /api/v1/events` (eventId UUID)
7. EventQueue: при network error событие остаётся в очереди
8. Ошибки:

   * `401` → очистить токен, вернуть на login
   * `403` → остановить очередь
   * `429` → backoff и повтор
   * `409` → обновить профиль, не начислять повторно

Артефакты:

* `ApiClient.cs`, `TokenStore.cs`, `EventQueue.cs`, DTO
* видео/скриншоты Unity Console (method/path/status/duration/requestId)
* README со сценарием проверки

---

## Вариант B: Web (быстрее для проверки)

Сделать:

1. HTML форма login
2. `apiFetch("/api/v1/auth/login")` → сохранить токен
3. Кнопка “Load profile” → `GET /api/v1/profile`
4. Кнопка “Send session_start” → `POST /api/v1/events` с eventId UUID
5. Очередь событий (минимум: массив + localStorage)
6. Обработка 401/429/409 как в таблице
7. Ошибки выводить на экран (не только console)

Артефакты:

* `api.js`, `auth.js`, `events.js`
* скрин DevTools Network (login, profile, events)
* README

---
