# PARALLAX OS — Séptima revisión técnica complementaria (C-20, C-21, C-22, C-23)

- **Fecha:** 2026-11-19
- **Fase 0:** `IN_PROGRESS`
- **ADR:** todos en `Propuesto`
- **Modo:** solo lectura. Este informe está en el scratchpad de la sesión y no forma parte del repositorio. No es un documento normativo.

**Convención de evidencia**

| Marca | Significado |
|---|---|
| `[V]` | Garantía documentada en fuente oficial (lectura completa salvo que se indique) |
| `[V-p]` | Documentada, pero la lectura fue parcial o resumida |
| `[R]` | Documentado en el repositorio |
| `[H]` | Inferencia técnica |
| `[VM]` | Requiere prueba en VM |
| `[HW]` | Requiere hardware |
| `[P]` | Decisión del propietario |

No se ha observado ningún comportamiento: no se ejecutó nada.

---

## 1. Resumen ejecutivo

El modelo funciona como sistema **solo con tres correcciones** a la sexta revisión y **una condición técnica sin verificar**.

**Correcciones**

1. **C-20.** La biblioteca de lanzamiento no puede vivir en `shell/shell/` si la van a consumir las aplicaciones C6. Se propone un crate común en `crates/`, creado en la fase del Shell (Fase 3), no en Fase 1. El crate no hace nada de Wayland: el token lo pide el solicitante y se lo pasa.
2. **C-22.** `parallax-session.target` debe declarar `BindsTo=graphical-session.target` (patrón de systemd.special [V]) y los componentes deben declarar `PartOf=`. La sexta revisión describía el grafo de forma ambigua. La parada de `graphical-session.target` cuando la sesión termina depende de cómo esté definida esa unidad en el paquete de systemd. Eso es `[VM]`, no una garantía.
3. **C-23.** R2 necesita una **generación del compositor** y una limpieza de los roles de la generación anterior **antes** de lanzar los nuevos. Sin ella, un agente polkit antiguo puede seguir registrado en polkitd e impedir el registro del nuevo (S5: «registration … can fail») [V].

**Condición sin verificar**

En la página oficial de `org.freedesktop.systemd1` **no encontré la propiedad `PIDs`** para crear un scope transitorio con procesos. Solo encontré `Scope.AttachProcesses()` [V]. La vía exacta de registro del hijo en el scope queda `[VM]`, y por tanto **R2 es viable con condiciones, no demostrada**.

**Seguridad**

- `PR_SET_DUMPABLE=0` en los roles impide `PTRACE_ATTACH` desde cualquier proceso del mismo UID sin `CAP_SYS_PTRACE` [V]. Es una mitigación más fuerte que Yama, que solo restringe `ATTACH`/`TRACEME` y nada dice de `/proc/<pid>/fd` [V].
- `pidfd_getfd` exige `PTRACE_MODE_ATTACH_REALCREDS` [V].
- Ninguna de estas medidas convierte el socket de rol en aislamiento del sistema operativo.

**Veredicto (§14): `LISTA CON CONDICIONES EXPLÍCITAS`.**

---

## 2. Alcance y fuentes consultadas

### Repositorio [R]

Leídos en esta sesión:

- `architecture/{arquitectura,ipc,eventos,servicios}.md`
- `specifications/{compositor,dock,shell,lockscreen-greeter}.md`
- `security/{modelo-seguridad,dependencias}.md`
- ADR-0007, ADR-0010, ADR-0014, ADR-0015
- `AGENTS.md`
- `.claude/agents/{ingeniero-seguridad,ingeniero-compositor}.md`

`git status` es idéntico al del inicio de la sesión.

Búsquedas de esta revisión:

- `graphical-session` **no aparece** en el repositorio.
- `parallax-session.target` aparece en `arquitectura.md` §2 (tabla de procesos, columna «Unidad» del compositor) y §5 (flujo de arranque).

### Fuentes externas

| Ref | Fuente | Lectura |
|---|---|---|
| K1 | Yama, docs.kernel.org/admin-guide/LSM/Yama.html | Completa |
| K2 | pidfd_getfd(2), man7.org | Completa |
| K3 | proc_pid_fd(5), man7.org | Completa |
| K4 | ptrace(2), sección «Ptrace access mode checking» | Parcial: el algoritmo y el párrafo de Yama solo resumidos |
| K5 | PR_SET_DUMPABLE(2const), man7.org | Completa |
| Y1 | org.freedesktop.systemd1(5) | Completa (8 bloques) |
| Y2 | systemd.scope(5) | Completa |
| Y3 | systemd.special(7): entradas `graphical-session*.target` | Completa en esas entradas |
| Y4 | systemd.unit(5): `BindsTo=`, `PartOf=`, `Requires=`, `PropagatesStopTo=`, `StopWhenUnneeded=` | Completa en esas directivas; la tabla de inversas, resumida |
| Y5 | systemd.kill(5) | Completa |
| Y6 | systemd.service(5), sd_notify(3) | Completa (sexta revisión) |
| L1 | org.freedesktop.login1(5): `TerminateSession`, `Session.Terminate` | Completa en esos métodos (sexta revisión) |
| L2 | logind.conf(5): `KillUserProcesses=` | Completa (sexta revisión) |
| D1 | D-Bus Specification | ~96 % literal (sexta revisión) |
| P1 | polkit: Authority, AuthenticationAgent, PolkitAgentListener, PolkitAgentSession, polkit(8) | Completa (sexta revisión) |
| E1 | Desktop Entry Spec: file-naming, recognized-keys, exec-variables, dbus | Completa (sexta revisión) |
| W1 | xdg-activation-v1, xdg-shell (espejo wayland.app) | Parcial |

---

## 3. C-20 — Biblioteca de lanzamiento

### 3.1 Contradicción confirmada

La sexta revisión, §7, coloca el módulo en `shell/shell/` y prevé que lo consuman aplicaciones C6.

Si una app importa ese módulo, aparece una dependencia **C6 → C5 de compilación**. Viola dos reglas:

