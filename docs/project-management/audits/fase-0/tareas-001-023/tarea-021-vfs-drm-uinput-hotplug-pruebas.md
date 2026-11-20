# Tarea 021 — VFS, autorización DRM, hotplug de uinput y pruebas

- **Fecha:** 2026-11-19
- **Estado del proyecto:** Fase 0 `IN_PROGRESS` · todos los ADR `Propuesto` · Fase 1 sin autorización
- **Naturaleza:** solo lectura. Es una propuesta. El informe está en el scratchpad y el repositorio no se ha tocado.
- **Marcas de evidencia:**
  - `[V]` código leído en local, en el tag exacto
  - `[V-p]` evidencia parcial
  - `[H]` inferencia
  - `[VM]` requiere máquina real
  - `[P]` decisión del propietario

---

## 1. Resumen ejecutivo

**Veredicto: `EVIDENCIA DOCUMENTAL SUFICIENTE; PRUEBAS PREPARADAS`** (§15).

Este veredicto significa que la documentación y las pruebas están listas. **No** significa que RV-37 esté resuelto.

### A. VFS

Queda demostrado en el código de Linux v7.3:

- `fork` copia la tabla de descriptores y hace `get_file()` de cada `struct file` (`dup_fd`).
- `dup`/`dup2` y `SCM_RIGHTS` también hacen `get_file()` sobre la **misma** `struct file`.
- `fput` solo llama a `__fput` cuando el contador `f_ref` llega a 0.
- `__fput` llama a `f_op->release`.
- `exec` cierra solo los descriptores marcados en `close_on_exec`.

**Conclusión:** la vida de `private_data` (`drm_file`, `evdev_client`) termina **con la última referencia, sea del proceso que sea** `[V]`.

### B. DRM

| Resultado | Pregunta | Respuesta |
|---|---|---|
| **R1** | ¿El hijo retiene el master? | **Sí** |
| **R2** | ¿Puede usar los ioctl protegidos? | **Sí.** `drm_ioctl_permit` solo comprueba `drm_is_current_master(file_priv)`, **no el TGID**. El hijo puede hacer `SETCRTC`, `ATOMIC`, `PAGE_FLIP`, `SETPLANE` y `CURSOR` |
| — | ¿Puede hacer `SET_MASTER`/`DROP_MASTER`? | **No.** `drm_master_check_perm` exige `was_master` y que `pid` coincida con el TGID del llamante, o `CAP_SYS_ADMIN`. El `pid` es el del proceso que abrió el fd (logind, o el compositor si abrió directamente) y no se actualiza tras haber sido master. El hijo recibe `EACCES` |
| **R3** | ¿Puede cambiar la pantalla? | **Sí** mientras siga siendo master, salvo restricciones propias del driver `[VM]` |
| — | `DROP_MASTER` de logind (root) | Pasa la comprobación con `capable` y quita el master a todas las copias |

### C. uinput y hotplug

- La cadena `UI_DEV_CREATE` → `uinput_create_device` → `input_register_device` → `device_add` (uevent) → `input_attach_handler` (`evdev_connect`, creación de `eventN`) está **leída en el código**.
- **El dispositivo existe y puede emitir eventos antes de que el greeter lo descubra.**
  - Esos eventos llegan a los handles abiertos, por ejemplo el handler de teclado de la VT `[H]`, y a los procesos que hayan abierto el nodo. Para eso necesitan permisos: root, `input` o la ACL de brltty.
  - El grab del greeter **no** se aplica a un dispositivo nuevo hasta que lo abre y lo captura.
- **No existe ninguna operación atómica de enumeración y grab.** La pérdida de eventos del monitor udev obliga a reconciliar.

Clasificación: **HOTPLUG_REDUCIDO**, la misma que en la Tarea 015.

### D. Pruebas preparadas, no ejecutadas

1. T-FD-INHERIT
2. T-LOGIND-SHARED-REVOKE
3. T-LOGIND-GC
4. T-BRLTTY-PASTE
5. T-TIOCSTI
6. **T-UINPUT-HOTPLUG**
7. **T-DRM-INHERITED-IOCTL**

---

## 2. Alcance y versiones

