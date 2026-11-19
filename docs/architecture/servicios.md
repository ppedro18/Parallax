# Arquitectura de servicios — PARALLAX

- **Proyecto:** PARALLAX
- **Estado:** Propuesto — pendiente de validación
- **Ámbito:** Servicios de sesión, servicios de sistema, seguridad y dependencias Linux
- **Fecha:** 2026-11-19

# 1. Objetivo

Definir la arquitectura de servicios de PARALLAX, sus responsabilidades, permisos, dependencias y mecanismos de ejecución.

La arquitectura debe:

- Separar servicios de usuario y servicios de sistema.
- Aplicar mínimo privilegio.
- Reutilizar servicios maduros de Linux.
- Evitar duplicación de responsabilidades.
- Aislar operaciones privilegiadas.
- Garantizar recuperación ante fallos.
- Mantener bajo consumo de recursos.
- Permitir pruebas independientes por servicio.

**Principio fundamental:** cada servicio debe tener una responsabilidad concreta, una identidad de ejecución, permisos explícitos y contratos IPC documentados.

---

# 2. Clasificación de servicios

Se distinguen tres categorías.

## 2.1. Servicios de sesión

Se ejecutan dentro de la sesión del usuario.

Responsabilidades:

- Configuración personal.
- Notificaciones.
- Métricas de la sesión.
- Coordinación con Shell.
- Preferencias visuales.
- Integración con aplicaciones.

No deben tener privilegios administrativos.

## 2.2. Servicios de sistema

Se ejecutan independientemente de la sesión gráfica.

Responsabilidades:

- Inventario global de hardware.
- Actualización del sistema.
- Recuperación.
- Operaciones administrativas.
- Diagnóstico privilegiado.

Deben utilizar usuarios dedicados y permisos limitados.

## 2.3. Servicios auxiliares bajo demanda

Se inician únicamente cuando una operación los requiere.

Ejemplos:

- Diagnóstico SMART.
- Creación de snapshots.
- Restauración.
- Recopilación de informes.
- Operaciones de mantenimiento.

No deben mantenerse ejecutándose permanentemente sin necesidad.

---

# 3. Catálogo de servicios propios

| Servicio | Ejecución | Responsabilidad |
|---|---|---|
| `parallax-settingsd` | Sesión | Configuración y persistencia |
| `parallax-notifyd` | Sesión | Notificaciones e historial |
| `parallax-hardwared` | Sistema | Inventario de hardware |
| `parallax-resourced` | Sesión | Observación y políticas de recursos |
| `parallax-updated` | Sistema | Coordinación de actualizaciones |
| `parallax-recoveryd` | Sistema | Recuperación y snapshots |
| `parallax-diagd` | Sesión + helper bajo demanda | Diagnósticos |

La lista puede evolucionar, pero no se crearán servicios adicionales sin una responsabilidad suficientemente independiente.

---

# 4. Endurecimiento de servicios systemd

La seguridad se definirá individualmente por servicio.

No existe una plantilla universal que pueda aplicarse sin modificaciones a todos los procesos.

## 4.1. Perfil base de endurecimiento

Ejemplo conceptual para un servicio de sistema sin privilegios administrativos:

```ini
[Service]
User=parallax-service
Group=parallax-service

NoNewPrivileges=yes

ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes

ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes

RestrictAddressFamilies=AF_UNIX
SystemCallArchitectures=native
SystemCallFilter=@system-service

MemoryDenyWriteExecute=yes
LockPersonality=yes

RestrictSUIDSGID=yes
RestrictRealtime=yes

UMask=0077

Restart=on-failure
RestartSec=2s
```

Este perfil es una base de evaluación, no una unidad lista para copiar a todos los servicios.

## 4.2. Identidad

Cada servicio tendrá:

- Usuario dedicado cuando corresponda.
- Grupo dedicado cuando corresponda.
- Directorios de estado explícitos.
- Permisos mínimos.
- Sin acceso innecesario a directorios personales.
- Sin credenciales compartidas.

`DynamicUser=yes` se utilizará únicamente cuando la identidad dinámica sea compatible con las necesidades de persistencia y acceso del servicio.

Cuando un servicio deba conservar estado, se definirá explícitamente su directorio persistente y el propietario de los datos.

No se combinará `DynamicUser=yes` con una política de persistencia improvisada.

## 4.3. Acceso al sistema de archivos