- «ningún binario importa a otro binario» (propuesta, §7).
- El principio de dependencias entre capas (`arquitectura.md` §1 [R]).

### 3.2 Qué necesita cada capa

| Capa | ¿Lanza aplicaciones? | Qué necesita |
|---|---|---|
| C4 compositor | Solo roles fijos y la terminal de recuperación | **No** necesita la biblioteca: sin Desktop Entry, sin `Exec` |
| C5 Shell | Sí (dock, launcher, notificaciones) | Resolución, validación, activación |
| C6 apps PARALLAX | Sí (p. ej. Files: «abrir con») | Igual que el Shell |
| Terceros | Con su propio toolkit | Nada de PARALLAX |
| Apps en sandbox | Vía portal (`OpenURI`) | Nada de PARALLAX [R `servicios.md` §13] |

### 3.3 Arquitectura mínima propuesta

```text
crates/<launch>  (nombre: D-21; crate común, no de Fase 1; se crea con el primer consumidor real, Fase 3)
 ├─ desktop_entry: resolver Desktop File ID → archivo (E1 file-naming), validar claves (E1 recognized-keys)
 ├─ exec: parsear Exec y field codes con las reglas de E1 exec-variables; nunca vía shell
 ├─ activation: si DBusActivatable=true → org.freedesktop.Application.Activate/Open/ActivateAction (E1 dbus)
 └─ unit: si no → StartTransientUnit del user manager con nombre app-parallax-<AppID>@<RANDOM>.service (S1)
Entrada obligatoria: token xdg-activation opcional, obtenido por el solicitante.
```

**Responsabilidades**

| Componente | Responsable de | **No** responsable de |
|---|---|---|
| Solicitante (Shell o app) | Pedir el token a su conexión Wayland con el serial de la interacción (W1); llamar a la biblioteca | Validar tokens; supervisar el proceso |
| Toolkit (`px-ui`, futuro) | Exponer al solicitante la petición de token de su superficie | Lanzar |
| Biblioteca | Resolver, validar, construir la llamada D-Bus o la unidad, pasar el token (`XDG_ACTIVATION_TOKEN` o `activation-token` en `platform_data`, E1) | Wayland; reintentos; política de recursos |
| Compositor | Generar, validar o ignorar tokens (W1); foco | Desktop Entry, `Exec`, unidades de aplicaciones |
| D-Bus / bus | Activar apps `DBusActivatable` | — |
| systemd --user | Iniciar y supervisar las unidades de app | Validar Desktop Entries |

**Separación interna**

- `desktop_entry` y `exec` son funciones puras, testeables sin E/S.
- `activation` y `unit` hacen E/S D-Bus.
- Todas viven en un crate para no duplicar la política de lanzamiento. Dividirlo en más crates no tiene justificación hoy (criterios de crecimiento).

**¿Servicio independiente?** No.

- Para apps normales no hay frontera de privilegio [H]: cualquier proceso del usuario puede crear unidades en su `systemd --user`.
- Un servicio no tendría serial de interacción, y el compositor puede ignorar sus tokens (W1).

**¿Una app puede lanzar otra sin el Shell?** Sí, con la biblioteca.

**Duplicación de política:** se evita con un solo crate y reglas tomadas de E1.

### 3.4 Repercusión en el calendario de crates

Esto no añade un crate a Fase 1: los cuatro de Fase 1 no cambian. La regla «módulo y después crate» se sustituye por «crate común desde el primer consumidor, porque el segundo ya está previsto (Fase 4, ADR-0011, roadmap)». Decisión D-21.

---

## 4. C-21 — Capacidades Wayland y seguridad

### 4.1 Conceptos que hay que separar

| Concepto | Qué es en PARALLAX |
|---|---|
| Identidad de conexión | La conexión entró por el socket del rol X, que el compositor creó con `socketpair` [R `ingeniero-compositor.md`; detalle H] |
| Capacidad de acceso a un rol | Posesión del extremo del socket (fd) [R `ingeniero-seguridad.md` regla 1] |
| Autenticación | **No existe** autenticación del cliente. La posesión del fd sustituye a una prueba [R] |
| Autorización | Los globals que el compositor anuncia o acepta en esa conexión [R] |
| Privilegios del proceso | Ninguno: UID del usuario y sin capacidades del kernel [R ADR-0010 §2] |
| Seguridad de la superficie de autenticación | Política del compositor sobre presentación y foco (ADR-0010 §6) [R] |

### 4.2 Hechos del kernel

| Mecanismo | Hecho | Fuente |
|---|---|---|
| Yama `ptrace_scope=1` | `PTRACE_ATTACH` solo a descendientes (salvo `PR_SET_PTRACER`) | [V K1] |
| Ámbito de Yama | Solo `PTRACE_ATTACH` y `PTRACE_TRACEME`. No menciona `/proc/<pid>/fd`, `process_vm_readv` ni `pidfd_getfd` | [V K1] |
| Yama y comprobaciones `ATTACH` | Yama restringe las comprobaciones de modo `ATTACH`, que incluyen `pidfd_getfd` (`ATTACH_REALCREDS`) | [V-p K4 resumido] + [V K2] → [H] |
| `pidfd_getfd` | Exige `PTRACE_MODE_ATTACH_REALCREDS`; el fd devuelto lleva `FD_CLOEXEC` | [V K2] |
| `/proc/<pid>/fd` | Leer los enlaces exige `PTRACE_MODE_READ_FSCREDS`, que Yama no restringe; un socket aparece como `socket:[inode]` | [V K3] + [V-p K4] |
| Reabrir un socket desde `/proc/<pid>/fd/N` | **No documentado** en K3 | [H] no se puede; [VM] |
| `PR_SET_DUMPABLE=0` | Impide `PTRACE_ATTACH` y cambia la propiedad de `/proc/<pid>`; se restablece al cambiar de UID o con un exec set-ID | [V K5] |
| Comprobación de modo ptrace | Deniega si el proceso no es «dumpable», salvo `CAP_SYS_PTRACE` | [V-p K4 resumido] |

