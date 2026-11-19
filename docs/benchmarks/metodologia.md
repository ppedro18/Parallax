# Metodología de rendimiento y presupuestos — PARALLAX

**Estado:** Propuesto · Pendiente de validación experimental y aprobación.

## 1. Objetivo

Establecer una metodología reproducible para medir, comparar y controlar:

- Consumo de memoria.
- Uso de CPU.
- Latencia de interacción.
- Rendimiento de renderizado.
- Tiempo de arranque.
- Consumo energético.
- Rendimiento de aplicaciones y componentes del Shell.
- Regresiones introducidas por cambios arquitectónicos o dependencias.

Los presupuestos son objetivos de ingeniería, no garantías de rendimiento.

Ningún componente se considerará optimizado únicamente por superar un microbenchmark. La aceptación requiere pruebas de integración sobre el sistema real.

## 2. Hardware de referencia

### HW-A — Equipo de referencia principal

- Arquitectura: x86_64.
- CPU: portátil Intel o AMD.
- GPU: iGPU Intel o AMD.
- RAM: 8 GB.
- Almacenamiento: NVMe.
- Pantalla: 1920×1080 a 60 Hz.
- Escenario alternativo: 2560×1600 con escala 1,5.

Este equipo determina los presupuestos principales de memoria, CPU y experiencia de usuario.

### HW-B — Equipo de referencia secundario

- Arquitectura: x86_64.
- GPU: AMD dedicada.
- RAM: 16 GB.
- Dos monitores.
- Almacenamiento SSD/NVMe.

Se utiliza para comprobar comportamiento con GPU dedicada, múltiples salidas, composición y cargas gráficas.

### VM — Entorno de CI

- QEMU/KVM.
- virtio-gpu con virgl.
- RAM: 4 GB.

La VM se utiliza para:

- Tests funcionales.
- Detección de regresiones relativas.
- Benchmarks de lógica y estructuras de datos.
- Pruebas de integración.
- Compilaciones y validaciones automatizadas.

**La VM no determina los presupuestos absolutos de rendimiento.**

No se utilizará como evidencia de consumo energético, latencia física, rendimiento real de GPU o compatibilidad DRM/KMS de hardware.

## 3. Escenarios de medición

### 3.1 Escenario de reposo integrado

Definición:

- Arranque completo del sistema.
- Sesión de usuario iniciada.
- Compositor y Shell activos.
- Topbar y dock visibles.
- Cuatro widgets predeterminados.
- Files abierto con una ventana.
- Terminal abierta con una ventana y una pestaña.
- Red conectada.
- Pantalla encendida.
- Cinco minutos de estabilización e inactividad.
- Sin otras aplicaciones abiertas.
- Sin actualizaciones, indexaciones ni operaciones de mantenimiento en segundo plano.

Este escenario determina los presupuestos principales del sistema.

### 3.2 Escenario de sesión mínima

- Compositor activo.
- Shell activo.
- Servicios esenciales.
- Sin aplicaciones abiertas.
- Sin widgets opcionales.

Se utiliza para conocer el coste intrínseco de PARALLAX.

### 3.3 Escenario de interacción

- Launcher abierto y cerrado repetidamente.
- Apertura y cierre de ventanas.
- Cambio de workspace.
- Redimensionamiento.
- Movimiento de ventanas.
- Scroll continuo.
- Cambio de monitor.
- Interacción con menús y widgets.

Se utiliza para medir latencia y fluidez.

### 3.4 Escenario de presión

- Apertura de varias aplicaciones.
- Múltiples ventanas.
- Consumo progresivo de memoria.
- Actividad de almacenamiento.
- Presión de memoria mediante PSI.
- Cierre y recuperación de aplicaciones.

Se utiliza para evaluar degradación controlada, respuesta de `resourced` y coexistencia con systemd-oomd.

## 4. Metodología de memoria

### 4.1 Métrica principal del sistema

Utilizar:

`MemTotal - MemAvailable`

Obtenido de `/proc/meminfo`.

Esta métrica representa una estimación de memoria no disponible para nuevas asignaciones sin recuperar memoria, y evita contar como ocupada toda la caché potencialmente recuperable.

