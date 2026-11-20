# Tarea 020 — Cierre de evidencia sobre descriptores, BrlTTY y GC de logind

- **Fecha:** 2026-11-19
- **Fase 0:** `IN_PROGRESS`
- **ADR:** todos `Propuesto`
- **Fase 1:** sin autorización
- **Naturaleza:** solo lectura; propuesta pendiente de aprobación. Este informe está en el scratchpad y no se ha incorporado al repositorio.
- **Marcas de evidencia:** `[V]` fuente suficiente · `[V-p]` evidencia parcial · `[R]` repositorio · `[H]` inferencia · `[VM]` requiere máquina real · `[P]` decisión del propietario

---

## 1. Resumen ejecutivo

**Veredicto: `EVIDENCIA PARCIAL; VALIDACIÓN EN MÁQUINA PENDIENTE`** (§14).

**Cambio de método.** Por primera vez el código fuente se leyó **en local**, en clones parciales de los **tags exactos** (Linux v7.3, systemd v262.2, BRLTTY-6.9.1), sin herramienta web intermedia. Las citas de esta tarea son **literales `[V]`** respecto a upstream. Siguen siendo upstream: el comportamiento en Arch y en ejecución no está probado.

**Hallazgos que cierran lagunas:**

1. **DRM.**
   - `drm_master_release` se ejecuta dentro de `drm_file_free`, que llama `drm_close_helper` desde `drm_release`. `drm_release` es el `.release` de `file_operations`, y el VFS solo lo invoca al soltarse **la última referencia** a la descripción de archivo abierta `[V código DRM + H VFS]`.
   - Por tanto, **un hijo que conserve una copia del fd mantiene el master** tras la muerte del compositor.
   - En cambio, `DROP_MASTER` ejecutado sobre **cualquier** copia actúa sobre el `drm_file` compartido. Si logind lo ejecuta sobre su copia, todas las copias pierden el master `[V]`.
2. **evdev.**
   - El grab pertenece al **`evdev_client`** (`file->private_data`), no al proceso `[V]`.
   - `evdev_release`, que hace el ungrab, solo se ejecuta al soltarse la última referencia. **Un hijo con una copia hereda el grab y sigue leyendo** `[V + H VFS]`.
   - `EVIOCREVOKE` sobre cualquier copia marca el cliente compartido (`revoked=true`), suelta el grab y produce `ENODEV` en todas las copias `[V]`.
3. **logind.**
   - `session_device_stop` hace `sd_drmdropmaster`/`sd_eviocrevoke` sobre **su** `sd->fd`. Ese es el mismo fd que entrega en `TakeDevice` (`sd_bus_reply_method_return(message, "hb", sd->fd, …)`), así que la revocación **alcanza a las copias del compositor y de sus hijos** `[V + H paso de fd por SCM_RIGHTS]`.
   - Los fds abiertos directamente **no** están cubiertos.
4. **BrlTTY.** El recorrido del pegado queda **completo en el código**:
   - `cpbPaste` → `pasteScreenCharacters` → `insertScreenKey` → `currentScreen->insertKey`.
   - En el driver Linux, `insertKey_LinuxScreen` consulta `KDGKBMODE`:
     - con `K_XLATE`/`K_UNICODE` inyecta con **`TIOCSTI`**;
     - con `K_RAW`/`K_MEDIUMRAW`/`K_OFF` usa `insertKeyCode` y, fuera del modo raw, el **teclado uinput** `[V]`.
   - Un compositor Wayland pone la VT en `K_OFF` `[H]`. **En ese caso el pegado de brltty llega como teclado uinput y lo recibe el dueño del grab, es decir, el greeter** `[V + H]`.
   - **RV-56 queda parcialmente confirmado en el código.** Requiere tres cosas: brltty activo con el driver de pantalla Linux, un cliente BrlAPI que escriba el portapapeles y un usuario que pulse PASTE.
5. **logind GC.**
   - **Una sesión `closing` sigue fijando al usuario.** `user_pinned_by_sessions` recorre `u->sessions` sin filtrar por estado, y la sesión solo se quita de esa lista en `session_free` `[V]`.
   - `session_may_gc` devuelve falso mientras el líder viva **o el scope esté activo** `[V]`.
   - **Consecuencia:** los procesos supervivientes en el scope impiden que se detenga `user@` (RV-55 reforzado).

