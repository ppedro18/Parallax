# Archivo histórico de auditorías — Fase 0

- **Creado:** 2026-11-19 (Tarea 025).
- **Naturaleza:** archivo histórico de investigación. **No es normativo.** No acepta ADR, no aprueba decisiones y no cierra riesgos.
- **Objetivo:** que la investigación de la Fase 0 pueda consultarse desde cualquier equipo sin depender de los directorios temporales de Claude Code.
- **Registro maestro vigente:** [`../../consolidacion-fase0.md`](../../consolidacion-fase0.md). Este índice lo complementa y no lo sustituye.

## 1. Estado del proyecto

| Elemento | Estado |
|---|---|
| Fase 0 | `IN_PROGRESS` |
| ADR | Todos `Propuesto` |
| Fase 1 | Sin autorización |
| RV-37 | Abierto |
| D-28 y D-31 a D-38 | Pendientes del propietario |
| DEV-01 | Inventario pendiente (no accesible desde la sesión) |
| VM | No creada |
| Pruebas | Ninguna ejecutada |
| Aprobaciones del propietario registradas en este archivo | **Ninguna** |

## 2. Niveles de evidencia que se usan

| Nivel | Significado | Dónde aparece |
|---|---|---|
| Investigación documental | Análisis de los documentos del repositorio | Revisiones 1–10, Tareas 010–011 |
| Código upstream leído con herramienta web | `[V-p]` | Tareas 012–019 |
| Código upstream leído en local, en el tag exacto | `[V]` frente a upstream | Tareas 020–021 |
| Empaquetado de Arch | `[V]` del paquete, sin el sistema instalado | Tarea 017 (addendum) |
| Comportamiento comprobado en ejecución | **Ninguno** | — |
| Pruebas experimentales | **Especificadas, no ejecutadas** | Tareas 021–022 |
| Decisiones aprobadas por el propietario | **Ninguna** | — |

## 3. Índice de tareas

| Tarea | Fecha | Documento | Tema | Veredicto de la propia tarea |
|---|---|---|---|---|
| 001 (revisiones 1–3) | 2026-11-19 | [`../../base-audit.md`](../../base-audit.md), [`-r2`](../../base-audit-r2.md), [`-r3`](../../base-audit-r3.md) | Auditoría base, P-xx | — |
| Revisión 4 | — | **No existe como archivo** (`consolidacion-fase0.md` §7) | C-01 a C-19 | — |
| Revisiones 5–10 | 2026-11-19 | [`../../revisiones/`](../../revisiones/) (r05–r10) | Sesión, lanzamiento, polkit, descriptores, flock | Ver cada informe |
| 002–009 | — | **No hay mapeo documentado** entre estos números de tarea y las revisiones 2–10 | — | — |
| 010 | 2026-11-19 | Cambios en el repositorio: [`../../consolidacion-fase0.md`](../../consolidacion-fase0.md), notas en ADR y documentos. El informe final solo existe en la conversación | Consolidación de las revisiones 1–10 | — |
| 011 | 2026-11-19 | Cambios en `consolidacion-fase0.md` §4.3, `spikes.md` (T-SESSION-FAIL), `arquitectura.md`. El informe final solo existe en la conversación | Corrección de la política X | — |
| 012 | 2026-11-19 | [`tareas-001-023/tarea-012-rv37.md`](tareas-001-023/tarea-012-rv37.md) | RV-37, alternativa E | Resuelto en diseño (corregido después en la 013) |
| 013 | 2026-11-19 | [`tarea-013-rv37-correctiva.md`](tareas-001-023/tarea-013-rv37-correctiva.md) | logind, DRM, evdev | Mitigación parcial |
| 014 | 2026-11-19 | [`tarea-014-greeter-entrada-segura.md`](tareas-001-023/tarea-014-greeter-entrada-segura.md) | M-GRAB, DRM | Viable con limitaciones |
| 015 | 2026-11-19 | [`tarea-015-correctiva-t014.md`](tareas-001-023/tarea-015-correctiva-t014.md) | hotplug, hidraw, uinput, planos | Mitigación parcial |
| 016 | 2026-11-19 | [`tarea-016-udev-arch.md`](tareas-001-023/tarea-016-udev-arch.md) | Reglas udev | No verificable |
| 017 | 2026-11-19 | [`tarea-017-reconciliacion.md`](tareas-001-023/tarea-017-reconciliacion.md) + [addendum](tareas-001-023/tarea-017-addendum-pkgbuild.md) | Empaquetado de Arch, brltty | Verificadas las fuentes de empaquetado; falta la instalación |
| 018 | 2026-11-19 | [`tarea-018-brlapi-brltty-logind.md`](tareas-001-023/tarea-018-brlapi-brltty-logind.md) | BrlAPI, ciclo de vida de logind | Evidencia parcial |
| 019 | 2026-11-19 | [`tarea-019-descriptores-brltty-logind.md`](tareas-001-023/tarea-019-descriptores-brltty-logind.md) | Herencia, G-a′ | Evidencia parcial |
| 020 | 2026-11-19 | [`tarea-020-cierre-evidencia-descriptores-brltty-logind.md`](tareas-001-023/tarea-020-cierre-evidencia-descriptores-brltty-logind.md) | DRM, evdev, GC (lectura local) | Evidencia parcial |
| 021 | 2026-11-19 | [`tarea-021-vfs-drm-uinput-hotplug-pruebas.md`](tareas-001-023/tarea-021-vfs-drm-uinput-hotplug-pruebas.md) | VFS, R1–R3, hotplug, 7 pruebas | Documental suficiente; pruebas preparadas |
| 022 | 2026-11-19 | [`tarea-022-protocolo-experimental-entorno-linux.md`](tareas-001-023/tarea-022-protocolo-experimental-entorno-linux.md) | Protocolo y entorno | Validado documentalmente; entorno pendiente |
| 023 / 023-B | 2026-11-19 | [`tarea-023-inventario-dev01-viabilidad-kvm.md`](tareas-001-023/tarea-023-inventario-dev01-viabilidad-kvm.md) | DEV-01 | No accesible; inventario pendiente |
| 024 | — | **No hay ningún informe con ese número** | — | — |
| 025 | 2026-11-19 | Este archivo y `manifests/` | Consolidación permanente | — |

