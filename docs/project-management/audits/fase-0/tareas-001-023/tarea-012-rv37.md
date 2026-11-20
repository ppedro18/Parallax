# Tarea 012 — Informe técnico dirigido a RV-37

- **Fecha:** 2026-11-19 · **Fase 0:** `IN_PROGRESS` · **ADR:** todos `Propuesto` · **Fase 1:** sin autorización.
- **Naturaleza:** investigación y diseño. **Propuesta pendiente de aprobación.** No se modificó el repositorio; este archivo está en el scratchpad.
- **Marcas:** `[V]` fuente oficial suficiente · `[V-p]` parcial · `[R]` repositorio · `[H]` inferencia · `[VM]` requiere prueba · `[P]` propietario.

---

## 1. Resumen del problema

RV-37 (consolidación §4.3, caso 4): si el comando de sesión muere, ninguna relación systemd detiene `parallax-session.target` ni el compositor; greetd vuelve a `[default_session]` y el greeter podría aparecer mientras la sesión anterior sigue viva y desbloqueada.

**Resultado de la investigación:** la seguridad de la presentación **no debe ni necesita** apoyarse en el proceso que falla. Hay una cadena de mecanismos ajenos al comando de sesión que impide que la sesión anterior siga **presentando** o **recibiendo entrada** cuando aparece el greeter:

1. greetd solo relanza el greeter **después** de recoger al *session worker*, y el worker solo sale tras `pam_close_session` y `pam_end` `[V-p código greetd 0.10.3, mirror GitHub]`.
2. El greeter corre en su propia sesión logind del mismo seat. Al activarse, logind desactiva la sesión anterior: hace **`drmDropMaster`** en el fd DRM y **`EVIOCREVOKE`** en los fds evdev, sin cooperación del controlador `[V-p systemd v257 logind-session-device.c]`.
3. DRM master es exclusivo: si el greeter puede presentar, la sesión anterior no es master `[H, semántica DRM]`.

La **limpieza de procesos** (compositor, roles, apps) sigue necesitando mecanismos propios, pero ya no es una condición de seguridad sino de higiene. Esto es lo que separa el diseño propuesto del de la Tarea 011.

---

## 2. Causas y condiciones del fallo

| Condición | Origen |
|---|---|
| El compositor corre en el **user manager** (`user@.service`), fuera del scope de la sesión logind | Diseño propuesto, rev. 8 §6.2 |
| No existe en systemd una dependencia entre una unidad de usuario y una sesión logind | `[H]`; no se encontró ninguna directiva en las páginas leídas |
| El comando de sesión no tiene relación systemd con las unidades que arranca | Consolidación §4.3 |
| greetd descarta el código de salida y trata igual salida normal, error o señal | `[V-p]` `worker.rs`: `waitpid(child, None)` y `Ok(_) => break` |
| greetd no mata grupos de procesos; solo `SIGTERM`/`SIGKILL` al worker y a su hijo | `[V-p]` `interface.rs`: `SessionChild::term()/kill()` |
| La limpieza propuesta en rev. 10 solo ocurre en el **siguiente** inicio | Consolidación D-29 |
| **Error adicional en el diseño vigente:** la limpieza de rev. 10 hace `StopUnit(parallax-session.target)`. Por la dirección de `BindsTo`, eso **no** detiene el compositor | Consolidación §4.3, tabla de direcciones |

---

## 3. Conceptos que no son equivalentes

| Estado | Qué significa | Quién lo establece |
|---|---|---|
| Proceso terminado | El PID ya no existe | Kernel |
| Unidad inactiva | systemd considera la unidad parada | systemd |
| Target inactivo | No implica que sus dependencias estén paradas (dirección de `BindsTo`) | systemd |
| Sesión logind terminada | Sesión eliminada (`SessionRemoved`) o en `closing` | logind |
| **Sesión gráfica inaccesible** | Sin DRM master y con los fds evdev revocados | logind + kernel |
| Sesión bloqueada | Estado lógico del compositor (`ext-session-lock`) | Compositor |
| **Sesión segura para presentar el greeter** | Sesión gráfica inaccesible **y** worker de greetd recogido (PAM cerrado) | §8 de este informe |

La propiedad de seguridad buscada es **«sesión gráfica inaccesible»**, no «procesos terminados».

---

## 4. Alternativas comparadas

