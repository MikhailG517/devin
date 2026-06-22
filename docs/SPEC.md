# Спецификация — Devin Balance Guard

Версия документа: 1.1.1 (полная спецификация для воспроизведения проекта).
Соответствует версии расширения `1.1.1` (`extension/manifest.json`).

> Для продолжения работы в другом аккаунте Devin/GitHub см.
> [`docs/HANDOFF.md`](HANDOFF.md) — что уже сделано, какие ключи и секреты
> нужно перенести, как пересобрать и опубликовать обновление.

---

## 1. Назначение

**Devin Balance Guard** — расширение для Chromium-браузеров (Manifest V3),
которое:

1. **Отображает остаток баланса Devin** — на бейдже иконки в тулбаре и в попапе.
2. **Автоматически останавливает активные (recent) сессии**, когда баланс
   опускается ниже (или поднимается выше) настраиваемого порога.
3. Даёт **ручную остановку** всех активных сессий по кнопке.

Расширение написано на чистом JS (ES-модули), без стороннего фреймворка,
бандлера или шага сборки. Все файлы в папке `extension/` — готовый к загрузке
код.

---

## 2. Термины и определения

| Термин | Значение |
| --- | --- |
| **Баланс On-demand** | Значение «On-demand usage → Remaining balance» со страницы Devin **Настройки → Usage & limits**, поле `overage_credits` из внутреннего API. |
| **Активная (running) сессия** | Сессия со `status_enum` из множества `{working, blocked, resume_requested, resume_requested_frontend, resumed}`. |
| **Recent** | Сортировка активных сессий по `updated_at` по убыванию (самые недавно обновлённые — первыми). |
| **Порог (threshold)** | Числовое значение, при пересечении которого срабатывает авто-остановка. |
| **Направление (thresholdMode)** | `below` — стоп, когда баланс `<` порога; `above` — стоп, когда значение `>` порога. |
| **Auth bundle** | Набор `{ token, orgId, userId }` — берётся из `localStorage` на `app.devin.ai` content-скриптом `devin-bridge.js` и передаётся фоновому worker'у. |
| **Автоматический API-ключ** | Личный ключ `apk_user_…`, получаемый через внутренний API `app.devin.ai` по auth bundle (без ручного ввода). |
| **Service worker** | Фоновый скрипт `background.js`, работающий как MV3 service worker. |

---

## 3. Функциональные требования

### FR-1. Отображение баланса

- Бейдж иконки показывает округлённое значение баланса (функция `formatBadge`
  в `lib/core.js`):
  - `value >= 100000` → `Math.round(value / 1000)` + `k` (например `120k`)
  - `value >= 1000` → `(value / 1000).toFixed(1)` + `k` (например `1.5k`)
  - `value >= 10` → `Math.round(value)` (например `42`)
  - `value < 10` → `value.toFixed(1)` (например `3.7`)
  - `value == null` → `?`
- Цвет фона бейджа:
  - **Синий** `#1a73e8` — норма.
  - **Красный** `#d93025` — баланс пересёк порог (`shouldTrigger === true`).
- Попап показывает точное значение:
  - Для On-demand — с символом `$` (например `$24.50`).
  - Для team — без символа (кредиты).
  - Для manual — без символа.
- Подпись под значением:
  - `ondemand` → `"Остаток On-demand"`.
  - `team` → `"доп.: X использовано / Y доступно"`.
  - `manual` → `"ручной бюджет"`.
- Если значение ещё не считано или пользователь не вошёл — показывается `—` и
  текст-подсказка.

### FR-1a. Источник API-ключа (`apiKeyMode`)

- **`auto`** (по умолчанию):
  1. Content-скрипт `content/devin-bridge.js` берёт токен сессии
     (`auth1_session`) и id организации из `localStorage` на `app.devin.ai`.
  2. Передаёт фоновому worker'у сообщение `{ type: "devinAuth", token, orgId, userId }`.
  3. Worker запрашивает личный ключ пользователя:
     `GET https://app.devin.ai/api/{orgId}/{userId}/api-key`
     → поле `api_key` (формат `apk_user_…`).
  4. Если ключа нет (404 / пустой ответ) — создаёт через
     `POST https://app.devin.ai/api/{orgId}/{userId}/api-key/regenerate`.
  5. Полученный ключ используется для всех вызовов публичного API v1.
- **`manual`**: используется ключ `apiKey`, введённый пользователем в настройках.

### FR-2. Источники баланса (`balanceSource`)

