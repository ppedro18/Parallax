# Tarea 013 — Revisión correctiva de RV-37

- **Fecha:** 2026-11-19 · **Fase 0:** `IN_PROGRESS` · **ADR:** todos `Propuesto` · **Fase 1:** sin autorización.
- **Naturaleza:** investigación. Contrasta la Tarea 012 sin defenderla. Es una propuesta pendiente de aprobación. Repositorio intacto; este informe está en el scratchpad.
- **Marcas:** `[V]` fuente suficiente · `[V-p]` parcial · `[R]` repositorio · `[H]` inferencia · `[VM]` requiere prueba · `[P]` propietario.
- **Aviso de método:** el código citado se obtuvo con una herramienta de lectura web que devuelve texto procesado por un modelo. No se comparó con un checkout local. Por eso **todo el código se marca `[V-p]`**, aunque la lectura haya sido completa.

---

## 1. Resumen ejecutivo

**Veredicto: `MITIGACIÓN PARCIAL, GARANTÍA NO DEMOSTRADA`** (§15).

La Tarea 012 trataba como una sola propiedad («sesión inaccesible») cinco propiedades distintas. Con el código de systemd v262.2 y del kernel v7.3, quedan así:

| Propiedad | Resultado |
|---|---|
| 1. No presentar | **Hay una condición comprobable.** DRM master es exclusivo: si el compositor del greeter obtiene master sobre su fd de logind, la sesión anterior no puede hacer modeset ni page flip. Si el `drmDropMaster` de logind falla, el greeter **no consigue** master y no presenta. El fallo es visible y deja la pantalla en el estado anterior; no da presentación simultánea |
| 2. No recibir entrada | **No está garantizada por logind.** Los errores de `EVIOCREVOKE` solo se registran en el log y la transición continúa. `EVIOCREVOKE` es por descripción de archivo: un fd abierto directamente (por ejemplo, gracias a la ACL `uaccess` mientras la sesión estaba activa) **nunca se revoca**. Se propone una mitigación nueva y comprobable (**EVIOCGRAB** exclusivo por el greeter), pendiente de verificar en el kernel |
| 3. Pantalla neutral | **Sin garantía.** El último framebuffer sigue visible hasta el primer commit del greeter `[H]` |
| 4. Limpieza de procesos | **Sin garantía.** L1 es voluntaria; L2 depende de condiciones sin verificar; L3 no detecta procesos fuera de unidades conocidas |
| 5. Aislamiento entre sesiones | **No existe** frente a procesos que ya tenían un fd abierto directamente. Afecta también a usuarios distintos (RV-43) |

**Conclusión principal:** la alternativa E **no** da una garantía independiente para la entrada. Para la presentación sí hay una condición comprobable. Para que E sea defendible hacen falta comprobaciones **del lado del greeter** antes de aceptar credenciales (§12).

---

## 2. Evidencia de systemd 262.2

**Fuente:** `github.com/systemd/systemd`, tag `v262.2`. Commit SHA no visible. **No se pudo leer el PKGBUILD de Arch** (gitlab.archlinux.org bloqueado por Anubis). Que Arch 262.2-1 use exactamente este tag sin parches **no está verificado** (RV-41).

### 2.1 `src/login/logind-session-device.c` (lectura completa) `[V-p]`

| Símbolo | Hecho | Consecuencia |
|---|---|---|
| `detect_device_type` | Solo `drm/card*`, `input/event*` y `hidraw*`; el resto es `DEVICE_TYPE_UNKNOWN` | No cubre render nodes `renderD*`, `/dev/uinput`, `/dev/tty*`, framebuffer `fb*`, `/dev/snd`, cámaras |
| `session_device_verify` | `UNKNOWN` → `-ENODEV`; si el seat no coincide → `-EPERM` | Los dispositivos desconocidos no pasan por logind |
| `sd_drmdropmaster` | `RET_NERRNO(ioctl(fd, DRM_IOCTL_DROP_MASTER, 0))` | Devuelve el error |
| `session_device_stop` | DRM: `sd_drmdropmaster(sd->fd);` con el **resultado ignorado** y `sd->active = false` en cualquier caso. Si `sd->fd < 0`: «Failed to de-activate DRM fd, as the fd was lost» y **vuelve sin marcar inactivo** | Un fallo de drop **no detiene** la transición |
| `sd_eviocrevoke` | `EINVAL` → aviso único, «continuing without revoking device access»; `ENODEV` silencioso; resto → aviso. Devuelve `void` | **El fallo de revocación no se propaga.** Tras el primer `EINVAL`, la variable estática `warned` **suprime los intentos posteriores** |
| `sd_hidiocrevoke` | Igual que evdev | Igual |
| `session_device_pause_all` | `session_device_stop`, luego notifica `"force"` (retorno ignorado) | El controlador se entera **después** |
| `session_device_complete_pause` | Sin timeout en este archivo | — |
| `session_device_free` | stop, luego `"gone"`, luego `safe_close(sd->fd)` | Cierra el fd **de logind**; la copia del controlador sigue abierta `[H]` |
| `session_device_start` (DRM) | Si `setmaster` falla, devuelve el error y el dispositivo queda en pausa | Útil para §12 |
| `session_device_new` | Si la sesión está activa y la apertura da `-EINVAL`, reintenta inactiva; **otros errores (p. ej. `EBUSY` de SET_MASTER) hacen fallar `TakeDevice`** | El greeter recibe un error y no un fd «activo» falso `[H a partir del código]` |

