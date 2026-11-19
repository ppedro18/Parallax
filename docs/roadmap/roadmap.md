# PARALLAX — Roadmap de desarrollo

- **Estado:** Fase 0 — Arquitectura y especificaciones.
- **Fecha:** 2026-11-19.
- **Principio:** Ninguna fase avanza sin revisión y aprobación explícita.
- **Objetivo:** Construir un entorno de escritorio Linux basado en Arch, con componentes propios desarrollados principalmente en Rust, arquitectura modular, rendimiento medible y recuperación ante fallos.

---

## 1. Principios de ejecución

El desarrollo de PARALLAX seguirá estos principios:

1. **Especificar antes de implementar:** ninguna funcionalidad importante se desarrollará sin requisitos y criterios de aceptación.
2. **Validación incremental:** cada fase debe producir software ejecutable y verificable.
3. **Hardware real:** las capacidades gráficas y de integración del sistema deberán probarse en equipos físicos.
4. **Rendimiento desde el principio:** memoria, CPU, GPU y latencia tendrán presupuestos medibles.
5. **Recuperación obligatoria:** los componentes críticos deberán disponer de estrategias de recuperación.
6. **Seguridad por diseño:** privilegios mínimos, validación de IPC y aislamiento de extensiones.
7. **Rust como lenguaje principal:** utilizar tecnologías maduras del ecosistema Linux cuando resulte técnicamente conveniente.
8. **Sin avances automáticos:** la aprobación de una fase no implica la aprobación de la siguiente.

## 2. Control de fases

Cada fase tendrá cinco elementos obligatorios:

- Objetivo.
- Alcance.
- Entregables.
- Criterios de aceptación.
- Revisión y aprobación.

Una fase se considerará completada únicamente cuando:

- Sus entregables estén disponibles.
- Las pruebas correspondientes hayan sido ejecutadas.
- Los defectos críticos estén resueltos.
- La documentación esté actualizada.
- Los ADR afectados estén aceptados o tengan excepciones documentadas.
- El propietario haya aprobado explícitamente el resultado.

Los elementos no terminados deberán registrarse como pendientes y no ocultarse dentro de la siguiente fase.

---

# FASE 0 — Arquitectura y especificaciones

**Estado:** Actual.

### Objetivo

Definir la arquitectura, las responsabilidades, las tecnologías y los requisitos verificables de PARALLAX antes de comenzar la implementación.

### Alcance

- Arquitectura general.
- Arquitectura del compositor.
- Gestión de ventanas.
- Sistema de diseño.
- Arquitectura del Shell.
- Modelo de procesos.
- IPC.
- Sistema de archivos.
- Actualizaciones y recuperación.
- Seguridad.
- Gestión de memoria.
- Rendimiento.
- Modelo de extensiones.
- Accesibilidad.
- Instalación y distribución.
- Agentes de desarrollo y documentación.

### ADR principales

- ADR-0001: Rust como lenguaje de componentes propios.
- ADR-0002: Base de paquetes Arch.
- ADR-0003: Smithay como infraestructura del compositor.
- ADR-0004: Gestor de ventanas en proceso.
- ADR-0005: Toolkit de interfaz.
- ADR-0006: Arquitectura de procesos del Shell.
- ADR-0007: IPC.
- ADR-0008: Licencias y dependencias.
- ADR-0009: Actualizaciones y recuperación.

### Entregables

- `specifications/`
- `architecture/`
- `security/`
- `benchmarks/`
- `agents/`
- `decisions/`
- `roadmap.md`
- `README.md`

### Criterios de aceptación

- Todos los ADR iniciales están revisados.
- Las decisiones pendientes están identificadas.
- Los componentes y sus responsabilidades están definidos.
- Las interfaces entre subsistemas están documentadas.
- Existe una especificación de rendimiento.
- Existe una metodología de benchmark.
- Existe una estrategia de pruebas.
- Los agentes tienen instrucciones y límites claros.

### Gate 0

**Aprobación:** arquitectura suficientemente definida para comenzar los experimentos técnicos sin comprometer prematuramente decisiones fundamentales.

---

# FASE 1 — Cimientos y validación tecnológica

### Objetivo

Preparar el entorno de desarrollo, validar las tecnologías seleccionadas y construir las primeras bibliotecas compartidas.

