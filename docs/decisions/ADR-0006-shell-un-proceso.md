# ADR-0006: Arquitectura de procesos del Shell y aislamiento de widgets

- **Estado:** Propuesto
- **Fecha:** 2026-11-19
- **Ámbito:** PARALLAX Shell, componentes internos, extensiones y widgets de terceros.
- **Relaciones:** ADR-0001 (Rust), ADR-0003 (Smithay), ADR-0004 (Gestor de ventanas), ADR-0005 (Toolkit de interfaz).

## 1. Contexto

PARALLAX necesita una arquitectura de procesos que permita mantener un consumo de memoria reducido, una experiencia visual coherente y una recuperación rápida ante fallos.

El Shell contiene múltiples elementos:

- Topbar.
- Dock.
- Launcher.
- Control Center.
- Notificaciones.
- Menús contextuales.
- Widgets del sistema.
- Indicadores de estado.
- Componentes de interacción.

Una arquitectura con un proceso independiente por elemento puede duplicar recursos, como fuentes, cachés, estructuras de interfaz, memoria del runtime y recursos gráficos.

Por otro lado, ejecutar todos los elementos dentro de un único proceso aumenta el impacto de un fallo interno.

También se contempla la posibilidad de permitir extensiones de terceros. Estas no deberán comprometer la estabilidad ni la seguridad del entorno.

## 2. Opciones evaluadas

| Arquitectura | Ventajas | Desventajas |
|---|---|---|
| Un proceso por componente | Aislamiento de fallos | Mayor consumo y comunicación IPC |
| Un único proceso Shell | Recursos compartidos y menor complejidad operativa | Mayor dominio de fallo |
| Shell con plugins dinámicos `.so` | Integración directa y comunicación rápida | ABI, seguridad y estabilidad |
| Shell + widgets externos | Aislamiento de terceros y flexibilidad | Coste de IPC y supervisión |

## 3. Decisión

Se adopta una arquitectura híbrida:

1. Un único proceso principal para los componentes oficiales del Shell.
2. Separación modular interna de los subsistemas.
3. Widgets de terceros ejecutados fuera del proceso principal.
4. Comunicación mediante un protocolo IPC controlado.
5. Prohibición de cargar plugins ejecutables `.so` dentro del proceso Shell.

La decisión busca reducir duplicaciones de recursos sin sacrificar el aislamiento de las extensiones externas.

## 4. Proceso principal del Shell

Los componentes oficiales de PARALLAX compartirán un único proceso.

Esto incluye:

- Topbar.
- Dock.
- Launcher.
- Control Center.
- Notificaciones.
- Menús del sistema.
- Indicadores internos.
- Widgets desarrollados y mantenidos por PARALLAX.

### 4.1 Recursos compartidos

El proceso podrá compartir:

- Runtime de interfaz.
- Fuentes y cachés tipográficas.
- Iconos y recursos visuales.
- Tokens y temas.
- Cachés de imágenes.
- Contextos y recursos gráficos compatibles.
- Bus interno de eventos.
- Estado visual compartido.
- Servicios internos del Shell.

El objetivo es evitar inicializaciones redundantes y duplicaciones innecesarias.

### 4.2 Organización interna

Un único proceso no implica una única unidad de código.

El Shell deberá mantener módulos con responsabilidades independientes.

Estructura conceptual:

```text
px-shell/
├── application/
├── runtime/
├── event_bus/
├── surfaces/
├── components/
│   ├── topbar/
│   ├── dock/
│   ├── launcher/
│   ├── control_center/
│   ├── notifications/
│   └── system_widgets/
├── rendering/
├── theme/
├── accessibility/
├── extensions/
│   ├── protocol/
│   ├── registry/
│   └── supervisor/
└── recovery/
```

Esta estructura representa módulos internos y no obliga a convertir cada directorio en un proceso independiente.

### 4.3 Independencia de componentes

Cada componente deberá mantener:

- Estado local controlado.
- Ciclo de vida definido.
- Contratos de eventos.
- Gestión explícita de recursos.
- Suscripciones cancelables.
- Separación entre lógica y presentación.

El fallo de un componente deberá poder detectarse y gestionarse cuando sea técnicamente posible.

No se permitirá que un componente mantenga referencias inválidas, recursos sin liberar o tareas asíncronas sin supervisión.

## 5. Widgets y extensiones de terceros

Los widgets externos se ejecutarán como procesos independientes.

Ningún código ejecutable de terceros podrá cargarse directamente dentro del proceso principal del Shell.

### 5.1 Modelo de ejecución

Cada widget externo tendrá su propio proceso supervisado.

El Shell será responsable de:

- Registrar widgets disponibles.
- Validar sus manifiestos.
- Autorizar capacidades.
- Iniciar y detener procesos.
- Supervisar su estado.
- Recibir sus declaraciones de interfaz.
- Validar las solicitudes de actualización.
- Gestionar errores y desconexiones.

Los widgets no tendrán acceso directo a los objetos internos del Shell.

### 5.2 Interfaz declarativa

Los widgets externos utilizarán una interfaz declarativa basada en un protocolo versionado.