**RV-37 sigue abierto.**

---

## 2. Alcance y fuentes

| Fuente | Ref | Commit del checkout | Archivos |
|---|---|---|---|
| Linux | tag `v7.3` | `c03c1a14507d…` | `drivers/gpu/drm/drm_{file,auth,ioctl,gem,…}.c`, `drivers/input/{evdev,input}.c`, `drivers/input/misc/uinput.c`, `drivers/tty/tty_io.c` |
| systemd | tag `v262.2` (el mismo que usa el PKGBUILD de Arch 262.2-1, Tarea 017) | `63511d7283d7…` | `src/login/*` |
| BRLTTY | tag `BRLTTY-6.9.1` (el mismo tarball que usa el PKGBUILD de Arch 6.9.1-3; no hay parches sobre estos archivos, Tarea 017) | `4fddd533bf60…` | `Programs/*`, `Drivers/Screen/Linux/*`, `Headers/*` |

**Ubicación:** `scratchpad/src/`, clones parciales (`--depth 1 --sparse`). No se ejecutó nada.

**No leído:**
- VFS (`fs/file_table.c` `fput`, `fs/file.c`), `net/unix` (`SCM_RIGHTS`), el VT del kernel (`K_OFF`, `legacy_tiocsti` en `tty_io.c`, que está clonado pero no se revisó).
- Smithay/libseat, para saber si ponen `K_OFF`.
- El kernel concreto de Arch.

**Equivalencia con Arch:**
- systemd y brltty: **mismo origen** que los paquetes, con los parches revisados `[V]`.
- Linux: **el kernel de Arch es desconocido** (RV-46).

---

## 3. DRM: herencia, referencias y master

| # | Pregunta | Respuesta | Evidencia |
|---|---|---|---|
| 1 | ¿Quién posee el master? | `struct drm_file` (`file_priv->master`, `is_master`); `dev->master` apunta al master activo | `drm_auth.c` `[V]` |
| 2 | ¿Qué se comparte con `fork`, `dup`, `SCM_RIGHTS`? | La `struct file` y, con ella, `private_data` = `drm_file` | `drm_release`: `struct drm_file *file_priv = filp->private_data;` `[V]` + VFS `[H]` |
| 3 | ¿Qué pasa al cerrar una referencia si quedan otras? | Nada en DRM: `.release` no se invoca | `[H VFS]` |
| 4 | ¿Cuándo exactamente se libera el master? | `drm_release` → `drm_close_helper` → `drm_file_free`, que ejecuta en este orden: `drm_events_release`, `drm_fb_release`, `drm_gem_release` y, si es primario, **`drm_master_release`**. Esta llama a `drm_drop_master` si el archivo es el master actual, revoca los leases y hace `drm_master_put` | `drm_file.c` líneas 250–264; `drm_auth.c` líneas 337–363 `[V]` |
| 5 | ¿Puede un hijo seguir haciendo modeset o flips tras morir el compositor? | **Sí**, si conserva una copia de una descripción que es master | `[V]` + `[H VFS]` |
| 6 | ¿`DROP_MASTER` sobre otra copia elimina la capacidad? | **Sí**: actúa sobre el mismo `drm_file`; los ioctls con `DRM_MASTER` dan `-EACCES` (Tarea 013) | `[V]` |
| 7 | ¿Y si falla? | logind ignora el resultado (`sd_drmdropmaster(sd->fd);` sin comprobarlo) y el hijo sigue siendo master | `logind-session-device.c` `[V]` |
| 8 | ¿Puede el hijo recuperar el master? | No: `drm_master_check_perm` exige el tgid del opener (logind) o `CAP_SYS_ADMIN` | Tarea 013 `[V-p]`; coherente con lo leído |

**Recursos y capacidad:**