Esta fase no busca producir un escritorio funcional.

Busca demostrar que la infraestructura técnica de PARALLAX puede construirse, compilarse y mantenerse de manera reproducible.

### 1.1 Entorno de desarrollo

Preparar:

- Linux Arch nativo como entorno de referencia.
- VM Arch para pruebas aisladas.
- Entorno complementario en macOS.
- Workspace Cargo.
- Toolchain Rust fijado.
- Dependencias del sistema documentadas.
- Repositorio local de dependencias.
- Cargo vendor para compilaciones offline.
- CI local.
- Scripts de desarrollo.
- Formato y análisis estático.
- Política de versiones.

La VM servirá para desarrollo y pruebas de integración, pero no sustituirá las pruebas DRM/KMS sobre hardware real.

### 1.2 Workspace Rust

Crear el workspace inicial:

```text
parallax/
├── Cargo.toml
├── Cargo.lock
├── rust-toolchain.toml
├── .cargo/
│   └── config.toml
├── crates/
│   ├── px-types/
│   ├── px-ipc/
│   ├── px-config/
│   └── px-telemetry/
├── apps/
├── services/
├── distribution/
├── specifications/
├── architecture/
├── security/
├── benchmarks/
├── tests/
└── tools/
```

Las bibliotecas deberán tener responsabilidades independientes.

No se crearán abstracciones sin consumidores o necesidades técnicas justificadas.

### 1.3 Bibliotecas iniciales

**px-types**
- Tipos compartidos.
- Identificadores.
- Geometría.
- Estados.
- Tipos de error.
- Tipos de dominio independientes.

**px-ipc**
- Tipos de mensajes.
- Versionado.
- Serialización.
- Validación.
- Límites de tamaño.
- Compatibilidad de protocolos.

**px-config**
- Carga de configuración.
- Validación.
- Valores predeterminados.
- Migraciones.
- Persistencia.

**px-telemetry**
- Eventos internos.
- Métricas.
- Trazas.
- Registro estructurado.
- Diagnóstico.

### 1.4 Spikes tecnológicos

Validar mediante prototipos independientes:

- Smithay y sus backends.
- Iced y alternativas del ADR-0005.
- Integración Wayland.
- Layer-shell.
- AccessKit.
- Renderizado acelerado.
- D-Bus mediante zbus.
- Transporte IPC local.
- Btrfs y snapshots.
- systemd-boot y boot counting.

Cada spike deberá tener hipótesis, implementación mínima, resultados y conclusión.

### 1.5 Benchmarks iniciales

Establecer una línea base para:

- Consumo de memoria.
- Tiempo de compilación.
- Inicialización de procesos.
- Latencia IPC.
- Renderizado.
- Uso de CPU.
- Rendimiento de listas.
- Accesibilidad.

### Entregables

- Workspace Cargo reproducible.
- Build offline funcional.
- CI local.
- Bibliotecas iniciales.
- Informe comparativo del toolkit.
- Informe de integración Wayland.
- Metodología de benchmark.
- ADR-0005 aceptado.
- Dependencias y licencias inventariadas.

### Criterios de aceptación

- Compilación offline desde un entorno preparado.
- Dependencias fijadas.
- Pruebas automatizadas funcionales.
- Formato y Clippy integrados.
- Resultados de benchmarks reproducibles.
- Toolkit seleccionado con evidencia técnica.
- Limitaciones conocidas documentadas.

### Gate 1

**Aprobación:** infraestructura de desarrollo reproducible y tecnologías fundamentales validadas.

---

# FASE 2 — Compositor mínimo

### Objetivo

Construir el primer compositor Wayland propio de PARALLAX utilizando Smithay como infraestructura.

El objetivo es conseguir una sesión gráfica funcional antes de desarrollar el Shell.

### 2.1 Backend gráfico

Implementar:

- Inicialización DRM/KMS.
- Descubrimiento de dispositivos.
- GBM y gestión de buffers según backend.
- EGL o backend de renderizado seleccionado.
- Gestión de presentación.
- VSync.
- Gestión de daños.
- Ciclo de vida del dispositivo gráfico.
- Recuperación ante errores del backend.

### 2.2 Sesión e input

Implementar:

- Integración con logind mediante la infraestructura seleccionada.
- Gestión de sesiones.
- Liberación y recuperación de dispositivos.
- libinput.
- Teclado.
- Ratón.
- Touchpad.
- Atajos básicos.
- Gestión de foco.

### 2.3 Protocolos Wayland

Implementar progresivamente:

- `wl_compositor`.
- `wl_shm`.
- `wl_seat`.
- `wl_output`.
- `xdg-shell`.
- `xdg-decoration`, cuando corresponda.
- `wlr-layer-shell`.
- Protocolos de escalado necesarios.

Layer-shell se incluye en esta fase porque es una dependencia del Shell.

Los protocolos se habilitarán según los requisitos de las aplicaciones y superficies.

### 2.4 Gestor de ventanas

Implementar `px-wm` como biblioteca independiente.

Funcionalidades iniciales:

- Ventanas flotantes.
- Movimiento.
- Redimensionamiento.
- Maximizado.
- Minimizado.
- Cierre.
- Foco.
- Stacking.
- Workspaces básicos.
- Gestión de monitores.

La lógica deberá probarse sin iniciar el compositor.

### 2.5 Multi-output

Implementar:

- Detección de monitores.
- Conexión y desconexión.
- Resoluciones.
- Frecuencias.
- Escalado.
- Reconfiguración.
- Asignación de workspaces.

### 2.6 Compatibilidad

Validar aplicaciones Wayland nativas.

XWayland se considerará una capacidad adicional planificada y no se asumirá como requisito completado por el simple hecho de disponer de xdg-shell.

### Entregables

- `px-compositor`.
- `px-wm`.
- Backend DRM/KMS.
- Backend de desarrollo para pruebas.
- Configuración básica.
- Herramientas de diagnóstico.
- Tests unitarios y de integración.
- Fuzzing de `px-wm`.

### Criterios de aceptación

En HW-A:

- Inicia una sesión gráfica.
- Abre foot.
- Abre Firefox en modo Wayland nativo.
- Administra ventanas flotantes.
- Gestiona foco y teclado.
- Detecta los monitores previstos.
- No presenta errores críticos de renderizado.
- Sobrevive a desconexiones de dispositivos de entrada.
- Mantiene estabilidad durante pruebas prolongadas.

En VM:

- Ejecuta el backend de desarrollo.
- Permite probar protocolos.
- Permite ejecutar clientes Wayland.
- Permite reproducir errores.

### Gate 2

**Aprobación:** compositor parcialmente aceptado sobre hardware real, con pruebas de protocolos, gestión de ventanas y estabilidad.

---

# FASE 3 — Shell núcleo

### Objetivo

Construir la experiencia de escritorio propia de PARALLAX sobre el compositor.

Esta fase transforma el compositor mínimo en un entorno utilizable diariamente.

### 3.1 Infraestructura de interfaz

Implementar:

- `px-ui`.
- Tokens visuales.
- Temas.
- Tipografía.
- Iconografía.
- Estados.
- Componentes.
- Accesibilidad.
- Animaciones.
- Integración con el sistema de profundidad.

### 3.2 Componentes principales

Implementar:

- Topbar.
- Dock.
- Launcher.
- Control Center.
- Notificaciones.
- Menús contextuales.
- Indicadores.
- Paneles flotantes.
- Widgets oficiales.

### 3.3 Servicios del Shell

Implementar:

- `parallax-shell`.
- `settingsd`.
- `notifyd`.
- Servicio de bloqueo.
- Gestión de configuración.
- Comunicación con compositor.
- Integración con D-Bus.
- Supervisión de widgets externos.

El bloqueo de sesión deberá utilizar un protocolo de bloqueo seguro compatible con el compositor. No se considerará suficiente dibujar una ventana que cubra la pantalla.

### 3.4 Arquitectura de procesos

Aplicar ADR-0006:

- Un proceso principal para componentes oficiales.
- Widgets externos fuera de proceso.
- IPC declarativo.
- Validación de mensajes.
- Capacidades y permisos.
- Recuperación del Shell.

### 3.5 Experiencia de usuario

Implementar:

- Gestión de escritorios.
- Cambios de workspace.
- Animaciones.
- Apertura de aplicaciones.
- Indicadores del sistema.
- Configuración visual.
- Escalado.
- Navegación por teclado.

### Entregables

