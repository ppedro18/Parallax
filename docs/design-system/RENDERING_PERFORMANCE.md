# Rendimiento de renderizado de efectos — PARALLAX

**Estado:** Propuesto · Pendiente de aprobación.  
**Responsable técnico:** `ingeniero-rendimiento`.  
**Ámbito:** `parallax-compositor`, `parallax-shell`, `px-ui` y efectos gráficos asociados.

## 1. Objetivo

Definir presupuestos verificables para los efectos visuales de PARALLAX.

Cada efecto debe disponer de:

- Presupuesto de CPU y GPU.
- Técnica de implementación.
- Condiciones de activación.
- Fallback funcional.
- Métricas de rendimiento.
- Pruebas de regresión.
- Criterios de degradación y recuperación.

Ningún efecto debe comprometer la capacidad del compositor para mantener una presentación fluida.

La calidad visual nunca debe tener prioridad sobre la estabilidad, la respuesta de entrada o la accesibilidad.

El agente `ingeniero-rendimiento` tiene autoridad para bloquear la incorporación de un efecto cuando no cumple sus presupuestos o no dispone de mediciones reproducibles.

## 2. Presupuesto de frame

### 2.1 Hardware de referencia

**HW-A — referencia inicial**

- Arquitectura: x86_64.
- CPU: portátil Intel o AMD.
- GPU: integrada.
- Resolución: 1920 × 1080.
- Frecuencia de referencia: 60 Hz.
- Sistema: Linux nativo.
- Backend: DRM/KMS.
- Renderizado: GPU acelerada.

La configuración exacta de CPU, GPU, controlador, kernel, Mesa y resolución debe quedar registrada en cada benchmark.

### 2.2 Objetivos generales

| Métrica | Objetivo |
|---|---:|
| Presupuesto CPU del compositor | ≤ 4 ms/frame |
| Presupuesto GPU del compositor | ≤ 4 ms/frame |
| Intervalo de presentación a 60 Hz | 16,67 ms |
| Frames perdidos en escenarios estables | < 1 % |
| Renderizado del compositor en reposo | 0 frames innecesarios |
| Trabajo GPU atribuible al compositor en reposo | 0 trabajo nuevo evitable |

Los presupuestos CPU y GPU son objetivos independientes. No deben sumarse para afirmar que el frame tarda 8 ms, porque parte del trabajo puede ejecutarse en paralelo.

Se deben medir también:

- Tiempo CPU de preparación y envío de comandos.
- Tiempo GPU de ejecución.
- Tiempo de espera por sincronización.
- Tiempo hasta la presentación.
- Presentación tardía.
- Frames descartados.
- Latencia de entrada a presentación cuando exista instrumentación válida.

El presupuesto de 4 ms no sustituye el límite de presentación. A 60 Hz el compositor dispone de un intervalo nominal de 16,67 ms, pero debe conservar margen para sincronización, variaciones del sistema y trabajo de otros procesos.

### 2.3 Presupuesto agregado

Los presupuestos individuales de los efectos no son aditivos de forma ilimitada.

El compositor debe disponer de un presupuesto agregado por output y escenario.

| Categoría | Presupuesto inicial |
|---|---:|
| Geometría, composición y operaciones básicas | Incluido en CPU/GPU general |
| Esquinas y clipping | 0,1 ms/ventana |
| Sombras | 0,1 ms/ventana |
| Blur | 1,0 ms/output |
| Scrim y atenuación | 0,05 ms/output |
| Animaciones | 0,3 ms/output |
| Overview | 2,0 ms/output |
| Filtros de accesibilidad | 0,5 ms/output |

Estos valores son presupuestos de ingeniería iniciales, no garantías de rendimiento.

Los efectos deben compartir recursos cuando sea posible.

Si varios efectos compiten por el presupuesto, el compositor aplica prioridades:

1. Presentación correcta y estabilidad.
2. Contenido de las ventanas.
3. Accesibilidad y legibilidad.
4. Composición básica.
5. Animaciones.
6. Sombras y blur decorativo.

El compositor no debe intentar mantener todos los efectos a costa de perder frames.

## 3. Presupuestos por efecto

### 3.1 Esquinas redondeadas

**Presupuesto:** ≤ 0,1 ms GPU por ventana.

**Técnica:**

- Signed Distance Field (SDF) en shader.
- Cálculo del clipping durante la composición.
- Parámetro de radio obtenido de los tokens de tema.
- Sin renderizado adicional de la aplicación cliente.

**Fallback:**