| Recurso | ¿Qué lo mantiene vivo? | ¿Qué permite modificar la pantalla? |
|---|---|---|
| Nodo primario + master | Cualquier referencia a la descripción | Ioctls `DRM_MASTER` (SETCRTC, ATOMIC, PAGE_FLIP…) |
| Nodo primario sin master | Ídem | Nada en la presentación |
| Render node | Ídem | Nada (sin modeset) |
| GEM | Handles del `drm_file`, `drm_gem_release` al liberarse; mmaps y dmabufs aparte `[H]` | Solo si un plano lo escanea |
| FB | `drm_fb_release` al liberarse el `drm_file` (los quita de los planos, Tarea 015) | Lo que esté en pantalla |
| mmap | El VMA mantiene el GEM `[H]` | Escribir en el buffer; solo es visible si está en pantalla |
| DMA-BUF exportado | El fd del dmabuf | Igual |

---

## 4. evdev: herencia, grab y revocación

| # | Pregunta | Respuesta | Evidencia |
|---|---|---|---|
| 1 | ¿El grab es del cliente o del proceso? | **Del cliente**: `rcu_assign_pointer(evdev->grab, client)` | `evdev_grab` `[V]` |
| 2 | ¿Qué pasa si el proceso original termina y otro conserva la referencia? | `evdev_release` (ungrab) **no** se ejecuta mientras quede alguna referencia | `[V]` + `[H VFS]` |
| 3 | ¿El hijo puede leer? | Sí, si no hay revocación | `[V]` |
| 4 | ¿Conserva el grab? | Sí: es el mismo `client` | `[V]` |
| 5 | ¿`EVIOCREVOKE` sobre otra copia afecta a todas? | Sí: `client->revoked = true; evdev_ungrab(...)` | `evdev_revoke` `[V]` |
| 6 | ¿Qué devuelven las operaciones tras la revocación? | `-ENODEV` en read, write e ioctl (`!evdev->exist \|\| client->revoked`) | `[V]` |
| 7 | ¿Cerrar frente a revocar? | Cerrar baja el contador; revocar desactiva el cliente para todas las copias | `[V]` |
| 8 | ¿Cuándo recibe el greeter `EBUSY`? | Cuando `evdev->grab` ya está ocupado (un cliente heredado vivo, otro proceso) o `input_grab_device` lo está (otro handler) | `evdev_grab` `[V]` |

---

## 5. logind: referencias compartidas y revocación

| Elemento | Hecho | Evidencia |
|---|---|---|
| fd que guarda logind | `sd->fd` (la apertura de `session_device_open`) | `[V]` |
| fd que recibe el compositor | El **mismo** `sd->fd`, enviado en la respuesta D-Bus (`"hb", sd->fd, …`), es decir, otra referencia a la misma descripción | `logind-session-dbus.c` línea 602 `[V]` + `[H SCM_RIGHTS]` |
| Desactivación | `session_device_stop`: DRM → `sd_drmdropmaster(sd->fd)`; evdev → `sd_eviocrevoke`; hidraw → `sd_hidiocrevoke`; desconocido → nada | `[V]` |
| ¿Alcanza a las copias? | **Sí**, por los §3 y §4 | `[V]` + `[H]` |
| Fallo | Se ignora o solo se registra (Tarea 013); `sd->fd < 0` → no hace nada | `[V]` |
| Dispositivos abiertos directamente por PARALLAX | **No cubiertos** | `[V]` (ausencia) |
| El compositor muere antes que logind actúe | Las copias de los hijos siguen siendo master o grab hasta la desactivación de la sesión | `[H]` |
| logind muere | Sin desactivación hasta que se reinicie; el reinicio restaura el estado desde el fd store (`sd->fd < 0`: «fd was lost») | `[V]` (mensaje) + `[H]` |

**No es una garantía absoluta:** depende de que la sesión se desactive, de que los ioctls no fallen y de que no haya fds abiertos directamente.

---

## 6. BrlTTY: TIOCSTI y uinput

### 6.1 TIOCSTI

Cadena (`Drivers/Screen/Linux/screen.c`) `[V]`:
- `insertKey_LinuxScreen` → `KDGKBMODE` → `K_XLATE`/`K_UNICODE` → `insertTranslated` → … → `insertByte`.
- `insertByte`: `controlCurrentConsole(TIOCSTI, &byte)`.
- `controlCurrentConsole` → `controlConsole(&currentConsoleDescriptor, selectedVirtualTerminal, …)`.

El **destino es la VT seleccionada** por brltty.

