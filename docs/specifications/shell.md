# Especificación: `parallax-shell`

**Componente:** `parallax-shell`  
**Estado:** Propuesto · Pendiente de aprobación  
**Capa:** C5 — Desktop Experience  
**Arquitectura:** Un único proceso para las superficies nativas del escritorio.

## 1. Objetivo

`parallax-shell` implementa la experiencia visual e interactiva del escritorio PARALLAX.

Integra:

- Desktop.
- Topbar.
- Dock.
- Launcher.
- Widgets internos.
- Notification UI.
- Control Center.
- Workspace Overview.
- Polkit agent UI.

El Shell utiliza `px-ui` y los protocolos Wayland correspondientes.

No es propietario del estado lógico de ventanas, dispositivos ni servicios del sistema.

## 2. Arquitectura

Un único proceso para los elementos nativos de Shell.

```text
parallax-shell
│
├── Desktop
├── Topbar
├── Dock
├── Launcher
├── Widget Host
├── Notification UI
├── Control Center
├── Workspace Overview
└── Internal Coordination
```

Cada superficie que requiera comportamiento independiente utiliza su propia superficie Wayland layer-shell.

El proceso comparte:

- Toolkit.
- Contexto de configuración.
- Recursos gráficos.
- Cache de iconos.
- Estado derivado.
- Coordinación interna.

Los widgets de terceros no se ejecutan dentro del proceso Shell.

## 3. Fuentes de verdad

| Información | Propietario |
|---|---|
| Ventanas, foco, outputs, workspaces | compositor / px-wm |
| Preferencias | settingsd |
| Notificaciones | notifyd |
| Métricas | resourced |
| Hardware | hardwared |
| Red | NetworkManager |
| Bluetooth | BlueZ |
| Audio | PipeWire |
| Energía | UPower / power-profiles-daemon |
| Autorización | polkit |

El Shell consume snapshots y deltas.

No mantiene copias autoritativas ni reemite eventos de otros servicios.

## 4. Desktop

### 4.1 Fondo

- Imagen.
- Color.
- Gradiente.
- Configuración por output.
- Ajuste de escala.
- Posicionamiento.
- Fallback cuando el archivo desaparece.

Capa: `background`.

### 4.2 Escritorio espacial

El escritorio utiliza el concepto de planos en profundidad.

Workspace Overview se activa con `Super`.

Debe mostrar:

- Workspaces.
- Ventanas reales.
- Miniaturas autorizadas.
- Indicadores de actividad.
- Transiciones espaciales.

El movimiento y profundidad deben seguir `DEPTH_SYSTEM`.

No utilizar miniaturas capturadas sin consentimiento o acceso autorizado del compositor.

### 4.3 Iconos

Los iconos tradicionales en el escritorio no forman parte de v1.

El escritorio está destinado a fondo y widgets.

### 4.4 Widgets

Los widgets se posicionan en la capa `bottom`, sujetos a la política de stacking y a la sesión de bloqueo.

## 5. Launcher

Atajo:

`Super+Espacio`

Fuentes:

- Aplicaciones `.desktop`.
- Archivos recientes.
- Ajustes.
- Acciones del sistema.
- Calculadora.

### 5.1 Indexación

Indexación de aplicaciones mediante archivos `.desktop`.

Utilizar notificaciones del filesystem cuando sea posible y reconciliar el índice cuando se pierdan eventos.

No indexar contenido completo del disco por defecto.

### 5.2 Proveedores

Los proveedores devuelven resultados estructurados.

El timeout inicial de 50 ms es un presupuesto de respuesta, no una garantía de ejecución.

Un proveedor lento no debe bloquear el resto.

Los proveedores externos requieren un modelo de aislamiento separado.

### 5.3 Ranking

- Coincidencia exacta.
- Coincidencia difusa.
- Frecuencia.
- Recencia.
- Preferencias del usuario.

El ranking debe ser local y transparente.

No enviar búsquedas a servicios externos.

### 5.4 Estados

- Empty.
- Typing.
- Results.
- NoResults.
- ProviderError.

La caída de un proveedor no bloquea Launcher.

### 5.5 Teclado

- Flechas: navegación.
- Enter: ejecutar.
- Tab: acciones secundarias.
- Escape: cerrar.

