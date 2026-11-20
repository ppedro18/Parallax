# Tarea 017 — Verificación de fuentes pendientes y reconciliación de las Tareas 013–016

- **Fecha:** 2026-11-19
- **Estado del proyecto:** Fase 0 `IN_PROGRESS` · todos los ADR `Propuesto` · Fase 1 sin autorización
- **Naturaleza:** solo lectura. Es una propuesta pendiente de aprobación. El informe está en el scratchpad y el repositorio no se ha modificado.
- **Marcas:** `[V]` fuente primaria suficiente · `[V-p]` evidencia parcial · `[R]` repositorio · `[H]` inferencia · `[VM]` requiere validación real · `[P]` decisión del propietario.

---

## 1. Resumen ejecutivo

**Veredicto: `EVIDENCIA PARCIAL; CONTRADICCIONES DOCUMENTALES RESUELTAS PARCIALMENTE`** (§15).

**PKGBUILD de systemd: no leído.**
- La web de empaquetado de Arch devolvió páginas de bloqueo (Anubis).
- Un intento posterior por la API de GitLab **fue denegado por el control de permisos de esta sesión**. No se buscó otra forma de saltar el bloqueo.
- Siguen sin conocerse los valores de Arch para `GROUP_RENDER_MODE`, `GROUP_RENDER_UACCESS` y `HAVE_ACL`.

**brltty: reglas leídas en upstream, no en el paquete de Arch.** Se leyeron las reglas y unidades de brltty con la ref `BRLTTY-6.9.1` `[V-p]`. Que coincidan con el paquete Arch 6.9.1-3 **no está verificado**, porque el PKGBUILD quedó bloqueado. Lo que muestran cambia el análisis de D-35 y RV-50:
- `/dev/uinput`, el hidraw de una lista cerrada de líneas braille y `tty0` reciben **una ACL para el usuario `brltty`**, no para el usuario activo ni para un grupo humano.
- El servicio `brltty@.service` se ejecuta como `User=brltty`, con los grupos suplementarios **`input`, `root`**, `tty`, `dialout`, `audio`, `pulse-access` y `brlapi`, y con **`AmbientCapabilities=cap_sys_admin`**, además de `cap_sys_tty_config` y `cap_mknod`.
- **Consecuencia:** con brltty activo, existe un proceso que **puede leer todos los teclados** (grupo `input`) y que **tiene `CAP_SYS_ADMIN`**. Esa capacidad pasa el control de `drm_master_check_perm` (Tarea 013), así que puede **tomar DRM master**.
- Por tanto, brltty **forma parte de la base de confianza (TCB)** cuando está instalado y activo. Ninguna mitigación del greeter lo contiene.

**Correcciones a las Tareas 013–016** (§10):
- Usuarios humanos y uinput: **no** reciben acceso por las reglas de brltty leídas.
- Teclados durante el grab: brltty **no** recibe eventos nuevos (el grab lo excluye), pero puede consultar el estado de teclas (RV-47).
- DRM master: un proceso con `cap_sys_admin` **sí** puede recuperarlo. Esto amplía RV-44.

---

## 2. Fuentes consultadas

| Fuente | Ref o versión | Resultado |
|---|---|---|
| PKGBUILD de systemd (gitlab.archlinux.org) | 262.2-1 | **Bloqueado** (Anubis en la Tarea 016; denegado por permisos en esta tarea) |
| PKGBUILD de brltty (API de GitLab) | main | **Bloqueado** (Anubis) |
| `brltty/brltty` `Autostart/Udev/{uinput,hid,beeper}.rules`, `usb-template.rules.in` | `BRLTTY-6.9.1` (tag o rama sin confirmar) | Leídos `[V-p]` |
| `brltty/brltty` `Autostart/Systemd/{sysusers,brltty@.service.in}` | Ídem | Leídos `[V-p]`. Son plantillas con `@…@` |
| Listado del paquete `brltty 6.9.1-3` (archlinux.org) | Tarea 016 | `[V-p]`: los nombres instalados son `90-brltty-*.rules` |
| Reglas de systemd v262.2 | Tarea 016 | `[V-p]` upstream |
| Steam, OpenRGB, QMK/VIA | — | **No investigados.** Son paquetes opcionales y no consta que formen parte de la instalación base de PARALLAX `[R: ausencia]`. No cambian las conclusiones de la base |

**Correspondencia entre nombres** (`[H]`, sin el PKGBUILD que lo confirme):