**Nota sobre `warned`:** en `sd_eviocrevoke` la condición es `if (!warned && ioctl(...) < 0)`. Una vez `warned` vale `true`, **el ioctl ya no se ejecuta** en llamadas posteriores. Solo ocurre tras un `EINVAL` (kernel sin soporte), así que no afecta a kernels modernos `[H]`. Se registra porque, en ese caso, la revocación desaparece en silencio.

### 2.2 `logind-seat.c` (lectura completa) `[V-p]`

`seat_set_active`:
1. `s->active = session`.
2. `session_device_pause_all(old_active)`, de forma **síncrona**.
3. `seat_trigger_devices`.
4. `static_node_acl`.
5. `seat_triggered_uevents_done`: reanuda los dispositivos de la sesión nueva.

Consecuencias:
- Los dispositivos viejos se detienen **antes** de reanudar los nuevos.
- Las **ACL `uaccess` cambian después** de la pausa: hay una ventana en la que el usuario anterior todavía tiene ACL sobre `/dev/dri/card*` y `/dev/input/event*` `[H]`.
- Si los pasos 3 o 4 fallan, el 5 no se ejecuta y la sesión nueva no recupera sus dispositivos. Es un fallo cerrado para el greeter.

### 2.3 `logind-session.c` (lectura **truncada** al 65 %) `[V-p]`

| Función | Hecho |
|---|---|
| `session_activate` | En seats con VT: `chvt(s->vtnr)`; la desactivación real llega con `seat_active_vt_changed` (asíncrono, vía kernel) |
| `session_release` | Temporizador de **20 s** (`RELEASE_USEC`) antes de `session_stop` |
| `session_stop` | Para el scope; **no toca dispositivos** |
| `session_finalize` | Libera los dispositivos, emite `SessionRemoved` y `seat_set_active(NULL)` si era la activa |
| `session_free` | `session_drop_controller`, libera los dispositivos |
| **No leído** | `session_set_controller`, `session_drop_controller`, `session_leave_vt`, `session_get_state` (el resumen del resto **no es fiable**) |

### 2.4 `logind-session-dbus.c` (lectura completa) `[V-p]`

- `method_take_control`: `if (uid != 0 && (force || uid != s->user->user_record->uid)) → ACCESS_DENIED`. Cualquier proceso con el **mismo UID** puede tomar el control de **su** sesión; nunca con `force` → **RV-39 confirmado en código**.
- `method_take_device`: requiere ser el controlador; rechaza duplicados; solo tipos verificables.
- Desconexión del controlador: **no leída**.

---

## 3. Evidencia DRM del kernel

**Fuente:** Linux v7.3 (también `drm_auth.c` de v6.17). `[V-p]`. Versión del kernel de Arch: **no comprobada**.

| Pregunta | Respuesta | Evidencia |
|---|---|---|
| ¿Quién puede hacer SET_MASTER? | `drm_master_check_perm`: (`was_master` **y** `file_priv->pid == task_tgid(current)`) o `CAP_SYS_ADMIN`; si no, `-EACCES` | `drm_auth.c` |
| ¿Qué pid se guarda? | El tgid del que abrió el fd. `drm_file_update_pid` lo actualiza en cada ioctl **salvo si `was_master`** | `drm_file.c`, `drm_ioctl.c` |
| fd abierto por logind | logind lo abrió y lo hizo master: `was_master = 1` y pid = logind. El compositor (otro tgid, sin `CAP_SYS_ADMIN`) **no puede recuperar master** sobre ese fd → `-EACCES` | Combinación de lo anterior `[H fuerte]` |
| fd abierto directamente por el compositor | El **primer** opener del nodo primario sin master **se vuelve master implícitamente** (`drm_master_open`) y queda `was_master`. Desde entonces puede recuperar master siempre que no haya otro | `drm_auth.c` → amenaza nueva (RV-44) |
| Errores de DROP_MASTER | `EACCES`, o `EINVAL` si no es el master actual o no hay master | `drm_auth.c` |
| ¿Qué cambia con DROP_MASTER? | `driver->master_drop` y `drm_master_put(&dev->master)`. **No** toca CRTC ni framebuffer en el código leído | `drm_auth.c` |
| ioctls que exigen master | SETCRTC, SETPLANE, CURSOR, PAGE_FLIP, DIRTYFB, ATOMIC, propiedades, gamma, leases. Sin master: `-EACCES` | `drm_ioctl.c` `drm_ioctl_permit` |
| ioctls sin master | ADDFB, CREATE_DUMB, MAP_DUMB, GET*, WAIT_VBLANK | `drm_ioctl.c` |
| fd duplicado | `dup()` o `SCM_RIGHTS` comparten `struct file` y por tanto `drm_file`: el estado de master es compartido | `[H]` modelo VFS |
| Render nodes | Sin modeset ni concepto de master | `drm-uapi.rst` `[V-p]` |

