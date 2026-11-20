# Registro de riesgos técnicos — PARALLAX

- **Proyecto:** PARALLAX
- **Estado:** Propuesto — registro inicial
- **Fecha:** 2026-11-19
- **Propietario:** Arquitectura del proyecto

## 1. Objetivo

Identificar, evaluar y gestionar los riesgos técnicos que pueden comprometer:

- La viabilidad del proyecto.
- La estabilidad del entorno de escritorio.
- La seguridad del sistema.
- La compatibilidad de hardware y aplicaciones.
- El rendimiento y consumo de recursos.
- La mantenibilidad del código.
- La distribución y actualización del sistema.

El registro se revisará al finalizar cada fase y cuando una decisión arquitectónica modifique significativamente el perfil de riesgo.

## 2. Escalas de evaluación

### Probabilidad

| Nivel | Significado |
|---|---|
| Baja | Poco probable con las condiciones actuales |
| Media | Puede ocurrir durante el desarrollo o uso |
| Alta | Es probable sin mitigaciones |
| Cierta | Es una limitación o condición que necesariamente debe gestionarse |

### Impacto

| Nivel | Significado |
|---|---|
| Bajo | Efecto localizado, sin comprometer el funcionamiento general |
| Medio | Afecta una funcionalidad o componente importante |
| Alto | Compromete estabilidad, seguridad o una fase del proyecto |
| Crítico | Puede impedir la viabilidad, arranque, recuperación o funcionamiento esencial del sistema |

La prioridad se determinará considerando probabilidad, impacto y capacidad de detección. No se asumirá que todos los riesgos de igual impacto tienen la misma prioridad.

---

# 3. Registro de riesgos

