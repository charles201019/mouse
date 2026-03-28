# Mouse Game - Implementation Plan

This document breaks the design (see `MOUSE_GAME_DESIGN.md`) into concrete implementation steps, ordered by dependency. Each step produces a testable result. The entire game lives in a single `index.html` file.

---

## File Skeleton

Before any phase begins, create `index.html` with the foundational structure:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>MOUSE</title>
  <style>/* CSS here */</style>
</head>
<body>
  <canvas id="gameCanvas"></canvas>
  <script>/* All JS here */</script>
</body>
</html>
```

### CSS Baseline
- `body`: margin 0, overflow hidden, background `#1a1c2c`, display flex, justify-content center, align-items center, height 100vh
- `canvas`: display block

### JS Module Structure (all inside one `<script>` block)
Organize code into clearly commented sections, in this order:
1. **Constants** — colors, sizes, key codes, audio params
2. **State** — game state object, initial values
3. **Utility functions** — distance, wrap, random cell
4. **Grid & sizing** — compute grid, handle resize
5. **Input** — keyboard listeners, buffer management
6. **Game logic** — tick, move, collision, food, obstacles
7. **Sprites** — drawing functions for each entity
8. **Particles** — particle system
9. **Audio** — AudioContext, SFX, music
10. **Rendering** — draw frame, HUD, overlays
11. **Screens** — start, pause, game-over, win
12. **Init** — entry point, event bindings, first render

---

## Phase 1: Core Game

### Step 1.1 — Constants & Game State Object

Define all constants and the central state object.

**Constants:**
```js
const COLORS = {
  bg: '#1a1c2c', grid: '#262b44', mouseHead: '#9e9e9e',
  cheese: '#f4b41a', obstacle: '#5d275d', text: '#f0f0f0', gold: '#f4b41a'
};
const HUD_HEIGHT = 40;
const PREFERRED_CELL_SIZE = 20;
const MIN_CELL_SIZE = 10;
const MIN_GRID = 15;
const SIMPLIFIED_SPRITE_THRESHOLD = 16;
const BASE_TICK = 150;
const MIN_TICK = 60;
const TICK_DECREASE = 10;
const CHEESE_PER_SPEED = 3;
const CHEESE_PER_OBSTACLE = 5;
const MAX_OBSTACLE_RATIO = 0.05;
const INPUT_BUFFER_CAP = 3;
const CHEESE_SPAWN_DISTANCE = 2;
const OBSTACLE_SPAWN_DISTANCE = 3;
const OBSTACLE_FORWARD_PATH = 3;
const OBSTACLE_PLACEMENT_RETRIES = 10;
const BFS_MIN_EMPTY = 10;
const BFS_MIN_EMPTY_RATIO = 0.10;
const DIRECTIONS = { UP: {x:0,y:-1}, DOWN: {x:0,y:1}, LEFT: {x:-1,y:0}, RIGHT: {x:1,y:0} };
const OPPOSITES = { UP:'DOWN', DOWN:'UP', LEFT:'RIGHT', RIGHT:'LEFT' };
```

**Game state object:**
```js
let state = {
  screen: 'start',        // 'start' | 'playing' | 'paused' | 'gameover' | 'win' | 'tooSmall'
  gridWidth: 0,
  gridHeight: 0,
  cellSize: 0,
  lockedCanvasWidth: 0,   // pixel width of canvas, locked when game starts
  lockedCanvasHeight: 0,  // pixel height of canvas, locked when game starts
  body: [],                // array of {x, y}, head is index 0
  effectiveDirection: 'RIGHT',
  inputBuffer: [],
  cheese: null,            // {x, y}
  obstacles: [],           // array of {x, y}
  score: 0,
  cheeseEaten: 0,
  speedLevel: 1,
  tickInterval: BASE_TICK,
  pendingObstacles: 0,
  highScore: 0,
  musicEnabled: true,
  pauseMuted: false,
  userPaused: false,
  resizeBlocked: false,
  resizeMuted: false,       // audio suppression during resize-block
  growing: false,
  tickTimer: null,
  particles: [],
  audioCtx: null,
  audioInitialized: false,
  musicPlaying: false       // whether background music is currently playing
};
```

**Test:** Page loads, no errors in console.

---

### Step 1.2 — Grid Computation & Canvas Setup