`ProtectSystem=strict` será la configuración preferida cuando sea viable.

Los directorios de escritura se concederán explícitamente.

Ejemplos:

- Directorio de estado.
- Caché.
- Archivos temporales.
- Directorios de trabajo.

No se utilizará `ReadWritePaths=/` ni permisos generales equivalentes para solucionar problemas de acceso.

## 4.4. Acceso a dispositivos

`PrivateDevices=yes` será el valor inicial para servicios que no necesiten acceder a dispositivos reales mediante `/dev`.

Se desactivará únicamente cuando exista una necesidad documentada.

No se concederá acceso general a dispositivos de bloque, entrada, GPU o USB.

## 4.5. Familias de direcciones

La restricción dependerá de las necesidades reales:

| Servicio | Familias previstas |
|---|---|
| settingsd | AF_UNIX |
| notifyd | AF_UNIX |
| hardwared | AF_UNIX, AF_NETLINK |
| resourced | AF_UNIX |
| updated | AF_UNIX y AF_INET/AF_INET6 en el componente de descarga |
| recoveryd | AF_UNIX |
| diagd | AF_UNIX; otras solo si se justifican |

La lista es orientativa y debe comprobarse mediante pruebas.

## 4.6. Watchdog

`WatchdogSec=` se utilizará cuando el proceso implemente correctamente el protocolo `sd_notify`.

El servicio deberá:

- Enviar `READY=1` cuando corresponda.
- Enviar `WATCHDOG=1` dentro del intervalo requerido.
- Detectar operaciones bloqueadas.
- Evitar enviar señales de vida desde un hilo que no represente la salud real del proceso.

No se habilitará watchdog solamente para cumplir una plantilla.

## 4.7. Tipo de servicio

La selección dependerá del ciclo de vida:

| Tipo | Uso |
|---|---|
| `simple` / `exec` | Servicio convencional |
| `notify` | Servicio que informa explícitamente cuándo está listo |
| `dbus` | Servicio cuya disponibilidad se establece mediante adquisición de un nombre D-Bus |
| `oneshot` | Operación puntual |
| `forking` | Solo cuando sea imprescindible para software que se bifurca |

`Type=dbus` no se utilizará como valor universal.

## 4.8. Capacidades

`CapabilityBoundingSet=` vacío será el punto de partida.

Si una operación requiere una capacidad:

1. Identificar la operación.
2. Justificarla.
3. Evaluar alternativas sin privilegios.
4. Limitar la capacidad al proceso necesario.
5. Documentar la decisión.
6. Incorporar pruebas de seguridad.

No se concederá `CAP_SYS_ADMIN` como solución genérica.

---

# 5. settingsd

**Servicio:** `parallax-settingsd`

**Tipo:** servicio de sesión.

**Bus:** sesión.

**Responsabilidad:** gestionar la configuración persistente de PARALLAX.

## Funcionalidades

- Esquemas tipados.
- Validación.
- Migración de versiones.
- Lectura y escritura.
- Persistencia TOML.
- Valores predeterminados.
- Recuperación de configuración.
- Publicación de cambios.

## Persistencia

Ubicación propuesta:

```text
$XDG_CONFIG_HOME/parallax/
├── settings/
│   ├── appearance.toml
│   ├── display.toml
│   ├── behavior.toml
│   └── power.toml
└── shell/
    └── shell.toml
```

## Escritura atómica

1. Validar datos.
2. Crear archivo temporal.
3. Escribir contenido.
4. Sincronizar el archivo cuando corresponda.
5. Renombrar atómicamente.
6. Sincronizar el directorio cuando se requiera durabilidad.
7. Publicar el evento de cambio.

## Seguridad

- No necesita root.
- No modifica servicios de sistema directamente.
- No ejecuta comandos.
- No permite escrituras arbitrarias en rutas externas.
- Accede únicamente a la configuración de la sesión propietaria.

La configuración de sistema pertenece a los servicios responsables de cada funcionalidad.

---

# 6. notifyd

**Servicio:** `parallax-notifyd`

**Tipo:** servicio de sesión.

**Bus:** sesión.

**Responsabilidad:** administrar notificaciones.

## Funcionalidades

- Implementación de `org.freedesktop.Notifications`.
- Historial.
- DND.
- Agrupación por aplicación.
- Identificación de notificaciones.
- Gestión de expiración.
- Persistencia.