| Upstream | Nombre en el paquete Arch |
|---|---|
| `uinput.rules` | `90-brltty-uinput.rules` |
| `hid.rules` | `90-brltty-hid.rules` |
| `beeper.rules` | `90-brltty-beeper.rules` |
| `usb-template.rules.in` (plantilla completada) | `90-brltty-usb-customized.rules` |
| `sysusers` | `sysusers.d/brltty.conf` |

---

## 3. PKGBUILD y opciones meson de systemd

| Opción | Valor por defecto upstream | Valor de Arch | Efecto | Evidencia |
|---|---|---|---|---|
| `group-render-mode` | `0666` | **Desconocido** | Modo de `renderD*` | upstream `[V-p]`; Arch: bloqueado |
| `GROUP_RENDER_UACCESS` | Depende de la opción meson de render-access (no localizada como tal en `meson_options.txt` en la Tarea 016) | **Desconocido** | `uaccess` en `renderD*` | — |
| `HAVE_ACL` | Depende de `acl` | **Desconocido** (`[H]`: activado, porque Arch depende de `acl`) | Activa el builtin `uaccess` | — |

No se deduce ningún valor a partir del «comportamiento habitual de Arch».

---

## 4. Reglas de brltty (upstream `BRLTTY-6.9.1`, `[V-p]`)

| Archivo | Condición | Propiedades modificadas | Acceso concedido | Servicio afectado |
|---|---|---|---|---|
| `uinput.rules` | `KERNEL=="uinput"`, `ACTION=="add\|change"`, `TEST=="/usr/bin/setfacl"` | Ninguna (ni `GROUP`, ni `MODE`, ni `uaccess`) | `RUN+="/usr/bin/setfacl -m u:brltty:rw /dev/$name"` → **ACL para el usuario `brltty`** | brltty |
| `hid.rules` | `SUBSYSTEM=="hidraw"`, `ACTION=="add"`, nombre en una **lista cerrada** (NLS eReader, Brailliant BI 40X/20X, APH Chameleon 20, APH Mantis Q40, Humanware BrailleOne, etc.) | `SYMLINK+="brltty/HID-…"` | ACL `u:brltty:rw` **solo en esos dispositivos** | brltty |
| `beeper.rules` | `KERNEL=="tty0"` | — | ACL `u:brltty:rw` en `tty0` | brltty |
| `usb-template.rules.in` | `usb_device`/`usb`; lista de IDs **vacía en la plantilla** (`BEGIN/END_USB_BRAILLE_DEVICES`) | `TAG+="systemd"`, `SYSTEMD_WANTS+="brltty-device@.service"`, `RUN+=…udev-wrapper` | Arranque del servicio cuando aparece una línea braille | brltty |

| Pregunta | Respuesta |
|---|---|
| 1. Propietario, grupo, modo | No los cambia; usa ACL |
| 2. `uaccess` | No |
| 3. Grupo dedicado | Crea el usuario `brltty` y el grupo `brlapi` |
| 4. Acceso a uinput | **Solo el usuario `brltty`** (y root), si existe `setfacl` |
| 5. hidraw | Solo la lista cerrada de líneas braille |
| 6. Dispositivos input | No cambia sus permisos. Accede a ellos **por pertenecer al grupo `input`** |
| 7. Diferencia upstream–Arch | **Desconocida** (PKGBUILD bloqueado) |
| 8. Usuario de ejecución | `User=brltty`. Grupos: `tty`, `dialout`, `root` (×2), `audio`, `pulse-access`, `input`, `brlapi` |
| 9. Privilegios efectivos | `cap_sys_admin`, `cap_sys_tty_config`, `cap_mknod` como capacidades ambientales. **`cap_sys_admin` equivale en la práctica a root** para DRM y otras áreas `[H]` |
| 10. Impacto de restringirlo | Quitar `input` o `cap_sys_admin` puede romper funciones de brltty (la lectura de teclado para órdenes braille). Sin validar `[VM]` |

---

## 5. Reglas de terceros

- No se identificó ningún otro paquete de la **base** de PARALLAX que conceda acceso a `input`, hidraw, uinput o DRM.
- Steam Input, OpenRGB y QMK/VIA son **opcionales**: no se tratan como dependencia y no se leyeron.
- Requisito propuesto `[P]`: si se instalan, auditar sus reglas como condición de despliegue.

---

## 6. Corrección del modelo de input y hotplug