**Clasificación pedida (sesión anterior tras el drop):**

| Capacidad | Estado |
|---|---|
| Enviar comandos de modeset | No (`-EACCES`) **si** el drop se completó y no es master por otra vía |
| Modificar lo presentado | Por ioctls, no. **Escribiendo en la memoria del buffer escaneado:** posible mientras ese buffer siga en pantalla y el proceso conserve un mapeo `[H]`. Desaparece cuando el greeter hace su primer commit con buffers propios |
| Persistencia del último framebuffer | Persiste hasta el siguiente commit `[H]`; el kernel solo restaura en `lastclose` |
| Recuperar master | No con el fd de logind `[H fuerte]`; **sí** con un fd abierto directamente que haya sido master alguna vez (RV-44) |
| Presentar desde otro fd o proceso | Solo si ese fd consigue master: cuando no hay ningún master, una apertura nueva lo obtiene (ventana entre el drop y el SET_MASTER del greeter) |

**Si el greeter no obtiene master:** `TakeDevice` falla (§2.1) y el greeter no puede presentar. La pantalla muestra el último framebuffer, o lo que siga presentando un master viejo si el drop falló. **Esto no expone entrada por sí solo**, pero el usuario podría ver la sesión anterior o un fotograma fijo.

**Pantalla neutral fiable:** solo existe **después** de que el compositor del greeter haga su primer commit atómico con un buffer negro o propio. No hay mecanismo de logind ni del kernel que lo haga antes `[V-p, ausencia en el código leído]`.

---

## 4. Evidencia evdev del kernel

**Fuente:** Linux v7.3, `drivers/input/evdev.c`; `hidraw.c` para HIDIOCREVOKE. `[V-p]`

| Concepto | Resultado |
|---|---|
| Ámbito de `EVIOCREVOKE` | Por `evdev_client`, que es uno por `open()` y se guarda en `file->private_data` |
| Descriptores duplicados | Comparten `struct file`: **revocados también** |
| Procesos que comparten el fd (`fork`, `SCM_RIGHTS`) | Revocados también (misma descripción) |
| Otro `open()` del mismo dispositivo | **No se revoca.** Cliente independiente |
| Tras revocar | read, write e ioctl devuelven `-ENODEV`; poll da `EPOLLHUP\|EPOLLERR`; no se encolan eventos; se suelta el grab (`evdev_ungrab`) |
| Errores de la operación | Argumento no NULL → `-EINVAL`; ya revocado → `-ENODEV` |
| Kernel sin soporte | logind registra el aviso y continúa (§2.1); la versión que lo introdujo no se determinó |

**Distinción pedida:**

| Estado | Significado |
|---|---|
| Sesión sin dispositivo activo | Estado lógico de logind (`sd->active = false`); **no implica** revocación efectiva |
| Descriptor revocado | `ENODEV` en esa descripción |
| Descriptor cerrado | logind cerró **su** copia (`session_device_free`); la del compositor sigue abierta y revocada, si la revocación funcionó |
| Abierto pero sin acceso efectivo | Revocado y no cerrado |
| **Descriptor alternativo utilizable** | Cualquier `open()` directo previo, posible mientras la ACL `uaccess` estaba vigente. **logind no lo cubre** |

Por tanto **no se puede afirmar «la sesión anterior no puede recibir entrada»**. Lo demostrable es: *el fd de entrada que logind entregó fue revocado, si el ioctl no falló*.

**Mitigación candidata nueva (M-GRAB):** que el compositor del greeter aplique `EVIOCGRAB` a cada dispositivo de entrada antes de aceptar credenciales.
- Un grab exclusivo hace que el kernel entregue eventos **solo** al cliente que lo tiene `[H, por la existencia de `evdev_ungrab` en `evdev_revoke`; `evdev_grab` y `evdev_events` no leídos]`.
- Si otro cliente ya tiene el grab, el greeter recibe `EBUSY`, lo que es un fallo **detectable**.
- Cubre los fds directos y los fallos de revocación.
- **No cubre:** fuentes de entrada fuera de evdev (`uinput` inyecta, no lee); dispositivos añadidos por hotplug antes del grab; hidraw (el teclado no se suele leer por hidraw, pero un lector hidraw sí ve los informes HID crudos `[H]`).
- Pendiente de verificar en el código del kernel (T-GRAB).

