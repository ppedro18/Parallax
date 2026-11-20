# Tarea 018 — BrlAPI, ciclo de vida de brltty y terminación de sesiones systemd/logind

- **Fecha:** 2026-11-19
- **Fase 0:** `IN_PROGRESS`
- **ADR:** todos en `Propuesto`
- **Fase 1:** sin autorización
- **Naturaleza:** investigación de solo lectura. Es una propuesta, no una decisión. El informe está en el scratchpad y el repositorio no se ha modificado.
- **Marcas de evidencia:**
  - `[V]` fuente primaria exacta suficiente
  - `[V-p]` evidencia parcial
  - `[R]` repositorio PARALLAX
  - `[H]` hipótesis o inferencia
  - `[VM]` requiere validación real
  - `[P]` decisión del propietario

---

## 1. Resumen ejecutivo

**Veredicto: `EVIDENCIA PARCIAL; COMPORTAMIENTO EN EJECUCIÓN PENDIENTE`** (§14).

### BrlAPI (bloque A)

- **La inyección de teclado por parte de un cliente BrlAPI no está demostrada.**
  - En el protocolo v8 no hay ningún paquete del cliente que el servidor convierta en una tecla o en un comando.
  - `BRLAPI_PACKET_KEY` solo va del servidor al cliente.
- No es una refutación completa. Quedan sin leer las rutas en las que brltty genera entrada para el sistema operativo (`cmd_keycodes.c`, `kbd_*.c`, `uinput.c`) y el destino del portapapeles, que un cliente **sí** puede escribir.
- **Autenticación:**
  - Por defecto es `keyfile` (+ `polkit`).
  - Arch pide añadir al usuario al grupo `brlapi`. El comentario del propio servicio indica que ese grupo sirve para leer la clave.
  - Tras autenticarse, **no hay controles por paquete**: un miembro de `brlapi` puede entrar en modo *raw* y escribir bytes arbitrarios en el dispositivo braille, suspender el driver y cambiar parámetros globales.

### brltty (bloque B)

- **Privilegios:** se ejecuta como `brltty`, con los grupos suplementarios `input` y `root`, y con las capacidades ambientales `cap_sys_admin`, `cap_sys_tty_config` y `cap_mknod`.
  - **Ninguna directiva de endurecimiento está especificada:** no hay `CapabilityBoundingSet`, `NoNewPrivileges`, `Protect*`, `PrivateDevices`, `DeviceAllow` ni `SystemCallFilter`.
  - Según el comentario del upstream, `cap_sys_admin` sirve para **inyectar caracteres con `TIOCSTI`** en terminales.
- **Activación:**
  - Por udev, cuando se conecta una línea braille USB de la lista; no necesita habilitación `[H fuerte]`.
  - O por `brltty.path`, si se habilita.
- **Por tanto, RV-53 es un riesgo condicional:** solo existe cuando brltty está instalado **y** activo.

### systemd/logind (bloque C)

- **Con `KillUserProcesses=no` (Arch), el scope de la sesión se abandona** y la sesión puede seguir en `closing` **indefinidamente** mientras le queden procesos `[V-p session_stop_scope]`.
- **El user manager (`user@UID`) se detiene por otro mecanismo:** cuando no queda ninguna sesión que fije al usuario, pasado el retardo de parada y sin *linger* `[V-p user_may_gc]`.
- **Hallazgo nuevo para RV-37 — variante G-a′:** si el compositor es el **propio comando de greetd** (`exec`), el escenario «muere el comando y el compositor sigue vivo» **deja de existir**. greetd espera exactamente a ese PID `[V-p, Tarea 012]`, y su muerte cierra todos sus fds DRM y de entrada.
  - Esto no resuelve la limpieza de los hijos, que quedan en el scope abandonado.
  - Contradice D-28 (G-c).
  - **No está demostrado en ejecución.**

---

## 2. Alcance y limitaciones

