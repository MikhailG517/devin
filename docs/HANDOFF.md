# Передача проекта (Handoff) — Devin Balance Guard

Документ для **продолжения работы в новом аккаунте** (Devin и/или GitHub).
Описывает текущее состояние, что обязательно перенести, как пересобрать и
опубликовать обновление и какие нюансы аккаунта учитывать.

---

## 1. Что это и где лежит

| Репозиторий | Назначение |
| --- | --- |
| [`MikhailG517/devin_cost`](https://github.com/MikhailG517/devin_cost) | **Исходный код** расширения, скрипты сборки/подписи, документация. |
| [`MikhailG517/devin`](https://github.com/MikhailG517/devin) | **Дистрибуция и автообновление**: подписанный `.crx`, `updates.xml`, страница установки, политики. Раздаётся через GitHub Pages. |

- **ID расширения (стабильный):** `gdobkpjnopnafcbnhmfanccelegodchc`
- **Update URL:** `https://mikhailg517.github.io/devin/updates.xml`
- **Страница установки:** `https://mikhailg517.github.io/devin/`
- **Текущая версия:** `1.1.1` (см. `extension/manifest.json`).

Документация в `devin_cost/docs/`: [`SPEC.md`](SPEC.md) (спецификация),
[`SOURCE_MAP.md`](SOURCE_MAP.md) (карта кода по файлам),
[`DEVELOPMENT.md`](DEVELOPMENT.md) (сборка/отладка/релиз),
[`INSTALL.md`](INSTALL.md) (установка и настройка).

---

## 2. Что уже сделано

1. **Получение API-ключа усилено.** Мост (`content/devin-bridge.js`) инжектится
   в уже открытые вкладки `app.devin.ai` при установке (право `scripting`);
   долгоживущий личный ключ `apk_user_…` больше не стирается при истечении
   короткого токена.
2. **Автообновление + self-host.** Стабильный `key` в манифесте → постоянный ID,
   `update_url`, подписчик CRX3 без зависимостей (`scripts/pack-crx.js`),
   генератор `updates.xml` (`scripts/make-updates-xml.js`), публикация через
   GitHub Pages, CI/CD (`devin/.github/workflows/release.yml`).
   Установка **без режима разработчика** — через корпоративную политику
   `ExtensionInstallForcelist` (готовые файлы в `devin/policies/`).
   Проверено вживую: Chrome сам обновился `1.1.0 → 1.1.1`.
3. **Пауза вместо полной остановки.** По умолчанию `stopMode: "pause"` —
   усыпление сессий через `POST app.devin.ai/api/sessions/bulk-archive`
   (обратимо). Режим `"terminate"` оставлен опцией (необратимый `DELETE` v1).
   Проверена обратимость: сессия уходит в Archived и возвращается после
   разархивации.
4. **Защита сессий и подтверждение.** Allow-list `protectedSessionIds`
   (никогда не останавливаются) и `confirmBeforeStop` перед ручной кнопкой.

PR с этими изменениями: <https://github.com/MikhailG517/devin_cost/pull/7>.

---

## 3. КРИТИЧНО перенести в новый аккаунт

### 3.1 Приватный ключ подписи (`.pem`)

**Без него автообновление ломается навсегда.** ID расширения выводится из этого
ключа; другой ключ = другой ID = браузеры не увидят обновление как «то же
расширение» (придётся переустанавливать у всех пользователей).

- Файл: приватный ключ RSA, соответствующий полю `key` в
  `extension/manifest.json` (публичная часть base64 в манифесте).
- Текущий ID, который должен сохраниться: `gdobkpjnopnafcbnhmfanccelegodchc`.
- Храните `.pem` в надёжном секрет-хранилище. **Никогда не коммитьте его в git.**

> Проверка соответствия ключа и ID: соберите `.crx` этим ключом
> (`CRX_KEY=path/to.pem bash scripts/build-release.sh`) и убедитесь, что
> сгенерированный `updates.xml` содержит `appid='gdobkpjnopnafcbnhmfanccelegodchc'`.

### 3.2 Секрет CI для автопубликации

В репозитории `devin` (**Settings → Secrets and variables → Actions**) должен
быть секрет:

- **`CRX_KEY_BASE64`** — base64 от приватного `.pem`. Сгенерировать:
  ```bash
  base64 -w0 path/to/devin-balance-guard.pem    # Linux
  base64 -i  path/to/devin-balance-guard.pem    # macOS
  ```
  Вставьте полученную строку как значение секрета.

Опционально, в репозитории `devin_cost`:

- **`RELEASE_DISPATCH_TOKEN`** — PAT с доступом к `devin`. Если задан, пуш в
  `main` (`devin_cost`) автоматически триггерит публикацию в `devin`
  (workflow `notify-release.yml` → `repository_dispatch`). Без него публикацию
  запускают вручную (Actions → «Publish extension» → Run workflow).

### 3.3 GitHub Pages

В репозитории `devin`: **Settings → Pages → Source = «Deploy from a branch»**,
branch = **`gh-pages`**, folder = **`/ (root)`**. Ветка `gh-pages` — основная
(default) ветка `devin`: в ней и публикуемые файлы, и workflow автопубликации.

---

## 4. Если меняется аккаунт/владелец (новый username)

URL-ы автообновления и Pages зашиты в код. При смене GitHub-владельца
(`MikhailG517` → другой) обновите согласованно:

| Где | Что поменять |
| --- | --- |
| `extension/manifest.json` | `update_url` → `https://<NEW_USER>.github.io/devin/updates.xml` |
| `devin/.github/workflows/release.yml` | `repository: <NEW_USER>/devin_cost` |
| `devin_cost/.github/workflows/notify-release.yml` | целевой репозиторий dispatch |
| `devin/policies/**`, `devin/index.html`, `devin/README.md`, `docs/INSTALL.md` | все ссылки `mikhailg517.github.io/devin/...` |

> **Поле `key` и `.pem` НЕ меняйте** — именно они держат стабильный ID. Смена
> владельца репозитория не требует смены ключа подписи.

После смены `update_url` поднимите версию и переиздайте `.crx`/`updates.xml`.

---

## 5. Как выпустить новую версию

1. Внесите изменения в `extension/**`.
2. Поднимите `version` в `extension/manifest.json` (например `1.1.1 → 1.1.2`).
3. Локально проверьте сборку:
   ```bash
   CRX_KEY=path/to/devin-balance-guard.pem bash scripts/build-release.sh
   # → release/devin-balance-guard.crx и release/updates.xml
   ```
4. Влейте в `main` (`devin_cost`).
5. Публикация в `devin`:
   - авто (если задан `RELEASE_DISPATCH_TOKEN`) — по пушу в `main`;
   - иначе вручную: репозиторий `devin` → Actions → **Publish extension** →
     Run workflow.
6. Chrome у установленных пользователей подтянет новую версию по `updates.xml`
   (можно форсировать: `chrome://extensions` → «Обновить»).

Подробности сборки/подписи — в [`DEVELOPMENT.md`](DEVELOPMENT.md).

---

## 6. Нюансы аккаунта и тестирования (важно при повторных тестах)

- **Публичный API быстро отдаёт `finished`.** На тестовом аккаунте сессии в
  публичном API v1 переходят в `finished` за секунды после создания (даже если
  в UI «Working») — сессии Devin почти сразу «засыпают». Поэтому попап может
  показывать 0 активных. Для проверки остановки/паузы удобно инжектировать
  синтетическую сессию с реальным ID в кэш расширения.
- **Гонка холодного старта service worker.** При «пробуждении» worker сразу
  делает поллинг, который может перезатереть инжектированные тестовые данные.
  Обходной путь: держать DevTools фона открытым или использовать keepalive-цикл
  во время теста.
- **Автообновление только для policy/CRX-установки.** У распакованного
  расширения (Load unpacked) автообновления нет в принципе — оно работает
  только для установки из `.crx` с `update_url` (т.е. через
  `ExtensionInstallForcelist`).
- **Пауза требует входа в `app.devin.ai`.** `bulk-archive` — внутренний
  эндпоинт; нужен живой auth-бандл из вкладки `app.devin.ai`. Полная остановка
  (`terminate`) работает по личному ключу `apk_user_…`.
- Скилл по сквозному тестированию: `.agents/skills/testing-balance-guard/`.

---

## 7. Бэклог / не сделано

- **Публикация в Chrome Web Store** — единственный путь установки «в один клик»
  без корпоративной политики. Требует Google-аккаунт разработчика и модерацию.
  ZIP для загрузки собирается через `scripts/build-release.sh`.
- Реальная демонстрация `resume` после паузы доведена до возврата сессии в
  активный список; полноценный прогон «возобновлена и снова Working» можно
  расширить.