---

## 5. Secuencia greetd / PAM / logind

**Fuentes y límites:**
- greetd 0.10.3: `worker.rs`, `context.rs`, `interface.rs`, `server.rs` del **mirror de GitHub** `[V-p]`. Las rutas `src/pam/session.rs` y `main.rs` dieron 404.
- Arch: paquete `greetd 0.10.3-2` `[V]` (página del paquete). **PKGBUILD, parches y `/etc/pam.d/greetd` no leídos** (Anubis).
- `pam_systemd(8)` latest `[V]`.
- Seats y VT: systemd.io dio 404.

| # | Evento | Actor | Tipo | Evidencia |
|---|---|---|---|---|
| 1 | El comando de sesión muere | — | — | — |
| 2 | El worker sale de `waitpid` (ignora el estado) | worker | Síncrono | `[V-p]` |
| 3 | `pam_close_session`, `setcred(DELETE)`, `pam_end` | worker | Síncrono en el worker | `[V-p]` |
| 3a | pam_systemd notifica a logind el fin de la sesión; logind arma el temporizador de 20 s (`session_release`) **o** para antes si el scope se vacía | logind | **Asíncrono** | `[V]` pam_systemd sin detalle del mecanismo; `[V-p]` `RELEASE_USEC`; desconocido si pam_systemd está en la pila PAM de Arch `[VM]` |
| 4 | El worker sale; greetd lo recoge (`check_children`) | greetd | Síncrono | `[V-p]` |
| 5 | greetd lanza el greeter (con 1 s de espera si la sesión duró menos de 1 s) | greetd | — | `[V-p]` |
| 6 | El greeter abre su sesión PAM y logind crea una sesión `greeter` | logind | Asíncrono respecto a 3a | `[V]` pam_systemd, clase `greeter` |
| 7 | La sesión del greeter se vuelve activa: `chvt` y después `seat_active_vt_changed` → `seat_set_active` | logind + kernel | **Asíncrono** (VT) | `[V-p]`; el caso de dos sesiones en la misma VT no se ha verificado `[VM]` |
| 8 | Pausa de la sesión vieja (drop + revoke), si era la activa | logind | Síncrono dentro de 7 | `[V-p]` |
| 8' | O bien `session_finalize` libera los dispositivos, si la sesión vieja terminó antes | logind | Asíncrono | `[V-p]` |
| 9 | Cambio de ACL `uaccess` | logind/udev | Después de 8 | `[V-p]`/`[H]` |
| 10 | El compositor del greeter: `TakeControl`, `TakeDevice` (DRM con SET_MASTER), primer commit | greeter | — | `[V-p]` |
| 11 | El greeter presenta y acepta entrada | greeter | — | — |

**Lo que no está garantizado:**
- Entre 4 y 8/8' **no hay ningún orden documentado**. greetd **no espera** a que logind desactive ni elimine la sesión vieja.
- La **seguridad la dan los pasos 10–11** (master obtenido; grab, si se adopta), no el orden de greetd.
- Si greetd muere, el worker recibe `PDEATHSIG` y probablemente no cierra PAM `[H]`. No aparece ningún greeter.

---

## 6. Reevaluación de S1–S5

| Req. | Pregunta | Evidencia | Caminos de error | Sin demostrar | Estado |
|---|---|---|---|---|---|
| S1 | ¿Todos los dispositivos relevantes pasan por logind? | `compositor.md` §3 «libseat/logind» `[R]`; tipos de `detect_device_type` `[V-p]` | Es una propiedad del **código de PARALLAX**, no algo que logind imponga; render nodes, `uinput`, `fb`, `hidraw` fuera de sesión no cubiertos; otros procesos del usuario pueden abrir dispositivos directamente mientras tienen ACL | Que el compositor no abra nada directamente (auditable en código, Fase 1) | `[R]` requisito; cumplimiento `[VM]` |
| S2 | ¿logind revoca pantalla y entrada aunque haya errores? | `session_device_stop` `[V-p]` | Drop ignorado; revoke solo registrado; fd perdido → no se marca inactivo; `warned` | Que nunca falle en Arch | **Refutado como garantía.** Es mejor esfuerzo |
| S3 | ¿El cierre de sesión libera todo? | `session_finalize`/`free` `[V-p]` | Igual que S2; asíncrono (20 s); `drop_controller` no leído | Momento y completitud | `[V-p]` + `[VM]` |
| S4 | ¿greetd cierra PAM antes del greeter? | `worker.rs`/`context.rs` `[V-p]` (mirror) | greetd muere → sin cierre; PAM de Arch no leído | Equivalencia con Arch; que pam_systemd esté en la pila | `[V-p]`; **no implica** desactivación de la sesión logind |
| S5 | ¿El greeter adquiere y conserva el control? | SET_MASTER exclusivo; `session_device_new` falla si no hay master `[V-p]` | Si otro proceso es master (drop fallido, RV-44) → el greeter falla | Que su compositor trate el fallo como fatal | `[V-p]`; **es la condición comprobable útil** |

