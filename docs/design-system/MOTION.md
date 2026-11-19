# Movimiento

## Tokens
| Token | Duración | Curva | Uso |
|---|---|---|---|
| `motion.instant` | 80 ms | linear | hover, color |
| `motion.fast` | 120 ms | `cubic-bezier(0.2, 0, 0, 1)` | menús, tooltips, toggles |
| `motion.base` | 200 ms | `cubic-bezier(0.2, 0, 0, 1)` | popovers, paneles, apertura de ventana |
| `motion.slow` | 320 ms | spring (rigidez 300, amortiguación 30, masa 1) | workspaces, overview, minimizar |
| `motion.exit` | 0,7 × entrada | `cubic-bezier(0.4, 0, 1, 1)` | toda salida |

## Reglas
1. Solo `transform` y `opacity` (y radio de blur del overview). Nunca animar layout (tamaño de contenido re-layouteado por frame).
2. Toda animación es **interrumpible** y continúa desde el estado actual (velocidad conservada en springs).
3. Ninguna animación bloquea input: el usuario puede actuar durante ella.
4. Salidas más rápidas que entradas.
5. El input directo (arrastre, gesto de touchpad) sigue al dedo 1:1, sin curva; al soltar, spring con la velocidad del gesto.
6. Máximo una animación "protagonista" a la vez.

## Coreografía del sistema
| Evento | Animación |
|---|---|
| Abrir ventana | escala 0,94 → 1 + opacidad 0 → 1, `motion.base` |
| Cerrar ventana | escala 1 → 0,96 + opacidad → 0, `motion.exit` (≈140 ms) |
| Minimizar | traslación + escala hacia el icono del dock, `motion.slow` |
| Maximizar / snap | interpolación de geometría con snapshot de textura (sin re-render del cliente por frame) |
| Workspace | deslizamiento horizontal + parallax del fondo, `motion.slow` |
| Overview | spring, ventanas a sus posiciones en rejilla |
| Launcher / Control Center | opacidad + traslación 8 px + escala 0,98, `motion.base` |
| Notificación | entrada desde la derecha 16 px, `motion.base`; apilado con spring |
| Dock rebote (lanzando) | 2 rebotes de 8 px, se detiene al mapear la ventana |

## Movimiento reducido
`accessibility.reduced_motion = true`: traslaciones/escalas → crossfade de 80 ms; sin parallax; sin rebote; spinners siguen girando (indican actividad). También se activa automáticamente si el compositor detecta frames perdidos sostenidos (RENDERING_PERFORMANCE).
