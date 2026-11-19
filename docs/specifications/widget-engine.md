# Especificación: Widget Engine — `px-widgets`

**Componente:** `px-widgets`  
**Host:** `parallax-shell`  
**Estado:** Propuesto · Pendiente de aprobación  
**Dependencias:** `px-ui`, `settingsd`, `resourced`, `notifyd`, modelo de eventos y ADR-0006.

## 1. Objetivo

El Widget Engine permite añadir componentes informativos e interactivos al escritorio PARALLAX.

Debe proporcionar widgets nativos de primera parte y permitir widgets externos aislados.

La seguridad y el rendimiento del Shell tienen prioridad sobre la extensibilidad.

## 2. Tipos

### 2.1 Internos

Se ejecutan dentro del proceso Shell.

Widgets iniciales:

- Calendar.
- Weather.
- Media.
- Audio.
- Network.
- Bluetooth.
- Battery.
- Performance.
- Processes.
- Notifications.
- Quick Settings.
- Workspace.
- Clock.
- Power.

Solo los widgets compilados y revisados por PARALLAX pueden ejecutarse en proceso.

### 2.2 Externos

Los widgets de terceros se ejecutan en procesos separados.

Publican una descripción declarativa de UI.

No pueden ejecutar código dentro del proceso Shell.

El contrato debe limitarse a un subconjunto seguro de componentes `px-ui`.

## 3. Manifiesto

Ejemplo:

```toml
id = "org.parallax.widget.performance"
name = "Rendimiento"
version = "1.0.0"

sizes = ["small", "medium", "large"]

permissions = [
  "resources.read"
]

topics = [
  "resources.metrics",
  "pressure"
]

min_update_ms = 1000
```

Campos obligatorios:

- ID.
- Nombre.
- Versión.
- Tamaños.
- Permisos.
- Topics.
- Frecuencia mínima.
- Esquema de configuración.
- Autor.
- Licencia.

El manifiesto declara lo que el widget solicita; no le concede permisos automáticamente.

## 4. Ciclo de vida

Estados:

- Registered.
- Instantiated.
- Active.
- Suspended.
- Error.
- Destroyed.

Transiciones controladas por el Widget Manager.

### 4.1 Suspensión

Suspender cuando:

- No está visible.
- Workspace oculto.
- Sesión bloqueada.
- Pantalla apagada.
- Presión de recursos elevada.
- El usuario lo solicita.

Suspender significa:

- Sin timers activos.
- Sin trabajo periódico.
- Sin suscripciones innecesarias.
- Sin renderizado.
- Sin mantener operaciones costosas.

Las operaciones necesarias para cerrar o liberar recursos deben completarse antes de suspender.

### 4.2 Frecuencia

La frecuencia efectiva debe respetar:

- `min_update_ms`.
- Política de energía.
- Política de recursos.
- Visibilidad.
- Preferencias del usuario.

En batería puede aplicarse ×2 como política inicial, salvo que el widget tenga una necesidad justificada.

## 5. API interna

Contrato conceptual:

```rust
trait Widget {
    fn manifest(&self) -> &Manifest;

    fn update(
        &mut self,
        event: &Event,
    ) -> Option<Redraw>;

    fn view(
        &self,
        size: Size,
    ) -> px_ui::Element;

    fn config_schema(&self) -> Schema;
}
```

La API definitiva debe evitar que el widget:

- Acceda directamente a servicios privilegiados.
- Modifique archivos de configuración.
- Realice operaciones de red sin permiso.
- Bloquee el event loop.
- Mantenga timers sin control.
- Genere trabajo de renderizado ilimitado.

## 6. Widgets externos

### 6.1 Proceso

Cada widget externo tiene su propio proceso y límites de recursos.

El proceso se ejecuta con el usuario actual y capacidades restringidas.

### 6.2 Transporte

El transporte propuesto es D-Bus para control y memfd para payloads grandes cuando sea necesario.

No utilizar memfd como requisito para cada actualización pequeña.

