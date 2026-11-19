# Arquitectura IPC y contratos

- **Proyecto:** PARALLAX
- **Estado:** Propuesto — requiere validación y aprobación
- **Fecha:** 2026-11-19
- **Ámbito:** Comunicación entre procesos, protocolos, contratos y transferencia de datos

# 1. Objetivo

Definir una arquitectura de comunicación entre los componentes de PARALLAX que sea:

- Predecible y eficiente.
- Segura por defecto.
- Versionable.
- Independiente de las implementaciones internas.
- Compatible con el ecosistema Linux.
- Recuperable ante desconexiones y reinicios.
- Adecuada para un entorno de escritorio con recursos limitados.

La arquitectura utilizará transportes específicos según el propósito de la comunicación.

No se implementará un bus universal ni se permitirá que cada componente invente su propio mecanismo de comunicación.

---

# 2. Transportes y responsabilidades

| Transporte | Uso | Justificación |
|---|---|---|
| Wayland | Superficies, ventanas, input, outputs, workspaces y operaciones del escritorio | Integración directa con el compositor |
| D-Bus (`zbus`) | Servicios, configuración, hardware, recursos, actualizaciones y notificaciones | Interoperabilidad, activación, introspección y autorización |
| Unix file descriptors + `memfd` | Transferencia de datos grandes y memoria compartida cuando esté justificado | Evita serializaciones y copias innecesarias |
| Archivos | Configuración persistente, perfiles y datos declarativos | Persistencia, recuperación y auditoría |
| APIs estándar Linux | Funcionalidades delegadas al ecosistema | Evita duplicar servicios existentes |

## 2.1. Reglas generales

- No crear protocolos IPC propios sin una necesidad documentada.
- No utilizar HTTP local para comunicación entre componentes del sistema.
- No utilizar D-Bus para transmitir datos por frame.
- No utilizar D-Bus como sustituto de Wayland para gestionar superficies.
- No transmitir buffers gráficos grandes mediante serialización D-Bus.
- No permitir acceso directo a archivos de configuración de otros servicios.
- No utilizar memoria compartida sin un contrato de sincronización y propiedad.
- No duplicar APIs existentes del ecosistema sin una justificación técnica.

La prohibición de sockets ad hoc se refiere a protocolos improvisados sin especificación, versionado, seguridad ni justificación. No prohíbe mecanismos estándar basados en Unix Domain Sockets cuando una dependencia o protocolo documentado los requiera.

---

# 3. Wayland

Wayland será el transporte principal para todo lo relacionado con el escritorio.

## 3.1. Funcionalidades

- Gestión de superficies.
- Ventanas y toplevels.
- Input.
- Foco.
- Workspaces.
- Outputs.
- Geometría.
- Estados de ventanas.
- Activación de aplicaciones.
- Integración con paneles y docks.
- Superficies especiales del Shell.
- Transferencia de buffers gráficos.

## 3.2. Prioridad de protocolos

Antes de desarrollar un protocolo privado, se debe evaluar si existe uno estándar o ampliamente adoptado que cubra el requisito.

Orden de preferencia:

1. Wayland core.
2. Protocolos estandarizados o mantenidos por freedesktop.
3. Protocolos del ecosistema wlroots cuando sean adecuados.
4. Protocolos privados PARALLAX.

La adopción de protocolos externos dependerá de su disponibilidad real en la versión de Smithay seleccionada.

No se debe asumir que todos los protocolos existentes están implementados por Smithay.

## 3.3. Protocolos que se deben evaluar

