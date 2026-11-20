# Consolidación de hallazgos de Fase 0 (revisiones 1–10)

- **Estado del documento:** registro maestro descriptivo. **No es normativo.** No acepta ADR, no aprueba decisiones y no autoriza pruebas.
- **Fase 0:** `IN_PROGRESS`. **ADR:** todos `Propuesto`. **Fase 1:** sin autorización.
- **Fecha:** 2026-11-19 (Tarea 010).
- **Punto de entrada para una sesión nueva:** leer este documento, después [`roadmap/fase0-revision.md`](../roadmap/fase0-revision.md) y [`base-audit-r3.md`](base-audit-r3.md).

## 0. Cómo leer este documento

| Marca | Significado |
|---|---|
| **Contrato** | Texto vigente en un documento del repositorio (aunque su ADR siga `Propuesto`) |
| **Propuesta** | Recomendación de una revisión. Pendiente de aprobación. No es norma |
| `[V]` / `[V-p]` | Evidencia verificada en fuente oficial (lectura completa / parcial) |
| `[R]` | Consta en el repositorio |
| `[H]` | Hipótesis o inferencia |
| `[VM]` | Requiere prueba en VM (no autorizada) |
| `[P]` | Decide el propietario |

«Resuelto en diseño» significa coherente en papel; **nunca** verificado ni implementado.

**Estados de evidencia que no deben confundirse** (Tarea 011): (1) API documentada; (2) API disponible en una versión concreta (p. ej. systemd 262.2, glibc 2.44); (3) comportamiento observado en Arch; (4) comportamiento validado por una prueba; (5) decisión aprobada por el propietario. **Ningún elemento de este documento ha alcanzado los estados 3, 4 o 5.** Una afirmación `[H]` o `[VM]` no pasa a `[V]` por repetirse en varias revisiones. `[V]` sobre una unidad upstream (rama `main`) no implica `[V]` sobre la unidad instalada en Arch.

### Registros centrales (un solo lugar por tema)

| Registro | Ubicación |
|---|---|
| Hallazgos técnicos C-xx / N9 / N10 | §3 de este documento |
| Riesgos | [`architecture/riesgos.md`](../architecture/riesgos.md) (R01–R28, originales) + §5 de este documento (RV-xx, de revisiones) |
| Decisiones pendientes D-xx | §4 de este documento (único registro) |
| Decisiones P-xx de la auditoría base | [`base-audit-r3.md`](base-audit-r3.md) §9–§10 |
| Spikes y pruebas | [`spikes.md`](spikes.md) (SPK-xxx y T-xxx) |
| Historial de auditorías | §1 de este documento |
| Estado de Fase 0 | [`roadmap/fase0-revision.md`](../roadmap/fase0-revision.md) |

---

## 1. Historial de revisiones (evidencia histórica, no normativa)

| Rev. | Archivo | Alcance | Veredicto de la propia revisión |
|---|---|---|---|
| 1 | [`base-audit.md`](base-audit.md) | Auditoría base, hallazgos P-xx | — |
| 2 | [`base-audit-r2.md`](base-audit-r2.md) | Corrección de rev. 1 | — |
| 3 | [`base-audit-r3.md`](base-audit-r3.md) | Vigente para P-xx; Gate 0, ADR por grupos | — |
| 4 | **No recuperable** (ver §7) | Hallazgos C-01 a C-19 y decisiones D-01 a D-17 (temas conocidos por referencias posteriores) | — |
| 5 | [`revisiones/r05-quinta.md`](revisiones/r05-quinta.md) | C-20 a C-28 | Requiere otra revisión |
| 6 | [`revisiones/r06-sexta.md`](revisiones/r06-sexta.md) | C-01 a C-33; D-01 a D-23 | Lista para revisión del propietario |
| 7 | [`revisiones/r07-septima.md`](revisiones/r07-septima.md) | C-20 a C-23, C-34 a C-38 | Lista con condiciones |
| 8 | [`revisiones/r08-octava.md`](revisiones/r08-octava.md) | `PIDs`, `posix_spawn`, grafo, greetd, D-24 a D-29 | Lista con condiciones |
| 9 | [`revisiones/r09-novena.md`](revisiones/r09-novena.md) | Descriptores, entorno, greetd verificado, alcance de X, D-30 | Lista con condiciones |
| 10 | [`revisiones/r10-decima.md`](revisiones/r10-decima.md) | `addclosefrom_np`, `flock` | Lista para revisión del propietario |

Los archivos `revisiones/r05`–`r10` son copias literales de los informes que estaban en el scratchpad de una sesión anterior; se archivan aquí para que no se pierdan. **No se editan.**

Las Tareas 012–023, posteriores a esta consolidación, junto con las evidencias de empaquetado de Arch y el manifiesto de fuentes, están archivadas en [`audits/fase-0/INDEX.md`](audits/fase-0/INDEX.md). Ese archivo es histórico y no normativo; sus riesgos (RV-38 a RV-56) y decisiones (D-31 a D-38) son **propuestas** que todavía no figuran en los §4 y §5.

---

## 2. Resumen del estado (para una sesión nueva)

1. La estructura documental está consolidada, pero **la Fase 0 sigue incompleta**. Persisten decisiones pendientes, contradicciones documentales y validaciones técnicas no ejecutadas. **Ningún ADR está aceptado** y los bloqueos de cierre P-xx siguen en `fase0-revision.md` §4.
2. Las revisiones 4–10 analizaron el ciclo de vida de la sesión gráfica, el lanzamiento de procesos, la autorización Wayland por socket y polkit. Produjeron **propuestas** que contradicen varios documentos originales (§6). Esos documentos solo llevan una nota de aviso; **su texto original no se ha reescrito**, porque cambiarlo requiere las decisiones de §4.
3. **Regla:** las propuestas vigentes representan la última recomendación técnica disponible para cada tema. No sustituyen los contratos actuales del repositorio ni adquieren carácter normativo hasta que el propietario apruebe las decisiones correspondientes y se actualicen los documentos afectados. Son, a fecha de hoy:
   - **Lanzamiento de apps:** crate común de lanzamiento (Fase 3), usado por Shell y apps; el compositor solo valida tokens y lanza roles fijos. Se propone retirar `Session1.Launch`.
   - **Roles del compositor (Shell, lock, auth, portal, settings-output; capacidades Wayland restringidas, sin privilegios del SO):** los lanza el compositor mediante `posix_spawn` de un **intermediario efímero**, registrado en un scope transitorio con `PIDs`; el compositor los supervisa (catálogo cerrado, 5/min, backoff, overlay).
   - **Descriptores:** orígenes duplicados a ≥10 con `F_DUPFD_CLOEXEC`; 0–4 explícitos; `posix_spawn_file_actions_addclosefrom_np(5)` como última acción; `envp` por lista blanca.
   - **Sesión:** greetd → comando de sesión propio → `parallax-session.target`; exclusión con `flock`; recomendación de que la caída del compositor termine la sesión gráfica de PARALLAX (política X). La propagación de la parada y el retorno al greeter **no están demostrados** (§4.3, T-SESSION-FAIL).
