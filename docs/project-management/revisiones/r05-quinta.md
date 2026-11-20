# PARALLAX OS — Quinta revisión técnica complementaria (C-20 a C-28)

- **Estado:** propuesta no normativa, pendiente de aprobación del arquitecto y del propietario.
- **Fase 0:** `IN_PROGRESS`.
- **ADR:** todos conservan su estado (`Propuesto`).
- **Fecha:** 2026-11-19.
- **Modo:** solo lectura sobre el repositorio. Este archivo está en el scratchpad de la sesión y no forma parte del repositorio.
- **Base:** la cuarta revisión (entregada en la conversación) y los documentos originales. No se repiten las revisiones anteriores.

Convención de evidencia:
- **[V]** hecho verificado en una fuente identificada.
- **[R]** documentado en el repositorio.
- **[H]** hipótesis o inferencia.
- **[VM]** requiere una prueba en VM, que no está autorizada (ADR-0013).

## Fuentes externas consultadas (lectura, 2026-11-19)

| Ref | Fuente |
|---|---|
| S1 | systemd.io/DESKTOP_ENVIRONMENTS |
| S2 | systemd.scope(5), systemd.service(5), systemd.unit(5), systemd.special(7) — freedesktop.org/software/systemd/man/latest |
| S3 | org.freedesktop.login1(5) |
| S4 | D-Bus Specification — dbus.freedesktop.org/doc/dbus-specification.html (lectura literal parcial; parte del documento solo se leyó resumida) |
| S5 | polkit: `org.freedesktop.PolicyKit1.Authority`, `PolkitAgentListener`, `PolkitAgentSession`, polkit(8) — polkit.pages.freedesktop.org |
| S6 | Desktop Entry Specification: file naming y D-Bus Activation — specifications.freedesktop.org/desktop-entry |
| S7 | xdg-activation-v1 y xdg-shell — wayland.app (espejo del XML) |

---

## 1. Evaluación de C-20 a C-28

| ID | Problema | Error de la cuarta revisión | Estado tras esta revisión |
|---|---|---|---|
| C-20 | Propietario del lanzamiento | Asignaba al compositor el lanzamiento genérico (resolver Desktop Entries, interpretar `Exec`, crear unidades). Eso lo convierte en lanzador de procesos sin necesidad | Pendiente (nueva recomendación, §2.1) |
| C-21 | Frontera de confianza del agente polkit | Trataba el socket de rol como si diera «contexto privilegiado». Un socket no eleva al proceso: solo da acceso a ciertos globals Wayland | Pendiente (el modelo de ADR-0015 está incompleto, §2.2) |
| C-22 | Cierre de sesión | Sustituía `Logout` por logind, que según S3 solo termina «forcibly». Faltaba el ciclo completo | Pendiente (§2.3) |
| C-23 | Ciclo de vida systemd | Proponía el Shell como `.scope` y eliminaba `Restart=always`. Un scope no tiene `Restart=` [V S2], así que la recuperación pasaba al compositor sin justificarlo | Pendiente (§2.4) |
| C-24 | `cargo-deny` | La cuarta revisión afirmaba que «`cargo deny` verifica la compatibilidad». `dependencias.md` regla 3 dice lo mismo («Verificado por `cargo deny`»): también sobreafirma | Parcial (§2.5) |
| C-25 | Snapshot y deltas | Atribuía la ausencia de pérdidas al event loop del compositor y suponía que la respuesta llegaba en orden respecto de las señales. S4 no documenta garantías de orden en lo leído | Parcial (§2.6) |
| C-26 | Identidad del productor D-Bus | Afirmaba que `busconfig` restringe quién posee el nombre. En el bus de sesión, cualquier proceso del mismo UID puede reclamar un nombre libre [H] | Parcial (§2.6) |
| C-27 | Infraestructura oficial de polkit | No contemplaba `libpolkit-agent-1` / `PolkitAgentSession`. Descartaba A sin considerar el modo `fallback` documentado [V S5] | Pendiente (§2.7) |
| C-28 | xdg-activation y `DBusActivatable` | Asumía que todo lanzamiento ejecuta `Exec` | Parcial (§2.8) |

---

## 2. Alternativas comparadas y recomendaciones

### 2.1 C-20 — Lanzamiento de aplicaciones

#### Responsabilidades que hay que separar

| Responsabilidad | Naturaleza | ¿Necesita privilegio o autoridad especial? |
|---|---|---|
| Autoridad sobre superficies y activación (validar el token en `xdg_activation_v1.activate`) | Wayland | **Sí, solo el compositor** [V S7: el compositor puede ignorar tokens] |
| Solicitar el token | Wayland | No: cualquier cliente puede pedirlo con `get_activation_token` y aportar serial y superficie [V S7] |
| Resolver el Desktop Entry y validar el Desktop File ID | Parsing de archivos | No |
| Interpretar `Exec` (códigos de campo) | Parsing | No; si se hace mal, es un riesgo de inyección |
| `DBusActivatable=true` | Llamada `org.freedesktop.Application.Activate` [V S6] | No |
| Iniciar el proceso y crear la unidad | systemd user manager | No: cualquier proceso del usuario puede hacerlo (mismo UID) [H] |
| Supervisión y recuperación | systemd | — |

**Conclusión clave [H, razonada]:** para aplicaciones normales no hay frontera de privilegio. Cualquier proceso del usuario puede crear unidades en su propio `systemd --user`. Centralizar la validación da coherencia, no seguridad. La única autoridad real está en el compositor (aceptar o ignorar el token) y en ADR-0015 (roles privilegiados), y ninguna de las dos exige que el compositor inicie procesos de aplicaciones.

