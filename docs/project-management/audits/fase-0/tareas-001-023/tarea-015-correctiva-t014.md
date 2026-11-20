# Tarea 015 — Revisión correctiva de la Tarea 014

- **Fecha:** 2026-11-19
- **Fase 0:** `IN_PROGRESS`
- **ADR:** todos `Propuesto`
- **Fase 1:** sin autorización
- **Naturaleza:** investigación de solo lectura. Es una propuesta, no una decisión. El informe está en el scratchpad y el repositorio no se ha modificado.
- **Marcas:** `[V]` fuente suficiente · `[V-p]` parcial · `[R]` repositorio · `[H]` inferencia · `[VM]` requiere prueba · `[P]` propietario.

---

## 1. Resumen ejecutivo

**Veredicto: `MITIGACIÓN PARCIAL CON LIMITACIONES EXPLÍCITAS`** (§14).

**Corrección principal a la Tarea 014.** Lo que M-GRAB demuestra en código es **exclusividad de distribución evdev**. No demuestra **confidencialidad de credenciales**. La confidencialidad depende además de tres cosas:

- el control de acceso a hidraw (ACL/udev, no el kernel);
- la ventana de hotplug (permisos del nodo, no el grab);
- que el greeter no habilite la entrada antes de tiempo.

En la documentación, la garantía debe quedar redactada **solo** como exclusividad evdev, y no como confidencialidad general.

| Problema | Resultado |
|---|---|
| P-01 Hotplug | `HOTPLUG_REDUCIDO`. No existe una barrera atómica en Linux. La ventana se acota con permisos, no con el grab. RV-48 sigue abierto |
| P-02 hidraw | Confidencialidad **no garantizable por el kernel**. Depende de las ACL/udev de hidraw, que no se han leído para Arch. Hay que elegir en D-34 |
| P-03 uinput | **Corrección:** los eventos de uinput entran con `input_event()` y **llegan al dueño del grab**, es decir, al greeter. El grab **no** filtra la inyección del dispositivo propio. Integridad no garantizada |
| P-04 DRM | **Corrección:** un commit atómico **no** cambia los planos ni los CRTC que no incluye. El greeter debe incluir explícitamente **todos** los planos y CRTC de **todos** los dispositivos `card*`. Nivel exigible en v1: `DRM_STATE_COMMITTED` + verificación por `GETPLANE`/`GETCRTC`. `PIXELS_VERIFIED` no es exigible |
| P-05 Sesión anterior | El reinicio del greeter **no** demuestra el fin de la sesión anterior. Se separan los estados. El ciclo completo no está demostrado y RV-37 sigue abierto |

---

## 2. Método y límites de la evidencia

- **Kernel:** Linux v7.3 upstream, leído con una herramienta web que procesa el texto, por lo que todo es `[V-p]`. Algunas partes solo están resumidas (`[V-p resumen]`). **La versión del kernel de Arch es desconocida** (RV-46).
- **systemd:** v262.2 upstream `[V-p]`. Faltan las reglas udev `70-uaccess`/`71-seat`/`73-seat-late` (404) y los parches de Arch (Anubis).
- **No leído:**
  - `drm_gem.c`
  - `drm_plane.c`
  - el cuerpo completo de `drm_atomic_helper.c`
  - drivers concretos
  - `hid_input_report` literal
  - `mousedev.c`
  - `sysrq.c`
  - libinput (Anubis)
- Esta tarea no añade lecturas nuevas: reanaliza las evidencias de las Tareas 013 y 014 y corrige sus conclusiones. Donde hace falta evidencia que no se ha leído, se marca `[H]` o `[VM]`.

---

## 3. P-01 — Hotplug

### 3.1 Secuencia real

| # | Evento | Quién lo produce | Acceso al nodo en ese momento |
|---|---|---|---|
| 1 | `input_register_device` + evdev `connect` | Kernel | — |
| 2 | Se crea `/dev/input/eventN` en devtmpfs | Kernel | Permisos por defecto de devtmpfs (root) `[H]` |
| 3 | uevent `add` al netlink del kernel | Kernel | — |
| 4 | udevd procesa las reglas: grupo `input`, `0660`, ACL `uaccess` para el usuario de la sesión activa | udevd | Desde aquí: root, grupo `input` y usuario activo `[H; reglas no leídas]` |
| 5 | udevd reenvía el evento a los monitores libudev | udevd | — |
| 6 | El greeter recibe el evento y marca el dispositivo `PENDING` | Greeter | — |
| 7 | `TakeDevice` + `EVIOCGRAB` | Greeter + logind | Exclusividad evdev desde aquí |

