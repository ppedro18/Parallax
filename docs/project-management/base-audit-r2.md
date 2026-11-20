# Auditoría de la base del proyecto — Tarea 001, revisión 2

> **Revisión 2 (histórica, no se modifica).** La revisión vigente es [`base-audit-r3.md`](base-audit-r3.md).

- **Fecha:** 2026-11-19
- **Base:** revisión 1 ([`base-audit.md`](base-audit.md), conservada sin cambios de contenido) sobre el commit `0da11d4`, más los cambios sin commitear de ADR-0008 §14 y `security/dependencias.md`.
- **Fase:** 0 — `IN_PROGRESS`. Ningún ADR cambia de estado en esta revisión.

## Historial de revisiones

| Rev. | Fecha | Cambios |
|---|---|---|
| 1 | 2026-11-19 | Auditoría inicial: inventario, P-01…P-18, matriz de ADR, Fase 1, bloqueos, plan. |
| 2 | 2026-11-19 | P-04 resuelto (corrección documental); separada la aceptación formal de ADR-0008; nueva regla de prevalencia por estado de ADR (P-19, P-20); comparación técnica para P-02 (§H); revisión de la matriz de hardware (P-21, P-22). |

Las secciones no citadas aquí siguen vigentes tal como están en la revisión 1.

---

## A. Estado del repositorio (cambios respecto a rev. 1)

### A.2 Inventario — filas modificadas

| Carpeta | Archivos | Estado |
|---|---|---|
| `docs/security/` | `modelo-seguridad.md` (37 líneas), `dependencias.md` | `dependencias.md` alineado con ADR-0008 §14 (ver P-04). `modelo-seguridad.md` sigue siendo mínimo (P-14). |
| `docs/decisions/` | Plantilla + ADR-0001…0015 | Todos `Propuesto`. ADR-0008 contiene ya la decisión redactada por el propietario (§14), **sin aceptar**. |
| `docs/project-management/` | `base-audit.md` (rev. 1), `base-audit-r2.md` (rev. 2) | — |

### A.1 Git
Cambios sin commitear: `ADR-0008-licencia-y-dependencias.md`, `security/dependencias.md`, `project-management/`. `.serena/` sigue sin versionar.

---

## B. Problemas

### B.1 Estado de los problemas de la revisión 1

| ID | Sev. | Estado rev. 2 | Evidencia / nota |
|---|---|---|---|
| P-01 | C | Abierto | `fase0-revision.md` sigue idéntico a ADR-0009. |
| P-02 | C | Abierto — análisis preparado | Ver §H. Se añade evidencia: `benchmarks/metodologia.md §2` define la VM de CI como **QEMU/KVM con virtio-gpu/virgl**, lo que presupone un anfitrión Linux con KVM; contradice también a ADR-0013 (VM en macOS). |
| P-03 | A | Abierto | ADR-0013 sin cambios. |
| P-04 | A | **Resuelto 2026-11-19** (corrección documental) | `security/dependencias.md` reescrito: regla 2 recoge las licencias de ADR-0008 §14.1–14.3; regla 3 sustituye "GPL solo para procesos separados / LGPL solo enlace dinámico" por revisión específica de GPL, LGPL, AGPL, MPL y propietarias (§14.5); regla 1 y tabla incluyen los campos de §14.5; assets remitidos a `design-system/assets/manifest.toml` (§14.4); regla 9 recoge avisos y SBOM CycloneDX (§14.6); regla 10 "bit a bit" pasa a "cuando sea viable" (roadmap §6.1); `cargo vet` pasa a regla 11, no vinculante, pendiente de decisión. **No implica la aceptación de ADR-0008**, que sigue `Propuesto`. Queda un efecto colateral: ver P-19. |
| P-05…P-18 | — | Abiertos | Sin cambios. |

### B.2 Problemas nuevos

