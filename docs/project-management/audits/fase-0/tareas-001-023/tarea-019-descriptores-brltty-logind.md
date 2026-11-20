# Tarea 019 — Herencia de descriptores, rutas de inyección de brltty y ciclo de vida de logind

- **Fecha:** 2026-11-19
- **Estado del proyecto:** Fase 0 `IN_PROGRESS` · todos los ADR `Propuesto` · Fase 1 sin autorización
- **Naturaleza:** solo lectura. Es una propuesta pendiente de aprobación. El informe está en el scratchpad; el repositorio no se ha modificado.
- **Marcas de evidencia:** `[V]` fuente suficiente · `[V-p]` evidencia parcial · `[R]` repositorio · `[H]` inferencia · `[VM]` requiere validación real · `[P]` decisión del propietario.

---

## 1. Resumen ejecutivo

**Veredicto: `EVIDENCIA PARCIAL; COMPORTAMIENTO EN EJECUCIÓN PENDIENTE`** (§16).

### A. G-a′: hipótesis parcialmente confirmada

**Lo que sí elimina.** G-a′ elimina la supervivencia del **compositor**: greetd espera a su PID.

**Lo que no elimina.** **No** impide que un descendiente conserve descriptores sensibles.
- Un fd heredado o recibido por `SCM_RIGHTS` comparte la **misma descripción de archivo abierta** (*open file description*).
- Esa descripción sigue viva mientras exista cualquier referencia.
- En DRM, el master pertenece al `drm_file` y solo se libera al cerrarse la **última** referencia.
- Por tanto, un hijo que conserve el fd con master **sigue siendo master**, incluso tras morir el compositor `[V-p + H fuerte]`.

**Lo que compensa en parte.** Para los fds que entregó **logind** (`TakeDevice`), las acciones de logind sobre **su** copia afectan a la misma descripción. `drmDropMaster` y `EVIOCREVOKE` alcanzan también a las copias de los hijos `[H fuerte]`. Pero logind lo hace con **mejor esfuerzo** (Tarea 013).

**Los fds abiertos directamente** por el compositor no los cubre nadie.

### B. brltty

El **demonio** puede inyectar entrada en el sistema operativo por dos vías:
- **`TIOCSTI`** en la consola Linux;
- un teclado virtual **uinput** («Linux Screen Driver Keyboard»).

Ambas las crea el driver de pantalla Linux `[V-p]`.

Lo que dispara la inyección:
- los atajos de teclas del dispositivo braille;
- las teclas que ningún cliente BrlAPI acepta, que procesa el núcleo de brltty `[V-p resumen]`.

**Un cliente BrlAPI no puede encolar comandos** (no se encontró ningún paquete para ello). **Sí puede escribir en el portapapeles** (`PARAM_CLIPBOARD_CONTENT`).

Hay además una **vía asistida, parcialmente confirmada**: un cliente escribe texto en el portapapeles y el usuario braille pulsa *PASTE* en su dispositivo. Entonces brltty inyecta ese texto, que el cliente eligió, con `pasteScreenCharacters`. El destino concreto de esa función no se verificó, y solo actúa sobre la pantalla principal `[V-p]`. **Requiere una acción física del usuario.**

### C. logind

- **Valores por defecto upstream verificados:**
  - `user_stop_delay` = 10 s;
  - `stop_idle_session_usec` = ∞;
  - `manager_shall_kill` excluye a root por defecto.
- En Arch, `kill_user_processes` = **false** (PKGBUILD) `[V]`.
- Las clases **`greeter` y `user` fijan al usuario** (`SESSION_CLASS_PIN_USER`) `[V-p]`.
- La GC de sesiones solo finaliza una sesión cuando el líder está muerto y el scope ha terminado `[V-p resumen]`.
- **Consecuencia para G-a′:** los hijos que sobrevivan dentro del scope mantienen la sesión en `closing`. Si esa sesión sigue contando para fijar al usuario, `user@` **no** se detiene, y la limpieza L2 falla `[H]`.

### RV-37

**Sigue abierto.**

---

## 2. Alcance y metodología