**Conclusión.** Para un rol con `PR_SET_DUMPABLE=0`, un proceso del mismo UID sin `CAP_SYS_PTRACE` no puede hacer `ptrace` ni `pidfd_getfd`. Leer `/proc/<pid>/fd` probablemente también queda limitado por la propiedad de `/proc/<pid>` (K5 remite a proc_pid(5), no leído) [H].

**No es aislamiento.** El mismo UID puede:
- Leer y modificar `$HOME`, incluida la configuración.
- Suplantar nombres D-Bus libres [V D1/S4b].
- Matar el proceso.
- Lanzar procesos.
- Interferir antes del `prctl`, en la ventana entre fork y `prctl` [H].

### 4.3 Modelo de amenazas

| # | Amenaza | Actor | Estado | Mitigación o razón |
|---|---|---|---|---|
| T1 | Cliente del socket público pide globals de rol | Cualquier cliente | **Mitigada por diseño** | El socket público no anuncia globals de rol [R] |
| T2 | Cliente en sandbox obtiene un rol | Flatpak | **Mitigada por diseño** | `security-context-v1` nunca recibe globals privilegiados [R] |
| T3 | Descendiente del rol hereda el fd | Hijos del rol | Mitigada parcialmente | `CLOEXEC` tras conectar, limpiar `WAYLAND_SOCKET`; los roles no deben lanzar hijos [H]; [VM] |
| T4 | Proceso del mismo UID, no descendiente, roba el fd con ptrace o `pidfd_getfd` | Mismo UID | Mitigada parcialmente | `PR_SET_DUMPABLE=0` [V K5]; Yama=1 como defensa adicional [V K1]; ventana fork→`prctl` [H] |
| T5 | Mismo UID lee `/proc/<pid>/fd` | Mismo UID | Mitigada parcialmente | Solo ve inodos [V K3]; reabrir el socket no está documentado [VM] |
| T6 | Mismo UID con `CAP_SYS_PTRACE` o root | Admin / root | **No mitigada** | Fuera del modelo (TCB) [R `modelo-seguridad.md`] |
| T7 | Mismo UID mata el rol o el compositor | Mismo UID | **No mitigada** | Disponibilidad; el límite de reinicios lleva al overlay |
| T8 | Mismo UID manipula `$HOME` o la configuración | Mismo UID | **No mitigada** | Límite reconocido en ADR-0015 regla 6 [R] |
| T9 | Cliente ordinario imita visualmente el diálogo de autenticación | Cualquier cliente | Mitigada parcialmente | Solo el rol `auth` presenta en ese nivel, con foco exclusivo [R ADR-0010 §6]. **No evita** que una ventana normal *parezca* un diálogo: hace falta un indicador visual que una app no pueda reproducir, cuya viabilidad queda pendiente de diseño |
| T10 | Proceso de un rol antiguo (tras reiniciar el compositor) presenta una superficie de rol | Rol antiguo | **Mitigada por diseño** | Su fd pertenecía al compositor muerto; el nuevo no le da conexión de rol [H, por construcción] |
| T11 | Agente polkit antiguo sigue registrado en polkitd | Rol antiguo | Pendiente de prueba | Limpieza de generación (§6) [VM] |

### 4.4 Reparto documental

| ADR-0015 | Matriz de protocolos del compositor (`ingeniero-compositor.md`) |
|---|---|
| Principio: capacidad = fd entregado por el compositor | Lista de globals por rol |
| Catálogo cerrado de roles (shell, lock, auth, portal, settings-output) | Rol `auth`: globals mínimos |
| Límites explícitos: T6–T8 no mitigadas; no hay aislamiento entre procesos del mismo UID | Nivel de presentación y foco exclusivo de `auth` |
| Higiene: `PR_SET_DUMPABLE=0`, `CLOEXEC`, limpiar el entorno, sin hijos | Prioridad entre Locked y `auth` (nunca encima de Locked) |
| Revocación: cerrar la conexión; generación del compositor (§6) | — |
| Relanzamiento por el compositor (R2) | — |

---

## 5. C-22 — Cierre de sesión y grafo systemd

### 5.1 Hechos

| Directiva | Semántica | Fuente |
|---|---|---|
| `BindsTo=` | Como `Requires=`, y además la unidad se detiene si la unidad listada se detiene inesperadamente o falla. Conviene combinarlo con `After=`. Inversa: `BoundBy=` | [V Y4] |
| `PartOf=` | Propaga parada y reinicio **de la unidad listada a esta**. «one-way dependency» | [V Y4] |
| `Requires=` | Esta unidad se detiene si la listada se detiene explícitamente | [V Y4] |
| `After=` | Solo orden; no propaga nada | [V Y4] |
| `PropagatesStopTo=` / `StopPropagatedFrom=` | Propagan peticiones de parada | [V Y4] |
| `graphical-session.target` | «A target for a particular session … starts and stops `graphical-session.target` with `BindsTo=`»; los servicios gráficos usan `PartOf=graphical-session.target`; los servicios los arrancan los `Wants=`/`Requires=` del target de sesión | [V Y3] |
| `graphical-session-pre.target` | Debe arrancarse antes del target de sesión | [V Y3] |
| `KillMode=control-group` (por defecto) | Al detener, mata todo el cgroup de la unidad, incluidos los scopes | [V Y5] |
| `TerminateSession` | «forcibly» | [V L1] |
| `KillUserProcesses=` | Mata los procesos del usuario al cerrar sesión; valor por defecto upstream `yes`; el valor efectivo en Arch es `[VM]` | [V L2] |

**Matiz sobre Y3.** Y3 afirma que el target de sesión «arranca y detiene» `graphical-session.target` mediante `BindsTo=`. Pero `BindsTo=`, según Y4, es una dependencia de **esta** unidad sobre la listada: la propagación de la parada hacia `graphical-session.target` no se deduce de Y4. Lo más probable es que la defina la propia unidad `graphical-session.target` del paquete (p. ej. con `StopWhenUnneeded=`) [H]. Verificación: inspeccionar la unidad instalada en la VM [VM].

### 5.2 Hallazgos documentales nuevos

