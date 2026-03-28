# Mouse Game - Testing Plan

This document defines all manual test cases for the Mouse game, organized by feature area. Each test has a unique ID, clear steps, and expected result. Tests are derived from `MOUSE_GAME_DESIGN.md` and `IMPLEMENTATION_PLAN.md`.

**How to use:** Work through each section after the corresponding implementation phase. Mark tests as PASS/FAIL. Re-run regression tests after each phase.

---

## 1. Grid & Board

### T1.1 — Default grid computation
**Steps:** Open the game in a browser window wider than 300px and taller than 340px.
**Expected:** Canvas renders with 20px cells. Grid fills available space. HUD occupies top 40px.

### T1.2 — Small viewport cell scaling
**Steps:** Resize the browser window to 250x280.
**Expected:** Cell size shrinks below 20px to fit a 15x15 minimum grid. Game remains playable.

### T1.3 — Minimum cell size clamp
**Steps:** Resize the browser window to 160x200.
**Expected:** Cell size is 10px. Grid is 15x15 or larger. Game renders correctly.

### T1.4 — "Screen too small" message
**Steps:** Resize the browser window below 150x190.
**Expected:** Game displays "Screen too small" message. No canvas/grid rendered.

### T1.5 — "Screen too small" recovery
**Steps:** From the "Screen too small" state, expand the window above 150x190.
**Expected:** Start screen renders normally. Grid is computed at the new size.

### T1.6 — Grid locked during play
**Steps:** Start a game, then resize the browser window (larger or smaller, but still above the locked canvas size).
**Expected:** Grid dimensions, cell size, and canvas size do not change. Canvas re-centers.

### T1.7 — HUD height
**Steps:** Inspect the HUD area at the top of the canvas.
**Expected:** HUD is exactly 40px tall. Two lines of 16px text with 4px padding top/bottom.

### T1.8 — Footer visibility
**Steps:** Observe the start screen, then start a game.
**Expected:** Control instructions footer is visible on the start screen. Footer is hidden during gameplay.

---

## 2. Mouse Movement

### T2.1 — Initial spawn position
**Steps:** Start a new game.
**Expected:** Mouse head is at `(floor(gridWidth/2), floor(gridHeight/2))`. Body is 3 segments extending left from the head.

### T2.2 — Initial direction
**Steps:** Start a new game without pressing any keys.
**Expected:** Mouse moves to the right automatically.

### T2.3 — Arrow key controls
**Steps:** During gameplay, press each arrow key (Up, Down, Left, Right).
**Expected:** Mouse changes direction accordingly.

### T2.4 — WASD controls
**Steps:** During gameplay, press W, A, S, D.
**Expected:** Mouse changes direction (W=up, A=left, S=down, D=right).

### T2.5 — Reversal prevention
**Steps:** While moving right, press Left. While moving up, press Down.
**Expected:** Direction does not change. Reversal input is discarded.

### T2.6 — Valid U-turn over two ticks
**Steps:** While moving right, quickly press Down then Left within one tick.
**Expected:** Mouse turns down on the next tick, then left on the tick after. Not treated as a reversal.

### T2.7 — Input buffer cap
**Steps:** During gameplay, rapidly press 5+ direction keys within one tick.
**Expected:** Only the first 3 inputs are buffered. Additional inputs are discarded.

### T2.8 — Input buffer flush on pause
**Steps:** Queue 2-3 direction inputs, then immediately press P to pause.
**Expected:** All buffered inputs are discarded. On unpause, mouse continues in its pre-pause direction.

### T2.9 — Continuous movement
**Steps:** Start a game and do not press any keys.
**Expected:** Mouse moves continuously in the initial direction (right) without stopping.

---

## 3. Wall Wrapping

### T3.1 — Wrap right to left
**Steps:** Move the mouse to the right edge of the grid.
**Expected:** Mouse exits right side and appears on the left side, same row.

### T3.2 — Wrap left to right
**Steps:** Move the mouse to the left edge of the grid.
**Expected:** Mouse exits left side and appears on the right side, same row.

### T3.3 — Wrap top to bottom
**Steps:** Move the mouse upward to the top edge.
**Expected:** Mouse exits top and appears at the bottom, same column.

