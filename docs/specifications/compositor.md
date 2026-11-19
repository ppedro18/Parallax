# Especificación: parallax-compositor

**Estado:** Propuesto · Pendiente de aprobación.  
**Componente:** `parallax-compositor`  
**Capa:** C4 — Core  
**Infraestructura:** Smithay + calloop + DRM/KMS + Wayland.  
**Gestor de ventanas:** `px-wm`.

## 1. Responsabilidad

`parallax-compositor` es el servidor Wayland principal de la sesión gráfica PARALLAX.

Es responsable de:

- Gestionar la sesión gráfica.
- Administrar dispositivos de entrada.
- Gestionar DRM/KMS y los outputs.
- Mantener el servidor Wayland.
- Gestionar superficies y clientes.
- Coordinar el renderizado y la composición.
- Aplicar las decisiones de `px-wm`.
- Gestionar foco, stacking y activación.
- Gestionar los protocolos Wayland soportados.
- Integrar la Shell con el escritorio.
- Aplicar las políticas de seguridad de superficies.
- Recuperarse de errores recuperables sin comprometer la sesión.

El compositor no implementa lógica de negocio de aplicaciones ni administra directamente servicios que ya tienen una autoridad propia en el sistema.

## 2. Principios arquitectónicos

1. `px-wm` es la fuente de verdad del estado lógico de ventanas.
2. Smithay proporciona infraestructura Wayland y de bajo nivel.
3. El compositor es el único propietario de DRM/KMS durante la sesión.
4. Ningún servicio externo modifica directamente el estado interno del compositor.
5. El hilo de renderizado no realiza operaciones bloqueantes de disco, red o D-Bus.
6. Las operaciones de entrada y composición deben ser predecibles.
7. Los errores de clientes no deben convertirse en fallos globales del servidor.
8. El bloqueo de sesión debe fallar de forma segura.
9. El renderizado solo se realiza cuando existe trabajo visual pendiente.
10. Las decisiones de recuperación deben estar respaldadas por pruebas.

## 3. Arquitectura interna

### 3.1 Modelo de ejecución

El bucle principal utiliza `calloop`.

Arquitectura conceptual:

```text
parallax-compositor
│
├── Main Event Loop (calloop)
│   │
│   ├── Session Backend
│   │   ├── libseat / logind
│   │   ├── VT switch
│   │   ├── Session pause/resume
│   │   └── Device ownership
│   │
│   ├── Device Discovery
│   │   └── udev
│   │
│   ├── DRM/KMS
│   │   ├── GPU devices
│   │   ├── Connectors
│   │   ├── CRTCs
│   │   ├── Planes
│   │   └── Outputs
│   │
│   ├── Input
│   │   ├── libinput
│   │   ├── Keyboard
│   │   ├── Pointer
│   │   ├── Touch
│   │   └── Input Pipeline
│   │
│   ├── Wayland Display
│   │   ├── xdg-shell
│   │   ├── layer-shell
│   │   ├── xdg-decoration
│   │   ├── ext-session-lock
│   │   ├── Output protocols
│   │   └── PARALLAX protocols
│   │
│   ├── px-wm::State
│   │
│   ├── Renderer
│   │   ├── GLES renderer
│   │   ├── Buffer management
│   │   ├── Damage tracking
│   │   ├── Effects
│   │   └── Presentation
│   │
│   ├── Xwayland
│   │   └── Optional compatibility backend
│   │
│   └── Internal Command Queue
│
└── D-Bus Integration
    ├── zbus connection
    ├── org.parallax.Session1
    └── Bounded command/event channel
```

### 3.2 Event loop

El compositor debe tener un único propietario lógico del estado de sesión y del estado de `px-wm`.

Reglas:

- El event loop procesa eventos de Wayland, input, DRM y temporizadores.
- Los eventos D-Bus se convierten en comandos internos.
- Las operaciones de renderizado se coordinan desde el event loop.
- Las tareas bloqueantes se delegan a workers.
- Las respuestas de workers regresan mediante canales acotados.
- No se permite modificar `px-wm::State` desde otros hilos.