| Protocolo | Propósito |
|---|---|
| `xdg-shell` | Superficies de aplicaciones |
| `xdg-decoration` | Decoraciones de ventanas |
| `xdg-activation` | Activación de aplicaciones |
| `ext-foreign-toplevel-list-v1` | Enumeración de toplevels |
| `ext-workspace-v1` | Workspaces |
| `wlr-layer-shell` | Paneles, dock y superficies del Shell |
| `xdg-output` | Información extendida de outputs |
| `fractional-scale` | Escalado fraccional |
| `viewporter` | Transformación de superficies |
| `linux-dmabuf` | Buffers gráficos |
| `linux-explicit-synchronization` | Sincronización explícita de buffers |
| `presentation-time` | Información de presentación |
| `pointer-constraints` | Restricciones de puntero |
| `relative-pointer` | Movimiento relativo |
| `text-input` | Entrada de texto |

La matriz de compatibilidad debe indicar:

- Versión del protocolo.
- Estado de mantenimiento.
- Disponibilidad en Smithay.
- Compatibilidad con aplicaciones.
- Necesidad de implementación propia.
- Dependencias del compositor.

---

# 4. Protocolos privados PARALLAX

Los protocolos privados se utilizarán únicamente para capacidades que no estén cubiertas adecuadamente por protocolos existentes.

Su implementación debe permanecer aislada en `core/proto`.

## 4.1. `parallax-shell-v1`

Propósito: proporcionar al Shell la información y operaciones específicas que necesita para representar y controlar el escritorio.

Consumidor autorizado: `parallax-shell`.

Operaciones conceptuales:

```text
get_toplevel_list()
activate(toplevel)
close(toplevel)
minimize(toplevel)
move_to_workspace(toplevel, workspace)
request_thumbnail(toplevel, max_size)
```

Eventos:

```text
toplevel_added
toplevel_changed
toplevel_removed
focus_changed
fullscreen_changed
output_layout_changed
```

Información de toplevel:

```text
id
app_id
title
state
workspace
output
pid
```

### Reglas

- El identificador de toplevel será opaco y administrado por el compositor.
- Los clientes no podrán utilizar identificadores después de recibir su destrucción.
- El PID es informativo y no constituye una identidad de seguridad.
- El compositor valida todas las operaciones.
- El Shell no podrá manipular directamente superficies de aplicaciones.
- El protocolo debe ofrecer una forma de obtener el estado inicial y recuperar sincronización.

### Autorización del Shell

No se confiará en el nombre del proceso ni en el `app_id` declarado por un cliente.

El compositor debe autenticar al cliente mediante el mecanismo de sesión establecido.

La conexión privilegiada del Shell debe obtenerse mediante un mecanismo controlado de sesión, no mediante una conexión disponible indiscriminadamente a cualquier cliente Wayland.

## 4.2. Zonas reservadas

Se debe evaluar primero `wlr-layer-shell` y sus mecanismos de exclusive zone.

No crear `reserve_area()` si el protocolo existente cubre el comportamiento requerido.

La política de auto-hide y las transiciones de reserva de espacio pertenecen al compositor y al Shell, no a un servicio D-Bus.

## 4.3. Miniaturas

La generación de miniaturas debe estar sujeta a políticas de privacidad y consumo de recursos.

El protocolo podrá solicitar una miniatura con dimensiones máximas.

La transferencia utilizará un mecanismo apropiado:

- Buffers Wayland.
- DMA-BUF cuando sea compatible.
- SHM para casos donde corresponda.

No se transmitirán imágenes grandes como arrays D-Bus.

El compositor debe poder rechazar solicitudes por:

- Tamaño excesivo.
- Frecuencia elevada.
- Cliente no autorizado.
- Superficie no disponible.
- Restricciones de privacidad.

La generación de miniaturas no debe bloquear el ciclo de presentación.

## 4.4. `parallax-workspace-v1`

Propósito: operaciones y estado específico de workspaces que no estén cubiertos por `ext-workspace-v1`.

Operaciones:

```text
get_workspaces()
create_workspace(name, output)
remove_workspace(workspace)
rename_workspace(workspace, name)
activate_workspace(workspace)
move_toplevel(toplevel, workspace)
set_layout_mode(workspace, mode)
```

Modos iniciales:

```text
floating
tiling
```

Eventos:

```text
workspace_added
workspace_removed
workspace_changed
workspace_activated
layout_changed
```

La semántica de los identificadores, el orden y las operaciones debe permanecer coherente con `px-wm`.

No se permitirá que el protocolo implemente una segunda lógica de gestión de ventanas independiente del WM.

## 4.5. Versionado Wayland

Los protocolos privados tendrán:

- XML versionado.
- Versión de interfaz.
- Documentación de operaciones.
- Reglas de autorización.
- Reglas de sincronización.
- Pruebas de compatibilidad.

Las extensiones compatibles pueden incrementar su versión manteniendo las operaciones anteriores.

Los cambios incompatibles deben diseñarse con una nueva interfaz o un mecanismo de migración explícito.

No se asumirá que incrementar el número de versión permite cambiar libremente la semántica de operaciones existentes.

---

# 5. D-Bus

D-Bus será el transporte de control y comunicación de servicios.

Implementación Rust: `zbus`.

Se utilizarán:

- Bus de sesión para servicios vinculados a la sesión gráfica.
- Bus del sistema para servicios con responsabilidades del sistema.

## 5.1. Convenciones de nombres

Nombre de servicio:

```text
org.parallax.<Service>1
```

Interfaz:

```text
org.parallax.<Service>1
```

Ruta de objeto:

```text
/org/parallax/<Service>1
```

Ejemplo:

```text
Bus name:   org.parallax.Settings1
Object:     /org/parallax/Settings1
Interface:  org.parallax.Settings1
```

Los nombres deben ser coherentes con las convenciones D-Bus.

Las interfaces estándar de freedesktop conservarán sus nombres originales.

## 5.2. Interfaces principales

| Interfaz | Propietario | Bus | Responsabilidad |
|---|---|---|---|
| `Session1` | Compositor / gestor de sesión | Sesión | Operaciones de sesión |
| `Settings1` | settingsd | Sesión | Configuración persistente |
| `org.freedesktop.Notifications` | notifyd | Sesión | Notificaciones estándar |
| `Notifications1` | notifyd | Sesión | Extensiones PARALLAX |
| `Hardware1` | hardwared | Sistema | Inventario de dispositivos |
| `Resources1` | resourced | Sesión | Métricas y políticas de recursos |
| `Update1` | updated | Sistema | Actualizaciones |
| `Recovery1` | recoveryd | Sistema | Recuperación |
| `Diagnostics1` | diagd | Sistema | Diagnósticos |

La implementación final debe definir explícitamente los nombres de bus, objetos, interfaces y permisos de cada servicio.

---

# 6. Contratos D-Bus

## 6.1. Session1

Propietario: compositor / gestor de sesión.

Bus: sesión.

Métodos:

```text
Launch(app_id)
Lock()
Logout()
```

Señales:

```text
FocusChanged(app_id, pid, cgroup)
VisibilityChanged(...)
Locked()
Unlocked()
```

### Reglas

- `Launch` debe utilizar identificadores de aplicación registrados.
- No debe permitir ejecutar comandos arbitrarios mediante una cadena proporcionada por el cliente.
- Los argumentos adicionales deben validarse mediante un contrato específico y no convertirse en una vía de ejecución arbitraria.
- La activación gráfica utilizará `xdg-activation` cuando corresponda.
- El token de activación no debe considerarse una credencial de autorización.
- `Lock` debe activar el mecanismo seguro de bloqueo.
- `Logout` debe integrarse con el gestor de sesión y logind.

La enumeración y gestión detallada de outputs pertenece principalmente a Wayland. No duplicarla en D-Bus salvo que un consumidor de servicios necesite una consulta específica.

## 6.2. Settings1

Propietario: settingsd.

Bus: sesión.

Métodos:

```text
Get(schema, key)
Set(schema, key, value)
Reset(schema, key)
ListSchemas()
GetSnapshot(schema)
```

Señales:

```text
Changed(schema, key, value, revision)
SchemaChanged(schema)
```