### T3.4 — Wrap bottom to top
**Steps:** Move the mouse downward to the bottom edge.
**Expected:** Mouse exits bottom and appears at the top, same column.

### T3.5 — Wrap while growing
**Steps:** Eat cheese near an edge so the mouse wraps on the same tick it grows.
**Expected:** Mouse wraps correctly. Tail is not removed (growing). No visual glitches.

---

## 4. Cheese

### T4.1 — Initial cheese spawn
**Steps:** Start a new game.
**Expected:** One cheese appears on the grid at a random position, at least 2 cells (Chebyshev, wrap-aware) from the mouse head.

### T4.2 — Cheese eaten and regrown
**Steps:** Move the mouse over the cheese.
**Expected:** Cheese disappears. Mouse grows by 1 segment. New cheese spawns at a different location.

### T4.3 — Cheese spawn distance
**Steps:** Eat 10+ cheese and observe spawn positions.
**Expected:** Cheese never spawns within 2 cells (Chebyshev, wrap-aware) of the mouse head. If constraint can't be met (late game), any empty cell is used.

### T4.4 — Cheese spawn wrap-aware distance
**Steps:** Position mouse near a grid edge and eat cheese.
**Expected:** Cheese distance calculation wraps around edges. A cheese 2 cells away via wrapping is treated as 2 cells, not `gridWidth - 2`.

### T4.5 — Score on eat
**Steps:** Eat cheese at different speed levels.
**Expected:** Score increases by `10 * speed_level` each time.

### T4.6 — Eat particle effect
**Steps:** Eat cheese.
**Expected:** Brief yellow particle burst at the cheese position.

---

## 5. Collision & Game Over

### T5.1 — Self-collision
**Steps:** Grow the mouse to 5+ segments, then turn into its own body.
**Expected:** Game ends. Game-over screen displays.

### T5.2 — Obstacle collision
**Steps:** After obstacles appear, steer the mouse into an obstacle.
**Expected:** Game ends. Game-over screen displays.

### T5.3 — Tail chase is legal
**Steps:** Move the mouse so its head enters the cell the tail just vacated (non-growing tick).
**Expected:** No collision. Mouse continues normally. The tail is removed before collision is checked.

### T5.4 — Game-over screen content
**Steps:** Trigger game over.
**Expected:** Screen shows "GAME OVER", final score, high score, "SPACE to restart", "M to toggle music". If new high score, "NEW HIGH SCORE!" in gold.

### T5.5 — Game-over freeze frame
**Steps:** Trigger game over.
**Expected:** Screen briefly freezes showing the collision point highlighted before the overlay appears.

### T5.6 — Death particle effect
**Steps:** Trigger game over.
**Expected:** Grey particle explosion at the collision point.

### T5.7 — Game-over SFX
**Steps:** Trigger game over with audio enabled.
**Expected:** Descending buzz sound plays.

---

## 6. Progressive Obstacles

### T6.1 — First obstacle at 5 cheese
**Steps:** Eat 5 cheese.
**Expected:** One obstacle appears on the grid.

### T6.2 — Obstacle spawn distance
**Steps:** Observe obstacle placement.
**Expected:** Obstacle is at least 3 cells (Chebyshev, wrap-aware) from the mouse head.

### T6.3 — Obstacle not in forward path
**Steps:** Observe obstacle placement.
**Expected:** Obstacle does not appear in the next 3 cells of the mouse's current direction.

### T6.4 — BFS cheese reachability
**Steps:** After obstacle placement, verify the cheese is still reachable.
**Expected:** A path exists from the mouse head to the cheese (no obstacle isolates the cheese).

### T6.5 — BFS minimum empty cells
**Steps:** After obstacle placement, count reachable empty cells.
**Expected:** At least 10 empty cells (or 10% of total empty, whichever is smaller) remain reachable. Cheese cell excluded from count.

### T6.6 — Deferred obstacle
**Steps:** Create a scenario where obstacle placement fails all 10 retries (e.g., very crowded board). *Note: Use a small grid (e.g., `MIN_GRID = 5`) or debug tooling to create a crowded board where placement retries are likely to fail.*
**Expected:** Obstacle spawn is deferred. `pendingObstacles` counter increments. On the next cheese eaten, one deferred obstacle is attempted.

