# Sistema de actualización y recuperación

- **Proyecto:** PARALLAX
- **Estado:** Propuesto — requiere implementación y validación
- **ADR relacionado:** ADR-0009
- **Versión:** 1.0
- **Fecha:** 2026-11-19

## 1. Objetivo

Definir un sistema de actualización, validación de arranque y recuperación que permita mantener PARALLAX actualizado sin comprometer la integridad del sistema ni los datos del usuario.

El sistema debe cumplir los siguientes principios:

1. Ninguna actualización debe considerarse exitosa antes de verificar su integridad.
2. Una actualización interrumpida no debe dejar el sistema sin una ruta de recuperación.
3. El usuario conserva el control sobre cuándo instalar actualizaciones y reiniciar.
4. El rollback del sistema no debe modificar los datos personales.
5. Los componentes críticos deben poder recuperarse independientemente.
6. Las operaciones de recuperación deben ser auditables y verificables.
7. La recuperación debe funcionar incluso cuando la interfaz gráfica no esté disponible.

**Regla fundamental:** la disponibilidad de un snapshot no demuestra que el sistema pueda recuperarse. Cada mecanismo debe probarse mediante escenarios de fallo reales.

---

# 2. Arquitectura general

PARALLAX utilizará inicialmente:

- `pacman` como gestor de paquetes.
- Repositorios oficiales de PARALLAX firmados.
- Snapshots Btrfs para recuperación.
- `systemd-boot` para gestionar las entradas de arranque.
- Boot counting para detectar arranques fallidos.
- `systemd-bless-boot` para confirmar arranques satisfactorios.
- `updated` como coordinador de actualizaciones.
- `recoveryd` como coordinador de recuperación.
- `settingsd` para gestionar configuración del sistema y del usuario.

## 2.1. Canales de distribución

| Canal | Propósito | Política |
|---|---|---|
| `stable` | Usuarios finales | Versiones validadas y actualizaciones controladas |
| `beta` | Pruebas públicas | Versiones candidatas con cambios recientes |
| `dev` | Desarrollo | Compilaciones experimentales |

Cada canal debe utilizar metadatos firmados y una referencia inequívoca a su snapshot de repositorio.

Los metadatos de una actualización deben incluir:

- Identificador de versión.
- Canal de distribución.
- Versión mínima compatible.
- Arquitectura.
- Identificadores y versiones de paquetes.
- Hashes criptográficos.
- Información de dependencias.
- Fecha de publicación.
- Firma digital.
- Información de compatibilidad y migraciones.

El cambio de canal debe ser explícito y advertir al usuario cuando implique instalar versiones menos estables.

---

# 3. Arquitectura de almacenamiento

## 3.1. Sistema de archivos

Btrfs será el sistema de archivos obligatorio para la raíz en la versión inicial de PARALLAX.

La distribución de subvolúmenes debe definirse antes de implementar `updated`.

Propuesta inicial:

| Subvolumen | Contenido | Comportamiento durante rollback |
|---|---|---|
| `@` | Sistema operativo | Se revierte |
| `@home` | Datos personales | Se conserva |
| `@var-log` | Registros | Se conserva |
| `@var-cache` | Cachés | Se conserva o limpia |
| `@snapshots` | Snapshots del sistema | Se conserva |
| `@state` | Estado persistente seleccionado | Se conserva con migraciones controladas |

La ubicación definitiva de `/var`, `/etc`, `/var/lib` y los datos de servicios debe documentarse en `architecture/storage.md`.

**Importante:** no separar indiscriminadamente `/var` del sistema raíz. Directorios como `/var/lib/pacman` deben mantener coherencia con la versión del sistema que se está ejecutando.

Los snapshots Btrfs no incluyen automáticamente los subvolúmenes anidados. Por tanto, `updated` debe conocer expresamente qué subvolúmenes forman parte de cada versión del sistema.

## 3.2. Identidad de cada versión

Cada instalación candidata debe tener:

- Identificador único.
- Subvolumen de origen.
- Subvolumen de destino.
- Versión de PARALLAX.
- Versión del conjunto de paquetes.
- Fecha de creación.
- Estado de validación.
- Entrada de arranque asociada.

