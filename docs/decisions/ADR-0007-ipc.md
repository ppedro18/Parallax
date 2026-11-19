# ADR-0007: Arquitectura IPC — Wayland para superficies y D-Bus para servicios

- **Estado:** Propuesto
- **Fecha:** 2026-11-19
- **Ámbito:** Comunicación entre compositor, Shell, servicios del sistema y extensiones.
- **Relaciones:** ADR-0003 (Smithay), ADR-0004 (Gestor de ventanas), ADR-0006 (Arquitectura de procesos).

## 1. Contexto

PARALLAX necesita una arquitectura de comunicación entre procesos que permita conectar sus componentes sin introducir protocolos innecesarios ni dependencias difíciles de mantener.

El sistema deberá comunicarse con:

- El compositor Wayland.
- El Shell.
- Los servicios internos de PARALLAX.
- systemd.
- NetworkManager.
- BlueZ.
- polkit.
- Portals de escritorio.
- Aplicaciones externas.
- Widgets de terceros.

No todas estas comunicaciones tienen las mismas necesidades.

Las operaciones relacionadas con superficies gráficas requieren integración directa con Wayland. Las operaciones de servicios necesitan interoperabilidad con el ecosistema Linux.

Por tanto, PARALLAX no utilizará un único mecanismo IPC para todas las comunicaciones.

## 2. Opciones evaluadas

| Tecnología | Ventajas | Consideraciones |
|---|---|---|
| D-Bus (`zbus`) | Estándar Linux, interoperabilidad y activación de servicios | No es apropiado para transportar flujos gráficos de alta frecuencia |
| Varlink | Protocolo moderno orientado a servicios | Ecosistema y adopción que deben evaluarse |
| gRPC sobre UDS | Contratos definidos y herramientas maduras | Complejidad y dependencias adicionales |
| Cap'n Proto sobre UDS | Serialización eficiente y esquemas | Mayor complejidad de integración |
| Arquitectura híbrida | Utiliza el mecanismo adecuado para cada responsabilidad | Requiere reglas claras de separación |

## 3. Decisión

Se adopta una arquitectura híbrida basada en tres mecanismos:

1. **Wayland:** comunicación relacionada con superficies, ventanas, entrada y presentación gráfica.
2. **D-Bus mediante zbus:** comunicación entre servicios, operaciones del sistema y capacidades de PARALLAX.
3. **memfd:** intercambio de payloads grandes cuando el transporte lo requiera y permita transferir descriptores de archivo.

No se implementará un protocolo IPC genérico propio mientras los mecanismos seleccionados cubran las necesidades del sistema.

Cada mecanismo tendrá responsabilidades delimitadas.

## 4. Wayland: comunicación gráfica

Wayland será el mecanismo principal para toda comunicación directamente relacionada con la experiencia gráfica.

El compositor implementado sobre Smithay será responsable de administrar las superficies y sus relaciones.

### 4.1 Responsabilidades

- Creación y destrucción de superficies.
- Gestión de ventanas.
- Configuración y geometría.
- Eventos de entrada.
- Foco.
- Presentación de buffers.
- Sincronización gráfica.
- Escalado.
- Gestión de outputs.
- Integración de superficies del Shell.
- Comunicación entre clientes y compositor.

### 4.2 Protocolos

Se utilizarán protocolos estándar siempre que exista una solución adecuada.

Entre ellos:

- `wl_compositor`.
- `wl_surface`.
- `xdg-shell`.
- `wl_seat`.
- `wl_output`.
- `wl_data_device`.
- `zwp_linux_dmabuf_v1`.
- `wp_viewporter`.
- `wp_presentation`.
- `wp_fractional_scale_manager_v1`.
- `zwp_linux_explicit_synchronization_v1`, cuando esté disponible y sea compatible.

La disponibilidad y versión de cada protocolo dependerá de la implementación del compositor y de los clientes.

No se asumirá que todos los protocolos estarán disponibles desde la primera versión.

### 4.3 Protocolos privados PARALLAX