No utilizar un runtime asíncrono global sin necesidad.

`calloop` será la infraestructura principal del compositor. Tokio puede utilizarse en servicios independientes o en integraciones justificadas, pero no debe introducirse en el camino crítico de renderizado sin una necesidad demostrada.

### 3.3 Frontera Smithay

Mantener una capa de adaptación entre Smithay y los componentes propios.

```text
Smithay
   |
   v
px-compositor-adapter
   |
   +---- px-wm
   |
   +---- px-render
   |
   +---- px-input
   |
   +---- px-protocols
```

No exponer tipos de Smithay por todo el código del proyecto.

La adaptación debe permitir actualizar Smithay sin propagar sus cambios a las reglas de gestión de ventanas.

## 4. Gestión de sesión

### 4.1 Estados

Los estados de sesión son:

- Starting.
- Active.
- Paused.
- Locked.
- Terminating.

Transiciones conceptuales:

```text
Starting
   |
   v
 Active <--------> Paused
   |
   v
 Locked
   |
   v
 Active
   |
   v
Terminating
```

Este diagrama no implica que todos los estados sean mutuamente excluyentes.

La implementación debe modelar separadamente:

- Estado de ejecución.
- Disponibilidad de la sesión gráfica.
- Estado de bloqueo.
- Estado de energía.
- Disponibilidad de outputs.

Por ejemplo, una sesión puede estar bloqueada y posteriormente pausada por un cambio de VT.

### 4.2 Starting

Durante el inicio:

1. Adquirir la sesión mediante libseat/logind.
2. Obtener acceso autorizado a dispositivos.
3. Inicializar DRM/KMS.
4. Detectar outputs.
5. Inicializar renderer.
6. Inicializar el servidor Wayland.
7. Inicializar input.
8. Inicializar `px-wm`.
9. Registrar protocolos.
10. Publicar disponibilidad de la sesión.
11. Esperar conexión de Shell.

El compositor no debe anunciarse como completamente disponible antes de inicializar sus recursos esenciales.

### 4.3 Active

La sesión está activa y puede:

- Recibir input.
- Presentar frames.
- Gestionar ventanas.
- Recibir comandos autorizados.
- Administrar outputs.

### 4.4 Paused

Se produce cuando logind/libseat informa pérdida temporal de la sesión o suspensión de dispositivos.

Comportamiento:

- Detener el acceso a dispositivos pausados.
- No intentar presentar frames sobre un DRM fd que ya no está disponible.
- Pausar operaciones dependientes del hardware.
- Mantener el estado lógico de ventanas.
- Reanudar únicamente cuando la sesión vuelva a estar disponible.

No asumir que todos los dispositivos recuperan automáticamente el mismo estado.

### 4.5 Locked

El bloqueo es una condición de seguridad prioritaria.

Debe utilizarse `ext-session-lock-v1` o la implementación equivalente validada para la versión de Smithay seleccionada.

Reglas:

- No permitir que el bloqueo sea una simple ventana superpuesta.
- Impedir la interacción con superficies ordinarias.
- No mostrar contenido de aplicaciones en el escritorio bloqueado.
- Renderizar exclusivamente las superficies autorizadas de bloqueo y las superficies internas imprescindibles para la seguridad.
- No aceptar un desbloqueo por pérdida de conexión.
- No aceptar un desbloqueo por timeout.
- No aceptar un desbloqueo por fallo del renderer.
- No aceptar un desbloqueo por reinicio de Shell.

El compositor debe mantener la condición de bloqueo independientemente del proceso cliente que la solicitó.

### 4.6 Fallo del cliente de bloqueo

Si el cliente de bloqueo muere:

1. Mantener el estado Locked.
2. Destruir o invalidar sus superficies.
3. Presentar una superficie de emergencia propiedad del compositor.
4. Ocultar contenido de aplicaciones.
5. Registrar el fallo.
6. Solicitar recuperación del cliente de bloqueo.
7. Mantener bloqueada la sesión hasta una autenticación válida.

La superficie de emergencia puede ser un color sólido, pero no debe ser el único mecanismo de protección.

