# Modelo de eventos

- **Proyecto:** PARALLAX
- **Estado:** Propuesto — requiere implementación y validación
- **Fecha:** 2026-11-19
- **Ámbito:** Compositor, Shell, servicios, aplicaciones y widgets

## 1. Objetivo

Definir un modelo uniforme de comunicación basado en eventos para los componentes de PARALLAX.

El modelo debe permitir:

- Mantener una única fuente de verdad por dato.
- Comunicar cambios sin acoplar implementaciones.
- Recuperar el estado después de desconexiones o reinicios.
- Evitar eventos duplicados y ciclos de publicación.
- Controlar la frecuencia de eventos.
- Evitar que consumidores lentos degraden el sistema.
- Aplicar autorización y aislamiento entre componentes.
- Facilitar pruebas deterministas y observabilidad.

La arquitectura utilizará tres mecanismos principales:

| Dominio | Transporte principal | Responsabilidad |
|---|---|---|
| Escritorio | Wayland y protocolos PARALLAX | Superficies, ventanas, foco, workspaces y outputs |
| Servicios | D-Bus mediante zbus | Configuración, recursos, hardware y servicios del sistema |
| Interno del Shell | Bus tipado local | Comunicación entre componentes del Shell |

No se implementará un bus global de eventos que replique indiscriminadamente toda la información del sistema.

---

# 2. Principios fundamentales

## 2.1. Fuente única de verdad

Cada dato tiene un único propietario lógico.

El propietario:

- Mantiene el estado autoritativo.
- Valida las modificaciones.
- Incrementa su revisión.
- Publica los cambios.
- Proporciona snapshots.
- Gestiona las reglas de autorización.

Los consumidores mantienen copias de lectura, nunca versiones independientes del estado autoritativo.

Una modificación debe solicitarse al propietario mediante una operación explícita.

Ejemplo:

```text
Shell
  |
  | Settings1.Set("theme", "dark")
  v
settingsd
  |
  ├── valida
  ├── persiste
  ├── actualiza estado
  └── publica Settings1.Changed
           |
           ├── Shell
           ├── Settings UI
           └── otros consumidores autorizados
```

El Shell no debe modificar directamente su copia local y asumir que el cambio ha sido aceptado.

## 2.2. Snapshot + delta

Todo consumidor que necesite estado persistente debe poder obtener:

1. Un snapshot inicial.
2. Los cambios posteriores.
3. Una nueva sincronización después de detectar pérdida de eventos.

Un consumidor desconectado nunca debe asumir que conoce el estado actual.

Los eventos no son sustitutos del estado autoritativo.

## 2.3. Publicación unidireccional

Un componente no debe volver a publicar como propio un evento que simplemente recibió.

Ejemplo incorrecto:

```text
settingsd → Shell → settingsd → Shell
```

Ejemplo correcto:

```text
settingsd
   |
   └── SettingsChanged
          |
          ├── Shell actualiza su representación
          ├── Settings UI actualiza su representación
          └── Widgets autorizados reciben el cambio
```

Si un consumidor necesita modificar un valor, debe realizar una operación explícita sobre el propietario.

## 2.4. Coalescencia

Los eventos de alta frecuencia deben agruparse.

Valores iniciales:

| Tipo | Política |
|---|---|
| Cambios de geometría | Máximo una actualización visual por frame |
| Títulos de ventanas | Coalescencia de cambios intermedios |
| Métricas de recursos | 1 Hz por defecto |
| PSI | Evaluación periódica con histéresis |
| Movimiento del cursor | No replicar mediante D-Bus |
| Cambios de configuración | No coalescer operaciones persistentes distintas |
| Notificaciones | No descartar notificaciones válidas por coalescencia |

La coalescencia no debe eliminar eventos cuya semántica implique una operación o transición importante.

## 2.5. Orden y consistencia

Cada propietario debe mantener una secuencia monotónica de revisiones.

Los consumidores deben poder detectar:

- Eventos duplicados.
- Eventos fuera de orden.
- Revisiones ausentes.
- Reinicios del propietario.
- Cambios de generación.

No se garantiza un orden global entre diferentes propietarios.

