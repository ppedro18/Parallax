# Arquitectura del sistema

## 1. Modelo de capas

```
┌────────────────────────────────────────────────────────────────────┐
│ C6 Native Apps   Files · Terminal · Settings · TaskMgr · Observatory│
├────────────────────────────────────────────────────────────────────┤
│ C5 Desktop Exp.  parallax-shell (topbar, dock, launcher, widgets,  │
│                  notificaciones, control center) · lockscreen      │
├────────────────────────────────────────────────────────────────────┤
│ C4 Core          parallax-compositor (WM, input, render, outputs)  │
├────────────────────────────────────────────────────────────────────┤
│ C3 Intelligence  parallax-resourced (memoria, CPU, GPU, energía,   │
│                  ciclo de vida)                                     │
├────────────────────────────────────────────────────────────────────┤
│ C2 Sys Services  hardwared · updated · recoveryd · diagd ·         │
│                  settingsd · notifyd · secd (polkit agent)          │
├────────────────────────────────────────────────────────────────────┤
│ C1 Foundation    Linux · systemd/logind/oomd · udev · D-Bus · PAM · │
│                  polkit · Mesa · PipeWire · NetworkManager · BlueZ ·│
│                  UPower · power-profiles-daemon                     │
└────────────────────────────────────────────────────────────────────┘
C7 Distribution (transversal, tiempo de build): image-builder · repo · installer · update delivery · recovery image
```

### Regla de dependencias
- Una capa **llama** solo a capas inferiores.
- La comunicación **ascendente** es exclusivamente por señales/eventos (D-Bus signals, eventos Wayland).
- C6 no habla con C1 directamente cuando existe un servicio C2/C3 dueño del dato.
- Excepción documentada: apps pueden usar APIs estándar del ecosistema (XDG portals, PipeWire, NetworkManager D-Bus) cuando PARALLAX no añade valor envolviéndolas.

## 2. Inventario de procesos

| Proceso | Capa | Usuario | Unidad | Runtime | Crítico |
|---|---|---|---|---|---|
| `parallax-greeter` | C5 | `parallax-greeter` | `parallax-greeter.service` (sistema, vía greetd) | loop toolkit | sí |
| `parallax-compositor` | C4 | usuario | `parallax-session.target` (usuario) | calloop | sí |
| `parallax-shell` | C5 | usuario | `parallax-shell.service` (usuario) | loop toolkit + hilo tokio para D-Bus | sí |
| `parallax-lock` | C5 | usuario | lanzado por compositor (session-lock) | loop toolkit | sí |
| `parallax-settingsd` | C2 | usuario | `parallax-settingsd.service` | tokio | sí |
| `parallax-notifyd` | C2 | usuario | D-Bus activado | tokio | no |
| `parallax-resourced` | C3 | usuario | `parallax-resourced.service` | tokio | no |
| `parallax-hardwared` | C2 | `parallax-hw` (sistema) | `parallax-hardwared.service` | tokio | no |
| `parallax-updated` | C2 | root (endurecido) | `parallax-updated.service` | tokio | no |
| `parallax-recoveryd` | C2 | root (endurecido) | `parallax-recoveryd.service` | tokio | no |
| `parallax-diagd` | C2 | `parallax-diag` | D-Bus activado | tokio | no |
| `parallax-polkit-agent` | C2 | usuario | dentro de shell (ADR-0010) | — | sí |
| Apps C6 | C6 | usuario | `app-parallax-<id>@<n>.scope` | loop toolkit | no |

**Decisión clave:** el shell es *un* proceso con todos los elementos (topbar, dock, launcher, widgets, notificaciones UI, control center) para compartir caché de fuentes/iconos/GPU (ADR-0006). Widgets de terceros corren fuera de proceso.

## 3. Diagrama de dependencias (crates)

```
px-types ◄── px-ipc ◄── px-config ◄── px-telemetry
   ▲            ▲           ▲
   │            │           │
px-ui (toolkit wrapper) ────┘
   ▲
   ├── parallax-shell ── px-widgets (motor de widgets)
   ├── parallax-lock / parallax-greeter
   └── apps/* (files, terminal, settings, taskmgr, observatory, capture)

px-wm (layout, puro, sin E/S) ◄── parallax-compositor ── smithay
px-proto (XML → bindings de parallax-shell-v1, parallax-workspace-v1)
   ▲ usado por compositor y shell

px-dbus (interfaces zbus generadas de docs/architecture/dbus/*.xml)
   ▲ usado por servicios y clientes

services/*: hardwared, settingsd, notifyd, updated, recoveryd, diagd
intelligence/resourced
```
`px-wm` es una librería pura (sin Wayland ni E/S): árbol de layout + operaciones → permite pruebas deterministas y fuzzing del WM.