#### Comparación

| Criterio | A. Compositor lanzador | B. Servicio independiente | C. Shell coordinador + mecanismo alternativo | D. Componente mínimo sobre el user manager |
|---|---|---|---|---|
| Propietario | `parallax-compositor` | Un binario nuevo (`parallax-launcherd`, hipotético) | `parallax-shell` | Código compartido usado por cada solicitante |
| Capa | C4 | C2 | C5 | La del solicitante |
| Interfaz | `Session1.Launch` | `Launcher1.Launch` (nuevo) | Interna del Shell; el overlay usa su propio camino | Llamadas a `org.freedesktop.systemd1` y `org.freedesktop.Application` |
| Dependencias | Parser de Desktop Entry dentro del compositor | Proceso, unidad y contrato nuevos | Ninguna nueva | Una biblioteca, condicional, no de Fase 1 |
| Token | El compositor lo genera directamente | Sin superficie, el compositor puede ignorarlo [V S7]; habría que pasárselo desde el solicitante | El Shell lo pide con el serial del clic: **es el caso típico del protocolo** | Lo pide el solicitante |
| Seguridad | Amplía la superficie de ataque del componente más crítico | Proceso extra sin frontera real | Sin cambio de privilegios | Sin cambio de privilegios |
| Ciclo de vida | Se lanza con el compositor y cae con él | Necesita supervisión | Con el Shell | Con el solicitante |
| Recuperación si cae el Shell | Disponible | Disponible | Hace falta un camino alternativo para el overlay | Depende del solicitante |
| ¿Gestor genérico de procesos? | **Sí, es el riesgo** | No | No | No |

#### Recomendación (pendiente de aprobación): C + D

1. **Lanzamiento iniciado por el usuario desde Shell** (dock, launcher, notificaciones):
   - El Shell pide el token con el serial y la superficie de la interacción.
   - Resuelve y valida el Desktop Entry.
   - Si `DBusActivatable=true`, llama a `Activate`.
   - Si no, crea `app-parallax-<ApplicationID>@<RANDOM>.service` en el user manager.
2. **Lanzamiento desde aplicaciones PARALLAX** (p. ej. Files, «abrir con»):
   - Usan el mismo procedimiento (código compartido) o los portales estándar. No pasan por el Shell.
3. **El compositor** solo:
   - Valida tokens.
   - Lanza **roles privilegiados fijos** (ADR-0015).
   - Para el overlay, inicia una **unidad estática preinstalada** (hipotética: `parallax-recovery-terminal.service`) con `StartUnit` sobre un nombre fijo. No interpreta Desktop Entries ni `Exec`.
4. **Supervisión:** systemd. Las apps no tienen `Restart=`.
5. **Código compartido:**
   - Empieza como un módulo del Shell.
   - Pasa a crate solo cuando haya un segundo consumidor real (criterios de crecimiento). **No es un crate de Fase 1.**

**Consecuencias:**
- `Session1.Launch` **se retira** del compositor.
- Se contradicen `arquitectura.md` §5 («en compositor»), `ipc.md` §6.1 y `dock.md`. Hay que corregirlos (§5).
- Si el Shell está caído, el usuario solo puede lanzar la terminal de recuperación desde el overlay. Es un comportamiento aceptable para el modo de emergencia y queda documentado.

### 2.2 C-21 — Frontera de confianza del agente polkit

#### Siete planos distintos

| # | Plano | En la alternativa B |
|---|---|---|
| 1 | Proceso | Corre con el UID del usuario, **sin privilegios adicionales**. No se eleva |
| 2 | Superficie de autenticación | La presenta el compositor según su política (prioridad, foco exclusivo). La seguridad visual está en el compositor (ADR-0010 §6) [R] |
| 3 | Identidad de la conexión Wayland | El compositor sabe por qué socket llegó la conexión. Esa es la única identidad fiable (ADR-0015) [R] |
| 4 | Autorización de protocolos | Los globals que el compositor anuncia en ese socket |
| 5 | Capacidad administrativa | **Ninguna.** El agente no ejecuta operaciones privilegiadas (ADR-0010 §2) [R] |
| 6 | Autorización de acciones | La decide polkitd (C1), no el agente [V S5, polkit(8)] |
| 7 | Credenciales y PAM | `PolkitAgentSession` las pasa a un «trusted suid helper» mediante una tubería [V S5]. El agente nunca valida contraseñas |

#### Qué garantiza un socket dedicado y qué no

| Garantiza | No garantiza |
|---|---|
| Que esa conexión concreta la creó el compositor y se entregó a un proceso que él lanzó [R `ingeniero-seguridad.md` regla 1] | Que el proceso sea el binario esperado después de lanzarlo; un proceso del mismo UID puede manipularlo con ptrace (límite reconocido en la regla 6) |
| Que los globals de rol no aparecen en el socket público [R] | Que el fd no se filtre: quien obtenga el fd (herencia, `SCM_RIGHTS`) comparte esa conexión [H, comportamiento de fd en Unix] |
| Que el compositor puede revocar el rol cerrando la conexión [H] | Que haya cualquier elevación de privilegios en el sistema |

#### Definición requerida (hoy no está en ADR-0015)