4. Lo pendiente no se resuelve con más revisiones en papel: son **decisiones del propietario** (§4) y **pruebas en VM no autorizadas** (§8).

---

## 3. Matriz de trazabilidad de hallazgos

Columnas: Rev. = revisión de origen (→ revisiones que lo corrigen). Estado según la última revisión que lo trata.

### 3.1 Hallazgos C-01 a C-19 (origen rev. 4, estado según rev. 6)

| ID | Tema | Documento afectado | Decisión | Prueba | Estado |
|---|---|---|---|---|---|
| C-01 | Modelo del agente polkit (A con `fallback` / B propio) | ADR-0010, `shell.md` §9 | D-01 | Polkit en VM | Pendiente de decisión |
| C-02 | `Session1` unificada | `ipc.md` §6.1, `compositor.md` §10.2 | D-12 | — | Propuesta |
| C-03 | Nombres de unidad según systemd.io | `arquitectura.md` §2 | D-05 | Escapado | Propuesta |
| C-04 | Lanzamiento de apps | `arquitectura.md` §5, `dock.md` | D-03 | — | Pendiente de decisión |
| C-05 | Retirar o versionar interfaz (ADR-0007 §5.3) | ADR-0007 | — | — | Propuesta |
| C-06 | Bus de diagd | `arquitectura.md`, `servicios.md`, `ipc.md` | D-06 | — | Pendiente de decisión |
| C-07 | Cinco árboles de workspace distintos | Varios | — | — | Propuesta (`servicios.md` §17 sigue discrepando) |
| C-08 | `cargo vet` | `workspace-rust.md` §6 | D-09 | — | Pendiente de decisión |
| C-09 | `vendor/` | `dependencias.md` regla 4 | D-08 | — | Pendiente de decisión |
| C-10 | Licencia por crate + revisión humana | ADR-0008 | — | — | Propuesta |
| C-11 | Nombre `task-manager` | `arquitectura.md` §4 | — | — | Propuesta |
| C-12 | Desktop File ID en lugar de `app_id` | `ipc.md`, `dock.md` | D-03 | — | Propuesta |
| C-13 | XML D-Bus en `docs/architecture/dbus/` (faltan) | `arquitectura.md` §3 | P-06 | — | Parcial |
| C-14 | Prefijo `parallax-*` en binarios | ADR-0006/0009/0010, roadmap | — | — | Propuesta |
| C-15 | ADR-0011 | ADR-0011 | — | — | **No verificable** (texto de rev. 4 no disponible) |
| C-16 | Entrega de fd a roles vs `Restart=always` | ADR-0015, `arquitectura.md` | D-18 | Scope | Pendiente de decisión |
| C-17 | Formalizar `LaunchPrivileged` | `ipc.md` | — | — | Propuesta |
| C-18 | Alinear con `ipc.md` §13 | — | — | — | Propuesta |
| C-19 | `libpolkit-agent-1` como helper | ADR-0010 §7, `dependencias.md` | D-16 | — | Pendiente de decisión |

### 3.2 Hallazgos C-20 a C-39 y N9/N10

