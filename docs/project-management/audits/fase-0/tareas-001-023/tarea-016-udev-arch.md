# Tarea 016 — Verificación de políticas udev y permisos de dispositivos en Arch

- **Fecha:** 2026-11-19
- **Estado del proyecto:** Fase 0 `IN_PROGRESS` · todos los ADR `Propuesto` · Fase 1 sin autorización.
- **Naturaleza:** solo lectura. Es una propuesta, no una decisión. El informe está en el scratchpad; el repositorio no se ha tocado.
- **Marcas:**
  - `[V]` fuente primaria suficiente
  - `[V-p]` parcial
  - `[R]` repositorio
  - `[H]` inferencia
  - `[VM]` requiere prueba
  - `[P]` propietario

---

## 1. Resumen ejecutivo

**Veredicto: `POLÍTICA NO VERIFICABLE CON LA EVIDENCIA DISPONIBLE`** (§14).

**Qué evidencia hay:**
- Se leyeron las reglas **upstream** de systemd v262.2 (`rules.d/`) y `meson_options.txt`, literales `[V-p]`.
- Se leyó el **listado de archivos** del paquete `systemd 262.2-1` de Arch, pero solo en resumen `[V-p]`, y el listado del paquete `brltty 6.9.1-3` `[V-p]`.

**Qué no se pudo leer:**
- El PKGBUILD de systemd en Arch, que fija los valores de meson: estaba bloqueado.
- El contenido de las reglas de brltty.
- Los archivos realmente instalados en el sistema de referencia: **no hay acceso a ninguna máquina Arch**.

**Las reglas upstream contradicen premisas de las Tareas 013–015.** Estas contradicciones obligan a revisar el **modelo de amenazas**, no el mecanismo del greeter:

1. **Teclados y ratones** (`input/event*`) **no reciben `uaccess`** en upstream: solo `GROUP="input"`. Las únicas entradas con `uaccess` son los joysticks.
   - Por tanto, el usuario de la sesión activa **no** obtiene ACL sobre teclados ni ratones.
   - La premisa de la Tarea 013 (RV-43: «un proceso del usuario anterior pudo abrir el teclado gracias a `uaccess`») **no se sostiene** con las reglas upstream. Solo pueden hacerlo root y los miembros del grupo `input`.
2. **Hotplug.** La Tarea 015 decía «root y usuario greeter». Con las reglas upstream es **root y grupo `input`**, salvo en joysticks, que dan acceso al usuario activo.
3. **hidraw.** Upstream no fija grupo ni modo. Solo concede `uaccess` a controladores AV, carteras hardware, ratones 3D, tokens de seguridad y lectores de tarjetas. **El hidraw de los teclados no recibe `uaccess`.** La opción D de D-34 coincide en esencia con el comportamiento upstream.
4. **`/dev/uinput`.** systemd no tiene ninguna regla para él. Pero **brltty instala `90-brltty-uinput.rules`**, cuyo contenido no se leyó. La afirmación «solo root» de la Tarea 015 **no está verificada** y puede ser falsa con brltty instalado.
5. **DRM.**
   - `card*`: `GROUP="video"` + `uaccess`.
   - `renderD*`: `GROUP="render"` y `MODE` = `{{GROUP_RENDER_MODE}}`, que **por defecto en meson es `0666`**, es decir, accesible a todos los usuarios. El valor que usa Arch es desconocido.

---

## 2. Método y límites de la evidencia