La consistencia se garantiza dentro del dominio de cada propietario.

---

# 3. Identidad y versionado de eventos

Los contratos deben incluir un mecanismo de identificación y revisión.

Modelo conceptual:

```rust
struct EventEnvelope<T> {
    owner_id: OwnerId,
    epoch: EpochId,
    revision: u64,
    event_id: EventId,
    payload: T,
}
```

## 3.1. Campos

| Campo | Descripción |
|---|---|
| `owner_id` | Identifica al propietario del dato |
| `epoch` | Identifica la generación del servicio |
| `revision` | Número monotónico de modificación |
| `event_id` | Identificador único del evento |
| `payload` | Datos específicos del evento |

`epoch` cambia cuando el propietario inicia una nueva generación y no puede garantizar continuidad del estado anterior.

Los consumidores no deben comparar revisiones de generaciones diferentes.

Los timestamps pueden utilizarse para diagnóstico y medición, pero no deben sustituir la revisión para determinar el orden.

## 3.2. Versionado de contratos

Los protocolos D-Bus y Wayland deben tener versiones explícitas.

Ejemplo:

```text
org.parallax.Settings1
org.parallax.Resources1
org.parallax.Hardware1
org.parallax.Session1

parallax-shell-v1
parallax-workspace-v1
```

Los cambios incompatibles requieren una nueva versión del contrato.

Se debe documentar:

- Tipos de datos.
- Operaciones.
- Señales.
- Permisos.
- Reglas de orden.
- Reglas de reconexión.
- Comportamiento ante errores.

Los contratos D-Bus se definirán en `docs/architecture/dbus/`.

Los protocolos Wayland se definirán en `core/proto/`.

---

# 4. Sincronización: snapshot + delta

## 4.1. Procedimiento obligatorio

El propietario debe coordinar la obtención del snapshot y la suscripción.

Procedimiento:

1. El consumidor solicita una suscripción.
2. El propietario registra la suscripción en un punto de sincronización.
3. El propietario devuelve el snapshot y su revisión.
4. Se entregan los eventos posteriores a esa revisión.
5. El consumidor aplica los deltas en orden.
6. Si detecta una discontinuidad, solicita resincronización.

Ejemplo:

```text
Consumer                 Owner
   |                        |
   |------ Subscribe ------>|
   |                        |
   |<-- Snapshot rev=40 ----|
   |<-- Delta rev=41 -------|
   |<-- Delta rev=42 -------|
   |<-- Delta rev=43 -------|
```

La implementación debe impedir que una modificación ocurra entre la captura del snapshot y el registro de la suscripción sin que quede reflejada en alguno de los dos.

## 4.2. Detección de pérdida

Si el consumidor recibe:

```text
revision = 41
revision = 42
revision = 45
```

Debe detectar que faltan las revisiones 43 y 44.

Acción:

1. Suspender la aplicación de deltas del dominio afectado.
2. Solicitar un nuevo snapshot.
3. Sustituir el estado local.
4. Continuar desde la nueva revisión.

No debe intentar reconstruir arbitrariamente el estado perdido.

## 4.3. Reinicio del propietario

Si cambia `epoch`:

- Invalidar el estado de sincronización anterior.
- Obtener un nuevo snapshot.
- Actualizar la generación local.
- Reanudar la suscripción.

Un consumidor no debe asumir que el servicio conserva su estado después de reiniciarse.

## 4.4. D-Bus

D-Bus no proporciona almacenamiento persistente ni reproducción automática de señales perdidas.

Por tanto, los servicios deben ofrecer métodos de consulta de estado y una revisión que permita recuperar el estado después de una desconexión.

No es obligatorio implementar un sistema de replay histórico para todos los servicios.

Los datos actuales y la revisión son suficientes para los dominios de estado reemplazable.

---

# 5. Catálogo de eventos

## 5.1. Compositor

El compositor es el propietario del estado de las ventanas y de la sesión gráfica.

