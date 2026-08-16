# AGENTS.md

Pac-Man clone in vanilla JS/HTML/CSS. No build system, no bundler, no `package.json`, no tests, no lint/typecheck. Educational repo for practicing the spec-driven development workflow (see `.agents/skills/spec` and `.agents/skills/spec-impl`).

## Running

Open `src/index.html` directly in a browser (file:// works — no fetch/modules). Entry HTML loads scripts in this strict order; reorder breaks globals:

```
js/maze.js -> js/game.js -> js/render.js -> js/main.js
```

## Architecture

Each file attaches its API to `window` (no ES modules, no imports). Globals in use:

- `maze.js`: `MAZE` (28×31 numeric grid, pristine — never mutated), `TUNNEL_ROW` (=14), `PACMAN_START` ({x:13,y:23}), `GHOST_STARTS` (array with `kind: 'blinky' | 'pinky' | 'inky' | 'clyde'`), `PEN_EXIT`, `RELEASE_FRAMES`, `POWER_PELLETS` (the 4 corner cells).
- `game.js`: `createGame()`, `update(game)`, `DIRS`.
- `render.js`: `draw(ctx, game, frame)`. `TILE=20`, canvas is 560×620.
- `main.js`: owns the `requestAnimationFrame` loop, keyboard input, and overlay/HUD. Orchestrates the others.

`createGame()` copies `MAZE` into `game.grid`, which is what gets mutated during play (dots eaten). Always read live state from `game.grid`, not `MAZE`.

## Tile encoding (maze.js)

Cell values: `1`=wall, `2`=dot, `0`=walkable empty, `3`=pen door, `4`=power pellet (eaten like a dot: +50 pts and triggers the fright effect). Maze is authored as 31 readable strings of 28 chars and parsed via `parseTile`. Coordinates are cell `(x,y)` with origin top-left; `x∈[0,27]`, `y∈[0,30]`. Grid is symmetric about the vertical axis between cols 13 and 14.

Wall rules (game.js `isWall`): Pacman and ghosts are both blocked by wall (`1`) AND pen door (`3`) — nobody can enter the pen; ghosts leave only via the release teleport. Tunnel wrap applies only on `TUNNEL_ROW` (row 14).

## Movement & AI quirks

- `PACMAN_SPEED=0.125` (1/8 cell/frame), `GHOST_SPEED=0.1` (1/10 cell/frame). Actors move on fractional cell coords; turns/decisions happen only when `aligned` (within 1e-3 of an integer).
- `nextDir` queues a turn; applied at the next alignment if `canMove`.
- Ghosts never reverse (`OPPOSITE` filtered out) except in a dead-end (no other valid exit).
- Ghosts pick directions per `kind`: blinky chases Pacman directly (greedy Manhattan), pinky targets 4 cells ahead of Pacman, inky reflects through blinky, clyde picks uniformly. There is no pathfinding and no chase/scatter mode cycling.
- Frightened mode: while `game.frightTimer > 0` (after eating a power pellet, 480 frames), the ghosts that were already released at that moment get `frightened=true`; they pick uniformly among valid choices and move at half speed (`GHOST_FRIGHT_SPEED=0.05`). A frightened ghost colliding with Pacman is eaten (+200/400/800/1600 pts doubling per ghost in the same effect) and teleported back to the pen to be released again with its original `RELEASE_FRAMES` — a ghost re-released mid-effect is **not** frightened. Collisions are harmless for 10 frames after eating a ghost (`graceFrames`). Losing a life cancels the effect.

## Game state

`game.state ∈ {start, playing, won, lost}`. Start with `lives=3`; collision (within 0.5 cell) costs a life and resets positions — unless the ghost is frightened (it is eaten instead) or `game.graceFrames > 0`. Win when `dotsRemaining===0` (power pellets count toward it); each dot is 10 points, each power pellet 50. Per-game state also tracks `game.frightTimer` (frames of fright left), `game.combo` (ghosts eaten in the current effect), and `game.graceFrames` (invulnerability after eating a ghost).

## Conventions

- Comments and UI strings are in Spanish; identifiers in English.
- Code style: single quotes, spaces inside parens for calls (`foo( x )`), 2-space indent. Match existing style when editing.
- Do not add a bundler, modules, or `package.json` unless explicitly asked — the file://-openable vanilla setup is intentional.

## Workflow (spec-driven)

This repo is a teaching example for the spec-driven method. For new features:

1. Use the `spec` skill to draft a spec (asks clarifying questions, builds spec section by section) before writing code.
2. Once the spec state is "Approved", use the `spec-impl` skill: it creates a git branch named after the spec, switches to it, and implements step by step with pauses for diff review.
3. Do not squash or amend during implementation unless asked; review commits separately.