| Nivel | Fuente | Estado |
|---|---|---|
| 1. Upstream | `systemd/systemd` tag `v262.2`: `rules.d/50-udev-default.rules.in`, `70-uaccess.rules.in`, `71-seat.rules.in`, `73-seat-late.rules.in`, `60-fido-id.rules`, `70-joystick.rules`, `meson_options.txt` | **Leído** `[V-p]`, por herramienta web. Las plantillas `{{…}}` dependen de los valores que fije cada distribución |
| 2. Paquete Arch | `archlinux.org/packages/core/x86_64/systemd/files/` (262.2-1) | Listado **resumido** a partir del carácter 39 401 `[V-p resumen]`. Incluye `50-udev-default`, `60-evdev`, `60-fido-id`, `60-persistent-hidraw`, `60-persistent-input`, `70-joystick`, `70-uaccess`, `71-seat`, `73-seat-late` y `99-systemd` |
| 2. Paquete Arch | PKGBUILD de systemd (opciones meson) | **No leído** (Anubis) |
| 2. Paquete Arch | `brltty 6.9.1-3`: `90-brltty-{beeper,hid,uinput,usb-customized}.rules`, `brltty@.service`, `sysusers.d/brltty.conf`, reglas de polkit | Solo el listado de archivos `[V-p]`; **contenido no leído** (404 en upstream) |
| 3. Instalado | `/usr/lib/udev/rules.d`, `/etc/udev/rules.d` en DEV-01 o en una VM | **Sin acceso** |
| 4. Tiempo de ejecución | `getfacl`, `stat`, `udevadm info` | No ejecutado (prohibido) |
| 5. Validado | Pruebas | Ninguna |

**Valores por defecto del kernel y de udev** (modo de devtmpfs; modo 0600 o 0660 cuando una regla fija `GROUP` sin `MODE`): **no leídos** en esta tarea → `[H]`.

---

## 3. Inventario de reglas (upstream v262.2)

| Archivo | Origen | Subsistema | Condición | Acción | Evidencia |
|---|---|---|---|---|---|
| 50-udev-default | systemd | input | `SUBSYSTEM=="input"` | `GROUP="input"` | `[V-p]` |
| 50-udev-default | systemd | input | `KERNEL=="js[0-9]*"` | `MODE="0664"` | `[V-p]` |
| 50-udev-default | systemd | drm | `KERNEL!="renderD*"` | `GROUP="video"` | `[V-p]` |
| 50-udev-default | systemd | drm | `KERNEL=="renderD*"` | `GROUP="render"`, `MODE="{{GROUP_RENDER_MODE}}"` (meson por defecto `0666`) | `[V-p]` |
| 50-udev-default | systemd | hidraw | `SUBSYSTEM=="hidraw"` | `IMPORT{builtin}="hwdb"`; **sin GROUP ni MODE** | `[V-p]` |
| 60-fido-id | systemd | hidraw | `SUBSYSTEM=="hidraw"` | `IMPORT{program}="fido_id"` → `ID_SECURITY_TOKEN` | `[V-p]` |
| 70-joystick | systemd | input | `ID_INPUT_JOYSTICK` y `event*` | hwdb | `[V-p]` |
| 70-uaccess | systemd | drm | `KERNEL=="card*"` | `TAG+="uaccess"` | `[V-p]` |
| 70-uaccess | systemd | drm/accel/kfd | `{% if GROUP_RENDER_UACCESS %}` `renderD*` | `uaccess`, `xaccess-render` | `[V-p]`; valor en Arch desconocido |
| 70-uaccess | systemd | input | `ENV{ID_INPUT_JOYSTICK}=="?*"` | `uaccess` | `[V-p]` |
| 70-uaccess | systemd | hidraw | `ID_AV_PRODUCTION_CONTROLLER`, `ID_HARDWARE_WALLET`, `ID_INPUT_3D_MOUSE` | `uaccess` | `[V-p]` |
| 70-uaccess | systemd | cualquiera | `ID_SECURITY_TOKEN`, `ID_SMARTCARD_READER` | `uaccess` | `[V-p]` |
| 70-uaccess | systemd | sound, v4l, dvb, media, rfkill, udmabuf | — | `uaccess` | `[V-p]` |
| 71-seat | systemd | — | `TAG=="uaccess"` (salvo sound) | `TAG+="seat"` | `[V-p]` |
| 71-seat | systemd | input, hidraw | `input*`, `hidraw*` | `TAG+="seat"` | `[V-p]` |
| 71-seat | systemd | drm | `card[0-9]*` | `seat`, `master-of-seat` | `[V-p]` |
| 73-seat-late | systemd | — | `TAG=="uaccess\|xaccess-*"` y `MAJOR` | `RUN{builtin}+="uaccess"` (si `HAVE_ACL`) | `[V-p]` |
| 90-brltty-uinput | brltty (Arch) | uinput `[H]` | ? | ? | **Contenido no leído** |
| 90-brltty-hid | brltty (Arch) | hidraw `[H]` | ? | ? | **Contenido no leído** |
| `/etc/udev/rules.d/*` | Local | — | — | — | **Sin acceso** |

