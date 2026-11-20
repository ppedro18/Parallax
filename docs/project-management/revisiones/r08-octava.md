# PARALLAX OS — Octava revisión técnica complementaria

- **Alcance:** C-20 a C-23, C-29 y C-34 a C-38.
- **Fecha:** 2026-11-19
- **Fase 0:** `IN_PROGRESS`
- **ADR:** todos en `Propuesto`
- **Modo:** solo lectura. Este informe está en el scratchpad de la sesión y no es normativo.

### Convención de evidencia

| Marca | Significado |
|---|---|
| `[V]` | Fuente oficial leída de forma completa |
| `[V-p]` | Fuente oficial leída de forma parcial o resumida |
| `[R]` | Repositorio |
| `[H]` | Inferencia técnica |
| `[VM]` | Requiere prueba en VM |
| `[HW]` | Requiere hardware |
| `[P]` | Requiere decisión del propietario |

Una API documentada no es una API comprobada en Arch. No se ejecutó nada.

---

## 1. Resumen ejecutivo

1. **R-20 cambia de clasificación.** systemd.io/CONTROL_GROUP_INTERFACE documenta la propiedad `PIDs` de los scopes: «used for construction of the scope only and specifies the initial PIDs» [V]. La séptima revisión decía que no existía. **Esa conclusión era incorrecta**: no aparecía en la página de `org.freedesktop.systemd1`, pero sí en la de interfaces de cgroups. Nueva clasificación: **API documentada, sin verificar en Arch**. El tipo D-Bus no figura en la página [V]; el habitual `au` es una inferencia [H].
2. **No se recomienda `fork()` desde el compositor.** En un programa multihilo el hijo solo puede llamar a funciones async-signal-safe hasta `execve` [V fork(2)], y `pre_exec` de Rust no garantiza `malloc`, `std::env` ni mutex [V]. Se recomienda la **opción C**: lanzar un **intermediario efímero** (no permanente) con `posix_spawn`. El intermediario espera el registro en el scope y después hace `exec` del rol.
3. **`PR_SET_DUMPABLE` lo debe poner el propio rol al arrancar.** Hacerlo antes del `exec` no basta: lo más probable es que un `exec` normal lo restablezca [H]; verificación `[VM]`.
4. **El grafo de sesión queda bien fundado.** La unidad upstream `graphical-session.target` declara `RefuseManualStart=yes` y `StopWhenUnneeded=yes` [V, rama `main`]. Esto explica la propagación de parada que la séptima revisión dejaba en duda. La versión instalada en Arch es `[VM]`.
5. **Para v1 se recomienda la política X en D-24:** si cae el compositor, termina toda la sesión gráfica. Sus ventajas:
   - Resuelve la garantía de bloqueo por construcción: no hay sesión que desbloquear y el greeter pide autenticación. Así D-26 deja de ser necesaria para la caída del compositor, aunque sigue aplicando a la caída de roles.
   - Evita aplicaciones huérfanas sin Wayland.
   - No convierte al compositor en gestor de procesos.
6. **greetd no se pudo verificar.** Su documentación devolvió HTTP 502. El flujo de arranque se propone como `[H]` y queda pendiente de fuente.

**Veredicto (§16): `LISTA CON CONDICIONES EXPLÍCITAS`.**

---

## 2. Alcance y fuentes consultadas

**Repositorio [R].** Los archivos se leyeron en esta sesión. `git status` coincide con el del inicio de la sesión.
- `graphical-session` no aparece en el repositorio.
- `parallax-session.target` aparece solo en `arquitectura.md` §2 y §5.

**Fuentes externas**

| Ref | Fuente | Lectura |
|---|---|---|
| G1 | systemd.io/CONTROL_GROUP_INTERFACE | Completa |
| G2 | `systemd-run(1)` | Completa |
| G3 | `systemctl(1)` | Parcial: la opción `--wait` solo leída resumida |
| G4 | `units/user/graphical-session.target` y `graphical-session-pre.target` (systemd, rama `main`) | Completa |
| G5 | `systemd.special(7)`, `systemd.unit(5)`, `systemd.scope(5)`, `systemd.kill(5)`, `org.freedesktop.systemd1(5)` | Completa en las secciones citadas (séptima revisión) |
| K1 | `fork(2)` | Completa |
| K2 | Rust `std::os::unix::process::CommandExt` | Completa |
| K3 | `posix_spawn(3)` | Completa |
| K4 | `close_range(2)` | Completa |
| K5 | `PR_SET_DUMPABLE(2const)`, `pidfd_getfd(2)`, `proc_pid_fd(5)`, Yama | Completa (séptima revisión) |
| L1 | `org.freedesktop.login1(5)`, sección Session | Parcial |
| W1 | Wayland client API (apb) | Parcial |
| — | greetd (`greetd(5)`) | **No leída: HTTP 502** |

---

## 3. C-23 — Registro de scopes con `PIDs`

### 3.1 Verificación

