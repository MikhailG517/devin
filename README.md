# Devin Balance Guard — дистрибуция и автообновление

Этот репозиторий **раздаёт собранное расширение** Devin Balance Guard и
обслуживает его **автообновление**. Исходный код живёт в
[`MikhailG517/devin_cost`](https://github.com/MikhailG517/devin_cost).

Содержимое публикуется через **GitHub Pages** на
`https://mikhailg517.github.io/devin/`:

| Файл | Назначение |
| --- | --- |
| `devin-balance-guard.crx` | Подписанная сборка расширения (CRX3). |
| `updates.xml` | Манифест автообновления (Chrome `update_url`). |
| `index.html` | Страница установки. |
| `policies/` | Готовые политики для установки без «режима разработчика». |

- **ID расширения:** `gdobkpjnopnafcbnhmfanccelegodchc`
- **Update URL:** `https://mikhailg517.github.io/devin/updates.xml`

## Установка

См. [`docs/INSTALL.md`](docs/INSTALL.md) или страницу
<https://mikhailg517.github.io/devin/>.

Коротко (Linux/Chrome, без режима разработчика):

```bash
sudo mkdir -p /etc/opt/chrome/policies/managed
sudo curl -fsSL https://mikhailg517.github.io/devin/policies/linux/devin-balance-guard.json \
  -o /etc/opt/chrome/policies/managed/devin-balance-guard.json
# полностью перезапустить Chrome
```

## Как обновляется

1. В `devin_cost` повышается `version` в `extension/manifest.json` и изменения
   попадают в `main`.
2. GitHub Actions собирает подписанный `.crx` + `updates.xml` и публикует их
   сюда (workflow `.github/workflows/release.yml`).
3. Chrome периодически опрашивает `updates.xml` и сам ставит новую версию.

> Подпись всегда одним и тем же ключом → ID расширения не меняется → обновление
> «поверх» существующей установки работает.

## Первичная настройка (один раз)

Чтобы автопубликация заработала, в **Settings → Secrets and variables →
Actions** этого репозитория нужно добавить:

- `CRX_KEY_BASE64` — приватный ключ подписи (`base64` от `.pem`). Тот же ключ,
  что соответствует `key` в `extension/manifest.json` исходников.

И включить **GitHub Pages**: Settings → Pages → Source = `Deploy from a branch`,
branch = `main`, folder = `/ (root)`.