- Recorte rectangular cuando el shader o backend no soporte el efecto correctamente.
- No se debe modificar el contenido original de la ventana.

Las esquinas deben respetar las reglas de `WINDOW_STATES`.

No deben recortar áreas de entrada, texto ni contenido que deba permanecer visible.

### 3.2 Sombras

**Presupuesto:** ≤ 0,1 ms GPU por ventana.

**Técnica:**

- Texturas de sombra 9-slice precalculadas.
- Cache por radio, nivel de elevación, escala y configuración del tema.
- Reutilización de recursos entre ventanas compatibles.
- Invalidación de cache solo cuando cambien sus parámetros.

**Fallback:**

- Eliminar sombras decorativas bajo sobrecarga.
- Mantener los bordes y la separación visual necesarios.

No recalcular sombras completas durante cada movimiento de ventana.

### 3.3 Vidrio y blur

**Presupuesto:** ≤ 1,0 ms GPU por output.

**Técnica inicial:**

- Dual Kawase o técnica equivalente.
- Pirámide de resolución reducida.
- Región de procesamiento limitada al área visible.
- Reutilización de buffers intermedios.
- Cache de resultados cuando el fondo no haya cambiado.
- Evitar múltiples cadenas de blur para superficies superpuestas.

La configuración inicial será:

- Resolución de trabajo: 1/4 lineal respecto al output.
- Número de pasadas: definido y validado mediante benchmark.
- Formato y precisión: seleccionados según soporte del backend.
- Regiones: limitadas por damage tracking y expansión de muestras.

El número de pasadas y la resolución son parámetros experimentales, no decisiones inmutables.

**Reglas:**

- No recalcular regiones cuyo contenido de fondo no haya cambiado.
- Si cambia el contenido bajo el vidrio, invalidar las regiones dependientes.
- Evitar blur de ventanas que actualizan continuamente su contenido.
- No permitir cadenas de blur superpuestas sin justificación y medición.

Cuando el fondo contiene vídeo u otra superficie de actualización frecuente:

- Preferir blur desactivado.
- Si se mantiene por decisión de diseño, reducir su resolución y frecuencia de actualización.
- Una resolución de 1/8 lineal será un modo degradado, no una garantía de coste constante.

**Fallback:**

`bg.glass.fallback`, superficie opaca derivada del tema.

La superficie debe mantener contraste y legibilidad.

### 3.4 Atenuación y scrim

**Presupuesto:** ≤ 0,05 ms GPU por output.

**Técnica:**

- Quad con alfa.
- Composición directa.
- Reutilización del pipeline existente.

**Fallback:**

No requiere fallback especial: debe poder representarse mediante una superficie opaca equivalente.

### 3.5 Animaciones de ventana

**Presupuesto:** ≤ 0,3 ms GPU por output.

**Técnica:**

- Transformaciones de superficies existentes.
- Interpolación de posición, escala y opacidad.
- Reutilización de buffers.
- Sin volver a rasterizar contenido estático por cada paso de animación.

Las animaciones no deben obligar a reconstruir la textura de la ventana.

**Fallback:**

- Transición corta mediante opacidad.
- Cambio inmediato cuando el sistema está bajo presión.
- Respeto absoluto de la preferencia de movimiento reducido.

No confundir una animación de superficie con la actualización del contenido de esa superficie.

### 3.6 Overview

**Presupuesto:** ≤ 2,0 ms GPU por output.

**Técnica:**

- Miniaturas cacheadas.
- Actualización incremental.
- Escalado de texturas existentes.
- Actualización máxima inicial de 10 Hz cuando sea necesario.
- No capturar ni reconstruir todas las ventanas en cada frame.

La cache debe invalidarse ante:

- Cambio de contenido.
- Cambio de tamaño.
- Cambio de superficie.
- Destrucción de ventana.
- Cambio de permisos o restricciones de captura.

**Fallback:**

- Miniaturas estáticas.
- Reducción de resolución.
- Desactivar efectos secundarios.
- Mantener una representación identificable de cada ventana.

Las miniaturas deben respetar las restricciones de privacidad y captura de contenido.

No mostrar contenido protegido ni superficies excluidas de captura.

### 3.7 Filtros de accesibilidad

**Presupuesto:** ≤ 0,5 ms GPU por output.

**Técnica:**

- Pipeline de postprocesamiento único cuando sea compatible.
- Reutilización de shaders y recursos.
- Evitar pasadas independientes innecesarias.

**Fallback:**

Cuando no pueda mantenerse el efecto, conservar una alternativa accesible siempre que sea técnicamente posible.

