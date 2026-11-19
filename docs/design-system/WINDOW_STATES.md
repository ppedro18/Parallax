# Especificación de estados de ventana — PARALLAX

**Estado:** Propuesto · Pendiente de validación con ADR-0003 y ADR-0004.

## 1. Objetivo

Definir el comportamiento visual, geométrico e interactivo de las ventanas administradas por `parallax-compositor`.

La especificación cubre:

- Decoraciones de ventanas.
- Estados de foco.
- Maximización y fullscreen.
- Tiling y snap.
- Minimización.
- Movimiento y redimensionamiento.
- Ventanas modales.
- Aplicaciones que no responden.
- Solicitudes de activación.
- Overview.
- Sombras, recorte y zonas de interacción.

El comportamiento del Window Manager debe permanecer dentro de `px-wm`, conforme a ADR-0004.

El compositor interpreta sus operaciones y las aplica mediante Smithay y los protocolos Wayland correspondientes.

## 2. Modelo de estados

Los estados de una ventana no deben representarse como un único enum que obligue a elegir entre estados incompatibles.

Una ventana puede estar activa, maximizada, en movimiento y tener una solicitud de activación pendiente en diferentes dimensiones de su ciclo de vida.

Se definen las siguientes dimensiones:

### 2.1 Estado de presentación

- Normal.
- Maximizada.
- Fullscreen.
- Tiled.
- Minimized.
- Oculta en Overview.

### 2.2 Estado de interacción

- Idle.
- Hover.
- Focused.
- Moving.
- Resizing.
- Modal.
- Disabled por política de modalidad.

### 2.3 Estado de actividad

- Active.
- Inactive.
- Urgent.
- Unresponsive.
- Closing.

### 2.4 Estado de geometría

- Floating.
- Tiled.
- Snapped.
- Maximized.
- Fullscreen.

La geometría debe conservar suficiente información para restaurar el tamaño y posición anteriores al cambiar entre modos.

No sobrescribir la geometría flotante al maximizar, entrar en fullscreen o aplicar snap.

## 3. Decoraciones de ventanas

### 3.1 Aplicaciones PARALLAX

Las aplicaciones propias utilizan decoraciones del cliente mediante `px-ui`.

Características:

- Barra de título integrada en la cabecera de la aplicación.
- Integración visual con el contenido.
- Controles de ventana consistentes con el sistema.
- Estados de foco y accesibilidad compartidos con el design system.

El compositor no debe dibujar una segunda decoración sobre una aplicación que gestione correctamente su propia decoración.

### 3.2 Aplicaciones de terceros

Utilizar `xdg-decoration` para negociar decoraciones de servidor cuando el cliente soporte el protocolo.

Preferencia:

1. Negociar Server-Side Decorations (SSD).
2. Utilizar Client-Side Decorations (CSD) cuando el cliente las solicite o no soporte el protocolo.
3. Evitar decoraciones duplicadas.
4. Mantener compatibilidad con aplicaciones que no implementen `xdg-decoration`.

No asumir que todas las aplicaciones de terceros aceptarán decoraciones del servidor.

El compositor debe mantener una estrategia de fallback para clientes CSD.

### 3.3 Decoración SSD

Altura: 36 px.

| Propiedad | Valor |
|---|---|
| Fondo | `bg.surface` |
| Título | `text.body.strong` |
| Título activo | `fg.primary` |
| Título inactivo | `fg.tertiary` |
| Alineación | Centrada |
| Controles | Configurables a izquierda o derecha |
| Radio de controles | 12 px visuales |
| Área interactiva mínima | 24 × 24 px |
| Hover | Glifo visible |
| Focus | Indicador visible y accesible |

Controles:

- Cerrar.
- Minimizar.
- Maximizar/restaurar.

Los colores de los controles utilizarán los tokens `danger`, `warning` y `success` en versiones atenuadas.

No copiar literalmente la apariencia de macOS.

Los controles deben admitir interacción por teclado y accesibilidad. El glifo no puede depender exclusivamente del hover para resultar comprensible.

La barra de título debe admitir:

- Arrastrar para mover.
- Doble clic para maximizar/restaurar, configurable.
- Menú contextual de ventana.
- Accesibilidad por teclado.

## 4. Estados visuales

### 4.1 Ventana activa