| ID | Sev. | Archivo / sección | Problema | Impacto | Corrección propuesta |
|---|---|---|---|---|---|
| P-19 | M | `security/dependencias.md` l.3 | La nota introducida al alinear P-04 dice "En caso de conflicto prevalece el ADR" sin distinguir el estado del ADR. ADR-0008 está `Propuesto`, de modo que, según la regla de §B.3, no debería prevalecer como decisión definitiva. | Puede leerse como que la política de licencias ya es normativa. | Reformular: "Refleja la decisión propuesta en ADR-0008 §14. Será normativa cuando ADR-0008 pase a `Aceptado`; hasta entonces es la política provisional de trabajo." No se modifica en esta tarea (restricción del propietario). |
| P-20 | B | `ADR-0000-plantilla.md` l.2 | La plantilla define los estados `Propuesto \| Aceptado \| Rechazado \| Sustituido por ADR-XXXX`. No existe `Obsoleto`. Además los ADR usan variantes libres ("Propuesto — requiere benchmark", "Propuesto — decisión redactada…"). | Estado no normalizado → la regla de prevalencia no puede aplicarse de forma mecánica. | Fijar el conjunto cerrado de estados en la plantilla (añadir `Obsoleto` si se desea) y mover los matices a un campo aparte (p. ej. "Condición de aceptación"). |
| P-21 | A | `benchmarks/metodologia.md §2`, `design-system/RENDERING_PERFORMANCE.md §2.1`, `roadmap.md §4` | HW-A y HW-B **no son equipos reales identificados**: son perfiles genéricos ("portátil Intel o AMD", "iGPU Intel o AMD", "GPU AMD dedicada"). Faltan modelo, CPU concreta, GPU concreta, kernel, firmware/BIOS, distribución y stack gráfico (Mesa, driver). El propio roadmap §4 exige la ficha y dice que no se valida sobre hardware no identificado. | Ningún presupuesto, spike DRM/KMS ni benchmark puede declararse válido. Bloquea los criterios de Fase 1 y 2. | El propietario identifica los equipos físicos disponibles; la ficha se rellena con datos leídos del equipo, no supuestos. Ver §I. |
| P-22 | M | `roadmap.md §4` vs `benchmarks/metodologia.md §2` | El roadmap describe HW-B como "compatibilidad alternativa de CPU/GPU" y HW-C como "multi-monitor"; `metodologia.md` describe HW-B con "dos monitores" y GPU AMD dedicada, solapando HW-C. HW-C y HW-D no tienen perfil en `metodologia.md`. | Ambigüedad sobre qué equipo cubre multi-monitor. | Fijar el propósito de cada perfil en un único documento y referenciarlo desde los demás. |

### B.3 Regla de prevalencia de ADR (verificación)

Regla propuesta por el propietario:

| Estado | Efecto |
|---|---|
| `Aceptado` | Decisión normativa vigente. |
| `Propuesto` | Decisión pendiente; orienta el trabajo de Fase 0 pero no prevalece como decisión definitiva. |
| `Rechazado` / `Obsoleto` (o `Sustituido por`) | No constituye decisión vigente. |

Comprobación contra las convenciones existentes:

- **`AGENTS.md` §Estado del proyecto:** "No se escribe código de producción hasta que … los ADR en estado `Propuesto` pasen a `Aceptado`". **Coherente**: un ADR `Propuesto` no autoriza implementación.
- **`AGENTS.md` §Cambios arquitectónicos:** cambios → ADR "aprobado por arquitecto-sistemas antes de implementar". **Coherente**, con un matiz: AGENTS.md exige la aprobación del arquitecto y el roadmap §2/§7 exige la del propietario. La regla debería indicar que `Aceptado` requiere ambas (AGENTS.md §Estado ya lo dice así).
- **`AGENTS.md` línea 3:** "Prevalecen sobre las instrucciones de cada agente". **Coherente**, pero choca con P-12: ADR-0014 y ADR-0015 delegan reglas normativas en `.claude/agents/*.md`, que por esta regla tienen menor rango que AGENTS.md y que los propios ADR.
- **`roadmap.md §6`:** "Una decisión arquitectónica aceptada no se modificará silenciosamente". **Coherente** (habla de aceptadas).
- **`criterios-aceptacion.md` §Cierre formal:** permite ADR "con excepción documentada" o "provisionales" hasta completar spikes. **Hueco**: la regla no define el efecto de un ADR `Propuesto` con excepción documentada. Propuesta: tratarlo como `Propuesto`, con la excepción como autorización acotada y explícita del propietario.
- **`ADR-0000-plantilla.md`:** no contempla `Obsoleto` (P-20).
- **`security/dependencias.md` l.3:** contradice la regla (P-19).

