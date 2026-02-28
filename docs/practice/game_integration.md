# 2. Практика. 6. Интеграция Unity с ИС сопровождения игрового продукта (strategy-support-is)

## Смысл раздела

Этот раздел — пошаговое руководство по тому, как Unity-клиент подключается к Flask-бэкенду
`strategy-support-is`, обменивается данными по REST API и отображает результаты игроку.

Задача раздела: получить **работающую связку** Unity ↔ Flask, где:
- Игрок входит в аккаунт через Unity-форму
- Данные профиля и прогресса подтягиваются с сервера
- Игровые события отправляются в реальном времени
- Результат матча записывается на сервере, клиент получает начисленные награды
- Таблица лидеров отображается из реальной БД

---

##  1. Теория: как Unity общается с сервером

### 1.1. Общая схема взаимодействия

```
Unity-клиент
    │
    │  HTTP-запрос (JSON + JWT)
    ▼
Flask REST API (http://127.0.0.1:5000)
    │
    │  SQL
    ▼
MySQL 8.0 (strategy_db)
```

Unity — **клиент**, Flask — **сервер**. Сервер является источником истины:
он хранит все данные игрока, рассчитывает награды, проверяет права.
Unity только отправляет факты и отображает ответ.

---

### 1.2. Что такое REST API применительно к Unity