## 6. Notificaciones

Integración con `notifyd`.

Características:

- Banner superior derecho.
- Máximo tres visibles simultáneamente.
- Autocierre de cinco segundos por defecto.
- Críticas persistentes según política.
- Acciones.
- Respuesta inline cuando esté soportada.
- Historial en Control Center.
- DND programable.

Las notificaciones críticas no deben saltarse el bloqueo de sesión.

La UI debe respetar privacidad y reglas de visibilidad.

## 7. Control Center

Panel modular desde Topbar.

Tiles:

- Wi-Fi.
- Bluetooth.
- Audio.
- Brillo.
- Energía.
- DND.
- Tema.
- Captura.
- Bloqueo.
- Tiles configurables.

Cada tile define:

- Estado.
- Acción primaria.
- Acción secundaria.
- Estado de servicio.
- Accesibilidad.

Si el hardware no existe, ocultar el tile o mostrar una explicación cuando la ausencia sea relevante.

No fingir que una operación tuvo éxito antes de recibir confirmación del servicio.

## 8. Workspace Overview

Activación: `Super`.

Muestra los workspaces como planos en profundidad.

Permite:

- Cambiar workspace.
- Seleccionar ventana.
- Arrastrar ventanas.
- Crear workspace cuando esté soportado.
- Cerrar o mover ventanas mediante acciones explícitas.

Las miniaturas deben tener límites de memoria y frecuencia de actualización.

## 9. Polkit agent UI

El agente de autenticación polkit se integra visualmente con el sistema de diseño, pero su ciclo de vida debe ser independiente del Shell.

Recomendación:

- Servicio de agente independiente.
- Ventana o superficie administrada por Shell cuando esté disponible.
- Fallback de autenticación si Shell cae.
- Prioridad sobre superficies ordinarias.
- Nunca por encima de un estado de bloqueo de forma que lo eluda.

El Shell no debe convertirse en propietario de la autorización privilegiada.

## 10. Recuperación

El estado del Shell es derivado.

Tras reinicio:

1. Reconectar servicios.
2. Obtener snapshots.
3. Recuperar configuración.
4. Reconstruir superficies.
5. Restaurar Dock, Topbar, widgets y paneles.
6. Suscribirse a deltas.

No persistir copias redundantes del estado de ventanas.

### 10.1 Caída

systemd reinicia el Shell.

El compositor conserva sus funciones básicas y puede presentar un overlay de emergencia.

El overlay no debe ser una implementación alternativa completa del escritorio.

## 11. Rendimiento

Objetivo inicial:

- PSS ≤120 MB con Topbar, Dock y cuatro widgets.
- CPU en reposo <0,5 %.
- Sin timers innecesarios.
- Sin polling de servicios.
- Sin frames vacíos.

El objetivo de cero wakeups por segundo debe medirse en una escena definida; el reloj, input, notificaciones y cambios de servicios pueden producir actividad legítima.

Los presupuestos se verifican con la metodología de rendimiento.

## 12. Accesibilidad

- AccessKit.
- AT-SPI.
- Orca.
- Navegación por teclado.
- Focus visible.
- Escalado.
- High contrast.
- Reduced motion.
- Reduced transparency.

Cada superficie layer-shell debe tener un orden de foco coherente.

## 13. Pruebas

- [ ] Reinicio del Shell.
- [ ] Caída de un proveedor.
- [ ] Caída de servicios.
- [ ] Reconexión y snapshots.
- [ ] Launcher.
- [ ] Topbar.
- [ ] Dock.
- [ ] Widgets.
- [ ] Notificaciones.
- [ ] DND.
- [ ] Workspace Overview.
- [ ] Polkit.
- [ ] Bloqueo.
- [ ] Multi-monitor.
- [ ] Accesibilidad.
- [ ] Rendimiento.
- [ ] Ausencia de frames en reposo.

## 14. Decisiones pendientes

1. Definir protocolo interno entre módulos.
2. Definir contrato de recuperación con compositor.
3. Definir política de stacking.
4. Definir acceso a miniaturas.
5. Definir ciclo de vida del polkit agent.
6. Definir límites de proveedores del Launcher.

**Estado: Propuesto. Pendiente de aprobación.**