- **Fuentes de brltty** (`BRLTTY-6.9.1`, tag; tarball del paquete Arch, Tarea 017):
  - leídos literalmente: `brlapi_protocol.h`, `brlapi.h.in` (parcial), `auth.c`, `brltty.conf.in`, `brltty@.service.in`, `brltty-device@.service`, `brltty.path` `[V-p]`;
  - `brlapi_server.c`: ~39 kB literales y el resto **resumido**;
  - no leídos: `cmd_keycodes.c`, `kbd_*.c`, `uinput.c`, `brltty@.path`, `systemd-wrapper`, `README.Systemd`.
- **systemd v262.2:**
  - `logind-user.c` leído completo `[V-p]`;
  - `logind-session.c` truncado en el 65 %; el resto resumido;
  - no leídos: `logind.c` (`manager_shall_kill`, interpretación de las opciones de configuración, cola de GC) y `logind-session.h` (macros de clase de sesión);
  - `systemd.special(7)`: las entradas de `graphical-session*` solo están resumidas.
- **Arch:** `scratchpad/arch-pkg/` en los tags `262.2-1` y `6.9.1-3` `[V]`.
- **No se tuvo acceso** a DEV-01 ni a archivos instalados. No se ejecutó nada.

---

## 3. Fuentes exactas

| Tema | Archivo y símbolo | Versión | Nivel |
|---|---|---|---|
| Paquetes del protocolo | `Programs/brlapi_protocol.h`, `BRLAPI_PACKET_*`, `BRLAPI_PROTOCOL_VERSION 8` | BRLTTY-6.9.1 | `[V-p]` literal |
| Tabla de handlers | `brlapi_server.c` `packetHandlers`, `processRequest` | Ídem | `[V-p resumen]` |
| Envío de teclas | `brlapi_server.c` `writeKey()` → `BRLAPI_PACKET_KEY` | Ídem | `[V-p]` literal |
| Modo raw | `handleEnterRawMode`, `handlePacket`, `checkDriverSpecificModePacket`, `isRawCapable` | Ídem | `[V-p]` literal |
| Parámetros escribibles | `handleParamValue`, `paramDispatch` | Ídem | `[V-p]` |
| Autenticación | `brlapi.h.in` `BRLAPI_DEFAUTH`; `auth.c` `authPerformServer`, `authPolkit_server` | Ídem | `[V-p]` literal |
| Unidades de brltty | `brltty@.service.in`, `brltty-device@.service`, `brltty.path` | Ídem | `[V-p]` literal |
| Parches de Arch | `brltty-6.2-systemd_sysusers_groups.patch`, `6.8-lock-brltty-user.patch` | 6.9.1-3 | `[V]` |
| Instalación en Arch | `PKGBUILD` `package_brltty()` | 6.9.1-3 | `[V]` |
| GC del usuario | `logind-user.c` `user_may_gc`, `user_pinned_by_sessions`, `user_stop_service`, `user_get_stop_delay` | v262.2 | `[V-p]` literal |
| Parada de la sesión | `logind-session.c` `session_stop`, `session_stop_scope`, `session_release` | v262.2 | `[V-p]` literal |
| `KillUserProcesses` en Arch | `PKGBUILD` `-Ddefault-kill-user-processes=false` | 262.2-1 | `[V]` |
| Unidades `user@`/slices | `user@.service(5)`, `systemd-logind.service(8)` | latest | `[V-p]` |

---

## 4. BrlAPI

### 4.1 Rutas de entrada (A.2)

| # | Ruta | Resultado |
|---|---|---|
| 1 | Cliente → brltty → dispositivo braille | **Sí:** `WRITE` (celdas); en modo raw, `PACKET` lleva bytes arbitrarios al driver con `writePacket` |
| 2 | Dispositivo → brltty → cliente | **Sí:** `writeKey` envía `BRLAPI_PACKET_KEY` |
| 3 | Cliente → brltty → teclado virtual | **No encontrada.** No hay ningún handler para un `KEY` enviado por el cliente; la API cliente no tiene `sendKey` ni `sendCommand` `[V-p]` |
| 4 | brltty → `/dev/uinput` → input → greeter | Posible **desde el propio brltty** (ACL `u:brltty`, `modprobe uinput`). No hay vínculo demostrado con peticiones de clientes |
| 5 | Cliente → otras vías | (a) `PARAM_CLIPBOARD_CONTENT` es escribible; si brltty lo pega después (por ejemplo, con un comando de pegado disparado por el **dispositivo**), el texto acaba en el SO por `TIOCSTI` o uinput `[H]`. (b) El modo raw permite enviar bytes al dispositivo; si el dispositivo los «reflejara» como teclas, `[H]` (raro). (c) `xbrlapi` usa XTest en X11, una vía ajena a brltty y que no aplica al greeter Wayland `[H]` |