**No restar nuevamente la caché reclamable.**

La caché, los buffers y la memoria reclamable se reportan como información adicional.

### 4.2 Desglose obligatorio

Reportar las siguientes métricas por separado.

**Procesos**

- PSS de cada proceso.
- RSS de cada proceso.
- Suma de PSS del conjunto de procesos medidos.
- Memoria anónima.
- Memoria compartida.
- Número de procesos e hilos.

Fuente: `/proc/<pid>/smaps_rollup`.

**Kernel**

Cuando estén disponibles:

- `Slab`.
- `SReclaimable`.
- `SUnreclaim`.
- `KernelStack`.
- `PageTables`.
- `VmallocUsed`.
- Memoria de otros subsistemas relevante.

**zram**

Obtener los datos de `/sys/block/zram*/mm_stat` y las estadísticas de swap.

Reportar:

- Memoria original almacenada.
- Memoria comprimida.
- Memoria física utilizada.
- Ratio de compresión.
- Actividad de lectura y escritura.

**GPU**

Cuando el controlador lo permita:

- Memoria VRAM.
- Memoria GTT.
- Memoria por proceso.
- Memoria compartida.
- Identificación del dispositivo y controlador.

Utilizar `/proc/<pid>/fdinfo` y los campos `drm-memory-*` cuando estén disponibles.

La ausencia de estadísticas de un controlador debe registrarse como dato no disponible, nunca como consumo cero.

### 4.3 Reglas de contabilización

- No sumar PSS, memoria de kernel, zram y GPU para crear una cifra de memoria total sin demostrar que no existen solapamientos.
- No interpretar RSS como memoria física exclusiva.
- No considerar memoria libre como indicador principal de eficiencia.
- No comparar procesos con permisos o visibilidad de `/proc` diferentes.
- Registrar el kernel, controlador y versión de herramientas utilizados.

### 4.4 Objetivo principal

<badge color="success">Objetivo de ingeniería</badge>

`MemTotal - MemAvailable < 1 GB`

Para el escenario de reposo integrado sobre HW-A.

Es un objetivo que debe validarse experimentalmente. Se reportará también el escenario mínimo y el desglose por proceso.

### 4.5 Repetibilidad

- Cinco ejecuciones independientes como mínimo.
- Registrar mediana y dispersión.
- Mantener iguales las condiciones de hardware y software.
- Registrar resultados individuales.
- No descartar ejecuciones sin documentar el motivo.

Para cambios de memoria importantes, realizar diez o más ejecuciones.

## 5. Presupuestos v1 — HW-A

Los siguientes valores son objetivos iniciales sujetos a validación.

| Componente | PSS objetivo | CPU en reposo |
|---|---:|---:|
| Compositor | ≤ 80 MB | < 0,2 % |
| Shell | ≤ 120 MB | < 0,5 % |
| settingsd + notifyd + resourced | ≤ 30 MB | < 0,1 % |
| hardwared + diagd | ≤ 20 MB | Próximo a 0 % |
| Files, una ventana | ≤ 60 MB | Próximo a 0 % |
| Terminal, una pestaña | ≤ 40 MB | Próximo a 0 % |

Los presupuestos PSS se aplican a cada proceso o conjunto indicado, no al total del sistema.

### Reglas de CPU

- Medir porcentaje normalizado respecto de un núcleo lógico.
- Registrar también tiempo de CPU acumulado.
- Medir durante al menos cinco minutos en reposo.
- Separar actividad periódica esperada de actividad inesperada.
- No permitir polling continuo innecesario.
- Identificar wakeups por proceso cuando sea posible.

Objetivo del Shell: comportamiento predominantemente dirigido por eventos, sin ciclos de polling innecesarios.

### Objetivos adicionales

| Métrica | Objetivo |
|---|---:|
| Memoria del sistema en reposo | < 1 GB |
| CPU total en reposo | < 1 % |
| Arranque a sesión utilizable | ≤ 8 s |
| Primer frame de Files | ≤ 250 ms |
| Latencia de entrada de Terminal | < 10 ms |
| Apertura del launcher | ≤ 100 ms |
| Frame de CPU del compositor | ≤ 4 ms |