**Orden:**
- udev aplica los archivos por orden léxico; las asignaciones posteriores sobrescriben `GROUP`/`MODE` `[H]`.
- Las etiquetas `TAG+=` se acumulan.
- La ACL se aplica en `73-seat-late` por medio del builtin `uaccess`, que da la ACL al **usuario de la sesión activa del seat** `[H; builtin no leído]`.
- Las reglas `90-brltty-*` se ejecutan **después** de las de systemd y pueden cambiar grupo y modo `[H]`.

---

## 4. Input (`/dev/input/event*`)

| Momento | Propietario | Grupo | Modo | ACL | Pueden abrir |
|---|---|---|---|---|---|
| Recién creado (devtmpfs) | root `[H]` | root `[H]` | 0600 `[H]` | — | root |
| Tras las reglas udev | root | **input** `[V-p upstream]` | 0660 `[H]` | **Ninguna** (teclado y ratón); **joystick: usuario activo** | root, grupo `input` (+ usuario activo en joysticks) |
| Sesión del greeter activa | Igual | Igual | Igual | Joystick: usuario greeter | root, `input` |
| Cambio de sesión | Igual | Igual | Igual | Joystick: se mueve al nuevo usuario activo `[H]` | Igual |
| Sesión anterior finalizada | Igual | Igual | Igual | — | Igual |

**Respuestas:**
1. Quién puede abrir un nodo recién creado: root (y `input` tras udev).
2. Permisos de devtmpfs: `[H]`.
3. Propietario, grupo y modo los fija `50-udev-default`.
4. `uaccess` no se aplica a teclados ni ratones.
5. Las ACL de joysticks se gestionan con el builtin `uaccess` `[H]`.
6. **Pertenecer a `input` da acceso con independencia de la sesión activa** `[V-p + H]`.
7. Hotplug: la misma política.
8. Eventos de udev fuera de orden o perdidos: no estudiado (fuente no leída) `[VM]`.
9. **Un fd ya abierto sigue siendo utilizable** tras retirar una ACL: las ACL se comprueban en `open()` `[H, semántica VFS]`.
10. **Los dispositivos no tienen la misma política:** el joystick es distinto.

---

## 5. hidraw

1. Arch concede `uaccess` a hidraw solo por clase: AV, cartera hardware, ratón 3D, `ID_SECURITY_TOKEN` y lector de tarjetas `[V-p upstream]`. En el paquete Arch no está verificado.
2. Propiedades que intervienen: `ID_SECURITY_TOKEN` (de `fido_id`), `ID_HARDWARE_WALLET`, `ID_AV_PRODUCTION_CONTROLLER`, `ID_INPUT_3D_MOUSE` (hwdb) y `ID_SMARTCARD_READER`.
3. **Teclados y ratones:** sin `uaccess` → root:root `[H]`, modo `[H]`.
4. No depende de `ID_INPUT_KEYBOARD`: la política es **una lista permitida**, no una lista denegada.
5. Prioridad: `90-brltty-hid.rules` puede ampliar el acceso, pero su contenido no se leyó. Reglas de terceros como steam-devices, OpenRGB o VIA/QMK pueden ampliarlo también `[H]`.
6. Paquetes de asistencia: brltty sí instala reglas propias.
7. La revocación de logind (`TakeDevice`) es independiente de la ACL: solo actúa sobre los fds de logind (Tarea 013).
8. Un fd abierto directamente se conserva tras perder la ACL `[H]`.
9. Restringir hidraw por clase **ya es** el modelo upstream.

**Clasificación errónea:** `fido_id` analiza el descriptor HID. Un token FIDO que no lo declare no recibe `uaccess`; un dispositivo que lo declare indebidamente sí lo recibe `[H]`. **No se asume** que la clasificación sea completa.

### D-34 (fichas; decide el propietario `[P]`)