La autenticación debe estar implementada mediante un componente de confianza independiente y un contrato seguro con el compositor.

El desbloqueo requiere una operación autenticada y explícita.

Un reinicio del cliente de bloqueo no puede equivaler a un desbloqueo.

### 4.7 Terminating

Durante el cierre:

- Rechazar nuevas operaciones.
- Detener presentación.
- Liberar dispositivos.
- Cerrar conexiones Wayland.
- Liberar recursos DRM.
- Cerrar workers.
- Registrar motivo de terminación.
- Finalizar limpiamente.

## 5. Integración de `px-wm`

`px-wm` mantiene la lógica pura de gestión de ventanas.

El compositor es responsable de:

- Convertir eventos de input en acciones.
- Validar solicitudes de clientes.
- Aplicar las operaciones producidas por `px-wm`.
- Actualizar superficies.
- Coordinar renderizado.
- Publicar eventos.

El compositor no debe duplicar las reglas de geometría, focus, layout y workspace de `px-wm`.

### 5.1 Modelo de estado

Los estados visuales de una ventana deben ser ortogonales.

Separar:

**Geometría**

- Floating.
- Tiled.
- Maximized.
- Fullscreen.

**Visibilidad**

- Mapped.
- Unmapped.
- Minimizado como política interna.

**Actividad**

- Focused.
- Unfocused.
- Urgent.

**Interacción**

- Idle.
- Moving.
- Resizing.
- Modal.

No representar todas estas combinaciones mediante un único enum.

### 5.2 Ciclo de vida

Ciclo conceptual:

```text
Created
   |
   v
Mapped
   |
   +---- Floating
   |
   +---- Tiled
   |
   +---- Maximized
   |
   +---- Fullscreen
   |
   +---- Minimized (interno)
   |
   v
Unmapped
   |
   v
Destroyed
```

Las transiciones pueden originarse en:

- Solicitudes Wayland.
- Acciones del usuario.
- Comandos de Shell.
- Reglas de ventanas.
- Cambios de output.
- Cambios de workspace.

El compositor debe validar las solicitudes antes de aplicar las acciones.

No todas las operaciones solicitadas por un cliente implican que el compositor deba obedecerlas.

### 5.3 Minimized

La minimización no es un estado estándar de `xdg-shell`.

Es una política interna de PARALLAX.

Debe especificarse:

- Qué ocurre con la superficie.
- Qué ocurre con el foco.
- Cómo se representa en Shell.
- Cómo se restaura.
- Qué sucede con las ventanas transitorias.
- Cómo se notifica al cliente cuando corresponda.

No confundir minimizar con desmapear o destruir una superficie.

### 5.4 Eventos

El compositor publica cambios mediante los contratos definidos en `event-model.md`.

Ejemplos:

- `toplevel.created`
- `toplevel.mapped`
- `toplevel.changed`
- `toplevel.unmapped`
- `toplevel.destroyed`
- `focus.changed`
- `workspace.changed`
- `output.added`
- `output.removed`

Los eventos deben tener:

- Propietario.
- Epoch.
- Revisión.
- Identificador.
- Payload tipado.

No emitir eventos duplicados ni reemitir eventos recibidos de otro propietario.

La Shell obtiene el estado inicial mediante snapshot y después consume deltas.

## 6. Renderizado

### 6.1 Renderer

Implementación inicial:

- GLES renderer mediante la infraestructura de Smithay.
- DRM/KMS.
- GBM cuando corresponda.
- Gestión explícita de buffers.
- Damage tracking por output.
- Presentación sincronizada con vblank.

Vulkan se evalúa como alternativa futura mediante benchmarks.

No asumir que Vulkan mejora necesariamente consumo de memoria, latencia o compatibilidad.

### 6.2 Pipeline

```text
Wayland surfaces
       |
       v
Surface tree
       |
       v
Damage calculation
       |
       v
Scene composition
       |
       +---- Window geometry
       +---- Shadows
       +---- Rounded corners
       +---- Blur
       +---- Overlays
       |
       v
Output renderer
       |
       v
DRM/KMS presentation
```