## Historial

Límite inicial: 500 notificaciones.

La política debe definir:

- Tamaño máximo por notificación.
- Límite de imágenes.
- Expiración.
- Política de agrupación.
- Política de eliminación.
- Protección de contenido sensible.

La persistencia debe estar limitada al usuario.

## Separación visual

`notifyd` no dibuja notificaciones.

`parallax-shell` se encarga de:

- Presentación.
- Animaciones.
- Posicionamiento.
- Interacciones visuales.

La caída del Shell no debe destruir el historial administrado por `notifyd`.

---

# 7. hardwared

**Servicio:** `parallax-hardwared`

**Tipo:** servicio de sistema.

**Usuario:** `parallax-hw`.

**Bus:** sistema.

**Responsabilidad:** inventario y observación de hardware.

## Funcionalidades

- Enumeración udev.
- Monitorización de hotplug.
- Lectura de sysfs.
- Identificación PCI/USB.
- Detección de controladores.
- Estado de firmware.
- Asociación de perfiles.
- Recomendaciones de compatibilidad.

## No hace

- Instalar drivers.
- Descargar firmware.
- Cargar módulos arbitrariamente.
- Cambiar parámetros del kernel.
- Controlar frecuencias.
- Administrar energía.

## Permisos

- Lectura limitada de sysfs.
- Acceso al mecanismo de monitorización udev.
- Acceso a la base de datos de dispositivos.
- Sin privilegios root permanentes.
- Sin escritura general en sysfs.
- Sin acceso indiscriminado a `/dev`.

`PrivateDevices=yes` debe evaluarse en el prototipo. Si el servicio necesita acceso a dispositivos reales, se concederán únicamente los accesos concretos necesarios.

---

# 8. resourced

**Servicio:** `parallax-resourced`

**Tipo:** servicio de sesión.

**Bus:** sesión.

**Responsabilidad:** observar recursos y coordinar políticas de rendimiento de la sesión.

## Fuentes

- PSI.
- cgroup v2.
- `/proc`.
- Información del compositor.
- `Session1`.
- UPower.
- Interfaces de energía disponibles.

## Funcionalidades

- Clasificación de aplicaciones.
- Observación de memoria.
- Observación de CPU.
- Observación de GPU cuando sea posible.
- Presión de memoria.
- Estado de aplicaciones.
- Políticas adaptativas.
- Métricas para Observatory.

## Gestión de cgroups

No debe escribir arbitrariamente en cgroupfs.

La integración debe respetar la propiedad de systemd sobre los scopes y slices.

Opciones:

- Interfaces D-Bus de systemd.
- Delegación explícita de un subárbol de cgroups.
- Políticas de usuario limitadas a scopes autorizados.

No se concederá control general de cgroups al proceso.

## No hace

- Sustituir systemd-oomd.
- Matar procesos arbitrariamente.
- Modificar frecuencias directamente.
- Alterar límites térmicos.
- Cambiar políticas globales sin autorización.

La terminación de procesos corresponde al gestor de procesos o a políticas explícitas del sistema.

---

# 9. updated

**Servicio:** `parallax-updated`

**Tipo:** servicio de sistema.

**Bus:** sistema.

**Responsabilidad:** coordinar actualizaciones de PARALLAX.

## Funcionalidades

- Consultar actualizaciones.
- Validar repositorios.
- Descargar paquetes.
- Verificar firmas.
- Preparar actualización.
- Aplicar actualización.
- Coordinar snapshots.
- Gestionar estados.
- Coordinar recuperación.

## Separación de privilegios

La descarga y la instalación no deben compartir automáticamente el mismo nivel de privilegio.

Modelo propuesto:

**Componente de descarga:**

- Descarga desde repositorios autorizados.
- Sin root.
- Sin acceso de escritura al sistema.
- Validación de tamaño y almacenamiento.
- Caché controlada.

**Componente privilegiado de aplicación:**

- Verificación de artefactos.
- Operaciones pacman.
- Preparación del sistema.
- Coordinación con snapshots.
- Actualización de entradas de arranque cuando corresponda.

El componente privilegiado no tendrá acceso de red general si no lo necesita.

## Reglas

- Repositorios firmados.
- Validación de paquetes.
- Confirmación del usuario según política.
- Operaciones largas mediante `Job1`.
- Registro de resultados.
- Recuperación ante interrupciones.
- No asumir que un snapshot garantiza atomicidad.