| | A. Riesgo residual | B. Restringir hidraw a usuarios no privilegiados | C. Solo logind | D. Por clase con excepciones |
|---|---|---|---|---|
| Cubre | Nada nuevo | Usuarios no root (en upstream ya ocurre salvo en las clases permitidas) | fds de logind | Igual que upstream + excepciones explícitas |
| No cubre | Root, `input` (no aplica a hidraw), terceros | Root; reglas de terceros | fds directos | Root; clasificación errónea; reglas de terceros |
| FIDO/U2F | — | **Se rompe** si se quita `ID_SECURITY_TOKEN` | — | Se conserva |
| Braille | — | brltty corre como root `[H]` | — | Depende de `90-brltty-hid` |
| Configuración de teclados | — | Necesitan reglas propias | — | Excepción explícita |
| Mantenimiento | Nulo | Medio | Nulo | Bajo: coincide con upstream |
| Pruebas | — | T-UDEV-HIDRAW | — | T-UDEV-HIDRAW |

**Propuesta `[P]`:** la opción D ≈ **no añadir reglas propias**. Bastaría con mantener la política upstream y **auditar las reglas de terceros instaladas**, lo que convierte la decisión en un requisito de despliegue, más A para lo que queda. Está **sujeta a verificar** el paquete y el sistema instalado.

---

## 6. uinput

1–3. **systemd v262.2 no contiene ninguna regla para `uinput`** `[V-p]`. Sin reglas, el nodo misc queda con los valores por defecto del kernel (root, modo `[H]`).
4. Paquetes que necesitan acceso: brltty (`90-brltty-uinput.rules`); de forma típica, Steam Input u otras herramientas `[H]`.
5. **brltty en Arch instala una regla de uinput** `[V-p listado]`. Su contenido y el grupo que concede: desconocidos.
6. Puede haber reglas locales o de terceros: `[VM]`.
7. **¿Puede un usuario normal crear un teclado virtual?** Solo si alguna regla le da acceso: si pertenece al grupo que conceda brltty o si existe otra regla de terceros. **No verificable** con lo leído.
8. Controles: el modo y la ACL de `/dev/uinput`. **No existe** ningún control del kernel sobre qué capacidades declara el dispositivo virtual `[H]`.

**Dimensiones de RV-50 (no se mezclan):**

| Dimensión | Efecto |
|---|---|
| Confidencialidad | Sin efecto |
| Integridad | Inyección en el greeter |
| Disponibilidad | Altas y bajas repetidas → `PENDING` |
| Suplantación | Eventos con aspecto humano |

Restringir `/dev/uinput` **no** elimina todas las vías de entrada sintética. Siguen abiertas:
- root;
- el propio compositor;
- otros clientes Wayland mediante protocolos virtuales, si el compositor del greeter los expone.

---

## 7. DRM

| Nodo | Grupo | Modo | `uaccess` | Pueden abrir |
|---|---|---|---|---|
| `card*` | video | `[H]` | **Sí** (usuario activo) | root, `video`, usuario activo |
| `renderD*` | render | `{{GROUP_RENDER_MODE}}`; **meson por defecto `0666`** | Solo si `GROUP_RENDER_UACCESS` | Con `0666`: **cualquier usuario** |

- `card*` es el nodo primario (modeset y master). `renderD*` no tiene modeset ni master (`drm-uapi.rst`, Tarea 014).
- **Acceso al nodo no equivale a DRM master.**
- Durante la sesión del greeter pueden abrir `card*`: root, `video` y el usuario greeter. El usuario anterior pierde la ACL, pero **conserva los fds ya abiertos** `[H]`.
- Como el primer opener se vuelve master (RV-44), cualquier miembro de `video` podría tomar master aprovechando un hueco.
- La ACL **no** da exclusividad de presentación: esa garantía la sigue dando el master del greeter con su detección de `EBUSY`.

---

## 8. Revisión de las afirmaciones de la Tarea 015