### 6.3 Renderizado sin trabajo

- Un frame se presenta cuando existe daño o una animación activa.
- No generar frames vacíos.
- No mantener timers de renderizado innecesarios.
- Invalidar solo regiones afectadas cuando sea posible.
- Coalescer múltiples cambios dentro de un mismo ciclo de presentación.

Una animación activa debe solicitar nuevos frames únicamente mientras tenga trabajo pendiente.

### 6.4 Damage tracking

Debe cubrir:

- Movimiento de ventanas.
- Cambios de geometría.
- Cambios de contenido.
- Cambios de foco.
- Cambios de stacking.
- Cambios de workspace.
- Efectos.
- Superficies de Shell.
- Cambios de output.

El daño debe propagarse correctamente entre superficies y outputs.

Evitar invalidaciones globales cuando pueda calcularse una región afectada.

### 6.5 Efectos

Los efectos deben cumplir `RENDERING_PERFORMANCE`.

| Efecto | Técnica | Condición |
|---|---|---|
| Esquinas redondeadas | SDF en shader | Según geometría |
| Sombras | Texturas 9-slice cacheadas | Solo cuando sean visibles |
| Blur | Dual Kawase | Superficies autorizadas por política |
| Scrim | Quad con alfa | Overlays |
| Animaciones | Transformación de textura | Sin rerasterizar contenido |
| Overview | Miniaturas cacheadas | Actualización limitada |

Los efectos deben poder degradarse individualmente.

### 6.6 Blur

- Solo sobre superficies que lo soliciten y estén autorizadas.
- Resolución reducida inicial de 1/4.
- No procesar regiones que no hayan cambiado.
- Cachear resultados cuando el fondo permanezca estable.
- Desactivar automáticamente cuando supere su presupuesto.
- Fallback opaco cuando sea necesario.

El tratamiento de vídeo y contenido dinámico debe seguir las reglas de `RENDERING_PERFORMANCE`.

### 6.7 Direct scanout

Fullscreen puede utilizar direct scanout cuando las condiciones de hardware y presentación lo permitan.

No garantizarlo para todos los casos.

Depende de:

- Formato de buffer.
- Modificadores.
- Planos disponibles.
- Composición.
- Overlay surfaces.
- Sincronización.
- Compatibilidad del driver.
- Estado de output.

Cuando se active direct scanout, los efectos que requieran composición deben desactivarse o provocar el retorno al pipeline compuesto.

### 6.8 VRR

VRR es opcional.

Debe validarse por:

- GPU.
- Driver.
- Monitor.
- Output.
- Modo de presentación.
- Compatibilidad con composición.

No introducirlo en la ruta crítica de v1 antes de tener un comportamiento estable de presentación sincronizada.

## 7. Input

### 7.1 Pipeline

El pipeline de input debe distinguir:

1. Estado de bloqueo y seguridad.
2. Grabs activos.
3. Inhibición de atajos.
4. Atajos globales permitidos.
5. Superficie con foco.

El orden no debe provocar que los atajos globales interrumpan una operación de move/resize ya iniciada.

### 7.2 Atajos globales

- Configurables.
- Basados en una política de atajos del escritorio.
- Conflictos detectados.
- Modificadores documentados.
- Integración con `settingsd`.

Los atajos no deben procesarse cuando el bloqueo de sesión exige una política más restrictiva.

### 7.3 Grabs

Move y resize:

- Capturan la interacción correspondiente.
- Tienen prioridad sobre atajos ordinarios.
- Se cancelan correctamente.
- Gestionan pérdida de dispositivo.
- Gestionan cambio de output.
- Gestionan destrucción de ventana.
- Mantienen invariantes de geometría.

### 7.4 Keyboard shortcuts inhibit

Soportar `zwp_keyboard_shortcuts_inhibit_manager_v1` cuando esté disponible en la infraestructura seleccionada.

Reglas:

- La inhibición se concede mediante política del compositor.
- No supone una autorización de ejecución.
- No debe desactivar protecciones de seguridad.
- El compositor conserva el control de bloqueo de sesión.
- La Shell puede ofrecer preferencias de autorización.
- La política debe distinguir aplicaciones confiables y escenarios de uso.