---

## 7. Reevaluación de L1–L3 y D-31

### L1 — El compositor sale al perder la sesión

| Caso | Efecto |
|---|---|
| `SessionRemoved`, `PauseDevice("force")` | Puede reaccionar si su bucle de eventos funciona |
| Pierde la conexión con logind | Comportamiento de logind al desconectarse el controlador: **no leído** |
| Bucle colgado | No reacciona. **L1 no es una garantía** |
| Vivo sin dispositivos | Consume recursos; no presenta si S5 se comprueba; recibe entrada solo si la revocación falló |
| Intenta `TakeControl` de una sesión posterior | Posible: mismo UID, si nadie tiene el control (RV-39). Mitigación: el compositor nuevo trata como fatal no ser el controlador; L3 limpia antes de arrancar |

### L2 — Parada del user manager

- Hechos `[V]` logind.conf: con `UserStopDelaySec` (10 s por defecto), `user@.service` se detiene «when the last session of the user has ended». Con linger, el usuario puede seguir ejecutando procesos.
- Ya cuenta como sesión **cualquier** otra del usuario: tty, ssh, otro entorno.
- No se leyó qué otras condiciones mantienen vivo al user manager (p. ej. un `systemd-run --user` en marcha sin sesión) `[VM]`.
- **Desactivar linger no basta** para garantizar la parada.
- Procesos que sobreviven: todo lo del user manager hasta que termine la última sesión del usuario.

### L3 — Limpieza en el siguiente inicio

| Aspecto | Estado |
|---|---|
| Parada por nombre (compositor, scopes `parallax-role-*`, `app-parallax-*`) | Diseño `[H]` |
| Unidades con otra nomenclatura (`DBusActivatable`, terceros) | **No cubiertas** (no pertenecen a PARALLAX) |
| Procesos fuera de unidades conocidas | No detectables por nombre. Detectables por contenido de `/proc/*/fd`: **no se propone** (coste y fragilidad) |
| Timeout → `KillUnit SIGKILL` | Diseño |
| Estado D | Sin solución (RV-40) |
| Carrera con el compositor nuevo | El compositor nuevo solo arranca cuando L3 confirma que todas las unidades PARALLAX están `inactive` **y** que `TakeControl` tiene éxito |
| Criterio de «limpio» | Unidades PARALLAX conocidas `inactive`. **El target inactivo no basta** |

### D-31 (para que decida el propietario)

| Opción | Naturaleza | Consecuencia |
|---|---|---|
| a. Exigir linger desactivado | **Política operativa / simplificación v1.** No es necesaria para la garantía de presentación ni de entrada, que se apoyan en §12 | L2 actúa más a menudo; rompe usos de linger (servicios de usuario persistentes) |
| b. Permitir linger | Válida | La limpieza depende de L1 y L3; residuos sin dispositivos |
| c. No decidir en v1 | Válida | Igual que b |

**Corrección a la Tarea 012:** D-31 **no** es una condición de seguridad.

---

## 8. Riesgos (sin renumerar)

| ID | Riesgo | Cambio |
|---|---|---|
| RV-37 | Greeter presente con la sesión anterior viva | **Reformulado:** presentación → condición comprobable (S5). Entrada → no garantizada por logind; depende de M-GRAB (sin verificar). Limpieza → mejor esfuerzo |
| RV-38 | Último framebuffer visible | **Se mantiene**; ampliado: un proceso con un mapeo del buffer escaneado podría modificarlo hasta el primer commit del greeter `[H]` |
| RV-39 | Carrera de `TakeControl` | **Confirmado en código** v262.2 (`method_take_control`) `[V-p]`. Detectable por el compositor nuevo |
| RV-40 | Estado D | Se mantiene |
| RV-41 | Diferencias de versión de systemd | **Reducido pero abierto:** se leyó v262.2 upstream; parches de Arch desconocidos |
| RV-42 | greetd upstream frente a Arch | **Abierto:** solo se verificó el número de versión; PKGBUILD, parches y PAM no leídos |
| **RV-43** (nuevo) | Fds de entrada abiertos directamente durante la sesión activa no se revocan al cambiar de sesión; el nuevo usuario (**incluso otro usuario**) puede teclear credenciales legibles por un proceso del usuario anterior | Evidencia: `evdev.c` por cliente + `seat_set_active` (la ACL cambia después; los fds abiertos persisten). Mitigación: M-GRAB |
| **RV-44** (nuevo) | Un proceso que abrió `/dev/dri/card*` directamente cuando no había master obtiene master implícito y `was_master`; puede recuperar master en cualquier hueco sin master (incluido el hueco entre el drop y el SET_MASTER del greeter) | Evidencia: `drm_master_open`, `drm_master_check_perm`. Mitigación: S5 detecta el `EBUSY`; el greeter no presenta |
| **RV-45** (nuevo) | Los fallos de `EVIOCREVOKE` y `drmDropMaster` solo se registran en el log; logind continúa | Evidencia: `session_device_stop`, `sd_eviocrevoke` |