| Afirmación | Resultado | Evidencia | Corrección |
|---|---|---|---|
| El grupo `input` vacío evita el acceso general | **Parcialmente cierto (upstream)**: `input` es la **única** vía no root para teclados y ratones | `50-udev-default` `[V-p]` | Verificar el paquete, el sistema instalado y la pertenencia real al grupo `[VM]` |
| `uaccess` limita el acceso a la sesión activa | **Falso para teclados y ratones**: no reciben `uaccess`. Cierto para joysticks, `card*` y hidraw de las clases permitidas | `70-uaccess` `[V-p]` | Reescribir |
| El hotplug queda reducido a root y usuario greeter | **Falso:** es root y grupo `input` (joysticks: + usuario activo) | Ídem | Corregir RV-48 |
| El hidraw de teclados no es accesible a otros usuarios | **Probable en upstream** (no hay `uaccess`; modo y grupo `[H]`) | `50-udev-default`, `70-uaccess` `[V-p]` | Reglas de terceros y brltty sin leer |
| FIDO conserva el acceso con la política D | **Probable en upstream** (`ID_SECURITY_TOKEN`) | `[V-p]` | Clasificación por `fido_id` no exhaustiva |
| brltty no se ve afectado | **No verificable**: instala reglas propias de hid y uinput | Listado `[V-p]` | Leer las reglas |
| `/dev/uinput` solo es accesible por root | **No verificado**; systemd no tiene regla, brltty sí | `[V-p]` | Leer la regla de brltty |
| DRM queda bajo control exclusivo del greeter | **No lo da la ACL**: `video` y el usuario activo pueden abrir `card*`; la exclusividad la da el master | `[V-p]` | Mantener la redacción «por master», no «por ACL» |

**Corrección retroactiva a la Tarea 013 (RV-43):** el escenario «un proceso del usuario anterior conserva un fd de teclado abierto gracias a `uaccess`» **no se apoya en las reglas upstream**. RV-43 queda acotado a **root**, al **grupo `input`** y a **joysticks**. M-GRAB sigue haciendo falta frente a esos actores y frente a fallos de revocación.

---

## 9. Impacto en D-34 y D-35

- **D-34:** la opción D coincide con upstream. La decisión real es si se **exige una auditoría** de las reglas de terceros (brltty, juegos, configuración de teclados) como requisito de despliegue `[P]`.
- **D-35:** depende del contenido de `90-brltty-uinput.rules`.
  - Si da acceso a un grupo `brltty` (creado por `sysusers.d/brltty.conf` `[H]`), entonces miembros de ese grupo pueden inyectar entrada.
  - Opciones `[P]`:
    - aceptarlo como **entrada de asistencia de confianza**;
    - exigir que nadie más que el servicio pertenezca al grupo;
    - pedir que el greeter muestre la presencia de dispositivos virtuales.

---

## 10. Impacto en riesgos (propuestas, no aplicadas)

| Riesgo | Cambio propuesto |
|---|---|
| RV-48 | Actores en la ventana: **root y grupo `input`**, y el usuario activo en joysticks (no «usuario greeter» en general) |
| RV-49 | Upstream no concede `uaccess` al hidraw de teclados; el riesgo pasa a **root, reglas de terceros y clasificación errónea** |
| RV-50 | Añadir la dependencia de `90-brltty-uinput.rules` y de reglas de terceros |
| RV-51 | Sin cambios (no depende de udev) |
| RV-43 | Acotar a root, `input` y joysticks |
| **RV-52** (nuevo) | `renderD*` con `0666` por defecto en meson: cualquier usuario tiene acceso a la GPU. **No** da modeset, pero sí cómputo y exposición del driver. Es una dimensión distinta (superficie de ataque del driver y no presentación) `[H]` |

---

## 11. Matriz de garantías y limitaciones

| Propiedad | Política upstream | Arch (paquete) | Arch (instalado) | Ejecución |
|---|---|---|---|---|
| Teclado y ratón legibles solo por root e `input` | `[V-p]` | `[V-p resumen]` (archivo presente) | `[VM]` | `[VM]` |
| hidraw de teclado sin `uaccess` | `[V-p]` | Igual | `[VM]` | `[VM]` |
| FIDO con `uaccess` | `[V-p]` | Igual | `[VM]` | `[VM]` |
| `uinput` solo root | **Sin regla** en systemd | **brltty añade una regla (contenido desconocido)** | `[VM]` | `[VM]` |
| `card*`: video + usuario activo | `[V-p]` | Igual | `[VM]` | `[VM]` |
| Modo de `renderD*` | Plantilla, `0666` por defecto | **Desconocido** (PKGBUILD) | `[VM]` | `[VM]` |
| Los fds abiertos sobreviven a la retirada de la ACL | `[H]` | — | — | `[VM]` |