#### `ondemand` (по умолчанию)
- Запрос: `GET https://app.devin.ai/api/{orgId}/billing/status`
  - Заголовки: `Authorization: Bearer <token>`, `x-cog-org-id: <orgId>`.
  - Ответ: JSON с полем `overage_credits` (число, USD).
- Авторизация через auth bundle (логин пользователя), отдельный ключ не нужен.
- При 401/403 → считается, что пользователь вышел: баланс и авто-ключ
  очищаются, в попапе — приглашение войти.
- Сетевые ошибки (обрыв, таймаут) **не считаются выходом** — сохраняется
  последнее успешное значение.

#### `team`
- Запрос: `POST https://server.codeium.com/api/v1/GetTeamCreditBalance`
  - Тело: `{ "service_key": "<serviceKey>" }`.
  - Ответ: JSON с полями `addOnCreditsAvailable`, `addOnCreditsUsed`,
    `promptCreditsPerSeat`, `numSeats`.
- Контролируемое значение: `remaining = max(0, addOnCreditsAvailable - addOnCreditsUsed)`.
- Требует enterprise-ключ с правом **Billing Read**.

#### `manual`
- Фиксированное число `manualBudget`, заданное пользователем.
- Не опрашивается — просто берётся из конфига.

### FR-3. Опрос и обновление

- Фоновый service worker создаёт будильник `chrome.alarms`:
  - Имя: `"devin-balance-poll"`.
  - `periodInMinutes`: значение из конфига (`pollIntervalMinutes`), минимум `0.5`.
  - `delayInMinutes: 0` (сразу после создания).
- Будильник перенастраивается при:
  - `onInstalled` (установка/обновление расширения).
  - `onStartup` (запуск браузера).
  - Изменение `pollIntervalMinutes` в `chrome.storage.sync`.
- На каждом тике будильника вызывается `pollCycle()` → `core.refresh()`:

**Алгоритм `refresh()` (lib/core.js):**

```
1. Прочитать config (sync) и cache (local).
2. Определить, использует ли конфиг внутреннюю сессию (ondemand или auto).
3. Если apiKeyMode === "auto":
   a. Есть auth bundle → getPersonalApiKey(auth) → autoApiKey.
   b. AuthError → loggedOut = true.
   c. Другая ошибка → запомнить autoApiKeyError, сохранить кэшированный ключ.
   d. Нет auth bundle → сохранить кэшированный autoApiKey (личный ключ
      долгоживущий; очищается только по явному logout от content-скрипта).
4. apiKey = (auto ? autoApiKey : config.apiKey).
5. Баланс:
   a. manual → value = manualBudget.
   b. ondemand → getOnDemandBalance(auth).
      - AuthError → loggedOut.
      - Сетевая ошибка → оставить предыдущее значение.
   c. team → getTeamCreditBalance(serviceKey).
6. Если loggedOut и ondemand → обнулить баланс, показать LOGIN_PROMPT.
7. Сессии:
   a. Есть apiKey → listSessions(apiKey).
   b. Нет apiKey + auto → показать LOGIN_PROMPT или autoApiKeyError.
   c. Нет apiKey + manual → "Не указан API-ключ".
8. Обновить бейдж (значение + цвет).
9. Авто-остановка:
   a. shouldTrigger(config, balance) && autoStopEnabled && нет ошибок сессий && есть apiKey.
   b. Отфильтровать running сессии, исключить protectedSessionIds, отсортировать recent.
   c. Если maxSessionsToStop > 0 → взять первые N.
   d. stopMode=="pause" → POST /api/sessions/bulk-archive {session_ids,[…], archive:true};
      stopMode=="terminate" → последовательно DELETE /v1/sessions/{id}.
   e. Показать уведомление: "Баланс X ниже Y. Приостановлено/Остановлено Z из W."
   f. Перечитать список сессий.
10. Записать кэш.
```

### FR-4. Авто-остановка

- Условие срабатывания: `shouldTrigger(config, value)`:
  - `thresholdMode === "below"` → `value < threshold`.
  - `thresholdMode === "above"` → `value > threshold`.
- Дополнительные условия: `autoStopEnabled === true`, список сессий получен
  без ошибки, apiKey есть.
- Останавливаются активные (`isRunning`) сессии в порядке recent,
  исключая `protectedSessionIds` (`isProtected`).
