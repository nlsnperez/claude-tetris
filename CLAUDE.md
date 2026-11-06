# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Vanilla-JS Tetris: three files (`index.html`, `style.css`, `game.js`), no dependencies, no build step, no tests. UI strings and README are in Spanish — keep new user-facing text in Spanish.

## Running

Open `index.html` directly (`start index.html` on Windows) — it works over `file://`. A static server is optional:

```bash
python3 -m http.server 8000   # or: npx serve .
```

There is no linter, test runner, or `package.json`. Verification is manual: load the page and play. The `/run` skill can drive a browser if you need to confirm a change visually.

## Architecture (`game.js`)

A single script in the global scope, `'use strict'`, loaded with a plain `<script>` tag at the end of `<body>` — no modules, no `DOMContentLoaded` guard (DOM lookups at the top of the file rely on that load position).

**State** is a flat set of module-level `let` bindings (`board`, `current`, `next`, `score`, `lines`, `level`, `paused`, `gameOver`, `lastTime`, `dropAccum`, `dropInterval`, `animId`). `init()` is both bootstrap and restart: it resets every one of them, cancels any in-flight animation frame, and hides the overlay. Anything you add to game state must be reset there too.

**Piece encoding is the key invariant.** `PIECES` and `COLORS` are parallel 1-based arrays whose index `0` is `null`, because `0` doubles as "empty cell" in the board matrix. A piece's type index *is* its color index, and that index is what gets written into `board` on lock — so a cell's value simultaneously says "occupied" and "which color". Adding a piece means appending to `PIECES` **and** `COLORS` at the same index, and widening the `Math.random() * 7` in `randomPiece()`.

**Rotation** is matrix transpose-and-reverse (`rotateCW`), not SRS. `tryRotate()` approximates wall kicks by retrying the rotated shape at x-offsets `[0, -1, 1, -2, 2]` and taking the first that doesn't collide; if none fit, the rotation is silently dropped. Piece shapes are square matrices padded with zeros, which is what makes naive transposition produce acceptable pivots.

**`collide(shape, ox, oy)`** is the single source of truth for legality and is reused for movement, rotation kicks, ghost projection, gravity, and game-over detection. It deliberately permits `ny < 0` (cells above the board) so pieces can spawn and rotate partially off-screen; it rejects `ny >= ROWS` and any out-of-range `nx`.

**Game loop**: `loop(ts)` accumulates `dt` into `dropAccum` and advances one row when it exceeds `dropInterval`, then redraws unconditionally. Pausing cancels the frame; unpausing resets `lastTime` to `performance.now()` before re-entering `loop` so the paused interval isn't credited as elapsed time. Game over cancels the frame and leaves state intact for the overlay.

**Rendering** is full-redraw each frame into `#board` (grid lines, locked board, ghost, current piece, in that order). `drawBlock` is shared by the board and the next-piece canvas via a `context` + `size` parameter; the ghost is just the same call with `alpha = 0.2`.

**Scoring/level**: `LINE_SCORES[cleared] * level`, plus 2/cell for hard drop and 1/row for soft drop. Level is derived from `lines` (`floor(lines / 10) + 1`) and drives `dropInterval = max(100, 1000 - (level - 1) * 90)`.

## Coupled values across files

These are hardcoded in more than one place and must be changed together:

- `COLS × BLOCK` and `ROWS × BLOCK` (`game.js`) must equal the `width`/`height` attributes of `<canvas id="board">` (`index.html`, currently 300 × 600).
- `drawNext()` centers the preview inside a hardcoded 4×4 grid at `NB = 30`, which assumes `#next-canvas` is 120 × 120.
- The DOM ids read at the top of `game.js` (`board`, `next-canvas`, `score`, `lines`, `level`, `overlay`, `overlay-title`, `overlay-score`, `restart-btn`) are dereferenced without null checks — renaming one in `index.html` breaks the script at load.
- The control list rendered in `index.html` duplicates the key bindings in the `keydown` handler.
- `README.md` documents the tunable constants and the game flow; update it when that behavior changes.
