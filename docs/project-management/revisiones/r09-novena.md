# PARALLAX OS — Novena revisión técnica complementaria

**Tema:** validación de la octava revisión (lanzamiento seguro, ciclo de sesión, recuperación).

- **Fecha:** 2026-11-19
- **Fase 0:** `IN_PROGRESS`
- **ADR:** todos en `Propuesto`
- **Modo:** solo lectura. Este informe está en el scratchpad de la sesión y no forma parte del repositorio.

**Marcas de evidencia:**

| Marca | Significado |
|---|---|
| `[V]` | Verificado en fuente oficial con lectura completa |
| `[V-p]` | Verificado con lectura parcial |
| `[R]` | Consta en el repositorio |
| `[H]` | Hipótesis o inferencia |
| `[VM]` | Requiere prueba en VM |
| `[HW]` | Requiere hardware |
| `[P]` | Decide el propietario |

---

## 1. Resumen ejecutivo

**El veredicto de la octava revisión no se acepta tal como estaba.** Esta revisión encuentra **cuatro errores de diseño o de afirmación** en la octava y los corrige en el diseño:

1. **Colisión en `dup2`.** Si los descriptores de origen ya ocupan los números 3 o 4, la secuencia `dup2(rol→3)`, `dup2(sync→4)` puede machacar el origen de la segunda acción antes de que se ejecute. Además, POSIX no aclara qué pasa con `dup2(fd, fd)` en `posix_spawn` (la página leída no trata el caso) [V-p].
   - **Corrección:** el padre duplica antes los orígenes a números altos con `F_DUPFD_CLOEXEC` y solo después hace `dup2(alto→3/4)`.
2. **Los descriptores 0–2 no estaban definidos, y la octava revisión confiaba en `CLOEXEC`.** Un hilo o una biblioteca C del compositor puede abrir descriptores sin `CLOEXEC`, y esos sobreviven al `posix_spawn`.
   - **Corrección:** 0–2 se fijan explícitamente. El intermediario, que es monohilo y recién creado, ejecuta como primera acción `close_range(5, ~0U, 0)` (cierre real, no solo `CLOEXEC`).
3. **El entorno no tenía lista blanca efectiva.** «Entorno mínimo» no estaba definido, y limpiar `WAYLAND_SOCKET` en el rol no evita lo que ya heredó el intermediario.
   - **Corrección:** el compositor pasa un `envp` construido por lista blanca; nunca su propio `environ`.
4. **La afirmación «la política X evita aplicaciones huérfanas» es falsa en general.** La política X detiene los roles y las unidades que PARALLAX lanza con `PartOf=graphical-session.target`. **No** detiene servicios activados por D-Bus ni unidades de terceros que no declaren esa dependencia.
   - **Corrección:** redefinir la garantía (§6).

**greetd verificado** con `greetd(5)` y `greetd(1)` de Arch, greetd 0.10.3-2 [V]:
- El greeter **no** es una unidad systemd: es el `command` de `[default_session]`, que se ejecuta con `sh` como usuario `greeter`.
- «Once this session terminates, the process starts over».
- **Confirmado C-38:** `parallax-greeter.service` es incorrecto.
- **No documentado:** el tratamiento del código de salida, el cierre de PAM y de logind.

**Veredicto (§15): `LISTA CON CONDICIONES EXPLÍCITAS`.** La justificación está en §15: tras estas correcciones el diseño es coherente en papel y cada hipótesis restante está delimitada y asignada a una prueba concreta.

---

## 2. Alcance y fuentes

**Repositorio [R]:** la octava revisión, en el scratchpad, y los documentos citados en ella. `git status` coincide con el del inicio de la sesión.

