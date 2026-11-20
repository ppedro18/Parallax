# PARALLAX OS — Sexta revisión técnica de cierre (C-01 a C-28)

- **Fecha:** 2026-11-19.
- **Fase 0:** `IN_PROGRESS`.
- **ADR:** todos en `Propuesto`.
- **Modo:** solo lectura. Este informe vive en el scratchpad de la sesión, no en el repositorio.
- **No es el documento definitivo.**

**Estados que este informe mantiene separados:**
- **Verificación técnica:** lo que dicen las fuentes.
- **Recomendación:** preferencia razonada de esta revisión.
- **Decisión pendiente:** alguien con autoridad debe elegir.
- **Aprobación del propietario:** no se ha producido ninguna.
- **Aceptación de ADR:** ninguno se ha aceptado.

**Convención de evidencia:**
- `[V]` verificado en fuente oficial.
- `[R]` documentado en el repositorio.
- `[H]` hipótesis o inferencia.
- `[VM]` requiere VM.
- `[HW]` requiere hardware.
- `[P]` decisión del propietario.

## Fuentes oficiales y alcance de la lectura

| Ref | Fuente | Sección o API | Lectura |
|---|---|---|---|
| S1 | systemd.io/DESKTOP_ENVIRONMENTS | Unit naming, slices | Completa |
| S2a | systemd.service(5) | `OpenFile=`, `FileDescriptorStoreMax=`, `FileDescriptorStorePreserve=`, `NotifyAccess=`, `Type=notify`, `Restart=` | Completa |
| S2b | systemd.unit(5) | `BindsTo=`, `PartOf=`, `Requires=`, `After=`, `StartLimit*` | Parcial (~73 %) |
| S2c | systemd.scope(5) | Ciclo de vida de los scopes | Completa |
| S2d | systemd.special(7) | `graphical-session.target` | Parcial (fragmento resumido) |
| S2e | sd_notify(3) | `FDSTORE=1`, `sd_pid_notify_with_fds` | Completa |
| S2f | org.freedesktop.systemd1(5) | `StartTransientUnit` | Parcial (~26 %) |
| S3a | org.freedesktop.login1(5) | `TerminateSession`, `Session.Terminate` | Completa en esos métodos |
| S3b | logind.conf(5) | `KillUserProcesses=` | Completa |
| S4 | D-Bus Specification | Nombres, `GetNameOwner`, `NameOwnerChanged`, `GetConnectionCredentials`, orden | ~96 % literal; el resto (Stats, Verbose, Glossary) solo resumido |
| S4b | dbus-daemon(1) | Política por defecto | Parcial (~78 %) |
| S5a | polkit: Authority | `Register…`, `…WithOptions` (`fallback`), `Unregister…` | Completa |
| S5b | polkit: AuthenticationAgent | `BeginAuthentication`, `CancelAuthentication` | Completa |
| S5c | polkit: `PolkitAgentListener`, `PolkitAgentSession`, polkit(8) | Registro, helper, señales | Completa |
| S6 | Desktop Entry Specification | file-naming, recognized-keys, exec-variables, dbus | Completa |
| S7 | xdg-activation-v1, xdg-shell (wayland.app) | Token y activación | Parcial: el espejo truncó parte |

---

## 1. Resumen ejecutivo

Esta revisión resuelve con evidencia las dos dudas técnicas que bloqueaban la quinta: la entrega de capacidades a servicios systemd y las garantías de D-Bus.

**Hallazgos decisivos:**

1. **No existe un mecanismo documentado** para que un compositor sin privilegios entregue un fd ya abierto a un `.service` supervisado por systemd.
   - `OpenFile=` abre una **ruta** desde el gestor de servicios [V S2a].
   - El fd store solo acepta descriptores de los procesos de la propia unidad, según `NotifyAccess=`, o mediante una invocación privilegiada [V S2a, S2e].
   - Por tanto, R1 no es viable tal como estaba planteado. La recomendación pasa a **R2**: el compositor crea el proceso y systemd lo registra como scope. Eso obliga a sustituir `Restart=always` del Shell por una política del compositor, ahora justificada con evidencia.
2. **El bus de sesión permite por defecto que cualquier conexión posea cualquier nombre** [V S4b]. La identidad del productor D-Bus solo es fiable a nivel de UID. `GetConnectionCredentials` ofrece `ProcessFD` (pidfd), que evita las carreras de PID pero no autentica el rol [V S4].
3. **D-Bus no documenta ninguna garantía de orden** en lo leído (~96 %) [V S4]. El protocolo de la quinta revisión no depende del orden, así que se mantiene.
4. **polkit:** `fallback` está documentado; `CancelAuthentication` existe; puede fallar el registro de un segundo agente; **no está documentado** qué ocurre cuando el agente se desconecta [V S5].
5. **Desktop Entry:** con `DBusActivatable=true` hay que ignorar `Exec` y enviar un mensaje D-Bus [V S6]. Las reglas de field codes son estrictas y verificables.

**Conclusión:** ya no queda ningún problema técnico abierto que impida presentar la propuesta. Lo que queda son decisiones con alternativas claras, trabajo de especificación de ADR-0015 (de ingeniero-seguridad, sujeto a aprobación) y verificaciones en VM que hay que autorizar.

**Veredicto (§13): `LISTA PARA REVISIÓN DEL PROPIETARIO`**, con las condiciones y límites que se detallan en §13. No implica aprobación.

---

## 2. Verificación de C-20 a C-28

### C-20 — Lanzamiento de aplicaciones

**Verificación:**

