# ADR-0003: Smithay como infraestructura del compositor

- **Estado:** Propuesto
- **Fecha:** 2026-11-19
- **Ámbito:** Compositor gráfico, gestión de superficies, integración Wayland y renderizado del shell.

## 1. Contexto

PARALLAX necesita un compositor gráfico propio basado en Wayland.

Desarrollar un compositor completo desde cero implicaría implementar y mantener numerosas responsabilidades de bajo nivel:

- Gestión de dispositivos DRM/KMS.
- Integración con GBM y EGL cuando corresponda.
- Gestión de sesiones y dispositivos.
- Entrada mediante teclado, ratón y dispositivos táctiles.
- Gestión de superficies Wayland.
- Sincronización de buffers.
- Presentación de frames.
- Integración con GPU.
- Gestión de protocolos Wayland.
- Ciclo de vida de clientes y superficies.

Implementar toda esta infraestructura desde cero supondría un coste elevado y desviaría recursos del desarrollo de las funcionalidades propias de PARALLAX.

El objetivo es aprovechar una infraestructura existente en Rust sin delegar el comportamiento del entorno de escritorio.

## 2. Opciones evaluadas

| Alternativa | Ventajas | Desventajas |
|---|---|---|
| Smithay | Rust, modular, infraestructura Wayland reutilizable | API en evolución y necesidad de adaptación |
| wlroots vía FFI | Ecosistema probado y amplio soporte | Implementación principal en C y frontera FFI |
| Compositor desde cero | Control completo | Coste y complejidad muy elevados |

## 3. Decisión

PARALLAX utilizará **Smithay como infraestructura principal para implementar su compositor Wayland**.

Smithay será una dependencia de infraestructura, no el propietario de la lógica de producto.

Todo el comportamiento específico de PARALLAX se implementará en código propio Rust.

La arquitectura deberá permitir evolucionar o sustituir componentes internos de Smithay sin reescribir la lógica central del entorno.

### 3.1 Responsabilidades de Smithay

Smithay proporcionará las abstracciones y mecanismos necesarios para:

- Crear y administrar un servidor Wayland.
- Gestionar clientes y superficies.
- Integrar dispositivos de entrada.
- Administrar sesiones gráficas.
- Integrar DRM/KMS.
- Gestionar buffers y renderizado.
- Implementar protocolos Wayland mediante sus módulos disponibles.
- Facilitar la integración con el sistema gráfico Linux.

La selección concreta de módulos dependerá de la versión fijada y de las capacidades verificadas de Smithay.

### 3.2 Responsabilidades propias de PARALLAX

PARALLAX mantendrá el control sobre:

- Window Manager.
- Reglas de posicionamiento de ventanas.
- Sistema de tiling.
- Sistema de ventanas flotantes.
- Gestión de workspaces.
- Navegación entre escritorios.
- Jerarquía de ventanas.
- Políticas de foco.
- Gestión de ventanas activas e inactivas.
- Animaciones.
- Transiciones.
- Efectos visuales.
- Profundidad espacial y parallax.
- Dock.
- Launcher.
- Control Center.
- Notificaciones.
- Gestión visual de monitores.
- Políticas de renderizado del shell.

La implementación deberá mantener separadas las responsabilidades de infraestructura, composición, gestión de ventanas y experiencia visual.

## 4. Arquitectura interna

El compositor deberá organizarse mediante módulos independientes.

Estructura conceptual:

```text
crates/
└── parallax-compositor/
    └── src/
        ├── main.rs
        ├── app.rs
        │
        ├── backend/
        │   ├── mod.rs
        │   ├── drm.rs
        │   ├── session.rs
        │   └── input.rs
        │
        ├── wayland/
        │   ├── mod.rs
        │   ├── server.rs
        │   ├── clients.rs
        │   ├── surfaces.rs
        │   └── protocols.rs
        │
        ├── window_manager/
        │   ├── mod.rs
        │   ├── window.rs
        │   ├── focus.rs
        │   ├── tiling.rs
        │   ├── floating.rs
        │   └── workspaces.rs
        │
        ├── rendering/
        │   ├── mod.rs
        │   ├── renderer.rs
        │   ├── frame.rs
        │   ├── effects.rs
        │   └── damage.rs
        │
        ├── parallax/
        │   ├── mod.rs
        │   ├── depth.rs
        │   ├── transitions.rs
        │   └── composition.rs
        │
        ├── shell/
        │   ├── mod.rs
        │   ├── dock.rs
        │   ├── launcher.rs
        │   ├── panel.rs
        │   └── notifications.rs
        │
        └── output/
            ├── mod.rs
            ├── monitor.rs
            ├── configuration.rs
            └── scaling.rs
```

