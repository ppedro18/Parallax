# Especificación: Framework gráfico `px-ui`

**Componente:** `px-ui`  
**Estado:** Propuesto · Pendiente de ADR-0005  
**Capa:** Shared UI Infrastructure  
**Consumidores:** `parallax-shell`, PARALLAX Files, Settings, Terminal, Observatory y aplicaciones nativas.

## 1. Objetivo

`px-ui` es la capa común de interfaz gráfica para las aplicaciones propias de PARALLAX.

Proporciona componentes, layout, renderizado, texto, accesibilidad, input, animaciones y tematización.

Su objetivo no es crear un toolkit nuevo desde cero, sino encapsular una infraestructura gráfica existente detrás de un contrato estable y coherente con el sistema de diseño PARALLAX.

## 2. Principios

1. Las aplicaciones PARALLAX no dependen directamente del toolkit seleccionado.
2. El toolkit subyacente no se expone en las APIs públicas de las aplicaciones.
3. Los componentes siguen el sistema de diseño.
4. La accesibilidad es obligatoria.
5. Las operaciones gráficas no bloquean el hilo UI.
6. Los cambios de tema se aplican en caliente.
7. Las animaciones son interrumpibles.
8. Las listas grandes deben virtualizarse.
9. El framework no debe convertirse en una capa de abstracción innecesariamente compleja.
10. La elección final depende de evidencia técnica y del ADR-0005.

## 3. Alcance funcional

Requisitos obligatorios de v1:

| Capacidad | Requisito |
|---|---|
| Wayland nativo | Obligatorio |
| Layer-shell | Obligatorio para superficies del escritorio |
| Fractional scale | Obligatorio |
| xdg-activation | Obligatorio |
| Renderizado acelerado | Obligatorio |
| Fallback software | Obligatorio para entornos compatibles |
| Texto shaping | Obligatorio |
| Bidi | Obligatorio |
| Font fallback | Obligatorio |
| Emoji | Obligatorio |
| AccessKit | Obligatorio |
| AT-SPI | Obligatorio |
| IME | Obligatorio |
| Animaciones interrumpibles | Obligatorio |
| Layout flexible | Obligatorio |
| Listas virtualizadas | Obligatorio |
| Tokens en runtime | Obligatorio |

## 4. Evaluación del toolkit

### 4.1 Alternativas

Evaluar:

- Iced.
- libcosmic.
- Slint.
- Toolkit propio.

| Criterio | Iced | libcosmic | Slint | Propio |
|---|---|---|---|---|
| Integración Wayland | Evaluar backend y versiones | Base de escritorio real | Evaluar | Por construir |
| Layer-shell | Integración adicional | Evaluar capacidades concretas | Evaluar | Por construir |
| Accesibilidad | Verificar AccessKit y cobertura | AccessKit integrado en su ecosistema | Verificar | Por construir |
| Licencia | MIT | MPL-2.0 | Revisar modalidad y distribución | Por definir |
| Mantenimiento | Medio | Dependencia del ecosistema COSMIC | Medio | Muy alto |
| Personalización | Alta | Adaptación sobre su arquitectura | Alta | Total |

No asumir que una integración de terceros garantiza compatibilidad de producción.

### 4.2 Recomendación provisional

Iced encapsulado mediante `px-ui` es la hipótesis inicial.

Investigar:

- `iced_layershell`.
- Integración SCTK.
- Input.
- IME.
- AccessKit.
- Fractional scale.
- Text shaping.
- Virtualización.
- Renderizado.
- Consumo de memoria.

La extracción selectiva de código de libcosmic requiere revisar licencia, dependencias, API y mantenimiento. No copiar componentes sin un análisis de compatibilidad.

Slint permanece como alternativa si Iced no cumple requisitos críticos.

No construir un toolkit propio en v1.

### 4.3 Gate de decisión

ADR-0005 no se aprueba hasta completar:

1. Spike funcional Wayland.
2. Spike layer-shell.
3. Spike AccessKit y AT-SPI.
4. Spike IME.
5. Galería de componentes.
6. Benchmarks.
7. Revisión de licencias.
8. Pruebas con aplicaciones reales.
9. Evaluación de mantenimiento.

## 5. Arquitectura interna

```text
Applications
     |
     v
   px-ui API
     |
     +---- Components
     +---- Layout
     +---- Typography
     +---- Tokens
     +---- Accessibility
     +---- Animation
     +---- Input
     |
     v
Toolkit Adapter
     |
     v
Selected Toolkit
     |
     v
Wayland / Renderer
```

### 5.1 API pública

Las aplicaciones importan únicamente `px-ui`.

No deben importar tipos del toolkit directamente.

El adaptador interno traduce:

- Eventos.
- Layout.
- Widgets.
- Estilos.
- Texto.
- Accesibilidad.
- Renderizado.

No duplicar innecesariamente las capacidades que el toolkit ya ofrece.

### 5.2 Componentes

Componentes de v1:

- Button.
- IconButton.
- Toggle.
- Checkbox.
- Radio.
- Slider.
- TextInput.
- SearchField.
- Select.
- List.
- Grid.
- Tabs.
- SplitView.
- Menu.
- ContextMenu.
- Popover.
- Dialog.
- Toast.
- Tooltip.
- ProgressBar.
- Spinner.
- Card.
- Sidebar.
- Breadcrumb.
- Table.
- EmptyState.
- Badge.
- Tile.

