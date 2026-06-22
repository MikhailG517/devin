# Установка Devin Balance Guard

- **ID расширения:** `gdobkpjnopnafcbnhmfanccelegodchc`
- **Update URL:** `https://mikhailg517.github.io/devin/updates.xml`
- **CRX:** `https://mikhailg517.github.io/devin/devin-balance-guard.crx`

После установки любым способом **войдите в `https://app.devin.ai`** —
расширение берёт доступ к API из вашей активной сессии в браузере.

---

## Способ 1. Без «режима разработчика» (рекомендуется) — корпоративная политика

Chrome **намеренно** запрещает обычную установку `.crx` не из Web Store.
Единственный поддерживаемый способ установить self-hosted расширение в один
шаг и без режима разработчика — политика `ExtensionInstallForcelist`. Готовые
файлы лежат в [`policies/`](../policies).

### Linux (Chrome)

```bash
sudo mkdir -p /etc/opt/chrome/policies/managed
sudo curl -fsSL https://mikhailg517.github.io/devin/policies/linux/devin-balance-guard.json \
  -o /etc/opt/chrome/policies/managed/devin-balance-guard.json
# полностью закрыть и снова открыть Chrome
```

Проверка: откройте `chrome://policy` → должна быть строка
`ExtensionInstallForcelist`, и `chrome://extensions` покажет установленное
расширение (удалить его сможет только политика).

### Windows (Chrome / Edge)

1. Скачайте [`policies/windows/devin-balance-guard.reg`](../policies/windows/devin-balance-guard.reg).
2. Запустите двойным кликом (нужны права администратора), подтвердите импорт.
3. Полностью перезапустите браузер. Для Edge раскомментируйте блок Edge в файле.

### macOS (Chrome)

```bash
sudo cp policies/macos/com.google.Chrome.ExtensionInstallForcelist.plist \
  /Library/Managed\ Preferences/com.google.Chrome.plist
# полностью перезапустить Chrome
```

Во всех случаях обновления приходят автоматически через `updates.xml`.

---

## Способ 2. Chrome Web Store (самый простой для конечного пользователя)

Установка из Web Store работает «в один клик» без политик и без режима
разработчика, и тоже автообновляется. Требует разовой публикации в Web Store
под аккаунтом разработчика (единоразовый взнос Google и модерация).

Сборку для загрузки в Web Store берите из артефакта
`devin-balance-guard-<version>.zip` (репозиторий `devin_cost`, папка `release/`
или вкладка Actions). После публикации **уберите** `key`/`update_url` —
Web Store назначает свой ID и сам обслуживает обновления.

---

## Способ 3. Ручная установка `.crx` / распакованной папки (нужен режим разработчика)

Для разовой проверки:

1. Скачайте [`devin-balance-guard.crx`](https://mikhailg517.github.io/devin/devin-balance-guard.crx).
2. `chrome://extensions` → включите **Developer mode**.
3. Перетащите `.crx` в окно. (Chrome может потребовать включить расширение
   вручную.) Либо «Load unpacked» для распакованной папки из `release/`.

Этот способ — только для отладки; для постоянного использования берите способ 1
или 2.
