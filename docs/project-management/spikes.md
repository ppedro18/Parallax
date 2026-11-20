# Registro de spikes

- **Estado del registro:** Propuesto (base-audit-r3 §6.3, P-24). Pendiente de aprobación.
- Un spike es código desechable fuera del producto para obtener evidencia. **Ningún spike está autorizado.** Ejecutar uno requiere aprobación expresa del propietario registrada aquí y, si usa dependencias, su entrada en `security/dependencias.md`.
- Las fichas detalladas (hipótesis, alcance, criterios) se prepararán en la tarea F.

## Campos
| Campo | Contenido |
|---|---|
| ID | `SPK-NNN` |
| ADR / criterio | Qué evidencia aporta |
| Entorno | VM funcional · hardware real DRM/KMS · macOS (solo crates puros) |
| Fase | Fase en la que puede ejecutarse |
| Autorización | — / fecha |
| Estado | Propuesto · Autorizado · En curso · Concluido · Descartado |
| Resultado | Enlace al informe |

## Registro

| ID | ADR / criterio | Entorno | Fase | Autorización | Estado |
|---|---|---|---|---|---|
| SPK-001 | ADR-0003 Smithay, backend winit/anidado | VM funcional | 1 | — | Propuesto |
| SPK-002 | ADR-0003 Smithay, DRM/KMS + libinput + logind | Hardware real | 1 | — | Propuesto |
| SPK-003 | ADR-0004 `px-wm` puro | VM o macOS (sin E/S) | 1 | — | Propuesto |
| SPK-004 | ADR-0005 toolkit: funcionalidad y accesibilidad | VM funcional | 1 | — | Propuesto |
| SPK-005 | ADR-0005 toolkit: memoria y frame time | Hardware real | 1 | — | Propuesto |
| SPK-006 | ADR-0007 zbus, coalescencia y memfd | VM funcional | 1 | — | Propuesto |
| SPK-007 | ADR-0009 Btrfs + snapshots + systemd-boot + boot counting | VM UEFI | 1 | — | Propuesto |
| SPK-008 | ADR-0009 EFI en firmware real | Hardware real | 1–6 | — | Propuesto |
| SPK-009 | Layer-shell con clientes de prueba | VM funcional | 1 | — | Propuesto |

Lista derivada de `roadmap.md §1.4` y base-audit-r3 §4.4; no añade spikes que el roadmap no prevea.

## T. Pruebas técnicas derivadas de las revisiones 5–10

Mismas reglas: **ninguna autorizada ni ejecutada**. Detalle en [`consolidacion-fase0.md` §8](consolidacion-fase0.md).

| ID | Prueba | Entorno | Autorización | Estado |
|---|---|---|---|---|
| T-FD | Herencia y cierre de descriptores (intermediario y rol) | VM | — | Propuesto |
| T-LOCK | `flock`: concurrencia, SIGKILL, recuperación, otro entorno | VM | — | Propuesto |
| T-GLIBC | `addclosefrom_np` en glibc 2.44 de Arch | VM | — | Propuesto |
| T-PIDS | `StartTransientUnit` con `PIDs` en systemd 262.2 | VM | — | Propuesto |
| T-PIDFD | `GetUnitByPIDFD` | VM | — | Propuesto |
| T-TRANS | `PartOf` y `Slice` en unidades transitorias | VM | — | Propuesto |
| T-GRAFO | Grafo de targets de sesión | VM | — | Propuesto |
| T-DUMP | `PR_SET_DUMPABLE` y acceso a descriptores | VM | — | Propuesto |
| T-GREETD | Ciclo greetd + PAM + logind | VM | — | Propuesto |
| T-KILL | Supervivencia de servicios D-Bus y unidades de terceros | VM | — | Propuesto |
| T-POLKIT | Registro y desconexión del agente polkit | VM | — | Propuesto |
| T-DBUSACT | Apps `DBusActivatable` e identificación por resourced | VM | — | Propuesto |
| T-SESSION-FAIL | Caída inesperada del compositor (diseño abajo) | VM + greetd real | — | Propuesto |

### T-SESSION-FAIL — Caída inesperada del compositor