**Tabla de distinciones:**

| Operación | ¿Puede hacerla un cliente? |
|---|---|
| Enviar comandos al dispositivo braille | Sí |
| Recibir eventos del dispositivo braille | Sí |
| Solicitar una acción de teclado | **No encontrada** |
| Crear o usar dispositivos virtuales de entrada | Solo brltty |
| Inyectar eventos directamente por uinput | Solo brltty o root |

### 4.2 Autorización (A.3)

- **Método por defecto:** `BRLAPI_DEFAUTH` = `keyfile:<ETCDIR>/brlapi.key` + `polkit`, si está compilado `[V-p]`.
  - En Arch, `polkit` está en `makedepends` y el paquete instala la política `org.a11y.brlapi.policy`.
  - La acción de polkit es `org.a11y.brlapi.write-display`, sin interacción `[V-p]`.
- **Grupo `brlapi`:** Arch indica «add your user to the brlapi group» `[V]`, y el servicio usa ese grupo «for reading the BrlAPI authorization key file» `[V-p]`.
  - Que el grupo dé lectura de `/etc/brlapi.key` es `[H]`: el modo y el grupo del archivo los fija `brltty-genkey`, que no se ha leído.
- **Tras autenticarse no hay privilegios por operación:** `processRequest` solo comprueba `c->auth`.
  - El modo raw y la suspensión solo exigen el *magic* y el nombre del driver.
  - Cualquier cliente autenticado tiene la API completa. **No es acceso administrativo al sistema**, pero sí control total del dispositivo braille (modo raw, suspender el driver) `[V-p]`.
- **Local frente a remoto:**
  - `Host=:0` limita la escucha a Unix local. El valor por defecto no está fijado en `brltty.conf.in`.
  - El valor por defecto de la librería cliente es TCP en localhost:4101 `[V-p]`. Que el servidor escuche en TCP por defecto: `[VM]`.
- **Dispositivo de otro usuario:** solo hay un demonio por dispositivo y la autorización no distingue usuarios humanos. Cualquier miembro de `brlapi` accede a la línea braille aunque la use otra persona `[H]`.

### 4.3 Clasificación (A.4)

**Inyección por cliente BrlAPI: `NO DEMOSTRADA`.**
- **No refutada:** faltan `cmd_keycodes.c`, `kbd_*.c`, `uinput.c` y la ruta del portapapeles.
- **Evidencia:** la tabla de handlers y la API cliente sin envío de teclas.

**RV-50 (conceptual):**

| Vía | Actores | Estado |
|---|---|---|
| uinput | root y brltty | Base del riesgo |
| BrlAPI | Miembros de `brlapi` | Sin vía de inyección demostrada. Quedan dos vías indirectas **por verificar**: portapapeles y modo raw |
| Integridad del dispositivo braille | Miembros de `brlapi` | **Nueva dimensión:** pueden escribir y suspender el dispositivo de otra persona. Es un riesgo de **asistencia y privacidad** (lo que se muestra en braille), no de credenciales del greeter `[H]` |

---

## 5. Instalación y ejecución de brltty en Arch

### 5.1 Qué instala (B.1)

`package_brltty()` `[V]`:

- `make install`, `install-systemd`, `install-udev`, `install-dracut`, `install-polkit`.
- `/etc/brltty.conf`.
- `brlapi.key`, generada en `post_install`.
- Se separa a otro paquete:
  - `90-brltty-usb-generic.rules` → paquete opcional **`brltty-udev-generic`**;
  - dracut, Java, OCaml, Python y TCL → paquetes aparte.
- Ninguna línea del PKGBUILD ni de `.install` ejecuta `systemctl enable` `[V]`.