| ID | Hallazgo | Rev. | Documento afectado | Riesgo | Decisión | Prueba | Estado / propuesta vigente |
|---|---|---|---|---|---|---|---|
| C-20 | Propietario del lanzamiento | 4 → 5 → 6 → 7 | `arquitectura.md` §5, `ipc.md` §6.1, `dock.md`, `compositor.md` §10.2 | RV-08, RV-16, RV-18 | D-03, D-21 | — | Propuesta vigente (rev. 7): **crate común** en `crates/`, Fase 3. Sustituye «módulo de `shell/shell/`» (rev. 6), que creaba dependencia C6→C5 (C-20b) |
| C-21 | Límites del socket de rol y rol `auth` | 5 → 6 → 7 → 8 → 9 → 10 | ADR-0015, `modelo-seguridad.md`, `ingeniero-compositor.md` | RV-22, RV-23, RV-31 | D-22 | T-FD, T-DUMP | Pendiente de decisión del propietario. Modelo de amenazas T1–T11 (rev. 7 §4.3) |
| C-22 | Cierre de sesión y grafo systemd | 5 → 6 → 7 → 8 → 9 | `arquitectura.md` §5, `ipc.md` §6.1, `lockscreen-greeter.md` | RV-15, RV-25 | D-19, D-23, D-24 | T-GRAFO, T-KILL | Propuesta: `Logout` = ciclo coordinado; grafo de rev. 8 §6.2 revisado documentalmente en rev. 9 §5; **pendiente de validación funcional en VM** (T-GRAFO, T-SESSION-FAIL). Ver §4.3 |
| C-23 | Registro de procesos en scope | 5 → 6 → 7 → 8 → 9 | `arquitectura.md` §2/§5 | RV-20, RV-30 | D-18, D-27 | T-PIDS, T-PIDFD | Propuesta: `StartTransientUnit` + `PIDs` + intermediario (rev. 8). Corrige rev. 7 (C-23b, C-23c) |
| C-24 | `cargo-deny` sobreafirmado | 4 → 5 → 6 | `dependencias.md` regla 3 | — | — | — | Propuesta: «filtrado automático; compatibilidad por revisión registrada» |
| C-25 | Orden de mensajes D-Bus | 5 → 6 | `eventos.md` §4 | — | D-20 | — | Propuesta: protocolo `AddMatch` previo + búfer + `epoch`/`revision`, sin depender del orden `[V]` |
| C-26 | Identidad del productor D-Bus | 5 → 6 | `modelo-seguridad.md` | RV-12 | — | — | Propuesta: en el bus de sesión solo se prueba el UID `[V]` |
| C-27 | Infraestructura oficial polkit | 5 → 6 | ADR-0010 | RV-02, RV-11, RV-14 | D-01, D-16 | T-POLKIT | Pendiente de decisión (B recomendada, ventaja moderada) |
| C-28 | xdg-activation y `DBusActivatable` | 5 → 6 → 7 | `dock.md`, ADR-0014 | RV-13 | — | T-DBUSACT | Propuesta: `Activate` + `activation-token`; resourced identifica por `GetUnitByPIDFD` |
| C-29 | «systemd reinicia `parallax-shell`» incompatible con ADR-0015 | 6 | `arquitectura.md` §5 | RV-01, RV-17 | D-18 | T-PIDS | Pendiente de decisión |
| C-30 | «peer credentials» autentican al servicio | 6 | `modelo-seguridad.md` (fila D-Bus) | RV-12 | — | — | Propuesta; nota añadida en el documento |
| C-31 | «ningún otro cliente puede enlazar la global» frente al mismo UID | 6 | `modelo-seguridad.md` (fila Shell) | — | D-22 | T-DUMP | Pendiente; nota añadida en el documento |
| C-32 | Cierre de sesión con pantalla bloqueada sin definir | 6 | `lockscreen-greeter.md` | — | D-23 | — | Pendiente de decisión |
| C-33 | Sin sitio para la lógica de lanzamiento | 6 → 7 | Workspace | — | D-21 | — | Absorbido por C-20 (crate común) |
| C-34 | Compositor figura con unidad `parallax-session.target` | 7 | `arquitectura.md` §2 | — | — | — | Propuesta: `parallax-compositor.service`; nota añadida |
| C-35 | No se mencionan `graphical-session*.target` | 7 | Varios | RV-25 | — | T-GRAFO | Propuesta |
| C-36 | Arranque de la sesión desde greetd sin definir | 7 → 8 → 9 | `arquitectura.md` §5 | RV-29 | D-28 | T-GREETD | Propuesta G-c (comando propio); encaja con greetd `[V]`; código de salida y PAM sin documentar |
| C-37 | Bloqueo tras caída del compositor | 7 → 8 | `compositor.md` §11.5 | RV-24, RV-37 | D-24 (D-26 si Y) | T-SESSION-FAIL | X es la recomendación v1 (la sesión nueva exige autenticación en el greeter). La propagación completa de la parada, el fin del comando de sesión y el retorno al greeter siguen **pendientes de definición y validación**; no hay garantía demostrada. El contrato vigente (`compositor.md` §11.5 «Reiniciar el compositor cuando corresponda») contradice X y no se ha modificado |
| C-38 | `parallax-greeter.service` | 7 → 8 → 9 | `arquitectura.md` §2, `lockscreen-greeter.md` | — | — | — | **Confirmado** `[V greetd(5) 0.10.3]`: el greeter es el `command` de `[default_session]`. Nota añadida |
| C-39 | Restauración de sesión tras caída | 8 | `arquitectura.md` §5 | RV-27 | D-24 | — | Propuesta: la restauración ocurre en la sesión nueva |
| N9-1 | Colisión de `dup2` (orígenes en 3/4) | 9 | `modelo-seguridad.md` | — | — | T-FD | Resuelto en diseño: orígenes ≥10 con `F_DUPFD_CLOEXEC` |
| N9-2 | fd 0–2 indefinidos; fd sin `CLOEXEC` sobreviven | 9 → 10 | `modelo-seguridad.md` | RV-31 | — | T-FD | 0–2 explícitos vigente; **`close_range` en `main` sustituido por N10-1** |
| N9-3 | Entorno sin lista blanca | 9 | `modelo-seguridad.md`, ADR-0015 | RV-32 | D-22 | T-FD | Resuelto en diseño: `envp` explícito, nunca `environ` |
| N9-4 | «La política X evita huérfanos» es falsa | 9 | `arquitectura.md` §5 | RV-28 | D-24 | T-KILL | Corregido: X solo detiene lo que PARALLAX lanza con `PartOf` |
| N10-1 | `close_range` en `main()` llega tarde | 10 | `modelo-seguridad.md`, ADR-0015 | RV-31 | D-22 | T-FD | Resuelto en diseño: `addclosefrom_np(5)` última acción |
| N10-2 | Versión mínima de glibc sin confirmar | 10 | ADR-0002 | RV-34 | D-27 | T-GLIBC | Pendiente de VM |
| N10-3 | ¿Expone el crate `libc` `addclosefrom_np`? | 10 | `dependencias.md` | RV-30 | D-27 | — | Diferible a Fase 1 (`extern "C"` si no) |
| N10-4 | `ActiveState` + arranque no es atómico (TOCTOU) | 10 | `arquitectura.md` §5 | — | D-29 | T-LOCK | Resuelto en diseño: `flock` |
| N10-5 | Alcance de D-29 | 10 | — | RV-36 | D-29 | — | Pendiente de decisión del propietario |
| N10-6 | Convivencia con otros entornos gráficos | 10 | — | RV-36 | D-29 | T-LOCK | Límite documentado: fuera de soporte v1 |
| N10-7 | Fallo de `closefrom` | 10 | — | — | — | T-FD | Fallo seguro: `_exit(127)`, no hay `exec` `[V-p spawni.c]` |
| N10-8 | Equivalencia glibc `master` ↔ 2.44 de Arch | 10 | ADR-0002 | RV-34 | — | T-GLIBC | **No verificada** |

---

## 4. Registro único de decisiones pendientes del propietario y técnicas

Todas: **estado = pendiente**. Ninguna se ha aceptado. Las marcadas **[P]** no pueden cerrarse sin el propietario.

### 4.1 Decisiones con ficha completa (exigidas por la Tarea 010)

#### D-18 [P] — Supervisión de roles por el compositor
- **Contexto:** ADR-0015 exige que la capacidad sea el fd que entrega el compositor. `arquitectura.md` §2/§5 define el Shell como `.service` con `Restart=always`. systemd no puede recibir un fd ya abierto por otro proceso (`OpenFile=` abre rutas; el fd store solo acepta fds de la propia unidad) `[V systemd.service(5), sd_notify(3)]`.
- **Alternativas:** R1 `.service` + entrega de fd (descartada: no existe mecanismo); **R2** compositor lanza + scope transitorio (recomendada); R3 socket con ruta (debilita ADR-0015); R4 pidfd + cgroup (manipulable por el mismo UID); R5 broker (traslada el problema).
- **Consecuencias:** el compositor implementaría una política de reinicio acotada para los roles del compositor. Valores de partida tomados del contrato actual del Shell (`arquitectura.md` §5 [R]): catálogo cerrado, 5 reinicios/min, backoff desde 200 ms, overlay al superar el límite. **No es una política cerrada.** Faltan por definir: ventana temporal exacta; número máximo de intentos; backoff máximo; condición de reinicio del contador; tratamiento de fallos permanentes; distinción entre fallo de arranque (antes de `RUNNING`) y caída posterior. Cambiaría `arquitectura.md` §2 y §5.
- **Evidencia:** `[V]` systemd.service(5), systemd.scope(5), sd_notify(3), CONTROL_GROUP_INTERFACE (`PIDs`).
- **Incertidumbres:** `PIDs` en systemd 262.2 de Arch; tipo D-Bus (`au` `[H]`); garantía de `JobRemoved=done`.
- **Documentos afectados:** `arquitectura.md` §2/§5, ADR-0015, ADR-0006, ADR-0010, `compositor.md`.
- **Estado:** pendiente de aprobación del propietario.