- **C-34.** `arquitectura.md` §2 pone `parallax-session.target` como «Unidad» del compositor. Un target no contiene procesos: el compositor necesita su propio `.service` [R].
- **C-35.** El repositorio no menciona `graphical-session.target` ni `graphical-session-pre.target` [R].
- **C-36.** El repositorio no define cómo arranca greetd `parallax-session.target`: qué comando de sesión se usa ni qué espera [R].

### 5.3 Grafo propuesto

Los nombres `parallax-compositor.service` y `parallax-settingsd.service` ya existen en `arquitectura.md`. Los scopes de rol no tienen nombre definido (pendiente).

| Origen | Destino | Directiva | Dirección efectiva | Parada normal | Fallo inesperado |
|---|---|---|---|---|---|
| `parallax-session.target` | `graphical-session.target` | `BindsTo=` + `After=graphical-session-pre.target` | session → graphical | Ver §5.1, matiz [VM] | Si `graphical-session.target` cae, cae `parallax-session.target` [V Y4] |
| `parallax-session.target` | `parallax-compositor.service` | `Wants=` (Y3: arranca lo que quiere el target) | target → compositor (solo arranque) | — | — |
| `parallax-compositor.service` | `graphical-session.target` | `PartOf=` | graphical → compositor (parada) | Parar `graphical-session.target` para el compositor [V Y4] | — |
| `parallax-compositor.service` | `parallax-settingsd.service` | `After=` (+ `Wants=`) | Solo orden | — | — |
| `parallax-settingsd.service` | `graphical-session.target` | `PartOf=` | graphical → settingsd | Se detiene | — |
| `parallax-session.target` | `parallax-compositor.service` | **`BindsTo=` (opción X)** o nada (opción Y) | session → compositor | — | X: si el compositor falla, cae la sesión gráfica. Y: el compositor se reinicia con `Restart=on-failure` y su límite. Decisión D-24 |
| Scopes de rol (Shell, lock, auth…) | `parallax-compositor.service` | Ninguna: los crea el compositor, y no hay garantía documentada de dependencias en scopes transitorios | — | Los limpia el compositor al detenerse (`RequestStop` / `Controller`, §6) [V Y1] | **Siguen vivos** si el compositor muere [H]; la limpieza se hace en la siguiente generación (§6) |
| Apps `app-parallax-…@….service` | `graphical-session.target` | `PartOf=` (propiedad transitoria; que se acepte es `[VM]`) | graphical → app | Se detienen | — |

**Orden frente a propagación.** `After=` nunca detiene nada. `PartOf=` detiene en una sola dirección. `BindsTo=` detiene al dependiente si cae la dependencia.

### 5.4 Ciclo de cierre

| Caso | Secuencia | Garantías |
|---|---|---|
| **Normal** | 1. Solicitud (Shell, overlay o atajo) → `Session1.Logout` (coordinado por el compositor). 2. Confirmación. 3. El compositor envía `close` a cada toplevel. 4. Tiempo de gracia (D-19). 5. Lista de apps que no responden: «forzar» o «cancelar». 6. `systemctl --user stop parallax-session.target` (la biblioteca o el compositor lo piden al user manager). 7. `graphical-session.target` y las unidades `PartOf` se detienen [V Y4, con el matiz de §5.1]. 8. Termina el comando de sesión de greetd y greetd cierra la sesión PAM; logind la cierra [H; depende de C-36]. 9. `KillUserProcesses` limpia lo que quede [V L2, valor `VM`] | **Ninguna de guardado**: `close` es una petición |
| **Forzado** | Pasos 1, 6–9 sin `close`; como último recurso, `TerminateSession` [V L1] | Pérdida de datos posible y advertida |
| **Caída del compositor** | Las apps pierden Wayland. Con X: cae `parallax-session.target` → paso 7 y siguientes. Con Y: el compositor se reinicia, arranca **Locked** (§6.5) y las apps ya cerradas no vuelven | Sin guardado |
| **Desde la pantalla bloqueada** | No se ofrece cierre normal sin autenticar (D-23). Si la política lo permite, solo cierre forzado tras confirmar | La sesión nunca se desbloquea |
| **Desde el greeter** | Fuera del ciclo: no hay sesión de usuario [R] | — |

---

## 6. C-23 — Fork, scopes y recuperación

### 6.1 Hechos de systemd

| Hecho | Fuente |
|---|---|
| Firma de `StartTransientUnit(name, mode, properties, aux)`; la unidad transitoria se libera cuando ya no está en ejecución ni referenciada | [V Y1] |
| **Propiedad `PIDs` para crear un scope: no encontrada** en Y1. Existe `Scope.AttachProcesses(subcgroup, pids)` | [V Y1] |
| Los scopes no tienen proceso principal; viven mientras queda algún proceso | [V Y2] |
| La propiedad `Controller` del scope y la señal `RequestStop()`: systemd avisa al controlador antes de terminar el scope; sin controlador, envía `SIGTERM` directamente | [V Y1] |
| `Scope.Abandon()` existe | [V Y1] |
| `KillUnit(name, who, signal)` y `StopUnit` existen | [V Y1] |
| `ListUnitsByPatterns(states, patterns)` existe (solo firma) | [V Y1] |
| `GetUnitByPIDFD` (desde v253) devuelve la unidad sin carreras de PID | [V Y1] |
| `JobRemoved` informa de `done`/`failed`/… | [V Y1] |
| `KillMode=` por defecto `control-group` | [V Y5] |

### 6.2 Registro sin carrera

**Procedimiento propuesto [H; los nombres de API, de Y1]:**

1. El compositor crea el `socketpair` del rol y una tubería de sincronización.
2. `fork()`. El hijo bloquea leyendo la tubería **antes de `exec`**. No ejecuta código del rol.
3. El padre registra el PID del hijo en un scope nuevo. **El mecanismo exacto es `[VM]`:** la propiedad de creación de scopes con PIDs no aparece en Y1, y `AttachProcesses` requiere un scope ya existente.
4. El padre espera el resultado del job (`JobRemoved=done`) [V Y1].
5. El padre comprueba la pertenencia con `GetUnitByPIDFD(pidfd del hijo)` == scope esperado [V Y1].
6. El padre escribe «continúa» en la tubería. El hijo aplica `PR_SET_DUMPABLE=0`, limpia el entorno y hace `exec` del binario del rol, que hereda solo el fd del socket.
7. El hijo conecta, el fd queda con `CLOEXEC` y se borra `WAYLAND_SOCKET`.