Conclusión: la regla **no contradice** las convenciones generales; requiere corregir P-19 y P-20 y precisar el caso de las excepciones documentadas.

---

## C. Matriz de ADR — filas modificadas

| ADR | Estado | Dependencias | Evidencia pendiente | Recomendación |
|---|---|---|---|---|
| 0008 Licencias | Propuesto (decisión redactada en §14) | — | Aprobación expresa del propietario. DCO vs CLA (§9) sigue sin decidir. | **Bloqueante**: aceptación formal por el propietario. La alineación documental ya no es requisito (P-04 resuelto). |
| 0013 Entorno desarrollo | Propuesto | 0002 | Formato completo (P-03); resolución P-02 (§H); fichas HW-A/HW-B (P-21) | **Bloqueante** para Fase 1 |

En C.1, ADR-0008, los puntos 5–7 quedan así:
5. **Riesgos abiertos:** R12 (se reduce cuando se acepte); DCO vs CLA sin decidir; licencia de `px-ui` (envuelve el toolkit) sin decidir.
6. **Evidencia que falta:** aprobación expresa del propietario.
7. **Condiciones:** el propietario acepta §14 y decide DCO/CLA (o lo difiere con excepción documentada).

---

## E. Bloqueos (revisados)

1. **Aceptación formal de ADR-0008** (propietario). La decisión está redactada; falta la autorización expresa. Hasta entonces la política de licencias es provisional.
   - *Separado:* la corrección documental de `dependencias.md` (P-04) está hecha; solo queda P-19 (redacción de la nota de prevalencia).
2. **Entorno de referencia y arquitectura de CI** (P-02; propietario + arquitecto). Análisis en §H.
3. **Identificación de HW-A y HW-B como equipos reales** (P-21; propietario).
4. **`fase0-revision.md` duplicado** (P-01).
5. **Lista de ADR críticos para Gate 0** (P-07, P-08) y **estados normalizados** (P-20).
6. **Estructura del workspace y nombres** (P-09, P-10, P-11).
7. **ADR-0001 aceptado** (propietario).
8. **Contratos D-Bus y `storage.md`** (P-06).

Riesgo propuesto en rev. 1 (sigue sin registrar en `riesgos.md`, pendiente de autorización): entorno aarch64 incompatible con snapshot ALA x86_64.

---

## F. Plan de trabajo (revisado)