| Fuente | Ref | Archivos nuevos en esta tarea |
|---|---|---|
| Linux | `v7.3` (commit del checkout `c03c1a14507d…`) | `fs/file_table.c`, `fs/file.c`, `fs/exec.c`, `net/core/scm.c`, `net/unix/af_unix.c`, `drivers/input/{input.c, misc/uinput.c}`, `drivers/gpu/drm/{drm_ioctl.c, drm_auth.c, drm_file.c}` |
| BRLTTY | `BRLTTY-6.9.1` | Rutas ya leídas en la Tarea 020 |
| systemd | `v262.2` | Rutas ya leídas en la Tarea 020 |

**Límites de esta lectura:**

- **El kernel de Arch sigue siendo desconocido** (RV-46).
- No se leyeron drivers DRM concretos (i915, amdgpu, virtio-gpu) ni NVIDIA.
- No se leyó el código de udevd ni el de libudev.
- No se leyó `vt.c` ni `keyboard.c` para comprobar el modo `K_OFF`.

---

## 3. VFS: referencias y ciclo de vida

| # | Pregunta | Respuesta | Evidencia |
|---|---|---|---|
| 1 | Descriptor frente a `struct file` | El descriptor es un índice en la tabla `fdtable` del proceso. La `struct file` es la descripción abierta, compartible, con su propio contador `f_ref` | `fs/file.c` `[V]` |
| 2 | `fork` | `dup_fd` recorre los descriptores y hace `get_file(f)` de cada uno. También copia el bitmap `close_on_exec` | `fs/file.c` líneas 383–462 (131, 458) `[V]` |
| 3 | `dup` / `dup2` | `do_dup2` / `get_file(file)` sobre la misma `struct file` | `fs/file.c` líneas 1292 y 1330 `[V]` |
| 4 | `SCM_RIGHTS` | Emisor: `scm_fp_copy` (referencias en `scm_fp_list`). Receptor: `scm_detach_fds` → `__scm_install_fd`, con `get_file(f)` y la **misma** `struct file` | `net/core/scm.c` líneas 69, 366 y 378 `[V]` |
| 5 | Contador de referencias | Sube con `get_file`. Baja con `fput` (`file_ref_put`) | `fs/file_table.c` línea 587 `[V]` |
| 6 | Muere el propietario original | Se cierran sus descriptores: un `fput` por cada uno | `[H]` (`exit_files` no leído) |
| 7 | Se cierra un descriptor con otra referencia viva | `fput` resta una referencia, sin llegar a `__fput` | `[V]` |
| 8 | Cuándo se ejecuta `.release` | `__fput` → `f_op->release(inode, file)`, solo cuando `f_ref` llega a 0 | `fs/file_table.c` líneas 486 y 512 `[V]` |
| 9 | `FD_CLOEXEC` | Es un bit **por descriptor** en `fdt->close_on_exec`. `do_close_on_exec`, llamado desde `exec` (`fs/exec.c` línea 1201), cierra los marcados. **No** protege contra `fork` sin `exec`, `dup` (el descriptor nuevo no hereda el flag `[H]`), `SCM_RIGHTS` (el receptor elige sus propios flags) ni `pidfd_getfd` | `[V]` |
| 10 | `O_CLOEXEC` frente a `FD_CLOEXEC` | `O_CLOEXEC` pone `FD_CLOEXEC` al abrir el descriptor, de forma atómica; el resultado es el mismo bit | `[H]` (`fs/open.c` no revisado) |
| 11 | ¿`SCM_RIGHTS` conserva el estado? | Sí: es la misma `struct file`, así que se conservan `private_data`, el offset y los flags de estado | `[V]` |
| 12 | Qué podría invalidar lo anterior | Drivers que no usen `private_data` por fichero; `close_range(CLOSE_RANGE_UNSHARE)`; espacios de nombres; LSM (Landlock, SELinux) que bloqueen ioctls | `[H]` |

**Esquema de referencias:**

```text
inicial:        A.fd3 ──► file(f_ref=1) ──► private_data = drm_file / evdev_client
fork:           A.fd3 ─┐
                B.fd3 ─┴► file(f_ref=2)              (dup_fd: get_file)
dup:            A.fd3, A.fd7 ─► file(f_ref=+1)       (do_dup2: get_file)
SCM_RIGHTS:     A.fd3 ─┐
                C.fd5 ─┴► file(f_ref=+1)             (scm_detach_fds: get_file)
cierre padre:   B.fd3 ──► file(f_ref=1)              (fput; sin __fput)
cierre hijo:    file(f_ref=0) → __fput → f_op->release (drm_release / evdev_release)
exec CLOEXEC:   do_close_on_exec → fput del descriptor marcado
exec sin CLOEXEC: el descriptor sobrevive en la imagen nueva
```

