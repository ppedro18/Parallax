# Arquitectura de hardware

- **Proyecto:** PARALLAX
- **Estado:** Propuesto — requiere implementación y validación
- **Ámbito:** Inventario, compatibilidad, GPU, energía, térmica y diagnóstico

# 1. Objetivo

Definir cómo PARALLAX detecta, identifica, clasifica, supervisa y diagnostica los dispositivos de hardware.

La arquitectura debe:

- Detectar dispositivos durante el arranque y en caliente.
- Construir un inventario coherente de hardware.
- Identificar controladores y firmware.
- Asociar perfiles de compatibilidad validados.
- Detectar problemas de hardware y configuración.
- Proporcionar información a Settings y Observatory.
- Facilitar optimizaciones de energía y rendimiento.
- Mantener la compatibilidad con el ecosistema Linux.
- Evitar duplicar funcionalidades de servicios existentes.

**Principio fundamental:** PARALLAX no reimplementa controladores, gestión de energía ni subsistemas del kernel. Los consume mediante interfaces estables.

---

# 2. Responsabilidades de hardwared

`parallax-hardwared` es el servicio responsable del inventario y la observación del hardware.

## 2.1. Responsabilidades

- Enumeración inicial de dispositivos.
- Monitorización de eventos udev.
- Lectura controlada de atributos sysfs.
- Identificación de fabricantes y modelos.
- Asociación de controladores.
- Detección de firmware requerido.
- Asociación de perfiles de hardware.
- Seguimiento del ciclo de vida de dispositivos.
- Exposición de información mediante `org.parallax.Hardware1`.

## 2.2. Responsabilidades excluidas

`hardwared` no debe:

- Administrar directamente frecuencias de CPU o GPU.
- Sustituir UPower.
- Sustituir NetworkManager.
- Sustituir PipeWire.
- Administrar directamente el consumo energético.
- Descargar firmware desde Internet.
- Cargar módulos del kernel arbitrariamente.
- Ejecutar comandos de shell proporcionados por perfiles.
- Aplicar parámetros del kernel sin una política explícita.
- Realizar pruebas SMART continuamente.

La administración se delega a los subsistemas correspondientes.

---

# 3. Detección e inventario

## 3.1. Fuentes de información

| Fuente | Responsabilidad |
|---|---|
| udev | Enumeración y eventos de dispositivos |
| sysfs | Atributos y relaciones del hardware |
| PCI sysfs | Dispositivos PCI y subsistemas |
| USB sysfs | Dispositivos USB |
| DMI | Fabricante y modelo del sistema |
| hwmon | Sensores térmicos y eléctricos |
| DRM sysfs | Dispositivos gráficos |
| Kernel journal | Diagnósticos del kernel |
| UPower | Estado de batería |
| systemd | Estado de servicios y unidades |

Los identificadores de fabricantes y productos PCI/USB se resolverán mediante bases locales empaquetadas.

No se realizarán consultas de red para identificar dispositivos.

## 3.2. Proceso de enumeración

Al iniciar:

1. Leer los dispositivos existentes mediante udev.
2. Obtener sus propiedades.
3. Resolver sus relaciones en sysfs.
4. Construir el inventario inicial.
5. Iniciar la monitorización de eventos.
6. Reconciliar eventos recibidos durante la enumeración.
7. Publicar el snapshot inicial.

La implementación debe contemplar altas, bajas y modificaciones concurrentes durante el arranque.

Los eventos duplicados o fuera de orden no deben producir entradas duplicadas en el inventario.

## 3.3. Modelo de dispositivo

Modelo conceptual:

```rust
struct Device {
    id: DeviceId,
    stable_id: Option<StableDeviceId>,
    parent_id: Option<DeviceId>,

    bus: Bus,
    class: DeviceClass,

    vendor_id: Option<u16>,
    product_id: Option<u16>,

    subsystem_vendor_id: Option<u16>,
    subsystem_product_id: Option<u16>,

    revision: Option<String>,
    model: Option<String>,
    manufacturer: Option<String>,

    sysfs_path: SysfsPath,
    driver: Option<DriverInfo>,
    firmware: Vec<FirmwareInfo>,

    profile: Option<ProfileMatch>,
    health: DeviceHealth,

    first_seen: Timestamp,
    last_seen: Timestamp,
}
```

Los tipos son conceptuales; la implementación definitiva debe utilizar tipos específicos y validación de datos.

### Identificadores