**Entre 1 y 7 no hay ninguna operación atómica.**
- Suscribirse antes de enumerar solo evita **perder** dispositivos; no cierra la ventana.
- Los monitores de libudev pueden perder eventos si se llena el búfer del socket `[H]`.
- Los nombres `eventN` se reutilizan, así que hay que identificar el dispositivo por `devnum` + `syspath`, no por el nombre `[H]`.

### 3.2 Alternativas

| Alternativa | Ventana que reduce | Ventana que queda | Quién puede abrir el nodo | Privilegio | Asistencia | Evidencia necesaria |
|---|---|---|---|---|---|---|
| Suscripción previa + enumeración | Pérdida de dispositivos | 1→7 | Root, `input`, usuario activo | Ninguno | Sin impacto | T-HOTPLUG-RACE |
| Reconciliación periódica (reenumerar cada T) | Monitores que pierden eventos | 1→7 + T | Igual | Ninguno | Sin impacto | Igual |
| **Permisos:** ningún usuario en el grupo `input`; `uaccess` solo para el usuario activo, que es el greeter | Restringe **quién** puede abrir en la ventana: root y el propio usuario greeter | La duración no cambia | Root, usuario greeter | Configuración del sistema `[P]` | Los demonios de asistencia root no se ven afectados | Lectura de las reglas de Arch + T-INPUT-COVERAGE |
| Gestión centralizada por logind/libseat | Ya está en uso | Igual | — | — | — | — |
| Servicio intermediario exclusivo | Igual: tampoco ve el nodo antes del paso 2 | Igual | — | Root | — | **Rechazado:** no cierra la carrera y duplica logind |
| Separación de seat | Dispositivos de otros seats | Del propio seat | — | — | — | — |
| No autenticar hasta estado estable (sin `PENDING` durante T_estable) | Tecleo prematuro | 1→6 si no llega ningún evento | — | — | Retraso | T-GREETER-SECURITY |

### 3.3 Clasificación: `HOTPLUG_REDUCIDO`

**Garantía limitada:**

> «Un nodo evdev creado durante la autenticación es accesible, hasta que el greeter lo captura, solo para root y para los procesos que tengan acceso al nodo según la política udev del sistema. Con la política propuesta (ningún usuario en el grupo `input`), eso significa root y el usuario del greeter. Las credenciales se deshabilitan cuando el greeter procesa el evento `add`. Lo tecleado entre la creación del nodo y ese momento no está protegido por el grab.»

Matiz importante: mientras la sesión del greeter es la activa, los **procesos de la sesión anterior** no tienen ACL sobre los nodos nuevos `[H]`. El riesgo real en la ventana se reduce a **root** y al **grupo `input`**.

**RV-48 sigue abierto**, reformulado como «carrera de hotplug acotada por permisos».

---

## 4. P-02 — hidraw

### 4.1 Hechos

| Hecho | Evidencia |
|---|---|
| `hidraw_open` no aplica exclusividad ni grab | v7.3 `hidraw.c` `[V-p]` |
| `hidraw_report_event` copia el informe a **todos** los lectores no revocados | `[V-p]` |
| La ruta HID → hidraw es independiente del dispatch de `input_dev` | `hid-core.c` `[V-p resumen]` |
| `HIDIOCREVOKE` actúa por `hidraw_list` (una por cada `open()`) | `[V-p]` |
| logind admite `hidraw` en `TakeDevice` y lo revoca al desactivar la sesión, **solo** sobre su propio fd | systemd v262.2 `[V-p]` |
| Qué nodos hidraw reciben `uaccess` en Arch | **No leído** |

**Conclusión:** la confidencialidad frente a hidraw **depende de quién puede abrir `/dev/hidraw*`**. Eso lo fijan udev y las ACL, no el kernel ni el greeter. Por eso no se atribuye ninguna garantía al kernel.

### 4.2 Matriz para D-34

