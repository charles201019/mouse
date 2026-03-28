# Mouse Game - Design Document

## Overview
A single-player browser-based game inspired by the classic Snake, re-themed as a mouse eating cheese. Built as a single-page HTML/CSS/JS application with no external dependencies.

---

## Game Concept
The player controls a mouse that moves continuously across a grid. The goal is to eat cheese to grow longer and score points while avoiding collisions with obstacles and the mouse's own body. The mouse wraps around screen edges seamlessly.

---

## Technical Specification

### Tech Stack
- **HTML5 Canvas** for rendering (pixel art style)
- **Vanilla JavaScript** for game logic
- **CSS** for UI/layout
- **Web Audio API** for programmatic SFX and background music
- **localStorage** for high score persistence
- **Single file**: `index.html` (all-in-one, zero dependencies)

### Grid & Board
- **Responsive grid**: dynamically calculates grid size based on viewport dimensions
- **Preferred cell size**: 20px per cell
- **HUD height**: fixed at **40px** (hard spec — two lines of 16px text with 4px padding top/bottom)
- **Footer**: the instruction bar ("Arrow Keys / WASD to move…") is shown **only on the start screen**. During gameplay it is hidden, so it does not consume vertical space. Relevant control hints are also shown contextually on pause, game-over, and win overlays (see Screens section)
- **Grid dimensions**: computed as `floor(availableWidth / cellSize) x floor(availableHeight / cellSize)`, where `availableWidth = viewportWidth` (full width, no deductions) and `availableHeight = viewportHeight - 40px` (HUD only; footer is not visible during play)
- **Minimum board**: 15x15 cells
- **Small viewport fallback**: if the viewport cannot fit a 15x15 grid at 20px cells (i.e., viewport < 300x340), the cell size is reduced proportionally: `cellSize = floor(min(viewportWidth, viewportHeight - 40) / 15)`, with an absolute minimum cell size of **10px**. If even 10px cells cannot fit 15x15 (viewport < 150x190), the game displays a "Screen too small" message instead of rendering
- **Window resize**: grid size is **locked once a game starts**; resizing mid-game does not change the grid. The canvas is re-centered on resize but the grid dimensions remain fixed until a new game begins. If the viewport shrinks below the locked canvas size mid-game, a **resize-block** state activates: the **input buffer is flushed immediately**, the game loop halts, a "Window too small — resize to continue" overlay is shown, and all inputs (including movement) are ignored and **not buffered**. When the viewport is large enough again, the resize-block is cleared — but the game only resumes if `userPaused` is false. If the player had manually paused (P key) before or during the resize, the game remains paused with the normal pause overlay. The two states are independent: `userPaused` (toggled by P key) and `resizeBlocked` (set/cleared automatically by viewport size). The game loop runs only when **both** are false. The "Screen too small" message (for viewports that cannot fit 15x15 at 10px cells) applies both at startup and on resize. Mid-game, this is a **presentation variant of `resizeBlocked`** — it uses the same state flag, just with different overlay text. `userPaused` is preserved independently and still respected when the viewport recovers

---

## Visual Style: Pixel Art

### Color Palette
| Element         | Color                        |
|-----------------|------------------------------|
| Background      | Dark navy `#1a1c2c`          |
| Grid lines      | Subtle `#262b44` (optional)  |
| Mouse head      | Grey `#9e9e9e` with ears     |
| Mouse body      | Lighter grey gradient        |
| Cheese          | Yellow `#f4b41a` with holes  |
| Obstacles       | Dark red/brown `#5d275d`     |
| UI text         | White `#f0f0f0`              |
| Score highlight  | Gold `#f4b41a`              |

### Sprites (drawn via Canvas)
All sprites are drawn programmatically using canvas pixel drawing — no image files needed.

- **Mouse head**: pixel art with round ears, small eyes, whiskers
- **Mouse body segments**: rounded rectangles with slight color variation
- **Cheese**: yellow square with darker circle "holes"
- **Obstacles**: brick-like pattern blocks
- **Particles**: small squares for eat/death effects
- **Scaling**: sprites are drawn at the current `cellSize`, not at a fixed 20x20. When `cellSize < 16px`, sprites switch to a **simplified variant** (fewer details — e.g., mouse head becomes a solid circle with two ear dots, cheese becomes a solid yellow square without holes). This avoids sub-pixel artifacts and keeps visuals clean on small screens

