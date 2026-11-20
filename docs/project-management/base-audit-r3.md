# Auditoría de la base del proyecto — Tarea 001, revisión 3

- **Fecha:** 2026-11-19
- **Base:** revisiones 1 ([`base-audit.md`](base-audit.md)) y 2 ([`base-audit-r2.md`](base-audit-r2.md)), conservadas; a cada una solo se le añadió una línea que apunta a la revisión siguiente.
- **Repositorio:** commit `0da11d4` más cambios sin commitear (ADR-0008 §14, `security/dependencias.md`, `docs/project-management/`).
- **Fase:** 0 — `IN_PROGRESS`. Ningún ADR cambia de estado. No se inicia Fase 1.

## Historial de revisiones

| Rev. | Cambios |
|---|---|
| 1 | Auditoría inicial (P-01…P-18). |
| 2 | P-04 resuelto; regla de prevalencia; P-19…P-22; análisis P-02 (§H) y hardware (§I). |
| 3 | Corrige el análisis de P-02 (VM de CI vs runner físico; host Linux x86_64 disponible; fuentes oficiales); separa equipos reales, disponibles y perfiles (P-21, P-22, `DEV-01`); corrige la atribución de ADR-0008 §14 y separa DCO/CLA y `px-ui`; propuesta de normalización de estados (P-20); delimita Gate 0. |

Lo no citado aquí sigue vigente según la revisión 2.

---

## 1. Hallazgos corregidos respecto a la revisión 2

| Ref. rev. 2 | Corrección |
|---|---|
| §H.2 tabla | Mezclaba "CI en VM" y "CI en equipo físico" en una única opción C. Ahora se separan (ver §3). |
| §H.1 "Anfitrión actual: macOS" | Presuponía que el Mac es el único equipo de desarrollo. El propietario indica que existe un equipo Linux x86_64 disponible. Se reconoce como **disponible pero no identificado** en el repositorio (ver §4). |
| §H.1 ALA / Arch Linux ARM | Estaba marcado "a confirmar". Ahora verificado con fuentes oficiales (§2). |
| §C fila ADR-0008, §E.1 | La rev. 2 decía "decisión redactada por el propietario". **No hay evidencia expresa de autoría ni de aceptación**: el texto llegó pegado en la conversación de trabajo y se incorporó a ADR-0008 §14 con estado `Propuesto`. Se corrige a "decisión redactada en §14, pendiente de aprobación". |
| §C.1 ADR-0008 riesgos | Unía licencia, DCO/CLA y `px-ui`. Se separan (§5). |
| §I | Trataba HW-A/HW-B como equipos a completar. Se aclara que son **perfiles objetivo**; un equipo real se registra aparte y luego se asigna a un perfil (§4). |
| §E.3 bloqueo de hardware | Bloqueaba todo. Se delimita qué pruebas requieren hardware real y cuáles avanzan en VM (§4.4). |
| §B.3 regla de prevalencia | Mantenía `Obsoleto` como posible estado. Se sustituye por `Sustituido`, ya presente en la plantilla (§6). |

---

## 2. Evidencias y fuentes (P-02)

| Afirmación | Evidencia | Estado |
|---|---|---|
| ALA publica snapshots fechados de repositorios de Arch | `https://archive.archlinux.org/repos/` lista directorios por año desde `2013/`; `…/repos/last/core/os/x86_64/core.db` responde HTTP 200 (consulta de solo lectura, 2026-11-19). | Verificado |
| ALA solo contiene x86_64 | `https://archive.archlinux.org/repos/2025/01/01/core/os/` lista únicamente `x86_64/`. | Verificado para la fecha muestreada |
| Arch Linux ARM es un port separado | `https://archlinuxarm.org/` se describe como "a port of Arch Linux" y habla de "collaboration with Arch Linux". | Verificado |
| Arch Linux ARM usa llavero propio | `https://archlinuxarm.org/about/package-signing`: paquete `archlinuxarm-keyring`, `pacman-key --populate archlinuxarm`, claves maestras de sus mantenedores, y declara que las bases de datos del repositorio no están firmadas. | Verificado |
| Arch Linux ARM no ofrece un archivo fechado equivalente a ALA | Ni la portada ni la página de firmas lo mencionan. | **No verificado** (ausencia de mención; no prueba inexistencia) |
| Arch Linux no tiene port oficial aarch64 | Las páginas de la Arch Wiki consultadas devolvieron bloqueo anti-bots. | **No verificado** |