| | A. Riesgo residual explícito | B. Restringir acceso a hidraw | C. Revocación por logind | D. Política por clase |
|---|---|---|---|---|
| Amenaza cubierta | Ninguna: solo se documenta | Usuarios no root sin ACL sobre el hidraw de teclados | Solo los fds obtenidos con `TakeDevice` de la sesión anterior | Teclados y punteros: B. FIDO, braille, etc.: excepción |
| Residual | Lectura por cualquiera con acceso al nodo | Root; reglas mal clasificadas | fds abiertos directamente | Clasificación incorrecta |
| Privilegios | — | Reglas udev del sistema (`foundation/udev`) `[P]` | Ninguno | Reglas udev |
| FIDO/U2F | Sin impacto | **Se rompe** si se restringe de forma indiscriminada | Sin impacto | Se mantiene (`ID_SECURITY_TOKEN`) `[H]` |
| Braille | Sin impacto | brltty corre como root: sin impacto `[H]` | — | Excepción |
| Teclados especializados (configuración por hidraw: QMK/VIA) | Sin impacto | La herramienta de configuración pierde acceso sin root | — | Excepción configurable |
| Accesibilidad | — | Riesgo bajo si las excepciones están bien | — | — |
| Compatibilidad con logind | Total | Total | Total | Total |
| Complejidad | Nula | Baja–media | Ya existe | Media |
| Pruebas | — | T-HIDRAW | T-HIDRAW | T-HIDRAW, T-ACCESSIBILITY |

**Recomendación técnica (no decisión): D + A.**
- No dar `uaccess` sobre hidraw a dispositivos cuya única función es teclado o puntero, con excepciones explícitas por clase.
- Documentar el riesgo residual que queda: root y dispositivos exceptuados.

**Requisito previo:** leer las reglas udev instaladas en Arch. Si ya **no** dan `uaccess` a hidraw de teclados, D coincide con el comportamiento por defecto y basta con documentarlo `[VM]`.

**Redacción de la garantía:** la confidencialidad que ofrece el greeter queda **limitada expresamente a los eventos evdev**. Solo se amplía a hidraw con la política D/B aprobada y verificada.

---

## 5. P-03 — uinput

### 5.1 Ruta real (v7.3 `[V-p]`)

```text
proceso con fd de /dev/uinput
  └─ write(ev) → uinput_inject_events() → input_event(udev->dev, …)        [uinput.c]
       └─ input_handle_event(dev, …) → input_pass_values(dev, …)           [input.c]
            └─ handle = dev->grab ? → SOLO handle (dueño del grab) : todos los handles abiertos
```

`input_inject_event()` (la que filtra `grab == handle`) se usa en las escrituras **a través de un handle**, por ejemplo `write()` sobre un fd evdev. **No** es la ruta de uinput.

### 5.2 Respuestas

| # | Pregunta | Respuesta |
|---|---|---|
| 1 | ¿Se puede inyectar en un `input_dev` cuyo handler evdev está en grab? | **Sí**, si es el dispositivo **creado por ese uinput**. `input_event` del driver no se filtra por grab `[V-p]` |
| 2 | ¿A qué handlers llega? | Solo al dueño del grab; sin grab, a todos |
| 3 | ¿El grab impide que llegue al dueño? | **No: llega precisamente al greeter** |
| 4 | ¿Puede un dispositivo uinput inyectar en otro dispositivo? | **No** por esta ruta: cada uinput emite en su propio `input_dev` `[V-p]` |
| 5 | ¿Puede aparecer como teclado del seat? | Sí: `ID_SEAT` por defecto es `seat0` `[V sd-login]`; con capacidades de teclado lo clasifican como teclado `[H]` |
| 6 | ¿Qué pasa con los eventos sintéticos tras el grab? | Llegan al greeter como entrada válida |
| 7 | ¿Se pueden crear dispositivos durante la autenticación? | Sí, si se tiene acceso a `/dev/uinput`. Pasa por hotplug (P-01) |
| 8 | Controles existentes | Permisos de `/dev/uinput` (no leídos para Arch; `[H]` root por defecto) |

### 5.3 Clasificación