- Se reutilizan las lecturas de las Tareas 013–018.
- Nuevas lecturas de esta tarea:
  - **brltty BRLTTY-6.9.1:** `Drivers/Screen/Linux/screen.c` (parcial; la cadena de inserción está resumida), `Programs/cmd_clipboard.c` (literal) y `brlapi_server.c` (parcial).
  - **systemd v262.2:** `logind-session.h` (completo), `logind-core.c` (completo), `logind.c` (81 %), `logind-user.c` (completo) y `logind-session.c` (65 %).
- **No leídos:**
  - de brltty: `Programs/uinput.c` (404; la ubicación real es desconocida), `system_linux.c`, `pasteScreenCharacters`, `cmd_keycodes.c`/`cmd_input.c` y `Drivers/Screen/AtSpi2`;
  - de systemd: `logind-gperf.gperf`, `logind-dbus.c` y el resto de `logind-session.c`.
- **Semántica de VFS** (descripciones de archivo, `fork`, `SCM_RIGHTS`, cierre al salir del proceso): no se volvió a leer en código en esta tarea. Se marca `[H fuerte]`, basada en la semántica estándar de POSIX y Linux y en `private_data` y `drm_file` (Tareas 013–014).
- No hubo acceso a DEV-01 y no se ejecutó nada.

---

## 3. Fuentes y versiones

| Tema | Archivo / símbolo | Versión | Nivel |
|---|---|---|---|
| Master por `drm_file`; `drm_master_release` al cerrar | `drm_auth.c`, `drm_file.c` | Linux v7.3 | `[V-p]` (Tareas 013–014) |
| Cliente evdev en `private_data`; revoke por cliente | `evdev.c` | v7.3 | `[V-p]` |
| Teclado uinput del driver de pantalla | `screen.c` `openKeyboard()`, `newUinputKeyboard("Linux Screen Driver Keyboard")` | BRLTTY-6.9.1 | `[V-p]` literal |
| `TIOCSTI` | `screen.c` `insertByte` → `controlCurrentConsole(TIOCSTI, …)` | Ídem | `[V-p resumen]` |
| `PASS*` → uinput | `screen.c` `handleCommand_LinuxScreen` | Ídem | `[V-p resumen]` |
| Pegado | `cmd_clipboard.c` `handleClipboardCommands`, `cpbPaste` → `pasteScreenCharacters` | Ídem | `[V-p]` literal |
| Teclas no aceptadas → núcleo de brltty | `brlapi_server.c` `api__handleKeyEvent`/`api__handleCommand` | Ídem | `[V-p resumen]` |
| Portapapeles escribible por cliente | `brlapi_server.c` `PARAM_CLIPBOARD_CONTENT` | Ídem | `[V-p]` (Tarea 018) + resumen |
| Clases de sesión | `logind-session.h` `SESSION_CLASS_*` | systemd v262.2 | `[V-p]` literal |
| `manager_shall_kill`, valores por defecto | `logind-core.c` | v262.2 | `[V-p]` literal |
| Bucle de GC | `logind.c` `manager_gc` | v262.2 | `[V-p]` literal |
| `session_may_gc` / `session_get_state` | `logind-session.c` | v262.2 | `[V-p resumen]` |
| `KillUserProcesses` en Arch | PKGBUILD 262.2-1 | Arch | `[V]` |

---

## 4. Herencia de descriptores

| # | Operación | Efecto |
|---|---|---|
| 1 | El compositor abre DRM (directamente o con `TakeDevice` → fd recibido por `SCM_RIGHTS`) | El fd del compositor apunta a una descripción de archivo. Con `TakeDevice`, logind **conserva otra referencia a la misma descripción** `[H fuerte]` |
| 2 | Abre evdev | Igual; la descripción contiene el `evdev_client` (`private_data`) |
| 3 | `fork()` | El hijo recibe **copias de todos los fds**, referencias a las mismas descripciones `[H POSIX]` |
| 4 | `execve()` | Se cierran los fds con `FD_CLOEXEC`; el resto sobrevive `[H POSIX]` |
| 5 | Hijo que recibe un fd por `SCM_RIGHTS` | Nueva referencia a la misma descripción |
| 6 | El compositor muere | El kernel cierra **sus** fds: baja el contador de referencias. La descripción sigue viva si otra referencia la mantiene `[H POSIX]` |
| 7 | El hijo sigue vivo | Conserva un `drm_file` o `evdev_client` **plenamente operativo**, salvo revocación |