| Aspecto | Propuesta | Estado |
|---|---|---|
| Quién crea la conexión | El compositor, con `socketpair` antes del fork; el hijo recibe un extremo como `WAYLAND_SOCKET` | Coherente con `ingeniero-compositor.md` [R]; detalle [H] |
| Asociación al rol | Tabla interna del compositor: conexión → rol | Hay que especificarlo |
| Globals | Un **rol `auth` separado**: solo los necesarios para una superficie de autenticación con foco exclusivo. Sin `layer-shell` general, sin captura, sin `output-management`, sin `foreign-toplevel` | Hay que especificarlo; **no invento el protocolo** |
| Suplantación | Una superficie de autenticación solo puede venir de una conexión con rol `auth`; las demás no pueden ocupar ese nivel de presentación | Hay que especificarlo |
| Filtración del fd | `CLOEXEC` tras conectar, borrar `WAYLAND_SOCKET`, no crear hijos que hereden el fd | [H]; prueba [VM] |
| ¿Se puede heredar o transferir? | Técnicamente sí. **No hay mecanismo que lo impida**, solo higiene del proceso | Límite que hay que documentar |
| Revocación | El compositor cierra la conexión; el rol no se reasigna sin un relanzamiento | Hay que especificarlo |
| Caída del compositor | Se pierden todas las conexiones. El nuevo compositor relanza el rol. Una autenticación en curso se cancela; polkitd la gestiona por su lado | Comportamiento de polkitd ante la desconexión del agente: [VM] |
| Foco y presentación | Foco de teclado exclusivo mientras la petición está activa; nunca por encima de Locked (`shell.md` §9) [R] | Hay que especificarlo |

**¿Hace falta un rol `auth` distinto de los roles administrativos?** **Sí** [H razonada]:
- El agente necesita **menos** capacidades que el Shell, pero una **política de presentación más fuerte** (exclusividad).
- Juntarlo con el rol del Shell le daría capacidades que no necesita (`foreign-toplevel`, `workspace`).
- Juntarlo con Settings (`output-management`) le daría capacidades administrativas de pantalla.

**Lo que ADR-0015 debe especificar antes de aceptar B:**
1. El catálogo de roles, incluido `auth`.
2. Los globals por rol.
3. La política de presentación y foco del rol `auth`.
4. Su interacción con Locked.
5. La higiene del fd.
6. La revocación.
7. El relanzamiento.

### 2.3 C-22 — Cierre de sesión

**Hechos:**
- logind `TerminateSession` y `Session.Terminate` terminan la sesión «forcibly» [V S3]. No ofrecen semántica de guardado.
- Wayland no tiene un protocolo estándar de «guardar antes de salir» [H; no hay ninguno documentado en el repositorio].
- `xdg_toplevel` recibe el evento `close`, que la aplicación puede atender o ignorar [H; xdg-shell].
- `graphical-session.target` está activo mientras hay una sesión gráfica, y la sesión lo arrastra con `BindsTo=` [V S2, citado desde un resumen].

**Ciclo propuesto (coordinador: el compositor)**

| Fase | Ejecutor | Comportamiento |
|---|---|---|
| 1. Solicitud | Shell (UI), overlay o atajo | Petición al coordinador (operación de `Session1`; firma en P-06) |
| 2. Confirmación | Shell; overlay si el Shell no está | Se omite en el cierre forzado |
| 3. Aviso y oportunidad de guardar | Compositor → `close` a cada toplevel | **Sin garantía de guardado**: depende de cada aplicación. Las apps PARALLAX con restauración de sesión (`arquitectura.md` §5) guardan su estado |
| 4. Cierre de aplicaciones | Las propias aplicaciones | Plazo configurable (valor por definir) |
| 5. Aplicaciones que no responden | Shell muestra la lista («forzar» / «cancelar») | El usuario decide; sin decisión, se cancela el cierre |
| 6. Detención de componentes | systemd: `stop` de `parallax-session.target`; las unidades con `PartOf`/`BindsTo` caen en orden inverso | Orden de dependencias [VM] |
| 7. `graphical-session.target` | Se desactiva por `BindsTo` | [V S2] |
| 8. Terminar la sesión | logind `TerminateSession` (paso final o forzado) | Forzoso por definición [V S3] |
| 9. Limpieza | logind y systemd (cgroups); `KillUserProcesses` según `logind.conf` | Semántica de `KillUserProcesses`: [VM] / falta leer logind.conf(5) |

**Por qué coordina el compositor:**
- Es el único que conoce los toplevels y sobrevive a la caída del Shell.
- **No supervisa procesos:** envía `close`, espera y delega en systemd y logind.
- No hace falta un gestor de sesión independiente (alternativa C de la cuarta revisión), porque las fases 6–9 ya las hacen systemd y logind.

| Modo | Fases |
|---|---|
| Normal | 1–9 |
| Forzado | 1, 6–9 (sin `close`) |
| Recuperación (compositor caído) | Las apps ya perdieron Wayland; systemd y logind limpian. Si la sesión estaba Locked, se mantiene la regla de `compositor.md` §11.5 |

**Consecuencia:** respecto a la cuarta revisión, `Logout` **vuelve** a `Session1`, pero redefinido como «iniciar el ciclo coordinado», no como terminación forzosa.

### 2.4 C-23 — Ciclo de vida systemd

**Hechos:**
- Un scope gestiona procesos creados externamente, no hace fork y su vida dura mientras quede algún proceso [V S2].
- `Restart=` está documentado para servicios y sujeto a `StartLimitIntervalSec=` y `StartLimitBurst=` [V S2]. Los scopes no se configuran con archivos de unidad [V S2], así que **no tienen reinicio por systemd**.