**Implement `computeGrid()`:**
1. Get `window.innerWidth` and `window.innerHeight`
2. `availableWidth = innerWidth`, `availableHeight = innerHeight - HUD_HEIGHT`
3. Try `cellSize = PREFERRED_CELL_SIZE`
4. Compute `cols = floor(availableWidth / cellSize)`, `rows = floor(availableHeight / cellSize)`
5. If `cols < MIN_GRID || rows < MIN_GRID`:
   - `cellSize = floor(min(availableWidth, availableHeight) / MIN_GRID)`
   - Clamp `cellSize` to `MIN_CELL_SIZE` minimum
   - If still can't fit 15x15 → set `state.screen = 'tooSmall'`, return
6. Recompute `cols` and `rows` with final `cellSize`
7. Set `state.gridWidth`, `state.gridHeight`, `state.cellSize`

**Implement `setupCanvas()`:**
1. Set `canvas.width = cols * cellSize`, `canvas.height = rows * cellSize + HUD_HEIGHT`
2. Center canvas via CSS

**Implement `lockGrid()`:**
- Called once when the game transitions from `start` → `playing`
- `state.lockedCanvasWidth = state.gridWidth * state.cellSize`
- `state.lockedCanvasHeight = state.gridHeight * state.cellSize + HUD_HEIGHT`
- From this point, `gridWidth`, `gridHeight`, and `cellSize` are immutable until reset

**Implement `window.onresize` handler:**
- If `state.screen === 'start'` or `state.screen === 'tooSmall'`:
  - Recompute grid via `computeGrid()` and re-render
  - If grid now fits → set `state.screen = 'start'` (recover from tooSmall)
  - If still too small → set `state.screen = 'tooSmall'`
- If game is active (`playing`, `paused`, `gameover`, `win`):
  - Do **not** call `computeGrid()` — grid dimensions are locked
  - Compare `window.innerWidth < state.lockedCanvasWidth` or `window.innerHeight < state.lockedCanvasHeight`
  - If viewport is too small:
    - `state.resizeBlocked = true`
    - `state.inputBuffer = []` (flush)
    - `clearTimeout(state.tickTimer)` (halt game loop)
    - Set `state.resizeMuted = true`, stop music (do **not** modify `pauseMuted` — resize uses its own independent flag)
    - If viewport < 150x190 → show "Screen too small" text variant
    - Else → show "Window too small — resize to continue"
  - If viewport has recovered and `resizeBlocked` was true:
    - `state.resizeBlocked = false`
    - Set `state.resizeMuted = false`; resume music only if `musicEnabled && !pauseMuted`
    - If `state.userPaused`:
      - Set `state.screen = 'paused'` and render the normal pause overlay (regardless of what screen was active when resize-block started). This covers the case where the player pressed P during resize-block while on the playing screen
    - Else if `state.screen === 'playing'` → restart tick loop
    - Else → remain on current screen (gameover or win)
- Always re-center canvas via CSS (do not resize it)

**Test:** Resize browser window. Canvas resizes on start screen, stays locked during game. "Screen too small" appears on tiny viewports. Recovering from tooSmall on start screen works. Mid-game resize-block activates/deactivates correctly. Grid never changes during play.

---

### Step 1.3 — Mouse Initialization & Rendering

**Implement `initMouse()`:**
1. `headX = floor(gridWidth / 2)`, `headY = floor(gridHeight / 2)`
2. `body = [{x: headX, y: headY}, {x: headX-1, y: headY}, {x: headX-2, y: headY}]`
3. `effectiveDirection = 'RIGHT'`

**Implement `drawCell(x, y, color)` utility:**
- Fill a rectangle at `(x * cellSize, y * cellSize + HUD_HEIGHT, cellSize, cellSize)`

**Implement `drawMouse()`:**
- Draw head at `body[0]` with `COLORS.mouseHead`
- Draw body segments with slightly lighter grey, slight color variation per index
- For now, use simple filled rectangles (pixel art sprites come in Phase 3)

**Implement `drawBoard()`:**
1. Fill entire canvas with `COLORS.bg`
2. Optionally draw grid lines with `COLORS.grid`
3. Call `drawMouse()`

**Test:** Game loads showing the mouse (3 segments) at center of grid.

---

### Step 1.4 — Input Handling & Movement

**Implement keyboard listener:**
```js
document.addEventListener('keydown', (e) => { ... });
```
- Map arrow keys and WASD to direction names (`UP`, `DOWN`, `LEFT`, `RIGHT`)
- If `state.screen === 'playing'` and key is a direction:
  - If `inputBuffer.length < INPUT_BUFFER_CAP` → push direction