- **Cerrar una referencia no equivale a revocar.** La revocación (`EVIOCREVOKE`) marca el **cliente**, es decir, la descripción, y afecta a todas sus copias. Cerrar solo baja el contador.
- `pidfd_getfd` permite a un proceso con `PTRACE_MODE_ATTACH` copiar fds de otro (Tarea 007). Mitigado por `PR_SET_DUMPABLE=0` (D-22).

### 4.1 Higiene propuesta

| Medida | Eficacia | Límite |
|---|---|---|
| `O_CLOEXEC`/`SOCK_CLOEXEC` en la creación | Evita la herencia por `exec` | No cubre `fork` sin `exec`, ni fds creados por bibliotecas sin el flag |
| Lista explícita de fds heredables + `posix_spawn_file_actions_addclosefrom_np(5)` (D-27) | Cierra todo lo demás **en el momento del spawn** | **No** cubre fds que el hijo reciba **después** por `SCM_RIGHTS`, ni fds que abra el propio hijo, ni hijos creados con `fork` fuera del intermediario |
| Ningún DRM ni input heredable en apps | Necesaria | Requiere auditar el código de Smithay y libinput (Fase 1) |
| Control de `SCM_RIGHTS` | Solo el compositor decide qué envía; los protocolos Wayland **sí** pasan fds (dmabuf, keymap) | dmabuf: la app recibe buffers, no el nodo DRM primario `[H]` |
| Inventario de fds sensibles | Auditoría | No es una garantía del kernel |

`addclosefrom_np` es **suficiente para el momento del spawn**. **No** es una garantía global.

---

## 5. DRM

| Pregunta | Respuesta |
|---|---|
| ¿Puede un hijo conservar un fd DRM tras morir el compositor? | **Sí**, si lo heredó sin `CLOEXEC` o lo recibió por `SCM_RIGHTS` `[H fuerte]` |
| ¿Conserva el master? | **Sí**: el master es de la descripción (`drm_file`) y se libera en `drm_master_release` **al cerrarse la última referencia** `[V-p + H]` |
| ¿Qué pasa al cerrarse el último fd relevante? | `drm_master_release` suelta el master; los FB del archivo se retiran (`drm_fb_release`, Tarea 015) |
| ¿Y si otro proceso mantiene el fd? | Puede seguir haciendo modeset y flips: puede **presentar**. El greeter recibe `EBUSY` y no presenta (Tareas 013–014) |
| Si el fd vino de logind | `DROP_MASTER` de logind sobre su copia afecta al `drm_file` compartido, y el hijo pierde el master `[H fuerte]`. **Solo si** logind lo ejecuta y no falla (Tarea 013) |
| Render node | El hijo puede seguir renderizando o calculando; **no** hace modeset ni tiene master. No da presentación |
| GEM/GBM, mmap | Un mapping mantiene vivo el objeto GEM con independencia del fd `[H]` (`drm_gem.c` no leído). Solo afecta a la pantalla si algún plano lo escanea (Tarea 015) |
| Fuera de la revocación de logind | fds abiertos directamente; render nodes; mappings; dmabufs exportados |

Capacidades separadas, para un hijo que conserve:

| Lo que conserva | Puede |
|---|---|
| Un fd master | Modeset y presentación |
| Un fd primario sin master | Crear buffers, sin presentar |
| Un render node | Renderizar |
| Un mapping | Escribir en un buffer, visible solo si está en pantalla |

---

## 6. evdev