**El dilema real (C-16):**
- **ADR-0015:** la capacidad es el fd que entrega el compositor.
- **`arquitectura.md` §2:** el Shell es `.service` con `Restart=always`, `RestartSec=200ms` y 5 reintentos por minuto.
- Las dos cosas son compatibles solo si un `.service` puede recibir ese fd del compositor. ADR-0015 cita `OpenFile=`, que abre una **ruta**, no recibe un fd existente [H; sin verificar].

| Opción | Arranque | Supervisión y reinicio | Ventaja | Inconveniente |
|---|---|---|---|---|
| R1: rol como `.service` + fd por un mecanismo de systemd | systemd | systemd (`Restart=always`, límite de arranques) | Conserva la política documentada; detección de bucles por systemd | El mecanismo de entrega **no está verificado** [VM] |
| R2: rol como `.scope` creado por el compositor | Compositor (fork + registro) | **Compositor** (sin `Restart=` en scopes) | El fd se entrega trivialmente | El compositor necesita su propia política de reintentos y detección de bucles: duplica lo que systemd ya hace |
| R3: el rol se conecta por un socket con ruta controlada | systemd | systemd | Sin entrega de fd | Una ruta es accesible a cualquier proceso del mismo UID [H]; debilita ADR-0015 |

**Recomendación:**
- **No eliminar `Restart=always`.**
- Mantener R1 como objetivo y comprobar en VM si existe una entrega de fd viable (p. ej. un fd store o un socket de activación controlado por el compositor) [VM].
- Si no existe, adoptar R2 solo para los roles privilegiados, con estas reglas explícitas en el compositor: los mismos límites que la documentación actual (5 por minuto), backoff, y al superar el límite, overlay de emergencia (`arquitectura.md` §5).
- La decisión queda pendiente (D-18).

**Modelo resultante (propuesta; `[VM]` donde se indica):**

| Componente | Inicia | Supervisa y reinicia | Unidad / slice | Orden y dependencias |
|---|---|---|---|---|
| greetd + `parallax-greeter` | systemd (sistema) | systemd | `greetd.service` (sistema) | Antes de la sesión [R] |
| `parallax-compositor` | `parallax-session.target` | systemd; política pendiente: una caída rompe las conexiones Wayland de todas las apps (`arquitectura.md` §5) | `.service`, `session.slice` | `BindsTo=graphical-session.target` [V S2 guía]; después de settingsd [R] |
| `parallax-shell` | R1: systemd / R2: compositor | R1: systemd `Restart=always`, 5/min / R2: compositor, el mismo límite | `session.slice` | Después del compositor |
| `parallax-polkit-agent` (si D-01 = B) | Como el Shell | Como el Shell | `session.slice` | Después del compositor; independiente del Shell |
| `parallax-lock` | Compositor (ya documentado [R]) | Compositor; **la sesión sigue Locked** si cae [R] | `.scope` [H] | Cuando se bloquea |
| Aplicaciones | Shell o solicitante (§2.1) | systemd; `Restart=no` | `app-parallax-…@<RANDOM>.service`, `app.slice` | — |
| Apps `DBusActivatable` | Activación del bus | Según su `.service` D-Bus (`SystemdService=`) [H] | La unidad de la propia app | §2.8 |
| Terminal de recuperación | Compositor, `StartUnit` de una unidad fija | systemd | `.service`, `app.slice` | Solo desde el overlay |
| Overlay | Dentro del compositor | — | — | — |

**Caídas:**
- **Compositor:** las apps pierden Wayland; las sesiones Locked siguen protegidas al reiniciar (`compositor.md` §11.5).
- **Shell:** las ventanas se conservan y el Shell se reinicia; al superar el límite, overlay [R].

Verificaciones [VM]:
- Orden real de arranque.
- `StartLimit` sobre `Restart=always`.
- Herencia de cgroup por los hijos.
- Entrega de fd (R1).

### 2.5 C-24 — Dependencias y licencias

| Herramienta / control | Qué hace | Qué **no** hace | ¿Automatizable? |
|---|---|---|---|
| Campo `license` en `Cargo.toml` | Declara la licencia **del crate propio** | No dice nada de las dependencias | Sí (declaración) |
| `cargo-deny` | Compara las licencias SPDX **declaradas** por las dependencias con una lista permitida; controla fuentes, duplicados y bans | No interpreta obligaciones, no valida que lo declarado sea correcto, no evalúa el tipo de integración (enlace estático, GPL frente a MIT/Apache) | Sí |
| `cargo-audit` | Busca avisos de RustSec en las versiones del lockfile | No evalúa licencias ni calidad | Sí, con la base actualizada (`workspace-rust.md` §6 flujo B) |
| `cargo-vet` | Registra auditorías humanas o importadas por versión | No es automático: necesita auditores. **Pendiente** (`dependencias.md` regla 11) | Parcial |
| Revisión de licencias | Analiza compatibilidad, obligaciones de redistribución y excepciones (`dependencias.md` reglas 1 y 3; ADR-0008) | — | **No:** es contextual y humana |
| SBOM (CycloneDX) | Inventario de lo distribuido (`dependencias.md` regla 9) | No decide compatibilidad | Sí (generación) |