- Handle SPACE, P, M keys based on current screen:
  - **SPACE**:
    - `start` → call `initAudio()`, `lockGrid()`, `initMouse()`, `spawnCheese()`, set `screen = 'playing'`, start tick loop, start music if `musicEnabled`, play menu SFX (best-effort)
    - `gameover` / `win` → reset state, same as start
    - Other screens → ignored
  - **P**:
    - `playing` → set `userPaused = true`, `pauseMuted = true`, flush input buffer, stop music, clear tick timer, set `screen = 'paused'`
    - `paused` → set `userPaused = false`, `pauseMuted = false`, resume music if `musicEnabled`, restart tick loop, set `screen = 'playing'`
    - `resizeBlocked` → toggle `userPaused` (and `pauseMuted` accordingly). Game stays halted because `resizeBlocked` is still true. When resize recovers, the game will check `userPaused` to decide whether to resume or show the pause screen
    - Other screens → ignored
  - **M** (works on ALL screens except `tooSmall` and `resizeBlocked`):
    - Call `initAudio()` (this may be the first user gesture)
    - Toggle `state.musicEnabled`
    - Play menu SFX (best-effort)
    - If currently playing: start/stop music immediately based on new `musicEnabled` value
    - If on start screen: only toggle `musicEnabled` (music does not play on the start screen — it begins on first SPACE). The HUD indicator updates to reflect the new state
    - If paused: only toggle `musicEnabled` (music stays silent due to `pauseMuted`, will apply on unpause)
    - If game-over / win: start/stop music immediately
    - Re-render to update HUD music indicator
- Also add a `click` listener on `document` that calls `initAudio()` (ensures AudioContext initializes on click as well as keypress, per design)

**Implement `processInput()`:**
1. While `inputBuffer` is not empty:
   - Dequeue first element
   - If it is the opposite of `effectiveDirection` → discard, continue loop
   - Else → set as new direction, break
2. Set `effectiveDirection` to resolved direction

**Implement `move()`:**
1. Get current head position
2. Add direction vector to head
3. Wrap: `newX = ((newX % gridWidth) + gridWidth) % gridWidth` (same for Y)
4. Return new head position `{x, y}`

**Implement basic game tick:**
```js
function tick() {
  processInput();
  const newHead = move();
  // (collision, food check come later — for now just move)
  state.body.unshift(newHead);
  state.body.pop();
  render();
  state.tickTimer = setTimeout(tick, state.tickInterval);
}
```

**Test:** Press arrow keys / WASD. Mouse moves around, wraps at edges. Reversals are ignored. Rapid key presses are buffered correctly.

---

### Step 1.5 — Cheese Spawning & Eating

**Implement `chebyshevDistance(a, b)` (wrap-aware):**
```js
function chebyshevDistance(a, b) {
  const dx = Math.min(Math.abs(a.x - b.x), state.gridWidth - Math.abs(a.x - b.x));
  const dy = Math.min(Math.abs(a.y - b.y), state.gridHeight - Math.abs(a.y - b.y));
  return Math.max(dx, dy);
}
```

**Implement `getEmptyCells(excludeSet)`:**
- Iterate all grid cells
- Exclude cells occupied by body, obstacles, and optionally the cheese
- Return array of `{x, y}`

**Implement `spawnCheese()`:**
1. Get empty cells
2. Filter to cells with `chebyshevDistance(cell, head) >= CHEESE_SPAWN_DISTANCE`
3. If no valid cells → relax constraint, use any empty cell
4. If no empty cells at all → return `null` (triggers win)
5. Pick random cell from valid set
6. Set `state.cheese = cell`

**Implement `drawCheese()`:**
- Draw at cheese position with `COLORS.cheese`

**Update `tick()` — add food check:**
- After computing `newHead`, check if `newHead` matches `state.cheese`
- If yes → set `state.growing = true`
- After body update: if `growing` was true → don't pop tail, call post-eat logic

**Post-eat logic (step 7 from design):**
1. `state.cheeseEaten++`
2. `state.speedLevel = floor(cheeseEaten / CHEESE_PER_SPEED) + 1`
3. `state.score += 10 * state.speedLevel`
4. `state.tickInterval = max(MIN_TICK, BASE_TICK - floor(cheeseEaten / CHEESE_PER_SPEED) * TICK_DECREASE)`
5. Call `spawnCheese()` — if returns null → trigger win
6. (Obstacle resolution added in Phase 2)
7. Reset `state.growing = false`

**Test:** Cheese appears. Mouse eats it, grows by 1 segment, score increases, new cheese appears elsewhere. Cheese never spawns adjacent to head.

---

### Step 1.6 — Collision Detection & Game Over

**Update `tick()` — full loop with collision:**