| Evento | Fuente | Transporte | Consumidores | Política |
|---|---|---|---|---|
| `toplevel_added` | Compositor | `parallax-shell-v1` | Shell | Inmediata |
| `toplevel_changed` | Compositor | `parallax-shell-v1` | Shell | Coalescida |
| `toplevel_removed` | Compositor | `parallax-shell-v1` | Shell | Inmediata |
| `focus_changed` | Compositor | Wayland / interfaz de sesión | Shell, resourced | Por cambio |
| `workspace_changed` | Compositor | `parallax-workspace-v1` | Shell | Por cambio |
| `layout_changed` | Compositor | `parallax-workspace-v1` | Shell | Coalescida |
| `output_added` | Compositor | `wl_output` y xdg-output | Shell, Settings | Por cambio |
| `output_removed` | Compositor | `wl_output` y xdg-output | Shell, Settings | Por cambio |

El protocolo debe permitir consultar el estado actual de las ventanas, workspaces y outputs.

Los consumidores no deben mantener identificadores de superficies como referencias permanentes después de su destrucción.

## 5.2. Foco y recursos

El compositor es el propietario del foco de ventanas.

`parallax-resourced` necesita conocer qué aplicación está activa para correlacionar actividad y consumo.

Se permitirá un contrato de sesión, por ejemplo:

```text
org.parallax.Session1.FocusChanged
```

La señal D-Bus será una representación del estado publicado por el compositor, no una segunda fuente de verdad.

El Shell utilizará el transporte Wayland correspondiente.

`resourced` utilizará el contrato de sesión.

No se publicarán datos de foco a aplicaciones o widgets que no los necesiten.

## 5.3. Configuración

| Evento | Fuente | Transporte | Consumidores |
|---|---|---|---|
| `setting_changed` | settingsd | `Settings1.Changed` | Shell, Settings, consumidores autorizados |
| `theme_changed` | settingsd | `Settings1.Changed` | Shell y componentes visuales |
| `display_settings_changed` | settingsd | `Settings1.Changed` | Shell y compositor mediante API de configuración |

`settingsd` es el propietario de la configuración persistente.

El compositor mantiene únicamente el estado operativo necesario para ejecutar las políticas visuales.

Las configuraciones que afectan al compositor deben aplicarse mediante operaciones explícitas y validadas.

## 5.4. Recursos

| Evento | Fuente | Transporte | Consumidores | Frecuencia |
|---|---|---|---|---|
| `pressure_changed` | resourced | `Resources1.PressureChanged` | Shell, widgets autorizados | Por transición |
| `memory_changed` | resourced | `Resources1` | Shell, Observatory | 1 Hz |
| `cpu_changed` | resourced | `Resources1` | Shell, Observatory | 1 Hz |
| `gpu_changed` | resourced | `Resources1` | Shell, Observatory | Según backend |
| `power_policy_changed` | resourced | `Resources1` | Shell, servicios autorizados | Por cambio |

Los valores de CPU, memoria y GPU no deben emitirse como eventos independientes a frecuencias elevadas por defecto.

Los consumidores pueden solicitar una consulta puntual para obtener valores más recientes.

La política de presión utilizará:

- Umbrales configurables.
- Histéresis.
- Tiempo mínimo de permanencia.
- Prevención de oscilaciones.
- Identificación de la fuente de presión.

Los dos segundos de histéresis son un valor inicial de referencia, no una constante universal para todos los tipos de presión.

## 5.5. Hardware

| Evento | Fuente | Transporte | Consumidores |
|---|---|---|---|
| `device_added` | hardwared | `Hardware1` | Settings, Observatory |
| `device_removed` | hardwared | `Hardware1` | Settings, Observatory |
| `device_changed` | hardwared | `Hardware1` | Settings, Observatory |

`hardwared` consume udev y otras fuentes del sistema.

No debe duplicar sin necesidad la funcionalidad de UPower, NetworkManager o BlueZ.

Cada dispositivo tendrá un identificador estable cuando el hardware y la fuente lo permitan.

Los identificadores temporales deben distinguirse de los identificadores persistentes.

## 5.6. Energía

UPower será la fuente de verdad para los datos que administra.