**Corrección:** `cargo-deny` es una barrera automática necesaria, no suficiente. La compatibilidad jurídica requiere la revisión humana que ya exige `dependencias.md` regla 1 (campo «compatibilidad», revisor y fecha). Cambio documental propuesto en `dependencias.md` regla 3: «verificado por `cargo deny`» pasa a «filtrado automáticamente por `cargo deny`; la compatibilidad la decide la revisión registrada». ADR-0008 sin cambios.

### 2.6 C-25 y C-26 — Snapshot, deltas e identidad D-Bus

**Hechos [V S4]:**
- Los nombres únicos no se reutilizan durante la vida del bus.
- `NameOwnerChanged(name, old_owner, new_owner)` informa de los cambios de propietario.
- `GetConnectionUnixUser` y `GetConnectionCredentials` devuelven credenciales del proceso conectado.
- **No encontré una garantía de orden entre mensajes** en lo leído literalmente; parte de S4 solo se leyó resumida. Por tanto, **el protocolo no depende del orden**.

**Cuatro conceptos que no deben confundirse:**

| Concepto | Qué prueba |
|---|---|
| Nombre bien conocido (`org.parallax.Session1`) | Quién lo posee **ahora**. En el bus de sesión, cualquier proceso del mismo UID puede reclamarlo si está libre [H; la política del bus de sesión no está verificada] |
| Nombre único (`:1.42`) | Una conexión concreta; no se reutiliza [V S4] |
| Credenciales de conexión | UID verificado por el bus; el PID tiene carreras [R `ingeniero-seguridad.md`] |
| Autorización del bus | Políticas `busconfig` y polkit por método [R] |

El campo `owner_id` del payload **nunca** se usa como identidad.

**Protocolo propuesto** (versión cuarta + correcciones):

```text
1. Conexión:     resourced conecta al bus de sesión.
2. Resolución:   GetNameOwner("org.parallax.Session1") → U (nombre único). Si no hay dueño: estado UNKNOWN, esperar NameOwnerChanged.
3. Verificación: GetConnectionUnixUser(U) == UID propio (límite: solo prueba UID).
4. Match:        AddMatch(type=signal, sender=U, interface=org.parallax.Session1, member=FocusChanged)  ← ANTES de suscribir.
                 AddMatch(NameOwnerChanged para org.parallax.Session1).
5. Suscripción:  llamada SubscribeFocus() dirigida a U.
6. Registro + captura + punto de sincronización: el propietario, de forma atómica según su contrato,
                 registra al consumidor, captura el estado en revision=n y lo devuelve: {epoch, n, snapshot}.
7. Entrega:      respuesta del método. Señales que lleguen antes de la respuesta → búfer.
8. Aplicación:   al recibir la respuesta: descartar del búfer revision ≤ n; aplicar n+1, n+2… en orden de revisión.
9. Resincronización ante hueco, epoch distinto o cambio de nombre único (abajo).
```

| Caso | Regla |
|---|---|
| `epoch` | Lo genera el propietario en cada arranque (valor aleatorio o monotónico persistido; por decidir). Distinto `epoch` → nueva generación (`eventos.md` §3) |
| `revision` | Monotónica dentro de cada `epoch` (`eventos.md` §2.5) |
| Duplicado | `revision` ≤ la última aplicada → descartar |
| Hueco | `revision` > la última + 1 → pasar a UNKNOWN y resincronizar (`eventos.md` §4.2) |
| Fuera de orden | Como no hay garantía de orden verificada, una revisión anterior a la ya aplicada se descarta y un salto se trata como hueco. No se reordena [H; diseño conservador] |
| Reinicio del propietario | Nuevo nombre único y nuevo `epoch` → descartar todo, volver al paso 2 |
| Cambio de nombre único | Con `NameOwnerChanged`, se ignoran las señales con otro `sender` |
| Servicio desaparecido | `new_owner` vacío → UNKNOWN, sin actuar |
| Recuperación del consumidor | Al reiniciar resourced: pasos 1–8; nunca reutiliza estado persistido como actual (`eventos.md` §2.2) |
| Información obsoleta | resourced actúa solo en estado SYNCED y después de la histéresis (ADR-0014) |

**Garantía que asume el contrato (no D-Bus):** el propietario debe hacer atómicos el registro y la captura (`eventos.md` §4.1). Que el compositor tenga un único event loop (`compositor.md` §3.2) es **cómo podría cumplirlo**, no una garantía del protocolo.

**Límite de identidad [H]:** en el bus de sesión solo se prueba el UID. Un proceso malicioso del mismo usuario que ocupe el nombre podría falsear el foco. Es coherente con el límite de ADR-0015 regla 6 (sin aislamiento entre procesos no sandbox del mismo usuario). El daño queda acotado porque resourced está en modo observación y sus acciones son reversibles (ADR-0014).

**Compilación:** resourced y compositor solo comparten `px-dbus`.

**Ejecución:** la llamada `SubscribeFocus` (de C3 a C4) es de lectura y necesita la excepción D-17.

### 2.7 C-27 — Infraestructura oficial de polkit

**Lo que proporciona la infraestructura oficial [V S5]:**
- `RegisterAuthenticationAgent(subject, locale, object_path)` en la Authority.
- `RegisterAuthenticationAgentWithOptions`: con la opción `fallback` a TRUE, el agente se usa solo si no hay otro agente sin `fallback`.
- La documentación dice: «registration of an authentication agent can fail; for example another authentication agent may already be registered for subject». No se garantiza que se pueda registrar un segundo agente normal.
- `PolkitAgentListener` y `polkit_agent_listener_register(…, flags, …)`, con `POLKIT_AGENT_REGISTER_FLAGS_RUN_IN_THREAD`.
- `PolkitAgentSession` «uses a trusted suid helper» con una tubería, y emite las señales `request`, `show-error`, `show-info` y `completed`. `polkit_agent_session_cancel()` emite `completed`.
- polkit(8): «an Authentication Agent per user session».