Los filtros necesarios para accesibilidad no pueden eliminarse silenciosamente por presión de rendimiento.

La compatibilidad y la prioridad de estos filtros deben definirse en su especificación funcional.

## 4. Damage tracking

El compositor y `px-ui` deben evitar renderizados innecesarios.

### 4.1 Compositor

Debe identificar:

- Superficies modificadas.
- Regiones dañadas.
- Cambios de geometría.
- Cambios de opacidad.
- Cambios de clipping.
- Cambios de sombra.
- Cambios de efectos.
- Cambios de stacking.

El damage de una superficie debe propagarse a los efectos dependientes.

Ejemplo: si cambia el contenido detrás de una ventana con blur, la región de blur afectada también debe invalidarse.

No basta con redibujar exclusivamente el rectángulo original de la superficie.

### 4.2 Shell y px-ui

Cada componente debe declarar sus dependencias visuales y evitar invalidaciones globales.

Un cambio local no debe provocar el redibujado de todo el Shell.

Los relojes, indicadores y widgets deben actualizar únicamente sus regiones.

### 4.3 Reposo

Cuando no existen cambios visuales:

- No solicitar nuevos frames.
- No ejecutar ciclos de renderizado vacíos.
- No mantener animaciones infinitas.
- No provocar wakeups periódicos innecesarios.

El compositor debe distinguir entre:

- Inactividad del compositor.
- Actividad de aplicaciones clientes.
- Actividad de vídeo o contenido externo.
- Presentaciones requeridas por el sistema.

La regla de cero frames se refiere a ausencia de trabajo nuevo solicitado por el compositor, no a que toda la GPU del equipo tenga que registrar 0 % de utilización.

## 5. Reglas de composición

### 5.1 Blur y contenido dinámico

No aplicar blur continuamente sobre contenido que cambia cada frame salvo que exista una justificación visual y un presupuesto verificado.

Para vídeo:

- Detectar actualizaciones frecuentes mediante damage tracking.
- Evitar invalidar toda la cadena de blur si solo cambia una región.
- Aplicar fallback opaco cuando la carga sea excesiva.

No depender exclusivamente de identificar el contenido como vídeo: cualquier superficie con actualizaciones frecuentes debe tratarse de manera equivalente.

### 5.2 Fullscreen y direct scanout

Cuando una superficie fullscreen pueda utilizar direct scanout:

- Evitar composición adicional.
- Desactivar efectos que impidan la ruta directa.
- Evitar blur, sombras y clipping decorativo.
- Evitar transformaciones incompatibles.

Direct scanout depende de:

- Formato del buffer.
- Modificadores.
- DRM planes disponibles.
- Escala y transformación.
- Sincronización.
- Superficies superpuestas.
- Capacidades del driver y del hardware.

El compositor debe verificar la elegibilidad y no asumir que toda ventana fullscreen puede utilizar direct scanout.

Si no es posible, utilizar composición normal con el mínimo de efectos.

### 5.3 Texto y glifos

Evitar rasterizaciones innecesarias.

Cada proceso que renderice texto debe utilizar mecanismos de cache apropiados para su toolkit.

`px-ui` puede mantener un atlas de glifos reutilizable dentro de su renderer.

No asumir que un único atlas GPU puede compartirse directamente entre compositor, Shell y aplicaciones de terceros.

Durante animaciones de ventanas:

- Transformar la superficie ya renderizada.
- No volver a rasterizar texto estático.
- Reutilizar buffers cuando sea posible.

El cambio de escala debe preservar una legibilidad aceptable.

### 5.4 Widgets

Cada widget declara:

- Frecuencia mínima de actualización.
- Fuentes de eventos.
- Dependencias visuales.
- Coste aproximado.
- Prioridad.
- Política de actualización en reposo.

No permitir polling continuo cuando existe una fuente de eventos.

Un widget que provoca más de un redraw por segundo sin una razón funcional en reposo debe considerarse una regresión.

Excepciones justificadas:

- Relojes.
- Animaciones activas.
- Indicadores de actividad.
- Datos que cambian realmente.
- Actualizaciones explícitamente solicitadas.

La frecuencia de actualización de datos y la frecuencia de renderizado deben ser independientes.

## 6. Degradación automática

El compositor puede reducir efectos decorativos cuando detecta presión de renderizado.

La degradación no debe modificar silenciosamente las preferencias persistentes del usuario.

### 6.1 Métricas

Mantener una ventana deslizante de 2 segundos.

Medir:

- CPU p50, p95 y p99.
- GPU p50, p95 y p99.
- Tiempo de presentación.
- Frames perdidos.
- Missed deadlines.
- Esperas de sincronización.
- Damage por frame.
- Coste por efecto cuando sea medible.

La medición GPU debe utilizar mecanismos compatibles con el backend y el driver.

Si la medición GPU no está disponible, indicar estado desconocido. No interpretar ausencia de datos como coste cero.

### 6.2 Niveles de calidad

**FULL**

- Blur normal.
- Sombras completas.
- Animaciones configuradas.
- Overview con miniaturas dinámicas.

**REDUCED**

- Blur a resolución reducida.
- Sombras simplificadas.
- Menos actualizaciones de miniaturas.
- Animaciones decorativas simplificadas.

**MINIMAL**

- Blur desactivado.
- Sombras desactivadas o mínimas.
- Transiciones simples.
- Overview estático.
- Composición básica prioritaria.

Los niveles no modifican la preferencia de accesibilidad `reduced_motion`.

### 6.3 Umbrales iniciales

Los umbrales siguientes son parámetros iniciales que deben calibrarse en HW-A.

| Condición sostenida | Acción |
|---|---|
| GPU p95 supera 80 % del presupuesto GPU | FULL → REDUCED |
| CPU p95 supera 80 % del presupuesto CPU | Reducir trabajo del Shell y efectos |
| GPU p95 supera 95 % del presupuesto o missed deadlines > 5 % | REDUCED → MINIMAL |
| Frames perdidos > 5 % durante la ventana de evaluación | Activar degradación inmediata |
| Carga elevada durante 30 segundos | Registrar incidente y notificar Observatory |

Los umbrales deben considerar tanto los presupuestos absolutos como el intervalo de presentación real.

No utilizar exclusivamente el porcentaje del intervalo de 16,67 ms para decidir si se supera el presupuesto de 4 ms.

### 6.4 Recuperación e histéresis

Recuperar calidad cuando:

- CPU y GPU estén por debajo del 60 % de sus presupuestos.
- No existan missed deadlines relevantes.
- La condición se mantenga durante al menos 10 segundos.

Recuperación gradual:

1. MINIMAL → REDUCED.
2. Evaluar estabilidad.
3. REDUCED → FULL.
4. Evaluar nuevamente.

Evitar oscilaciones continuas entre perfiles.

### 6.5 Batería

Cuando la batería sea inferior al 20 %:

- Puede activarse transparencia reducida si el usuario habilitó la modalidad automática.
- Se pueden reducir efectos decorativos.
- No modificar las preferencias persistentes.
- No desactivar funciones de accesibilidad.

El umbral debe ser configurable.

### 6.6 Renderizado por software

En renderizado por software:

- Desactivar blur.
- Desactivar sombras costosas.
- Desactivar transformaciones de escala animadas.
- Priorizar composición básica.
- Utilizar transiciones sencillas y cortas cuando sean compatibles.

La sesión debe seguir siendo funcional.

No desactivar indicadores esenciales, foco, contraste ni funcionalidades de accesibilidad.

## 7. Perfiles de hardware

`hardware-profiles` puede proporcionar un perfil de efectos inicial:

```toml
[graphics]
effects = "reduced"
```

Valores permitidos:

- `full`
- `reduced`
- `minimal`

Los perfiles son recomendaciones iniciales, no una verdad permanente sobre la GPU.

La selección debe considerar:

- GPU.
- Driver.
- Versión de kernel.
- Versión de Mesa o driver propietario.
- Backend de composición.
- Resolución.
- Número de outputs.
- Formatos y modificadores.
- Capacidades de sincronización.

No fijar universalmente `reduced` para Intel Gen9 ni asumir que una GPU de una generación determinada tiene siempre el mismo rendimiento.

El rendimiento real debe medirse en ejecución.

El perfil de hardware establece el máximo nivel inicial permitido. La degradación dinámica puede reducirlo temporalmente.

## 8. Observabilidad

`parallax-compositor` debe exponer métricas a `Observatory`.

Métricas mínimas:

- `frame.cpu_time_ms`
- `frame.gpu_time_ms`
- `frame.presentation_interval_ms`
- `frame.missed_deadlines`
- `frame.dropped_frames`
- `render.damage_area`
- `render.damage_regions`
- `effects.blur_cost_ms`
- `effects.shadow_cost_ms`
- `effects.overview_cost_ms`
- `effects.quality_level`
- `effects.degradation_reason`
- `effects.scanout_active`

