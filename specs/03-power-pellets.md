# SPEC 03 — Power pellets en las esquinas: fantasmas comestibles

> **Status:** Approved
> **Depends on:** SPEC 01, SPEC 02
> **Date:** 2026-08-16
> **Objective:** Añadir 4 power pellets en las esquinas del laberinto que, al ser comidos, vuelven comestibles a los fantasmas liberados durante 8 segundos, permitiendo a Pacman comerlos para sumar puntos.

## Scope

**In:**

- 4 power pellets en las celdas clásicas de las esquinas: `(1,3)`, `(26,3)`, `(1,23)`, `(26,23)`.
- Nuevo tile `4` en el grid (se edita en `MAZE` y se come como un dot).
- Al comer un pellet: +50 puntos, efecto "frightened" de 480 frames (8 s) sobre todos los fantasmas liberados.
- Fantasmas comestibles: color azul (parpadeo blanco/azul los últimos 120 frames), movimiento aleatorio a velocidad reducida (0.05).
- Comer un fantasma comestible: +200/400/800/1600 puntos (doble por cada fantasma dentro del mismo efecto), el fantasma vuelve al pen y se relanza con su `RELEASE_FRAMES` original.
- Periodo de gracia de 10 frames tras comer un fantasma (evita el bug clásico de solape).
- Los pellets cuentan en `dotsRemaining` para la condición de victoria.
- Actualizar `AGENTS.md`.

**Out of scope (for future specs):**

- Modos chase/scatter (ciclos) y flash de modo al comerse el último dot.
- Retorno del fantasma comido como "ojos" caminando hasta el pen.
- Sonidos.
- Power pellets que respawn tras ser comidos.
- Diferentes duraciones/velocidades según nivel.

## Data model

```js
// maze.js → parseTile gana 'o' → 4. En MAZE_STR, las 4 esquinas pasan de '.' a 'o':
//   fila 3 : '#o####.#####.##.#####.####o#'
//   fila 23: '#o..##................##..o#'
const POWER_PELLETS = [
  { x: 1,  y: 3  },
  { x: 26, y: 3  },
  { x: 1,  y: 23 },
  { x: 26, y: 23 },
];

// game.js → nuevas constantes
const FRIGHT_FRAMES = 480;          // 8 s a 60fps
const GHOST_FRIGHT_SPEED = 0.05;    // mitad de GHOST_SPEED
const GRACE_FRAMES = 10;            // invulnerabilidad post-comida
const GHOST_EAT_BASE = 200;         // primer fantasma comido

// game.js → estado nuevo por partida
//   frightTimer: int  (frames restantes de efecto; 0 = sin efecto)
//   combo:       int  (fantasmas comidos en el efecto actual; se resetea por pellet)
//   graceFrames: int  (frames de invulnerabilidad tras comer un fantasma)
```

Un fantasma es **comestible** cuando `g.released === true && game.frightTimer > 0`. No se añade flag por fantasma: el temporizador es la única fuente de verdad.

## Comportamiento (game.js)

- **`createGame`:** contar `v === 2 || v === 4` en `dotsRemaining`. Inicializar `frightTimer: 0`, `combo: 0`, `graceFrames: 0`.
- **`movePacman`:** al estar sobre una celda `4`: ponerla a `0`, +50 puntos, `game.frightTimer = FRIGHT_FRAMES`, `game.combo = 0`, `game.dotsRemaining--`. Comer un pellet con el efecto activo reinicia el temporizador completo (clásico).
- **`decideGhost`:** si el fantasma es comestible, elegir dirección aleatoria entre las opciones válidas (misma rama que `clyde`) y salir temprano, antes de los branches por `kind`.
- **`moveGhost`:** usar `GHOST_FRIGHT_SPEED` (0.05) mientras el fantasma sea comestible.
- **`update`:** decrementar `frightTimer` (al llegar a 0 el efecto termina) y `graceFrames` cada frame.
- **Colisión en `update`:** si `collides` con un fantasma liberado:
  - Comestible: +`GHOST_EAT_BASE * 2 ** combo` (200/400/800/1600), `combo++`, teletransportar el fantasma a su posición en `GHOST_STARTS`, `released = false`, `releaseTimer = RELEASE_FRAMES[kind]`, `graceFrames = GRACE_FRAMES`. El fantasma del pen se relanza con su orden original (0/2/4/6 s).
  - No comestible y `graceFrames <= 0`: perder vida y resetear posiciones como ahora.

Un fantasma que se re-libera durante un efecto activo sale **normal** (no comestible). Coherente con el clásico: solo los liberados en el momento del pellet se asustan.

## Implementation plan