Impacto para ADR-0002/0013: un entorno aarch64 consumiría paquetes de otro origen, con otro llavero y con bases de datos sin firmar, frente al requisito de ADR-0002 §3.3 ("firmar las bases de datos de los repositorios propios" y verificar procedencia). Esto refuerza P-02 sin depender de los dos puntos no verificados.

---

## 3. P-02 corregido: entorno de desarrollo y CI

### 3.1 Definiciones
- **Host de desarrollo:** equipo donde se edita, compila y ejecuta el compositor anidado. No es entorno de aceptación.
- **VM de CI aislada:** máquina virtual efímera (QEMU/KVM) que reconstruye desde cero, offline, contra el snapshot. Ofrece aislamiento y repetibilidad; **no** ofrece DRM/KMS real.
- **Runner físico independiente:** equipo físico dedicado a ejecutar pruebas de hardware (DRM/KMS, GPU, energía, multi-monitor). Su instalación es controlada y no se usa para desarrollo diario.
- **Equipo de aceptación:** equipo real asignado a un perfil HW-A…HW-D (§4).

### 3.2 Opciones

| Criterio | A. VM x86_64 emulada en el Mac | B. VM aarch64 en el Mac | C1. Host Linux x86_64 + VM de CI KVM en el mismo host | C2. Host Linux x86_64 + runner físico independiente |
|---|---|---|---|---|
| Compatibilidad ALA | Sí | No (§2) | Sí | Sí |
| Reproducibilidad frente a release | Alta | Baja | Alta (VM efímera desde snapshot) | Alta en la VM; el runner valida hardware, no reproducibilidad |
| Aislamiento respecto al host de desarrollo | Sí | Sí | Sí, si la CI corre solo en VM efímera | Sí |
| Compilación / tiempo de CI | Inviable (emulación TCG) | Rápido, arquitectura errónea | Rápido (KVM) | Rápido |
| DRM/KMS real | No | No | Solo si el propio host es equipo de aceptación, fuera de la VM | Sí |
| Equipos necesarios | Mac | Mac | 1 Linux x86_64 | 2 (host + runner) |
| Riesgo | Ciclos de horas | Falsos verdes | Host de desarrollo "contamina" resultados de aceptación si se usa a la vez como HW-A | Coste y disponibilidad de un segundo equipo |

### 3.3 ¿Puede el equipo Linux x86_64 disponible ser host de desarrollo y CI?
Sí, con condiciones:
1. La CI corre **solo dentro de VM efímeras** sobre KVM, nunca directamente en el sistema del host (AGENTS.md regla 4: "Todo build/test ocurre en el workspace, en VM o contenedor").
2. Disponer de KVM, espacio para el espejo del snapshot ALA y el `vendor/`: **no verificado** porque el equipo no está identificado (§4).
3. Si además se quiere usar como equipo de aceptación (HW-A), las mediciones se hacen en una instalación dedicada y documentada (otro arranque o partición), no en el sistema de desarrollo diario. Si no es posible, HW-A debe ser otro equipo.
4. El Mac queda como equipo complementario (edición, crates puros, opcionalmente VM aarch64 de portabilidad no bloqueante).

### 3.4 Recomendación técnica (decisión del propietario)
- **Arquitectura de release v1: x86_64.**
- **Fase 1: opción C1** sobre el equipo Linux x86_64 disponible: CI en VM KVM efímera contra snapshot ALA. Es suficiente para los entregables de Fase 1 que no son de hardware (§4.4).
- **Antes de Gate 1 para los spikes DRM/KMS y benchmarks, y obligatoriamente antes de Gate 2: C2**, o bien registrar el mismo equipo como HW-A con instalación de aceptación separada (condición 3).
- aarch64: CI auxiliar no bloqueante, opcional.

---

## 4. Hardware: equipos reales, disponibles y perfiles (P-21, P-22)

### 4.1 Tres categorías
| Categoría | Qué es | Ejemplo |
|---|---|---|
| Perfil objetivo | Clase de hardware que fija presupuestos | HW-A…HW-D |
| Equipo disponible | Existe y el proyecto puede usarlo, aún sin ficha | Equipo Linux x86_64 mencionado por el propietario; Mac Apple Silicon |
| Equipo real identificado | Tiene ficha completa leída del propio equipo | Ninguno todavía |

Un equipo identificado puede asignarse a un perfil; un perfil nunca se "completa" con datos supuestos.

