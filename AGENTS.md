# AGENTS.md — Reglas globales de PARALLAX OS

Aplican a todo agente (humano o IA) que trabaje en este repositorio. Prevalecen sobre las instrucciones de cada agente en `.claude/agents/`.

## Estado del proyecto
**Fase 0 — Arquitectura y especificación.** No se escribe código de producción hasta que el Arquitecto Principal y el responsable del proyecto aprueben `docs/architecture/` y los ADR en estado `Propuesto` pasen a `Aceptado`.

## Reglas absolutas
1. Todo componente propio de espacio de usuario se escribe en Rust (edición 2024, MSRV fijado en `rust-toolchain.toml`). Excepciones solo vía ADR.
2. Prohibido: Electron, Python/JS/TS en componentes centrales, aplicaciones de GNOME/KDE como sustitutos, colecciones de scripts como producto.
3. No presentar código de demostración como funcionalidad terminada. Una función está *terminada* solo si cumple su criterio de aceptación (`docs/specifications/*` §Aceptación) y QA lo verificó.
4. No modificar el sistema anfitrión del desarrollador. Todo build/test ocurre en el workspace, en VM o contenedor.
5. No instalar ni añadir dependencias sin la entrada correspondiente en `docs/security/dependencias.md` y aprobación (ver ADR-0008).
6. No ejecutar operaciones destructivas ni descargas en tiempo de ejecución del sistema.
7. No duplicar lo que Linux ya resuelve (scheduler, OOM killer, gestión de dispositivos, logind).
8. Ningún proceso gráfico corre con privilegios de root. La elevación pasa por polkit con acciones granulares.

## Propiedad de componentes (un dueño por área)
| Área | Dueño | Revisa |
|---|---|---|
| Contratos, límites de capas, ADR (autoridad, no implementación) | arquitecto-sistemas | todos |
| Workspace Rust, crates comunes (`px-*`) | ingeniero-rust | arquitecto |
| Integración Linux, servicios de sistema y de sesión (systemd, D-Bus, PAM, udev) | ingeniero-linux | seguridad |
| Compositor y WM (`core/`) | ingeniero-compositor | rendimiento |
| Especificación visual y de interacción (`docs/design-system/`) | disenador-producto | framework-ui |
| Toolkit UI (`px-ui`) | ingeniero-framework-ui | disenador, rendimiento |
| Hardware (`services/hardware`) | ingeniero-hardware | linux |
| Resource Intelligence (`intelligence/`), dentro de ADR-0014 | ingeniero-recursos | linux, seguridad, rendimiento |
| Shell, bloqueo, greeter (`shell/`) | ingeniero-shell | disenador, seguridad |
| Aplicaciones nativas (`applications/`), un responsable por app | ingeniero-aplicaciones | disenador |
| Seguridad, autorización (ADR-0015), permisos, firmas | ingeniero-seguridad | arquitecto |
| Benchmarks y presupuestos | ingeniero-rendimiento | qa |
| Validación y releases (puede rechazar entregas; no aprueba arquitectura) | qa-sistemas | — |

Dos agentes no modifican el mismo componente a la vez. Cambios cruzados: el dueño implementa, el otro revisa.

## Cambios arquitectónicos
Cualquier cambio en: contratos IPC, límites entre capas, selección tecnológica, modelo de seguridad o dependencias nuevas → ADR en `docs/decisions/` (plantilla `ADR-0000-plantilla.md`), aprobado por arquitecto-sistemas antes de implementar.

## Informe obligatorio de cada agente al terminar una tarea
```
Analizado:     …
Decidido:      … (ADR si aplica)
Archivos:      …
Dependencias:  … (o "ninguna")
Riesgos:       … (añadidos a docs/architecture/riesgos.md)
Pendiente:     …
```

## Convenciones
- Idioma de documentación: español. Identificadores de código, crates y protocolos: inglés.
- Crates internos: prefijo `px-`. Binarios: `parallax-<nombre>`. Bus D-Bus: `org.parallax.<Servicio>1`.
- Errores: `thiserror` en librerías, `anyhow` solo en binarios. Sin `unwrap()` fuera de tests.
- Observabilidad: `tracing` en todo proceso; journald como sumidero.
- `unsafe` requiere comentario `// SAFETY:` y revisión de ingeniero-rust.

## Mapa de documentación
- `docs/vision/` — visión y alcance
- `docs/architecture/` — capas, componentes, IPC, eventos, servicios, riesgos
- `docs/specifications/` — especificación funcional y técnica por componente
- `docs/design-system/` — sistema de diseño (índice en README.md)
- `docs/security/` — modelo de seguridad, dependencias
- `docs/benchmarks/` — metodología y presupuestos
- `docs/roadmap/` — fases, pruebas, criterios de aceptación
- `docs/decisions/` — ADR
