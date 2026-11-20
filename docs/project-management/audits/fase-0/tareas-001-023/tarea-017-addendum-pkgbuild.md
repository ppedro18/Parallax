# Tarea 017 — Addendum: lectura directa del empaquetado de Arch

- **Fuente:** clones git de `gitlab.archlinux.org/archlinux/packaging/packages/{systemd,brltty}`, URLs facilitadas por el usuario. El clon por SSH falló por permisos y se hizo por HTTPS.
- **Copia local:** el contenido está en `scratchpad/arch-pkg/`. Solo se leyó; no se ejecutó nada.
- **Commits leídos:**
  - systemd: tag `262.2-1`, commit `9bf08f852e3f…`, «upgpkg: 262.2-1: new upstream release», 2026-11-05.
  - brltty: tag `6.9.1-3`, commit `9c34ac958b9e…`, «upgpkg: 6.9.1-3 (ocaml-5.5.0-1 rebuild)».
- **Nivel de evidencia:** nivel 2–3 del encargo (configuración de compilación y contenido del paquete), lectura literal de los archivos `[V]`. **No** cubre los archivos instalados en el sistema de referencia ni el comportamiento en ejecución.

---

## 1. systemd 262.2-1

**Origen y parches**

- `source=` apunta a `git+https://github.com/systemd/systemd#tag=v${pkgver}?signed`, es decir, al tag **v262.2 firmado** `[V]`. Las reglas upstream leídas en la Tarea 016 corresponden a ese mismo tag.
- Hay un único parche que toca reglas: `0001-Use-Arch-Linux-device-access-groups.patch` `[V]`. Solo cambia `cdrom` por `optical`, `dialout` por `uucp` y `tape` por `storage` en `50-udev-default.rules.in`, `meson.build`, `meson_options.txt` y `sysusers.d/basic.conf.in`.
  - **No toca** `input`, `hidraw`, `drm`, `uinput` ni `70-uaccess`/`71-seat`/`73-seat-late`.
  - Por tanto, **las reglas de input, hidraw, DRM y uaccess del paquete Arch son las de upstream v262.2** `[V]`. Lo que se leyó en la Tarea 016 sigue siendo `[V-p]`, porque se leyó con una herramienta web.

**Opciones meson (`_meson_options` de `build()`)** `[V]`

| Opción | Valor upstream por defecto | Valor que fija Arch | Efecto |
|---|---|---|---|
| `group-render-mode` | `0666` (T016, `[V-p]`) | **No la fija**, así que se aplica el valor por defecto | `renderD*` con modo **0666** `[V]` + `[V-p]` |
| Opción de `uaccess` en render (`GROUP_RENDER_UACCESS`) | Desconocido (nombre de la opción sin localizar) | **No la fija** | Se aplica el valor por defecto upstream, que sigue sin conocerse |
| `acl` | Característica `auto` | No la fija. `arch-meson` activa las características `auto`, y `acl` está en `makedepends` y `depends` | `HAVE_ACL` probablemente activo `[H fuerte]` |
| **`default-kill-user-processes`** | `true` (según logind.conf(5), Tarea 012) | **`false`** | **En Arch, `KillUserProcesses=no` por defecto** `[V]` |

**Hallazgo nuevo que no estaba en el encargo.** Las Tareas 012 y 013 trataban `KillUserProcesses=yes` como valor por defecto. En Arch es **no**: al cerrar la sesión, el scope de la sesión se **abandona** en lugar de matarse (logind.conf(5), Tarea 012).

- En la propuesta, el compositor y las apps viven en `user@`, no en el scope de la sesión, así que no les afecta directamente.
- Pero la variante G-a de la Tarea 012 (compositor dentro del scope de sesión) **pierde su mecanismo de limpieza** salvo que PARALLAX fije `KillUserProcesses=yes`. Esa sería una decisión de despliegue `[P]`.

---

## 2. brltty 6.9.1-3

**Origen y parches**

