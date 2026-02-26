# 1. Теория. 6. Интеграция с игрой (Unity / Web / Mobile)

## 6.1. Что именно интегрируем (MVP-сценарии)

Студент должен уметь связать клиент с backend по цепочке:

1. **Регистрация/логин** → получить `accessToken` (JWT)
2. **Профиль** → `GET /profile` (витрина: user + progress)
3. **События** → `POST /events` (телеметрия: match_finish, currency_change…)
4. **Прогресс** → `POST /progress` (если вы обновляете snapshot отдельно)
5. **Лидерборд/стата** → `GET /leaderboard`, `GET /stats`
6. **Ошибки/лимиты/бан** → корректно обработать 400/401/403/409/429/500

Почему именно эти 6 шагов: они покрывают **весь контур “клиент → сервер → БД → обратно”**, и показывают, что студент понимает архитектуру, а не “просто отправил запрос”.

---

## 6.2. Архитектура клиентского сетевого слоя (как должно быть устроено)

### 6.2.1. Обязательные компоненты на клиенте

**1) ApiClient** — умеет отправлять HTTP, добавлять JWT, парсить ответы
**2) TokenStore** — хранит токен (в Unity: PlayerPrefs/secure storage; Web: memory/secure storage)
**3) EventQueue** — очередь событий при офлайне/ошибках
**4) ErrorMapper** — перевод `error.code` в пользовательские сообщения и действия (logout/backoff)

**Почему нельзя “прямо в кнопке сделать fetch()”:**

* токен забудут добавить в одном из запросов;
* ошибки будут обработаны по-разному;
* нельзя централизованно добавить ретраи, логирование, таймаут.

---

## 6.3. Контракт HTTP/JSON: что обязано соблюдаться

### 6.3.1. Единый формат ответа (чтобы клиент всегда знал, что парсить)

**Успех:**

```json
{ "ok": true, "data": { } }
```

**Ошибка:**

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

**Почему `requestId` обязателен:**
в практике это основной инструмент отладки: студент пишет в отчёте “получил ошибку с requestId=...”, а преподаватель/серверный лог легко находит причину.

---

### 6.3.2. Разделяем “события” и “прогресс” (не смешиваем)

* `/events` — **журнал фактов** (“что произошло”)
* `/progress` — **снимок состояния** (“что сейчас”)

**Почему это принципиально:**
если вы “пишете прогресс как событие” — вы теряете удобство витрин и начинаете пересчитывать всё из events.
если вы “пишете события как прогресс” — вы теряете историю (“почему валюта стала 1200?”).

---

### 6.3.3. Дедупликация событий: `eventId` как защита от повторов

**Проблема:** мобильная сеть → запрос улетел → ответ не пришёл → клиент повторил → сервер записал событие дважды.

**Решение:** `eventId` (UUID) + дедуп на сервере.

Пример payload:

```json
{
  "eventId": "c2a4f7be-0f0c-4e69-9f20-93f8c4f50a4d",
  "eventType": "match_finish",
  "sessionId": 555,
  "clientTime": "2026-02-26T12:20:10.000Z",
  "payload": { "score": 12450, "durationSeconds": 410 }
}
```

**Почему UUID лучше, чем “timestamp”:**
timestamp может совпасть/быть подменён, UUID — почти гарантирует уникальность.

---

## 6.4. Unity интеграция (UnityWebRequest) — “правильно, не хрупко”

### 6.4.1. Почему `JsonUtility` часто недостаточен

`JsonUtility`:

* плохо работает с `Dictionary`, `object payload`, массивами сложных типов,
* не поддерживает “универсальный payload”.

**На практике:** используйте **Newtonsoft.Json** (Json.NET for Unity) — это нормальная инженерная реальность.

---

### 6.4.2. DTO и универсальные ответы (Unity + Newtonsoft)

**DTO:**