- **Estado:** Propuesto. **No autorizado, no ejecutado.** Solo diseño (Tarea 011).
- **Objetivo:** observar el comportamiento real del ciclo de vida de la sesión cuando el compositor termina inesperadamente, y cuando termina el comando de sesión. Contrasta el grafo **propuesto** (rev. 8 §6.2) y los escenarios de [`consolidacion-fase0.md` §4.3](consolidacion-fase0.md); no lo da por válido.
- **Relaciona:** D-24, D-28, D-29, C-22, C-37, RV-25, RV-27, RV-37; complementa T-GRAFO, T-GREETD, T-KILL, T-LOCK.
- **Entorno:** VM Arch con systemd 262.2, greetd 0.10.3 y logind reales; versiones exactas registradas en el informe.
- **Unidades involucradas (todas de prueba):** `parallax-session.target`, `graphical-session.target`, `graphical-session-pre.target`, `parallax-compositor.service`, `parallax-settingsd.service` (o un servicio de sesión equivalente), un scope de rol, una app transitoria `app-parallax-…@….service`, el comando de sesión, greetd, la sesión logind, y como terceros: un servicio activado por D-Bus, una unidad de usuario sin `PartOf` y PipeWire.

| # | Escenario | Precondiciones | Acciones | Estado esperado (hipótesis a contrastar) | Evidencia a recopilar | Aprobación | Fallo |
|---|---|---|---|---|---|---|---|
| S1 | Línea base | Sesión iniciada por greetd → comando de sesión; lock `flock` tomado | Arrancar una app PARALLAX, un rol y el servicio de sesión; arrancar los tres terceros | Todo `active` | `systemctl --user list-units`, `show -p ActiveState,SubState`, `loginctl session-status`, `ls -l /proc/<pid>/fd` del comando | Inventario completo y reproducible | Algún componente no arranca o no se puede inventariar |
| S2 | Caída inesperada del compositor | S1 | `kill -SEGV` (o `SIGKILL`) al proceso principal de `parallax-compositor.service` | Compositor `failed`; target parado por `BindsTo`; `graphical-session.target` parado por `StopWhenUnneeded` si nada lo mantiene; rol, app y servicio de sesión parados por `PartOf`; terceros pueden seguir | `journalctl --user -o json` con marcas de tiempo; `systemctl --user show` de cada unidad antes y después; jobs emitidos; qué unidad mantiene `graphical-session.target` (`list-dependencies --reverse`) | Para cada transición se identifica la dependencia o mecanismo que la causó; ninguna se explica por suposición | Alguna unidad PARALLAX queda activa sin causa explicada, o una transición no tiene mecanismo identificable |
| S3 | Detección por el comando de sesión | S2 | Observar el comando | Detecta `inactive` del target, libera el lock y sale | Código de salida, tiempo hasta la salida, estado del `flock` (`/proc/locks`) | Sale en un tiempo acotado y libera el lock | Se queda colgado o sale sin que el target esté inactivo |
| S4 | Retorno al greeter | S3 | Observar greetd | greetd relanza `default_session`; la sesión logind anterior se cierra | `journalctl -u greetd`, `loginctl list-sessions`, VT activa, procesos del greeter | Greeter visible; sesión anterior cerrada; nueva autenticación necesaria | No vuelve el greeter, o la sesión logind anterior sigue activa sin explicación |
| S5 | Caída del comando de sesión (RV-37) | S1 | `kill -KILL` al comando de sesión | Comportamiento **desconocido**: registrar si el compositor y la sesión siguen vivos, si greetd relanza el greeter y si logind revoca los dispositivos | Igual que S2 + S4; propietario del DRM master; acceso a input | Ningún estado deja visible o interactiva una sesión gráfica de usuario sin autenticación | Cualquier momento en que la sesión anterior sea visible o reciba input sin autenticar |
| S6 | Terceros | S2 y S5 | Inventario tras la parada | Terceros pueden sobrevivir hasta que terminen logind o el user manager (`KillUserProcesses`) | Lista de supervivientes, su cgroup y su destino tras cerrar la sesión logind | Cada superviviente queda explicado y documentado | Un superviviente no tiene explicación, o PARALLAX lo mata sin estar en su ciclo de vida |
| S7 | Sin sesión desbloqueada | S2–S5 | Revisar toda la traza | Nunca hay una sesión gráfica de usuario presentada sin autenticación | Capturas de pantalla o de la VT, foco, journal | Ninguna transición la permite | Existe al menos una transición que la permite |

**Criterio global:** la prueba aprueba solo si S2–S7 aprueban. Un fallo en S5 o S7 reabre D-24 y D-28 antes de cualquier implementación.