Estados:

`PREPARING`, `READY`, `PENDING_BOOT`, `BOOTING`, `HEALTHY`, `FAILED`, `ROLLED_BACK`, `RETIRED`.

Los estados deben persistirse de manera segura para permitir recuperar el proceso después de un corte eléctrico.

---

# 4. Sistema de actualización — ADR-0009

## 4.1. Responsabilidades de `updated`

`updated` coordina las actualizaciones, pero no sustituye a `pacman`, Btrfs ni `systemd-boot`.

Sus responsabilidades son:

- Consultar actualizaciones disponibles.
- Validar el canal configurado.
- Verificar firmas y metadatos.
- Resolver dependencias.
- Comprobar espacio disponible.
- Preparar la versión candidata.
- Coordinar la instalación de paquetes.
- Preparar la entrada de arranque.
- Solicitar el reinicio cuando corresponda.
- Registrar el resultado.
- Facilitar el rollback cuando sea necesario.

No debe modificar directamente la base de datos de paquetes ni implementar un segundo gestor de dependencias.

## 4.2. Flujo de actualización

La operación debe dividirse en las siguientes etapas.

### Etapa 1 — Descubrimiento

1. Consultar los metadatos del canal configurado.
2. Comprobar la firma del repositorio.
3. Verificar que la versión sea compatible.
4. Comparar los paquetes instalados con los disponibles.
5. Mostrar al usuario los cambios relevantes.
6. Solicitar confirmación.

La descarga puede realizarse antes de la confirmación si el usuario lo permite, pero no debe aplicarse ningún cambio al sistema sin autorización.

### Etapa 2 — Prevalidación

Antes de modificar el sistema:

- Verificar espacio libre.
- Verificar estado de Btrfs.
- Verificar disponibilidad de la partición EFI.
- Comprobar que existe una versión anterior recuperable.
- Comprobar integridad de los metadatos.
- Verificar dependencias y conflictos.
- Detectar operaciones incompatibles con el esquema de almacenamiento.
- Registrar el estado inicial del sistema.

Si alguna condición crítica falla, se cancela la operación.

### Etapa 3 — Preparación de la versión candidata

Se creará un snapshot de referencia del sistema actual y se preparará una versión candidata independiente.

La estrategia preferida es:

1. Mantener intacta la raíz actualmente operativa.
2. Crear un snapshot de referencia.
3. Crear un destino escribible para la nueva versión.
4. Aplicar los paquetes en ese destino mediante un procedimiento validado.
5. Mantener coherentes el sistema de archivos y la base de datos de `pacman`.
6. Preparar los artefactos de arranque correspondientes.

**Decisión técnica pendiente:** validar mediante un prototipo que la instalación de paquetes en un root alternativo funciona correctamente con los scriptlets, hooks, initramfs, kernel, servicios y rutas del sistema. No asumir que ejecutar `pacman` contra otro directorio garantiza aislamiento completo.

Si no se demuestra que el procedimiento es seguro, no se debe anunciar la actualización como transaccional. Se deberá utilizar una estrategia alternativa con recuperación probada antes de habilitarla en `stable`.

### Etapa 4 — Validación previa al arranque

Antes de activar la versión candidata:

- Verificar los paquetes instalados.
- Verificar la base de datos de `pacman`.
- Verificar los archivos críticos.
- Generar y verificar el initramfs.
- Verificar el kernel y los módulos.
- Verificar la configuración de arranque.
- Verificar los metadatos de la versión candidata.
- Verificar que la versión anterior siga siendo arrancable.

Si una validación falla, la versión candidata se descarta y se conserva la versión anterior.

### Etapa 5 — Activación

Una vez validada:

1. Crear una entrada de arranque independiente.
2. Configurar boot counting.
3. Asociar la entrada con la raíz candidata.
4. Registrar el estado `PENDING_BOOT`.
5. Marcar la entrada candidata como siguiente opción de arranque.
6. Informar al usuario de que necesita reiniciar.