| Fallo | Comportamiento |
|---|---|
| `StartTransientUnit` falla | Matar al hijo (sigue bloqueado), estado `FAILED` y aplicar backoff |
| El hijo muere antes del registro | Se detecta con `pidfd` o `waitpid`; `FAILED` |
| El compositor muere durante el registro | El hijo bloqueado pierde la tubería (EOF) → **sale sin hacer `exec`** [H]; queda un scope vacío o huérfano, que systemd libera al no tener procesos [V Y2] |
| La pertenencia no coincide | Matar al hijo; `FAILED` |

**Ventana `fork → prctl`.** Hasta el `exec` el hijo es una copia del compositor. Hay que poner `PR_SET_DUMPABLE=0` justo después del fork, en el hijo. Ojo: el valor se restablece en ciertos `exec` [V K5]. Hay que comprobar si sobrevive al `exec` de un binario normal o si hay que repetirlo en el rol [VM].

### 6.3 Generación del compositor

**Sí es necesaria.**

| Aspecto | Propuesta |
|---|---|
| Creación | En cada arranque del compositor: valor aleatorio de 128 bits [H; D-20 amplía su alcance] |
| Pertenencia | Cada scope de rol registra la generación (en el nombre del scope o en una propiedad; **el mecanismo concreto no está verificado**: `Description` es una opción posible [H]) |
| Identificación del rol | Rol y generación en el nombre del scope (formato pendiente, D-25) |
| Invalidación | Al arrancar una generación nueva, todas las anteriores dejan de ser válidas |
| Limpieza | Al arrancar y **antes de lanzar roles**: `ListUnitsByPatterns` sobre el patrón de los roles → para cada scope de otra generación, `StopUnit`; si no termina, `KillUnit(…, "all", SIGKILL)` [V Y1] |
| No interferencia | Wayland: los procesos antiguos no tienen fd con el compositor nuevo (T10). D-Bus/polkit: si sigue vivo, el agente antiguo puede estar registrado (T11), de ahí que la limpieza vaya primero |
| Si la limpieza falla | No lanzar ese rol; estado `DEGRADED`; overlay con aviso; para `auth`, sin agente → polkitd no puede autenticar [H] y las peticiones fallan, no se conceden |

### 6.4 Máquina de estados por rol

```text
STOPPED ──start──► STARTING ──registrado+verificado──► REGISTERED ──exec+conexión Wayland──► RUNNING
   ▲                  │ fallo                            │ fallo / timeout                 │ salida / crash
   │                  ▼                                  ▼                                  ▼
   │               FAILED ◄─────────────────────────────┴──────────────────────────────── EXITED
   │                  │ backoff (dentro del límite)                                         │
   │                  └────────────────────────► STARTING                                   │
   │                  │ límite superado                                                     │
   │                  ▼                                                                     │
   │               DEGRADED (overlay; sin reintentos hasta acción del usuario)             │
   │                                                                                        │
   └──────── STOPPING ◄── stop / RequestStop / cierre de sesión ─── RUNNING                 │
                 │ (StopUnit; KillUnit tras timeout)                                        │
                 └──► STOPPED ◄─────────────────────────────────────────────────────────────┘ (si no se reinicia)
```

| Transición | Autoridad |
|---|---|
| `start`, backoff, límite, `DEGRADED` | Compositor (catálogo cerrado; 5/min y 200 ms como en `arquitectura.md` §5 [R]) |
| Registro en cgroup, kill, stop | systemd (lo pide el compositor) |
| Detectar la salida | Compositor (`pidfd`/`waitpid`); systemd, en paralelo, libera el scope [V Y2] |
| `RequestStop` | systemd → compositor como `Controller` [V Y1] |

**Unicidad:** como máximo un proceso por rol y generación. Antes de `STARTING`, el scope anterior del mismo rol debe estar en `STOPPED` (se comprueba con `ListUnitsByPatterns`).

### 6.5 Bloqueo y autenticación

| Caso | Regla |
|---|---|
| Cae `parallax-lock` | La sesión sigue Locked; el compositor lo relanza [R `compositor.md` §4.6] |
| Cae el compositor con la sesión Locked | El compositor nuevo **arranca siempre en Locked tras una caída** [H, recomendación conservadora]. Sin esto se incumple `compositor.md` §11.5 [R]. Alternativa: consultar el estado de bloqueo en logind (no verificado en L1: `VM`) |
| Cae el agente `auth` con una autenticación en curso | `BeginAuthentication` queda sin respuesta; polkitd puede cancelarla con `CancelAuthentication` [V P1]; qué hace al desconectarse el agente no está documentado [VM]. La petición **no se concede** sin autenticación [H; polkitd es la autoridad] |
| Petición de `auth` durante Locked | No se presenta por encima del bloqueo [R `shell.md` §9]; se pospone hasta desbloquear o se cancela (D-22) |

### 6.6 ¿Es viable R2?

**Viable con condiciones:**

1. Que el hijo pueda entrar en un scope antes del `exec` con API documentada `[VM]`.
2. Que la limpieza de generaciones funcione con `ListUnitsByPatterns` y `StopUnit`/`KillUnit` en el user manager `[VM]`.
3. Que `PR_SET_DUMPABLE=0` se mantenga en el rol `[VM]`.
4. Que el compositor arranque Locked tras una caída (decisión D-26).

Si la condición 1 falla, R2 no es viable tal como está descrita y C-16/C-29 se reabren.

---

## 7. Revisión cruzada