- Radio: `radius.window`.
- Sombra: `elev.3`.
- Borde: `stroke.default`.
- Título: `fg.primary`.
- Controles: colores de estado.
- Contenido sin modificaciones de luminosidad.

### 4.2 Ventana inactiva

- Radio: `radius.window`.
- Sombra: `elev.2`.
- Borde: `stroke.subtle`.
- Título: `fg.tertiary`.
- Controles: grises.
- Contenido sin oscurecer ni desaturar.

El contenido de la aplicación debe conservar su apariencia original.

### 4.3 Hover sobre ventana inactiva

- No cambia el foco automáticamente.
- No modifica la geometría.
- Los controles muestran su estado hover cuando corresponda.
- La activación requiere una acción explícita del usuario o una solicitud autorizada.

### 4.4 Maximizada

- Radio: 0.
- Sombra: ninguna.
- Borde: ninguno.
- Decoración: visible.
- Utiliza el área de trabajo disponible del monitor.
- Respeta las zonas exclusivas configuradas por la Shell.

La geometría maximizada debe recalcularse cuando cambien:

- Resolución.
- Escala.
- Orientación.
- Zonas exclusivas.
- Distribución de monitores.

### 4.5 Fullscreen

- Radio: 0.
- Sombra: ninguna.
- Borde: ninguno.
- Decoración: oculta.
- La ventana ocupa el área fullscreen del output.
- La política de visibilidad de topbar y dock se gestiona desde la integración Shell/compositor.

Fullscreen no implica automáticamente que todas las superficies de la Shell deban desaparecer: su visibilidad depende de la política de sesión, capas y excepciones configuradas.

#### Direct scanout

El direct scanout es una optimización opcional.

Solo se utilizará cuando se cumplan las condiciones técnicas del backend y del hardware:

- Formato de buffer compatible.
- Compatibilidad de modificadores.
- Configuración de planes DRM compatible.
- Ausencia de composición adicional incompatible.
- Sincronización y presentación correctas.
- Compatibilidad con las superficies visibles y overlays.

Si hay redondeo, sombras, overlays u otros efectos que requieran composición, el compositor debe utilizar su ruta de renderizado normal.

No presentar direct scanout como una garantía del estado fullscreen.

### 4.6 Tiled

- Radio: `radius.window`, configurable a 0.
- Sombra: `elev.1`.
- Borde: `stroke.subtle`.
- Ventana activa: anillo `accent` de 2 px.
- Decoración: visible.

Separación:

- Entre ventanas: `space.2`.
- Entre ventanas y borde del área de trabajo: `space.3`.
- Configuración de separación: 0–24 px.

El cálculo de geometría debe descontar los espacios antes de distribuir las ventanas.

Los huecos visuales no deben producir solapamientos ni áreas de entrada incorrectas.

### 4.7 Snap

Modos iniciales:

- Mitad izquierda.
- Mitad derecha.
- Cuarto superior izquierdo.
- Cuarto superior derecho.
- Cuarto inferior izquierdo.
- Cuarto inferior derecho.

La vista previa del destino:

- Fondo: `accent` al 15 %.
- Borde: `accent`.
- Radio: `radius.window`.
- No recibe entrada.
- No modifica la geometría real hasta confirmar el movimiento.

El usuario puede cancelar el snap antes de soltar la ventana.

El algoritmo debe considerar el área de trabajo disponible, monitores, escala y espacios configurados.

### 4.8 Minimizada

La ventana minimizada:

- No tiene superficie visible en el escritorio.
- No participa en la composición.
- Mantiene su identidad y estado de aplicación.
- Aparece en el Dock.
- Puede restaurarse desde el Dock o mediante un comando de ventana.

La minimización es una política propia de PARALLAX; no asumir que `xdg-shell` proporciona una operación estándar de minimizar.

La aplicación no debe cerrarse ni terminarse al minimizarla.

### 4.9 Moving / Resizing

Durante movimiento o redimensionamiento:

- Sombra: `elev.4`.
- Conservar la decoración.
- Mantener el estado de foco.
- Utilizar cursor según borde o dirección.
- Respetar restricciones de tamaño del cliente.
- Mantener la geometría dentro de límites válidos.

Zona de agarre para redimensionar:

- Extensión inicial: 8 px alrededor del borde visible.
- Debe ser configurable.
- No debe invadir de forma ambigua la zona interactiva de ventanas adyacentes.
- No debe extenderse fuera de la geometría en fullscreen o tiled cuando el redimensionamiento esté deshabilitado.