No eliminar ni sobrescribir la entrada anterior durante esta operación.

### Etapa 6 — Primer arranque

La nueva versión debe arrancar con un máximo inicial de tres intentos.

El mecanismo de boot counting de `systemd-boot` controla los intentos de arranque. Si se agotan, la entrada queda marcada como fallida y el cargador puede continuar con otra entrada válida.

No se debe asumir que el contador por sí solo restaura el snapshot: las entradas deben estar correctamente asociadas a raíces independientes y existir una entrada anterior funcional.

### Etapa 7 — Confirmación de salud

La versión candidata no se considera exitosa solamente porque Linux haya arrancado.

Se deben comprobar:

- Montaje correcto de la raíz.
- Integridad del sistema de archivos.
- Servicios esenciales.
- Acceso al gestor de sesiones.
- Inicio del compositor.
- Inicio de la sesión gráfica.
- Disponibilidad de los servicios críticos de PARALLAX.

La comprobación debe tener un timeout definido y evitar que un servicio no esencial impida confirmar el arranque.

Se utilizará `boot-complete.target` como punto de integración con el mecanismo de confirmación de systemd.

El componente responsable de la salud de la sesión debe informar su resultado antes de que `systemd-bless-boot` marque la entrada como válida.

La confirmación debe ser explícita y verificable; no basta con crear un target con el nombre `parallax-session-healthy.target`.

Una vez superadas las comprobaciones:

- Marcar la versión como `HEALTHY`.
- Confirmar el arranque mediante `systemd-bless-boot`.
- Conservar la versión anterior durante el periodo de retención configurado.
- Registrar el resultado.

Referencias técnicas:
- `systemd-boot`
- `systemd-bless-boot`
- `boot-complete.target`
- `systemd-boot-check-no-failures.service`

## 4.3. Política de actualizaciones

Por defecto, el usuario confirma todas las actualizaciones.

Se podrán habilitar actualizaciones automáticas de seguridad mediante una opción explícita.

La política automática debe permitir configurar:

- Descarga automática.
- Instalación automática.
- Reinicio automático o pendiente de confirmación.
- Horarios permitidos.
- Notificaciones.
- Actualizaciones críticas.

Una actualización que requiera reinicio no debe cerrar sesiones ni reiniciar el equipo sin respetar la política configurada.

## 4.4. Actualizaciones offline

PARALLAX permitirá instalar actualizaciones desde medios locales.

Formato inicial:

`.parallax-update`

El paquete debe contener:

- Manifiesto firmado.
- Metadatos de versión.
- Canal de destino.
- Paquetes necesarios.
- Hashes.
- Información de dependencias.
- Firma verificable.

El archivo no debe considerarse confiable por su extensión o por proceder de un dispositivo USB.

Antes de instalar:

1. Verificar la firma con una clave de confianza.
2. Verificar todos los hashes.
3. Comprobar arquitectura y compatibilidad.
4. Comprobar dependencias.
5. Verificar que no se trate de una versión inferior no autorizada.
6. Ejecutar el mismo flujo de prevalidación que una actualización online.

---

# 5. Recuperación del sistema

PARALLAX tendrá cinco niveles de recuperación.

Cada nivel debe poder ejecutarse sin depender de componentes del nivel que ha fallado.

## Nivel 1 — Recuperación de componentes

**Disparador:** un servicio o componente individual deja de funcionar.

Acción:

- Reinicio controlado mediante systemd.
- Registro del fallo.
- Aplicación de límites de reinicio.
- Notificación cuando el fallo afecta al usuario.

Las políticas deben definir:

- `Restart=`
- `RestartSec=`
- `StartLimitIntervalSec=`
- `StartLimitBurst=`

No todos los servicios deben reiniciarse indefinidamente.

Los servicios esenciales y los opcionales tendrán políticas distintas.

## Nivel 2 — Recuperación del Shell

**Disparador:** el Shell falla repetidamente.

Umbral inicial propuesto: cinco fallos en un minuto.

Acción:

1. El supervisor detecta el bucle de fallos.
2. Detiene los reinicios repetitivos.
3. Solicita al compositor mostrar una superficie de emergencia.
4. Desactiva temporalmente los widgets externos.
5. Ofrece iniciar el Shell con configuración predeterminada.
6. Permite restablecer la configuración del usuario.

La configuración original no debe eliminarse.

Al restablecerla:

- Crear una copia de seguridad.
- Renombrar la configuración problemática.
- Generar una configuración predeterminada.
- Registrar el motivo de la recuperación.

La interfaz de emergencia debe depender lo mínimo posible del Shell normal.

## Nivel 3 — Recuperación de la sesión gráfica

**Disparador:** el compositor falla repetidamente o no consigue iniciar una sesión funcional.

Umbral inicial propuesto: tres fallos dentro de una ventana de tiempo configurable.

Acción:

1. Terminar la sesión gráfica fallida.
2. Registrar el diagnóstico.
3. Intentar iniciar una sesión segura.
4. Mostrar un greeter independiente del Shell normal.

La sesión segura debe ofrecer:

- Configuración predeterminada.
- Sin widgets de terceros.
- Sin efectos visuales opcionales.
- Sin transparencias ni animaciones complejas.
- Sin extensiones experimentales.
- Renderizado por software como alternativa cuando el hardware y el backend lo permitan.

**Importante:** el renderizado por software no garantiza que cualquier fallo del compositor pueda recuperarse. La sesión segura necesita una ruta de inicio y configuración independiente y debe probarse en hardware real.

El usuario podrá:

- Iniciar sesión segura.
- Restablecer configuración.
- Consultar registros.
- Intentar iniciar la sesión normal.
- Acceder al modo de recuperación.

## Nivel 4 — Recuperación mediante snapshots

**Disparador:**

- Actualización fallida.
- Sistema inestable después de una actualización.
- Solicitud explícita del usuario.

Acción:

1. Identificar snapshots disponibles.
2. Verificar su integridad y asociación con una versión.
3. Mostrar fecha, versión y estado.
4. Permitir seleccionar una versión anterior.
5. Preparar una entrada de arranque válida.
6. Reiniciar hacia la versión seleccionada.
7. Verificar que el sistema arranca correctamente.

La restauración debe preservar `@home` y los datos personales.

No se deben eliminar snapshots automáticamente si son la única ruta de recuperación disponible.

La política de retención debe considerar:

- Espacio disponible.
- Número de versiones anteriores.
- Antigüedad.
- Estado de validación.
- Existencia de una copia externa cuando corresponda.

## Nivel 5 — Imagen de recuperación

**Disparador:** el sistema instalado no puede arrancar o los mecanismos normales de recuperación fallan.

Se proporcionará una imagen de recuperación independiente.

Características:

- Arranque independiente del sistema instalado.
- Entorno mínimo.
- Herramientas de diagnóstico.
- Acceso a Btrfs.
- Verificación de volúmenes.
- Restauración de snapshots.
- Reparación del arranque.
- Reinstalación del sistema.
- Acceso controlado a los datos personales.

La imagen debe estar firmada y su integridad debe verificarse antes de utilizarla.

### Protección mediante dm-verity

La imagen de recuperación podrá utilizar dm-verity para verificar la integridad de su sistema raíz de solo lectura.

dm-verity protege la integridad de los bloques verificados; no cifra los datos ni garantiza por sí solo la autenticidad de la imagen. La autenticidad debe establecerse mediante una cadena de confianza y firmas verificadas.

### Opciones del entorno de recuperación

| Opción | Comportamiento |
|---|---|
| Diagnosticar | Recopila información del sistema y errores |
| Reparar arranque | Reconstruye las entradas y artefactos de arranque cuando sea posible |
| Restaurar snapshot | Recupera una versión anterior |
| Reparar sistema | Ejecuta comprobaciones y reparaciones compatibles |
| Reinstalar PARALLAX | Reinstala el sistema sin formatear automáticamente los datos personales |
| Abrir terminal | Permite diagnóstico avanzado |

**Regla de seguridad:** nunca formatear, borrar ni sobrescribir `@home` automáticamente durante una reinstalación. Las operaciones destructivas requieren confirmación explícita y una advertencia clara.