### 5.2 Activación

| Disparador | Mecanismo | ¿Necesita habilitación? |
|---|---|---|
| Línea braille USB en la lista «customized» (incluida en `brltty`) | udev `TAG+="systemd"`, `SYSTEMD_WANTS+="brltty-device@.service"` → `brltty-device@` `Requires=brltty@-%i.service` `[V-p]` | **No** `[H fuerte]`: `SYSTEMD_WANTS` actúa sin `enable` |
| Dispositivos USB genéricos (convertidores serie) | `90-brltty-usb-generic.rules` | Solo si se instala `brltty-udev-generic` |
| `/etc/brltty.conf` presente | `brltty.path` `PathExists=` → `brltty@-etc-brltty.conf.service` | **Sí:** solo si se habilita `brltty.path`. El paquete instala `/etc/brltty.conf`, así que **al habilitarlo arrancaría siempre, con o sin hardware** `[H]` |
| Desconexión | `brltty-device@` `BindsTo=%i.device`; `brltty@` `StopWhenUnneeded=true` → se detiene | — |

`brltty@` tiene `ConditionPathExists=%I` y `RefuseManualStart=true` `[V-p]`.

**Sin hardware conectado y sin `brltty.path` habilitado, brltty no se ejecuta** `[H]` `[VM]`.

### 5.3 Matriz de privilegios efectivos (B.2)

| Directiva | Valor | Fuente |
|---|---|---|
| `User=` | `brltty` (bloqueado con `u!`) | `[V-p]` + parche `[V]` |
| `Group=` | No especificado (grupo primario del usuario) | `[V-p]` |
| `SupplementaryGroups=` | `tty`, `uucp` (Arch), `root` (×2), `audio`, `input`, `brlapi`; `pulse-access` comentado por Arch | `[V]` parche + `[V-p]` |
| `AmbientCapabilities=` | `cap_sys_tty_config`, **`cap_sys_admin`**, `cap_mknod` (tres líneas) | `[V-p]` |
| `CapabilityBoundingSet=` | **No especificado** | `[V-p]` |
| `NoNewPrivileges=` | No especificado | `[V-p]` |
| `DeviceAllow=` / `PrivateDevices=` | No especificados | `[V-p]` |
| `ProtectSystem=` / `ProtectHome=` | No especificados | `[V-p]` |
| `RestrictAddressFamilies=` / `SystemCallFilter=` | No especificados | `[V-p]` |
| `ExecStartPre=` | `-+/sbin/modprobe pcspkr`, `-+/sbin/modprobe uinput` (con `+`, es decir, como root) | `[V-p]` |

**Notas:**
- Las capacidades **ambientales** que el bounding set (completo por defecto) no reduce pasan a ser **efectivas** en el proceso `[H, semántica de capabilities(7)]`.
- Que systemd **acumule** las líneas repetidas de `AmbientCapabilities=` es `[H]`; no está verificado en v262.2.

### 5.4 TCB condicional (B.3)

| Pregunta | Respuesta |
|---|---|
| Privilegios | Usuario no root con `cap_sys_admin` (amplísima), los grupos `input` y `root`, y ACL sobre `uinput` y el hidraw braille |
| Qué puede leer | Todos los nodos `event*` (grupo `input`), salvo eventos nuevos de dispositivos en grab del greeter (Tarea 014); el estado de teclas (`EVIOCGKEY`); archivos del grupo `root`; `/dev/vcs*` (grupo `tty`) |
| Qué puede crear | Dispositivos uinput; nodos de dispositivo (`cap_mknod`) |
| Qué puede modificar | Inyectar en ttys (`TIOCSTI` con `cap_sys_admin`); emitir entrada por uinput hacia el greeter (Tarea 015) |
| Qué puede disputar al greeter | **DRM master** en un hueco (`cap_sys_admin` pasa `drm_master_check_perm`, Tarea 013) → el greeter recibe `EBUSY` (fallo de disponibilidad, detectable) |
| Qué queda fuera de su alcance | Eventos nuevos de un nodo en grab del greeter; un master ya en manos del greeter |