| Punto | Resultado |
|---|---|
| Cualquier cliente puede pedir un token; el compositor puede ignorarlo | [V S7] |
| `DBusActivatable=true` → ignorar `Exec` y activar por D-Bus | [V S6 recognized-keys] |
| `Exec` es obligatorio si no hay `DBusActivatable` | [V S6] |
| Field codes: como máximo uno de `%f %u %F %U`; `%F`, `%U` e `%i` solo como argumento aislado; los códigos obsoletos se eliminan; un código desconocido invalida la línea; reglas de comillas | [V S6 exec-variables] |
| `org.freedesktop.Application` con `Activate`, `Open`, `ActivateAction` y `platform_data` (`activation-token`, `desktop-startup-id`) | [V S6 dbus] |
| Crear unidades en el `systemd --user` propio no requiere privilegio adicional | [H]; coherente con `StartTransientUnit` en el gestor de usuario [S2f, lectura parcial] |

**¿Debe el Shell ser el único coordinador?** No.
- Las aplicaciones PARALLAX (Files con «abrir con», Settings, Terminal) también lanzan.
- Si todas pasan por el Shell, la disponibilidad de Files depende del Shell y se crea un acoplamiento C6 → C5 sin necesidad.
- Si cada app implementa el lanzamiento, se duplica la lógica: parsing de `Exec`, `DBusActivatable`, unidad, token.

**Arquitectura mínima recomendada:**

| Elemento | Propietario | Capa | Notas |
|---|---|---|---|
| Lógica de lanzamiento (validar, resolver, field codes, `DBusActivatable`, unidad, token) | **Una biblioteca compartida**, inicialmente un módulo; pasa a crate solo con un segundo consumidor real | C5/C6 | No es crate de Fase 1. Nombre: decisión D-21 |
| Solicitantes | Shell y apps PARALLAX, cada uno con su propio token (serial de su interacción) | C5/C6 | No hace falta un servicio |
| Validación de tokens | Compositor | C4 | Su autoridad Wayland |
| Roles privilegiados | Compositor (`LaunchPrivileged`) | C4 | ADR-0015 |
| Terminal de recuperación | Compositor, con `StartUnit` sobre una unidad estática | C4 | Sin parsing |
| Apps de terceros que lanzan otras apps | Su propio toolkit | — | El proceso hijo puede acabar en la unidad del solicitante [H]; resourced no debe asumir una unidad por app (ADR-0014 ya lo reconoce [R]) |
| Portales | `xdg-desktop-portal` (OpenURI) para apps sandbox | C1/C2 | Se mantiene `servicios.md` §13 [R] |

**¿Por qué no un servicio compartido?** Para apps normales no hay frontera de privilegio [H]. Un servicio solo añadiría un proceso y un contrato. Además, sin superficie propia el servicio no tendría un serial de interacción y el compositor podría ignorar su token [V S7].

**Retirar `Session1.Launch` obliga a cambiar:**
- `ipc.md` §6.1.
- `arquitectura.md` §5.
- `dock.md` (líneas 328 y 735–756).
- `compositor.md` §10.2 (no lo lista, sin cambio).
- `lockscreen-greeter.md` (no lo usa).
- `.claude/agents/ingeniero-seguridad.md`: `LaunchPrivileged` se mantiene, sin cambio.

**Estado:** corregido en propuesta, pendiente de aprobación (D-03).

### C-21 — Rol Wayland `auth`

**Lo que existe en el repositorio [R]:**
- Capacidad por socket creada por el compositor y entregada como fd a procesos que él lanza («la posesión del fd es la capacidad»).
- El socket público no anuncia globals privilegiados.
- Los clientes en sandbox nunca reciben globals privilegiados.
- Límite: no hay aislamiento entre procesos del mismo UID; mitigación `ptrace_scope=1` (`ingeniero-seguridad.md` reglas 1–6).

| Concepto | Qué es aquí |
|---|---|
| Identidad | La conexión llegó por el socket del rol X. Es la única identidad que conoce el compositor [R] |
| Autenticación | **No hay autenticación criptográfica.** La posesión del fd vale como prueba [R] |
| Autorización | El compositor anuncia o rechaza globals según el rol [R] |
| Capacidad | El fd |
| Privilegio del proceso | **Ninguno adicional.** UID del usuario, sin capacidades del kernel [R ADR-0010 §2] |

**¿Es una frontera frente a procesos maliciosos del mismo UID?** Parcialmente:
- Un proceso que no es descendiente del rol, con `kernel.yama.ptrace_scope=1`, no puede hacer ptrace al rol ni robar su fd [H basada en semántica de Yama; verificar en VM].
- Sí puede: leer `$HOME`, suplantar nombres D-Bus [V S4b] o lanzar procesos.
- **No es una frontera completa.** Es una barrera contra procesos no sandbox ajenos y contra clientes en sandbox.

**Viabilidad por aspecto:**

| Aspecto | ¿Implementable con lo documentado? | Requiere diseño adicional |
|---|---|---|
| Catálogo de roles | Sí: tabla en el compositor [R] | Añadir `auth` y sus globals |
| Creación del socket | `socketpair` y `WAYLAND_SOCKET` [R, `ingeniero-compositor.md`] | — |
| Entrega del fd | Solo a un hijo directo del compositor (§C-23) | — |
| Herencia y transferencia | No se pueden impedir técnicamente [H] | Higiene: `CLOEXEC`, borrar `WAYLAND_SOCKET` |
| Revocación | El compositor cierra la conexión [H] | Definir cuándo |
| Suplantación de la superficie | Política del compositor: solo el rol `auth` presenta al nivel de autenticación | Especificarlo en ADR-0015 |
| Foco exclusivo | Política del compositor (ADR-0010 §6 [R]) | Especificarlo |
| Relación con Locked | `shell.md` §9: nunca por encima de un bloqueo [R] | Política de cola: si llega una petición en Locked, se pospone o se cancela |
| Caída del compositor | Se pierden las conexiones; al reiniciar se relanza el rol | La cancelación de la petición la gestiona polkitd [H, sin documentar; VM] |

