# Ну, погоди! — Remaster

Браузерная переинтерпретация советской электронной игры «Электроника ИМ-02» (1984). Волк ловит яйца в четырёх позициях, скорость растёт, жизни тают, рекорд сохраняется.

Чистый HTML + JavaScript, без зависимостей. Запускается с локального http-сервера или из деплоя.

## Демо

_(добавить ссылку после деплоя в Phase 7)_

## Стек

- Vanilla JavaScript (ES6+ modules)
- Canvas 2D API
- Web Audio API
- localStorage для рекордов и настроек
- Никаких npm-зависимостей и сборщиков

## Запуск локально

ES modules требуют http(s), двойной клик по `index.html` не подойдёт.

```bash
# Python 3
python3 -m http.server 8000

# или Node.js
npx serve

# или в VSCode — расширение Live Server
```

Открыть `http://localhost:8000` в браузере.

## Структура проекта

```
/
├── index.html
├── style.css
├── README.md
├── src/                 # игровой код
├── assets/
│   ├── sprites/         # графика
│   └── audio/           # звуки
└── docs/                # документация и промпты
    ├── 00-readme.md
    ├── 01-gdd.md
    ├── 02-architecture.md
    ├── 03-art-direction.md
    ├── 04-asset-manifest.md
    ├── 05-audio-manifest.md
    ├── 06-roadmap.md
    ├── 07-progress-log.md
    ├── 08-decisions.md
    ├── 09-workflow.md
    └── 10-prompts/
        └── nano-banana/  # промпты для генерации ассетов
```

## Документация

Вся работа над проектом ведётся по документам в `docs/`. Полный список: [`docs/00-readme.md`](docs/00-readme.md).

Управление и геймплей: [`docs/01-gdd.md`](docs/01-gdd.md).

Перед любой правкой кода — открыть релевантные документы.

## Текущий статус

См. [`docs/06-roadmap.md`](docs/06-roadmap.md).

## Контрибьюции

Проект личный, но если хочется предложить идею — issue или PR. Перед PR — прочитать [`docs/07-progress-log.md`](docs/07-progress-log.md).

## Лицензия

_(определить перед публичным релизом — MIT или CC BY-NC)_

## Благодарности

- «Электроника ИМ-02», 1984 — оригинальная игра, без неё ничего бы не было.
- Nano Banana — генератор пиксель-арта для remaster-скина.