| Punto | Resultado |
|---|---|
| 1. ¿`StartTransientUnit` admite `PIDs`? | Sí, para scopes [V G1] |
| 2. Tipo D-Bus | No indicado en G1 [V]. `au` es una inferencia [H]; hay que confirmarlo [VM] |
| 3. ¿Solo para `.scope`? | Sí, «`PIDs` for scope units» [V G1] |
| 4. ¿Uno o varios PID iniciales? | «initial PIDs», en plural [V G1] |
| 5–7. Restricciones del user manager, permisos, sistema frente a usuario | No documentado en G1 [V]. Inferencia: el user manager acepta PIDs del mismo usuario [H] [VM] |
| 8. Versión | La propiedad aparece sin versión en G1. `GetUnitByPIDFD` requiere systemd ≥ 253 [V G5] |
| 9. ¿Y si el PID desaparece antes de terminar el job? | No documentado [VM]. Mitigación: el padre no recolecta (*reap*) al hijo, así que el PID no se puede reutilizar mientras sea zombi [H] |
| 10. Garantías del resultado del job | `JobRemoved` con `done`, `failed`, etc. [V G5]. Que `done` implique que el PID está dentro del scope **no está documentado** [H] |
| 11. Verificación de pertenencia | `GetUnitByPIDFD` [V G5] |
| 12. Alternativa | `Scope.AttachProcesses` sobre un scope existente [V G5] |

`systemd-run --scope` ejecuta el comando como proceso hijo de `systemd-run` y de forma síncrona [V G2]. **El orden exacto entre registro y `exec` no está documentado** [V G2].

### 3.2 Carreras

| Carrera | Tratamiento |
|---|---|
| El hijo muere antes del registro | El job falla o el scope queda vacío. El padre lo detecta por el pidfd [H] [VM] |
| El compositor muere durante la solicitud | El intermediario recibe EOF por la tubería y sale sin hacer `exec`. El scope vacío se libera [V G5: un scope vive mientras tiene procesos] |
| Job aceptado pero el proceso ya terminó | Se detecta con el pidfd; se trata como `FAILED` |
| Reutilización de PID | Imposible mientras el padre no recolecte al hijo [H]. El pidfd lo confirma |
| El job falla | `FAILED` y backoff |
| Se pierde la respuesta D-Bus | Se suscribe a `JobRemoved` antes de llamar, con timeout, y después se verifica con `GetUnitByPIDFD` [H] |
| Cambio de cgroup antes de la verificación | El mismo UID podría mover el proceso dentro de la delegación del usuario [H]. La verificación se hace justo antes de liberar al intermediario. Después del `exec` no hay garantía (límite conocido) |
| El padre muere tras el registro y antes de liberar | EOF → el intermediario sale; el scope se libera |

**Uso del pidfd:** el padre obtiene un `pidfd_open` tras el spawn. Sirve para detectar la muerte del hijo y para consultar `GetUnitByPIDFD` sin carreras [V G5].

### 3.3 Alternativas

| | A. `PIDs` en la creación | B. Scope vacío + `AttachProcesses` | C. Auxiliar dedicado | D. `systemd-run --scope` |
|---|---|---|---|---|
| Complejidad | Baja | Media | Alta | Baja |
| Garantías | Documentada (G1) | Documentada (G5) | Igual que A o B | Orden de registro no documentado (G2) |
| Limitaciones | Tipo y permisos sin documentar | ¿Se puede crear un scope sin PID? No documentado [H] | Proceso adicional | Proceso `systemd-run` como padre y un nivel extra |
| Versión | No indicada | No indicada | — | — |
| Riesgo de carrera | Bajo con hijo bloqueado | Bajo | Igual | Desconocido |
| User manager | [VM] | [VM] | [VM] | [VM] |
| Recuperación | Sencilla | Media | Compleja | Sencilla |

**Recomendación:** **A**, con el hijo bloqueado (§4).

**R-20:** «**API documentada, sin verificar en Arch**».

---

## 4. C-21/C-23 — Creación segura de procesos desde un compositor Rust multihilo

### 4.1 Hechos

| Tema | Hecho |
|---|---|
| Hilos tras `fork` | Solo se replica el hilo que llama [V K1] |
| Restricción | El hijo solo puede llamar funciones async-signal-safe hasta `execve` [V K1] |
| Mutex | Los que retenían otros hilos quedan bloqueados para siempre en el hijo [H, consecuencia de K1] |
| Rust `pre_exec` | «`malloc`, accessing environment variables through `std::env` or acquiring a mutex are not guaranteed to work» [V K2] |
| Rust `Command` | La documentación de `CommandExt` no dice si `std` usa `posix_spawn` [V K2]. No se puede afirmar |
| Mapear fd con `Command` (API estable) | `CommandExt` no tiene un método estable para mapear fd arbitrarios [V K2: lista de métodos] |
| `posix_spawn` | Usa `clone(CLONE_VM\|CLONE_VFORK)` desde glibc 2.24. Ejecuta las acciones de archivo (`open`/`close`/`dup2`) y después cierra los fd con `FD_CLOEXEC`. **No permite pausar al hijo antes de `exec`** [V K3] |
| `close_range` | `CLOSE_RANGE_CLOEXEC` desde Linux 5.11 [V K4] |

