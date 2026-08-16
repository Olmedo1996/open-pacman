# SPEC 02 — Corrección del pen: 4 fantasmas dentro, sin reingreso

> **Status:** Approved
> **Depends on:** SPEC 01
> **Date:** 2026-08-16
> **Objective:** Corregir el punto de partida de los fantasmas: los 4 arrancan dentro del pen, salen por teleport a PEN_EXIT con el orden actual (0/2/4/6 s) y ya no pueden reingresar al pen ni quedarse atrapados.

## Scope

**In:**

- Mover el inicio de Blinky de `(13,11)` (fuera del pen) a `(14,14)` (dentro), manteniendo `pinky(13,14)`, `inky(11,14)` y `clyde(15,14)`.
- Bloquear la puerta del pen (`tile 3`) para los fantasmas: en `isWall`, la puerta es muro para Pacman y fantasmas. Elimina el bug de raíz: un fantasma liberado ya no puede entrar al pen (única entrada) ni quedarse atrapado en su interior.
- Mantener el mecanismo de liberación actual: teleport a `PEN_EXIT` con `RELEASE_FRAMES` 0/2/4/6 s (blinky, pinky, inky, clyde).
- Eliminar el parámetro `actor` de `isWall` y `canMove` (queda sin uso al ser la puerta muro para ambos actores).
- Actualizar `AGENTS.md` (regla de paredes).

**Out of scope (for future specs):**

- Modos chase/scatter/frightened y power-pellets.
- Pathfinding real (BFS/A*).
- Animación de caminar dentro del pen o salida caminando por la puerta.
- Liberación anticipada por conteo de dots.
- Nuevas estructuras de datos.

## Data model

No se introducen estructuras nuevas. Cambian los valores de una constante existente:

```js
// maze.js → GHOST_STARTS (blinky pasa de (13,11) a (14,14))
const GHOST_STARTS = [
  { x: 14, y: 14, kind: 'blinky' }, // dentro del pen
  { x: 13, y: 14, kind: 'pinky'  }, // dentro del pen
  { x: 11, y: 14, kind: 'inky'   }, // dentro del pen
  { x: 15, y: 14, kind: 'clyde'  }, // dentro del pen
];
```

`PEN_EXIT` y `RELEASE_FRAMES` permanecen igual (spec 01).

## Regla de paredes (game.js)

```js
// La puerta (3) es muro para Pacman Y fantasmas. El pen queda sellado;
// la única salida es el teleport de liberación.
function isWall( grid, x, y ) {
  if ( y < 0 || y >= grid.length ) return true;
  if ( x < 0 || x >= grid[ 0 ].length ) return true;
  const v = grid[ y ][ x ];
  return v === 1 || v === 3;
}
```

`canMove` pierde su parámetro `actor` y se actualizan sus 3 llamadas (`movePacman`, `decideGhost`, `moveGhost`). La geometría del pen (muros en col 10/17, fila 16, techo fila 12 salvo la puerta) garantiza que bloquear los dos tiles `3` sella el interior por completo.

## Implementation plan

1. **maze.js:** cambiar el inicio de Blinky a `(14,14)` en `GHOST_STARTS`. Verificar que el juego carga y Blinky sale al instante por teleport.
2. **game.js:** en `isWall`, tratar `v === 3` como muro para cualquier actor; eliminar `actor` de `isWall`/`canMove` y de las llamadas. El juego sigue funcional: los liberados ya no entran al pen.
3. **AGENTS.md:** actualizar la sección de reglas de paredes ("los fantasmas pasan la puerta" → la puerta bloquea a ambos).
4. Verificación manual final (criterios de aceptación).

Cada paso deja el juego funcional y commiteable.

## Acceptance criteria

- [ ] Al abrir `src/index.html`, en la pantalla de inicio se ven los 4 fantasmas dentro del pen.
- [ ] Al iniciar, Blinky sale al instante (teleport a `(13,11)`); Pinky/Inky/Clyde salen a los ~2/4/6 s.
- [ ] Durante la partida ningún fantasma liberado entra al interior del pen (se detiene ante la puerta).
- [ ] Tras perder una vida, `resetPositions` restaura los 4 dentro del pen y repite el orden de salida.
- [ ] La puerta se sigue dibujando (render.js sin cambios).
- [ ] No hay errores en la consola del navegador.
- [ ] `AGENTS.md` refleja la nueva regla de paredes.

## Decisions

- **Yes:** Bloquear la puerta para fantasmas. Elimina el bug de raíz con un cambio mínimo y sin pathfinding.
- **No:** Mantener la puerta abierta y bloquear el interior. Requiere marcar celdas interiores del pen (más código, mismo resultado).
- **No:** Lógica de escape dentro del pen. Roca el pathfinding y contradice la filosofía del repo.
- **Yes:** Blinky dentro en `(14,14)`, conservando las otras 3 posiciones de spec 01.
- **No:** Layout simétrico moviendo a Clyde a la col 16. Cambio innecesario sin ganancia visible.
- **Yes:** Liberación por teleport con `RELEASE_FRAMES` 0/2/4/6 s. Comportamiento idéntico al actual para el orden de salida.
- **No:** Salida caminando por la puerta. Contradice el cierre del pen.
- **Yes:** Eliminar el parámetro `actor` de `isWall`/`canMove`. Queda sin uso; el código honesto vale más que la compatibilidad de firma.
- **Yes:** Actualizar `AGENTS.md`. La regla de "los fantasmas pasan la puerta" queda obsoleta con este cambio.

## Risks

| Riesgo | Mitigación |
| --- | --- |
| Visual: un fantasma persiguiendo cerca del pen ahora "rebota" en la puerta en vez de entrar. | Aceptable y correcto: gira en la celda alineada como con cualquier muro. Sin estado raro. |
| La regla de spec 01 ("los fantasmas pasan la puerta") queda desactualizada. | Se documenta aquí y se actualiza AGENTS.md en el mismo paso. |
| Futura spec de frightened/power-pellets que requiera entrar al pen (animación de retorno). | Se revisará entonces; el cierre del pen se puede relajar en esa spec con lógica de salida. |

## What is **not** in this spec

- Modos chase/scatter/frightened y power-pellets.
- Pathfinding real (BFS/A*).
- Animación de los fantasmas dentro del pen o salida caminando por la puerta.
- Liberación anticipada por dots comidos.
- Nuevas estructuras de datos.