| Pregunta | Respuesta |
|---|---|
| ¿Puede un hijo conservar un fd evdev tras morir el compositor? | **Sí** (misma descripción) `[H fuerte]` |
| ¿Puede seguir leyendo? | **Sí**, si no está revocado y nadie más tiene grab |
| ¿Si fue revocado? | `-ENODEV` en todas las copias (el revoke es por cliente) `[V-p]` |
| Revocar frente a cerrar | Revocar desactiva la descripción para todos; cerrar solo baja el contador |
| ¿Mantiene o disputa el grab? | El grab pertenece al **cliente**: si el compositor hizo grab y el hijo comparte la descripción, **el hijo hereda el grab** `[V-p evdev_grab + H]` |
| ¿Muere el dueño original y el hijo conserva la referencia? | El grab **sigue activo** mientras la descripción esté abierta: `evdev_release`, que hace el ungrab, solo se ejecuta al cerrar la última referencia `[V-p + H]`. El greeter recibe `EBUSY` hasta que logind revoque (revoke → `evdev_ungrab`) |

`[VM]`: todo lo anterior requiere prueba en el kernel de Arch.

---

## 7. brltty y BrlAPI

### 7.1 Rutas de inyección en el sistema operativo (B.2)

| Ruta | Proceso | Privilegio | Interfaz | Origen | ¿Lo activa un cliente BrlAPI? |
|---|---|---|---|---|---|
| `TIOCSTI` | brltty (driver de pantalla Linux) | `cap_sys_admin` (necesaria si `legacy_tiocsti` está desactivado `[H]`) | `ioctl` sobre la consola actual | Inserción de caracteres desde atajos braille o desde PASTE | Solo de forma indirecta (portapapeles + acción del usuario) |
| uinput «Linux Screen Driver Keyboard» | brltty | ACL `u:brltty` en `/dev/uinput` | `writeKeyEvent` | `PASSXT/PASSAT/PASSPS2` y `insertLinuxKey` | No se encontró ruta directa |
| API de terminal | Ídem | `tty` | `KDGKBMODE` | Selección de vía | — |
| X11/Wayland/AT-SPI | No verificado | — | — | — | Sin afirmación |
| `xbrlapi` (XTest, cliente) | Proceso del usuario | X11 | — | — | Es un cliente y no aplica a Wayland `[H]` |

### 7.2 Portapapeles (B.3)

| Aspecto | Hecho |
|---|---|
| Quién escribe | Los clientes BrlAPI autenticados (`PARAM_CLIPBOARD_CONTENT`, escribible) `[V-p]`; los comandos `CLIP_*`/`COPY_*` del propio usuario braille |
| Qué representa | El portapapeles **interno** de brltty (con historial y `cpbRestore` desde un archivo) |
| ¿Escribir dispara algo? | No se encontró ninguna acción automática `[V-p]` |
| Pegado | Lo hacen `BRL_CMD_PASTE*`, comandos del **núcleo** de brltty que salen de los atajos del dispositivo o de teclas no aceptadas por clientes `[V-p]` |
| ¿El pegado genera eventos del sistema? | `cpbPaste` → `pasteScreenCharacters`, solo si `isMainScreen()` y fuera del modo *routing*; probablemente por `insertKey` del driver (TIOCSTI o uinput) `[H]` |
| ¿Requiere acción física? | **Sí**, salvo que un cliente pudiera encolar `PASTE`, cosa que no se encontró |

**Clasificación:**
- Inyección directa por un cliente: **no demostrada**.
- Inyección **asistida**, con contenido elegido por el cliente y pegado por el usuario: **parcialmente confirmada**.

### 7.3 Impacto en riesgos (B.4)

- **RV-50:** añadir «brltty (demonio): TIOCSTI y uinput confirmados en el código como capacidades. Cliente BrlAPI: sin vía directa».
- **RV-54:** mantenerlo como riesgo de privacidad y control del dispositivo braille.
- **Nuevo, RV-56 (propuesto):** *inyección asistida por portapapeles*. Un miembro de `brlapi` cambia el portapapeles y el usuario braille, al pegar, introduce texto ajeno. Es una dimensión distinta (*confused deputy*). **No es una vulnerabilidad confirmada:** requiere una acción física y la ruta final no está verificada.
- La pertenencia a `brlapi` da escritura global en el portapapeles y control raw del dispositivo **sin autorización adicional** `[V-p]`. La conveniencia de exigirla es una decisión `[P]`.

---

