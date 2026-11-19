# ADR-0009: Actualizaciones transaccionales y recuperación del sistema

- **Estado:** Propuesto
- **Fecha:** 2026-11-19
- **Ámbito:** Sistema base, paquetes, kernel, initramfs, configuración y recuperación del arranque.
- **Documentación complementaria:** `architecture/actualizacion-recuperacion.md`
- **Relacionado con:** ADR-0002 (Base Arch), ADR-0008 (Licencias).

## 1. Contexto

PARALLAX utilizará una base de paquetes derivada de Arch Linux, administrada mediante pacman y repositorios firmados.

El modelo rolling tradicional implica que una actualización puede introducir incompatibilidades entre paquetes, kernel, bibliotecas, controladores o componentes del entorno.

PARALLAX necesita controlar cuándo se incorporan actualizaciones y disponer de un mecanismo de recuperación ante:

- Actualizaciones defectuosas.
- Incompatibilidades entre paquetes.
- Errores durante una transacción de pacman.
- Fallos del kernel o initramfs.
- Problemas de configuración.
- Interrupciones de energía durante una actualización.
- Fallos durante el primer arranque después de actualizar.

La recuperación deberá permitir volver a un estado anterior conocido sin depender necesariamente de un medio de instalación externo.

El sistema distinguirá entre actualización, recuperación del sistema y restauración de datos personales.

## 2. Opciones evaluadas

| Alternativa | Ventajas | Consideraciones |
|---|---|---|
| pacman + Btrfs + boot counting | Compatible con Arch, recuperación mediante snapshots | Requiere diseñar correctamente subvolúmenes y arranque |
| Imágenes A/B inmutables | Actualización por generaciones y rollback controlado | Mayor complejidad de construcción y distribución |
| OSTree | Versionado de despliegues y actualización basada en árboles | Mayor adaptación del ecosistema Arch |

## 3. Decisión

Se adopta una estrategia evolutiva.

### Versión 1

- pacman como gestor de paquetes.
- Repositorios firmados de PARALLAX.
- Btrfs obligatorio como sistema de archivos de la raíz.
- Snapshots previos y posteriores a las transacciones.
- systemd-boot como gestor de arranque.
- Boot counting para nuevas entradas.
- Validación del arranque satisfactorio.
- Rollback automático cuando se agoten los intentos permitidos.
- Conservación de una generación anterior conocida como funcional.

### Versión 2

Se evaluará una arquitectura de imágenes A/B inmutables, utilizando tecnologías como `mkosi` y `systemd-sysupdate`.

OSTree permanecerá como alternativa si los resultados justifican su adopción.

La evolución no deberá obligar a cambiar el modelo de paquetes de manera prematura.

## 4. Sistema de archivos obligatorio

La instalación oficial de PARALLAX utilizará Btrfs como sistema de archivos de la raíz.

No se soportará ext4 como raíz en la primera versión.

La decisión se justifica por la necesidad de crear snapshots y restaurar generaciones del sistema.

### 4.1 Distribución de subvolúmenes

Se utilizará inicialmente una estructura similar a:

```text
BTRFS
│
├── @
│   └── /
│
├── @home
│   └── /home
│
├── @snapshots
│   └── /snapshots
│
├── @var_log
│   └── /var/log       (evaluar)
│
└── @var_cache
    └── /var/cache     (evaluar)

EFI System Partition
└── /efi
```

La estructura definitiva deberá definirse en el instalador y documentarse en `architecture/storage.md`.

### 4.2 Reglas

- `@` contendrá el sistema base y la base de datos de paquetes.
- `/var/lib/pacman` deberá permanecer dentro del subvolumen raíz.
- `/home` será independiente de los snapshots del sistema.
- `/snapshots` almacenará las generaciones de recuperación.
- La partición EFI tendrá una estrategia independiente de respaldo.
- Los subvolúmenes adicionales deberán tener una política explícita de snapshot y recuperación.

No se separará todo `/var` automáticamente.

Separar `/var` completo puede dejar fuera del rollback archivos fundamentales para la consistencia del sistema, incluyendo el estado de pacman.

Los datos persistentes de servicios y aplicaciones deberán evaluarse individualmente.

### 4.3 Limitación de snapshots anidados

Un snapshot de Btrfs no incorpora automáticamente los subvolúmenes anidados.

Por tanto, el instalador y el gestor de actualizaciones deberán conocer la estructura real del sistema de archivos.

No se podrá asumir que un snapshot de `@` contiene todos los datos montados debajo de `/`.