| # | Tarea | Responsable | Depende de | Estado |
|---|---|---|---|---|
| 1 | Aceptar formalmente ADR-0008 | Propietario | — | Pendiente (decisión redactada) |
| 2 | Decidir entorno de referencia e identificar HW-A/HW-B (P-02, P-21) | Propietario + arquitecto | — | Pendiente; análisis en §H/§I |
| 3 | Reescribir ADR-0013 con la decisión de 2 | arquitecto-sistemas | 2 | Pendiente |
| 4 | Sustituir `fase0-revision.md`, definir ADR críticos de Gate 0 (P-01, P-07, P-08) | arquitecto-sistemas | — | Pendiente |
| 5 | Alinear `security/dependencias.md` con ADR-0008 (P-04) | ingeniero-seguridad | — | **Completada 2026-11-19** (queda P-19) |
| 5b | Corregir la nota de prevalencia (P-19) y normalizar estados en la plantilla (P-20) | arquitecto-sistemas | — | Pendiente de autorización |
| 6 | Unificar workspace y nombres (P-09, P-10, P-11) | arquitecto + ingeniero-rust | — | Pendiente |
| 7 | Reglas de ADR-0014/0015 a `docs/` (P-12); ampliar `modelo-seguridad.md` (P-14) | ingeniero-seguridad, ingeniero-recursos | — | Pendiente |
| 8 | `storage.md`, `configuration.md`, XML D-Bus o excepción de Fase 1 (P-06) | ingeniero-linux, arquitecto | 6 | Pendiente |
| 9 | `README.md` y erratas (P-05, P-15, P-16, P-17); unificar perfiles HW (P-22) | cualquiera | 4 | Pendiente |
| 10 | Re-evaluar checklist de cierre con evidencia; registrar riesgo | arquitecto + qa-sistemas | 1–9 | Pendiente |
| 11 | Revisión del propietario y decisión de Gate 0 | Propietario | 10 | Pendiente |

---

## G. Recomendación del Arquitecto (revisada)

Sin cambios en el veredicto: **Fase 0 no está preparada para cerrarse** y continúa en `IN_PROGRESS`.

Progreso desde rev. 1: la política de licencias está redactada y la documentación de dependencias es coherente con ella (P-04). Siguen bloqueando: aceptación formal de ADR-0008, P-02 (entorno), P-21 (hardware real sin identificar) y P-01.

---

## H. Preparación de la resolución de P-02 (sin modificar ADR-0002 ni ADR-0013)

### H.1 Hechos de partida
- **Arquitectura de release v1:** x86_64 (ADR-0013 §Contexto; `metodologia.md §2`, HW-A y HW-B x86_64).
- **Arch Linux Archive (ALA)** publica snapshots fechados de los repositorios oficiales de Arch, que son x86_64. Arch Linux ARM es un proyecto independiente con otros mantenedores, otro llavero y sin un archivo fechado oficial equivalente a ALA. *Dato de conocimiento general del ecosistema, a confirmar por ingeniero-linux antes de citarlo en un ADR.*
- **Anfitrión actual:** macOS en Apple Silicon (arm64) según ADR-0013. No se ha verificado en esta tarea.
- **Requisitos que fijan los ADR:** snapshot inmutable (ADR-0002), builds `--offline --locked` y reproducibles (ADR-0001, `dependencias.md`), DRM/KMS en hardware real (roadmap §1.1 y principio 3).

### H.2 Comparación

| Criterio | A. VM x86_64 en el Mac (emulada) | B. VM aarch64 en el Mac (nativa) | C. Host Linux x86_64 + CI independiente x86_64 |
|---|---|---|---|
| Compatibilidad con ALA | Total: mismos paquetes y firmas que la release | **No**: requiere Arch Linux ARM, otro origen, sin snapshot oficial; habría que espejar y fechar por cuenta propia | Total |
| Reproducibilidad respecto a la release | Alta (misma arquitectura, mismo snapshot) | Baja: binarios, `Cargo.lock` igual pero dependencias nativas (Mesa, libinput, libseat) de otro origen y versión | Alta |
| Dependencias nativas | Las de la release | Divergentes; posibles diferencias ABI y de versión | Las de la release |
| Virtualización | Emulación TCG sin aceleración en Apple Silicon: lenta | Hypervisor nativo: rápida | KVM nativo en Linux; la VM de CI que ya describe `metodologia.md` (QEMU/KVM + virtio-gpu/virgl) encaja aquí |
| Compilación Rust | Muy lenta bajo emulación (orden de varias veces más lenta; no medido) | Rápida | Rápida |
| Rendimiento de CI | Inviable como CI principal | Bueno, pero valida otra arquitectura | Bueno y representativo de la arquitectura |
| Pruebas DRM/KMS reales | No (virtio-gpu emulado) | No (virtio-gpu) | Sí en el host físico, si el host es HW-A o similar |
| Coste / requisitos | Ninguno adicional; tiempo | Ninguno adicional | Requiere un equipo Linux x86_64 físico (lo exige igualmente la validación en HW-A) |
| Riesgo principal | Tiempos de ciclo inaceptables | Falsos verdes: CI pasa en una arquitectura que no se publica | Disponibilidad del equipo; coordinar host de desarrollo y CI |

