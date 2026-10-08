# Техническая спецификация: «Classic Snake Web» (Single-file)

**Назначение:** Документ предназначен для автономной кодогенерации и пошаговой реализации агентом Antigravity CLI.  
**Целевой артефакт:** Единственный файл `index.html` (включающий `<style>` и `<script>`). Без внешних зависимостей (шрифтов, CDN, изображений, сторонних библиотек).

---

## 1. Архитектурный контракт и ограничения

1. **Автономность:** Приложение запускается простым открытием файла `file:///path/to/index.html` в браузере, а также через локальный HTTP-сервер. Никаких сетевых запросов (Network Zero).
2. **Файловая структура:**
   ```text
   .
   └── index.html  # Содержит HTML5, вложенный CSS (<style>), JS-логику (<script>)
   ```
3. **Хранилище:** `localStorage` по ключу `snake_high_score`.
4. **Рендеринг:** HTML5 Canvas 2D Context с аппаратным масштабированием под Retina/HiDPI (`devicePixelRatio`).
5. **Игровой цикл:** `requestAnimationFrame` с интервальным расчетом дельты времени (time accumulator pattern) для независимости от частоты обновления экрана (60 Hz / 120 Hz / 144 Hz).

---

## 2. Структура DOM (HTML Layout)

```html
<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>Classic Snake</title>
  <style>/* См. раздел 3 */</style>
</head>
<body>
  <div id="app">
    <!-- Верхняя панель со счетом и кнопками управления UI -->
    <header id="scoreboard">
      <div class="score-box">СЧЕТ: <span id="current-score">0</span></div>
      <div class="score-box">РЕКОРД: <span id="high-score">0</span></div>
      <button id="btn-pause" aria-label="Пауза" class="icon-btn">⏸</button>
    </header>

    <!-- Игровая сцена с Canvas и оверлеями -->
    <main id="game-stage">
      <canvas id="game-canvas"></canvas>

      <!-- Оверлей модальных экранов -->
      <div id="overlay" class="overlay hidden">
        <div class="modal-card">
          <h2 id="modal-title">ЗМЕЙКА</h2>
          <p id="modal-subtitle">Используйте стрелки, WASD или свайпы</p>
          <div id="modal-actions">
            <button id="btn-action" class="primary-btn">ИГРАТЬ</button>
          </div>
        </div>
      </div>
    </main>

    <!-- Виртуальный D-Pad для мобильных экранов (показывается по медиа-запросу) -->
    <nav id="touch-controls" class="touch-controls">
      <button class="dpad-btn" data-dir="UP" id="dpad-up">▲</button>
      <div class="dpad-middle">
        <button class="dpad-btn" data-dir="LEFT" id="dpad-left">◀</button>
        <div class="dpad-spacer"></div>
        <button class="dpad-btn" data-dir="RIGHT" id="dpad-right">▶</button>
      </div>
      <button class="dpad-btn" data-dir="DOWN" id="dpad-down">▼</button>
    </nav>
  </div>
  <script>/* См. разделы 4, 5, 6, 7 */</script>
</body>
</html>
```

---

## 3. Стилизация и адаптивность (CSS)

### 3.1. CSS-переменные и базовая геометрия
```css
:root {
  --bg-color: #0f172a;
  --surface-color: #1e293b;
  --accent-color: #22c55e;
  --accent-glow: rgba(34, 197, 94, 0.4);
  --food-color: #ef4444;
  --text-primary: #f8fafc;
  --text-muted: #94a3b8;
  --board-size: min(85vw, 65vh, 480px);
}
```

### 3.2. Ограничения мобильного ввода и скролла
* `html, body`:
  * `height: 100%`, `overflow: hidden`, `margin: 0`, `padding: 0`.
  * `touch-action: none` (критично для устранения pull-to-refresh, зума и паразитного скролла).
  * `user-select: none; -webkit-user-select: none;` — отключение выделения текста.
* `#app`: Flex-контейнер (`flex-direction: column`, `align-items: center`, `justify-content: space-between`, `height: 100%`, `padding: 12px`).

