# Accesibilidad

Objetivo: **WCAG 2.2 AA** en todo componente PARALLAX; AAA en contraste de texto principal.

## Visión
- Contraste texto ≥ 4,5:1 (≥ 3:1 en ≥ 18 px o 14 px negrita); iconos y bordes de controles ≥ 3:1.
- Verificación automática: todo token de `fg` contra todo `bg` permitido, y contra el peor caso de vidrio.
- Modo **alto contraste**: `stroke.*` → `stroke.strong`, sin vidrio, `fg.secondary` → `fg.primary`, anillo de foco 3 px.
- Escalado de texto 100–200 %; escalado de interfaz por output (fraccional).
- Nunca comunicar sólo con color (estados con icono o texto).
- Filtros de color (daltonismo) e inversión en el compositor (shader de post-proceso, coste medido).
- Lupa del compositor (`Super+Alt+=`).

## Movimiento
`reduced_motion` (MOTION.md), `reduced_transparency` (THEMING.md). Nada parpadea > 3 veces/s.

## Teclado
- Todo operable por teclado; orden de tabulación = orden visual.
- Foco visible siempre; nunca se pierde (al cerrar un popover vuelve al disparador).
- Regiones: F6 cicla entre regiones de una ventana; `Super+F10` topbar, `Super+F11` dock.
- Atajos documentados en un único registro (`settings.shortcuts`), sin conflictos (validado en build).
- Teclas lentas / pegajosas / repetición configurables (libinput/xkb).

## Lector de pantalla
- AccessKit en `px-ui` → AT-SPI → Orca. Shell y compositor exponen su árbol (topbar, dock, notificaciones como `alert` vivo).
- Todo IconButton/imagen informativa tiene nombre accesible; decorativo marcado como tal.
- Roles, estados (checked, expanded, busy) y relaciones (labelled-by) correctos.

## Motor y táctil
- Objetivos ≥ 24 × 24 px (shell), ≥ 32 × 32 (apps); sin acciones que requieran doble clic o arrastre sin alternativa.
- Tiempo de auto-cierre de notificaciones ampliable o desactivable.

## Lenguaje
Textos claros, voz activa, errores con causa y solución. Soporte RTL completo (layouts espejados).