| # | Pregunta | Respuesta |
|---|---|---|
| 1 | ¿Quién puede abrir un teclado nuevo antes de `EVIOCGRAB`? | root, los miembros del grupo `input` y **el servicio `brltty` si está activo** (pertenece a `input`). Upstream **no** da `uaccess` al usuario activo `[V-p]` |
| 2 | ¿Y si `input` tiene usuarios humanos? | Esos usuarios leen cualquier teclado en la ventana previa al grab y en cualquier momento sin grab |
| 3 | ¿Y si está vacío? | Quedan root y los servicios del sistema que tengan `input` como grupo suplementario (brltty) |
| 4 | Joysticks y otros con `uaccess` | El usuario activo, el del greeter, los puede abrir; el usuario anterior pierde la ACL al cambiar de sesión `[H]` |
| 5 | fds abiertos antes del cambio de sesión | Siguen abiertos `[H]`. Solo los revoca logind, y solo si son suyos (Tarea 013) |
| 6 | ¿Retirar una ACL cierra los fds abiertos? | No `[H; la ACL se comprueba en `open()`]` |
| 7 | ¿Y si una regla posterior cambia `GROUP` o `MODE`? | Sustituye el valor anterior `[H]`. brltty no lo hace en las reglas leídas |
| 8 | ¿Qué depende de qué? | **Del sistema:** quién puede abrir el nodo. **Del greeter:** la exclusividad evdev mientras dura el grab (Tarea 014) |

**Redacción propuesta para RV-43:**
> «Los procesos que mantienen abierto un fd evdev directo pueden ser root, los miembros del grupo `input` (incluidos los servicios con `input` como grupo suplementario, como brltty) y el usuario activo para joysticks. Upstream no concede `uaccess` a teclados ni ratones.»

**Redacción propuesta para RV-48:**
> «En la ventana de hotplug pueden abrir el nodo root, el grupo `input` (incluidos servicios como brltty) y, para joysticks, el usuario activo.»

---

## 7. Corrección del modelo hidraw

| Vía de acceso | Situación |
|---|---|
| ACL | `uaccess` upstream para tokens de seguridad, carteras, controladores AV, ratones 3D y lectores de tarjetas; ACL `u:brltty` para la lista braille |
| Grupo | upstream no fija grupo en hidraw. El grupo `root` de brltty **no** le da acceso si el modo es 0600 `[H]` |
| Permisos tradicionales | root:root, 0600 por defecto `[H]` |
| Root y `cap_sys_admin` | Acceso total (brltty tiene `cap_sys_admin`, pero eso no salta permisos de archivo; `CAP_DAC_OVERRIDE` haría falta `[H]`) |
| fds directos | Sin revocación |
| fds de logind | Revocados (Tarea 013) |
| Clasificación | `fido_id` y la lista de brltty son **listas permitidas**. Un dispositivo braille fuera de la lista no recibe ACL (fallo funcional, no de seguridad) |

**¿Cubre la política upstream lo que PARALLAX quiere soportar?**
- Teclados: **no se exponen** a usuarios.
- FIDO: **sí**, si `fido_id` lo clasifica.
- Líneas braille hidraw: **solo las de la lista**.

**Recomendación técnica para D-34 `[P]`:** mantener la política upstream sin reglas propias (opción D ≈ upstream), más A (riesgo residual: root, servicios privilegiados y clasificación incompleta), más una auditoría de las reglas de terceros instaladas.

---

## 8. Corrección del modelo uinput

| Dimensión | Con las reglas de brltty leídas |
|---|---|
| 1. Acceso al nodo | root y el usuario `brltty` (ACL). **Ningún usuario humano** por estas reglas |
| 2. Creación de dispositivos virtuales | Los mismos actores |
| 3. Inyección de eventos | Los mismos actores (Tarea 015: llegan al dueño del grab) |
| 4. Recepción por el greeter | Sí (ruta `input_event`) |
| 5. Suplantación de entrada humana | Posible por root o por brltty |
| 6. Denegación de servicio | Posible por root o por brltty, con altas y bajas repetidas |

Restringir uinput no elimina otras vías de entrada sintética: root, el compositor y los protocolos virtuales de Wayland.

**Opciones para D-35 `[P]`:**

| Opción | Consecuencia |
|---|---|
| a) brltty como servicio de confianza (TCB) | Lo que ya implica su configuración: lee teclados, toma master e inyecta. Documentarlo |
| b) Endurecer brltty: retirar `cap_sys_admin` o el grupo `input` mediante un drop-in | Hay que demostrar que no rompe la asistencia `[VM]`. Supone modificar un paquete de terceros |
| c) No instalarlo por defecto; activarlo a petición | Reduce la TCB por defecto. **Impacto en accesibilidad:** el greeter tiene que ofrecer una forma accesible de activarlo |
| d) Modo de asistencia declarado en el greeter, con un indicador de entrada virtual | Transparente para el usuario; no reduce privilegios |

