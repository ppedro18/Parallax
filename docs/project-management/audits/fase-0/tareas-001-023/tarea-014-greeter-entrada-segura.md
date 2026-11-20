# Tarea 014 — Protocolo de entrada segura del greeter

- **Fecha:** 2026-11-19 · **Fase 0:** `IN_PROGRESS` · **ADR:** todos `Propuesto` · **Fase 1:** sin autorización.
- **Naturaleza:** investigación y diseño. **Propuesta pendiente de aprobación.** El repositorio no se ha tocado; este informe vive en el scratchpad.
- **Marcas:** `[V]` fuente suficiente · `[V-p]` parcial · `[R]` repositorio · `[H]` inferencia · `[VM]` requiere prueba · `[P]` propietario.
- **Método:** todo el código se leyó con una herramienta web que devuelve texto procesado por un modelo, sin comparar con un checkout local. Por eso **todo el código citado lleva `[V-p]`**, aunque la lectura fuera completa. Las partes que la herramienta solo resumió se marcan `[V-p resumen]`.
- **Versión del kernel:** Linux **v7.3** upstream (`raw.githubusercontent.com/torvalds/linux/v7.3`). **No se conoce la versión del kernel de Arch** que usará PARALLAX: nada de lo dicho aquí se atribuye a Arch (RV-46).

---

## 1. Resumen ejecutivo

**Veredicto: `VIABLE CON LIMITACIONES EXPLÍCITAS`** (§16).

El código de v7.3 confirma la propiedad central de M-GRAB:

- `EVIOCGRAB` fija un grab **por dispositivo de entrada** (`input_dev->grab`). Mientras está activo, `input_pass_values` entrega los eventos **solo** a ese handle.
- Quedan excluidos de la entrega:
  - todos los demás clientes evdev, incluidos los fds abiertos directamente y los compartidos;
  - el teclado de la VT;
  - `mousedev`.
- Si ya hay un grab, el greeter recibe `EBUSY`, y lo puede detectar.
- Al cerrar el fd o revocarlo, el grab se libera.

**Lo que M-GRAB no cubre**, y debe declararse:

1. **`hidraw`** recibe los informes HID crudos por otro camino, sin concepto de grab.
2. **Eventos ya encolados** en el búfer de otros clientes antes del grab, que siguen siendo legibles.
3. **Ventana de hotplug** entre la creación de un nodo nuevo y el grab del greeter. El kernel no ofrece ninguna operación atómica de enumeración más grab.
4. **Entrada inyectada** por `uinput`. Es suplantación, no lectura de credenciales.
5. Fuentes de entrada que no pasan por evdev.

**Presentación:** se puede llegar a «nuevo estado enganchado por el hardware en un vblank» (commit atómico bloqueante o evento de flip). **No** se puede demostrar con lo leído que los píxeles visibles sean los del greeter.

**La garantía que sí puede ofrecerse:** «Mientras el greeter acepta credenciales, ningún otro cliente evdev de ningún dispositivo de entrada evdev del seat recibe los eventos nuevos que produce el usuario, y la sesión anterior no puede cambiar el estado de presentación DRM». **No** cubre hidraw, ni fuentes que no sean evdev, ni la ventana de hotplug.

---

## 2. Evidencia del kernel sobre EVIOCGRAB

Linux v7.3. `evdev.c` se leyó completo; `input.c` de forma literal en la parte que contiene el grab y la entrega; `uapi/linux/input.h` completo. Todo `[V-p]`.