- `maxSessionsToStop > 0` → только первые N; `maxSessionsToStop === 0` → **все**.
- Действие зависит от `stopMode`:
  - `"pause"` (по умолчанию) — **пауза/сон**: `POST https://app.devin.ai/api/sessions/bulk-archive`
    с телом `{ "session_ids": [...], "archive": true }` (тот же эндпоинт, что и кнопка
    «Archive» в веб-интерфейсе). **Обратимо** — сессию можно возобновить (`archive:false`).
    Требует входа в `app.devin.ai` (auth-бандл: `Authorization: Bearer`, `x-cog-org-id`).
  - `"terminate"` — **полная остановка**: `DELETE https://api.devin.ai/v1/sessions/{session_id}`.
    **Необратима** — завершённую сессию нельзя возобновить. Работает с одним API-ключом.
- После остановки:
  - Системное уведомление `chrome.notifications.create`.
  - Список сессий перечитывается через `listSessions`.

### FR-5. Ручное управление

- Кнопка **«Остановить все»** в попапе → отправляет `{ type: "stopAll" }`:
  - При `confirmBeforeStop` попап спрашивает подтверждение с числом
    незащищённых активных сессий перед отправкой.
  - Worker вызывает `stopRecentSessions(config, lastSessions, auth)`.
  - Текст кнопки/подтверждения/уведомления зависит от `stopMode` («Приостановить» vs «Остановить»).
  - Учитывается `maxSessionsToStop`; `protectedSessionIds` никогда не останавливаются.
  - После остановки — `refresh()`, результат отправляется обратно.
- Кнопка **«Обновить»** → отправляет `{ type: "refresh" }` → немедленный
  `pollCycle()`.
- Поле порога в попапе (`threshold-input`) → при `change` сохраняет в конфиг
  и делает `refresh`.
- Тумблер авто-остановки (`autostop-toggle`) → при `change` сохраняет и
  перечитывает состояние.

### FR-6. Список сессий в попапе

- Показывает активные (`isRunning`) сессии, отсортированные recent.
- Каждая сессия:
  - Ссылка на `https://app.devin.ai/sessions/{id}` (с удалением префикса `devin-`).
  - Текст: `session.title || session.session_id`.
  - Бейдж статуса: `session.status_enum || session.status`.
- Если `loggedIn === false` → статус «Войдите в Devin: откройте app.devin.ai».
- Если `lastSessionsError` → статус с текстом ошибки.
- Если сессий нет → «Нет активных сессий.»

---

## 4. Протокол сообщений (runtime messages)

Все сообщения передаются через `chrome.runtime.sendMessage` /
`chrome.runtime.onMessage`.

### Входящие (от popup/options/content → background)

| `type` | Отправитель | Payload | Ответ |
| --- | --- | --- | --- |
| `"refresh"` | popup | — | `{ ok, result: cache }` |
| `"getState"` | popup | — | `{ ok, config, cache }` |
| `"stopAll"` | popup | — | `{ ok, report, result: cache }` |
| `"devinAuth"` | devin-bridge | `token, orgId, userId` или `loggedOut: true` | `{ ok }` |

### Обработка `devinAuth`

- Если `loggedOut || !token || !orgId` → очистить auth bundle и autoApiKey.
- Иначе → записать `authToken, authOrgId, authUserId, authAt` в кэш.
- После записи → `refresh()`.

---

## 5. Конфигурация

### Пользовательские настройки (`chrome.storage.sync`)

Определены в `lib/storage.js` → `DEFAULTS`:

| Ключ | Тип | По умолчанию | Описание |
| --- | --- | --- | --- |
| `apiKeyMode` | `"auto" \| "manual"` | `"auto"` | Источник API-ключа. |
| `apiKey` | `string` | `""` | Личный API-ключ Devin (`apk_user_…`) для режима `manual`. |
| `serviceKey` | `string` | `""` | Сервисный ключ биллинга (для `team`). |
| `balanceSource` | `"ondemand" \| "team" \| "manual"` | `"ondemand"` | Источник баланса. |
| `manualBudget` | `number` | `0` | Фиксированный бюджет (для `manual`). |
| `thresholdMode` | `"below" \| "above"` | `"below"` | Направление срабатывания. |
| `threshold` | `number` | `5` | Порог остановки (USD для On-demand). |
| `autoStopEnabled` | `boolean` | `true` | Включена ли авто-остановка. |
| `pollIntervalMinutes` | `number` | `1` | Интервал опроса (минимум 0.5). |
| `maxSessionsToStop` | `number` | `0` | Лимит остановки (`0` = все). |
| `protectedSessionIds` | `string[]` | `[]` | ID сессий, которые никогда не останавливаются (сравнение без префикса `devin-`). |
| `confirmBeforeStop` | `boolean` | `true` | Спрашивать подтверждение перед ручной кнопкой «Остановить все». |
| `stopMode` | `"pause" \| "terminate"` | `"pause"` | Пауза (archive/сон, обратимо) или полная остановка (DELETE, необратимо). |

