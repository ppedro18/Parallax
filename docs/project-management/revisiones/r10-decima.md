# PARALLAX OS — Décima revisión técnica complementaria

**Tema:** cierre de descriptores antes del intermediario y exclusión atómica de sesiones.

- **Fecha:** 2026-11-19
- **Fase 0:** `IN_PROGRESS`
- **ADR:** todos `Propuesto`
- **Modo:** solo lectura. Este informe está en el scratchpad, fuera del repositorio.

**Marcas de evidencia**

| Marca | Significado |
|---|---|
| `[V]` | Verificado en fuente oficial, lectura completa |
| `[V-p]` | Verificado en fuente oficial, lectura parcial |
| `[R]` | Consta en el repositorio |
| `[H]` | Hipótesis o inferencia |
| `[VM]` | Requiere prueba en VM |
| `[P]` | Decide el propietario |

---

## 1. Resumen ejecutivo

**1. Descriptores.** La solución de la novena revisión **no basta y se corrige.**

- `close_range()` al principio de `main()` llega tarde. Antes de `main` ya se han ejecutado el cargador dinámico y los inicializadores, y todos ellos pueden ver los descriptores heredados.
- La corrección es `posix_spawn_file_actions_addclosefrom_np(5)` como **última acción de archivo**. glibc la ejecuta **en el hijo, antes de `exec`, y en el orden en que se añadió**. La implementa con `close_range` más un mecanismo de respaldo, y si falla el hijo sale con 127 y `posix_spawn` devuelve el error al padre.
- Fuente: código de `spawni.c`, rama `master` [V-p].
- glibc en Arch: **2.44** [V]. La versión mínima de la función es 2.34 [H]: NEWS no se pudo leer; el archivo fuente data de 2021.
- **El fallo es seguro por construcción:** si el cierre no se completa, el intermediario no llega a ejecutarse.

**2. Exclusión de sesiones.** La consulta de `ActiveState` seguida del arranque **no es atómica y se sustituye.**

- Se propone `flock(LOCK_EX|LOCK_NB)` sobre `$XDG_RUNTIME_DIR/parallax/session.lock`, abierto con `O_CLOEXEC` y mantenido por el comando de sesión durante toda la sesión.
- `flock` va asociado a la descripción de archivo abierta y se libera cuando se cierran todas sus copias [V], lo que incluye SIGKILL [H].
- D-29 queda redefinida como **una sola instancia de PARALLAX por usuario**, con garantía interna, más la **detección** de otra sesión gráfica. La exclusividad frente a otros entornos no se puede garantizar.

**Veredicto (§12): `LISTA PARA REVISIÓN DEL PROPIETARIO`.**

- Las dos incertidumbres de diseño quedan resueltas documentalmente.
- Lo que falta son pruebas en VM y decisiones del propietario. Otra revisión en papel no añadiría evidencia.

---

## 2. Fuentes y versiones

| Ref | Fuente | Versión | Lectura |
|---|---|---|---|
| F1 | `posix/spawn_faction_addclosefrom.c`, glibc (espejo bminor/glibc) | Rama `master` | Completa |
| F2 | `sysdeps/unix/sysv/linux/spawni.c`, glibc | Rama `master` | Parcial: casos `spawn_do_closefrom` y `spawn_do_dup2`, bucle de acciones y etiqueta `fail` |
| F3 | Página de manual de Arch `posix_spawn_file_actions_addclosefrom_np.3` | — | **No disponible (404)** |
| F4 | NEWS de glibc 2.34 y `posix/spawn.h` | — | **No leídos** |
| F5 | Paquete `glibc` de Arch | `2.44+r78+gfcfb867c58ab-1` (2026-11-15) | Página del paquete |
| F6 | `flock(2)`, man7 | man-pages 6.19 | Completa |
| F7 | `ld.so(8)`, man7 | man-pages 6.19 | Completa. No documenta constructores antes de `main` |
| F8 | `org.freedesktop.systemd1(5)` | — | Parcial: «name … must be unique». No se encontró el error ante un nombre duplicado |
| — | `close_range(2)`, `posix_spawn(3)`, `fork(2)` | — | Completa (revisiones octava y novena) |

