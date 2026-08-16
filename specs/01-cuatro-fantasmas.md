# SPEC 01 — Cuatro fantasmas con comportamientos distintos

> **Status:** Approved
> **Depends on:** —
> **Date:** 2026-08-16
> **Objective:** Reemplazar los 2 fantasmas actuales por 4 (Blinky, Pinky, Inky, Clyde) con un comportamiento distinto cada uno, liberación retardada del pen y un color propio, manteniendo el diseño greedy sin pathfinding.

## Scope

**In:**

- Ampliar `GHOST_STARTS` a 4 entradas con `kind` ∈ `{'blinky','pinky','inky','clyde'}` y posiciones de inicio concretas.
- Reescribir `decideGhost` en `game.js` para ramificar por `kind` en 4 estrategias distintas (greedy Manhattan; apuntar 4 celdas adelante de Pacman; flanqueo usando a Blinky; elección aleatoria entre opciones válidas).
- Lógica de liberación retardada del pen: contador de frames por fantasma; al expirar, teleport a `(13,11)` con `dir='up'`.
- Colores por fantasma en `render.js` (rojo/rosa/cian/naranja).
- Reemplazo del campo antiguo `kind: 'hunter'|'random'` en `GHOST_STARTS`.

**Out of scope (for future specs):**

- Modos chase/scatter/frightened (ciclos).
- Power-pellets y fantasmas comestibles.
- Pathfinding real (BFS/A*) — sigue greedy Manhattan.
- Animación de caminar dentro del pen (los fantasmas esperan quietos).
- Liberación anticipada por conteo de dots.
- Sonidos

## Data model

```js
// maze.js → GHOST_STARTS (reemplaza la versión de 2 entradas)
const GHOST_STARTS = [
  { x: 13, y: 11, kind: 'blinky' }, // afuera, listo para perseguir
  { x: 13, y: 14, kind: 'pinky'  }, // dentro del pen
  { x: 11, y: 14, kind: 'inky'   }, // dentro del pen
  { x: 15, y: 14, kind: 'clyde'  }, // dentro del pen
];
const PEN_EXIT = { x: 13, y: 11 };

// Retardos de liberación en frames a 60fps (0/2/4/6 s)
const RELEASE_FRAMES = {
  blinky: 0,
  pinky:  120,
  inky:   240,
  clyde:  360,
};

// game.js → cada ghost en game.ghosts gana dos campos
//   released: bool   (¿ya salió del pen?)
//   releaseTimer: int (frames restantes antes de salir)
```

## Comportamientos (`decideGhost`, game.js)

- **blinky:** greedy Manhattan hacia `(round(pacman.x), round(pacman.y))`. (Equivalente al `hunter` actual.) Es el "agresivo directo".
- **pinky:** greedy Manhattan hacia la celda 4 adelante de Pacman: `target = (pacman.x + DIRS[pacman.dir].x*4, pacman.y + DIRS[pacman.dir].y*4)`. Sin wrap del túnel en el target (si cae fuera, se topa con la celda del borde).
- **inky:** target = `2*pacman.frontal − blinky.pos`, donde `frontal = pacman + DIRS[pacman.dir]*2`. Greedy Manhattan hacia ese target. Requiere leer la posición del fantasma `kind==='blinky'`.
- **clyde:** elección uniforme entre las opciones válidas (equivalente al `random` actual).

Reglas comunes que se mantienen: filtro de `OPPOSITE` (sin reversa salvo callejón), velocidad `GHOST_SPEED = 0.1`, decisiones solo con `aligned`.

## Liberación del pen (game.js)

- Al crear la partida, cada ghost recibe `released = (kind === 'blinky')` y `releaseTimer = RELEASE_FRAMES[kind]`.
- En `moveGhost`, si `!released`: decrementar `releaseTimer`; si llega a 0, fijar `x=PEN_EXIT.x`, `y=PEN_EXIT.y`, `dir='up'`, `released=true` y **no moverlo este frame**.
- Los fantasmas `released===false` no consumen dots ni causan colisión efectiva mientras esperan (pacman no puede entrar al pen, así que es improbable, pero se deja `isWall` para `pacman` igual).
- `resetPositions` (tras perder una vida) restaura `released` y `releaseTimer` a sus valores iniciales para cada `kind`.