Mensajes literales del código al fallar `TIOCSTI`:
- «BRLTTY is running without the CAP_SYS_ADMIN kernel capability»
- «the sysctl parameter dev.tty.legacy_tiocsti is off»

Interpretación:
- `CAP_SYS_ADMIN` se usa **para saltarse la restricción de `legacy_tiocsti`** `[V]` (mensaje) + `[H]` (`tty_io.c` no revisado).
- Con `legacy_tiocsti=1`, no haría falta para la VT propia `[H]`.

Qué la activa:
- comandos del núcleo de brltty (pasar caracteres, puntos, PASTE) desde los atajos del dispositivo;
- teclas no aceptadas por clientes, que se tratan como comandos del núcleo.

**Con una VT gráfica en `K_OFF` esta ruta no se usa.**

### 6.2 uinput

`Programs/system_linux.c` `[V]`:
- `newUinputKeyboard` → `newUinputObject` (abre `/dev/uinput` en `O_WRONLY`) → `enableKeyboardKeys` → `EV_REP` → `createUinputDevice` (`UI_DEV_CREATE`).
- `writeKeyEvent` escribe los eventos.
- En `screen.c`:
  - `openKeyboard` crea «Linux Screen Driver Keyboard»;
  - `injectKeyboardEvent` → `writeKeyEvent`.
- Llamantes: `insertLinuxKey` (pegado en modos no raw), `PASSXT`/`PASSAT`/`PASSPS2`.

| # | Pregunta | Respuesta |
|---|---|---|
| 1 | Usuario y permisos | `brltty`, con ACL `u:brltty:rw` (Tarea 017) |
| 2 | Reglas udev | `90-brltty-uinput.rules` |
| 3 | Eventos | Teclas de teclado + autorrepetición |
| 4 | Receptores | Los handlers del `input_dev` virtual, como cualquier teclado |
| 5 | ¿Afecta el grab del greeter? | El greeter captura el dispositivo virtual y **recibe** sus eventos; el grab no lo bloquea (Tarea 015) |
| 6 | Impacto | Confidencialidad: ninguno. **Integridad:** inyección. Disponibilidad: hotplug y saturación |

---

## 7. BrlAPI: portapapeles e inyección asistida

| Pregunta | Respuesta | Evidencia |
|---|---|---|
| ¿El cliente puede ordenar PASTE directamente? | **No:** solo el servidor escribe `BRLAPI_PACKET_KEY` (`writeKey`); `api__handleCommand` envía comandos **del núcleo al cliente**, no al revés | `brlapi_server.c` líneas 549 y 4391 `[V]` |
| ¿Puede modificar el portapapeles? | **Sí** (`BRLAPI_PARAM_CLIPBOARD_CONTENT`, escribible) | línea 2293 `[V-p]` (tabla; handler no releído) |
| ¿Qué acción física hace falta? | Que el usuario braille active PASTE con el atajo de su dispositivo | `cmd_clipboard.c` `[V]` (Tarea 019) |
| ¿Qué función inserta? | `pasteScreenCharacters` → `insertScreenKey` → `currentScreen->insertKey` | `scr.c` líneas 325–340 `[V]` |
| ¿TIOCSTI o uinput? | **Según el modo de la VT:** XLATE/UNICODE → TIOCSTI; RAW/MEDIUMRAW/OFF → teclado uinput (salvo RAW, que inserta bytes) | `screen.c` `[V]` |
| ¿Destino? | TIOCSTI: la VT. uinput: **el sistema de entrada**, es decir, quien tenga el grab o quien lea | `[V]` |
| ¿Puede afectar al greeter? | **Sí, si** brltty usa el driver de pantalla Linux, la VT del greeter está en `K_OFF` `[H]` y el cursor de brltty está en la «pantalla principal» (`isMainScreen()`) | `[V]` + `[H]` |
| ¿Autorización adicional? | Ninguna aparte de estar autenticado (Tarea 018) | `[V-p]` |

**Clasificación:**

| Vía | Estado |
|---|---|
| Inyección **directa** por el cliente | **No existe** en el código leído |
| Inyección **indirecta** por acción del usuario (RV-56) | **Recorrido confirmado en el código**, salvo que `K_OFF` y el destino real se comprueben en máquina. **No** se clasifica como vulnerabilidad confirmada |
| Inyección **por el demonio** con sus propios privilegios | Confirmada (capacidad) |