Cada componente define:

- API.
- Estados.
- Variantes.
- Tamaños.
- Eventos.
- Teclado.
- Accesibilidad.
- Tokens.
- Comportamiento responsive.

La especificación completa de componentes es `design-system/components.md`.

## 6. Estados universales

Estados:

- Idle.
- Hover.
- Pressed.
- Focus-visible.
- Selected.
- Disabled.
- Loading.
- Error.
- Active.
- Dragging.

Reglas:

- Focus visible con indicador de contraste suficiente.
- Disabled no responde a interacción ordinaria.
- Loading conserva geometría para evitar saltos.
- Error incluye mensaje comprensible y acción cuando sea posible.
- Reduced motion respeta las preferencias.
- No depender exclusivamente del color para comunicar estados.

## 7. Tokens

Los tokens proceden de `settingsd` y del sistema de diseño.

Ejemplos:

- Color.
- Typography.
- Spacing.
- Radius.
- Elevation.
- Motion.
- Density.
- Focus.
- Opacity.

El framework consume tokens resueltos.

No escribe directamente la configuración del usuario.

El cambio de tema debe actualizar los componentes activos sin reiniciar las aplicaciones cuando el toolkit lo permita.

Si una actualización no puede aplicarse inmediatamente, debe documentarse el fallback.

## 8. Texto

El subsistema debe soportar:

- Shaping.
- Bidi.
- Fallback de fuentes.
- Emoji.
- CJK.
- IME.
- Ligaduras cuando proceda.
- Escalado de texto.
- Selección.
- Clipboard.
- Accesibilidad.

Evaluar `cosmic-text` y `parley` mediante pruebas reales.

No asumir que renderizar glifos equivale a implementar un sistema completo de edición de texto.

## 9. Accesibilidad

Integración con AccessKit.

Verificar:

- Roles.
- Nombres.
- Estados.
- Relaciones.
- Navegación.
- Foco.
- Lectores de pantalla.
- AT-SPI.
- Orca.
- Teclado.
- Escalado de texto.
- Alto contraste.
- Reduced motion.

Cada componente debe tener una representación accesible.

Los componentes personalizados no pueden depender de que su apariencia visual sea suficiente para comprenderlos.

## 10. IME y teclado

Soportar el protocolo de input method correspondiente, incluido `text-input-v3` para el lado de la aplicación cuando proceda.

Validar:

- Composición.
- Confirmación.
- Cancelación.
- Cursor.
- Selección.
- Surrounding text.
- CJK.
- Teclados alternativos.

No considerar IME resuelto únicamente porque se reciben eventos de teclado.

## 11. Renderizado

### 11.1 GPU

El backend acelerado se selecciona después del benchmark.

Evaluar GLES/wgpu según la compatibilidad real con el toolkit.

### 11.2 Fallback software

`tiny-skia` puede servir para escenarios compatibles, pero no debe asumirse que sustituye todas las capacidades del backend GPU.

El fallback debe indicar:

- Capacidades disponibles.
- Efectos desactivados.
- Límites.
- Motivo de activación.

### 11.3 Rendimiento

- No bloquear UI con E/S.
- Virtualizar listas grandes.
- Cachear recursos.
- Evitar rerasterización innecesaria.
- No generar frames sin cambios.
- Medir PSS y tiempo de frame.
- Cumplir la metodología de rendimiento.

## 12. Animaciones

- Interrumpibles.
- Reversibles cuando corresponda.
- Basadas en tokens.
- Sin bloquear input.
- Compatibles con reduced motion.
- Limitadas al presupuesto de renderizado.

La animación es una presentación visual de una transición de estado, no la fuente de verdad del estado.

## 13. Licencias

La licencia de `px-ui` debe decidirse en ADR-0008 y ser compatible con:

- Toolkit.
- Dependencias.
- Componentes reutilizados.
- Recursos gráficos.
- Distribución de PARALLAX.

No copiar código de terceros sin registrar procedencia y licencia.

## 14. Pruebas

- Componentes y estados.
- Teclado.
- Accesibilidad.
- IME.
- RTL.
- Fractional scale.
- Text scaling 100–200 %.
- Light/dark/high contrast.
- Reduced motion.
- Reduced transparency.
- Virtualización.
- Renderizado GPU.
- Fallback software.
- PSS.
- Latencia.
- Pruebas visuales deterministas.

## 15. Criterios de aceptación

- [ ] Toolkit seleccionado mediante ADR-0005.
- [ ] Wayland nativo funcional.
- [ ] Layer-shell funcional.
- [ ] Fractional scale.
- [ ] xdg-activation.
- [ ] AccessKit y AT-SPI.
- [ ] Orca.
- [ ] IME probado.
- [ ] Shaping, bidi, CJK y emoji.
- [ ] Galería de componentes.
- [ ] Temas dinámicos.
- [ ] Listas virtualizadas.
- [ ] Fallback software.
- [ ] Benchmarks en HW-A.
- [ ] Revisión de licencias.
- [ ] Sin dependencias directas del toolkit en aplicaciones.

## 16. Decisiones pendientes

1. Resolver ADR-0005.
2. Seleccionar backend gráfico.
3. Definir integración exacta de layer-shell.
4. Definir estrategia de IME.
5. Definir política de fuentes y fallback.
6. Definir API pública estable.
7. Completar benchmark de UI.

**Estado: Propuesto. No aprobado.**