```csharp
public class ApiResponse<T>
{
    public bool ok;
    public T data;
    public ApiError error;
}

public class ApiError
{
    public string code;
    public string message;
    public string requestId;
}
```

**Login:**

```csharp
public class LoginRequest { public string email; public string password; }
public class LoginData { public string accessToken; public string tokenType; public int expiresInSeconds; }
```

**Event:**

```csharp
public class EventRequest<TPayload>
{
    public string eventId;
    public string eventType;
    public int sessionId;
    public string clientTime;
    public TPayload payload;
}
```

---

### 6.4.3. Unity ApiClient: POST/GET, JWT, таймаут, парсинг JSON

```csharp
using System;
using System.Text;
using System.Collections;
using UnityEngine;
using UnityEngine.Networking;
using Newtonsoft.Json;

public class ApiClient : MonoBehaviour
{
    [SerializeField] private string baseUrl = "https://localhost:3000/api/v1";
    public string AccessToken { get; private set; }

    public void SetToken(string token)
    {
        AccessToken = token;
        PlayerPrefs.SetString("accessToken", token); // в проде лучше secure storage
    }

    public IEnumerator Post<TReq, TRes>(string path, TReq body, Action<TRes> onOk, Action<ApiError> onErr)
    {
        var url = baseUrl + path;
        var json = JsonConvert.SerializeObject(body);
        var bytes = Encoding.UTF8.GetBytes(json);

        using var req = new UnityWebRequest(url, "POST");
        req.uploadHandler = new UploadHandlerRaw(bytes);
        req.downloadHandler = new DownloadHandlerBuffer();
        req.SetRequestHeader("Content-Type", "application/json");
        if (!string.IsNullOrEmpty(AccessToken))
            req.SetRequestHeader("Authorization", "Bearer " + AccessToken);

        req.timeout = 15;

        var started = Time.realtimeSinceStartup;
        yield return req.SendWebRequest();
        var durationMs = (Time.realtimeSinceStartup - started) * 1000f;

        // 1) Сетевой уровень
        if (req.result != UnityWebRequest.Result.Success)
        {
            onErr?.Invoke(new ApiError { code = "NETWORK_ERROR", message = req.error, requestId = null });
            yield break;
        }

        // 2) JSON уровень
        ApiResponse<TRes> resp = null;
        try { resp = JsonConvert.DeserializeObject<ApiResponse<TRes>>(req.downloadHandler.text); }
        catch
        {
            onErr?.Invoke(new ApiError { code = "BAD_JSON", message = "Response is not valid JSON", requestId = null });
            yield break;
        }

        // 3) Контрактный уровень
        if (resp != null && resp.ok)
            onOk?.Invoke(resp.data);
        else
            onErr?.Invoke(resp?.error ?? new ApiError { code = "UNKNOWN_ERROR", message = "Unknown API error", requestId = null });

        Debug.Log($"[API] POST {path} status={req.responseCode} ({durationMs:F0} ms)");
    }

    public IEnumerator Get<TRes>(string path, Action<TRes> onOk, Action<ApiError> onErr)
    {
        var url = baseUrl + path;

        using var req = UnityWebRequest.Get(url);
        req.downloadHandler = new DownloadHandlerBuffer();
        if (!string.IsNullOrEmpty(AccessToken))
            req.SetRequestHeader("Authorization", "Bearer " + AccessToken);
        req.timeout = 15;

        yield return req.SendWebRequest();

        if (req.result != UnityWebRequest.Result.Success)
        {
            onErr?.Invoke(new ApiError { code = "NETWORK_ERROR", message = req.error, requestId = null });
            yield break;
        }

        ApiResponse<TRes> resp = null;
        try { resp = JsonConvert.DeserializeObject<ApiResponse<TRes>>(req.downloadHandler.text); }
        catch
        {
            onErr?.Invoke(new ApiError { code = "BAD_JSON", message = "Response is not valid JSON", requestId = null });
            yield break;
        }

        if (resp != null && resp.ok) onOk?.Invoke(resp.data);
        else onErr?.Invoke(resp?.error ?? new ApiError { code="UNKNOWN_ERROR", message="Unknown API error" });
    }
}
```

