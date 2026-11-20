# Auditoría de la base del proyecto — Tarea 001

> **Revisión 1 (histórica, no se modifica).** La revisión vigente es [`base-audit-r2.md`](base-audit-r2.md).

- **Fecha:** 2026-11-19
- **Commit auditado:** `0da11d4` (rama `main`)
- **Fase:** 0 — Arquitectura y especificación (sin cambios de estado)
- **Alcance:** solo lectura de `AGENTS.md`, `README.md`, `docs/**`, `.claude/agents/**` y estado Git. No se ha modificado ningún otro archivo, no se ha cambiado el estado de ningún ADR y no se ha instalado nada en el anfitrión.

---

## A. Estado del repositorio

### A.1 Git
- Rama `main`, dos commits (`e7239d8` inicial, `0da11d4` Fase 0).
- Único cambio sin versionar: `.serena/` (directorio de la herramienta Serena del usuario). No se toca. Ver P-17.

### A.2 Inventario (75 archivos versionados, ~21 300 líneas en `docs/`)

| Carpeta | Archivos | Estado |
|---|---|---|
| raíz | `AGENTS.md`, `README.md`, `.gitignore` | `README.md` tiene 3 líneas, sin índice (P-05) |
| `.claude/agents/` | 13 agentes | Completo respecto a la tabla de propiedad de `AGENTS.md` |
| `docs/vision/` | `vision.md` | Presente |
| `docs/architecture/` | `arquitectura`, `ipc`, `eventos`, `servicios`, `hardware`, `actualizacion-recuperacion`, `riesgos`, `workspace-rust` | Presente; faltan `storage.md` y `dbus/*.xml` referenciados (P-06) |
| `docs/specifications/` | 13 especificaciones (compositor, window-manager, shell, topbar, dock, widget-engine, lockscreen-greeter, framework-ui, resource-intelligence, files, terminal, settings, observatory) | Falta `configuration.md` referenciado (P-06); no hay specs para notifyd, launcher, control center, task manager, capture, servicios C2 |
| `docs/design-system/` | `README.md` + 12 documentos | Índice completo y coherente |
| `docs/security/` | `modelo-seguridad.md` (37 líneas), `dependencias.md` (17 líneas) | Mínimos; `dependencias.md` contradice ADR-0008 (P-04) |
| `docs/benchmarks/` | `metodologia.md` | Presente |
| `docs/roadmap/` | `roadmap`, `criterios-aceptacion`, `estrategia-pruebas`, `fase0-revision` | `fase0-revision.md` es una copia de ADR-0009 (P-01) |
| `docs/decisions/` | Plantilla + ADR-0001…0015 | Todos `Propuesto`; formatos heterogéneos (P-10) |
| `docs/project-management/` | este informe | Nueva carpeta |

### A.3 Documentos de AGENTS.md
Todos los directorios del "Mapa de documentación" existen. Los archivos citados (`docs/security/dependencias.md`, `docs/architecture/riesgos.md`, `docs/decisions/ADR-0000-plantilla.md`, `docs/specifications/*`) existen. `rust-toolchain.toml` no existe, lo cual es correcto en Fase 0 (se crea en Fase 1).

---

## B. Problemas encontrados

Severidad: **C** crítico (bloquea cierre de Fase 0) · **A** alto · **M** medio · **B** bajo.