| ID | Riesgo | Prob. | Impacto | Mitigación | Dueño |
|---|---|---|---|---|---|
| R01 | Alcance excesivo para el tamaño del equipo | Alta | Crítico | Fases con gates; limitar v1; reutilizar infraestructura Linux; priorizar compositor y Shell antes de las seis aplicaciones | Arquitectura |
| R02 | Inmadurez de toolkits Rust en accesibilidad, IME, texto y controles complejos | Alta | Alto | ADR-0005; benchmark real; abstracción `px-ui`; prototipos funcionales; contribuir upstream cuando sea viable | Framework UI |
| R03 | Caída o corrupción del compositor provoca pérdida de superficies y sesión gráfica | Media | Crítico | `px-wm` puro; pruebas de invariantes y fuzzing; aislamiento de fallos; persistencia de layout; protocolo de recuperación cooperativa | Compositor |
| R04 | Incompatibilidad NVIDIA con Wayland, DRM/GBM o sincronización explícita | Alta | Alto | Soporte secundario en v1; matriz GPU/driver/kernel/Mesa; pruebas en hardware real; fallback cuando sea viable | Hardware |
| R05 | Cambios frecuentes en API de Smithay | Alta | Medio | Fijar versión; encapsular dependencias; pruebas de integración; revisión periódica de upstream | Compositor |
| R06 | Objetivo de memoria inferior a 1 GB no alcanzable en determinados escenarios | Media | Alto | Definir escenario base; presupuestos por proceso; medir PSS y memoria total; separar sesión base de aplicaciones adicionales | Rendimiento |
| R07 | Vulnerabilidades en snapshots congelados de Arch Linux | Media | Alto | Cadencia de snapshots; monitorización de avisos de seguridad; proceso de parches urgentes; promoción controlada entre canales | Linux / Seguridad |
| R08 | Entorno macOS ARM64 no permite validar la pila gráfica Linux x86_64 real | Cierta | Alto | Desarrollo auxiliar en macOS; VM Linux para pruebas generales; hardware Linux x86_64 real para DRM/KMS, GPU y drivers | Arquitectura |
| R09 | Aplicaciones GTK, Qt y Electron no mantienen coherencia visual | Cierta | Medio | Temas compatibles; portales; xdg-decoration cuando sea compatible; excepciones documentadas; no exigir uniformidad absoluta | Diseño |
| R10 | Fallo del proceso Shell deja al usuario sin interfaz de escritorio | Media | Alto | Shell reiniciable; estado recuperable desde compositor; overlay de emergencia independiente; pruebas de reconexión | Shell |
| R11 | Incompatibilidad de aplicaciones X11 con XWayland | Media | Medio | Componente XWayland separado; evaluar xwayland-satellite; matriz de pruebas; documentar aplicaciones incompatibles | Compositor |
| R12 | Licencia del proyecto no decidida bloquea la selección de dependencias | Alta | Alto | Resolver ADR-0008 antes de incorporar dependencias con licencias incompatibles o condiciones especiales | Arquitectura |
| R13 | Blur y translucidez generan sobrecoste en GPU de gama baja | Media | Medio | Reducir resolución de efectos; caché; limitar regiones; perfil de calidad; modo de transparencia reducida | Rendimiento |
| R14 | Dependencia excesiva de Caffyne como referencia visual | Baja | Medio | Identidad propia; componentes y assets originales; revisión de diseño y licencias | Diseño |
| R15 | Vendor de crates crece excesivamente y aumenta el coste de auditoría | Alta | Bajo | Minimizar dependencias; revisar árboles transitivos; fijar versiones; usar cargo-deny y cargo-vet según corresponda | Rust |
| R16 | Actualizaciones y rollback Btrfs no son realmente transaccionales | Media | Crítico | Prototipo obligatorio; validar subvolúmenes, pacman, hooks, initramfs, EFI y boot counting; no declarar rollback garantizado hasta demostrarlo | Distribución |
| R17 | Restauración de sesión incompleta después de una caída del compositor | Alta | Alto | Protocolo de recuperación cooperativa; guardar metadatos de ventanas; restauración por aplicación; documentar limitaciones | Compositor |
| R18 | Configuraciones incorrectas o incompatibles bloquean el Shell | Media | Alto | Esquemas; validación; escritura atómica; defaults seguros; recuperación de última configuración válida | Settings |
| R19 | Servicio privilegiado comprometido o mal configurado | Media | Crítico | Mínimo privilegio; polkit; usuarios dedicados; sandboxing systemd; auditoría; pruebas de autorización | Seguridad |
| R20 | Widgets externos consumen recursos o acceden a información no autorizada | Media | Alto | Procesos aislados; capacidades explícitas; límites de CPU y memoria; IPC versionado; revocación de permisos | Shell / Seguridad |
| R21 | Dependencias del ecosistema Linux cambian o dejan de mantenerse | Media | Alto | Inventario de dependencias; evaluación de mantenimiento; alternativas documentadas; política de actualización | Arquitectura |
| R22 | Firmware ausente o incompatible provoca fallos de hardware difíciles de diagnosticar | Media | Alto | Inventario de firmware; detección de errores del kernel; perfiles versionados; diagnóstico reproducible | Hardware |
| R23 | Presupuestos de memoria y CPU se incumplen por procesos auxiliares | Media | Alto | Presupuesto por proceso; mediciones PSS; límites systemd; evitar polling innecesario; pruebas prolongadas | Rendimiento |
| R24 | Falta de cobertura de hardware real durante el desarrollo | Alta | Alto | Definir matriz HW-A/B/C; pruebas automatizadas en VM y validación física; registrar kernel, GPU y firmware | QA / Hardware |
| R25 | IPC incompatible o pérdida de sincronización entre servicios | Media | Alto | Contratos XML; snapshot + delta; revisión de versiones; pruebas de reconexión; detección de lag | Arquitectura |
| R26 | Complejidad de accesibilidad y compatibilidad con lectores de pantalla | Alta | Alto | Pruebas con Orca y AT-SPI; evaluación de AccessKit; navegación por teclado; auditoría WCAG | Framework UI |
| R27 | Cambios de kernel rompen drivers, módulos o comportamiento gráfico | Media | Alto | Kernel fijado por release; pruebas de regresión; reconstrucción de módulos; canal de actualización gradual | Linux / Hardware |
| R28 | Fallos de almacenamiento o espacio insuficiente impiden actualización o recuperación | Media | Crítico | Comprobación previa de espacio; snapshots; protección de partición EFI; pruebas de recuperación ante disco lleno | Distribución |