**RV-53:** debe describirse como **riesgo condicional**. Aparece solo si brltty está instalado **y** activo, por hardware de la lista o por `brltty.path` habilitado. **No** es un riesgo permanente de la imagen base salvo que D-36 decida instalarlo y habilitarlo `[P]`.

---

## 6. systemd y logind

### 6.1 Modelo de unidades (C.1)

```text
user.slice
└─ user-UID.slice
   ├─ user@UID.service            (user manager; BindsTo user-runtime-dir@UID.service [H: unidad no leída])
   │   └─ unidades de usuario: graphical-session(-pre).target (systemd), parallax-session.target* , compositor* , roles*, apps*
   └─ session-N.scope             (proceso de greetd worker + comando de sesión)
user-runtime-dir@UID.service      (/run/user/UID)
* = hipotético PARALLAX
```

- `graphical-session.target` y `graphical-session-pre.target` son unidades estándar de systemd, pero **nada en systemd las activa automáticamente**. Las activa un target de sesión del entorno de escritorio `[V-p resumen]`. Sin PARALLAX, en Arch no se activan solas.
- greetd y PAM (`pam_systemd`) crean `session-N.scope` (Tarea 013).

### 6.2 `KillUserProcesses` y vida del user manager (C.2)

| Mecanismo | Comportamiento | Evidencia |
|---|---|---|
| Fin de la sesión con `KillUserProcesses=no` | `session_stop_scope`: «always abandon the scope first». Sin kill: «Waiting for processes to exit», y la sesión puede persistir en `closing` **indefinidamente** | `[V-p]` literal |
| Con `KillUserProcesses=yes` (o `KillOnlyUsers`) | `manager_stop_unit(scope)`: systemd mata todo el scope | `[V-p]`; la lógica de `manager_shall_kill` no se leyó |
| `StopIdleSessionSec` | Si está activo y la clase de sesión lo permite, `session_stop(force=true)`: el scope se para **aunque** `KillUserProcesses=no` | `[V-p]` |
| Parada del user manager | `user_may_gc`: falso si alguna sesión fija al usuario, si no ha pasado el retardo de parada o si hay *linger* con el manager activo | `[V-p]` literal |
| Qué sesiones fijan al usuario | `USER_GC_BY_ANY` (cualquier sesión) o `BY_PIN` (`SESSION_CLASS_PIN_USER`) | Macros no leídas: **qué clases fijan es desconocido** |
| Retardo | `user_get_stop_delay` → `manager->user_stop_delay` (relación con `UserStopDelaySec` sin verificar en `logind.c`; 10 s según la documentación, Tarea 012) | `[V-p]` |
| Cómo se detiene `user@` | Se para `user-runtime-dir@`, y `user@` cae por `BindsTo=` | `[V-p]` comentario; unidad no leída |
| Varias sesiones, TTY o SSH del mismo usuario | Fijan al usuario: `user@` **no** se detiene `[H, porque son clase user]` | — |
| Fin de la última sesión gráfica con otra sesión abierta | `user@` sigue activo; las unidades PARALLAX solo se detienen por su propio grafo, L1 o L3 | — |

### 6.3 Matriz de escenarios de terminación (C.3)

Notación: **G-b** = compositor dentro del user manager (la propuesta actual); **G-a′** = compositor como comando de greetd con `exec`.