## 8. Ciclo de vida en logind (v262.2 + Arch)

| Estado | Condición | Tipo de transición |
|---|---|---|
| Active / Online | `session_is_active` | Síncrona con el cambio de VT (asíncrona respecto al kernel) |
| Closing | `stopping` o temporizador de release (20 s) `[V-p resumen]` | Temporizador o `session_stop` |
| Removed | `session_may_gc`: líder muerto y ningún job ni scope `[V-p resumen]` → `session_finalize` + `session_free` en `manager_gc` `[V-p]` | Cola de GC (eventos) |
| Scope activo | Mientras queden procesos | — |
| Scope parándose | Solo si hay kill (`force`, `kill_processes` o `manager_shall_kill`); si no, abandonado `[V-p]` | Job de systemd |
| `user@` activo | Mientras alguna sesión con `SESSION_CLASS_PIN_USER` (todas salvo `manager`/`none`) exista, o con linger | — |
| `user@` parándose o parado | `user_may_gc` → `user_stop` → parar `user-runtime-dir@` → `BindsTo` | Temporizador (10 s) + jobs |

**Que una sesión esté `closing` no significa que sus procesos hayan terminado.**

**Comportamiento en Arch (C.3)** (`kill_user_processes=false` `[V]`; root excluido por defecto `[V-p]`):

- El scope se **abandona** y la sesión queda `closing` mientras haya procesos.
- **Otras sesiones** (TTY, SSH, con clase `user`) fijan al usuario, y `user@` sigue activo.
- **Linger:** `user@` persiste.
- **Retardo de parada:** 10 s por defecto upstream.
- **Abierto `[H]`:** si una sesión `closing` sigue fijando al usuario. `user_pinned_by_sessions` recorre `u->sessions` sin filtrar por estado en el código leído (Tarea 018). Si es así, unos hijos supervivientes en el scope **mantienen vivo a `user@`**.
- **Modo de GC al crear el usuario** (`BY_ANY` en `user_new`; `BY_PIN` si falla la carga): sin verificar en `logind-dbus.c`.

---

## 9. Comparación G-a / G-b / G-a′

| | G-a (compositor en el scope, lanzado por otro proceso) | G-b (compositor en `user@`) | G-a′ (compositor = comando de greetd) |
|---|---|---|---|
| Qué supervisa greetd | El comando de sesión | El comando de sesión | **El compositor** |
| Si muere | Se cierra PAM; el scope se abandona | Igual | Igual |
| Unidades que se detienen | Ninguna automáticamente (Arch) | Las del grafo PARALLAX, si el comando las detiene | Ninguna automáticamente |
| Quién sobrevive | Compositor (si no era el comando) + hijos | Todo lo de `user@` | **Hijos del compositor** |
| Fds que pueden quedar | Todos | Todos | Los heredados o transferidos a hijos |
| TTY/SSH mantiene `user@` | Sí | Sí | Sí |
| ¿Greeter sin interferencias? | Master + grab | Master + grab | Master + grab |
| ¿Sesión nueva segura? | Con L3 | Con L3 | Con L3 |

**No se elige ninguna variante.**

---

## 10. Matriz de escenarios

| Escenario | Proceso principal | Hijos | Scope | User manager | DRM/input | Greeter |
|---|---|---|---|---|---|---|
| Logout normal | Sale (diseño) | Condicional (L1/L3) | Abandonado y vaciado si salen todos: condicional | Para tras 10 s si no hay otra sesión ni linger: garantizado por la fuente `[V-p]` | Libres si nadie hereda: condicional | Garantizado por la fuente (greetd) `[V-p]` |
| Caída del compositor | Muerto | Sobreviven: no garantizado | `closing`: requiere validación | Fijado si los hijos fijan: no demostrado | Fds de logind revocados al activar el greeter (mejor esfuerzo): condicional | Se presenta si obtiene master y grab: condicional |
| Caída del Shell | Vivo | — | — | — | Sin cambio | No aplica |
| Hijo mantiene fd DRM | — | Vivo con master | `closing` | Fijado `[H]` | **El master persiste** salvo `DROP_MASTER` de logind sobre un fd compartido: no garantizado | `EBUSY` → no presenta: requiere validación |
| Hijo mantiene fd evdev | — | Vivo y lee | `closing` | Fijado `[H]` | Lee salvo revocación; puede conservar el grab heredado: no garantizado | `EBUSY` en el grab → cierre en fallo: requiere validación |
| Sesión TTY adicional | — | Sobreviven | — | Activo: garantizado por la fuente `[V-p]` | Como arriba | Condicional |
| Sesión SSH adicional | Igual | | | | | |
| Linger | — | Sobreviven | — | Permanente `[V-p]` | Como arriba | Condicional |
| Terminación forzada (`TerminateSession`) | Muere el scope | Mueren los del scope | Parado | Según otras sesiones | Fds cerrados si todo estaba en el scope: requiere validación | Condicional |