REST API — это набор URL-адресов (endpoint'ов) на сервере, к которым Unity обращается
по протоколу HTTP. Каждый endpoint выполняет одну конкретную функцию:

| Endpoint | Метод | Что делает |
|---|---|---|
| `/api/v1/auth/register` | POST | Создать новый аккаунт |
| `/api/v1/auth/login` | POST | Войти, получить JWT-токен |
| `/api/v1/profile` | GET | Получить профиль и прогресс игрока |
| `/api/v1/events` | POST | Отправить игровое событие |
| `/api/v1/match/finish` | POST | Завершить матч, получить награды |
| `/api/v1/leaderboard` | GET | Получить таблицу лидеров |
| `/health` | GET | Проверить что сервер работает |

---

### 1.3. Формат данных: JSON

Все запросы и ответы используются в формате JSON. Примеры:

**Запрос логина:**
```json
{ "email": "alice@example.com", "password": "password123" }
```

**Успешный ответ (все ответы имеют единую оболочку):**
```json
{
  "ok": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "userId": 1,
    "nickname": "Commander_Alice"
  }
}
```

**Ответ с ошибкой:**
```json
{
  "ok": false,
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Invalid credentials",
    "requestId": "req_a1b2c3d4"
  }
}
```

Поле `ok` всегда присутствует: `true` — успех, `false` — ошибка.
При успехе данные лежат в `data`, при ошибке — в `error`.

---

### 1.4. JWT — токен авторизации

**JWT (JSON Web Token)** — это строка, которую сервер выдаёт при успешном логине.
Она содержит зашифрованную информацию о пользователе (userId, roles) и срок действия.

После логина каждый защищённый запрос к API должен содержать заголовок:
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

Без этого заголовка сервер вернёт `401 UNAUTHORIZED`.

В Unity токен хранится в статическом классе `AuthSession` в памяти приложения.
`ApiClient` добавляет заголовок автоматически при каждом запросе, если токен есть.

---

### 1.5. Корутины — асинхронность в Unity

Unity не позволяет блокировать главный поток во время HTTP-запроса.
Для этого используются **корутины** (IEnumerator + StartCoroutine).

Схема работы:
```
Нажата кнопка "Войти"
    │
    ▼
StartCoroutine(api.Login(...))  ← запускает фоновое выполнение
    │
    │  кадры игры продолжают рендериться
    │
    ▼
HTTP-запрос выполнен
    │
    ▼
onSuccess(resp) или onError(err)  ← вызывается в главном потоке
```

Ключевой паттерн:
```csharp
StartCoroutine(api.Login("email", "pass",
    onSuccess: resp => Debug.Log("Добро пожаловать, " + resp.nickname),
    onError:   err  => Debug.LogError("Ошибка: " + err)));
```

---

### 1.6. UnityWebRequest — HTTP в Unity

`UnityWebRequest` — встроенный класс Unity для HTTP-запросов.
`ApiClient.cs` оборачивает его в удобные методы `Get<T>` и `Post<T,R>`.

Что делает `ApiClient` автоматически:
- Добавляет `Content-Type: application/json`
- Добавляет `Authorization: Bearer <token>` если пользователь залогинен
- Разбирает ответ из JSON в C#-класс
- Вызывает `onSuccess` или `onError` в зависимости от статуса ответа

---

##  2. Структура скриптов Unity

### 2.1. Дерево файлов

```
Assets/
  Scripts/
    API/
      ApiConfig.cs          ← адрес сервера и все URL
      ApiModels.cs          ← все DTO (классы запросов и ответов)
      AuthSession.cs        ← хранит JWT, userId, nickname в памяти
      ApiClient.cs          ← низкоуровневый HTTP: Get/Post через UnityWebRequest
      GameApiService.cs     ← высокоуровневый сервис: Login/Profile/Events/Match/Leaderboard
    UI/
      AuthController.cs     ← MonoBehaviour экрана входа/регистрации
      ProfileController.cs  ← MonoBehaviour экрана профиля
      LeaderboardController.cs ← MonoBehaviour таблицы лидеров
    Game/
      MatchController.cs    ← MonoBehaviour управления матчем
      EventSender.cs        ← статический хелпер отправки событий
```

### 2.2. Зависимости между слоями

```
AuthController, ProfileController, LeaderboardController, MatchController
                          │
                          │  вызывают
                          ▼
                   GameApiService        ← добавляется на GameObject как компонент
                          │
              ┌───────────┴───────────┐
              │                       │
              ▼                       ▼
          ApiClient              AuthSession
              │                       │
              ▼                       │
       UnityWebRequest           (статический,
       (встроен в Unity)          в памяти)
```

Правило: UI-скрипты не работают напрямую с `ApiClient` — только через `GameApiService`.

---

### 2.3. `ApiConfig.cs` — единая точка конфигурации

```csharp
namespace StrategyGame.API
{
    public static class ApiConfig
    {
        // Меняется только здесь при смене сервера
        public const string BaseUrl = "http://127.0.0.1:5000";

        // Все endpoint'ы
        public const string Register    = BaseUrl + "/api/v1/auth/register";
        public const string Login       = BaseUrl + "/api/v1/auth/login";
        public const string Profile     = BaseUrl + "/api/v1/profile";
        public const string Events      = BaseUrl + "/api/v1/events";
        public const string MatchFinish = BaseUrl + "/api/v1/match/finish";
        public const string Leaderboard = BaseUrl + "/api/v1/leaderboard";

        // Параметры по умолчанию для лидерборда
        public const string DefaultBoard  = "default";
        public const int    DefaultSeason = 1;
        public const int    DefaultLimit  = 10;
    }
}
```

**Когда менять BaseUrl:**
- Локальная разработка: `http://127.0.0.1:5000`
- Flask запущен на другой машине в сети: `http://192.168.1.100:5000`
- Production: `https://your-server.com`

---

### 2.4. `ApiModels.cs` — классы данных (DTO)

Каждый класс точно соответствует JSON-полям в запросах и ответах сервера.
Атрибут `[Serializable]` обязателен — без него `JsonUtility` не сможет преобразовать объект.

Имена полей в C#-классах должны совпадать с именами JSON-ключей сервера:
сервер возвращает `"accessToken"` → поле называется `public string accessToken`.

```csharp
// Запрос логина — POST /api/v1/auth/login
[Serializable]
public class LoginRequest
{
    public string email;
    public string password;
}

// Ответ логина — data: { accessToken, userId, nickname }
[Serializable]
public class LoginResponse
{
    public string accessToken;
    public int    userId;
    public string nickname;
}

// Ответ профиля — data: { userId, email, nickname, roles, progress }
[Serializable]
public class ProfileResponse
{
    public int           userId;
    public string        email;
    public string        nickname;
    public string[]      roles;
    public PlayerProgress progress;
}

[Serializable]
public class PlayerProgress
{
    public int level;
    public int xp;
    public int softCurrency;
    public int hardCurrency;
}
```

---

### 2.5. `AuthSession.cs` — хранение токена

Статический класс. Хранит данные текущей сессии в памяти приложения.

```csharp
public static class AuthSession
{
    public static string AccessToken { get; private set; }
    public static int    UserId      { get; private set; }
    public static string Nickname    { get; private set; }
    public static bool   IsLoggedIn  => !string.IsNullOrEmpty(AccessToken);

    // Вызывается после успешного логина
    public static void Save(LoginResponse resp)
    {
        AccessToken = resp.accessToken;
        UserId      = resp.userId;
        Nickname    = resp.nickname;
    }

    // Вызывается при выходе из аккаунта
    public static void Clear()
    {
        AccessToken = null;
        UserId      = 0;
        Nickname    = null;
    }
}
```

`ApiClient` читает `AuthSession.IsLoggedIn` и `AuthSession.AccessToken` автоматически
перед каждым запросом.

---

### 2.6. `ApiClient.cs` — HTTP-слой

Низкоуровневый компонент. Не используется напрямую из UI — только через `GameApiService`.

**Метод GET:**
```csharp
public static IEnumerator Get<TResponse>(
    string url,
    Action<TResponse> onSuccess,
    Action<string>    onError)
{
    using var req = UnityWebRequest.Get(url);

    // Добавить JWT если пользователь залогинен
    if (AuthSession.IsLoggedIn)
        req.SetRequestHeader("Authorization", "Bearer " + AuthSession.AccessToken);

    yield return req.SendWebRequest();

    if (req.result != UnityWebRequest.Result.Success)
    {
        onError?.Invoke($"[{req.responseCode}] {req.error}");
        yield break;
    }

    var wrapper = JsonUtility.FromJson<ApiResponse<TResponse>>(req.downloadHandler.text);
    if (wrapper.ok)
        onSuccess?.Invoke(wrapper.data);
    else
        onError?.Invoke(wrapper.error?.message ?? "Unknown error");
}
```

**Метод POST:**
```csharp
public static IEnumerator Post<TRequest, TResponse>(
    string    url,
    TRequest  body,
    Action<TResponse> onSuccess,
    Action<string>    onError,
    bool withAuth = true)
{
    string json = JsonUtility.ToJson(body);
    using var req = new UnityWebRequest(url, "POST");
    req.uploadHandler   = new UploadHandlerRaw(Encoding.UTF8.GetBytes(json));
    req.downloadHandler = new DownloadHandlerBuffer();
    req.SetRequestHeader("Content-Type", "application/json");

    if (withAuth && AuthSession.IsLoggedIn)
        req.SetRequestHeader("Authorization", "Bearer " + AuthSession.AccessToken);

    yield return req.SendWebRequest();

    // аналогично Get: разобрать ответ и вызвать onSuccess/onError
}
```

---

### 2.7. `GameApiService.cs` — высокоуровневый сервис

MonoBehaviour-компонент. Добавляется на GameObject в сцене.
Предоставляет удобные методы для всех операций с API.

```csharp
public class GameApiService : MonoBehaviour
{
    // Логин — автоматически сохраняет токен в AuthSession после успеха
    public IEnumerator Login(string email, string password,
        Action<LoginResponse> onSuccess, Action<string> onError) { ... }

    // Профиль
    public IEnumerator GetProfile(
        Action<ProfileResponse> onSuccess, Action<string> onError) { ... }

    // Событие — eventId генерируется автоматически через Guid.NewGuid()
    public IEnumerator SendEvent(string eventType, object payload,
        Action<EventResponse> onSuccess, Action<string> onError,
        int? sessionId = null, string clientTime = null) { ... }

    // Завершение матча
    public IEnumerator FinishMatch(int matchId, bool isWin, int score, int durationSeconds,
        Action<MatchFinishResponse> onSuccess, Action<string> onError,
        int? powerDelta = null) { ... }

    // Лидерборд
    public IEnumerator GetLeaderboard(
        Action<LeaderboardResponse> onSuccess, Action<string> onError,
        string boardCode = ApiConfig.DefaultBoard,
        int season = ApiConfig.DefaultSeason,
        int limit  = ApiConfig.DefaultLimit) { ... }
}
```

---

##  3. Подготовка: что нужно до начала работы в Unity

### 3.1. Запустить Flask-сервер

Перед любым тестированием в Unity сервер должен быть запущен.

1. Открыть терминал и перейти в папку проекта:
```bash
cd C:\Users\gvadoskr\Desktop\project\strategy-support-is
```

2. Активировать виртуальное окружение:
```bash
venv\Scripts\activate
```

3. Запустить сервер:
```bash
python wsgi.py
```

Ожидаемый вывод в терминале:
```
 * Running on http://127.0.0.1:5000
 * Debug mode: on
```

4. Проверить в браузере: открыть `http://127.0.0.1:5000/health`

Ожидаемый ответ:
```json
{ "ok": true, "data": { "status": "ok" } }
```

> ⚠️ Если браузер показывает ошибку — сервер не запущен. Нельзя тестировать Unity
> до тех пор, пока Flask не отвечает на `/health`.

---

### 3.2. Проверить наличие тестовых данных в БД

Для работы примеров нужны данные из `seed.sql`. Если они ещё не залиты:
```bash
mysql -u root -p strategy_db < db/schema.sql
mysql -u root -p strategy_db < seed.sql
```

Проверить в MySQL:
```sql
USE strategy_db;
SELECT id, email, nickname FROM users LIMIT 5;
SELECT id, user_id, status FROM matches WHERE id = 9;
```

Матч `id=9` должен иметь `status = 'started'` и `user_id = 1` (alice).

---

### 3.3. Создать проект Unity

1. Открыть Unity Hub.

2. Нажать **New Project**.

3. Выбрать шаблон **2D** (для простоты; можно 3D — разницы в скриптах нет).

4. Назвать проект: `StrategyClient`.

5. Нажать **Create project**. Дождаться инициализации.

---

### 3.4. Скопировать скрипты в проект

1. В окне Project (нижняя панель Unity) перейти в папку `Assets`.

2. Создать структуру папок (ПКМ → Create → Folder):
```
Assets/
  Scripts/
    API/
    UI/
    Game/
```

3. Скопировать C#-файлы в соответствующие папки:

| Файл | Папка назначения |
|---|---|
| `ApiConfig.cs` | `Assets/Scripts/API/` |
| `ApiModels.cs` | `Assets/Scripts/API/` |
| `AuthSession.cs` | `Assets/Scripts/API/` |
| `ApiClient.cs` | `Assets/Scripts/API/` |
| `GameApiService.cs` | `Assets/Scripts/API/` |
| `AuthController.cs` | `Assets/Scripts/UI/` |
| `ProfileController.cs` | `Assets/Scripts/UI/` |
| `LeaderboardController.cs` | `Assets/Scripts/UI/` |
| `MatchController.cs` | `Assets/Scripts/Game/` |
| `EventSender.cs` | `Assets/Scripts/Game/` |

4. Убедиться, что в Console (Window → Console) нет красных ошибок компиляции.

Типичные ошибки после копирования и их решения:

| Ошибка | Причина | Решение |
|---|---|---|
| `The type or namespace 'TMPro' could not be found` | TextMeshPro не установлен | Window → Package Manager → TextMeshPro → Install |
| `The type 'MonoBehaviour' could not be found` | Неверный namespace | Проверить что файл лежит в папке Assets |
| `CS0246: type not found` | Опечатка в имени класса | Проверить имена классов в файлах |

---

### 3.5. Установить TextMeshPro

Скрипты UI используют `TMP_Text` и `TMP_InputField` вместо стандартных `Text` и `InputField`.
TextMeshPro нужно установить отдельно.

1. Window → Package Manager.

2. В выпадающем списке вверху выбрать **Unity Registry**.

3. Найти **TextMeshPro** в списке.

4. Нажать **Install**.

5. Появится диалог **Import TMP Essentials** — нажать **Import TMP Essentials**.

После этого в Console должны пропасть все ошибки `TMPro not found`.

---

##  4. Сцена AuthScene — экран входа и регистрации

### 4.1. Что получится в результате

После выполнения всех шагов этого раздела у вас будет работающий экран входа:
- Поля email и пароль
- Кнопка «Войти» — отправляет запрос `POST /api/v1/auth/login`
- При успехе — переход в сцену `MainMenu`
- При ошибке — отображение текста ошибки под кнопкой
- Кнопка перехода к форме регистрации
- Форма регистрации — отправляет `POST /api/v1/auth/register`

---

### 4.2. Создать сцену

1. File → New Scene.

2. File → Save As... → создать папку `Assets/Scenes/` → имя `AuthScene`.

---

### 4.3. Создать Canvas

1. В Hierarchy (левая панель) ПКМ → UI → Canvas.

2. Выбрать созданный объект `Canvas` в Hierarchy.

3. В Inspector настроить Canvas Scaler:
- **UI Scale Mode:** Scale With Screen Size
- **Reference Resolution:** 1920 × 1080

---

### 4.4. Создать LoginPanel

1. В Hierarchy ПКМ на Canvas → UI → Panel. Переименовать в `LoginPanel`.

2. Настроить RectTransform: растянуть на весь экран
(в Inspector нажать кнопку якоря → выбрать Stretch All).

3. Добавить дочерние объекты в `LoginPanel`
(ПКМ на LoginPanel → UI → нужный тип):

| Имя объекта | Тип UI | Назначение |
|---|---|---|
| `Title` | Text - TextMeshPro | Заголовок «Войти» |
| `EmailInput` | Input Field - TextMeshPro | Поле ввода email |
| `PasswordInput` | Input Field - TextMeshPro | Поле ввода пароля |
| `LoginButton` | Button - TextMeshPro | Кнопка «Войти» |
| `StatusText` | Text - TextMeshPro | Статус / текст ошибки |
| `GoToRegisterButton` | Button - TextMeshPro | Кнопка «Нет аккаунта?» |

4. Настроить поле пароля — скрыть вводимые символы:
- Выбрать `PasswordInput` → в Inspector найти компонент **TMP_InputField**
- Поле **Content Type** → выбрать `Password`

5. Расположить элементы по экрану через RectTransform:
```
         [Title]                ← верхняя , по центру, крупный шрифт
         [EmailInput]           ← середина экрана, ширина ~400px
         [PasswordInput]        ← под email, та же ширина
         [LoginButton]          ← под паролем, зелёный
         [StatusText]           ← под кнопкой, мелкий, по умолчанию пустой
         [GoToRegisterButton]   ← внизу, мелкий серый текст
```

---

### 4.5. Создать RegisterPanel

1. В Hierarchy ПКМ на Canvas → UI → Panel. Переименовать в `RegisterPanel`.

2. Добавить дочерние объекты:

| Имя объекта | Тип UI | Назначение |
|---|---|---|
| `Title` | Text - TextMeshPro | Заголовок «Регистрация» |
| `EmailInput` | Input Field - TextMeshPro | Поле email |
| `PasswordInput` | Input Field - TextMeshPro | Поле пароля (Content Type: Password) |
| `NicknameInput` | Input Field - TextMeshPro | Поле никнейма |
| `RegisterButton` | Button - TextMeshPro | Кнопка «Зарегистрироваться» |
| `StatusText` | Text - TextMeshPro | Статус ошибки |
| `GoToLoginButton` | Button - TextMeshPro | Кнопка «Уже есть аккаунт?» |

---

### 4.6. Создать менеджер сцены

1. В Hierarchy ПКМ → Create Empty. Переименовать в `AuthManager`.

2. Выбрать `AuthManager` в Hierarchy.

3. В Inspector нажать **Add Component** и добавить по очереди:
- `AuthController`
- `GameApiService`

4. В компоненте `AuthController` привязать поля (перетащить из Hierarchy):

| Поле в Inspector | Откуда брать объект |
|---|---|
| **Login Panel** | `LoginPanel` |
| **Register Panel** | `RegisterPanel` |
| **Login Email** | `LoginPanel/EmailInput` |
| **Login Password** | `LoginPanel/PasswordInput` |
| **Login Button** | `LoginPanel/LoginButton` |
| **Reg Email** | `RegisterPanel/EmailInput` |
| **Reg Password** | `RegisterPanel/PasswordInput` |
| **Reg Nickname** | `RegisterPanel/NicknameInput` |
| **Register Button** | `RegisterPanel/RegisterButton` |
| **Status Text** | `LoginPanel/StatusText` |
| **Scene After Login** | Ввести строку `MainMenu` вручную |

---

### 4.7. Настроить кнопки переключения панелей

**Кнопка GoToRegisterButton (в LoginPanel):**

1. Выбрать `LoginPanel/GoToRegisterButton` в Hierarchy.
2. В Inspector → компонент **Button** → секция **On Click ()** → нажать **+**.
3. Перетащить `AuthManager` в появившееся поле объекта.
4. В выпадающем списке функции выбрать `AuthController → ShowRegister`.

**Кнопка GoToLoginButton (в RegisterPanel):**

1. Выбрать `RegisterPanel/GoToLoginButton`.
2. On Click () → **+** → перетащить `AuthManager` → выбрать `AuthController → ShowLogin`.

---

### 4.8. Добавить сцены в Build Settings

Чтобы `SceneManager.LoadScene("MainMenu")` сработал, сцена должна быть в списке сборки.

1. File → Build Settings.

2. Нажать **Add Open Scenes** — текущая `AuthScene` добавится в список.

3. Убедиться что `AuthScene` стоит на индексе 0 (первая в списке).
Это значит она запустится первой при старте игры.

Позже туда же нужно добавить `MainMenu` и `GameScene`.

---

### 4.9. Запустить и протестировать

1. Убедиться что Flask запущен (`python wsgi.py`).

2. Нажать **▶ Play** в Unity.

3. Ввести тестовые данные из `seed.sql`:
- Email: `alice@example.com`
- Password: `password123`

4. Нажать кнопку «Войти».

**Ожидаемый результат:**
- В Console (Window → Console) появится лог с `Authorization: Bearer eyJ...`
- Поле StatusText изменится на «Добро пожаловать, Commander_Alice!»
- Произойдёт переход в сцену `MainMenu` (если она уже создана)

**Проверка ошибочных сценариев:**

- Ввести неверный пароль → StatusText = «Invalid credentials» (красный)
- Ввести email `banned@example.com` → StatusText = «Account is banned»
- Остановить Flask и нажать «Войти» → StatusText = «Could not connect»

**Диагностика если кнопка не реагирует:**
- Проверить что `AuthManager` содержит компоненты `AuthController` и `GameApiService`
- Проверить что поле **Login Button** в Inspector привязано к объекту `LoginButton`
- Проверить что в Console нет красных ошибок компиляции

**Диагностика ошибки CORS:**
- Убедиться что `flask-cors` установлен: `pip install flask-cors`
- Перезапустить сервер

**Диагностика `401 UNAUTHORIZED`:**
- Проверить в MySQL: `SELECT * FROM users WHERE email='alice@example.com';`
- Если строк нет — залить seed.sql: `mysql -u root -p strategy_db < seed.sql`

---

##  5. Сцена MainMenu — профиль и лидерборд

### 5.1. Что получится в результате

После выполнения этого раздела:
- На экране отображается никнейм, уровень, XP и валюта из БД
- Нажатие кнопки «Обновить» запрашивает свежие данные с сервера
- Таблица лидеров загружается автоматически при открытии экрана
- Есть кнопка перехода в игровую сцену

---

### 5.2. Создать сцену MainMenu

1. File → New Scene.

2. File → Save As... → `Assets/Scenes/MainMenu`.

3. File → Build Settings → Add Open Scenes.

---

### 5.3. Создать Canvas

ПКМ в Hierarchy → UI → Canvas. Настройки — как в AuthScene
(Scale With Screen Size, Reference Resolution 1920×1080).

---

### 5.4. Панель профиля

1. ПКМ на Canvas → UI → Panel. Переименовать в `ProfilePanel`.

2. Добавить дочерние Text-TMP и кнопки:

| Объект | Тип | Отображает |
|---|---|---|
| `NicknameText` | Text - TextMeshPro | Никнейм игрока |
| `EmailText` | Text - TextMeshPro | Email |
| `LevelText` | Text - TextMeshPro | «Уровень 42» |
| `XpText` | Text - TextMeshPro | «XP: 85400» |
| `SoftCurrencyText` | Text - TextMeshPro | «Монеты: 12500» |
| `HardCurrencyText` | Text - TextMeshPro | «Гемы: 250» |
| `StatusText` | Text - TextMeshPro | Статус загрузки |
| `RefreshButton` | Button - TextMeshPro | Кнопка «Обновить» |
| `PlayButton` | Button - TextMeshPro | Кнопка «Играть» |

---

### 5.5. Настроить менеджер профиля

1. ПКМ в Hierarchy → Create Empty → `ProfileManager`.

2. Add Component:
- `ProfileController`
- `GameApiService`

3. В компоненте `ProfileController` привязать поля:

| Поле Inspector | Объект из Hierarchy |
|---|---|
| **Nickname Text** | `ProfilePanel/NicknameText` |
| **Email Text** | `ProfilePanel/EmailText` |
| **Level Text** | `ProfilePanel/LevelText` |
| **Xp Text** | `ProfilePanel/XpText` |
| **Soft Currency Text** | `ProfilePanel/SoftCurrencyText` |
| **Hard Currency Text** | `ProfilePanel/HardCurrencyText` |
| **Status Text** | `ProfilePanel/StatusText` |

4. Настроить кнопку «Обновить»:
- Выбрать `RefreshButton` → On Click () → **+**
- Перетащить `ProfileManager` → выбрать `ProfileController.Refresh`

5. Настроить кнопку «Играть» — переход в GameScene.
Создать отдельный скрипт `SceneLoader.cs`:
```csharp
using UnityEngine;
using UnityEngine.SceneManagement;

public class SceneLoader : MonoBehaviour
{
    public void LoadGame()   => SceneManager.LoadScene("GameScene");
    public void LoadMenu()   => SceneManager.LoadScene("MainMenu");
    public void LoadAuth()   => SceneManager.LoadScene("AuthScene");
}
```

Добавить `SceneLoader` на `ProfileManager`, затем:
- `PlayButton` → On Click () → `ProfileManager` → `SceneLoader.LoadGame`

---

### 5.6. Панель лидерборда

1. ПКМ на Canvas → UI → Panel → `LeaderboardPanel`.

2. Добавить в `LeaderboardPanel`:
- `TitleText` — Text-TMP, текст «Таблица лидеров»
- `StatusText` — Text-TMP, текст статуса загрузки
- `ScrollView` — UI → Scroll View

3. В `ScrollView` → Inspector → компонент **Scroll Rect**:
- **Horizontal:** отключить (таблица скроллится только вертикально)

---

### 5.7. Создать Prefab строки лидерборда

Каждая строка таблицы — отдельный Prefab. Скрипт создаёт экземпляры программно при загрузке.

1. В Hierarchy ПКМ → Create Empty → `LeaderboardRow`.

2. Add Component → Layout → **Horizontal Layout Group**.
Настройки: Child Alignment = Middle Left, Spacing = 20.

3. Добавить три дочерних Text-TMP в `LeaderboardRow`:

| Объект | Начальный текст | Примерная ширина |
|---|---|---|
| `RankText` | «1» | 60px |
| `NicknameText` | «Commander» | 280px |
| `ScoreText` | «9850» | 120px |

Настроить ширину через компонент **Layout Element** (Add Component → Layout → Layout Element)
на каждом из трёх объектов: установить **Preferred Width**.

4. Создать папку `Assets/Prefabs/`.

5. Перетащить `LeaderboardRow` из Hierarchy в папку `Assets/Prefabs/`.
Объект в Hierarchy станет синим — это признак Prefab.

6. Удалить `LeaderboardRow` из Hierarchy (теперь он нужен только как Prefab).

---

### 5.8. Настроить менеджер лидерборда

1. ПКМ в Hierarchy → Create Empty → `LeaderboardManager`.

2. Add Component:
- `LeaderboardController`
- `GameApiService`

3. В `LeaderboardController` привязать:

| Поле Inspector | Что привязать |
|---|---|
| **Row Container** | `LeaderboardPanel/ScrollView/Viewport/Content` |
| **Row Prefab** | Перетащить Prefab `LeaderboardRow` из Assets/Prefabs |
| **Status Text** | `LeaderboardPanel/StatusText` |
| **Board Code** | Ввести строку `default` |
| **Season** | Ввести `1` |
| **Limit** | Ввести `10` |

---

### 5.9. Протестировать сцену MainMenu

> Важно: `AuthScene` должна быть первой в Build Settings (индекс 0),
> `MainMenu` — второй. Запуск напрямую из `MainMenu` без логина даст пустые данные
> и ошибки 401, потому что `AuthSession.IsLoggedIn = false`.

1. Запустить Flask: `python wsgi.py`

2. В Unity: убедиться что в Build Settings открыта `AuthScene` первой.

3. ▶ Play → ввести `alice@example.com` / `password123` → войти.

**Ожидаемые результаты в MainMenu:**
- `NicknameText` = «Commander_Alice»
- `LevelText` = «Уровень 42»
- `XpText` = «XP: 85400»
- `SoftCurrencyText` = «Монеты: 12500»
- В таблице лидеров появятся строки:
  - 1. Siege_Eve — 9850
  - 2. WarLord_Carol — 8720
  - 3. Empress_Grace — 7600
  - ...

**Если профиль не загружается:**
- Console: строка `[401]` → нет токена → нужно зайти через AuthScene
- Console: строка `[404]` → пользователь не найден → залить seed.sql

**Если лидерборд пустой:**
- Console: строка `[400]` → неверные параметры → проверить поля Board Code / Season / Limit в Inspector
- Проверить в MySQL: `SELECT * FROM leaderboard_scores WHERE board_code='default';`
  Если пусто — залить seed.sql

---

##  6. Сцена GameScene — матч и события

### 6.1. Что получится в результате

После выполнения этого раздела:
- При старте сцены автоматически отправляется событие `match_start` на сервер
- Идёт таймер матча
- Кнопки «Победа» и «Поражение» завершают матч через `POST /api/v1/match/finish`
- После ответа сервера отображаются начисленные XP и монеты

---

### 6.2. Создать сцену GameScene

1. File → New Scene → Save As → `Assets/Scenes/GameScene`.

2. File → Build Settings → Add Open Scenes.

После этого порядок в Build Settings должен быть:
```
0: AuthScene
1: MainMenu
2: GameScene
```

---

### 6.3. Создать Canvas

ПКМ в Hierarchy → UI → Canvas.

---

### 6.4. Создать UI элементы

Добавить в Canvas:

| Объект | Тип | Назначение |
|---|---|---|
| `TimerText` | Text - TMP | Таймер матча в формате `MM:SS` |
| `StatusText` | Text - TMP | Статус (например, «Матч начался!») |
| `WinButton` | Button - TMP | Кнопка «Победа (1850 очков)» |
| `LoseButton` | Button - TMP | Кнопка «Поражение (300 очков)» |
| `ResultPanel` | Panel | Панель результата (изначально скрытая) |

В `ResultPanel` добавить:
- `RewardText` — Text-TMP: будет показывать «+100 XP  +50 монет»

---

### 6.5. Создать менеджер матча

1. ПКМ в Hierarchy → Create Empty → `MatchManager`.

2. Add Component:
- `MatchController`
- `GameApiService`

3. В `MatchController` привязать поля:

| Поле Inspector | Значение или объект |
|---|---|
| **Match Id** | `9` (матч из seed.sql, принадлежит alice) |
| **Timer Text** | `TimerText` |
| **Status Text** | `StatusText` |
| **Reward Text** | `ResultPanel/RewardText` |
| **Result Panel** | `ResultPanel` |

4. В файле `MatchController.cs` добавить два публичных метода:

```csharp
// Добавить в класс MatchController:
public void EndMatchWin()  => EndMatch(isWin: true,  score: 1850);
public void EndMatchLose() => EndMatch(isWin: false, score:  300);
```

5. Настроить кнопки:
- `WinButton` → On Click () → `MatchManager` → `MatchController.EndMatchWin`
- `LoseButton` → On Click () → `MatchManager` → `MatchController.EndMatchLose`

---

### 6.6. Как работает MatchController (логика)

```
Start()
  │
  ├─ _running = true, _elapsed = 0
  ├─ StatusText = "Матч начался!"
  └─ Отправить событие "match_start" через EventSender
         └─ POST /api/v1/events { eventType: "match_start", eventId: <uuid>, payload: {...} }

Update() каждый кадр:
  └─ Если _running: _elapsed += Time.deltaTime, обновить TimerText ("01:23")

EndMatch(isWin, score)
  │
  ├─ _running = false (таймер остановлен)
  ├─ duration = round(_elapsed) в секундах
  ├─ StatusText = "Отправляем результат..."
  │
  └─ StartCoroutine(api.FinishMatch(matchId=9, isWin, score, duration, ...))
                           │
                           │  POST /api/v1/match/finish
                           │  { "matchId": 9, "result": { "isWin": true, "score": 1850, "durationSeconds": 73 } }
                           │
                           ▼
                    Сервер отвечает:
                    { "ok": true, "data": { "xpGained": 100, "softCurrencyGained": 50 } }
                           │
                           ▼
                    ShowResult(data)
                      ├─ ResultPanel.SetActive(true)
                      ├─ RewardText = "+100 XP\n+50 монет"
                      └─ StatusText = "🏆 ПОБЕДА!" или "💀 ПОРАЖЕНИЕ"
```

---

### 6.7. Как сервер рассчитывает награды

Награды фиксированы в коде сервера (`app/services/match_service.py`):

```python
_XP_WIN   = 100   # XP за победу
_XP_LOSS  = 30    # XP за поражение
_SOFT_WIN = 50    # монеты за победу
_SOFT_LOSS = 10   # монеты за поражение
```

**Клиент не присылает суммы наград** — только факты (`isWin`, `score`, `durationSeconds`).
Все расчёты выполняет сервер. Это защита от читерства.

Сервер дополнительно в одной транзакции:
1. Проверяет что матч `id=9` принадлежит текущему пользователю из JWT
2. Проверяет что статус матча = `started`
3. Обновляет: `matches` (статус → finished), `battle_results`, `player_progress`, `statistics_daily`, `leaderboard_scores`
4. При любой ошибке откатывает все изменения (`rollback`)

---

### 6.8. Протестировать GameScene

1. Запустить Flask: `python wsgi.py`

2. Запустить AuthScene → войти как `alice@example.com`.

3. Нажать «Играть» → перейти в GameScene.

4. В Console должно появиться:
```
[Match] match_start принят
```

Это значит событие `match_start` успешно записалось в БД.

5. Подождать несколько секунд — убедиться что TimerText считает: `00:01`, `00:02`...

6. Нажать «Победа».

**Ожидаемый результат:**
- ResultPanel появляется
- RewardText = «+100 XP  +50 монет»
- StatusText = «🏆 ПОБЕДА!»
- Console: `[Match] Завершён. isWin=True xp=100 soft=50`

7. Нажать «Победа» ещё раз.

**Ожидаемый результат:**
- Console: `[Match] FinishMatch error: [409] Match is not in started state`
- StatusText = «Ошибка: ...»

Это корректное поведение — матч уже завершён. Для повторного тестирования:
```sql
UPDATE matches SET status='started', ended_at=NULL WHERE id=9;
```

---

##  7. Отправка событий из любого места игры

### 7.1. Что такое игровые события

**Игровые события** — факты о действиях игрока, которые записываются в таблицу `game_events`.
Используются для аналитики, статистики и балансировки игры.

Примеры событий для стратегической игры:

| `eventType` | Когда отправлять |
|---|---|
| `match_start` | В начале матча |
| `unit_deploy` | Игрок разместил отряд |
| `unit_retreat` | Отряд отступил |
| `building_upgrade` | Игрок улучшил здание |
| `resource_collect` | Собраны ресурсы |

---

### 7.2. Как отправить событие из любого MonoBehaviour

```csharp
using StrategyGame.Game;

// Пример: игрок разместил кавалерию
var payload = new UnitDeployPayload
{
    matchId  = 9,
    unitCode = "cavalry",
    amount   = 40,
    atSecond = 15
};

EventSender.Send(this, "unit_deploy", payload,
    onSuccess: () => Debug.Log("Событие записано"),
    onError:   err => Debug.LogWarning("Ошибка отправки: " + err));
```

`EventSender.Send` автоматически:
- Генерирует уникальный `eventId` через `Guid.NewGuid().ToString()`
- Находит `GameApiService` в сцене
- Отправляет `POST /api/v1/events`

---

### 7.3. Дедупликация событий

Каждое событие имеет уникальный `eventId` (UUID). Если одно и то же событие
отправить дважды — сервер отклонит второй запрос с кодом `409 EVENT_REJECTED`.

`EventSender` генерирует новый UUID при каждом вызове, поэтому дублей
при нормальной работе не будет. Дедупликация защищает от:
- Повторной отправки при прерывании соединения
- Баги двойной отправки в клиентском коде

---

### 7.4. Добавить новый тип события

1. В файле `EventSender.cs` добавить новый payload-класс:

```csharp
[Serializable]
public class ResourceCollectPayload
{
    public string resourceType;  // "gold" / "wood" / "food"
    public int    amount;
    public int    matchId;
}
```

2. Вызвать из нужного места в игре:

```csharp
EventSender.Send(this, "resource_collect", new ResourceCollectPayload
{
    resourceType = "gold",
    amount       = 250,
    matchId      = 9
});
```

3. Проверить в MySQL что событие записалось:

```sql
SELECT event_type, payload_json, created_at
FROM game_events
ORDER BY created_at DESC
LIMIT 5;
```

---

##  8. Сохранение токена между сессиями (PlayerPrefs)

По умолчанию `AuthSession` хранит токен только в памяти.
При перезапуске приложения пользователю придётся входить снова.

Чтобы сохранять токен между сессиями, используйте `PlayerPrefs`:

### 8.1. Сохранение после успешного логина

Добавить в `OnLoginClick` в `AuthController.cs` сразу после `AuthSession.Save(resp)`:

```csharp
PlayerPrefs.SetString("token",    AuthSession.AccessToken);
PlayerPrefs.SetInt   ("userId",   AuthSession.UserId);
PlayerPrefs.SetString("nickname", AuthSession.Nickname);
PlayerPrefs.Save();
```

### 8.2. Восстановление при старте приложения

Создать скрипт `AppBootstrap.cs` и добавить его на GameObject в `AuthScene`:

```csharp
using UnityEngine;
using UnityEngine.SceneManagement;
using StrategyGame.API;

public class AppBootstrap : MonoBehaviour
{
    void Awake()
    {
        if (PlayerPrefs.HasKey("token"))
        {
            // Токен сохранён — восстановить сессию
            var resp = new LoginResponse
            {
                accessToken = PlayerPrefs.GetString("token"),
                userId      = PlayerPrefs.GetInt("userId"),
                nickname    = PlayerPrefs.GetString("nickname")
            };
            AuthSession.Save(resp);

            // Пропустить экран логина
            SceneManager.LoadScene("MainMenu");
        }
        // Если токена нет — остаться на AuthScene
    }
}
```

### 8.3. Выход из аккаунта

Добавить метод в кнопку «Выйти» в MainMenu:

```csharp
public void LogOut()
{
    PlayerPrefs.DeleteKey("token");
    PlayerPrefs.DeleteKey("userId");
    PlayerPrefs.DeleteKey("nickname");
    AuthSession.Clear();
    SceneManager.LoadScene("AuthScene");
}
```

> ⚠️ Токен хранится как обычная строка в PlayerPrefs — это небезопасно
> на устройствах с root/jailbreak. Для production-сборок используйте
> платформенные Keychain (iOS) / Keystore (Android) API.

---

##  9. Смена сервера (dev / stage / production)

В файле `ApiConfig.cs` изменить только одну строку `BaseUrl`:

```csharp
// Локальная разработка (Flask запущен на той же машине):
public const string BaseUrl = "http://127.0.0.1:5000";

// Flask запущен на другой машине в локальной сети:
public const string BaseUrl = "http://192.168.1.100:5000";

// Production-сервер (HTTPS обязателен для публичного деплоя):
public const string BaseUrl = "https://strategy-api.your-domain.com";
```

`UnityWebRequest` поддерживает HTTPS нативно — в коде Unity ничего менять не нужно.
На сервере должен быть валидный SSL-сертификат.

---

##  10. Отладка

### 10.1. Инструменты отладки

| Инструмент | Как открыть | Что показывает |
|---|---|---|
| Unity Console | Window → Console | Debug.Log, ошибки C#, ответы API |
| Flask-терминал | Терминал где запущен `python wsgi.py` | Все HTTP-запросы с кодами ответов и временем |
| MySQL | `mysql -u root -p strategy_db` | Содержимое таблиц после тестов |
| test_client.html | Двойной клик на файле в Explorer | Ручное тестирование API без Unity |

---

### 10.2. Таблица частых ошибок

| Ошибка в Console | Причина | Решение |
|---|---|---|
| `[0] Could not connect` | Flask не запущен | `python wsgi.py` в терминале |
| `[401] JWT is missing` | Нет токена | Войти через AuthScene, не запускать MainMenu напрямую |
| `[403] Insufficient role` | Роль пользователя не совпадает | Проверить роли: `SELECT * FROM user_roles WHERE user_id=1;` |
| `[404] Match not found` | Неверный matchId | Проверить что в Inspector стоит `matchId = 9` |
| `[409] Match is not in started state` | Матч уже завершён | `UPDATE matches SET status='started', ended_at=NULL WHERE id=9;` |
| `[409] Duplicate eventId` | Повтор UUID | Ошибка логики — проверить что `EventSender` не вызывается дважды |
| `[500] Unexpected error` | Ошибка на сервере | Найти traceback в терминале Flask |
| `JsonException` или `null data` | Поле DTO не совпадает с JSON | Проверить имена полей в `ApiModels.cs` — должны точно совпадать |
| `NullReferenceException` | Поле не привязано в Inspector | Проверить все поля компонентов через Inspector |
| `CORS error` | Flask отклоняет запрос | `pip install flask-cors`, перезапустить сервер |

---

### 10.3. Проверка данных в БД после теста

После нажатия «Победа» в GameScene выполнить в MySQL:

```sql
-- Матч должен стать finished
SELECT id, status, ended_at FROM matches WHERE id = 9;

-- Результат боя должен быть записан
SELECT match_id, is_win, score, duration_seconds FROM battle_results WHERE match_id = 9;

-- XP и монеты alice должны вырасти
SELECT xp, soft_currency FROM player_progress WHERE user_id = 1;

-- Дневная статистика обновлена
SELECT day, wins, losses, score_sum
FROM statistics_daily
WHERE user_id = 1
ORDER BY day DESC LIMIT 1;

-- Лидерборд обновлён (должен показать новый score)
SELECT score, updated_at
FROM leaderboard_scores
WHERE user_id = 1 AND board_code = 'default';
```

---

### 10.4. Чтение логов Flask

В терминале Flask каждый запрос от Unity выводится в формате:
```
req_a1b2c3d4  user=1  POST /api/v1/match/finish  200  45ms
req_b2c3d4e5  user=1  GET  /api/v1/profile       200  12ms
req_c3d4e5f6  user=-  POST /api/v1/auth/login    401  8ms
```

Поля: `requestId | userId | метод | путь | статус | время_мс`

`user=-` означает что запрос без токена или с невалидным токеном.

Если видите много строк `401` после входа — скорее всего `AuthSession`
не сохраняется между сценами. Убедитесь что переход из AuthScene, а не
прямой запуск MainMenu через ▶ Play.

---

##  11. Тестовые учётные данные (из seed.sql)

Все пользователи имеют пароль `password123`.

| Email | Никнейм | Роли | Особенности |
|---|---|---|---|
| `alice@example.com` | Commander_Alice | player, admin | Матч id=9 в статусе started |
| `bob@example.com` | General_Bob | player, moderator | — |
| `carol@example.com` | WarLord_Carol | player | 2-е место в лидерборде |
| `dave@example.com` | Tactician_Dave | player | — |
| `eve@example.com` | Siege_Eve | player | 1-е место (9850 очков) |
| `frank@example.com` | Scout_Frank | player | — |
| `grace@example.com` | Empress_Grace | player | 3-е место (7600 очков) |
| `hank@example.com` | Iron_Hank | player | — |
| `ivan@example.com` | Phantom_Ivan | player | — |
| `banned@example.com` | Cheater_X | player | `is_banned=1` → вернёт 403 при логине |

> ⚠️ Для тестирования `POST /api/v1/match/finish` обязательно войти как **alice** —
> матч `id=9` принадлежит ей. При попытке завершить этот матч под другим аккаунтом
> сервер вернёт `403 FORBIDDEN`.

---

## Итог: порядок полной сборки и проверки

Ниже — минимальный путь от нуля до работающей связки Unity ↔ Flask.

### 1. Подготовка Flask-сервера
```bash
cd C:\Users\gvadoskr\Desktop\project\strategy-support-is
venv\Scripts\activate
mysql -u root -p strategy_db < db/schema.sql
mysql -u root -p strategy_db < seed.sql
python wsgi.py
```
Проверка в браузере: `http://127.0.0.1:5000/health` → `{"ok": true}`

### 2. Подготовка Unity-проекта
1. Создать проект `StrategyClient` в Unity Hub (шаблон 2D)
2. Установить TextMeshPro (Window → Package Manager → TMP → Install → Import Essentials)
3. Создать папки `Assets/Scripts/API`, `UI`, `Game`
4. Скопировать все 10 `.cs`-файлов
5. Убедиться что Console без красных ошибок компиляции

### 3. AuthScene (экран входа)
1. Создать Canvas с LoginPanel и RegisterPanel
2. Создать `AuthManager` → Add Component: AuthController + GameApiService
3. Привязать все поля через Inspector
4. Добавить в Build Settings на индекс 0
5. ▶ Play → войти как `alice@example.com` / `password123` → «Добро пожаловать»

### 4. MainMenu (профиль + лидерборд)
1. Создать ProfilePanel с TMP-полями
2. Создать LeaderboardPanel + ScrollView
3. Создать Prefab `LeaderboardRow` (3 TMP-поля + Horizontal Layout Group)
4. Создать `ProfileManager` (ProfileController + GameApiService) и `LeaderboardManager` (LeaderboardController + GameApiService)
5. Привязать все поля в Inspector
6. Добавить в Build Settings
7. ▶ Play из AuthScene → войти → перейти в MainMenu → профиль и лидерборд загрузятся

### 5. GameScene (матч)
1. Создать кнопки Win/Lose, TimerText, StatusText, ResultPanel с RewardText
2. Создать `MatchManager` → Add Component: MatchController + GameApiService
3. Установить в Inspector: `matchId = 9`
4. Привязать кнопки Win/Lose к `EndMatchWin()` и `EndMatchLose()`
5. Добавить в Build Settings
6. ▶ Play из AuthScene → войти → GameScene → нажать «Победа» → `+100 XP`, `+50 монет`

### Итоговая проверка в БД
```sql
SELECT status, ended_at FROM matches WHERE id=9;
-- Ожидается: finished

SELECT xp, soft_currency FROM player_progress WHERE user_id=1;
-- Ожидается: xp выросло на 100, soft_currency на 50

SELECT score FROM leaderboard_scores WHERE user_id=1 AND board_code='default';
-- Ожидается: 7200 или больше (зависит от score в EndMatchWin)
```

---


##  12. Описание каждого скрипта

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

##  13. Расширение проекта

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

# Ссылка на пример проекта
[!Ссылка на пример проекта](https://github.com/OlgaKraven/strategy-support-is.git)