Que el hijo haga `exec` enseguida **no** hace seguro un `fork` si antes ejecuta lógica de Rust.

### 4.2 Alternativas

| Criterio | A. `fork` en el compositor | B. Auxiliar permanente monohilo | C. `posix_spawn` + intermediario efímero | D. Otro mecanismo |
|---|---|---|---|---|
| Seguridad | Riesgo de deadlock o UB por código no async-signal-safe | Buena | Buena: el hijo es un ejecutable nuevo desde el primer momento | No hay otra opción oficial que permita pausar antes de `exec` |
| Complejidad | Alta (código en el hijo restringido) | Alta (IPC, supervisión) | Media | — |
| Latencia | Mínima | IPC adicional | Un `exec` adicional (~ms) [H] | — |
| Procesos permanentes | 0 | +1 | **0** (el intermediario se convierte en el rol con `exec`) | — |
| Memoria | — | Un residente más | Despreciable: el intermediario sale o se transforma | — |
| Rust | `unsafe pre_exec` | Normal | `posix_spawn` mediante `libc` (`unsafe`) o un crate; dependencia que requiere ADR-0008 | — |
| systemd | Igual | Igual | Igual | — |
| Recuperación | — | Otro punto de fallo | EOF → sale | — |
| Auditoría | Difícil | Media | Fácil: el intermediario es pequeño y de responsabilidad única | — |

**Recomendación: C.** Ejecutable intermediario mínimo; nombre pendiente (D-27); ubicación propuesta `core/compositor/` como segundo `[[bin]]`, porque comparte dueño, release y privilegio.

```text
compositor:  socketpair(rol, SOCK_CLOEXEC); pipe2(sync, O_CLOEXEC)
             posix_spawn(intermediario, file_actions: dup2(rol→3), dup2(sync→4); resto con CLOEXEC → cerrado)
             pidfd_open(pid)
             StartTransientUnit(scope, PIDs=[pid], Slice=session.slice, PartOf=graphical-session.target)
             esperar JobRemoved; GetUnitByPIDFD == scope
             escribir "go" en sync
intermediario: read(4) bloqueante; EOF → exit(1)
               validar fd 3; WAYLAND_SOCKET=3; entorno mínimo (lista blanca)
               execve(rol)
rol:         prctl(PR_SET_DUMPABLE,0) como primera instrucción; conectar; fcntl(3, CLOEXEC); unsetenv(WAYLAND_SOCKET)
```

`dup2` limpia `FD_CLOEXEC` en el destino. Así solo sobreviven los fd 3 y 4 [V K3 + H].

### 4.3 `PR_SET_DUMPABLE`

| Punto | Respuesta |
|---|---|
| Cuándo | Primera operación del rol, después del `exec` [H] |
| ¿Antes del `exec`? | No basta: lo más probable es que `execve` de un binario normal restablezca el atributo [H] [VM]. K5 enumera cambios de UID, `exec` set-ID y capacidades de archivo |
| Qué protege | `PTRACE_ATTACH` [V K5] y, por la comprobación de modo ptrace, `pidfd_getfd` [V-p] |
| Qué no protege | `kill`, `$HOME`, D-Bus, la ventana entre `exec` y `prctl` |
| `/proc/<pid>/fd` | Cambia la propiedad de `/proc/<pid>` (K5 remite a `proc_pid(5)`, no leído) [V-p] |
| Yama | Sigue cubriendo a los no descendientes durante la ventana [V] |
| Mismo UID | No es aislamiento |

---

## 5. C-21 — Política de descriptores

| Clase | Política |
|---|---|
| Lista blanca del rol | fd 3 (socket de rol) y fd 4 (sync, solo para el intermediario; se cierra antes del `exec`) |
| Todo lo demás | Creado con `O_CLOEXEC`/`SOCK_CLOEXEC` en el compositor y cerrado por `posix_spawn` [V K3]. Defensa adicional: `close_range(5, ~0, CLOSE_RANGE_CLOEXEC)` en el intermediario [V K4] |
| DRM, D-Bus, pipes, memfd, logs, configuración | Nunca en la lista blanca. Se exige CLOEXEC en su creación; se revisa por auditoría de código [H] |
| fd heredados por el compositor | `close_range` en el intermediario |
| `WAYLAND_SOCKET` | Lo fija el intermediario; el rol lo borra tras conectar |
| Descendientes del rol | Los roles no lanzan hijos. Si lo necesitan, deben usar la biblioteca de lanzamiento (§C-20), que no hereda el fd |

| Vía de acceso | ¿Mitigada? |
|---|---|
| Herencia por `fork` | Sí, por CLOEXEC y la lista blanca |
| Conservación tras `exec` | Solo el fd 3 |
| Transferencia por D-Bus | Solo si el rol lo envía explícitamente: prohibido [H] |
| Vía `/proc` | Lectura de inodo [V]; reapertura de sockets no documentada [VM] |
| `pidfd_getfd` | Bloqueado por `dumpable=0` y por Yama frente a no descendientes [V-p] |