### 3.3. Canvas Container
* `#game-stage`: Позиционирование `relative`, размеры строго равны `var(--board-size)`.
* `#game-canvas`: Размеры CSS `100%` на `100%`, `border-radius: 8px`, `background: #020617`, `box-shadow: 0 0 20px rgba(0,0,0,0.5)`.

### 3.4. Адаптивность D-Pad
* По умолчанию `#touch-controls` имеет `display: flex; flex-direction: column; align-items: center;`.
* Медиа-запрос для десктопа:
  ```css
  @media (hover: hover) and (pointer: fine) and (min-width: 769px) {
    #touch-controls {
      display: none; /* Скрываем экранные кнопки на мыши/клавиатуре */
    }
  }
  ```
* Кнопки D-Pad: размер не менее `48px × 48px` (соответствие рекомендациям доступности WCAG/Apple HIG), с `touch-action: manipulation`.

---

## 4. Архитектура скрипта и структуры данных (JS)

### 4.1. Конфигурация (`CONFIG`)
```javascript
const CONFIG = {
  GRID_SIZE: 20,           // 20x20 ячеек
  INITIAL_SPEED_MS: 150,   // Базовый тик движения (мс)
  MIN_SPEED_MS: 70,        // Максимальная скорость игры
  SPEED_STEP_MS: 2,        // Ускорение за каждую съеденную еду
  SWIPE_THRESHOLD_PX: 30,  // Минимальная длина свайпа для детекции
  STORAGE_KEY: 'snake_high_score'
};
```

### 4.2. Машина состояний (`GameState`)
```javascript
const GameState = {
  START: 'START',
  PLAYING: 'PLAYING',
  PAUSED: 'PAUSED',
  GAME_OVER: 'GAME_OVER'
};
```

### 4.3. Направления (`Direction`)
```javascript
const Direction = {
  UP:    { x:  0, y: -1 },
  DOWN:  { x:  0, y:  1 },
  LEFT:  { x: -1, y:  0 },
  RIGHT: { x:  1, y:  0 }
};
```

### 4.4. Состояние игрового процесса (`AppState`)
```javascript
const state = {
  status: GameState.START,
  score: 0,
  highScore: 0,
  speedMs: CONFIG.INITIAL_SPEED_MS,
  snake: [], // Array<{x: number, y: number}>; [0] - голова
  direction: Direction.RIGHT,
  nextDirection: Direction.RIGHT, // Буфер направления для защиты от 180° в рамках тика
  food: { x: 0, y: 0 },
  lastTickTimestamp: 0,
  touchStartX: 0,
  touchStartY: 0
};
```

---

## 5. Алгоритмическая спецификация механик

### 5.1. HiDPI Canvas Scaling (Четкость рендера)
Функция масштабирования вызывается при инициализации и событии `window.onresize`:
```javascript
function resizeCanvas(canvas, containerSize) {
  const dpr = window.devicePixelRatio || 1;
  canvas.width = containerSize * dpr;
  canvas.height = containerSize * dpr;
  const ctx = canvas.getContext('2d');
  ctx.resetTransform();
  ctx.scale(dpr, dpr);
}
```

### 5.2. Предотвращение самоубийства при быстром вводе (Input Buffer Pattern)
**Проблема:** Игрок двигается `RIGHT`. Быстро жмет `UP`, затем `LEFT` до наступления следующего тика. Если менять направление сразу в `direction`, змейка развернется сама в себя.  
**Решение:**
1. Нажатие клавиши пишет во временную переменную `state.nextDirection`, если `nextDirection` не противоположен текущему `state.direction`.
2. Только в момент фактического шага сетки (Tick) выполняется:
   ```javascript
   state.direction = state.nextDirection;
   ```