| ID | Sev. | Archivo / sección | Problema | Impacto | Corrección propuesta |
|---|---|---|---|---|---|
| P-01 | C | `roadmap/fase0-revision.md` (todo) | Es **byte a byte idéntico** a `decisions/ADR-0009-actualizaciones.md` (mismo MD5 `1d62e786…`). No contiene ninguna revisión de Fase 0. | `arquitectura.md §6` y `criterios-aceptacion.md` (checklist, último ítem) remiten a este archivo como "resumen de ADR pendientes" → la referencia apunta a contenido equivocado. El criterio de cierre "ADR críticos aceptados" no tiene documento de soporte. Riesgo de editar una copia y divergir de ADR-0009. | Reemplazar el contenido por una revisión real de Fase 0 (puede partir de las secciones C y E de este informe). Requiere autorización: no se modifica en esta tarea. |
| P-02 | C | `ADR-0013` §Decisión vs `ADR-0002` §3 y `roadmap.md §1.1` | ADR-0013 elige una **VM Arch Linux ARM aarch64** como entorno de desarrollo/CI. ADR-0002 basa la distribución en snapshots de **Arch Linux Archive**, que solo cubre x86_64; Arch Linux ARM es un proyecto distinto, sin archivo fechado equivalente ni firma por el mismo llavero. El roadmap §1.1 dice "Linux Arch nativo como entorno de referencia". | Tres documentos describen tres entornos de referencia distintos. El entorno de CI no podría consumir el snapshot que define la distribución → builds no reproducibles respecto a la release, riesgo de diferencias ABI. | Decisión del propietario: (a) VM x86_64 emulada para CI + aarch64 solo para edición/compilación rápida, o (b) host Linux x86_64 dedicado para CI, o (c) aceptar aarch64 con snapshot propio de Arch ARM y documentar la divergencia. Registrar como riesgo nuevo (ver §E). |
| P-03 | A | `ADR-0013` (12 líneas) | Formato mínimo: sin alternativas evaluadas en detalle, sin sección de revisión, sin criterios de aceptación. Afirma "Rust 1.99.1 de Homebrew" en el anfitrión como hecho de contexto. | No se puede aceptar con la evidencia actual; además mezcla un dato del anfitrión (efímero) con una decisión. | Ampliar al formato de ADR-0001…0012 y quitar dependencias de datos del anfitrión. |
| P-04 | A | `security/dependencias.md` §Reglas 2, 4, 10 vs `ADR-0008` §5.1 y `roadmap.md §6.1` | `dependencias.md`: "GPL solo para procesos separados", "LGPL solo enlace dinámico", introduce `cargo vet` y builds reproducibles "bit a bit por release". ADR-0008 permite GPL-3.0-or-later si es compatible, no menciona `cargo vet`; el roadmap dice reproducibles "cuando sea viable". | Política de dependencias contradictoria: la regla 2 haría ilegal enlazar una dependencia GPL en un componente que ADR-0008 ya declara GPL-3.0-or-later. `cargo vet` sería una herramienta nueva sin ADR. | Alinear `dependencias.md` con ADR-0008 una vez decidido (o recoger estas reglas en ADR-0008). Decidir si `cargo vet` entra en el toolchain (ADR-0001 §4 lo omite). |
| P-05 | A | `README.md` | 3 líneas; no indexa documentación, estado de fase ni ADR. | Punto de entrada inútil para revisores; `AGENTS.md` cumple el papel de índice. | Añadir estado, mapa de documentación y enlace a ADR (tras autorización). |
| P-06 | A | Referencias rotas | No existen: `architecture/storage.md` (citado por ADR-0009 §4.1 y `actualizacion-recuperacion.md`), `specifications/configuration.md` (`ipc.md`), `docs/architecture/dbus/*.xml` (`arquitectura.md §3`, agente arquitecto). Esperables en Fase 1: `docs/benchmarks/ui-toolkit.md`, `design-system/tokens/*.toml`. | `storage.md` y los contratos D-Bus XML son base de ADR-0009 y ADR-0007; sin ellos esos ADR no pueden cerrar y el ítem transversal "contratos IPC tienen versión, errores y compatibilidad" queda sin evidencia. | Crear `storage.md` y los XML D-Bus (o marcar como pendiente de Fase 1 explícitamente). `configuration.md` como especificación nueva. |
| P-07 | A | `criterios-aceptacion.md` §Fase 0 | 15 ítems marcados `[x]` mientras las 14 "verificaciones transversales" del mismo archivo están todas `[ ]`. "Tecnologías justificadas — ADR-0001…0013" omite ADR-0014/0015. | El checklist sugiere que Fase 0 está casi cerrada; las verificaciones transversales indican que no. Señal contradictoria al propietario. | Re-evaluar cada `[x]` con evidencia (archivo + sección). Ampliar a ADR-0015. |
| P-08 | M | `roadmap.md` Fase 0 §ADR principales | Lista solo ADR-0001…0009; existen hasta ADR-0015 (0010 polkit, 0011 terminal, 0012 Xwayland, 0013 entorno, 0014 cgroups, 0015 autorización Wayland). | ADR críticos para seguridad (0015) y entorno (0013) no figuran en el gate. | Actualizar la lista y clasificar cuáles son "críticos" para Gate 0. |
| P-09 | M | `roadmap.md §1.2` vs `arquitectura.md §4` | Estructura del workspace distinta: roadmap usa `apps/`, `specifications/`, `architecture/` en la raíz y `tools/`; arquitectura usa `applications/`, `docs/`, `core/`, `shell/`, `intelligence/`, `foundation/`, `vendor/`. | El ingeniero-rust no sabría qué árbol crear en Fase 1. | Elegir uno (arquitectura.md es el más detallado y coherente con AGENTS.md) y corregir el otro. |
| P-10 | M | Nombres de binarios | AGENTS.md: binarios `parallax-<nombre>`. Roadmap y ADR-0009/0011 usan `px-compositor`, `px-update`, `px-terminal`, `px-files`, `px-settings`, `px-task-manager`; arquitectura.md usa `parallax-compositor`, `parallax-updated`. `px-` es el prefijo de **crates**. | Ambigüedad crate vs binario; nombres de unidades systemd y paquetes inconsistentes. | Normalizar a `parallax-*` para binarios en roadmap y ADR. |
| P-11 | M | `ADR-0007 §5.3` vs `AGENTS.md` §Convenciones | AGENTS.md: bus `org.parallax.<Servicio>1` (versionado). ADR-0007 da ejemplos sin sufijo (`org.parallax.Settings`). `arquitectura.md` usa `org.parallax.Session1`. | Contradice la política de versionado de ADR-0007 §9. | Usar sufijo de versión en los ejemplos de ADR-0007. |
| P-12 | M | `ADR-0014`, `ADR-0015` §Decisión | Las reglas normativas ("ámbito, origen, consentimiento, reversibilidad y no-matar"; matriz de protocolos) viven en `.claude/agents/*.md`, no en `docs/`. | Una decisión arquitectónica depende de instrucciones de agentes, que pueden cambiar sin ADR. | Mover esas reglas a `docs/security/` o `docs/specifications/` y referenciarlas desde el ADR. |
| P-13 | M | ADR-0013/0014/0015 | Formato corto distinto del de ADR-0001…0012 y sin "Aprobado por". La plantilla `ADR-0000` también es mínima frente a los ADR largos. | Criterios de cierre no homogéneos. | Fijar una plantilla única (decisión del arquitecto). |
| P-14 | M | `security/modelo-seguridad.md` (37 líneas) | Documento muy breve frente a ADR-0010/0015 y a la verificación transversal "fronteras de confianza documentadas". | La verificación transversal no tiene soporte suficiente. | Ampliar con fronteras de confianza, procesos privilegiados y matriz de amenazas. |
| P-15 | B | `design-system/README.md` l.3 | "Sustituye a `docs/design/sistema-diseno.md`", archivo inexistente. | Nota histórica; confunde. | Eliminar la frase o indicar que el original fue retirado. |
| P-16 | B | `architecture/riesgos.md` R14 | Menciona "Caffyne" como referencia visual sin contexto ni enlace. | Riesgo de licencia/identidad no trazable. | Documentar qué es y su licencia, o retirar. |
| P-17 | B | `.gitignore` | `.serena/` no ignorado. | Riesgo de versionar estado local de herramienta. | El usuario decide si añadirlo a `.gitignore`. |
| P-18 | B | Especificaciones | No existen specs para notifyd, launcher, control center, task manager, capture ni servicios C2 (hardwared, updated, recoveryd, diagd, settingsd). `servicios.md` cubre parte. | Fase 3/4/5 dependen de ellas; no bloquea Fase 1. | Programarlas antes de su fase. |