- `px-ui`.
- `parallax-shell`.
- `settingsd`.
- `notifyd`.
- Lock.
- Sistema de temas.
- Sistema de widgets.
- Configuración inicial.
- Manual de interacción.

### Criterios de aceptación

- Sesión diaria utilizable.
- Dock, Topbar y Launcher funcionales.
- Notificaciones funcionales.
- Bloqueo seguro de sesión.
- Configuración persistente.
- Accesibilidad básica y navegación por teclado.
- Aplicaciones de terceros funcionales.
- Reinicio del Shell sin reiniciar el compositor.
- Widgets externos aislados.
- Consumo de recursos medido.

### Gate 3

**Aprobación:** entorno de escritorio funcional para trabajo diario, sin depender de aplicaciones propias todavía.

---

# FASE 4 — Aplicaciones núcleo

### Objetivo

Construir aplicaciones esenciales para administrar y utilizar PARALLAX.

Las aplicaciones deberán utilizar `px-ui` y los servicios oficiales del sistema.

### Aplicaciones

**Terminal**
- Emulador de terminal.
- Gestión de sesiones.
- Perfiles.
- Atajos.
- Configuración visual.
- Integración con portales cuando corresponda.

**Files**
- Navegación de archivos.
- Operaciones básicas.
- Copiar y mover.
- Búsqueda.
- Papelera.
- Permisos.
- Vistas.
- Integración con volúmenes.

**Settings**
- Apariencia.
- Pantallas.
- Teclado y ratón.
- Sonido.
- Red.
- Bluetooth.
- Usuarios.
- Privacidad.
- Accesibilidad.
- Actualizaciones.

**Task Manager**
- Procesos.
- CPU.
- Memoria.
- GPU cuando sea posible.
- Terminación controlada.
- Servicios.
- Diagnóstico básico.

### Entregables

- `px-terminal`.
- `px-files`.
- `px-settings`.
- `px-task-manager`.

### Criterios de aceptación

Cada aplicación deberá contar con:

- Especificación funcional.
- Diseño validado.
- Pruebas unitarias.
- Pruebas de integración.
- Navegación por teclado.
- Gestión de errores.
- Rendimiento medido.
- Documentación de uso.

### Gate 4

**Aprobación:** conjunto de aplicaciones núcleo funcional y suficientemente estable.

---

# FASE 5 — Inteligencia y servicios del sistema

### Objetivo

Desarrollar servicios de supervisión, diagnóstico y optimización del sistema.

Esta fase deberá mejorar la experiencia sin introducir consumo permanente innecesario.

### Servicios

**resourced**
- Supervisión de recursos.
- Presupuestos.
- Presión de memoria.
- Uso de CPU.
- Señales de carga.
- Políticas de optimización.

**hardwared**
- Inventario de hardware.
- Detección de dispositivos.
- Estado de sensores disponibles.
- Información de batería.
- Información térmica.
- Capacidades gráficas.

**diagd**
- Diagnóstico.
- Registro de errores.
- Estado de servicios.
- Detección de problemas.
- Informes de soporte.

**Observatory**
- Visualización de métricas.
- Historial.
- Alertas.
- Estado del sistema.
- Diagnóstico visual.

**Widgets**
- Widgets oficiales.
- Integración con servicios.
- Extensiones externas.
- Gestión de permisos.

### Principios

- No realizar polling frecuente cuando existan eventos del sistema.
- Utilizar interfaces del kernel y servicios existentes.
- No duplicar funcionalidades maduras de systemd.
- Limitar la frecuencia de actualización.
- Evitar servicios permanentes cuando puedan activarse bajo demanda.
- Separar monitorización de políticas de optimización.

### Entregables

- Servicios implementados.
- Observatory.
- Sistema de métricas.
- Widgets.
- Pruebas de estrés.
- Informe de consumo.

### Criterios de aceptación

- Consumo de reposo medido.
- Límites de CPU y memoria.
- Pruebas de presión de memoria.
- Comportamiento correcto bajo carga.
- Recuperación de servicios.
- Ausencia de polling innecesario.
- Diagnóstico reproducible.
- No degradación significativa de la experiencia.

### Gate 5

**Aprobación:** servicios estables y beneficios medibles sin comprometer los presupuestos de recursos.

---

# FASE 6 — Distribución, instalación y recuperación

### Objetivo