Implement the precise 9-step order:
1. `processInput()`
2. `const newHead = move()`
3. `const growing = (newHead.x === cheese.x && newHead.y === cheese.y)`
4. If `!growing` → `body.pop()` (tail removal)
5. Check collision: `newHead` against `body` and `obstacles`
   - `body.some(seg => seg.x === newHead.x && seg.y === newHead.y)`
   - `obstacles.some(obs => obs.x === newHead.x && obs.y === newHead.y)`
   - If collision → call `gameOver()`, return
6. `body.unshift(newHead)`
7. If `growing` → post-eat effects
8. `render()`
9. `state.tickTimer = setTimeout(tick, state.tickInterval)`

**Implement `gameOver()`:**
1. `clearTimeout(state.tickTimer)`
2. `state.screen = 'gameover'`
3. Check/update high score in localStorage
4. Render game-over overlay (basic version — polished in Phase 3)

**Test:** Mouse dies when hitting itself. Game-over screen shows. Score and high score display.

---

### Step 1.7 — Wall Wrapping Verification

Wall wrapping was implemented in step 1.4. This step is verification:

**Test cases:**
- Mouse exits right edge → enters left
- Mouse exits left edge → enters right
- Mouse exits top → enters bottom
- Mouse exits bottom → enters top
- Cheese spawns near edge, wrap-aware distance is correct

---

## Phase 2: Progression

### Step 2.1 — Score System & HUD

**Implement `drawHUD()`:**
1. Draw HUD bar at top (40px height, dark background)
2. Line 1: `Score: {score}` left-aligned, `Spd: {speedLevel}` right-aligned
3. Line 2: `High: {highScore}` left-aligned, `[M]` indicator right-aligned
4. Use pixel-art styled icons (simple canvas-drawn icons for cheese, bolt, trophy, note)
5. Font: monospace, 16px, `COLORS.text`; score value in `COLORS.gold`

**Implement `loadHighScore()` / `saveHighScore()`:**
```js
function loadHighScore() {
  return parseInt(localStorage.getItem('mouseGameHighScore')) || 0;
}
function saveHighScore(score) {
  localStorage.setItem('mouseGameHighScore', score);
}
```

**Update game-over logic:**
- If `state.score > state.highScore` → update and save

**Test:** HUD shows score, speed, high score. Score increases on eating. High score persists after page reload.

---

### Step 2.2 — Speed Increase

Already implemented in step 1.5 post-eat logic. This step is verification:

**Test cases:**
- Eat 3 cheese → speed level 2, tick interval 140ms
- Eat 6 cheese → speed level 3, tick interval 130ms
- Eat 27 cheese → speed level 10, tick interval 60ms (capped)
- Eat 30 cheese → speed level 11, tick interval still 60ms, but score per cheese = 110
- HUD displays correct speed level throughout

---

### Step 2.3 — Progressive Obstacles

**Implement `getForwardPath(head, direction, count)`:**
- Return array of next `count` cells in the given direction (wrap-aware)

**Implement `bfsReachability(startCell, obstacles, body)`:**
1. BFS/flood fill from `startCell`
2. Walls: obstacle cells and all body cells (including tail) are impassable
3. Board wraps (toroidal BFS — neighbors wrap around edges)
4. Return `{ reachable: Set, cheeseReachable: boolean, emptyCount: number }`
   - `cheeseReachable`: whether cheese position is in the reachable set
   - `emptyCount`: count of reachable cells excluding cheese, body, and obstacles

**Implement `tryPlaceObstacle()`:**
1. Check if current obstacle count >= `floor(gridWidth * gridHeight * MAX_OBSTACLE_RATIO)` → return false (at cap)
2. Loop up to `OBSTACLE_PLACEMENT_RETRIES`:
   a. Pick random empty cell
   b. Check `chebyshevDistance(cell, head) >= OBSTACLE_SPAWN_DISTANCE` → skip if not
   c. Check cell is not in `getForwardPath(head, effectiveDirection, OBSTACLE_FORWARD_PATH)` → skip if in path
   d. Tentatively add cell to obstacles
   e. Run `bfsReachability(head, obstacles, body)`
   f. Check: cheese reachable AND emptyCount >= `min(BFS_MIN_EMPTY, floor(totalEmpty * BFS_MIN_EMPTY_RATIO))`
   g. If both pass → keep obstacle, return true
   h. If fail → remove tentative obstacle, try next cell
3. All retries failed → return false

**Implement obstacle resolution in post-eat (step 7.4):**
1. If `cheeseEaten % CHEESE_PER_OBSTACLE === 0` → `pendingObstacles++`
2. If `pendingObstacles > 0`:
   - Check if at obstacle cap → discard all pending (`pendingObstacles = 0`), return
   - Call `tryPlaceObstacle()`
   - If success → `pendingObstacles--`, play obstacle SFX (stub for now)