| | A. Comando de sesión como supervisor | B. Comando dentro de un scope/unidad vinculada | C. greetd ejecuta el compositor dentro del scope de sesión (G-a) | D. Supervisor independiente (unidad systemd) | **E. Arbitraje de dispositivos de logind + limpieza en capas (recomendada)** |
|---|---|---|---|---|---|
| ¿Depende del proceso que falla? | **Sí** | La parada sí, si el comando es el único proceso del scope | No: `KillUserProcesses` mata el scope | No para detectar; sí depende de su propia supervisión | **No** para la seguridad; parcialmente para la limpieza |
| Mecanismo documentado | SIGKILL no se puede capturar | Ninguna directiva vincula el user manager a un scope de sesión `[H]` | `[V]` `KillUserProcesses=yes` por defecto upstream | Unidad con `BindsTo=` a… ¿qué? No hay unidad que represente la sesión logind en el user manager `[H]` | `[V-p]` drop master + `EVIOCREVOKE`; `[V-p]` orden de greetd |
| Coste | Ninguno | Medio | Pierde el grafo systemd de la sesión (rev. 8) | Un proceso permanente más | Ninguno nuevo: logind y greetd ya existen |
| Veredicto | **Rechazada como garantía**; útil como capa | Insuficiente sola | Viable; contradice D-28 (G-c) | No justificada: no detecta nada que E no cubra | **Recomendada** |

### 4.1 Sobre las directivas systemd pedidas (B)

| Directiva | Dirección y efecto `[V systemd.unit(5)]` | ¿Resuelve RV-37? |
|---|---|---|
| `BindsTo=` | El que la declara cae si la otra cae | Necesitaría una unidad de usuario que represente la vida del comando. El comando no es una unidad del user manager |
| `PartOf=` | La parada va de la unidad listada a la que declara | Igual |
| `Requires=`, `Wants=` | Arrastran arranque; `Requires` propaga paradas explícitas | No |
| `PropagatesStopTo=` | Propaga peticiones de **parada** (no fallos) | No cubre la muerte del comando |
| `StopWhenUnneeded=` | Depende de que nada más la referencie | No es garantía (consolidación §4.3) |
| `KillMode=` | Qué se mata al parar **esa** unidad | No dispara la parada |
| `ExecStop=`, `ExecStopPost=` | Se ejecutan al parar **esa** unidad | No dispara la parada |
| `OnFailure=` | Arranca otra unidad cuando esta falla | Requiere que el comando sea una unidad cuyo fallo vea el user manager |
| Scope transitorio del comando en el **user manager** | Posible con `StartTransientUnit`+`PIDs` `[V]` | Si el comando muere, el scope queda vacío y se libera `[V systemd.scope(5)]`. Un servicio con `BindsTo=` sobre ese scope caería `[H]` → **variante B1**, evaluada abajo |

**Variante B1:** el comando de sesión se mete en un scope del user manager, `parallax-session-cmd.scope`* (nombre no aceptado), y `parallax-compositor.service` declara `BindsTo=` sobre él.

- **A favor:** si el comando muere, el scope se vacía y el compositor cae. Eso no depende del comando.
- **En contra:**
  - El comando tendría que estar a la vez en el scope de la sesión logind (system manager) y en un scope del user manager. Un proceso pertenece a **un solo cgroup** `[V cgroups v2]`; moverlo al user manager lo saca del scope de logind y cambia la contabilidad de la sesión `[H]` `[VM]`.
  - Que `BindsTo=` sobre un scope transitorio se dispare al liberarse el scope: `[VM]`.
- **Conclusión:** B1 queda como **alternativa a probar**, no como base.

---

## 5. Diseño recomendado (E) — propuesta

### 5.1 Capa de seguridad (no depende del comando ni del compositor)

| # | Requisito | Evidencia |
|---|---|---|
| S1 | El compositor obtiene **todos** los dispositivos DRM y de entrada por `TakeControl`/`TakeDevice` de **su** sesión logind (libseat backend logind). Nunca abre `/dev/dri` ni `/dev/input` directamente | `[R]` `compositor.md` §3: «Adquirir la sesión mediante libseat/logind». `[V]` login1: logind silencia el fd si la sesión está inactiva |
| S2 | La sesión anterior queda inaccesible cuando se activa otra en el seat: logind hace `drmDropMaster` y `EVIOCREVOKE` **sin cooperación** del controlador. Después notifica `PauseDevice("force")` | `[V-p]` v257 `session_device_stop`, `session_device_pause_all` |
| S3 | Al liberarse los dispositivos de la sesión eliminada ocurre lo mismo (stop, luego cierre) | `[V-p]` v257 `session_device_free` |
| S4 | El greeter solo se ejecuta tras recoger al worker, que solo sale tras `close_session`/`end` de PAM | `[V-p]` greetd 0.10.3 `check_children()`, `worker()` |
| S5 | El greeter corre en su propia sesión logind en el mismo seat; al presentar ya es DRM master, así que la sesión anterior no lo es | `[H]` semántica DRM; `[VM]` |