| Elemento | Código | Hecho |
|---|---|---|
| Estado | `struct evdev_client __rcu *grab;` en `struct evdev` | El grab apunta a **un cliente** (una descripción de archivo) |
| `evdev_grab` | `if (evdev->grab) return -EBUSY; error = input_grab_device(&evdev->handle); ... rcu_assign_pointer(evdev->grab, client);` | Doble nivel: cliente evdev y `input_dev->grab` = handle |
| `input_grab_device` (input.c) | `if (dev->grab) return -EBUSY; rcu_assign_pointer(dev->grab, handle);` con `dev->mutex`; puede dar `-EINTR` | Exclusivo por `input_dev`, sea cual sea el handler |
| `evdev_ungrab` | `if (grab != client) return -EINVAL;` y después `synchronize_rcu()`, `input_release_device` | Solo el dueño puede soltarlo |
| Ioctl | `case EVIOCGRAB: if (p) return evdev_grab(...); else return evdev_ungrab(...);` dentro de `evdev_ioctl_handler`, que devuelve `-ENODEV` si el dispositivo no existe o el cliente está revocado | Cualquier argumento distinto de 0 equivale a grab |
| Entrega evdev | `client = rcu_dereference(evdev->grab); if (client) evdev_pass_values(client, ...); else list_for_each... ` | Con grab, solo el cliente que lo tiene |
| Entrega en el núcleo | `handle = rcu_dereference(dev->grab); if (handle) { handle->handle_events(...); break; }` | Excluye **todos** los demás handles: kbd/VT, mousedev, joydev, evbug |
| Inyección | `input_inject_event`: `if (!grab \|\| grab == handle) input_handle_event(...)` | Con grab, se descartan las escrituras evdev de otros clientes |
| No afectado por el grab | `input_get_disposition` actualiza `dev->key` y `dev->led`; `dev->event()` sigue llamándose | El **estado** del dispositivo (teclas pulsadas, LED) sigue visible por `EVIOCGKEY` u otros ioctls de consulta para cualquier cliente `[V-p]` → RV-47 |
| `evdev_open` | Sin comprobación de capacidades: `file->private_data = client` | El acceso lo controlan los permisos y ACL del nodo |
| Búfer previo | `evdev_grab` **no vacía** los búferes de otros clientes; `evdev_read` no comprueba el grab | Los eventos encolados **antes** del grab siguen siendo legibles |
| Cierre | `evdev_release` llama a `evdev_ungrab(evdev, client)` | Al cerrar el fd del dueño se libera el grab |
| Revocación | `evdev_revoke` llama a `evdev_ungrab` | Igual |
| Desconexión | `evdev_cleanup` → `evdev_hangup`; `input_close_device` → `__input_release_device`, que **suelta** `dev->grab` | El dispositivo y el grab desaparecen |
| SysRq, teclado de la VT | Son handles en `dev->h_list`; los excluye `input_pass_values` | `kbd_event` y `sysrq.c` **no se leyeron** `[H]` |

### 2.1 Respuestas a §4.1 del encargo

| # | Pregunta | Respuesta |
|---|---|---|
| 1–2 | ¿Qué objeto guarda el grab? | `input_dev->grab` (handle evdev) más `evdev->grab` (cliente). El grab se aplica al **dispositivo de entrada lógico** y lo posee una **descripción de archivo** |
| 3–5 | `dup`, `fork`, `SCM_RIGHTS` | Comparten la descripción y por tanto el cliente y su grab `[V-p private_data + H VFS]` |
| 6 | Antes del grab | Reciben eventos todos los clientes abiertos y todos los handlers |
| 7 | Después del grab | Solo el dueño |
| 8 | El dueño cierra | Se libera; la entrega vuelve a todos |
| 9 | El dispositivo se desconecta | Se libera; el nodo desaparece |
| 10 | Segundo grab | `-EBUSY`, también para el mismo cliente |

### 2.2 Errores y carreras (§4.2)

| Caso | Resultado |
|---|---|
| `EBUSY` | Hay otro dueño. **Detectable** por el greeter |
| `EINVAL` | Ungrab sin ser el dueño |
| `ENODEV` | Dispositivo eliminado, o el cliente del greeter fue revocado |
| `EINTR` | Mutex interrumpido: reintentar |
| `EACCES` | En `open()`, por permisos del nodo; no en el ioctl |
| Entre `open()` y `EVIOCGRAB` del greeter | Otros clientes reciben los eventos de esa ventana. **No hay grab atómico al abrir** |
| Entre el grab y la llegada de eventos | Ninguna carrera: con RCU, tras `rcu_assign_pointer` la entrega va al dueño `[V-p]`. Los eventos «en vuelo» de un lote ya despachado pueden haber llegado a otros `[H]` |
| Grab frente a desconexión | Si el dispositivo desaparece, `ENODEV` y el dispositivo pasa a `DISCONNECTED` |
| Desaparición durante la inicialización | Igual: el dispositivo no cuenta como `PROTECTED`, y no hace falta que lo esté |

**¿Hay una ventana tras empezar el protocolo?** Sí: entre que un nodo existe o se abre y el grab del greeter. La regla de §9 hace que en esa ventana **no se acepten credenciales**. Lo que el usuario teclee en ese intervalo puede llegar a otros clientes, pero no es una credencial porque el campo está deshabilitado. Queda el riesgo de que el usuario teclee su contraseña «a ciegas» antes de tiempo (RV-48).

---

## 3. Alcance real del grab (§4.3)