La estrategia definitiva depende del prototipo de actualización y recuperación definido en ADR-0009.

---

# 10. recoveryd

**Servicio:** `parallax-recoveryd`

**Tipo:** servicio de sistema.

**Usuario:** identidad privilegiada controlada.

**Bus:** sistema.

**Responsabilidad:** recuperación del sistema.

## Funcionalidades

- Listar snapshots.
- Crear snapshots.
- Validar snapshots.
- Restaurar snapshots.
- Coordinar recuperación de configuración.
- Preparar modo de recuperación.

## Seguridad

Las operaciones de restauración requieren autorización explícita.

El servicio debe:

- Validar identificadores.
- Verificar rutas.
- Comprobar estado de almacenamiento.
- Evitar restauraciones ambiguas.
- Registrar operaciones.
- Proteger los snapshots necesarios.
- Evitar borrar datos fuera de su ámbito.

No permitirá ejecutar comandos arbitrarios a través de D-Bus.

Las operaciones de recuperación deben ser trabajos asíncronos.

La recuperación de sistema y la recuperación de configuración del usuario serán operaciones diferentes.

---

# 11. diagd

**Servicio:** `parallax-diagd`

**Tipo:** servicio de sesión con helper privilegiado bajo demanda.

**Bus:** sistema o sesión según la operación.

**Responsabilidad:** diagnóstico y generación de informes.

## Diagnóstico no privilegiado

- Estado de servicios.
- Información de hardware disponible.
- Métricas de recursos.
- Logs accesibles.
- Configuración.
- Estado de aplicaciones.

## Diagnóstico privilegiado

- SMART.
- NVMe.
- Consultas restringidas.
- Operaciones que requieren acceso especial.

Estas tareas se ejecutarán mediante un helper limitado o una interfaz privilegiada concreta.

## Recopilación

El informe se generará localmente.

No se enviará información fuera del equipo.

El usuario debe poder revisar el contenido antes de exportarlo.

## Privacidad

Excluir por defecto:

- Contraseñas.
- Tokens.
- Cookies.
- Claves privadas.
- Números de serie.
- Información personal innecesaria.
- Contenido privado de archivos.

El informe debe incluir únicamente datos necesarios para el diagnóstico.

---

# 12. Servicios externos reutilizados

PARALLAX reutilizará los siguientes componentes.

No se reimplementarán sin una justificación técnica y un ADR.

| Componente | Responsabilidad |
|---|---|
| systemd | Gestión de servicios y arranque |
| systemd-logind | Sesiones y dispositivos |
| systemd-oomd | Gestión de presión de memoria |
| journald | Registro |
| greetd | Gestión del greeter |
| NetworkManager | Red |
| BlueZ | Bluetooth |
| PipeWire | Audio, vídeo y multimedia |
| WirePlumber | Política de PipeWire |
| UPower | Batería |
| power-profiles-daemon | Perfiles energéticos |
| polkit | Autorización |
| udisks2 | Gestión de almacenamiento extraíble |
| Mesa | OpenGL/Vulkan |
| fwupd | Actualización de firmware |
| xdg-desktop-portal | APIs de escritorio para aplicaciones |

PARALLAX consumirá estos servicios mediante sus interfaces existentes.

No creará servicios duplicados para las mismas responsabilidades.

---

# 13. XDG Desktop Portal

Se desarrollará un backend propio:

`xdg-desktop-portal-parallax`

**Lenguaje:** Rust.

**Responsabilidad:** integrar las operaciones de escritorio con el Shell y el compositor.

## Interfaces a evaluar

- FileChooser.
- Screenshot.
- ScreenCast.
- RemoteDesktop.
- Background.
- GlobalShortcuts.
- Settings.

No se implementarán todas las interfaces automáticamente.

Cada interfaz se incorporará según las capacidades reales del escritorio y las necesidades de las aplicaciones.

## Seguridad

El backend debe:

- Respetar el consentimiento del usuario.
- Validar solicitudes.
- Aplicar permisos por aplicación.
- Utilizar identificadores de sesión.
- Evitar capturas silenciosas.
- Revocar sesiones cuando corresponda.
- Evitar acceso arbitrario al sistema de archivos.
- Integrarse con los mecanismos de autorización del compositor.

