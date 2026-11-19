# Estrategia de pruebas de PARALLAX

- **Estado:** Propuesto
- **Ámbito:** Todos los componentes propios de PARALLAX y sus integraciones con el ecosistema Linux.
- **Objetivo:** Garantizar corrección funcional, estabilidad, seguridad, recuperación, accesibilidad y rendimiento medible.

## 1. Principios

La estrategia de pruebas de PARALLAX seguirá estos principios:

1. Cada componente tendrá pruebas asociadas a sus responsabilidades.
2. La lógica pura deberá poder probarse sin iniciar el entorno gráfico.
3. Los contratos entre procesos deberán probarse de forma independiente.
4. El compositor deberá probarse tanto en modo headless como sobre hardware real.
5. Los fallos deberán probarse deliberadamente y no únicamente mediante pruebas de funcionamiento normal.
6. El rendimiento deberá medirse con escenarios reproducibles.
7. Las pruebas visuales deberán detectar regresiones sin depender de comparaciones exactas de píxeles cuando no sean apropiadas.
8. La accesibilidad será un requisito funcional verificable.
9. Los resultados de las pruebas deberán poder reproducirse y rastrearse.
10. Ninguna prueba automatizada sustituye la validación de hardware y experiencia real.

## 2. Niveles de pruebas

| Nivel | Qué se valida | Herramientas | Frecuencia |
|---|---|---|---|
| Unitarias | Lógica pura, políticas, estados, parsers y utilidades | `cargo test`, `proptest` | Cada PR |
| Fuzzing | Parsers, protocolos, configuración, `px-wm` y VT | `cargo-fuzz`, libFuzzer | Nocturno y campañas específicas |
| Integración de componentes | Compositor, clientes de prueba, IPC y servicios | Harness propio, Smithay, D-Bus privado | Cada PR |
| Visual | Regresiones de Shell y aplicaciones | Capturas, screencopy y comparación perceptual | Cada PR en componentes visuales |
| Sistema | Arranque y comportamiento de imagen completa | QEMU, scripts y cliente Wayland de automatización | Nocturno |
| Recuperación | Fallos de procesos, configuración, actualización y almacenamiento | Harness de fault injection | Nocturno |
| Presión de memoria | Comportamiento bajo presión de RAM y crecimiento de procesos | `stress-ng`, aplicaciones sintéticas, métricas del sistema | Nocturno |
| Estabilidad | Fugas, degradación y fallos prolongados | VM y hardware de referencia | Soak semanal |
| Hardware | Compatibilidad de GPU, monitores, dispositivos y drivers | Matriz de hardware validado | Por release |
| Rendimiento | CPU, memoria, GPU, latencia y frames | Benchmarks definidos en `benchmarks/metodologia.md` | Cada PR en VM; mensual en hardware |
| Accesibilidad | Árbol semántico, navegación y tecnologías de asistencia | AccessKit, AT-SPI y harness | Cada PR de UI |
| Seguridad | Dependencias, permisos, IPC, procesos privilegiados y entradas hostiles | `cargo audit`, `cargo deny`, `cargo vet`, fuzzing y pruebas específicas | Cada PR y revisión de release |

## 3. Pruebas unitarias

Se probará toda la lógica que pueda ejecutarse de manera independiente.

Componentes prioritarios:

- `px-wm`.
- `px-fileops`.
- Parsers de configuración.
- Parsers de protocolos.
- Políticas de seguridad.
- Gestión de estados.
- Cálculos geométricos.
- Algoritmos de layout.
- Sistemas de preferencias.
- Validaciones de entrada.

### Requisitos

- Sin dependencia de GPU.
- Sin necesidad de una sesión gráfica.
- Resultados deterministas.
- Casos normales, límites y errores.
- Verificación de invariantes.
- Cobertura de regresiones conocidas.

`proptest` se utilizará para generar entradas y secuencias de operaciones.

Cuando sea aplicable, las propiedades incluirán:

- Ausencia de geometrías inválidas.
- Conservación de invariantes del estado.
- Determinismo de las transiciones.
- Validez de configuraciones aceptadas.
- Rechazo seguro de entradas inválidas.

No se impondrá un porcentaje global de cobertura como sustituto de calidad. Cada componente crítico deberá justificar sus objetivos de cobertura.

## 4. Fuzzing

Se utilizará `cargo-fuzz` para identificar errores ante entradas inesperadas o malformadas.