**Conclusión:** todo es implementable con mecanismos existentes. Ninguna parte depende de una garantía inexistente, **siempre que la documentación no presente el socket como frontera frente al mismo UID**. Falta especificarlo en ADR-0015 (responsable: ingeniero-seguridad; aprobación [P]).

**Estado:** pendiente de decisión (D-01, D-22).

### C-22 — Cierre de sesión

**Verificación:**

| Hecho | Fuente |
|---|---|
| `TerminateSession` y `Session.Terminate` terminan la sesión «forcibly» | [V S3a] |
| `KillUserProcesses=` mata los procesos del usuario al cerrar sesión. Valor por defecto «yes» según la página upstream; la distribución puede cambiarlo; root excluido por defecto | [V S3b]. El valor efectivo en Arch: [VM] |
| `BindsTo=`: detiene la unidad si la unidad listada se detiene inesperadamente; `PartOf=`: propaga parada y reinicio; `After=`: solo orden | [V S2b] |
| `graphical-session.target` está activo durante la sesión gráfica y la sesión lo arrastra con `BindsTo=` | [V S2d, desde un resumen] |
| Wayland no garantiza el guardado; el evento `close` es una petición que la app puede ignorar | [H], según la semántica de xdg-shell |

**¿Compositor o gestor de sesión independiente?**

| Criterio | Compositor como coordinador | Session manager independiente |
|---|---|---|
| Conoce los toplevels | Sí | No; necesitaría `Session1` |
| Sobrevive a la caída del Shell | Sí | Sí |
| Si el compositor cae | El cierre ya no tiene sentido gráfico; limpian systemd y logind | Podría terminar la limpieza, pero también la hacen systemd y logind |
| Proceso nuevo | No | Sí, sin responsabilidad única: las fases 6–9 ya las cubren systemd y logind |

**Recomendación:** el compositor coordina las fases 1–5 y pide las 6–9 a systemd y logind.

**Modos de cierre:**

| Modo | Comportamiento propuesto |
|---|---|
| Normal | Fases 1–9 de la quinta revisión |
| Forzado | Sin `close`: se para `parallax-session.target` y después `TerminateSession` |
| Caída del compositor | Las apps pierden Wayland. Si `parallax-session.target` cae por `BindsTo` y `Requires`, la sesión termina según las dependencias configuradas [VM]. No hay guardado |
| Desde el greeter | No hay sesión de usuario; greetd y PAM cierran la sesión del greeter [R `arquitectura.md` §5]. Fuera del ciclo |
| Durante Locked | **No se ofrece un cierre normal sin autenticar**: cerrar las apps sin autenticación permitiría a un tercero destruir el trabajo del usuario [H]. Si la política permite un «cerrar sesión» desde la pantalla de bloqueo, es forzado y pide confirmación: decisión D-23 [P] |

Ninguna fase promete que se guarden los datos.

**Estado:** corregido en propuesta, pendiente de aprobación (D-19, D-23).

### C-23 — Ciclo de vida systemd

**Verificación de los mecanismos:**

| Mecanismo | Qué demuestra | Qué no demuestra |
|---|---|---|
| `OpenFile=` [V S2a] | El gestor de servicios abre una **ruta** (o hace `connect()` a un socket `AF_UNIX`) y pasa el fd al servicio | Que acepte un fd ya abierto por otro proceso: **no** |
| `FileDescriptorStoreMax=` y `FDSTORE=1` [V S2a, S2e] | Los procesos de la unidad, según `NotifyAccess=`, pueden guardar fds en el gestor y recuperarlos en el siguiente arranque | Que un proceso externo y sin privilegios (el compositor) pueda guardarlos: según S2e, saltarse `NotifyAccess` requiere una invocación privilegiada |
| `FileDescriptorStorePreserve=` [V S2a] | Conserva los fds entre reinicios (valor por defecto `restart`) | — |
| `Type=notify` / `NotifyAccess=` [V S2a] | Valores `none`/`main`/`exec`/`all` | — |
| `StartTransientUnit` [V S2f parcial] | Firma `(name, mode, properties, aux)` | No encontré propiedades que transporten fds en lo leído (~26 %) |
| Scopes [V S2c] | Gestionan procesos externos; viven mientras quede un proceso; no tienen `Restart=` | — |
| `Restart=` y `StartLimit*` [V S2a, S2b] | Reinicio con límite de frecuencia | — |

**Conclusión [H a partir de V]:** no hay un mecanismo documentado y seguro para entregar a un `.service` la conexión Wayland que crea el compositor. R1 queda **descartada mientras no aparezca una evidencia nueva**.

**Alternativas:**

| Opción | Seguridad (frente a ADR-0015) | Supervisión y recuperación | Complejidad | Autoridad sobre el rol | ¿Duplica políticas? |
|---|---|---|---|---|---|
| **R2:** el compositor hace fork y registra un scope | Conserva el modelo de fd [R] | El compositor reinicia; systemd agrupa y limita recursos | Media | Compositor | Sí, pero acotada a un catálogo fijo de roles |
| R3: `.service` + `OpenFile=` sobre un socket con ruta | Cualquier proceso del mismo UID puede conectarse a esa ruta [H]. Debilita la barrera de no descendientes | systemd | Baja | Difusa | No |
| R4: `.service` que se conecta y el compositor lo autentica por `ProcessFD` (pidfd) más su unidad o cgroup | El pidfd evita la carrera de PID [V S4]. La pertenencia a un cgroup dentro de la delegación del usuario puede manipularla el mismo UID [H] | systemd | Alta | Compositor más systemd | No |
| R5: broker de capacidades independiente | Traslada el problema, que sigue igual | Proceso nuevo | Alta | El broker | — |

