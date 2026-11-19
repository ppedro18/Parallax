# ADR-0012: Compatibilidad X11 mediante Xwayland bajo demanda

- **Estado:** Propuesto — requiere validación de compatibilidad (Fase 2)
- **Fecha:** 2026-11-19
- **Ámbito:** Compatibilidad con aplicaciones X11 heredadas.
- **Relaciones:** ADR-0003 (Smithay), ADR-0004 (Gestor de ventanas), ADR-0006 (Procesos del Shell).

## 1. Contexto

PARALLAX utilizará Wayland como protocolo principal para las aplicaciones gráficas.

Sin embargo, algunas aplicaciones todavía dependen de X11 y no ofrecen una implementación Wayland funcional.

Eliminar completamente la compatibilidad X11 en la primera versión podría limitar el uso de aplicaciones importantes, especialmente herramientas de desarrollo, aplicaciones Java, software antiguo y determinados juegos.

Implementar directamente un gestor de ventanas X11 dentro del compositor supondría mantener una segunda arquitectura de gestión de ventanas.

PARALLAX debe proporcionar compatibilidad con X11 sin convertir X11 en parte central de su arquitectura.

## 2. Opciones evaluadas

| Alternativa | Ventajas | Consideraciones |
|---|---|---|
| Xwayland integrado con Smithay | Integración directa y control completo | Mayor complejidad y necesidad de gestionar ventanas X11 |
| `xwayland-satellite` | Proceso independiente, implementación Rust, aislamiento | Dependencia externa y compatibilidad que debe verificarse |
| Sin compatibilidad X11 | Menor complejidad | Aplicaciones heredadas incompatibles |

## 3. Decisión provisional

Se selecciona `xwayland-satellite` como mecanismo de compatibilidad X11.

Se ejecutará bajo demanda, únicamente cuando una aplicación requiera X11.

El objetivo es mantener una arquitectura Wayland-first y evitar implementar un gestor de ventanas X11 propio.

La selección definitiva dependerá de las pruebas de integración realizadas durante la Fase 2.

La versión, licencia y dependencias del proyecto deberán verificarse antes de incorporarlo a la distribución.

## 4. Arquitectura

```text id="f1r8vn"
Aplicación X11
      |
      v
   Xwayland
      |
      v
xwayland-satellite
      |
      v
Wayland Protocol
      |
      v
Smithay Compositor
      |
      v
    px-wm
      |
      v
PARALLAX Rendering
```

El flujo es conceptual y deberá ajustarse a la integración real de la versión seleccionada de `xwayland-satellite`.

Xwayland proporcionará el servidor X11.

`xwayland-satellite` actuará como intermediario para integrar las ventanas X11 en el compositor Wayland.

Smithay y `px-wm` conservarán el control de la gestión de ventanas dentro de PARALLAX.

## 5. Gestión de ventanas

Las ventanas X11 deberán integrarse en el modelo de ventanas de PARALLAX.

Se contemplará:

- Creación y destrucción.
- Foco.
- Geometría.
- Movimiento.
- Redimensionamiento.
- Minimización.
- Maximización.
- Pantalla completa.
- Decoraciones.
- Orden de apilamiento.
- Gestión de múltiples monitores.
- Workspaces.

Las ventanas X11 deberán mantener una identidad de superficie y un ciclo de vida coherentes con las ventanas nativas Wayland.

No se implementará una política de gestión de ventanas independiente para X11.

`px-wm` será responsable de aplicar las políticas generales del escritorio cuando la información disponible y las capacidades de integración lo permitan.

## 6. Ejecución bajo demanda

Xwayland y `xwayland-satellite` no deberán ejecutarse permanentemente si no existen aplicaciones que requieran X11.

El sistema deberá:

1. Detectar una solicitud de ejecución X11.
2. Comprobar si el servicio ya está activo.
3. Iniciar Xwayland y el intermediario cuando sea necesario.
4. Esperar a que el servidor esté disponible.
5. Preparar el entorno de ejecución de la aplicación.
6. Mantener los procesos mientras existan clientes activos.
7. Detenerlos cuando ya no sean necesarios y sea seguro hacerlo.

Se deberá evitar iniciar instancias duplicadas.

La terminación del servicio deberá considerar procesos auxiliares, clientes activos y estado de sesión.

## 7. Integración con aplicaciones

Las aplicaciones Wayland nativas tendrán prioridad.

Las aplicaciones X11 se ejecutarán mediante Xwayland cuando sea necesario.

Se evaluarán las variables y mecanismos de selección de backend gráfico, incluyendo:

- `WAYLAND_DISPLAY`.
- `DISPLAY`.
- `XDG_SESSION_TYPE`.
- `GDK_BACKEND`.
- `QT_QPA_PLATFORM`.

No se forzará un backend global que impida a las aplicaciones utilizar Wayland nativo.

El entorno deberá permitir que una aplicación utilice Wayland mientras otra utiliza X11.

## 8. Compatibilidad

La Fase 2 deberá validar aplicaciones representativas.

### Aplicaciones de escritorio

- Aplicaciones GTK con backend X11.
- Aplicaciones Qt con backend X11.
- Aplicaciones Electron heredadas.
- Aplicaciones Java/AWT y Swing.
- Aplicaciones con menús y diálogos secundarios.

### Desarrollo

