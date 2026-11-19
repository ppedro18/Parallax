# Especificación: PARALLAX Settings

**Aplicación:** `parallax-settings`  
**Servicio de configuración:** `settingsd`  
**Estado:** Propuesto · Pendiente de aprobación  
**Capa:** C6 — Native Apps

## 1. Objetivo

PARALLAX Settings es la aplicación central para configurar el escritorio y administrar las preferencias disponibles del sistema.

Es un cliente de servicios especializados. No es propietario de los archivos de configuración ni debe ejecutar operaciones privilegiadas directamente.

Cada área tiene un servicio responsable de validar y aplicar los cambios.

## 2. Principios

1. Settings es una interfaz cliente.
2. `settingsd` administra las preferencias propias de PARALLAX.
3. Los servicios del sistema mantienen la autoridad sobre sus dominios.
4. Las operaciones privilegiadas utilizan mecanismos de autorización.
5. Las preferencias inválidas se rechazan.
6. Los cambios temporales que puedan dejar la pantalla inutilizable deben ser reversibles.
7. La UI debe informar cuando un servicio no está disponible.
8. No escribir archivos de sistema directamente.

## 3. Arquitectura

```text
parallax-settings
       |
       +---- Settings1
       |
       +---- Session1
       |
       +---- Display Management
       |
       +---- NetworkManager
       |
       +---- PipeWire
       |
       +---- BlueZ
       |
       +---- UPower
       |
       +---- systemd
       |
       +---- polkit
       |
       +---- updated / recoveryd / hardwared
```

Cada integración debe usar APIs oficiales.

No crear una segunda base de configuración que contradiga al servicio propietario.

## 4. Secciones

| Sección | Propietario |
|---|---|
| Apariencia, temas, fondos, transparencia | settingsd |
| Animaciones, widgets, Dock, Topbar | settingsd |
| Workspaces y preferencias de escritorio | settingsd / compositor |
| Pantallas | compositor |
| Teclado, ratón y touchpad | settingsd + compositor |
| Audio | PipeWire / WirePlumber |
| Red | NetworkManager |
| Bluetooth | BlueZ |
| Energía | power-profiles-daemon, UPower, logind |
| Usuarios | accountsservice y polkit |
| Seguridad | polkit y servicios correspondientes |
| Aplicaciones predeterminadas | mimeapps.list |
| Autostart | FreeDesktop / configuración de usuario |
| Permisos de portales | permission store |
| Servicios | systemd D-Bus |
| Actualizaciones | updated |
| Recuperación | recoveryd |
| Hardware | hardwared |

La tabla define propietarios de datos y acciones, no implica que todos los servicios deban exponerse directamente a la UI.

## 5. Esquemas de configuración

Cada clave de configuración debe tener:

- Identificador.
- Tipo.
- Valor predeterminado.
- Rango o valores permitidos.
- Descripción.
- Ámbito.
- Propietario.
- Versión de esquema.
- Política de migración.

Ejemplo conceptual:

```toml
[keys."theme.mode"]
type = "enum"
values = ["dark", "light", "auto"]
default = "dark"

[keys."dock.position"]
type = "enum"
values = ["bottom", "left", "right"]
default = "bottom"
```

`settingsd` valida el esquema antes de persistir.

No permitir valores arbitrarios que puedan inyectar código, CSS o scripts.

## 6. Apariencia

Opciones:

- Tema claro.
- Tema oscuro.
- Automático.
- Alto contraste.
- Color de acento.
- Fondo por output.
- Transparencia.
- Blur.
- Densidad.
- Radios.
- Animaciones.
- Tipografía.
- Escala de texto.
- Posición y tamaño de Dock.
- Topbar.
- Widgets.

Los cambios se aplican mediante el modelo de tokens definido en el sistema de diseño.

## 7. Pantallas

La configuración de outputs pertenece al compositor.

Debe incluir:

- Resolución.
- Refresh rate.
- Escala.
- Orientación.
- Posición.
- VRR.
- Color cuando esté soportado.

### 7.1 Aplicación temporal

Al modificar una configuración potencialmente problemática:

1. Aplicar cambio temporal.
2. Iniciar cuenta regresiva de 15 segundos.
3. Mostrar diálogo de confirmación.
4. Si el usuario confirma, persistir.
5. Si no confirma o pierde conexión, revertir.