**No documentado en lo leído (requiere [VM] o leer más fuentes):**
- El nombre `polkit-agent-helper-1`, que no aparece en esas páginas.
- La desregistración automática cuando el agente se desconecta del bus.
- El método `CancelAuthentication` de la interfaz del agente (no leída).
- La sustitución en caliente.

**Integración desde Rust (D-16):**

| Opción | Evaluación |
|---|---|
| FFI a `libpolkit-agent-1` (`PolkitAgentListener` + `PolkitAgentSession`) | **Recomendada.** Usa el helper suid oficial y la conversación PAM. Coste: GLib/GObject y su bucle (mitigado con `RUN_IN_THREAD`), `unsafe` con revisión, y entradas en `dependencias.md` y ADR-0008 |
| Implementar la interfaz `AuthenticationAgent` en Rust (zbus) y hablar con el helper directamente | **No recomendada.** El protocolo de la tubería es interno, no documentado como API |
| Helper PAM propio | **Rechazada.** Sería un binario setuid propio, contrario a ADR-0010 §7 («no implementará autenticación PAM directamente») y a `servicios.md` §17 |

**Comparación A/B revisada:**

| Criterio | A: Shell + agente de recuperación con `fallback=true` | B: proceso propio único |
|---|---|---|
| Mecanismo oficial | **Documentado:** el agente del Shell sin fallback tiene prioridad y el de recuperación está siempre registrado con `fallback` [V S5] | Un único registro normal [V S5] |
| Superficie que ve credenciales | Shell (grande) y recuperación | Solo el agente |
| Agentes múltiples | Coexisten por diseño de `fallback` (el comportamiento exacto requiere [VM]) | Ninguno |
| Sustitución y recuperación | Si cae el Shell, actúa el agente de recuperación | systemd o el compositor relanza (§2.4); hay un hueco sin agente durante el reinicio |
| Cancelación | `polkit_agent_session_cancel` en ambos | Igual |
| Dos implementaciones de UI | Sí | No |
| Rol Wayland | El del Shell, más un rol de recuperación | Rol `auth` nuevo (§2.2) |
| ADR-0010 | Se cumple (§9 ya prevé el agente de recuperación) | Requiere revisión |
| ADR-0015 | Sin cambios para el Shell; rol nuevo para el de recuperación | Rol nuevo `auth` |

**Recomendación (pendiente del propietario): B**, ahora con la justificación corregida:
- Minimiza dónde residen las credenciales.
- Usa la infraestructura oficial (`libpolkit-agent-1`) en un solo proceso.

**Alternativa válida y documentada:** A con `fallback`, que no requiere revisar ADR-0010.

**Variante de B a evaluar [H]:** registrar también B con `fallback=false` y mantener un agente de texto (`pkttyagent`, citado en polkit(8)) para la sesión de recuperación. Pendiente.

### 2.8 C-28 — xdg-activation y `DBusActivatable`

| Concepto | Definición | Uso |
|---|---|---|
| `app_id` | Lo declara el cliente [V S7] | Agrupación y presentación |
| Desktop File ID | Ruta relativa con `.desktop` [V S6] | Lo que se lanza |
| `Exec` | Línea de comando con códigos de campo | Solo si **no** se usa `DBusActivatable` [H: la sección de claves reconocidas de S6 no se leyó; pendiente de confirmar] |
| `DBusActivatable=true` | La app se activa por D-Bus en el nombre igual al Application ID | `org.freedesktop.Application.Activate`, `Open` o `ActivateAction` [V S6] |
| `org.freedesktop.Application` | Interfaz que implementa la app | `platform_data` con `activation-token` (mismo valor que `XDG_ACTIVATION_TOKEN`) y `desktop-startup-id` [V S6] |
| Token | Lo genera el compositor a petición de un cliente [V S7] | Solo para foco y activación |
| Identidad y autorización | — | El token **no** es una credencial (`ipc.md` §6.1) [R] |

**Flujo:**
1. **Solicita:** el Shell (u otro solicitante) pide el token con `get_activation_token`, aportando el serial y la superficie de la interacción del usuario [V S7].
2. **Genera:** el compositor (evento `done`).
3. **Entrega:**
   - App con `Exec`: `XDG_ACTIVATION_TOKEN` en el entorno de la unidad.
   - App `DBusActivatable`: `platform_data["activation-token"]` en `Activate` [V S6].
4. **Valida:** el compositor, cuando la app llama a `activate` con el token [V S7].
5. **Sin interacción válida** (sin serial reciente o sin foco): el compositor puede ignorar el token [V S7]. La aplicación se abre **sin robar el foco**; el lanzamiento no falla.

**Consecuencia para systemd y ADR-0014 [H]:** una app `DBusActivatable` la arranca el bus (o systemd vía `SystemdService=` de su archivo de servicio D-Bus). Su unidad **no** sigue el esquema `app-parallax-…`, y resourced debe reconocerla por el nombre de su unidad o servicio. Verificación [VM].

---

## 3. Recomendaciones técnicas (resumen justificado)

