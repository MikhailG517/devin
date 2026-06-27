# Devin Balance Guard — бинарники и автообновление

Этот репозиторий **раздаёт собранное десктоп-приложение** Devin Balance Guard
(трей: баланс Devin On-demand + остановка активных сессий) и обслуживает его
**проверку обновлений**. Исходный код, спецификации и инструкции живут в
[`MikhailG517/devin_cost`](https://github.com/MikhailG517/devin_cost) (папка
`desktop/`).

Содержимое публикуется через **GitHub Pages** на
`https://mikhailg517.github.io/devin/`:

| Путь | Назначение |
| --- | --- |
| `index.html` | Страница загрузки (Windows / macOS / Linux). |
| `downloads/devin-balance-guard-windows.zip` | Сборка для Windows (`.exe`). |
| `downloads/devin-balance-guard-macos.zip` | Сборка для macOS (`.app`). |
| `downloads/devin-balance-guard-linux.zip` | Сборка для Linux. |
| `latest.json` | Манифест версии (приложение читает его для проверки обновлений). |
| `docs/INSTALL.md` | Установка и запуск. |

## Установка

См. [`docs/INSTALL.md`](docs/INSTALL.md) или страницу
<https://mikhailg517.github.io/devin/>: скачать ZIP под свою ОС, распаковать,
запустить.

## Как обновляется

1. В `devin_cost` повышается `__version__` десктоп-приложения, изменения
   попадают в основную ветку.
2. CI (`devin_cost/.github/workflows/desktop-build.yml`) собирает бинарники под
   три ОС.
3. Новые ZIP-файлы и обновлённый `latest.json` (с новым `version`) кладутся
   сюда, в ветку `gh-pages`.
4. Приложение периодически читает `latest.json`; при более высокой версии
   показывает уведомление и пункт меню **«Скачать обновление»**, открывающий
   эту страницу.

> Тихая «бесшовная» замена работающего бинарника не делается намеренно: на
> Windows исполняемый файл заблокирован во время работы, на macOS нужно
> заменять подписанный `.app`. Поэтому обновление — это уведомление + ручная
> загрузка свежего ZIP (надёжно и одинаково на всех ОС).

## Настройка (один раз)

Включить **GitHub Pages**: Settings → Pages → Source = `Deploy from a branch`,
branch = `gh-pages`, folder = `/ (root)`. Ветка `gh-pages` — основная (default)
ветка этого репозитория.