Separar:

1. Geometría visual.
2. Geometría de entrada.
3. Extensión de sombra.
4. Zona de redimensionamiento.

Estas geometrías no deben confundirse.

### 4.10 Modal hijo

Una ventana modal debe estar relacionada con su padre mediante la relación de superficies correspondiente.

Cuando exista soporte del protocolo adecuado:

- Mantener la relación padre-hijo.
- Respetar la modalidad.
- Mantener la ventana dentro del contexto de su padre.
- Impedir la interacción con el padre mientras la modalidad esté activa.
- Gestionar correctamente múltiples niveles de diálogos.

Apariencia:

- Radio: `radius.window`.
- Sombra: `elev.5`.
- Padre atenuado mediante `scrim` al 30 %.
- Centrado sobre el padre cuando corresponda.

El scrim no sustituye al bloqueo real de entrada.

No asumir que toda ventana transitoria es modal.

Si el padre desaparece, se cierra o cambia de workspace, aplicar una política definida de cierre, reubicación o gestión del diálogo.

### 4.11 No responde

Utilizar el mecanismo `ping/pong` de `xdg_wm_base` para detectar clientes que no responden.

Timeout inicial: 5 segundos, configurable y sujeto a validación.

Al vencer:

- Marcar la ventana como `Unresponsive`.
- Mostrar un indicador visual.
- Ofrecer las acciones «Esperar» y «Forzar cierre».
- Evitar bloquear el bucle de eventos del compositor.

Apariencia:

- Mantener geometría y decoración.
- Desaturar visualmente el contenido un 50 %, si la ruta de composición lo permite.
- Mostrar un diálogo de recuperación gestionado por PARALLAX.

El timeout no demuestra por sí solo que la aplicación esté bloqueada. Considerar latencia, carga del sistema y actividad de renderizado antes de declarar un fallo definitivo.

«Esperar» permite continuar supervisando el cliente.

«Forzar cierre» solicita primero el cierre normal cuando proceda y, tras confirmación del usuario, termina el proceso o scope correspondiente mediante el gestor de sesión.

El compositor no debe matar procesos arbitrarios fuera del ámbito autorizado.

### 4.12 Urgente

Una solicitud de activación que no puede o no debe recibir foco se representa como `Urgent`.

Comportamiento:

- Indicador en el Dock.
- Punto de notificación en topbar.
- Sin robo automático del foco.
- Permitir activación explícita por el usuario.

La solicitud se evalúa mediante `xdg-activation` y la política de foco del compositor.

Un token inválido, ausente, caducado o no verificable no concede permiso de activación.

Un token válido tampoco obliga al compositor a romper las políticas de foco: la activación debe evaluarse según la procedencia, contexto de interacción y política de sesión.

La señal de urgencia debe ser visualmente distinguible de una notificación normal.

### 4.13 Overview

Durante Overview:

- Reducir las ventanas mediante una transformación de escala.
- Mantener proporciones.
- Sombra: `elev.3`.
- Título debajo de cada ventana.
- Hover: anillo `accent`.
- Selección explícita para activar.
- No permitir interacción accidental con controles de la ventana transformada.

El Overview debe mantener una representación consistente de las ventanas minimizadas, ocultas y de otros workspaces según la política de producto.

## 5. Geometría y renderizado

### 5.1 Esquinas redondeadas

Las esquinas se aplican mediante una máscara o shader de recorte en la composición.

No modificar ni solicitar al cliente que vuelva a renderizar su contenido para aplicar el radio.

Consideraciones:

- Evitar halos en los bordes.
- Evitar filtrado incorrecto.
- Mantener correspondencia entre recorte visual y entrada.
- Respetar transparencia del buffer.
- Desactivar el efecto cuando sea incompatible con una ruta de presentación directa.

### 5.2 Sombras

Las sombras se dibujan fuera de la geometría del contenido.

No deben:

- Reducir el área de contenido.
- Interceptar entrada.
- Cambiar el tamaño lógico de la ventana.
- Afectar al cálculo de layout.
- Crear zonas de solapamiento no definidas.

La sombra pertenece al compositor y no debe enviarse al cliente como contenido.

### 5.3 Escalado

Todos los valores geométricos deben considerar la escala del output.

Los valores en píxeles de diseño se convertirán correctamente a coordenadas lógicas y físicas.