**Recomendación: R2** para los roles privilegiados (Shell, lock, agente `auth`, backend de portal, Settings con salidas):
- Es la única opción que mantiene la garantía de ADR-0015 con mecanismos documentados.
- Coste: el compositor implementa una política de reinicio **acotada**:
  - Catálogo fijo; nunca arbitrario.
  - Los mismos límites que la documentación actual (5 por minuto, `RestartSec` de 200 ms) [R `arquitectura.md` §5].
  - Backoff.
  - Detección de bucles: superado el límite, overlay de emergencia [R].
- **Esto justifica sustituir** `parallax-shell.service` con `Restart=always` por la política del compositor. Es un cambio en `arquitectura.md` §2 y §5 que exige aprobación (D-18).

**Modelo resultante:**

| Componente | Lo inicia | Lo supervisa y reinicia | Unidad / slice | Orden |
|---|---|---|---|---|
| greetd / `parallax-greeter` | systemd (sistema) | systemd | sistema | Antes de la sesión [R] |
| `parallax-compositor` | `parallax-session.target` | systemd (`Restart=` por decidir: un reinicio no recupera las apps [R]) | `.service`, `session.slice` | `BindsTo=graphical-session.target` [V S2d]; `After=` settingsd [R] |
| Shell, agente `auth`, lock, portal, Settings con salidas | Compositor (fork + scope) | Compositor (catálogo, límites, backoff) | `.scope`, `session.slice` (Settings: `app.slice`) | Después de que el compositor esté listo |
| Apps | Solicitante (biblioteca de lanzamiento) | systemd, sin reinicio | `app-parallax-<AppID>@<RANDOM>.service`, `app.slice` [V S1] | — |
| Apps `DBusActivatable` | Activación D-Bus | systemd si su archivo de servicio D-Bus declara `SystemdService=` [V S4, S2a ejemplo 7] | La unidad de la app | — |
| Terminal de recuperación | Compositor, `StartUnit` | systemd | `.service` estático | — |

**Verificaciones [VM]:**
- Registro del scope tras el fork sin carrera (el hijo espera en un pipe).
- Herencia del cgroup.
- `BindsTo` y `PartOf` al caer el compositor.
- Valor efectivo de `KillUserProcesses` en Arch.

**Estado:** pendiente de decisión (D-18).

### C-24 — Licencias

| Documento | Texto | ¿Contradicción? |
|---|---|---|
| ADR-0008 §6 [R] | «`cargo-deny`: auditoría de licencias, fuentes y dependencias»; §5 [R]: «Todas las dependencias deberán pasar por una revisión de licencia» | No: exige revisión **y** herramienta |
| `dependencias.md` regla 3 [R] | «… compatibilidad con el consumidor. … Verificado por `cargo deny`» | **Sobreafirma**: atribuye a la herramienta la verificación de compatibilidad; la regla 1 sí exige el campo «compatibilidad» con revisor |

La separación de la quinta revisión (§2.5) se mantiene, con un matiz: **ADR-0008 y `dependencias.md` no se contradicen en lo sustancial**. Solo la última frase de la regla 3 es imprecisa.

Corrección propuesta: «Filtrado automático por `cargo deny`; la compatibilidad la determina la revisión registrada (regla 1)». ADR-0008 sin cambios.

Obligaciones de distribución: `dependencias.md` regla 9 [R]; ADR-0008 §… (avisos y fuentes) [R].

**Estado:** corregido en propuesta, pendiente de aprobación.

### C-25 y C-26 — D-Bus

**Verificación [V S4, S4b]:**

| Elemento | Resultado |
|---|---|
| Orden de mensajes | **No hay garantía documentada** (~96 % leído literalmente; las únicas apariciones de «order» no tratan de entrega) |
| Nombres únicos | «never reused for two different connections to the same bus» |
| `GetNameOwner` | Devuelve el nombre único; error `NameHasNoOwner` si no hay dueño |
| `NameOwnerChanged` | Notifica el cambio de propietario |
| `GetConnectionCredentials` | `UnixUserID`, `UnixGroupIDs`, `ProcessID`, **`ProcessFD`** (pidfd), `LinuxSecurityLabel` |
| Posesión de nombres | Sin la opción de reemplazo, solo se obtiene el nombre si no tiene dueño. El bus de sesión «normally allows any connection to own any name» [V S4b] |

**¿Pierde eventos el protocolo?**
1. `AddMatch` filtrado por el nombre único U se instala **antes** de `SubscribeFocus`.
2. Las señales que lleguen antes de la respuesta van a un búfer.
3. La respuesta trae `(epoch, n)`.
4. Se descarta todo lo que tenga revisión ≤ n.

Un evento con revisión n+1 emitido antes de la respuesta queda en el búfer y no se pierde. Uno emitido antes del `AddMatch` tiene revisión ≤ n o queda reflejado en el snapshot, **siempre que el propietario haga atómicos el registro y la captura** (obligación del contrato, `eventos.md` §4.1 [R]).

Si llega primero una señal de revisión mayor y la respuesta después, el búfer resuelve el caso. Si se pierde una señal, el hueco obliga a resincronizar. **El protocolo es correcto sin suponer orden** [H; razonamiento sobre V].

Riesgo de carrera: entre `GetNameOwner` y `AddMatch` el propietario puede cambiar. Mitigación: comprobar `NameOwnerChanged` y repetir desde el paso 1 si U cambió [H].

**Modelo de amenaza para resourced:**
- Como el bus de sesión permite poseer cualquier nombre libre [V S4b], un proceso del mismo UID puede tomar `org.parallax.Session1` si el compositor no lo tiene, o anticiparse al arrancar.
- `UnixUserID` no lo distingue.
- `ProcessFD` permite comprobar **qué proceso** es (p. ej. su unidad mediante systemd) sin carreras de PID [V S4]; usarlo así es una hipótesis que habría que verificar.
- Que el compositor tome el nombre lo antes posible es una mitigación [H].