| Fuente | ¿Protegida por M-GRAB? |
|---|---|
| Otros clientes evdev del mismo nodo (directos, compartidos, de otro UID) | **Sí**, para eventos nuevos |
| Otros fds del propio greeter sobre el mismo nodo | No recibirían nada (cliente distinto) |
| Procesos que comparten la descripción del greeter | Comparten el grab: no deben existir. El fd se recibe de logind y nunca se pasa a otros procesos |
| Otro dispositivo físico | Solo si también se le hace grab |
| **Varios nodos evdev de un mismo dispositivo físico** (p. ej. un teclado con nodos para teclas y multimedia) | Cada `input_dev` se protege por separado: **hay que hacer grab de todos** |
| `mousedev` (`/dev/input/mice`), teclado de la VT | Sí, mientras el `input_dev` origen esté en grab |
| **`hidraw`** | **No.** `hidraw_report_event` copia el informe a todos los lectores sin grab `[V-p]`; la ruta en hid-core es `[V-p resumen]` |
| **`uinput`** | El dispositivo virtual se puede capturar como cualquier otro. **No** se impide que su creador inyecte eventos (suplantación) |
| Dispositivos añadidos por hotplug | Solo tras el grab |
| Fuentes fuera de evdev (serie, Bluetooth HID por hidraw, braille por USB o serie) | **No** |

**Conclusión:** `EVIOCGRAB` protege **nodos evdev**, no «toda la entrada del sistema».

---

## 4. Cobertura de dispositivos

**Lo que se sabe de logind** (Tarea 013, v262.2 `[V-p]`): `TakeDevice` solo admite `input/event*`, `drm/card*` y `hidraw*`, con el seat correcto. Los nodos `hidraw` **también** se pueden obtener y revocar por logind.

**Lo que se sabe de udev** (`sd-login` en latest `[V]`):
- La etiqueta `seat` hace al dispositivo elegible para un seat; `ID_SEAT` vale `seat0` por defecto.
- `uaccess` ata el acceso a la sesión activa.
- **Las reglas `71-seat`/`73-seat-late` de v262.2 no se pudieron leer** (404), y tampoco la lista de clases que reciben ACL. Que `hidraw` y `uinput` tengan ACL `uaccess`: `[VM]`.

**Relaciones:**

```text
seat (seat0) ⊃ dispositivo físico (udev, padre común) ⊃ N × input_dev ↔ /dev/input/eventK
eventK ⊃ M × evdev_client (uno por open) ↔ descripción de archivo ↔ k fds (dup/fork/SCM_RIGHTS)
grab: 1 por input_dev, propiedad de 1 evdev_client
```

### 4.1 Política de cobertura (propuesta)

| Estado | Criterio |
|---|---|
| `PROTECTED` | Nodo `event*` del seat obtenido con `TakeDevice` y con `EVIOCGRAB` = 0 |
| `UNPROTECTED` | Nodo `event*` del seat con `EBUSY`, `TakeDevice` fallido u otro error |
| `UNSUPPORTED` | Nodo relevante sin grab posible: el `hidraw` de un teclado o dispositivo de entrada (ver D-34) |
| `DISCONNECTED` | Eliminado |
| `PENDING` | Visto por udev o libinput y todavía sin grab |

Regla: **las credenciales solo se habilitan si no hay ningún dispositivo `UNPROTECTED` ni `PENDING`**. Lo que se haga con `UNSUPPORTED` lo decide el propietario (D-34).

**¿Se puede hacer sin carreras?** **No del todo.**
- La enumeración (udev) y el grab no son atómicos.
- Un nodo puede aparecer entre la enumeración y la habilitación.

Mitigación: el greeter se suscribe a los eventos de udev **antes** de enumerar, y cualquier `add` lo devuelve a `PENDING`, lo que deshabilita las credenciales. Lo que no se puede evitar es la ventana entre la creación del nodo y su procesamiento, en la que el campo puede estar todavía habilitado **un instante** `[H]` (RV-48).

### 4.2 Hotplug

1. Llega un `add` de udev con `ID_SEAT` del seat del greeter.
2. **Deshabilitar de inmediato** la entrada de credenciales y vaciar el campo si el usuario estaba escribiendo (decisión D-33: vaciar o conservar).
3. Clasificar: ¿es un `event*`?
4. `TakeDevice` y luego `EVIOCGRAB`.
5. Si va bien: `PROTECTED`; si no hay más pendientes, volver a habilitar.
6. Si falla: `UNPROTECTED` y la entrada queda deshabilitada mientras exista el dispositivo. Aviso: «Dispositivo de entrada no protegido: desconéctelo para continuar».
7. Un `remove` pasa el dispositivo a `DISCONNECTED`, que no bloquea.