---

## 6. C-22 — Grafo systemd

### 6.1 Semánticas [V G5, G4]

| Directiva | Arranque | Orden | Parada | Reinicio | Fallo |
|---|---|---|---|---|---|
| `Wants=` | Arrastra | No | No | No | No |
| `Requires=` | Arrastra | No | Se detiene si el otro se detiene explícitamente | Sí | Si el otro falla con `After=`, no arranca |
| `BindsTo=` | Arrastra | No | Se detiene si el otro se detiene **o cae** | — | Sí |
| `PartOf=` | No | No | De la unidad listada a esta (una sola dirección) | Sí | — |
| `After=` / `Before=` | — | Sí | — | — | — |
| `StopWhenUnneeded=yes` | — | — | Se detiene cuando nadie la necesita | — | — |
| `PropagatesStopTo=` | — | — | Propaga peticiones de parada | — | — |

`graphical-session.target` upstream: `Requires=basic.target`, `RefuseManualStart=yes`, `StopWhenUnneeded=yes` [V G4].

Por tanto: **lo arrastra el target de sesión y se detiene solo cuando deja de ser necesario**. Esa es la parada que buscaba la séptima revisión. La versión instalada en Arch es `[VM]`.

### 6.2 Grafo propuesto (nombres no aceptados marcados *)

| Unidad | Tipo | Manager | Dependencias | Lo inicia | Lo detiene | Si cae | En cierre normal |
|---|---|---|---|---|---|---|---|
| greetd | service | sistema | — | systemd | — | — | — |
| Comando de sesión* (`parallax-session`*) | proceso | — | — | greetd [H; fuente no leída] | Él mismo, al salir | Termina la sesión de greetd y logind [H] | Sale |
| `parallax-session.target` | target | usuario | `BindsTo=graphical-session.target`, `Wants=graphical-session-pre.target`, `After=graphical-session-pre.target`, `BindsTo=parallax-compositor.service`, `After=parallax-compositor.service` (política X) | Comando de sesión | Comando de sesión o caída del compositor | — | Parado |
| `graphical-session-pre.target` | target | usuario | upstream: `Before=graphical-session.target`, `StopWhenUnneeded=yes` [V] | Arrastrado | Cuando nadie lo necesita | — | — |
| `graphical-session.target` | target | usuario | upstream [V] | Arrastrado | `StopWhenUnneeded` | — | Se detiene |
| `parallax-compositor.service` | service | usuario | `PartOf=graphical-session.target`, `After=graphical-session-pre.target parallax-settingsd.service`, `Wants=parallax-settingsd.service`, **`Restart=no`** (X) | Target | `PartOf` / target | → el target cae por `BindsTo` → se detiene `graphical-session.target` | Se detiene |
| `parallax-settingsd.service` | service | usuario | `PartOf=graphical-session.target` | Compositor (`Wants`) | `PartOf` | systemd (`Restart=`) | Se detiene |
| Scopes de rol* (Shell, lock, auth…) | scope | usuario | `PartOf=graphical-session.target` en la creación (G1: «all dependency types» se pueden fijar) [V]; `Slice=session.slice` | Compositor (§4) | `PartOf` o el compositor | El compositor los reinicia (catálogo, límites) | Se detienen |
| Apps `app-parallax-…@….service` | service | usuario | `PartOf=graphical-session.target`, `Slice=app.slice` | Biblioteca de lanzamiento | `PartOf` | systemd: sin reinicio | Se detienen |
| Apps `DBusActivatable` | Unidad del paquete | usuario | **No controlamos sus dependencias** [H] | Bus | `KillUserProcesses` o el usuario | — | Pueden sobrevivir [VM] |

---

## 7. C-36 — Arranque desde greetd

**Sin fuente oficial de greetd (502):** todo esta sección es `[H]` salvo lo indicado.

| Opción | Descripción | Ventajas | Inconvenientes |
|---|---|---|---|
| G-a | greetd ejecuta el compositor directamente | Simple | Sin el grafo systemd: los servicios se ordenan a mano |
| G-b | `systemctl --user start --wait parallax-session.target` | Sin binario propio | `--wait` verificado solo por resumen [V-p G3]; que espere a un **target** hasta que se detenga no está demostrado |
| **G-c** | Comando de sesión propio y mínimo (nombre*: D-28): arranca el target, espera su estado y sale al detenerse | Control de errores explícito; evita sesiones parcialmente iniciadas | Un binario más (no residente pesado; espera bloqueada) |

**Recomendación: G-c.**

1. Proceso en primer plano: el comando de sesión.
2. Arranca `parallax-session.target` y espera el `JobRemoved`.
3. Si el resultado no es `done`, detiene el target y sale con un código distinto de 0. greetd vuelve al greeter [H].
4. Si es `done`, vigila `ActiveState` del target hasta que pase a `inactive` y entonces sale.

