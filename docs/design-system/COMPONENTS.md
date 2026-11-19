# Sistema de componentes — PARALLAX

**Estado:** Propuesto · Pendiente de aprobación.  
**Biblioteca:** `px-ui`.  
**Ámbito:** Shell y aplicaciones nativas de PARALLAX.

## 1. Objetivo

Definir un sistema de componentes reutilizables, consistente, accesible y eficiente para todas las aplicaciones propias de PARALLAX.

`px-ui` constituye la implementación de referencia para:

- Controles interactivos.
- Componentes de navegación.
- Contenedores visuales.
- Indicadores de estado.
- Componentes de formularios.
- Patrones de interacción.
- Accesibilidad.
- Integración con el sistema de temas.

Ninguna aplicación propia debe implementar controles equivalentes de forma independiente cuando exista un componente oficial.

Las aplicaciones pueden crear composiciones y componentes especializados utilizando los componentes base de `px-ui`.

No deben duplicar su lógica de interacción, accesibilidad ni estados visuales.

### 1.1 Alcance

Aplicaciones propias:

- Shell.
- Settings.
- Files.
- Terminal, cuando utilice controles gráficos.
- Task Manager.
- Observatory.
- Aplicaciones futuras.

Aplicaciones de terceros:

- GTK.
- Qt.
- Electron.
- Aplicaciones Wayland nativas.

No están obligadas a utilizar `px-ui`. Su integración visual se gestiona mediante temas, portales y convenciones de escritorio.

## 2. Arquitectura

### 2.1 Responsabilidades de px-ui

`px-ui` proporciona:

- Implementación visual.
- Estados de interacción.
- Gestión de foco.
- Navegación por teclado.
- Semántica accesible.
- Integración con `px-theme`.
- Eventos de interacción.
- Comportamientos consistentes.
- Gestión de recursos gráficos.
- Contratos de componentes.

No debe contener lógica de negocio de las aplicaciones.

### 2.2 Dependencias

```text
             px-theme
                |
                v
              px-ui
                |
       +--------+--------+
       |        |        |
     Shell   Settings   Files
       |        |        |
       +--------+--------+
                |
         Observatory
```

Reglas:

- `px-ui` consume tokens de `px-theme`.
- No depende de aplicaciones concretas.
- No contiene lógica de sistema privilegiada.
- No accede directamente a servicios del sistema.
- No implementa políticas de negocio.
- No depende del compositor para renderizar controles normales.

La integración con el compositor se limita a las capacidades necesarias para superficies, ventanas y eventos de entrada.

### 2.3 Componentes base y compuestos

Organizar la biblioteca en tres niveles:

**Primitives**

- Button.
- IconButton.
- TextInput.
- Checkbox.
- Radio.
- Toggle.
- Slider.
- ProgressBar.
- Spinner.

**Components**

- Select.
- Menu.
- Dialog.
- Popover.
- Tooltip.
- Tabs.
- List.
- Table.
- Grid.
- Sidebar.
- SplitView.

**Patterns**

- EmptyState.
- LoadingState.
- ErrorState.
- PermissionState.
- Confirmation.
- SearchPanel.
- SettingsSection.
- DataView.

Los patrones utilizan componentes existentes y no implementan controles duplicados.

## 3. Estados universales

Todos los componentes deben declarar explícitamente los estados que les correspondan.

| Estado | Tratamiento |
|---|---|
| Reposo | Tokens base |
| Hover | `bg.hover`, transición `motion.instant` |
| Presionado | `bg.pressed`; escala 0,98 en botones cuando sea apropiado |
| Foco | Anillo de 2 px `focus.ring`, offset 2 px |
| Seleccionado | `bg.selected` + `fg.primary` |
| Deshabilitado | `fg.disabled`, sin interacción normal |
| Cargando | Spinner o skeleton sin alterar dimensiones |
| Error | Borde/texto `danger` y explicación |
| Activo | Estado funcional específico del componente |
| Arrastrando | Tratamiento visual específico cuando corresponda |

### 3.1 Foco

El foco debe ser visible durante la navegación por teclado.

Se utilizará una política equivalente a `:focus-visible`, pero implementada mediante el sistema de entrada de `px-ui`.