No exigir una confirmación por cada evento de inhibición si el usuario ya estableció una política aplicable.

### 7.5 Input y foco

- El foco se controla mediante `px-wm`.
- No permitir robo de foco arbitrario.
- Validar tokens de activación cuando corresponda.
- Respetar políticas de activación.
- Gestionar correctamente dispositivos hotplug.
- Mantener comportamiento coherente en multi-monitor.

## 8. Multi-monitor y GPU

### 8.1 Modelo de outputs

Cada output debe tener:

- Identificador interno estable durante su ciclo de vida.
- Identidad de conector.
- EDID cuando esté disponible.
- Resolución.
- Refresh rate.
- Escala.
- Transformación.
- Posición lógica.
- Estado conectado/desconectado.
- GPU asociada.

No depender exclusivamente de EDID para identificar permanentemente un monitor.

El EDID puede faltar, estar duplicado o contener datos incompletos.

Usar una identidad compuesta con fallback por conector y dispositivo.

### 8.2 Multi-GPU

El diseño debe distinguir:

- GPU de renderizado.
- GPU propietaria del output.
- GPU de importación de buffers.
- GPU de presentación.

No asumir que todos los outputs pueden compartir directamente los mismos buffers.

La interoperabilidad entre GPUs debe validarse mediante DMA-BUF, formatos, modificadores y sincronización.

### 8.3 Escala

Soportar escala fraccional por output mediante los protocolos correspondientes.

La escala lógica no debe modificar la geometría física del monitor.

Las aplicaciones deben poder utilizar los protocolos estándar disponibles para escala fraccional y viewport.

### 8.4 Hotplug

Cuando desaparezca un output:

1. Detectar desconexión.
2. Actualizar el modelo de outputs.
3. Seleccionar output de destino.
4. Migrar ventanas según política de `px-wm`.
5. Conservar workspace cuando sea posible.
6. Recalcular geometría.
7. Actualizar la Shell.
8. Publicar evento de cambio.

La migración debe preservar el estado de las ventanas y evitar geometrías fuera de los límites del output destino.

## 9. Xwayland

Xwayland es un componente de compatibilidad independiente.

- Inicio bajo demanda.
- Gestión de ciclo de vida independiente.
- Integración con el servidor Wayland.
- Soporte de superficies X11 mediante la infraestructura disponible.
- Aislamiento de fallos cuando sea posible.
- No introducir dependencias de X11 en el modelo interno de ventanas.

La referencia a ADR-0012 debe revisarse: la decisión de Xwayland debe apuntar al ADR que realmente defina compatibilidad y ciclo de vida de Xwayland.

No asumir que una caída de Xwayland implica una caída del compositor.

## 10. D-Bus

El compositor expone `org.parallax.Session1` mediante zbus.

La integración D-Bus debe mantenerse fuera del camino crítico de renderizado.

### 10.1 Comunicación

- Hilo o worker dedicado cuando sea necesario.
- Canal acotado hacia el event loop.
- Comandos tipados.
- Validación de permisos.
- Rechazo de comandos obsoletos.
- Control de backpressure.
- Cancelación de operaciones.
- Respuestas estructuradas.

El worker D-Bus no modifica directamente `px-wm::State`.

### 10.2 API conceptual

`org.parallax.Session1`:

- `GetSessionState`
- `GetOutputs`
- `GetWorkspaces`
- `GetToplevels`
- `ActivateToplevel`
- `MoveToplevel`
- `SetToplevelState`
- `SwitchWorkspace`
- `GetCapabilities`

Estas operaciones son conceptuales. Las firmas y permisos definitivos deben definirse en el contrato D-Bus versionado.

No exponer comandos arbitrarios que permitan saltarse la política de activación o el modelo de seguridad del compositor.

### 10.3 Autorización

- Operaciones ordinarias de sesión no requieren privilegios administrativos.
- Operaciones privilegiadas deben delegarse al servicio correspondiente.
- El compositor no sustituye polkit.
- El estado de bloqueo no se puede modificar mediante una llamada D-Bus ordinaria.