## Implementation plan

1. **maze.js:** reemplazar `GHOST_STARTS` por las 4 entradas y añadir constantes `PEN_EXIT` y `RELEASE_FRAMES`. Exponerlas en `window`. Verificar que el juego sigue cargando.
2. **game.js / `createGame`:** inicializar cada ghost con `released` y `releaseTimer` según `kind`.
3. **game.js / `moveGhost`:** añadir el bloque de liberación retardada (decremento + teleport al expirar).
4. **game.js / `decideGhost`:** reescribir el `if (g.kind === 'hunter') ... else` en 4 ramas (blinky/pinky/inky/clyde). Caso especial: si `!released`, dejar `dir` sin cambios y salir temprano.
5. **game.js / `resetPositions`:** restaurar `released` y `releaseTimer`.
6. **render.js:** tabla de colores por `kind` y aplicarla al dibujar cada ghost.

Cada paso deja el juego funcional y commiteable.

## Acceptance criteria

- [ ] Al abrir `src/index.html` aparecen 4 fantasmas con colores distintos (rojo rosa cian naranja).
- [ ] Blinky persigue directamente a Pacman (greedy Manhattan hacia su celda).
- [ ] Pinky se dirige sistemáticamente 4 celdas adelante de Pacman según su dirección.
- [ ] Inky apunta a un target que depende de la posición de Blinky y de Pacman.
- [ ] Clyde elige direcciones al azar entre las válidas.
- [ ] Al iniciar la partida solo Blinky se mueve; Pinky/Inky/Clyde salen del pen a los ~2/4/6 s.
- [ ] Tras perder una vida, el stagger de liberación se repite correctamente.
- [ ] Ningún fantasma revierte dirección salvo en callejón sin salida.
- [ ] No hay errores en la consola del navegador.

## Decisions

- **Yes:** 4 comportamientos estilo clásico (Blinky/Pinky/Inky/Clyde). Diferenciación táctica clara y coincide con "uno agresivo".
- **No:** 4 niveles de agresividad por velocidad. Menos variedad táctica.
- **Yes:** `kind` con nombres clásicos. Autodescriptivos.
- **No:** `kind` por comportamiento ('hunter'/'predict'/'flank'/'random'). Menos memorables.
- **Yes:** Blinky greedy Manhattan sin pathfinding. Consistente con AGENTS.md ("no pathfinding").
- **No:** BFS hacia Pacman. Rompe el patrón del repo y sube presión excesiva.
- **Yes:** Liberación retardada con temporizadores 0/120/240/360 frames. Ritmo fiel al original.
- **No:** Liberación por conteo de dots. Más complejo y acopla a `dotsRemaining`.
- **Yes:** Teleport a `(13,11)` al liberar. Evita micropathfinding dentro del pen.
- **No:** Caminar dentro del pen hasta la puerta. Sobreingeniería para el alcance.
- **Yes:** Un color por fantasma en render.js. Cambio mínimo y diferenciación visual inmediata.
- **No:** Sprites/formas propias. Fuera de alcance visual en esta spec.

## Risks

| Riesgo | Mitigación |
| --- | --- |
| Inky depende de la posición de Blinky; si Blinky aún no liberó, `inky` calcula target con Blinky dentro del pen (ruido). | Aceptable: el target queda cerca del pen y empuja a Inky a salir. No requiere lógica especial. |
| Resetear `releaseTimer` por vida puede volver la partida lenta tras varias muertes. | El stagger total es de 6 seg, tolerable. Se documenta como decisión; si molesta se baja a 0/2/4 en otra spec. |
| `aligned` con velocidad 0.1 (1/10) alinea cada 10 frames; el teleport del pen puede romper la alineación esperada. | El `PEN_EXIT` usa coords enteras, así que `aligned` es `true` de inmediato. |

## What is **not** in this spec

- Modos chase/scatter/frightened y power-pellets.
- Pathfinding real (BFS/A*).
- Animación de los fantasmas dentro del pen mientras esperan.
- Liberación anticipada por dots comidos.
- Sprites o formas distintas por fantasma.