**Hotplug continuo** (un dispositivo que se conecta y desconecta en bucle): la entrada queda deshabilitada mientras haya `PENDING`. Es un fallo cerrado; el coste es la disponibilidad.

---

## 5. Hotplug y carreras: resumen

| Carrera | Mitigación | Residuo |
|---|---|---|
| Un nodo nuevo antes del grab | Credenciales deshabilitadas en cuanto llega el `add` de udev | Ventana entre la creación en el kernel y el evento de udev en el greeter |
| Otro proceso abre el nodo nuevo | Necesita ACL `uaccess` del usuario activo (el del greeter) o ser root `[H]` | Si las ACL no se actualizaron (Tarea 013, §2.2) |
| Eventos encolados antes del grab | Irrelevantes para las credenciales si el campo estaba deshabilitado | Pulsaciones previas (no credenciales) legibles |
| Grab previo de un proceso antiguo | `EBUSY` → `UNPROTECTED` → bloqueo | — |

---

## 6. Evidencia DRM

Linux v7.3. `drm_framebuffer.c` se leyó completo; `drm_atomic_uapi.c` y `drm_atomic_helper.c` parcialmente, el resto como resumen. Tarea 013 para `drm_auth.c`.

| Tema | Hecho | Evidencia |
|---|---|---|
| Qué es master | El estado `dev->master` es de **un** `drm_file` (descripción de archivo); los ioctls de presentación exigen `drm_is_current_master` | Tarea 013 `[V-p]` |
| Descriptor con master | El fd que entrega `TakeDevice` (lo abrió logind; `was_master`) | `[V-p]` + `[H]` |
| Otro master presente | `SET_MASTER` → `EBUSY` → falla `TakeDevice` → el greeter no presenta | Tarea 013 |
| `DROP_MASTER` fallido | logind sigue adelante; el greeter verá `EBUSY` | Tarea 013 |
| Descriptores duplicados | Comparten el master (misma `drm_file`) `[H VFS]` | — |
| Buffers escaneados | Un commit del greeter con su propio FB sustituye al FB del plano; el FB antiguo deja de estar referenciado por el plano `[H]` | `drm_framebuffer.c` |
| Mapeos persistentes | Siguen escribiendo en el objeto GEM antiguo; **solo** afectan a la pantalla si algún plano todavía lo usa `[H]` (`drm_gem.c` no leído) | — |
| Cierre del fd antiguo | `drm_fb_release` → `drm_framebuffer_remove` desactiva los planos que usan ese FB (`atomic_remove_fb`); CRTC solo en el reintento | `[V-p]` |
| `CLOSEFB` | Desvincula el FB del fd **sin** sacarlo de la pantalla | `[V-p]` |
| `GETFB` | Solo devuelve un handle al master o a `CAP_SYS_ADMIN`: otro proceso no puede obtener el buffer en pantalla así | `[V-p]` |
| Commit atómico bloqueante | Con el helper por defecto: `commit_tail` → `hw_done` → `wait_for_vblanks` → devuelve. **Estado enganchado en un vblank** | `[V-p resumen]`; un driver con `atomic_commit_tail` propio puede diferir |
| Evento de flip | `DRM_EVENT_FLIP_COMPLETE` cuando se completa `flip_done` (vblank) | `[V-p]` |
| Verificar los píxeles | CRC por debugfs y conectores de writeback: **no leídos**. debugfs suele requerir root `[H]` | — |

### 6.1 Presentación: estados distintos

| Paso | Qué demuestra |
|---|---|
| Master obtenido | Nadie más puede cambiar la presentación por ioctls |
| Buffer creado | Nada sobre la pantalla |
| `ATOMIC` (bloqueante) = 0 | El nuevo estado está enganchado en un vblank (helper por defecto) |
| `FLIP_COMPLETE` recibido | Igual, de forma asíncrona |
| Presentación efectiva (píxeles) | **No demostrable** sin readback |
| Framebuffer anterior | Deja de mostrarse si el commit cubre **todos** los planos y CRTC activos (hay que desactivar explícitamente los planos que no use el greeter) `[H]` |

---

## 7. Protocolo de pantalla neutral (propuesta)