---

## 9. Corrección del modelo DRM y render

- **`renderD*`:** el modo es `{{GROUP_RENDER_MODE}}` (upstream `0666`); **el valor de Arch es desconocido**.
  - Un render node no permite modeset ni tener master (`drm-uapi.rst`, Tarea 014).
  - Con `0666`, cualquier proceso de usuario puede usar la GPU para renderizar o calcular. Es una decisión de diseño habitual en Linux.
  - El riesgo es la **superficie de ataque del driver** (fallos del kernel), no la presentación.
  - **RV-52 sigue propuesto**, sin calificarlo como vulnerabilidad.
- **`card*`:** `video` + `uaccess`.
- **Nuevo hecho:** cualquier proceso con `CAP_SYS_ADMIN`, brltty incluido, pasa `drm_master_check_perm` y puede obtener master cuando no hay otro (`SET_MASTER`) `[V-p Tarea 013 + V-p brltty]`.
  - Si el greeter ya tiene master, el resultado es `EBUSY`.
  - En un hueco sin master, brltty podría tomarlo; el greeter lo detectaría con su comprobación de master. Es un problema de disponibilidad, no de exposición.
  - **RV-44 debe ampliarse para incluir los procesos con `cap_sys_admin`.**

---

## 10. Matriz de contradicciones T013–T016

| Tema | Afirmación original | Evidencia posterior | Corrección | Estado |
|---|---|---|---|---|
| RV-43 / `uaccess` en teclados | T013: el usuario anterior pudo abrir el teclado vía `uaccess` | T016: upstream no da `uaccess` a teclados | Actores: root, `input`, servicios (brltty), joysticks | **Refutada** (en upstream) |
| RV-48 / hotplug | T015: root y usuario greeter | T016/T017 | root, `input`, brltty, joysticks | **Refutada** |
| RV-49 / hidraw | T014/T015: riesgo medio por la ACL de usuarios | T016: sin `uaccess` en teclados; T017: brltty solo en la lista braille | Riesgo limitado a root, servicios y terceros | **Debilitada** |
| RV-50 / uinput | T015: «solo root»; T016: desconocido | T017: root y el usuario `brltty` (ACL) | Añadir brltty | **Corregida** (upstream; Arch sin verificar) |
| RV-51 / mappings DRM | T015 | Sin evidencia nueva | — | **Desconocida** (sin cambios) |
| RV-52 / render | T016: `0666` por defecto | Valor de Arch desconocido | Sin cambio | **Desconocida** |
| RV-44 / master | T013: solo procesos que fueron master | T017: también `cap_sys_admin` (brltty) | Ampliar | **Debilitada** (el alcance era incompleto) |
| M-GRAB | T014/T015: exclusividad evdev | Sin cambios; excluye también a brltty de los eventos nuevos | — | Se mantiene (pendiente de prueba) |
| DRM master | Exclusividad detectable | Un proceso con `cap_sys_admin` puede adelantarse en un hueco → `EBUSY` detectable | — | Se mantiene |
| Alternativa E | Candidata | brltty forma parte de la TCB si está activo | Declararlo | Se mantiene |
| D-34 | D + A | Coincide con upstream | — | Sin cambios |
| D-35 | Grab ajeno o lista de servicios | brltty no hace grab (lee por el grupo `input`) e inyecta por uinput `[H]` | Nuevas opciones a–d | Reformulada |

El historial no se borra: las afirmaciones originales siguen en sus informes.

---

## 11. Matriz de garantías