---

## 8. logind: GC de sesiones y user manager (v262.2 `[V]`)

| # | Pregunta | Respuesta | Evidencia |
|---|---|---|---|
| 1 | ¿Una sesión `closing` sigue contando? | **Sí.** `user_pinned_by_sessions` recorre `u->sessions` sin mirar el estado; el `LIST_REMOVE` está en `session_free` | `logind-user.c` línea 771; `logind-session.c` línea 177 |
| 2 | ¿Qué estados fijan? | Todos, mientras la sesión esté en la lista. Con `BY_PIN`, solo las clases `SESSION_CLASS_PIN_USER` (todas salvo `manager`, `manager-early` y `none`) | `logind-session.h` |
| 3 | ¿Cuándo sale de la lista? | `manager_gc` → `session_may_gc` → `session_finalize` + `session_free` | `logind.c`, `logind-session.c` |
| 4 | ¿Los procesos del scope evitan la GC? | **Sí:** `session_may_gc` devuelve falso si el líder vive o si `manager_unit_is_active(s->scope)` | `logind-session.c` líneas 1323–1365 |
| 5 | ¿`KillUserProcesses=no` (Arch)? | El scope se abandona → sigue activo mientras haya procesos → la sesión no se libera → **el usuario queda fijado** | Tarea 018 + lo anterior |
| 6 | ¿`KillOnlyUsers`? | `manager_shall_kill`: si la lista no está vacía, solo mata a esos usuarios; root se excluye por defecto | `logind-core.c` (Tarea 019) `[V-p]`; clave `Login.KillOnlyUsers` en gperf `[V]` |
| 7 | ¿Otra sesión TTY? | Clase `user`: fija al usuario | `[V]` |
| 8 | ¿SSH? | Igual (pam_systemd, clase `user`) `[H]` | — |
| 9 | ¿Linger? | `user_may_gc` falso si existe el archivo de linger y la unidad está activa; desactivar linger fuerza `BY_PIN` | `logind-dbus.c` línea 1692 `[V]` |
| 10 | ¿Cuándo se detiene `user@`? | Sin sesiones que fijen + retardo cumplido + sin linger + sin jobs → `user_stop` → parada de `user-runtime-dir@` → `BindsTo` | `[V]` / `[V-p]` |
| 11 | ¿Qué es el retardo de 10 s? | `user_stop_delay` (`Login.UserStopDelaySec`) desde la **última sesión que fija**, no desde el cierre gráfico | `[V]` |
| 12 | ¿Caída frente a cierre normal? | En ambos casos `user@` solo se detiene si el scope queda vacío. En la caída, los supervivientes mantienen el scope y el usuario | `[V]` + `[H]` |

`gc_mode`:
- Al crear la primera sesión de una clase que fija, el usuario pasa a `BY_PIN` (`logind-dbus.c` líneas 1010–1015 `[V]`).
- Pasar de `BY_ANY` a `BY_PIN` no cambia el resultado anterior: las sesiones gráficas son de una clase que fija.

**Que `user@` se detenga no significa que mueran todos los procesos relacionados:** los del scope abandonado no pertenecen a `user@` `[H]`.

`SessionsMax` es un límite de creación de sesiones; no influye en la GC.

---

## 9. Matriz de escenarios

Abreviaturas: **D** demostrado · **P** parcial · **H** hipótesis · **NV** no verificado · **NA** no aplica.

