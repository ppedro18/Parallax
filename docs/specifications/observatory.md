# Especificación: PARALLAX Observatory y Task Manager

**Componentes:** `parallax-task-manager`, `parallax-observatory`  
**Estado:** Propuesto · Pendiente de aprobación  
**Capa:** C6 — Native Apps  
**Dependencias:** `parallax-resourced`, `hardwared`, `diagd`, systemd, journald, UPower, UDisks2.

## 1. Objetivo

Proporcionar dos herramientas complementarias:

- Task Manager: supervisión y acciones rápidas sobre aplicaciones y procesos.
- Observatory: diagnóstico profundo del sistema, hardware, servicios, recursos y eventos.

Ambas aplicaciones consumen fuentes de datos existentes. No deben convertirse en propietarias de métricas que pertenecen a otros servicios.

## 2. Task Manager

### 2.1 Activación

Atajo:

`Ctrl+Shift+Esc`

Debe ser configurable y respetar la política de atajos del compositor.

### 2.2 Vista principal

Agrupar aplicaciones por cgroup y `app_id` cuando exista información fiable.

Mostrar:

- Aplicaciones.
- Procesos.
- CPU.
- Memoria PSS.
- Memoria adicional disponible.
- GPU.
- E/S de disco.
- Red.
- Estado.
- Tiempo de ejecución.

No presentar datos inexistentes como cero.

Si el driver no expone uso de GPU por proceso, indicar "No disponible".

### 2.3 Acciones

#### Finalizar

Secuencia:

1. Solicitar SIGTERM.
2. Esperar cinco segundos como valor inicial configurable.
3. Comprobar si el proceso terminó.
4. Ofrecer SIGKILL si continúa activo.

No ejecutar SIGKILL automáticamente sin una política explícita.

#### Suspender y reanudar

Utilizar cgroup v2 cuando exista autorización y delegación adecuada.

La operación debe aplicarse al grupo de procesos seleccionado, no a procesos arbitrarios por PID sin validar su identidad.

#### Prioridad

Modificar prioridad solo cuando esté permitido.

Distinguir:

- Nice.
- I/O priority.
- CPU weight del cgroup.

No ofrecer opciones que requieran privilegios sin informar de ello.

### 2.4 Otros usuarios

Los procesos de otros usuarios se muestran en modo de solo lectura salvo autorización específica.

No exponer argumentos, entorno o datos sensibles de otros usuarios sin necesidad.

## 3. Observatory

### 3.1 Vistas

| Vista | Fuente principal |
|---|---|
| CPU | `/proc/stat`, cpufreq, PSI |
| Memoria | `/proc/meminfo`, PSS, cgroupfs |
| GPU | DRM fdinfo y herramientas disponibles |
| Procesos | resourced y cgroupfs |
| Servicios | systemd D-Bus |
| Temperatura | hwmon mediante hardwared |
| Ventiladores | hwmon mediante hardwared |
| Almacenamiento | UDisks2 y hardwared |
| SMART | hardwared / herramientas autorizadas |
| Batería | UPower |
| Eventos | journald |
| Drivers | hardwared |
| Firmware | hardwared |
| Diagnóstico | diagd |

### 3.2 CPU

Mostrar:

- Uso global.
- Uso por núcleo.
- Frecuencia.
- Estado de frecuencia.
- PSI CPU.
- Histórico.

La frecuencia debe identificarse como dato reportado por el driver, no necesariamente como frecuencia física instantánea exacta.

### 3.3 Memoria

Mostrar por separado:

- MemTotal.
- MemAvailable.
- Uso estimado.
- Caché.
- SReclaimable.
- SUnreclaim.
- Swap.
- Zram.
- PSI memory.
- Top consumidores por PSS.

No restar nuevamente la caché reclamable de `MemTotal - MemAvailable`: `MemAvailable` ya considera parte de la memoria recuperable.

PSS es una métrica de atribución compartida, no una medición del consumo total de RAM del sistema.

### 3.4 GPU

Datos desde DRM fdinfo cuando el driver los proporciona:

- Memoria GPU.
- Uso por engine.
- Uso por proceso.
- Información disponible por dispositivo.

La cobertura depende del driver y kernel.

Mostrar capacidades no soportadas como no disponibles.

