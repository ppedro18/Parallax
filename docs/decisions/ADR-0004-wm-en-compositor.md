# ADR-0004: Gestor de ventanas dentro del proceso del compositor como biblioteca de lógica pura

- **Estado:** Propuesto
- **Fecha:** 2026-11-19
- **Ámbito:** Gestión de ventanas, distribución espacial, foco, tiling y workspaces.

## 1. Contexto

PARALLAX necesita un gestor de ventanas propio que controle completamente el comportamiento espacial del escritorio.

El sistema debe gestionar:

- Posicionamiento de ventanas.
- Tiling y floating.
- Workspaces.
- Foco y navegación.
- Maximización y minimización.
- Reglas de distribución.
- Jerarquía de ventanas.
- Transiciones entre estados.
- Coordinación con el sistema de profundidad PARALLAX.

La gestión de ventanas está directamente relacionada con la composición gráfica y las animaciones.

Un gestor externo introduce una frontera de comunicación adicional entre el compositor y el Window Manager, que puede complicar la sincronización de eventos, configuraciones y estados visuales.

Se necesita una arquitectura determinista, de baja latencia y fácilmente verificable.

## 2. Opciones evaluadas

| Alternativa | Ventajas | Desventajas |
|---|---|---|
| WM externo mediante protocolo | Aislamiento de procesos y reinicios independientes | IPC, sincronización y estados distribuidos |
| WM dentro del compositor | Baja latencia y estado unificado | Fallos compartidos y mayor responsabilidad del proceso |

## 3. Decisión

El Window Manager se implementará dentro del proceso del compositor mediante una biblioteca propia denominada **`px-wm`**.

`px-wm` será una biblioteca de lógica pura, determinista y sin efectos secundarios externos.

No tendrá acceso directo a:

- Wayland.
- Smithay.
- DRM/KMS.
- GPU.
- Dispositivos de entrada.
- Archivos.
- Red.
- IPC.
- Servicios externos.

Su responsabilidad será recibir un estado y una acción, procesarlos y devolver el nuevo estado junto con las operaciones necesarias para actualizar la composición.

## 4. Arquitectura funcional

El modelo de funcionamiento será:

```text
Eventos externos
       |
       v
PARALLAX Compositor
       |
       v
Normalización de eventos
       |
       v
       px-wm
   (Lógica pura)
       |
       v
Nuevo estado + efectos
       |
       v
Adaptador del compositor
       |
       v
Smithay / Wayland
       |
       v
Composición y presentación
```

`px-wm` no ejecutará directamente las operaciones gráficas.

El compositor interpretará los resultados y aplicará los cambios correspondientes mediante Smithay.

## 5. Modelo de estado

El Window Manager mantendrá un estado explícito que incluya:

- Ventanas conocidas.
- Identificadores estables.
- Estado de cada ventana.
- Geometría.
- Workspace asociado.
- Monitor asociado.
- Ventana enfocada.
- Orden de apilamiento.
- Reglas de distribución.
- Estado de tiling.
- Estado de floating.
- Configuración espacial.

Las estructuras internas deberán utilizar tipos propios de PARALLAX y evitar dependencias de Smithay.

### 5.1 Modelo de acciones

Las modificaciones se expresarán mediante acciones explícitas.

Ejemplos:

- `WindowCreated`
- `WindowDestroyed`
- `WindowFocused`
- `WindowMoved`
- `WindowResized`
- `WindowMaximized`
- `WindowMinimized`
- `WindowToggledFloating`
- `WorkspaceCreated`
- `WorkspaceSwitched`
- `WorkspaceMoved`
- `LayoutChanged`
- `MonitorAdded`
- `MonitorRemoved`

Cada acción deberá producir una transición de estado verificable.

### 5.2 Modelo de resultados

El procesamiento devolverá:

1. Nuevo estado.
2. Cambios geométricos.
3. Cambios de foco.
4. Cambios de workspace.
5. Cambios de apilamiento.
6. Efectos que deberá ejecutar el compositor.

Los efectos no deberán ejecutarse dentro de `px-wm`.

## 6. Determinismo

Para una misma configuración, estado inicial y secuencia de acciones, `px-wm` deberá producir siempre el mismo resultado.

Reglas:

- Evitar estado global mutable.
- Evitar acceso al reloj del sistema.
- Evitar aleatoriedad implícita.
- Evitar operaciones de E/S.
- Evitar tareas asíncronas internas.
- Evitar dependencias de orden de iteración no garantizado.
- Utilizar identificadores y ordenamientos explícitos.

Las animaciones recibirán el tiempo transcurrido como dato de entrada cuando corresponda.