Se distinguen dos tipos:

**DeviceId**

Identifica una instancia dentro del inventario de PARALLAX. Puede derivarse de la ruta del dispositivo y su generación.

**StableDeviceId**

Identifica un dispositivo físico cuando existe un identificador fiable.

No todos los dispositivos disponen de una identidad persistente.

No se utilizarán exclusivamente los PCI ID o USB ID como identificadores únicos de dispositivos.

Los números de serie se tratarán como información potencialmente sensible y no se expondrán a consumidores que no los necesiten.

## 3.4. Ciclo de vida

Estados:

- `discovered`
- `initializing`
- `active`
- `degraded`
- `unavailable`
- `removed`

El cambio de estado debe publicarse mediante `Hardware1`.

Los consumidores deben poder obtener un snapshot actualizado después de desconectarse.

---

# 4. Perfiles de hardware

Los perfiles permiten definir compatibilidad, requisitos y excepciones conocidas.

Ubicación:

```text
distribution/hardware-profiles/
├── vendors/
│   ├── lenovo/
│   ├── asus/
│   ├── dell/
│   └── hp/
├── gpus/
│   ├── amd/
│   ├── intel/
│   └── nvidia/
└── devices/
    ├── wifi/
    ├── audio/
    └── bluetooth/
```

La estructura puede evolucionar según el número de perfiles.

## 4.1. Ejemplo de perfil

```toml
schema_version = 1
id = "lenovo-model-example"
version = "1.0.0"

[match]
dmi.sys_vendor = "LENOVO"
dmi.product_name = "21K..."
dmi.board_name = "..."

[match.pci]
vendor_id = "0x1002"
device_id = "0x..."

[gpu]
expected_driver = "amdgpu"
required_firmware_packages = ["linux-firmware-amdgpu"]

[quirks]
kernel_cmdline = []

[validation]
status = "community"
tested_release = "0.3.0"
tested_kernel = "..."
tested_hardware_revision = "..."
```

Los valores son ilustrativos y deben sustituirse por identificadores reales durante la validación.

## 4.2. Reglas de coincidencia

Los perfiles deben permitir:

- Coincidencia DMI.
- Coincidencia PCI.
- Coincidencia USB.
- Coincidencia de revisión.
- Coincidencia compuesta.
- Prioridad explícita.
- Exclusiones.
- Detección de perfiles ambiguos.

Una coincidencia DMI general no debe prevalecer sobre una coincidencia más específica sin una regla documentada.

Si dos perfiles tienen la misma prioridad y coinciden con el dispositivo, `hardwared` debe reportar ambigüedad y no seleccionar uno arbitrariamente.

## 4.3. Estados de validación

| Estado | Significado |
|---|---|
| `validated` | Probado por QA en hardware y release identificados |
| `community` | Probado por colaboradores externos con evidencia |
| `untested` | Perfil creado sin validación suficiente |
| `deprecated` | Perfil obsoleto o incompatible |
| `blocked` | Problemas conocidos que impiden recomendar su uso |

Cada perfil debe registrar:

- Release PARALLAX.
- Versión del kernel.
- Versión del controlador.
- Identificación del hardware.
- Firmware utilizado.
- Resultado de las pruebas.
- Fecha de validación.
- Responsable de validación.

`validated` no significa compatibilidad universal con todas las revisiones del mismo modelo.

## 4.4. Seguridad de los perfiles

Los perfiles son datos declarativos, no scripts.

No se permitirá:

- Ejecutar comandos arbitrarios.
- Ejecutar scripts de instalación.
- Modificar archivos del sistema directamente.
- Descargar recursos automáticamente.
- Instalar paquetes sin confirmación.
- Deshabilitar mecanismos de seguridad.

Los parámetros de kernel y las excepciones deben pasar por revisión técnica.

Las excepciones se aplicarán mediante un mecanismo controlado de configuración de distribución, con registro, justificación y posibilidad de reversión.

---

# 5. Arquitectura GPU

La compatibilidad gráfica depende de varios componentes:

1. Dispositivo físico.
2. Driver del kernel.
3. Firmware.
4. Driver de espacio de usuario.
5. Mesa o implementación propietaria.
6. API gráfica.
7. Backend DRM/GBM.
8. Compatibilidad con el compositor.

No se considerará compatible una GPU simplemente porque su módulo del kernel esté cargado.

## 5.1. Intel