**Implement `drawObstacles()`:**
- Draw each obstacle at its position with `COLORS.obstacle`

**Update collision detection** to include obstacles (already done in step 1.6).

**Test:** Obstacle appears after 5th cheese. Obstacle never spawns next to mouse head or in forward path. BFS ensures cheese is reachable after placement. Deferred obstacles resolve one per cheese. Obstacles cap at 5%.

---

### Step 2.4 — High Score & Win Path Completeness

Already implemented in step 2.1. This step adds win-path parity:

**Update win condition:**
- On win, check/update high score (same logic as game-over)
- Set `state.screen = 'win'`

**Implement basic win screen (placeholder until Phase 3 polish):**
- Draw overlay with "YOU WIN!" text
- Show final score and high score
- If new high score → show "NEW HIGH SCORE!" text
- Show hint: "SPACE to restart · M to toggle music"
- This ensures the win path is fully functional after Phase 2, matching the design's requirements. Phase 3 will add pixel art, enhanced celebrations, and visual polish

**Implement basic game-over screen (placeholder until Phase 3 polish):**
- Draw overlay with "GAME OVER" text
- Show final score and high score
- If new high score → show "NEW HIGH SCORE!" text
- Show hint: "SPACE to restart · M to toggle music"

**Test:** High score updates on game-over and win. Persists across reloads. Displayed on HUD, game-over, and win screens. Win screen shows all required info. Game-over screen shows all required info. Both screens are fully playable (SPACE restarts, M toggles music).

---

## Phase 3: Polish

### Step 3.1 — Pixel Art Sprites

Replace simple rectangles with canvas-drawn pixel art.

**Implement `drawMouseHead(x, y, direction)`:**
- Draw at `(x * cellSize, y * cellSize + HUD_HEIGHT)`
- Full detail (cellSize >= 16): grey circle body, two round ears, two dot eyes, whisker lines, orient toward direction
- Simplified (cellSize < 16): solid grey circle with two ear dots

**Implement `drawMouseBody(x, y, index)`:**
- Rounded rectangle with color gradient (lighter grey, slight variation by `index`)
- Simplified: plain filled rectangle

**Implement `drawCheeseSprite(x, y)`:**
- Full detail: yellow square with 2-3 darker circle "holes"
- Simplified: solid yellow square

**Implement `drawObstacleSprite(x, y)`:**
- Full detail: brick pattern (darker lines forming brick grid)
- Simplified: solid colored square

**Implement HUD icons (small canvas-drawn pixel art):**
- Cheese icon (8x8), bolt icon, trophy icon, music note icon

**Test:** All sprites render correctly at 20px and 10px cell sizes. Simplified variants kick in below 16px.

---

### Step 3.2 — Particle Effects

**Implement particle system:**
```js
// Particle: { x, y, vx, vy, life, maxLife, color, size }
function spawnParticles(x, y, count, color) { ... }
function updateParticles(dt) { ... }  // move, decrease life, remove dead
function drawParticles() { ... }
```

**Eat particles:** 5-8 small yellow squares burst outward from cheese position.

**Death particles:** 10-15 grey squares explode from collision point.

**Win celebration:** 20-30 multi-colored particles fountain from center.

**Integrate into render loop:**
- Update and draw particles every frame in `requestAnimationFrame` loop (separate from game tick)

**Implement `requestAnimationFrame` render loop:**
```js
function renderLoop() {
  render();           // draws current state + particles
  updateParticles();
  requestAnimationFrame(renderLoop);
}
```
- Game logic stays on `setTimeout` tick
- Rendering runs independently on `requestAnimationFrame`

**Test:** Eating cheese produces yellow burst. Dying produces grey explosion. Winning produces colorful fountain.

---

### Step 3.3 — Screen Overlays

All overlays are semi-transparent dark rectangles drawn on the canvas with centered text.

**Implement `drawOverlay(alpha)`:**
- Fill canvas with `rgba(0, 0, 0, alpha)` (alpha ~0.7)

**Implement `drawStartScreen()`:**
1. Render empty board behind
2. Draw overlay
3. Title: "MOUSE" — large pixel font (32px), centered
4. Controls list: "Arrow Keys / WASD to move", "P to pause", "M to toggle music"
5. "Press SPACE to Start" — blinking text (toggle visibility every 500ms)

**Implement `drawPauseScreen()`:**
1. Draw overlay over current game state (frozen frame)
2. "PAUSED" — large centered text
3. Hint: "P to resume · M to toggle music"