| # | Escenario | Compositor | Descendientes | Sesión logind | Scope | `user@` | DRM | evdev | Greeter | Riesgo residual | Evidencia |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Logout normal | Sale | Según L1/L3: P | `closing` → eliminada si el scope se vacía: D | Abandonado: D | Se para si no hay otras sesiones: D | Libres si no hay copias: P | Ídem | Lo lanza greetd: D (T012) | Terceros | V código |
| 2 | Caída del compositor | Muerto | Sobreviven: H | `closing` mientras vivan: D | Activo: D | **Fijado**: D | logind revoca al activarse el greeter: P | Ídem: P | Presenta si hay master y grab: P | RV-55 | V + H |
| 3 | + hijo con master | — | Master **conservado**: D (mecanismo) | Ídem | Ídem | Fijado | Hasta `DROP_MASTER` de logind: P | — | `EBUSY` → no presenta: P | Disponibilidad | V + VM |
| 4 | + hijo con evdev | — | Grab y lectura conservados: D (mecanismo) | Ídem | Ídem | Fijado | — | Hasta `EVIOCREVOKE`: P | `EBUSY` → falla cerrado: P | Lectura antes de la revocación | V + VM |
| 5 | + fd por `SCM_RIGHTS` | — | Igual que 3–4 | | | | Igual | Igual | Igual | Igual | V + H |
| 6 | + fd abierto directamente | — | **Sin revocación** | | | | Master permanente: D (ausencia) | Lectura y grab permanentes | No presenta o falla cerrado | **Alto** | V |
| 7 | + sesión TTY | — | Sobreviven | — | — | Activo: D | Como 2 | Como 2 | P | RV-55 | V |
| 8 | + SSH | Igual | | | | Activo: H | | | | | H |
| 9 | Linger | — | — | — | — | Permanente: D | Como 2 | | P | RV-55 | V |
| 10 | `TerminateSession` | Muere con el scope si está en él (G-a/G-a′): H | Ídem | Se para: D | Se para: D | Según otras sesiones | fds cerrados al morir todos: P | Ídem | P | Procesos de `user@` (G-b) | V + H |
| 11 | `EVIOCREVOKE` correcto | — | `ENODEV` en todas las copias: D | — | — | — | — | Revocado: D | Grab posible: D | — | V |
| 12 | `EVIOCREVOKE` fallido | — | Sigue leyendo: D | — | — | — | — | Activo | `EBUSY` si hay grab; si no lo hay, **lectura paralela** hasta el grab del greeter | Lectura en la ventana | V |
| 13 | `DROP_MASTER` correcto | — | Sin master: D | — | — | — | Liberado | — | Master posible | — | V |
| 14 | `DROP_MASTER` fallido | — | Master: D | — | — | — | Retenido | — | `EBUSY` → no presenta: D | Disponibilidad; última imagen visible | V |
| 15 | El greeter no obtiene DRM | NA | — | — | — | — | — | — | No presenta; cierre en fallo (diseño T014) | Disponibilidad | V + diseño |
| 16 | El greeter no obtiene el grab | NA | — | — | — | — | — | — | No acepta credenciales (diseño) | Disponibilidad | Diseño |
| 17 | brltty activo con uinput | NA | — | — | — | — | — | Teclado virtual capturado por el greeter | Acepta su entrada | Integridad (RV-50) | V |
| 18 | Cliente BrlAPI + PASTE | NA | — | — | — | — | — | Texto inyectado por uinput si la VT está en `K_OFF` | **Recibe el texto pegado** | RV-56 (asistida) | V + H |

---

## 10. Reconciliación de riesgos y decisiones

| ID | Estado anterior | Evidencia de la Tarea 020 | Estado actual |
|---|---|---|---|
| RV-37 | Abierto | Los hijos conservan master y grab; logind revoca sobre las copias (mejor esfuerzo); los fds directos no se cubren | **Abierto** |
| RV-43 | Actores acotados | Mecanismo de herencia confirmado | Abierto |
| RV-44 | + `cap_sys_admin`, + hijos | Confirmado en el código | Abierto |
| RV-48 | Sin cambios | — | Abierto |
| RV-50 | El demonio inyecta | uinput y TIOCSTI confirmados literalmente | Abierto |
| RV-53 | Condicional | — | Condicional |
| RV-54 | Control del dispositivo | — | Abierto |
| RV-55 | Reforzado `[H]` | **Una sesión `closing` fija al usuario `[V]`** | Abierto, con evidencia de código |
| RV-56 | Propuesto | Recorrido completo en el código; falta `K_OFF` en máquina | Propuesto (**parcialmente confirmado**) |
| D-28 | Pendiente | G-a′ no evita la herencia | `[P]` |
| D-31 | Pendiente | `KillOnlyUsers` verificado | `[P]` |
| D-32 a D-35 | Pendientes | — | `[P]` |
| D-36 | Pendiente | El pegado de brltty puede llegar al greeter | `[P]` |
| D-37 | Pendiente | — | `[P]` |

