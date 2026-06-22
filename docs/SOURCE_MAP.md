# Карта исходного кода — Devin Balance Guard

Подробное описание каждого файла проекта: назначение, экспорты, ключевые
функции и константы. Используйте этот документ для понимания кодовой базы
при воспроизведении проекта на другом аккаунте.

---

## Оглавление

1. [manifest.json](#manifestjson)
2. [background.js](#backgroundjs)
3. [lib/storage.js](#libstoragejs)
4. [lib/api.js](#libapijs)
5. [lib/internalApi.js](#libinternalapijs)
6. [lib/core.js](#libcorejs)
7. [popup.html](#popuphtml)
8. [popup.js](#popupjs)
9. [options.html](#optionshtml)
10. [options.js](#optionsjs)
11. [styles.css](#stylescss)
12. [content/devin-bridge.js](#contentdevin-bridgejs)
13. [scripts/build-release.sh](#scriptsbuild-releasesh)
14. [scripts/pack-crx.js](#scriptspack-crxjs)
15. [scripts/make-updates-xml.js](#scriptsmake-updates-xmljs)
16. [.github/workflows](#githubworkflows)

---

## manifest.json

**Расположение:** `extension/manifest.json`

Манифест Chrome Extension (Manifest V3). Определяет метаданные, разрешения
и точки входа расширения.

### Ключевые поля

| Поле | Значение | Назначение |
| --- | --- | --- |
| `manifest_version` | `3` | Manifest V3 |
| `name` | `"Devin Balance Guard"` | Название расширения |
| `version` | `"1.1.1"` | Текущая версия |
| `key` | base64 публичного ключа | Фиксирует стабильный ID `gdobkpjnopnafcbnhmfanccelegodchc` |
| `update_url` | `https://mikhailg517.github.io/devin/updates.xml` | Источник автообновления |
| `permissions` | `["storage", "alarms", "notifications", "scripting"]` | API Chrome (`scripting` — инъекция моста в открытые вкладки) |
| `host_permissions` | `api.devin.ai`, `server.codeium.com`, `app.devin.ai` | Доступ к хостам |
| `background.service_worker` | `"background.js"` | Фоновый скрипт |
| `background.type` | `"module"` | ES-модули в service worker |
| `content_scripts[0].matches` | `["https://app.devin.ai/*"]` | Где работает мост |
| `content_scripts[0].js` | `["content/devin-bridge.js"]` | Файл моста |
| `content_scripts[0].run_at` | `"document_idle"` | Когда запускать |
| `action.default_popup` | `"popup.html"` | HTML попапа |
| `options_page` | `"options.html"` | Страница настроек |

---

## background.js

**Расположение:** `extension/background.js`  
**Тип:** ES-модуль (service worker)  
**Импорты:** `core.js` → `refresh`, `stopRecentSessions`; `storage.js` → `getConfig`, `getCache`, `setCache`

### Константы

- `ALARM_NAME = "devin-balance-poll"` — имя будильника Chrome.

### Функции

#### `pollCycle()`
Вызывает `refresh()` из `core.js`. Обёртка для единообразного вызова.

#### `setupAlarm()`
Читает `pollIntervalMinutes` из конфига, приводит к минимуму 0.5, создаёт
(или пересоздаёт) будильник Chrome с данным интервалом.

### Обработчики событий

| Событие | Действие |
| --- | --- |
| `chrome.runtime.onInstalled` | `setupAlarm()` + `pollCycle()` |
| `chrome.runtime.onStartup` | `setupAlarm()` + `pollCycle()` |
| `chrome.alarms.onAlarm` | Если `alarm.name === ALARM_NAME` → `pollCycle()` |
| `chrome.storage.onChanged` | Если `area === "sync"` и изменился `pollIntervalMinutes` → `setupAlarm()` |

### Обработчик сообщений (`chrome.runtime.onMessage`)

| `msg.type` | Действие | Ответ |
| --- | --- | --- |
| `"refresh"` | `pollCycle()` → прочитать кэш | `{ ok: true, result: cache }` |
| `"devinAuth"` | Записать auth bundle или очистить при logout → `refresh()` | `{ ok: true }` |
| `"getState"` | Прочитать config + cache | `{ ok: true, config, cache }` |
| `"stopAll"` | Определить apiKey + auth-бандл → `stopRecentSessions(config, sessions, auth)` → `refresh()` | `{ ok: true, report, result }` |

Возвращает `true` из listener для поддержки асинхронного `sendResponse`.

---

## lib/storage.js

**Расположение:** `extension/lib/storage.js`  
**Тип:** ES-модуль

### Экспорты

#### `DEFAULTS` (объект)
Дефолтные пользовательские настройки, хранящиеся в `chrome.storage.sync`:

```
apiKeyMode: "auto"       // "auto" | "manual"
apiKey: ""               // личный ключ (для manual)
serviceKey: ""           // сервисный ключ (для team)
balanceSource: "ondemand" // "ondemand" | "team" | "manual"
manualBudget: 0          // число (для manual)
thresholdMode: "below"   // "below" | "above"
threshold: 5             // порог (USD)
autoStopEnabled: true    // вкл/выкл авто-остановки
pollIntervalMinutes: 1   // интервал опроса (мин, мин. 0.5)
maxSessionsToStop: 0     // лимит остановки (0 = все)
protectedSessionIds: []  // ID, которые никогда не останавливаются
confirmBeforeStop: true  // спрашивать подтверждение перед «Остановить все»
stopMode: "pause"        // "pause" (archive/сон, обратимо) | "terminate" (DELETE, необратимо)
```

#### `CACHE_DEFAULTS` (внутренний объект)
Дефолтный рантайм-кэш в `chrome.storage.local`:

```
lastBalance: null        // последнее значение баланса
lastBalanceRaw: null     // сырые данные от API
lastBalanceError: ""     // текст ошибки
lastSessions: []         // последний список сессий
lastSessionsError: ""    // текст ошибки
lastChecked: 0           // timestamp
lastStopReport: null     // отчёт авто-остановки
authToken: ""            // токен сессии из app.devin.ai
authOrgId: ""            // ID организации
authUserId: ""           // ID пользователя
authAt: 0                // timestamp получения auth bundle
autoApiKey: ""           // авто-полученный ключ
autoApiKeyError: ""      // ошибка авто-получения
loggedIn: null           // null | true | false
```

#### `getAuthBundle()` → `Promise<{ token, orgId, userId }>`
Извлекает auth bundle из кэша.

#### `getConfig()` → `Promise<object>`
Читает `chrome.storage.sync`, мержит с `DEFAULTS`.

#### `setConfig(patch)` → `Promise<void>`
Записывает `patch` в `chrome.storage.sync`.

#### `getCache()` → `Promise<object>`
Читает `chrome.storage.local`, мержит с `CACHE_DEFAULTS`.

#### `setCache(patch)` → `Promise<void>`
Записывает `patch` в `chrome.storage.local`.

---

## lib/api.js

**Расположение:** `extension/lib/api.js`  
**Тип:** ES-модуль

Обёртка над **публичным** API Devin v1 и enterprise-биллингом.

### Константы

- `DEVIN_API_BASE = "https://api.devin.ai/v1"`
- `BALANCE_URL = "https://server.codeium.com/api/v1/GetTeamCreditBalance"`
- `RUNNING_STATUSES = ["working", "blocked", "resume_requested", "resume_requested_frontend", "resumed"]`

### Классы

#### `ApiError extends Error`
- Поля: `message`, `status` (HTTP-код или 0 при сетевой ошибке).
- Используется для всех ошибок публичного API.

### Функции

#### `request(url, { method, apiKey, body })` (внутренняя)
Базовый HTTP-клиент. Добавляет `Authorization: Bearer`, парсит JSON,
бросает `ApiError` при `!res.ok`.

#### `listSessions(apiKey, { limit })` → `Promise<session[]>`
- `GET /v1/sessions?limit=100`
- Бросает `ApiError("Не указан API-ключ", 401)` если `apiKey` пуст.
- Возвращает `data.sessions || []`.

#### `terminateSession(apiKey, sessionId)` → `Promise<response>`
- `DELETE /v1/sessions/{sessionId}`
- Бросает `ApiError` если `apiKey` пуст.

#### `getTeamCreditBalance(serviceKey)` → `Promise<object>`
- `POST` на `BALANCE_URL` с телом `{ service_key }`.
- Вычисляет `remaining = max(0, addOnCreditsAvailable - addOnCreditsUsed)`.
- Возвращает расширенный объект с `remaining`, `promptCreditsPerSeat`,
  `numSeats`, `addOnCreditsAvailable`, `addOnCreditsUsed`.

#### `isRunning(session)` → `boolean`
Проверяет, является ли сессия активной: `status_enum` (или `status`)
входит в `RUNNING_STATUSES`.

---

## lib/internalApi.js

**Расположение:** `extension/lib/internalApi.js`  
**Тип:** ES-модуль

Обёртка над **внутренним** (неофициальным) API `app.devin.ai`. Позволяет
использовать вход пользователя в браузере вместо ручного ввода ключей.

### Константы

- `INTERNAL_API_BASE = "https://app.devin.ai/api"`

### Классы

#### `AuthError extends Error`
- Поля: `message`, `status`.
- Бросается при 401/403 — означает, что пользователь вышел или токен истёк.
- Отличается от обычного `Error` (сетевые ошибки) — `core.js` различает
  «выход» и «временную ошибку».

### Функции

#### `hasAuthBundle(auth)` → `boolean`
- `true` если есть `auth.token` и `auth.orgId`.

#### `internalRequest(path, auth, method)` (внутренняя)
Базовый HTTP-клиент для внутреннего API:
- Заголовки: `Authorization: Bearer <token>`, `x-cog-org-id: <orgId>`,
  `Accept: application/json`.
- `credentials: "include"` (для cookie).
- 401/403 → `AuthError`.
- Сетевая ошибка → обычный `Error` (не `AuthError`).

#### `getOnDemandBalance(auth)` → `Promise<number|null>`
- `GET /api/{orgId}/billing/status`
- Возвращает `data.overage_credits` (число) или `null`.

#### `getPersonalApiKey(auth)` → `Promise<string>`
- `GET /api/{orgId}/{userId}/api-key`
- Если ключа нет (404 / «not found» / пустой `api_key`):
  - Создаёт через `POST /api/{orgId}/{userId}/api-key/regenerate`.
- Возвращает строку `apk_user_…`.

#### `isKeyMissing(e)` (внутренняя)
- `true` если `e.status === 404` или `e.message` содержит «not found».

#### `createPersonalApiKey(auth)` (внутренняя)
- `POST /api/{orgId}/{userId}/api-key/regenerate`

#### `pauseSessions(auth, sessionIds, shouldArchive = true)` → `Promise<object>`
- `POST /api/sessions/bulk-archive` с телом `{ session_ids: [devin-…], archive: shouldArchive }`.
- Тот же эндпоинт, что и кнопка «Archive» в веб-интерфейсе — «пауза»/сон.
- `archive: false` — возобновление (разархивация). Требует auth-бандла.

---

## lib/core.js

**Расположение:** `extension/lib/core.js`  
**Тип:** ES-модуль

Главная бизнес-логика: опрос, проверка порога, остановка сессий.

### Константы

- `LOGIN_PROMPT = "Вы не вошли в Devin. Откройте app.devin.ai и войдите."`

### Функции

#### `usesInternalSession(config)` (внутренняя) → `boolean`
- `true` если `balanceSource === "ondemand"` или `apiKeyMode === "auto"`.

#### `friendlySessionError(e)` (внутренняя) → `string`
- 401/403 → «Неверный API-ключ (нужен личный ключ apk_…)»
- Иначе → `e.message`.

#### `shouldTrigger(config, value)` → `boolean`
- `value == null` → `false`.
- `thresholdMode === "above"` → `value > threshold`.
- Иначе → `value < threshold`.

#### `formatBadge(value)` (внутренняя) → `string`
- Форматирует число для бейджа иконки (см. SPEC.md, FR-1).

#### `updateBadge(value, belowThreshold)` (внутренняя)
- `chrome.action.setBadgeText` + `setBadgeBackgroundColor`.
- Синий `#1a73e8` в норме, красный `#d93025` при пересечении порога.

#### `notify(title, message)` (внутренняя)
- `chrome.notifications.create({ type: "basic", ... })`.

#### `stopRecentSessions(config, sessions, auth)` → `Promise<report>`
- Фильтрует `isRunning`, исключает `protectedSessionIds`, сортирует по `updated_at` desc.
- Если `maxSessionsToStop > 0` → берёт первые N.
- Ветвление по `config.stopMode`:
  - `"pause"` (по умолчанию) → один вызов `pauseSessions(auth, ids)` (bulk-archive, обратимо).
    Без auth-бандла возвращает ошибку LOGIN_PROMPT по каждой цели.
  - `"terminate"` → последовательно `terminateSession` (DELETE v1, необратимо).
- Возвращает `{ attempted, mode, results: [{ session_id, title, ok, error? }], at }`.

#### `refresh()` → `Promise<cache>`
Главный цикл опроса (см. алгоритм в SPEC.md, FR-3):

1. Читает config + cache.
2. Определяет `loggedOut` (нет auth bundle + internal mode).
3. Auto-ключ: `getPersonalApiKey` (при ошибке — `AuthError` = logout,
   другая ошибка = запомнить, сохранить кэшированный).
4. Баланс: `manual` / `ondemand` / `team`.
5. Сессии: `listSessions` если есть ключ.
6. `updateBadge`.
7. Авто-остановка: если `shouldTrigger && autoStopEnabled && apiKey && нет ошибок`.
8. Записывает кэш, возвращает новое состояние.

---

## popup.html

**Расположение:** `extension/popup.html`

HTML-разметка попапа, который открывается при клике на иконку расширения.

### Структура

```
<header>
  h1 "Devin Balance Guard"
  button#settings-btn (⚙ — открыть настройки)

<section.balance-card>
  .balance-label   "Остаток баланса"
  .balance-value   #balance-value (число или "—")
  .balance-sub     #balance-sub (подпись источника)

<section.threshold-row>
  label + input#threshold-input + span#threshold-unit

<section.toggle-row>
  switch input#autostop-toggle + label "Авто-остановка"

<section.actions>
  button#refresh-btn  "Обновить"
  button#stop-all-btn "Остановить все"

<section.status> #status

<section.sessions>
  span "Активные сессии" + span#sessions-count
  ul#sessions-list

<footer>
  span#last-checked
```

---

## popup.js

**Расположение:** `extension/popup.js`  
**Тип:** ES-модуль  
**Импорты:** `api.js` → `isRunning`; `storage.js` → `setConfig`

### Функции

#### `sendMessage(msg)` → `Promise<response>`
Обёртка над `chrome.runtime.sendMessage` с промисом.

#### `fmtNumber(n)` → `string`
Форматирует число с `toLocaleString` (макс. 2 знака после запятой).

#### `timeAgo(ts)` → `string`
Человекочитаемый интервал: «X с назад», «X мин назад», «X ч назад».

#### `setStatus(text, kind)`
Устанавливает текст и CSS-класс элемента `#status`.

#### `render(state)`
Рендерит весь попап по данным `{ config, cache }`:
- Баланс (значение, подпись, стиль карточки).
- Порог и тумблер.
- Список сессий (фильтр `isRunning`, сортировка recent, ссылки).
- Время последней проверки.
- Статус авто-остановки (если была в последние 60 секунд).

#### `load()`
Отправляет `getState` → `render`.

#### `refresh()`
Отправляет `refresh` → `load`.

### Обработчики событий

| Элемент | Событие | Действие |
| --- | --- | --- |
| `#settings-btn` | `click` | `chrome.runtime.openOptionsPage()` |
| `#refresh-btn` | `click` | `refresh()` |
| `#stop-all-btn` | `click` | Отправить `stopAll`, показать результат |
| `#threshold-input` | `change` | `setConfig({ threshold })` + `refresh()` |
| `#autostop-toggle` | `change` | `setConfig({ autoStopEnabled })` + `load()` |

---

## options.html

**Расположение:** `extension/options.html`

HTML-разметка страницы настроек расширения.

### Элементы управления

| ID | Тип | Конфиг-поле |
| --- | --- | --- |
| `api-key-mode` | `<select>` | `apiKeyMode` |
| `api-key` | `<input type="password">` | `apiKey` |
| `balance-source` | `<select>` | `balanceSource` |
| `service-key` | `<input type="password">` | `serviceKey` |
| `manual-budget` | `<input type="number">` | `manualBudget` |
| `threshold` | `<input type="number">` | `threshold` |
| `threshold-mode` | `<select>` | `thresholdMode` |
| `auto-stop` | `<input type="checkbox">` | `autoStopEnabled` |
| `max-sessions` | `<input type="number">` | `maxSessionsToStop` |
| `poll-interval` | `<input type="number">` | `pollIntervalMinutes` |
| `stop-mode` | `<select>` | `stopMode` (`pause` / `terminate`) |
| `protected-sessions` | `<textarea>` | `protectedSessionIds` (по ID на строку) |
| `confirm-before-stop` | `<input type="checkbox">` | `confirmBeforeStop` |
| `save-btn` | `<button>` | Сохранить |
| `test-btn` | `<button>` | Проверить подключение |
| `options-status` | `<p>` | Статус (результат операции) |

### Условная видимость полей

| Поле | Показывается когда |
| --- | --- |
| `#api-key-field` | `apiKeyMode === "manual"` |
| `#service-key-field` | `balanceSource === "team"` |
| `#manual-budget-field` | `balanceSource === "manual"` |
| `#ondemand-field` | `balanceSource === "ondemand"` |

---

## options.js

**Расположение:** `extension/options.js`  
**Тип:** ES-модуль  
**Импорты:** `storage.js` → `getConfig`, `setConfig`, `getCache`;
`api.js` → `listSessions`, `getTeamCreditBalance`

### Функции

#### `sendBg(msg)` → `Promise<response|null>`
Обёртка для отправки сообщений в background (с проглатыванием ошибок).

#### `setStatus(text, kind)`
Устанавливает текст и CSS-класс `#options-status`.

#### `syncSourceFields()`
Показывает/скрывает поля в зависимости от выбранного `balanceSource`.

#### `syncKeyFields()`
Показывает/скрывает поле API-ключа в зависимости от `apiKeyMode`.

#### `load()`
Читает конфиг через `getConfig()`, заполняет все поля формы.

#### `save()`
Собирает значения из полей формы, валидирует (min/max), записывает через
`setConfig()`, отправляет `refresh` в background.

#### `testConnection()`
Проверка подключения:
1. **API-ключ**:
   - `auto` → отправить `refresh` в background, прочитать кэш → проверить
     `autoApiKey`.
   - `manual` → взять из поля.
2. **Сессии**: `listSessions(apiKey)` → OK или ошибка.
3. **Баланс**:
   - `team` → `getTeamCreditBalance(serviceKey)`.
   - `ondemand` → прочитать из кэша.
4. Результат: все строки содержат «OK» → зелёный, иначе → красный.

---

## styles.css

**Расположение:** `extension/styles.css`

Общие стили для попапа и страницы настроек.

### CSS-переменные (`:root`)

```css
--bg: #ffffff        /* фон */
--fg: #1f2329        /* текст */
--muted: #5f6368     /* вторичный текст */
--border: #e0e0e0    /* границы */
--blue: #1a73e8      /* акцент (норма) */
--red: #d93025       /* опасность / ошибка */
--green: #1e8e3e     /* успех */
--card-bg: #f8f9fa   /* фон карточки */
```

### Основные секции

| Селектор | Назначение |
| --- | --- |
| `.popup` | Контейнер попапа (ширина 340px) |
| `.balance-card` | Карточка баланса |
| `.balance-card.danger` | Красная карточка (порог пересечён) |
| `.balance-card.error` | Жёлтая карточка (ошибка) |
| `.threshold-row` | Строка с полем порога |
| `.toggle-row` | Строка с тумблером |
| `.switch` / `.slider` | CSS-тумблер (checkbox → визуальный переключатель) |
| `.actions` | Кнопки «Обновить» / «Остановить все» |
| `.btn.primary` | Синяя кнопка |
| `.btn.danger` | Красная кнопка |
| `.sessions` | Секция списка сессий |
| `.options main` | Контейнер настроек (max-width 560px) |
| `.field` | Группа поля настроек |
| `.hint` | Подсказка под полем |

---

## content/devin-bridge.js

**Расположение:** `extension/content/devin-bridge.js`  
**Тип:** IIFE (не ES-модуль — MV3 не поддерживает модули в content scripts)  
**Внедряется на:** `https://app.devin.ai/*` (`run_at: "document_idle"`)

### Константы

- `ORG_RE = /org-[0-9a-f]{32}/i` — regex для org ID.
- `USER_RE = /user-[0-9a-f]{32}/i` — regex для user ID.

### Функции

#### `readUserId(session)` → `string|null`
1. `session.userId` (если есть).
2. Fallback: сканирует ключи `localStorage` на `USER_RE`.

#### `readOrgId()` → `string|null`
1. `localStorage.getItem("last-internal-org-for-external-org-v1-null")` → `ORG_RE`.
2. Fallback: сканирует все ключи/значения `localStorage` на `ORG_RE`.

#### `readBundle()` → `object`
1. `JSON.parse(localStorage.getItem("auth1_session"))`.
2. Если нет `session` или `session.token` → `{ loggedOut: true }`.
3. `readOrgId()` — если нет → `{ loggedOut: true }`.
4. Возвращает `{ token, userId, orgId }`.

#### `send()`
1. `readBundle()`.
2. `chrome.runtime.sendMessage({ type: "devinAuth", ...payload })`.
3. Проглатывает `lastError` (worker может спать).

### Автоматические вызовы

- `send()` — сразу при загрузке.
- `setInterval(send, 60000)` — каждые 60 секунд.
- `window.addEventListener("focus", send)`.
- `document.addEventListener("visibilitychange", ...)` → при `visible`.

---

## scripts/build-release.sh

**Расположение:** `scripts/build-release.sh`  
**Тип:** Bash-скрипт

### Алгоритм

1. Определяет корень проекта (`ROOT`), источник (`SRC = extension/`), выход
   (`OUT = release/`).
2. Читает версию из `manifest.json` через `node -e "..."`.
3. Удаляет старый билд (`release/devin-balance-guard/` и ZIP).
4. Копирует `extension/` → `release/devin-balance-guard/`.
5. Удаляет `icons/icon_src.png` (исходник иконки, не нужен в билде).
6. Создаёт ZIP-архив: `zip -qr -X release/devin-balance-guard-<version>.zip .`
7. Выводит пути созданных файлов.

8. Если задан ключ подписи (`CRX_KEY=<path.pem>` или `CRX_KEY_BASE64`):
   собирает подписанный `release/devin-balance-guard.crx`
   (`scripts/pack-crx.js`) и `release/updates.xml`
   (`scripts/make-updates-xml.js`).

### Требования

- `bash`
- `node` (для чтения JSON, упаковки CRX и генерации updates.xml)
- `zip` (для создания архива)

---

## scripts/pack-crx.js

**Расположение:** `scripts/pack-crx.js`  
**Тип:** Node-скрипт (без сторонних зависимостей, только `crypto`)

Упаковывает подписанный пакет **CRX3** из ZIP-архива расширения:
`node scripts/pack-crx.js <input.zip> <private-key.pem> <output.crx>`.

Формат CRX3: `"Cr24" | uint32le(3) | uint32le(headerLen) | CrxFileHeader | zip`.
ID расширения и публичный ключ в заголовке выводятся из приватного ключа,
поэтому подпись одним и тем же ключом всегда даёт **одинаковый ID** — это и
делает возможным автообновление.

---

## scripts/make-updates-xml.js

**Расположение:** `scripts/make-updates-xml.js`  
**Тип:** Node-скрипт

Генерирует Chrome update manifest:
`node scripts/make-updates-xml.js <private-key.pem> <version> <output.xml>`.

`codebase` и `appid` выводятся из `update_url` манифеста и ключа подписи, так
что всегда согласованы. Chrome периодически опрашивает этот XML и подтягивает
новую версию `.crx`.

---

## .github/workflows

- **`ci.yml`** — на каждый push/PR: валидация `manifest.json`, `node --check`
  всех JS, сборка ZIP, self-test упаковщика CRX (генерирует одноразовый ключ и
  проверяет магию `Cr24`).
- **`notify-release.yml`** — при изменении `extension/**` или `scripts/**` на
  `main` шлёт `repository_dispatch` (`new-release`) в `MikhailG517/devin`,
  запуская пересборку и публикацию `.crx`/`updates.xml`. Требует секрет
  `RELEASE_DISPATCH_TOKEN`.