## 5. Arquitectura de actualización V1

La actualización será administrada mediante un servicio propio de PARALLAX.

Nombre conceptual:

`px-update`

Sus responsabilidades serán:

- Consultar actualizaciones disponibles.
- Verificar repositorios y firmas.
- Resolver dependencias.
- Comprobar espacio disponible.
- Crear snapshots.
- Ejecutar transacciones de pacman.
- Gestionar kernels e initramfs.
- Generar entradas de arranque.
- Supervisar el estado de actualización.
- Registrar resultados.
- Preparar rollback cuando sea necesario.

Pacman continuará siendo responsable de instalar y retirar paquetes.

`px-update` no implementará un gestor de paquetes alternativo.

## 6. Ciclo de actualización

Toda actualización del sistema seguirá una secuencia controlada.

### Fase 1 — Preparación

Antes de modificar el sistema:

1. Verificar la disponibilidad del repositorio.
2. Validar firmas y metadatos.
3. Resolver dependencias.
4. Comprobar espacio libre.
5. Verificar la existencia de una generación anterior recuperable.
6. Comprobar que no existe otra transacción activa.
7. Registrar el identificador de actualización.

Si falla una validación crítica, la actualización no comenzará.

### Fase 2 — Snapshot previo

Crear un snapshot del subvolumen raíz antes de ejecutar pacman.

Este snapshot representará el último estado conocido anterior a la actualización.

Metadatos asociados:

- Identificador de actualización.
- Fecha.
- Snapshot de origen.
- Versión anterior del sistema.
- Versión objetivo.
- Motivo de actualización.
- Estado de la transacción.

El snapshot previo deberá conservarse hasta que la nueva generación haya sido validada.

### Fase 3 — Transacción de paquetes

Ejecutar pacman utilizando exclusivamente repositorios autorizados.

Reglas:

- No permitir transacciones concurrentes.
- Registrar la salida y los errores.
- No interrumpir deliberadamente pacman.
- No modificar manualmente la base de datos de paquetes.
- Mantener el snapshot previo durante toda la operación.

Las actualizaciones del sistema no se considerarán completadas hasta que se hayan validado también los artefactos de arranque.

### Fase 4 — Preparación del arranque

Después de instalar los paquetes:

1. Generar o actualizar initramfs.
2. Instalar o actualizar el kernel.
3. Verificar los artefactos generados.
4. Actualizar las entradas de systemd-boot.
5. Mantener disponible la entrada anterior.
6. Preparar el contador de intentos de arranque.
7. Registrar la nueva generación como candidata.

La partición EFI no está protegida por el snapshot de Btrfs.

Por tanto, los kernels, initramfs, UKI y entradas de arranque deberán disponer de una estrategia propia de conservación y recuperación.

Se recomienda evaluar Unified Kernel Images (UKI) para simplificar la relación entre kernel, initramfs y parámetros de arranque.

No se eliminará una entrada anterior funcional hasta validar la nueva generación.

### Fase 5 — Reinicio y validación

La nueva generación se iniciará como candidata.

Se utilizará boot counting de systemd-boot.

El número de intentos deberá configurarse explícitamente y ser pequeño, evitando ciclos de arranque indefinidos.

Después de iniciar el sistema se ejecutarán comprobaciones de salud.

Entre ellas:

- Disponibilidad de systemd.
- Estado del sistema de archivos.
- Disponibilidad de servicios críticos.
- Inicialización del compositor.
- Disponibilidad del entorno gráfico.
- Ausencia de fallos críticos definidos por PARALLAX.

La validación deberá ejecutarse mediante unidades y dependencias de systemd.

El arranque solo se considerará satisfactorio cuando se alcance el punto de finalización configurado y se confirme la salud mínima del sistema.

La integración deberá evaluarse mediante:

- `boot-complete.target`.
- `systemd-bless-boot`.
- `systemd-boot-check-no-failures`, cuando corresponda.

No se marcará una generación como buena simplemente porque el kernel haya comenzado a ejecutarse.

### Fase 6 — Confirmación

Si las comprobaciones terminan correctamente:

- Marcar la entrada como funcional.
- Confirmar la nueva generación.
- Registrar el resultado.
- Conservar los snapshots según la política de retención.
- Mantener al menos una generación anterior recuperable.

La confirmación de arranque no implica que los datos personales o las bases de datos de aplicaciones hayan sido verificados.

## 7. Rollback automático