**Comparación de variantes** (sin elegir ninguna):
- **G-a / G-a′:** el compositor está en el scope. Su muerte no basta, porque los hijos conservan los fds. `TerminateSession` o `KillUserProcesses` por usuario mataría el scope entero.
- **G-b:** el compositor está en `user@`. Necesita L1–L3. Con sesiones `closing` y otras sesiones, `user@` persiste.
- **Alternativa E y M-GRAB:** el master y el grab del greeter son la defensa que falla de forma cerrada (`EBUSY`) frente a copias heredadas. **Se mantienen.**
- **L1 (salida voluntaria del compositor):** sin cambios.
- **L2 (parada de `user@`):** **más débil de lo supuesto**, porque una sesión `closing` con procesos fija al usuario.
- **L3 (limpieza al siguiente inicio):** sin cambios.

---

## 11. Contradicciones y correcciones a las Tareas 013–019

| Tarea | Afirmación | Corrección |
|---|---|---|
| T012 / T013 | `KillUserProcesses=yes` por defecto | Refutada (T017/T018) |
| T018 | G-a′ «elimina el escenario central» | Matizado en T019 y confirmado aquí: los hijos retienen los fds |
| T019 | «Sesiones `closing` podrían fijar» `[H]` | **Confirmado** `[V]` |
| T019 | «Destino de `pasteScreenCharacters` sin verificar» | **Verificado:** `insertKey` del driver |
| T019 | Ubicación de uinput desconocida | Está en `Programs/system_linux.c` |
| T013–T019 | Lecturas por herramienta web `[V-p]` | Las rutas releídas aquí pasan a `[V]` frente a upstream; el resto se queda como estaba |

---

## 12. Pruebas futuras (no autorizadas, requieren máquina real)

| ID | Contenido |
|---|---|
| T-FD-INHERIT | Un hijo con copia del fd DRM master y evdev con grab; el compositor muere. Comprobar el master del hijo, el `EBUSY` del greeter y la lectura |
| T-LOGIND-SHARED-REVOKE | Activar el greeter. Comprobar que el hijo recibe `ENODEV` y pierde el master |
| T-LOGIND-GC | Sesión `closing` con hijos vivos. Comprobar que `user@` no se detiene; repetir con `KillOnlyUsers` |
| T-BRLTTY-PASTE | VT del greeter en `K_OFF`. Un cliente BrlAPI escribe el portapapeles, el usuario pulsa PASTE y se mira si llegan eventos al greeter |
| T-TIOCSTI | `legacy_tiocsti` en el kernel de Arch; ¿es necesaria `CAP_SYS_ADMIN`? |

---

## 13. Decisiones pendientes del propietario

| Decisión | Tema |
|---|---|
| D-28 | Variante G-a / G-b / G-a′ |
| D-31 | Linger / `KillOnlyUsers` |
| D-32 a D-35 | Greeter, hotplug, hidraw, asistencia |
| D-36 | brltty en la imagen base |
| D-37 | Política de sesión |
| — | Autorización de cualquier prueba y del acceso a DEV-01 |

---

## 14. Veredicto final: `EVIDENCIA PARCIAL; VALIDACIÓN EN MÁQUINA PENDIENTE`

**Justificación:**
- Las lagunas de código quedan cerradas frente a upstream con lectura literal:
  - posesión del master y del grab;
  - revocación sobre copias;
  - fd compartido por logind;
  - recorrido del pegado de brltty;
  - fijación de usuarios por sesiones `closing`.
- Faltan cuatro cosas:
  - la semántica de VFS y `SCM_RIGHTS`, que es estándar pero no se ha leído;
  - el kernel de Arch;
  - `K_OFF` en la VT del greeter;
  - toda la ejecución real.
- **No basta para una decisión arquitectónica cerrada:** RV-37 depende de pruebas en máquina (herencia, revocación, GC).

---

## ESTADO DEL PROYECTO

| Elemento | Estado |
|---|---|
| Fase 0 | `IN_PROGRESS` |
| ADR | Todos `Propuesto` |
| Fase 1 | Sin autorización |
| RV-37 | **Abierto** |
| D-28 y D-31 a D-37 | Pendientes |
| Repositorio | Sin modificaciones |
| Pruebas | Ninguna ejecutada |
| DEV-01 | No accedido |