### 5.3. Генерация еды без коллизий (Food Spawn Algorithm)
```javascript
function spawnFood(snake, gridSize) {
  const emptyCells = [];
  const occupied = new Set(snake.map(segment => `${segment.x},${segment.y}`));

  for (let x = 0; x < gridSize; x++) {
    for (let y = 0; y < gridSize; y++) {
      if (!occupied.has(`${x},${y}`)) {
        emptyCells.push({ x, y });
      }
    }
  }

  if (emptyCells.length === 0) {
    // Редкий кейс: победа (заполнено все поле)
    return null;
  }

  const randomIndex = Math.floor(Math.random() * emptyCells.length);
  return emptyCells[randomIndex];
}
```

### 5.4. Движение и расчет коллизий
На каждом тике времени:
1. Вычислить координаты новой головы:
   ```javascript
   const newHead = {
     x: state.snake[0].x + state.direction.x,
     y: state.snake[0].y + state.direction.y
   };
   ```
2. **Проверка стен:**
   `newHead.x < 0 || newHead.x >= CONFIG.GRID_SIZE || newHead.y < 0 || newHead.y >= CONFIG.GRID_SIZE` -> `triggerGameOver()`.
3. **Проверка тела:**
   Если `state.snake.some(seg => seg.x === newHead.x && seg.y === newHead.y)` -> `triggerGameOver()`.
4. **Проверка еды:**
   * Если `newHead.x === state.food.x && newHead.y === state.food.y`:
     * Змейка растет: `state.snake.unshift(newHead)` (хвост **не** удаляется).
     * `state.score += 10`.
     * `state.speedMs = Math.max(CONFIG.MIN_SPEED_MS, state.speedMs - CONFIG.SPEED_STEP_MS)`.
     * `state.food = spawnFood(...)`.
     * Воспроизвести аудиоэффект «съедание».
   * Иначе:
     * `state.snake.unshift(newHead)`.
     * `state.snake.pop()` (удаляем сегмент хвоста).

---

## 6. Управление (Input Controllers)

### 6.1. Десктопный ввод (Keyboard)
* Слушать событие `window.addEventListener('keydown', handleKeyDown)`.
* Привязка клавиш:
  * Вверх: `ArrowUp`, `KeyW`
  * Вниз: `ArrowDown`, `KeyS`
  * Влево: `ArrowLeft`, `KeyA`
  * Вправо: `ArrowRight`, `KeyD`
  * Пауза: `Space`, `KeyP`
  * Перезапуск (в состоянии `GAME_OVER`): `Enter`, `Space`
* Вызывать `event.preventDefault()` для задействованных клавиш, предотвращая скролл страницы стрелками.

### 6.2. Мобильный ввод: Touch-свайпы
Слушатели на элементе `#game-stage`:
```javascript
let touchStartX = 0;
let touchStartY = 0;

canvas.addEventListener('touchstart', (e) => {
  touchStartX = e.changedTouches[0].clientX;
  touchStartY = e.changedTouches[0].clientY;
}, { passive: false });

canvas.addEventListener('touchend', (e) => {
  const dx = e.changedTouches[0].clientX - touchStartX;
  const dy = e.changedTouches[0].clientY - touchStartY;
  
  if (Math.abs(dx) < CONFIG.SWIPE_THRESHOLD_PX && Math.abs(dy) < CONFIG.SWIPE_THRESHOLD_PX) {
    return; // Недостаточная амплитуда (клик/тап)
  }

  if (Math.abs(dx) > Math.abs(dy)) {
    // Горизонтальный свайп
    if (dx > 0) tryChangeDirection(Direction.RIGHT);
    else tryChangeDirection(Direction.LEFT);
  } else {
    // Вертикальный свайп
    if (dy > 0) tryChangeDirection(Direction.DOWN);
    else tryChangeDirection(Direction.UP);
  }
}, { passive: false });
```

### 6.3. Мобильный ввод: D-Pad
* Кнопкам `#touch-controls .dpad-btn` навешиваются обработчики `touchstart` (или `pointerdown`).
* Для исключения задержек браузера на touch-событиях использовать:
  ```javascript
  btn.addEventListener('pointerdown', (e) => {
    e.preventDefault();
    const dirKey = e.currentTarget.dataset.dir;
    tryChangeDirection(Direction[dirKey]);
  });
  ```

---

## 7. Аудиоэффекты (Web Audio API Synth)