Reglas:

- El foco por teclado siempre debe ser perceptible.
- El foco por ratón puede evitar indicadores decorativos innecesarios.
- El foco programático debe gestionarse explícitamente.
- Los diálogos deben administrar el foco inicial y su restauración.
- Nunca eliminar el indicador de foco sin una alternativa equivalente.

El anillo debe respetar el radio del componente y no quedar recortado por contenedores.

### 3.2 Deshabilitado

Un control deshabilitado:

- No ejecuta acciones.
- No responde a hover o pressed.
- Comunica su estado a la tecnología de asistencia.
- Puede permanecer disponible para exploración accesible cuando necesite explicar por qué está deshabilitado.

Cuando exista una condición que impida una acción, debe exponerse el motivo mediante texto accesible o una explicación asociada.

No utilizar el tooltip como único mecanismo para explicar restricciones importantes.

### 3.3 Cargando

- Mantener dimensiones.
- Evitar saltos de layout.
- Exponer estado de progreso cuando corresponda.
- Impedir operaciones duplicadas cuando la acción no admita concurrencia.
- Permitir cancelación si la operación lo soporta.

No bloquear indiscriminadamente toda una vista por la carga de un elemento independiente.

### 3.4 Error

Los errores deben comunicar:

1. Qué ocurrió.
2. Qué elemento está afectado.
3. Qué puede hacer el usuario.

Distinguir:

- Error de validación.
- Error de operación.
- Error de conectividad.
- Error de permisos.
- Error irrecuperable.

Los errores importantes deben exponerse mediante los mecanismos de anuncios accesibles correspondientes.

## 4. Catálogo de componentes

### 4.1 Button

Variantes:

- Primary.
- Secondary.
- Ghost.
- Danger.

Tamaños:

- Small: 24 px de altura visual.
- Medium: 32 px.
- Large: 40 px.

Padding horizontal:

- Small: `space.3`.
- Medium: `space.4`.
- Large: `space.5`.

Radio: `radius.sm`.

Comportamiento:

- Enter y Espacio ejecutan la acción.
- Soporta icono y texto.
- Puede presentar estado de carga.
- Puede ocupar el ancho disponible.
- Tiene estados hover, pressed, focus, disabled y loading.

El objetivo visual de 24 px no implica un área interactiva de 24 px.

El hit target debe ampliarse cuando sea necesario para cumplir los requisitos de accesibilidad.

### 4.2 IconButton

Variantes:

- Ghost.
- Tonal.

Características:

- Área visual cuadrada.
- Iconografía del sistema.
- Nombre accesible obligatorio.
- Tooltip recomendado para acciones cuyo significado no sea evidente.
- Estados universales.

No se acepta un IconButton sin nombre accesible.

Enter y Espacio ejecutan la acción.

### 4.3 Toggle

Switch de configuración.

Dimensiones visuales:

- 36 × 20 px.
- Pulgar de 16 px.

Características:

- Color `accent` cuando está activo.
- Cambio inmediato.
- Estado persistente gestionado por la aplicación.
- Nombre accesible.
- Estado activado/desactivado comunicado semánticamente.

Teclado:

- Espacio: alternar.

El área interactiva debe ampliarse sin alterar necesariamente el tamaño visual.

### 4.4 Checkbox y Radio

Tamaño visual: 16 px.

Área interactiva objetivo: mínimo 32 × 32 px.

Checkbox:

- Selección independiente.
- Estados checked, unchecked e indeterminate cuando corresponda.
- Espacio para alternar.

Radio:

- Selección exclusiva dentro de un grupo.
- Flechas para cambiar selección.
- Espacio para confirmar cuando corresponda.

Los grupos deben tener una etiqueta y semántica accesible.

### 4.5 Slider

Variantes:

- Continuo.
- Con pasos.

Características:

- Pista de 4 px.
- Pulgar de 16 px.
- Valor accesible.
- Indicador del valor durante el arrastre.
- Límites mínimo y máximo.
- Unidad opcional.

Teclado:

- Flechas: paso configurado.
- Page Up / Page Down: incremento mayor.
- Home: mínimo.
- End: máximo.