El resultado del arranque llega a greetd por el código de salida [H].

**Fallo del compositor durante el arranque:** con `BindsTo=` y `After=`, el target no se activa [V G5]. El comando de sesión ve el fallo y limpia.

**C-38 (unidad del greeter):** greetd lanza el greeter como comando de su `default_session` [H; fuente no leída]. Si es así, `parallax-greeter.service` (`arquitectura.md` §2) sería incorrecto. Pendiente de leer la fuente.

---

## 8. D-24 — Recuperación de aplicaciones

### 8.1 Hechos

| Punto | Resultado |
|---|---|
| Error de conexión Wayland | Los errores son fatales y el display ya no se puede usar [V-p W1] |
| ¿El proceso termina? | **No hay garantía** [V-p W1]: depende de la aplicación. Muchos toolkits terminan [H]; otros procesos siguen vivos (tareas de fondo, PipeWire, D-Bus) [H] |
| Buffers y superficies | Las destruye el compositor al morir [H] |
| Apps `DBusActivatable` y servicios D-Bus | Pueden seguir vivos y en el bus [H] |
| Apps en sandbox | Su proceso (bwrap) depende de su unidad [H] |
| Unidades de aplicación | Se mantienen mientras viva el proceso [V G5] |

### 8.2 Generaciones

| Identificador | Creación | Pertenencia | Registro | Invalidación |
|---|---|---|---|---|
| Sesión | Id de sesión de logind (existente) [V-p L1] | Todo lo arrastrado por `graphical-session.target` | `PartOf=` | Al terminar la sesión |
| Compositor | Cada arranque | Scopes de rol | Nombre o `Description` del scope (D-25) | Al morir el compositor (con X coincide con la sesión) |
| Rol | Cada reinicio de un rol | Un scope | Nombre del scope | Cuando el proceso sale |
| App | Desktop File ID + `<RANDOM>` | Su unidad | Nombre de la unidad (S1) | — |

**Con X**, la generación del compositor coincide con la de la sesión. Solo hace falta gestionar aparte la **generación de rol**, para los reinicios de roles dentro de una misma vida del compositor.

**Ámbito:** `systemd --user` es único por usuario [H]. `graphical-session.target` está activo si hay alguna sesión gráfica [V G5]. **PARALLAX v1 debe admitir una sola sesión gráfica por usuario** (D-29); si no, la limpieza afectaría a otra sesión del mismo usuario.

### 8.3 Políticas

| Criterio | X: terminar la sesión gráfica | Y: reiniciar el compositor en la misma sesión | Z: híbrida |
|---|---|---|---|
| Seguridad | **Bloqueo garantizado por construcción**: nueva autenticación en el greeter | Requiere D-26 y limpieza por generación | Igual que Y |
| Integridad de datos | Se pierde lo no guardado. Nota: en Y también se pierde, porque la app ya no tiene Wayland | Igual | Igual |
| Compatibilidad | Total | Apps sin UI vivas y confusas | Requiere declarar «recuperable» |
| Complejidad | **Baja** | Alta | Muy alta |
| Tiempo de recuperación | Vuelta al login | Más rápido | Medio |
| Sesión Locked | Segura | Depende de D-26 | Depende |
| logind | Sesión nueva | Misma sesión | Misma sesión |
| systemd | `BindsTo` + `PartOf` | Restart, generación, limpieza | Lo anterior + declaración |
| Recursos | Limpios | Posibles huérfanos | Parcial |
| Implementación | Directa | Compleja | Compleja |

**Recomendación para v1: X.**
- Separa continuidad de sesión y continuidad de aplicaciones.
- Cumple que una caída nunca desbloquea.
- Ninguna app conserva una generación anterior.
- No hace falta declarar procesos persistentes.
- El compositor no gestiona aplicaciones.

La restauración cooperativa (`arquitectura.md` §5: las apps PARALLAX guardan su estado y se relanzan) ocurre en la sesión **nueva**.

Los servicios de fondo del usuario que no son `PartOf=graphical-session.target` (p. ej. PipeWire) sobreviven. Su destino al cerrar la sesión de logind depende de `KillUserProcesses` y del lingering [VM].

**Y queda como evolución v2** (D-24), junto con un protocolo de restauración.

---

## 9. Revisión cruzada de seguridad

| # | Comprobación | Resultado |
|---|---|---|
| 1 | Un rol antiguo no accede al socket nuevo | Sí: los fd mueren con el compositor (T10) [H] |
| 2 | Un agente polkit antiguo no bloquea al nuevo | Con X, `PartOf` lo detiene con la sesión [V G5]. En reinicios de rol, la unicidad por rol exige el scope anterior en `STOPPED` [VM] |
| 3 | Sin generación nueva antes de controlar la anterior | Con X, no hay generación nueva en la misma sesión. En roles, la máquina de estados de la séptima revisión |
| 4 | Una sesión bloqueada sigue bloqueada tras una caída | Compositor: sí, con X. Lock: sí [R `compositor.md` §4.6] |
| 5 | Una app no imita la superficie de autenticación | Mitigación parcial (T9) |
| 6 | No se confunde mismo UID con frontera | Documentado (§4.3, §5) |
| 7 | Los fd privilegiados no llegan a las apps | Sí: la biblioteca de lanzamiento no hereda fd del compositor (§5) |
| 8 | resourced no compite con systemd | Sí (ADR-0014 [R]) |
| 9 | La biblioteca de lanzamiento no se vuelve privilegiada | Sí: mismo UID, sin capacidades |
| 10 | La limpieza no afecta a otras sesiones | Solo con D-29 (una sesión gráfica por usuario) |