### B.1 Especificaciones que dependen de decisiones pendientes
- `framework-ui.md`, todo `design-system/` (tokens → `px-ui`), `shell.md`, `topbar.md`, `dock.md`, `widget-engine.md`, apps → **ADR-0005** (toolkit, requiere benchmark).
- `compositor.md`, `window-manager.md` → ADR-0003, ADR-0004, ADR-0015, ADR-0012.
- `resource-intelligence.md` → ADR-0014.
- `lockscreen-greeter.md` → ADR-0010, ADR-0015.
- `terminal.md` → ADR-0011.
- Todas → ADR-0008 (licencia condiciona qué dependencias se pueden elegir; R12).

---

## C. Matriz de ADR

| ADR | Estado | Dependencias | Evidencia pendiente | Recomendación |
|---|---|---|---|---|
| 0001 Rust nativo | Propuesto | — (base de todos) | Versión de toolchain; decisión sobre `cargo vet` (P-04) | **Aceptable ahora** por el propietario |
| 0002 Base Arch | Propuesto | 0008, 0009, 0013 | Conflicto aarch64/ALA (P-02); política de retención; infraestructura de espejo y claves | Aceptar tras resolver P-02 |
| 0003 Smithay | Propuesto | 0001, 0004, 0007 | Spike Smithay (winit + DRM) | Requiere spike Fase 1 |
| 0004 WM en compositor | Propuesto | 0003 | Prototipo `px-wm` puro | Aceptable en principio; validar en Fase 2 |
| 0005 Toolkit UI | Propuesto — benchmark | 0001, 0008 | Benchmark (`benchmarks/ui-toolkit.md`) | Requiere benchmark; aceptación en Gate 1 |
| 0006 Shell un proceso | Propuesto | 0005, 0007, 0015 | Prototipo de aislamiento de widgets | Aceptable en principio |
| 0007 IPC | Propuesto | 0003, 0004, 0006 | Contratos `dbus/*.xml`; benchmark escenarios A–D; P-11 | Aceptar principio híbrido; benchmark en Fase 1 |
| 0008 Licencias | Propuesto — decisión propietario | — | Decisión del propietario; alineación con `dependencias.md` (P-04) | **Bloqueante**: decisión del propietario |
| 0009 Actualizaciones | Propuesto | 0002, 0008 | `storage.md` (P-06); spike Btrfs + systemd-boot | Requiere spike |
| 0010 Polkit agent | Propuesto | 0006, 0015 | Modelo de seguridad ampliado (P-14) | Revisión de seguridad |
| 0011 Terminal VT | Propuesto — spike Fase 4 | 0005 | Spike | Diferible (no crítico para Gate 0) |
| 0012 Xwayland | Propuesto — validación Fase 2 | 0003 | Matriz de compatibilidad | Diferible |
| 0013 Entorno desarrollo | Propuesto | 0002 | Formato completo (P-03); resolución P-02 | **Bloqueante** para Fase 1 |
| 0014 Delegación cgroups | Propuesto | 0001, 0007 | Reglas fuera de `docs/` (P-12) | Diferible (modo observación) |
| 0015 Autorización Wayland | Propuesto | 0003, 0007 | Reglas fuera de `docs/` (P-12); detalle de relanzamiento en Fase 2 | Aceptable en principio tras P-12 |