---

## 11. Reconciliación histórica

| ID | Conclusión anterior | Evidencia de la Tarea 019 | Estado actual | Acción propuesta |
|---|---|---|---|---|
| RV-37 | T018: escenario central eliminable con G-a′ | Los hijos heredan el master y el grab | **Abierto.** G-a′ solo elimina al compositor | Higiene de fds + revocación de logind + master/grab del greeter |
| RV-44 | + `cap_sys_admin` | + hijos con fd master heredado | Ampliado | Redactar |
| RV-50 | BrlAPI no demostrada | Demonio: TIOCSTI y uinput confirmados; cliente: no | Precisado | Redactar |
| RV-53 | Condicional | Capacidad real de inyección del demonio | Condicional, reforzado | — |
| RV-54 | Control del dispositivo | + portapapeles | Se mantiene | — |
| RV-55 | Supervivencia con `KillUserProcesses=no` | Las sesiones `closing` podrían fijar `user@` `[H]` | Reforzado | Prueba T-LOGIND-GC |
| RV-56 | — | Inyección asistida por portapapeles | **Nuevo (propuesto)** | — |
| D-28 | G-c frente a G-a′ | G-a′ no evita la herencia | Pendiente `[P]` | — |
| D-31 | Linger / `KillUserProcesses` | `KillOnlyUsers` verificado en `manager_shall_kill` | Pendiente `[P]` | — |
| D-32 a D-35 | — | — | Pendientes `[P]` | — |
| D-36 | brltty en la base | Capacidad de inyección del demonio confirmada | Pendiente `[P]` | — |
| D-37 | Política de sesión | — | Pendiente `[P]` | — |
| G-a | Necesita `KillUserProcesses` | Sin cambios | Pendiente | — |
| G-b | Necesita L1–L3 | Sin cambios | Pendiente | — |
| G-a′ | «Elimina el escenario central» (T018) | **Solo elimina la supervivencia del compositor**; los descendientes pueden conservar descriptores | **Parcialmente confirmada.** La afirmación de T018 se matiza, no se borra | — |
| Alternativa E | Se mantiene | El master y el grab del greeter siguen siendo la defensa que funciona cerrando en fallo | Se mantiene | — |
| M-GRAB | Viable con limitaciones | Los grabs heredados producen `EBUSY` (detectable) | Se mantiene | — |
| `KillUserProcesses` | Arch `no` | `manager_shall_kill` leído | Confirmado | — |

---

## 12. Riesgos afectados

RV-37, RV-44, RV-50, RV-53, RV-54, RV-55 y **RV-56** (nuevo, propuesto). No se modifica ningún registro.

---

## 13. Recomendaciones técnicas (no normativas)

### A. Higiene de descriptores

1. Usar **solo fds de logind** (`TakeDevice`) para DRM e input, nunca `open()` directo. Así la revocación de logind alcanza a todas las copias.
2. `CLOEXEC` en todo; lista explícita de fds heredables; `addclosefrom_np` en el intermediario.
3. Prohibir que roles y apps reciban fds DRM primarios o evdev; solo dmabufs y keymaps por Wayland.
4. Auditar Smithay y libinput en Fase 1 para comprobar que no hay aperturas sin `CLOEXEC`.

### B. Ciclo de sesión