| Ref | Fuente | Versión | Lectura |
|---|---|---|---|
| GD1 | `greetd(5)`, man.archlinux.org | greetd 0.10.3-2 (páginas del 2026-03-27) | Completa |
| GD2 | `greetd(1)`, man.archlinux.org | Igual | Completa |
| GD3 | Código de greetd (`worker.rs`) | — | **No leído (HTTP 502)** |
| GD4 | `greetd-ipc(7)` | — | No leído |
| PS1 | `posix_spawn(3)`, man7 | — | Completa (octava revisión) |
| PS2 | `posix_spawn_file_actions_adddup2(3p)`, man7 | — | Completa; **no trata `fildes == newfildes`** |
| PS3 | `posix_spawn_file_actions_addclosefrom_np` / NEWS de glibc 2.34 | — | **No leído** (man7 404; sourceware bloqueado) |
| RS1 | `library/std/src/sys/process/unix/unix.rs` | rust-lang/rust `main` | Parcial |
| RS2 | Crate `libc` en docs.rs | 0.2.191, `MIT OR Apache-2.0` | Página de `posix_spawn` |
| RS3 | Crate `rustix` en docs.rs | 1.1.6, `Apache-2.0 WITH LLVM-exception OR Apache-2.0 OR MIT` | Lista de módulos (sin módulo `spawn`); `process` no abierto |
| SD1 | Paquete systemd de Arch | **262.2-1** (2026-11-10) | Página del paquete |
| SD2 | `graphical-session*.target` | Rama `main` de systemd | Completa (octava revisión). **La copia instalada en 262.2 no se ha comprobado** |
| Otras | Fuentes de systemd, kernel y polkit de las revisiones sexta a octava | — | Según se indicó entonces |

---

## 3. Auditoría de descriptores y `posix_spawn`

### 3.1 Qué hace realmente `std::process::Command` [V-p RS1]

- Usa `posix_spawn` solo si no hay uid, gid, `groups`, `chroot` ni cierres `pre_exec`, y con glibc ≥ 2.24.
- Las únicas acciones `dup2` que añade son las de stdin, stdout y stderr.
- No cierra ningún otro descriptor ni llama a `addclosefrom_np`.
- **No permite colocar descriptores en 3 o 4 sin `pre_exec`**, y usar `pre_exec` lo devuelve al camino `fork`.

**Conclusión:** para el intermediario, `Command` no sirve. Hay que llamar a `posix_spawn` directamente (D-27).

### 3.2 Lista blanca real de descriptores

| fd | Contenido | Cómo se garantiza |
|---|---|---|
| 0 | `/dev/null` | `addopen(0, "/dev/null", O_RDONLY)` |
| 1, 2 | Stream del journal heredado del compositor, o `/dev/null` | `adddup2(stdout_alto→1)` / `adddup2(stderr_alto→2)`. Decisión: **el log del rol debe ir al journal** (`AGENTS.md`: tracing → journald [R]) |
| 3 | Socket de rol | `adddup2(rol_alto→3)` |
| 4 | Tubería de sincronización (lado de lectura) | `adddup2(sync_alto→4)` |
| ≥ 5 | Nada | (a) `posix_spawn` cierra los `CLOEXEC` [V PS1]; (b) el intermediario ejecuta `close_range(5, ~0U, 0)` al empezar [V `close_range(2)`] |

**Orden y colisiones:**

1. En el padre: `rol_alto = fcntl(rol, F_DUPFD_CLOEXEC, 10)`, y lo mismo para `sync`, `stdout` y `stderr`. Así ningún origen es menor que 10 y ninguno coincide con 0–4 [H; semántica estándar de `fcntl`].
2. Las acciones de archivo se ejecutan en orden [V PS1, «File actions» después de los atributos]. Como los orígenes (≥ 10) no coinciden con los destinos (0–4), no hay colisión. Que `dup2` quite `FD_CLOEXEC` al destino es la semántica de `dup2` cuando origen ≠ destino [H estándar]. **No dependemos del caso origen == destino** (PS2 no lo documenta).
3. Los descriptores `_alto` tienen `CLOEXEC` y desaparecen en el `exec` del intermediario.

**Hilos concurrentes:** un hilo puede abrir un descriptor sin `CLOEXEC` (Mesa, libinput, bibliotecas C) durante el spawn. Sobrevive a `posix_spawn`, pero lo **cierra el `close_range` del intermediario** antes de cualquier otra acción. La ventana de exposición queda limitada al intermediario, que es código propio y monohilo y no lee ese descriptor [H].

**`addclosefrom_np` (glibc 2.34):** **no verificado** (PS3). No se usa. El `close_range` en el intermediario cubre la misma función sin depender de la versión de glibc. `close_range` existe desde Linux 5.9 [V-p]; `CLOSE_RANGE_CLOEXEC`, desde 5.11 [V].

**En el rol:** tras el `exec`, el intermediario cierra el 4 antes de hacer `exec` (o lo marca `CLOEXEC`). El rol hereda 0–3. El rol conecta con el 3, lo marca `CLOEXEC` y borra `WAYLAND_SOCKET`.

### 3.3 Entorno