### Reglas

- Cada configuración debe pertenecer a un esquema.
- Los valores deben tener tipos definidos.
- Las claves desconocidas deben rechazarse.
- Las modificaciones deben validarse.
- Las escrituras deben ser atómicas.
- Debe conservarse una versión recuperable.
- Las revisiones deben permitir detectar cambios.
- Los consumidores no pueden modificar archivos directamente.

`settingsd` será el único propietario lógico de la configuración PARALLAX.

Los archivos TOML serán la representación persistente, no una interfaz de escritura libre para todos los procesos.

La modificación manual de archivos externos se detectará mediante un mecanismo controlado de reconciliación y validación, sin sobrescribir silenciosamente cambios del usuario.

## 6.3. Notifications

Propietario: notifyd.

Bus: sesión.

Interfaz estándar:

```text
org.freedesktop.Notifications
```

Extensiones:

```text
org.parallax.Notifications1
```

Métodos adicionales:

```text
ListHistory()
Dismiss(notification_id)
SetDnd(enabled)
```

Señales:

```text
Added(...)
Removed(...)
DndChanged(enabled)
```

La interfaz estándar se implementará respetando su semántica original.

Las extensiones no deben romper la interoperabilidad con aplicaciones Linux existentes.

## 6.4. Hardware1

Propietario: hardwared.

Bus: sistema.

Métodos:

```text
ListDevices()
GetDevice(device_id)
GetDeviceHealth(device_id)
GetRecommendedDrivers(device_id)
```

Señales:

```text
DeviceAdded(...)
DeviceRemoved(...)
DeviceChanged(...)
DeviceHealthChanged(...)
```

Las consultas deben respetar las políticas de privacidad.

No exponer números de serie ni información sensible sin necesidad.

## 6.5. Resources1

Propietario: resourced.

Bus: sesión.

Métodos:

```text
Snapshot()
GetPolicy()
SetPolicy(policy)
GetAppClass(app_id)
SetAppClass(app_id, class)
```

Señales:

```text
PressureChanged(level, revision)
AppStateChanged(app_id, state)
MetricsChanged(snapshot)
```

Las métricas ordinarias utilizarán una frecuencia limitada.

### Transferencia de métricas

No se utilizará `SubscribeMetrics() → memfd` como contrato inicial.

Primera implementación:

- `Snapshot()` para obtener el estado actual.
- Señales D-Bus para cambios relevantes.
- Frecuencia configurable y limitada para métricas periódicas.

Si las pruebas demuestran que D-Bus genera una carga significativa, se evaluará un canal de memoria compartida.

Ese canal deberá especificar:

- Estructura del buffer.
- Tamaño máximo.
- Versión del formato.
- Identidad del productor.
- Índices de lectura y escritura.
- Sincronización.
- Comportamiento ante desbordamiento.
- Recuperación tras desconexión.
- Ciclo de vida del descriptor.

## 6.6. Update1

Propietario: updated.

Bus: sistema.

Métodos:

```text
Check()
Download()
Apply()
Rollback()
GetState()
```

Señales:

```text
Progress(job_id, percentage, stage)
StateChanged(state)
UpdateAvailable(...)
```

Las operaciones que puedan tardar más que el presupuesto normal de una llamada deben ejecutarse como trabajos.

No se debe mantener una llamada D-Bus bloqueada durante toda una actualización.

Las operaciones privilegiadas estarán sujetas a autorización y políticas de actualización.

## 6.7. Recovery1

Propietario: recoveryd.

Bus: sistema.

Métodos:

```text
ListSnapshots()
CreateSnapshot()
RestoreSnapshot(snapshot_id)
ResetUserShell()
GetRecoveryState()
```

Señales:

```text
SnapshotCreated(snapshot_id)
RecoveryStateChanged(state)
```

Las operaciones de restauración requieren autorización explícita.

El servicio debe verificar que el snapshot existe y es recuperable antes de iniciar una restauración.