## 4. Estructura del repositorio

```
parallax-os/
├── AGENTS.md  README.md  Cargo.toml  Cargo.lock  rust-toolchain.toml  deny.toml
├── .cargo/config.toml          # source replacement → vendor/
├── .claude/agents/
├── docs/                       # Fase 0
├── crates/                     # librerías compartidas px-*
├── core/compositor/  core/wm/  core/proto/
├── intelligence/resourced/
├── services/{hardwared,settingsd,notifyd,updated,recoveryd,diagd}/
├── shell/{shell,widgets,lock,greeter}/
├── applications/{files,terminal,settings,taskmgr,observatory,capture}/
├── distribution/{packages,image,installer,repo,hardware-profiles}/
├── foundation/                 # unidades systemd, reglas udev, polkit, PAM, sysctl
├── vendor/                     # cargo vendor (offline)
└── tests/{integration,vm,recovery,perf}/
```
Se agrupa por capa porque el dueño, las reglas de privilegios y el ciclo de release difieren por capa.

## 5. Flujos críticos

### Arranque
`systemd` → `greetd` → `parallax-greeter` (PAM) → logind crea sesión → `systemd --user` → `parallax-session.target` → `parallax-settingsd` → `parallax-compositor` (toma DRM vía logind) → `parallax-shell` (cliente privilegiado vía socket heredado) → autostart de apps.

### Apertura de app desde el dock
Shell → `org.parallax.Session1.Launch(app_id)` (en compositor) → compositor genera token xdg-activation, lanza vía `systemd-run --user --scope` en `app.slice` → app se conecta, presenta toplevel con token → WM lo coloca según reglas → evento `toplevel_added` al shell → dock actualiza indicador → resourced recibe foco por `org.parallax.Session1.FocusChanged`.

### Caída del shell
systemd reinicia `parallax-shell` (`Restart=always`, `RestartSec=200ms`, límite 5/min). El compositor mantiene ventanas y layout; al reconectar, envía estado completo (`toplevel` snapshot). Si supera el límite → compositor muestra overlay mínimo de emergencia propio (reloj + lanzador de terminal + "reiniciar sesión").

### Caída del compositor
Las apps Wayland pierden conexión (limitación del protocolo). Mitigación: (a) compositor sin estado complejo propio de apps, (b) `px-wm` probado por fuzzing, (c) restauración de sesión: apps PARALLAX guardan estado y se relanzan; layout se persiste cada 30 s y al cambiar.

> **Nota de Tarea 010 (propuestas pendientes; el texto anterior sigue siendo el contrato vigente).** Las revisiones 6–10 proponen, sin aprobación:
> - **Greeter:** no es `parallax-greeter.service`; greetd lo ejecuta como `command` de `[default_session]` [V greetd(5)] (C-38).
> - **Compositor:** `parallax-compositor.service` con `PartOf=graphical-session.target`, deseado por `parallax-session.target` (C-34).
> - **Arranque:** greetd → comando de sesión propio (D-28) con exclusión `flock` (D-29) → `parallax-session.target` (`BindsTo=graphical-session.target`).
> - **Shell y roles:** los lanza y reinicia el compositor (intermediario + scope transitorio), no systemd (D-18, C-29).
> - **Apertura de apps:** crate común de lanzamiento, sin `Session1.Launch` (D-03, C-20).
> - **Caída del compositor (política X, D-24, recomendación no aprobada):** se propone que termine la sesión gráfica de PARALLAX y se vuelva al greeter; solo se detendría lo que PARALLAX lanza con `PartOf=graphical-session.target`. La propagación de la parada, el fin del comando de sesión y el retorno al greeter **no están demostrados** (escenarios y huecos, incluida la caída del comando de sesión: consolidación §4.3, RV-37, T-SESSION-FAIL). Servicios D-Bus, apps `DBusActivatable` de terceros y unidades de usuario ajenas pueden sobrevivir hasta que terminen la sesión logind o el user manager. La restauración ocurre en la sesión nueva.
>
> Trazabilidad completa en [`project-management/consolidacion-fase0.md`](../project-management/consolidacion-fase0.md).

## 6. Decisiones abiertas
Ver `docs/decisions/` — ADR en estado `Propuesto` requieren aprobación; ver resumen en `docs/roadmap/fase0-revision.md`.
