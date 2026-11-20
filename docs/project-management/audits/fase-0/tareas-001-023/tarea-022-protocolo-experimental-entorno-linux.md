# Tarea 022 — Corrección del protocolo experimental y diseño del entorno Linux

- **Fecha:** 2026-11-19
- **Estado del proyecto:** Fase 0 `IN_PROGRESS` · todos los ADR `Propuesto` · Fase 1 sin autorización
- **Naturaleza:** solo lectura. Es una propuesta pendiente de aprobación. El informe está en el scratchpad y no se modificó el repositorio.
- **Marcas de evidencia:** `[V]` código leído en local, en el tag exacto · `[V-p]` evidencia parcial · `[R]` repositorio · `[H]` inferencia · `[VM]` requiere ejecución · `[P]` decisión del propietario

---

## 1. Resumen ejecutivo

**Veredicto: `PROTOCOLO EXPERIMENTAL VALIDADO DOCUMENTALMENTE; ENTORNO PENDIENTE`** (§13).

Este veredicto **no** significa que se hayan ejecutado pruebas ni que algún riesgo esté resuelto.

### Correcciones principales sobre la Tarea 021

1. **Cómo saber si un fd es master.**
   - No se usa `AUTH_MAGIC`, que es lo que hace `drmIsMaster()` de libdrm.
   - Observación directa: `/sys/kernel/debug/dri/<N>/clients`. `drm_debugfs.c` imprime una columna `master` calculada con `drm_is_current_master(priv)` para cada cliente `[V]`. Requiere root y debugfs.
   - Para comprobar la autorización: se lanza un ioctl con bandera `DRM_MASTER` y **argumentos deliberadamente inválidos**. `drm_ioctl_kernel` ejecuta `drm_ioctl_permit` **antes** del handler (`drm_ioctl.c` líneas 808–813) `[V]`, así que:
     - `EACCES` significa sin autorización;
     - otro errno (`EINVAL`, `ENOENT`) significa que la autorización pasó y el rechazo vino de los argumentos.
2. **Los ioctls de presentación no devuelven 0 de forma incondicional.** Depende del estado del CRTC, de los FB y del driver. Para separar permisos de efecto se usan dos pasos:
   - una sonda de permisos con argumentos inválidos;
   - una prueba de efecto con un estado válido preparado, verificada con `GETCRTC`/`GETPLANE` y una captura de la VM (`screendump` de QEMU).
3. **`DROP_MASTER` desde el hijo.** `drm_master_check_perm` acepta `was_master && pid == tgid` **o `CAP_SYS_ADMIN`** `[V]`.
   - Si el hijo de la prueba es **root**, `DROP_MASTER` **tiene éxito**. La expectativa de `EACCES` en la Tarea 021 era incorrecta para un hijo root.
   - Para medir la restricción, el hijo debe **perder sus privilegios** (`setresuid` a un usuario de prueba y capacidades vacías) antes del intento.
4. **Hotplug.** Hay que medir las marcas de tiempo de cada fase y demostrar que las credenciales **nunca** se habilitan con un dispositivo sin capturar. No basta con comprobar que acabó capturado.
5. **Entorno.**
   - El anfitrión actual es **macOS ARM64** `[R ADR-0013]`. Una VM x86_64 con KVM **no es posible** ahí; solo emulación TCG (lenta) o una VM aarch64, que no es la plataforma de v1.
   - DEV-01 (Linux x86_64) es el anfitrión KVM natural, pero **no está autorizado**.

---

## 2. Base documental

- Tareas 012–021 en el scratchpad.
- Código en `scratchpad/src/`: Linux v7.3, systemd v262.2, BRLTTY-6.9.1.
- Empaquetado de Arch en `scratchpad/arch-pkg/`: systemd 262.2-1, brltty 6.9.1-3.
- Repositorio PARALLAX (`[R]`):
  - ADR-0013: anfitrión macOS ARM64; VM con aprobación del propietario.
  - base-audit-r3 §4.2: DEV-01 es x86_64 y su ficha está pendiente.

---

## 3. Auditoría de las siete pruebas

Columnas:

| Abreviatura | Pregunta |
|---|---|
| Obj | ¿El objetivo está bien definido? |
| Prec | ¿Las precondiciones son suficientes? |
| Pasos | ¿Los pasos producen el estado que se quiere comprobar? |
| Depende del driver | ¿El resultado esperado depende del driver? |
| Ambigüedad | ¿El resultado puede ser ambiguo? |
| Medición más directa | ¿Hay una forma mejor de medirlo? |
| Afecta al anfitrión | ¿Puede afectar a la sesión anfitriona? |
| HW | ¿Necesita hardware físico? |

Las cuestiones de root y aislamiento se tratan en las matrices de los §7 y §9.

| Prueba | Obj | Prec | Pasos | Depende del driver | Ambigüedad | Medición más directa | Afecta al anfitrión | HW |
|---|---|---|---|---|---|---|---|---|
| T-FD-INHERIT | Sí | **Insuficientes**: no exigían VT libre sin otro master ni fbcon controlado | El primer opener solo es master si no hay otro | Poco | Sí, si se usa `AUTH_MAGIC` | debugfs `clients` + sonda `EACCES` | No, en VM | No |
| T-DRM-INHERITED-IOCTL | Sí | Faltaba un estado válido (FB, CRTC, conector) | `ATOMIC TEST_ONLY`/`PAGE_FLIP` pueden fallar por estado | **Sí** | Sí: no separaba permiso de argumentos | Sonda con argumentos inválidos + efecto con estado válido | No | Sí, para drivers concretos |
| T-LOGIND-SHARED-REVOKE | Sí | Faltaba el controlador de sesión real (TakeControl) | Correctos | Poco | Bajo | debugfs + sonda + `read` | No | No |
| T-LOGIND-GC | Sí | Faltaba fijar `UserStopDelaySec` y confirmar `KillUserProcesses` | Correctos | No | Bajo | `loginctl` + `systemctl show -p ActiveState` con marcas de tiempo | No | No |
| T-BRLTTY-PASTE | Sí | Faltaba una línea braille (o simulador) y la configuración del driver de pantalla | Dependen del hardware braille | No | Medio | `evtest` sobre el teclado virtual + traza del greeter | No | **Sí**, probablemente |
| T-TIOCSTI | Sí | Faltaban usuario y VT de prueba | Correctos | No | Bajo | Matriz de errno | No | No |
| T-UINPUT-HOTPLUG | Incompleto: no medía la ventana | Falta instrumentar el greeter de prueba | Corregidos (§4.E) | No | Alta si solo se mira el resultado final | Marcas de tiempo `CLOCK_MONOTONIC` + `SEQNUM` del uevent | No | No |

---

## 4. Correcciones técnicas

### A. Observación del estado DRM

| Lo que se quiere saber | Método | Evidencia |
|---|---|---|
| El fd sigue abierto | `ls -l /proc/<pid>/fd` → enlace a `/dev/dri/cardN` | Identificador del dispositivo (`stat` → `st_rdev`) |
| El `drm_file` sigue vivo | `/sys/kernel/debug/dri/<N>/clients`: una fila por `drm_file` (comm, pid, master) | `drm_debugfs.c` `[V]`; el **pid** mostrado es `file->pid`, que no cambia tras ser master |
| Es master | Columna `master` en `clients` | `drm_is_current_master(priv)` `[V]` |
| Puede usar ioctls protegidos | **Sonda:** `DRM_IOCTL_MODE_SETCRTC` con `crtc_id=0` (inexistente) | `EACCES` = sin autorización; `ENOENT`/`EINVAL` = autorizado (`drm_ioctl_permit` va antes del handler) `[V]` |
| Cambia realmente la pantalla | `SETCRTC` válido (FB propio de color conocido) → `GETCRTC.fb_id` == FB propio + `screendump` de QEMU con el color esperado | En hardware real: captura externa o CRC si está disponible |

### B. Ioctls DRM

| Resultado | Cómo se reconoce |
|---|---|
| Rechazo por permisos | `EACCES` en la sonda |
| Argumentos inválidos | `EINVAL`/`ENOENT` con la sonda (esperado) |
| Estado de hardware | `EBUSY`, `EINVAL` con un estado válido en un CRTC ocupado |
| Restricción del driver | Errno distinto con el mismo estado entre drivers → se registra por driver |
| Éxito | 0 + `GETCRTC`/`GETPLANE` coherentes + captura |