| Componente | Tecnología |
|---|---|
| Kernel | i915 o xe según hardware y soporte |
| OpenGL | Mesa Iris o driver correspondiente |
| Vulkan | Mesa ANV |
| Gestión | DRM/KMS y mecanismos de energía del kernel |

La selección de i915 o xe debe basarse en la generación, el identificador del dispositivo y la compatibilidad de la versión de kernel elegida.

No se aplicará una regla universal basada únicamente en Gen9 o superior.

Intel será una plataforma prioritaria de validación.

## 5.2. AMD

| Componente | Tecnología |
|---|---|
| Kernel | amdgpu |
| OpenGL | Mesa RadeonSI |
| Vulkan | Mesa RADV |
| Gestión | DRM/KMS, hwmon y runtime power management |

AMD será una plataforma prioritaria de validación.

La compatibilidad exacta dependerá de la generación y del soporte de kernel, Mesa y firmware.

No se debe presentar GCN4+ como límite universal de RADV. Los requisitos de soporte deben documentarse por generación y release.

## 5.3. NVIDIA

NVIDIA tendrá soporte secundario durante v1.

La matriz debe distinguir:

- Módulos abiertos del kernel NVIDIA.
- Driver propietario de espacio de usuario.
- Nouveau.
- NVK.
- Nova.
- Compatibilidad DRM/GBM.
- GSP firmware.
- Compatibilidad con el compositor.

La selección de driver será por hardware y versión de distribución.

Para Turing y generaciones posteriores se evaluará prioritariamente la pila NVIDIA de módulos abiertos del kernel cuando esté soportada por la versión elegida.

Nouveau/NVK y Nova se evaluarán de forma independiente. No se declararán como alternativas equivalentes hasta demostrar compatibilidad funcional y estabilidad en el hardware de referencia.

No se asumirá que todas las GPU NVIDIA modernas tienen idéntico comportamiento en portátiles, especialmente en configuraciones híbridas.

## 5.4. GPU híbridas

Objetivo:

- Mantener la iGPU como dispositivo principal cuando sea apropiado.
- Utilizar la dGPU bajo demanda.
- Evitar mantener la dGPU activa sin necesidad.
- Permitir render offload.
- Mostrar información de consumo y estado.

Integración:

- PRIME render offload.
- `switcheroo-control`, cuando resulte adecuado.
- APIs de lanzamiento de aplicaciones.
- Políticas de energía existentes.

El launcher podrá ofrecer:

**Abrir con GPU dedicada**

La operación debe utilizar los mecanismos compatibles con el driver y la plataforma.

No debe asumir que cambiar variables de entorno garantiza que todas las aplicaciones utilicen la GPU dedicada.

El usuario debe poder comprobar qué GPU está utilizando una aplicación.

---

# 6. Energía y temperatura

## 6.1. Batería

UPower será la fuente de verdad para:

- Porcentaje.
- Estado de carga.
- Tiempo estimado.
- Estado de alimentación.
- Información de batería disponible.

PARALLAX consumirá sus interfaces D-Bus.

No implementará un segundo administrador de baterías.

## 6.2. Perfiles energéticos

Se utilizará `power-profiles-daemon` cuando esté disponible y sea compatible con el hardware.

PARALLAX no duplicará sus políticas.

Los perfiles podrán incluir:

- Rendimiento.
- Equilibrado.
- Ahorro energético.

Los nombres y las capacidades reales deben reflejar los perfiles disponibles en el sistema.

## 6.3. Temperaturas

Los sensores se obtendrán principalmente desde hwmon.

Cada lectura debe identificar:

- Dispositivo.
- Sensor.
- Etiqueta.
- Unidad.
- Valor.
- Fuente.
- Momento de lectura.
- Estado de validez.

No se debe asumir que todos los sensores representan la temperatura del mismo componente.

Las lecturas inválidas, ausentes o desactualizadas deben marcarse como tales.

## 6.4. Política adaptativa de resourced

`resourced` puede adaptar:

- Frecuencia de actualización de widgets.
- Intensidad de animaciones.
- Efectos visuales opcionales.
- Frecuencia de sondeo de métricas no críticas.
- Actividad de tareas en segundo plano.

La política debe considerar:

- Batería.
- Temperatura.
- Presión de memoria.
- Carga de CPU.
- Disponibilidad de GPU.
- Perfil energético activo.

Las adaptaciones deben ser reversibles y configurables.