---

## 4. `SCM_RIGHTS`

`SCM_RIGHTS` transfiere **referencias** a la misma `struct file`.

Consecuencia práctica: el descriptor que logind envía con `TakeDevice` y la copia que guarda logind (`sd->fd`) son **la misma descripción abierta** `[V]`. Por eso un `DROP_MASTER` o un `EVIOCREVOKE` sobre la copia de logind alcanza también a la del compositor y a la de cualquier hijo.

---

## 5. DRM: autorización de ioctls heredados

| Operación | ¿Requiere master? | ¿Comprueba TGID? | Resultado para el hijo heredero (fd master) | Evidencia |
|---|---|---|---|---|
| `MODE_SETCRTC` | Sí (`DRM_MASTER`) | No | **Permitida** | `drm_ioctl.c` línea 687 + `drm_ioctl_permit` `[V]` |
| `MODE_SETPLANE` | Sí | No | Permitida | línea 689 `[V]` |
| `MODE_CURSOR` | Sí | No | Permitida | línea 690 `[V]` |
| `MODE_PAGE_FLIP` | Sí | No | Permitida | línea 706 `[V]` |
| `MODE_ATOMIC` | Sí | No | Permitida | línea 714 `[V]` |
| `SET_MASTER` | Comprobación propia | **Sí**: `was_master && pid == task_tgid(current)`, o `CAP_SYS_ADMIN` | **`EACCES`**: el `pid` es el de quien abrió el fd y no se actualiza (`drm_file_update_pid` sale antes si `was_master`) | `drm_auth.c` líneas 233–243; `drm_file.c` `[V]` |
| `DROP_MASTER` | Ídem | Ídem | **`EACCES`** | Ídem |
| `GETFB` / `GETFB2` | Devuelve handle solo a master o `CAP_SYS_ADMIN` | No | Recibe el handle (es master) | Tarea 015 `[V-p]` |
| `ADDFB`, `CREATE_DUMB`, `MAP_DUMB`, `GET*` | No | No | Permitidas; no cambian la pantalla por sí solas | Tarea 013 `[V-p]` |

**Si el compositor abrió el nodo directamente** (sin logind), el `pid` guardado es el del compositor. Al morir este, nadie coincide con ese `pid`, y solo `CAP_SYS_ADMIN` podría soltar el master `[V]`.

**`DROP_MASTER` de logind:** logind es root, así que pasa `capable()`. `drm_dropmaster_ioctl` sobre el `drm_file` compartido suelta el master para todas las copias `[V]`.

---

## 6. Retención del master frente a capacidad de presentación

- **R1, retención:** sí, mientras exista cualquier referencia y nadie haga `DROP_MASTER`.
- **R2, autorización de ioctls protegidos:** sí. No se comprueba el TGID.
- **R3, presentación efectiva:** sí, si el hijo puede componer buffers y hacer commit. Depende del driver `[VM]`.
- **Interferencia sin modeset:** retener el master bloquea el `SET_MASTER` del greeter, que recibe `EBUSY`. Es un problema de disponibilidad.
- **Recursos compartidos sin master:**
  - GEM y mmap del propio archivo;
  - dmabufs exportados;
  - render nodes, que no presentan.

---

## 7. BrlTTY: creación del dispositivo uinput

Cadena del lado de brltty `[V]` (Tarea 020):

1. `openKeyboard` → `newUinputKeyboard("Linux Screen Driver Keyboard")`.
2. `newUinputObject`: `open("/dev/uinput", O_WRONLY)`, `system_linux.c` línea 997.
3. `enableKeyboardKeys` (`UI_SET_KEYBIT`) y `EV_REP`.
4. `createUinputDevice` → `ioctl(UI_DEV_CREATE)`, línea 1198.

Cadena del lado del kernel `[V]`:

- `uinput.c` línea 939, `case UI_DEV_CREATE` → `uinput_create_device` (línea 327) → `input_register_device` (línea 383).

**Momento de creación:**
- `openKeyboard` se llama en `construct_LinuxScreen`, al arrancar el driver (Tarea 019 `[V-p resumen]`).
- Por tanto, el teclado virtual existe **desde que brltty arranca**, no solo cuando se pega `[H]`.

---

## 8. Kernel input: registro y hotplug

`input_register_device` (`input.c` líneas 2385–2455) `[V]`:

1. `device_add(&dev->dev)` → uevent `add` para `inputN`.
2. Con `input_mutex`, `list_add_tail(&dev->node, &input_dev_list)`.
3. `input_attach_handler` para cada handler:
   - `evdev_connect` → `cdev_device_add` → nodo `eventN` y su uevent `[V]`;
   - el handler de teclado de la VT, si coincide.

**Cuándo hay entrega de eventos:**
- `input_pass_values` solo entrega a handles abiertos.
- El handle evdev se abre cuando el primer cliente abre el nodo (`evdev_open_device` → `input_open_device`, `evdev.c` líneas 385–396) `[V]`.
- El handler de teclado de la VT abre el dispositivo al conectar `[H]` (`keyboard.c` no leído).

**Qué no hace el kernel:**
- No hay ningún mecanismo que haga grab automático de un dispositivo nuevo.
- El grab de un `input_dev` **no se extiende** a otros dispositivos `[V]` (`dev->grab` es de cada dispositivo).

---

## 9. Greeter: enumeración, suscripción y adquisición (diseño de las Tareas 014–015)

| Fase | Mecanismo | ¿Cierra la ventana o solo la reduce? |
|---|---|---|
| Suscripción a udev antes de enumerar | `udev_monitor` + enumeración | **Reduce.** Evita perder dispositivos entre enumeración y suscripción, pero no es atómico |
| Apertura | `TakeDevice` (logind) | No cierra nada |
| `EVIOCGRAB` | Exclusivo por `input_dev` | **Cierra** la recepción por otros **desde ese momento** |
| Errores / `EBUSY` | `UNPROTECTED` → bloqueo de credenciales | Cierra en fallo (diseño) |
| Pérdida de eventos udev | Reconciliación periódica (re-enumerar) | Reduce |
| Cambio de nombre o duplicados | Identificar por `devnum` + `syspath` | Reduce |

---

## 10. Matriz de carreras

| # | Escenario | ¿Recibe otro proceso antes del grab? | ¿Llega al greeter? | Lo cierra / lo reduce |
|---|---|---|---|---|
| 1 | uinput antes de la enumeración inicial | Sí, desde que existe (la VT y quien tenga el nodo abierto) | Sí, tras el grab | El grab inicial lo cierra **después** |
| 2 | Después de la enumeración | Sí | Tras el `add` de udev y el grab | Monitor udev: reduce |
| 3 | Entre enumeración y suscripción | Sí | Solo si la suscripción es anterior o hay reconciliación | Suscribir antes: reduce |
| 4 | Entre suscripción y apertura | Sí | Tras el grab | Reduce |
| 5 | Tras abrir otros dispositivos | Sí | Tras el grab | Reduce |
| 6 | Durante la autenticación | Sí | El dispositivo pasa a `PENDING` y se deshabilitan las credenciales | Reduce (RV-48) |
| 7 | Desaparece y reaparece | Sí | Nuevo `PENDING` | Reduce |
| 8 | Se pierde el evento de hotplug | Sí | No, hasta la reconciliación | Reconciliación: reduce |
| 9 | `EBUSY` al hacer grab | El dueño del grab recibe | No | Cierre en fallo |
| 10 | El dispositivo virtual emite durante la adquisición | Sí, a los handles abiertos | Tras el grab, sí | Reduce |

**Ninguna ventana se cierra de forma atómica.** El grab solo cierra la recepción **a partir** de su aplicación.

---

## 11. Especificaciones de las siete pruebas (no ejecutadas)

Condiciones comunes a todas:

- VM o máquina **dedicada**, con el kernel y los paquetes de Arch registrados (`uname -r`, versiones de `pacman -Q`).
- Sin credenciales reales.
- Sin la sesión habitual del usuario.
- Instantánea previa de la VM.
- **Ninguna prueba se ejecuta en DEV-01 sin autorización.**

### T-FD-INHERIT

1. Un programa de prueba (root, VT libre, sin compositor) abre `/dev/dri/card0`; se convierte en master porque es el primero.
2. Abre `/dev/input/eventX` y hace `EVIOCGRAB`.
3. Hace `fork`. El hijo no cierra nada.
4. El padre termina (`SIGKILL`).
5. El hijo comprueba:
   - `drmIsMaster` (o `AUTH_MAGIC`) → master;
   - lectura de eventos de `eventX` → recibe;
   - grab de un tercer proceso sobre `eventX` → `EBUSY`.