Si la nueva generación agota sus intentos de arranque sin completar la validación, systemd-boot deberá seleccionar la entrada anterior disponible.

La generación anterior deberá apuntar a un snapshot recuperable.

El mecanismo de rollback deberá asegurar que el sistema arranque utilizando una vista coherente de:

- Sistema de archivos.
- Base de datos de pacman.
- Configuración.
- Kernel.
- Initramfs.
- Parámetros de arranque.

No bastará con seleccionar un snapshot desde el menú de arranque.

El procedimiento deberá preparar un subvolumen de recuperación arrancable y mantener correctamente sus referencias.

Se evaluará el uso de una operación equivalente a `snapper rollback` o de un mecanismo propio que cree un nuevo subvolumen de trabajo a partir del snapshot seleccionado.

La implementación deberá distinguir entre:

- Snapshot de solo lectura.
- Subvolumen de recuperación.
- Subvolumen raíz activo.

No se deberá intentar ejecutar un sistema completo de escritura sobre un snapshot de solo lectura sin un mecanismo explícito que proporcione una raíz de escritura.

## 8. Recuperación manual

El usuario deberá disponer de una opción de recuperación desde el menú de arranque.

Esta opción permitirá seleccionar una generación anterior.

La interfaz de recuperación deberá mostrar:

- Fecha.
- Identificador.
- Versión del sistema.
- Motivo del snapshot.
- Resultado de validación.
- Espacio ocupado.

Las operaciones de recuperación deberán ser explícitas y registrar sus resultados.

## 9. Tratamiento de `/etc` y `/var`

### 9.1 `/etc`

`/etc` formará parte del snapshot del sistema.

Se deberá tener en cuenta que los paquetes pueden introducir archivos nuevos, modificar configuraciones y ejecutar scripts de instalación.

Las configuraciones modificadas por el usuario deberán tener mecanismos de comparación y recuperación.

No se sobrescribirán silenciosamente configuraciones personales al restaurar una generación.

### 9.2 `/var`

El tratamiento de `/var` será una decisión central de la arquitectura de almacenamiento.

Para V1:

- La base de datos de pacman permanecerá en la raíz.
- Los datos de estado esenciales para el sistema deberán mantener coherencia con la generación.
- Los directorios de caché podrán separarse cuando exista una justificación.
- Los datos de aplicaciones deberán evaluarse individualmente.
- Los servicios con bases de datos deberán tener mecanismos de recuperación propios.

No se garantizará la reversión automática de todas las bases de datos de aplicaciones mediante snapshots del sistema.

Las migraciones de esquemas deberán ser compatibles con la política de rollback o disponer de una estrategia específica.

## 10. Protección de la partición EFI

La partición EFI no forma parte del sistema de snapshots de Btrfs cuando se encuentra en un sistema de archivos independiente.

Se deberá implementar una estrategia de actualización segura de artefactos de arranque.

Requisitos:

- Conservar al menos una entrada funcional anterior.
- No sobrescribir el único kernel arrancable.
- Validar los archivos antes de modificar las entradas activas.
- Mantener correspondencia entre la generación y sus artefactos.
- Evitar referencias a kernels eliminados.
- Mantener espacio suficiente en la partición EFI.

Los cambios de kernel y bootloader deberán probarse con interrupciones simuladas.

## 11. Retención y limpieza

Los snapshots deberán tener una política de retención.

Se distinguirán:

- Snapshots anteriores a actualizaciones.
- Snapshots posteriores a actualizaciones.
- Generaciones confirmadas.
- Generaciones de recuperación.
- Snapshots manuales.

La limpieza no podrá eliminar:

- La generación activa.
- La última generación funcional.
- Una generación pendiente de validación.
- Artefactos necesarios para arrancar una generación conservada.

Se deberán establecer límites por cantidad y espacio disponible.

La limpieza deberá ser conservadora cuando el almacenamiento esté cerca de su capacidad.

Los snapshots no sustituyen una copia de seguridad externa.

## 12. Fallos e interrupciones

La implementación deberá contemplar:

| Fallo | Comportamiento esperado |
|---|---|
| Error al descargar paquetes | No iniciar transacción |
| Firma inválida | Rechazar actualización |
| Espacio insuficiente | Cancelar antes de modificar |
| Error durante pacman | Conservar snapshot previo y entrar en recuperación |
| Corte eléctrico durante pacman | Permitir arrancar la generación anterior |
| Error al generar initramfs | No promover la generación |
| Kernel defectuoso | Utilizar entrada anterior |
| Fallo de servicios críticos | No confirmar arranque |
| Fallo del Shell | No confundirlo automáticamente con un fallo del kernel |
| Snapshot dañado | Utilizar otra generación validada o recuperación externa |