| Evento | Fuente | Transporte | Consumidores |
|---|---|---|---|
| `battery_changed` | UPower | D-Bus UPower | Shell, resourced |
| `power_supply_changed` | UPower | D-Bus UPower | Shell, Settings |
| `battery_warning` | UPower / política PARALLAX | D-Bus | Shell |

`resourced` puede utilizar estos datos para políticas de recursos, pero no debe duplicar el estado autoritativo de batería.

## 5.7. Red

NetworkManager será el propietario del estado de conectividad.

El Shell y Settings podrán consumir directamente sus APIs D-Bus estándar.

PARALLAX solo debe introducir una abstracción propia si existe una necesidad concreta de políticas, permisos o normalización.

No crear un evento duplicado de red en `hardwared`.

## 5.8. Audio

PipeWire será el propietario del grafo y estado de audio.

Los componentes que necesiten información detallada podrán consumir la API correspondiente mediante `pipewire-rs`.

No es necesario convertir todos los cambios del grafo de audio en señales D-Bus.

El Shell puede mantener un adaptador de lectura para volumen y dispositivos predeterminados.

## 5.9. Notificaciones

`notifyd` administrará las notificaciones recibidas y las políticas propias de PARALLAX.

Debe considerar la compatibilidad con `org.freedesktop.Notifications`.

| Evento | Fuente | Transporte | Consumidores |
|---|---|---|---|
| `notification_created` | notifyd | D-Bus | Shell |
| `notification_updated` | notifyd | D-Bus | Shell |
| `notification_closed` | notifyd | D-Bus | Shell |

La persistencia y expiración deben seguir políticas explícitas.

Una notificación no debe descartarse simplemente porque un consumidor UI esté temporalmente desconectado, cuando su política requiera conservarla.

---

# 6. Bus interno de parallax-shell

El Shell utilizará un bus tipado local.

Su propósito es distribuir cambios entre:

- Topbar.
- Dock.
- Launcher.
- Control Center.
- Notificaciones.
- Widgets.
- Indicadores.
- Componentes visuales internos.

## 6.1. Implementación

La implementación inicial podrá utilizar:

- `tokio::sync::broadcast` para eventos efímeros.
- `tokio::sync::watch` para estado actual reemplazable.
- `tokio::sync::mpsc` para comandos y colas dirigidas a consumidores.

No utilizar un único canal para todos los tipos de información.

### broadcast

Adecuado para eventos que pueden distribuirse a múltiples consumidores y cuya pérdida se puede detectar.

Ejemplos:

- Cambio de foco.
- Ventana creada.
- Cambio de workspace.
- Notificación de actualización visual.

### watch

Adecuado para estados en los que importa el valor más reciente, no cada transición intermedia.

Ejemplos:

- Tema actual.
- Volumen actual.
- Estado de conectividad.
- Nivel de presión.
- Configuración visual actual.

### mpsc

Adecuado para comandos o mensajes dirigidos a un consumidor específico.

Ejemplos:

- Solicitud de recarga.
- Comando de actualización de un componente.
- Solicitud de resincronización.

## 6.2. Capacidad y backpressure

La capacidad inicial de `broadcast` será 64 mensajes por tema.

Este valor es provisional y debe validarse mediante benchmarks.

Cuando un consumidor reciba `Lagged`:

1. No continuar aplicando eventos como si el estado estuviera completo.
2. Invalidar su estado derivado.
3. Solicitar o reconstruir el snapshot.
4. Reanudar el procesamiento.

Los consumidores lentos no deben bloquear el hilo de UI ni retrasar la publicación de otros eventos.

No se permitirá crecimiento ilimitado de colas.

## 6.3. Separación por temas

El bus tendrá temas tipados.

Ejemplo conceptual:

```rust
enum ShellEvent {
    Window(WindowEvent),
    Workspace(WorkspaceEvent),
    Focus(FocusEvent),
    Settings(SettingsEvent),
    Resources(ResourcesEvent),
    Notification(NotificationEvent),
}
```

La implementación final puede utilizar tipos genéricos por dominio en lugar de un único enum.

La separación por temas debe permitir:

- Suscripción selectiva.
- Límites independientes.
- Coalescencia específica.
- Métricas por consumidor.
- Recuperación independiente.

## 6.4. Widgets externos