| # | Escenario | Unidad que termina | Puede sobrevivir | Procesos vivos | DRM/input | ¿Sesión nueva? | ¿Riesgo de sesión anterior desbloqueada? | Limpieza |
|---|---|---|---|---|---|---|---|---|
| 1 | Logout normal | G-b: el comando para el target (diseño); la sesión logind pasa a `closing`; `user@` cae a los 10 s si es la última sesión | Terceros D-Bus | Terceros hasta la parada de `user@` | Compositor cerrado → fds cerrados | Sí | No (master + grab del greeter) | Condicional |
| 2 | Muere el comando de greetd | G-b: **nada en `user@`**; scope abandonado | Compositor y apps (G-b) | Ídem | Logind pausa y revoca al activarse el greeter (mejor esfuerzo, Tarea 013) | Sí | **Mitigado** por M-GRAB y comprobación de master; no eliminado | No demostrada (L1/L2/L3). **G-a′: el comando es el compositor, así que su muerte cierra todos los fds** |
| 3 | Muere el compositor | G-b: `BindsTo` → target (diseño) | Terceros | Roles y apps hasta `PartOf` | fds cerrados | Sí | No | Condicional `[VM]` |
| 4 | Muere el Shell | Ningún efecto de sesión; el compositor lo relanza (D-18) | — | — | — | — | No | — |
| 5 | Parada forzada del scope (`TerminateSession`) | El scope entero | `user@` y sus unidades (G-b) | Compositor (G-b) | Como en 2 | Sí | Como en 2 | G-b: no demostrada; **G-a′: el compositor muere con el scope** |
| 6 | Fin de la última sesión | `user@` tras el retardo, sin linger | Nada del usuario | — | — | Sí | No | **Garantizada por logind tras el retardo** `[V-p]` |
| 7 | Sesión TTY del mismo usuario abierta | Solo el scope gráfico | `user@` y todo lo suyo | Compositor (G-b) | Como en 2 | Sí | Como en 2 | No |
| 8 | Sesión SSH abierta | Igual que 7 | Igual | Igual | Igual | Sí | Igual | No |
| 9 | *Linger* activo | Scope | `user@` permanente | Todo | Como en 2 | Sí | Como en 2 | Solo L1/L3 |
| 10 | `user@` activo mientras greetd lanza el greeter | — | Todo | Compositor (G-b) | Como en 2 | Sí | Como en 2 | L3 en el siguiente inicio |

---

## 7. Variantes G-a y G-b (C.4)

| | G-a (compositor en el scope de sesión) | **G-a′** (compositor = comando de greetd con `exec`) | G-b (compositor en `user@`, `parallax-session.target`) |
|---|---|---|---|
| Vínculo comando ↔ compositor | Del árbol de procesos del scope | **El mismo PID**: greetd espera por él (`waitpid` del hijo que ejecuta `exec <cmd>`, Tarea 012 `[V-p]`) | Ninguno. El comando solo vigila el target |
| ¿Depende de `KillUserProcesses=yes`? | Sí, para matar a los hijos | **No** para el compositor; sí para los hijos que queden en el scope | No aplica |
| Roles y apps | Hijos en el scope; con `no`, sobreviven abandonados | Igual; se pueden lanzar como unidades de `user@` con `PartOf=graphical-session.target` y que las detenga el compositor | Unidades de `user@` |
| Grafo systemd | Se pierde | Parcial: el compositor no es una unidad, pero puede arrancar y parar el target | Completo |
| ¿Resuelve RV-37 (caída del comando con compositor vivo)? | Parcialmente | **El escenario no existe por construcción del árbol de procesos** `[H fuerte]` `[VM]` | No (necesita L1–L3) |
| Dependencias systemd | — | — | `BindsTo`, `PartOf`, `Wants` y `After` **solo dentro del user manager**. Ninguna relación del user manager puede detener `session-N.scope`, que pertenece al gestor del sistema, ni al revés `[H]` |

**Direcciones relevantes** (Tarea 011 `[V]`):
- `BindsTo=X` en Y: si X cae, cae Y.
- `PartOf=X` en Y: parar X para Y.
- `Requires` y `Wants` arrastran el arranque.
- `After` solo ordena.
- Parar un target **no** para el servicio que lo enlaza con `BindsTo` desde el target hacia el servicio.

**¿Hay alguna variante que resuelva RV-37 sin `KillUserProcesses=yes`?**
- **G-a′ elimina el escenario central:** muere el comando, pero el compositor queda vivo con dispositivos.
- **No** garantiza la limpieza de los hijos ni de los terceros.
- No está demostrada en ejecución.
- Contradice D-28 (G-c, comando de sesión propio que supervisa el target).

---

## 8. Política de sesión (C.5): alternativas, no decisión