No se debe deshabilitar automáticamente la composición, accesibilidad o funcionalidad esencial para ahorrar energía.

No se deben modificar frecuencias, voltajes ni límites térmicos directamente desde `resourced`.

---

# 7. Diagnóstico

`parallax-diagd` coordinará las comprobaciones y generará informes.

`hardwared` proporcionará inventario y estado de dispositivos.

## 7.1. Diagnósticos gráficos

Comprobaciones:

- Driver esperado frente al cargado.
- Dispositivo DRM disponible.
- Firmware requerido.
- Fallos de inicialización.
- Reinicios de GPU.
- Errores de DRM.
- Capacidades de renderizado.
- Compatibilidad GBM.
- Compatibilidad del compositor.
- Estado de aceleración gráfica.

No basta con detectar que el módulo del kernel está cargado.

## 7.2. Firmware

Se distinguirán:

| Estado | Significado |
|---|---|
| `required` | El dispositivo requiere firmware |
| `package_missing` | No está instalado el paquete esperado |
| `load_failed` | El kernel reportó un fallo de carga |
| `loaded` | Existe evidencia suficiente de carga |
| `unknown` | No puede determinarse el estado |

El nombre del paquete no demuestra por sí solo que el firmware se haya cargado correctamente.

Los diagnósticos utilizarán eventos del kernel, logs y datos del dispositivo.

No se analizará continuamente todo el journal.

## 7.3. Almacenamiento

Comprobaciones:

- Estado SMART.
- Estado NVMe.
- Errores de medio.
- Temperatura.
- Espacio disponible.
- Estado de dispositivos.
- Errores de E/S.

Las herramientas y consultas deben ejecutarse bajo demanda.

El acceso privilegiado estará limitado a operaciones concretas y documentadas.

No se ejecutarán pruebas destructivas ni pruebas de escritura intensivas automáticamente.

## 7.4. Temperatura y refrigeración

Comprobaciones:

- Sensores fuera de rango.
- Temperaturas sostenidas.
- Estado de ventiladores cuando exista una interfaz disponible.
- Eventos de thermal throttling.
- Problemas de alimentación.

Los umbrales serán específicos del dispositivo cuando existan datos fiables.

No se utilizará un único límite térmico global para todos los procesadores y GPU.

## 7.5. Informe de diagnóstico

Cada informe incluirá:

- Fecha.
- Release PARALLAX.
- Kernel.
- Dispositivo afectado.
- Driver.
- Firmware.
- Perfil asociado.
- Síntomas.
- Evidencias.
- Nivel de severidad.
- Recomendaciones.
- Acciones realizadas.

No incluirá números de serie ni datos sensibles por defecto.

---

# 8. API y eventos

El servicio expondrá:

```text
org.parallax.Hardware1
```

Operaciones conceptuales:

```text
GetInventory()
GetDevice(device_id)
GetDeviceHealth(device_id)
GetMatchedProfile(device_id)
GetHardwareSummary()
RequestDiagnostic(device_id, diagnostic_type)
```

Eventos:

```text
DeviceAdded
DeviceRemoved
DeviceChanged
DeviceHealthChanged
ProfileMatched
ProfileValidationChanged
```

Se aplicará el modelo común de eventos PARALLAX:

- Snapshot + delta.
- Revisión monotónica.
- Identidad de generación.
- Autorización.
- Límites de frecuencia.
- Resincronización.
- Observabilidad.

Los eventos de sensores no deben utilizarse para transmitir muestras de alta frecuencia sin necesidad.

Los consumidores podrán solicitar consultas puntuales o suscribirse a métricas agregadas.

---

# 9. Seguridad

`hardwared` se ejecutará con un usuario de sistema dedicado.

Principios:

- Sin privilegios root permanentes.
- Lectura de sysfs limitada a las rutas necesarias.
- Acceso udev controlado.
- Sin ejecución de comandos arbitrarios.
- Sin escritura directa sobre parámetros sensibles del kernel.
- D-Bus con políticas de acceso.
- Validación de entradas.
- Límites de memoria y CPU.
- Reinicio supervisado por systemd.

Las operaciones privilegiadas de diagnóstico se separarán cuando sea necesario en componentes pequeños y endurecidos.

No se concederán permisos generales de acceso a todos los dispositivos por comodidad.

---

# 10. Estructura de implementación