**¿Es suficiente?** Sí, para resourced, por tres motivos:
- Actúa en modo observación y de forma reversible (ADR-0014 [R]).
- Un proceso del mismo UID ya puede afectar a las apps del usuario (límite de ADR-0015 regla 6 [R]).
- Las acciones privilegiadas pasan por polkit [R].

**Corrección documental:** `modelo-seguridad.md`, fila D-Bus («verificación de peer credentials») debe aclarar que en el bus de sesión eso solo prueba el UID.

**Estado:** corregido en propuesta, pendiente de aprobación.

### C-27 — polkit

**Verificación [V S5]:**

| Punto | Resultado |
|---|---|
| Registro normal | `RegisterAuthenticationAgent(subject, locale, object_path)` |
| `fallback` | Solo se usa si no hay otro agente sin `fallback` |
| Segundo agente | «registration … can fail; for example another authentication agent may already be registered for subject». **No es seguro que coexistan dos agentes** |
| Desregistro | `UnregisterAuthenticationAgent(subject, object_path)` |
| Cancelación | El demonio llama a `CancelAuthentication(cookie)`; el agente responde con `Error.Cancelled` cuando el usuario descarta el diálogo; `polkit_agent_session_cancel()` emite `completed` |
| Desconexión del agente | **No documentado** [VM] |
| `PolkitAgentListener` | Flag `RUN_IN_THREAD` |
| `PolkitAgentSession` | «trusted suid helper» con tubería; señales `request`/`show-error`/`show-info`/`completed` |
| Nombre del helper | Las páginas leídas no nombran `polkit-agent-helper-1`, solo «private setuid helper». No lo presento como verificado |
| PAM | A cargo del helper [H, coherente con S5 y ADR-0010 §2] |

**¿Justifica B el coste de revisar ADR-0010 y completar ADR-0015?**

| Criterio | A (Shell + recuperación con `fallback`) | B (proceso propio) |
|---|---|---|
| Ventaja de seguridad frente al mismo UID | Ninguna | **Ninguna real**: mismo UID [R regla 6] |
| Superficie de código que procesa contraseñas | El Shell completo (widgets, render, extensiones) | Un proceso pequeño |
| Aislamiento de fallos | Un fallo del Shell interrumpe la autenticación; entra el agente de recuperación | Independiente del Shell |
| Coexistencia de agentes | Depende de `fallback` (documentado), con comportamiento real [VM] | No aplica |
| Implementaciones de UI | Dos | Una |
| Cambios de ADR | Ninguno en 0010; ADR-0015: rol para el agente de recuperación | 0010 (§2 y §9) y 0015 (rol `auth`) |
| Integración oficial | `libpolkit-agent-1` en dos procesos | En uno |

**Conclusión técnica:** B aporta menos superficie y aislamiento de fallos, no una frontera de seguridad. La diferencia es de **robustez y mantenimiento**, y ambas opciones son técnicamente viables. Es una decisión para el propietario [P]. Recomiendo B por mantenimiento (una sola implementación) y por reducir el código que procesa credenciales, pero la ventaja es moderada.

Se descarta un helper PAM propio (ADR-0010 §7 [R]).

**Estado:** pendiente de decisión (D-01, D-16).

### C-28 — Desktop Entry y activación

| Punto | Resultado |
|---|---|
| Desktop File ID | Incluye `.desktop`; ruta relativa con `/` → `-` [V S6] |
| `Exec` y field codes | Reglas de C-20 [V S6] |
| `DBusActivatable` | Ignorar `Exec` y enviar un mensaje D-Bus [V S6] |
| Métodos | `Activate`, `Open`, `ActivateAction` con `platform_data` [V S6] |
| Token | `activation-token` = valor de `XDG_ACTIVATION_TOKEN`; `desktop-startup-id` = `DESKTOP_STARTUP_ID` [V S6] |
| `SystemdService` | Los archivos de servicio D-Bus pueden declararlo [V S4]; systemd.service(5), ejemplo 7 [V S2a] |

**Cómo funciona sin `Exec`:**
1. La biblioteca de lanzamiento llama a `org.freedesktop.Application.Activate` (u `Open` con URIs) en el nombre igual al Application ID, con el token.
2. El bus activa el servicio. Si declara `SystemdService=`, lo arranca systemd.

**Identificación por resourced:**
- La unidad de una app `DBusActivatable` la define el paquete, no sigue `app-parallax-…`.
- resourced **no debe asumir nombres de unidad**. Debe identificar la app por:
  - La unidad que contiene el PID; se obtiene de systemd y se correlaciona con `ProcessFD` o el cgroup [H].
  - Su relación con el Desktop Entry: `SystemdService` → nombre D-Bus → Application ID [H].
- Si no puede correlacionarla, la trata como no identificada: observa y no actúa (ADR-0014 [R]).

**Estado:** corregido en propuesta, pendiente de aprobación. El reconocimiento por resourced queda [VM].

---

## 3. Hallazgos nuevos

| ID | Hallazgo | Fuente |
|---|---|---|
| C-29 | `arquitectura.md` §5 (caída del Shell: «systemd reinicia `parallax-shell`») es incompatible con ADR-0015 si el Shell recibe un fd del compositor: no hay entrega documentada a servicios | [V S2a, S2e] + [R] |
| C-30 | `modelo-seguridad.md`, fila D-Bus, sugiere que «peer credentials» autentican al servicio; en el bus de sesión solo prueban el UID, y cualquier conexión puede poseer cualquier nombre | [V S4b] |
| C-31 | `modelo-seguridad.md`, fila Shell («ningún otro cliente puede enlazar la global») es cierto en Wayland, pero no contra procesos del mismo UID que hereden o roben el fd | [R] + [H] |
| C-32 | Ningún documento define el cierre de sesión con la pantalla bloqueada | [R] (ausencia) |
| C-33 | Retirar `Session1.Launch` exige un sitio para la lógica de lanzamiento. Ningún documento prevé una biblioteca de lanzamiento | [R] (ausencia) |

