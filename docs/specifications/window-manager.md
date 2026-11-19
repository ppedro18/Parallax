# Especificación: Gestor de ventanas — `px-wm`

**Componente:** `px-wm`  
**Estado:** Propuesto · Pendiente de aprobación  
**Capa:** C4 — Core  
**Integración:** `parallax-compositor`  
**Arquitectura:** Librería Rust pura y determinista.

## 1. Objetivo

`px-wm` es el motor de gestión de ventanas de PARALLAX.

Administra:

- Ventanas.
- Workspaces.
- Outputs.
- Foco.
- Stacking.
- Geometría.
- Floating.
- Tiling.
- Maximización.
- Fullscreen.
- Minimización.
- Reglas.
- Distribución de ventanas.

No implementa Wayland, DRM/KMS, input físico ni renderizado.

Es una librería independiente del compositor para facilitar pruebas deterministas, fuzzing y evolución de los algoritmos de layout.

## 2. Principios

1. No realiza I/O.
2. No depende de Smithay.
3. No depende de Wayland.
4. No conoce dispositivos físicos.
5. No mantiene timers internos.
6. No utiliza estado global mutable.
7. No ejecuta efectos secundarios.
8. Cada transición mantiene invariantes.
9. Los algoritmos de layout son independientes del ciclo de vida de ventanas.
10. El compositor ejecuta los efectos producidos.

## 3. Arquitectura

```text
parallax-compositor
        |
        v
px-compositor-adapter
        |
        v
      px-wm
        |
        +---- px-wm-types
        |
        +---- px-wm-layouts
        |
        +---- px-wm-rules
```

El compositor transforma eventos Wayland e input en comandos tipados.

`px-wm` procesa los comandos y produce un nuevo estado y una lista de efectos.

El compositor aplica los efectos sobre Smithay y las superficies reales.

## 4. Modelo de estado

Modelo conceptual:

```rust
struct State {
    outputs: Vec<OutputState>,
    windows: SlotMap<WindowId, Window>,
    focus_stack: Vec<WindowId>,
    rules: RuleSet,
}

struct OutputState {
    id: OutputId,
    workspaces: Vec<Workspace>,
    active_workspace: WorkspaceId,
}

struct Workspace {
    id: WorkspaceId,
    mode: WorkspaceMode,
    floating: Vec<WindowId>,
    tiling: TileTree,
    fullscreen: Option<WindowId>,
}
```

La estructura definitiva debe evitar redundancia entre listas, árboles y referencias.

Un mismo dato no debe tener varias fuentes de verdad.

### 4.1 Identificadores

Utilizar identificadores internos estables:

- `WindowId`
- `OutputId`
- `WorkspaceId`

No utilizar PID, título o `app_id` como identificador interno de ventana.

Estos valores pueden cambiar o repetirse.

## 5. Ventanas

Cada ventana mantiene:

- Identificador.
- `app_id`.
- Título.
- Output.
- Workspace.
- Geometría.
- Estado de visibilidad.
- Estado de presentación.
- Transient parent cuando exista.
- Estado de foco.
- Estado de urgencia.
- Reglas aplicables.

Los datos específicos de protocolo permanecen en el adaptador del compositor.

`px-wm` utiliza tipos propios.

## 6. Estados ortogonales

No modelar todos los estados visuales como un único enum.

Separar:

### Geometría

- Floating.
- Tiled.
- Maximized.
- Fullscreen.

### Visibilidad

- Mapped.
- Unmapped.
- Minimized.

### Actividad

- Focused.
- Unfocused.
- Urgent.

### Interacción

- Idle.
- Moving.
- Resizing.
- Modal.

Una ventana puede estar maximizada y enfocada simultáneamente.

Una ventana puede ser urgente y no estar enfocada.

El modelo debe representar estas combinaciones sin estados contradictorios.

## 7. Workspaces

Cada output contiene uno o más workspaces.

Un workspace tiene:

- Identificador.
- Modo.
- Ventanas flotantes.
- Árbol de tiling.
- Ventana fullscreen.
- Estado de layout.
- Configuración.

Modos:

- Floating.
- Tiling.

El modo se puede configurar por workspace.

### 7.1 Cambio de workspace

Al cambiar:

1. Actualizar workspace activo.
2. Recalcular visibilidad.
3. Actualizar foco.
4. Generar operaciones de configuración.
5. Generar efectos de transición.
6. Notificar cambios.

No ejecutar animaciones dentro de `px-wm`.

El motor produce efectos declarativos que el compositor interpreta.

### 7.2 Crear y eliminar

Crear workspaces dinámicamente.

Al eliminar un workspace, migrar sus ventanas mediante una política explícita.

No destruir ventanas solo porque desaparezca su workspace.

## 8. Floating

Modo predeterminado de PARALLAX.

