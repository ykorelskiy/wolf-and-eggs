# Архитектура

## Поток управления

```
index.html
  └─ src/main.js (entry point)
       └─ Game (src/game.js) — оркестратор
            ├─ src/core/state-manager
            ├─ src/core/input-manager  ─┐
            ├─ src/core/audio-manager   │
            ├─ src/core/asset-loader    │
            ├─ src/renderer/renderer    │  все управляются Game
            ├─ src/entities/wolf        │
            ├─ src/systems/egg-spawner  │
            ├─ src/systems/score        │
            ├─ src/systems/difficulty   │
            └─ src/ui/ui               ─┘
```

## Модули и контракты

### `src/config.js`

Все константы. Никакой логики.

```js
export const CONFIG = {
  LOGICAL_WIDTH: 320,
  LOGICAL_HEIGHT: 240,
  RENDER_WIDTH: 640,
  RENDER_HEIGHT: 480,
  TARGET_FPS: 60,
  RENDER_MODE: 'remaster', // 'remaster' | 'lcd'
  STARTING_LIVES: 3,
  POSITIONS: {
    TOP_LEFT: 0,
    BOTTOM_LEFT: 1,
    TOP_RIGHT: 2,
    BOTTOM_RIGHT: 3
  },
  DIFFICULTY_CURVE: [ /* см. GDD */ ],
  DEATH_PENALTY_FACTOR: 1.3,
  DEATH_PENALTY_RECOVERY_MS: 10000
}
```

### `src/main.js`

Точка входа. Создаёт `Game`, вызывает `game.start()`.

### `src/game.js` — Game

**Роль:** оркестратор. Держит game loop через `requestAnimationFrame`.

**Публичное API:**
- `start()`
- `pause()`
- `resume()`
- `reset()`

**Внутренние методы:**
- `update(dt)` — вызывает update всех систем
- `render()` — вызывает renderer

**Зависимости:** все менеджеры.

### `src/core/state-manager.js` — StateManager

**Роль:** конечный автомат состояний игры.

**State:** `'MENU' | 'PLAYING' | 'PAUSED' | 'GAME_OVER'`

**API:**
- `current` (getter)
- `transition(newState)` — с валидацией перехода
- `on(state, callback)` — подписка на вход в state

### `src/core/input-manager.js` — InputManager

**Роль:** ввод с клавиатуры и touch.

**API:**
- `init()` — навешивает listeners
- `on(event, callback)` — подписка
- `destroy()` — снимает listeners

**Эмитит события через event-bus:**
- `'input:move'` с `{ direction: 'left'|'right' }`
- `'input:pause'` с `{}`
- `'input:confirm'` с `{}`

**НЕ ЗНАЕТ** о Wolf, Game и прочем.

### `src/entities/wolf.js` — Wolf

**Роль:** состояние и отрисовка волка.

**State:**
- `position: 0..3`

**API:**
- `setPosition(p)`
- `getCatchZone()` — возвращает прямоугольник, где волк ловит яйцо
- `render(ctx, renderer)` — вызывает renderer для отрисовки

**Подписан на:** событие `'input:move'` от InputManager.

**Wolf пассивен и НЕ эмитит события шины.**

### `src/systems/egg-spawner.js` — EggSpawner

**Роль:** управление всеми яйцами на экране.

**State:**
- `eggs: Egg[]` — массив активных яиц

**API:**
- `update(dt, difficultyParams)` — двигает яйца, спавнит новые
- `checkCollision(wolf)` → `{ caught: Egg[], missed: Egg[] }`
- `clear()` — очищает все яйца (на game over / reset)
- `render(ctx, renderer)`

**Эмитит события через event-bus:**
- `'egg:spawned'` с `{ id, lane, speed }`
- `'egg:caught'` с `{ id, lane }`
- `'egg:missed'` с `{ id, lane }`

#### Egg (внутренняя структура)

```js
{
  id: number,
  chuteIndex: 0..3,     // на каком жёлобе
  progress: 0..1,        // 0 = верх жёлоба, 1 = край
  fallDuration: number,  // мс на полное скатывание
  state: 'falling' | 'caught' | 'breaking'
}
```

### `src/systems/score.js` — Score

**Роль:** счёт, жизни, рекорд.

**State:**
- `score: number`
- `lives: number`
- `highScore: number` (из localStorage)

**API:**
- `addScore(n)`
- `loseLife()` → `boolean` (true если жизней 0)
- `reset()`
- `saveHighScore()`

**Эмитит события через event-bus:**
- `'life:lost'` с `{ livesLeft }`
- `'score:changed'` с `{ score, best }`

### `src/systems/difficulty.js` — Difficulty

**Роль:** вычисление текущих параметров сложности.

**API:**
- `getParams(score, timeSinceLastDeath)` → `{ fallDuration, spawnInterval }`

Применяет:
1. Базовую кривую от score (интерполяция между точками)
2. Death penalty (если `timeSinceLastDeath < 10000`)

### `src/core/asset-loader.js` — AssetLoader

**Роль:** загрузка и кэширование PNG/JSON ассетов (спрайтов, звуков).

**API:**
- `load()` → Promise — загружает все ассеты из манифеста
- `getSprite(id)` → ImageBitmap
- `getSound(id)` → ArrayBuffer

### `src/renderer/renderer.js` — Renderer

**Роль:** вся отрисовка. Знает о `RENDER_MODE`.

**API:**
- `init(canvas)`
- `clear()`
- `drawWolf(position)`
- `drawEgg(chuteIndex, progress, state)`
- `drawBackground()`
- `drawUI(score, lives, highScore)`
- `drawGameOver(score, isNewRecord)`
- `screenShake(intensity, duration)`