No debe eliminar automáticamente snapshots de recuperación por una solicitud no validada.

## 6.8. Diagnostics1

Propietario: diagd.

Bus: sistema.

Métodos:

```text
RunCheck(check_id)
Collect()
GetReport(report_id)
```

Señales:

```text
CheckStarted(job_id)
CheckFinished(job_id, result)
```

La entrega de informes grandes podrá utilizar transferencia de descriptores de archivo.

El contenido de los informes debe respetar las políticas de privacidad.

---

# 7. Transferencia de memoria y descriptores

## 7.1. Uso de memfd

`memfd` se reservará para datos cuyo tamaño o frecuencia justifique evitar la serialización tradicional.

Casos potenciales:

- Históricos de métricas.
- Miniaturas.
- Datos binarios.
- Buffers compartidos.
- Resultados de diagnóstico grandes.

No se utilizará para configuraciones pequeñas, operaciones ordinarias o eventos de control.

## 7.2. Transferencia de FD

Los descriptores se transferirán mediante mecanismos compatibles con el transporte:

- Unix FD passing en D-Bus.
- Mecanismos de buffers de Wayland cuando corresponda.
- Unix Domain Sockets documentados cuando el contrato lo requiera.

Un descriptor recibido no debe considerarse confiable automáticamente.

El receptor debe validar:

- Tipo de recurso.
- Tamaño.
- Permisos.
- Sellos aplicados.
- Formato.
- Versión.
- Propietario lógico.

## 7.3. Sellos

Los buffers inmutables utilizarán sellos adecuados de `memfd`, como:

- `F_SEAL_WRITE`.
- `F_SEAL_GROW`.
- `F_SEAL_SHRINK`.
- `F_SEAL_SEAL`.

Los buffers compartidos mutables requieren un modelo distinto: no se deben aplicar sellos que impidan las escrituras legítimas del productor.

El uso de sellos dependerá del tipo de buffer y de su ciclo de vida.

## 7.4. Memoria compartida de métricas

Si se adopta un ring buffer, debe especificarse:

- Cabecera.
- Versión.
- Tamaño.
- Índice de escritura.
- Índice de lectura.
- Secuencia.
- Estado de productor.
- Política de overflow.
- Orden de memoria.
- Sincronización entre procesos.

No se permitirá compartir estructuras Rust dependientes de layout interno sin una representación binaria estable.

---

# 8. Archivos y configuración persistente

Ubicación:

```text
$XDG_CONFIG_HOME/parallax/
├── settings/
│   ├── appearance.toml
│   ├── display.toml
│   ├── behavior.toml
│   └── power.toml
└── shell/
    └── shell.toml
```

La estructura definitiva se definirá en `specifications/configuration.md`.

## 8.1. Reglas

- Cada archivo tiene un propietario lógico.
- `settingsd` controla las escrituras de configuración.
- Los archivos se validan mediante esquemas.
- Las escrituras son atómicas.
- Se conserva una copia recuperable.
- Los valores desconocidos se manejan según la política de compatibilidad.
- Las configuraciones inválidas no deben bloquear el arranque.
- No se almacenan secretos en texto plano.

## 8.2. Escritura atómica

Procedimiento:

1. Crear archivo temporal en el mismo sistema de archivos.
2. Escribir el contenido completo.
3. Validar el contenido.
4. Aplicar permisos.
5. Sincronizar el archivo cuando corresponda.
6. Renombrar atómicamente.
7. Sincronizar el directorio cuando la durabilidad lo requiera.
8. Publicar el evento de configuración actualizada.

Un evento de éxito debe emitirse solamente cuando la operación haya alcanzado el estado de persistencia definido por el contrato.

## 8.3. Separación de configuración

Se distinguirán:

- Configuración del usuario.
- Configuración del sistema.
- Valores predeterminados de distribución.
- Perfiles de hardware.
- Estado temporal de sesión.