#### D-22 [P] — Límites de seguridad, rol `auth` y lista blanca de entorno
- **Contexto:** el socket Wayland que el compositor entrega por fd a un rol es una **capacidad de conexión**: da acceso a ciertos globals en esa conexión. No concede privilegios del sistema operativo (el rol corre con el UID del usuario y sin capacidades del kernel) ni aísla entre procesos del mismo UID. Hay que fijar qué protege y qué no.
- **Alternativas:** aceptar el modelo propuesto (capacidad por fd + `PR_SET_DUMPABLE=0` + higiene de fd + `envp` por lista blanca + rol `auth` con globals mínimos); o no aceptar y mantener ADR-0015 sin catálogo de roles (bloquea la opción B de D-01).
- **Consecuencias:** se documenta explícitamente que **no hay sandbox entre procesos del mismo UID** (T6–T8 no mitigadas: kill, `$HOME`, nombres D-Bus, CAP_SYS_PTRACE/root). T9 (imitación visual del diálogo) queda parcialmente mitigada.
- **Evidencia:** `[V]` Yama, `pidfd_getfd(2)`, `proc_pid_fd(5)`, `PR_SET_DUMPABLE(2const)`; `[V-p]` ptrace(2) access mode.
- **Incertidumbres:** si `dumpable` sobrevive a `execve` (se propone aplicarlo en el rol tras el `exec`); reapertura de sockets desde `/proc/<pid>/fd`; lista blanca exacta (riesgo RV-32 de omitir variables de accesibilidad).
- **Lista blanca propuesta (pendiente de validación funcional; no es contrato).** Antes de fijarla hay que evaluar las variables que necesitan la accesibilidad (AT-SPI, p. ej. `AT_SPI_BUS_ADDRESS` [H]), la localización, los métodos de entrada (IME, p. ej. `XMODIFIERS`, `GTK_IM_MODULE`, `QT_IM_MODULE` [H]) y el funcionamiento normal de cada rol (RV-32). Candidatas actuales: `HOME`, `USER`, `LOGNAME`, `LANG`, `LC_*`, `XDG_RUNTIME_DIR`, `XDG_SESSION_ID`, `XDG_SESSION_TYPE=wayland`, `XDG_CURRENT_DESKTOP`, `DBUS_SESSION_BUS_ADDRESS`, `PATH` fijo, `WAYLAND_SOCKET=3`. Excluidas: `WAYLAND_DISPLAY`, `XDG_ACTIVATION_TOKEN`, `DESKTOP_STARTUP_ID`, `LD_*`, `SSH_AUTH_SOCK`, depuración.
- **Documentos afectados:** ADR-0015, `modelo-seguridad.md`, `ingeniero-seguridad.md`, `ingeniero-compositor.md`, `shell.md` §9.
- **Estado:** pendiente de aprobación del propietario.

#### D-24 [P] — Política X de recuperación en v1
- **Contexto:** si cae el compositor, todas las apps pierden Wayland (limitación del protocolo).
- **Alternativas:** **X** terminar la sesión gráfica y volver al greeter (recomendada v1); Y reiniciar el compositor en la misma sesión (requiere D-26 «arrancar Locked», generaciones y limpieza); Z híbrida.
- **Recomendación (no aprobada):** la caída del compositor termina la sesión gráfica de PARALLAX; el usuario vuelve al greeter; la restauración cooperativa de apps ocurre en una sesión nueva; las unidades de terceros ajenas al ciclo de vida de PARALLAX pueden sobrevivir. **No** se afirma que se terminen todas las aplicaciones o servicios del usuario.
- **Consecuencias esperadas (sin demostrar):** el usuario pierde todas las ventanas y vuelve al login, donde debe autenticarse. Que no exista ninguna transición que deje una sesión gráfica desbloqueada **no está demostrado** (ver §4.3, caso 4, y RV-37). **Alcance previsto (corregido en rev. 9):** X detendría compositor, settingsd, roles y apps lanzadas por PARALLAX (`PartOf=graphical-session.target`), si la cadena de §4.3 se comporta como se espera. **No** detiene servicios D-Bus, apps `DBusActivatable` de terceros, PipeWire ni unidades de usuario sin `PartOf`; esos sobreviven hasta que logind o el user manager terminen (`KillUserProcesses`, lingering). La restauración cooperativa ocurre en la sesión nueva.
- **Evidencia:** `[V]` unidad upstream `graphical-session.target` en la rama `main` (`StopWhenUnneeded=yes`, `RefuseManualStart=yes`); `[V]` semántica documentada de `BindsTo`/`PartOf` (ambas en **una sola dirección**); `[V]` greetd vuelve a `default_session` cuando termina el comando de sesión.
- **Incertidumbres:** unidad instalada en systemd 262.2; si otra unidad mantiene `graphical-session.target` como necesaria; `KillUserProcesses` efectivo en Arch; cierre de PAM y logind por greetd; caso de caída del comando de sesión (§4.3).
- **Documentos afectados:** `arquitectura.md` §5, `compositor.md` §11.5, `servicios.md` §14, ADR-0015.
- **Estado:** pendiente de aprobación del propietario.