Se permitirán protocolos Wayland privados cuando representen capacidades específicas del compositor que no estén cubiertas adecuadamente por protocolos existentes.

Ejemplos posibles:

- Integración avanzada del Shell.
- Operaciones especiales sobre workspaces.
- Capacidades específicas de superficies del entorno.
- Información de estado que deba intercambiarse directamente con clientes autorizados.

Cada protocolo privado deberá:

- Tener una especificación documentada.
- Definir sus interfaces y eventos.
- Tener versionado explícito.
- Validar parámetros recibidos.
- Definir permisos y clientes autorizados.
- Mantener compatibilidad cuando sea posible.
- Tener pruebas de interoperabilidad.

Los protocolos privados no deberán utilizarse para sustituir interfaces estándar existentes.

### 4.4 Seguridad

La exposición de un protocolo privado no implica que cualquier cliente pueda utilizarlo.

El compositor deberá controlar:

- Qué clientes pueden acceder a cada interfaz.
- Qué operaciones están autorizadas.
- Qué parámetros son válidos.
- Qué recursos puede modificar cada cliente.

No se expondrán operaciones administrativas o privilegiadas mediante protocolos gráficos sin controles de autorización.

## 5. D-Bus: comunicación entre servicios

D-Bus será el mecanismo principal de comunicación para servicios y operaciones del sistema.

Se utilizará `zbus` como implementación Rust.

### 5.1 Responsabilidades

- Consulta de estado de servicios.
- Gestión de configuración.
- Operaciones de red.
- Bluetooth.
- Gestión de energía.
- Notificaciones.
- Gestión de dispositivos.
- Comunicación con servicios internos.
- Solicitudes de capacidades.
- Integración con servicios externos.

### 5.2 Integración con el ecosistema Linux

Se priorizará la interoperabilidad con interfaces existentes.

Servicios relevantes:

| Servicio | Responsabilidad |
|---|---|
| systemd | Ciclo de vida de servicios |
| polkit | Autorización de operaciones privilegiadas |
| NetworkManager | Gestión de conectividad |
| BlueZ | Gestión Bluetooth |
| Portals | Acceso controlado a recursos del escritorio |
| UPower | Información y estado de energía, cuando esté disponible |
| logind | Sesiones, dispositivos y administración de sesión |

PARALLAX no deberá implementar nuevamente servicios que ya estén resueltos por el ecosistema Linux.

Se utilizarán sus interfaces públicas y mecanismos de autorización.

### 5.3 Organización de servicios propios

Los servicios de PARALLAX deberán utilizar nombres, interfaces y rutas de objetos consistentes.

Convención conceptual:

```text
org.parallax.Shell
org.parallax.Compositor
org.parallax.Settings
org.parallax.Power
org.parallax.Widgets
org.parallax.Notifications
```

Las interfaces deberán tener responsabilidades acotadas.

Ejemplo:

```text
org.parallax.Settings
    /org/parallax/Settings
        org.parallax.Settings

org.parallax.Power
    /org/parallax/Power
        org.parallax.Power
```

Los nombres definitivos se asignarán según la arquitectura real de servicios.

### 5.4 Activación y ciclo de vida

Se aprovecharán las capacidades de activación y supervisión del ecosistema D-Bus y systemd cuando corresponda.

Los servicios deberán definir:

- Nombre de servicio.
- Propietario del nombre.
- Interfaces públicas.
- Dependencias.
- Política de inicio.
- Política de apagado.
- Comportamiento ante desconexiones.
- Política de reintentos.

El Shell no deberá asumir que todos los servicios están disponibles permanentemente.

Las desconexiones y reinicios deberán tratarse como situaciones normales.

## 6. Política de frecuencia y rendimiento

D-Bus no será utilizado como canal de transporte para actualizaciones gráficas de alta frecuencia.

Las operaciones de control y consulta son adecuadas para D-Bus, pero los flujos de eventos intensivos requieren mecanismos específicos.

### 6.1 Coalescencia