La recuperación no deberá depender de que el sistema actualizado pueda ejecutar herramientas de recuperación.

Cuando sea necesario, deberá existir un mecanismo desde initramfs o desde un entorno de rescate.

## 13. Integridad y seguridad

- Todos los repositorios de PARALLAX deberán estar firmados.
- Los paquetes deberán verificarse antes de instalarse.
- Las operaciones de actualización requerirán privilegios controlados.
- El estado de cada actualización deberá registrarse.
- Los snapshots no se considerarán copias de seguridad inmutables frente a un atacante con privilegios de administrador.
- La recuperación deberá verificar que los artefactos de arranque corresponden a la generación seleccionada.
- Los procesos de actualización no deberán aceptar paquetes de repositorios no autorizados.

## 14. Evaluación de V2: imágenes A/B inmutables

Al finalizar la Fase 6 se evaluará la migración hacia un modelo de imágenes A/B.

La propuesta consistirá en mantener dos generaciones independientes:

- Slot A: sistema activo.
- Slot B: sistema candidato o de recuperación.

El actualizador preparará la generación inactiva, validará sus artefactos y configurará el siguiente arranque.

La generación anterior permanecerá disponible hasta confirmar el arranque de la nueva.

Se evaluarán:

- `mkosi`.
- `systemd-sysupdate`.
- Unified Kernel Images.
- Gestión de estado persistente.
- Migraciones de configuración.
- Espacio necesario para dos generaciones.
- Actualizaciones interrumpidas.
- Recuperación automática.
- Compatibilidad con el ecosistema Arch.

OSTree seguirá siendo una alternativa de evaluación, no una decisión de implementación.

La migración a A/B solo se aprobará si aporta mejoras demostrables de seguridad, consistencia, mantenimiento o experiencia de actualización.

## 15. Benchmark y criterios de aceptación

Antes de considerar V1 estable deberán superarse pruebas reproducibles.

### Escenarios

1. Actualización normal.
2. Actualización de kernel.
3. Actualización de compositor y Shell.
4. Interrupción eléctrica durante una transacción.
5. Fallo al generar initramfs.
6. Entrada de arranque defectuosa.
7. Fallo de comprobación de salud.
8. Rollback automático.
9. Rollback manual.
10. Restauración de configuración.
11. Limpieza de snapshots.
12. Recuperación con poco espacio libre.

### Criterios de aceptación

- La instalación oficial exige Btrfs en la raíz.
- Cada transacción tiene un snapshot previo recuperable.
- La generación anterior permanece arrancable.
- La actualización no se confirma antes de superar la comprobación de salud.
- El sistema puede volver a una generación anterior.
- El compositor no depende del éxito de la actualización del Shell para mantener la integridad del sistema.
- Los cambios de kernel y EFI tienen protección independiente.
- La base de datos de pacman permanece coherente con la generación activa.
- La política de `/etc`, `/var` y `/home` está documentada.
- Los escenarios de interrupción y rollback se prueban automáticamente.

## 16. Consecuencias

### Positivas

- Compatibilidad con el ecosistema Arch.
- Recuperación ante actualizaciones defectuosas.
- Control de versiones del sistema instalado.
- Menor dependencia de medios externos para recuperar fallos habituales.
- Conservación de una generación funcional.
- Base para evolucionar hacia actualizaciones A/B.

### Negativas

- Btrfs obligatorio como raíz.
- Mayor complejidad del instalador.
- Gestión adicional de snapshots y espacio.
- Necesidad de coordinar Btrfs, pacman, initramfs y systemd-boot.
- La recuperación de datos persistentes requiere políticas independientes.
- Los snapshots no sustituyen copias de seguridad.

## 17. Revisión

Este ADR se revisará al finalizar la Fase 6 utilizando datos de:

- Fallos reales de actualización.
- Tiempo de recuperación.
- Consumo de almacenamiento.
- Incidencias de arranque.
- Complejidad de mantenimiento.
- Fiabilidad del rollback.
- Coste de mantener el modelo basado en pacman.

## 18. Principio rector

**Una actualización no se considera exitosa cuando termina de instalar paquetes; se considera exitosa cuando PARALLAX demuestra que puede arrancar y funcionar correctamente con la nueva generación.**