Objetivos iniciales:

- `px-wm`.
- Parsers de configuración.
- Parsers de protocolos IPC.
- Parsers VT de terminal.
- Deserialización de mensajes.
- Validación de manifiestos de widgets.
- Entradas de servicios expuestos mediante D-Bus.

### Frecuencia

- Ejecución nocturna automatizada.
- Campañas extendidas antes de releases.
- Reproducción obligatoria de cada fallo encontrado.
- Incorporación de casos de regresión al corpus.

### Requisitos

- No aceptar panics inesperados.
- No aceptar accesos fuera de límites.
- No permitir desbordamientos peligrosos.
- Limitar tamaños de entrada y consumo de recursos.
- Registrar y conservar los casos que producen fallos.

El corpus de fuzzing deberá versionarse y mantenerse.

Los resultados de fuzzing deberán incluir la versión del código, configuración y duración de la campaña.

## 5. Integración de componentes

Las pruebas de integración verificarán los contratos reales entre componentes.

### Compositor

- Inicio y cierre.
- Creación y destrucción de superficies.
- Foco.
- Geometría.
- Workspaces.
- Eventos de entrada.
- Integración con `px-wm`.
- Gestión de múltiples clientes.
- Errores de clientes.

Se utilizará un compositor headless para pruebas automatizadas que no dependan de DRM/KMS.

### Servicios

Se utilizará un bus D-Bus privado y temporal mediante `dbus-daemon --session`.

Se probarán:

- Registro de servicios.
- Métodos y señales.
- Errores de comunicación.
- Reconexión.
- Activación.
- Políticas de permisos.
- Cancelación.
- Comportamiento ante desconexiones.

### Contratos IPC

Cada contrato deberá probar:

- Mensajes válidos.
- Mensajes inválidos.
- Versiones incompatibles.
- Límites de tamaño.
- Errores de transporte.
- Desconexión inesperada.
- Cancelación de operaciones.
- Limpieza de recursos.

Ubicación:

`tests/integration/`

## 6. Pruebas visuales

Se utilizarán capturas del compositor y de las interfaces para detectar regresiones.

Los resultados se compararán con referencias aprobadas.

### Entorno reproducible

Cada captura deberá especificar:

- Resolución.
- Escala.
- Tema.
- Fuentes.
- Versión de componentes.
- Estado de ventanas.
- Datos de prueba.
- Estado de animaciones.
- Entorno de renderizado.

Se deberán desactivar o controlar reloj, notificaciones aleatorias, cursores y otros elementos variables.

### Comparación

Se utilizará comparación perceptual cuando sea apropiado.

Se diferenciarán:

- Cambios estructurales.
- Cambios geométricos.
- Cambios de tipografía.
- Cambios de color.
- Diferencias de antialiasing.
- Diferencias de renderizado por GPU.

Las diferencias que superen los umbrales definidos generarán una revisión.

No se actualizarán referencias visuales automáticamente sin aprobación.

### Cobertura visual

- Topbar.
- Dock.
- Launcher.
- Control Center.
- Notificaciones.
- Menús.
- Diálogos.
- Widgets.
- Estados de ventanas.
- Temas claro y oscuro.
- Escalado.
- Estados de error y carga.

Ubicación:

`tests/visual/`

## 7. Pruebas de sistema en VM

Se utilizará una imagen completa de PARALLAX ejecutada en QEMU.

La automatización utilizará un cliente de pruebas Wayland propio, con acciones y verificaciones reproducibles.

Se validará:

- Arranque.
- Inicio de sesión.
- Inicio del compositor.
- Inicio del Shell.
- Apertura de aplicaciones.
- Gestión de ventanas.
- Workspaces.
- IPC.
- Servicios.
- Configuración.
- Actualizaciones.
- Apagado y reinicio.

Las pruebas deberán verificar resultados observables y códigos de salida, no limitarse a ejecutar comandos.

### Limitaciones

La VM no sustituye las pruebas sobre GPU y hardware real.

El renderizado virtual puede ocultar o alterar problemas de drivers, sincronización, latencia, aceleración y presentación.

Ubicación:

`tests/vm/`

## 8. Pruebas de recuperación

Se realizarán pruebas deliberadas de fallos.

### Procesos

- Finalización inesperada del Shell.
- Finalización de widgets externos.
- Fallo de Xwayland.
- Fallo de `xwayland-satellite`.
- Fallo de servicios auxiliares.
- Desconexión de D-Bus.
- Fallo de aplicaciones.