`ATOMIC TEST_ONLY` **no** es adecuado como sonda de permisos, porque valida el estado y puede devolver `EINVAL` aunque haya autorización. Se usa solo como prueba de efecto con un estado válido.

`PAGE_FLIP` exige un CRTC activo con un FB compatible.

### C. `DROP_MASTER`

Ruta de autorización: `drm_dropmaster_ioctl` → `drm_master_check_perm`, que acepta `was_master && pid == task_tgid(current)` **o** `capable(CAP_SYS_ADMIN)` `[V]`. Si pasa, devuelve `EINVAL` cuando el fichero no es el master actual.

**No** utiliza la bandera `DRM_MASTER` de `drm_ioctl_permit`.

| Hijo | Resultado esperado |
|---|---|
| Root con `CAP_SYS_ADMIN` | 0, y se suelta el master |
| Sin privilegios, sin ser el opener | `EACCES` |
| El opener original (mismo tgid) | 0 |

### D. `EVIOCGRAB`

| Estado | Cómo se observa |
|---|---|
| Grab activo | Un tercer cliente, con su propio `open()`, hace `EVIOCGRAB` → `EBUSY` |
| Lectura de eventos | Eventos inyectados por un uinput de prueba recibidos en el hijo |
| Eventos ya encolados | Inyectar **antes** del grab sin leer; tras el grab, el cliente sin grab lee los eventos previos y ninguno posterior |
| Revocación | `EVIOCREVOKE` → `ENODEV` en todas las copias |
| Cierre de la última referencia | Tras el cierre, el tercer cliente hace grab → 0 |
| Apertura independiente | Un `open()` nuevo es otro cliente y no recibe nada mientras haya grab |

### E. Hotplug: marcas de tiempo

| Marca | Fuente |
|---|---|
| T0 `UI_DEV_CREATE` devuelve | Herramienta uinput de prueba, `CLOCK_MONOTONIC` |
| T1 uevent `add` del kernel (`SEQNUM`) | `udevadm monitor -k -p` (marcas de tiempo del monitor) |
| T2 nodo `eventN` disponible (evento de udev) | `udevadm monitor -u -p` |
| T3 descubrimiento por el greeter | Registro del greeter de prueba |
| T4 `TakeDevice`/apertura | Ídem |
| T5 `EVIOCGRAB` = 0 | Ídem |
| T6 estado de credenciales | Ídem; se registra cada cambio |

**Criterio:** las credenciales deben estar deshabilitadas en todo el intervalo [T3, T5]. Y para **cualquier** evento inyectado en [T0, T5], el greeter **no** lo procesa como credencial.

T0–T3 no está controlado por el greeter: se **mide** y se documenta (RV-48).

---

## 5. Diseño del entorno Linux (no se crea)

| Elemento | Propuesta |
|---|---|
| Anfitrión | **Linux x86_64 con KVM.** DEV-01 sería el candidato, pero requiere autorización. Alternativa: macOS ARM64 con QEMU TCG x86_64 (lento) o una VM aarch64 (no es la plataforma de v1; los resultados de kernel serían de otra arquitectura) |
| Invitado | Arch Linux x86_64, versiones **fijadas** con Arch Linux Archive en una fecha concreta |
| Firmware y arranque | OVMF (UEFI), systemd-boot, Btrfs (coherente con ADR-0009, sin probar) |
| Kernel | `linux` de Arch en la fecha fijada. **Versión exacta: se registra al preparar, sin inventarla** |
| Paquetes | `systemd 262.2-1`, `greetd 0.10.3-2`, `brltty 6.9.1-3` (versiones confirmadas en las Tareas 009 y 017); `libinput`, `evtest`, `libdrm` (utilidades de prueba como `modetest`), `strace`, `bpftrace` (opcional) |
| GPU virtual | `virtio-gpu` con KMS y atómico |
| Entrada | `virtio-keyboard`/`usb-kbd` de QEMU + uinput del invitado. **Ningún passthrough** de dispositivos del anfitrión |
| Usuarios | `pxadmin` (sudo) y `pxtest` (sin privilegios, sin grupos `input` ni `video`); contraseñas de prueba generadas, nunca reales |
| Sesiones | greetd con un greeter de prueba mínimo; una sesión de prueba, sin compositor de producción |
| Registros | `journalctl -o short-monotonic`, salidas de `udevadm monitor`, `strace -f -tt`, debugfs `clients` |
| Snapshots | Snapshot QEMU antes de cada prueba destructiva |
| Aislamiento | Red NAT sin puertos expuestos; sin carpetas compartidas con escritura; sin USB passthrough en los perfiles A y B |

