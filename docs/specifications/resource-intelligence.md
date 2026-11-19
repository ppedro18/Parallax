# Especificación: Resource Intelligence — `parallax-resourced`

**Componente:** `parallax-resourced`  
**Estado:** Propuesto · Pendiente de aprobación  
**Capa:** C3 — Intelligence  
**Dependencias:** systemd, cgroup v2, PSI, UPower, compositor, hardwared y settingsd.

## 1. Objetivo

`parallax-resourced` supervisa la presión de recursos y aplica políticas adaptativas para mantener la estabilidad y capacidad de respuesta de PARALLAX.

No sustituye al kernel, systemd, systemd-oomd ni a los controladores de hardware.

Su responsabilidad es interpretar señales, clasificar cargas y coordinar políticas de recursos dentro de límites seguros.

## 2. Principios

1. No matar procesos directamente.
2. No reemplazar systemd-oomd.
3. No modificar frecuencias térmicas o eléctricas directamente.
4. No congelar aplicaciones sin una política explícita.
5. No perjudicar la capacidad de respuesta del compositor.
6. Aplicar histéresis.
7. Registrar las decisiones.
8. Permitir desactivación de políticas adaptativas.
9. Evitar polling innecesario.
10. No inferir actividad de una aplicación únicamente por su nombre.

## 3. Entradas

### 3.1 Sesión

Desde `Session1`:

- Ventana enfocada.
- Ventanas visibles.
- Ventanas minimizadas.
- Workspace activo.
- Estado de fullscreen.
- Estado de bloqueo.

La visibilidad real debe proceder del compositor y no inferirse únicamente del estado de la aplicación.

### 3.2 PSI

Fuentes:

- `/proc/pressure/cpu`
- `/proc/pressure/memory`
- `/proc/pressure/io`

Utilizar triggers de PSI cuando sea posible.

No confundir triggers con `avg10`.

Por ejemplo, un trigger `some 150000 1000000` representa 150 ms de presión en una ventana de un segundo, equivalente a un 15 % de tiempo de espera durante esa ventana; no es directamente un umbral `avg10`.

### 3.3 cgroup v2

Consultar:

- `cpu.stat`.
- `cpu.weight`.
- `memory.current`.
- `memory.stat`.
- `memory.events`.
- `memory.pressure`.
- `io.stat`.
- `cgroup.events`.

Las escrituras deben respetar la jerarquía y delegación configuradas por systemd.

### 3.4 Energía

UPower y power-profiles-daemon proporcionan estado de batería y perfil.

No controlar directamente frecuencias CPU/GPU.

## 4. Clasificación de aplicaciones

Clases:

- Foreground.
- Visible.
- Background.
- Suspendible.
- System.

La clasificación es dinámica.

### 4.1 Foreground

Aplicación con ventana enfocada.

Puede recibir mayor prioridad relativa mediante cgroup weight cuando la política y delegación lo permitan.

### 4.2 Visible

Aplicación con ventanas visibles sin foco.

No debe penalizarse como una aplicación totalmente en segundo plano.

### 4.3 Background

Aplicación sin ventanas visibles.

La ausencia de ventanas no implica que pueda congelarse: puede estar reproduciendo audio, descargando, sincronizando o realizando tareas solicitadas por el usuario.

### 4.4 Suspendible

Solo puede clasificarse como suspendible si:

- La aplicación declara compatibilidad.
- El usuario lo permite o lo solicita.
- No hay actividad protegida.
- No existe una operación crítica.
- No hay audio o transferencia activa.
- El estado de la aplicación es recuperable.

El tiempo de diez minutos es una heurística configurable, no una autorización automática.

## 5. Políticas de cgroup

Los valores iniciales propuestos:

| Clase | cpu.weight | io.weight |
|---|---:|---:|
| Foreground | 400 | 400 |
| Visible | 200 | 200 |
| Background | 50 | 50 |
| Suspendible | 10 | 10 |

Son valores experimentales.

Deben probarse en cargas reales y compararse con la configuración predeterminada de systemd.

`memory.low` y `memory.high` no se asignan automáticamente a partir de una clase.

### 5.1 memory.low

Es una protección relativa bajo presión, no una reserva física garantizada.

No asignar valores rígidos sin medir working set y efecto sobre otras aplicaciones.

### 5.2 memory.high

Puede limitar crecimiento y provocar reclaim/throttling.

Aplicarlo gradualmente y con capacidad de revertir.

No utilizarlo como mecanismo para castigar toda aplicación background.

### 5.3 Protección del sistema

La protección de compositor y Shell debe coordinarse con systemd y systemd-oomd.

Los valores iniciales de 64 MB para Shell y 128 MB para compositor son hipótesis de diseño, no reservas definitivas.

`memory.min` puede causar problemas de presión si se protege memoria excesiva.