**Implement `drawGameOverScreen()`:**
1. Draw overlay over final frame (collision highlighted)
2. "GAME OVER" — large centered text
3. `Score: {score}` and `High: {highScore}`
4. If new high score → "NEW HIGH SCORE!" in gold
5. Hint: "SPACE to restart · M to toggle music"

**Implement `drawWinScreen()`:**
1. Draw overlay with celebration particles
2. "YOU WIN!" — large centered text in gold
3. `Score: {score}` and `High: {highScore}`
4. If new high score → "NEW HIGH SCORE!" with enhanced particles
5. Hint: "SPACE to restart · M to toggle music"

**Implement `drawResizeBlockScreen()`:**
1. Draw overlay
2. "Window too small" or "Screen too small" depending on severity
3. "Resize to continue"

**Test:** All screens render correctly. Transitions work (SPACE to start, P to pause/resume, game-over/win on appropriate conditions).

---

### Step 3.4 — Resize-Block Overlay

The core resize-block logic is already implemented in the `window.onresize` handler from Step 1.2. This step adds the visual overlay and audio integration.

**Resize-block audio behavior:**
- On entering resize-block: suppress all audio the same way pause does — stop music, prevent SFX. Internally, set a `resizeMuted` flag (analogous to `pauseMuted`). The `playSFX()` function should check: `if (state.pauseMuted || state.resizeMuted) return`
- On exiting resize-block: clear `resizeMuted`. Music resumes only if `musicEnabled` is true AND `pauseMuted` is false (i.e., the player wasn't also manually paused). SFX resume based on the same conditions
- `resizeMuted` is independent of `pauseMuted` — both can be true simultaneously if the player paused and then resized

**Add `resizeMuted` to state object** (in Step 1.1):
```js
resizeMuted: false,       // audio suppression during resize-block
```

**"Screen too small" variant:**
- If viewport < 150x190 → show "Screen too small" text instead of "Window too small"
- Uses same `resizeBlocked` flag, same `resizeMuted` audio behavior

**Test:** Shrink window during game → game pauses with overlay, audio stops. Expand → game resumes (unless manually paused), audio resumes correctly. Pause then resize → both states preserved, audio stays silent. Unpause after resize recovery → audio resumes. Extreme shrink shows "Screen too small."

---

### Step 3.5 — HUD Polish

**Finalize HUD layout:**
```
[cheese-icon] Score: 120  [bolt-icon] Spd: 3
[trophy-icon] High: 450   [note-icon] [M]
```

- All icons are pixel-art drawn via canvas (from step 3.1)
- Score value in gold, other text in white
- Music indicator: [note-icon] filled when `musicEnabled`, outlined when off
- Monospace font, 16px
- 4px padding top/bottom

**Test:** HUD renders cleanly at all grid sizes. Icons are clear. Music state indicator updates on M press.

---

## Phase 4: Audio

### Step 4.1 — AudioContext & Initialization

**Implement `initAudio()`:**
```js
function initAudio() {
  if (state.audioInitialized) return;
  state.audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  state.audioInitialized = true;
}
```

**Hook into first user gesture:**
- In the keydown/click handler, call `initAudio()` on any interaction
- If `audioCtx.state === 'suspended'` → call `audioCtx.resume()`

**Test:** AudioContext initializes on first user gesture (keypress or click). No console warnings about autoplay. Clicking the canvas before any keypress also initializes audio.

---

### Step 4.2 — Sound Effects

**Implement `playSFX(type)`:**

```js
function playSFX(type) {
  if (!state.audioCtx || state.pauseMuted || state.resizeMuted) return;
  const ctx = state.audioCtx;
  const osc = ctx.createOscillator();
  const gain = ctx.createGain();
  osc.connect(gain);
  gain.connect(ctx.destination);

  switch(type) {
    case 'eat':
      osc.type = 'sine';
      osc.frequency.setValueAtTime(200, ctx.currentTime);
      osc.frequency.linearRampToValueAtTime(600, ctx.currentTime + 0.1);
      gain.gain.setValueAtTime(0.3, ctx.currentTime);
      gain.gain.linearRampToValueAtTime(0, ctx.currentTime + 0.1);
      osc.start(); osc.stop(ctx.currentTime + 0.1);
      break;
    case 'gameover':
      osc.type = 'square';
      osc.frequency.setValueAtTime(400, ctx.currentTime);
      osc.frequency.linearRampToValueAtTime(100, ctx.currentTime + 0.3);
      gain.gain.setValueAtTime(0.3, ctx.currentTime);
      gain.gain.linearRampToValueAtTime(0, ctx.currentTime + 0.3);
      osc.start(); osc.stop(ctx.currentTime + 0.3);
      break;
    case 'obstacle':
      osc.type = 'triangle';
      osc.frequency.setValueAtTime(80, ctx.currentTime);
      gain.gain.setValueAtTime(0.4, ctx.currentTime);
      gain.gain.linearRampToValueAtTime(0, ctx.currentTime + 0.15);
      osc.start(); osc.stop(ctx.currentTime + 0.15);
      break;
    case 'menu':
      osc.type = 'sine';
      osc.frequency.setValueAtTime(800, ctx.currentTime);
      gain.gain.setValueAtTime(0.2, ctx.currentTime);
      gain.gain.linearRampToValueAtTime(0, ctx.currentTime + 0.05);
      osc.start(); osc.stop(ctx.currentTime + 0.05);
      break;
  }
}
```

**Integrate SFX calls:**
- Eat cheese → `playSFX('eat')` in post-eat effects
- Game over → `playSFX('gameover')` in `gameOver()`
- Obstacle placed → `playSFX('obstacle')` in `tryPlaceObstacle()`
- SPACE / M pressed → `playSFX('menu')` (best-effort on first gesture)

**Test:** Each sound plays at the correct event. No sound during pause. Sounds resume after unpause.

---

### Step 4.3 — Background Music

**Implement `startMusic()` / `stopMusic()`:**

Create a simple 4-bar chiptune loop:
```js
function startMusic() {
  if (!state.audioCtx || !state.musicEnabled || state.musicPlaying) return;
  // Create a looping pattern using square/triangle oscillators
  // Schedule 4 bars of notes at 120 BPM (beat = 500ms)
  // Use a ScriptProcessorNode or schedule ahead with setTimeout
  state.musicPlaying = true;
}

function stopMusic() {
  // Stop all music oscillators
  state.musicPlaying = false;
}
```

**Music melody** (simple 4-bar loop, 120 BPM):
- Use a pattern of note frequencies (e.g., C4, E4, G4, C5 arpeggios)
- Square wave for melody, triangle wave for bass
- Loop by rescheduling when pattern ends

**Integrate music state:**
- Start music on SPACE (start game) if `musicEnabled`
- On pause → `stopMusic()` (via `pauseMuted`)
- On unpause → `startMusic()` if `musicEnabled`
- On M press → toggle `musicEnabled`, start/stop immediately
- On game-over/win → music continues (M still toggles)

**Test:** Music plays during gameplay. Stops on pause. M toggles on all screens. Music state persists across restarts.

---

### Step 4.4 — Music Toggle (M Key)

**Update keydown handler for M key on all screens:**

```js
case 'KeyM':
  initAudio();  // ensure AudioContext exists
  playSFX('menu');
  state.musicEnabled = !state.musicEnabled;
  // Only start/stop music if not on start screen (music begins on first SPACE)
  if (state.screen !== 'start') {
    if (state.musicEnabled && !state.pauseMuted && !state.resizeMuted) {
      startMusic();
    } else {
      stopMusic();
    }
  }
  render();  // update HUD music indicator
  break;
```

**Screen-specific behavior:**
- Start screen: toggle `musicEnabled` only (no music playback yet — music begins on first SPACE)
- Playing: toggle works, immediate effect
- Paused: toggle `musicEnabled` only (music stays silent due to `pauseMuted`, but will resume correctly on unpause)
- Game over / Win: toggle works, immediate start/stop of music

**Test:** M key works on every screen. Music indicator in HUD updates. State persists across game restarts.

---

## Phase 5: Integration & Final Testing

### Step 5.1 — Full Game Loop Verification

Run through the complete 9-step tick loop and verify each step:

| Step | Check |
|------|-------|
| 1. Input Processing | Buffer consumes until legal direction found or empty; `effectiveDirection` set |
| 2. Move | New head wraps correctly at all 4 edges |
| 3. Food Check | `growing` flag set when head matches cheese, body not modified |
| 4. Tail Removal | Tail removed only when not growing; removed before collision check |
| 5. Collision | Checks post-tail-removal body + obstacles; moving into vacated tail cell is legal |
| 6. Apply Head | New head prepended to body array |
| 7. Post-eat | Score → speed → cheese spawn → obstacle resolve (in order); win check at 7.3; 7.4 skipped on win |
| 8. Render | Frame drawn via requestAnimationFrame |
| 9. Schedule | Next tick via setTimeout at current interval |

---

### Step 5.2 — State Transition Matrix

Verify every key works correctly on every screen:

| Key   | Start  | tooSmall | Playing | Paused | Game Over | Win | Resize-Blocked |
|-------|--------|----------|---------|--------|-----------|-----|----------------|
| Arrows/WASD | ignored | ignored | buffer direction | ignored (not buffered) | ignored | ignored | ignored (not buffered) |
| SPACE | lockGrid, start game + menu SFX | ignored | ignored | ignored | restart + menu SFX | restart + menu SFX | ignored |
| P     | ignored | ignored | set userPaused, flush buffer, pauseMuted, stop music | clear userPaused, clear pauseMuted, resume music if musicEnabled | ignored | ignored | toggle userPaused (preserves independently of resizeBlocked; game stays halted until both are false) |
| M     | initAudio, toggle musicEnabled (no music playback yet — starts on SPACE) + menu SFX | ignored | toggle + immediate start/stop + menu SFX | toggle musicEnabled only (silent, pauseMuted active) + menu SFX | toggle + immediate start/stop + menu SFX | toggle + immediate start/stop + menu SFX | ignored |
| click | initAudio | initAudio | initAudio | initAudio | initAudio | initAudio | initAudio |

**Audio suppression flags:**
| State | `pauseMuted` | `resizeMuted` | Music plays? | SFX play? |
|-------|-------------|--------------|-------------|----------|
| Playing | false | false | if `musicEnabled` | yes |
| Paused | true | false | no | no |
| Resize-blocked | false | true | no | no |
| Paused + resize-blocked | true | true | no | no |
| Game over / Win | false | false | if `musicEnabled` | yes |

---

### Step 5.3 — Edge Case Testing

- [ ] Eat cheese at grid edge (wrap-aware spawn)
- [ ] Mouse wraps while growing (tail not removed on wrap tick)
- [ ] Obstacle spawns when no valid cell exists (defers correctly)
- [ ] Multiple deferred obstacles resolve one per cheese
- [ ] Obstacle cap reached — pending obstacles discarded
- [ ] Pause during resize-block — both `userPaused` and `resizeBlocked` preserved
- [ ] Resume from resize-block while paused — stays paused, `pauseMuted` still active
- [ ] Resize-block audio — music and SFX suppressed via `resizeMuted`
- [ ] Resize-block recovery — audio resumes only if neither `pauseMuted` nor `resizeMuted`
- [ ] tooSmall on start screen — resize to fit recovers to start screen
- [ ] Grid dimensions locked during play — resize never changes gridWidth/gridHeight/cellSize
- [ ] High score updates on both game-over and win
- [ ] Music state preserved across restart
- [ ] First gesture initializes AudioContext (any keypress — M, SPACE, etc. — or click)
- [ ] Rapid key input — buffer caps at 3, reversals filtered at dequeue

---

### Step 5.4 — Performance Check

- [ ] No memory leaks in particle system (dead particles removed)
- [ ] No orphaned setTimeout timers on state transitions
- [ ] requestAnimationFrame loop doesn't duplicate on restart
- [ ] BFS doesn't block the main thread on large grids (900+ cells)
- [ ] AudioContext oscillators are properly stopped and garbage collected

---

## Implementation Order Summary

| Order | Step | Description | Depends On |
|-------|------|-------------|------------|
| 1 | 1.1 | Constants & state object | — |
| 2 | 1.2 | Grid computation & canvas | 1.1 |
| 3 | 1.3 | Mouse init & basic rendering | 1.2 |
| 4 | 1.4 | Input handling & movement | 1.3 |
| 5 | 1.5 | Cheese spawning & eating | 1.4 |
| 6 | 1.6 | Collision detection & game over | 1.5 |
| 7 | 1.7 | Wall wrapping verification | 1.4 |
| 8 | 2.1 | Score system & HUD | 1.6 |
| 9 | 2.2 | Speed increase verification | 1.5 |
| 10 | 2.3 | Progressive obstacles & BFS | 2.1 |
| 11 | 2.4 | High score localStorage | 2.1 |
| 12 | 3.1 | Pixel art sprites | 1.3 |
| 13 | 3.2 | Particle effects & rAF loop | 3.1 |
| 14 | 3.3 | Screen overlays | 3.1 |
| 15 | 3.4 | Resize-block overlay | 3.3, 1.2 |
| 16 | 3.5 | HUD polish | 3.1, 2.1 |
| 17 | 4.1 | AudioContext init | — |
| 18 | 4.2 | Sound effects | 4.1 |
| 19 | 4.3 | Background music | 4.1 |
| 20 | 4.4 | Music toggle | 4.3 |
| 21 | 5.1 | Full loop verification | All |
| 22 | 5.2 | State transition matrix | All |
| 23 | 5.3 | Edge case testing | All |
| 24 | 5.4 | Performance check | All |