### C.1 Análisis de ADR fundacionales

#### ADR-0001 — Rust nativo
1. **Decisión:** todo componente propio en Rust estable; FFI encapsulado; toolchain fijado con `rust-toolchain.toml`.
2. **Alternativas:** no evalúa alternativas explícitamente (C/C++, Zig, Go); se presenta como requisito del proyecto.
3. **Consecuencias:** seguridad de memoria y uniformidad; compilaciones lentas, ecosistema gráfico menos maduro (R02).
4. **Dependencias:** base de 0003, 0005, 0007; "toda nueva dependencia FFI requiere un ADR" enlaza con 0008.
5. **Riesgos abiertos:** R02, R15; la regla "un ADR por dependencia FFI" puede ser costosa (libinput, libseat, Mesa vía Smithay).
6. **Evidencia que falta:** versión de toolchain concreta y MSRV; edición 2024 (AGENTS.md) no figura en el ADR; `cargo vet` no está en la lista de herramientas.
7. **Condiciones de aceptación:** fijar la política de versión (no la versión, que se fija en Fase 1), aclarar si las FFI que llegan transitivamente vía Smithay necesitan ADR propio.
8. **Veredicto:** **puede aceptarse ahora** por decisión del propietario; no necesita spike.

#### ADR-0002 — Base Arch con snapshot congelado
1. **Decisión:** snapshot fechado de Arch Linux Archive, espejado y firmado; canales dev/beta/stable; PKGBUILD para paquetes propios.
2. **Alternativas:** Arch rolling, snapshot fechado, Debian/Fedora, LFS.
3. **Consecuencias:** control de versiones y offline; carga operativa de espejo, claves y backports (R07).
4. **Dependencias:** 0009 (rollback), 0008 (licencias de paquetes), 0013 (entorno).
5. **Riesgos abiertos:** R07, R21, R27; **conflicto aarch64 (P-02)** no registrado en `riesgos.md`.
6. **Evidencia que falta:** arquitectura de destino explícita (x86_64 según ADR-0013); tamaño y coste del espejo; política de retención; procedimiento de claves.
7. **Condiciones:** resolver P-02; declarar x86_64 como arquitectura de la release v1 dentro del ADR.
8. **Veredicto:** **necesita decisión del propietario** (P-02) antes de aceptarse; no necesita spike para la decisión, sí para la infraestructura (Fase 6).