| Alternativa | Limpieza de procesos | Liberación de dispositivos | Seguridad de presentación |
|---|---|---|---|
| Terminar solo los procesos de PARALLAX (L3) | Sus unidades; nunca terceros | Si el compositor termina | Greeter: master + grab |
| Terminar todo el user manager al cerrar la última sesión | Lo hace logind de forma nativa; no con *linger* ni con otras sesiones | Sí | Igual |
| Compositor en el scope de sesión (G-a/G-a′) | Hijos solo con `KillUserProcesses=yes` o `TerminateSession` | **Sí, con la muerte del comando** (G-a′) | Igual |
| `user@` activo + scopes dedicados (G-b) | L1–L3 | Depende de L1 | Igual |
| Activar `KillUserProcesses=yes` solo para PARALLAX (`KillOnlyUsers=` o por usuario) | Mata el scope; no afecta a `user@` | Si el compositor está en el scope | Igual |

En todos los casos, **la seguridad de presentación y entrada la dan el master y el grab del greeter**, no la limpieza.

---

## 9. Reconciliación de las Tareas 013–017

| Tema | Conclusión anterior | Evidencia nueva | Conclusión actual | Estado |
|---|---|---|---|---|
| RV-37 | T015: ciclo completo sin demostrar | G-a′; `session_stop_scope` | Escenario central eliminable con G-a′; limpieza sin garantizar | **Abierto** |
| RV-43 | T017: root, `input`, servicios, joysticks | — | Sin cambios | Abierto |
| RV-44 | T017: + `cap_sys_admin` | Servicio de brltty confirmado | Sin cambios | Abierto |
| RV-48 | T017: root, `input`, brltty | — | Sin cambios | Abierto |
| RV-49 | Debilitado | — | Sin cambios | Abierto |
| RV-50 | root + brltty + BrlAPI `[H]` | BrlAPI: sin inyección demostrada | BrlAPI **no demostrada**; dimensión nueva: control del dispositivo braille | Abierto |
| RV-51 | Desconocido | — | Sin cambios | Abierto |
| RV-52 | `renderD*` 0666 | — | Sin cambios | Propuesto |
| RV-53 | brltty en la TCB | Activación por hardware de la lista; sin endurecimiento | **Condicional** | Propuesto |
| D-34 | Política upstream + auditoría | — | Sin cambios | Pendiente `[P]` |
| D-35 | Opciones a–d | El endurecimiento (b) debe tener en cuenta que `cap_sys_admin` sirve para `TIOCSTI` | Sin cambios de fondo | Pendiente `[P]` |
| D-36 | brltty en la base | Sin `enable` por defecto; se activa con hardware | Sin cambios | Pendiente `[P]` |
| M-GRAB | Viable con limitaciones | Excluye a brltty de los eventos nuevos | Sin cambios | Sin validar |
| Alternativa E | Se mantiene | G-a′ como complemento posible | Se mantiene | Propuesta |
| `KillUserProcesses` | T012/T013: `yes` | Arch `no`; `session_stop_scope` abandona | **Refutada** la premisa de T012/T013 | — |
| G-a / G-b | T012: G-a viable con `KillUserProcesses` | G-a′ cambia el análisis | G-a′ nueva; G-b necesita L1–L3 | Pendiente (D-28) |

Las afirmaciones originales se conservan en sus informes y no se reescriben.

---

## 10. Riesgos y decisiones afectadas (propuestas, sin aplicar)

| Elemento | Propuesta |
|---|---|
| RV-53 | Redactarlo como **condicional**: solo con brltty activo |
| RV-50 | Añadir «BrlAPI: inyección no demostrada; pendiente de verificar el portapapeles y el modo raw» |
| **RV-54** (nuevo) | Un cliente BrlAPI autenticado (miembros de `brlapi`) puede tomar el control del dispositivo braille de otro usuario: modo raw, suspensión y parámetros globales. Dimensión de asistencia y privacidad |
| **RV-55** (nuevo) | Con `KillUserProcesses=no` y sesiones adicionales o *linger*, los procesos de una sesión gráfica terminada sobreviven **indefinidamente** |
| D-28 | Añadir G-a′ como alternativa evaluable |
| D-31 | Añadir `KillOnlyUsers` y `KillUserProcesses` como alternativa a desactivar *linger* |
| **D-37** (nuevo) `[P]` | Política de sesión de §8 |

---