#### D-27 — Intermediario y dependencia libc/glibc
- **Contexto:** `fork()` en un compositor Rust multihilo solo permite funciones async-signal-safe hasta `execve` `[V fork(2)]`; `pre_exec` no garantiza `malloc`/`std::env`/mutex `[V]`; `std::process::Command` no coloca fds en 3/4 sin `pre_exec` `[V-p]`.
- **Alternativas:** A `fork` en el compositor (descartada); B auxiliar permanente (proceso extra); **C `posix_spawn` + intermediario efímero** (recomendada); `systemd-run --scope` (orden de registro no documentado).
- **Consecuencias:** segundo `[[bin]]` en `core/compositor/` (nombre pendiente), mínimo, sin dependencias salvo `libc`. Dependencia del crate `libc` (`MIT OR Apache-2.0` `[V docs.rs 0.2.191]`) o `extern "C"` propio; `unsafe` con `// SAFETY:`. **PARALLAX v1 depende de glibc** (≥ 2.34 `[H]`; Arch tiene 2.44 `[V]`); musl fuera de alcance.
- **Evidencia:** `[V]` posix_spawn(3), close_range(2); `[V-p]` glibc `spawni.c` (rama `master`).
- **Incertidumbres:** T-GLIBC; si `libc` expone `addclosefrom_np`; MSRV (`rust-toolchain.toml` aún no existe).
- **Documentos afectados:** ADR-0002, ADR-0008, `dependencias.md`, `compositor.md` §15.
- **Estado:** pendiente (técnica + registro ADR-0008; la dependencia requiere aprobación del propietario).

#### D-28 — Comando de sesión propio
- **Contexto:** greetd ejecuta un comando arbitrario y vuelve a `default_session` cuando termina `[V greetd(1)/(5) 0.10.3-2]`.
- **Alternativas:** G-a greetd ejecuta el compositor (sin grafo systemd); G-b `systemctl --user start --wait` (`--wait` sobre target no demostrado); **G-c comando propio mínimo** (recomendada).
- **Consecuencias:** un binario más, no residente pesado. Flujo (con rev. 10): abrir lock → `flock` → limpiar residuos → arrancar `parallax-session.target` → esperar `JobRemoved` → mantener lock hasta `inactive` → salir.
- **Incertidumbres:** tratamiento del código de salida y cierre de PAM/logind por greetd (no documentado; código `worker.rs` no leído, HTTP 502).
- **Documentos afectados:** `arquitectura.md` §5, `lockscreen-greeter.md`.
- **Estado:** pendiente.

#### D-29 [P] — Exclusión de sesiones mediante `flock` y garantía limitada
- **Contexto:** `systemd --user` y `graphical-session.target` son compartidos por usuario; dos sesiones gráficas interferirían (política X, limpieza).
- **Historia:** rev. 8 introduce «una sola sesión gráfica por usuario»; rev. 9 propone consultar `ActiveState` antes de arrancar; **rev. 10 lo sustituye** (TOCTOU) por `flock`.
- **Propuesta vigente (rev. 10):** el comando de sesión abre `$XDG_RUNTIME_DIR/parallax/session.lock` con `O_RDWR|O_CREAT|O_CLOEXEC, 0600` y hace `flock(LOCK_EX|LOCK_NB)`; mantiene el fd abierto toda la sesión; con el lock adquirido, detiene un `parallax-session.target` residual; si `graphical-session.target` está activo sin `parallax-session.target`, rechaza (otro entorno).
- **Requisito añadido en Tarea 010 (sin verificar):** comprobar en la creación/apertura que `parallax/` es directorio propio (no symlink, propietario = UID, modo 0700) y que `session.lock` es archivo regular propio (`O_NOFOLLOW`, `fstat` tras abrir).
- **Límite de `O_CLOEXEC`:** evita que el descriptor sobreviva a un `execve`, pero **no** evita que un proceso creado con `fork()` (antes de cualquier `exec`) conserve una copia; mientras exista una copia abierta, el `flock` sigue tomado [V flock(2): asociado a la descripción de archivo abierta]. El comando de sesión no debe crear hijos, o debe cerrar el fd en ellos. Verificación del comportamiento real: T-LOCK.
- **Garantía propuesta (sin validar):** como máximo **una instancia PARALLAX por usuario**. `flock` **no** resuelve la convivencia con otros entornos gráficos del mismo usuario. **No** garantiza que otro entorno no cooperativo arranque una sesión del mismo usuario: escenario **no soportado en v1**.
- **Evidencia:** `[V flock(2)]` atómico, `EWOULDBLOCK`, se conserva en `execve`, se libera al cerrar todas las copias.
- **Incertidumbres:** T-LOCK; `XDG_RUNTIME_DIR` 0700 creado por logind `[H]`.
- **Documentos afectados:** `arquitectura.md` §5, `lockscreen-greeter.md`.
- **Estado:** pendiente de aprobación del propietario.

#### D-30 — Comunicación al usuario del rechazo de una segunda sesión
- **Contexto:** greetd no documenta un canal para devolver el motivo del rechazo al greeter.
- **Alternativas:** mensaje genérico en el greeter; canal propio (archivo/IPC) leído por el greeter; investigar `greetd-ipc(7)` (no leído).
- **Consecuencias:** sin decisión, el usuario ve un fallo de login sin explicación (RV-33).
- **Estado:** pendiente; **no está suficientemente definida** hasta leer `greetd-ipc(7)`.

### 4.3 Ciclo de vida de la sesión: escenarios (Tarea 011)

**Base:** grafo **propuesto** en rev. 8 §6.2 (no incorporado a ningún contrato). En el repositorio [R] solo existe la cadena lineal de `arquitectura.md` §5 y `compositor.md` §11.5 («Reiniciar el compositor cuando corresponda»); ningún documento normativo define `graphical-session.target`, `BindsTo=` ni `PartOf=`.

**Relaciones propuestas y su dirección** (semántica `[V]` systemd.unit(5); comportamiento en 262.2 `[VM]`):

| Relación propuesta | Qué propaga | Qué **no** propaga |
|---|---|---|
| `parallax-session.target` `BindsTo=` + `After=` `parallax-compositor.service` | Si el compositor se detiene o falla, se detiene el target | Parar el target **no** para el compositor por esta relación |
| `parallax-session.target` `BindsTo=graphical-session.target` | Si `graphical-session.target` se detiene, se detiene el target | Parar el target **no** para `graphical-session.target` por esta relación |
| `graphical-session.target` `StopWhenUnneeded=yes` (upstream) | Se detiene cuando ninguna unidad activa la requiere/quiere | No se detiene si otra unidad (p. ej. de terceros) la sigue requiriendo `[H]` |
| Compositor, settingsd, roles, apps `PartOf=graphical-session.target` | Parar `graphical-session.target` los detiene | Parar uno de ellos **no** para `graphical-session.target` |