1. Master obtenido (`TakeDevice` DRM correcto).
2. Crear FB propios **negros u opacos** para el primario de cada CRTC conectado.
3. Commit atómico **bloqueante** con `ALLOW_MODESET`:
   - asigna los FB propios a cada primario;
   - **desactiva todos los planos de superposición y cursor** (no heredar ninguno);
   - incluye todas las CRTC y conectores.
4. Esperar el retorno correcto **y** el evento `FLIP_COMPLETE` de cada CRTC.
5. Solo entonces se considera `INITIAL_FRAME_COMMITTED`.

**Garantía:** hasta el nivel «enganchado por el hardware». Sin readback de píxeles.

**Fallo:** el greeter no pasa de estado. Si no puede presentar, no puede mostrar avisos; la sesión anterior queda con su último fotograma y sin entrada aceptada (RV-38).

---

## 8. Máquina de estados del greeter

| Estado | Actor | Operación | Verificable | Errores | Timeout | Recuperación | ¿Acepta credenciales? |
|---|---|---|---|---|---|---|---|
| `INITIALIZING` | Compositor del greeter | Arranque | — | — | — | — | No |
| `LOGIND_CONTROLLER_ACQUIRED` | Ídem | `TakeControl(false)` | Respuesta OK | `ACCESS_DENIED`, ya hay controlador (RV-39) | T1 | Reintentar con backoff → `REFUSED` | No |
| `DRM_DEVICE_ACQUIRED` | Ídem | `TakeDevice(card)` | fd con `inactive=false` | `EBUSY` (otro master), `ENODEV` | T2 | → `RECOVERY` | No |
| `DRM_MASTER_ACQUIRED` | Ídem | Comprobación explícita del master (`drmIsMaster` o equivalente) | Verdadero | — | — | → `RECOVERY` | No |
| `INITIAL_BUFFER_READY` | Ídem | Crear FB | OK | `ENOMEM` | — | `RECOVERY` | No |
| `INITIAL_FRAME_COMMITTED` | Ídem | §7 | Retorno + `FLIP_COMPLETE` por CRTC | `EINVAL`, `EBUSY` | T3 | `RECOVERY` | No |
| `INPUT_ENUMERATION_STARTED` | Ídem | Suscribirse a udev **y luego** enumerar | Suscripción activa | Fallo de udev | — | `SECURITY_DEGRADED` | No |
| `INPUT_DEVICES_ENUMERATED` | Ídem | Lista de `event*` del seat | Lista | Timeout | T4 | Reintentar | No |
| `INPUT_GRABS_ACQUIRED` | Ídem | `TakeDevice` + `EVIOCGRAB` de cada uno | 0 en todos | `EBUSY`, `ENODEV`, `EINTR` | T5 | Por dispositivo: `UNPROTECTED` | No |
| `INPUT_COVERAGE_VERIFIED` | Ídem | Sin `UNPROTECTED` ni `PENDING`; política `UNSUPPORTED` aplicada | Predicado | — | — | — | No |
| `CREDENTIAL_INPUT_ENABLED` | Greeter (cliente) | Habilitar el campo | Mensaje del compositor al cliente | — | — | — | **Sí** |
| `SECURITY_DEGRADED` | Compositor | Cobertura perdida (hotplug, `ENODEV` del propio grab, `PauseDevice`) | — | — | — | Deshabilitar y volver a `INPUT_ENUMERATION_STARTED` | No |
| `RECOVERY` | Compositor | Reintento completo con backoff (D-32) | — | — | Tope N | → `REFUSED` | No |
| `REFUSED` | Compositor | Estado final; aviso fijo si puede presentar | — | — | — | Intervención externa (reinicio de greetd o del equipo) | No |

T1–T5 son timeouts por definir (D-32).

Pérdida de la conexión con logind, del fd DRM o un `PauseDevice` → `SECURITY_DEGRADED` o `RECOVERY`. Si falla el compositor del greeter, greetd lo relanza `[V-p]` y el estado empieza de cero.

---

## 9. Regla de habilitación de credenciales

**El greeter acepta credenciales solo si se cumplen todas estas condiciones a la vez, y solo mientras se sigan cumpliendo:**

1. Es controlador de su sesión logind y su sesión es la activa del seat.
2. Tiene DRM master en **todos** los dispositivos `card*` del seat que presentan.
3. Se completó `INITIAL_FRAME_COMMITTED` en todas las CRTC activas.
4. Todos los nodos `event*` del seat están en `PROTECTED`; no hay ninguno `PENDING` ni `UNPROTECTED`.
5. Se aplica la política aprobada para `UNSUPPORTED` (D-34).
6. No ha llegado ningún `PauseDevice` ni `SessionRemoved`, ni evento de udev, posterior al último cálculo de cobertura.