### Configuración

- Archivo inválido.
- Archivo incompleto.
- Preferencias incompatibles.
- Configuración corrupta.
- Migración interrumpida.

### Actualizaciones

- Interrupción durante la actualización.
- Fallo de instalación de paquetes.
- Kernel defectuoso.
- Fallo durante el arranque.
- Rollback.
- Recuperación de configuración.

### Operaciones de archivos

- Apagado durante una copia.
- Interrupción durante una operación de movimiento.
- Falta de espacio.
- Permisos insuficientes.
- Destino inexistente.
- Archivo parcialmente escrito.

### Validación

No será suficiente comprobar que el proceso vuelve a ejecutarse.

También deberá verificarse:

- Integridad de los datos.
- Consistencia del estado.
- Ausencia de recursos huérfanos.
- Ausencia de ciclos de reinicio.
- Continuidad de los componentes no afectados.
- Capacidad de volver a un estado operativo.

Las pruebas destructivas se ejecutarán exclusivamente en VM, discos de prueba o entornos aislados.

Ubicación:

`tests/recovery/`

## 9. Presión de memoria

Se evaluará el comportamiento bajo presión de memoria mediante `stress-ng --vm` y aplicaciones sintéticas.

Se medirán:

- PSS.
- RSS.
- Memoria privada.
- Memoria compartida.
- Memoria GPU cuando esté disponible.
- CPU.
- Latencia de respuesta.
- Tiempo de frame.
- Tiempo de recuperación.
- Crecimiento de memoria durante la ejecución.

### Escenarios

- Sistema en reposo.
- Shell con todos sus componentes.
- Múltiples ventanas.
- Múltiples workspaces.
- Widgets activos.
- Terminales abiertas.
- Aplicaciones con consumo creciente.
- Presión intensa de memoria.
- Recuperación después de liberar memoria.

### Objetivos

- El compositor deberá conservar capacidad de respuesta.
- El Shell no deberá colapsar por presión ordinaria.
- Las tareas no críticas deberán poder cancelarse o limitarse.
- Los componentes deberán evitar crecimiento de memoria no acotado.
- Los fallos de aplicaciones no deberán derribar el entorno.

Se registrarán los casos en que el OOM killer intervenga.

La supervivencia del sistema no se considerará demostrada únicamente porque el proceso principal siga vivo.

Ubicación:

`tests/perf/`

## 10. Pruebas de estabilidad

Se realizarán pruebas prolongadas de 24 horas en VM y hardware de referencia.

Se evaluará:

- Crecimiento de memoria.
- Fugas.
- Descriptores abiertos.
- Procesos huérfanos.
- Consumo de CPU en reposo.
- Estabilidad de IPC.
- Fallos del Shell.
- Fallos del compositor.
- Degradación del rendimiento.

Se registrarán métricas periódicas y eventos durante toda la ejecución.

La prueba deberá finalizar con un informe de estabilidad.

Frecuencia: semanal y antes de releases importantes.

## 11. Validación de hardware

Se mantendrá una matriz de hardware validado.

Cada perfil incluirá:

- CPU.
- GPU.
- Driver.
- Kernel.
- Resoluciones.
- Monitores.
- Escalado.
- Conectividad.
- Dispositivos de entrada.
- Audio.
- Suspensión y reanudación.

Los perfiles tendrán estados:

- `validated`: validación completa.
- `partial`: validación parcial.
- `unsupported`: incompatibilidad conocida.
- `untested`: sin validación.

Una release deberá identificar qué perfiles fueron probados y cuáles siguen sin validar.

## 12. Rendimiento

La metodología oficial estará documentada en:

`benchmarks/metodologia.md`

Se medirán:

- Tiempo de arranque.
- Tiempo de inicio de sesión.
- Tiempo de apertura de ventanas.
- Latencia de entrada.
- Frame time.
- p50, p95 y p99.
- Uso de CPU.
- PSS y RSS.
- Memoria GPU.
- Consumo en reposo.
- Recuperación después de presión de recursos.

Los benchmarks de VM se utilizarán para detectar regresiones relativas.

Los valores absolutos de rendimiento gráfico deberán validarse en hardware real.

Los umbrales y tolerancias deberán definirse por escenario, resolución, frecuencia de refresco y hardware.