**Lo que S1–S5 no garantizan (documentado como límite):**

- **Último fotograma visible.** Tras el drop de master, el CRTC puede seguir mostrando el último framebuffer hasta que el greeter haga modeset `[H]`. Es exposición de información, sin interacción. Si el greeter no arranca, la imagen puede quedarse fija (RV-38).
- **Recuperar master.** Desde Linux 5.8, un proceso sin `CAP_SYS_ADMIN` puede volver a ser master si fue quien abrió el fd y no hay otro master `[H, kernel no leído]`. Aquí el fd lo abre logind, no el compositor, por lo que la excepción no debería aplicarse `[H]`. Prueba: T-DRM.
- **Otros procesos del mismo UID.** Si vuelve a entrar el mismo usuario, recupera las ACL `uaccess` de `/dev/input` y otro proceso suyo podría abrirlo. Es la amenaza T8, fuera del modelo (D-22).
- **Carrera de `TakeControl`.** Solo comprueba el UID `[V login1]`. Un compositor antiguo aún vivo podría llamar a `TakeControl` sobre la **nueva** sesión antes que el compositor nuevo (RV-39). Mitigación: la limpieza de 5.2 termina **antes** de arrancar el compositor nuevo, y el compositor sale al perder su sesión (capa L1).

### 5.2 Capa de limpieza de procesos (higiene, en capas)

| Capa | Mecanismo | ¿Depende del proceso que falló? | Evidencia |
|---|---|---|---|
| L1 | El compositor vigila **su** sesión logind (`SessionRemoved`, `State`, `PauseDevice("force")` sin `ResumeDevice` con la sesión en `closing`). Ante eso, sale con fallo; `BindsTo` detiene `parallax-session.target` y el resto según §4.3 | No depende del comando; sí del compositor (voluntaria) | `[V]` señales login1; comportamiento `[VM]` |
| L2 | **Parada del user manager:** al terminar la última sesión del usuario, `user@.service` se detiene tras `UserStopDelaySec` (10 s por defecto) si no hay *linger*. Mata **todo**, terceros incluidos | **No** | `[V]` logind.conf; requiere linger desactivado y una sola sesión (D-29) |
| L3 | **Limpieza en el siguiente inicio:** el comando, con el `flock` tomado y **antes** de arrancar nada, detiene por **nombre**: `parallax-compositor.service`, scopes de rol y `app-parallax-*`. Corrige el error de rev. 10: parar solo el target **no** detiene el compositor. Espera `inactive`; si pasa el timeout, `KillUnit(all, SIGKILL)`; si aun así falla, rechaza el inicio | No depende del proceso que falló | `[V]` API systemd1; `[VM]` |
| L4 | Capa A como optimización: el comando captura SIGTERM y para las unidades antes de salir | Sí; **no cuenta como garantía** | — |

---

## 6. Grafo de dependencias recomendado

Es el de rev. 8 §6.2 con tres cambios propuestos (no incorporados):

1. El compositor declara `Restart=no` (ya estaba en X) y **sale** cuando pierde su sesión logind (L1).
2. La limpieza L3 actúa sobre **unidades concretas**, no sobre el target.
3. Requisito de despliegue: `linger` **desactivado** para usuarios de PARALLAX en v1 y `UserStopDelaySec` sin valor `infinity` (decisión D-31 propuesta).

No se añade ninguna unidad ni proceso permanente. La variante B1 queda como alternativa a probar (T-BIND).

---

## 7. Máquina de estados del ciclo de terminación