Por tanto, la parada de `parallax-session.target` **solo** llega a `graphical-session.target` de forma indirecta, vía `StopWhenUnneeded`, y **solo** si nada más la mantiene. Esto no está demostrado.

| # | Escenario | Detecta | Inicia la terminación | Detiene roles y apps PARALLAX | Finaliza el comando de sesión | Vuelta al greeter | Estado de la evidencia |
|---|---|---|---|---|---|---|---|
| 1 | Logout normal | — (solicitud del usuario) | Compositor (ciclo coordinado, C-22): `close` a toplevels; después pide `stop parallax-session.target` al user manager `[H]` | Cadena: target parado → `graphical-session.target` innecesario → `PartOf` detiene compositor, roles, apps `[H]` | Observa `ActiveState=inactive` del target y sale `[H]` | greetd al terminar el comando `[V]` | Diseño; quién emite exactamente el `stop` sin cortar el propio ciclo del compositor está **sin definir** |
| 2 | Parada ordenada del compositor (`stop` explícito) | systemd | systemd: `BindsTo` detiene el target `[V semántica]` | Igual que 1 (indirecto) `[H]` | Igual que 1 `[H]` | `[V]` | `[VM]` |
| 3 | Caída inesperada del compositor | systemd (proceso terminado; `Restart=no` en X) | systemd: `BindsTo` detiene el target `[V semántica]` | Igual que 1 (indirecto); los roles pierden además su conexión Wayland `[H]` | Igual que 1 `[H]` | `[V]` | `[VM]`; T-SESSION-FAIL |
| 4 | Caída del comando de sesión | greetd (su comando terminó) `[V]` | **Nadie** detiene el target: no existe relación entre el comando y las unidades | **No se detienen**; el compositor y la sesión gráfica pueden seguir activos | — | greetd relanza el greeter `[V]` mientras la sesión anterior quizá sigue viva `[H]`; `flock` liberado | **Problema abierto (RV-37)**: posible coexistencia de greeter y sesión desbloqueada; dependencia de cómo greetd cierre PAM/logind y de la revocación de dispositivos por logind `[VM]`. La rev. 10 solo lo limpia en el **siguiente** inicio |
| 5 | Parada de `parallax-session.target` | — | Quien lo pare | Indirecto vía `StopWhenUnneeded` `[H]`; el compositor **no** se detiene por `BindsTo` en esta dirección | Igual que 1 | `[V]` | `[VM]` |
| 6 | Parada de `graphical-session.target` | systemd | systemd: `BindsTo` detiene el target; `PartOf` detiene compositor, roles, apps `[V semántica]` | Directo | Igual que 1 | `[V]` | `[VM]` |
| 7 | Terceros y unidades D-Bus activadas | — | — | No: no declaran `PartOf` y PARALLAX no los gestiona | — | — | Pueden sobrevivir hasta que logind o el user manager terminen (`KillUserProcesses`, lingering) `[H]`/`[VM]`; T-KILL |

**Alternativas pendientes para los huecos (sin decidir):** que parallax-session.target o el compositor declaren `PropagatesStopTo=graphical-session.target` [V-p existencia de la directiva; uso no evaluado]; que el comando de sesión, al salir, pare explícitamente el target; que el compositor dependa con `BindsTo=` del ciclo del comando de sesión (p. ej. vía un scope del propio comando) [H]. Ninguna está evaluada; requieren decisión del arquitecto y prueba (T-SESSION-FAIL, T-GRAFO).

### 4.2 Resto de decisiones (la sección 4.3, ciclo de vida de la sesión, aparece antes por proximidad con D-24/D-28/D-29) (resumen; detalle en `revisiones/r06-sexta.md` §9 y r07/r08)

| ID | Decisión | Recomendación de revisiones | Responsable | Estado |
|---|---|---|---|---|
| D-01 | Agente polkit A (`fallback`) / B (proceso propio) | B, ventaja moderada (robustez, no seguridad) | **[P]** | Pendiente |
| D-02 | Capa del agente | C5 | Arquitecto | Pendiente |
| D-03 | Lanzamiento | Crate común + compositor mínimo | Arquitecto + seguridad | Pendiente |
| D-04 | Unidades de app `.service` / `.scope` | `.service` | Arquitecto + Linux | Pendiente |
| D-05 | Escapado de nombres de unidad | — | Evidencia VM | Pendiente |
| D-06 | Bus de diagd | — | Arquitecto + seguridad | Pendiente |
| D-07 | Ubicación XML | `docs/architecture/dbus/` | Arquitecto | Pendiente |
| D-08 | `vendor/` | — | **[P]** (bloquea Gate 0 workspace) | Pendiente |
| D-09 | `cargo vet` | — | **[P]** | Pendiente |
| D-10 | Spike del overlay | Autorizar en Fase 1 | **[P]** | Pendiente |
| D-11 | Dueño de `distribution/` | — | **[P]** | Pendiente |
| D-12 | `Session1` unificada | A | Arquitecto | Pendiente |
| D-13 | Texto de N-1 a N-12 | — | **[P]** (aportar) | Pendiente |
| D-14 | Renombrar `Session1` | Mantener | Arquitecto | Pendiente |
| D-15 | Código generado | Versionado + CI | Arquitecto + Rust | Pendiente |
| D-16 | Helper de credenciales | `libpolkit-agent-1` | Seguridad + **[P]** | Pendiente |
| D-17 | Excepción de lectura C3→C4 | Sí | Arquitecto | Pendiente |
| D-19 | Plazos del cierre de sesión | Por definir | Arquitecto | Pendiente |
| D-20 | Generación del `epoch` | Aleatoria 128 bits `[H]` | Arquitecto | Pendiente |
| D-21 | Nombre/fase del crate de lanzamiento | Crate común, Fase 3 | Arquitecto + Rust | Pendiente |
| D-23 | Cierre de sesión desde Locked | No ofrecerlo `[H]` | **[P]** | Pendiente |
| D-25 | Formato del nombre de scopes de rol | Nombre fijo por rol (exclusión por nombre `[VM]`) | Linux | Pendiente |
| D-26 | Arrancar Locked tras caída | Innecesaria para el compositor si D-24 = X; aplica a roles | **[P]** | Pendiente (condicionada a D-24) |

---

## 5. Riesgos de las revisiones (RV-xx) y correspondencia de identificadores