---

# 4. Aclaraciones técnicas obligatorias

## R03 — Caída del compositor

No se considerará `catch_unwind` una solución de recuperación general.

Solo permite capturar determinados panics de Rust cuando se utiliza unwinding y el punto de captura es válido.

No protege frente a:

- `panic=abort`.
- Fallos de memoria.
- Undefined Behavior.
- Segmentation faults.
- Fallos del driver.
- Terminación forzada del proceso.

La estrategia debe distinguir:

1. Error recuperable de una operación.
2. Panic dentro de una frontera controlada.
3. Caída completa del compositor.
4. Pérdida de conexión Wayland de las aplicaciones.
5. Recuperación de sesión.

La restauración exacta de todas las aplicaciones no está garantizada. Depende de que las aplicaciones cooperen con el mecanismo de recuperación.

## R04 — NVIDIA

La compatibilidad se validará por combinación de:

- Modelo exacto de GPU.
- Driver del kernel.
- Driver de espacio de usuario.
- Firmware.
- Kernel.
- Mesa cuando corresponda.
- Backend DRM/GBM.
- Sincronización de buffers.
- Smithay y configuración del compositor.

No se declarará compatible toda una generación NVIDIA basándose en una única prueba.

## R06 — Memoria

El objetivo inferior a 1 GB debe medirse bajo escenarios definidos.

Escenario base obligatorio:

- Sistema iniciado.
- Compositor activo.
- Shell activo.
- Servicios esenciales activos.
- Sin aplicaciones del usuario.
- Sin navegador.
- Sin tareas de diagnóstico intensivas.

Escenarios adicionales:

- Terminal abierta.
- Administrador de archivos abierto.
- Múltiples ventanas.
- Aplicaciones GTK/Qt.
- Navegador.
- Widgets externos.
- Presión de memoria.

Se registrarán:

- PSS por proceso.
- RSS por proceso como dato complementario.
- Memoria total del sistema.
- Caché y memoria recuperable.
- Swap o zram.
- Memoria GPU cuando pueda medirse.

El objetivo de memoria base no debe presentarse como consumo total con aplicaciones abiertas.

## R08 — Desarrollo macOS ARM64

El MacBook M3 Pro puede utilizarse para:

- Edición.
- Compilación cruzada cuando sea viable.
- Desarrollo de lógica pura.
- Pruebas unitarias.
- Documentación.
- Diseño de interfaces.
- Herramientas auxiliares.

Una VM ARM64 puede validar parte del entorno Linux, pero no demuestra compatibilidad con hardware gráfico x86_64.

La validación definitiva de la pila gráfica requiere un sistema Linux x86_64 real.

## R09 — Aplicaciones externas

`xdg-decoration` negocia decoraciones; no obliga a todos los clientes a utilizar decoraciones del servidor.

GTK, Qt y Electron pueden tener comportamientos diferentes.

PARALLAX debe ofrecer una experiencia coherente mediante temas, portales y convenciones visuales, sin asumir control absoluto sobre las interfaces de aplicaciones externas.

## R10 — Recuperación del Shell

La recuperación debe funcionar sin depender del propio Shell.

El compositor debe conservar:

- Estado de ventanas.
- Workspaces.
- Foco.
- Geometría.
- Outputs.
- Información necesaria para reconstruir la interfaz.

El overlay de emergencia debe pertenecer al compositor o a un componente independiente.

El objetivo de reinicio inferior a un segundo será un objetivo de rendimiento medible, no una garantía arquitectónica.

## R12 — Licencias

La decisión de licencia del proyecto debe tomarse antes de incorporar dependencias importantes.

Se deben revisar:

- Licencia directa.
- Licencias transitivas.
- Obligaciones de redistribución.
- Condiciones de uso comercial.
- Compatibilidad con GPL.
- Assets, iconos y tipografías.

`cargo-deny`, `cargo-license` y `cargo-vet` cumplen funciones distintas y no sustituyen una revisión legal cuando sea necesaria.