#### ADR-0007 — IPC
1. **Decisión:** híbrido Wayland (superficies) + D-Bus/zbus (servicios) + memfd (payloads grandes); sin protocolo IPC propio.
2. **Alternativas:** D-Bus, Varlink, gRPC/UDS, Cap'n Proto/UDS, híbrido.
3. **Consecuencias:** interoperabilidad con systemd/NM/BlueZ/polkit; necesidad de coalescencia y límites; protocolos Wayland privados con autorización (0015).
4. **Dependencias:** 0003, 0004, 0006, 0015, 0008 (licencia de zbus).
5. **Riesgos abiertos:** R25; tráfico de widgets de alta frecuencia (§8) podría requerir canal UDS adicional.
6. **Evidencia que falta:** contratos `docs/architecture/dbus/*.xml` (P-06); benchmark escenarios A–D; nombres versionados (P-11).
7. **Condiciones:** XML de las interfaces de los servicios críticos (Session1, Settings1) con versión y errores; benchmark A–C ejecutable en VM.
8. **Veredicto:** el **principio arquitectónico puede aceptarse ahora**; los criterios de §11 requieren benchmark en Fase 1.

#### ADR-0008 — Licencias y dependencias
1. **Decisión (recomendada):** GPL-3.0-or-later para compositor, shell, apps y servicios; MIT OR Apache-2.0 para `px-*`; CC BY-SA 4.0 para documentación.
2. **Alternativas:** GPL-3.0-or-later, MPL-2.0, MIT OR Apache-2.0.
3. **Consecuencias:** copyleft del entorno; las `px-*` permisivas no pueden depender de código GPL; auditoría obligatoria.
4. **Dependencias:** condiciona 0002 (paquetes), 0005 (licencia del toolkit), 0007 (zbus), `dependencias.md`.
5. **Riesgos abiertos:** R12; contradicción con `dependencias.md` (P-04); DCO vs CLA sin decidir.
6. **Evidencia que falta:** decisión del propietario; si `px-ui` (que envuelve el toolkit) es MIT/Apache o GPL.
7. **Condiciones:** propietario elige licencias; se alinea `dependencias.md`; se decide DCO/CLA.
8. **Veredicto:** **necesita decisión del propietario**; es el bloqueo n.º 1 porque condiciona toda dependencia de Fase 1.

#### ADR-0013 — Entorno de desarrollo
1. **Decisión:** desarrollo y CI en VM Arch Linux aarch64 en el Mac; validación obligatoria en hardware x86_64 (HW-A/HW-B); nada se instala en macOS; la VM se crea con aprobación del propietario.
2. **Alternativas:** VM aarch64, VM x86_64 emulada, máquina Arch x86_64 directa (una línea, sin análisis).
3. **Consecuencias:** anfitrión intacto; rendimiento de VM no representativo; ARM64 se vuelve plataforma de facto.
4. **Dependencias:** 0002 (snapshot), 0001 (toolchain), roadmap §1.1.
5. **Riesgos abiertos:** R08, R24; P-02 (snapshot ALA solo x86_64); virtio-gpu en VM no reproduce el comportamiento DRM/KMS real.
6. **Evidencia que falta:** decisión sobre P-02; definición de HW-A (no existe ficha); herramienta de virtualización concreta y su licencia; cómo se ejecuta CI (local, autoalojado).
7. **Condiciones:** ADR ampliado al formato estándar; P-02 resuelto; ficha HW-A identificada.
8. **Veredicto:** **necesita decisión del propietario** (entorno de referencia y hardware) y reescritura; es bloqueante para iniciar Fase 1.

