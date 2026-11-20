# Revisión de Fase 0 — Arquitectura y especificación

- **Estado de la fase:** `IN_PROGRESS` (no aprobada).
- **Última actualización:** 2026-11-19.
- **Fuentes:** auditoría de base, revisiones [1](../project-management/base-audit.md), [2](../project-management/base-audit-r2.md) y [3](../project-management/base-audit-r3.md) (vigente).
- **Naturaleza:** documento descriptivo de seguimiento. No acepta ADR, no cambia estados y no sustituye la decisión del propietario (`roadmap.md §7`).

> Historial: hasta 2026-11-19 este archivo contenía por error una copia íntegra de `decisions/ADR-0009-actualizaciones.md` (hallazgo P-01). ADR-0009 sigue siendo la única fuente de esa decisión.

## 1. Resumen

La documentación de Fase 0 cubre visión, arquitectura por capas, IPC, eventos, servicios, hardware, actualizaciones y recuperación, riesgos, sistema de diseño, 13 especificaciones, benchmarks, estrategia de pruebas y 15 ADR. Ningún ADR está aceptado. La fase **no puede cerrarse** todavía; los bloqueos están en §4.

## 2. Estado de los ADR

Todos en estado `Propuesto`. Clasificación propuesta para Gate 0 (base-audit-r3 §7), pendiente de aprobación del propietario:

| ADR | Tema | Grupo propuesto | Pendiente principal |
|---|---|---|---|
| 0001 | Rust nativo | Necesario para cerrar Fase 0 | Aprobación |
| 0002 | Base Arch (snapshot ALA) | Necesario | Arquitectura de destino; P-02 |
| 0007 | IPC híbrido | Necesario (principio) | Contratos XML y benchmarks como condiciones |
| 0008 | Licencias (§14, política L1) | Necesario | Aprobación; DCO/CLA y `px-ui` separados |
| 0013 | Entorno de desarrollo | Necesario | Reescritura (P-03) y decisión P-02 |
| 0015 | Autorización Wayland | Necesario (seguridad) | Reglas normativas fuera de agentes (P-12) |
| 0003 | Smithay | Aceptable con spike en Fase 1 | Spike winit + DRM/KMS |
| 0004 | WM en compositor | Aceptable con spike | Prototipo `px-wm` |
| 0005 | Toolkit UI | Gate 1 | Benchmark |
| 0009 | Actualizaciones | Aceptable con spike | `storage.md`; spike Btrfs + systemd-boot |
| 0006 | Shell un proceso | Diferible (Fase 3) | — |
| 0010 | Agente polkit | Diferible (revisar antes de Gate 2) | — |
| 0011 | Terminal | Diferible (Fase 4) | Spike |
| 0012 | Xwayland | Diferible (Fase 2) | Validación |
| 0014 | Delegación cgroups | Diferible (Fase 5, modo observación) | P-12 |

## 3. Criterios de aceptación de Fase 0 (`roadmap.md`)

| Criterio | Estado | Evidencia / pendiente |
|---|---|---|
| Todos los ADR iniciales están revisados | Parcial | Revisados en la auditoría; sin aprobaciones registradas |
| Decisiones pendientes identificadas | Sí | base-audit-r3 §9–§10 |
| Componentes y responsabilidades definidos | Sí | `architecture/arquitectura.md §2`, AGENTS.md |
| Interfaces entre subsistemas documentadas | Parcial | `ipc.md`, `eventos.md`; faltan `architecture/dbus/*.xml` (P-06) |
| Especificación de rendimiento | Sí | `design-system/RENDERING_PERFORMANCE.md`, `benchmarks/metodologia.md` |
| Metodología de benchmark | Sí | `benchmarks/metodologia.md`; hardware sin identificar (P-21) |
| Estrategia de pruebas | Sí | `roadmap/estrategia-pruebas.md` |
| Agentes con instrucciones y límites | Sí, con reserva | `.claude/agents/`; contienen reglas normativas que deberían estar en `docs/` (P-12) |

Las 14 verificaciones transversales de `criterios-aceptacion.md` siguen sin marcar.

## 4. Bloqueos de cierre

1. Aceptación de la política de licencias (ADR-0008 §14).
2. Entorno de referencia y CI (P-02) → reescritura de ADR-0013 y arquitectura de destino en ADR-0002.
3. Aceptación de ADR-0001, 0007 y 0015 (tras P-12).
4. Checklist de cierre y lista de ADR del gate (P-07, P-08).
5. Estructura del workspace y nomenclatura (P-09, P-10, P-11).
6. Normalización de estados y aprobaciones de ADR (P-20).

No bloquean el cierre de Fase 0 (sí Gate 1/2): identificación de hardware real (P-21, P-22), ADR-0005, DCO/CLA. Excepción propuesta: contratos D-Bus XML y `storage.md` al inicio de Fase 1 (P-06).

## 5. Problemas abiertos

Ver base-audit-r3 §1 y §8 y la tabla de rev. 2 §B. Resueltos: P-01 (este documento), P-04 (`security/dependencias.md`), P-25.

## 6. Decisiones reservadas al propietario

Ver base-audit-r3 §9.2 (P-xx) y, para las decisiones D-01 a D-30 de las revisiones técnicas 4–10, el registro único en [`consolidacion-fase0.md` §4](../project-management/consolidacion-fase0.md).

## 7. Revisiones técnicas complementarias (4–10)

Consolidadas en [`project-management/consolidacion-fase0.md`](../project-management/consolidacion-fase0.md) (Tarea 010). Ninguna cambia el estado de la fase ni de los ADR. Pruebas derivadas: `spikes.md` §T, todas sin autorizar. Investigaciones posteriores (Tareas 012–023): [`project-management/audits/fase-0/INDEX.md`](../project-management/audits/fase-0/INDEX.md).