Esta estructura es conceptual. La separación definitiva deberá responder a dependencias reales, límites de compilación y responsabilidades de cada módulo.

## 5. Capa de adaptación de Smithay

PARALLAX deberá mantener una frontera interna entre la infraestructura Smithay y la lógica del compositor.

Reglas:

- Evitar que tipos internos de Smithay se propaguen innecesariamente por toda la aplicación.
- Centralizar las integraciones específicas de Smithay.
- Evitar dependencias directas de APIs inestables en módulos de producto.
- Mantener interfaces internas para ventanas, superficies, entradas y salidas.
- Documentar las abstracciones que dependan directamente de Smithay.

La capa de adaptación no deberá duplicar innecesariamente la funcionalidad de Smithay ni introducir abstracciones que compliquen el rendimiento.

## 6. Backend gráfico y renderizado

La arquitectura deberá soportar inicialmente un backend DRM/KMS para ejecución nativa sobre Linux.

Se evaluarán las rutas de renderizado disponibles en Smithay según:

- Compatibilidad con GPU.
- Rendimiento.
- Soporte de formatos de buffer.
- Sincronización.
- Presentación de frames.
- Soporte multimonitor.
- Gestión de memoria gráfica.

La implementación deberá contemplar:

- Renderizado acelerado por GPU.
- Composición de superficies.
- Damage tracking.
- Sincronización de buffers.
- VSync y presentación de frames.
- Gestión de recursos gráficos.
- Recuperación ante errores del backend.

La ruta de renderizado definitiva se seleccionará mediante pruebas técnicas, no por preferencia tecnológica.

## 7. Protocolos Wayland

PARALLAX implementará los protocolos necesarios para ofrecer compatibilidad con aplicaciones Linux.

Se priorizarán:

- `xdg-shell`.
- `xdg-decoration`.
- `xdg-output`.
- `wl-output`.
- `wl-seat`.
- `wl-data-device`.
- `primary-selection`.
- `pointer-constraints`.
- `relative-pointer`.
- `pointer-gestures`.
- `text-input`.
- `fractional-scale`.
- `viewporter`.
- `presentation-time`.
- `linux-dmabuf`.
- `linux-explicit-synchronization`.

La disponibilidad, versión y madurez de cada protocolo deberán verificarse en la versión seleccionada de Smithay.

No se implementarán protocolos innecesarios únicamente por completar una lista.

Los protocolos deberán habilitarse progresivamente según su utilidad, estabilidad y requisitos de interoperabilidad.

## 8. Gestión de entradas y sesiones

El compositor deberá soportar:

- Teclados.
- Ratones.
- Touchpads.
- Pantallas táctiles.
- Múltiples dispositivos de entrada.
- Hotplug.
- Configuración de distribución de teclado.
- Gestos.
- Administración de sesiones gráficas.

La integración deberá respetar las responsabilidades de libinput, libseat y el sistema de sesiones utilizado.

## 9. Compatibilidad con aplicaciones

PARALLAX deberá priorizar la compatibilidad con aplicaciones Wayland nativas.

También deberá evaluar mecanismos para ejecutar aplicaciones X11 heredadas mediante XWayland.

La compatibilidad deberá comprobarse con aplicaciones reales, incluyendo navegadores, herramientas de desarrollo, aplicaciones multimedia y aplicaciones GTK/Qt.

El compositor no deberá exigir modificaciones en aplicaciones externas para ofrecer funcionalidad Wayland estándar.