| Dimensión | Efecto de uinput |
|---|---|
| Confidencialidad | **Ninguno.** No lee la entrada de otros dispositivos |
| Integridad | **Afectada.** Puede escribir o enviar en el greeter |
| Disponibilidad | Puede saturar o forzar hotplug continuo (`PENDING`) |
| Suplantación | Puede imitar a un usuario; por sí sola **no** autentica sin conocer la contraseña |
| Control de dispositivos virtuales | Solo por los permisos de `/dev/uinput` |

**Riesgos:** RV-47 trata el estado de teclas observable y debe seguir siendo eso. Se propone un riesgo nuevo, **RV-50, entrada sintética por uinput**, porque es una dimensión distinta (integridad).

No se rechaza uinput por defecto: brltty lo usa. Mitigación posible: marcar los dispositivos virtuales en la interfaz del greeter y aplicar la política D-35.

---

## 6. P-04 — DRM y mappings

### 6.1 Ciclo de vida

| Objeto | Comportamiento | Evidencia |
|---|---|---|
| fd DRM del compositor anterior | Sigue abierto; sin master tras el drop | Tarea 013 `[V-p]` |
| Su FB | Referenciado por los planos mientras estos lo usen; al cerrar el fd, `drm_fb_release` lo quita de la presentación | `drm_framebuffer.c` `[V-p]` |
| Objeto GEM | Vive mientras haya referencias: FB, handles y mmaps `[H]` | `drm_gem.c` no leído |
| Mapping persistente | Escribe en la memoria del GEM | `[H]` |
| Plano que usa el FB | Mantiene su estado hasta que un commit lo cambie | Semántica atómica `[H fuerte]` |
| Commit del greeter | **Solo cambia** los objetos incluidos en el estado | `[H fuerte]`; los helpers parten del estado actual |
| Liberación | El FB anterior queda libre cuando no lo referencia ningún plano ni handle | `[H]` |

### 6.2 Respuestas

| # | Pregunta | Respuesta |
|---|---|---|
| 1 | ¿Qué pasa si el anterior conserva un mapping? | Puede escribir en el GEM |
| 2 | ¿Puede escribir en un buffer asociado a un plano activo? | **Sí, mientras ese plano lo escanee: es un cambio visible** |
| 3 | ¿Qué cambia al sustituir el FB? | El plano apunta al FB del greeter; el GEM antiguo deja de escanearse en **ese** plano |
| 4 | ¿Se liberan solas todas las referencias de los planos? | **No:** solo las de los planos incluidos en el commit |
| 5 | ¿Overlay no desactivado? | **Sigue mostrando el FB anterior, y el mapping puede alterarlo** |
| 6 | ¿CRTC no cubierto? | Mantiene su estado anterior |
| 7 | ¿Cursores hardware? | Son planos (`DRM_PLANE_TYPE_CURSOR`) con el capability universal planes: hay que desactivarlos igual `[H]` |
| 8 | ¿Commit bloqueante? | Con el helper por defecto: estado aplicado y enganchado en un vblank `[V-p resumen]` |
| 9 | ¿`FLIP_COMPLETE`? | Que el flip del CRTC se enganchó en un vblank `[V-p]` |
| 10 | ¿Qué depende del driver? | `atomic_commit_tail` propio, planos ocultos, compresión, PSR (panel self refresh) `[H]` |
| 11 | ¿Qué se verifica sin píxeles? | `GETPLANE` (`fb_id`, `crtc_id`) y `GETCRTC` de **todos** los objetos: cada uno debe tener un FB del greeter o 0 `[H; ioctls sin master, Tarea 013]` |
| 12 | ¿Qué necesita hardware real? | PSR, drivers concretos, multi-GPU, `PIXELS_VERIFIED` |

### 6.3 Protocolo corregido

1. Para **cada** `card*` del seat: `TakeDevice` → master.
2. `DRM_CLIENT_CAP_UNIVERSAL_PLANES` + `ATOMIC`. Enumerar **todos** los planos, CRTC y conectores.
3. **Un commit global por dispositivo** (en atómico no se pueden mezclar dispositivos) que incluya:
   - cada CRTC conectado: activo, con el FB del greeter en su primario;
   - cada CRTC sin uso: `ACTIVE=0`;
   - **cada plano** que no se use (overlay, cursor, primarios sobrantes): `FB_ID=0`, `CRTC_ID=0`.