```text
ACTIVE ──(logout | caída compositor | caída comando | SIGTERM comando)──► TERMINATING
TERMINATING:
  T1 greetd recoge al worker (PAM cerrado)            [greetd]
  T2 greeter inicia su sesión logind y se activa      [greetd + logind]
  T3 logind: drop master + EVIOCREVOKE de la anterior ──► INACCESSIBLE   [logind/kernel]
INACCESSIBLE ──(L1 compositor sale | L2 user@ stop)──► CLEANED
INACCESSIBLE ──(nuevo login antes de CLEANED)──► CLEANUP_ON_START (L3)
CLEANUP_ON_START ──ok──► STARTING (sesión nueva)
CLEANUP_ON_START ──timeout tras SIGKILL──► REFUSED (salida ≠ 0; greetd vuelve al greeter; D-30)
```

El greeter puede presentarse a partir de **T2** porque T3 ocurre en la activación (S2). No es necesario esperar a CLEANED.

---

## 8. Criterios verificables para permitir el greeter

Mínimo para considerar que la sesión anterior **no puede presentar una superficie desbloqueada ni recibir interacción**:

1. El worker de greetd de la sesión anterior está recogido (traza de greetd).
2. La sesión logind anterior está eliminada, o en `closing`/`online` y **no** es `ActiveSession` del seat (`loginctl`).
3. El fd DRM del compositor anterior no es master (`DRM_IOCTL_AUTH_MAGIC`/`drmIsMaster` = falso en la prueba).
4. Sus fds evdev devuelven `ENODEV` (revocados).
5. El compositor anterior no tiene ningún fd DRM/evdev abierto **fuera** de los obtenidos con `TakeDevice` (`/proc/<pid>/fd`).

Que estos puntos se cumplan **solo por construcción no está demostrado**. Se apoyan en código de logind v257 leído parcialmente y en greetd 0.10.3 leído desde el mirror de GitHub; Arch usa systemd 262.2. Hay que comprobarlos en VM (T-SESSION-FAIL, T-DRM).

---

## 9. Matriz de escenarios

Columnas: **Detecta** · **Inicia la terminación** · **Qué se detiene** · **Terceros** · **Confirma el fin** · **Lock liberado** · **Greeter** · **Si falla la limpieza**.

| # | Escenario | Detecta | Inicia | Se detiene | Terceros | Confirma | Lock | Greeter | Si falla |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Logout normal | — | Compositor (C-22) → parar el compositor por nombre | Compositor (→ `BindsTo` target), roles, apps | Sobreviven hasta L2 | Comando: compositor `inactive` | Al salir el comando | Tras recoger al worker | L2/L3 |
| 2 | Caída del compositor | systemd | `BindsTo` → target | Target; roles y apps vía `PartOf` `[H]` o L2 | Hasta L2 | Comando | Al salir | Ídem | L2/L3 |
| 3 | Caída del comando | greetd (`waitpid`) | **Nadie** en el user manager; logind desactiva la sesión al activarse el greeter (S2) | Nada de inmediato; L1, después L2 | Hasta L2 | logind (sesión eliminada) | Kernel, al morir el proceso `[V flock(2)]` | Tras PAM close; sesión anterior **inaccesible** desde T3 | L3 en el siguiente inicio |
| 4 | SIGTERM al comando | Comando (handler) | L4: el comando para las unidades | Como en 1 | Hasta L2 | Comando | Al salir | Ídem | Igual que 3 |
| 5 | SIGKILL al comando | greetd | Igual que 3 | Igual que 3 | Hasta L2 | logind | Kernel | Igual que 3 | Igual que 3 |
| 6 | Caen a la vez comando y compositor | greetd + systemd | `BindsTo` + S2 | Como en 2 | Hasta L2 | logind | Kernel | Igual que 3 | L2/L3 |
| 7 | Una unidad PARALLAX no se detiene | L3 (timeout) | L3 → `KillUnit SIGKILL` | Esa unidad | — | systemd (`inactive`) | — | Ya presentado (seguro por S2) | REFUSED en el nuevo inicio (D-30) |
| 8 | Timeout de terminación de un proceso | systemd (`TimeoutStopSec`) | systemd → SIGKILL (`KillMode=control-group`) | El cgroup | — | systemd | — | No depende | Proceso en estado D: RV-40 |
| 9 | El user manager sigue activo (linger o `UserStopDelaySec` sin expirar) | — | L1/L3 | Lo que es de PARALLAX | **Sobreviven** | L3 | — | Seguro por S2 | Con linger, solo L1/L3 (D-31) |
| 10 | Otra unidad mantiene `graphical-session.target` | — | L3 por nombre, no por target | Unidades PARALLAX | Sobreviven | L3 | — | Seguro por S2 | Si es otro entorno, rechazo (D-29) |
| 11 | Nuevo login inmediato tras una caída | Comando nuevo (`flock` + L3) | L3 **antes** de arrancar | Residuos PARALLAX | Hasta L2 (que se cancela con el nuevo login) | L3 | Se toma al empezar | — | REFUSED; carrera de `TakeControl` mitigada (RV-39) |
| 12 | Apagado durante la limpieza | systemd (shutdown) | systemd | Todo | Todo | — | — | No hay greeter | greetd con SIGTERM no hace PAM teardown `[V-p]`; irrelevante al apagar |