Convertir PARALLAX en un sistema instalable, actualizable y recuperable.

### 6.1 Repositorios

Implementar:

- Repositorio oficial firmado.
- Gestión de claves.
- PKGBUILD.
- Construcciones reproducibles cuando sea viable.
- Canales dev, beta y stable.
- Inventario de paquetes.
- SBOM.
- Auditoría de dependencias.

### 6.2 Construcción de imágenes

Evaluar y seleccionar herramientas:

- archiso para imágenes basadas en Arch.
- mkosi para construcción y pruebas de imágenes.
- systemd-sysupdate para la evolución hacia actualizaciones A/B.

La herramienta seleccionada deberá responder a los requisitos reales de instalación y mantenimiento.

### 6.3 Instalador

Implementar:

- Detección de hardware.
- Particionado.
- Btrfs.
- Subvolúmenes.
- EFI.
- Instalación del sistema.
- Configuración de usuarios.
- Configuración del arranque.
- Instalación de bootloader.
- Configuración inicial.
- Validación de instalación.

### 6.4 Actualizaciones

Implementar:

- `px-update`.
- Validación de repositorios.
- Firmas.
- Snapshots previos.
- Transacciones pacman.
- Actualización de kernel.
- Generación de initramfs.
- Entradas systemd-boot.
- Boot counting.
- Confirmación de arranque.
- Rollback automático.

### 6.5 Recuperación

Implementar:

- `recoveryd`.
- Menú de recuperación.
- Restauración de generaciones.
- Diagnóstico de fallos.
- Recuperación manual.
- Conservación de una generación funcional.

### Entregables

- Repositorio firmado.
- Image builder.
- ISO instalable.
- Instalador.
- `px-update`.
- `recoveryd`.
- Documentación de instalación.
- Documentación de recuperación.

### Criterios de aceptación

- ISO instalable en HW-A.
- Instalación limpia.
- Primer arranque funcional.
- Actualización normal funcional.
- Actualización interrumpida recuperable.
- Fallo de kernel recuperable.
- Rollback automático probado.
- Rollback manual probado.
- Verificación de firmas.
- Recuperación sin perder la capacidad de arrancar.
- Procedimiento de rescate documentado.

### Gate 6

**Aprobación:** PARALLAX puede instalarse, actualizarse y recuperarse de fallos reproducibles.

---

# FASE 7 — Alfa pública

### Objetivo

Preparar una versión alfa que pueda ser probada por usuarios externos bajo condiciones documentadas.

La alfa no representa una versión estable ni una promesa de compatibilidad universal.

### 7.1 Endurecimiento

- Auditoría de seguridad.
- Revisión de permisos.
- Auditoría IPC.
- Pruebas de aislamiento.
- Auditoría de dependencias.
- Pruebas de recuperación.
- Revisión de fallos críticos.

### 7.2 Accesibilidad

- Navegación por teclado.
- Lectores de pantalla.
- Contraste.
- Escalado.
- Foco.
- Estados accesibles.
- Reduced motion.
- High contrast.
- Validación con tecnologías de asistencia.

### 7.3 Rendimiento

- Benchmarks de memoria.
- Benchmarks de CPU.
- Benchmarks GPU.
- Frame times.
- Tiempo de inicio.
- Consumo en reposo.
- Consumo bajo carga.
- Pruebas prolongadas.
- Fugas de memoria.

### 7.4 Matriz de hardware

Validar progresivamente:

- HW-A: hardware principal de desarrollo.
- HW-B: equipo Intel/AMD alternativo.
- HW-C: GPU y monitor adicionales.
- HW-D: portátil y gestión de energía.

La matriz deberá registrar modelos, controladores, firmware, resoluciones y limitaciones.

### 7.5 Distribución pública

- ISO alfa.
- Notas de versión.
- Requisitos de hardware.
- Problemas conocidos.
- Guía de instalación.
- Guía de recuperación.
- Sistema de reportes.
- Política de seguridad.
- Licencias.
- Procedimiento de actualización.

### Criterios de aceptación

- Instalación reproducible.
- Arranque funcional en la matriz declarada.
- Sesión gráfica estable.
- Aplicaciones núcleo utilizables.
- Actualizaciones y recuperación verificadas.
- Problemas conocidos publicados.
- Seguridad y licencias revisadas.
- Rendimiento documentado.
- Documentación de soporte disponible.

