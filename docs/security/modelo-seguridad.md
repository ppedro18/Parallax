# Modelo de seguridad

## Fronteras de confianza
```
[ Kernel ]─────────────────────────────────────────────── TCB
[ systemd, logind, polkit, PAM, D-Bus daemon ] ────────── TCB del SO
[ updated, recoveryd (root endurecido) ] ───────────────── privilegiados, superficie mínima
[ hardwared, diagd (usuarios dedicados) ] ──────────────── sistema sin root
[ compositor, shell, settingsd, resourced (usuario) ] ──── sesión de usuario, confiables para su usuario
[ apps PARALLAX ] ───────────────────────────────────────── usuario
[ apps de terceros / Xwayland / widgets externos ] ──────── no confiables
```

## Amenazas principales (STRIDE resumido)
| Componente | Amenaza | Mitigación |
|---|---|---|
| Compositor | App lee pantalla/teclado de otras (keylogging, screen scraping) | Wayland aísla por diseño; protocolos privilegiados solo para clientes autorizados; Xwayland aislado en su propio servidor (apps X ven solo apps X) |
| Shell (`parallax-shell-v1`) | Cliente malicioso se hace pasar por shell | Socket del shell pasado por fd heredado (`WAYLAND_SOCKET`) desde el compositor al lanzarlo; ningún otro cliente puede enlazar la global |
| updated | Paquete malicioso | Firmas pacman obligatorias con llavero PARALLAX; metadatos firmados con expiración (anti-rollback / freeze) |
| polkit | Elevación indiscriminada | Acciones granulares, `auth_admin_keep` máx 5 min, nunca `yes` para usuarios activos salvo acciones inocuas |
| Lock | Bypass por caída del cliente | `ext-session-lock-v1`: compositor bloquea si el cliente muere |
| Widgets externos | Código arbitrario en shell | Fuera de proceso, árbol declarativo validado, permisos en manifiesto |
| Thumbnailer | Parser explotable (imágenes/PDF) | Subproceso con seccomp, sin red, sin `$HOME` escritura (Landlock) |
| D-Bus | Llamadas no autorizadas | Políticas `busconfig` restrictivas + polkit por método + verificación de `peer credentials` |

## Acciones polkit (iniciales)
`org.parallax.update.apply`, `.update.rollback`, `org.parallax.recovery.snapshot`, `.recovery.restore`, `org.parallax.hardware.install-driver`, `org.parallax.files.change-ownership`, `org.parallax.settings.system` (zona horaria, hostname — delegando en `timedated`/`hostnamed`).

## Aislamiento de apps
- v1: cgroup por app + portals para recursos sensibles (cámara, pantalla, archivos fuera de sandbox).
- Apps de terceros: Flatpak soportado como formato recomendado (bubblewrap); PARALLAX no inventa sandbox propio en v1 (ADR futuro).

## Integridad
Secure Boot con shim + clave PARALLAX (fase distribución), UKI firmadas, raíz con btrfs snapshots; dm-verity para imagen de recuperación. SBOM CycloneDX por release.

## Auditoría
Eventos de seguridad con `MESSAGE_ID` estables en journald: auth fallida, elevación polkit, cambio de configuración de sistema, instalación de paquetes.