El widget describirá su interfaz mediante un conjunto limitado de elementos permitidos.

Ejemplos:

- Texto.
- Iconos.
- Imágenes autorizadas.
- Contenedores.
- Filas y columnas.
- Indicadores.
- Botones.
- Listas.
- Valores de estado.
- Acciones declaradas.

El Shell será responsable de interpretar y renderizar estas declaraciones.

Los widgets no podrán introducir código Rust, JavaScript, HTML arbitrario ni shaders ejecutables dentro del proceso principal.

La interfaz declarativa deberá tener límites de profundidad, cantidad de nodos, tamaño de contenido y frecuencia de actualización.

### 5.3 Comunicación IPC

La comunicación utilizará un protocolo local versionado.

Se evaluará un transporte basado en Unix Domain Sockets con mensajes estructurados y límites de tamaño.

El protocolo deberá contemplar:

- Identificación del widget.
- Versión del protocolo.
- Registro y desregistro.
- Declaración de interfaz.
- Actualización de propiedades.
- Eventos de interacción.
- Solicitudes de capacidades.
- Respuestas y errores.
- Señales de vida.
- Cancelación y desconexión.

No se permitirá que un widget envíe instrucciones arbitrarias para ejecutar operaciones internas del Shell.

### 5.4 Capacidades y permisos

Los widgets funcionarán bajo el principio de mínimo privilegio.

Cada widget deberá declarar las capacidades que necesita.

Ejemplos:

- Lectura de hora y fecha.
- Información meteorológica mediante un proveedor autorizado.
- Estado de batería.
- Información de red no sensible.
- Acceso a archivos mediante una autorización explícita.
- Notificaciones mediante una API controlada.

El acceso a capacidades requerirá autorización del sistema.

Los permisos deberán poder revocarse sin necesidad de reiniciar el Shell.

Un widget no tendrá acceso automático a:

- Archivos personales.
- Credenciales.
- Variables de entorno sensibles.
- Procesos del usuario.
- Memoria del Shell.
- Dispositivos.
- Operaciones administrativas.

La disponibilidad de una capacidad no implica que el widget tenga permiso para utilizarla.

### 5.5 Aislamiento

Los procesos de widgets deberán ejecutarse con restricciones adecuadas.

Se evaluarán mecanismos disponibles en Linux:

- Separación de usuario y permisos.
- Namespaces.
- Seccomp.
- Landlock.
- Restricciones de acceso al sistema de archivos.
- Límites de memoria y CPU.
- Restricciones de comunicación IPC.

Las restricciones concretas dependerán de las capacidades del kernel y del modelo de ejecución seleccionado.

Un widget no podrá bloquear el proceso principal mediante operaciones de renderizado, mensajes excesivos o solicitudes sin límites.

## 6. Prohibición de plugins dinámicos `.so`

Se prohíbe cargar plugins ejecutables `.so` de terceros dentro del proceso principal del Shell en la primera versión.

Motivos:

- Rust no ofrece una ABI estable general para interfaces dinámicas arbitrarias.
- La compatibilidad entre versiones puede romperse.
- Los plugins comparten el espacio de memoria del proceso anfitrión.
- Un fallo puede derribar todo el Shell.
- El código ejecutado tendría acceso potencial a recursos internos.
- La actualización y distribución se vuelven más complejas.

La prohibición no impide utilizar bibliotecas dinámicas del sistema cuando formen parte de dependencias controladas y evaluadas.

Cualquier cambio de esta política requerirá un ADR específico.

## 7. Gestión de memoria y rendimiento

La arquitectura de un solo proceso pretende reducir el consumo de recursos compartidos.

Se establece como hipótesis inicial un ahorro potencial de 60–150 MB frente a una implementación equivalente con seis procesos independientes.

Este valor no constituye una garantía ni un presupuesto definitivo.

Deberá comprobarse mediante mediciones reproducibles.

### 7.1 Métricas

Se medirán:

- PSS total del Shell.
- RSS de cada proceso.
- Memoria gráfica cuando pueda atribuirse.
- Memoria de widgets externos.
- Consumo en reposo.
- Consumo con todos los componentes activos.
- Crecimiento de memoria durante sesiones prolongadas.
- Tiempo de inicialización.
- Latencia de IPC.
- Tiempo de actualización de interfaces.

### 7.2 Límites

Los widgets externos deberán disponer de límites configurables para:

- Memoria máxima.
- Uso de CPU.
- Frecuencia de actualización.
- Cantidad de nodos declarativos.
- Tamaño máximo de mensajes.
- Cantidad de solicitudes pendientes.

El sistema deberá poder suspender o finalizar widgets que excedan los límites autorizados.

La terminación de un widget no deberá afectar a los componentes oficiales del Shell.

## 8. Recuperación ante fallos

El Shell será un proceso de larga duración, pero deberá asumir que puede fallar.

La recuperación se realizará mediante un supervisor externo al proceso Shell.

Se evaluará utilizar un servicio de usuario de systemd para iniciar, supervisar y reiniciar el proceso.

### Nivel 1 — Recuperación de componentes

Cuando un componente interno falle de manera recuperable:

- Registrar el error.
- Cancelar tareas asociadas.
- Liberar recursos cuando sea posible.
- Restaurar el estado inicial del componente.
- Evitar propagar el fallo a otros componentes.

No se asumirá que cualquier panic puede recuperarse de forma segura.

### Nivel 2 — Reinicio del Shell

Cuando el proceso principal termine inesperadamente:

1. El supervisor detectará la terminación.
2. Se registrará el motivo cuando esté disponible.
3. Se reiniciará el proceso.
4. El Shell reconstruirá sus superficies.
5. Se restaurará la configuración persistente.
6. Se volverán a registrar los widgets autorizados.

La recuperación no dependerá de conservar el estado volátil del proceso anterior.

Se implementará una política de reinicio con límites y backoff para evitar ciclos de fallos continuos.

### Estado persistente

El Shell no será propietario de datos críticos que deban sobrevivir a un fallo.

Se distinguirá entre:

- Configuración persistente.
- Preferencias del usuario.
- Estado temporal de las interfaces.
- Estado de los widgets.
- Estado de las superficies.

La configuración y preferencias deberán almacenarse mediante mecanismos independientes del ciclo de vida del Shell.

El estado temporal se reconstruirá cuando sea posible.

### Límite de recuperación

El reinicio del Shell no deberá reiniciar automáticamente el compositor ni cerrar toda la sesión gráfica.

El compositor y el Shell mantendrán ciclos de vida independientes.

## 9. Seguridad y estabilidad

Se aplicarán las siguientes reglas:

- El Shell no ejecutará código de widgets externos.
- Los mensajes IPC se validarán antes de procesarse.
- Las declaraciones de interfaz tendrán límites estrictos.
- Los widgets no podrán bloquear el hilo principal del Shell.
- Los procesos externos tendrán capacidades explícitas.
- Las desconexiones se tratarán como eventos normales.
- Los errores de extensiones no podrán propagarse al compositor.
- Las operaciones sensibles requerirán autorización.
- Los componentes internos deberán gestionar correctamente la cancelación de tareas.

## 10. Benchmark y criterios de aceptación

La arquitectura deberá validarse mediante pruebas reproducibles.

### Escenario A: Shell en reposo

- Iniciar el Shell.
- Mostrar los componentes principales.
- Medir PSS y RSS.
- Medir memoria gráfica disponible.
- Comparar con una implementación multiproceso equivalente.

### Escenario B: Shell completo

- Activar Topbar.
- Activar Dock.
- Activar Launcher.
- Activar Control Center.
- Activar Notificaciones.
- Activar widgets oficiales.

Medir memoria y latencia.

### Escenario C: Widgets externos

- Ejecutar varios widgets.
- Medir consumo individual y total.
- Simular actualizaciones frecuentes.
- Simular desconexiones.
- Simular mensajes inválidos.
- Comprobar aislamiento.

### Escenario D: Recuperación

- Finalizar un widget.
- Finalizar el proceso Shell.
- Provocar un error en un componente interno.
- Comprobar el reinicio.
- Verificar la reconstrucción de superficies.
- Confirmar que el compositor permanece activo.

### Criterios de aceptación

- Los componentes oficiales funcionan en un único proceso.
- Los widgets externos se ejecutan fuera del proceso Shell.
- No se cargan plugins ejecutables de terceros mediante `.so`.
- El protocolo IPC está versionado y validado.
- Los widgets están sujetos a límites de recursos.
- El Shell puede reiniciarse sin reiniciar el compositor.
- La configuración persistente sobrevive al reinicio.
- No existe crecimiento sostenido de memoria durante pruebas prolongadas.
- El ahorro de memoria se mide y documenta, sin asumir el valor estimado.

## 11. Consecuencias

### Positivas

- Menor duplicación de recursos.
- Posibilidad de compartir fuentes, iconos y cachés.
- Mayor coherencia visual.
- Menor complejidad de comunicación entre componentes oficiales.
- Aislamiento de widgets externos.
- Mejor control sobre las extensiones.
- Recuperación del Shell independiente del compositor.

### Negativas

- Un fallo grave del proceso principal afecta a todos los componentes oficiales.
- El Shell requiere una arquitectura interna modular.
- El protocolo IPC requiere mantenimiento.
- Los widgets externos consumen memoria adicional.
- El supervisor debe gestionar procesos y reinicios.
- La interfaz declarativa limita las posibilidades de personalización de terceros.

## 12. Revisión

Este ADR deberá revisarse si:

- El consumo del Shell supera el presupuesto definido.
- La arquitectura monoproceso provoca fallos difíciles de aislar.
- La comunicación IPC limita significativamente la experiencia de los widgets.
- Se requiere un aislamiento más estricto.
- Aparecen nuevas necesidades de extensibilidad.
- El coste de mantener el protocolo declarativo supera sus beneficios.

## 13. Principio rector

**El Shell debe compartir recursos internamente y aislar los riesgos externamente.**

Los componentes oficiales de PARALLAX compartirán un proceso, pero mantendrán responsabilidades y límites internos definidos.

Las extensiones de terceros nunca deberán tener el mismo nivel de confianza que el código oficial del sistema.