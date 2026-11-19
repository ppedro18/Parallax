# Sistema de profundidad

La identidad de PARALLAX: el sistema se organiza en **planos** a distintas profundidades. La profundidad comunica jerarquía y temporalidad (lo más alto es lo más efímero).

## Planos (de fondo a frente)
| Plano | z | Contenido | Capa layer-shell |
|---|---|---|---|
| P0 Fondo | 0 | wallpaper | `background` |
| P1 Escritorio | 1 | widgets anclados | `bottom` |
| P2 Ventanas | 2 | toplevels (floating/tiled) | — |
| P3 Shell persistente | 3 | topbar, dock | `top` |
| P4 Shell transitorio | 4 | launcher, control center, popovers, menús | `top`/`overlay` |
| P5 Notificaciones | 5 | banners | `overlay` |
| P6 Sistema crítico | 6 | polkit, bloqueo, overlay de emergencia | `overlay` exclusivo / session-lock |

## Elevación
| Nivel | Uso | Sombra (y, blur, opacidad) | Borde |
|---|---|---|---|
| `elev.0` | plano, contenido | ninguna | — |
| `elev.1` | tarjetas, widgets | 0 1 2 /30 % + 0 2 6 /15 % | `stroke.subtle` |
| `elev.2` | ventana inactiva, topbar, dock | 0 4 12 /35 % | `stroke.subtle` |
| `elev.3` | ventana activa, popovers | 0 8 24 /40 % | `stroke.default` |
| `elev.4` | launcher, control center | 0 16 40 /45 % | `stroke.default` |
| `elev.5` | modales, polkit | 0 24 48 /50 % + `scrim` | `stroke.strong` |
Sombras siempre negras y verticales (luz cenital única). Implementación: 9-slice precalculado por (radio, nivel, escala), nunca blur gaussiano en tiempo real.

## Vidrio (translucidez)
Solo en P3–P5. Receta: blur del contenido inferior (radio 24 px lógicos, dual-kawase, 1/4 resolución) + `bg.glass` + saturación 1,2 + `stroke.glass.highlight` superior.
- Nunca en contenido de apps (P2) ni en texto largo.
- Contraste del texto sobre vidrio se verifica contra el peor fondo (blanco puro tras el blur) → el tinte de `bg.glass` garantiza ≥ 4,5:1.
- Fallback sin blur: `bg.glass.fallback` opaco (RENDERING_PERFORMANCE define cuándo).

## Efecto parallax
Desplazamiento sutil de planos para reforzar profundidad, solo en transiciones (nunca continuo con el ratón, que costaría frames y cansa):
- Cambio de workspace: P0 se desplaza al 10 % de la distancia de P2.
- Overview (`Super`): P2 se reduce a planos con escala 0,85 y separación en z; P0 se oscurece 30 %.
- Apertura de P4: el contenido de P2 se atenúa 0 → 8 % (sin blur adicional).
`reduced_motion` → sin desplazamiento, solo crossfade.

## Atenuación de inactivos
Ventana sin foco: sombra `elev.2`, contenido sin cambios (no se oscurece: legibilidad). Decoraciones de servidor con `fg.tertiary`.