| Propiedad | Upstream | Configuración Arch | Instalación real | Garantía del greeter | Limitación |
|---|---|---|---|---|---|
| Exclusividad evdev | — | — | — | Eventos nuevos solo al greeter (T014) | Kernel de Arch sin verificar |
| Confidencialidad frente a hidraw | Teclados sin `uaccess` `[V-p]` | Desconocida | `[VM]` | Ninguna | root, servicios, terceros |
| Integridad frente a uinput | Sin regla en systemd; ACL `brltty` `[V-p]` | Desconocida | `[VM]` | Ninguna | root, brltty |
| Carrera de hotplug | `input` sin `uaccess` | Desconocida | `[VM]` | Deshabilitación en `PENDING` | root, `input`, brltty |
| Estado de teclas | — | — | — | Ninguna | `EVIOCGKEY` por root, `input`, brltty |
| DRM master | — | — | — | Detección de `EBUSY` | `cap_sys_admin` en un hueco |
| Estado de presentación | — | — | — | `DRM_STATE_COMMITTED` verificado (T015) | Driver |
| Render nodes | `0666` por defecto | Desconocido | `[VM]` | No aplica | Superficie del driver |
| Fin de la sesión anterior | — | — | — | Detectable (`ListSessions`) | Asíncrono |
| Aislamiento entre procesos | — | — | — | **Ninguno** | — |

---

## 12. Decisiones pendientes

| ID | Estado |
|---|---|
| D-31 a D-33 | Sin cambios |
| D-34 | Recomendación técnica: política upstream + auditoría + riesgo residual `[P]` |
| D-35 | Nuevas opciones a–d (§8) `[P]` |
| **Nueva propuesta D-36** `[P]` | ¿brltty se instala por defecto en la imagen base y, en ese caso, entra en la TCB declarada? |
| **Autorización** `[P]` | Obtener por un medio aceptable el PKGBUILD de systemd y de brltty, o una copia de los archivos instalados en DEV-01 |

---

## 13. Riesgos propuestos (sin cambios formales)

| ID | Propuesta |
|---|---|
| RV-43 | Redacción de §6 |
| RV-44 | Añadir los procesos con `CAP_SYS_ADMIN` (brltty) |
| RV-48 | Redacción de §6 |
| RV-49 | Limitado a root, servicios, terceros y clasificación |
| RV-50 | Actores: root y el usuario `brltty` |
| RV-52 | Sigue propuesto |
| **RV-53** (nuevo; dimensión distinta: TCB) | Un servicio de asistencia con `input`, grupo `root` y `cap_sys_admin` puede leer teclados fuera del grab, consultar el estado de teclas, inyectar y tomar master. El modelo de amenazas debe declararlo como parte de la base de confianza o endurecerlo (D-35/D-36) |

---

## 14. Pruebas futuras

| ID | Resuelto en documentación | Sigue necesitando |
|---|---|---|
| T-UDEV-INPUT | Política upstream | **PKGBUILD + archivos instalados** (Arch de referencia), root de solo lectura (`stat`, `getfacl`, `getent group input`) |
| T-UDEV-HIDRAW | Upstream + brltty | Archivos instalados; **hardware** (teclado USB, FIDO, línea braille de la lista y otra que no esté en ella) |
| T-UDEV-UINPUT | brltty (ACL) | Archivos instalados; comprobar la ACL de `/dev/uinput` con y sin brltty |
| T-UDEV-DRM | Plantilla | **PKGBUILD** (modo de `renderD*`); `stat` de `card*`/`renderD*` |
| T-HOTPLUG-RACE | Actores | VM + root: un usuario fuera de `input` debe recibir `EACCES`; el usuario `brltty` debe poder abrir |
| T-INPUT-COVERAGE | — | Sin cambios |
| T-ACCESSIBILITY | — | **Hardware braille**; brltty con `cap_sys_admin` y sin ella (si se elige la opción b de D-35) |

Cada prueba distingue los resultados esperados: «`EACCES` frente a apertura correcta» y «ACL presente frente a ausente».

---

## 15. Veredicto: `EVIDENCIA PARCIAL; CONTRADICCIONES DOCUMENTALES RESUELTAS PARCIALMENTE`

**Resuelto:**
- Las reglas upstream de brltty: ACL `u:brltty`, ningún acceso humano, grupos y capacidades del servicio.
- Las contradicciones de RV-43, RV-48, RV-49 y RV-50 en el nivel upstream.

**No resuelto:**
- El PKGBUILD de systemd y el de brltty: bloqueados, el segundo también por los permisos de esta sesión.
- Si las reglas de Arch coinciden con las de upstream.
- Los archivos instalados.
- RV-51 y RV-52.

**Hallazgo que cambia el modelo:** brltty, si está activo, pertenece a la base de confianza (RV-53).

**Estado:**
- Fase 0 `IN_PROGRESS`.
- Todos los ADR `Propuesto`.
- Fase 1 sin autorización.
- RV-37 abierto.
- D-31 a D-35 pendientes.
- Riesgos sin cambios formales.
- Repositorio sin modificaciones.
- Ninguna prueba ejecutada.