4. Commit bloqueante, **y además** esperar `FLIP_COMPLETE` de cada CRTC activo.
5. **Verificar** con `GETPLANE`/`GETCRTC` que ningún objeto referencia un FB que no sea del greeter.
6. Hotplug de conectores o de un `card*` nuevo → vuelve a `PENDING` en presentación.

**Niveles:**

| Nivel | ¿Exigible en v1? |
|---|---|
| `DRM_MASTER` | **Sí** |
| `DRM_STATE_COMMITTED` | **Sí** (commit correcto + `FLIP_COMPLETE`) |
| `SCANOUT_CONFIRMED` (paso 5) | **Sí** en el sentido de estado consultado. No confirma lo que hace el hardware internamente (PSR) |
| `PIXELS_VERIFIED` | **No** en v1 (requiere CRC/writeback, normalmente con root o debugfs) |

**RV-38** se actualiza así:
- Tras el paso 5, ningún plano escanea un buffer anterior según el estado del kernel.
- Antes del paso 5, el último fotograma está expuesto y puede modificarse a través de un mapping.
- **Riesgo nuevo propuesto, RV-51:** cuando un plano no se incluye en el commit, un mapping antiguo puede alterar lo que se presenta. Existe una vía explotable si el protocolo no cubre todos los planos.

---

## 7. P-05 — Recuperación y sesión anterior

### 7.1 Estados separados

| Estado | Condición observable | Quién la observa |
|---|---|---|
| `GREETER_PROCESS_RUNNING` | El proceso existe | greetd |
| `GREETER_PRESENTATION_READY` | §6.3 completo | Compositor del greeter |
| `GREETER_INPUT_READY` | Todos los `event*` en grab (T014 §9) | Compositor del greeter |
| `PREVIOUS_SESSION_STOPPING` | Otra sesión del seat en `closing` (`ListSessions`/`State`) | Greeter (lectura de login1, sin privilegio) `[V]` |
| `PREVIOUS_SESSION_REMOVED` | `SessionRemoved` de esa sesión, o ausencia en `ListSessions` | Greeter |
| `PREVIOUS_SESSION_PROCESSES_RECONCILED` | Unidades PARALLAX del usuario anterior `inactive` | **Solo** el comando de sesión del siguiente inicio de **ese** usuario (L3), o el fin de `user@` |
| `GREETER_AUTHENTICATION_READY` | `PRESENTATION_READY` ∧ `INPUT_READY` ∧ la sesión del greeter es `ActiveSession` del seat ∧ ninguna otra sesión del seat está `active` | Greeter |

`PREVIOUS_SESSION_REMOVED` y `PROCESSES_RECONCILED` **no** forman parte de la condición de autenticación:
- El greeter no puede esperarlos sin bloquearse indefinidamente.
- Las garantías de presentación y entrada no dependen de ellos.
- Quedan como garantías de **finalización de sesión**, separadas.

### 7.2 Secuencias de fallo

| # | Fallo | Qué ocurre | Garantía |
|---|---|---|---|
| 1 | Muere el compositor del greeter | greetd ve terminar el greeter: sin sesión programada → error («greeter exited without creating a session») `[V-p context.rs]`; el reinicio depende de la unidad greetd `[VM]` | Ninguna sobre la sesión anterior |
| 2 | Muere greetd | No hay teardown PAM (`terminate` sin espera) `[V-p]`; el worker por `PDEATHSIG` `[H]`; scope vacío → logind finaliza **asíncronamente** | Ninguna inmediata |
| 3 | Muere el comando de sesión | Tarea 012/013: PAM cerrado → greeter; la sesión se cierra de forma asíncrona (hasta 20 s + vaciado del scope) | Presentación y entrada: §6.3/M-GRAB |
| 4 | Muere el compositor PARALLAX | `BindsTo` (diseño) → target parado; el comando sale | `[VM]` |
| 5 | Termina la sesión PAM | `session_release` → temporizador de 20 s → `session_stop` → finalize `[V-p]` | Asíncrona |
| 6 | logind tarda | La sesión sigue en `closing`; sus dispositivos ya se pausaron cuando el greeter pasó a activo (mejor esfuerzo) | — |
| 7 | El greeter arranca con procesos antiguos vivos | Permitido por diseño; mitigado por master + grab | Sin presentación ni recepción evdev de la sesión antigua |
| 8 | Otra sesión del mismo usuario | `user@` sigue activo; los procesos sobreviven | Ninguna |
| 9 | Linger | `user@` nunca se detiene | Ninguna (D-31) |
| 10 | Procesos fuera del cgroup gestionado | Sobreviven hasta `KillUserProcesses` o el fin de `user@` | Ninguna |