No extrapolar porcentajes de uso GPU cuando el kernel no los expone.

### 3.5 Temperaturas

Fuente: hwmon a través de hardwared.

Cada sensor debe identificar:

- Nombre.
- Fuente.
- Unidad.
- Timestamp.
- Validez.
- Dispositivo.

No asumir que todos los sensores representan la temperatura del CPU o GPU.

### 3.6 Almacenamiento

Mostrar:

- Capacidad.
- Espacio libre.
- E/S.
- Filesystem.
- Estado del dispositivo.
- SMART cuando esté soportado.

Los tests SMART intensivos no se ejecutan automáticamente.

### 3.7 Batería

UPower proporciona:

- Estado.
- Porcentaje.
- Salud cuando esté disponible.
- Ciclos cuando el dispositivo los exponga.
- Tiempo estimado.
- Consumo estimado.

No todos los dispositivos proporcionan todas estas métricas.

### 3.8 Servicios

Integración con systemd D-Bus.

Acciones:

- Ver estado.
- Ver fallos.
- Consultar logs.
- Reiniciar servicios de usuario autorizados.

Las operaciones sobre servicios de sistema requieren política de autorización apropiada.

### 3.9 Eventos

Integración con journald.

Filtros:

- Unidad.
- Prioridad.
- Intervalo temporal.
- Texto.
- Componente.

Redactar o restringir datos sensibles según la política del sistema.

## 4. Histórico

El histórico pertenece a `parallax-resourced`.

La interfaz no mantiene una copia independiente.

### 4.1 Política

- Histórico de alta resolución bajo demanda.
- Histórico de baja resolución para tendencias.
- Retención de 24 horas como objetivo inicial.
- Muestreo de 1 Hz durante suscripción activa.
- Muestreo de 0,1 Hz para histórico de baja resolución.

### 4.2 Transporte

Comenzar con snapshot y señales D-Bus de baja frecuencia.

Un ring buffer compartido mediante memfd es una optimización opcional, no una obligación inicial.

Solo implementarlo si el benchmark demuestra que D-Bus es insuficiente.

Si se utiliza memfd, definir:

- Layout.
- Versión.
- Sincronización.
- Límites.
- Permisos.
- Ciclo de vida.
- Recuperación.
- Validación de offsets.

El presupuesto aproximado de 20 MB debe validarse con el conjunto real de métricas y retención.

## 5. Seguridad

- Solo lectura por defecto.
- Acciones privilegiadas mediante servicios autorizados.
- No ejecutar comandos arbitrarios.
- No permitir finalización de procesos del sistema sin autorización.
- Redactar información sensible.
- Limitar exportación de informes.
- No enviar diagnósticos a la red automáticamente.

## 6. Organización

```text
apps/
├── parallax-task-manager/
└── parallax-observatory/

crates/
└── px-observability-client/
```

Las métricas se obtienen mediante contratos tipados de los servicios propietarios.

## 7. Pruebas

- [ ] CPU por núcleo.
- [ ] Memoria y PSS.
- [ ] PSI.
- [ ] Zram.
- [ ] GPU con drivers soportados.
- [ ] Datos no disponibles.
- [ ] Temperaturas.
- [ ] Servicios systemd.
- [ ] Logs.
- [ ] Procesos de otros usuarios.
- [ ] SIGTERM y SIGKILL.
- [ ] Congelación y reanudación autorizadas.
- [ ] Histórico y retención.
- [ ] Recuperación de conexión.
- [ ] Carga de UI y rendimiento.

## 8. Criterios de aceptación

- No duplicar fuentes de verdad.
- Mostrar correctamente métricas no disponibles.
- No ejecutar operaciones privilegiadas sin autorización.
- Actualizar datos sin bloquear UI.
- Recuperar snapshots tras desconexión.
- Cumplir límites de memoria y rendimiento definidos por la metodología.

## 9. Decisiones pendientes

1. Definir contrato final de métricas.
2. Definir esquema de histórico.
3. Definir si memfd es necesario.
4. Definir política de acceso a procesos ajenos.
5. Definir exportación de diagnósticos.
6. Definir métricas GPU soportadas por HW-A y HW-B.

**Estado: Propuesto. Pendiente de aprobación.**