- `source=` es el tarball `https://github.com/brltty/brltty/archive/refs/tags/BRLTTY-6.9.1.tar.gz`, con `sha512sums` fijado `[V]`. La ref `BRLTTY-6.9.1` leída en la Tarea 017 es **un tag**, y es el código fuente del paquete.
- Parches `[V]`:
  - `6.2-systemd_sysusers_groups`: en `brltty@.service` y en `sysusers`, `dialout` pasa a **`uucp`** y se **comenta** `pulse-access`. Se mantienen `root`, `audio` e **`input`**.
  - `6.8-lock-brltty-user`: `u brltty` pasa a `u! brltty`, es decir, el usuario queda bloqueado y sin login.
  - `6.4-x11_autostart`: cambia la ruta de autoarranque de X11; no tiene efecto en udev.
- **No hay parches sobre `Autostart/Udev/*.rules`.** Las reglas `uinput`, `hid` y `beeper` del paquete son las de upstream que se leyeron en la Tarea 017 `[V]` (contenido `[V-p]`).
- Instalación: `make install-systemd install-udev install-polkit` `[V]`.
  - `90-brltty-usb-generic.rules` se separa en el paquete **opcional** `brltty-udev-generic`.
  - `brltty-6.8-udev-tty-perms.rules` (`tty[0-9]*`, `GROUP="tty"`, `MODE="0660"`) figura en `source=`, pero **no se instala en el `package_brltty()` leído**. Su uso: desconocido `[H]`.
- `brltty.install` genera `/etc/brlapi.key` y muestra el aviso «Please add your user to the brlapi group» `[V]`.
  - El grupo `brlapi` da a los usuarios humanos acceso a la **API BrlAPI** del demonio.
  - Si esa API permite inyectar entrada, a través de la cual brltty usa uinput: `[H]` / `[VM]`. Es una vía que RV-50 no contemplaba.

**Lo que queda confirmado del servicio:**
- `User=brltty` (bloqueado).
- Grupos `uucp`, `root`, `audio`, `input` y `brlapi`.
- `AmbientCapabilities` `cap_sys_admin`, `cap_sys_tty_config` y `cap_mknod` (no los toca ningún parche).
- ACL `u:brltty` sobre `/dev/uinput` y sobre el hidraw de la lista braille.

---

## 3. Efecto sobre las conclusiones de la Tarea 017

| Punto | Antes | Ahora |
|---|---|---|
| PKGBUILD de systemd | No leído | **Leído** `[V]` |
| Modo de `renderD*` en Arch | Desconocido | **0666** (Arch no cambia el valor por defecto) `[V]` + `[V-p]` del valor por defecto |
| ¿Reglas input/hidraw/DRM de Arch iguales a las de upstream? | Desconocido | **Sí** `[V]` (parche revisado) |
| Reglas de brltty en Arch iguales a las de upstream | Desconocido | **Sí** (sin parches en `Udev/`) `[V]` |
| Grupos de brltty | `dialout`, `pulse-access`… | `uucp`, sin `pulse-access`; **`input` y `root` se mantienen** |
| RV-53 (brltty en la base de confianza, TCB) | Propuesto | **Reforzado**: el paquete conserva `input` y `cap_sys_admin` |
| RV-50 | root + `brltty` | Añadir la vía **BrlAPI** para miembros del grupo `brlapi` `[H]` |
| RV-52 | Desconocido | `renderD*` 0666 en Arch. **Sigue como propuesto**: es una decisión de diseño habitual en Linux, no una vulnerabilidad confirmada |
| `KillUserProcesses` | Se suponía `yes` | **`no` en Arch** `[V]`. Afecta a la limpieza (L2/G-a) y a la Tarea 012 |

**Veredicto de la Tarea 017, revisado:** `FUENTES DE EMPAQUETADO VERIFICADAS; INSTALACIÓN PENDIENTE`.

Matices del veredicto:
- El valor por defecto de la opción de `uaccess` en render sigue sin conocerse.
- Faltan los archivos instalados en el sistema de referencia, incluidas las reglas locales de `/etc/udev/rules.d`.
- Falta comprobar el comportamiento de BrlAPI.

**Sin cambios:** Fase 0 `IN_PROGRESS`, todos los ADR `Propuesto`, Fase 1 sin autorización, RV-37 abierto, D-31 a D-35 pendientes, ningún riesgo modificado, repositorio sin modificaciones y ninguna prueba ejecutada.