---

## 10. Fallos parciales y timeouts

| Caso | Tratamiento propuesto |
|---|---|
| L1 no ocurre (compositor colgado) | La seguridad no depende de L1 (S2). Lo cubren L2 y L3 |
| L2 no ocurre (linger, otra sesión) | Lo cubre L3. Requisito D-31 |
| L3: `StopUnit` no termina | Timeout (valor por definir) → `KillUnit(all, SIGKILL)` → nuevo timeout → REFUSED |
| Proceso en estado D (no muere ni con SIGKILL) | Sin solución documentada. RV-40, aceptado como límite del kernel |
| El greeter no arranca | La sesión anterior sigue inaccesible si ya no es la activa del seat. Con el seat sin sesión activa, la anterior **queda activa** `[H]`. Si el worker ya salió, la sesión está en `closing` o eliminada y sus dispositivos liberados (S3) `[V-p]`; confirmar en VM |
| greetd muere | El worker recibe `PDEATHSIG` y salta el cierre de PAM `[H]`. La sesión logind termina al vaciarse el scope `[H]`. Sin greetd no aparece ningún greeter |

---

## 11. Riesgos residuales (numeración propuesta, a continuación de RV-37)

| ID | Riesgo |
|---|---|
| RV-37 | Pasa de «sesión posiblemente desbloqueada» a «procesos residuales sin acceso a pantalla ni entrada» **si** se validan S1–S5 |
| RV-38 | Último fotograma de la sesión anterior visible hasta el modeset del greeter |
| RV-39 | Carrera de `TakeControl` entre un compositor antiguo vivo y la sesión nueva del mismo usuario |
| RV-40 | Procesos que no terminan (estado D) impiden la limpieza L3 → rechazo del inicio |
| RV-41 | La afirmación S2 se apoya en logind v257 (lectura parcial) y no en 262.2 de Arch |
| RV-42 | greetd 0.10.3 se leyó del mirror de GitHub; la coincidencia con el paquete de Arch no está verificada |

---

## 12. Fuentes y límites de la evidencia

| Fuente | Versión | Qué demuestra | Qué no demuestra |
|---|---|---|---|
| greetd `worker.rs`, `interface.rs`, `context.rs`, `server.rs` | Tag 0.10.3, mirror GitHub (sr.ht devolvió 502) `[V-p]` | `setsid`, PAM open/close/end, `waitpid` que ignora el estado, greeter tras recoger al worker, sin `killpg` | Igualdad con el paquete de Arch; `pam/session.rs` no leído |
| org.freedesktop.login1(5) | Rama «latest», que indica systemd 262 `[V]` | `TakeControl` por UID; mute de fds inactivos; tipos de `PauseDevice`; `KillSession`/`TerminateSession` | Momento exacto de `closing` o eliminación |
| logind.conf(5) | latest `[V]` | `KillUserProcesses` (yes upstream), `UserStopDelaySec`=10 s, linger | Valor efectivo en Arch |
| `logind-session-device.c` | v257 `[V-p]` | drop master y `EVIOCREVOKE` sin cooperación; stop antes de free | Comportamiento en 262.2; `logind-session.c` y `logind-session-dbus.c` no leídos |
| systemd.unit(5), systemd.scope(5), flock(2) | Revisiones anteriores `[V]` | Direcciones de las dependencias; liberación de scopes; flock | — |
| Kernel DRM (`drm_master_check_perm`), semántica de `EVIOCREVOKE` sobre fds duplicados | **No leídos** `[H]` | — | S5, RV-39 |
| Arch pam_systemd | No leído | — | Liberación de la sesión en `close_session` |

**Lo que no debe confundirse aquí:** las APIs están documentadas; la versión 262.2 no se ha comprobado; ningún comportamiento se ha observado en Arch ni validado con prueba.

---

## 13. Recomendación técnica (propuesta)