| Pregunta | Respuesta |
|---|---|
| A1. ¿La biblioteca crea unidades sin que el compositor gestione procesos? | Sí: la biblioteca llama al user manager; el compositor no interviene (§3) |
| A2. ¿`StartTransientUnit` es coherente con `DBusActivatable`? | Sí, porque no se usa: con `DBusActivatable=true` la biblioteca llama a `Activate` (E1). La unidad la define el paquete (`SystemdService=`) [V D1] |
| A3. ¿Identifica resourced la unidad? | Con `GetUnitByPIDFD` (Y1) obtiene la unidad real sin carreras. **No debe deducir la app del nombre de la unidad**. Si no puede correlacionarla con un Desktop Entry, la trata como no identificada y solo observa (ADR-0014 [R]) |
| B1. ¿Es compatible la entrega por fork con el ciclo de vida? | Sí, con la máquina de §6.4 |
| B2. ¿Qué pasa con los fd si cae el compositor? | El kernel cierra el lado del compositor; las conexiones de rol mueren [H]; el compositor nuevo crea fd nuevos |
| B3. ¿Pueden confundirse los roles antiguos con los nuevos? | En Wayland, no (T10). En D-Bus y polkit, sí mientras sigan vivos → limpieza antes de lanzar (§6.3) |
| C1. ¿Puede un proceso antiguo presentar una superficie de autenticación? | No como rol `auth`. Sí como ventana normal que imite el diálogo (T9, mitigada parcialmente) |
| C2. ¿Puede quedar la sesión desbloqueada por una caída? | No, si se aprueba D-26 (arrancar Locked tras una caída). Sin esa decisión, **no está garantizado** |
| C3. ¿Una caída del cliente de bloqueo nunca desbloquea? | Se mantiene [R `compositor.md` §4.6] |
| D. Autoridades | Compositor: superficies, foco, tokens, roles. settingsd: configuración. systemd: unidades. logind: sesiones. polkitd: autorización. Las apps no dependen del Shell. resourced no mata procesos (ADR-0014: modo observación y `FreezeUnit` mediante systemd) [R] |
| D. Dependencias ascendentes | Solo la excepción de lectura C3→C4 (D-17). La dependencia C6→C5 desaparece con §3 |

---

## 8. Contradicciones documentales

| ID | Contradicción | Fuente |
|---|---|---|
| C-20b | Biblioteca en `shell/shell/` consumida por C6 (sexta revisión, §7) | Informe de la sexta revisión |
| C-29 | `arquitectura.md` §5 («systemd reinicia `parallax-shell`») frente a R2 | [R] |
| C-34 | `arquitectura.md` §2: unidad del compositor = target | [R] |
| C-35 | No se mencionan `graphical-session*.target` | [R] |
| C-36 | No se define el arranque de la sesión desde greetd | [R] |
| C-37 | `compositor.md` §11.5 exige no desbloquear tras reiniciar, pero no define cómo sabe el compositor nuevo que la sesión estaba bloqueada | [R] |
| C-38 | `arquitectura.md` §2: `parallax-greeter.service` «vía greetd»; `lockscreen-greeter.md` dice que se ejecuta en el contexto que configura greetd. No queda claro si el greeter es un servicio propio o el comando de greetd | [R] |

---

## 9. Propuestas de corrección por archivo (no aplicadas)

### P1. `architecture/arquitectura.md` §2, fila `parallax-compositor`

- **Texto actual:** «`parallax-session.target` (usuario)»
- **Texto propuesto:** «`parallax-compositor.service` (usuario), deseado por `parallax-session.target`; `PartOf=graphical-session.target`»
- **Justificación:** un target no contiene procesos (C-34); patrón de Y3.
- **ADR relacionado:** —
- **Requiere aprobación:** Arquitecto.

### P2. `architecture/arquitectura.md` §2, fila `parallax-shell`, y §5 «Caída del shell»

- **Texto actual:** «`parallax-shell.service` (usuario)… systemd reinicia `parallax-shell` (`Restart=always`, `RestartSec=200ms`, límite 5/min)»
- **Texto propuesto:** «Proceso de rol lanzado por el compositor en un scope transitorio (ADR-0015). El compositor lo reinicia con un límite de 5 por minuto y backoff a partir de 200 ms. Al superar el límite, overlay de emergencia. systemd agrupa y limita recursos, pero no reinicia el proceso (un scope no tiene `Restart=`).»
- **Justificación:** C-29; Y2; Y6.
- **ADR relacionado:** 0015, 0006.
- **Requiere aprobación:** D-18 [P].

### P3. `architecture/arquitectura.md` §5 «Arranque»

- **Texto propuesto (añadir):** «`parallax-session.target` declara `BindsTo=graphical-session.target` y `After=graphical-session-pre.target`. Los componentes gráficos declaran `PartOf=graphical-session.target`. El comando de sesión de greetd y su forma de esperar al target están pendientes (C-36).»
- **Justificación:** Y3, Y4.
- **ADR relacionado:** —
- **Requiere aprobación:** Arquitecto. La propagación de la parada es `[VM]`.

### P4. `architecture/arquitectura.md` §5 «Apertura de app desde el dock»

- **Texto actual:** «Shell → `org.parallax.Session1.Launch(app_id)` (en compositor) → … lanza vía `systemd-run --user --scope`»
- **Texto propuesto:** «Shell → pide un token xdg-activation con el serial del clic → biblioteca común de lanzamiento: si `DBusActivatable=true`, `org.freedesktop.Application.Activate` con `activation-token`; si no, unidad transitoria `app-parallax-<ApplicationID>@<RANDOM>.service` en `app.slice` con `XDG_ACTIVATION_TOKEN` → la app activa su toplevel con el token → el compositor valida o ignora el token.»
- **Justificación:** §3; E1; W1; S1.
- **ADR relacionado:** 0007, 0006.
- **Requiere aprobación:** D-03, D-21.

### P5. `architecture/ipc.md` §6.1 `Session1`

- **Texto propuesto:** quitar `Launch(app_id)`; `Logout()` pasa a ser «inicia el cierre coordinado (§ciclo de cierre); no equivale a `TerminateSession`»; añadir `SubscribeFocus` (lectura) y `LaunchPrivileged` (catálogo cerrado, polkit).
- **Justificación:** §3, §5.4; C-17.
- **ADR relacionado:** 0007, 0015.
- **Requiere aprobación:** D-12.