**Почему здесь три уровня ошибок:**

* сеть (нет интернета, DNS, TLS),
* JSON (сервер прислал не JSON),
* контракт (сервер прислал JSON, но `ok=false`).

---

### 6.4.4. Пример: Unity login → profile

```csharp
public class ProfileData {
    public UserDto user;
    public ProgressDto progress;
}
public class UserDto { public int id; public string email; public string nickname; public string createdAt; }
public class ProgressDto { public int level; public int xp; public int softCurrency; public int hardCurrency; }

public void DoLogin(ApiClient api, string email, string password)
{
    StartCoroutine(api.Post<LoginRequest, LoginData>(
        "/auth/login",
        new LoginRequest { email = email, password = password },
        data => { api.SetToken(data.accessToken); Debug.Log("Logged in"); },
        err => { Debug.LogError($"{err.code}: {err.message} req={err.requestId}"); }
    ));
}

public void LoadProfile(ApiClient api)
{
    StartCoroutine(api.Get<ProfileData>(
        "/profile",
        data => { Debug.Log($"Hello {data.user.nickname}, lvl={data.progress.level}"); },
        err => HandleApiError(err)
    ));
}

private void HandleApiError(ApiError err)
{
    if (err.code == "INVALID_CREDENTIALS" || err.code == "UNAUTHORIZED")
    {
        // разлогинить
    }
    Debug.LogError($"API error {err.code}: {err.message} req={err.requestId}");
}
```

---

## 6.5. Web интеграция (fetch) — “правильно, не ломко”

### 6.5.1. apiFetch с таймаутом (AbortController)

```js
const API_BASE = "https://localhost:3000/api/v1";
let accessToken = null;

export function setToken(t) { accessToken = t; }

export async function apiFetch(path, { method="GET", body=null, query=null, timeoutMs=15000 } = {}) {
  const url = new URL(API_BASE + path);
  if (query) Object.entries(query).forEach(([k,v]) => url.searchParams.set(k, String(v)));

  const ctrl = new AbortController();
  const timer = setTimeout(() => ctrl.abort(), timeoutMs);

  try {
    const res = await fetch(url.toString(), {
      method,
      headers: {
        "Content-Type": "application/json",
        ...(accessToken ? { Authorization: `Bearer ${accessToken}` } : {})
      },
      body: body ? JSON.stringify(body) : undefined,
      signal: ctrl.signal
    });

    let json;
    try { json = await res.json(); }
    catch { throw { code: "BAD_JSON", message: "Server returned non-JSON" }; }

    if (!res.ok || json.ok === false) throw (json.error ?? { code: "HTTP_ERROR", message: `HTTP ${res.status}` });

    return json.data;
  } finally {
    clearTimeout(timer);
  }
}
```

**Почему нужен AbortController:**
иначе fetch может “висеть” очень долго, пользователь думает, что игра зависла.

---

### 6.5.2. Пример: login + event + leaderboard

```js
// login
const login = await apiFetch("/auth/login", {
  method: "POST",
  body: { email, password }
});
setToken(login.accessToken);

// send event
await apiFetch("/events", {
  method: "POST",
  body: {
    eventId: crypto.randomUUID(),
    eventType: "session_start",
    sessionId: 555,
    clientTime: new Date().toISOString(),
    payload: { platform: "web", locale: "ru-RU" }
  }
});

// leaderboard
const lb = await apiFetch("/leaderboard", {
  query: { boardCode: "td_score", season: 1, limit: 50 }
});
console.log(lb.items);
```

---

## 6.6. Mobile-специфика (Android/iOS) — что студент обязан учитывать