| Etapa | Entorno |
|---|---|
| Compositor | Su propio entorno (heredado del user manager) |
| Intermediario | **`envp` explícito** pasado a `posix_spawn` [V RS2: parámetro `envp`], construido por lista blanca en el compositor |
| Rol | `envp` explícito en el `execve` del intermediario: la lista recibida más `WAYLAND_SOCKET=3` |

**Lista blanca propuesta [H, a revisar por seguridad]:**
`HOME`, `USER`, `LOGNAME`, `LANG`, `LC_*`, `XDG_RUNTIME_DIR`, `XDG_SESSION_ID`, `XDG_SESSION_TYPE=wayland`, `XDG_CURRENT_DESKTOP`, `DBUS_SESSION_BUS_ADDRESS`, `PATH` (fijo), `WAYLAND_SOCKET=3`.

**Excluidas explícitamente:**
- `WAYLAND_DISPLAY`: el rol debe usar el socket de rol, no el público.
- `XDG_ACTIVATION_TOKEN`, `DESKTOP_STARTUP_ID`, `LD_*`, `SSH_AUTH_SOCK`.
- Cualquier variable de depuración.

El binario del rol se ejecuta por **ruta absoluta de solo lectura** (`ingeniero-seguridad.md` regla 1 [R]), nunca por búsqueda en `PATH`.

### 3.4 Carreras: quién detecta, quién limpia, garantía

| Carrera | Detecta | Limpia | Garantía |
|---|---|---|---|
| El compositor muere antes del `posix_spawn` | — | — | No hay proceso [H] |
| Muere tras el spawn y antes de crear el scope | El intermediario (EOF en el 4) | El intermediario sale; el kernel lo reasigna como huérfano a su subreaper [H] | EOF al cerrarse todos los extremos de escritura [H estándar]. **Exige que el extremo de escritura no se herede a nadie más**, lo que garantiza §3.2 |
| Muere con `StartTransientUnit` pendiente | systemd (el job puede completarse) y el intermediario (EOF) | El intermediario sale; el scope queda vacío y se libera [V systemd.scope] | Con la política X, la sesión termina de todos modos por `BindsTo` |
| El job termina y se pierde la respuesta | Compositor (señal `JobRemoved` suscrita antes, más timeout) | Compositor: `GetUnitByPIDFD` | [V API]; [VM] comportamiento |
| El proceso termina antes de `GetUnitByPIDFD` | Compositor (pidfd legible) | Compositor: estado `FAILED`; el scope vacío lo libera systemd | El PID no se reutiliza mientras no se recoja con `wait` [H estándar] |
| El scope existe y el intermediario no recibe la orden de liberación | Compositor (timeout) | Compositor: cierra el 4 → EOF → el intermediario sale | [H] |
| El intermediario recibe EOF | Intermediario | Sale con código ≠ 0 | [H] |
| El rol falla nada más hacer `exec` | Compositor (pidfd) | Compositor: backoff | — |
| Se relanza el rol con el scope anterior activo | **systemd**: si el nombre del scope es fijo por rol (p. ej. `parallax-role-shell.scope`*), `StartTransientUnit` falla mientras exista una unidad con ese nombre [H] [VM] | Compositor: espera `STOPPED` o hace `StopUnit` | Exclusión mutua por nombre de unidad, **si se verifica** |

\* Nombre propuesto, no aceptado (D-25).

**Doble instancia:** se evita con dos capas. (1) Un solo hueco por rol en el compositor. (2) Un nombre de scope único por rol, que debe probarse.

La tabla describe el comportamiento esperado. **Solo están documentadas las APIs**; las garantías de comportamiento quedan `[VM]`.

---

## 4. `PR_SET_DUMPABLE` y la frontera de seguridad

| Aspecto | Resultado |
|---|---|
| Cuándo se aplica | Primera instrucción del rol tras el `exec` (y también en el intermediario, por defensa) |
| Qué protege | `PTRACE_ATTACH` [V]; la comprobación de modo ptrace (que cubre `pidfd_getfd`) [V-p]; la propiedad de `/proc/<pid>` [V-p, remite a `proc_pid(5)`, no leído] |
| Qué no protege | `kill`, `$HOME`, nombres D-Bus, la ventana previa al `prctl` |
| Ventana previa al `prctl` | Yama=1 impide `ATTACH` desde no descendientes [V]. El compositor es ascendiente y puede (es confiable). La lectura con `readlink` de `/proc/<pid>/fd` (modo READ) **no la restringe Yama** [V-p] y muestra inodos. La reapertura de sockets no está documentada [VM] |
| setuid, setgid, file capabilities | Restablecen `dumpable` [V]. Los roles **no** deben ser setuid ni tener capacidades. Se fija en empaquetado [H] |
| Mismo UID | Conserva `kill`, archivos y D-Bus |
| Sockets heredados | Solo el 3, y ya se ha limpiado (§3) |