### P6. `specifications/compositor.md` §11.5

- **Texto propuesto (añadir):** «Tras una terminación inesperada, el compositor arranca en estado Locked. Antes de lanzar roles, detiene los roles de generaciones anteriores. Si no puede detenerlos, no lanza ese rol y muestra el overlay.»
- **Justificación:** C-37; §6.3; §6.5.
- **ADR relacionado:** 0015, 0010.
- **Requiere aprobación:** D-26 [P] (afecta a la experiencia: tras cada caída hay que desbloquear).

### P7. `specifications/compositor.md` §10.2

- **Texto propuesto:** sin `Launch`. Añadir: «El compositor no resuelve Desktop Entries ni crea unidades de aplicaciones.»
- **Justificación:** §3.
- **ADR relacionado:** 0007.
- **Requiere aprobación:** D-03.

### P8. `specifications/dock.md` (líneas 328 y 735–756)

- **Texto actual:** «Solicitar lanzamiento mediante `Session1.Launch`» / «Servicio de lanzamiento»
- **Texto propuesto:** «Solicitar el lanzamiento mediante la biblioteca común de lanzamiento, con el token de la interacción.»
- **Requiere aprobación:** D-03, D-21.

### P9. `security/modelo-seguridad.md`, filas Shell y D-Bus

- **Texto propuesto (Shell):** «… el fd de rol es una capacidad Wayland, no un aislamiento: un proceso del mismo UID puede matar el rol o alterar su configuración. Los roles aplican `PR_SET_DUMPABLE=0`, `CLOEXEC` y no lanzan hijos.»
- **Texto propuesto (D-Bus):** «En el bus de sesión, las credenciales del peer solo prueban el UID; cualquier conexión puede poseer un nombre libre.»
- **Justificación:** T3–T8; D1.
- **ADR relacionado:** 0015.
- **Requiere aprobación:** Seguridad y propietario (D-22).

### P10. `.claude/agents/ingeniero-compositor.md`, matriz de protocolos

- **Texto propuesto:** nueva fila «Privilegiado: autenticación | globals mínimos de superficie (lista pendiente) | solo `parallax-polkit-agent` lanzado por el compositor; presentación exclusiva; nunca sobre Locked».
- **ADR relacionado:** 0015, 0010.
- **Requiere aprobación:** D-01, D-22.

### P11. `specifications/lockscreen-greeter.md` §2

- **Texto propuesto (añadir):** «Desde el estado Locked no se ofrece cierre de sesión cooperativo sin autenticación. Si la política lo habilita, solo cierre forzado con confirmación.»
- **Requiere aprobación:** D-23 [P].

---

## 10. Matriz actualizada (solo hallazgos afectados)

| ID | Documento | Evidencia | Problema | Corrección | Dependencias | Responsable | Decisión pendiente | Prueba | Estado |
|---|---|---|---|---|---|---|---|---|---|
| C-20 | `arquitectura.md` §5, `ipc.md` §6.1, `dock.md`, `compositor.md` §10.2 | [V E1, W1], [R] | Autoridad de lanzamiento; dependencia C6→C5 | Crate común (P4, P5, P7, P8) | 0006, 0007 | Arquitectura, Rust | Sí: D-03, D-21 (arquitecto) | Ninguna | Corregido en propuesta, pendiente de aprobación |
| C-21 | ADR-0015; `modelo-seguridad.md`; `ingeniero-compositor.md` | [V K1–K5], [R] | Límites del socket de rol | Modelo de amenazas §4.3 (P9, P10) | 0010, 0015 | Seguridad | Sí: D-22 (seguridad; aprueba el propietario) | VM: T3–T5, `dumpable` | Pendiente |
| C-22 | `arquitectura.md` §5; `ipc.md` §6.1; `lockscreen-greeter.md` | [V Y3, Y4, L1, L2] | Grafo y ciclo de cierre | §5.3, §5.4 (P3, P5, P11) | 0007 | Linux, arquitectura | Sí: D-19, D-23 [P], D-24 | VM: propagación de parada, `KillUserProcesses` | Pendiente |
| C-23 | `arquitectura.md` §2/§5; `compositor.md` §11.5 | [V Y1, Y2, Y5], [H] | Registro, generación y limpieza | §6 (P2, P6) | 0015, 0010 | Compositor, Linux, seguridad | Sí: D-18 [P], D-25, D-26 [P] | VM: registro del scope, limpieza | Pendiente |
| C-28 | `dock.md`; ADR-0014 | [V E1, D1, Y1] | Identificación de apps `DBusActivatable` | `GetUnitByPIDFD`; no deducir del nombre | 0014 | Recursos | No | VM | Corregido en propuesta, pendiente de aprobación |
| C-29 | `arquitectura.md` §5 | [V Y2, Y6] | systemd frente a R2 | P2 | 0015 | Arquitectura | Sí: D-18 [P] | VM | Pendiente |
| C-31 | `modelo-seguridad.md` | [V K1–K5] | Límites frente al mismo UID | P9 | 0015 | Seguridad | Sí: D-22 | VM | Pendiente |
| C-34 | `arquitectura.md` §2 | [R] | Unidad del compositor = target | P1 | — | Arquitectura | No | — | Corregido en propuesta, pendiente de aprobación |
| C-35 | Varios | [R], [V Y3] | Falta `graphical-session` | P3 | — | Linux | No | VM | Corregido en propuesta, pendiente de aprobación |
| C-36 | `arquitectura.md` §5; `lockscreen-greeter.md` | [R] | Arranque de la sesión desde greetd | — | — | Linux | Sí: arquitecto | VM | Pendiente |
| C-37 | `compositor.md` §11.5 | [R] | Bloqueo tras caída | P6 | 0015 | Compositor, seguridad | Sí: D-26 [P] | VM | Pendiente |
| C-38 | `arquitectura.md` §2; `lockscreen-greeter.md` | [R] | Unidad del greeter | — | — | Linux | Sí: arquitecto | — | Pendiente |

Ningún hallazgo está «resuelto en las fuentes originales».