### Рантайм-кэш (`chrome.storage.local`)

Определён в `lib/storage.js` → `CACHE_DEFAULTS`:

| Ключ | Тип | По умолчанию | Описание |
| --- | --- | --- | --- |
| `lastBalance` | `number \| null` | `null` | Последнее значение баланса. |
| `lastBalanceRaw` | `object \| null` | `null` | Сырые данные от API баланса. |
| `lastBalanceError` | `string` | `""` | Текст ошибки баланса. |
| `lastSessions` | `array` | `[]` | Последний список сессий. |
| `lastSessionsError` | `string` | `""` | Текст ошибки сессий. |
| `lastChecked` | `number` | `0` | Timestamp последней проверки. |
| `lastStopReport` | `object \| null` | `null` | Отчёт об авто-остановке. |
| `authToken` | `string` | `""` | Токен сессии из `app.devin.ai`. |
| `authOrgId` | `string` | `""` | ID организации. |
| `authUserId` | `string` | `""` | ID пользователя. |
| `authAt` | `number` | `0` | Timestamp получения auth bundle. |
| `autoApiKey` | `string` | `""` | Авто-полученный личный ключ. |
| `autoApiKeyError` | `string` | `""` | Ошибка авто-получения ключа. |
| `loggedIn` | `boolean \| null` | `null` | `null` = неизвестно, `true` = вошёл, `false` = вышел. |

---

## 6. Внешние интерфейсы (API)

### Публичный API Devin (v1)

| Действие | Метод | URL | Авторизация | Тело | Ответ |
| --- | --- | --- | --- | --- | --- |
| Список сессий | `GET` | `https://api.devin.ai/v1/sessions?limit=100` | `Authorization: Bearer <apiKey>` | — | `{ sessions: [...] }` |
| Остановка сессии | `DELETE` | `https://api.devin.ai/v1/sessions/{session_id}` | `Authorization: Bearer <apiKey>` | — | — |

### Внутренний API app.devin.ai

| Действие | Метод | URL | Заголовки | Ответ |
| --- | --- | --- | --- | --- |
| Баланс On-demand | `GET` | `https://app.devin.ai/api/{orgId}/billing/status` | `Authorization: Bearer <token>`, `x-cog-org-id: <orgId>` | `{ overage_credits: number, ... }` |
| Личный API-ключ | `GET` | `https://app.devin.ai/api/{orgId}/{userId}/api-key` | `Authorization: Bearer <token>`, `x-cog-org-id: <orgId>` | `{ api_key: "apk_user_..." }` |
| Создание ключа | `POST` | `https://app.devin.ai/api/{orgId}/{userId}/api-key/regenerate` | `Authorization: Bearer <token>`, `x-cog-org-id: <orgId>` | `{ api_key: "apk_user_..." }` |
| Пауза/возобновление сессий | `POST` | `https://app.devin.ai/api/sessions/bulk-archive` | `Authorization: Bearer <token>`, `x-cog-org-id: <orgId>`, тело `{ "session_ids": [...], "archive": true\|false }` | `{ "status": "success" }` |

### Enterprise-биллинг

| Действие | Метод | URL | Тело | Ответ |
| --- | --- | --- | --- | --- |
| Командный баланс | `POST` | `https://server.codeium.com/api/v1/GetTeamCreditBalance` | `{ "service_key": "<key>" }` | `{ addOnCreditsAvailable, addOnCreditsUsed, promptCreditsPerSeat, numSeats }` |

### Манифест (permissions)

```json
{
  "permissions": ["storage", "alarms", "notifications"],
  "host_permissions": [
    "https://api.devin.ai/*",
    "https://server.codeium.com/*",
    "https://app.devin.ai/*"
  ]
}
```

---

## 7. Мост сессии (`content/devin-bridge.js`)

Запускается на `https://app.devin.ai/*` (`"run_at": "document_idle"`).

### Алгоритм

1. **Чтение сессии**: `JSON.parse(localStorage.getItem("auth1_session"))` →
   `{ token, userId }`. Если `null` или нет `token` → `{ loggedOut: true }`.

2. **Чтение orgId** (функция `readOrgId`):
   - Приоритет: ключ `"last-internal-org-for-external-org-v1-null"` → regex
     `org-[0-9a-f]{32}`.
   - Fallback: любой ключ/значение в `localStorage`, содержащий `org-…`.