No se mezclarán configuraciones de sistema y usuario en un único directorio.

---

# 9. Seguridad y autorización

## 9.1. Clasificación de operaciones

No todas las operaciones modificadoras requieren polkit.

Se distinguen:

| Categoría | Ejemplo | Autorización |
|---|---|---|
| Usuario | Cambiar tema | Propietario de sesión |
| Usuario | Cambiar preferencias del Shell | Propietario de sesión |
| Sesión | Bloquear sesión | Política de sesión |
| Sistema | Instalar actualizaciones | Polkit |
| Sistema | Restaurar snapshot del sistema | Polkit |
| Hardware | Modificar configuración privilegiada | Polkit o política específica |
| Diagnóstico | Lectura de información pública | Política del servicio |
| Diagnóstico | Operación privilegiada | Polkit o autorización específica |

Las operaciones deben seguir el principio de mínimo privilegio.

## 9.2. Polkit

Las operaciones privilegiadas se implementarán mediante acciones polkit explícitas.

Cada acción debe definir:

- Identificador.
- Descripción.
- Privilegios requeridos.
- Contexto de sesión.
- Reglas de autorización.
- Posibilidad de autenticación.
- Auditoría.

No se debe utilizar polkit para operaciones que el propietario de la sesión ya está autorizado a realizar.

## 9.3. Validación

Todo contrato debe validar:

- Tipos.
- Tamaños.
- Rutas.
- Identificadores.
- Enumeraciones.
- Valores numéricos.
- Límites de frecuencia.
- Permisos.

No confiar en que los clientes son componentes propios de PARALLAX.

Las aplicaciones de terceros pueden interactuar con interfaces públicas.

---

# 10. Contratos y errores

## 10.1. Tipos

Los valores deben ser explícitos y tipados.

`a{sv}` se permitirá únicamente cuando el contrato necesite extensibilidad declarada.

No se utilizará como sustituto de un esquema de datos.

Los contratos deben documentar:

- Tipo.
- Rango.
- Unidad.
- Valores permitidos.
- Valores predeterminados.
- Compatibilidad.
- Reglas de validación.

## 10.2. Errores

Espacio de nombres:

```text
org.parallax.Error.InvalidArgument
org.parallax.Error.NotPermitted
org.parallax.Error.NotFound
org.parallax.Error.Busy
org.parallax.Error.Unsupported
org.parallax.Error.Timeout
org.parallax.Error.Internal
```

Los errores deben ser estables y documentados.

No incluir rutas privadas, secretos ni trazas internas en mensajes de error públicos.

## 10.3. Timeouts

Presupuesto inicial de cliente:

- Consultas simples: 2 segundos.
- Operaciones locales: 2 segundos, salvo contrato específico.
- Operaciones largas: trabajos asíncronos.

Los valores deben validarse mediante pruebas y ajustarse a las operaciones reales.

Un timeout del cliente no significa que la operación del servidor haya sido cancelada.

Los clientes deben consultar el estado del trabajo antes de repetir una operación que pueda producir efectos duplicados.

---

# 11. Modelo de trabajos asíncronos

Las operaciones largas utilizarán un contrato común `Job`.

Ejemplos:

- Descargar actualización.
- Aplicar actualización.
- Crear snapshot.
- Restaurar snapshot.
- Ejecutar diagnóstico completo.
- Recopilar informes.

## 11.1. Estados

```text
queued
running
waiting_for_authorization
completed
failed
cancelled
```

## 11.2. Interfaz conceptual

```text
org.parallax.Job1
```

Métodos:

```text
GetState()
GetProgress()
GetResult()
Cancel()
```

Señales:

```text
StateChanged(state)
ProgressChanged(percentage, stage)
Completed(result)
Failed(error)
```

Cada trabajo tendrá:

- Identificador.
- Propietario.
- Estado.
- Fecha de creación.
- Progreso cuando corresponda.
- Resultado.
- Error.
- Política de cancelación.
- Política de persistencia.