---

## 10. Contradicciones documentales

| ID | Contradicción |
|---|---|
| C-23b | La séptima revisión (§6.1) decía que `PIDs` no estaba documentada: **incorrecto** (G1) |
| C-23c | La séptima revisión (§6.2) proponía `fork` en el compositor y `PR_SET_DUMPABLE` antes del `exec`. Sustituido por §4 |
| C-29 | `arquitectura.md` §5: el Shell se reinicia con systemd. Con roles en scope, el reinicio pasa al compositor |
| C-34 | `arquitectura.md` §2: el compositor figura como target |
| C-35 | No se menciona `graphical-session*.target` |
| C-36 | Arranque de la sesión desde greetd sin definir |
| C-37 | `compositor.md` §11.5 («reiniciar el compositor cuando corresponda»). Con X, el compositor no se reinicia dentro de la sesión |
| C-38 | `parallax-greeter.service` frente al comando de greetd |
| C-39 (nuevo) | `arquitectura.md` §5, «Caída del compositor» («restauración de sesión… se relanzan») es compatible con X si la restauración ocurre en la sesión nueva. Hay que aclararlo |

---

## 11. Propuestas de corrección (no aplicadas)

### P1. `architecture/arquitectura.md` §2 — fila del compositor

- **Texto actual:** «`parallax-session.target`»
- **Texto propuesto:** «`parallax-compositor.service` (usuario), `PartOf=graphical-session.target`, sin reinicio automático; `parallax-session.target` lo declara con `BindsTo=` y `After=`.»
- **Justificación:** G4, G5; política X.
- **ADR relacionado:** —
- **Revisa:** Linux.
- **Decisión:** D-24.
- **Prueba:** VM.

### P2. `architecture/arquitectura.md` §2 — filas del Shell, del lock y del agente

- **Texto propuesto:** «Proceso de rol: lo lanza el compositor mediante un intermediario efímero y lo registra en un scope transitorio (`PIDs`; `Slice=session.slice`; `PartOf=graphical-session.target`). El compositor lo reinicia según el catálogo de roles: 5 por minuto con backoff; al superar el límite, overlay.»
- **Justificación:** G1, K1–K3.
- **ADR relacionado:** 0015, 0006, 0010.
- **Revisa:** Linux y seguridad.
- **Decisión:** D-18 [P].
- **Prueba:** VM.

### P3. `architecture/arquitectura.md` §5 — Arranque

- **Texto propuesto:** «greetd → comando de sesión PARALLAX (nombre pendiente) → arranca `parallax-session.target` (`BindsTo=graphical-session.target`, `After=graphical-session-pre.target`) → espera su resultado; si falla, lo detiene y sale con error. El proceso permanece en primer plano hasta que el target se detiene.»
- **Justificación:** G4, G5; greetd [H].
- **ADR relacionado:** —
- **Revisa:** Linux.
- **Decisión:** D-28.
- **Prueba:** VM y lectura de greetd(5).

### P4. `architecture/arquitectura.md` §5 — Caída del compositor

- **Texto propuesto:** «En v1, la caída del compositor termina la sesión gráfica: `parallax-session.target` cae por `BindsTo=`, `graphical-session.target` se detiene por `StopWhenUnneeded`, y las unidades con `PartOf=` (roles y apps) se detienen. El usuario vuelve al greeter. Las apps PARALLAX restauran su estado en la sesión nueva. No se garantiza el guardado.»
- **ADR relacionado:** 0015.
- **Decisión:** D-24 [P recomendado: afecta a la experiencia].

### P5. `specifications/compositor.md` §11.5

- **Texto actual:** «Reiniciar el compositor cuando corresponda.»
- **Texto propuesto:** «En v1 el compositor no se reinicia dentro de la misma sesión (ver `arquitectura.md` §5). El bloqueo se conserva por construcción: la sesión nueva requiere autenticación.»
- **Decisión:** D-24.

### P6. `security/modelo-seguridad.md` — nueva fila «Creación de roles»

- **Texto propuesto:** «Los roles se crean con `posix_spawn` de un intermediario efímero. Solo hereda el fd del socket de rol (lista blanca; resto CLOEXEC más `close_range`). El rol aplica `PR_SET_DUMPABLE=0` al arrancar. No es aislamiento frente al mismo UID.»
- **ADR relacionado:** 0015.
- **Revisa:** seguridad.
- **Decisión:** D-22.
- **Prueba:** VM.