## 11. Errores y recuperación

### 11.1 Cliente envía protocolo inválido

Comportamiento:

1. Registrar diagnóstico.
2. Desconectar el cliente según las reglas del protocolo.
3. Liberar sus recursos.
4. Mantener consistentes las superficies y referencias.
5. Continuar la sesión.

### 11.2 Pánico interno

No utilizar `catch_unwind` como mecanismo general de aislamiento de clientes.

Un pánico puede ocurrir después de modificar parcialmente estructuras internas o durante una transición que deja invariantes incumplidas.

Reglas:

- Evitar panics en el camino de eventos.
- Validar entradas.
- Utilizar `Result`.
- Mantener transiciones atómicas.
- Documentar invariantes.
- Usar `catch_unwind` únicamente en fronteras donde exista una estrategia real de recuperación.
- No continuar después de un pánico si el estado interno puede haber quedado inconsistente.

Un cliente que envía un mensaje inválido debe desconectarse por la ruta normal de error de protocolo, no por un pánico artificial.

### 11.3 Fallo del renderer

Si se pierde el contexto gráfico:

1. Detener presentación.
2. Marcar recursos GPU como potencialmente inválidos.
3. Intentar recrear renderer.
4. Revalidar formatos y capacidades.
5. Reimportar buffers cuando sea posible.
6. Invalidar caches.
7. Reconstruir damage tracking.
8. Reanudar presentación.

La reutilización de buffers depende de su validez y del backend. No asumir que todos los recursos pueden reimportarse.

Si falla repetidamente:

- Desactivar efectos opcionales.
- Intentar un backend o modo seguro previamente validado.
- Si no puede garantizarse una presentación segura, terminar controladamente.

El número de intentos y el intervalo de recuperación deben configurarse y probarse; tres fallos es un valor inicial propuesto, no una garantía de recuperación.

### 11.4 Shell no disponible

Si Shell no se conecta dentro de cinco segundos:

- Mostrar overlay de emergencia propiedad del compositor.
- Permitir las operaciones mínimas de seguridad.
- Mantener acceso a bloqueo y salida de sesión.
- Registrar el estado.
- Solicitar reinicio de Shell mediante systemd.

El overlay no debe implementar una segunda Shell completa.

### 11.5 Caída del compositor

Una caída del compositor destruye la conexión Wayland y las superficies de la sesión.

No garantizar que las aplicaciones recuperen sus ventanas automáticamente.

La recuperación debe:

- Registrar el motivo.
- Reiniciar el compositor cuando corresponda.
- Restaurar la sesión de forma cooperativa.
- Solicitar a las aplicaciones compatibles que restauren su estado.
- Mantener protegida la sesión si estaba bloqueada.

El reinicio del compositor no puede convertir una sesión bloqueada en una sesión desbloqueada.

La estrategia de recuperación de sesión debe coordinarse con los servicios de sesión y bloqueo.

## 12. Seguridad

### 12.1 Límites de confianza

El compositor procesa mensajes de clientes no confiables.

Debe asumir que:

- Las superficies pueden destruirse en cualquier momento.
- Los buffers pueden ser inválidos.
- Las solicitudes pueden ser malformadas.
- Los clientes pueden desconectarse durante operaciones.
- Los recursos pueden agotarse.
- Los clientes pueden intentar robar foco.
- Los clientes pueden intentar bloquear la respuesta del servidor.

### 12.2 Protección

- Validación de mensajes.
- Límites de recursos por cliente.
- Gestión de buffers.
- Límites de superficies.
- Límites de memoria.
- Límites de operaciones pendientes.
- Gestión de timeouts.
- Aislamiento de Xwayland.
- Protección de bloqueo.
- Registro de errores sin filtrar datos sensibles.

### 12.3 Seguridad del bloqueo

La sesión bloqueada debe mantener un estado seguro ante:

- Muerte del lock client.
- Caída de Shell.
- Fallo del renderer.
- Desconexión de un output.
- Suspensión.
- Reanudación.
- Cambio de VT.
- Reinicio de servicios.