3. **Чтение userId** (функция `readUserId`):
   - Приоритет: `session.userId`.
   - Fallback: любой ключ в `localStorage`, содержащий `user-[0-9a-f]{32}`.

4. **Отправка**: `chrome.runtime.sendMessage({ type: "devinAuth", token, orgId, userId })`.
   - Если сессии нет → `{ type: "devinAuth", loggedOut: true }`.
   - Ошибка `lastError` проглатывается (worker может спать).

5. **Повторная отправка**:
   - `setInterval(send, 60000)` — каждые 60 секунд.
   - `window.addEventListener("focus", send)`.
   - `document.addEventListener("visibilitychange", ...)` → при `visible`.

### Зачем повтор

Токен может ротироваться; пользователь может выйти в SPA без перезагрузки
страницы. Повтор гарантирует, что worker получает актуальный auth bundle.

---

## 8. Обработка ошибок

| Ситуация | Поведение |
| --- | --- |
| Внутренний API 401/403 | `AuthError` → `loggedOut = true`, очистка баланса; авто-ключ `apk_user_…` **сохраняется** (долгоживущий, валиден для v1), очищается только по явному logout от content-скрипта. |
| Сетевая ошибка (обрыв, таймаут) | Бросается обычный `Error`, **не** `AuthError`. Последнее успешное значение сохраняется. |
| Публичный API 401/403 при списке сессий | Текст ошибки: «Неверный API-ключ (нужен личный ключ apk_… с доступом к API v1).» |
| Нет auth bundle + auto mode | «Войдите в Devin: откройте app.devin.ai» |
| Нет apiKey + manual mode | «Не указан API-ключ» |
| Ошибка остановки одной сессии | Записывается в `results[i].error`, остальные сессии продолжают останавливаться. |

---

## 9. Нефункциональные требования

- Без шага сборки и сторонних зависимостей (ванильный JS, ES-модули).
- Учётные данные хранятся только в `chrome.storage` и отправляются только на
  хосты Devin (`api.devin.ai`, `app.devin.ai`, `server.codeium.com`).
- Логин пользователя и токены третьим лицам **не передаются**.
- Совместимость: Chromium-браузеры с поддержкой MV3 (Chrome 110+, Edge,
  Yandex Browser, Brave, Opera).
- Скрипт `devin-bridge.js` — обычный IIFE (не ES-модуль), т.к. content
  scripts MV3 не поддерживают `type: "module"`.

---

## 10. Известные ограничения и риски

1. Действие над сессиями зависит от `stopMode`. По умолчанию `"pause"` —
   **обратимо** (archive/сон, сессию можно возобновить). Режим `"terminate"`
   (`DELETE /v1/sessions/{id}`) — **необратим**. В обоих режимах действуют
   защиты: allow-list `protectedSessionIds` (никогда не трогаются) и
   подтверждение `confirmBeforeStop` перед ручной кнопкой (по умолчанию
   включено); текст подтверждения отражает обратимость режима.
2. Внутренний API `app.devin.ai` — неофициальный. При сильном редизайне
   эндпоинтов или структуры `localStorage` мост может потребовать правки.
3. Источник `team` требует enterprise-ключа Billing Read.
4. Минимальный интервал опроса — 0.5 минут (ограничение `chrome.alarms`).
5. Content-скрипт не работает на других доменах Devin (если появятся).

---

## 11. Доработки

Реализовано:
- **Allow-list защищённых сессий** (`protectedSessionIds`) — перечисленные ID
  никогда не останавливаются; в попапе помечены «защищена».
- **Подтверждение перед массовой остановкой** (`confirmBeforeStop`) для ручной
  кнопки «Остановить все».
- **Инъекция моста при установке** — фон через `chrome.scripting` инжектит
  `devin-bridge.js` в уже открытые вкладки `app.devin.ai` (не нужно вручную
  перезагружать вкладку после установки).
- **Дистрибуция и автообновление** — подписанный `.crx` + `updates.xml` на
  GitHub Pages (`MikhailG517/devin`), стабильный ID через `key` в манифесте и
  `update_url`; установка без режима разработчика через
  `ExtensionInstallForcelist`. Сборка/публикация в GitHub Actions.

Backlog:
- Публикация в Chrome Web Store / Microsoft Edge Add-ons.
- Локализация через `_locales/` (сейчас строки захардкожены на русском).
- Тёмная тема (через `prefers-color-scheme` или ручной переключатель).
- Графики расхода баланса за период.