## R15 — Vendor y auditoría

El tamaño del vendor y el coste de auditoría no son el mismo riesgo.

La estrategia debe distinguir:

- Tamaño del código fuente descargado.
- Número de dependencias directas.
- Número de dependencias transitivas.
- Dependencias abandonadas.
- Vulnerabilidades.
- Licencias.
- Auditorías de procedencia.

La compilación offline es importante, pero no justifica conservar dependencias innecesarias.

## R16 — Actualización y recuperación

Este es uno de los riesgos de mayor prioridad.

Una snapshot Btrfs no garantiza por sí sola una actualización transaccional.

Se debe demostrar:

- Coherencia entre root y subvolúmenes.
- Coherencia de la base de datos de pacman.
- Tratamiento de hooks.
- Tratamiento de initramfs.
- Gestión de kernel y módulos.
- Recuperación de entradas EFI.
- Boot counting.
- Confirmación de arranque satisfactorio.
- Recuperación cuando el sistema de archivos está lleno.

No se anunciará rollback automático hasta superar pruebas destructivas controladas.

---

# 5. Riesgos bloqueantes por fase

| Fase | Riesgos que deben resolverse o reducirse antes de aprobar |
|---|---|
| Fase 0 — Arquitectura | R01, R12, R15, R19, R21 |
| Fase 1 — Cimientos | R02, R05, R08, R24, R26 |
| Fase 2 — Compositor | R03, R04, R11, R17, R25, R27 |
| Fase 3 — Shell | R09, R10, R13, R18, R20 |
| Fase 4 — Aplicaciones | R02, R09, R23, R26 |
| Fase 5 — Servicios | R19, R20, R22, R23, R25 |
| Fase 6 — Distribución | R07, R16, R27, R28 |
| Fase 7 — Alpha | Validación integral de todos los riesgos críticos y altos |

Un riesgo no necesita desaparecer completamente para avanzar, pero debe tener una mitigación probada, un responsable y un riesgo residual aceptado.

---

# 6. Proceso de gestión

Cada riesgo tendrá:

- Identificador permanente.
- Descripción.
- Probabilidad.
- Impacto.
- Responsable.
- Mitigación.
- Estado.
- Evidencia.
- Riesgo residual.
- Fecha de última revisión.

Estados permitidos:

```text
OPEN
IN_PROGRESS
MITIGATED
ACCEPTED
CLOSED
```

### Reglas

- `OPEN`: riesgo identificado sin mitigación suficiente.
- `IN_PROGRESS`: mitigación en ejecución.
- `MITIGATED`: mitigación implementada y verificada.
- `ACCEPTED`: riesgo residual aceptado explícitamente.
- `CLOSED`: el riesgo dejó de ser aplicable o fue eliminado mediante evidencia.

No se cambiará un riesgo a `MITIGATED` únicamente porque exista documentación o código sin pruebas.

Los riesgos críticos requieren aprobación explícita del responsable de arquitectura antes de superar el gate correspondiente.

---

# 7. Revisión periódica

El registro se revisará:

1. Al finalizar cada fase.
2. Antes de aprobar un ADR que afecte seguridad o estabilidad.
3. Después de un incidente grave.
4. Cuando se incorpore una dependencia crítica.
5. Cuando cambie la matriz de hardware.
6. Antes de publicar una versión Alpha, Beta o estable.

Cada revisión debe registrar:

- Riesgos nuevos.
- Cambios de probabilidad.
- Cambios de impacto.
- Mitigaciones completadas.
- Evidencias de validación.
- Riesgos aceptados.
- Riesgos pendientes.

**Estado del registro: PROPUESTO — PENDIENTE DE APROBACIÓN DE FASE 0.**
## Riesgos de las revisiones técnicas 5–10

Los riesgos identificados en las revisiones complementarias se registran como **RV-NN** en [`project-management/consolidacion-fase0.md` §5](../project-management/consolidacion-fase0.md), con su correspondencia a R01–R28. Los identificadores `R-NN` de los informes de revisión **no** equivalen a los `RNN` de esta tabla.