---

## 6. Perfiles

| Perfil | Entorno | Pruebas |
|---|---|---|
| **A** — Kernel y dispositivos virtuales | VM KVM, virtio-gpu, uinput | T-FD-INHERIT, T-DRM-INHERITED-IOCTL (parte virtio), T-TIOCSTI, T-UINPUT-HOTPLUG |
| **B** — Sesiones y logind | Misma VM, con greetd y logind | T-LOGIND-SHARED-REVOKE, T-LOGIND-GC, T-UINPUT-HOTPLUG (con el greeter), T-BRLTTY-PASTE (parte sin hardware) |
| **C** — Hardware físico | HW-A/HW-B con i915/amdgpu, teclado USB real y línea braille | T-DRM-INHERITED-IOCTL (drivers reales), T-BRLTTY-PASTE (completa), confirmación de A y B |

---

## 7. Matriz de cobertura

| Prueba | VM KVM | Hardware real | Root | Sesión gráfica | Riesgo | Evidencia |
|---|---|---|---|---|---|---|
| T-FD-INHERIT | Sí | Confirmación | Sí | No (VT) | Bajo en VM | debugfs, sonda, `evtest` |
| T-DRM-INHERITED-IOCTL | Solo virtio | **Sí** (i915, amdgpu) | Sí | No | Medio (pantalla) | Sonda, `GETCRTC`, captura |
| T-LOGIND-SHARED-REVOKE | Sí | Confirmación | Sí | Mínima | Bajo | debugfs, errno, journal |
| T-LOGIND-GC | Sí | No | Sí | No | Bajo | `loginctl`/`systemctl` + marcas de tiempo |
| T-TIOCSTI | Sí | No | Sí (para la matriz) | No | Bajo | errno |
| T-UINPUT-HOTPLUG | Sí | Confirmación con USB real | Sí | Greeter de prueba | Bajo | T0–T6 |
| T-BRLTTY-PASTE | Parcial (sin línea real) | **Sí** | Sí | Greeter de prueba | Bajo | `evtest`, registro del greeter |

**Diferencias entre drivers** (no se suponen equivalentes):

| Driver | Particularidades |
|---|---|
| virtio-gpu | Sin overlays reales; flips simulados |
| i915 | Planos múltiples, PSR |
| amdgpu | DC, cursor y overlay propios |
| NVIDIA propietario | Módulo fuera del árbol: **fuera del alcance de todo lo leído** |

Una prueba solo en virtio-gpu **no** valida el comportamiento en hardware.

---

## 8. Procedimientos de medición

- **Procesos y fds:**
  - `ps -o pid,tgid,ppid,uid,comm`;
  - `ls -l /proc/<pid>/fd` y `cat /proc/<pid>/fdinfo/<n>` (incluye `drm-client-id` en DRM `[H]`);
  - `stat` del nodo (`st_rdev`);
  - marca de tiempo de cada `close` con `strace -tt -e close`.
- **DRM:**
  - debugfs `name` y `clients`;
  - errno de cada ioctl;
  - `modetest -c -p` (CRTC/planos) antes y después;
  - `screendump` de QEMU.
- **evdev y uinput:**
  - `udevadm info -q all` (syspath, devnum, VID:PID);
  - `udevadm monitor -k -u -p`;
  - `evtest` por cliente;
  - errno de `EVIOCGRAB` y `EVIOCREVOKE`.