### T6.7 — One deferred per cheese
**Steps:** Accumulate 3+ deferred obstacles. *Note: Use the same small-grid approach as T6.6 to force multiple deferrals.*
**Expected:** Only one deferred obstacle is resolved per cheese eaten. No sudden burst.

### T6.8 — Obstacle cap (5% of grid)
**Steps:** Play until obstacle count reaches 5% of total grid cells.
**Expected:** No more obstacles spawn. Any pending obstacles are discarded (counter reset to 0).

### T6.9 — Obstacle SFX
**Steps:** Trigger obstacle placement with audio enabled.
**Expected:** Low thud sound plays for both on-threshold and deferred placements.

### T6.10 — Cheese spawns before obstacle in post-eat
**Steps:** Eat the 5th cheese (triggering both cheese spawn and obstacle spawn). Observe the board.
**Expected:** New cheese is already on the board when the obstacle is placed. The obstacle's BFS validates reachability to the *new* cheese, not the old one.

---

## 7. Speed & Scoring

### T7.1 — Speed level formula
**Steps:** Track cheese eaten and speed level.
**Expected:** `speed_level = floor(cheese_eaten / 3) + 1`. Matches at all values.

### T7.2 — Tick interval decrease
**Steps:** Eat 3 cheese. Observe movement speed.
**Expected:** Tick interval decreases by 10ms (150ms → 140ms). Mouse moves noticeably faster.

### T7.3 — Maximum speed cap
**Steps:** Eat 27+ cheese.
**Expected:** Tick interval is 60ms and does not decrease further. Movement speed is capped.

### T7.4 — Score continues past speed cap
**Steps:** Eat cheese after reaching max speed (27+ cheese).
**Expected:** `speed_level` continues to increase. Score per cheese continues to grow (e.g., at 30 cheese: speed_level=11, 110 points per cheese).

### T7.5 — HUD displays speed level
**Steps:** Observe HUD during gameplay.
**Expected:** Speed level shown, not raw tick interval. Updates on every 3rd cheese.

---

## 8. Win Condition

### T8.1 — Win trigger
**Steps:** Fill the entire grid (mouse body + obstacles occupy all cells so no cheese can spawn). *Note: This is nearly impossible on a standard grid. To test manually, temporarily set `MIN_GRID = 5` and `PREFERRED_CELL_SIZE` accordingly to create a tiny 5x5 board, or add a debug key that fills the grid. Alternatively, verify via code inspection that the win check fires when `spawnCheese()` returns null.*
**Expected:** Game transitions to win screen.

### T8.2 — Win screen content
**Steps:** Trigger win.
**Expected:** "YOU WIN!" in gold, final score, high score, "SPACE to restart", "M to toggle music".

### T8.3 — Win new high score
**Steps:** Win with a score higher than current high score.
**Expected:** "NEW HIGH SCORE!" displayed. Celebration effect is enhanced. High score updated in localStorage.

### T8.4 — Win celebration particles
**Steps:** Trigger win.
**Expected:** Multi-colored particle fountain from center of screen.

---

## 9. High Score

### T9.1 — High score persistence
**Steps:** Score 100 points. Close and reopen the browser.
**Expected:** High score of 100 is displayed on HUD. Also visible on game-over and win screens.

### T9.2 — High score update on game over
**Steps:** Score higher than existing high score, then die.
**Expected:** High score updates immediately. Stored in localStorage under `mouseGameHighScore`.

### T9.3 — High score update on win
**Steps:** Win with a score higher than existing high score.
**Expected:** High score updates. Displayed on win screen.

### T9.4 — High score not overwritten by lower score
**Steps:** Score lower than the existing high score, then die.
**Expected:** High score remains unchanged.

### T9.5 — High score on HUD
**Steps:** Play with an existing high score.
**Expected:** High score is visible on HUD throughout gameplay.

### T9.6 — High score on game-over and win screens
**Steps:** Trigger game over and win.
**Expected:** Both screens display the current high score.

---

## 10. Screens & Overlays

### T10.1 — Start screen rendering
**Steps:** Load the game.
**Expected:** Semi-transparent overlay on top of empty game board. Title "MOUSE", full controls list, "Press SPACE to Start" (blinking).

