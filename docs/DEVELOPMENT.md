# Руководство разработчика — Devin Balance Guard

Полное руководство для воспроизведения, доработки и отладки расширения.

---

## 1. Технологический стек

| Компонент | Технология |
| --- | --- |
| Язык | JavaScript (ES2020+, ES-модули) |
| UI | HTML5 + CSS3 (ванильный, без фреймворков) |
| Платформа | Chrome Extensions Manifest V3 |
| Service worker | `background.js` (type: `"module"`) |
| Content script | `content/devin-bridge.js` (IIFE, не модуль) |
| Сборка | Нет (статические файлы, копирование) |
| Зависимости | Нет (только Chrome APIs) |
| Node.js | Только для `build-release.sh` (чтение версии из JSON) |

---

## 2. Требования к окружению

- Браузер на Chromium с поддержкой MV3 (Chrome 110+, Edge, Yandex, Brave, Opera).
- Node.js ≥ 14 — только для скрипта сборки `scripts/build-release.sh` (чтение
  `manifest.json` → версия) и проверки синтаксиса. Для запуска расширения
  Node.js **не нужен**.
- `zip` (CLI) — для создания архива в `build-release.sh`.

---

## 3. Структура проекта

```
devin_cost/
├── extension/                 # Исходный код расширения (загружается как unpacked)
│   ├── manifest.json          # Манифест MV3
│   ├── background.js          # Service worker (точка входа)
│   ├── popup.html             # HTML попапа тулбара
│   ├── popup.js               # Логика попапа
│   ├── options.html           # HTML страницы настроек
│   ├── options.js             # Логика настроек
│   ├── styles.css             # Общие стили (попап + настройки)
│   ├── lib/                   # Библиотечные модули
│   │   ├── api.js             # Обёртка над публичным API v1 (sessions, balance)
│   │   ├── internalApi.js     # Внутренний API app.devin.ai (баланс + личный ключ)
│   │   ├── storage.js         # DEFAULTS, getConfig/setConfig, getCache/setCache
│   │   └── core.js            # shouldTrigger, stopRecentSessions, refresh
│   ├── content/               # Content scripts
│   │   └── devin-bridge.js    # Мост: передаёт auth bundle из app.devin.ai
│   └── icons/                 # Иконки (16/32/48/128 png)
│       ├── icon16.png
│       ├── icon32.png
│       ├── icon48.png
│       └── icon128.png
├── docs/                      # Документация
│   ├── SPEC.md                # Спецификация
│   ├── DEVELOPMENT.md         # Это руководство
│   ├── INSTALL.md             # Инструкция по установке
│   └── SOURCE_MAP.md          # Карта исходного кода
├── release/                   # Готовый билд (генерируется build-release.sh)
│   ├── devin-balance-guard/   # Распакованное расширение
│   └── *.zip                  # Архив для магазинов
├── scripts/
│   └── build-release.sh       # Скрипт сборки релиза
├── README.md                  # Обзор проекта
└── .gitignore
```

---

## 4. Архитектура

### 4.1 Обзор компонентов

```
┌──────────────────────────────────────────────────────────────────────┐
│ Браузер (Chromium)                                                   │
│                                                                      │
│  ┌─────────────────────┐     ┌──────────────────────────────────┐   │
│  │ app.devin.ai (таб)  │     │ Service Worker (background.js)   │   │
│  │                     │     │                                  │   │
│  │  devin-bridge.js    │──── │  chrome.alarms → pollCycle()     │   │
│  │  (content script)   │msg  │  ↓                               │   │
│  │                     │     │  refresh() [core.js]             │   │
│  │  localStorage:      │     │  ├─ getPersonalApiKey() [auto]   │   │
│  │   auth1_session     │     │  ├─ getOnDemandBalance()         │   │
│  │   org-…             │     │  ├─ listSessions()               │   │
│  └─────────────────────┘     │  ├─ updateBadge()                │   │
│                              │  └─ stopRecentSessions() [auto]  │   │
│  ┌─────────────────────┐     │                                  │   │
│  │ Popup (popup.html)  │──── │  onMessage: refresh/getState/    │   │
│  │  popup.js           │msg  │             stopAll/devinAuth    │   │
│  └─────────────────────┘     └──────────────────────────────────┘   │
│                                        │                             │
│  ┌─────────────────────┐               │ fetch()                    │
│  │ Options             │               ↓                             │
│  │  (options.html)     │     ┌──────────────────────────────────┐   │
│  │  options.js         │     │ Внешние API                      │   │
│  └─────────────────────┘     │  api.devin.ai/v1/sessions       │   │
│                              │  app.devin.ai/api/…/billing      │   │
│                              │  server.codeium.com/api/…        │   │
│                              └──────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.2 Поток данных

```
1. Пользователь открывает app.devin.ai
   → devin-bridge.js читает localStorage
   → отправляет { type: "devinAuth", token, orgId, userId }