Comenzar con mensajes declarativos acotados y evaluar memfd mediante benchmark.

### 6.3 Árbol declarativo

El árbol UI debe incluir:

- Componentes permitidos.
- Propiedades tipadas.
- Acciones declaradas.
- Accesibilidad.
- Tamaños.
- Versión del protocolo.

No permitir HTML, JavaScript, código nativo o shaders arbitrarios.

El Shell valida el árbol antes de renderizarlo.

### 6.4 Aislamiento

- Sin acceso directo a D-Bus del sistema por defecto.
- Sin red por defecto.
- Filesystem restringido.
- Límites CPU.
- Límites memoria.
- Timeout.
- Reinicio controlado.
- Revocación de permisos.

## 7. Permisos

Permisos iniciales:

- `resources.read`
- `notifications.read`
- `network`
- `media.read`
- `audio.control`

El permiso `network` debe concederse explícitamente.

Weather puede solicitarlo para obtener información actualizada.

Sin conexión debe mostrar el último dato disponible con timestamp, si existe.

No afirmar que Weather es el único widget que puede usar red: los permisos se definen por capacidad, no por nombre de widget.

## 8. Operaciones de usuario

- Añadir desde galería.
- Eliminar.
- Mover.
- Redimensionar.
- Ocultar.
- Configurar.
- Restablecer.
- Revocar permisos.

El layout se persiste en:

`widgets.toml`

La configuración debe ser propiedad de settingsd o de un servicio de configuración de widgets formalmente definido.

No permitir que cada widget escriba directamente el archivo compartido.

## 9. Layout

- Rejilla de 8 px.
- Tamaños small, medium y large.
- Posicionamiento por output.
- Límites de colisión.
- Persistencia.
- Reacomodo al cambiar resolución o escala.

La UI debe permitir que los widgets se adapten a distintas densidades y tamaños.

## 10. Errores

Widget interno:

- Fallo en `update`: capturar error recuperable.
- Tres fallos consecutivos: mostrar estado Error.
- Ofrecer reintentar.
- No bloquear Shell.

Widget externo:

- Sin respuesta en 500 ms: marcar contenido como no actualizado.
- Mantener último estado válido si es seguro.
- Mostrar indicador.
- Reiniciar según política.
- Revocar suscripciones si se destruye.

El timeout no debe implicar que el proceso se haya terminado.

## 11. Seguridad

- Validar manifiestos.
- Validar permisos.
- Validar árboles declarativos.
- Limitar tamaños de mensajes.
- Limitar frecuencia de actualización.
- Evitar deserialización ilimitada.
- Aislar procesos.
- No permitir código arbitrario en Shell.
- Registrar errores sin datos sensibles.

## 12. Rendimiento

- Widgets invisibles suspendidos.
- Sin polling redundante.
- Coalescing de eventos.
- Redraw solo cuando cambia contenido.
- Frecuencia limitada.
- Cache de recursos.
- Presupuesto por widget.
- Presupuesto total del Shell.

## 13. Pruebas

- [ ] Registro.
- [ ] Instalación.
- [ ] Eliminación.
- [ ] Permisos.
- [ ] Revocación.
- [ ] Árbol inválido.
- [ ] Mensaje excesivo.
- [ ] Widget lento.
- [ ] Widget que falla.
- [ ] Widget externo desconectado.
- [ ] Suspensión y reanudación.
- [ ] Presión de recursos.
- [ ] Red no disponible.
- [ ] Persistencia.
- [ ] Multi-monitor.
- [ ] Accesibilidad.
- [ ] Aislamiento de proceso.

## 14. Decisiones pendientes

1. Definir protocolo declarativo versionado.
2. Definir API de instalación.
3. Definir política de firma y procedencia.
4. Definir capacidades exactas.
5. Definir límites de recursos.
6. Definir persistencia.
7. Definir compatibilidad con ADR-0006.
8. Evaluar si memfd aporta valor real.

**Estado: Propuesto. Pendiente de aprobación.**