**Correcciones entre tareas:** se conservan en los propios informes. Hay que leer siempre el informe más reciente sobre cada tema. Las principales:
- La 013 corrige a la 012.
- La 015 corrige a la 014.
- Las 016 y 017 corrigen premisas de udev de las 013–015.
- La 017 (addendum) y la 018 refutan que `KillUserProcesses=yes` sea el valor por defecto en Arch.
- La 019 matiza G-a′, que había propuesto la 018.

## 4. Evidencias

| Carpeta | Contenido |
|---|---|
| [`evidencias/arch-pkg/systemd-262.2-1/`](evidencias/arch-pkg/systemd-262.2-1/) | PKGBUILD, `.SRCINFO`, el parche de grupos de dispositivos y las opciones de meson (`-Ddefault-kill-user-processes=false`) |
| [`evidencias/arch-pkg/brltty-6.9.1-3/`](evidencias/arch-pkg/brltty-6.9.1-3/) | PKGBUILD y los parches de sysusers, grupos y usuario bloqueado |
| [`evidencias/registros/`](evidencias/registros/) | Registro del clonado de los repositorios de empaquetado |

RV-37, DRM, evdev, logind, greetd, systemd y BrlTTY se tratan dentro de los informes 012–022. Cada afirmación lleva su marca de evidencia.

## 5. Fuentes externas

Ver [`fuentes/sources-manifest.md`](fuentes/sources-manifest.md): URL, tag, commit, rutas consultadas, licencias y cómo reproducir cada consulta.

## 6. Decisiones pendientes del propietario

| Decisión | Tema | Origen |
|---|---|---|
| D-01 a D-30 | — | Registro en `consolidacion-fase0.md` §4 |
| D-31 | Linger, `KillUserProcesses`, `KillOnlyUsers` | 012, 018 |
| D-32 | Recuperación y timeouts del greeter | 013, 015 |
| D-33 | Campo de credenciales ante hotplug | 014, 015 |
| D-34 | hidraw | 014–017 |
| D-35 | Asistencia con grab previo, uinput | 014–018 |
| D-36 | brltty en la imagen base | 017 |
| D-37 | Política de sesión | 018 |
| D-38 | Herramientas de prueba desechables | 022 |
| Autorizaciones | VM, ejecución de pruebas, acceso de lectura a DEV-01, hardware físico, riesgos que bloquean Gate 0 | 022 |

**Ninguna de estas decisiones figura todavía en `consolidacion-fase0.md` §4.**

## 7. Riesgos abiertos, propuestos en las tareas

RV-37, RV-38, RV-43, RV-44, RV-46, RV-47, RV-48, RV-49, RV-50, RV-51, RV-52, RV-53, RV-54, RV-55 y RV-56.

`consolidacion-fase0.md` §5 solo registra hasta RV-37. **RV-38 a RV-56 se proponen en los informes y no están en el registro formal.**

## 8. Pruebas propuestas y no ejecutadas

| Origen | Pruebas |
|---|---|
| `spikes.md` §T | T-FD, T-LOCK, T-GLIBC, T-PIDS, T-PIDFD, T-TRANS, T-GRAFO, T-DUMP, T-GREETD, T-KILL, T-POLKIT, T-DBUSACT, T-SESSION-FAIL |
| Tareas 013–022 | T-DRM, T-BIND, T-GRAB, T-INPUT-COVERAGE, T-GREETER-SECURITY, T-ACCESSIBILITY, T-HOTPLUG-RACE, T-HIDRAW, T-UINPUT-GRAB, T-DRM-MAPPING, T-UDEV-*, T-BRLAPI, T-BRLTTY-ACT, T-BRLTTY-CAPS, T-LOGIND-GC, T-KILL-ARCH, T-GA-PRIME |
| Tareas 021–022 (especificación más reciente) | T-FD-INHERIT, T-LOGIND-SHARED-REVOKE, T-LOGIND-GC, T-BRLTTY-PASTE, T-TIOCSTI, T-UINPUT-HOTPLUG, T-DRM-INHERITED-IOCTL |

Ninguna está autorizada ni se ha ejecutado.

## 9. Integridad

[`manifests/transfer-manifest.md`](manifests/transfer-manifest.md) registra el origen, el tamaño y el SHA-256 de cada archivo copiado, y lo que se inspeccionó sin copiar.