---

## D. Preparación de Fase 1

### D.1 Requisitos por entorno

| Requisito | VM Linux | Hardware Linux real | macOS (complementario) | Depende de decisión |
|---|---|---|---|---|
| Toolchain Rust fijado (`rust-toolchain.toml`) | Sí | Sí | Solo edición/`cargo check` de crates sin dependencias Linux | ADR-0001 |
| Workspace Cargo + `Cargo.lock` + `vendor/` | Sí | — | Parcial (crates puros) | ADR-0001, ADR-0008, P-09 |
| Build offline `--offline --locked` | Sí | Sí | Parcial | ADR-0008 (qué se puede vendorizar) |
| `cargo fmt`, `clippy`, `cargo-deny`, `cargo-audit` | Sí | — | Sí (crates puros) | ADR-0008 (`deny.toml` deriva de la política) |
| CI local (scripts) | Sí | Ejecutores de aceptación | No como referencia | ADR-0013 (dónde corre CI) |
| Pruebas unitarias / propiedades (`px-types`, `px-ipc`, `px-config`, `px-wm`) | Sí | — | Sí | — |
| Spike Smithay backend winit/anidado | Sí | — | No (Smithay requiere Linux) | ADR-0003 |
| Spike Smithay backend DRM/KMS + libinput + logind | Parcial (virtio-gpu, no representativo) | **Obligatorio** | No | ADR-0003, ficha HW-A |
| Layer-shell, clientes Wayland de prueba (foot) | Sí | Validación | No | ADR-0003 |
| Benchmark toolkit UI (Iced/Slint/libcosmic) | Funcional sí; rendimiento no | **Obligatorio** para memoria/frame time | Prototipos visuales solo como referencia | ADR-0005, ADR-0008 |
| AccessKit / AT-SPI / Orca | Sí | Validación | No (AT-SPI es Linux) | ADR-0005 |
| D-Bus con zbus, latencia IPC, memfd | Sí | Medición final | No | ADR-0007 |
| Btrfs + snapshots + systemd-boot + boot counting | Sí (VM con UEFI/OVMF) | Validación EFI real | No | ADR-0009, `storage.md` |
| Presupuestos de memoria / CPU / frame | No representativo | **Obligatorio** | No | `benchmarks/metodologia.md` |
| Snapshot de paquetes Arch | Sí si VM x86_64 | Sí | No | **P-02** |

### D.2 Herramientas (solo a definir, no instalar)
- Virtualización para la VM (pendiente de ADR-0013; QEMU/UTM con OVMF para UEFI).
- Arch Linux con `base-devel`, Mesa, libinput, libseat, Wayland, `wayland-protocols`, `pkgconf`, `btrfs-progs`, `systemd-boot`.
- Rust: `rustup` o paquete Arch (decidir en ADR-0001), `rustfmt`, `clippy`, `cargo-deny`, `cargo-audit`, opcionalmente `cargo-vet` (P-04), Criterion, Miri.
- Clientes de prueba: `foot`, `weston-*` demos, Firefox Wayland.
- Cada herramienta que entre en el build debe registrarse en `security/dependencias.md` (regla 5 de AGENTS.md).

### D.3 Pruebas mínimas de Fase 1
1. `cargo build --workspace --offline --locked` reproducible en dos ejecuciones limpias.
2. `cargo fmt --check`, `cargo clippy -D warnings`, `cargo deny check`, `cargo audit` en CI local.
3. Unitarias y de propiedades de las cuatro bibliotecas iniciales.
4. Cada spike con hipótesis, implementación mínima, resultados y conclusión (roadmap §1.4).
5. Línea base de benchmarks según `metodologia.md`, con hardware identificado.

---

## E. Bloqueos (prioridad descendente)