**Clasificación:**
1. **Separación lógica de roles:** sí (catálogo y socket).
2. **Capacidad representada por un descriptor:** sí.
3. **Restricciones del kernel:** sí, parciales (`dumpable`, Yama).
4. **Aislamiento real entre procesos del mismo UID:** **no**. No es un sandbox.

---

## 5. Auditoría del grafo systemd

| # | Comprobación | Resultado |
|---|---|---|
| 1 | ¿`parallax-session.target` activa los targets necesarios? | `BindsTo=graphical-session.target` también activa la unidad (equivale a `Requires=`) [V]. `RefuseManualStart=yes` impide arrancarla a mano, no como dependencia [V SD2]. `Wants=graphical-session-pre.target` la activa [V]. **Sí** |
| 2 | ¿El fallo del compositor detiene `parallax-session.target`? | `BindsTo=parallax-compositor.service` + `After=` → el target se detiene si el compositor cae [V]. Requiere `Restart=no` en el compositor: con `Restart=`, el estado intermedio de reinicio sin `BindsTo` queda `[VM]` |
| 3 | ¿Al parar el target, `graphical-session.target` deja de ser necesario? | Si ninguna otra unidad activa lo requiere o lo quiere [H]. `PartOf` no crea necesidad [H] |
| 4 | ¿`StopWhenUnneeded=yes` lo detiene? | Upstream sí [V SD2]. **La copia de la 262.2 no se ha comprobado** [VM] |
| 5 | ¿`PartOf` alcanza a roles y apps? | Los scopes y servicios transitorios aceptan «all dependency types» en la creación [V CONTROL_GROUP_INTERFACE] [VM] |
| 6 | `settingsd` | `Wants=` + `After=` desde el compositor; `PartOf=graphical-session.target`. **Corrección:** el compositor necesita settingsd para arrancar (`arquitectura.md` §5 [R]). Si settingsd falla, `Wants` no impide el arranque: el compositor debe tolerarlo con valores por defecto (coherente con el overlay sin settingsd) |
| 7 | Ciclos | Orden: `graphical-session-pre` → `settingsd` → `compositor` → `parallax-session.target`. `graphical-session.target` no tiene orden respecto al compositor. Sin ciclos aparentes [H]; `systemd-analyze verify` [VM] |
| 8 | ¿La parada normal y la caída siguen caminos coherentes? | Normal: el comando de sesión detiene el target → `graphical-session.target` innecesario → se detiene → `PartOf` detiene compositor, roles y apps. Caída: compositor `failed` → `BindsTo` detiene el target → mismo camino. **Coherentes** [V semántica; VM ejecución] |

`After=`/`Before=` solo ordenan. `Wants=`/`Requires=`/`BindsTo=` activan. `BindsTo=`/`Requires=`/`PartOf=` propagan la parada (con direcciones distintas). `StopWhenUnneeded` detiene por falta de necesidad. Ninguna de estas relaciones se ha confundido.

---

## 6. Aplicaciones D-Bus y procesos externos

| Clase | ¿Termina con la sesión gráfica? | Por qué |
|---|---|---|
| Apps PARALLAX lanzadas por la biblioteca | Sí | Servicio transitorio con `PartOf=graphical-session.target` |
| Apps `DBusActivatable` | **No necesariamente** | Las arranca el bus. Su unidad (`SystemdService=`) la define el paquete [V], y su cgroup puede ser el del bus [H] |
| Servicios activados por D-Bus | No necesariamente | Igual |
| Unidades de usuario independientes | No, salvo que declaren `PartOf` | — |
| Flatpak lanzado por la biblioteca | Sí (es la unidad transitoria) | — |
| Flatpak activado por D-Bus | No necesariamente | — |
| Procesos de fondo y PipeWire | No (deben sobrevivir) | — |

**Después:** al cerrarse la sesión de logind se aplica `KillUserProcesses=` (por defecto upstream `yes` [V]). Si es la última sesión del usuario y no hay *lingering*, el user manager se detiene y todas las unidades de usuario terminan [H] [VM].

