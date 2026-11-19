# Especificación: Dock — PARALLAX

**Estado:** Propuesto · Pendiente de aprobación.  
**Componente:** `parallax-shell`  
**Subcomponente:** `Dock`  
**Capa:** C5 — Desktop Experience.  
**Superficie:** Wayland Layer Shell.

## 1. Responsabilidad

El Dock es un componente de `parallax-shell` que proporciona acceso rápido a aplicaciones favoritas, aplicaciones abiertas, ventanas activas y Papelera.

Responsabilidades:

- Mostrar aplicaciones fijadas.
- Mostrar aplicaciones abiertas no fijadas.
- Agrupar ventanas por aplicación.
- Permitir activar, alternar y gestionar ventanas.
- Permitir fijar y desfijar aplicaciones.
- Permitir iniciar aplicaciones.
- Mostrar indicadores de notificaciones y progreso.
- Permitir drag-and-drop de archivos.
- Gestionar visibilidad automática.
- Adaptarse a múltiples monitores.
- Proporcionar navegación mediante teclado y tecnologías de asistencia.

El Dock no es responsable de:

- Crear procesos directamente.
- Administrar ventanas.
- Implementar el gestor de ventanas.
- Administrar notificaciones.
- Administrar la Papelera a nivel de almacenamiento.
- Gestionar políticas de GPU.
- Ejecutar comandos arbitrarios.

Estas operaciones se delegan a los componentes y servicios correspondientes.

## 2. Arquitectura

El Dock vive dentro del proceso único `parallax-shell`, según ADR-0006.

```text
parallax-shell
│
├── Dock
│   ├── DockController
│   ├── DockView
│   ├── DockLayout
│   ├── DockItems
│   ├── DockInteractions
│   ├── DockAutoHide
│   ├── DockDragDrop
│   ├── DockTooltips
│   ├── DockPreviews
│   └── DockAccessibility
│
├── SessionClient
│   └── org.parallax.Session1
│
├── CompositorClient
│   └── Wayland / PARALLAX protocols
│
├── SettingsClient
│   └── org.parallax.Settings1
│
└── NotificationsClient
    └── org.freedesktop.Notifications / PARALLAX
```

El Dock no mantiene una segunda fuente de verdad de las ventanas abiertas.

Consume el estado del compositor y mantiene únicamente estado de presentación e interacción.

## 3. Superficie y posicionamiento

### 3.1 Wayland Layer Shell

El Dock utiliza una superficie Layer Shell:

- Layer: `top`.
- Anchors: borde inferior, izquierdo o derecho.
- Keyboard interactivity: `none` en funcionamiento normal, salvo durante interacciones que requieran captura explícita.
- Exclusive zone: según el modo de visibilidad.
- Output: asociado a un monitor determinado.

La superficie debe utilizar el protocolo de Layer Shell disponible en la versión de Smithay seleccionada.

### 3.2 Posiciones

Opciones:

- Bottom.
- Left.
- Right.

Configuración inicial:

- Bottom.
- Centrado.
- Flotante.
- Separación configurable del borde.
- Margen configurable.
- Radio de esquina según `DEPTH_SYSTEM`.

El Dock vertical debe reorganizar sus elementos sin alterar las reglas de agrupación ni las interacciones.

### 3.3 Outputs

Modos de visualización:

- `primary`: solo el output primario.
- `all`: un Dock por output conectado.
- `selected`: outputs elegidos por el usuario.

El modo predeterminado será `primary`.

Cada Dock asociado a un output debe tener su propio estado visual, pero compartir las mismas favoritas y el mismo estado global de aplicaciones.

No duplicar una aplicación como dos procesos por aparecer en dos Docks.

### 3.4 Hotplug

Cuando se desconecta un monitor:

1. Detectar la pérdida del output.
2. Destruir la superficie del Dock asociada.
3. Seleccionar un output válido.
4. Recrear el Dock según la configuración.
5. Mantener las favoritas y el orden.
6. Evitar duplicar indicadores o suscripciones.

Cuando se conecta un nuevo monitor, aplicar la política `primary`, `all` o `selected`.

## 4. Contenido

### 4.1 Secciones

Orden visual:

```text
┌─────────────────────────────────────────────┐
│ Favoritas │ Separador │ Abiertas │ Separador │ Papelera │
└─────────────────────────────────────────────┘
```

Secciones:

1. Aplicaciones favoritas.
2. Separador.
3. Aplicaciones abiertas no fijadas.
4. Separador.
5. Papelera.