- IDEs.
- Herramientas gráficas.
- Aplicaciones de depuración.
- Herramientas de administración.

### Juegos

- Steam.
- Juegos que utilizan X11.
- Aplicaciones OpenGL.
- Aplicaciones Vulkan.
- Software con múltiples ventanas.

No se asumirá que la compatibilidad de Xwayland garantiza automáticamente la compatibilidad completa de todas estas aplicaciones.

Cada escenario deberá probarse en el hardware objetivo.

## 9. Gráficos y rendimiento

La integración deberá evaluar:

- OpenGL.
- Vulkan.
- Aceleración por GPU.
- VSync.
- Presentación de frames.
- Escalado.
- HiDPI.
- Pantalla completa.
- Latencia de entrada.
- Uso de memoria.
- Compatibilidad con múltiples monitores.

Se priorizará la aceleración gráfica y se evitarán rutas de copia innecesarias.

Los resultados deberán compararse con aplicaciones Wayland nativas cuando sea posible.

## 10. Seguridad y aislamiento

X11 tiene un modelo de seguridad menos estricto que Wayland.

Por ello, las aplicaciones X11 deberán tratarse como un dominio de compatibilidad con restricciones adicionales.

Se evaluará:

- Separación de procesos.
- Acceso al socket X11.
- Gestión de cookies de autenticación.
- Restricciones de acceso a dispositivos.
- Acceso al portapapeles.
- Acceso al sistema de ventanas.
- Aislamiento entre aplicaciones.

No se asumirá que ejecutar Xwayland en un proceso separado aísla automáticamente a los clientes X11 entre sí.

El acceso al servidor X11 deberá controlarse de acuerdo con las capacidades reales del mecanismo utilizado.

## 11. Recuperación

Un fallo de Xwayland o `xwayland-satellite` no deberá derribar el compositor ni el Shell.

Se implementará supervisión del ciclo de vida.

En caso de fallo:

1. Detectar la terminación.
2. Registrar el error.
3. Identificar las aplicaciones afectadas.
4. Limpiar sockets y recursos temporales cuando sea seguro.
5. Permitir reiniciar el servicio.
6. Mantener operativa la sesión Wayland nativa.

La recuperación no deberá provocar el reinicio de toda la sesión gráfica.

## 12. Benchmark y criterios de aceptación

Se realizarán pruebas de:

### Inicio

- Iniciar PARALLAX sin aplicaciones X11.
- Confirmar que Xwayland no se inicia innecesariamente.
- Abrir la primera aplicación X11.
- Medir tiempo de inicialización.

### Ejecución

- Abrir varias aplicaciones X11.
- Comprobar gestión de ventanas.
- Comprobar foco y teclado.
- Comprobar portapapeles.
- Comprobar escalado.

### Gráficos

- Probar aplicaciones OpenGL y Vulkan.
- Comprobar aceleración.
- Probar pantalla completa.
- Evaluar rendimiento.

### Fallos

- Finalizar Xwayland.
- Finalizar `xwayland-satellite`.
- Cerrar inesperadamente una aplicación X11.
- Comprobar recuperación.
- Verificar que Wayland continúa operativo.

### Criterios de aceptación

- Las aplicaciones Wayland nativas funcionan sin Xwayland.
- Xwayland se inicia bajo demanda.
- No se crean instancias duplicadas.
- Las ventanas X11 se integran con `px-wm`.
- Las aplicaciones X11 no pueden derribar el compositor.
- La terminación de Xwayland no afecta a las aplicaciones Wayland.
- Las aplicaciones representativas de la Fase 2 superan las pruebas definidas.
- El consumo adicional de memoria queda medido.

## 13. Gestión de versiones

La versión de Xwayland y `xwayland-satellite` deberá fijarse explícitamente.

Se realizará seguimiento de upstream.

Toda actualización deberá validar:

- Compatibilidad con Smithay.
- Compatibilidad con Wayland.
- Gestión de ventanas.
- Aceleración gráfica.
- Seguridad.
- Recuperación.
- Consumo de memoria.

La licencia y las dependencias deberán registrarse en el inventario de componentes de PARALLAX.

## 14. Consecuencias

### Positivas

- Arquitectura Wayland-first.
- Compatibilidad con aplicaciones heredadas.
- Menor complejidad en el compositor.
- Aislamiento de fallos X11.
- No requiere implementar un WM X11 propio.
- Inicio bajo demanda.
- Posibilidad de eliminar X11 en futuras instalaciones si deja de ser necesario.

### Negativas

- Dependencia de proyectos externos.
- Consumo adicional cuando se ejecutan aplicaciones X11.
- Compatibilidad variable entre aplicaciones.
- Modelo de seguridad heredado de X11.
- Complejidad de supervisión y ciclo de vida.
- Posibles diferencias de comportamiento frente a Wayland nativo.

## 15. Revisión

Revisar este ADR al finalizar las pruebas de compatibilidad de la Fase 2.

Cambiar la alternativa si:

- `xwayland-satellite` no satisface los requisitos.
- La integración con Smithay resulta insuficiente.
- Aplicaciones importantes no funcionan.
- El rendimiento gráfico es inaceptable.
- Existen problemas de seguridad no mitigables.

## 16. Principio rector

**Wayland será la plataforma nativa de PARALLAX. X11 será una capa de compatibilidad aislada, iniciada únicamente cuando sea necesaria.**