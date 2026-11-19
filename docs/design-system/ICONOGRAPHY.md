# Iconografía

## Iconos de sistema (simbólicos)
- Rejilla 24 × 24, área viva 20 × 20, trazo 1,5 px, extremos y uniones redondeados, esquinas 2 px.
- Tamaños de render: 16, 20, 24, 32 (diseñados a 24, ajustados a 16 a mano cuando el escalado pierda legibilidad).
- Monocromos: se colorean con `fg.*` o `accent` en tiempo de ejecución (SVG con `currentColor`).
- Variantes: `regular` y `filled` (para estados activos: toggle activado, pestaña seleccionada).
- Formato SVG, render con `resvg`, cache rasterizado por (icono, tamaño, escala, color).
- Nomenclatura freedesktop (`network-wireless-signal-good-symbolic`) para compatibilidad con temas y apps de terceros.

## Iconos de aplicación
- Forma "squircle" (superelipse n = 5) 1024 base, con una capa de profundidad: plano frontal con glifo + plano trasero desplazado 2 % hacia abajo con tono más oscuro.
- Sombra integrada no; la sombra la aplica el dock (`elev.1`).
- Apps de terceros sin icono conforme: se muestran tal cual sobre una placa squircle `bg.elevated` si no son cuadradas (opción, por defecto activada).

## Set inicial obligatorio (v1)
Navegación (atrás, adelante, arriba, menú, cerrar, más), archivo (carpeta, documento, imagen, audio, video, archivo comprimido, código), acciones (copiar, cortar, pegar, borrar, renombrar, buscar, filtrar, compartir), sistema (red ×5 niveles, bluetooth, batería ×10 + carga, volumen ×4, brillo, notificaciones, DND, energía, bloqueo, ajustes), ventanas (cerrar, minimizar, maximizar, restaurar, flotar, tiling), estado (éxito, aviso, error, info, cargando).

## Licencia
Set propio bajo CC-BY-SA-4.0. Prohibido incorporar iconos de Caffyne, Apple (SF Symbols) o terceros sin licencia compatible. Fallback: tema `hicolor`/Adwaita simbólicos solo para iconos que el set no cubra, registrado en `security/dependencias.md`.