Una regresión superior al umbral establecido deberá bloquear el PR o requerir una excepción documentada.

## 13. Accesibilidad

Se comprobará la accesibilidad de las interfaces mediante:

- Árbol semántico AccessKit.
- Integración AT-SPI.
- Navegación por teclado.
- Orden de foco.
- Estados accesibles.
- Nombres y descripciones.
- Contraste.
- Escalado.
- Reduced Motion.
- Compatibilidad con tecnologías de asistencia.

Los tests automatizados deberán verificar que los componentes expongan información semántica correcta.

Se realizarán pruebas manuales con lectores de pantalla en las releases.

No se considerará suficiente que una interfaz simplemente exponga un árbol AccessKit: también debe comprobarse que las tecnologías de asistencia puedan consumirlo correctamente.

## 14. Seguridad

Se integrarán controles automáticos en CI.

### Dependencias

- `cargo audit`: vulnerabilidades conocidas.
- `cargo deny`: licencias, fuentes y dependencias.
- `cargo vet`: auditoría de dependencias cuando corresponda.

### Interfaces

- Fuzzing de parsers.
- Validación de mensajes.
- Pruebas de límites.
- Pruebas de permisos D-Bus.
- Pruebas de IPC.
- Pruebas de aislamiento de widgets.

### Privilegios

Se probarán especialmente:

- Agente Polkit.
- Operaciones privilegiadas.
- Acceso a sockets.
- Servicios del sistema.
- Permisos de archivos.
- Separación entre Shell y procesos externos.

### Requisitos

- Ninguna credencial en logs.
- Ninguna autorización basada únicamente en datos visuales no confiables.
- Rechazo seguro de mensajes inválidos.
- Permisos mínimos necesarios.
- Registro de excepciones de seguridad.

## 15. CI y clasificación de resultados

Los resultados se clasificarán en tres categorías:

### Bloqueantes

Fallos que impiden integrar el cambio:

- Unit tests.
- Integración aplicable.
- Regresiones funcionales críticas.
- Vulnerabilidades bloqueantes.
- Violaciones de contratos.
- Regresiones de rendimiento fuera de tolerancia.
- Fallos de accesibilidad en funcionalidades modificadas.

### Informativos

Resultados que requieren seguimiento, pero no bloquean automáticamente:

- Benchmarks exploratorios.
- Comparaciones de rendimiento con variabilidad alta.
- Métricas de consumo no estabilizadas.
- Resultados de compatibilidad no relacionados con el cambio.

### De release

Pruebas que deben superarse para aprobar una versión:

- Matriz de hardware.
- Recuperación completa.
- Soak test.
- Compatibilidad gráfica.
- Seguridad.
- Accesibilidad.
- Rendimiento en hardware real.

Los tests inestables deberán registrarse y corregirse. No se permitirá ocultar fallos mediante reintentos ilimitados ni desactivar pruebas sin una excepción documentada.

## 16. Trazabilidad

Cada ejecución deberá conservar:

- Commit.
- Versión del código.
- Toolchain Rust.
- Versión del kernel.
- Configuración de pruebas.
- Hardware o VM.
- Resultado.
- Duración.
- Logs.
- Artefactos relevantes.
- Identificador del test.

Los fallos deberán poder reproducirse localmente cuando sea posible.

Las pruebas de regresión deberán conservar el caso que originó el fallo.

## 17. Definición de terminado (Definition of Done)

Un componente se considerará terminado cuando:

1. Su especificación funcional esté cumplida.
2. Sus pruebas unitarias aplicables hayan pasado.
3. Sus pruebas de integración aplicables hayan pasado.
4. Sus pruebas de seguridad aplicables estén aprobadas.
5. Sus pruebas visuales y de accesibilidad aplicables hayan pasado.
6. Cumpla los presupuestos de rendimiento establecidos.
7. No tenga regresiones críticas conocidas sin excepción aprobada.
8. Su comportamiento ante fallos esté validado.
9. Exista documentación suficiente para mantenerlo.
10. El responsable del componente haya revisado el resultado.
11. QA haya aprobado los criterios de aceptación correspondientes.

Una excepción deberá incluir motivo, impacto, responsable y condición para resolverla.

## 18. Principio rector

**PARALLAX no se considera estable porque arranque correctamente. Se considera estable cuando sus componentes cumplen sus contratos, mantienen su rendimiento, resisten fallos y pueden recuperarse de manera verificable.**