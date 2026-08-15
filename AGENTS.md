# AGENTS.md

Pac-Man clone in vanilla JS/HTML/CSS. No build system, no bundler, no `package.json`, no tests, no lint/typecheck. Educational repo for practicing the spec-driven development workflow (see `.agents/skills/spec` and `.agents/skills/spec-impl`).

## Running

Open `src/index.html` directly in a browser (file:// works — no fetch/modules). Entry HTML loads scripts in this strict order; reorder breaks globals:

```
js/maze.js -> js/game.js -> js/render.js -> js/main.js
```

## Architecture

Each file attaches its API to `window` (no ES modules, no imports). Globals in use:

- `maze.js`: `MAZE` (28×31 numeric grid, pristine — never mutated), `TUNNEL_ROW` (=14), `PACMAN_START` ({x:13,y:23}), `GHOST_STARTS` (array with `kind: 'hunter' | 'random'`).
- `game.js`: `createGame()`, `update(game)`, `DIRS`.
- `render.js`: `draw(ctx, game, frame)`. `TILE=20`, canvas is 560×620.
- `main.js`: owns the `requestAnimationFrame` loop, keyboard input, and overlay/HUD. Orchestrates the others.

`createGame()` copies `MAZE` into `game.grid`, which is what gets mutated during play (dots eaten). Always read live state from `game.grid`, not `MAZE`.

## Tile encoding (maze.js)

Cell values: `1`=wall, `2`=dot, `0`=walkable empty, `3`=pen door. Maze is authored as 31 readable strings of 28 chars and parsed via `parseTile`. Coordinates are cell `(x,y)` with origin top-left; `x∈[0,27]`, `y∈[0,30]`. Grid is symmetric about the vertical axis between cols 13 and 14.

Wall rules (game.js `isWall`): Pacman is blocked by wall (`1`) AND pen door (`3`); ghosts are blocked only by wall (`1`) — ghosts can pass the door. Tunnel wrap applies only on `TUNNEL_ROW` (row 14).

## Movement & AI quirks

- `PACMAN_SPEED=0.125` (1/8 cell/frame), `GHOST_SPEED=0.1` (1/10 cell/frame). Actors move on fractional cell coords; turns/decisions happen only when `aligned` (within 1e-3 of an integer).
- `nextDir` queues a turn; applied at the next alignment if `canMove`.
- Ghosts never reverse (`OPPOSITE` filtered out) except in a dead-end (no other valid exit).
- `hunter` ghosts pick the neighbor minimizing Manhattan distance to Pacman; `random` ghosts pick uniformly among valid choices. There is no pathfinding, no mode cycling (chase/scatter/frightened), and no pen-release logic.

## Game state

`game.state ∈ {start, playing, won, lost}`. Start with `lives=3`; collision (within 0.5 cell) costs a life and resets positions. Win when `dotsRemaining===0`; each dot is 10 points.

## Conventions

- Comments and UI strings are in Spanish; identifiers in English.
- Code style: single quotes, spaces inside parens for calls (`foo( x )`), 2-space indent. Match existing style when editing.
- Do not add a bundler, modules, or `package.json` unless explicitly asked — the file://-openable vanilla setup is intentional.

## Workflow (spec-driven)

This repo is a teaching example for the spec-driven method. For new features:

1. Use the `spec` skill to draft a spec (asks clarifying questions, builds spec section by section) before writing code.
2. Once the spec state is "Approved", use the `spec-impl` skill: it creates a git branch named after the spec, switches to it, and implements step by step with pauses for diff review.
3. Do not squash or amend during implementation unless asked; review commits separately.