## 7. Integración con el sistema Parallax

`px-wm` será responsable de la geometría y las relaciones espaciales de las ventanas.

El sistema de profundidad será responsable de interpretar los niveles de elevación y sus transiciones visuales.

El compositor coordinará ambos sistemas.

La arquitectura deberá permitir:

- Elevar una ventana activa.
- Reducir la prominencia de ventanas inactivas.
- Coordinar cambios de foco con animaciones.
- Transicionar entre floating y tiling.
- Cambiar de workspace con continuidad espacial.
- Reorganizar ventanas al modificar la configuración de monitores.

La lógica geométrica no deberá depender del motor de animaciones.

## 8. Tiling y layouts

El gestor deberá admitir diferentes estrategias de distribución:

- Tiling automático.
- Floating.
- Master-stack.
- Dwindle.
- Monocle.
- Layout configurable.
- Reglas personalizadas de distribución.

Los layouts deberán operar sobre un modelo espacial independiente del renderizado.

La incorporación de nuevos algoritmos de distribución no deberá requerir modificar la infraestructura Wayland.

## 9. Seguridad y robustez

El gestor forma parte de un proceso crítico.

Un fallo en `px-wm` puede comprometer la sesión completa, por lo que se establecerán las siguientes medidas:

- Evitar `panic!` en rutas de procesamiento de eventos.
- Utilizar `Result` para errores recuperables.
- Validar dimensiones y geometrías.
- Evitar desbordamientos aritméticos.
- Establecer límites razonables de ventanas y workspaces.
- Validar identificadores y referencias.
- Evitar estados parcialmente actualizados.
- Aplicar transiciones de estado atómicas desde la perspectiva del modelo.
- Mantener invariantes verificables.

Los errores derivados de eventos externos deberán rechazarse o manejarse de forma controlada.

## 10. Estrategia de pruebas

`px-wm` deberá ser completamente testeable sin iniciar Wayland ni Smithay.

Se establecerán:

### Pruebas unitarias
Para acciones, layouts, foco, geometría y transiciones.

### Pruebas basadas en propiedades
Para validar invariantes del estado y operaciones geométricas.

### Fuzzing
Para secuencias arbitrarias de acciones y entradas malformadas.

### Pruebas de regresión
Para conservar comportamientos espaciales previamente validados.

### Pruebas deterministas
Para comprobar que una misma secuencia de eventos genera resultados idénticos.

### Pruebas de rendimiento
Para medir latencia de procesamiento, asignaciones de memoria y comportamiento con grandes cantidades de ventanas.

## 11. Rendimiento

El gestor deberá minimizar:

- Asignaciones innecesarias.
- Clonaciones de estructuras grandes.
- Recálculos geométricos redundantes.
- Operaciones costosas en cada evento.
- Bloqueos de ejecución.

Las optimizaciones deberán estar justificadas mediante benchmarks.

No se priorizará complejidad de implementación sobre mejoras de rendimiento demostrables.

## 12. Consecuencias

### Positivas

- Baja latencia.
- Estado centralizado.
- Mayor coherencia entre ventanas y compositor.
- Coordinación directa de animaciones.
- Arquitectura determinista.
- Facilidad de pruebas.
- Independencia de la infraestructura Wayland.

### Negativas

- Un fallo grave puede afectar toda la sesión.
- Mayor responsabilidad de robustez.
- Necesidad de fuzzing.
- El aislamiento de procesos es menor que con un WM externo.
- Requiere definir cuidadosamente las fronteras entre lógica y efectos.

## 13. Criterios de aceptación

La implementación será válida cuando:

1. `px-wm` pueda compilarse como biblioteca independiente.
2. No dependa de Smithay ni Wayland.
3. No realice operaciones de E/S.
4. Procese acciones mediante transiciones deterministas.
5. Mantenga invariantes verificables.
6. Soporte tiling y floating.
7. Gestione workspaces y múltiples monitores.
8. Supere pruebas unitarias y basadas en propiedades.
9. Supere campañas de fuzzing.
10. No produzca pánicos ante entradas externas inválidas.
11. Disponga de benchmarks reproducibles.

## 14. Revisión

Este ADR deberá revisarse cuando:

- La complejidad del Window Manager justifique aislamiento independiente.
- Existan problemas de estabilidad atribuibles a `px-wm`.
- Se requiera paralelizar partes del procesamiento.
- Los requisitos de rendimiento cambien sustancialmente.

## 15. Principio rector

**El Window Manager de PARALLAX será una máquina de estados determinista, no un servicio externo que controla el escritorio mediante mensajes.**

La lógica espacial será propia, verificable y completamente independiente de la infraestructura gráfica.