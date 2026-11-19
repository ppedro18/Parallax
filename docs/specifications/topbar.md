# Especificación: PARALLAX Topbar

**Componente:** `parallax-shell`  
**Módulo:** Topbar  
**Estado:** Propuesto · Pendiente de aprobación  
**Capa:** C5 — Desktop Experience  
**Dependencias:** compositor, `px-wm`, `settingsd`, `notifyd`, NetworkManager, PipeWire, BlueZ y UPower.

## 1. Objetivo

La Topbar es la superficie principal de información y acceso rápido al estado de la sesión.

Debe mantener una composición visual limpia, accesible y adaptable a diferentes outputs.

No es propietaria de los datos que muestra: cada indicador consume el servicio correspondiente.

## 2. Arquitectura y superficie

- Una superficie layer-shell `top` por output configurado.
- Altura inicial: 32 px lógicos.
- Margen: 6 px.
- Radio: `radius.lg`.
- Modo flotante o pegado al borde.
- Transparencia según tokens.
- Posicionamiento configurable.

El uso de zona exclusiva depende del modo de Topbar.

No permitir que la Topbar invada el área de bloqueo ni interfiera con superficies de seguridad.

## 3. Distribución

| Zona | Elemento | Fuente |
|---|---|---|
| Izquierda | Logo / menú de sistema | Shell |
| Izquierda | Workspaces | compositor |
| Izquierda | App activa | compositor |
| Centro | Fecha y hora | reloj local |
| Derecha | Red | NetworkManager |
| Derecha | Audio | PipeWire |
| Derecha | Bluetooth | BlueZ |
| Derecha | Batería | UPower |
| Derecha | Notificaciones | notifyd |
| Derecha | Quick Settings | Shell |

La distribución debe ser configurable sin permitir que un elemento tape otro.

En outputs estrechos, aplicar prioridades y colapsar indicadores secundarios.

## 4. Menú del sistema

Clic en el logo:

- Acerca de.
- Ajustes.
- Suspender.
- Reiniciar.
- Apagar.
- Cerrar sesión.

Las acciones se ejecutan mediante servicios del sistema.

No ejecutar `shutdown`, `reboot` o comandos equivalentes directamente desde la UI.

Las operaciones que requieren autorización deben utilizar la política correspondiente.

## 5. Workspaces

Fuente: estado del compositor y protocolo de workspace seleccionado.

Interacción:

- Clic: activar workspace.
- Scroll: siguiente/anterior, configurable.
- Drag: mover ventana al workspace indicado.

No asumir que un protocolo privado es necesario si el protocolo estándar soportado cubre la función.

Los cambios se reflejan mediante snapshots y deltas.

## 6. Aplicación activa

Mostrar la aplicación con foco.

Fuente: `focus_changed` del compositor.

Clic abre el menú de aplicación:

- Cerrar.
- Flotar.
- Mover a workspace.
- Acciones permitidas por el compositor.

No ejecutar acciones sobre una ventana obsoleta: validar el identificador antes de enviar el comando.

## 7. Reloj

- Hora local.
- Zona horaria configurada.
- Formato 12/24 horas.
- Fecha opcional.
- Segundos opcionales.

Clic abre calendario.

No actualizar cada frame. Utilizar el intervalo mínimo necesario para la presentación elegida.

## 8. Indicadores

### Red

NetworkManager:

- Estado.
- Tipo de conexión.
- Acceso al tile de red.

### Audio

PipeWire:

- Salida.
- Volumen.
- Entrada.
- Acceso al panel.

Scroll sobre el indicador puede modificar volumen si el usuario lo habilita.

### Bluetooth

BlueZ.

Ocultar cuando no exista adaptador compatible o mostrar estado no disponible según preferencia.

### Batería

UPower.

Ocultar en equipos sin batería o mostrar estado de energía según configuración.

Tooltip con tiempo estimado únicamente cuando esté disponible.

### Notificaciones

notifyd.

Mostrar punto cuando existan notificaciones no leídas.

Clic abre Control Center.

### Quick Settings

Abre el panel de control.

## 9. Estados

- Normal.
- Fullscreen activo.
- Modo inmersivo.
- Servicio no disponible.
- Sesión bloqueada.
- Output desconectado.

Fullscreen y modo inmersivo no son necesariamente equivalentes.

El compositor determina la política de visibilidad.

Un servicio caído nunca debe provocar que Shell falle.

## 10. Accesibilidad

- Cada elemento tiene nombre accesible.
- Botones con rol correcto.
- Focus visible.
- Tooltips.
- Navegación por teclado.
- Escalado.
- Alto contraste.
- Reduced motion.

Atajo:

`Super+F10` enfoca Topbar.

La navegación debe permitir recorrer los elementos sin entrar en bucles de foco.

## 11. Rendimiento

- No polling continuo.
- Eventos de los servicios.
- Cache de iconos.
- Reloj con actualización programada.
- Sin frames vacíos.
- Actualizaciones coalescidas.
- Sin bloquear el hilo UI.

## 12. Pruebas y aceptación

- [ ] Un output.
- [ ] Varios outputs.
- [ ] Output desconectado.
- [ ] Aplicación activa.
- [ ] Workspace.
- [ ] Red.
- [ ] Audio.
- [ ] Bluetooth ausente.
- [ ] Batería ausente.
- [ ] Servicio caído.
- [ ] Fullscreen.
- [ ] Modo inmersivo.
- [ ] Accesibilidad.
- [ ] Super+F10.
- [ ] Escalado y densidad.
- [ ] Reposo sin polling.

## 13. Decisiones pendientes

1. Definir distribución adaptable.
2. Definir configuración por output.
3. Definir política de fullscreen e inmersivo.
4. Definir protocolo de workspace.
5. Definir comportamiento en outputs estrechos.
6. Definir preferencias de scroll sobre audio.

**Estado: Propuesto. Pendiente de aprobación.**