### 7.3 Respuestas

| # | Pregunta | Respuesta |
|---|---|---|
| 1 | ¿Quién termina la sesión anterior? | logind (scope) y systemd (unidades); PARALLAX solo con L1 y L3 |
| 2 | ¿Qué evidencia demuestra que terminó la sesión logind? | `SessionRemoved` o la ausencia en `ListSessions` `[V]` |
| 3 | ¿Qué procesos quedan? | Los del `user@` (compositor si L1 falla, apps, terceros, D-Bus) |
| 4 | ¿greetd muere antes de cerrar PAM? | PAM sin cerrar; logind limpia cuando el scope se vacía `[H]` |
| 5 | ¿`user@` sigue activo? | Los procesos sobreviven hasta L3 |
| 6 | ¿Apps fuera del target? | Sobreviven |
| 7 | ¿Terceros por D-Bus? | Sobreviven |
| 8 | ¿Otra sesión del mismo usuario? | Todo sobrevive |
| 9 | ¿Puede reaparecer el greeter con el compositor anterior conservando capacidades? | **Sí**, puede reaparecer; con §6.3 y M-GRAB no conserva master ni recepción evdev. **Sí conserva** mappings (inocuos tras el paso 5), hidraw (P-02) y uinput (P-03) |
| 10 | ¿Cuándo es completo el retorno? | Para **autenticar**: `GREETER_AUTHENTICATION_READY`. Para declarar la sesión **terminada**: `REMOVED` ∧ `RECONCILED`, que PARALLAX solo puede garantizar sobre sus unidades y nunca sobre procesos externos |

---

## 8. Matriz transversal de garantías

| Propiedad | Garantía propuesta | Evidencia | Limitación | Estado |
|---|---|---|---|---|
| Eventos evdev nuevos | Solo el greeter los recibe mientras dura el grab | `input_pass_values`, `evdev_events` v7.3 `[V-p]` | Kernel de Arch | **Demostrado en código** (upstream); requiere prueba |
| Eventos evdev ya encolados | Ninguna; irrelevante si el campo estaba deshabilitado | `evdev_grab` no vacía búferes `[V-p]` | — | Demostrado en código (límite) |
| Estado de teclas | Ninguna | `input_get_disposition` `[V-p]` | `EVIOCGKEY` durante el grab | **No garantizable** (RV-47) |
| Informes hidraw | Ninguna por el kernel; depende de udev/ACL | `hidraw.c` `[V-p]` | Reglas de Arch sin leer | **Requiere decisión** (D-34) + prueba |
| Entrada uinput | Ninguna de integridad | `uinput.c` → `input_event` `[V-p]` | Permisos de `/dev/uinput` | **No garantizable**; requiere decisión (D-35) |
| Hotplug | `HOTPLUG_REDUCIDO` (§3.3) | Inferido de la secuencia udev | Ventana 1→7 | Inferido; requiere prueba |
| DRM master | Exclusivo del greeter o fallo detectado | `drm_auth.c` `[V-p]` | RV-44 (disponibilidad) | Demostrado en código |
| Estado de presentación | `DRM_STATE_COMMITTED` + verificación por consulta | `drm_atomic*` `[V-p resumen]` + `[H]` | Driver, PSR | Inferido; requiere prueba y hardware |
| Mappings antiguos | Inocuos tras el paso 5 de §6.3 | `[H]` (`drm_gem.c` no leído) | Planos no cubiertos (RV-51) | Inferido; requiere prueba |
| Sesión anterior | Detectable (`ListSessions`), no forzable por el greeter | login1 `[V]` | Asíncrona | Demostrado en documentación |
| Procesos externos | Ninguna | logind.conf `[V]` | linger, otras sesiones | **No garantizable** |
| Accesibilidad | Sin exclusión por defecto | `[H]` | Conflictos D-34/D-35 | Requiere decisión + prueba |

---

## 9. Evaluación de M-GRAB