Los tiempos de apertura y primer frame se medirán desde un estado definido y con caché fría y caliente reportadas por separado.

El presupuesto de 4 ms corresponde al trabajo de CPU del compositor, no al tiempo completo de presentación del frame.

## 6. Latencia y renderizado

### 6.1 Latencia de interacción

Objetivo:

`input → presentación del frame, p99 < 16 ms`

Medir:

- Recepción del evento de entrada.
- Procesamiento del evento.
- Actualización del estado.
- Preparación del frame.
- Renderizado.
- Presentación.

Utilizar timestamps de presentación de Wayland cuando estén disponibles.

Los resultados deben distinguir entre:

- Latencia interna del compositor.
- Latencia de presentación.
- Latencia física input-to-photon, si se dispone de instrumentación externa.

No afirmar que los timestamps de presentación equivalen a latencia física de pantalla.

### 6.2 Frame pacing

Medir:

- Frame time promedio.
- p50, p95 y p99.
- Frames perdidos.
- Frames tardíos.
- Tiempo de CPU por frame.
- Tiempo de GPU cuando sea posible.
- Sincronización con VSync.

Evaluar a 60 Hz y en configuraciones adicionales disponibles.

### 6.3 Apertura del launcher

Medir desde la solicitud de apertura hasta el primer frame visible.

Reportar:

- Mediana.
- p95.
- p99.
- Caché fría.
- Caché caliente.

Objetivo inicial: ≤ 100 ms.

## 7. Tiempo de arranque

Objetivo inicial:

`Boot → sesión utilizable ≤ 8 s`

Definir los hitos:

1. Inicio del firmware y bootloader.
2. Inicio del kernel.
3. Inicio de systemd.
4. Servicios esenciales preparados.
5. Compositor iniciado.
6. Shell visible.
7. Sesión utilizable.

Reportar por separado el tiempo de firmware/bootloader y el tiempo de userspace.

Utilizar `systemd-analyze` y registros de inicio de los componentes propios.

No considerar una sesión utilizable únicamente porque el proceso del Shell se haya iniciado: debe ser posible interactuar con el escritorio.

## 8. Energía

Objetivo inicial:

`Consumo de reposo PARALLAX ≤ Linux base + 0,5 W`

La comparación debe realizarse:

- En el mismo equipo.
- Con el mismo kernel.
- Con el mismo firmware.
- Con el mismo brillo.
- Con la misma frecuencia de pantalla.
- Con la misma conectividad.
- Con el mismo perfil energético.
- Con los mismos periféricos.
- Durante una ventana de medición equivalente.

Reportar:

- Consumo promedio.
- Variación.
- Consumo en batería.
- Consumo conectado a corriente, cuando corresponda.
- Estado de suspensión y reanudación por separado.

No atribuir al Shell diferencias energéticas provocadas por controladores, firmware o políticas de energía distintas.

## 9. Herramientas

### Sistema y arranque

- `systemd-analyze`
- `/proc/meminfo`
- `/proc/<pid>/smaps_rollup`
- `systemd-cgtop`
- PSI de `/proc/pressure/`

### CPU y perfiles

- `perf`
- `bpftrace`
- `turbostat`

### Memoria

- `heaptrack`
- `bytehound`
- herramientas de análisis de `smaps`

### Energía

- `powertop`
- `turbostat`
- interfaces de energía disponibles en el hardware

### GPU y renderizado

- `intel_gpu_top`
- `radeontop`
- estadísticas DRM disponibles.
- Feedback de presentación de Wayland.

### Trazas

- `tracing`
- `tracy`, mediante feature opcional `profiling`.

El profiling debe poder deshabilitarse en builds de producción. Su sobrecarga debe medirse y documentarse.

## 10. Benchmark de interfaz — ADR-0005

La selección del toolkit debe realizarse con una aplicación de referencia real.

### Escenario