1. **Lanzamiento:** C + D. El compositor sigue siendo la autoridad Wayland (tokens, roles) y deja de lanzar aplicaciones. Elimina el riesgo de convertirlo en gestor de procesos sin perder seguridad, porque las apps normales no cruzan ninguna frontera de privilegio.
2. **Polkit:** B con `libpolkit-agent-1` y un rol Wayland `auth` mínimo. Exige completar ADR-0015 (§2.2). A con `fallback` sigue siendo una alternativa documentada.
3. **Cierre de sesión:** lo coordina el compositor. Ejecutan las apps, systemd y logind. Sin garantías de guardado.
4. **systemd:** conservar `Restart=always` para el Shell. Elegir R1 o R2 tras la verificación en VM.
5. **Licencias:** `cargo-deny` es un filtro; la compatibilidad la decide la revisión humana registrada.
6. **Sincronización:** el match se instala antes de suscribir, hay búfer hasta la respuesta, revisiones por `epoch` y se filtra por nombre único. Sin depender del orden de D-Bus.
7. **Activación:** el token lo pide el solicitante; `DBusActivatable` usa `Activate` con `activation-token`.

---

## 4. Contratos y responsabilidades resultantes

| Elemento | Propietario | Cambio respecto a la cuarta revisión |
|---|---|---|
| `Session1`: foco, visibilidad, `SubscribeFocus`, toplevels y workspaces | Compositor | Sin cambio |
| `Session1.Lock` / `Locked` / `Unlocked` | Compositor | Sin cambio |
| `Session1.Logout` (ciclo coordinado, §2.3) | Compositor | **Vuelve**, redefinido |
| `Session1.Launch` | — | **Se retira** (§2.1) |
| `Session1.LaunchPrivileged` (roles fijos, polkit) | Compositor | Sin cambio (C-17) |
| Lanzamiento de aplicaciones | Shell y solicitantes (C + D) | Nuevo |
| Validación de tokens | Compositor | Explícito |
| Roles Wayland: `shell`, `lock`, `portal`, `settings-output`, **`auth`** | Compositor (ADR-0015) | `auth` es nuevo (si D-01 = B) |
| Terminación forzosa | logind | Sin cambio |
| Supervisión de apps | systemd | Sin cambio |

---

## 5. Cambios que requerirían los documentos (no aplicados)

| Documento | Cambio |
|---|---|
| `architecture/arquitectura.md` §2 | Unidades según §2.4; política de reinicio del Shell sin cambiar hasta D-18 |
| `architecture/arquitectura.md` §5 | Flujo de apertura (C + D); cierre de sesión |
| `architecture/ipc.md` §6.1 | Quitar `Launch`; redefinir `Logout`; añadir `SubscribeFocus` y `LaunchPrivileged`; regla del `sender` por nombre único |
| `architecture/eventos.md` §4 | Paso de `AddMatch` previo y búfer; no depender del orden de D-Bus |
| `specifications/dock.md` | `Session1.Launch` → lanzamiento del Shell; tokens |
| `specifications/compositor.md` §10.2, §11.4 | Sin `Launch`; terminal de recuperación como unidad fija; coordinación del cierre |
| `specifications/shell.md` §9 | Rol `auth` (si B) |
| `specifications/lockscreen-greeter.md` | Sin cambio |
| `security/dependencias.md` regla 3 | Matiz de `cargo deny` (§2.5) |
| `security/modelo-seguridad.md` | Límite de identidad en el bus de sesión; rol `auth` |
| `.claude/agents/ingeniero-seguridad.md`, `ingeniero-compositor.md` | Catálogo de roles con `auth` (los revisa ingeniero-seguridad) |

## 6. Impacto en ADR (ningún estado cambia)

| ADR | Impacto |
|---|---|
| 0006 | El Shell pasa a coordinar el lanzamiento |
| 0007 | `Session1` sin `Launch`; `Logout` redefinido |
| 0008 | `libpolkit-agent-1` y GLib como dependencias; matiz de `cargo-deny` |
| 0010 | B requiere revisión de §2 y §9; A con `fallback` no la requiere |
| 0014 | Apps `DBusActivatable` fuera del esquema de nombres; estado SYNCED/UNKNOWN |
| 0015 | Catálogo de roles, rol `auth`, higiene del fd, revocación y entrega de fd a servicios (R1) |

## 7. Riesgos nuevos o modificados

| ID | Riesgo | Mitigación |
|---|---|---|
| R-01 (mod.) | No existe una entrega de fd viable a un `.service` | R2 con política de reintentos en el compositor |
| R-02 (mod.) | El comportamiento de `fallback` y de la desregistración no está probado | [VM] |
| R-08 (mod.) | Si el compositor lanzara apps, se convertiría en gestor genérico | Resuelto por C + D, si se aprueba |
| R-11 | Hueco sin agente polkit durante el reinicio (B) | Reinicio rápido; polkitd devuelve error al solicitante [H] |
| R-12 | Un proceso del mismo UID ocupa el nombre D-Bus y falsea el foco | Modo observación; límite documentado |
| R-13 | `DBusActivatable` escapa al control de resourced | Reconocer su unidad [VM] |
| R-14 | GLib y `unsafe` por FFI con `libpolkit-agent-1` | Revisión de ingeniero-rust; aislamiento en el proceso del agente |
| R-15 | Aplicaciones que ignoran `close` bloquean el cierre de sesión | Fase 5 con decisión del usuario; modo forzado |
| R-16 | Los apps sin Shell solo pueden lanzar la terminal de recuperación | Documentado como comportamiento de emergencia |