6. El hijo cierra; el tercero repite el grab → 0, y un tercero hace `SET_MASTER` → 0.

### T-LOGIND-SHARED-REVOKE

1. Sesión de prueba con `TakeControl` + `TakeDevice` (DRM y evdev).
2. `fork`: el hijo conserva los fds.
3. Se activa otra sesión en el seat (`loginctl activate`).
4. El hijo comprueba:
   - un ioctl `DRM_MASTER` (`SETCRTC` con argumentos de prueba) → `EACCES`;
   - `read` de evdev → `ENODEV`.
5. Variante de fallo: no hay forma segura de forzar el fallo de logind; se documenta como no comprobable.

### T-LOGIND-GC

1. Sesión gráfica de prueba con un hijo inmortal en el scope.
2. Cerrar la sesión. Registrar:
   - `loginctl show-session` (`State=closing`);
   - `systemctl status session-N.scope` (activo);
   - `user@UID` (activo).
3. Matar al hijo. Comprobar que la sesión se elimina y que `user@` se para tras unos 10 s.
4. Repetir con:
   - una sesión TTY abierta;
   - una sesión SSH abierta;
   - linger activo;
   - `KillOnlyUsers=<usuario>`.

### T-BRLTTY-PASTE

1. brltty con línea braille de prueba (o simulada, si existe) y el driver de pantalla Linux.
2. VT del greeter de prueba en `K_OFF` (verificar con `KDGKBMODE`).
3. El cliente BrlAPI escribe en el portapapeles.
4. Se pulsa PASTE en la línea braille.
5. Capturar `evtest` del dispositivo «Linux Screen Driver Keyboard» y los eventos recibidos por el greeter de prueba.

### T-TIOCSTI

Matriz de condiciones:
- `sysctl dev.tty.legacy_tiocsti` a 0 y a 1;
- un proceso con `CAP_SYS_ADMIN` y otro sin ella;
- VT propia y VT ajena.

Registrar el resultado del ioctl en cada combinación.

### T-UINPUT-HOTPLUG

1. Greeter de prueba ya inicializado con sus grabs.
2. Crear un dispositivo uinput.
3. Registrar:
   - momento de `UI_DEV_CREATE`;
   - uevent `add` (`udevadm monitor -k -u`);
   - `PENDING` en el greeter;
   - `TakeDevice` + grab;
   - eventos recibidos.
4. Variantes:
   - otro proceso ya tiene el grab → `EBUSY` y bloqueo;
   - desconexión y reconexión;
   - eventos udev perdidos (saturar la cola del monitor) → reconciliación.

### T-DRM-INHERITED-IOCTL

1. Igual que T-FD-INHERIT, pero con el fd obtenido por `TakeDevice` (opener = logind).
2. Tras la muerte del padre, el hijo:
   - `ATOMIC` en modo `TEST_ONLY` → se espera 0;
   - `PAGE_FLIP` → se espera 0;
   - `DROP_MASTER` → se espera `EACCES`.
3. El greeter de prueba hace `SET_MASTER` → se espera `EBUSY`.
4. Después, logind `DROP_MASTER` (activar otra sesión): el hijo hace `ATOMIC` → se espera `EACCES`.

---

## 12. Criterios de aceptación

| Prueba | Requisito | Precondiciones | Pasos | Resultado esperado | Resultado de fallo | Evidencia | Entorno |
|---|---|---|---|---|---|---|---|
| T-FD-INHERIT | Confirmar R1 y la herencia del grab | VM, root, VT libre | §11 | El hijo es master y lee; el tercero recibe `EBUSY`; tras cerrar, 0 | Cualquier otro resultado | PID/TGID, `ls -l /proc/<pid>/fd`, códigos de ioctl | VM con KMS (virtio-gpu) |
| T-LOGIND-SHARED-REVOKE | Confirmar que la revocación alcanza a las copias | Sesión de prueba, logind | §11 | `EACCES` / `ENODEV` en el hijo | El hijo conserva el acceso | Errno, `loginctl`, journal de logind | VM |
| T-LOGIND-GC | Confirmar que `closing` fija al usuario | Usuario de prueba | §11 | `user@` activo con el hijo vivo; parado unos 10 s después de su muerte | `user@` parado con el hijo vivo | `loginctl`, `systemctl` con marcas de tiempo | VM |
| T-BRLTTY-PASTE | Confirmar RV-56 | brltty, VT `K_OFF` | §11 | Clasificación binaria: el greeter recibe o no recibe el texto | Ambiguo → repetir | `evtest`, registro del greeter | **Hardware braille** o simulador |
| T-TIOCSTI | `legacy_tiocsti` | Kernel de Arch | §11 | Matriz completa de resultados | — | Errno por celda | VM |
| T-UINPUT-HOTPLUG | Hotplug en el greeter | Greeter de prueba | §11 | El dispositivo nuevo queda capturado o se bloquean las credenciales; **ningún** evento llega a las credenciales antes del grab | Credenciales habilitadas con un dispositivo sin grab | Marcas de tiempo del kernel, udev y el greeter | VM |
| T-DRM-INHERITED-IOCTL | R2 y R3 | VM con KMS | §11 | 0/0/`EACCES`/`EBUSY` y después `EACCES` | Otra combinación | Errno por paso | VM; **hardware real** para drivers concretos |