**Resultado: `VIABLE CON LIMITACIONES EXPLÍCITAS`**, con un alcance más estrecho que en la Tarea 014.

Garantía precisa:

> «Mientras dura el grab, el greeter es el único receptor de los eventos **evdev** nuevos de cada `input_dev` capturado. **No** garantiza: confidencialidad frente a hidraw, integridad frente a uinput, ocultación del estado de teclas, cobertura atómica durante el hotplug.»

---

## 10. Fichas de decisión D-32 a D-35 (pendientes del propietario)

### D-32 — Recuperación y timeouts

| Elemento | Opciones o valor inicial *para pruebas* |
|---|---|
| Reintentos | Backoff exponencial 1 s → 30 s; tope de 10 intentos *(valor de prueba)* |
| Recuperable | `EBUSY` transitorio, `EINTR`, pérdida de logind con reconexión |
| Fallo permanente | Agotado el tope, o `UNPROTECTED` persistente |
| No puede presentar | Sin aviso posible: pantalla en el estado anterior **sin aceptar entrada**; registro en el journal; intervención externa (reinicio) |
| No puede proteger la entrada | Aviso en pantalla; campo deshabilitado |
| Riesgo de reintentos infinitos | Hotplug continuo o atacante con uinput → tope y estado `REFUSED` |

### D-33 — Campo de credenciales ante hotplug

| Opción | Privacidad | UX | Accesibilidad | Entrada a ciegas |
|---|---|---|---|---|
| Vaciar | Alta | Media | Coste de repetir | Evita completar a ciegas |
| Conservar deshabilitado | Media: lo ya tecleado fue con cobertura | Alta | Alta | Riesgo bajo |
| Cancelar y reiniciar | Alta | Baja | Baja | Evita |
| Bloquear hasta recuperar | Alta | Baja | Media | Evita |

### D-34 — hidraw

Matriz de §4.2. Recomendación técnica: D + A, sujeta a la lectura de las reglas de Arch.

### D-35 — Asistencia con grab previo

| Opción | Efecto |
|---|---|
| Rechazar | **Excluye** tecnologías de asistencia: no recomendable |
| Lista explícita de servicios de confianza (p. ej. brltty como root) | Requiere identificar al dueño del grab, que el kernel **no expone** `[H]` → identificación por unidad systemd del servicio |
| Modo de asistencia con garantía reducida, explícito para el usuario | Viable; transparente |
| Interfaz de entrada controlada por el compositor (el demonio inyecta por uinput en lugar de capturar) | Ya es el modelo de brltty: compatible con M-GRAB |

---

## 11. Riesgos propuestos (no aplicados)

| ID | Acción |
|---|---|
| RV-37 | Sigue abierto: ciclo completo sin demostrar (§7) |
| RV-38 | Actualizar: expuesto hasta el paso 5 de §6.3 |
| RV-43 | Actualizar: mitigado para evdev; hidraw en RV-49 |
| RV-44 | Sin cambios (disponibilidad) |
| RV-46 | Sin cambios |
| RV-47 | Mantener: estado de teclas |
| RV-48 | Reformular: carrera acotada por permisos |
| RV-49 | Mantener; enlazar con D-34 |
| **RV-50** (nuevo) | Entrada sintética por uinput: integridad, no confidencialidad |
| **RV-51** (nuevo) | Mapping antiguo visible en planos no incluidos en el commit |

---

## 12. Pruebas pendientes (no autorizadas)