`F2` es la rama `master`, no necesariamente idéntica a la 2.44 de Arch. La equivalencia es `[H]` y se verifica en VM.

---

## 3. Auditoría del cierre de descriptores antes del intermediario

| Etapa | Código que se ejecuta | ¿Puede ver descriptores heredados? |
|---|---|---|
| Hijo de `posix_spawn`, antes de `exec` | Código de glibc (`__spawni_child`) que aplica las acciones de archivo [V F2] | Sí, pero es código de glibc, no del intermediario |
| `execve` del intermediario | El kernel cierra los descriptores con `CLOEXEC` [V `posix_spawn(3)`] | — |
| Cargador dinámico e inicializadores (`init_array`, constructores de dependencias) | Antes de `main` [H; F7 no lo documenta] | **Sí, con la solución de la novena revisión** |
| `main()` → `close_range` | Código del intermediario | Ya tarde |

**Conclusión:** la novena revisión dejaba expuestos los descriptores no `CLOEXEC` al cargador y a los inicializadores. El riesgo es pequeño, porque el intermediario es propio y tiene dependencias mínimas, pero **no es una garantía**.

---

## 4. Comparación de alternativas

| | A. `addclosefrom_np` | B. `CLOEXEC` obligatorio en todo el compositor | C. Arranque especial (estático o `_start` propio) |
|---|---|---|---|
| Cuándo cierra | En el hijo, antes de `exec` [V F2] | Lo hace el kernel en `exec`, solo para los descriptores marcados | Antes del runtime |
| Garantía | Cierra todo lo que esté por encima de `from` [V F2: `close_range(lowfd, ~0U, 0)`] | **No verificable**: Mesa, libinput y otras bibliotecas C, aperturas concurrentes de otros hilos, APIs sin `O_CLOEXEC`, descriptores heredados al arrancar el compositor | Depende del código propio |
| Orden | Igual al orden de inserción [V F2: bucle `for cnt…`] | — | — |
| Fallo | `goto fail` → `_exit(127)`; `posix_spawn` devuelve el error [V F2] | Fuga silenciosa | Depende |
| Versión | glibc ≥ 2.34 [H; F4 no leído]; Arch tiene 2.44 [V F5]; `close_range` ≥ Linux 5.9 [V-p], con respaldo `__closefrom_fallback` [V F2; respaldo no leído] | — | Linker y destino concretos |
| Complejidad y portabilidad | Baja; extensión `_np` de glibc (no musl) [H] | Alta, y no se puede auditar | Alta: ensamblador o `no_std` por arquitectura (x86_64 y aarch64, ADR-0013) |

**Recomendación: A.** B se mantiene como buena práctica, pero no como garantía. C solo tendría sentido si A no estuviera disponible, y lo está.

**Requisito derivado:** PARALLAX v1 depende de glibc. ADR-0002 (Arch) ya lo implica [R]. Hay que documentarlo.

---

## 5. Diseño recomendado para `posix_spawn`

1. **En el padre:** `r = fcntl(rol, F_DUPFD_CLOEXEC, 10)`; lo mismo para el extremo de lectura de sync, stdout y stderr. Así ningún origen es menor que 10.
2. Acciones, **en este orden** [V F2: se ejecutan en orden]:
   1. `addopen(0, "/dev/null", O_RDONLY, 0)`
   2. `adddup2(stdout_alto, 1)`
   3. `adddup2(stderr_alto, 2)`
   4. `adddup2(r, 3)`
   5. `adddup2(sync_alto, 4)`
   6. **`addclosefrom_np(5)`**: última acción. Cierra también los orígenes altos.
3. Llamar a `posix_spawn` con un `envp` construido por lista blanca y la **ruta absoluta** del intermediario.
4. Si `posix_spawn` devuelve error (lo que incluye un fallo de `closefrom`): rol en estado `FAILED`, registro en el log y backoff. **Nunca se reintenta sin cerrar.**
5. **Intermediario** (por defensa): comprueba que solo existen 0–4 (recorriendo `/proc/self/fd`) [H]. Si aparece cualquier otro, sale con error **sin hacer `exec`**. Después espera en el descriptor 4 y hace `execve` del rol con `envp` explícito y el descriptor 4 con `CLOEXEC`.
6. **Rol:** `PR_SET_DUMPABLE=0`, conexión por el descriptor 3, `CLOEXEC` sobre él y `unsetenv("WAYLAND_SOCKET")`.

