# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with this repository.

## Project

Tetris in vanilla JavaScript + HTML5 Canvas + CSS. No dependencies, no build step, no package manager, no tests, no linter. The README and UI text are in Spanish; keep user-facing strings in Spanish.

## Running

Open `index.html` directly in a browser, or serve the folder (e.g. `python -m http.server` or VS Code Live Server) and reload after edits.

## Architecture

All game logic lives in `game.js` (a single classic script loaded by `index.html`, `'use strict'`, no modules). It relies on fixed DOM ids in `index.html` (`board`, `next-canvas`, `hold-canvas`, `score`, `lines`, `level`, `overlay`, `overlay-title`, `overlay-score`, `restart-btn`, `theme-toggle`, `tnt-status`) — renaming one requires updating both files. Canvas sizes are hardcoded in the HTML (`300x600` board = `COLS*BLOCK` x `ROWS*BLOCK`; `120x120` next and hold previews), so changing `COLS`/`ROWS`/`BLOCK` in `game.js` also requires changing the canvas attributes.

Key design points:
- **State is module-level `let` globals** (`board`, `current`, `next`, `score`, `lines`, `level`, `paused`, `gameOver`, `dropInterval`, ...), reset in `init()`, which is also the restart handler.
- **Pieces and board are numeric**: cell value `0` = empty, `1–7` = piece type, which indexes both `PIECES` (shape matrices) and `COLORS`. Index 0 of both arrays is `null`.
- **Game loop**: `loop(ts)` via `requestAnimationFrame` accumulates `dropAccum` and applies gravity when it reaches `dropInterval`, then redraws. Pause/game over cancel the frame; unpausing restarts `loop` with a fresh `lastTime`.
- **Piece lifecycle**: `lockPiece()` = `merge()` → `clearLines()` → `spawn()`. `spawn()` promotes `next`, generates a new one, and calls `endGame()` if the spawn position collides. Pieces are chosen with plain `Math.random()` (no 7-bag).
- **TNT power-up**: piece type `8` (`[[8]]`, 1x1; never merged into `board`). Each cleared row has a 10% chance (`addLines`) to set `tntReady` and show the `tnt-status` banner; key T (`activateTnt`) swaps the current piece for a TNT. When it lands, `lockPiece()` calls `explode()`, which removes 3 rows centered on it (clamped to the board) and scores them via `addLines(3)`.
- **Guardar (hold)**: key S (`holdPiece`) stores the current piece in `held` (one slot, drawn in `hold-canvas`). Pressing S again drops the stored piece from the top and the falling piece becomes `next` (the old `next` is discarded). `canHold` limits it to once per piece (reset in `lockPiece()`); TNT can't be held.
- **Input** is handled in one `keydown` listener (arrows, X rotate, Space hard drop, P pause, T TNT, S guardar); movement/rotation is validated through `collide(shape, ox, oy)`. Rotation is clockwise only, with simple horizontal wall kicks `[0,-1,1,-2,2]`.
- **Scoring/speed** (in `clearLines`): `LINE_SCORES[cleared] * level`; level = `floor(lines/10)+1`; `dropInterval = max(100, 1000 - (level-1)*90)`. Soft drop gives +1/cell, hard drop +2/cell.