**Política realista v1:**
- **Terminan:** roles, compositor, settingsd y apps PARALLAX (por `PartOf`).
- **Pueden sobrevivir hasta que logind o el user manager terminen:** servicios D-Bus, apps `DBusActivatable` de terceros, PipeWire y unidades de usuario.
- **Identificación:** solo lo que PARALLAX crea lleva `PartOf`. Nada más se toca.
- **Externos que no respetan el modelo:** no se matan desde PARALLAX (`AGENTS.md` regla 7; ADR-0014: resourced no mata [R]).

**Lo que PARALLAX no puede garantizar:** que una app de terceros termine con la sesión gráfica, ni que guarde sus datos.

**Afirmación corregida:** la política X **elimina los procesos que PARALLAX lanza**, no todas las aplicaciones huérfanas.

---

## 7. Política X y sesiones simultáneas

| Punto | Resultado |
|---|---|
| ¿La caída termina las unidades de sesión? | Sí por diseño (§5.8) [VM] |
| ¿greetd recibe el resultado? | Cuando el comando de sesión sale, «the process starts over» [V GD2]. **El tratamiento del código de salida no está documentado** [V: ausencia en GD1/GD2] |
| ¿Vuelve el greeter? | Sí: `default_session` se ejecuta «when no other session is running, such as when a user session terminates» [V GD1] |
| ¿Puede el usuario volver a autenticarse sin reiniciar? | Sí, por lo anterior [V]; ciclo completo [VM] |
| Datos no guardados | Se pierden (igual que en Y: sin Wayland, la app ya los ha perdido) |
| Procesos que no dependen de Wayland | Ver §6 |
| Segunda sesión del mismo usuario | `systemd --user` es uno por usuario [H]; `graphical-session.target` está activo si hay cualquier sesión gráfica [V]. Una caída en una sesión afectaría a la otra |
| Otro entorno que también use `graphical-session.target` | El mismo conflicto |

**D-29: exclusión implementable.**

| Nivel | Qué hay |
|---|---|
| Restricción de producto | «Una sola sesión gráfica por usuario en v1» [P] |
| Restricción de systemd | Ninguna nativa; `graphical-session.target` es compartido por usuario [V/H] |
| Restricción de logind | Ninguna para el mismo usuario [H] |
| **Mecanismo** | El comando de sesión, **antes** de arrancar el target, consulta al user manager `ActiveState` de `graphical-session.target` (y de `parallax-session.target`). Si está `active` o `activating`, sale con error y greetd vuelve al greeter [V GD1/GD2 para la vuelta; H para la consulta] |
| Estado residual tras una caída | No hay archivo de bloqueo: el estado es el de las unidades del user manager. `StopWhenUnneeded` lo limpia [V SD2]. Si quedara colgado, `systemctl --user stop` como recuperación manual documentada [H] |
| Comunicar el rechazo al usuario | greetd no documenta un canal para el motivo [V: ausencia]. El greeter no sabrá por qué. **Pendiente** (D-30) |

---

## 8. Verificación de greetd

| Pregunta | Resultado |
|---|---|
| `default_session` | `command` (ejecutado con `sh`), `user` (por defecto `greeter`) [V GD1] |
| Inicio del greeter | greetd lo arranca cuando no hay otra sesión [V GD1] |
| Inicio de la sesión de usuario | El greeter crea la sesión por IPC (`GREETD_SOCK`), autentica y pide arrancar; greetd la inicia cuando el greeter termina [V GD2]. Campos de la petición: `greetd-ipc(7)`, no leído |
| Proceso en primer plano | El comando de la sesión: «Once this session terminates, the process starts over» [V GD2] |
| Salida con código 0, distinto de 0 o por señal | **No documentado** [V: ausencia]. Por GD2, en todos los casos «starts over» [H] |
| Vuelta al greeter | `default_session` [V GD1] |
| Cierre de la sesión PAM y de logind | **No documentado** en GD1/GD2. Código no leído (502). Hipótesis: greetd cierra PAM al terminar el comando [H] |
| ¿Encaja `parallax-session.target` con un comando propio? | **Sí**: greetd ejecuta un comando arbitrario y espera a que termine [V GD2]. El comando propio arranca el target, espera y sale |
| `[initial_session]` (inicio automático) | Existe; se ejecuta una vez, controlado por el *runfile* [V GD1]. No se usa en v1 salvo decisión [P] |
| Terminal | `vt`, `switch`; conflicto con `getty@ttyN` [V GD1] |