## 11.3. Recuperación

Los trabajos deben definir su comportamiento después de reiniciar el servicio.

Opciones:

- Reanudar.
- Marcar como interrumpido.
- Revertir.
- Requerir intervención.

No todos los trabajos pueden reanudarse automáticamente.

Los trabajos de actualización y recuperación deben tener una estrategia explícita de consistencia.

---

# 12. Versionado de interfaces

Cada interfaz tendrá una versión mayor identificable.

Ejemplos:

```text
org.parallax.Session1
org.parallax.Settings1
org.parallax.Hardware1
org.parallax.Resources1
```

Una ruptura de compatibilidad requiere una nueva versión.

## 12.1. Reglas

- No cambiar tipos existentes de forma incompatible.
- No cambiar la semántica de operaciones existentes.
- No reutilizar identificadores retirados.
- No cambiar unidades sin versionar.
- Documentar campos opcionales.
- Permitir detectar capacidades soportadas.
- Mantener compatibilidad durante la transición cuando sea viable.

Una nueva versión podrá coexistir con la anterior durante un periodo de migración.

El periodo de coexistencia se establecerá en la política de releases.

No se garantizará coexistencia indefinida.

## 12.2. Compatibilidad de clientes

Los clientes deben:

- Detectar la versión disponible.
- Comprobar capacidades.
- Manejar errores de interfaz no disponible.
- Evitar asumir que todos los servicios están activos.
- Recuperarse de reinicios.
- No bloquear indefinidamente esperando una respuesta.

---

# 13. Generación de bindings

Los contratos XML D-Bus serán la fuente de verdad de las interfaces.

Ubicación:

```text
docs/
└── architecture/
    └── dbus/
        ├── session1.xml
        ├── settings1.xml
        ├── notifications1.xml
        ├── hardware1.xml
        ├── resources1.xml
        ├── update1.xml
        ├── recovery1.xml
        ├── diagnostics1.xml
        └── job1.xml
```

Los protocolos Wayland se definirán en:

```text
core/
└── proto/
    ├── xml/
    │   ├── parallax-shell-v1.xml
    │   └── parallax-workspace-v1.xml
    └── generated/
```

Las herramientas de generación y sus versiones deben estar fijadas en el entorno de desarrollo.

`zbus_xmlgen` podrá utilizarse para generar bindings cuando corresponda, pero se debe verificar que el XML, la herramienta y la versión elegida produzcan las interfaces esperadas.

## 13.1. Reglas de generación

- No editar manualmente código generado.
- Mantener XML versionado.
- Validar contratos en CI.
- Detectar rupturas incompatibles.
- Generar bindings reproducibles.
- Ejecutar pruebas de interoperabilidad.
- Documentar dependencias de generación.

El código generado no sustituye las pruebas de los contratos.

---

# 14. Observabilidad

Los componentes deben registrar:

- Conexiones.
- Desconexiones.
- Errores.
- Timeouts.
- Operaciones privilegiadas.
- Trabajos iniciados y finalizados.
- Cambios de versión.
- Fallos de sincronización.

Métricas iniciales:

| Métrica | Propósito |
|---|---|
| `ipc_requests_total` | Solicitudes |
| `ipc_errors_total` | Errores |
| `ipc_timeouts_total` | Timeouts |
| `ipc_latency` | Latencia |
| `ipc_connections` | Conexiones activas |
| `ipc_fd_transfers_total` | Transferencias de FD |
| `ipc_jobs_active` | Trabajos activos |
| `ipc_reconnections_total` | Reconexiones |

No registrar argumentos sensibles ni contenido privado.

La instrumentación debe ser de bajo coste.

---

# 15. Pruebas obligatorias

| ID | Prueba | Resultado esperado |
|---|---|---|
| IPC-001 | Cliente conecta a servicio | Conexión válida |
| IPC-002 | Servicio no disponible | Error controlado |
| IPC-003 | Rein