Las actualizaciones repetitivas deberán agruparse cuando solo importe el estado más reciente.

Ejemplos:

- Porcentaje de batería.
- Estado de conectividad.
- Cambios de volumen.
- Actualizaciones de indicadores.
- Cambios de configuración visual.

Si llegan múltiples actualizaciones antes de que el consumidor pueda procesarlas, se podrá conservar únicamente la última actualización relevante.

### 6.2 Límites

Los consumidores deberán evitar:

- Suscripciones innecesarias.
- Consultas periódicas excesivas.
- Mensajes duplicados.
- Acumulación ilimitada de eventos.
- Procesamiento costoso en el hilo de interfaz.

Se definirán políticas de frecuencia, colas acotadas y cancelación de solicitudes.

Las operaciones que no necesiten respuesta inmediata deberán ejecutarse de forma asíncrona.

### 6.3 Operaciones y tiempos de espera

Las llamadas D-Bus deberán establecer:

- Timeout apropiado.
- Gestión de errores.
- Cancelación cuando sea posible.
- Política de reintentos.
- Identificación de operaciones idempotentes.
- Comportamiento ante respuestas tardías.

No se repetirán automáticamente operaciones que puedan provocar efectos duplicados sin comprobar su idempotencia.

## 7. memfd: transferencia de payloads grandes

Se utilizará `memfd` como mecanismo auxiliar para transferir datos grandes entre procesos cuando resulte conveniente.

No reemplazará a Wayland ni a D-Bus.

### 7.1 Casos de uso

- Documentos temporales.
- Datos binarios grandes.
- Snapshots de estado.
- Payloads estructurados.
- Recursos que no deban copiarse repetidamente entre procesos.

La utilización de `memfd` deberá justificarse frente a una transferencia convencional.

### 7.2 Transferencia de descriptores

Cuando se utilice D-Bus para coordinar una transferencia basada en `memfd`, se aprovechará el paso de descriptores de archivo mediante las capacidades de transporte compatibles.

No se enviarán los contenidos completos de archivos grandes como mensajes D-Bus convencionales.

### 7.3 Seguridad

Los descriptores recibidos deberán validarse antes de utilizarse.

Se establecerán controles sobre:

- Tamaño máximo.
- Tipo de contenido.
- Permisos.
- Ciclo de vida.
- Propiedad del descriptor.
- Acceso de lectura y escritura.
- Validez del contenido.

Cuando corresponda, se utilizarán seals de `memfd` para impedir modificaciones posteriores a la publicación del contenido.

No se confiará en el contenido recibido por el simple hecho de provenir de otro proceso local.

### 7.4 Diferencia con buffers gráficos

Los buffers de superficies Wayland seguirán utilizando los mecanismos gráficos correspondientes.

Ejemplos:

- Shared memory buffers.
- DMA-BUF.
- Sincronización explícita cuando esté disponible.

No se diseñará un mecanismo paralelo basado en D-Bus para transportar frames o buffers gráficos.

## 8. Widgets externos y extensiones

En concordancia con ADR-0006, los widgets externos deberán comunicarse con el Shell mediante interfaces controladas.

Se utilizará D-Bus para operaciones de control cuando resulte adecuado:

- Registro.
- Identificación.
- Solicitud de capacidades.
- Configuración.
- Estado.
- Activación.
- Desactivación.
- Gestión del ciclo de vida.

Las declaraciones de interfaz y sus actualizaciones deberán respetar el límite de frecuencia y tamaño definido por el protocolo.

Si las pruebas demuestran que el tráfico de widgets requiere una comunicación sostenida de alta frecuencia, se evaluará un canal dedicado sobre Unix Domain Sockets mediante un ADR específico.

No se introducirá automáticamente un protocolo adicional en la primera versión.

Los widgets no tendrán acceso directo a interfaces privilegiadas del compositor.

## 9. Versionado y compatibilidad

Los servicios deberán definir interfaces versionadas.

Se priorizará la evolución compatible:

- Añadir métodos sin romper consumidores existentes.
- Añadir propiedades opcionales.
- Definir comportamientos para campos desconocidos.
- Evitar cambios incompatibles sin una migración documentada.
- Mantener compatibilidad entre componentes de distintas versiones cuando sea viable.

Los protocolos privados Wayland tendrán su propio control de versiones.

La compatibilidad deberá verificarse durante las actualizaciones del sistema.

## 10. Varlink y alternativas futuras

Varlink permanecerá como alternativa para una futura revisión.

Se reevaluará cuando su adopción dentro del ecosistema systemd y las herramientas relevantes de Linux justifique su incorporación.

No se adoptará un segundo mecanismo de comunicación de servicios únicamente por diferencias de rendimiento teórico.

La decisión deberá basarse en interoperabilidad, mantenimiento, seguridad y resultados medidos.

gRPC y Cap'n Proto tampoco se incorporarán a la arquitectura base mientras D-Bus cubra las necesidades de comunicación entre servicios.

## 11. Benchmark y criterios de aceptación

Se realizarán pruebas para validar el comportamiento de los mecanismos elegidos.

### Escenario A: D-Bus

- Latencia de llamadas.
- Frecuencia de mensajes.
- Consumo de memoria.
- Comportamiento con múltiples consumidores.
- Desconexión y reconexión.
- Carga de eventos.

### Escenario B: Coalescencia

- Enviar múltiples actualizaciones repetitivas.
- Medir mensajes procesados.
- Medir latencia del estado final.
- Comprobar que no se acumulan eventos indefinidamente.

### Escenario C: memfd

- Transferir payloads de distintos tamaños.
- Medir latencia.
- Medir memoria.
- Validar límites.
- Comprobar permisos y seals.
- Simular descriptores inválidos.

### Escenario D: Wayland

- Creación y destrucción de superficies.
- Comunicación con clientes.
- Gestión de eventos.
- Presentación gráfica.
- Integración con protocolos privados.
- Validación de clientes no autorizados.

### Criterios de aceptación

- Las superficies utilizan Wayland.
- Los servicios utilizan D-Bus mediante zbus.
- No se transportan frames gráficos mediante D-Bus.
- Los eventos repetitivos tienen políticas de coalescencia.
- Las transferencias grandes tienen límites y validaciones.
- Los descriptores recibidos tienen ciclo de vida controlado.
- Los protocolos privados están documentados y versionados.
- Los servicios gestionan desconexiones y errores.
- No se introduce un protocolo adicional sin justificación técnica.

## 12. Consecuencias

### Positivas

- Integración con el ecosistema Linux.
- Interoperabilidad con systemd, polkit y otros servicios.
- Separación clara entre comunicación gráfica y comunicación de servicios.
- Menor cantidad de protocolos propios.
- Aprovechamiento de mecanismos existentes.
- Posibilidad de compartir datos grandes sin transportar su contenido como mensajes convencionales.
- Arquitectura extensible.

### Negativas

- D-Bus no es adecuado para todos los patrones de comunicación.
- Se necesita controlar la frecuencia de mensajes.
- Los protocolos privados Wayland requieren mantenimiento.
- La gestión de descriptores requiere validaciones adicionales.
- La arquitectura híbrida exige reglas de responsabilidad claras.

## 13. Revisión

Este ADR deberá revisarse si:

- D-Bus representa un cuello de botella medible.
- Varlink alcanza una adopción suficiente en el ecosistema objetivo.
- Los widgets requieren un canal dedicado.
- Los protocolos privados se vuelven difíciles de mantener.
- Se identifican problemas de seguridad o compatibilidad.
- Aparecen necesidades de comunicación que no cubren los mecanismos seleccionados.

## 14. Principio rector

**Wayland comunica el espacio gráfico. D-Bus comunica los servicios. memfd facilita el intercambio de datos grandes cuando es necesario.**

PARALLAX utilizará los mecanismos nativos del ecosistema Linux antes de crear infraestructura de comunicación propia.