**C-38:** **confirmado.** El greeter es el `command` de `default_session`, ejecutado por greetd como usuario `greeter`. No es `parallax-greeter.service`.

**C-36:** el flujo G-c **encaja** con greetd. Falta el comportamiento ante códigos de salida y el cierre de PAM [VM o lectura del código].

---

## 9. Evaluación del intermediario (D-27)

| Opción | Responsabilidad y privilegio | Ciclo de vida | Superficie | Mantenimiento |
|---|---|---|---|---|
| **Segundo `[[bin]]` del paquete del compositor** | Igual que el compositor: mismo UID, sin capacidades | Efímero (se convierte en el rol con `exec`) | Mínima | Mismo dueño y release (regla de la propuesta, §7.8) |
| Ejecutable independiente | Igual | Igual | Igual | Otro paquete sin motivo |
| Utilidad compartida | Riesgo de que otros lo usen para obtener roles | — | Mayor | — |
| Daemon | Proceso permanente, sin justificación | — | — | Rechazado |

**Recomendación:** segundo `[[bin]]`. Debe ser `no_std` o mínimo, sin dependencias aparte de `libc` [H].

**Dependencia para `posix_spawn` (lado del compositor):**

| Opción | Licencia | Estado | Notas |
|---|---|---|---|
| Crate `libc` 0.2.191 | `MIT OR Apache-2.0` [V RS2] | Ampliamente usado (rust-lang); probablemente ya transitivo vía Smithay [H] | `unsafe` con `// SAFETY:` (`AGENTS.md`); entrada en `dependencias.md` (ADR-0008) |
| `rustix` 1.1.6 | `Apache-2.0 WITH LLVM-exception OR Apache-2.0 OR MIT` [V RS3] | Sin módulo `spawn` en el índice [V-p] | No cubre `posix_spawn` [H] |
| `std::process::Command` | — | No puede colocar descriptores en 3 y 4 sin `pre_exec` [V-p RS1] | Descartado |

`close_range` en el intermediario: está en `libc` [H; no se comprobó en docs.rs].

MSRV: depende de `rust-toolchain.toml`, que todavía no existe [R]. Pendiente.

---

## 10. Matriz de hallazgos

| ID | Problema | Evidencia | Impacto | Corrección propuesta | Decisión | Prueba | Clasificación |
|---|---|---|---|---|---|---|---|
| N9-1 | Colisión de `dup2` y caso `fildes == newfildes` | [V-p PS2] | Herencia incorrecta del descriptor | Orígenes ≥ 10 con `F_DUPFD_CLOEXEC` | No | VM | Resuelto en diseño, pendiente de implementación |
| N9-2 | Descriptores 0–2 indefinidos; los que no tienen `CLOEXEC` sobreviven | [V PS1], [H] | Fuga de descriptores privilegiados (DRM, D-Bus) | 0–2 explícitos; `close_range(5,…)` en el intermediario | No | VM | Resuelto en diseño, pendiente de implementación |
| N9-3 | Entorno sin lista blanca | [V RS2 `envp`] | Fuga de variables | `envp` por lista blanca; excluir `WAYLAND_DISPLAY` | Sí (seguridad, D-22) | VM | Resuelto en diseño, pendiente de implementación |
| N9-4 | «X elimina huérfanos» | [V], [H] | Promesa falsa | §6 | No | VM | Resuelto documentalmente (en propuesta) |
| C-21 | Límites frente al mismo UID | [V], [V-p] | — | §4 | D-22 [P] | VM | Pendiente de decisión del propietario |
| C-22 | Grafo | [V SD2], [V] | — | §5 | D-24 [P] | VM | Pendiente de prueba en VM |
| C-23 | `PIDs` y spawn | [V G1], [V RS1] | — | §3 | D-18 [P], D-27 | VM | Pendiente de prueba en VM |
| C-36 | greetd | [V GD1/GD2] | — | G-c | D-28 | VM + código | Resuelto en diseño; códigos de salida pendientes |
| C-38 | Greeter como servicio | [V GD1] | Documento incorrecto | P2 (§11) | No | No | Resuelto documentalmente (en propuesta) |
| R-20 | `PIDs` en 262.2 | [V G1] | — | — | — | VM | Pendiente de prueba en VM |
| R-21 | Agente antiguo | — | — | Nombre de scope fijo + `PartOf` | D-25 | VM | Pendiente de prueba en VM |
| R-22 | Ventana sin `dumpable` | [V] | Bajo (Yama) | §4 | — | VM | Pendiente de prueba en VM |
| R-23 | Suplantación visual | [H] | — | — | D-22 | — | Diferible (diseño del rol `auth`, Fase 3) |
| R-25 | `StopWhenUnneeded` en 262.2 | [V upstream] | — | — | — | VM | Pendiente de prueba en VM |
| R-26 | Huérfanos | — | — | §6 | — | VM | Resuelto documentalmente (alcance redefinido) |
| R-27 | Pérdida total de ventanas y vuelta al login | — | Experiencia | — | D-24 [P] | — | Pendiente de decisión del propietario |
| R-28 | Terceros sin `PartOf` | [V] | — | §6 | — | VM | Aceptado como límite |
| R-29 | greetd | [V GD1/GD2] | Reducido | — | — | Código | Pendiente de evidencia (código de salida, PAM) |
| R-30 | `unsafe` y `libc` | [V RS2] | — | §9 | D-27 | — | Resuelto en diseño, pendiente de implementación |
| D-29 | Sesión única | [H] | — | §7 | [P] + D-30 | VM | Pendiente de decisión del propietario |