---

## 12. Pruebas pendientes (no autorizadas)

| ID | Hipótesis | Entorno | VM | HW | Root | Procedimiento | Aprobado si | Cambia el diseño si |
|---|---|---|---|---|---|---|---|---|
| **T-UDEV-INPUT** | event* = `root:input`, sin ACL salvo joystick | Arch de referencia | Sí | Teclado, ratón, joystick | Sí (lectura) | `stat`, `getfacl`, `udevadm info` antes y después de cambiar de sesión; `getent group input` | Coincide con §4 | Hay `uaccess` en teclados o el grupo `input` tiene miembros |
| **T-UDEV-HIDRAW** | Sin `uaccess` en hidraw de teclado; FIDO sí | Ídem | Passthrough | **Sí** (teclado, FIDO, braille) | Sí | Ídem + leer `90-brltty-hid.rules` | Coincide con §5 | El hidraw del teclado es accesible a usuarios |
| **T-UDEV-UINPUT** | Modo y grupo reales de `/dev/uinput` | Ídem, con y sin brltty | Sí | No | Sí | Leer `90-brltty-uinput.rules`; `stat`/`getfacl` | Solo root o un grupo de servicio sin usuarios | Usuarios con acceso |
| **T-UDEV-DRM** | `card*` `video` + ACL; modo de `renderD*` | Ídem | Sí | GPU real recomendable | Sí | Ídem | Coincide | `renderD*` 0666 → evaluar RV-52 |
| T-HOTPLUG-RACE | Ventana acotada a root e `input` | Ídem | Sí | No | Sí | Hotplug virtual; intento de apertura por un usuario sin grupo | Recibe `EACCES` | Lo consigue abrir |
| T-INPUT-COVERAGE | Igual que T015 | — | — | — | — | — | — | — |
| T-ACCESSIBILITY | brltty funcional con la política | Ídem | Sí | **Braille real** | Sí | — | Funciona | Se rompe |

**Prerrequisito sin VM:** leer el PKGBUILD de systemd y las reglas de brltty desde una fuente accesible (repositorio de empaquetado o tarball fuente), o pedir una copia de los archivos instalados en DEV-01. Esto requiere **autorización explícita** del propietario.

---

## 13. Recomendaciones técnicas (propuesta)

1. **Corregir el modelo de amenazas de las Tareas 013–015.** El acceso de usuarios no privilegiados a teclados y ratones depende del grupo `input`, no de `uaccess`.
2. **Requisito de despliegue `[P]`:**
   - grupo `input` sin usuarios humanos;
   - auditoría de las reglas de terceros sobre `input`, `hidraw` y `uinput`.
3. **Mantener M-GRAB y la comprobación de master.** Siguen siendo necesarios frente a root, `input`, `video`, joysticks y fallos de revocación.
4. **No añadir reglas udev propias** mientras la política upstream cubra el caso.
5. Prioridad siguiente:
   - leer `90-brltty-uinput.rules`, `90-brltty-hid.rules` y el PKGBUILD de systemd;
   - después, T-UDEV-INPUT y T-UDEV-UINPUT.

---

## 14. Veredicto final: `POLÍTICA NO VERIFICABLE CON LA EVIDENCIA DISPONIBLE`

**Justificación:**
- No hay acceso a los archivos instalados ni al PKGBUILD. Las reglas de brltty no se leyeron.
- Solo se conoce la política **upstream** y la presencia de archivos en el paquete, por resumen.
- La política upstream **contradice** premisas anteriores (`uaccess` en teclados, actores del hotplug, «uinput solo root»). Hay que revisar los textos de las Tareas 013–015 antes de cualquier decisión.
- Las contradicciones **no invalidan** el mecanismo del greeter: el grab y el master siguen siendo necesarios.

**Estado:**
- Fase 0 `IN_PROGRESS`.
- Todos los ADR `Propuesto`.
- Fase 1 sin autorización.
- RV-37 abierto.
- D-31 a D-35 pendientes del propietario.
- Repositorio sin modificaciones.
- Ninguna prueba ejecutada.
