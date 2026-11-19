# Tipografía

## Familias
| Rol | Familia | Licencia | Fallback |
|---|---|---|---|
| UI | Inter (variable) | OFL-1.1 | Noto Sans |
| Mono | JetBrains Mono | OFL-1.1 | Noto Sans Mono |
| CJK / scripts | Noto Sans CJK / Noto Sans <script> | OFL-1.1 | — |
| Emoji | Noto Color Emoji | OFL-1.1 | — |
Fuentes empaquetadas con el sistema (sin descarga). Una tipografía propia se evalúa en v2.

## Escala
| Token | Tamaño / interlínea | Peso | Uso |
|---|---|---|---|
| `text.caption` | 11 / 14 | 500 | etiquetas, badges |
| `text.footnote` | 12 / 16 | 400 | metadatos |
| `text.body` | 13 / 18 | 400 | texto por defecto |
| `text.body.strong` | 13 / 18 | 600 | énfasis |
| `text.callout` | 15 / 20 | 500 | controles grandes, launcher |
| `text.title3` | 18 / 24 | 600 | cabeceras de sección |
| `text.title2` | 22 / 28 | 600 | títulos de página |
| `text.title1` | 28 / 34 | 700 | onboarding, widgets grandes |
| `text.display` | 36 / 40 | 300 | reloj de bloqueo y widget Clock |
| `text.mono` | 13 / 18 | 400 | terminal, código |
Tracking: −1 % a partir de 18 px; +1 % en `caption`.

## Reglas
- Cifras tabulares (`tnum`) en relojes, métricas, tamaños, tablas.
- Máx. 3 tamaños por vista. Longitud de línea 45–90 caracteres.
- Truncado con elipsis al final; nombres de archivo con elipsis en medio (`informe…final.pdf`); tooltip con texto completo.
- Escalado de texto del usuario 100–200 % sin truncar contenido esencial (layouts fluidos).

## Render
cosmic-text (shaping HarfBuzz vía rustybuzz, bidi, fallback). Antialias en escala de grises (sin subpixel: incompatible con transformaciones y escalas fraccionales). Hinting ligero. Atlas de glifos compartido por proceso.