**Es viable**: todas las condiciones son predicados locales del compositor del greeter. **No es atómica** frente al hotplug (§4.1). Un estado parcial **nunca** habilita.

---

## 10. Modelo de amenazas

| # | Amenaza | Capacidad | Condición | Mitigación | Limitación | Evidencia | Residuo |
|---|---|---|---|---|---|---|---|
| 1 | Proceso anterior del mismo UID | Leer evdev; tener master | fds previos | Revoke (mejor esfuerzo) + **grab** + comprobación de master | Estado de teclas por `EVIOCGKEY` (RV-47) | `[V-p]` | Bajo |
| 2 | Proceso anterior de **otro** UID | Igual | fds previos | Igual | Igual | `[V-p]` | Bajo (cambia respecto a RV-43) |
| 3 | fd evdev directo | Leer eventos | open previo | Grab | Búfer previo; `EVIOCGKEY` | `[V-p]` | Bajo |
| 4 | fd compartido | Igual que 3 | — | Grab | — | `[V-p]` | Bajo |
| 5 | **hidraw** | Leer informes HID crudos del teclado | ACL o root sobre `/dev/hidraw*` | Revoke por logind **solo** si `hidraw` pasó por `TakeDevice` de la sesión anterior | El grab no lo cubre; fds directos sin revocar | `[V-p]` | **Medio** → D-34 |
| 6 | **uinput** | Inyectar entrada falsa en el greeter | Acceso a `/dev/uinput` | El dispositivo virtual queda en grab, pero su creador sigue inyectando | No puede **leer** credenciales; puede hacer clic o escribir | `[V-p]` | Suplantación, no fuga |
| 7 | Mapeo de FB | Escribir en un buffer antiguo | mmap previo | Primer commit con FB propios y planos desactivados | Antes del commit: RV-38 | `[V-p]` + `[H]` | Bajo tras el commit |
| 8 | fd DRM directo | Master (RV-44) | `was_master` | El greeter recibe `EBUSY` y no presenta | Disponibilidad | `[V-p]` | Disponibilidad |
| 9 | Dispositivo añadido durante la autenticación | Recibir eventos antes del grab | Proceso con ACL | `PENDING` deshabilita la entrada | Ventana udev (RV-48) | `[H]` | Bajo |
| 10 | Entrada sin evdev | Leer o inyectar | Serie, Bluetooth crudo, `hidraw` | Ninguna en el greeter | — | — | Fuera del modelo salvo D-34 |

**No hay aislamiento entre sesiones.** Lo que se protege es la **recepción de eventos evdev nuevos** durante la entrada de credenciales y el **control DRM de la presentación**.

---

## 11. Evaluación de M-GRAB: `VIABLE CON LIMITACIONES EXPLÍCITAS`

| Criterio | Resultado |
|---|---|
| Protege a los clientes evdev existentes | **Sí**, para eventos nuevos `[V-p]` |
| Protege fds directos | **Sí** (el grab es por dispositivo) `[V-p]` |
| Cubre nodos duplicados de un mismo dispositivo | Solo si se hace grab de cada `input_dev` |
| Cubre todos los dispositivos del seat | Solo los evdev; **no** hidraw ni fuentes sin evdev |
| Resuelve las carreras de hotplug | **No del todo**; queda la ventana de §4.1 |
| Detecta fallos | **Sí**: `EBUSY`/`ENODEV` |
| Requiere privilegios adicionales | **No**: el fd viene de `TakeDevice` |
| Obliga a reducir el modelo de amenazas | **Sí**: hidraw, `uinput` (suplantación), estado de teclas, ventana de hotplug |

**Garantía formulable:** «Con las credenciales habilitadas, todos los nodos evdev del seat están en grab exclusivo del greeter: ningún otro cliente o handler de esos nodos recibe los eventos nuevos, y el greeter es el único master DRM con un estado de presentación propio enganchado en el hardware».

---

## 12. Alternativas