1. Si se adopta G-a′, que el compositor **mate de forma explícita** a sus hijos de rol al terminar, aunque es una defensa voluntaria.
2. Valorar `KillOnlyUsers`/`KillUserProcesses` para los usuarios de PARALLAX (D-31).
3. Antes de una sesión nueva, que L3 compruebe que no queda ningún proceso del usuario con fds de `/dev/dri/card*` o `/dev/input/event*`. Desde el propio usuario es posible leer `/proc/*/fd` de sus procesos (dumpable `[H]`).

### C. Accesibilidad

1. No desactivar brltty por defecto sin una alternativa accesible de activación.
2. Si se endurece, conservar uinput y TIOCSTI, que son su función, y limitar `cap_sys_admin` con un bounding set solo si TIOCSTI funciona sin ella con `legacy_tiocsti=1` `[VM]`.
3. Valorar que el greeter muestre un **indicador** cuando un dispositivo virtual (brltty) esté presente.

---

## 14. Pruebas futuras (no autorizadas)

| ID | Contenido |
|---|---|
| T-FD-INHERIT | Un hijo hereda o recibe el fd DRM master y evdev; el compositor muere; comprobar master, `EBUSY` del greeter y lectura de eventos |
| T-LOGIND-SHARED-REVOKE | `DROP_MASTER` y `EVIOCREVOKE` de logind sobre un fd compartido con un hijo → el hijo pierde master y lectura |
| T-LOGIND-GC | ¿Una sesión `closing` con hijos vivos fija `user@`? ¿TTY/SSH? ¿Linger? |
| T-BRLTTY-INJECT | Atajos PASTE/PASS* → TIOCSTI y uinput; cliente BrlAPI que escribe el portapapeles + PASTE manual |
| T-TIOCSTI | `legacy_tiocsti` en el kernel de Arch; efecto de `cap_sys_admin` |

---

## 15. Decisiones pendientes del propietario

D-28, D-31, D-32, D-33, D-34, D-35, D-36 y D-37. Además, la autorización de cualquier prueba y del acceso a DEV-01.

---

## 16. Veredicto final: `EVIDENCIA PARCIAL; COMPORTAMIENTO EN EJECUCIÓN PENDIENTE`

**Justificación:**
- La semántica de descriptores es estándar, pero no se ha leído en código en esta tarea.
- La cadena de inyección de brltty está en buena parte **resumida**, y `uinput.c` no se localizó.
- Partes clave de la GC de logind (`session_may_gc`, la fijación por sesiones `closing`, `gc_mode`) son resúmenes o inferencias.
- No hay ninguna contradicción que obligue a rediseñar. G-a′ queda **matizada** y la defensa del greeter (master + grab, cerrando en fallo) sigue siendo la que importa.

---

## ESTADO DEL PROYECTO

| Elemento | Estado |
|---|---|
| Fase 0 | `IN_PROGRESS` |
| ADR | Todos `Propuesto` |
| Fase 1 | Sin autorización |
| RV-37 | **Abierto.** Ni G-a′ ni logind garantizan que los descendientes pierdan los descriptores sensibles |
| D-28 y D-31 a D-37 | Pendientes |
| Repositorio | Sin modificaciones |
| Pruebas | Ninguna ejecutada |
| DEV-01 | No accedido |

**1. Hechos demostrados en las fuentes (`[V-p]`/`[V]`):**
- Arch usa `KillUserProcesses=no`.
- `manager_shall_kill` excluye a root por defecto.
- El retardo de parada de `user@` es de 10 s.
- Las clases `user` y `greeter` fijan al usuario.
- El demonio brltty tiene un teclado uinput y usa TIOCSTI.
- El portapapeles de brltty es escribible por clientes BrlAPI.
- Los clientes BrlAPI no pueden encolar comandos (en lo leído).

**2. Hipótesis pendientes:**
- La herencia del master y del grab por los hijos.
- Que la revocación de logind alcance a las copias compartidas.
- La fijación de `user@` por sesiones `closing`.
- El destino final de `pasteScreenCharacters`.

**3. Recomendaciones técnicas:** las de §13.

**4. Decisiones del propietario:** D-28, D-31 a D-37, las pruebas y DEV-01.