### 4.2 Registro propuesto `DEV-01`
Recomendado registrar el equipo Linux como `DEV-01` (host de desarrollo), **separado** de los identificadores HW-*.

| Campo | Valor |
|---|---|
| Rol | Host de desarrollo y anfitrión de VM de CI |
| Arquitectura | x86_64 (según el propietario; sin ficha) |
| Modelo, CPU, GPU, RAM, almacenamiento, kernel, firmware, distribución, stack gráfico, KVM | **Pendiente** — obtener del propio equipo con autorización |
| ¿Candidato a HW-A? | Pendiente de decisión del propietario y de que encaje en el perfil HW-A |

No se registra ningún valor no leído del equipo.

### 4.3 Fuente única de verdad
Hoy los perfiles aparecen en tres sitios con contenido distinto: `roadmap.md §4` (propósito de HW-A…D), `benchmarks/metodologia.md §2` (HW-A, HW-B, VM de CI) y `design-system/RENDERING_PERFORMANCE.md §2.1` (HW-A).

Recomendación: crear **`docs/benchmarks/hardware-referencia.md`** como única fuente para perfiles HW-A…HW-D, equipos `DEV-*` y fichas de equipos reales; los otros tres documentos solo lo referencian. Alternativa: usar `metodologia.md §2` como fuente. Se prefiere un archivo propio porque lo consumen roadmap, diseño, QA y rendimiento, no solo benchmarks. (No se crea en esta tarea.)

P-22 (solape HW-B/HW-C) se resuelve en ese documento: HW-B = CPU/GPU alternativa; HW-C = multi-monitor; quitar "dos monitores" de HW-B o justificar la fusión.

### 4.4 Alcance real del bloqueo de hardware
| Puede avanzar en VM / host de desarrollo | Requiere equipo real identificado |
|---|---|
| Workspace, `Cargo.lock`, `vendor/`, build offline | Spike Smithay con backend DRM/KMS, libinput y logind en sesión real |
| fmt, clippy, cargo-deny, cargo-audit, CI | Presupuestos de memoria, CPU y frame time (roadmap §4: no se valida sobre hardware no identificado) |
| Unitarias, propiedades y fuzzing de `px-types`, `px-ipc`, `px-config`, `px-wm` | Benchmark de rendimiento del toolkit (ADR-0005) |
| Spike Smithay con backend winit/anidado; layer-shell con clientes de prueba | Multi-monitor, hotplug, escalado real (HW-C) |
| zbus, latencia IPC funcional, memfd | Energía y suspensión (HW-D) |
| Btrfs + snapshots + systemd-boot + boot counting en VM UEFI | Validación EFI en firmware real |
| AccessKit/AT-SPI funcional | — |

Conclusión: P-21 **no bloquea el cierre de Fase 0 ni el inicio de Fase 1**; bloquea los criterios de Gate 1 relativos a toolkit y benchmarks, y Gate 2 completo. Se reclasifica de bloqueo de Fase 0 a bloqueo de Gate 1.

---

## 5. ADR-0008 revisado

- **P-04:** sigue resuelto (rev. 2 §B.1).
- **Autoría y estado:** §14 contiene una decisión **redactada**, sin aprobación registrada. No hay evidencia expresa de que el propietario sea su autor ni de que la haya aceptado. Estado `Propuesto`, sin cambios.
- **Tres decisiones separadas:**

| Decisión | Contenido | Dónde | Depende de | Bloquea Gate 0 |
|---|---|---|---|---|
| L1. Política de licencias del proyecto | §14.1–14.7 | ADR-0008 | Aprobación del propietario | **Sí** (condiciona toda dependencia de Fase 1) |
| L2. Modelo de contribuciones (DCO o CLA) | §9 deja la elección abierta | ADR-0008 §9 o ADR propio | Propietario | No mientras no se acepten contribuciones externas; sí antes de abrirlas |
| L3. Licencia definitiva de `px-ui` | §14.2 aplica `MIT OR Apache-2.0` a `px-*` "no sujetos a obligaciones incompatibles" | ADR-0008 + ADR-0005 | ADR-0005 (toolkit elegido) y revisión de compatibilidad de su licencia y la de sus dependencias | No; se fija al aceptar ADR-0005 |

Recomendación: aceptar L1 sin esperar a L2 y L3, dejando explícito en el ADR que `px-ui` queda fuera de la asignación automática hasta ADR-0005.

---

## 6. Propuesta de normalización de estados y excepciones (P-20)