**Colisión detectada:** las revisiones usan `R-01`…`R-36`, que chocan visualmente con `R01`…`R28` de `riesgos.md` y **no significan lo mismo** (p. ej. `R-20` de las revisiones ≠ `R20` de `riesgos.md`). En la documentación permanente se usan **RV-NN**, con NN = número original de la revisión. `riesgos.md` conserva R01–R28 sin cambios.

| RV | Riesgo | Rev. origen → últ. | Relación con `riesgos.md` | Estado |
|---|---|---|---|---|
| RV-01 | Sin entrega de fd a un `.service` | 5 → 6 | R10 | Confirmado; mitigación R2 (D-18) |
| RV-02 | Coexistencia/desregistro de agentes polkit | 5 → 6 | R19 | Abierto `[VM]` |
| RV-08 | Compositor convertido en gestor de procesos | 5 | — | Mitigado en propuesta (C-20) |
| RV-11 | Hueco sin agente polkit durante reinicio (B) | 5 | R19 | Abierto |
| RV-12 | Suplantación de nombre D-Bus por el mismo UID | 5 → 6 | R25 | Límite reconocido en propuesta (pendiente D-22) |
| RV-13 | `DBusActivatable` escapa a resourced | 5 → 7 | — | Mitigado en propuesta (`GetUnitByPIDFD`) |
| RV-14 | GLib y `unsafe` por FFI polkit | 5 → 6 | R21 | Abierto |
| RV-15 | Apps que ignoran `close` bloquean el cierre | 5 | — | Abierto (modo forzado) |
| RV-16 | Sin Shell solo se lanza la terminal de recuperación | 5 | R10 | Comportamiento de emergencia propuesto |
| RV-17 | El compositor acumula política de reinicio | 6 | R03 | Catálogo fijo y límites |
| RV-18 | Apps lanzadas por terceros en el cgroup del solicitante | 6 | — | resourced no actúa sobre lo no identificado |
| RV-19 | Ventaja de seguridad de B sobrevalorada | 6 | — | Corregido en rev. 6 |
| RV-20 | `PIDs` en scopes: comportamiento en systemd 262.2 | 7 → 8 → 9 | R03 | **Reclasificado:** rev. 7 «no documentada» → rev. 8 «API documentada, sin verificar en Arch». Abierto `[VM]` |
| RV-21 | Agente polkit antiguo bloquea el nuevo | 7 → 9 | R19 | Abierto `[VM]` |
| RV-22 | Ventana antes de `PR_SET_DUMPABLE` | 7 → 9 | — | Abierto `[VM]` (Yama cubre no descendientes) |
| RV-23 | Imitación visual del diálogo `auth` | 7 → 9 | R19 | Diferible (diseño Fase 3) |
| RV-24 | Arrancar Locked tras cada caída empeora la UX | 7 → 8 | — | Sustituido por D-24 = X (si se aprueba) |
| RV-25 | Parada de `graphical-session.target` no se propaga | 7 → 9 | R03 | Propuesta fundada upstream; abierto en 262.2 `[VM]` |
| RV-26 | Roles huérfanos tras caída | 7 → 9 | R03 | Alcance redefinido (N9-4) |
| RV-27 | X pierde todas las ventanas y vuelve al login | 8 | R03, R17 | Pendiente de decisión (D-24) |
| RV-28 | Terceros sin `PartOf` sobreviven a la sesión | 8 → 9 | — | Límite reconocido en propuesta |
| RV-29 | Código de salida de greetd y cierre de PAM sin documentar | 8 → 10 | — | Abierto (T-GREETD) |
| RV-30 | `posix_spawn` requiere `unsafe` o dependencia | 8 → 10 | R21 | Pendiente D-27 |
| RV-31 | Bibliotecas C abren fd sin `CLOEXEC` | 9 → 10 | — | Mitigado en diseño (`addclosefrom_np`) `[VM]` |
| RV-32 | Lista blanca de entorno omite variables necesarias | 9 | — | Abierto (fallo funcional, no de seguridad) |
| RV-33 | Rechazo de segunda sesión inexplicable al usuario | 9 | — | Abierto (D-30) |
| RV-34 | Dependencia de glibc (extensión `_np`) | 10 | R21 | Abierto (T-GLIBC); requiere mención en ADR-0002 |
| RV-35 | Un hijo hereda el fd del lock sin `O_CLOEXEC` | 10 | — | Mitigado por regla + T-LOCK |
| RV-36 | Otro entorno no cooperativo del mismo usuario | 10 | — | Fuera de soporte v1 (D-29) |
| RV-37 | Caída del comando de sesión deja la sesión gráfica activa mientras greetd relanza el greeter; posible sesión desbloqueada sin autenticación | **Tarea 011** (análisis de §4.3, sin revisión de origen) | R03 | Abierto; T-SESSION-FAIL; alternativas en §4.3 |

RV-03 a RV-07, RV-09 y RV-10 proceden de la revisión 4: **no recuperables** (§7).

---

## 6. Contradicciones y correcciones entre revisiones

### 6.1 Correcciones de una revisión por otra (propuesta vigente en negrita)

| Tema | Antes | Después |
|---|---|---|
| Lanzamiento | rev. 4: el compositor lanza (`Session1.Launch`) | rev. 5: Shell coordina → rev. 6: biblioteca, módulo en `shell/shell/` → **rev. 7: crate común en `crates/`** (C-20b) |
| `Logout` | rev. 4: sustituido por logind | **rev. 5: vuelve a `Session1` como ciclo coordinado** |
| Supervisión del Shell | rev. 4: Shell como `.scope` sin `Restart=` | rev. 5: mantener `Restart=always` (R1) → **rev. 6: R2, R1 descartada** |
| Ventaja de polkit B | rev. 5: frontera de seguridad | **rev. 6: solo robustez/mantenimiento** |
| `PIDs` en scopes | rev. 7: no documentada (R-20 bloqueante) | **rev. 8: documentada en CONTROL_GROUP_INTERFACE** (C-23b) |
| Creación de procesos | rev. 7: `fork` en el compositor + `dumpable` antes de `exec` | **rev. 8: `posix_spawn` + intermediario; `dumpable` en el rol** (C-23c) |
| Propagación de parada | rev. 7: dudosa | **rev. 8: `StopWhenUnneeded=yes` upstream `[V]`** como mecanismo indirecto; Tarea 011: solo funciona si nada más mantiene el target, no cubre la caída del comando de sesión y no está validado (§4.3) |
| Garantía de bloqueo con X | rev. 8: «por construcción» | **Tarea 011: retirada**; pendiente de T-SESSION-FAIL (RV-37) |
| Descriptores | rev. 8: `CLOEXEC` + `close_range(…, CLOEXEC)` | rev. 9: orígenes ≥10, 0–2 explícitos, `close_range` en `main` → **rev. 10: `addclosefrom_np(5)` como última acción** |
| Exclusión de sesiones | rev. 9: consulta `ActiveState` | **rev. 10: `flock`** |
| Alcance de X | rev. 8: «evita apps huérfanas» | **rev. 9: solo lo que PARALLAX lanza** (N9-4) |
| greetd | rev. 8: no verificado (502) | **rev. 9: verificado `greetd(5)`/`(1)`; C-38 confirmado** |
| D-26 | rev. 7: necesaria | **rev. 8: innecesaria para el compositor con X** |