### T10.2 — Start screen to playing
**Steps:** Press SPACE on start screen.
**Expected:** Overlay disappears. Game begins. Mouse starts moving. Music starts if `musicEnabled`.

### T10.3 — Pause screen
**Steps:** Press P during gameplay.
**Expected:** Semi-transparent overlay with "PAUSED" and "P to resume · M to toggle music". Game is frozen.

### T10.4 — Unpause
**Steps:** Press P while paused.
**Expected:** Overlay disappears. Game resumes from exact state. Music resumes if `musicEnabled`.

### T10.5 — Game-over to restart
**Steps:** On game-over screen, press SPACE.
**Expected:** Game resets. New game begins from initial state. High score preserved.

### T10.6 — Win to restart
**Steps:** On win screen, press SPACE.
**Expected:** Game resets. New game begins. High score preserved.

### T10.7 — Resize-block overlay
**Steps:** Shrink window below locked canvas size during gameplay.
**Expected:** "Window too small — resize to continue" overlay. Game halted.

### T10.8 — Resize-block "Screen too small" variant
**Steps:** Shrink window below 150x190 during gameplay.
**Expected:** "Screen too small" text instead of "Window too small". Same `resizeBlocked` state.

### T10.9 — Resize-block recovery
**Steps:** Expand window after resize-block.
**Expected:** Overlay disappears. Game resumes if `userPaused` is false. If `userPaused` is true, normal pause screen shows.

---

## 11. State Transitions

### T11.1 — SPACE ignored during gameplay
**Steps:** Press SPACE while playing.
**Expected:** Nothing happens.

### T11.2 — SPACE ignored during pause
**Steps:** Press SPACE while paused.
**Expected:** Nothing happens. Game stays paused.

### T11.3 — SPACE ignored during tooSmall
**Steps:** Press SPACE on "Screen too small" screen.
**Expected:** Nothing happens.

### T11.4 — SPACE ignored during resize-block
**Steps:** Press SPACE during resize-block.
**Expected:** Nothing happens.

### T11.5 — P ignored on start screen
**Steps:** Press P on start screen.
**Expected:** Nothing happens.

### T11.6 — P ignored on game-over
**Steps:** Press P on game-over screen.
**Expected:** Nothing happens.

### T11.7 — P ignored on win
**Steps:** Press P on win screen.
**Expected:** Nothing happens.

### T11.8 — P during resize-block
**Steps:** Press P during resize-block while game was playing.
**Expected:** `userPaused` toggled to true. `pauseMuted` set to true accordingly. When resize recovers, pause screen shows instead of resuming. Audio remains suppressed (both `pauseMuted` and `resizeMuted` were active; after recovery `resizeMuted` clears but `pauseMuted` persists).

### T11.9 — P during resize-block (already paused)
**Steps:** Pause the game, then trigger resize-block, then press P.
**Expected:** `userPaused` is cleared. `pauseMuted` is cleared accordingly. When resize recovers, game resumes playing. Music resumes if `musicEnabled` is true (since neither `pauseMuted` nor `resizeMuted` remain active).

### T11.10 — Arrows ignored on non-playing screens
**Steps:** Press arrow keys on start, tooSmall, paused, game-over, win screens.
**Expected:** No direction input buffered on any of these screens.

### T11.11 — Arrows not buffered during resize-block
**Steps:** Press arrow keys during resize-block.
**Expected:** Inputs are ignored and not buffered. No stale turns on recovery.

### T11.12 — M on tooSmall
**Steps:** Press M on "Screen too small" screen.
**Expected:** Music toggle is ignored (`musicEnabled` does not change). However, AudioContext may initialize if this is the first user gesture.

### T11.13 — M during resize-block
**Steps:** Press M during resize-block.
**Expected:** Music toggle is ignored (`musicEnabled` does not change). However, AudioContext may initialize if this is the first user gesture.

---

## 12. Audio

### T12.1 — AudioContext init on keypress
**Steps:** Load the game. Press any key (e.g., M or SPACE).
**Expected:** AudioContext is created and resumed. No console warnings about autoplay.

### T12.2 — AudioContext init on click
**Steps:** Load the game. Click anywhere on the page without pressing any key first.
**Expected:** AudioContext is created and resumed.