- **systemd y logind:**
  - `loginctl show-session -p State,Active,Scope`;
  - `systemctl show session-N.scope user@UID.service -p ActiveState,SubState`;
  - `loginctl show-user -p Linger`;
  - `systemd-analyze cat-config systemd/logind.conf`;
  - `journalctl -o short-monotonic -u systemd-logind`.
- **Seguridad:**
  - solo credenciales de prueba;
  - VM sin passthrough;
  - sin la sesión habitual;
  - snapshot y restauración;
  - temporizador de seguridad que restaura la consola si la VT queda en un modo gráfico tras una prueba DRM (por ejemplo, `chvt` desde SSH al invitado).

---

## 9. Criterios de aceptación

Clasificación de cada prueba: **CONFIRMADO** (la evidencia positiva coincide con la hipótesis), **REFUTADO** (la evidencia negativa la contradice), **INCONCLUSO** (precondición no cumplida, errno inesperado sin explicación o ruido) o **NO APLICABLE** (el entorno no tiene el elemento, por ejemplo sin línea braille).

| ID | Objetivo | Precondiciones | Instrumentación | Positivo | Negativo | Inconcluso | Errno esperado | Riesgo | Recuperación | Entorno |
|---|---|---|---|---|---|---|---|---|---|---|
| T-FD-INHERIT | El hijo retiene el master y el grab | VT libre, sin otro master (debugfs), `pxadmin` | debugfs, sonda, `evtest`, uinput | El hijo aparece `master=y`; la sonda da `ENOENT`/`EINVAL`; lee eventos; un tercero recibe `EBUSY` | El hijo no es master o no lee | Ya había otro master al empezar | Sonda ≠ `EACCES` | Bajo | Matar los procesos; snapshot | A |
| T-DRM-INHERITED-IOCTL | R2 y R3 en el hijo heredero | fd obtenido con TakeDevice; hijo sin privilegios | Sonda, `GETCRTC`, captura | Sonda autorizada; efecto visible; `DROP_MASTER` del hijo sin privilegios = `EACCES`; greeter `SET_MASTER` = `EBUSY`; tras el `DROP_MASTER` de logind, sonda = `EACCES` | Cualquier divergencia | Driver que rechaza el estado válido por motivos de hardware | Ver texto | Medio | `chvt`; snapshot | A (virtio), C |
| T-LOGIND-SHARED-REVOKE | La revocación alcanza a las copias | Sesión de prueba con TakeControl | debugfs, sonda, `read` | Tras activar otra sesión: hijo `master=n`, sonda `EACCES`, `read` `ENODEV` | El hijo conserva acceso | logind no llegó a desactivar (journal) | `EACCES`, `ENODEV` | Bajo | Snapshot | B |
| T-LOGIND-GC | Una sesión `closing` fija `user@` | `KillUserProcesses=no` confirmado; delay 10 s | `loginctl`, `systemctl`, marcas de tiempo | `user@` activo con el hijo vivo; parado en ≤ delay + ε tras su muerte; TTY/SSH/linger lo mantienen; `KillOnlyUsers` mata el scope | `user@` parado con el hijo vivo | Configuración distinta de la supuesta | — | Bajo | Snapshot | B |
| T-TIOCSTI | Matriz de `legacy_tiocsti` | Usuario y VT de prueba | errno | Matriz completa y coherente | — | — | `EIO`/`EPERM` según el caso `[H]` | Bajo | `sysctl` restaurado | A |
| T-UINPUT-HOTPLUG | Ninguna credencial con un dispositivo sin grab | Greeter de prueba instrumentado | T0–T6 | Credenciales deshabilitadas en [T3, T5]; ningún evento de [T0, T5] aceptado como credencial; `EBUSY` → bloqueo; reconciliación tras saturar el monitor | Credencial habilitada con un dispositivo `PENDING` | Instrumentación incompleta | — | Bajo | Snapshot | B |
| T-BRLTTY-PASTE | RV-56 | brltty + línea braille; VT `K_OFF` | `evtest` del teclado virtual; registro del greeter | Texto del portapapeles recibido por el greeter | No se recibe con todas las precondiciones cumplidas | Sin hardware → **NO APLICABLE** | — | Bajo | Detener brltty | C |