**Pruebas destructivas o que afectan a la sesión:** T-FD-INHERIT, T-DRM-INHERITED-IOCTL y T-LOGIND-*. Necesitan una VM aislada y una instantánea previa.

---

## 13. Reconciliación de riesgos y decisiones (sin cambio normativo)

| ID | Hallazgo de la Tarea 021 | Tipo |
|---|---|---|
| RV-37 | Un hijo con master **puede presentar** (R2 sin comprobación de TGID) | Refuerza el riesgo; requiere prueba |
| RV-43 | La herencia de evdev y su grab, demostradas en el VFS | Cierra una laguna documental |
| RV-44 | Un hijo no puede recuperar el master tras el `DROP_MASTER` de logind; uno con `CAP_SYS_ADMIN`, sí | Cierra una laguna |
| RV-46 | Kernel de Arch desconocido | Sigue abierto |
| RV-48 | Hotplug no atómico, confirmado en el código | Refuerza; requiere prueba |
| RV-50 | El teclado uinput de brltty existe desde el arranque | Refuerza |
| RV-55 | — | Sin cambio (requiere prueba) |
| RV-56 | — | Requiere T-BRLTTY-PASTE |
| D-28 | G-a′ no evita R1, R2 ni R3 en los hijos | Requiere decisión del propietario |
| D-31 | — | Requiere decisión del propietario |
| D-32 | Política ante `EBUSY` heredado | Requiere decisión del propietario |
| D-33 | Hotplug durante la autenticación | Requiere decisión del propietario |
| D-34, D-35, D-36, D-37 | — | Requieren decisión del propietario |

No se elige ninguna de las variantes G-a, G-b, G-a′ ni la alternativa E.

**Nueva incertidumbre:** si los drivers concretos imponen comprobaciones adicionales a los ioctls de presentación (por ejemplo, el *lessee* o el estado de la VT) `[VM]`.

---

## 14. Limitaciones y fuentes que faltan

- Kernel de Arch.
- Drivers DRM concretos.
- `vt.c` y `keyboard.c` (modo `K_OFF`, handler de la VT).
- Código de udevd y libudev (pérdida de eventos, orden).
- `fs/open.c` (`O_CLOEXEC`) y `exit_files`, no revisados en esta tarea.
- Smithay/libseat: si ponen `K_OFF`.

---

## 15. Veredicto final: `EVIDENCIA DOCUMENTAL SUFICIENTE; PRUEBAS PREPARADAS`

**Justificación:**
- Las cuatro cuestiones del encargo tienen evidencia de código literal en el tag exacto: el VFS, la autorización DRM (R1, R2, R3), la cadena uinput–input y la no atomicidad del hotplug.
- Las siete pruebas tienen pasos, resultado esperado, resultado de fallo, evidencia que recoger y entorno necesario.
- Lo que queda pendiente es comportamiento en ejecución y versiones concretas, que **no** pueden resolverse con más lectura.

**RV-37 no está resuelto.** El hallazgo de esta tarea lo **refuerza**: un hijo con master puede presentar en pantalla.

---

## ESTADO DEL PROYECTO

| Elemento | Estado |
|---|---|
| Fase 0 | `IN_PROGRESS` |
| ADR | Todos `Propuesto` |
| Fase 1 | Sin autorización |
| RV-37 | Abierto |
| D-28 y D-31 a D-37 | Pendientes |
| Repositorio | Sin modificaciones |
| Pruebas | Ninguna ejecutada |
| DEV-01 | No accedido |