- Ventana vacía.
- Lista virtualizada de 10 000 filas.
- Scroll continuo.
- 200 botones con estados hover.
- Panel con blur.
- Texto multilingüe:
  - Latín.
  - Árabe y escritura RTL.
  - CJK.
  - Emoji.
- Navegación por teclado.
- Accesibilidad mediante árbol de accesibilidad.
- Escalado de interfaz.

### Métricas

- PSS.
- RSS.
- Tiempo al primer frame.
- Frame time p95 y p99.
- CPU durante scroll.
- Uso de GPU.
- Tiempo de respuesta de eventos.
- Memoria al cargar y liberar recursos.
- Comportamiento de virtualización.
- Correcta representación de texto.
- Integración de accesibilidad.

### Criterios de evaluación

No seleccionar un toolkit únicamente por su rendimiento de renderizado.

Debe cumplir también:

- Accesibilidad funcional.
- Soporte de teclado.
- IME y entrada de texto.
- RTL.
- Multilingüismo.
- Escalado.
- Integración con Wayland.
- Compatibilidad con layer-shell cuando sea necesaria.
- Licencia compatible con ADR-0008.
- Mantenimiento y estabilidad.

Los umbrales concretos de aceptación del toolkit se aprobarán después de ejecutar el benchmark sobre HW-A.

## 11. Regresión de rendimiento

### CI en VM

Ejecutar:

- Tests unitarios.
- Benchmarks de algoritmos.
- Benchmarks de estructuras de datos.
- Benchmarks de serialización.
- Benchmarks de IPC.
- Pruebas de integración.
- Benchmarks de lógica de ventanas.
- Benchmarks de configuración.

Las comparaciones serán relativas al mismo entorno de ejecución.

No utilizar la VM para aprobar presupuestos absolutos de memoria, energía o renderizado.

### Regresión

Una degradación superior al 5 % en un benchmark estable debe generar una alerta y revisión.

El bloqueo automático de fusiones requiere:

- Benchmark reproducible.
- Variabilidad conocida.
- Entorno comparable.
- Repeticiones suficientes.
- Evidencia de que la diferencia supera el ruido de medición.

No bloquear por una única ejecución anómala.

### Hardware real

Ejecutar mensualmente sobre HW-A y HW-B:

- Escenario de reposo.
- Escenario de interacción.
- Escenario de presión.
- Benchmark de interfaz.
- Energía.
- Latencia.
- GPU y múltiples monitores.

Ejecutar pruebas adicionales cuando cambien Smithay, Mesa, el kernel, el toolkit, el backend de renderizado o componentes críticos.

## 12. Registro de resultados

Cada ejecución debe guardar:

- Identificador del benchmark.
- Fecha.
- Commit de PARALLAX.
- Versión del toolchain.
- Kernel.
- Mesa.
- Controlador gráfico.
- Modelo de CPU y GPU.
- RAM.
- Resolución y frecuencia.
- Configuración del sistema.
- Escenario.
- Herramientas y versiones.
- Resultados individuales.
- Mediana y dispersión.
- Logs relevantes.

Estructura propuesta:

```text
benchmarks/
├── methodology.md
├── budgets.md
├── hardware/
│   ├── hw-a.toml
│   ├── hw-b.toml
│   └── vm.toml
├── suites/
│   ├── memory/
│   ├── cpu/
│   ├── compositor/
│   ├── shell/
│   ├── applications/
│   └── energy/
└── results/
    └── <commit>/<hardware>/<scenario>/
```

## 13. Criterios de aceptación

La metodología se considera implementada cuando:

- Los escenarios están automatizados y documentados.
- Las métricas se obtienen de fuentes identificadas.
- Las ejecuciones son reproducibles.
- Se separan las métricas de procesos, kernel, zram y GPU.
- HW-A y HW-B tienen perfiles de referencia documentados.
- CI detecta regresiones sin confundir ruido con degradación.
- Los benchmarks de toolkit incluyen accesibilidad y texto multilingüe.
- Los resultados se conservan vinculados al commit.
- Los presupuestos tienen evidencia experimental.

**Estado: Propuesto. No se consideran validados los presupuestos hasta completar las primeras mediciones sobre hardware real.**