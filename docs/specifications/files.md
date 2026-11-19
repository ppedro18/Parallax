# Especificación: PARALLAX Files

## Arquitectura
- UI (`px-ui`) + núcleo `px-fileops` (librería, testeable sin UI) + motor de operaciones en hilo aparte (cola persistida).
- Listado: `rustix` `getdents` en hilo de E/S, entradas en lotes de 256 para no bloquear la UI; watch con inotify (crate `notify`).
- Miniaturas: estándar freedesktop thumbnail spec (`~/.cache/thumbnails`), generadas en subproceso aislado (`parallax-thumbnailer`, seccomp, sin red) para imágenes (`image`), SVG (`resvg`), PDF (evaluar `pdfium`/`mupdf` como excepción C documentada).
- Papelera: freedesktop Trash spec (compatible con otras apps).
- Montaje: udisks2 vía D-Bus (sin root).
- MIME: shared-mime-info database + `mimeapps.list`.

## Funcionalidades y comportamiento
| Función | Detalle |
|---|---|
| Navegación | barra de ruta editable/breadcrumbs, atrás/adelante/arriba, historial por pestaña |
| Pestañas | `Ctrl+T/W`, reordenables, restaurables al reabrir |
| Vistas | iconos, lista, detalles (columnas configurables, orden), por carpeta recordada |
| Vista previa | `Espacio` (imágenes, texto, PDF, audio/video vía metadatos) |
| Búsqueda | en carpeta actual: nombre (incremental); recursiva en hilo con cancelación; contenido: v1.x |
| Filtros | tipo, tamaño, fecha, ocultos |
| Favoritos | barra lateral, arrastrables |
| Operaciones | copiar, mover, renombrar (F2, masivo con patrón), eliminar → papelera, eliminar permanente (`Shift+Supr`, confirmación) |
| Drag & drop | interno y con otras apps (`wl_data_device`, `text/uri-list`) |
| Progreso | panel de operaciones: velocidad, ETA, pausar, cancelar |
| Conflictos | diálogo: reemplazar, omitir, mantener ambos, renombrar, "aplicar a todos"; comparación de tamaño/fecha |
| Permisos | propiedades: owner, grupo, modo rwx; cambios de archivos ajenos → polkit vía helper |
| Propiedades | tamaño (cálculo recursivo cancelable), fechas, tipo, abrir con |
| Dispositivos | lista de udisks2, montar/desmontar/expulsar seguro con estado "ocupado" explicando qué proceso lo usa |

## Motor de operaciones
- Cada operación = `Job { id, kind, sources, dest, state, progress }`. Estados: `Queued → Running ⇄ Paused → (Completed|Failed|Cancelled)`.
- Copia: `copy_file_range` / reflink cuando sea posible; escritura a archivo temporal + `rename` para no dejar archivos parciales; preserva mtime/permisos/xattrs.
- Mover en el mismo FS = `renameat2`; entre FS = copiar + verificar + borrar.
- Cola persistida: si Files cae durante una copia, al reabrir ofrece reanudar.

## Errores
Permiso denegado, disco lleno (pausa y avisa), dispositivo retirado (falla limpio, lista lo completado), nombre inválido, enlace roto. Ningún error deja datos parciales con el nombre final.

## Aceptación
- Copiar 100 000 archivos pequeños y 1 archivo de 20 GB con verificación de hash.
- UI responde (< 50 ms) listando carpeta de 100 000 entradas.
- Matar Files a mitad de copia → sin archivos corruptos con nombre final.