El tamaño del incremento mayor debe definirse por componente, no asumirse siempre como un 10 %.

### 4.6 TextInput

Variantes:

- Default.
- Search.
- Password.

Características:

- Altura visual de 32 px.
- Fondo `bg.sunken`.
- Estados normal, focus, disabled y error.
- Mensaje de validación debajo.
- Soporte IME.
- Selección, cursor y edición estándar.
- Pegado y composición de texto.

La implementación debe soportar correctamente:

- Unicode.
- CJK.
- Árabe y RTL.
- Emoji.
- Fuentes alternativas.
- Escalado de texto.

La validación de formularios no debe interrumpir la composición de texto IME.

### 4.7 SearchField

Características:

- Icono de búsqueda.
- Campo de texto.
- Acción para limpiar.
- Placeholder descriptivo.

Teclado:

- Escape: limpiar contenido.
- Segundo Escape: salir o cerrar el contexto de búsqueda.
- Ctrl+F: enfocar cuando el contexto de la aplicación lo permita.

No registrar Ctrl+F como atajo global obligatorio del escritorio.

### 4.8 Select / Dropdown

Características:

- Menú anclado.
- Selección simple o múltiple según variante.
- Indicador de selección.
- Búsqueda opcional.
- Estados de carga y vacío.

Teclado:

- Flechas: navegar.
- Enter: seleccionar.
- Escape: cerrar.
- Escritura: búsqueda por caracteres.

Debe gestionar correctamente:

- Posición del menú.
- Límites de pantalla.
- Scroll.
- Foco.
- Selección actual.

### 4.9 Menu / ContextMenu

Características:

- Iconos.
- Atajos.
- Submenús.
- Separadores.
- Estados deshabilitados.
- Elementos seleccionables cuando corresponda.

Estilo:

- `bg.elevated`.
- `radius.md`.
- `elevation.3`.
- Altura visual inicial de 28 px por elemento.

Teclado:

- Flechas: navegación.
- Enter: ejecutar.
- Escape: cerrar.
- Escritura: búsqueda por letra.
- Flecha lateral: abrir o cerrar submenú según orientación.

Los elementos destructivos deben estar visualmente diferenciados.

### 4.10 Popover

Variantes:

- Con flecha.
- Sin flecha.

Características:

- `elevation.3`.
- Posicionamiento relativo al elemento de origen.
- Cierre mediante clic externo cuando corresponda.
- Cierre con Escape.
- Gestión de foco según contenido.

No todos los popovers deben atrapar el foco.

La trampa de foco solo se utiliza cuando el contenido y su semántica lo requieren.

### 4.11 Dialog

Variantes:

- Information.
- Confirmation.
- Destructive.

Ancho máximo inicial: 480 px.

Características:

- Título.
- Descripción.
- Contenido.
- Acciones.
- Foco inicial definido.
- Restauración de foco al cerrar.
- Gestión de interacción modal.

Teclado:

- Escape: cancelar cuando esté permitido.
- Enter: ejecutar la acción predeterminada cuando sea segura.
- Tab / Shift+Tab: recorrer controles dentro del diálogo.

Reglas:

- La acción destructiva nunca es predeterminada.
- La acción principal se presenta a la derecha.
- Las operaciones irreversibles deben describir claramente sus consecuencias.
- No cerrar un diálogo mientras exista una operación crítica sin resolver, salvo que se permita cancelarla.

### 4.12 Toast

Variantes:

- Info.
- Success.
- Error.
- Con acción.

Posición:

- Parte inferior central dentro de la ventana correspondiente.

Duración inicial:

- 4 segundos.

Características:

- No bloqueante.
- Acción opcional.
- Lectura accesible.
- Gestión de varias notificaciones.
- Pausa del temporizador durante interacción o hover cuando corresponda.

Teclado:

- F6: acceder a la región de notificaciones cuando esté disponible.

Los errores críticos no deben desaparecer antes de que el usuario pueda comprenderlos.

Las operaciones importantes deben conservar acceso a su resultado mediante una vista persistente o historial.

### 4.13 Tooltip

Características:

- Retardo inicial: 500 ms.
- Tipografía `text.footnote`.
- Fondo `bg.elevated`.
- Aparición al enfocar mediante teclado.
- Posicionamiento adaptativo.

No debe ser el único medio para comunicar información crítica.

No debe interceptar el foco ni impedir la interacción con el elemento de origen.

### 4.14 List

Variantes:

- Simple.
- Con icono.
- Dos líneas.

Alturas:

- 28 px.
- 36 px.
- 48 px.

Características:

- Virtualización.
- Selección simple y múltiple.
- Filas identificadas mediante IDs estables.
- Soporte para elementos dinámicos.
- Actualización incremental.

Teclado:

- Flechas: navegación.
- Shift: selección por rango.
- Ctrl: selección múltiple cuando corresponda.
- Escritura: búsqueda incremental.

La virtualización no debe eliminar la posición lógica del elemento enfocado ni provocar pérdida de selección.

### 4.15 Table

Características:

- Ordenación.
- Columnas redimensionables.
- Cabecera fija.
- Alineación numérica a la derecha.
- Filas virtualizadas cuando sea necesario.
- Selección de filas y celdas según variante.

Teclado:

- Flechas: navegación.
- Enter: abrir o activar elemento.
- Tab: recorrer controles interactivos.

La tabla debe comunicar semánticamente:

- Encabezados.
- Columnas.
- Filas.
- Orden actual.
- Selección.
- Estado de carga.
- Estado vacío.

### 4.16 Grid

Características:

- Iconos.
- Celdas de 96 o 128 px.
- Selección.
- Navegación bidimensional.
- Distribución adaptable.

Teclado:

- Flechas: desplazamiento en dos dimensiones.
- Enter: activar.
- Ctrl+A: seleccionar todo cuando corresponda.

### 4.17 Tabs

Variantes:

- Barra.
- Segmentado.

Características:

- Pestaña activa con `bg.surface`.
- Pestañas cerrables cuando se permita.
- Reordenamiento cuando corresponda.
- Indicador de selección.
- Soporte para contenido asociado.

Teclado:

- Ctrl+Tab: siguiente pestaña cuando el contexto lo permita.
- Ctrl+Shift+Tab: anterior.
- Ctrl+1..9: acceso directo opcional.

La navegación por flechas entre pestañas debe estar soportada.

No imponer atajos que entren en conflicto con los atajos de la aplicación.

### 4.18 Sidebar

Ancho inicial: 220 px.

Rango configurable: 180–320 px.

Características:

- Secciones colapsables.
- Elementos de navegación.
- Indicadores de selección.
- Iconos.
- Scroll.

Teclado:

- F6: cambiar región de navegación.
- Flechas: navegar por los elementos cuando corresponda.

### 4.19 SplitView

Variantes:

- Horizontal.
- Vertical.

Características:

- Divisor visual de 1 px.
- Área de arrastre de 8 px.
- Límites mínimo y máximo por panel.
- Redimensionamiento accesible.
- Persistencia opcional de dimensiones.

Teclado:

- Flechas: redimensionar.
- Home / End: límites.
- Atajo de restablecimiento opcional.

### 4.20 Breadcrumb

Características:

- Segmentos navegables.
- Representación jerárquica.
- Elemento actual identificado.
- Posibilidad de edición mediante interacción explícita.

Ctrl+L puede enfocar el modo de edición cuando la aplicación lo implemente y el contexto lo permita.

No registrar Ctrl+L globalmente en el sistema.

### 4.21 ProgressBar / Spinner

Variantes:

- Determinado.
- Indeterminado.

ProgressBar:

- Altura visual de 4 px.
- Valor mínimo y máximo.
- Estado accesible.

Spinner:

- Tamaños de 16 y 24 px.
- Animación compatible con movimiento reducido.
- Sin animación infinita innecesaria cuando la vista no está visible.

### 4.22 Card

Características:

- `bg.surface`.
- `radius.md`.
- `elevation.1`.
- Padding `space.4`.

La tarjeta no es interactiva por defecto.

Cuando actúe como superficie interactiva debe exponer semántica, foco y comportamiento de teclado.

### 4.23 Badge / Chip

Variantes:

- Informativo.
- Estado.
- Seleccionable.
- Removible.

Características:

- `radius.xs` o `radius.full`.
- Contraste adecuado.
- Nombre accesible.
- Estado semántico cuando corresponda.

No utilizar badges como único medio para comunicar estados críticos.

### 4.24 EmptyState

Características:

- Icono de 48 px.
- Título.
- Descripción.
- Acción primaria opcional.

Debe explicar por qué no hay contenido y qué puede hacer el usuario.

### 4.25 Tile

Uso: Control Center.

Variantes:

- Toggle.
- Slider.
- Detalle.

Dimensiones:

- 2 × 1 celdas.
- 2 × 2 celdas.
- Celda de referencia: 72 px.

Teclado:

- Flechas: navegación.
- Espacio: activar cuando corresponda.
- Enter: abrir detalle cuando corresponda.

Las Tiles deben presentar claramente su estado actual y su acción.

## 5. Patrones obligatorios

Toda vista que dependa de datos debe contemplar los siguientes estados:

### 5.1 Vacío

- Mensaje explicativo.
- Contexto.
- Acción disponible cuando exista.

### 5.2 Cargando

- Indicador apropiado.
- Evitar bloquear contenido que ya está disponible.
- Mantener estabilidad del layout.

### 5.3 Error

- Descripción del problema.
- Acción de recuperación cuando sea posible.
- Información técnica adicional disponible en un nivel secundario.

### 5.4 Sin permiso

- Explicación de la restricción.
- Identificación de la acción que requiere permiso.
- Mecanismo de autorización cuando corresponda.

No mostrar acciones de elevación de privilegios cuando no sean necesarias.

### 5.5 Sin conexión

Cuando aplique:

- Estado de conectividad.
- Última información disponible.
- Posibilidad de reintentar.
- Indicación de operaciones pendientes.

## 6. Formularios

Reglas:

- Validación al salir del campo o al enviar.
- No mostrar errores prematuramente al escribir el primer carácter.
- No interrumpir composición IME.
- Mantener el contenido cuando falle una operación.
- Asociar errores con sus campos.
- Comunicar errores mediante accesibilidad.
- Permitir corrección sin perder el resto del formulario.

Las validaciones que dependan del servidor deben diferenciarse de las validaciones locales.

## 7. Confirmaciones y acciones

### 7.1 Acciones destructivas

Solicitar confirmación cuando una acción sea irreversible o tenga consecuencias importantes difíciles de revertir.

La confirmación debe explicar qué se perderá o modificará.

### 7.2 Acciones reversibles

Preferir ejecución inmediata con posibilidad de Deshacer mediante Toast o mecanismo equivalente.

No llenar la interfaz de diálogos de confirmación innecesarios.

### 7.3 Ajustes

Aplicación inmediata por defecto.

Excepción: cambios de pantalla o configuración de salida gráfica que puedan dejar al usuario sin imagen.

Estos cambios requieren:

1. Aplicación temporal.
2. Diálogo de confirmación.
3. Temporizador de 15 segundos.
4. Confirmación explícita.
5. Reversión automática si no se confirma.

La reversión debe ser responsabilidad de la operación de configuración correspondiente, no una función visual aislada del Dialog.

## 8. Accesibilidad

### 8.1 Objetivo

WCAG 2.2 AA como referencia para el diseño y comportamiento.

Integración con:

- AccessKit.
- AT-SPI.
- Orca.
- Navegación por teclado.
- Escalado de texto.
- Alto contraste.
- Movimiento reducido.

### 8.2 Semántica

Cada componente debe definir:

- Rol.
- Nombre accesible.
- Descripción cuando corresponda.
- Estado.
- Valor.
- Relaciones con otros controles.

No basta con representar visualmente un control.

### 8.3 Áreas interactivas

Las áreas de interacción deben respetar los objetivos de accesibilidad definidos para PARALLAX.

Los controles compactos pueden conservar dimensiones visuales pequeñas siempre que su área interactiva real sea suficiente y no genere conflictos con elementos vecinos.

### 8.4 Foco

- Orden lógico.
- Foco visible.
- Sin trampas accidentales.
- Restauración del foco tras cerrar superficies temporales.
- Gestión de regiones independientes.