Las aplicaciones fijadas y abiertas deben fusionarse visualmente cuando representen la misma aplicación.

Una aplicación fijada que está abierta aparece una sola vez, con sus indicadores de ejecución.

### 4.2 Favoritas

Fuente de verdad:

`settingsd`

Clave:

`dock.pinned`

Cada favorita debe almacenar un identificador estable de aplicación, preferentemente el Desktop Entry ID validado.

Ejemplo conceptual:

```toml
[dock]
pinned = [
  "org.example.Browser.desktop",
  "org.example.Files.desktop",
  "org.example.Terminal.desktop"
]
```

Reglas:

- El orden persistido es el orden visual.
- Eliminar una aplicación instalada no debe borrar automáticamente su favorita.
- Una favorita no disponible debe mostrar un estado identificable.
- Al reinstalar la aplicación, debe recuperarse su representación.
- No almacenar rutas ejecutables arbitrarias como identificadores.
- No guardar duplicados.
- La escritura debe delegarse a `settingsd`.

El Dock actualiza su representación después de recibir confirmación o un evento de configuración.

### 4.3 Aplicaciones abiertas

El compositor es la autoridad del estado de toplevels.

El Dock obtiene:

- `app_id`.
- Identificador de ventana.
- Título.
- Estado de activación.
- Output.
- Workspace.
- Estado de minimización interno, si corresponde.
- Indicadores de urgencia.

Priorizar protocolos estándar disponibles, como `ext-foreign-toplevel-list-v1`, cuando sean suficientes.

Para operaciones que no estén cubiertas por los protocolos estándar, utilizar los contratos privados definidos en la arquitectura IPC.

No utilizar el título de ventana como identificador de aplicación.

### 4.4 Agrupación

Agrupar ventanas por identidad de aplicación.

Reglas:

- Una aplicación puede tener varias ventanas.
- Un icono representa el grupo.
- Un indicador muestra el número de ventanas.
- Hasta tres puntos visibles.
- Más de tres ventanas se representan mediante un indicador compacto de cantidad.
- El orden de las ventanas debe ser estable y determinista.
- Las ventanas de otros workspaces pueden aparecer según una preferencia configurable.

La identidad debe resolverse mediante una asociación controlada entre `app_id` y Desktop Entry ID.

No asumir que `app_id` siempre coincide literalmente con el nombre del archivo `.desktop`.

### 4.5 Iconos

Resolución:

1. Icono declarado por el Desktop Entry validado.
2. Icono del tema de iconos PARALLAX.
3. Icono del tema freedesktop disponible.
4. Icono genérico de aplicación.

Los iconos deben respetar las dimensiones y estados de `ICONOGRAPHY`.

No ejecutar código ni cargar recursos arbitrarios desde una aplicación para representar su icono.

### 4.6 Papelera

La Papelera es una entrada fija al final del Dock.

Debe:

- Abrir el gestor de archivos en la Papelera.
- Mostrar un indicador cuando contenga elementos.
- Mostrar un estado vacío cuando no tenga contenido, si la integración lo permite.
- Respetar las operaciones de la especificación de almacenamiento.

No implementar una segunda Papelera dentro del Dock.

No eliminar archivos directamente desde una interacción ambigua.

## 5. Indicadores

### 5.1 Estado de ejecución

Representación:

- No ejecutada: sin indicador.
- Ejecutada: indicador inferior.
- Activa: indicador destacado.
- Urgente: indicador de atención sin robar foco.

El estilo debe ser coherente con los estados de ventana de PARALLAX.

### 5.2 Número de ventanas

- 1 ventana: un punto.
- 2 ventanas: dos puntos.
- 3 ventanas: tres puntos.
- Más de 3: tres puntos y cantidad compacta o contador.

No mostrar contadores redundantes cuando el grupo no tenga varias ventanas.

### 5.3 Notificaciones

Fuente de verdad:

`notifyd`.

El Dock recibe indicadores agregados por aplicación.

Reglas:

- No almacenar el historial de notificaciones.
- No duplicar la gestión de notificaciones.
- No mostrar contenido sensible en badges.
- Permitir ocultar badges mediante configuración.
- Respetar la política de privacidad de notificaciones.

Los badges deben representar información agregada y no necesariamente el número de ventanas.

### 5.4 Progreso de aplicación

Soportar compatibilidad con `com.canonical.Unity.LauncherEntry` cuando sea viable.

Interpretar:

- Progreso.
- Estado de progreso.
- Conteo.
- Urgencia.

La compatibilidad debe aislarse en un adaptador.

No convertir la compatibilidad Unity en el contrato interno del Dock.

La API interna de PARALLAX debe utilizar un modelo tipado independiente.

Los indicadores de progreso deben expirar o limpiarse cuando la aplicación desaparezca o deje de proporcionar información.

## 6. Interacciones

### 6.1 Clic sobre aplicación cerrada

Comportamiento:

1. Resolver el identificador de aplicación.
2. Validar que la aplicación esté disponible.
3. Solicitar lanzamiento mediante `Session1.Launch`.
4. Iniciar animación de lanzamiento.
5. Esperar el evento de creación/mapeo del toplevel correspondiente.
6. Detener animación al detectar el lanzamiento.

Timeout visual inicial: 10 segundos.

Si no aparece una ventana:

- Detener la animación.
- No mostrar un error modal automáticamente.
- Mantener la favorita.
- Permitir reintentar.
- Registrar el resultado cuando exista información de fallo.

El timeout de animación no debe confundirse con el timeout de ejecución del proceso.

El servicio de sesión debe devolver un identificador de operación o resultado de lanzamiento para permitir correlación.

### 6.2 Clic sobre aplicación abierta con una ventana

Comportamiento:

- Si no está activa: solicitar activación.
- Si está activa: solicitar minimización mediante la política interna de PARALLAX.

La minimización debe implementarse mediante el contrato compositor–Shell.

No asumir que existe un método Wayland estándar para minimizar ventanas.

El Dock no debe cambiar visualmente el estado de la ventana hasta recibir confirmación del compositor.

### 6.3 Clic sobre aplicación con varias ventanas

Comportamiento predeterminado:

- Abrir un selector de ventanas si existen varias ventanas.
- El selector presenta miniaturas y títulos.
- La ventana elegida recibe una solicitud de activación.

Una preferencia opcional puede permitir ciclo entre ventanas.

El ciclo debe ser determinista y mantener el orden de ventanas.

No alternar silenciosamente entre ventanas sin ofrecer una forma accesible de identificar el destino.

### 6.4 Hover

Al pasar el cursor:

- Mostrar tooltip.
- Mostrar título de aplicación.
- Mostrar contador cuando corresponda.
- Si existen varias ventanas, mostrar selector de miniaturas después de un breve retraso.
- No bloquear la interacción con el Dock.

El hover debe ser configurable para reducir movimiento o animaciones.

### 6.5 Clic derecho

Menú contextual:

- Ventanas abiertas.
- Acciones declaradas en el Desktop Entry.
- Fijar.
- Desfijar.
- Cerrar todas las ventanas.
- Abrir con GPU dedicada, si está disponible.

Las acciones de Desktop Entry deben validarse y ejecutarse mediante el servicio de lanzamiento.

No permitir que el Dock interprete acciones arbitrarias como comandos internos.

### 6.6 Cerrar todas las ventanas

Proceso:

1. Identificar las ventanas del grupo.
2. Solicitar cierre normal.
3. Esperar la respuesta de los clientes.
4. Mostrar estado de cierre cuando corresponda.
5. Permitir una acción explícita de cierre forzado si el sistema ofrece esa capacidad.

No matar procesos arbitrariamente desde el Dock.

El cierre forzado debe requerir una acción diferenciada del usuario.

### 6.7 Arrastrar y reordenar

Las favoritas pueden reordenarse mediante drag-and-drop.

Comportamiento:

- Mostrar indicador de posición.
- Actualizar orden visual durante el arrastre.
- Persistir el nuevo orden al finalizar.
- Cancelar y restaurar si la operación se interrumpe.
- Evitar duplicados.
- Mantener identidad de aplicación.

El orden final se guarda mediante `settingsd`.

### 6.8 Desfijar

Arrastrar una favorita fuera del área válida del Dock puede iniciar una operación de desfijado.

Debe existir una zona de descarte claramente identificable.

Reglas:

- No desfijar al mover accidentalmente el cursor fuera de la superficie.
- Mostrar feedback visual.
- Persistir solo después de completar la operación.
- Si la aplicación sigue abierta, conservarla en la sección de aplicaciones abiertas.

### 6.9 Drag-and-drop de archivos

Permitir soltar archivos sobre una aplicación.

Utilizar el mecanismo estándar de drag-and-drop de Wayland.

Reglas:

- Respetar MIME types.
- Respetar las capacidades del destino.
- Validar el tipo de aplicación.
- Utilizar los datos transferidos por el protocolo.
- No acceder a rutas fuera de los permisos concedidos.
- No ejecutar archivos.
- No asumir que toda aplicación acepta cualquier archivo.

Si la aplicación está cerrada, solicitar su lanzamiento con los archivos como argumentos de datos, utilizando el mecanismo validado por el servicio de sesión.

La operación debe respetar los permisos y la semántica de la aplicación.

### 6.10 Scroll

Sin acción por defecto.

Motivo: evitar cambios de ventana accidentales.

No implementar scroll-to-switch como comportamiento predeterminado.

### 6.11 Abrir con GPU dedicada

Mostrar únicamente si:

- El hardware ofrece una GPU alternativa.
- Existe un mecanismo de offload compatible.
- El servicio de lanzamiento puede aplicar la configuración.
- La aplicación puede iniciarse con esa política.

La opción no debe aparecer como funcional en equipos sin soporte.

No modificar directamente frecuencias ni perfiles de GPU desde el Dock.

La decisión se delega al servicio de lanzamiento y al mecanismo de offload soportado.

## 7. Auto-hide

### 7.1 Modos

Tres modos:

- `always_visible`
- `intelligent`
- `always_hidden`

Valor inicial: `always_visible`.

### 7.2 Siempre visible

- Dock visible permanentemente.
- Reserva espacio mediante exclusive zone.
- Las ventanas respetan el espacio reservado.
- La reserva se actualiza cuando cambia el tamaño o posición.

### 7.3 Inteligente

El Dock se oculta cuando existe una ventana que invade la región de visibilidad del Dock.

La detección de solapamiento depende de la geometría de ventanas y del estado de outputs.

El compositor debe proporcionar al Shell información suficiente para calcular o confirmar el solapamiento.

No intentar inferir todas las ventanas mediante coordenadas del puntero.

Comportamiento:

- Sin solapamiento: visible.
- Solapamiento: ocultar después de un retraso configurable.
- El cursor entra en la zona de revelado: mostrar.
- El cursor sale: iniciar temporizador de ocultación.
- Una interacción activa mantiene el Dock visible.

Las ventanas fullscreen pueden activar una política específica para ocultar el Dock.

### 7.4 Siempre oculto

- Superficie no visible.
- Sin exclusive zone.
- No reservar espacio permanente.
- Revelar mediante interacción con el borde configurado.

El usuario puede mantener el Dock visible temporalmente durante una interacción.

### 7.5 Revelado por presión de borde

Umbral inicial: 100 ms.

El comportamiento requiere:

- Detección de entrada al borde.
- Confirmación de presión o permanencia.
- Compatibilidad con la posición del Dock.
- Prevención de revelados accidentales.
- Soporte para varios outputs.

No depender únicamente de que el puntero entre en una región de la superficie oculta.

El compositor y el Shell deben acordar cómo se detecta el borde cuando la superficie está oculta.

Puede utilizarse una superficie de revelado mínima o una capacidad específica del compositor, siempre que no cree conflictos con otras superficies y dispositivos.

### 7.6 Exclusive zone

La reserva de espacio se gestiona según el modo:

| Modo | Dock visible | Exclusive zone |
|---|---|---|
| Siempre visible | Sí | Activa |
| Inteligente | Sí | Activa |
| Inteligente | Oculto | Desactivada |
| Siempre oculto | No | Desactivada |

`reserve_area` no se considera por sí mismo un mecanismo de detección de solapamiento.

La gestión de exclusive zone debe respetar la semántica del protocolo Layer Shell y la implementación concreta de Smithay.

## 8. Magnificación

### 8.1 Configuración

Magnificación:

- Desactivada por defecto.
- Activable por usuario.
- Limitada a elementos próximos al cursor.
- Configurable por intensidad.
- Compatible con reduced motion.

### 8.2 Comportamiento

Cuando está activa:

- El icono bajo el cursor aumenta de tamaño.
- Los iconos vecinos reciben magnificación progresiva.
- El Dock mantiene su geometría y posición.
- Los elementos no deben saltar ni superponerse de forma inesperada.
- La animación debe ser reversible e interrumpible.

La magnificación no debe provocar que el Dock altere repetidamente su exclusive zone.

## 9. Miniaturas y previews

Las miniaturas se utilizan para seleccionar ventanas de una misma aplicación.

Reglas:

- Generación bajo demanda.
- Cache con límite de memoria.
- Invalidación al cambiar contenido o destruir la ventana.
- Frecuencia de actualización limitada.
- No capturar contenido protegido.
- No mostrar miniaturas de aplicaciones sensibles cuando la sesión esté bloqueada.
- No mantener buffers innecesarios.
- No generar miniaturas continuamente en reposo.

La integración debe respetar las políticas de captura de superficies y las capacidades del compositor.

Si una miniatura no está disponible, mostrar un placeholder con el título de la ventana.

## 10. Teclado y accesibilidad

### 10.1 Atajos

`Super+1` a `Super+9`:

- Si la favorita correspondiente no está abierta, solicitar lanzamiento.
- Si está abierta, activar la ventana o mostrar selector si hay varias.
- No modificar el orden de favoritas.
- No generar una nueva instancia si la política de aplicación indica que debe reutilizarse una existente.

Los atajos son una política global del compositor.

El Dock no captura directamente Super ni implementa atajos globales por su cuenta.

Debe existir una única fuente de verdad para el registro y resolución de atajos.

### 10.2 Navegación

- Tab y Shift+Tab para recorrer elementos.
- Flechas para desplazamiento según orientación.
- Enter para activar.
- Espacio para abrir el menú contextual cuando corresponda.
- Escape para cancelar interacciones.
- Atajos anunciados en tooltips y accesibilidad.

### 10.3 Accesibilidad

- Nombres accesibles para aplicaciones.
- Estado de ejecución anunciado.
- Número de ventanas anunciado.
- Estado de favorito anunciado.
- Indicadores de progreso accesibles.
- Focus visible.
- Navegación completa sin ratón.
- Objetivos táctiles adecuados.
- Escalado de texto.
- Compatibilidad con AccessKit/AT-SPI.
- Soporte para reduced motion y high contrast.

No depender únicamente de color, posición o animación para comunicar estados.

## 11. Configuración

Configuración conceptual:

```toml
[dock]
position = "bottom"
display_mode = "primary"
visibility = "always_visible"

pinned = [
  "org.example.Browser.desktop",
  "org.example.Files.desktop",
  "org.example.Terminal.desktop"
]

icon_size = 48
spacing = 10
margin = 12
floating = true

magnification = false
magnification_factor = 1.25

show_badges = true
show_progress = true
show_window_previews = true

show_other_workspaces = true
cycle_windows_on_click = false

edge_reveal_delay_ms = 100
hide_delay_ms = 300
```

Los valores son iniciales y deben validarse contra el sistema de tokens y los límites de diseño.

`settingsd` es el propietario de la persistencia y validación.

El Dock no escribe directamente el archivo de configuración.

## 12. Rendimiento

El Dock debe respetar `RENDERING_PERFORMANCE` y el presupuesto general de Shell.

Reglas:

- No renderizar frames cuando no haya cambios.
- No actualizar badges mediante polling continuo.
- No consultar ventanas en bucles de alta frecuencia.
- No generar miniaturas si no se necesitan.
- Cachear iconos.
- Cachear geometría.
- Evitar asignaciones repetidas en hover.
- Utilizar animaciones reversibles.
- Evitar efectos de blur cuando no estén disponibles.
- Limitar memoria de miniaturas y recursos gráficos.

La magnificación y las animaciones no deben provocar renderizado constante cuando el Dock está inactivo.

## 13. Seguridad y privacidad

- El Dock no ejecuta comandos arbitrarios.
- Las aplicaciones se resuelven mediante identificadores validados.
- Los Desktop Entries deben validarse antes de ejecutar acciones.
- Los datos de drag-and-drop se procesan respetando los permisos del protocolo.
- Las miniaturas respetan el estado de bloqueo.
- Los badges no muestran contenido sensible.
- Las operaciones de cierre se delegan al compositor.
- La Papelera utiliza la implementación del sistema.
- La opción de GPU dedicada utiliza un mecanismo de lanzamiento autorizado.
- Los eventos D-Bus no se consideran confiables solo por incluir un `app_id`.

## 14. Integración con otros componentes

| Componente | Responsabilidad |
|---|---|
| `settingsd` | Favoritas, orden, posición y preferencias |
| `parallax-compositor` | Toplevels, foco, geometría, activación, minimización y outputs |
| `Session1` | Lanzamiento de aplicaciones y operaciones de sesión |
| `notifyd` | Estado agregado de notificaciones |
| `px-ui` | Componentes visuales, menús, tooltips y accesibilidad |
| Gestor de archivos | Papelera y apertura de archivos |
| Servicio de lanzamiento | Desktop Entries, acciones y GPU offload |
| `Observatory` | Diagnóstico y métricas |