### 6.2 Contradicciones con documentos del repositorio (abiertas)

| Documento | Texto vigente (contrato) | Propuesta | Tratamiento en Tarea 010 |
|---|---|---|---|
| `arquitectura.md` §2 | Greeter = `parallax-greeter.service` | `command` de `[default_session]` `[V]` | Nota añadida; texto original intacto |
| `arquitectura.md` §2 | Unidad del compositor = `parallax-session.target` | `parallax-compositor.service` | Nota añadida |
| `arquitectura.md` §5 | systemd reinicia `parallax-shell` | Compositor reinicia roles (D-18) | Nota añadida |
| `arquitectura.md` §5 | `Session1.Launch` en compositor | Crate de lanzamiento (D-03) | Nota añadida |
| `arquitectura.md` §5 | Restauración tras caída sin alcance | Política X con alcance real (D-24) | Nota añadida |
| `ipc.md` §6.1, `dock.md`, `compositor.md` §10.2/§11.5 | `Launch`, reinicio del compositor | Ver C-20, C-37 | **No modificados**; trazados aquí |
| `modelo-seguridad.md` | Fila Shell / D-Bus sobreafirman | Límites del mismo UID | Nota añadida |
| `dependencias.md` regla 3 | «Verificado por `cargo deny`» | Filtro, no verificación | No modificado (C-24); trazado aquí |
| ADR-0015 Consecuencias | Relanzamiento «se define en Fase 2» con `OpenFile=` como opción | `OpenFile=` no sirve; R2 | Nota de propuesta añadida, estado sin cambio |
| `servicios.md` §17 | Árbol con `portals/`, `systemd/` en raíz | Árbol único (C-07) | No modificado |

---

## 7. Hallazgos que no pudieron recuperarse

| Elemento | Situación |
|---|---|
| **Revisión 4 completa** | Se entregó en la conversación de una sesión anterior y no se guardó en disco. Se conocen sus IDs (C-01–C-19, D-01–D-17, R-01–R-10) solo por las tablas de las revisiones 5 y 6. Sus argumentos y fuentes no están disponibles |
| C-15 (ADR-0011) | Sin contenido recuperable: «No verificable» |
| RV-03 a RV-07, RV-09, RV-10 | Solo existen en rev. 4 |
| N-1 a N-12 | Texto original nunca aportado; temas de N-3, N-4, N-6–N-9 reconstruidos por referencias. Requiere al propietario (D-13) |
| Fuentes no leídas | `greetd-ipc(7)`, código `worker.rs` de greetd, NEWS de glibc 2.34, `posix/spawn.h`, `proc_pid(5)`, man de Arch de `addclosefrom_np` (404) |

No se han inventado conclusiones para estos elementos.

---

## 8. Pruebas pendientes (todas **no autorizadas, no ejecutadas**)

Registradas también en [`spikes.md`](spikes.md) §T. Ejecutar cualquiera requiere autorización del propietario y VM conforme a ADR-0013.

| ID | Prueba | Decisión / hallazgo |
|---|---|---|
| T-FD | Descriptores trampa (bajo, alto, otro hilo, sin `CLOEXEC`) en el compositor; `/proc/<pid>/fd` del intermediario (0–4) y del rol (0–3); forzar fallo de `closefrom` → 127; `envp` efectivo | N9-1..3, N10-1, N10-7, D-22 |
| T-LOCK | 100 arranques concurrentes (exactamente uno gana); SIGKILL con lock tomado → recuperación y parada del residuo; otro entorno activo → rechazo; symlink/permisos del directorio de lock | D-29, N10-4..6 |
| T-GLIBC | Disponibilidad y comportamiento de `posix_spawn_file_actions_addclosefrom_np` en glibc 2.44 de Arch; equivalencia con `master` | N10-2, N10-8, D-27 |
| T-PIDS | `StartTransientUnit` con `PIDs` en systemd 262.2: tipo, PID muerto, garantía de `JobRemoved=done`, exclusión por nombre de scope | RV-20, D-18, D-25 |
| T-PIDFD | `GetUnitByPIDFD` en el user manager de Arch | C-23, C-28 |
| T-TRANS | `PartOf=` y `Slice=` como propiedades de unidades transitorias (scopes y services) | C-22 |
| T-GRAFO | `graphical-session.target` instalado (`StopWhenUnneeded`); cadena `BindsTo` → parada → `PartOf`; `systemd-analyze verify` sin ciclos | D-24, RV-25 |
| T-DUMP | `PR_SET_DUMPABLE` tras `execve`; `ptrace`/`pidfd_getfd` desde mismo UID; reapertura vía `/proc/<pid>/fd` | D-22, RV-22 |
| T-GREETD | Ciclo greetd + PAM + logind: código de salida, cierre de PAM, vuelta al greeter, reautenticación | D-28, RV-29 |
| T-KILL | `KillUserProcesses` efectivo; supervivencia de servicios D-Bus, `DBusActivatable` y unidades de terceros | D-24, RV-28 |
| T-SESSION-FAIL | Caída inesperada del compositor y del comando de sesión: estado de cada unidad, mecanismo de cada transición, fin del comando, vuelta al greeter, ausencia de sesión desbloqueada (diseño en `spikes.md` §T) | D-24, D-28, C-22, C-37, RV-37 |
| T-POLKIT | Registro, `fallback`, desregistro y desconexión del agente | D-01, RV-02, RV-21 |
| T-DBUSACT | Apps `DBusActivatable` e identificación por resourced | C-28 |

---

## 9. Lo que este documento no hace

- No cambia el estado de ningún ADR (todos `Propuesto`).
- No aprueba ninguna decisión D-xx ni P-xx.
- No autoriza pruebas, VM ni spikes.
- No declara Fase 0 completada ni inicia Fase 1.