### 8.5 Tecnologías de asistencia

Los componentes deben probarse con AccessKit y AT-SPI en la integración real.

No considerar suficiente que el árbol accesible exista: también debe comunicar correctamente acciones, estados y cambios.

## 9. Rendimiento

`px-ui` debe cumplir la metodología de `docs/benchmarks/metodologia.md` y la especificación de rendimiento de efectos.

Reglas:

- Virtualizar listas grandes.
- Evitar reconstrucciones globales.
- Evitar cálculos costosos en el hilo de UI.
- No renderizar componentes invisibles innecesariamente.
- Compartir recursos compatibles.
- Evitar animaciones constantes.
- Aplicar damage tracking.
- Utilizar caches con invalidación explícita.

Cada componente complejo debe tener benchmarks propios.

Especial atención:

- List.
- Table.
- Grid.
- Menu.
- Popover.
- Dialog.
- Sidebar.
- SplitView.

## 10. Contrato de implementación

Cada componente debe proporcionar:

1. API tipada.
2. Propiedades y eventos.
3. Estados soportados.
4. Comportamiento de teclado.
5. Semántica accesible.
6. Tokens utilizados.
7. Variantes.
8. Casos límite.
9. Pruebas funcionales.
10. Pruebas visuales.
11. Benchmark cuando sea relevante.

No permitir componentes con comportamiento implícito no documentado.

Las propiedades visuales deben utilizar tokens y no colores hardcodeados.

## 11. Pruebas obligatorias

### Funcionales

- Estados universales.
- Interacciones.
- Navegación.
- Eventos.
- Validaciones.
- Cambios de tema.
- Cambios de escala.

### Teclado

- Navegación completa.
- Orden de foco.
- Escape.
- Enter.
- Espacio.
- Atajos contextuales.
- Navegación por regiones.

### Accesibilidad

- AccessKit.
- AT-SPI.
- Orca.
- Nombre y descripción accesible.
- Roles y estados.
- Alto contraste.
- Movimiento reducido.
- Escalado al 200 %.

### Idiomas

- Español.
- Inglés.
- Árabe RTL.
- CJK.
- Emoji.
- Textos largos.
- Fuentes alternativas.

### Rendimiento

- Listas de 10.000 elementos.
- Tablas grandes.
- Grid con muchos elementos.
- Apertura y cierre repetido de diálogos.
- Menús anidados.
- Cambios de tema.
- Escalado de texto.

## 12. Organización del código

Estructura conceptual:

```text
crates/
└── px-ui/
    └── src/
        ├── lib.rs
        ├── theme/
        ├── accessibility/
        ├── interaction/
        ├── primitives/
        │   ├── button/
        │   ├── icon_button/
        │   ├── toggle/
        │   ├── checkbox/
        │   ├── radio/
        │   ├── slider/
        │   └── text_input/
        ├── components/
        │   ├── select/
        │   ├── menu/
        │   ├── dialog/
        │   ├── popover/
        │   ├── tooltip/
        │   ├── list/
        │   ├── table/
        │   ├── grid/
        │   └── tabs/
        ├── patterns/
        ├── layout/
        └── testing/
```

La organización definitiva dependerá del toolkit seleccionado en ADR-0005.

No crear abstracciones que dupliquen las capacidades del toolkit sin una necesidad demostrada.

## 13. Criterios de aceptación

El sistema se considera implementado cuando:

- Existe un catálogo oficial.
- Las aplicaciones propias consumen `px-ui`.
- Los controles tienen APIs tipadas.
- Los estados están definidos.
- La navegación por teclado funciona.
- La semántica accesible está integrada.
- Los componentes utilizan tokens.
- Los cambios de tema se aplican correctamente.
- Las vistas contemplan estados vacíos, carga, error y permisos.
- Las listas y tablas grandes están virtualizadas.
- Las pruebas funcionales y de accesibilidad están automatizadas.
- Los benchmarks no muestran regresiones relevantes.
- Los componentes no dependen de lógica de negocio.

**Estado: Propuesto. Pendiente de aprobación arquitectónica y validación con el toolkit seleccionado.**