---

## Gameplay Mechanics

### Initial State
- The mouse head spawns at grid position `(floor(gridWidth / 2), floor(gridHeight / 2))` — on even-sized boards this is one cell below-right of true center
- Initial direction: **right**
- Initial body length: **3 segments**, laid out horizontally to the left of the head (head at center, body extending left: `[head, head-1, head-2]`)

### Movement
- Mouse moves continuously in the current direction at a fixed tick rate
- **Controls**: Arrow keys (↑↓←→) and WASD
- Direction cannot be instantly reversed (e.g., moving right cannot switch to left)
- Input is buffered: if the player presses two keys within one tick, both are queued
- Input buffer capped at **3 inputs** to prevent queue buildup from key mashing
- **Reversal filtering**: at each tick, the game maintains a variable `effectiveDirection` (the direction the mouse actually moved this tick). When dequeuing the next input from the buffer: compare the queued direction against `effectiveDirection`. If it is the exact opposite (e.g., Left vs Right), discard it and dequeue the next input (if any). Otherwise, apply it as the new direction. This means for a buffer like `[Down, Left]` while `effectiveDirection = Right`: Down is dequeued, compared against Right, not opposite → applied. Next tick, Left is dequeued, compared against Down, not opposite → applied. The net result is a valid U-turn over two ticks

### Wall Wrapping
- The mouse wraps around screen edges seamlessly
- Exiting the right side enters from the left, etc.

### Food (Cheese)
- One cheese appears at a random empty cell at a time
- Cheese will not spawn within **2 cells** of the mouse head, measured by **Chebyshev distance** (max of horizontal and vertical distance). Distance is calculated as the **minimum across wrap-around** (i.e., on a toroidal board, `dx = min(|x1-x2|, gridWidth - |x1-x2|)`, same for `dy`). If no valid cell satisfies the distance constraint (late game with a long body), the constraint is relaxed and any empty cell is used
- After the mouse grows from eating cheese, if no cell is available for new cheese placement, the player **wins the game** (see Win Condition below)
- Eating cheese:
  - Adds 1 segment to the mouse body
  - Awards points (10 base × current speed multiplier)
  - Triggers eat SFX
  - Spawns a brief particle effect

### Progressive Obstacles
- Game starts with **0 obstacles**
- A new obstacle is placed every **5 cheese eaten**
- Obstacles are placed at random empty cells with the following safety rules:
  - Must not be within **3 cells** (Chebyshev distance, wrap-aware) of the mouse head
  - Must not be placed on the mouse's **projected forward path** (the next 3 cells in the current movement direction, accounting for wrapping)
  - After placement, a **pathfinding check** (BFS/flood fill from the mouse head) verifies two conditions: **(a)** the current cheese position is still reachable, and **(b)** the reachable region contains at least **10 empty cells** (or 10% of total empty cells, whichever is smaller) to ensure the mouse has room to maneuver. "Empty cells" here means cells not occupied by the mouse body, obstacles, or the cheese — the cheese cell is counted separately via condition (a) and is **excluded** from the empty-cell count. For BFS purposes, the mouse body is treated as **fully occupied** (including the tail). The BFS does not attempt to simulate future tail movement — this is a conservative check that may occasionally reject a valid placement, which is acceptable since the obstacle spawn simply defers to the next cheese eaten rather than blocking gameplay. If either condition fails, the obstacle placement is retried at a different cell (up to 10 attempts). If all attempts fail, the obstacle spawn is deferred — a `pendingObstacles` counter increments. On each subsequent cheese eaten, the game attempts to place **one** pending obstacle (up to 10 placement attempts). If successful, the counter decrements. Only one pending obstacle is resolved per cheese eaten, regardless of how many have accumulated, so the player never faces a sudden burst. Regular threshold spawns (every 5 cheese) still add to the counter normally and are also subject to the one-per-cheese-eaten limit. If `pendingObstacles` is nonzero and the board has already reached the **5% obstacle cap**, all pending obstacles are **discarded** (counter reset to 0) — no further obstacles will spawn until capacity becomes available, which in practice it will not since obstacles are permanent