**Внутри:** выбирает sprite sheet и правила анимации на основе `CONFIG.RENDER_MODE`. В LCD-режиме `progress` яйца округляется до ближайшей из 3 дискретных позиций перед отрисовкой.

### `src/core/audio-manager.js` — AudioManager

**Роль:** воспроизведение SFX и музыки.

**API:**
- `init()` — загрузка звуков
- `play(soundId)`
- `setMuted(bool)`
- `playMusic(trackId)` / `stopMusic()`

Внутри выбирает банк звуков по `CONFIG.RENDER_MODE`.

### `src/core/event-bus.js` — EventBus

**Роль:** глобальная шина событий.

**API:**
- `on(event, handler)` — подписаться
- `emit(event, payload)` — отправить

### `src/core/events.js` — EVENTS

**Роль:** константы событий.

```js
export const EVENTS = {
  INPUT_MOVE: 'input:move',
  INPUT_PAUSE: 'input:pause',
  INPUT_CONFIRM: 'input:confirm',
  EGG_SPAWNED: 'egg:spawned',
  EGG_CAUGHT: 'egg:caught',
  EGG_MISSED: 'egg:missed',
  LIFE_LOST: 'life:lost',
  SCORE_CHANGED: 'score:changed',
  GAME_OVER: 'game:over'
}
```

### `src/ui/ui.js` — UI

**Роль:** отрисовка меню, паузы, game over, HUD.

**API:**
- `drawMenu(highScore)`
- `drawHUD(score, lives)`
- `drawPause()`
- `drawGameOver(score, isNewRecord)`

## События

Модули не вызывают друг друга напрямую. Вся коммуникация — через глобальную шину событий.

### Шина

Один объект-синглтон с двумя методами:

- `bus.on(event, handler)` — подписаться
- `bus.emit(event, payload)` — отправить

Реализация — в `src/core/event-bus.js`. Все модули импортируют один и тот же экземпляр.

### Список событий

| Событие         | Кто эмитит       | Кто слушает            | Payload                           |
|-----------------|------------------|------------------------|-----------------------------------|
| `input:move`    | `input-manager`  | `wolf`                 | `{ direction: 'left'|'right' }`   |
| `input:pause`   | `input-manager`  | `game`                 | `{}`                              |
| `input:confirm` | `input-manager`  | `game`                 | `{}`                              |
| `egg:spawned`   | `egg-spawner`    | (логирование)          | `{ id, lane, speed }`             |
| `egg:caught`    | `egg-spawner`    | `score`                | `{ id, lane }`                    |
| `egg:missed`    | `egg-spawner`    | `score`, `wolf`        | `{ id, lane }`                    |
| `life:lost`     | `score`          | `game`                 | `{ livesLeft }`                   |
| `score:changed` | `score`          | `hud`                  | `{ score, best }`                 |
| `game:over`     | `game`           | `hud`, `input-manager` | `{ score, best, isNewBest }`      |

### Соглашения по именованию

- Формат: `subject:action` (двоеточие как разделитель).
- `subject` — сущность или подсистема (`input`, `egg`, `score`, `game`, `life`).
- `action` — глагол в прошедшем времени (`spawned`, `caught`, `missed`, `changed`, `lost`) или для ввода — глагол в инфинитиве (`move`, `pause`, `confirm`).
- Названия событий — строковые литералы, перечислены в `src/core/events.js` как константы:

  ```js
  export const EVENTS = {
    INPUT_MOVE: 'input:move',
    EGG_CAUGHT: 'egg:caught',
    // ...
  };
  ```

- Использовать константы, а не строки, при `emit`/`on`.

### Правила

- Payload — всегда объект (даже если одно поле). Это позволяет добавлять поля без поломки слушателей.
- Эмитер не знает о слушателях. Если слушателей нет — это не ошибка.
- Подписка — в конструкторе/инициализации модуля. Отписка — при уничтожении (актуально для рестарта игры).
- Wolf пассивен и не эмитит события. Все события, связанные с яйцами (egg:caught, egg:missed), эмитит EggSpawner.

## Поток данных в одном кадре

```
1.  Game.loop(dt):
2.    InputManager обработал события → bus.emit('input:move', ...)
3.    Wolf подписан на 'input:move' → обновил position
4.    Difficulty.getParams(score, timeSinceDeath) → params
5.    EggSpawner.update(dt, params) → двигает яйца, спавнит, bus.emit('egg:spawned', ...)
6.    EggSpawner.checkCollision(wolf):
        - caught → Score.addScore(1), audio.play('catch'), bus.emit('egg:caught', ...)
        - missed → Score.loseLife(), audio.play('miss'),
                   renderer.screenShake(), startDeathPenaltyTimer(),
                   bus.emit('egg:missed', ...)
7.    Если жизней 0 → StateManager.transition('GAME_OVER'), bus.emit('game:over', ...)
8.    Renderer.clear()
9.    Renderer.drawBackground()
10.   EggSpawner.render()
11.   Wolf.render()
12.   UI.drawHUD()
```

## Skinning (визуальные режимы)

`CONFIG.RENDER_MODE` управляет:
1. Какой sprite sheet грузится в AssetLoader
2. Какой банк звуков использует AudioManager
3. Правила анимации в Renderer (плавная для `remaster`, дискретная для `lcd`)

Логика игры (Wolf, EggSpawner, Score, Difficulty) **абсолютно идентична** в обоих режимах.

В Phase 5 добавляется переключатель в меню, который меняет `CONFIG.RENDER_MODE` и перезагружает ассеты.