### 6.1 Estados
Conjunto cerrado: **`Propuesto`**, **`Aceptado`**, **`Rechazado`**, **`Sustituido por ADR-NNNN`**.

- Coincide con la plantilla actual salvo el texto libre que los ADR añaden tras "Propuesto".
- No se añade `Obsoleto`: `Sustituido` cubre la retirada por otra decisión, y una decisión retirada sin sustituta puede expresarse con un ADR nuevo que la sustituya. No hay caso en el repositorio que justifique un quinto estado.
- No se añade `Provisional`: "aceptado con condiciones" se expresa con `Aceptado` + campo de condiciones, para no duplicar estados.

### 6.2 Campos de cabecera propuestos
```
- Estado: Propuesto | Aceptado | Rechazado | Sustituido por ADR-NNNN
- Fecha:
- Dueño técnico:
- Aprobación técnica (arquitecto-sistemas): — | fecha
- Aprobación del propietario: — | fecha
- Condiciones de aceptación: (lista verificable)
- Evidencias: (enlaces a spikes, benchmarks, documentos)
- Condiciones pendientes tras aceptación: (si se acepta con condiciones; fase y responsable)
- Excepciones vigentes: (ref. al registro de excepciones)
```
`Aceptado` requiere **ambas** aprobaciones (AGENTS.md §Estado del proyecto y §Cambios arquitectónicos; roadmap §2).

### 6.3 Reglas
1. Solo `Aceptado` es normativo. `Propuesto` orienta, no prevalece (rev. 2 §B.3).
2. Un ADR `Aceptado` con condiciones pendientes, o una excepción de cierre de fase, **no autoriza código de producción** por sí mismo. La autorización de producción la da el gate de la fase correspondiente (AGENTS.md §Estado; reglas 3 y 5).
3. **Spikes:** código desechable fuera del producto. Cada spike requiere autorización explícita, hipótesis, alcance (qué crates, qué dependencias, en qué entorno), criterios de éxito/fracaso, fecha límite y destino del código (se archiva, no se promueve). Las dependencias que use se registran igualmente (regla 5 de AGENTS.md). Un spike satisfactorio aporta evidencia a un ADR; no lo acepta.
4. **Excepción documentada:** registro con ADR afectado, motivo, alcance exacto (qué permite y qué no), responsable, fecha de caducidad o condición de cierre, y aprobación del propietario.

---

## 7. Delimitación de Gate 0

| Grupo | ADR | Justificación |
|---|---|---|
| **G0-A. Necesarios para cerrar Fase 0** (aceptados) | 0001 Rust | Base de todo el toolchain. |
| | 0002 Base Arch | Fija origen de paquetes y arquitectura; depende de P-02. |
| | 0007 IPC (principio) | Contrato entre todas las capas; los benchmarks quedan como condiciones. |
| | 0008 Licencias (L1) | Condiciona cualquier dependencia de Fase 1. |
| | 0013 Entorno | Sin él no hay entorno autorizado para Fase 1. Requiere reescritura (P-03). |
| | 0015 Autorización Wayland | **Seguridad**: define el modelo de autorización del compositor que los spikes de Smithay ya deben respetar; aceptarlo tarde obligaría a rediseñar. Requiere P-12. |
| **G0-B. Aceptables con condiciones y spikes en Fase 1** | 0003 Smithay | Spike winit + DRM/KMS. |
| | 0004 WM en compositor | Prototipo `px-wm` puro. |
| | 0005 Toolkit UI | Benchmark; criterio de Gate 1 según roadmap. Puede quedar `Propuesto` hasta Gate 1 en lugar de aceptado con condiciones. |
| | 0009 Actualizaciones | Spike Btrfs + systemd-boot en VM UEFI; `storage.md`. |
| **G0-C. Diferibles** (`Propuesto`, sin bloquear) | 0006 Shell un proceso | Se aplica en Fase 3. |
| | 0010 Polkit agent | Fase 3; **seguridad**: diferible solo porque ningún proceso de Fases 1–2 solicita elevación. Revisar antes de Gate 2. |
| | 0011 Terminal | Fase 4. |
| | 0012 Xwayland | Fase 2. |
| | 0014 cgroups | Fase 5; modo observación por defecto. **Seguridad/recursos**: diferible porque no actúa sobre procesos hasta Fase 5. |