1. **ADR-0008 — licencia del proyecto** (propietario). Bloquea cualquier selección de dependencias (Smithay, zbus, toolkit) y la redacción de `deny.toml`.
2. **Entorno de referencia: aarch64 vs x86_64** (P-02, ADR-0013, ADR-0002; propietario + arquitecto). Bloquea CI, la VM y el snapshot de paquetes.
3. **Hardware HW-A identificado** (propietario). Sin ficha, ningún spike DRM/KMS ni benchmark es válido (roadmap §4).
4. **`fase0-revision.md` duplicado** (P-01). El criterio de cierre "ADR críticos aceptados" no tiene documento de soporte.
5. **Lista de ADR críticos para Gate 0** (P-07, P-08). Sin ella no se puede declarar qué falta aceptar.
6. **Estructura del workspace y nombres** (P-09, P-10, P-11). Debe decidirse antes de crear el workspace en Fase 1.
7. **ADR-0001 aceptado** (propietario). Formalidad, pero previa al toolchain.
8. **Contratos D-Bus y `storage.md`** (P-06). Pueden completarse al inicio de Fase 1 si se declaran como excepción documentada.

Riesgo nuevo a registrar en `architecture/riesgos.md` (no se ha editado; pendiente de autorización): *"Entorno de desarrollo aarch64 incompatible con snapshot ALA x86_64 de la distribución"* — Prob. Cierta, Impacto Alto, dueño Arquitectura/Linux.

---

## F. Plan de trabajo

| # | Tarea | Responsable | Depende de |
|---|---|---|---|
| 1 | Decidir licencia (ADR-0008) | Propietario | — |
| 2 | Decidir entorno de referencia e identificar HW-A (P-02) | Propietario + arquitecto | — |
| 3 | Reescribir ADR-0013 en formato completo con la decisión de 2 | arquitecto-sistemas | 2 |
| 4 | Sustituir `fase0-revision.md` por revisión real y definir ADR críticos de Gate 0 (P-01, P-07, P-08) | arquitecto-sistemas | — |
| 5 | Alinear `security/dependencias.md` con ADR-0008 (P-04) | ingeniero-seguridad | 1 |
| 6 | Unificar estructura del workspace y nombres de binarios/bus (P-09, P-10, P-11) | arquitecto + ingeniero-rust | — |
| 7 | Mover reglas normativas de ADR-0014/0015 a `docs/` (P-12) y ampliar `modelo-seguridad.md` (P-14) | ingeniero-seguridad, ingeniero-recursos | — |
| 8 | Crear `architecture/storage.md`, `specifications/configuration.md` y XML D-Bus de Session1/Settings1, o declararlos excepción de Fase 1 (P-06) | ingeniero-linux, arquitecto | 6 |
| 9 | Completar `README.md` (P-05) y erratas menores (P-15, P-16, P-17) | cualquiera | 4 |
| 10 | Re-evaluar checklist de cierre con evidencia; registrar riesgo de §E | arquitecto + qa-sistemas | 1–9 |
| 11 | Revisión del propietario y decisión de Gate 0 | Propietario | 10 |

Las tareas 1, 2, 4, 6 y 7 son independientes y pueden avanzar en paralelo.

---

## G. Recomendación del Arquitecto

**El proyecto no está preparado para cerrar Fase 0.**

La documentación es extensa y en general coherente en su intención (capas, IPC híbrido, recuperación, sistema de diseño), pero:

- El documento que debería resumir la revisión de Fase 0 es una copia de ADR-0009 (P-01).
- Ningún ADR está aceptado y no hay lista de cuáles son críticos para el Gate 0.
- Dos decisiones del propietario bloquean la Fase 1: **licencia (ADR-0008)** y **entorno/arquitectura de referencia (ADR-0013 vs ADR-0002)**.
- Las 14 verificaciones transversales de `criterios-aceptacion.md` están sin marcar, y varias carecen de documento de soporte (contratos D-Bus versionados, fronteras de confianza, matriz de compatibilidad).

Condiciones mínimas para cerrar Fase 0:
1. ADR-0001, 0002, 0007, 0008 y 0013 aceptados (o con excepción documentada); 0003, 0005 y 0009 aceptados como **provisionales** con spike/benchmark asignado a Fase 1, como permite `criterios-aceptacion.md` §Cierre formal.
2. Problemas P-01 a P-07 resueltos.
3. Verificaciones transversales marcadas con evidencia o con excepción explícita.
4. Aprobación explícita del propietario registrada según `roadmap.md §7`.

Fase 0 sigue en estado `IN_PROGRESS`. No se avanza a Fase 1.