**Gate 0:** ningún punto es bloqueante. Todos son de Fase 1 o posteriores (spikes) o son decisiones con alternativas definidas. Según `fase0-revision.md` [R], el agente polkit (ADR-0010) es diferible hasta Gate 2.

---

## 11. Correcciones documentales propuestas (no aplicadas)

1. **`architecture/arquitectura.md` §2, fila `parallax-greeter`**
   - **Actual:** «`parallax-greeter.service` (sistema, vía greetd)».
   - **Propuesto:** «Comando `[default_session]` de greetd, ejecutado como usuario `greeter` (greetd(5)); no es una unidad systemd propia.»
   - **Justificación:** GD1. Revisa: Linux.
2. **`specifications/lockscreen-greeter.md` (líneas 156–158)** — **Añadir:** «greetd lanza el greeter en `default_session` cuando no hay otra sesión y lo vuelve a lanzar cuando la sesión de usuario termina (greetd(1)).»
3. **`security/modelo-seguridad.md`, nueva fila «Creación de roles»** — **Texto:** «`posix_spawn` con `envp` por lista blanca y acciones de archivo que colocan explícitamente 0–4 (orígenes ≥ 10). El intermediario ejecuta `close_range(5, ~0U, 0)` y después `execve` por ruta absoluta. El rol aplica `PR_SET_DUMPABLE=0`. No es aislamiento frente al mismo UID.» ADR-0015. Revisa: seguridad. Decisión: D-22.
4. **`architecture/arquitectura.md` §5, «Caída del compositor»** — **Precisar:** «La política X detiene las unidades que PARALLAX crea (`PartOf=graphical-session.target`). Los servicios de terceros o activados por D-Bus pueden sobrevivir hasta que terminen la sesión de logind o el user manager (`KillUserProcesses`).» Decisión: D-24.
5. **`architecture/arquitectura.md` §5, «Arranque»** — **Añadir:** «El comando de sesión rechaza el arranque si `graphical-session.target` ya está activo en el user manager (una sola sesión gráfica por usuario en v1).» Decisión: D-29.
6. **`security/dependencias.md`, registro (cuando se apruebe)** — `libc` (`MIT OR Apache-2.0`), uso: `posix_spawn` y `close_range` en el compositor y en el intermediario. Decisión: D-27 y ADR-0008.
7. **`specifications/compositor.md` §15** — Añadir el segundo `[[bin]]` (intermediario, nombre pendiente).

Las propuestas P1–P10 de la octava revisión se mantienen, salvo su P7, que esta sustituye por las propuestas 1 y 2.

---

## 12. Decisiones pendientes del propietario

| ID | Decisión | ¿Suficientemente definida para decidir? |
|---|---|---|
| D-18 | Supervisión de los roles por el compositor en lugar de `Restart=always` | Sí: alternativas y evidencia en la séptima y octava revisiones |
| D-22 | Límites de seguridad, rol `auth` y lista blanca de entorno | Sí, con §3–§4 |
| D-24 | Política X en v1 (pérdida de ventanas y vuelta al login) | Sí, con el alcance corregido en §6 |
| D-27 | Intermediario como `[[bin]]` del compositor y dependencia `libc` | Sí (técnica). Requiere el registro de ADR-0008 |
| D-28 | Comando de sesión propio | Sí; falta el código de salida de greetd |
| D-29 | Una sesión gráfica por usuario; mecanismo §7 | Sí |
| **D-30 (nueva)** | Cómo informa el greeter del rechazo de una segunda sesión | No: falta `greetd-ipc(7)` |