Los widgets de terceros no recibirán acceso directo al bus interno.

Cada widget declarará en su manifiesto los temas que necesita.

Ejemplo:

```toml
[widget]
id = "system-monitor"
version = "1.0"

[subscriptions]
topics = [
    "resources.memory",
    "resources.cpu",
]

[permissions]
network = false
filesystem = "none"
```

El manifiesto expresa las necesidades del widget, pero no concede permisos automáticamente.

El Shell o el gestor de widgets debe:

1. Validar el manifiesto.
2. Comprobar la política de seguridad.
3. Autorizar las capacidades solicitadas.
4. Crear una suscripción limitada.
5. Entregar únicamente los campos permitidos.
6. Revocar el acceso al finalizar el proceso.

Los widgets no podrán suscribirse libremente a eventos de foco, aplicaciones, archivos, sesión o hardware sensible.

Los datos se expondrán mediante un protocolo de comunicación entre procesos limitado y versionado.

---

# 7. Política de publicación

Todo propietario debe respetar estas reglas:

1. Validar el cambio antes de modificar el estado.
2. Aplicar la modificación en el propietario.
3. Incrementar la revisión.
4. Publicar el evento correspondiente.
5. Registrar métricas y errores de publicación.

Una modificación rechazada no debe generar un evento de éxito.

Las operaciones de escritura deben devolver un resultado explícito.

Cuando una operación es persistente, la semántica de éxito debe indicar si el dato ya está guardado o si solamente se encuentra aplicado en memoria.

No se debe afirmar que un cambio persistió en disco antes de completar la operación de persistencia correspondiente.

---

# 8. Seguridad y aislamiento

## 8.1. Autorización

Cada interfaz debe definir:

- Quién puede consultar.
- Quién puede suscribirse.
- Quién puede modificar.
- Qué datos son visibles.
- Qué operaciones requieren polkit.

Los consumidores no deben recibir información sensible por el simple hecho de conectarse al bus.

## 8.2. Identidad del emisor

No confiar en un campo `owner_id` enviado por un cliente como prueba de identidad.

La identidad debe verificarse mediante las credenciales del transporte y las políticas del servicio.

El emisor de cada evento debe ser el propietario real del dato.

## 8.3. Widgets

Los widgets externos deben ejecutarse fuera del proceso principal del Shell.

Se aplicarán límites de:

- CPU.
- Memoria.
- Tiempo de ejecución.
- Acceso a archivos.
- Red.
- Suscripciones.
- Frecuencia de actualización.

El fallo de un widget no debe provocar la caída del Shell.

---

# 9. Observabilidad

El modelo de eventos debe poder diagnosticarse.

Métricas iniciales:

| Métrica | Propósito |
|---|---|
| `events_published_total` | Eventos publicados |
| `events_delivered_total` | Eventos entregados |
| `events_dropped_total` | Eventos descartados |
| `subscriber_lag_total` | Consumidores retrasados |
| `snapshot_requests_total` | Solicitudes de snapshot |
| `resync_total` | Resincronizaciones |
| `event_processing_duration` | Tiempo de procesamiento |
| `event_queue_depth` | Tamaño de las colas |

Los logs deben incluir:

- Componente.
- Dominio.
- Evento.
- Revisión.
- Generación.
- Resultado.
- Correlation ID cuando corresponda.

No registrar datos personales, contenido de notificaciones ni información sensible de aplicaciones sin una justificación explícita.

La telemetría no debe introducir una carga significativa en los eventos de alta frecuencia.

---

# 10. Pruebas obligatorias

| ID | Prueba | Resultado esperado |
|---|---|---|
| EVT-001 | Consumidor se conecta | Recibe snapshot válido |
| EVT-002 | Cambio durante suscripción | No se pierde el delta |
| EVT-003 | Evento duplicado | No corrompe el estado |
| EVT-004 | Evento fuera de orden | Se detecta inconsistencia |
| EVT-005 | Revisión ausente | Se solicita resincronización |
| EVT-006 | Reinicio del propietario | Se invalida el epoch anterior |
| EVT-007 | Consumidor lento | No bloquea al propietario