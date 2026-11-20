# Manifiesto de fuentes externas — Fase 0

Este manifiesto registra las fuentes de terceros que se consultaron en las Tareas 012–023. **No se copió ningún árbol de código externo al repositorio.**

## 1. Repositorios consultados en local (clones parciales)

Todas las consultas son de 2026-11-19.

### Código upstream

Clones de tipo `--depth 1 --filter=blob:none --sparse`. Su copia de trabajo está en el scratchpad de la sesión, en `scratchpad/src/`.

| Proyecto | URL | Tag | Commit leído | Rutas consultadas | Licencia | Tareas |
|---|---|---|---|---|---|---|
| Linux | `https://github.com/torvalds/linux.git` | `v7.3` | `c03c1a14507dfaf8b856c2608f9ac216021fc222` | `drivers/gpu/drm/drm_{file,auth,ioctl,debugfs,gem,…}.c`, `drivers/input/{evdev,input}.c`, `drivers/input/misc/uinput.c`, `fs/{file_table,file,exec}.c`, `net/core/scm.c`, `net/unix/af_unix.c`, `drivers/tty/tty_io.c` (clonado, sin revisar) | GPL-2.0 | 020, 021, 022 |
| systemd | `https://github.com/systemd/systemd.git` | `v262.2` | `63511d7283d7d0e2a2fc28075481afaf1a7be8f5` | `src/login/*` (`logind-session.c`, `logind-session-device.c`, `logind-session-dbus.c`, `logind-user.c`, `logind-core.c`, `logind.c`, `logind-dbus.c`, `logind-gperf.gperf`, `logind-session.h`) | LGPL-2.1-or-later | 020 |
| BRLTTY | `https://github.com/brltty/brltty.git` | `BRLTTY-6.9.1` | `4fddd533bf6023820266879203839db511808d63` | `Programs/{brlapi_server.c,scr.c,system_linux.c,cmd_clipboard.c}`, `Drivers/Screen/Linux/screen.c` | LGPL-2.1-or-later | 020 |

### Empaquetado de Arch Linux

Los archivos que importan para las conclusiones están copiados en `../evidencias/arch-pkg/`.

| Proyecto | URL | Tag | Commit | Tareas |
|---|---|---|---|---|
| Arch systemd | `https://gitlab.archlinux.org/archlinux/packaging/packages/systemd.git` | `262.2-1` | `9bf08f852e3fb165b385c9bc07de161c24530945` | 017 (addendum) |
| Arch brltty | `https://gitlab.archlinux.org/archlinux/packaging/packages/brltty.git` | `6.9.1-3` | `9c34ac958b9e7081ee9787aec3cd2f24efee677d` | 017 (addendum), 018 |

Licencias del empaquetado de Arch: el `LICENSE` y el `REUSE.toml` de cada paquete, copiados a `evidencias/`.

## 2. Fuentes leídas por herramienta web (evidencia `[V-p]`)

Estas fuentes se leyeron a través de una herramienta web, sin copia local:

- Tareas 012–019: código de Linux v7.3, systemd v262.2 y BRLTTY-6.9.1 en `raw.githubusercontent.com`.
- Páginas de manual de freedesktop.org (`latest`).
- greetd 0.10.3 en el mirror de GitHub.
- Ficha del paquete en `archlinux.org`.

El detalle de cada lectura y su nivel de evidencia está en cada informe, en `../tareas-001-023/`. **No se conserva ninguna copia** de esas páginas.

## 3. Cómo reproducir las consultas locales

```bash
git clone --depth 1 --filter=blob:none --sparse --branch v7.3 https://github.com/torvalds/linux.git linux
git -C linux sparse-checkout set --no-cone '/drivers/gpu/drm/drm_*.c' '/drivers/input/evdev.c' '/drivers/input/input.c' '/drivers/input/misc/uinput.c' '/fs/file_table.c' '/fs/file.c' '/fs/exec.c' '/net/core/scm.c' '/net/unix/af_unix.c'
```

```bash
git clone --depth 1 --filter=blob:none --sparse --branch v262.2 https://github.com/systemd/systemd.git systemd
git -C systemd sparse-checkout set --no-cone '/src/login/'
```

```bash
git clone --depth 1 --filter=blob:none --sparse --branch BRLTTY-6.9.1 https://github.com/brltty/brltty.git brltty
git -C brltty sparse-checkout set --no-cone '/Programs/' '/Drivers/Screen/Linux/' '/Headers/'
```

```bash
git clone https://gitlab.archlinux.org/archlinux/packaging/packages/systemd.git arch-systemd
git -C arch-systemd checkout 262.2-1
```

```bash
git clone https://gitlab.archlinux.org/archlinux/packaging/packages/brltty.git arch-brltty
git -C arch-brltty checkout 6.9.1-3
```

Después, comprobar que `git rev-parse HEAD` coincide con el commit de la tabla del §1.

## 4. Archivos de fuentes que se podrían incorporar más adelante (propuesta, no aplicada)

Ninguno es indispensable: las conclusiones citan función y línea en el commit fijado, y se pueden reproducir con el §3. Si el propietario quiere independencia total de la red, se podrían añadir extractos de:

- `drm_auth.c`
- `drm_ioctl.c`
- `evdev.c`
- `logind-user.c`
- `logind-session.c`
- `screen.c`

Antes habría que revisar la licencia (GPL-2.0 / LGPL-2.1) según ADR-0008 `[P]`.