### T12.3 — Eat SFX
**Steps:** Eat cheese with audio enabled.
**Expected:** Short rising chirp (sine wave, 200-600Hz, ~100ms).

### T12.4 — Game-over SFX
**Steps:** Trigger game over with audio enabled.
**Expected:** Descending buzz (square wave, 400-100Hz, ~300ms).

### T12.5 — Obstacle SFX
**Steps:** Trigger obstacle placement with audio enabled.
**Expected:** Low thud (triangle wave, 80Hz, ~150ms).

### T12.6 — Menu select SFX on SPACE
**Steps:** Press SPACE to start or restart.
**Expected:** Quick blip sound (sine wave, 800Hz, ~50ms). Best-effort — may be silent on very first gesture.

### T12.7 — Menu select SFX on M
**Steps:** Press M to toggle music.
**Expected:** Quick blip sound. Best-effort on first gesture.

### T12.8 — SFX suppressed during pause
**Steps:** Pause the game. Press M (which normally plays menu select SFX).
**Expected:** No menu select sound plays. `pauseMuted` suppresses all SFX.

### T12.9 — SFX suppressed during resize-block
**Steps:** Trigger resize-block. Press P (which is allowed during resize-block). Verify no SFX plays for any internal event.
**Expected:** No sound plays. `resizeMuted` suppresses all SFX. Note: M is ignored during resize-block, so this test verifies the flag is set correctly for any code path that calls `playSFX()`.

### T12.10 — SFX resume after unpause
**Steps:** Pause, then unpause. Eat cheese.
**Expected:** Eat SFX plays normally.

### T12.11 — Game-over SFX plays with music disabled
**Steps:** Press M to disable music during gameplay. Then trigger game over.
**Expected:** Game-over descending buzz still plays. SFX are independent of `musicEnabled` — only `pauseMuted` and `resizeMuted` suppress them.

### T12.12 — SFX resume after resize-block recovery
**Steps:** Trigger resize-block. Recover by expanding the window. Eat cheese.
**Expected:** Eat SFX plays normally. `resizeMuted` is cleared on recovery.

---

## 13. Background Music

### T13.1 — Music starts on SPACE
**Steps:** Press SPACE to start the game with `musicEnabled = true`.
**Expected:** Background chiptune music begins playing.

### T13.2 — Music does NOT play on start screen
**Steps:** Load the game. Wait on start screen.
**Expected:** No music plays, even though `musicEnabled` defaults to true.

### T13.3 — M on start screen toggles flag only
**Steps:** Press M on start screen.
**Expected:** `musicEnabled` toggles. HUD indicator updates. No music plays or stops.

### T13.4 — M on start screen then SPACE
**Steps:** Press M (disable music) on start screen, then press SPACE.
**Expected:** Game starts. No music plays.

### T13.5 — M during gameplay
**Steps:** Press M while playing.
**Expected:** Music starts or stops immediately based on new `musicEnabled` value.

### T13.6 — M during pause
**Steps:** Press M while paused.
**Expected:** `musicEnabled` toggles. Music stays silent (due to `pauseMuted`). On unpause, music state matches new `musicEnabled`.

### T13.7 — M on game-over screen
**Steps:** Press M on game-over screen.
**Expected:** Music starts or stops immediately.

### T13.8 — M on win screen
**Steps:** Press M on win screen.
**Expected:** Music starts or stops immediately. Behavior identical to game-over screen.

### T13.9 — Music stops on pause
**Steps:** Play with music enabled, then press P.
**Expected:** Music stops immediately.

### T13.10 — Music resumes on unpause
**Steps:** Unpause with `musicEnabled = true`.
**Expected:** Music resumes.

### T13.11 — Music does not resume if disabled before pause
**Steps:** Press M to disable music, then pause, then unpause.
**Expected:** No music on unpause. `musicEnabled` is false.

### T13.12 — Music stops on resize-block
**Steps:** Trigger resize-block with music playing.
**Expected:** Music stops. `resizeMuted` is true.

### T13.13 — Music resumes on resize-block recovery
**Steps:** Recover from resize-block with `musicEnabled = true` and `userPaused = false`.
**Expected:** Music resumes.