### H.3 Arquitectura de release vs arquitectura de CI
- La CI que **decide** (gate de merge, builds reproducibles, SBOM, benchmarks) debe ejecutarse en la arquitectura de release (x86_64) y contra el mismo snapshot ALA. Si no, la reproducibilidad de ADR-0002 y ADR-0008 §14.6 no se puede afirmar.
- Una CI **auxiliar** aarch64 es útil para portabilidad (ADR-0013 pide código sin asunciones x86) y para crates puros sin dependencias nativas, pero no sustituye a la anterior.
- El macOS anfitrión sirve como entorno de edición y para `cargo check`/tests de crates puros; no como referencia (rev. 1 §D.1).

### H.4 Recomendación (la decisión corresponde al propietario)
**Opción C**: equipo Linux x86_64 físico como host de referencia y CI x86_64 independiente (VM QEMU/KVM sobre ese host, como ya describe `metodologia.md`), contra el snapshot ALA de la release. Complementos opcionales: VM aarch64 en el Mac para ciclo rápido de edición y CI de portabilidad marcada como **no bloqueante**.

Fundamento: es la única opción en la que CI, snapshot y release coinciden en arquitectura y origen de paquetes, y el equipo físico es necesario de todos modos para DRM/KMS (roadmap §1.1, P-21). La opción A no es viable por rendimiento; la B rompe la compatibilidad con ALA.

Condiciones para que el propietario decida:
1. Confirmar si existe un equipo Linux x86_64 disponible (enlaza con P-21).
2. Confirmar que v1 es solo x86_64.
3. Decidir si aarch64 se mantiene como CI auxiliar no bloqueante.
4. Tras la decisión: reescribir ADR-0013, añadir la arquitectura de destino a ADR-0002 y registrar el riesgo en `riesgos.md`.

---

## I. Revisión de la matriz de hardware

Fuentes: `roadmap.md §4`, `benchmarks/metodologia.md §2`, `design-system/RENDERING_PERFORMANCE.md §2.1`. No se han inventado ni completado datos.

| Campo requerido | HW-A | HW-B |
|---|---|---|
| Equipo / modelo | **No identificado** ("portátil Intel o AMD") | **No identificado** |
| CPU | **Falta** (solo "Intel o AMD") | **Falta** |
| GPU | **Falta** (solo "iGPU Intel o AMD") | **Falta** (solo "AMD dedicada") |
| Arquitectura | x86_64 | x86_64 |
| RAM | 8 GB | 16 GB |
| Almacenamiento | NVMe | SSD/NVMe |
| Pantalla | 1920×1080 a 60 Hz; alt. 2560×1600 ×1,5 | Dos monitores, sin modelo ni resolución |
| Kernel | **Falta** | **Falta** |
| Firmware / BIOS | **Falta** | **Falta** |
| Distribución | **Falta** ("Linux nativo") | **Falta** |
| Stack gráfico (Mesa, driver, backend) | Parcial: "DRM/KMS, GPU acelerada"; sin Mesa ni driver | **Falta** |
| Dispositivos de entrada | **Falta** | **Falta** |

Veredicto: **HW-A y HW-B son perfiles objetivo, no equipos reales** (P-21). HW-C y HW-D solo tienen propósito en el roadmap (P-22). La ficha debe completarse a partir del equipo físico (p. ej. salida de `lscpu`, `lspci -k`, `uname -r`, `dmidecode`, `glxinfo`/`vulkaninfo`), ejecutada en ese equipo y con autorización del propietario.