2. background.js принимает сообщение
   → записывает auth bundle в chrome.storage.local
   → вызывает refresh()

3. refresh() [core.js]:
   → getPersonalApiKey(auth) → autoApiKey [если apiKeyMode="auto"]
   → getOnDemandBalance(auth) → balance.value [если balanceSource="ondemand"]
   → listSessions(apiKey) → sessions[]
   → updateBadge(value, belowThreshold)
   → если shouldTrigger() && autoStopEnabled → stopRecentSessions()
   → setCache(newState)

4. Пользователь открывает попап
   → popup.js отправляет { type: "getState" }
   → background отвечает { config, cache }
   → popup.js рендерит UI
```

### 4.3 Хранение данных

```
chrome.storage.sync (синхронизируется между устройствами):
  └── Настройки пользователя (DEFAULTS из storage.js)

chrome.storage.local (локальный кэш):
  └── Рантайм-состояние (CACHE_DEFAULTS из storage.js)
      ├── lastBalance, lastSessions, lastChecked
      ├── authToken, authOrgId, authUserId (auth bundle)
      └── autoApiKey (авто-полученный ключ)
```

---

## 5. Модули — API каждого файла

### `lib/storage.js`

```js
DEFAULTS           // объект с дефолтными настройками
CACHE_DEFAULTS     // объект с дефолтным кэшем (внутренний)

getConfig()        // → Promise<config> — прочитать настройки (sync)
setConfig(patch)   // → Promise<void>  — записать изменённые настройки
getCache()         // → Promise<cache>  — прочитать кэш (local)
setCache(patch)    // → Promise<void>  — записать в кэш
getAuthBundle()    // → Promise<{ token, orgId, userId }> — извлечь auth bundle из кэша
```

### `lib/api.js`

```js
DEVIN_API_BASE     // "https://api.devin.ai/v1"
BALANCE_URL        // "https://server.codeium.com/api/v1/GetTeamCreditBalance"
RUNNING_STATUSES   // ["working", "blocked", "resume_requested", ...]

class ApiError(message, status)  // расширение Error с полем .status

listSessions(apiKey, { limit })  // → Promise<session[]>
terminateSession(apiKey, id)     // → Promise<response>
getTeamCreditBalance(serviceKey) // → Promise<{ remaining, ... }>
isRunning(session)               // → boolean
```

### `lib/internalApi.js`

```js
INTERNAL_API_BASE  // "https://app.devin.ai/api"

class AuthError(message, status)  // ошибка авторизации (401/403)

hasAuthBundle(auth)               // → boolean
getOnDemandBalance(auth)          // → Promise<number|null>
getPersonalApiKey(auth)           // → Promise<string> (apk_user_…)
pauseSessions(auth, ids, archive) // → POST /api/sessions/bulk-archive (пауза/возобновление)
```

### `lib/core.js`

```js
LOGIN_PROMPT                    // строка-приглашение войти

shouldTrigger(config, value)    // → boolean (пора ли останавливать)
stopRecentSessions(config, sessions, auth) // → Promise<report> (pause/terminate по stopMode)
refresh()                       // → Promise<cache> (один цикл опроса)
```

### `background.js`

Точка входа service worker. Экспортов нет.

- Создаёт будильник `"devin-balance-poll"`.
- Обрабатывает `onInstalled`, `onStartup`, `onAlarm`.
- Обрабатывает сообщения: `refresh`, `devinAuth`, `getState`, `stopAll`.
- Перенастраивает будильник при изменении `pollIntervalMinutes`.

### `popup.js`

UI попапа. Импортирует `isRunning` и `setConfig`. Рендерит баланс, сессии,
обрабатывает кнопки и inline-настройки.

### `options.js`

Страница настроек. Импортирует `getConfig`, `setConfig`, `getCache`,
`listSessions`, `getTeamCreditBalance`. Сохраняет конфиг, проверяет
подключение.

### `content/devin-bridge.js`

Content script (IIFE). Читает `localStorage` на `app.devin.ai`, отправляет
auth bundle в background. Повторяет при фокусе/visibility и каждые 60 с.

---

## 6. Локальный запуск и разработка

### 6.1 Загрузка расширения

1. Откройте `chrome://extensions` (или аналог).
2. Включите **Режим разработчика**.
3. **Загрузить распакованное расширение** → выберите папку `extension/`.

### 6.2 Цикл разработки

| Что изменили | Действие |
| --- | --- |
| `background.js`, `lib/*.js` | Нажать **↻ Reload** на карточке расширения |
| `content/devin-bridge.js` | Reload расширения **+** перезагрузить вкладку `app.devin.ai` |
| `popup.html/js`, `styles.css` | Закрыть и заново открыть попап |
| `options.html/js` | Перезагрузить страницу настроек |
| `manifest.json` | Reload расширения |

### 6.3 Отладка