### T13.14 — Music does not resume if paused during resize-block
**Steps:** Press P during resize-block. Recover from resize-block.
**Expected:** Music stays silent. `pauseMuted` is true. Pause screen shows.

### T13.15 — Music state persists across restart
**Steps:** Disable music with M. Restart the game with SPACE.
**Expected:** Music remains disabled in the new game.

### T13.16 — Music is a looping chiptune
**Steps:** Let the game play for 30+ seconds with music enabled.
**Expected:** Music loops seamlessly. 4-bar pattern at ~120 BPM.

### T13.17 — `musicEnabled` defaults to true on first load
**Steps:** Clear localStorage. Load the game for the first time. Observe the HUD.
**Expected:** Music note icon in HUD is filled (indicating `musicEnabled = true`). Music will start when SPACE is pressed.

---

## 14. Visual & Sprites

### T14.1 — Pixel art sprites at 20px
**Steps:** Play the game at default 20px cell size.
**Expected:** Mouse head has ears, eyes, whiskers. Cheese has holes. Obstacles have brick pattern.

### T14.2 — Simplified sprites below 16px
**Steps:** Play the game with cell size < 16px (small viewport).
**Expected:** Mouse head is a solid circle with ear dots. Cheese is a solid yellow square. No sub-pixel artifacts.

### T14.3 — Mouse head orientation
**Steps:** Move the mouse in all 4 directions.
**Expected:** Mouse head faces the direction of movement.

### T14.4 — Body color gradient
**Steps:** Grow the mouse to 10+ segments.
**Expected:** Body segments have slight color variation (lighter grey, gradient by index).

### T14.5 — HUD pixel art icons
**Steps:** Observe the HUD.
**Expected:** Cheese, bolt, trophy, and music note icons are pixel-art styled (not emojis). Music note filled when enabled, outlined when disabled.

### T14.6 — Collision highlight
**Steps:** Trigger game over.
**Expected:** The cell where collision occurred is visually highlighted in the final frame.

---

## 15. Particles

### T15.1 — Eat particles
**Steps:** Eat cheese.
**Expected:** 5-8 small yellow squares burst outward from cheese position. Particles fade and disappear.

### T15.2 — Death particles
**Steps:** Trigger game over.
**Expected:** 10-15 grey squares explode from collision point.

### T15.3 — Win celebration particles
**Steps:** Trigger win.
**Expected:** 20-30 multi-colored particles fountain from center.

### T15.4 — Enhanced win celebration
**Steps:** Win with a new high score.
**Expected:** Celebration effect is visually enhanced compared to regular win.

### T15.5 — Particles don't leak
**Steps:** Play through many eat/death cycles.
**Expected:** Dead particles are removed. No performance degradation over time.

---

## 16. Resize-Block State

### T16.1 — Input buffer flushed on resize-block
**Steps:** Queue direction inputs, then immediately shrink the window below canvas size.
**Expected:** All buffered inputs are discarded.

### T16.2 — Inputs not buffered during resize-block
**Steps:** Press direction keys during resize-block.
**Expected:** Inputs are ignored. Buffer remains empty.

### T16.3 — `userPaused` independent of `resizeBlocked`
**Steps:** Pause game (P), then trigger resize-block, then recover.
**Expected:** Game shows pause screen after recovery (both states tracked independently).

### T16.4 — `resizeMuted` independent of `pauseMuted`
**Steps:** Trigger resize-block without pausing first. Check audio flags.
**Expected:** `resizeMuted = true`, `pauseMuted = false`. Both tracked independently.

### T16.5 — Double mute (pause + resize)
**Steps:** Pause, then resize-block. Recover from resize. Check audio.
**Expected:** Audio still suppressed (`pauseMuted` still true). Unpause to restore audio.

### T16.6 — Grid never recomputed during active game
**Steps:** Resize window multiple times during gameplay (above canvas size).
**Expected:** `gridWidth`, `gridHeight`, and `cellSize` never change. Only canvas centering adjusts.

---

## 17. Performance

### T17.1 — No orphaned timers
**Steps:** Start, pause, unpause, die, restart, pause, resize-block, recover — rapidly.
**Expected:** No duplicate tick timers. Only one `setTimeout` active at a time.

