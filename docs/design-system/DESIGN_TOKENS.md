# Tokens de diseño

Nomenclatura: `categoría.rol[.variante]`. Valores para tema **oscuro** (por defecto); el tema claro en THEMING.md.

## Color — superficies
| Token | Valor | Uso |
|---|---|---|
| `bg.base` | `#0E0F13` | fondo de ventana de app |
| `bg.surface` | `#16181E` | tarjetas, barras laterales |
| `bg.elevated` | `#1E2129` | menús, popovers, diálogos |
| `bg.sunken` | `#0A0B0E` | campos de texto, pozos |
| `bg.glass` | `rgba(22,24,30,0.72)` | shell translúcido (con blur, DEPTH_SYSTEM) |
| `bg.glass.fallback` | `#1A1C23` | vidrio sin blur |
| `bg.hover` | `rgba(255,255,255,0.05)` | superposición hover |
| `bg.pressed` | `rgba(255,255,255,0.09)` | superposición presionado |
| `bg.selected` | `accent` @ 18 % | fila/elemento seleccionado |
| `scrim` | `rgba(0,0,0,0.45)` | detrás de modales |

## Color — contenido
| Token | Valor | Contraste sobre `bg.base` |
|---|---|---|
| `fg.primary` | `#ECEDF1` | 15,6:1 |
| `fg.secondary` | `#A3A7B3` | 7,8:1 |
| `fg.tertiary` | `#7F8492` | 4,9:1 (texto ≥ 13 px) |
| `fg.disabled` | `#555A66` | no informativo |
| `fg.on-accent` | `#0E0F13` | ≥ 4,5:1 sobre `accent` |

## Color — acento y semántica
| Token | Valor | Notas |
|---|---|---|
| `accent` | `#7C8CFF` | configurable (THEMING); validado ≥ 4,5:1 sobre `bg.base` |
| `accent.hover` / `accent.pressed` | +8 % / −8 % luminosidad (OKLCH) | derivados |
| `focus.ring` | `accent` | 2 px |
| `success` | `#4CC38A` | |
| `warning` | `#F2B544` | |
| `danger` | `#F2555A` | |
| `info` | `#5AB8F2` | |

## Bordes
| Token | Valor |
|---|---|
| `stroke.subtle` | `rgba(255,255,255,0.06)` |
| `stroke.default` | `rgba(255,255,255,0.10)` |
| `stroke.strong` | `rgba(255,255,255,0.18)` |
| `stroke.glass.highlight` | `rgba(255,255,255,0.08)` borde superior interno 1 px |

## Espaciado (rejilla 4 px)
`space.0`=0 · `1`=4 · `2`=8 · `3`=12 · `4`=16 · `5`=20 · `6`=24 · `8`=32 · `10`=40 · `12`=48

## Forma
| Token | Valor | Uso |
|---|---|---|
| `radius.xs` | 4 | chips, badges |
| `radius.sm` | 6 | botones, campos |
| `radius.md` | 10 | tarjetas, menús |
| `radius.lg` | 14 | topbar, dock, popovers |
| `radius.xl` | 20 | control center, launcher |
| `radius.window` | 12 | ventanas |
| `radius.full` | 9999 | píldoras, avatares |
Regla: radio interior = radio exterior − padding (mínimo `radius.xs`).

## Tamaños
`size.control.sm/md/lg` = 24 / 32 / 40 px · `size.icon.sm/md/lg/xl` = 16 / 20 / 24 / 32 · `size.topbar` = 32 · `size.dock.icon` = 48 (40–72 configurable) · `size.hit.min` = 24 (shell), 32 (apps).

## Densidad
`compact` (×0,875 espaciado y controles), `normal`, `comfortable` (×1,125). Nunca afecta a `size.hit.min`.