Для сохранения нулевой зависимости от ассетов использовать встроенный генератор частот.

```javascript
class SoundController {
  constructor() {
    this.ctx = null;
  }
  
  init() {
    if (!this.ctx) {
      const AudioCtx = window.AudioContext || window.webkitAudioContext;
      this.ctx = new AudioCtx();
    }
    if (this.ctx.state === 'suspended') {
      this.ctx.resume();
    }
  }

  playEat() {
    if (!this.ctx) return;
    const osc = this.ctx.createOscillator();
    const gain = this.ctx.createGain();
    osc.type = 'sine';
    osc.frequency.setValueAtTime(600, this.ctx.currentTime);
    osc.frequency.exponentialRampToValueAtTime(880, this.ctx.currentTime + 0.1);
    gain.gain.setValueAtTime(0.15, this.ctx.currentTime);
    gain.gain.linearRampToValueAtTime(0.01, this.ctx.currentTime + 0.1);
    osc.connect(gain);
    gain.connect(this.ctx.destination);
    osc.start();
    osc.stop(this.ctx.currentTime + 0.1);
  }

  playGameOver() {
    if (!this.ctx) return;
    const osc = this.ctx.createOscillator();
    const gain = this.ctx.createGain();
    osc.type = 'sawtooth';
    osc.frequency.setValueAtTime(220, this.ctx.currentTime);
    osc.frequency.exponentialRampToValueAtTime(55, this.ctx.currentTime + 0.3);
    gain.gain.setValueAtTime(0.2, this.ctx.currentTime);
    gain.gain.linearRampToValueAtTime(0.01, this.ctx.currentTime + 0.3);
    osc.connect(gain);
    gain.connect(this.ctx.destination);
    osc.start();
    osc.stop(this.ctx.currentTime + 0.3);
  }
}
```
*Важно:* Метод `sound.init()` вызывается по первому пользовательскому жесту (клик по кнопке «Старт» / «Играть»), соблюдая правила автовоспроизведения (Autoplay Policy).

---

## 8. Отрисовка графики (Render Pipeline)

Рендеринг выполняется процедурно:
1. **Очистка и фон:** Очистка `clearRect`, заливка цветом сетки.
2. **Отрисовка сетки (Subtle Grid):** Тонкие линии с прозрачностью `rgba(255, 255, 255, 0.03)` для аркадного вида.
3. **Еда:**
   * Круг или скругленный квадрат красного цвета с легким свечением `ctx.shadowColor = '#ef4444'`.
4. **Змейка:**
   * Голова змейки: скругленный прямоугольник или круг акцентного цвета (зеленый / изумрудный). Рисуются два глаза (точки контрастного цвета) в зависимости от текущего направления движения.
   * Тело: закругленные сегменты с плавным переходом радиуса от головы к кончику хвоста (`ctx.roundRect`).
   * Сброс эффектов теней (`shadowBlur = 0`) после отрисовки светящихся элементов.

---

## 9. Чек-лист верификации для Antigravity CLI

При реализации файла `index.html` агент должен проверить соответствие следующим критериям:

- [ ] **Zero Asset Check:** В файле нет ни одного тега `<img src="...">`, `<link href="http...">`, `import` или `fetch`.
- [ ] **No Page Scroll:** При свайпах внутри холста и многократных тапах по D-Pad страница не скроллится и не зумится ни на iOS Safari, ни на Android Chrome.
- [ ] **Direction Turn Lock:** Быстрое нажатие клавиш (например, `Right -> Down` при движении `Up`) не приводит к преждевременной коллизии хвоста с головой.
- [ ] **Persistence:** Значение рекорда успешно читается из `localStorage` при старте и сохраняется при его превышении.
- [ ] **No Memory Leak:** При сворачивании вкладки браузера `requestAnimationFrame` корректно обрабатывает таймстемпы и не совершает сотен моментальных шагов при возвращении.
- [ ] **Valid Layout:** На экранах от 320px до 2560px интерфейс полностью помещается в область видимости без появления полос прокрутки.