---

## 9. Máquina de estados corregida

No es lineal; cada flecha indica si es síncrona, asíncrona o **no garantizada**.

```text
ACTIVE
 ├─(comando muere / SIGKILL / SIGTERM sin handler)──► TERMINATING
 │
TERMINATING ──síncrono(worker)──► WORKER_REAPED?   ← no: primero PAM_CLOSED dentro del worker
TERMINATING ──síncrono──► PAM_CLOSED ──síncrono──► WORKER_REAPED
PAM_CLOSED ──asíncrono, no garantizado (pam_systemd en Arch [VM]; 0–20 s)──► LOGIND_SESSION_CLOSING
WORKER_REAPED ──greetd──► (greeter lanzado)
(greeter lanzado) ──asíncrono (chvt)──► GREETER_SESSION_ACTIVE
GREETER_SESSION_ACTIVE ──síncrono en logind, mejor esfuerzo──► DEVICES_REVOKED?   (puede quedar "solicitado pero fallido": RV-45)
LOGIND_SESSION_CLOSING ──asíncrono──► (finalize) ──► DEVICES_REVOKED?
GREETER_SESSION_ACTIVE ──TakeDevice DRM ok + commit inicial──► SCREEN_NEUTRAL
SCREEN_NEUTRAL ──EVIOCGRAB ok en todos los dispositivos (M-GRAB)──► GREETER_PRESENTING (acepta credenciales)
SCREEN_NEUTRAL ──TakeDevice/GRAB falla──► REFUSED (greeter no acepta entrada; mensaje fijo; reintento con backoff)
ACTIVE/cualquier ──► CLEANUP_PENDING (residuos en user@) ──L1/L2/L3──► CLEANED
CLEANUP_PENDING ──nuevo login──► L3 ──ok──► nueva sesión | ──fallo──► REFUSED
```

`DEVICES_REVOKED` **no es observable** de forma fiable desde fuera de logind. Por eso no se usa como condición de seguridad.

---

## 10. Matriz de escenarios

Abreviaturas: **M** = DRM master del greeter obtenido (S5); **G** = grab exclusivo obtenido (M-GRAB); **R** = revocación por logind (mejor esfuerzo).