### 7.1 Excepciones de cierre
- Única excepción recomendable para Gate 0: **contratos D-Bus XML y `storage.md` (P-06)** se entregan al inicio de Fase 1. Alcance: permite cerrar Fase 0 sin esos documentos; **no** permite implementar servicios D-Bus ni el instalador hasta que existan.
- No se recomiendan excepciones para G0-A.

---

## 8. Hallazgos nuevos

| ID | Sev. | Hallazgo | Propuesta |
|---|---|---|---|
| P-23 | M | ADR-0002 exige firmar bases de datos de repositorios propios; Arch Linux ARM declara no firmar las suyas (§2). Refuerza P-02. | Incluir en la resolución de P-02. |
| P-24 | M | No existe registro de excepciones ni de spikes autorizados. | Crear junto a la normalización de P-20 (p. ej. `docs/project-management/excepciones.md`, `spikes.md`). |
| P-25 | B | La revisión 2 atribuía §14 de ADR-0008 al propietario sin evidencia. | Corregido en esta revisión (§1, §5). |
| P-26 | M | `roadmap.md §1.1` lista "Linux Arch nativo como entorno de referencia" y "VM Arch", sin distinguir host de desarrollo, VM de CI, runner y equipo de aceptación. | Alinear tras decidir P-02. |

---

## 9. Decisiones

### 9.1 Recomendadas (técnicas)
1. Release v1 x86_64; CI en VM KVM efímera sobre el host Linux x86_64 (C1); runner físico o HW-A identificado antes de los spikes de hardware.
2. Registrar `DEV-01` separado de los perfiles HW-*.
3. `docs/benchmarks/hardware-referencia.md` como fuente única de hardware.
4. Estados cerrados (§6.1) y cabecera con campos separados (§6.2).
5. Gate 0 = G0-A aceptados + G0-B con condiciones (o 0005 en `Propuesto`) + excepción única P-06.

### 9.2 Reservadas al propietario
1. Aceptar L1 (ADR-0008 §14).
2. Decidir DCO o CLA (L2), o diferirla hasta abrir contribuciones.
3. Arquitectura de release v1 y modelo de CI (P-02).
4. Identificar `DEV-01` y autorizar la lectura de su ficha; decidir si es candidato a HW-A.
5. Aprobar la lista de Gate 0 y la excepción P-06.
6. Aprobar la normalización de estados (P-20) y la corrección de P-19.
7. Aceptar cada ADR de G0-A (más la aprobación técnica del arquitecto).

---

## 10. Bloqueos reales de Fase 0

1. Aceptación de ADR-0008 L1.
2. Decisión P-02 → reescritura y aceptación de ADR-0013; ADR-0002 con arquitectura de destino.
3. Aceptación de ADR-0001, 0007 y 0015 (este último tras P-12).
4. P-01 (`fase0-revision.md` duplicado) y P-07/P-08 (checklist y lista de ADR del gate).
5. P-09, P-10, P-11 (estructura y nombres), porque condicionan el primer entregable de Fase 1.
6. P-20 normalizado, para poder registrar aprobaciones de forma verificable.

**Ya no son bloqueos de Fase 0:** P-21/P-22 (pasan a Gate 1/2), P-06 (excepción propuesta), L2 y L3 de ADR-0008, ADR-0005 (Gate 1).

---

## 11. Actividades que pueden adelantarse sin vulnerar AGENTS.md

Todas son documentales o de lectura; ninguna crea `Cargo.toml`, crates ni código de producción, instala dependencias ni toca el anfitrión. Requieren la autorización correspondiente para editar documentos.

1. Reescribir ADR-0013 con las opciones de §3 (sin cambiar su estado).
2. Sustituir `fase0-revision.md` por una revisión real (P-01).
3. Redactar `hardware-referencia.md` con perfiles y `DEV-01` vacío de datos no verificados.
4. Proponer la nueva plantilla de ADR (P-20) y los registros de excepciones y spikes (P-24).
5. Mover las reglas normativas de ADR-0014/0015 a `docs/` (P-12).
6. Unificar estructura del workspace y nombres en documentación (P-09–P-11).
7. Redactar las fichas de los spikes de Fase 1 (hipótesis, alcance, criterios), sin ejecutarlos.
8. Con autorización expresa: leer la ficha de `DEV-01` en ese equipo (`lscpu`, `lspci -k`, `uname -r`, etc.); operación de solo lectura.

No se adelanta: crear la VM, instalar herramientas, espejar ALA, crear el workspace ni ejecutar spikes (requieren Gate 0 o autorización de spike explícita).