- Max obstacles: **5% of total grid cells** (e.g., 45 on a 30x30 grid) to scale with board size
- Obstacles are solid — colliding with one ends the game

### Speed Increase
- Base tick interval: **150ms** (≈6.7 moves/sec)
- Every **3 cheese eaten**, tick interval decreases by **10ms**
- Minimum tick interval: **60ms** (≈16.7 moves/sec, cap)
- Reaches max speed after **27 cheese** eaten
- Current speed level is displayed on the HUD

### Collision & Game Over
The game ends when the mouse collides with:
- Its own body
- An obstacle

On game over:
- Freeze the screen briefly
- Play game-over SFX
- Render a final frame with the collision highlighted
- Display final score, high score, and "Press SPACE to restart" prompt
- If new high score, show celebration effect

### Win Condition
- After the mouse grows from eating cheese, if no cell remains available for new cheese placement (all non-obstacle cells are occupied by the mouse body), the player **wins**
- Display a "YOU WIN!" screen with final score, high score, and celebration effect. If the win score is a new high score, the celebration effect is enhanced
- This is extremely rare but provides a definitive end state

---

## Scoring & High Scores

### Scoring Formula
```
speed_level = floor(cheese_eaten / 3) + 1
points_per_cheese = 10 × speed_level
```
- `speed_level` is derived purely from **cheese count**, not from the actual tick interval
- `speed_level` continues to increment past the tick-interval cap (60ms), so scoring rewards keep increasing even after max speed is reached
- Example: at 30 cheese eaten, `speed_level = 11`, each cheese = 110 points, even though tick interval has been capped at 60ms since cheese #27
- The HUD displays `speed_level`, not the raw tick interval

### High Score Storage
- Stored in `localStorage` under key `mouseGameHighScore`
- Top score displayed on HUD, game-over screen, and win screen
- High score persists across browser sessions

---

## Audio

### Sound Effects (Web Audio API - programmatic)
All sounds generated using oscillators and gain nodes — no audio files required.

| Event        | Sound Description                              |
|--------------|-------------------------------------------------|
| Eat cheese   | Short rising chirp (sine wave, 200→600Hz, 100ms) |
| Game over    | Descending buzz (square wave, 400→100Hz, 300ms)  |
| New obstacle | Low thud (triangle wave, 80Hz, 150ms) — plays for both on-threshold and deferred obstacle placements |
| Menu select  | Quick blip (sine wave, 800Hz, 50ms) — plays on SPACE (start/restart) and M (music toggle). Best-effort on the very first user gesture: if this interaction is also initializing the AudioContext, the SFX may be silent, which is acceptable |

### Background Music (Programmatic Chiptune)
- Simple looping chiptune melody generated with Web Audio API
- 4-bar loop using square/triangle wave oscillators
- Tempo: ~120 BPM
- Toggle on/off with **M** key (music only — SFX are always enabled unless paused)
- **Default state**: `musicEnabled` is **true** on first load. Music begins playing when the first game starts (SPACE). The player can toggle it off on the start screen before starting

### Audio Autoplay Policy
- Modern browsers block audio playback until a user interaction event
- The `AudioContext` is created and resumed on the **first user gesture** (any keypress or click — not limited to SPACE)
- All audio calls are guarded to handle suspended AudioContext gracefully

---

## User Interface

### Layout
HUD elements use pixel-art styled icons (not emojis) for visual consistency.

```
┌──────────────────────────────────────┐
│  [cheese] Score: 120  [bolt] Spd: 3 │
│  [trophy] High: 450   [note] [M]    │
├──────────────────────────────────────┤
│                                      │
│                                      │
│           GAME CANVAS                │
│                                      │
│                                      │
└──────────────────────────────────────┘
(Footer with controls info shown on Start Screen only, hidden during gameplay)
```