| ID | Hipótesis | Kernel | Entorno | VM | HW | Root | Aprobado si | Cambia el diseño si |
|---|---|---|---|---|---|---|---|---|
| T-GRAB | §2 de T014 | El de Arch | Dispositivo virtual y USB | Sí | Opcional | Sí | Solo el dueño recibe | Otro cliente recibe |
| T-INPUT-COVERAGE | Clasificación completa | Arch | Varios dispositivos | Sí | Recomendado | Sí | Ningún `event*` sin clasificar | Nodos fuera del inventario |
| **T-HOTPLUG-RACE** | §3.3 | Arch | Hotplug virtual en bucle | Sí | No | Sí | Ningún proceso sin permiso lee; las credenciales se deshabilitan ante el `add` | Un usuario sin ACL lee el nodo |
| **T-HIDRAW** | §4 | Arch | Teclado USB, FIDO | Passthrough | **Sí** | Sí | ACL según la política aprobada | Un usuario lee el hidraw de un teclado |
| **T-UINPUT-GRAB** | §5 | Arch | `/dev/uinput` | Sí | No | Sí | Los eventos inyectados llegan al dueño del grab | Comportamiento distinto (corregir §5) |
| T-DRM | Master | Arch | virtio-gpu + real | Sí | **Sí** | Sí | `EBUSY` detectado | Master compartido |
| **T-DRM-MAPPING** | §6 | Arch | GPU real (planos overlay) | Parcial | **Sí** | Sí | Escribir en el mapping tras el paso 5 no cambia la pantalla | Cambio visible |
| T-GREETER-SECURITY | Regla de habilitación | Arch | Greeter instrumentado | Sí | No | No | Ninguna habilitación indebida | Alguna |
| T-SESSION-FAIL | §7.2 | Arch | Sesión completa | Sí | Recomendado | Sí | Estados observados según §7.1 | Transiciones no previstas |
| T-ACCESSIBILITY | Sin exclusión | Arch | brltty, Orca, IME | Sí | Braille real | Sí | Todo funciona | Exclusión |

---

## 13. Recomendación arquitectónica (propuesta)

1. **Mantener la alternativa E**, con estos requisitos del greeter:
   - M-GRAB;
   - el protocolo DRM de §6.3 (todos los planos y CRTC, verificación por consulta);
   - la regla `GREETER_AUTHENTICATION_READY`.
2. **Redactar las garantías por dimensión.** Nunca usar «seguro» o «confidencial» de forma global.
3. Hacer de la política udev (grupo `input` vacío; hidraw según D-34; `/dev/uinput` solo root) una **condición de despliegue**, no una garantía del greeter. No hace falta ningún servicio privilegiado nuevo.

---

## 14. Veredicto final: `MITIGACIÓN PARCIAL CON LIMITACIONES EXPLÍCITAS`

| # | Pregunta | Respuesta |
|---|---|---|
| 1 | ¿Se puede cerrar la ventana de hotplug? | **No.** `HOTPLUG_REDUCIDO`, acotada por permisos |
| 2 | ¿Confidencialidad frente a hidraw? | **No la garantiza ni el kernel ni el greeter.** Depende de udev/ACL (D-34) |
| 3 | ¿uinput durante un grab? | Sus eventos **llegan al greeter**: afecta a la integridad, no a la confidencialidad |
| 4 | ¿Puede un mapping antiguo alterar la presentación tras el commit? | Sí, en cualquier plano que el commit no incluya. No, tras cubrir y verificar todos los planos (sujeto a T-DRM-MAPPING) |
| 5 | ¿Qué demuestra el primer commit? | `DRM_STATE_COMMITTED` de los objetos incluidos; nada de los no incluidos; nada sobre los píxeles |
| 6 | ¿Qué evidencia demuestra que terminó la sesión anterior? | `SessionRemoved` o la ausencia en `ListSessions` (sesión logind). Los procesos, solo por unidades conocidas |
| 7 | ¿Qué procesos pueden quedar? | Los de `user@`: compositor, apps, terceros, D-Bus, procesos fuera del cgroup |
| 8 | ¿Garantías sin servicios privilegiados nuevos? | Exclusividad evdev; master DRM; estado de presentación verificado por consulta; habilitación cerrada de credenciales. **No:** hidraw, integridad uinput, limpieza de procesos externos, aislamiento |
| 9 | ¿Decisiones del propietario? | D-31, D-32, D-33, D-34, D-35 y la política udev de despliegue |
| 10 | ¿Mantener E? | **Sí**, con los requisitos de §13 |
| 11 | ¿Pruebas imprescindibles? | T-GRAB, T-UINPUT-GRAB, T-HIDRAW, T-DRM-MAPPING, T-HOTPLUG-RACE, T-GREETER-SECURITY |
| 12 | ¿Sigue abierto RV-37? | **Sí** |

**Estado final:**
- Fase 0 `IN_PROGRESS`.
- Todos los ADR `Propuesto`.
- Fase 1 sin autorización.
- Repositorio sin modificaciones.
- D-32 a D-35 pendientes.
- RV-37 abierto.