| Alternativa | Resuelve | No resuelve | Privilegio | Coste | Accesibilidad/IME | Verificación |
|---|---|---|---|---|---|---|
| **M-GRAB + regla §9** (base) | Amenazas 1–4, 7–9 | 5, 6, 10 | Ninguno | Bajo | Sin impacto: el compositor sigue entregando la entrada a sus clientes (IME, AT-SPI) | T-GRAB, T-GREETER-SECURITY |
| Grab también de `hidraw` | — | No existe grab en hidraw `[V-p]` | — | — | — | — |
| Política udev: quitar `uaccess` a `hidraw` de dispositivos de clase teclado | Amenaza 5 frente a usuarios no root | Root; dispositivos que necesitan hidraw (FIDO, braille) | Configuración del sistema (`foundation/udev`) | Bajo | Puede romper FIDO y braille por hidraw | T-INPUT-COVERAGE |
| Bloquear credenciales si existe un `hidraw` de teclado del seat (`UNSUPPORTED` bloqueante) | 5 | — | Ninguno | Bajo | **Excluye** teclados con hidraw, que son casi todos los USB `[H]` → **inaceptable** como regla por defecto | — |
| Revocar `hidraw` con logind (`TakeDevice` del greeter) | Solo los fds de logind | fds directos | — | — | — | — |
| Servicio intermediario de entrada | — | Duplica logind/libseat; **rechazado** (AGENTS.md regla 7) | Root | Alto | — | — |
| Seats separados | Otros seats | El mismo seat | — | — | — | — |

---

## 13. Impacto en accesibilidad e internacionalización

| Elemento | Impacto |
|---|---|
| Teclados físicos, distribuciones, IME | Ninguno: el grab lo tiene el compositor del greeter, que sigue procesando con libinput y xkb y entregando la entrada a los clientes Wayland (IME vía protocolos) `[H]` |
| Lector de pantalla | Corre dentro de la sesión del greeter; recibe la entrada a través del compositor `[H]` |
| Teclado virtual en pantalla | Es un cliente Wayland: sin impacto |
| **brltty / braille** | Lee la línea braille por USB, serie o hidraw (**no** se ve afectado por el grab) e **inyecta** por `uinput`. El dispositivo virtual quedaría en grab del greeter y su entrada se aceptaría. **No** se deben rechazar los dispositivos `uinput` por defecto |
| Dispositivos de asistencia evdev sin grab posible (`EBUSY` por un demonio propio que ya lo tiene) | Quedarían `UNPROTECTED` y bloquearían la entrada. **Conflicto:** hace falta una lista de demonios de asistencia de confianza o aceptar el dispositivo explícitamente (D-35 [P]). La seguridad no debe excluirlos por defecto |
| Hotplug legítimo | Bloqueo breve mientras está `PENDING`; sin exclusión |

---

## 14. Pruebas pendientes (no autorizadas, no ejecutadas)

| ID | Objetivo | Preparación | Procedimiento | Resultado esperado | Evidencia | Aprobación | No cubre |
|---|---|---|---|---|---|---|---|
| **T-GRAB** | Semántica de grab en el kernel de Arch | VM con el kernel de Arch; dispositivo evdev virtual (`uinput`) y físico (USB passthrough) | Dos clientes; grab; segundo grab; `dup`, `fork`, `SCM_RIGHTS`; revocación; desconexión; hotplug; grab previo de otro proceso; búfer previo; `EVIOCGKEY` durante el grab | Solo el dueño recibe; `EBUSY` en el segundo; el grab se libera al cerrar o revocar; el búfer previo es legible; `EVIOCGKEY` revela el estado | Trazas `evtest`, strace, `uname -r` | Todo coincide con §2 | hidraw |
| **T-INPUT-COVERAGE** | Enumeración del seat | VM con varios dispositivos (teclado con varios nodos, touchpad, pantalla táctil, `uinput`, hidraw) | Enumerar por udev; comparar con `/dev/input`; hotplug y desconexión durante la autenticación; leer las reglas de seat y `uaccess` instaladas | Todos los `event*` del seat se clasifican; hotplug → `PENDING`; ACL de `hidraw`/`uinput` documentadas | `udevadm info`, `getfacl` | Sin nodos no clasificados | Fuentes sin evdev |
| **T-DRM** (ampliada) | Master y pantalla neutral | VM con KMS (virtio-gpu) y hardware real (`[HW]`) | Master ocupado; `DROP_MASTER` fallido simulado; fd directo; fd duplicado; recuperación; mapeo de un FB antiguo y escritura tras el commit del greeter; commit fallido; `FLIP_COMPLETE`; CRC o writeback si existe | `EBUSY` detectado; escritura en el mapeo antiguo **sin** efecto en pantalla tras el commit; commit enganchado | Capturas, debugfs CRC (root, solo en la prueba) | Coincide | Drivers con `commit_tail` propio |
| **T-GREETER-SECURITY** | Regla §9 | Greeter de prueba instrumentado | Hacer fallar cada condición por separado; timeouts; hotplug durante la autenticación; pérdida de logind; reintento; estado `REFUSED` | El campo nunca se habilita con una condición falsa | Log de estados con marcas de tiempo | Cero habilitaciones indebidas | Errores de implementación no ejercitados |
| **T-ACCESSIBILITY** | Sin regresiones | VM con IME (fcitx5/ibus), varias distribuciones, Orca, teclado en pantalla, brltty con `uinput` | Autenticarse con cada uno; hotplug de un dispositivo de asistencia | Todos funcionan; brltty no queda bloqueado | Grabación | Sin exclusiones | Dispositivos no probados |