### 6.6.1. Offline-first для событий (очередь)

**Правило:** события не теряем. Если сеть недоступна — кладём в очередь и отправляем позже.

Минимальная реализация:

* очередь в памяти + сохранение в файл/локальную БД (SQLite) раз в N секунд,
* flush при восстановлении сети.

**Почему:** в мобильной игре пользователь часто играет в транспорте/лифте.

---

### 6.6.2. Backoff при 429 (rate limit)

Если сервер отвечает:

* `429 RATE_LIMITED` → не спамить повтором.

**Правило:** экспоненциальная задержка:

* 1s → 2s → 4s → 8s (до потолка, например 30s).

**Почему:** иначе студент “сам устроит нагрузочное тестирование” серверу.

---

## 6.7. Очередь событий (EventQueue) — минимальный паттерн для практики

### 6.7.1. Что должен делать клиент

1. При клике/действии → сформировать `EventRequest` с `eventId`
2. Положить в очередь
3. Отправлять по одному/пакетами
4. При успехе → удалить
5. При 401/403 → остановить очередь и разлогинить
6. При 429 → backoff
7. При NETWORK_ERROR → оставить в очереди

---

### 6.7.2. Почему лучше отправлять события пачками (batch) — опционально

Можно сделать `POST /events/batch`:

* меньше overhead TLS/HTTP,
* быстрее на мобилках.

Но для MVP допустимо одиночное `/events`.

---

## 6.8. Таблица “ошибка → действие клиента” (очень полезно студентам)

| Ситуация             | HTTP / code            | Действие клиента                                                 |
| -------------------- | ---------------------- | ---------------------------------------------------------------- |
| Не тот JSON/поля     | 400 / VALIDATION_ERROR | показать “ошибка данных”, логировать requestId (это баг клиента) |
| Нет токена/истёк     | 401 / UNAUTHORIZED     | разлогинить, открыть экран login                                 |
| Пользователь забанен | 403 / USER_BANNED      | показать “доступ запрещён”, остановить отправку событий          |
| Конфликт состояния   | 409                    | обновить данные (GET /profile), показать “состояние изменилось”  |
| Слишком часто        | 429 / RATE_LIMITED     | backoff, временно отключить кнопку/отправку                      |
| Серверная ошибка     | 500                    | показать “попробуйте позже”, логировать requestId                |

---

## 6.9. Практические примеры сериализации payload под разные eventType

### 6.9.1. session_start (универсальный)

```json
{
  "eventId": "uuid",
  "eventType": "session_start",
  "sessionId": 555,
  "clientTime": "2026-02-26T12:10:05.120Z",
  "payload": {
    "platform": "unity",
    "clientVersion": "1.0.3",
    "device": { "os": "Android", "model": "Pixel 7" },
    "locale": "ru-RU"
  }
}
```

**Почему важно:** с этого события строятся retention, сегментация по платформам, диагностика версии клиента.

---

### 6.9.2. currency_change (аудит экономики)

```json
{
  "eventId": "uuid",
  "eventType": "currency_change",
  "sessionId": 555,
  "clientTime": "2026-02-26T12:12:00.000Z",
  "payload": {
    "currency": "soft",
    "delta": 300,
    "reason": "reward_match_finish"
  }
}
```

**Почему важно:** прогресс говорит “сколько стало”, а это событие — “почему стало”.

---

### 6.9.3. match_finish (запускает транзакции на сервере)

```json
{
  "eventId": "uuid",
  "eventType": "match_finish",
  "sessionId": 555,
  "clientTime": "2026-02-26T12:20:10.000Z",
  "payload": {
    "mode": "td",
    "mapCode": "forest_01",
    "season": 1,
    "isWin": true,
    "score": 12450,
    "durationSeconds": 410,
    "waveReached": 12
  }
}
```

**Почему выделяем:** это “итоговый факт”, который обновляет progress/stats/leaderboard.