No permitir que una ruta de recuperación desactive el bloqueo sin autenticación.

## 13. Rendimiento

Aplicar `docs/benchmarks/metodologia.md` y `RENDERING_PERFORMANCE`.

### 13.1 Presupuesto inicial

HW-A, 60 Hz, 1080p:

- CPU del compositor: máximo 4 ms por output.
- GPU de composición: máximo 4 ms por output.
- Presentación: sincronizada con vblank cuando corresponda.
- Sin renderizado ocioso.
- Damage tracking activo.

Los presupuestos CPU y GPU son límites independientes de trabajo, no una garantía de que el frame completo se presente dentro de 8 ms.

Medir también:

- Tiempo de espera.
- Composición.
- Presentación.
- Frame deadline.
- Frames perdidos.
- Jitter.
- Picos de latencia.

### 13.2 Memoria

Medir:

- PSS del proceso.
- RSS.
- Memoria GPU.
- Buffers importados.
- Cachés.
- Recursos por cliente.
- Crecimiento durante soak tests.

No confundir memoria GPU con PSS del proceso.

### 13.3 Observabilidad

Exponer métricas de diagnóstico:

- Frame time CPU.
- Frame time GPU.
- Frames presentados.
- Frames perdidos.
- Daño por output.
- Tiempo de composición.
- Estado del renderer.
- Uso de buffers.
- Número de superficies.
- Número de clientes.
- Eventos de recuperación.

Las métricas de alta frecuencia deben agregarse antes de enviarse a Observatory.

## 14. Pruebas

### 14.1 Clientes Wayland

- weston-simple-shm.
- weston-simple-egl.
- foot.
- Firefox.
- Aplicaciones GTK4.
- Aplicaciones Qt6.

Las aplicaciones de prueba deben ejecutarse con versiones y configuraciones registradas.

### 14.2 Xwayland

- Aplicaciones X11 representativas.
- Gestión de ventanas.
- Input.
- Clipboard.
- Resize.
- Multi-monitor.
- Fallos de cliente.

### 14.3 Protocolos

- xdg-shell.
- xdg-decoration.
- layer-shell.
- output management aplicable.
- fractional-scale.
- viewporter.
- presentation-time.
- linux-dmabuf.
- explicit synchronization cuando esté disponible.
- session-lock.
- keyboard-shortcuts-inhibit.
- protocolos privados PARALLAX.

Cada protocolo debe tener pruebas positivas, negativas y de desconexión inesperada.

### 14.4 Multi-monitor

- Conexión y desconexión.
- Cambios de resolución.
- Cambios de refresh rate.
- Escala fraccional.
- Orientación.
- Hotplug.
- Migración de ventanas.
- GPU híbrida cuando esté disponible.

### 14.5 Seguridad y bloqueo

- Bloqueo normal.
- Muerte del lock client.
- Muerte de Shell durante bloqueo.
- Desconexión de output.
- Suspensión durante bloqueo.
- Reanudación.
- VT switch.
- Intento de desbloqueo no autorizado.

Criterio fundamental: ningún fallo puede provocar desbloqueo implícito.

### 14.6 Soak test

Duración inicial: 24 horas.

Escenario:

- Apertura y cierre continuo de ventanas.
- Movimiento y resize.
- Cambios de workspace.
- Cambios de foco.
- Actualización de superficies.
- Cambios de output cuando sea posible.

Criterios:

- Sin crashes.
- Sin deadlocks.
- Sin errores críticos acumulativos.
- Sin crecimiento de PSS superior al 5 % respecto a una línea base estabilizada, considerando la variabilidad y el número de ventanas.
- Sin crecimiento sostenido de recursos GPU.
- Sin acumulación de superficies huérfanas.

Registrar baseline, condiciones y métricas durante toda la ejecución.

### 14.7 Latencia

Objetivo:

- Input-to-photon p99 inferior a 16 ms en HW-A a 60 Hz, para escenarios de interacción definidos.

La medición debe diferenciar:

- Input recibido por el sistema.
- Evento procesado.
- Frame generado.
- Frame presentado.
- Fotón visible en pantalla.