---

## 15. Trazabilidad propuesta (no aplicada)

| Destino | Cambio |
|---|---|
| RV-37 | «Entrada: M-GRAB **viable con limitaciones** (Tarea 014); presentación: master + commit enganchado; pendiente de T-GRAB, T-DRM y T-GREETER-SECURITY; límites: hidraw, `uinput`, ventana de hotplug, estado de teclas» |
| RV-43 | Mitigado para evdev por M-GRAB; **abierto para hidraw** |
| Nuevos (justificados por evidencia nueva) | **RV-46** kernel de Arch desconocido. **RV-47** `EVIOCGKEY`/estado del dispositivo visible durante el grab (`input_get_disposition`). **RV-48** ventana de hotplug y tecleo prematuro. **RV-49** `hidraw` sin grab (`hidraw_report_event`) |
| `spikes.md` | Actualizar T-GRAB y T-DRM; añadir T-INPUT-COVERAGE, T-GREETER-SECURITY y T-ACCESSIBILITY |
| `lockscreen-greeter.md` (tras la decisión) | Regla §9, máquina de estados §8, política de cobertura §4.1 y protocolo §7 como requisitos del greeter |
| Decisiones nuevas | **D-32** [P]: recuperación y timeouts del greeter (T1–T5, N reintentos, aviso). **D-33** [P]: vaciar o conservar el campo ante hotplug. **D-34** [P]: tratamiento de `hidraw` (aceptar el riesgo / restringir `uaccess` / bloquear). **D-35** [P]: dispositivos de asistencia con grab ajeno |

---

## 16. Veredicto final: `VIABLE CON LIMITACIONES EXPLÍCITAS`

| # | Pregunta | Respuesta |
|---|---|---|
| 1 | ¿M-GRAB protege a los clientes evdev anteriores? | **Sí, para eventos nuevos** (`input_pass_values` y `evdev_events` en v7.3 `[V-p]`). No borra lo ya encolado ni oculta el estado de las teclas (RV-47) |
| 2 | ¿Cubre fds directos y compartidos? | **Sí**: el grab es del `input_dev` y excluye a cualquier otro cliente o handler |
| 3 | ¿Se puede garantizar la cobertura de todos los dispositivos relevantes? | **Solo de los nodos evdev del seat**, y no de forma atómica. **No** cubre hidraw ni fuentes sin evdev |
| 4 | ¿Y el hotplug? | `PENDING` deshabilita la entrada. Queda una ventana corta sin garantía atómica (RV-48) |
| 5 | ¿Se puede demostrar la pantalla neutral? | Hasta «estado propio enganchado en un vblank». **Los píxeles no** sin readback |
| 6 | ¿Se puede demostrar que no se aceptan credenciales antes de completar la protección? | **En diseño, sí:** la regla §9 son predicados locales. Hay que demostrarlo con T-GREETER-SECURITY |
| 7 | ¿Qué amenazas quedan fuera? | hidraw, inyección por `uinput`, estado de teclas, fuentes sin evdev, último fotograma antes del commit, ventana de hotplug, limpieza de procesos, aislamiento entre sesiones |
| 8 | ¿Qué pruebas son imprescindibles? | T-GRAB, T-DRM y T-GREETER-SECURITY. T-INPUT-COVERAGE para cerrar D-34 |
| 9 | ¿Sigue siendo recomendable E? | **Sí, con M-GRAB y la regla §9** como requisitos del greeter |
| 10 | ¿Se puede cerrar RV-37? | **No. Sigue abierto**, pendiente de las pruebas y de las decisiones D-32 a D-35 |

No se modificó el repositorio. Ningún ADR cambió de estado (todos `Propuesto`). La Fase 0 sigue `IN_PROGRESS` y la Fase 1 no está autorizada.