El compositor y el backend del portal deben tener responsabilidades claramente separadas.

El portal coordina solicitudes y permisos; el compositor controla las superficies y capacidades gráficas que le corresponden.

La integración de ScreenCast y RemoteDesktop requiere especial atención a la autorización, la duración de las sesiones y la revocación.

---

# 14. Ciclo de vida y recuperación

Cada servicio tendrá:

- Política de inicio.
- Política de reinicio.
- Timeout de inicio.
- Timeout de parada.
- Estado de salud.
- Política de watchdog cuando corresponda.
- Política de recuperación.
- Presupuesto de recursos.

## Clasificación de criticidad

| Servicio | Criticidad |
|---|---|
| settingsd | Alta |
| notifyd | Media |
| hardwared | Media |
| resourced | Media |
| updated | Bajo durante operación normal; alto durante actualización |
| recoveryd | Bajo durante operación normal; crítico durante recuperación |
| diagd | Baja |

La criticidad puede variar según el estado del sistema.

Un servicio de actualización no debe reiniciarse ciegamente mientras ejecuta una operación no idempotente.

Las operaciones deben diseñarse para detectar interrupciones y recuperar su estado.

---

# 15. Presupuestos de recursos

Cada servicio tendrá un presupuesto medible.

Los valores definitivos se establecerán durante Fase 1 mediante benchmarks.

Se medirán:

- PSS.
- RSS.
- CPU en reposo.
- CPU bajo carga.
- Descriptores abiertos.
- Threads.
- Frecuencia de wakeups.
- Latencia IPC.
- Crecimiento de memoria.
- Tiempo de arranque.

No se utilizarán límites arbitrarios que puedan provocar terminaciones en operaciones legítimas.

Los límites de memoria y CPU se definirán según el rol del servicio y sus escenarios de uso.

---

# 16. Pruebas obligatorias

## Seguridad

- [ ] Servicio sin privilegios.
- [ ] Acceso denegado a rutas no autorizadas.
- [ ] Acceso denegado a dispositivos no autorizados.
- [ ] Restricciones de syscall.
- [ ] Restricciones de familias de direcciones.
- [ ] Comportamiento ante D-Bus no autorizado.
- [ ] Validación de entradas.
- [ ] Recuperación ante fallo.

## Persistencia

- [ ] Escritura atómica.
- [ ] Interrupción durante escritura.
- [ ] Recuperación de configuración.
- [ ] Migración de esquema.
- [ ] Archivos inválidos.
- [ ] Permisos incorrectos.

## Operación

- [ ] Inicio.
- [ ] Parada.
- [ ] Reinicio.
- [ ] Timeout.
- [ ] Desconexión D-Bus.
- [ ] Watchdog.
- [ ] Reinicio de sesión.
- [ ] Presión de memoria.

## Actualización y recuperación

- [ ] Descarga interrumpida.
- [ ] Verificación de firma fallida.
- [ ] Aplicación interrumpida.
- [ ] Disco lleno.
- [ ] Snapshot inválido.
- [ ] Fallo de arranque.
- [ ] Restauración.
- [ ] Recuperación desde entorno externo.

---

# 17. Estructura de implementación

```text
services/
├── settingsd/
├── notifyd/
├── hardwared/
├── updated/
├── recoveryd/
└── diagd/

intelligence/
└── resourced/

portals/
└── xdg-desktop-portal-parallax/

systemd/
├── system/
├── user/
├── slices/
├── targets/
└── tmpfiles.d/
```

Los helpers privilegiados tendrán una ubicación y un contrato explícitos.

No se crearán binarios auxiliares con permisos especiales sin un ADR de seguridad.

---

# 18. Decisiones pendientes

1. ADR de endurecimiento y sandboxing de servicios.
2. ADR de separación entre descarga y aplicación de actualizaciones.
3. Definir el modelo de identidad y permisos de cada servicio.
4. Definir las unidades systemd de usuario y sistema.
5. Definir la estrategia de cgroups de resourced.
6. Definir el modelo de autorización de recoveryd.
7. Definir el helper privilegiado de diagd.
8. Definir el alcance de `xdg-desktop-portal-parallax`.
9. Definir los presupuestos de recursos.
10. Validar todas las directivas de endurecimiento mediante pruebas reales.

**Estado final: PROPUESTO — PENDIENTE DE VALIDACIÓN.**