---

## 13. Pruebas y spikes pendientes (no autorizados)

| Prueba | Tipo |
|---|---|
| `posix_spawn`: solo sobreviven 0–4; `close_range` en el intermediario | VM |
| Entorno efectivo del rol (`/proc/<pid>/environ` en la prueba) | VM |
| `PIDs` en `StartTransientUnit` (systemd 262.2); exclusión por nombre de scope | VM |
| `PartOf=` y `Slice=` en unidades transitorias | VM |
| `graphical-session.target` instalado en Arch; cadena `BindsTo` → `StopWhenUnneeded` → `PartOf` | VM |
| `PR_SET_DUMPABLE` tras `execve`; `pidfd_getfd`; `/proc/<pid>/fd` | VM |
| greetd: código de salida, cierre de PAM y logind, vuelta al greeter | VM + lectura del código fuente |
| `KillUserProcesses` efectivo y supervivencia de servicios D-Bus | VM |
| Rechazo de una segunda sesión | VM |

---

## 14. Riesgos residuales

| ID | Riesgo |
|---|---|
| R-20, R-25 | Comportamiento de systemd 262.2 no comprobado |
| R-29 | Semántica de salida de greetd no documentada |
| R-31 (nuevo) | Una biblioteca C del compositor abre descriptores sin `CLOEXEC`. Queda mitigado por `close_range`, pero en el compositor mismo siguen visibles a sus hilos (sin impacto entre procesos) |
| R-32 (nuevo) | La lista blanca de entorno omite una variable que un rol necesita (p. ej. de accesibilidad): fallo funcional, no de seguridad |
| R-33 (nuevo) | El rechazo de una segunda sesión no se puede explicar al usuario (D-30) |
| R-27, R-28 | Se mantienen |

---

## 15. Veredicto: `LISTA CON CONDICIONES EXPLÍCITAS`

**Se distingue:**

| Dimensión | Estado |
|---|---|
| Coherencia en papel | **Sí**, tras corregir N9-1 a N9-4 |
| APIs documentadas | Sí: `posix_spawn`, `close_range`, `PIDs` en scopes, `GetUnitByPIDFD`, directivas de unidad, `graphical-session.target` upstream, `default_session` y reinicio del ciclo de greetd |
| Funcionamiento en Arch (systemd 262.2, greetd 0.10.3) | **No verificado** (§13) |
| Implementación segura | No evaluable sin código. El diseño indica qué hay que probar |
| Experiencia de sesión aceptable | Decisión del propietario (D-24, D-29) |

**Por qué no es «REQUIERE OTRA REVISIÓN TÉCNICA DIRIGIDA»:**
- Los errores encontrados se han corregido en el diseño.
- Lo pendiente no se resuelve con otra revisión documental: necesita VM, el código de greetd o decisiones del propietario.

**Por qué no es «LISTA PARA REVISIÓN DEL PROPIETARIO» sin condiciones:**
- La viabilidad depende de pruebas no ejecutadas: `PIDs`, `StopWhenUnneeded` en 262.2 y el ciclo completo de greetd.
- Faltan dos lecturas: el código de greetd y `greetd-ipc(7)`.

**Condiciones:**
1. Registrar las pruebas de §13 como spikes (autoriza el propietario).
2. Leer el código fuente de greetd (`worker.rs`) y `greetd-ipc(7)` cuando estén accesibles.
3. Decisiones del propietario: D-18, D-22, D-24, D-29.
4. Registrar `libc` en `dependencias.md` antes de implementar (D-27, ADR-0008).

---

## Estado al cierre

| Pregunta | Respuesta |
|---|---|
| ¿Cambió el estado de algún ADR? | **No**. Todos siguen en `Propuesto` |
| ¿Cambió el estado de la Fase 0? | **No**. Sigue en `IN_PROGRESS` |
| ¿Se modificó algún archivo del repositorio? | **Ninguno** |
| ¿Se ejecutó alguna prueba o spike? | **Ninguno** |
| ¿Hay autorización para la Fase 1? | **No existe** |