---

## 4. Contradicciones transversales

| Área | Verificación | Resultado |
|---|---|---|
| Capas | Única excepción: C3 → C4 de solo lectura (D-17). Ninguna otra llamada sube de capa | Coherente si se aprueba D-17 |
| IPC | Wayland para superficies, D-Bus para servicios; memfd solo como optimización (`widget-engine.md` 176, `observatory.md` 239 [R]); contratos versionados; snapshot+delta | Coherente |
| Fuente única de verdad | Foco: compositor; configuración: settingsd; XML: `docs/architecture/dbus/` | Coherente |
| Shell sin privilegios | Sí; recibe el fd de rol (capacidad Wayland, no privilegio) | Coherente |
| Captura, inyección de entrada, `foreign-toplevel`, `layer-shell` | Matriz de `ingeniero-compositor.md` [R]; captura solo por el backend de portal | Coherente; el rol `auth` debe añadirse a esa matriz |
| Lock | El compositor es dueño de Locked; el lock es un rol (scope) | Coherente |
| Workspace | `px-*` = biblioteca, `parallax-*` = binario; la licencia no se deduce del prefijo | Coherente |
| Dependencias | Ningún binario importa a otro; `px-wm` puro; compositor sin `px-ui`; la biblioteca de lanzamiento no es crate hasta tener un segundo consumidor | Coherente |
| **Pendiente** | `servicios.md` §17 (`portals/`, `systemd/` en la raíz) frente al árbol propuesto (C-07) | Sigue discrepando en el original |

Esta revisión **no introduce contradicciones nuevas** respecto a la cuarta y la quinta. Sí **corrige** la quinta en tres puntos:
- R1 se descarta.
- Se reconoce que la ventaja de B es moderada.
- Las apps también lanzan, no solo el Shell.

---

## 5. Arquitectura recomendada (recomendación, no decisión)

```text
C6 Apps ──┐
C5 Shell ─┼─ biblioteca de lanzamiento ─► systemd --user (StartTransientUnit) / org.freedesktop.Application
          │        └─ xdg_activation token (del propio solicitante)
C4 Compositor ── valida tokens; roles privilegiados por fork+scope+fd; overlay; coordina cierre
          │      Session1: foco, ventanas, Lock, Logout (coordinado), SubscribeFocus, LaunchPrivileged
C3 resourced ── px-dbus (proxy) ── SubscribeFocus (excepción de lectura D-17) ── observa
C2 servicios ── D-Bus versionado (XML en docs/architecture/dbus/)
C1 logind (Terminate), systemd (supervisión), polkitd + helper oficial, D-Bus
```

## 6. Contratos e interfaces afectados

| Contrato | Cambio propuesto |
|---|---|
| `Session1` | − `Launch`; `Logout` = ciclo coordinado; + `SubscribeFocus`; + `LaunchPrivileged` (formalizarlo) |
| Matriz de protocolos del compositor | + rol `auth` (globals mínimos, foco exclusivo, nunca sobre Locked) |
| ADR-0015 | Catálogo de roles, `auth`, higiene del fd, revocación, relanzamiento por el compositor (R2), límite frente al mismo UID |
| `eventos.md` §4 | `AddMatch` previo, búfer, sin dependencia del orden, comprobación de `NameOwnerChanged` |
| `org.freedesktop.Application` | Se consume tal cual (sin contrato propio) |
| `dependencias.md` regla 3 | Matiz sobre `cargo deny` |

## 7. Árbol del workspace recomendado

Sin cambios estructurales respecto a la cuarta y quinta revisiones:
- `docs/architecture/dbus/`: XML.
- `crates/`: los cuatro de Fase 1; `px-dbus` y `px-ui` condicionales.
- `core/{compositor,wm,proto}`.
- `intelligence/resourced`.
- `services/*`.
- `shell/{shell,widgets,lock,greeter,[polkit-agent]}`.
- `applications/*`.
- `foundation/{systemd,dbus(busconfig),udev,polkit,pam,sysctl}`.
- `distribution/`, `tests/`, `tools/` (si aparece) y `vendor/` (D-08).

La biblioteca de lanzamiento empieza como módulo de `shell/shell/`. Al aparecer el segundo consumidor (una app), se convierte en crate en `crates/` (nombre: D-21). **No es de Fase 1.**

## 8. Matriz C-01 a C-33