El cambio de escala debe actualizar:

- Tamaños de decoración.
- Radios.
- Sombras.
- Zonas de redimensionamiento.
- Áreas de interacción.
- Vista previa de snap.

## 6. Reglas de foco

- El foco es propiedad del compositor.
- No cambiar el foco únicamente por hover.
- Las solicitudes de activación deben pasar por la política de `xdg-activation`.
- Las ventanas urgentes no roban foco.
- Las ventanas modales bloquean la interacción de acuerdo con su modalidad.
- Cambiar de workspace debe conservar una política coherente de foco.
- Al cerrar la ventana activa, seleccionar la siguiente ventana según el orden de foco definido.
- Nunca activar una ventana que haya sido destruida o cuya identidad haya quedado obsoleta.

El identificador interno de ventana debe ser estable durante su ciclo de vida y no reutilizarse de forma que pueda confundir eventos pendientes.

## 7. Animaciones

Las transiciones visuales deben seguir los tokens globales de motion:

- Rápidas: 120 ms.
- Normales: 200 ms.
- Complejas: 320 ms.

Animar únicamente propiedades compatibles con el presupuesto de renderizado.

Priorizar transformaciones y opacidad.

Respetar `prefers-reduced-motion` o la configuración equivalente de PARALLAX.

En modo de movimiento reducido, sustituir desplazamientos complejos por transiciones simples o inmediatas.

Las animaciones no deben bloquear entrada ni impedir la respuesta a solicitudes del usuario.

## 8. Integración con `px-wm`

El WM mantiene el estado lógico de las ventanas y calcula las operaciones necesarias.

El compositor:

- Traduce eventos Wayland a acciones de `px-wm`.
- Ejecuta las operaciones de geometría y foco.
- Gestiona superficies, buffers y renderizado.
- Aplica decoraciones y efectos.
- Publica los cambios a la Shell mediante los mecanismos definidos en la arquitectura IPC.

`px-wm` no debe depender directamente de Smithay, DRM, GPU o APIs Wayland.

Las transiciones deben mantener invariantes de geometría, foco, workspace y ciclo de vida.

## 9. Pruebas obligatorias

### Funcionales

- Crear y destruir ventanas.
- Maximizar y restaurar.
- Fullscreen y salida de fullscreen.
- Tiling y snap.
- Minimizar y restaurar.
- Movimiento y redimensionamiento.
- Cambio de workspace.
- Cambio de monitor.
- Cambio de escala.
- Ventanas modales.
- Clientes que no responden.
- Solicitudes de activación válidas e inválidas.

### Visuales

- Activa e inactiva.
- Hover.
- Estados de controles.
- Sombras y recorte.
- Escalado.
- Overview.
- Transiciones.
- Tema claro y oscuro.
- Movimiento reducido.

### Robustez

- Destrucción de ventana durante una animación.
- Desconexión de cliente.
- Cierre del padre de una ventana modal.
- Desconexión de monitor.
- Cambio de resolución.
- Pérdida y recuperación de GPU.
- Solicitudes de activación obsoletas.
- Saturación de eventos.
- Fallo de renderizado.

### Rendimiento

- Coste de composición de sombras.
- Coste del recorte redondeado.
- Frame time con múltiples ventanas.
- Tiling con múltiples ventanas.
- Overview con múltiples superficies.
- Consumo de memoria de decoraciones.
- Latencia de movimiento y redimensionamiento.
- Comportamiento con direct scanout habilitado y deshabilitado.

## 10. Criterios de aceptación

La especificación se considera implementada cuando:

- Las ventanas mantienen estados independientes y coherentes.
- El foco no se roba por hover ni solicitudes no autorizadas.
- SSD y CSD tienen una estrategia de compatibilidad.
- Maximizado, fullscreen, tiled y snap funcionan con múltiples monitores.
- Las zonas de entrada coinciden con la geometría visual.
- Las sombras no modifican el área de contenido.
- La modalidad bloquea realmente la entrada al padre.
- Los clientes que no responden pueden gestionarse sin bloquear el compositor.
- El recorte no produce artefactos visuales.
- Direct scanout se utiliza solo cuando las condiciones técnicas lo permiten.
- Los comportamientos se validan en HW-A y HW-B.

**Estado: Propuesto. Pendiente de aprobación y de implementación progresiva.**