---

## 11. Pruebas pendientes y autorizaciones necesarias

Ninguna está autorizada. Su registro en `spikes.md` lo decide el propietario.

| Prueba | Tipo | Decisión o condición |
|---|---|---|
| Crear un scope con un PID externo antes del `exec` (propiedad de creación frente a `AttachProcesses`) | VM | Condición 1 de R2 |
| `GetUnitByPIDFD` en el user manager de Arch (versión de systemd ≥ 253) | VM | §6.2 |
| `ListUnitsByPatterns` + `StopUnit`/`KillUnit` para limpiar generaciones | VM | §6.3 |
| `Controller` / `RequestStop` en scopes transitorios | VM | §6.4 |
| `PR_SET_DUMPABLE=0` después del `exec`; robo del fd con ptrace/`pidfd_getfd`; reabrir el socket desde `/proc` | VM | T3–T5 |
| Propagación de la parada de `graphical-session.target` y `PartOf` en unidades transitorias | VM | §5.1, §5.3 |
| `KillUserProcesses` efectivo en Arch; cierre de la sesión logind al terminar el comando de greetd | VM | §5.4 |
| Agente polkit: desconexión, nuevo registro tras limpieza, peticiones durante `DEGRADED` | VM | T11 |
| Ciclo completo de caída del compositor con la sesión Locked | VM + HW | D-26 |

---

## 12. Decisiones pendientes del propietario

| ID | Decisión | Por qué la decide el propietario |
|---|---|---|
| D-01 | Modelo del agente polkit | Revisión de ADR-0010 |
| D-18 | Sustituir `Restart=always` del Shell por supervisión del compositor | Cambia la política documentada de recuperación |
| D-22 | Aprobar la especificación del rol `auth` y los límites de seguridad | ADR-0015 |
| D-23 | Cierre de sesión desde la pantalla bloqueada | Política de producto y seguridad |
| **D-26 (nueva)** | Arrancar siempre en Locked tras una caída del compositor | Seguridad frente a experiencia de uso |
| — | Autorizar las pruebas en VM de §11 | Ningún agente puede autorizarlas |

**Técnicas, para los responsables (no requieren al propietario):**

| ID | Decisión | Responsable |
|---|---|---|
| D-21 | Nombre y fase del crate de lanzamiento | Arquitecto, Rust |
| **D-24 (nueva)** | Opción X o Y ante el fallo del compositor | Arquitecto, Linux |
| **D-25 (nueva)** | Formato del nombre de los scopes de rol con generación | Arquitecto, Linux; no inventar sin VM |
| D-19 | Tiempo de gracia del cierre | Arquitecto |

---

## 13. Riesgos residuales

| ID | Riesgo | Estado |
|---|---|---|
| R-20 | Sin API documentada para meter un PID externo en un scope nuevo, R2 no es viable | Abierto; `VM` |
| R-21 | La limpieza de la generación anterior falla y el agente polkit antiguo bloquea el registro del nuevo | Mitigado con `DEGRADED`; `VM` |
| R-22 | Ventana entre fork y `prctl(PR_SET_DUMPABLE)` | Mitigación: aplicarlo en el hijo justo después del fork; `VM` |
| R-23 | Suplantación visual del diálogo `auth` por una ventana normal | Parcial; pendiente de diseño (T9) |
| R-24 | Arrancar Locked tras cada caída empeora la experiencia | Decisión D-26 |
| R-25 | La parada de `graphical-session.target` no se propaga como se espera | `VM` |
| R-26 | Roles huérfanos tras la caída del compositor consumen recursos hasta la limpieza | Mitigado por la limpieza al arrancar |

---

## 14. Veredicto: `LISTA CON CONDICIONES EXPLÍCITAS`

### Criterios de aceptación

| Criterio | Resultado |
|---|---|
| La biblioteca de lanzamiento respeta las capas | **Sí**, con el crate común (§3) |
| Ninguna app depende del Shell para lanzar | **Sí** |
| El compositor mantiene la autoridad Wayland sin gestionar procesos | **Sí**: solo un catálogo cerrado de roles y la terminal de recuperación |
| Las capacidades de rol tienen límites documentados | **Sí**, en este informe (§4.3). En las fuentes, pendiente (P9, P10) |
| No se atribuye a Yama más de lo que garantiza | **Sí** (§4.2) |
| El grafo de cierre es coherente | **Sí** en su estructura (§5.3). La propagación hacia `graphical-session.target` es `[VM]` |
| No se confunde `BindsTo=` con una dependencia bidireccional | **Sí** (§5.1) |
| Los roles antiguos no interfieren | **Sí, con la limpieza por generación**, sujeta a `[VM]` |
| El bloqueo sigue siendo seguro ante fallos | **Solo si se aprueba D-26** |
| Las autoridades están separadas | **Sí** (§7 D) |
| Las pruebas pendientes están identificadas | **Sí** (§11) |

### Condiciones explícitas para presentarlo al propietario como cerrado

1. Aceptar que R2 depende de la prueba en VM del registro del PID en un scope (R-20). Si la prueba falla, C-16, C-23 y C-29 vuelven a abrirse.
2. Que el propietario decida D-26. Sin esa decisión, la garantía «una caída nunca desbloquea» no está cubierta tras reiniciar el compositor.
3. Que el arquitecto resuelva C-36 (arranque desde greetd) y D-24 antes de redactar el grafo definitivo.
4. Que ingeniero-seguridad incorpore §4.3 y §4.4 a la propuesta de revisión de ADR-0015, sin cambiar su estado.

No es `REQUIERE OTRA REVISIÓN TÉCNICA` porque ninguna contradicción queda sin una alternativa documentada. Lo que falta son pruebas identificadas y decisiones con autoridad asignada.

No es `LISTA PARA REVISIÓN DEL PROPIETARIO` sin más, porque la viabilidad de R2 depende de una API que no encontré documentada.

### Lo que este veredicto no hace

- No aprueba nada.
- No acepta ningún ADR.
- No autoriza VM, hardware ni spikes.
- La Fase 0 sigue en `IN_PROGRESS`.