```text
services/
├── hardwared/
│   ├── src/
│   │   ├── discovery/
│   │   ├── inventory/
│   │   ├── identity/
│   │   ├── profiles/
│   │   ├── drivers/
│   │   ├── firmware/
│   │   ├── events/
│   │   └── dbus/
│   └── Cargo.toml
│
├── diagd/
│   ├── src/
│   │   ├── gpu/
│   │   ├── storage/
│   │   ├── thermal/
│   │   ├── firmware/
│   │   ├── reports/
│   │   └── dbus/
│   └── Cargo.toml
│
intelligence/
└── resourced/
    ├── src/
    │   ├── cpu/
    │   ├── memory/
    │   ├── gpu/
    │   ├── energy/
    │   ├── thermal/
    │   └── policies/
    └── Cargo.toml

distribution/
└── hardware-profiles/
    ├── vendors/
    ├── gpus/
    ├── devices/
    └── schema/
```

La separación entre `hardwared`, `diagd` y `resourced` evita mezclar inventario, diagnóstico y políticas de rendimiento.

---

# 11. Pruebas y validación

## 11.1. Inventario

- [ ] Enumeración inicial.
- [ ] Alta de dispositivo en caliente.
- [ ] Retirada de dispositivo.
- [ ] Eventos duplicados.
- [ ] Eventos fuera de orden.
- [ ] Reconexión de hardware.
- [ ] Dispositivos sin identificadores persistentes.
- [ ] Dispositivos sin driver.
- [ ] Dispositivos con atributos sysfs incompletos.

## 11.2. Perfiles

- [ ] Coincidencia DMI.
- [ ] Coincidencia PCI.
- [ ] Coincidencia compuesta.
- [ ] Perfil ambiguo.
- [ ] Perfil no validado.
- [ ] Perfil obsoleto.
- [ ] Parámetro de kernel no autorizado.
- [ ] Perfil malformado.

## 11.3. GPU

- [ ] Intel iGPU.
- [ ] AMD iGPU.
- [ ] AMD dGPU.
- [ ] NVIDIA dGPU.
- [ ] Configuración híbrida.
- [ ] Render offload.
- [ ] Monitor externo.
- [ ] Suspensión y reanudación.
- [ ] Reinicio del driver.
- [ ] Recuperación ante fallo gráfico.

## 11.4. Energía y temperatura

- [ ] Batería.
- [ ] Alimentación AC.
- [ ] Cambio de perfil energético.
- [ ] Sensores ausentes.
- [ ] Sensores inválidos.
- [ ] Temperatura elevada.
- [ ] Thermal throttling.
- [ ] Adaptación reversible de efectos.

## 11.5. Diagnóstico

- [ ] Firmware ausente.
- [ ] Driver incorrecto.
- [ ] Error de inicialización.
- [ ] SMART.
- [ ] NVMe.
- [ ] Error de GPU.
- [ ] Dispositivo desconectado.
- [ ] Permisos insuficientes.
- [ ] Informe sin información sensible.

---

# 12. Criterios de aceptación

La arquitectura se considerará implementada cuando:

1. El inventario sea reproducible y coherente.
2. Los eventos udev no provoquen duplicaciones.
3. Los identificadores distingan instancia e identidad persistente.
4. Los perfiles tengan esquema validado.
5. Los perfiles no ejecuten código arbitrario.
6. Los drivers se identifiquen correctamente.
7. Los fallos de firmware sean detectables.
8. Las métricas de temperatura indiquen su fuente.
9. La integración con UPower y power-profiles-daemon funcione.
10. Los diagnósticos privilegiados estén aislados.
11. Los eventos permitan resincronización.
12. El servicio cumpla sus presupuestos de memoria y CPU.
13. La compatibilidad se valide en hardware real.

---

# 13. Decisiones pendientes

- ADR para selección y compatibilidad de GPU NVIDIA.
- ADR para políticas de perfiles de hardware.
- Definición de la API `org.parallax.Hardware1`.
- Definición de los permisos de hardwared.
- Esquema definitivo de hardware profiles.
- Política de compatibilidad de kernel y Mesa.
- Matriz de hardware de referencia.
- Presupuestos de recursos para hardwared y diagd.
- Política de diagnósticos y privacidad.

**Estado final: PROPUESTO — PENDIENTE DE VALIDACIÓN.**

La implementación debe comenzar por inventario, identificación y diagnóstico de solo lectura. La administración avanzada de hardware y las optimizaciones automáticas se incorporarán únicamente después de demostrar compatibilidad y seguridad.