---

## 10. Orden de ejecución (propuesta `[P]`)

| Grupo | Pruebas | Motivo |
|---|---|---|
| **RV-37** | 1. T-FD-INHERIT, 2. T-DRM-INHERITED-IOCTL (virtio), 3. T-LOGIND-SHARED-REVOKE, 5. T-LOGIND-GC | Deciden si la revocación de logind basta frente a los hijos |
| **Seguridad del greeter** | 4. T-UINPUT-HOTPLUG | Ventana de credenciales |
| **Complementaria** | 6. T-TIOCSTI | Afecta solo a la ruta de consola |
| **Accesibilidad** | 7. T-BRLTTY-PASTE | Requiere hardware |

Se mantiene el orden propuesto, con un ajuste: **T-LOGIND-SHARED-REVOKE pasa justo después de T-FD-INHERIT.** Si la revocación funciona sobre las copias, la gravedad práctica del R3 de T-DRM-INHERITED-IOCTL depende solo de la ventana hasta la desactivación de la sesión.

---

## 11. Riesgos y limitaciones

- **Anfitrión.** macOS ARM64 no puede ejecutar KVM x86_64. Sin DEV-01 o sin otro anfitrión Linux, el perfil A solo es viable con emulación (tiempos no representativos) o en aarch64, que es otra arquitectura.
- **Kernel de Arch desconocido** (RV-46). Se fija al preparar el entorno.
- **debugfs:** requiere root y `CONFIG_DEBUG_FS`; el kernel de Arch lo habilita `[H]`.
- **virtio-gpu no representa a i915 ni a amdgpu.** NVIDIA queda fuera.
- **brltty sin línea braille:** T-BRLTTY-PASTE queda NO APLICABLE, salvo que exista un simulador; no se verificó que lo haya.
- **Herramientas de prueba:** las pruebas necesitan pequeños programas (sonda de ioctls, inyector uinput, greeter instrumentado). **No se han escrito.** Escribirlos requiere autorización explícita (D-38, ver §12).

---

## 12. Decisiones pendientes del propietario

1. ¿Autorizar la preparación de una VM Linux aislada? ¿En qué anfitrión: DEV-01, otro Linux x86_64 o emulación en el Mac?
2. ¿Autorizar la ejecución de las pruebas en esa VM, por perfil o por prueba?
3. ¿Autorizar el acceso **de solo lectura** a DEV-01 para registrar hardware y versiones (ficha DEV-01)?
4. ¿Autorizar más adelante pruebas en hardware físico (perfil C), con qué equipos y si hay una línea braille disponible?
5. ¿Qué riesgos bloquean Gate 0 y cuáles pueden quedar abiertos con mitigación documentada (RV-37, RV-48, RV-50, RV-55, RV-56)?
6. **Nueva, D-38 (propuesta):** ¿autorizar la escritura de herramientas de prueba desechables (sonda de ioctls, inyector uinput, greeter instrumentado) **fuera** del producto, conforme a las reglas de spikes?

No se solicita autorización para la Fase 1.

---

## 13. Veredicto final: `PROTOCOLO EXPERIMENTAL VALIDADO DOCUMENTALMENTE; ENTORNO PENDIENTE`

- Las siete pruebas están corregidas. Cada una tiene un criterio binario o una clasificación explícita, instrumentación definida y una separación clara entre permiso, argumentos, estado de hardware, driver y efecto.
- El entorno está diseñado, pero **no existe**. Depende de la decisión 1 y del anfitrión.
- No se ha ejecutado ninguna prueba. Ningún riesgo cambia de estado.

---

## ESTADO DEL PROYECTO

| Elemento | Estado |
|---|---|
| Fase 0 | `IN_PROGRESS` |
| ADR | Todos `Propuesto` |
| Fase 1 | Sin autorización |
| RV-37 | Abierto |
| D-28 y D-31 a D-37 | Pendientes (D-38 propuesta) |
| Repositorio | Sin modificaciones |
| Pruebas | Ninguna ejecutada |
| VM | No creada |
| DEV-01 | No accedido |