| # | Escenario | Detecta | Inicia | PAM | logind | DRM | Entrada | Sobreviven | Greeter | Condición de seguridad | Si falla una operación | Prueba |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Logout normal | — | Compositor | Cerrado | Cierre normal | Liberado | Liberada | Terceros hasta L2 | Normal | M+G | — | T-SESSION-FAIL |
| 2 | Caída del compositor | systemd | `BindsTo` | Cerrado al salir el comando | Cierra | El fd muere con el proceso | Igual | Roles y apps hasta `PartOf`/L2 | Normal | M+G | — | T-SESSION-FAIL |
| 3 | Caída del comando | greetd | Nadie en el user manager | Cerrado | Asíncrono | R; **M lo comprueba** | R; **G lo comprueba** | Compositor (sin dispositivos si R) | Tras M+G | M+G | R falla → M/G fallan → REFUSED | T-DRM, T-GRAB |
| 4 | SIGTERM al comando | Comando | L4 | Cerrado | — | Como 1 o 3 | — | — | — | M+G | Como 3 | — |
| 5 | SIGKILL al comando | Como 3 | | | | | | | | | | |
| 6 | Caen comando y compositor | greetd + systemd | `BindsTo` | Cerrado | Cierra | fds cerrados al morir | Igual | Roles, apps | Normal | M+G | — | T-SESSION-FAIL |
| 7 | Una unidad PARALLAX no se detiene | L3 | L3 SIGKILL | — | — | Si tiene fds: R/M | R/G | Esa unidad | Ya presentado | M+G | Nuevo login → REFUSED | T-LOCK |
| 8 | Timeout de terminación | systemd | SIGKILL | — | — | — | — | Estado D | — | M+G | RV-40 | — |
| 9 | User manager activo | — | L1/L3 | — | — | Sin dispositivos si R | Igual | Todo lo de user@ | Normal | M+G | — | T-KILL |
| 10 | Otra unidad mantiene `graphical-session.target` | — | L3 por nombre | — | — | — | — | Terceros | Normal | M+G | — | T-GRAFO |
| 11 | Login inmediato | Comando nuevo | L3 + `TakeControl` fatal | — | Sesión nueva | Compositor nuevo con M | G no aplica (sesión del usuario) | Hasta L3 | — | L3 limpio + `TakeControl` ok | REFUSED | T-LOCK, T-DRM |
| 12 | Apagado | systemd | systemd | Puede no cerrarse | — | — | — | Nada | Ninguno | — | — | — |
| 13 | Falla `DROP_MASTER` | Greeter (SET_MASTER `EBUSY`) | — | — | Continúa (RV-45) | **El viejo sigue siendo master** | — | Compositor viejo | **No presenta** | M falla | Pantalla con la sesión vieja **visible**; sin entrada si G/R | T-DRM |
| 14 | Falla `EVIOCREVOKE` | Greeter (`EBUSY` si el viejo tiene grab; si no, **nadie**) | — | — | Continúa | — | **El viejo sigue leyendo** | — | Presenta | **Solo G** lo detecta o lo impide | Sin G: fuga de credenciales | T-GRAB |
| 15 | Greeter sin master | Greeter | — | — | — | — | — | — | No presenta | — | Fotograma fijo; REFUSED con reintento | T-DRM |
| 16 | greetd muere | systemd (sistema) | Reinicio de greetd si su unidad lo hace `[VM]` | Probablemente no se cierra | Scope huérfano | — | — | Sesión vieja **activa** en la VT | Ninguno hasta reiniciar | La sesión vieja sigue activa y desbloqueada: **fuera del alcance de RV-37** (no aparece el greeter), pero es un riesgo propio | — | T-GREETD |
| 17 | Varias sesiones del mismo usuario | — | — | — | — | — | — | user@ no se detiene | Normal | M+G | L2 no actúa | T-KILL |
| 18 | Otro compositor en el mismo seat | Greeter (M o G fallan) | — | — | — | Otro master | Otro grab | — | No presenta | M+G | REFUSED | T-DRM |
| 19 | fd DRM/evdev abierto fuera de logind | Greeter (M o G) | — | — | **No lo cubre** | RV-44 | RV-43 | El proceso | Según M/G | M+G | Sin G: fuga de entrada | T-GRAB, T-DRM |
| 20 | Proceso viejo hace `TakeControl` en el nuevo inicio | Compositor nuevo (su `TakeControl` falla) | L3 | — | El viejo es controlador | El viejo puede obtener dispositivos de la sesión nueva | Igual | Compositor viejo | — | `TakeControl` fallido = fatal + L3 | Sesión rechazada | T-DRM |

---

## 11. Alternativas arquitectónicas adicionales

| Alternativa | Resuelve | No resuelve | Dependencia | Debe sobrevivir | Superficie | Verificación | Coste |
|---|---|---|---|---|---|---|---|
| **M-GRAB + comprobación de M en el greeter** (recomendada) | RV-43, RV-45 para evdev, escenarios 13, 14, 19 | hidraw, fotograma fijo, limpieza | Semántica de EVIOCGRAB `[H]` | Nada nuevo | Ninguna nueva | T-GRAB, T-DRM | Bajo; requisito del greeter |
| Compositor dentro del scope de sesión (G-a) | Limpieza ligada a `KillUserProcesses` | Entrada directa (RV-43) | Pierde el grafo del user manager | — | — | T-SESSION-FAIL | Medio; contradice D-28 |
| Supervisor independiente | Nada que M+G no detecte; no ve fds ajenos | — | Proceso nuevo | Sí | Nueva | — | Alto. **No justificado** |
| Cgroup dedicado (`cgroup.kill`) | Matar todo lo de PARALLAX de golpe | Procesos fuera del cgroup | Delegación (ADR-0014) | — | — | VM | Medio |
| Pantalla neutral previa | RV-38 | — | Requiere master: es el primer commit del greeter | — | — | T-DRM | Incluido en M |
| No reutilizar el seat hasta confirmar | Solo si hubiera una señal fiable; `DEVICES_REVOKED` no es observable | — | — | — | — | — | No viable tal cual |

Nota: ninguna dependencia systemd del user manager puede detener unidades del system manager, ni al revés `[H]`.

---

## 12. Condición mínima para permitir el greeter (propuesta)

El greeter (su compositor) **no acepta entrada de credenciales** hasta cumplir **todas** estas condiciones:

1. Ser controlador de su sesión logind y obtener con `TakeDevice` el DRM `card*` **activo** (master obtenido).
2. Haber hecho un primer commit con contenido propio (pantalla neutral).
3. Haber obtenido `EVIOCGRAB` sobre **cada** dispositivo de entrada que use (M-GRAB).
4. Ante cualquier fallo de 1–3: no aceptar entrada, mostrar un aviso fijo si puede presentar, y reintentar con backoff (el detalle es una decisión, D-32).

**Propiedades que da (si se verifican T-DRM y T-GRAB):**
- La sesión anterior no presenta (1).
- Ningún otro cliente evdev recibe las pulsaciones dirigidas al greeter (3).

**Propiedades que no da:**
- Exposición del último fotograma antes de 2.
- Lectura por hidraw.
- Limpieza de procesos.
- Aislamiento de sesiones.

**Si M-GRAB no se valida, no existe una condición comprobable para la entrada.** En ese caso RV-37, en su parte de entrada, queda **NO RESUELTO**.

---

## 13. Pruebas pendientes (no autorizadas)

Prioridad 1: **T-GRAB** y **T-DRM**. Determinan si existe garantía alguna.

| ID | Contenido |
|---|---|
| **T-GRAB** (nueva) | Leer `evdev_grab` y `evdev_events` del kernel de Arch. En VM: (a) un proceso con fd directo previo, (b) revocación fallida simulada, (c) el greeter hace EVIOCGRAB → verificar que el otro cliente no recibe eventos y que un grab previo produce `EBUSY` |
| **T-DRM** | Drop por logind; el compositor viejo no recupera master (`EACCES`); fd abierto directamente en el hueco (RV-44); el greeter obtiene `EBUSY` si el viejo es master; fotograma visible hasta el primer commit |
| T-SESSION-FAIL | Ampliada con los escenarios 13–20 |
| T-GREETD | PKGBUILD y parches de Arch; `/etc/pam.d/greetd` con pam_systemd; muerte de greetd |
| T-LOGIND | Leer el resto de `logind-session.c` (controlador, `leave_vt`, estados); desconexión del controlador |
| T-LOCK, T-KILL | Como antes, con L3 por nombre |

---

## 14. Propuesta de actualización de trazabilidad (no aplicada)

**El §14 de la Tarea 012 debe corregirse antes de aplicarse.** No se aplica tal cual:

| Punto de la Tarea 012 §14 | Decisión |
|---|---|
| RV-37 «seguridad por arbitraje de logind S1–S5» | **Corregir:** presentación por S5 (condición comprobable); entrada por M-GRAB (sin verificar); logind como mejor esfuerzo |
| Añadir RV-38 a RV-42 | Aplicable, con los textos de §8 de este informe; **añadir RV-43, RV-44 y RV-45** |
| §4.3, fila 4 | Corregir con la matriz §10 (filas 3, 13, 14, 19) |
| D-29: L3 por nombre | Aplicable |
| D-24 | Corregir: la inaccesibilidad **no** se apoya en logind solo |
| D-28: alternativas G-a y B1 | Aplicable |
| D-31 como condición | **Corregir:** política operativa, no requisito de seguridad |
| T-SESSION-FAIL S8/S9, T-DRM, T-BIND | Aplicable, con **T-GRAB** añadida y prioridad 1 para T-GRAB y T-DRM |
| Nueva **D-32** [P] | Comportamiento del greeter cuando falla la condición §12 |
| `lockscreen-greeter.md` (tras decisión) | Requisito §12 para el greeter |

---

## 15. Veredicto final: `MITIGACIÓN PARCIAL, GARANTÍA NO DEMOSTRADA`

**Justificación:**
- **S2 queda refutado como garantía:** el código v262.2 continúa la transición aunque fallen `drmDropMaster` o `EVIOCREVOKE`.
- **S1 no lo impone logind:** los fds abiertos directamente no se cubren (RV-43, RV-44).
- **S4 ordena PAM, no logind.**
- **S5 sí da una condición comprobable para la presentación.**
- **Para la entrada no hay garantía demostrada.** M-GRAB es una mitigación candidata cuya semántica del kernel no se ha leído.
- Escenarios críticos 13, 14 y 19: sin M-GRAB, el 14 y el 19 permiten que un proceso de la sesión anterior lea lo que se escribe en el greeter.

**Cuándo cambiaría el veredicto:**
- A `RESUELTO EN DISEÑO, PENDIENTE DE VALIDACIÓN`: si la lectura del kernel confirma la semántica de EVIOCGRAB y el propietario acepta los límites (último fotograma, hidraw, limpieza de mejor esfuerzo).
- A `NO RESUELTO` en la parte de entrada: si no se confirma.

No se modificó el repositorio. Ningún ADR cambió (todos `Propuesto`). La Fase 0 sigue `IN_PROGRESS` y la Fase 1 no está autorizada.