1. **maze.js:** añadir `'o' → 4` en `parseTile`, cambiar las 4 esquinas en `MAZE_STR`, añadir `POWER_PELLETS` y exponerla en `window`. Verificar que el juego carga (los pellets se siguen dibujando como dots pequeños).
2. **game.js:** constantes nuevas, `createGame` (conteo 2|4, campos nuevos), comer pellet en `movePacman`, decremento de temporizadores y colisión comestible en `update`, rama aleatoria en `decideGhost`, velocidad reducida en `moveGhost`.
3. **render.js:** dibujar los pellets (valor 4) como círculo grande (r=7); color azul `#2121ff` para fantasmas liberados comestibles, con parpadeo blanco/azul cada 10 frames cuando `frightTimer < 120`.
4. **AGENTS.md:** documentar el tile 4, el estado de juego (`frightTimer`/`combo`/`graceFrames`) y las reglas de movimiento asustado.
5. Verificación manual (criterios de aceptación).

Cada paso deja el juego funcional y commiteable.

## Acceptance criteria

- [ ] Al abrir `src/index.html` se ven 4 pellets grandes en las esquinas `(1,3)`, `(26,3)`, `(1,23)`, `(26,23)`.
- [ ] Comer un pellet suma 50 puntos y los fantasmas liberados se vuelven azules.
- [ ] Los fantasmas azules se mueven al azar y a la mitad de velocidad (0.05).
- [ ] Comer un fantasma azul suma 200 puntos; el siguiente dentro del mismo efecto 400, luego 800 y 1600.
- [ ] El fantasma comido desaparece y reaparece en el pen, saliendo de nuevo con su orden (0/2/4/6 s).
- [ ] A los 8 s el efecto termina: los fantasmas recuperan su color y vuelven a matar.
- [ ] Los últimos 2 s del efecto los fantasmas parpadean blanco/azul.
- [ ] Comer otro pellet durante el efecto reinicia el temporizador a 8 s.
- [ ] Tras comer un fantasma, un solape inmediato con otro fantasma normal no cuesta vida (gracia de 10 frames).
- [ ] Sin comer los 4 pellets la partida no gana (`dotsRemaining` nunca llega a 0).
- [ ] No hay errores en la consola del navegador.

## Decisions

- **Yes:** Tile `4` en el grid (se come como un dot). Coherente con `MAZE` como fuente única de verdad, igual que los dots.
- **No:** Lista de coordenadas de pellets separada del grid. Duplica estado; el grid ya registra lo comido.
- **Yes:** Pellets en las celdas clásicas `(1,3)`, `(26,3)`, `(1,23)`, `(26,23)`. Posiciones simétricas, ya existentes como dots.
- **Yes:** `FRIGHT_FRAMES = 480` (8 s) y parpadeo los últimos 120 frames (2 s). Ritmo clásico y observable.
- **Yes:** Velocidad reducida a 0.05 mientras son comestibles. Fiel al clásico y da a Pacman la ventaja táctica.
- **No:** Huir de Pacman (Manhattan inverso). Menos clásico y más difícil de leer visualmente.
- **Yes:** Puntos 200/400/800/1600 con `combo` reseteado por cada pellet. Fiel al original, costo mínimo.
- **No:** Puntos fijos por fantasma comido. Pierde la emoción de encadenar.
- **Yes:** Gracia de 10 frames tras comer un fantasma. Mitiga el bug clásico de que el fantasma siguiente mate en el mismo frame.
- **Yes:** Fantasma comido → teleport al pen y relanzamiento con `RELEASE_FRAMES`. Reutiliza el mecanismo de SPEC 01 y respeta el pen sellado de SPEC 02.
- **No:** Retorno como "ojos" caminando. Requiere pathfinding y contradice el pen sellado (SPEC 02).
- **Yes:** Solo los fantasmas liberados se pintan azules; los del pen conservan su color (no son alcanzables).
- **No:** Los fantasmas del pen se vuelven azules y parpadean. Coste visual sin beneficio de juego.
- **Yes:** Los pellets cuentan en `dotsRemaining`. La condición de victoria sigue siendo única y fiel al clásico.
- **Yes:** Comer pellet con efecto activo reinicia el temporizador completo. Comportamiento clásico.
- **No:** Fantasma re-liberado durante un efecto sale comestible. Contradice el clásico y haría el juego demasiado fácil.

## Risks

| Riesgo | Mitigación |
| --- | --- |
| Bug clásico de solape: al comer un fantasma, otro superpuesto mata al instante. | `graceFrames = 10` tras cada comida. |
| Fantasma re-liberado durante un efecto activo sale con su color normal mientras los demás están azules. | Decisión consciente y fiel al clásico; se documenta en la sección de decisiones. |
| Velocidad 0.05 (1/20) alinea cada 20 frames; decisiones de giro más espaciadas. | Aceptable: los fantasmas asustados giran menos, que es exactamente el efecto buscado. |
| Comer pellet con efecto activo y fantasmas ya comidos: el combo se resetea, perdiendo el multiplicador. | Clásico; el jugador decide cuándo activar el siguiente pellet. |

## What is **not** in this spec

- Modos chase/scatter (ciclos) y flash de modo al comerse el último dot.
- Retorno del fantasma comido como "ojos" caminando hasta el pen.
- Sonidos.
- Power pellets que respawn tras ser comidos.
- Diferentes duraciones/velocidades según nivel.

Cada uno de esos, si llega, va en su propia spec.