`presentation-time` ayuda a medir la presentación de frames, pero no demuestra por sí solo input-to-photon.

La medición completa requiere instrumentación adecuada y, para una validación real de fotones, hardware externo como cámara de alta velocidad o fotodiodo.

### 14.8 VT y suspensión

- 100 ciclos de VT switch.
- 100 ciclos de suspensión y reanudación.
- Combinaciones de ambos escenarios.
- Validación de dispositivos.
- Validación de outputs.
- Validación de renderer.
- Validación de bloqueo.
- Sin pérdida permanente de la sesión.

## 15. Organización del código

Estructura conceptual:

```text
crates/
├── parallax-compositor/
│   └── src/
│       ├── main.rs
│       ├── event_loop/
│       ├── session/
│       ├── wayland/
│       ├── protocols/
│       ├── input/
│       ├── outputs/
│       ├── renderer/
│       ├── security/
│       ├── recovery/
│       ├── dbus/
│       └── diagnostics/
│
├── px-wm/
├── px-wm-layouts/
├── px-compositor-adapter/
├── px-render/
├── px-input/
├── px-protocols/
└── px-types/
```

No dividir módulos en crates independientes hasta que exista una necesidad real de reutilización, aislamiento o pruebas.

## 16. Criterios de aceptación

El compositor estará listo para solicitar aprobación cuando:

- [ ] Inicializa y administra una sesión Wayland.
- [ ] Utiliza DRM/KMS en hardware real.
- [ ] Gestiona input mediante libinput.
- [ ] Integra `px-wm` sin duplicar lógica.
- [ ] Ejecuta clientes Wayland de prueba.
- [ ] Soporta aplicaciones GTK4 y Qt6.
- [ ] Soporta Xwayland como componente opcional.
- [ ] Gestiona varios outputs.
- [ ] Gestiona hotplug.
- [ ] Implementa damage tracking.
- [ ] No renderiza frames vacíos en reposo.
- [ ] Cumple los presupuestos de rendimiento.
- [ ] Mantiene el bloqueo ante fallos del cliente de bloqueo.
- [ ] No utiliza `catch_unwind` como mecanismo ficticio de aislamiento.
- [ ] Supera el soak test de 24 horas.
- [ ] Supera 100 ciclos de VT switch.
- [ ] Supera 100 ciclos de suspensión y reanudación.
- [ ] Cumple la metodología de latencia.
- [ ] Tiene métricas y diagnóstico.
- [ ] No tiene defectos críticos abiertos.

## 17. Dependencias arquitectónicas

Esta especificación depende de:

- ADR-0001: Rust.
- ADR-0003: Smithay.
- ADR-0004: `px-wm`.
- ADR-0007: IPC.
- ADR-0009: actualización y recuperación, para el comportamiento de sesión y sistema.
- Modelo de eventos.
- Arquitectura de servicios.
- Arquitectura de hardware.
- Arquitectura de IPC y contratos.
- Metodología de rendimiento.
- Especificación de estados de ventana.
- Sistema de profundidad.
- Especificación de bloqueo de sesión.

Las referencias a ADR deben mantenerse sincronizadas con los documentos realmente existentes.

## 18. Decisiones pendientes

Antes de cerrar la especificación:

1. Definir backend DRM y estrategia multi-GPU.
2. Definir el mecanismo definitivo de recuperación del renderer.
3. Definir la implementación y el cliente de `ext-session-lock-v1`.
4. Definir el servicio de autenticación de bloqueo.
5. Definir el contrato entre compositor y Shell para el overlay de emergencia.
6. Definir los protocolos Wayland privados que realmente serán necesarios.
7. Definir la estrategia de Xwayland y su ADR correspondiente.
8. Definir la instrumentación input-to-photon.
9. Definir límites de recursos por cliente.
10. Validar la versión de Smithay y disponibilidad de los protocolos requeridos.

**Estado final: Propuesto. No aprobado. La implementación queda condicionada a la resolución de las decisiones pendientes y a la aprobación del gate arquitectónico correspondiente.**