**Notas sobre `dup2`:**
- Con `fd == newfd`, glibc quita `FD_CLOEXEC` (Austin Group #411) [V F2].
- El diseño no depende de ese caso, porque los orígenes son siempre mayores o iguales que 10.

**Dependencia:**
- Si el crate `libc` expone `posix_spawn_file_actions_addclosefrom_np`: `[H]`, no comprobado en docs.rs.
- Si no la expone, hay que declararla con `extern "C"` (`unsafe` con `// SAFETY:`).
- Afecta a D-27 y ADR-0008.

**Prueba de VM (T-FD):** lanzar el rol con descriptores trampa abiertos en el compositor sin `CLOEXEC` (en número bajo, alto y creados desde otro hilo) y comprobar `/proc/<pid>/fd` en el intermediario y en el rol. Lo esperado es ver solo 0–3 (0–4 en el intermediario). Repetir con `close_range` forzado a fallar, si es posible, para observar el código 127.

---

## 6. Auditoría de exclusión atómica

**Carrera confirmada:** consultar y después arrancar es un patrón TOCTOU: A y B pueden observar a la vez que no hay sesión.

| Mecanismo | Atómica | Dos inicios a la vez | Cierre normal | SIGKILL o caída | Estado residual | Riesgo de bloqueo permanente | Necesidad de mantener un descriptor |
|---|---|---|---|---|---|---|---|
| **`flock` en `$XDG_RUNTIME_DIR`** | Sí: `LOCK_EX\|LOCK_NB` → `EWOULDBLOCK` si otro lo tiene [V F6] | Solo uno lo obtiene [V F6] | Se libera al cerrar | Se libera cuando se cierran todas las copias del descriptor; el kernel las cierra al morir el proceso [V F6 + H] | El archivo puede quedarse, pero **sin bloqueo**: no impide nada | Solo si algún hijo hereda el descriptor → `O_CLOEXEC` obligatorio [V F6: «preserved across an execve»] | Sí, durante toda la sesión |
| Unidad systemd que representa el bloqueo | Depende del manager | — | — | Ligado al ciclo de vida de la unidad | Una unidad colgada bloquea | Medio | No |
| Unidad transitoria con nombre único | «name … must be unique» [V-p F8]; el error y el comportamiento ante un duplicado no se encontraron | [VM] | — | Scope vacío liberado [V] | Bajo | [VM] | No |

**`XDG_RUNTIME_DIR`:** es por usuario, con permisos 0700, y logind lo crea y lo elimina [H; logind no se leyó en esta revisión]. El bloqueo pertenece al usuario y solo un proceso del mismo UID puede interferir. Eso es coherente con el límite ya aceptado (ADR-0015, regla 6 [R]).

**Recomendación: `flock`.** Es la opción documentada, es atómica y no deja bloqueos residuales.

**Flujo recomendado (comando de sesión):**

1. `open($XDG_RUNTIME_DIR/parallax/session.lock, O_RDWR|O_CREAT|O_CLOEXEC, 0600)`. Si el directorio falta, se crea con 0700.
2. `flock(fd, LOCK_EX|LOCK_NB)`.
   - `EWOULDBLOCK`: ya hay una instancia de PARALLAX. Se rechaza el inicio (salida ≠ 0) y greetd vuelve al greeter [V greetd(1)].
3. **Con el bloqueo adquirido**, se revisa el estado residual:
   - Si `parallax-session.target` está `active`/`activating`: es una sesión PARALLAX anterior cuyo comando murió. Como se tiene el bloqueo, no hay otra instancia viva. Se ejecuta `StopUnit` y se espera a `inactive`.
   - Si `graphical-session.target` sigue activo **sin** `parallax-session.target`: hay **otro entorno**. Se rechaza el inicio (Política 1, con detección).
4. Arrancar `parallax-session.target` y esperar el resultado del job. Si falla, ejecutar `StopUnit` y salir con código ≠ 0.
5. Mantener el descriptor abierto hasta que el target llegue a `inactive`.
6. Salir. El descriptor se cierra y el bloqueo se libera.

**Ciclo de vida:**
- El comando de sesión no lanza ningún proceso que herede el descriptor: el compositor lo arranca systemd, no el comando.
- Si el comando muere (también con SIGKILL), el bloqueo se libera [V/H]. Las unidades que queden las limpia el paso 3 del inicio siguiente.
- Ventana: si el comando muere mientras el target está activo, la sesión gráfica sigue sin bloqueo hasta que greetd vuelve a lanzar el greeter. En ese momento, otro inicio entraría por el paso 3 y detendría la sesión anterior.
  - Efecto: el usuario pierde esa sesión. Es aceptable porque su comando ya ha muerto y greetd considera la sesión terminada [H].

**Segunda sesión de otro entorno:** el `flock` de PARALLAX **no impide** que otro entorno, que no coopera, arranque su propia sesión y comparta `graphical-session.target` [H; el user manager es compartido].

| Nivel | Qué hay |
|---|---|
| Garantía interna | Una sola instancia de PARALLAX por usuario |
| Detección | Paso 3: si `graphical-session.target` está activo sin `parallax-session.target`, PARALLAX no arranca |
| Cooperación | Solo si el otro entorno hace una comprobación equivalente. No se puede exigir |
| Limitación | Si otro entorno arranca después, la política X de PARALLAX (`StopWhenUnneeded`) y la del otro entorno interfieren en `graphical-session.target` [H] |

---

## 7. Definición precisa de D-29 (para que decida el propietario)

> **D-29 (propuesta).** En v1, PARALLAX garantiza como máximo una instancia de sesión PARALLAX por usuario, mediante un bloqueo `flock` exclusivo que el comando de sesión mantiene en `$XDG_RUNTIME_DIR/parallax/session.lock` durante toda la sesión. PARALLAX se niega a iniciar si detecta otra sesión gráfica activa en el gestor de usuario (`graphical-session.target` activo sin `parallax-session.target`). PARALLAX **no garantiza** que otros entornos gráficos se abstengan de iniciar una sesión del mismo usuario mientras PARALLAX está activo. Ese escenario no está soportado en v1.

**Prueba de concurrencia (T-LOCK):**
1. Lanzar 100 veces, en paralelo, dos comandos de sesión del mismo usuario. Exactamente uno debe adquirir el bloqueo; el otro debe salir con el código de rechazo.
2. Matar con SIGKILL al comando que lo tiene. Un nuevo inicio debe adquirirlo y detener el `parallax-session.target` residual.
3. Con otro entorno activo que use `graphical-session.target`: PARALLAX debe rechazar el inicio.

Las tres pruebas son `[VM]`.

---

## 8. Revisión cruzada

| Elemento | Efecto | Documento que requiere corrección posterior |
|---|---|---|
| D-18 | Ninguno | — |
| D-22 | Lista blanca de descriptores con `addclosefrom_np` y fallo seguro | `security/modelo-seguridad.md` (sustituye la propuesta 3 de la novena revisión) |
| D-24 | Ninguno. El paso 3 limpia lo que deja una caída | — |
| D-27 | Dependencia de `posix_spawn_file_actions_addclosefrom_np` de glibc (≥ 2.34 [H]); posible `extern "C"` | `security/dependencias.md` (`libc` o binding propio); ADR-0008 |
| D-28 | El comando de sesión mantiene el `flock` y limpia el estado residual | `architecture/arquitectura.md` §5 (Arranque) |
| ADR-0015 | Higiene de descriptores en los roles | Propuesta de revisión (sin cambiar su estado) |
| ADR-0002 | Requisito implícito de glibc | Mención explícita |
| `arquitectura.md` | Propuesta 5 de la novena revisión sustituida por el flujo de §6 | §5 |

---

## 9. Matriz de hallazgos

| ID | Hallazgo | Evidencia | Estado actual | Corrección | Decisión del propietario | Prueba pendiente |
|---|---|---|---|---|---|---|
| N10-1 | `close_range` en `main` llega tarde (cargador e inicializadores) | [H] + [V F2] | **Resuelto en diseño, pendiente de implementación** | `addclosefrom_np(5)` como última acción; comprobación defensiva en el intermediario | No (técnica) | T-FD (VM) |
| N10-2 | Versión mínima de glibc sin confirmar | [V F5 2.44], [H 2.34] | Pendiente de prueba en VM | Confirmar en NEWS o con un enlace de prueba | No | VM |
| N10-3 | ¿El crate `libc` expone `addclosefrom_np`? | [H] | Diferible a Fase 1 | `extern "C"` si no la expone | D-27 (ADR-0008) | — |
| N10-4 | Exclusión no atómica | [V F6] | **Resuelto en diseño, pendiente de implementación** | `flock` + flujo de §6 | No | T-LOCK (VM) |
| N10-5 | Alcance de D-29 | [H] | **Pendiente de decisión del propietario** | Definición de §7 | **Sí** | — |
| N10-6 | Convivencia con otros entornos | [H] | Resuelto documentalmente como límite | Detección y rechazo | Sí (dentro de D-29) | T-LOCK.3 (VM) |
| N10-7 | Fallo de `closefrom` | [V F2] | **Resuelto documentalmente** | Fallo seguro: no hay `exec` | No | T-FD (VM) |
| N10-8 | Equivalencia entre `master` y glibc 2.44 | [V-p F2] | Pendiente de prueba en VM | — | No | VM |

**Gate 0:** ningún hallazgo es bloqueante.

---

## 10. Pruebas y spikes pendientes (no autorizados)

| ID | Prueba |
|---|---|
| T-FD | Descriptores trampa: comprobar `/proc/<pid>/fd` en el intermediario y en el rol; fallo de `closefrom` → 127 |
| T-LOCK | Concurrencia ×100, SIGKILL con el bloqueo tomado, convivencia con otro entorno |
| T-GLIBC | `posix_spawn_file_actions_addclosefrom_np` disponible y con el comportamiento descrito en glibc 2.44 de Arch |

Se mantienen las pruebas de la novena revisión (§13).

---

## 11. Riesgos residuales

| ID | Riesgo |
|---|---|
| R-34 | La función `_np` es una extensión de glibc: dependencia de glibc (no de musl) |
| R-35 | Un hijo del comando de sesión podría heredar el descriptor del bloqueo si se olvida `O_CLOEXEC`. Se mitiga con la regla explícita y con T-LOCK |
| R-36 | Otro entorno gráfico del mismo usuario no coopera (fuera de soporte en v1) |
| R-29 | Siguen sin documentarse el código de salida de greetd y el cierre de PAM |

---

## 12. Veredicto: `LISTA PARA REVISIÓN DEL PROPIETARIO`

**Justificación:**
- **Descriptores:** el mecanismo recomendado está **documentado en el código de glibc** (orden de las acciones, uso de `close_range`, fallo seguro con 127) y **disponible en Arch** (glibc 2.44). El fallo es cerrado: si no puede cerrar, no ejecuta. Lo que falta es probarlo (T-FD), no diseñarlo.
- **Exclusión:** `flock` es atómico y se libera al morir el proceso (`flock(2)`). La definición de D-29 no promete más de lo que el sistema puede garantizar. Lo que falta es T-LOCK y la decisión del propietario.
- **Por qué otra revisión documental no aportaría valor:** las incertidumbres que quedan son de comportamiento en la versión instalada (glibc 2.44, systemd 262.2, greetd 0.10.3). Solo una prueba en VM puede eliminarlas.

**Qué significa este veredicto y qué no:**

| Pregunta | Estado |
|---|---|
| ¿El diseño es coherente en papel? | Sí |
| ¿Las APIs están documentadas? | Sí |
| ¿Funciona en Arch? | Sin verificar |
| ¿La implementación es segura? | Sin evaluar, porque no hay código |
| ¿La experiencia de usuario es aceptable? | La decide el propietario (D-24, D-29) |

Estar lista para revisión **no** significa estar aprobada.

---

## Confirmaciones finales

- Todos los ADR siguen en `Propuesto`.
- La Fase 0 sigue en `IN_PROGRESS`.
- No se modificó ningún archivo del repositorio.
- No se ejecutó ninguna prueba ni ningún spike.
- No existe autorización para la Fase 1.