### P7. `specifications/lockscreen-greeter.md` — Greeter

- **Texto propuesto (añadir):** «greetd ejecuta el greeter como comando de su sesión por defecto (pendiente de confirmar en greetd(5)); no es una unidad systemd propia.»
- **Decisión:** C-38.

### P8. `architecture/servicios.md` §14

- **Texto propuesto (añadir):** «Los servicios de sesión gráfica declaran `PartOf=graphical-session.target`. Los que deban sobrevivir a la sesión gráfica no lo declaran y se documentan explícitamente.»

### P9. ADR-0015 — propuesta de revisión (sin cambiar su estado)

- **Texto propuesto (añadir a Consecuencias):** «El relanzamiento de los componentes privilegiados lo hace el compositor (scope transitorio), porque systemd no puede recibir un fd ya abierto por el compositor (`OpenFile=` abre rutas; el almacén de fd solo acepta los de la propia unidad).»
- **Revisa:** seguridad.
- **Decisión:** D-18, D-22.

### P10. ADR-0006 y ADR-0010

- **Cambio:** solo referencia al ciclo de vida del Shell y del agente como roles (P2). Sin otros cambios.

---

## 12. Matriz de hallazgos

| ID | Documento | Evidencia | Problema | Corrección | Dependencias | Responsable | Decisión pendiente | Prueba | Estado |
|---|---|---|---|---|---|---|---|---|---|
| C-20 | `arquitectura.md` §5, `ipc.md` §6.1, `dock.md` | [V], [R] | Lanzamiento | Crate común (séptima revisión) | 0006, 0007 | Arquitectura, Rust | Sí (D-21) | No | Corregido en propuesta |
| C-21 | `modelo-seguridad.md`, ADR-0015 | [V K1–K5] | fd y `dumpable` | §4, §5 (P6) | 0015 | Seguridad | Sí (D-22 [P]) | VM | Abierto |
| C-22 | `arquitectura.md` §5 | [V G4, G5] | Grafo | §6 (P1, P3, P4, P8) | — | Linux | Sí (D-24) | VM | Corregido en propuesta |
| C-23 | `arquitectura.md` §2 | [V G1, K1–K3] | Registro y spawn | §3, §4 (P2) | 0015 | Linux, Rust | Sí (D-18 [P], D-27) | VM | Abierto |
| C-29 | `arquitectura.md` §5 | [V] | Reinicio del Shell | P2 | 0015, 0006 | Arquitectura | Sí (D-18 [P]) | VM | Abierto |
| C-34 | `arquitectura.md` §2 | [R] | Compositor como target | P1 | — | Linux | No | No | Corregido en propuesta |
| C-35 | Varios | [V G4] | `graphical-session` | P3, P8 | — | Linux | No | VM | Corregido en propuesta |
| C-36 | `arquitectura.md` §5 | [H], greetd no leído | Arranque | G-c (P3) | — | Linux | Sí (D-28) | VM + fuente | Abierto |
| C-37 | `compositor.md` §11.5 | [R] | Bloqueo tras caída | X (P5) | 0015 | Compositor | Sí (D-24) | VM | Corregido en propuesta |
| C-38 | `arquitectura.md` §2, `lockscreen-greeter.md` | [H] | Greeter | P7 | — | Linux | No | Fuente | No verificable |
| C-39 | `arquitectura.md` §5 | [R] | Restauración | P4 | — | Arquitectura | No | No | Corregido en propuesta |
| R-20 | — | [V G1] | `PIDs` | — | — | Linux | — | VM | **API documentada, sin verificar en Arch** |
| R-21 | — | [V G5] | Agente antiguo | `PartOf` + unicidad | 0010 | Seguridad | — | VM | Abierto |
| R-22 | — | [V K5] | Ventana sin `dumpable` | `prctl` en el rol | — | Seguridad | — | VM | Abierto |
| R-23 | — | [H] | Suplantación visual | Diseño | 0010 | Seguridad | D-22 | — | Abierto |
| R-24 | — | — | Experiencia tras caída | X sustituye a D-26 | — | Propietario | D-24 | — | Corregido en propuesta |
| R-25 | — | [V G4] | Propagación de parada | `StopWhenUnneeded` | — | Linux | — | VM | Corregido en propuesta |
| R-26 | — | [V G5] | Huérfanos | `PartOf` | — | Linux | — | VM | Corregido en propuesta |
| D-18 | — | — | Supervisión de roles | P2 | 0015 | **Propietario** | Sí | VM | Abierto |
| D-21 | — | — | Crate de lanzamiento | — | — | Arquitectura, Rust | Sí | No | Abierto |
| D-22 | — | — | Rol `auth` y límites | — | 0015 | Seguridad + **propietario** | Sí | VM | Abierto |
| D-23 | — | — | Cierre de sesión desde Locked | — | — | **Propietario** | Sí | No | Abierto |
| D-24 | — | — | Política ante caída | **X** | — | Arquitecto; se recomienda confirmación del propietario | Sí | VM | Abierto |
| D-25 | — | — | Formato del nombre de los scopes | — | — | Linux | Sí | VM | Abierto |
| D-26 | — | — | Arrancar en Locked | Sustituida por X para el compositor; se mantiene para lock | — | **Propietario** | Solo si Y | — | Abierto |
| **D-27** | — | — | Nombre y forma del intermediario y de `posix_spawn` (`libc` o crate) | — | 0008 | Rust, seguridad | Sí | No | Abierto |
| **D-28** | — | — | Comando de sesión (G-c) | — | — | Linux | Sí | VM | Abierto |
| **D-29** | — | — | Una sola sesión gráfica por usuario en v1 | — | — | **Propietario** | Sí | VM | Abierto |