### Gate 7

**Aprobación:** publicación de PARALLAX Alpha para evaluación externa.

---

# FASE 8 — Beta y estabilización

Esta fase se ejecutará después de recopilar resultados de la alfa.

### Objetivo

Reducir regresiones, estabilizar APIs y preparar una distribución de uso general.

### Alcance

- Corrección de errores.
- Estabilización del compositor.
- Compatibilidad con aplicaciones.
- Estabilidad del Shell.
- Actualizaciones.
- Seguridad.
- Accesibilidad.
- Rendimiento.
- Instalador.
- Documentación.
- Compatibilidad de hardware.

### Gate 8

**Aprobación:** cumplimiento de los criterios de versión estable v1.0.

---

## 3. Dependencias entre fases

Las fases son secuenciales para su aprobación, pero algunas actividades de investigación podrán desarrollarse en paralelo cuando no comprometan las decisiones pendientes.

Relaciones principales:

```text
FASE 0
  |
  v
FASE 1
  |
  v
FASE 2 ───── Compositor / px-wm / Layer-shell
  |
  v
FASE 3 ───── Shell / px-ui / Servicios visuales
  |
  v
FASE 4 ───── Aplicaciones
  |
  v
FASE 5 ───── Servicios / Observatory
  |
  v
FASE 6 ───── Instalador / Actualización / Recovery
  |
  v
FASE 7 ───── Alfa pública
  |
  v
FASE 8 ───── Beta / v1.0
```

La investigación de distribución, licencias y seguridad podrá comenzar antes de la Fase 6.

Sin embargo, sus criterios de aceptación deberán cumplirse antes de publicar una imagen instalable.

## 4. Hardware de referencia

Se establecerá una matriz de hardware antes de la Fase 2.

| Identificador | Propósito |
|---|---|
| HW-A | Equipo principal de desarrollo y aceptación |
| HW-B | Compatibilidad alternativa de CPU/GPU |
| HW-C | Pruebas multi-monitor y diferentes resoluciones |
| HW-D | Portátiles, batería y suspensión |

Cada perfil deberá tener una ficha con:

- CPU.
- GPU.
- Controlador.
- Kernel.
- RAM.
- Firmware.
- Pantallas.
- Resolución.
- Frecuencia.
- Dispositivos de entrada.

No se considerará validado un componente sobre hardware que no esté identificado.

## 5. Gestión de calidad

Cada componente tendrá una clasificación:

- Experimental.
- En desarrollo.
- Funcional.
- Validado.
- Estable.

No se utilizará la etiqueta estable sin pruebas de regresión y aceptación.

Las pruebas se organizarán en:

1. Unitarias.
2. Propiedades.
3. Fuzzing.
4. Integración.
5. Sistema.
6. Rendimiento.
7. Seguridad.
8. Accesibilidad.
9. Recuperación.

## 6. Política de cambios

Una decisión arquitectónica aceptada no se modificará silenciosamente.

Si una implementación contradice un ADR:

1. Registrar el problema.
2. Documentar la evidencia.
3. Proponer una modificación.
4. Actualizar el ADR correspondiente.
5. Obtener aprobación.
6. Implementar el cambio.

No se aceptarán soluciones provisionales que se conviertan accidentalmente en arquitectura permanente.

## 7. Registro de aprobaciones

Cada gate deberá registrar:

- Fase.
- Fecha.
- Commit o versión evaluada.
- Entregables.
- Pruebas ejecutadas.
- Resultados.
- Problemas pendientes.
- Excepciones autorizadas.
- Decisión del propietario.

Estados permitidos:

- `PENDING`
- `IN_PROGRESS`
- `BLOCKED`
- `READY_FOR_REVIEW`
- `APPROVED`
- `REJECTED`

## 8. Principio rector del roadmap

**PARALLAX no avanzará por cantidad de código escrito, sino por capacidades demostradas.**

Cada fase deberá producir un resultado verificable, reproducible y suficientemente estable para convertirse en fundamento de la siguiente.

El objetivo no es construir rápidamente una interfaz que parezca un sistema operativo.

El objetivo es construir un entorno Linux propio, coherente, eficiente, mantenible y capaz de recuperarse de sus propios fallos.