### Screens
1. **Start Screen**: Rendered as a semi-transparent overlay on top of the game canvas (which shows an empty board at the computed grid size). Title "MOUSE", full controls instructions, "Press SPACE to Start". M key works on the start screen — it toggles `musicEnabled` immediately (this also serves as the first user gesture to initialize the AudioContext). Because the canvas is already rendered behind the overlay, the small-viewport sizing rules apply at startup — the grid is computed and the "Screen too small" message triggers here if needed
2. **Game Screen**: Canvas + HUD overlay (no footer)
3. **Pause Screen**: Semi-transparent overlay with "PAUSED" text and hint line: "P to resume · M to toggle music". On entering pause, the **input buffer is flushed immediately** — any queued turns are discarded. While paused, movement inputs are ignored and **not buffered**. **Audio behavior during pause**: the game maintains two independent states — `musicEnabled` (toggled by M key, persists across pause/unpause, controls **music only**) and `pauseMuted` (automatically applied on pause, removed on unpause, suppresses **all audio** including music and SFX). On pause: all audio is suppressed via `pauseMuted`. On unpause: `pauseMuted` is cleared; music resumes **only if `musicEnabled` is true**; SFX resume unconditionally. The M key works while paused to toggle `musicEnabled`, so the player can pre-set their music preference before resuming. SFX have no independent mute toggle — they play whenever the game is not paused
4. **Game Over Screen**: Final score, high score, hint line: "SPACE to restart · M to toggle music". Pressing M toggles `musicEnabled` **immediately** — if music was playing, it stops; if it was off, it starts. The change persists into the next run
5. **Win Screen**: "YOU WIN!" with final score, high score, celebration effect, and hint line: "SPACE to restart · M to toggle music". M behavior is identical to game-over screen (immediate toggle). If the win score is a new high score, the celebration effect is enhanced

### Keyboard Shortcuts
| Key       | Action          |
|-----------|-----------------|
| ↑↓←→      | Move mouse      |
| WASD      | Move mouse      |
| SPACE     | Start / Restart |
| P         | Pause / Resume  |
| M         | Toggle music    |

---

## File Structure
```
/mnt/data/bob/charles/game/
├── index.html              # Complete game (HTML + CSS + JS, all-in-one)
├── MOUSE_GAME_DESIGN.md    # This design document
```

---

## Game Loop Architecture

```
                 ┌──────────┐
                 │  START    │
                 │  SCREEN   │
                 └────┬─────┘
                      │ SPACE
                 ┌────▼──────┐
            ┌───►│ 1. INPUT   │
            │    │ PROCESSING │
            │    └────┬──────┘
            │         │
            │    ┌────▼──────┐
            │    │ 2. MOVE    │
            │    │ (wrap)     │
            │    └────┬──────┘
            │         │
            │    ┌────▼──────┐
            │    │ 3. FOOD    │
            │    │ CHECK      │
            │    │ (set flag) │
            │    └────┬──────┘
            │         │
            │    ┌────▼──────┐
            │    │ 4. REMOVE  │
            │    │ TAIL (if   │
            │    │ !growing)  │
            │    └────┬──────┘
            │         │
            │    ┌────▼──────┐
            │    │ 5. COLLIDE │──Yes──┐
            │    │ (body/obs) │       │
            │    └────┬──────┘       │
            │         │ No            │
            │    ┌────▼──────┐       │
            │    │ 6. APPLY   │       │
            │    │ HEAD       │       │
            │    └────┬──────┘       │
            │         │               │
            │    ┌────▼──────┐       │
            │    │ 7. POST-   │       │
            │    │ EAT FX     │       │
            │    └────┬──────┘       │
            │         │               │
            │    Board full?          │
            │    ┌─No─┴──Yes─┐       │
            │    │            │       │
            │    ▼            ▼       │
            │ ┌────────┐ ┌────────┐  │
            │ │8. RENDER│ │ RENDER │  │
            │ │ FRAME   │ │  WIN   │  │
            │ └───┬────┘ └───┬────┘  │
            │     │           │       │
            │ ┌───▼─────┐ ┌──▼────┐  │
            │ │9. SCHED. │ │ WIN   │  │
            │ │NEXT TICK │ │SCREEN │  │
            │ └───┬─────┘ └──┬────┘  │
            │     │           │       │
            └─────┘           │  ┌────▼─────┐
                              │  │  RENDER   │
                              │  │ GAME OVER │
                              │  └────┬─────┘
                              │       │
                              │  ┌────▼─────┐
                              │  │ GAME OVER │
                              │  │  SCREEN   │
                              │  └────┬─────┘
                              │       │
                              │  SPACE │ SPACE
                              │  ┌────▼─────┐
                              └─►│  RESET   │
                                 └──────────┘
```