Ningún hallazgo está corregido en los archivos originales.

---

## 13. Pruebas y spikes pendientes (no autorizados)

| Prueba | Tipo | Hallazgo o decisión |
|---|---|---|
| `StartTransientUnit` con `PIDs` en el user manager de Arch; tipo; PID muerto; garantía de `JobRemoved=done` | VM | R-20 |
| `GetUnitByPIDFD` (versión de systemd en Arch) | VM | C-23 |
| `PartOf=` y `Slice=` como propiedades de creación de un scope | VM | C-22 |
| `posix_spawn` + intermediario: solo sobreviven los fd 3 y 4 | VM | C-21 |
| `PR_SET_DUMPABLE` después de `execve` | VM | R-22 |
| `graphical-session.target` instalado en Arch (`StopWhenUnneeded`) | VM | R-25 |
| Cadena `BindsTo` al caer el compositor; vuelta al greeter | VM | D-24 |
| `KillUserProcesses` efectivo; supervivencia de PipeWire y de apps `DBusActivatable` | VM | §8.3 |
| Leer greetd(5) cuando el servidor responda | Fuente | C-36, C-38 |

---

## 14. Decisiones pendientes del propietario

| ID | Decisión |
|---|---|
| D-18 | Sustituir `Restart=always` del Shell por la supervisión del compositor |
| D-22 | Aprobar los límites de seguridad y el rol `auth` (revisión de ADR-0015) |
| D-23 | Cierre de sesión desde Locked |
| D-24 | Política X en v1. La recomienda el arquitecto, pero cambia la experiencia ante fallos: **se recomienda confirmación del propietario** |
| D-29 | Una sola sesión gráfica por usuario en v1 |
| D-01, D-08, D-09, D-10, D-11, D-13, D-16 | Siguen como en revisiones anteriores |
| — | Autorizar las pruebas de §13 |

---

## 15. Riesgos residuales

| ID | Riesgo |
|---|---|
| R-20 | `PIDs` puede comportarse distinto en la versión de Arch |
| R-27 (nuevo) | La política X pierde todas las ventanas tras una caída del compositor. Es inevitable en Wayland, pero ahora el usuario además vuelve al login |
| R-28 (nuevo) | Las apps `DBusActivatable` y los servicios de terceros no declaran `PartOf` y pueden sobrevivir a la sesión |
| R-29 (nuevo) | Sin la documentación de greetd, el flujo G-c puede no encajar con su modelo de sesión |
| R-30 (nuevo) | `posix_spawn` desde Rust requiere `unsafe` o una dependencia (ADR-0008) |
| R-22, R-23 | Siguen abiertos |

---

## 16. Veredicto: `LISTA CON CONDICIONES EXPLÍCITAS`

### Objetivos de la revisión

| Objetivo | Resultado |
|---|---|
| Registro de scopes documentado correctamente | **Sí** (G1). La séptima revisión queda corregida. Comportamiento en Arch: `[VM]` |
| Creación de procesos segura en Rust multihilo | **Sí, con diseño**: `posix_spawn` + intermediario, sin código en el hijo [V K1–K3]. Implementación: D-27 |
| Política de herencia de fd verificable | **Sí** (§5). Comprobación: `[VM]` |
| Grafo systemd con semántica correcta | **Sí** (G4, G5). Unidad de Arch: `[VM]` |
| Flujo de arranque desde greetd | **Definido como hipótesis**: falta la fuente oficial de greetd (C-36) |
| Recuperación de apps tras una caída especificada | **Sí**: política X |
| Política de bloqueo segura | **Sí con X**; con Y dependería de D-26 |
| Responsabilidades separadas | **Sí**: systemd (unidades), logind (sesión), compositor (Wayland y roles), greetd (login) |

### Condiciones

1. Leer greetd(5) antes de cerrar C-36 y C-38.
2. Que el propietario decida o confirme D-18, D-22, D-24 y D-29.
3. Registrar como spikes pendientes las pruebas de §13. Son condición de viabilidad, no de redacción.
4. Elegir el mecanismo de `posix_spawn` (D-27) conforme a ADR-0008.

### Lo que este veredicto no hace

- No aprueba nada.
- No acepta ningún ADR.
- No autoriza VM, spikes ni Fase 1.
- La Fase 0 sigue en `IN_PROGRESS`.