| ID | Documento afectado | Evidencia | Corrección | Decisión | Prueba pendiente | Estado |
|---|---|---|---|---|---|---|
| C-01 | ADR-0010, `shell.md` §9, `arquitectura.md` | S5, [R] | B o A con `fallback` | D-01 | Polkit en VM | Pendiente de decisión |
| C-02 | `ipc.md` §6.1, `compositor.md` §10.2 | [R] | `Session1` unificada | D-12 | — | Corregido en propuesta, pendiente de aprobación |
| C-03 | `arquitectura.md` §2 | S1 | Esquema S1 | — | Escapado de nombres | Corregido en propuesta, pendiente de aprobación |
| C-04 | `arquitectura.md` §5, `dock.md` | S6, S7 | Biblioteca compartida | D-03 | — | Pendiente de decisión |
| C-05 | ADR-0007 §5.3 | [R] | Retirar o versionar | — | — | Corregido en propuesta, pendiente de aprobación |
| C-06 | `arquitectura.md`, `servicios.md`, `ipc.md` | [R] | — | D-06 | — | Pendiente de decisión |
| C-07 | Cinco árboles | [R] | Árbol único | — | — | Corregido en propuesta, pendiente de aprobación |
| C-08 | `workspace-rust.md` §6 | [R] | `cargo vet` pendiente | D-09 | — | Pendiente de decisión |
| C-09 | `dependencias.md` regla 4 | [R] | — | D-08 | — | Pendiente de decisión |
| C-10 | ADR-0008 | [R] | Licencia por crate + revisión | — | — | Corregido en propuesta, pendiente de aprobación |
| C-11 | `arquitectura.md` §4 | [R] | `task-manager` | — | — | Corregido en propuesta, pendiente de aprobación |
| C-12 | `ipc.md`, `dock.md` | S6 | Desktop File ID | D-03 | — | Corregido en propuesta, pendiente de aprobación |
| C-13 | `arquitectura.md` §3 | `ipc.md` §13 | Ruta ya fijada; faltan los XML | P-06 | — | Parcial |
| C-14 | ADR-0006, 0009, 0010, roadmap | `AGENTS.md` | `parallax-*` | — | — | Corregido en propuesta, pendiente de aprobación |
| C-15 | ADR-0011 | — | — | ADR-0011 | — | No verificable |
| C-16 | ADR-0015, `arquitectura.md` | S2a, S2e | R2 | D-18 | Scope tras fork | Pendiente de decisión |
| C-17 | `ipc.md` | [R] | Formalizar `LaunchPrivileged` | — | — | Corregido en propuesta, pendiente de aprobación |
| C-18 | Propuesta anterior | `ipc.md` §13 | Alinear | — | — | Corregido en propuesta, pendiente de aprobación |
| C-19 | ADR-0010 §7, `dependencias.md` | S5 | `libpolkit-agent-1` | D-16 | — | Pendiente de decisión |
| C-20 | `ipc.md`, `dock.md`, `arquitectura.md` | S6, S7 | Biblioteca compartida + compositor mínimo | D-03, D-21 | — | Pendiente de decisión |
| C-21 | ADR-0015, reglas de los agentes | [R], S4b | Rol `auth`; límites | D-22 | Fd/Yama | Pendiente de decisión |
| C-22 | `ipc.md`, `compositor.md` | S2b, S3a, S3b | Ciclo coordinado | D-19, D-23 | `KillUserProcesses` | Corregido en propuesta, pendiente de aprobación |
| C-23 | `arquitectura.md` §2 y §5 | S2a, S2c, S2e | R2 | D-18 | VM | Pendiente de decisión |
| C-24 | `dependencias.md` regla 3 | [R] | Matiz | — | — | Corregido en propuesta, pendiente de aprobación |
| C-25 | `eventos.md` §4 | S4 | Protocolo independiente del orden | D-20 | — | Corregido en propuesta, pendiente de aprobación |
| C-26 | `modelo-seguridad.md` | S4, S4b | Límite de UID; pidfd | — | — | Corregido en propuesta, pendiente de aprobación |
| C-27 | ADR-0010 | S5 | A frente a B | D-01 | Desregistro | Pendiente de decisión |
| C-28 | `dock.md`, ADR-0014 | S6, S4 | `Activate` + token | — | Reconocimiento por resourced | Corregido en propuesta, pendiente de aprobación |
| C-29 | `arquitectura.md` §5 | S2a, S2e | R2 | D-18 | — | Pendiente de decisión |
| C-30 | `modelo-seguridad.md` | S4b | Aclarar | — | — | Corregido en propuesta, pendiente de aprobación |
| C-31 | `modelo-seguridad.md` | [R] | Aclarar el límite | — | — | Corregido en propuesta, pendiente de aprobación |
| C-32 | `lockscreen-greeter.md` | — | D-23 | D-23 | — | Pendiente de decisión |
| C-33 | Workspace | — | Módulo y luego crate | D-21 | — | Corregido en propuesta, pendiente de aprobación |

Ningún hallazgo está «Resuelto en fuentes originales»: ningún documento del repositorio se ha modificado.

## 9. Decisiones D-01 a D-23