### T17.2 — No duplicate rAF loops
**Steps:** Restart the game multiple times.
**Expected:** Only one `requestAnimationFrame` loop runs. No frame rate degradation.

### T17.3 — BFS performance
**Steps:** Play on a large grid (30x30 = 900 cells) with many obstacles.
**Expected:** No visible frame stutter when obstacles are placed. BFS completes within one tick.

### T17.4 — Audio cleanup
**Steps:** Eat 50+ cheese (triggering 50+ SFX oscillators).
**Expected:** No audio glitches. Oscillators are stopped and garbage collected.

### T17.5 — Particle cleanup
**Steps:** Eat 100+ cheese (spawning 500+ particles total).
**Expected:** Dead particles are removed each frame. Memory usage stable.

---

## 18. Regression Checklist

Run these after each implementation phase to catch regressions:

### After Phase 1 (Core Game)
- [ ] T1.1, T1.6 — Grid renders and locks
- [ ] T2.1-T2.9 — Movement and input
- [ ] T3.1-T3.5 — Wall wrapping
- [ ] T4.1-T4.5 — Cheese spawn, eat, distance, and score
- [ ] T5.1, T5.3 — Self-collision and tail chase

### After Phase 2 (Progression)
- [ ] T7.1-T7.5 — Speed and scoring
- [ ] T6.1-T6.8 — Obstacles and BFS
- [ ] T9.1-T9.4 — High score
- [ ] T8.1-T8.3 — Win trigger, screen content, high score (T8.4 particles deferred to Phase 3)
- [ ] All Phase 1 regression tests

### After Phase 3 (Polish)
- [ ] T14.1-T14.6 — Sprites
- [ ] T15.1-T15.5 — Particles
- [ ] T4.6 — Eat particle effect (deferred from Phase 1)
- [ ] T8.4 — Win celebration particles (deferred from Phase 2)
- [ ] T10.1-T10.9 — Screen overlays
- [ ] T16.1-T16.6 — Resize-block
- [ ] All Phase 1-2 regression tests

### After Phase 4 (Audio)
- [ ] T12.1-T12.12 — SFX
- [ ] T13.1-T13.17 — Music
- [ ] All Phase 1-3 regression tests

### Final (Phase 5)
- [ ] T11.1-T11.13 — All state transitions
- [ ] T17.1-T17.5 — Performance
- [ ] Full regression of all 111 tests

---

## Test Summary

| Section | Tests | Coverage |
|---------|-------|----------|
| 1. Grid & Board | T1.1-T1.8 | Grid sizing, fallback, lock, HUD, footer |
| 2. Movement | T2.1-T2.9 | Spawn, direction, reversal, buffer, flush |
| 3. Wall Wrapping | T3.1-T3.5 | All 4 edges + wrap while growing |
| 4. Cheese | T4.1-T4.6 | Spawn, eat, distance, score, particles |
| 5. Collision | T5.1-T5.7 | Self, obstacle, tail chase, game-over |
| 6. Obstacles | T6.1-T6.10 | Spawn, safety, BFS, defer, cap, SFX, ordering |
| 7. Speed & Scoring | T7.1-T7.5 | Formula, decrease, cap, HUD |
| 8. Win Condition | T8.1-T8.4 | Trigger, screen, high score, particles |
| 9. High Score | T9.1-T9.6 | Persistence, update, display |
| 10. Screens | T10.1-T10.9 | Start, pause, game-over, win, resize-block |
| 11. State Transitions | T11.1-T11.13 | Key behavior on every screen |
| 12. Audio | T12.1-T12.12 | Init, SFX, suppression, resume, independence |
| 13. Music | T13.1-T13.17 | Toggle, pause, resize, persist, default |
| 14. Visual | T14.1-T14.6 | Sprites, scaling, orientation, HUD |
| 15. Particles | T15.1-T15.5 | Eat, death, win, cleanup |
| 16. Resize-Block | T16.1-T16.6 | Buffer, state independence, grid lock |
| 17. Performance | T17.1-T17.5 | Timers, rAF, BFS, audio, particles |
| 18. Regression | Checklists | Per-phase regression gates |
| **Total** | **111 tests** | |