La operación debe tener un identificador para evitar confirmar una transacción obsoleta.

### 7.2 Recuperación

Si el compositor o Settings cae durante la cuenta regresiva, el compositor debe poder revertir el estado temporal.

No depender exclusivamente de que la UI permanezca viva.

## 8. Teclado, ratón y touchpad

`settingsd` almacena preferencias.

El compositor aplica la configuración de dispositivos de entrada.

Opciones:

- Layout.
- Repetición.
- Velocidad.
- Aceleración.
- Natural scrolling.
- Tap-to-click.
- Gestos.
- Atajos.

Los dispositivos se configuran según capacidades detectadas.

No mostrar opciones incompatibles como si estuvieran disponibles.

## 9. Audio, red y Bluetooth

### Audio

PipeWire/WirePlumber:

- Dispositivos.
- Volumen.
- Entrada.
- Salida.
- Perfiles.
- Aplicaciones.

### Red

NetworkManager:

- Conexiones.
- Wi-Fi.
- Estado.
- VPN cuando esté soportado.

### Bluetooth

BlueZ:

- Adaptadores.
- Dispositivos.
- Emparejamiento.
- Conexión.
- Desconexión.

Los errores de servicio deben reflejarse en la interfaz sin provocar fallos de Settings.

## 10. Energía

Integraciones:

- power-profiles-daemon.
- UPower.
- logind.

Configuraciones:

- Perfil de energía.
- Acciones de tapa.
- Suspensión.
- Hibernación cuando esté soportada.
- Preferencias de batería.

No duplicar políticas del servicio de energía.

## 11. Usuarios y seguridad

La administración de usuarios se realiza mediante accountsservice y las políticas autorizadas.

Settings no modifica directamente `/etc/passwd`, grupos, PAM o archivos de seguridad.

La sección Seguridad puede mostrar:

- Estado de polkit.
- Estado de firewall.
- Estado de cifrado de disco.
- Estado de actualizaciones de seguridad.

En v1, la información de firewall y cifrado puede ser de solo lectura si no existe un servicio de gestión validado.

## 12. Aplicaciones

Configurar:

- Aplicaciones predeterminadas.
- Asociaciones MIME.
- Autostart.
- Permisos de portales.

No conceder permisos de portal mediante una modificación arbitraria de archivos: utilizar la interfaz de permission store correspondiente.

## 13. Servicios

Mostrar:

- Estado.
- Disponibilidad.
- Último error.
- Información de diagnóstico.

Permitir reiniciar servicios de usuario cuando sea seguro.

Los servicios de sistema se gestionan mediante políticas de autorización.

No exponer un terminal de comandos dentro de Settings.

## 14. Actualización y recuperación

Páginas específicas para:

- Actualizaciones.
- Canales.
- Historial.
- Estado de actualización.
- Snapshots.
- Recuperación.
- Hardware.
- Diagnóstico.

Los trabajos deben mostrar estados, progreso y errores mediante los contratos de servicios.

## 15. Búsqueda global

La búsqueda indexa:

- Nombre de ajuste.
- Descripción.
- Categoría.
- Palabras clave.
- Alias.

Debe funcionar sin requerir conexión de red.

Los resultados deben abrir directamente la sección correspondiente.

## 16. Estado de servicios

Cada página debe manejar:

- Disponible.
- Conectando.
- No disponible.
- Sin hardware compatible.
- Sin autorización.
- Error temporal.

No ocultar silenciosamente una funcionalidad esencial cuando el servicio falla.

## 17. Pruebas y aceptación

- [ ] Validación de esquemas.
- [ ] Rechazo de valores inválidos.
- [ ] Aplicación en caliente.
- [ ] Reinicio de settingsd.
- [ ] Recuperación de configuración.
- [ ] Reversión de display en 15 segundos.
- [ ] Desconexión del compositor.
- [ ] Servicios ausentes.
- [ ] Polkit.
- [ ] Asociaciones MIME.
- [ ] Accesibilidad.
- [ ] Búsqueda offline.
- [ ] No escribe archivos de sistema directamente.

## 18. Decisiones pendientes

1. Definir esquema versionado.
2. Definir API Settings1.
3. Definir contrato de display rollback.
4. Definir migraciones.
5. Definir integración de permission store.
6. Definir inventario definitivo de preferencias.

**Estado: Propuesto. Pendiente de aprobación.**