| ID | Decisión | Alternativas | Recomendación | Evidencia | Responsable | Bloquea Fase 0 | VM/HW | ADR | Si se pospone |
|---|---|---|---|---|---|---|---|---|---|
| D-01 | Agente polkit | A con `fallback` / B | B (moderada) | S5 | **Propietario** | No (Gate 2, `fase0-revision.md` [R]) | VM | 0010, 0015 | Fase 3 sin diseño de autenticación |
| D-02 | Capa del agente | C5 / C2 | C5 | [R] | Arquitecto | No | No | 0010 | Inconsistencia documental |
| D-03 | Lanzamiento | Compositor / servicio / Shell / biblioteca | Biblioteca compartida + compositor mínimo | S6, S7 | Arquitecto + seguridad | No | No | 0006, 0007 | `dock.md` sin contrato |
| D-04 | Unidades de app | `.service` / `.scope` | `.service` | S1 | Arquitecto + linux | No | VM | 0014 | — |
| D-05 | Escapado de nombres de unidad | — | — | S1 (sin definir) | Evidencia | No | VM | — | Nombres ambiguos |
| D-06 | Bus de diagd | Sistema / sesión | — | [R] | Arquitecto + seguridad | No | No | — | `Diagnostics1` indefinido |
| D-07 | XML | — | `docs/architecture/dbus/` | `ipc.md` §13 | Arquitecto | No | No | 0007 | — |
| D-08 | `vendor/` | Fase 1 / diferido | — | [R] | **Propietario** | Sí (Gate 0, workspace) | No | 0008 | Sin build offline |
| D-09 | `cargo vet` | Adoptar / no | — | [R] | **Propietario** | No | No | 0008 | Sin auditoría formal |
| D-10 | Spike del overlay | Autorizar / no | Autorizar en Fase 1 | [R] | **Propietario** | No | VM + HW | — | Overlay sin evidencia |
| D-11 | Dueño de `distribution/` | — | — | [R] | **Propietario** | No | No | — | C7 sin dueño |
| D-12 | `Session1` unificada | A / B / C | A | [R] | Arquitecto (P-06) | No (excepción P-06 [R]) | No | 0007 | — |
| D-13 | Texto de N-1 a N-12 | — | — | — | **Propietario** | No | No | — | Trazabilidad incompleta |
| D-14 | Renombrar `Session1` | Mantener / `Compositor1` | Mantener | [R] | Arquitecto | No | No | 0007 | — |
| D-15 | Código generado | Versionado / generado en el build | Versionado + comparación en CI | [H] | Arquitecto + rust | No | No | 0008 | — |
| D-16 | Helper de credenciales | `libpolkit-agent-1` / otras | `libpolkit-agent-1` | S5 | Seguridad + **propietario** (dependencia) | No | No | 0008, 0010 | — |
| D-17 | Excepción de lectura C3 → C4 | Sí / no | Sí | `eventos.md` §5.2 | Arquitecto | No | No | — | resourced sin foco |
| D-18 | Supervisión de roles | R2 / R3 / R4 / R5 | R2 | S2a, S2c, S2e | Arquitecto + seguridad + **propietario** (cambia `Restart=always`) | No | VM | 0015 | Contradicción C-16/C-29 abierta |
| D-19 | Plazos del cierre de sesión | — | Por definir | — | Arquitecto | No | No | — | — |
| D-20 | Generación del `epoch` | Aleatoria / persistida | Aleatoria (128 bits) [H] | `eventos.md` | Arquitecto | No | No | — | — |
| **D-21** (nueva) | Nombre y momento de la biblioteca de lanzamiento | — | Módulo y luego crate | §2 C-20 | Arquitecto + rust | No | No | — | Duplicación |
| **D-22** (nueva) | Especificar el rol `auth` en ADR-0015 | — | Lista de §2 C-21 | [R] | Seguridad; aprobación del **propietario** | No | VM | 0015 | Bloquea B |
| **D-23** (nueva) | Cierre de sesión desde Locked | No ofrecerlo / forzado con confirmación | No ofrecerlo [H] | [R] | **Propietario** (política de producto) | No | No | — | Política de bloqueo incompleta |

Los agentes recomiendan y documentan. Las decisiones marcadas **Propietario** no pueden cerrarse sin él.

## 10. Pruebas pendientes en VM o hardware (no autorizadas)

| Prueba | Tipo | Decisión |
|---|---|---|
| Escapado de nombres (`systemd-escape`) y `systemd-analyze verify` | VM | D-05 |
| Scope tras fork, herencia de cgroup, `FreezeUnit` | VM | D-18, ADR-0014 |
| `BindsTo` y `PartOf` al caer el compositor; valor efectivo de `KillUserProcesses` | VM | C-22 |
| Registro, `fallback`, desregistro y desconexión del agente polkit | VM | D-01 |
| Yama `ptrace_scope=1` frente a robo del fd | VM | C-21 |
| Apps `DBusActivatable` y su identificación por resourced | VM | C-28 |
| Overlay: render y accesibilidad | VM + HW | D-10 |

## 11. Información que debe aportar el propietario

1. El texto de N-1 a N-12.
2. Las decisiones D-01, D-08, D-09, D-10, D-11, D-16 (dependencia), D-18 (cambio de política de reinicio), D-22 (aprobar la especificación) y D-23.
3. La autorización de VM y spikes (registro en `spikes.md`).

**Trazabilidad de N-1 a N-12:**

| Elementos | Situación |
|---|---|
| Rastreables en el repositorio | Ninguno con ese identificador |
| Conocidos solo por instrucciones posteriores | Los temas de N-3, N-4, N-6, N-7, N-8 y N-9 |
| Requieren el texto original | N-1, N-2, N-5, N-10, N-11, N-12, y el texto de todos |

## 12. Riesgos residuales

| ID | Riesgo | Estado |
|---|---|---|
| R-01 | Sin entrega de fd a un `.service` | Confirmado; se mitiga con R2 |
| R-02 | Coexistencia o desregistro de agentes polkit | [VM] |
| R-12 | Suplantación de nombres D-Bus por el mismo UID | Aceptado; impacto acotado |
| R-14 | FFI con GLib | Revisión de `unsafe` |
| R-17 | El compositor acumula una política de reinicio (R2) | Catálogo fijo y límites |
| R-18 | Apps lanzadas por terceros quedan en el cgroup del solicitante | resourced no actúa sobre lo no identificado |
| R-19 | La ventaja de seguridad de B está sobrevalorada en documentos | Corregido en este informe |

## 13. Veredicto: `LISTA PARA REVISIÓN DEL PROPIETARIO`

**Justificación:**
- Las dos dudas técnicas que bloqueaban la quinta revisión ya tienen respuesta con fuente oficial:
  - La entrega de fd a servicios no está documentada, así que se recomienda R2.
  - El orden de D-Bus no está garantizado, y el protocolo no depende de él.
- Ninguna recomendación se apoya en una garantía inexistente.
- No hay contradicciones arquitectónicas abiertas sin una alternativa documentada.

**Lo que queda no es un problema técnico sin resolver:**
- Decisiones explícitas del propietario (§11).
- Trabajo de especificación delimitado (D-22, ADR-0015).
- Verificaciones en VM que requieren autorización (§10).

**Límites de este veredicto:**
- No aprueba nada.
- No acepta ningún ADR.
- No autoriza la VM, la Fase 1 ni la implementación.
- La opción B del agente polkit no puede elegirse sin D-22.
- R2 implica cambiar `arquitectura.md` §2 y §5 (D-18).
- La Fase 0 sigue en `IN_PROGRESS`.