---

# 6. Gestión de configuración — `settingsd`

`settingsd` será responsable de la persistencia segura de la configuración.

## 6.1. Escritura atómica

Toda modificación debe seguir este procedimiento:

1. Validar los nuevos valores.
2. Serializar la configuración.
3. Escribir un archivo temporal en el mismo sistema de archivos.
4. Sincronizar los datos con disco.
5. Validar el archivo temporal.
6. Crear una copia de seguridad de la última versión válida.
7. Sustituir el archivo original mediante una operación atómica.
8. Sincronizar el directorio cuando corresponda.

No debe sobrescribirse directamente el archivo original.

## 6.2. Recuperación ante corrupción

Si el archivo principal no puede parsearse o no supera la validación:

1. Intentar cargar `.bak`.
2. Validar su contenido y versión de esquema.
3. Restaurar la última versión válida.
4. Registrar el incidente.
5. Notificar al usuario.
6. Si ambas versiones fallan, utilizar valores predeterminados seguros.

La copia `.bak` no debe considerarse válida sin verificar su contenido.

## 6.3. Migraciones de configuración

Cada esquema debe tener una versión explícita.

Ejemplo conceptual:

- `schema_version: 1`
- `schema_version: 2`
- `schema_version: 3`

Las migraciones deben ser:

- Versionadas.
- Repetibles de forma segura.
- Probadas.
- Reversibles cuando sea técnicamente posible.
- Independientes de la configuración de otros usuarios.

Una migración fallida no debe destruir la configuración anterior.

---

# 7. Seguridad

El sistema de actualización y recuperación debe cumplir estas reglas:

1. Verificar firmas antes de instalar paquetes.
2. No confiar en medios externos sin verificar.
3. No permitir que aplicaciones normales modifiquen las entradas de arranque.
4. Ejecutar las operaciones privilegiadas mediante servicios específicos.
5. Aplicar autorización mediante polkit cuando corresponda.
6. Registrar operaciones administrativas.
7. Evitar que un widget o extensión pueda ejecutar operaciones de recuperación.
8. Proteger las claves privadas de firma fuera del sistema de usuario.
9. No permitir downgrade silencioso de componentes críticos.
10. Mantener una ruta de recuperación disponible después de cada actualización.

Las operaciones privilegiadas deben exponerse mediante una API limitada y validada, no mediante acceso genérico a comandos de root.

---

# 8. Componentes del sistema

| Componente | Responsabilidad |
|---|---|
| `updated` | Coordinar actualizaciones |
| `recoveryd` | Coordinar recuperación y restauración |
| `settingsd` | Configuración persistente y migraciones |
| `px-boot` | Integración y gestión controlada de entradas de arranque |
| `px-snapshot` | Operaciones Btrfs y metadatos de snapshots |
| `px-health` | Comprobación de salud del sistema y la sesión |
| `parallax-recovery` | Entorno de recuperación independiente |

Estos nombres representan responsabilidades lógicas. No implican que cada función deba convertirse inmediatamente en un proceso separado.

Se priorizará una arquitectura pequeña y modular. No crear servicios independientes sin una justificación de aislamiento, seguridad o ciclo de vida.

---

# 9. Evolución: sistema A/B

## Versión 1

- Arch Linux como base.
- `pacman`.
- Repositorio PARALLAX firmado.
- Btrfs.
- Snapshots.
- `systemd-boot`.
- Boot counting.
- Rollback probado.

## Versión 2 — Imagen A/B

Evaluar:

- `mkosi`.
- `systemd-sysupdate`.
- UKI.
- Particiones o imágenes versionadas.
- Integridad mediante dm-verity.
- Activación de versiones independientes.
- Actualizaciones atómicas.

La arquitectura A/B permitirá preparar una versión completamente separada de la versión en ejecución y activarla cuando esté validada.

`systemd-sysupdate` ofrece mecanismos para gestionar recursos versionados A/B, pero su estado de estabilidad debe verificarse en la versión concreta de systemd que adopte PARALLAX. No se debe convertir en dependencia de producción sin una evaluación de compatibilidad y madurez.