Antes de usarlo, medir el working set real y validar el comportamiento del sistema completo.

## 6. Política ante presión

Utilizar señales de PSI con histéresis y persistencia temporal.

Los umbrales iniciales son propuestas para experimentación, no valores universales.

| Nivel | PSI memory some avg10 inicial | Acción |
|---|---|---|
| Normal | Menor a 5 % | Sin intervención |
| Bajo | 5–15 % | Solicitar liberación de cachés y reducir frecuencia de widgets |
| Medio | 15–40 % | Aplicar políticas reversibles y evaluar aplicaciones suspendibles |
| Crítico | Mayor a 40 % | Delegar decisiones de OOM a systemd-oomd y proteger la sesión |

La política debe considerar:

- Duración de la presión.
- `full` y `some`.
- Memoria disponible.
- Eventos de cgroup.
- Actividad de usuario.
- Capacidad del hardware.

No actuar sobre una única muestra.

### 6.1 Nivel bajo

- Emitir evento de presión.
- Solicitar liberación voluntaria de cachés.
- Reducir frecuencia de widgets.
- Evitar trabajo no prioritario.

### 6.2 Nivel medio

- Aplicar `memory.high` solo cuando exista una política validada.
- Evaluar congelación de aplicaciones declaradas suspendibles.
- No congelar aplicaciones activas ni procesos con tareas protegidas.
- Registrar cada intervención.

### 6.3 Nivel crítico

systemd-oomd mantiene la responsabilidad de decisiones de OOM a nivel de userspace.

`parallax-resourced` puede proporcionar contexto y políticas de protección, pero no debe competir con el mecanismo de selección y terminación del sistema.

No prometer que siempre se protegerá el foreground: el kernel puede necesitar recuperar memoria y la configuración debe ser realista.

## 7. Congelación

Congelar una aplicación requiere:

1. Identificar la unidad systemd correcta dentro del ámbito delegado (ADR-0014).
2. Verificar que la aplicación sea suspendible.
3. Registrar estado previo.
4. Llamar a `FreezeUnit` de `systemd --user` (nunca escribir `cgroup.freeze` directamente; ADR-0014).
5. Verificar resultado.
6. Registrar acción.

Al volver a ser foreground:

1. Descongelar.
2. Verificar estado.
3. Esperar a que la aplicación pueda procesar eventos.
4. Permitir interacción.

La descongelación debe ocurrir antes de esperar una respuesta visual de la aplicación.

Si falla, informar y no bloquear el compositor.

## 8. Energía

En batería:

- Widgets con intervalos ×2 como política inicial.
- Reducir trabajo de fondo.
- Evitar animaciones de fondo innecesarias.
- Sugerir `power-saver` por debajo del 20 %.

El usuario puede desactivar estas políticas.

No modificar frecuencias ni límites térmicos directamente.

## 9. Registro y transparencia

Todas las intervenciones deben registrar:

- Timestamp.
- Aplicación/cgroup.
- Señal de origen.
- Política aplicada.
- Valor anterior.
- Valor nuevo.
- Resultado.
- Motivo.

La información debe estar disponible en Observatory.

No almacenar datos innecesarios del contenido de las aplicaciones.

## 10. Configuración

El usuario puede:

- Desactivar Resource Intelligence.
- Configurar políticas de energía.
- Definir exclusiones.
- Desactivar suspensión automática.
- Revisar intervenciones.
- Restaurar valores predeterminados.

`settingsd` es propietario de las preferencias.

## 11. Pruebas

- Presión CPU.
- Presión de memoria.
- Presión I/O.
- Carga sostenida.
- Aplicaciones suspendibles.
- Aplicaciones con audio.
- Descargas activas.
- Desconexión de compositor.
- Desconexión de UPower.
- Cambio de batería a corriente.
- Histéresis.
- Reversión de políticas.
- Interacción con systemd-oomd.

## 12. Aceptación

Escenario:

`stress-ng --vm` genera presión de memoria controlada.

Criterios:

- El compositor mantiene p99 inferior a 33 ms en el escenario HW-A definido.
- Shell no se cierra.
- No se congelan aplicaciones protegidas.
- La política puede desactivarse.
- Las acciones quedan registradas.
- No hay oscilaciones constantes de freeze/unfreeze.
- Se recupera cuando desaparece la presión.

El objetivo de 33 ms no es una garantía universal: debe medirse con una carga reproducible, resolución, GPU y configuración registradas.

## 13. Decisiones pendientes

1. Definir autoridad de escritura sobre cgroups.
2. Definir delegación systemd.
3. Validar pesos CPU/I/O.
4. Definir memoria protegida del compositor y Shell.
5. Definir umbrales PSI por hardware.
6. Definir contrato de aplicaciones suspendibles.
7. Validar interacción con systemd-oomd.

**Estado: Propuesto. Pendiente de aprobación.**