Comportamiento:

- Ventanas nuevas centradas en el output del cursor.
- Posición y tamaño respetan límites.
- Snap a bordes.
- Mitades y cuartos.
- Superposición controlada.
- Diálogos flotantes cuando corresponda.

El centrado debe utilizar información del compositor recibida como entrada; `px-wm` no consulta el cursor directamente.

### 8.1 Snap

Operaciones:

- SnapLeft.
- SnapRight.
- SnapTop.
- SnapBottom.
- SnapTopLeft.
- SnapTopRight.
- SnapBottomLeft.
- SnapBottomRight.

El snap modifica geometría y estado según la política de Floating.

No debe confundirse con un layout de tiling persistente.

## 9. Tiling

El usuario puede activar tiling por workspace.

Algoritmos iniciales:

- Dwindle.
- Columns.

Los algoritmos pertenecen a `px-wm-layouts`.

### 9.1 Dwindle

Divide progresivamente el espacio disponible.

Cada división tiene:

- Eje.
- Ratio.
- Hijos.

### 9.2 Columns

Organiza ventanas en columnas ajustables.

### 9.3 Diálogos

Las ventanas transitorias y diálogos con restricciones de tamaño pueden flotar sobre el layout.

No todos los diálogos deben identificarse por título o tamaño: utilizar las relaciones de protocolo disponibles y reglas explícitas.

### 9.4 Gaps

Configurar:

- Gaps internos.
- Gaps externos.
- Tamaño mínimo.
- Ratio.
- Restricciones de ventana.

Evitar geometrías negativas o tamaños inferiores a los mínimos permitidos.

## 10. Modelo TileTree

Modelo conceptual:

```rust
enum TileTree {
    Empty,

    Leaf(WindowId),

    Split {
        axis: Axis,
        ratio: Ratio,
        first: Box<TileTree>,
        second: Box<TileTree>,
    },
}
```

La implementación debe definir operaciones para:

- Insertar.
- Eliminar.
- Intercambiar.
- Dividir.
- Cambiar ratio.
- Recorrer.
- Recalcular geometría.
- Validar invariantes.

No permitir nodos vacíos innecesarios ni referencias a ventanas inexistentes.

## 11. Comandos

Comandos iniciales:

```text
Focus(Direction | WindowId)
Move(Direction | WorkspaceId | OutputId)
Resize(Edge, Delta)
ToggleFloating
ToggleTilingWorkspace
Maximize
Minimize
Fullscreen
Close
SwapMaster
SetRatio
SnapLeft
SnapRight
SnapQuarter
CreateWorkspace
RemoveWorkspace
SwitchWorkspace
MoveWorkspaceToOutput
MoveWindowToWorkspace
MoveWindowToOutput
SetLayout
```

La API definitiva debe utilizar tipos fuertes y evitar parámetros ambiguos.

### 11.1 Validación

Cada comando debe comprobar:

- Existencia de IDs.
- Geometría.
- Output destino.
- Workspace destino.
- Estado de ventana.
- Compatibilidad de operación.
- Límites numéricos.

Los comandos inválidos devuelven un error tipado sin modificar parcialmente el estado.

## 12. Foco

Política predeterminada:

`Focus follows click`.

Opcionalmente:

`Focus follows mouse`.

Reglas:

- No robar foco arbitrariamente.
- No enfocar ventanas inexistentes.
- No enfocar ventanas minimizadas.
- Respetar activación solicitada por clientes según política del compositor.
- Mantener orden de foco.
- Recuperar foco al cerrar una ventana.

### 12.1 Focus stack

El `focus_stack` debe contener identificadores válidos.

La política de orden debe ser determinista.

Cuando una ventana se destruye, eliminar sus referencias.

Si la ventana activa desaparece, seleccionar una alternativa visible según el orden establecido.

## 13. Reglas de ventanas

Reglas basadas en:

- `app_id`.
- Identidad de aplicación.
- Título, cuando esté permitido.
- Output.
- Workspace.
- Modo.
- Geometría.

Acciones:

- Asignar workspace.
- Asignar output.
- Iniciar flotante.
- Iniciar tiled.
- Tamaño inicial.
- Posición inicial.

Las reglas deben tener prioridad explícita.

No permitir que una regla sobre un título genérico capture aplicaciones no relacionadas.

Las reglas deben validarse antes de activarse.

## 14. Minimización

La minimización es una política propia de PARALLAX.

Al minimizar:

1. Retirar la ventana del layout visible.
2. Actualizar foco.
3. Conservar estado lógico.
4. Informar al compositor.
5. Permitir restauración.

No destruir la ventana ni confundir minimización con `unmap`.

La representación en Dock pertenece a Shell.

## 15. Maximización y fullscreen

Maximizar:

- Utiliza el espacio disponible del output.
- Respeta la política de zonas exclusivas.
- Conserva información necesaria para restaurar geometría anterior.

Fullscreen:

- Utiliza el espacio de pantalla según la política del compositor.
- No conserva decoraciones ordinarias.
- No implica necesariamente direct scanout.
- Puede cambiar la visibilidad de Shell.

El compositor determina las condiciones de presentación y composición.

## 16. Animaciones

Animaciones de:

- Apertura.
- Cierre.
- Cambio de workspace.
- Minimización.
- Snap.
- Movimiento.
- Maximización.

`px-wm` no ejecuta animaciones.

Puede emitir efectos que incluyan:

- Tipo.
- Ventana.
- Geometría inicial.
- Geometría final.
- Identificador de transición.

El compositor utiliza el sistema de motion.

Todas las transiciones deben ser interrumpibles.

Con `reduced_motion`, la duración puede ser cero sin alterar el resultado lógico.

## 17. Persistencia

Persistir layout y preferencias mediante `settingsd` o el servicio de configuración establecido.

Ruta propuesta:

`$XDG_STATE_HOME/parallax/layout.toml`

Contenido:

- Workspaces.
- Modo.
- Ratios.
- Layout seleccionado.
- Reglas.
- Preferencias de salida.

No persistir identificadores de ventanas de una sesión anterior como si fueran válidos en una nueva sesión.

La restauración debe reconstruir el layout y aplicar reglas a ventanas nuevas.

## 18. API determinista

La interfaz recomendada:

```rust
fn apply(
    state: &State,
    command: Command,
) -> Result<Transition, WmError>;
```

Donde:

```rust
struct Transition {
    next_state: State,
    effects: Vec<Effect>,
}
```

Esta forma evita mutaciones parciales y facilita pruebas.

Si el rendimiento exige una API mutable interna, debe conservarse la semántica transaccional observable.

La lógica no puede depender de:

- Reloj global.
- Random global.
- I/O.
- Red.
- Wayland.
- GPU.
- Variables de entorno.
- Iteración no determinista.

Todo dato externo necesario debe recibirse explícitamente.

## 19. Invariantes

Invariantes obligatorios:

1. Toda ventana administrada tiene un único propietario lógico.
2. Una ventana no puede pertenecer simultáneamente a dos workspaces.
3. Una ventana tiled aparece exactamente una vez en el árbol de tiling.
4. Una ventana floating aparece exactamente una vez en la colección floating.
5. Las ventanas minimizadas no aparecen en el layout visible.
6. El foco apunta a una ventana válida y visible o es `None`.
7. Un output no puede tener un workspace activo inexistente.
8. El fullscreen apunta a una ventana válida.
9. No existen referencias a ventanas destruidas.
10. Los ratios están dentro de límites.
11. La geometría no contiene overflow.
12. Las transiciones inválidas no alteran el estado.

## 20. Pruebas

### Unitarias

- Comandos.
- Foco.
- Geometría.
- Workspace.
- Floating.
- Tiling.
- Maximización.
- Minimización.
- Fullscreen.
- Snap.
- Reglas.

### Property-based

- Invariantes después de cada comando.
- Secuencias aleatorias.
- Insertar y eliminar.
- Cambios de output.
- Cambios de workspace.
- Cierre de ventanas.
- Geometría extrema.

### Fuzzing

- Comandos aleatorios.
- IDs inválidos.
- Ratios extremos.
- Geometría extrema.
- Secuencias de transiciones.
- Reglas conflictivas.

### Determinismo

Misma entrada y mismo estado inicial deben producir el mismo estado final y los mismos efectos.

## 21. Criterios de aceptación

- [ ] API sin dependencia de Wayland.
- [ ] Integración con compositor.
- [ ] Floating.
- [ ] Tiling.
- [ ] Dwindle.
- [ ] Columns.
- [ ] Workspaces.
- [ ] Multi-output.
- [ ] Foco.
- [ ] Snap.
- [ ] Maximización.
- [ ] Fullscreen.
- [ ] Minimización.
- [ ] Reglas.
- [ ] Persistencia.
- [ ] Property-based testing.
- [ ] Fuzzing durante una hora sin pánicos ni invariantes rotas.
- [ ] Cobertura de líneas ≥90 % como objetivo de calidad, acompañada de cobertura de ramas y propiedades.

La cobertura porcentual no sustituye pruebas de invariantes.

## 22. Decisiones pendientes

1. Definir API pública.
2. Definir estructura definitiva de `TileTree`.
3. Definir algoritmos iniciales y su prioridad.
4. Definir reglas de ventanas.
5. Definir persistencia y migraciones.
6. Definir política de foco.
7. Definir modelo de geometría y escalas.
8. Definir relación con estados de ventana y compositor.

**Estado: Propuesto. Pendiente de aprobación.**