- **Service worker**: `chrome://extensions` → карточка расширения → ссылка
  «service worker» (Inspect views) → откроется консоль фона.
- **Попап**: ПКМ по попапу → «Просмотреть код» → DevTools.
- **Content-скрипт**: DevTools на любой странице `app.devin.ai`, вкладка
  Console.
- **Кэш и конфиг**: в консоли фона выполнить:
  ```js
  chrome.storage.local.get(console.log)   // кэш
  chrome.storage.sync.get(console.log)    // настройки
  ```

---

## 7. Проверка кода

Перед коммитом рекомендуется проверить синтаксис:

```bash
# Проверить синтаксис всех JS-модулей
for f in extension/background.js extension/popup.js extension/options.js \
         extension/lib/*.js extension/content/*.js; do
  node --check "$f"
done

# Проверить валидность manifest.json
node -e "JSON.parse(require('fs').readFileSync('extension/manifest.json','utf8'))"
```

---

## 8. Сборка релиза

```bash
./scripts/build-release.sh
```

Создаёт:
- `release/devin-balance-guard/` — распакованное расширение (без `icons/icon_src.png`).
- `release/devin-balance-guard-<version>.zip` — архив для магазинов/раздачи.

Версия берётся из `extension/manifest.json` → поле `version`.

### Подписанный CRX + автообновление

Если задан ключ подписи, скрипт дополнительно собирает подписанный `.crx` и
update-манифест:

```bash
CRX_KEY=/path/to/devin-balance-guard.pem ./scripts/build-release.sh
# или в CI: CRX_KEY_BASE64=<base64 of .pem> ./scripts/build-release.sh
```

Дополнительно создаёт:
- `release/devin-balance-guard.crx` — подписанный CRX3 (self-hosted установка).
- `release/updates.xml` — Chrome update manifest (автообновление).

Подпись одним и тем же ключом всегда даёт один и тот же **ID расширения**
(`gdobkpjnopnafcbnhmfanccelegodchc`), что и обеспечивает автообновление.
Публикация (копирование `.crx`/`updates.xml` в `MikhailG517/devin` на GitHub
Pages) автоматизирована через GitHub Actions — см.
[`docs/SOURCE_MAP.md`](SOURCE_MAP.md) → `.github/workflows`.

> Приватный ключ **не** хранится в репозитории. В CI он подаётся секретом
> `CRX_KEY_BASE64`; для триггера публикации нужен секрет `RELEASE_DISPATCH_TOKEN`.

**Процесс нового релиза:**
1. Обновить `version` в `extension/manifest.json`.
2. Выполнить `./scripts/build-release.sh` (с `CRX_KEY`, если нужен `.crx`).
3. Закоммитить изменения + содержимое `release/` и запушить в `main` —
   GitHub Actions опубликует обновление, браузеры подтянут его автоматически.

---

## 9. Соглашения по коду

- Ванильный JS, ES-модули (`type: "module"` у service worker;
  `<script type="module">` для popup/options).
- Content script `devin-bridge.js` — IIFE (MV3 не поддерживает модули в
  content scripts).
- Без сторонних библиотек и бандлеров.
- Строки интерфейса — на русском (захардкожены в HTML/JS).
- Сетевые вызовы — только через `lib/api.js` (публичный) и
  `lib/internalApi.js` (внутренний).
- Новые хосты → добавлять в `host_permissions` манифеста.
- Хранение данных — только через `lib/storage.js` (`getConfig`, `setConfig`,
  `getCache`, `setCache`).

---

## 10. Тестирование (ручное)

Автотестов нет. Минимальный сценарий приёмки:

1. Загрузить расширение, войти в `app.devin.ai`. Режим ключа `auto`,
   источник `ondemand` — по умолчанию. Открыть настройки → «Проверить
   подключение» → ожидается «Ключ: OK · Сессии: OK · Баланс On-demand: OK».

2. Открыть попап → баланс отображается с `$`, бейдж синий.

3. Установить порог выше текущего баланса → бейдж красный; ниже → синий.

4. Список активных сессий отображается со статусами и ссылками.

5. Включить авто-остановку + порог выше баланса → «Обновить» → сессии
   приостанавливаются (режим `pause` по умолчанию), приходит уведомление.
   Проверить обратимость: сессия уходит в Archived и возвращается после
   разархивации.

> **Внимание:** в режиме `pause` (по умолчанию) остановка обратима; в
> режиме `terminate` — реальна и необратима. Тестировать авто-остановку
> только на отдельном аккаунте с одноразовыми сессиями.

---

## 11. Иконки

Иконки в `extension/icons/`:
- `icon16.png` (16×16) — фавикон в тулбаре.
- `icon32.png` (32×32) — фавикон @2x.
- `icon48.png` (48×48) — страница расширений.
- `icon128.png` (128×128) — Chrome Web Store, уведомления.

Исходный файл `icon_src.png` (если есть) — в `.gitignore`, не включается в
билд.