### Core Loop Steps (precise update order)
1. **Input Processing**: Consume buffered inputs until one legal (non-reversal) direction is applied or the buffer is empty. Discarded reversals do not count as the tick's input — the loop keeps dequeuing until a valid direction is found or there is nothing left (in which case the current direction is retained). After resolution, set `effectiveDirection` to the direction that will be used for this tick's move (either the newly applied direction or the retained current direction). All subsequent reversal checks in future ticks compare against this value
2. **Move**: Calculate new head position (with wrapping)
3. **Food Check**: Determine if the new head position is on cheese (sets a `growing` flag, but does not modify the body yet)
4. **Tail Removal**: If `growing` is false, remove the last tail segment from the body array. This happens **before** collision detection, so moving into the cell the tail just vacated is a **legal move**
5. **Collision Detection**: Check new head position against the **post-tail-removal** body array and all obstacles. If collision → game over
6. **Apply Head**: Prepend the new head position to the body array
7. **Post-eat Effects**: If `growing` was true, execute in this order:
   1. Award score, play eat SFX, spawn eat particles
   2. Check for speed increase
   3. Spawn new cheese (or trigger **win** if no cell is available → branch to WIN SCREEN)
   4. Resolve obstacles (skipped if win was triggered at step 7.3): first, add any new threshold spawn to `pendingObstacles` (if `cheese_eaten % 5 == 0`). Then attempt to place **one** pending obstacle (up to 10 attempts with BFS validation against the *newly placed* cheese). If successful, decrement counter and play obstacle SFX. This ordering ensures the BFS always validates reachability to the current cheese, and threshold + deferred spawns are unified in the same counter before resolution
8. **Render**: Clear canvas, draw grid, obstacles, cheese, mouse, particles, HUD (render runs on `requestAnimationFrame` for smooth visuals)
9. **Schedule Next Tick**: Game logic tick scheduled via `setTimeout` at current speed interval

---

## Implementation Milestones

### Phase 1: Core Game
- [ ] Canvas setup with responsive grid
- [ ] Mouse movement and direction control
- [ ] Cheese spawning and eating
- [ ] Body growth
- [ ] Self-collision detection
- [ ] Wall wrapping

### Phase 2: Progression
- [ ] Score system with speed multiplier
- [ ] Speed increase every 3 cheese
- [ ] Progressive obstacles every 5 cheese
- [ ] High score with localStorage

### Phase 3: Polish
- [ ] Pixel art rendering for all sprites
- [ ] Particle effects (eat, death, win celebration)
- [ ] Start, pause, game-over, and win screens
- [ ] Resize-block overlay ("Window too small" / "Screen too small")
- [ ] HUD with score, speed, high score

### Phase 4: Audio
- [ ] Eat SFX
- [ ] Game over SFX
- [ ] Obstacle spawn SFX
- [ ] Background chiptune music loop
- [ ] Music toggle (M key)

---

## Summary

| Attribute        | Value                          |
|------------------|--------------------------------|
| Genre            | Arcade / Snake-like            |
| Theme            | Mouse eating cheese            |
| Tech             | HTML5 Canvas + Vanilla JS      |
| Style            | Pixel art                      |
| Board            | Responsive grid (20px cells)   |
| Walls            | Wrap-around                    |
| Obstacles        | Progressive (every 5 cheese)   |
| Speed            | Increases every 3 cheese       |
| Controls         | Arrow keys + WASD              |
| Audio            | Programmatic SFX + chiptune    |
| High Scores      | localStorage                   |
| Multiplayer      | No (single player)             |
| File Count       | 1 (`index.html`)               |