Los nombres son identificadores conceptuales y deben normalizarse con el contrato definitivo de métricas.

Los datos deben incluir:

- Timestamp.
- Output.
- Backend.
- Resolución.
- Frecuencia de refresco.
- Nivel de calidad.
- Estado de disponibilidad de medición.

No registrar métricas de alta frecuencia mediante logs de texto por frame.

Utilizar agregación y coalescencia.

## 9. Benchmarks específicos

Todos los escenarios se ejecutan en HW-A y se repiten en HW-B cuando esté disponible.

### Escenario A: ventanas apiladas

- Diez ventanas.
- Sombras activadas.
- Esquinas redondeadas.
- Movimiento y cambio de foco.
- Alternancia entre ventanas.
- Sin contenido animado.

Medir coste individual y agregado.

### Escenario B: Shell sobre vídeo

- Topbar.
- Dock.
- Control Center abierto.
- Vídeo 4K.
- Vidrio habilitado.
- Movimiento y daño de contenido.
- Cambios de foco.

Comparar:

1. Blur normal.
2. Blur a 1/8 lineal.
3. Blur desactivado.
4. Superficie opaca.

Registrar consumo GPU, latencia y frames perdidos.

### Escenario C: Overview

- Veinte ventanas.
- Miniaturas cacheadas.
- Animación de entrada.
- Movimiento entre ventanas.
- Actualización de contenido.
- Salida del overview.

Medir el coste inicial, sostenido y de actualización de miniaturas.

### Escenario D: cambio de workspace

- Cambio continuo durante 60 segundos.
- Diez ventanas distribuidas entre workspaces.
- Animaciones activadas.
- Diferentes estados de ventana.
- Varias superficies con contenido dinámico.

Medir CPU, GPU, missed deadlines y latencia de entrada.

### Escenario E: reposo

- Shell visible.
- Dock y topbar.
- Widgets configurados.
- Aplicaciones abiertas sin cambios visuales.
- Cinco minutos de observación.

Validar que no existen frames vacíos, polling innecesario ni wakeups de renderizado injustificados.

### Escenario F: presión de recursos

- Carga de CPU.
- Carga GPU.
- Vídeo.
- Cambio de workspace.
- Overview.
- Varias ventanas.

Validar degradación, recuperación e histéresis.

### Escenario G: accesibilidad

- Alto contraste.
- Movimiento reducido.
- Escalado de texto.
- Filtros de accesibilidad.
- Cambios de tema.
- Transparencia reducida.

Verificar que la degradación no elimina funciones esenciales.

## 10. Metodología

Seguir `docs/benchmarks/metodologia.md`.

Registrar:

- Hardware exacto.
- Kernel.
- Driver.
- Mesa.
- Backend.
- Resolución.
- Frecuencia.
- Escala.
- Configuración de efectos.
- Temperatura y estado térmico.
- Perfil energético.
- Carga de otras aplicaciones.
- Número de ejecuciones.

Usar como mínimo cinco ejecuciones para escenarios reproducibles y diez cuando se evalúen regresiones pequeñas.

Informar mediana, p95, p99 y dispersión cuando las métricas lo permitan.

No comparar resultados obtenidos con configuraciones de hardware, drivers o frecuencia diferentes sin documentar las diferencias.

## 11. Política de aprobación

`ingeniero-rendimiento` puede bloquear:

- Efectos sin fallback.
- Efectos sin benchmark.
- Efectos que excedan su presupuesto.
- Efectos que generen renderizado innecesario.
- Efectos que afecten a la latencia de entrada.
- Efectos que produzcan regresiones reproducibles.
- Efectos incompatibles con los perfiles de accesibilidad.

Para aprobar un efecto se requiere:

1. Implementación funcional.
2. Medición reproducible.
3. Cumplimiento del presupuesto individual.
4. Cumplimiento del presupuesto agregado.
5. Fallback validado.
6. Prueba bajo presión.
7. Compatibilidad con movimiento reducido.
8. Evidencia documentada.

Un efecto no se considera aprobado únicamente porque funcione correctamente en una GPU de desarrollo.

## 12. Estructura documental

```text
docs/
└── benchmarks/
    ├── metodologia.md
    ├── rendering-effects.md
    ├── rendering-results/
    │   ├── hw-a/
    │   └── hw-b/
    └── regressions/
```

La implementación debe mantener sincronizados los presupuestos, los resultados y las decisiones de degradación.

**Estado: Propuesto. Pendiente de aprobación arquitectónica y validación experimental.**