## 15. Pruebas

### 15.1 Funcionales

- Fijar aplicación.
- Desfijar aplicación.
- Reordenar favoritas.
- Reiniciar Shell y recuperar configuración.
- Abrir aplicación cerrada.
- Activar aplicación abierta.
- Minimizar mediante política interna.
- Agrupar varias ventanas.
- Seleccionar miniaturas.
- Cerrar todas las ventanas.
- Mostrar y limpiar badges.
- Mostrar progreso.
- Abrir Papelera.
- Soltar archivos.
- Ejecutar acciones de Desktop Entry.
- Abrir con GPU dedicada cuando esté soportado.

### 15.2 Auto-hide

- Siempre visible.
- Inteligente.
- Siempre oculto.
- Revelado por borde.
- Cambios de workspace.
- Ventanas fullscreen.
- Ventanas maximizadas.
- Ventanas flotantes.
- Múltiples outputs.
- Hotplug.
- Cambio de posición del Dock.

### 15.3 Persistencia

- Reiniciar Shell.
- Reiniciar sesión.
- Desinstalar aplicación fijada.
- Reinstalar aplicación.
- Aplicar cambios simultáneos de configuración.
- Interrumpir un drag-and-drop.
- Verificar ausencia de favoritos duplicados.

### 15.4 Accesibilidad

- Navegación por teclado.
- Focus visible.
- Screen reader.
- Texto escalado.
- High contrast.
- Reduced motion.
- RTL.
- Orientación horizontal y vertical.

### 15.5 Rendimiento

- 10, 20 y 50 aplicaciones abiertas.
- Varias ventanas por aplicación.
- Muchos iconos distintos.
- Badges actualizándose.
- Miniaturas.
- Magnificación.
- Cambios frecuentes de workspace.
- Dos o más outputs.

Verificar ausencia de polling innecesario, renderizado ocioso y crecimiento de memoria.

## 16. Criterios de aceptación

- [ ] Se integra dentro de `parallax-shell`.
- [ ] Utiliza Layer Shell.
- [ ] Soporta posiciones inferior, izquierda y derecha.
- [ ] Soporta modos primary, all y selected.
- [ ] Favoritas persistidas mediante `settingsd`.
- [ ] Aplicaciones abiertas obtenidas desde el compositor.
- [ ] Agrupación estable por identidad de aplicación.
- [ ] Indicadores de ejecución y número de ventanas.
- [ ] Badges integrados con `notifyd`.
- [ ] Compatibilidad de progreso mediante adaptador Unity.
- [ ] Lanzamiento mediante `Session1`.
- [ ] Activación y minimización mediante el compositor.
- [ ] Menú contextual.
- [ ] Acciones de Desktop Entry validadas.
- [ ] Drag-and-drop de archivos.
- [ ] Reordenamiento persistente.
- [ ] Auto-hide funcional.
- [ ] Revelado por presión de borde.
- [ ] Exclusive zone coherente con la visibilidad.
- [ ] Miniaturas con límites de memoria y privacidad.
- [ ] Magnificación opcional.
- [ ] Navegación por teclado.
- [ ] Atajos Super+1..9 gestionados por el compositor.
- [ ] Multi-monitor y hotplug.
- [ ] Sin renderizado ocioso innecesario.
- [ ] Sin operaciones privilegiadas directas.
- [ ] Pruebas de rendimiento y accesibilidad superadas.

## 17. Decisiones pendientes

1. Definir el contrato de identidad entre `app_id` y Desktop Entry ID.
2. Definir el contrato exacto de `Session1.Launch`.
3. Definir las operaciones privadas de activación y minimización.
4. Decidir si el clic sobre grupos utiliza selector o ciclo por defecto.
5. Definir el mecanismo de detección de solapamiento para auto-hide inteligente.
6. Definir el mecanismo de revelado por borde con la superficie oculta.
7. Definir la política de miniaturas y superficies protegidas.
8. Definir el contrato de progreso interno y compatibilidad Unity.
9. Definir el servicio de GPU offload.
10. Definir el comportamiento del Dock durante bloqueo y recuperación de Shell.

**Estado final: Propuesto. No aprobado. La implementación depende de los contratos de compositor, sesión, settingsd y notificaciones.**