## 8. Matriz C-01 a C-28

Estado = situación del hallazgo en los documentos originales. «Corregido en propuesta» no significa resuelto.

| ID | Corrección propuesta | Documento corregido | Decisión aprobada | Evidencia | Estado |
|---|---|---|---|---|---|
| C-01 | D-01 (B recomendada; A con fallback viable) | No | No | ADR-0010, S5 | Pendiente |
| C-02 | `Session1` unificada | No | No | `ipc.md`, `compositor.md` | Parcial |
| C-03 | Esquema S1 | No | No | S1 | Parcial |
| C-04 | C + D (§2.1) | No | No | S7 | Pendiente |
| C-05 | Retirar o versionar | No | No | ADR-0007 | Parcial |
| C-06 | — | No | No | — | Pendiente |
| C-07 | Árbol único | No | No | Cinco árboles | Parcial |
| C-08 | `cargo vet` pendiente | No | No | `dependencias.md` regla 11 | Parcial |
| C-09 | D-08 | No | No | — | Pendiente |
| C-10 | Licencia por crate + revisión humana | No | No | ADR-0008, §2.5 | Parcial |
| C-11 | `task-manager` | No | No | — | Parcial |
| C-12 | Desktop File ID; lanzamiento por el Shell | No | No | S6 | Parcial |
| C-13 | `docs/architecture/dbus/` | No | No | `ipc.md` §13 | Parcial |
| C-14 | `parallax-*` | No | No | `AGENTS.md` | Parcial |
| C-15 | — | No | No | — | No verificable |
| C-16 | R1/R2 (§2.4) | No | No | S2 | Pendiente |
| C-17 | `LaunchPrivileged` en `ipc.md` | No | No | Reglas de los agentes | Pendiente |
| C-18 | Alinear con `ipc.md` §13 | — | No | `ipc.md` §13 | Parcial |
| C-19 | `libpolkit-agent-1` | No | No | S5 | Pendiente |
| C-20 | C + D | No | No | S7, S1 | Pendiente |
| C-21 | Siete planos; rol `auth`; especificar ADR-0015 | No | No | Reglas de los agentes, S5 | Pendiente |
| C-22 | Ciclo coordinado | No | No | S3, S2 | Pendiente |
| C-23 | Tabla de ciclo de vida; R1/R2 | No | No | S2 | Pendiente |
| C-24 | Matiz de `cargo-deny` | No | No | `dependencias.md`, ADR-0008 | Parcial |
| C-25 | Protocolo sin dependencia de orden | No | No | S4, `eventos.md` | Parcial |
| C-26 | Filtro por nombre único y límite de UID | No | No | S4 | Parcial |
| C-27 | Infraestructura oficial | No | No | S5 | Pendiente |
| C-28 | Token y `Activate` | No | No | S6, S7 | Parcial |

N-1 a N-12: sin cambios. N-3 a N-9 solo tienen el tema reconstruido a partir de instrucciones posteriores; el resto, PENDIENTE DE EVIDENCIA.

**Decisiones nuevas o modificadas:**
- **D-03:** C + D (antes compositor). Deciden arquitecto y seguridad.
- **D-16:** `libpolkit-agent-1`. Deciden seguridad y propietario.
- **D-18 (nueva):** R1 o R2 para los roles, tras la VM.
- **D-19 (nueva):** plazo y política del cierre de sesión.
- **D-20 (nueva):** generación del `epoch`.

El resto (D-01 a D-17) sigue como en la cuarta revisión.

## 9. Árbol del workspace

Sin cambios estructurales respecto a la cuarta revisión:
- El lanzamiento empieza como **módulo de `shell/shell/`**, no como crate.
- `shell/polkit-agent/` sigue condicional a D-01 = B.
- La unidad fija de la terminal de recuperación iría en `foundation/systemd/user/`.
- Fase 1: solo `px-types`, `px-ipc`, `px-config`, `px-telemetry`.

## 10. Veredicto: `REQUIERE OTRA REVISIÓN TÉCNICA`

**Por qué no está lista para el propietario:** en la comprobación cruzada hay respuestas negativas o sin evidencia.

- **¿La arquitectura de autorización respeta ADR-0015?** Solo si ADR-0015 se completa (catálogo de roles, rol `auth`, revocación, higiene del fd). Hoy está **incompleto** para B (§2.2).
- **¿El agente se recupera sin el Shell?** Sí en B, pero la política de reinicio depende de R1/R2, sin verificar (§2.4).
- **¿Los nombres de unidad son compatibles con systemd?** Siguen S1, pero el escapado y las apps `DBusActivatable` necesitan [VM].
- **¿Orden de mensajes D-Bus?** Sin garantía verificada. El protocolo ya no depende de ella, pero la lectura de S4 fue parcial.

**Lo que falta para pasar a revisión del propietario:**
1. Que ingeniero-seguridad especifique en ADR-0015 los puntos de §2.2 (es trabajo técnico, no del propietario).
2. Leer las fuentes que faltan: la sección de claves reconocidas de S6, la interfaz `AuthenticationAgent` de polkit, logind.conf(5) y el resto de S4.
3. Registrar en `spikes.md`, cuando se autorice, las pruebas [VM] de R1, la sustitución de polkit, el escapado de unidades y `DBusActivatable`.
4. Que el propietario aporte el texto de N-1 a N-12.

La propuesta **no** está aprobada. La Fase 0 sigue en `IN_PROGRESS` y todos los ADR conservan su estado.