Adoptar **E**: la seguridad la da el arbitraje de dispositivos de logind y el orden de greetd, y la limpieza se hace en capas L1–L3, con L4 solo como optimización.

**Por qué:**

- Es la única alternativa cuya garantía de seguridad **no depende** del comando, del compositor ni de la propagación de systemd.
- No añade procesos permanentes.
- Mantiene G-c (D-28).

**Condiciones:**

- S1 obligatorio en el compositor.
- L3 corregido para parar unidades por nombre.
- Decisión D-31 sobre linger.
- Pruebas de §15.

---

## 14. Propuesta de actualización de trazabilidad (no aplicada)

| Documento | Cambio exacto | Motivo |
|---|---|---|
| `consolidacion-fase0.md` §5, RV-37 | Estado: «Mitigación propuesta (Tarea 012, alternativa E): seguridad por arbitraje de dispositivos logind (S1–S5) + limpieza L1–L3; pendiente T-SESSION-FAIL y T-DRM» | Refleja la propuesta sin darla por validada |
| `consolidacion-fase0.md` §5 | Añadir RV-38 a RV-42 (§11) | Riesgos nuevos |
| `consolidacion-fase0.md` §4.3, fila 4 | Detecta: greetd; seguridad: logind (S2); limpieza: L1/L2/L3; sustituir «Nadie…» por esa cadena con marcas `[V-p]`/`[VM]` | Escenario 4 |
| `consolidacion-fase0.md` §4.1 D-29 | La limpieza de residuos detiene **unidades por nombre**, no `parallax-session.target` | Error detectado (§2) |
| `consolidacion-fase0.md` §4.1 D-24 | Añadir que la inaccesibilidad de la sesión anterior se apoya en logind, no en el grafo; incertidumbres RV-38/39/41 | Precisión |
| `consolidacion-fase0.md` §4.1 D-28 | Añadir alternativa C (G-a, compositor en el scope de sesión) y variante B1 como alternativas evaluadas | Alternativas nuevas |
| `consolidacion-fase0.md` §4 | Nueva **D-31 [P]**: exigir linger desactivado y `UserStopDelaySec` finito en v1 | Condición de L2 |
| `spikes.md` T-SESSION-FAIL | Añadir a S5 la recogida de los criterios §8.1–5; nuevas filas S8 (login inmediato tras SIGKILL, carrera de `TakeControl`) y S9 (el greeter no arranca: estado del CRTC y de la sesión) | Cubre la nueva garantía |
| `spikes.md` | Nuevas T-DRM (drop master, imposibilidad de recuperarlo, `EVIOCREVOKE` sobre el fd entregado) y T-BIND (variante B1) | §15 |
| `specifications/compositor.md` §3/§11.5 | (Tras decisión) S1 como requisito y L1 como comportamiento | Contrato futuro |
| `architecture/arquitectura.md` §5 | (Tras decisión) flujo de terminación §7 | Contrato futuro |

---

## 15. Pruebas en VM necesarias (no autorizadas)

| ID | Qué valida |
|---|---|
| T-SESSION-FAIL (ampliada) | Escenarios 1–12 y criterios §8 |
| T-DRM (nueva) | Tras desactivar la sesión, el fd del compositor no es master, no puede recuperarlo y los fds evdev devuelven `ENODEV`; estado del CRTC (RV-38) |
| T-BIND (nueva) | Variante B1: `BindsTo=` sobre un scope transitorio vaciado; efecto de mover el comando fuera del scope de logind |
| T-GREETD | Coincidencia de 0.10.3 en Arch; orden worker → PAM → greeter; `pam_systemd` en el cierre |
| T-LOCK | Con L3 corregido |

---

## 16. Veredicto: `RESUELTO EN DISEÑO, PENDIENTE DE VALIDACIÓN`

- **Hay un diseño coherente (E)** en el que la presentación del greeter no depende del proceso que falló.
- **Depende de pruebas no ejecutadas:** T-SESSION-FAIL, T-DRM, T-GREETD y T-LOCK.
- **Depende de una decisión del propietario:** D-31 sobre linger. Sin linger desactivado, la limpieza completa depende solo de L1 y L3, aunque la inaccesibilidad (S2) se mantiene.
- **Si T-DRM muestra que la sesión anterior conserva master o entrada, RV-37 vuelve a «NO RESUELTO».**

No se modificó el repositorio. Ningún ADR cambió (todos `Propuesto`). La Fase 0 sigue `IN_PROGRESS` y la Fase 1 no está autorizada.