## 11. Pruebas futuras (no autorizadas)

| ID | Hipótesis | Entorno | Aprobado si |
|---|---|---|---|
| T-BRLAPI | Ningún paquete del cliente provoca entrada en el SO; portapapeles y modo raw | Arch + brltty + línea braille real o simulada | Ningún evento evdev o tty generado por el cliente |
| T-BRLTTY-ACT | Activación solo por hardware de la lista o por `brltty.path` habilitado | Arch | `systemctl status` coincide con §5.2 |
| T-BRLTTY-CAPS | Capacidades efectivas del proceso | Arch + brltty activo | `/proc/<pid>/status` (`CapEff`) coincide con §5.3 |
| T-LOGIND-GC | `user@` cae tras el retardo; no cae con TTY/SSH ni con *linger* | VM | Coincide con §6.2 |
| T-KILL-ARCH | Scope abandonado con `no`; matado con `KillOnlyUsers` | VM | Coincide |
| T-GA-PRIME | Con G-a′, la muerte del comando cierra los fds DRM/input y deja los hijos en `closing` | VM + KMS | Coincide |
| T-SESSION-FAIL | Escenarios de §6.3 | VM | Coincide |

---

## 12. Recomendaciones técnicas (no normativas)

1. Evaluar **G-a′** frente a G-c (D-28). G-a′ elimina el caso más grave de RV-37 sin depender de `KillUserProcesses`.
2. Tratar brltty como **TCB condicional**. Si se instala, proponer un endurecimiento compatible que conserve `TIOCSTI` y uinput, y validarlo con usuarios de braille (T-ACCESSIBILITY).
3. No dar por limpia la sesión anterior basándose en `user@`: con otras sesiones o *linger*, sobrevive.
4. Leer `cmd_keycodes.c`, `uinput.c` y `logind.c` antes de cerrar RV-50 y la parte de GC.

**Separación:**
- Lo anterior son **recomendaciones técnicas**.
- **Decisiones de arquitectura `[P]`:** D-28, D-31, D-34, D-35, D-36 y D-37.

---

## 13. Decisiones que requieren al propietario

| ID | Decisión |
|---|---|
| D-28 | Comando de sesión: G-c frente a G-a′ |
| D-31 | *Linger* y `KillUserProcesses`/`KillOnlyUsers` |
| D-32 a D-35 | Sin cambios |
| D-36 | brltty en la imagen base |
| D-37 | Política de sesión (§8) |
| Autorizaciones | Pruebas de §11 y acceso a DEV-01 (ninguna concedida) |

---

## 14. Veredicto final: `EVIDENCIA PARCIAL; COMPORTAMIENTO EN EJECUCIÓN PENDIENTE`

**Justificación:**
- BrlAPI está analizado a partir de las fuentes exactas, pero sin las rutas de generación de entrada.
- La unidad de brltty y su activación se conocen en las fuentes y el paquete. Su comportamiento efectivo (capacidades, arranque) necesita ejecución.
- La GC de logind y el abandono del scope están leídos en v262.2, pero sin `logind.c` ni las macros de clase.
- G-a′ es una inferencia fuerte sin validar.
- No hay contradicciones que obliguen a rediseñar. Sí hay una **premisa refutada** (`KillUserProcesses`) y una **alternativa nueva**.

---

## ESTADO DEL PROYECTO

| Elemento | Estado |
|---|---|
| Fase 0 | `IN_PROGRESS` |
| ADR | Todos `Propuesto` |
| Fase 1 | Sin autorización |
| RV-37 | **Abierto.** El escenario central es eliminable en diseño con G-a′, pero falta validarlo y la limpieza no está garantizada |
| D-31 a D-36 | Pendientes; D-37 propuesto |
| Repositorio | Sin modificaciones |
| Pruebas | Ninguna ejecutada |
| DEV-01 | No accedido |

**Decisiones que requieren autorización del propietario:**
- D-28, D-31, D-32, D-33, D-34, D-35, D-36 y D-37.
- Cualquier prueba.
- Cualquier acceso a DEV-01.

**Recomendaciones técnicas**, separadas de esas decisiones: las de §12.