La decisión se tomará después de recopilar resultados de la versión 1.

---

# 10. Pruebas obligatorias

Ninguna implementación se considerará terminada sin pruebas de fallos.

| ID | Escenario | Resultado esperado |
|---|---|---|
| UPD-001 | Firma inválida | Actualización rechazada |
| UPD-002 | Paquete corrupto | Instalación cancelada |
| UPD-003 | Espacio insuficiente | No se modifica el sistema operativo activo |
| UPD-004 | Corte eléctrico durante descarga | El sistema anterior sigue arrancando |
| UPD-005 | Corte eléctrico durante instalación candidata | La versión anterior sigue disponible |
| UPD-006 | Corte eléctrico al activar entrada | Existe una entrada anterior válida |
| UPD-007 | Kernel nuevo no arranca | Boot counting permite fallback |
| UPD-008 | Servicios esenciales fallan | La versión no se marca como válida |
| UPD-009 | Shell falla | Se ofrece recuperación |
| UPD-010 | Compositor falla | Se ofrece sesión segura |
| UPD-011 | Configuración corrupta | Se recupera `.bak` |
| UPD-012 | Configuración principal y backup corruptos | Se utilizan valores predeterminados seguros |
| REC-001 | Restaurar snapshot | El sistema vuelve a una versión anterior |
| REC-002 | Restaurar snapshot con `@home` independiente | Los datos personales permanecen intactos |
| REC-003 | Sistema sin arranque | La imagen de recuperación inicia |
| REC-004 | Entrada EFI corrupta | Se puede reconstruir el arranque |
| REC-005 | Reinstalación | No se formatea `@home` automáticamente |
| SEC-001 | Medio offline manipulado | Firma o hash inválido, instalación rechazada |
| SEC-002 | Intento de operación sin privilegios | Operación denegada |

Las pruebas deben incluir interrupciones reales de procesos y simulación de pérdida de alimentación en entornos controlados.

---

# 11. Criterios de aceptación

La implementación inicial solo podrá aprobarse cuando:

- [ ] Una actualización pueda prepararse sin modificar la raíz activa.
- [ ] La versión anterior permanezca arrancable.
- [ ] Los paquetes y metadatos se verifiquen criptográficamente.
- [ ] Se pueda detectar una actualización incompleta.
- [ ] El boot counting funcione en hardware compatible.
- [ ] `systemd-bless-boot` confirme solamente un arranque saludable.
- [ ] El rollback recupere una versión anterior.
- [ ] `@home` permanezca intacto durante el rollback.
- [ ] El Shell pueda recuperarse sin depender de sí mismo.
- [ ] Exista una ruta de inicio de sesión segura.
- [ ] La configuración se escriba atómicamente.
- [ ] La recuperación offline funcione.
- [ ] Las pruebas de interrupción y recuperación estén documentadas.

---

# 12. Decisiones pendientes

Antes de aprobar esta especificación deben resolverse:

1. **Estrategia real de actualización sobre Btrfs:** demostrar cómo se instala en una raíz candidata sin inconsistencias entre paquetes, hooks, scriptlets, `/var/lib/pacman` y artefactos de arranque.
2. **Diseño de subvolúmenes:** cerrar la política de `/var`, `/etc`, `/home`, estado persistente y snapshots.
3. **Confirmación de salud:** establecer qué servicios y comprobaciones permiten alcanzar `boot-complete.target`.
4. **Gestión EFI:** definir cómo se conservan y recuperan las entradas y los artefactos de arranque.
5. **Recuperación independiente:** determinar cómo se inicia la imagen cuando el cargador o la instalación principal están dañados.
6. **Retención:** definir cuántas versiones anteriores se conservan y bajo qué condiciones se eliminan.
7. **Cifrado:** definir cómo se gestiona el acceso a volúmenes cifrados durante la recuperación.

**Estado final:** PROPUESTO — NO APROBADO.

La implementación queda bloqueada hasta cerrar las decisiones pendientes y demostrar el flujo de actualización y rollback en hardware Linux real.