## 10. Gestión de versiones y mantenimiento

La versión de Smithay deberá fijarse explícitamente en el proyecto.

Se mantendrá un proceso de seguimiento upstream mensual.

Toda actualización deberá:

1. Revisar cambios incompatibles.
2. Evaluar modificaciones en APIs públicas.
3. Ejecutar pruebas de compilación.
4. Ejecutar pruebas de integración.
5. Validar el backend gráfico.
6. Evaluar regresiones de rendimiento.
7. Actualizar la capa de adaptación cuando corresponda.

No se actualizará Smithay automáticamente en versiones stable de PARALLAX.

Las actualizaciones críticas de seguridad se evaluarán independientemente del calendario mensual.

## 11. Estrategia de pruebas

Se establecerán pruebas para:

- Inicio y cierre del compositor.
- Creación y destrucción de superficies.
- Gestión de ventanas.
- Foco y entrada.
- Redimensionamiento.
- Tiling.
- Workspaces.
- Conexión y desconexión de monitores.
- Escalado.
- Sincronización de frames.
- Recuperación ante errores gráficos.
- Compatibilidad con aplicaciones Wayland.
- Compatibilidad con XWayland.

Se utilizarán backends de prueba o entornos virtualizados cuando sean compatibles con las pruebas requeridas.

Las pruebas sobre hardware real seguirán siendo obligatorias para validar DRM/KMS, GPU y dispositivos físicos.

## 12. Rendimiento

El compositor deberá priorizar:

- Baja latencia de entrada.
- Presentación de frames estable.
- Uso eficiente de CPU y GPU.
- Control de asignaciones de memoria.
- Evitar copias innecesarias de buffers.
- Evitar renderizados cuando no existan cambios.
- Gestión eficiente de múltiples superficies.
- Recuperación controlada ante fallos.

Los objetivos de rendimiento deberán medirse mediante benchmarks y trazas.

No se asumirán garantías de rendimiento por el simple hecho de utilizar Rust.

## 13. Consecuencias

### Positivas

- Infraestructura Wayland en Rust.
- Menor esfuerzo de implementación de bajo nivel.
- Aprovechamiento de componentes existentes.
- Mayor concentración en las funcionalidades propias.
- Posibilidad de contribuir al ecosistema upstream.
- Menor necesidad de desarrollar abstracciones gráficas desde cero.

### Negativas

- Dependencia de un proyecto comunitario.
- Evolución de APIs.
- Necesidad de mantener compatibilidad entre versiones.
- Curva de aprendizaje.
- Posibles limitaciones en funcionalidades específicas.
- Necesidad de mantener una capa de adaptación.

## 14. Criterios de aceptación

La implementación se considerará válida cuando:

1. PARALLAX pueda iniciar un servidor Wayland funcional.
2. Pueda ejecutar aplicaciones Wayland nativas.
3. Disponga de gestión propia de ventanas.
4. Soporte ventanas flotantes y tiling.
5. Disponga de gestión de workspaces.
6. Soporte entradas y salidas de vídeo.
7. Funcione con renderizado acelerado por GPU en hardware compatible.
8. Soporte escalado y múltiples monitores.
9. Disponga de mecanismos de recuperación ante errores.
10. La lógica propia no dependa innecesariamente de detalles internos de Smithay.

## 15. Revisión

Este ADR deberá revisarse cuando:

- Smithay presente bloqueos superiores a seis meses en una funcionalidad crítica.
- Una limitación técnica impida alcanzar los objetivos del producto.
- Aparezca una alternativa Rust significativamente superior.
- Los costes de mantenimiento de la integración superen sus beneficios.
- Cambien sustancialmente los requisitos gráficos de PARALLAX.

## 16. Principio rector

**Smithay proporciona los mecanismos para construir el compositor. PARALLAX define cómo funciona, cómo se comporta y cómo se experimenta.**

PARALLAX no será una personalización de otro compositor ni una réplica de un escritorio existente.

Será un entorno gráfico propio construido sobre infraestructura abierta y mantenible.