# Temas y personalización — PARALLAX

**Estado:** Propuesto · Pendiente de aprobación.

## 1. Objetivo

Definir un sistema centralizado de temas y personalización para PARALLAX.

El sistema debe proporcionar:

- Apariencia consistente entre compositor, Shell y aplicaciones propias.
- Temas claros, oscuros y de alto contraste.
- Personalización controlada por el usuario.
- Cambios en caliente sin reiniciar aplicaciones.
- Contraste y accesibilidad verificables.
- Temas comunitarios declarativos y seguros.
- Adaptadores para aplicaciones de terceros.
- Bajo consumo de recursos.
- Independencia entre preferencias, resolución de tokens y renderizado.

Los temas no pueden sustituir componentes, ejecutar código ni modificar la lógica de la interfaz.

## 2. Arquitectura

### 2.1 Componentes

**settingsd**

Responsabilidades:

- Almacenar las preferencias del usuario.
- Validar los valores recibidos.
- Gestionar la versión de configuración.
- Ejecutar migraciones de preferencias.
- Resolver qué tema y personalizaciones están seleccionados.
- Publicar cambios mediante `Settings1`.
- Mantener un snapshot coherente de la configuración efectiva.

No debe implementar lógica de renderizado ni depender del toolkit gráfico.

**px-theme**

Biblioteca Rust compartida y sin estado global mutable.

Responsabilidades:

- Cargar y validar esquemas de temas.
- Resolver tokens semánticos.
- Aplicar herencia y overrides.
- Validar rangos y combinaciones.
- Calcular colores derivados.
- Evaluar restricciones de contraste.
- Generar una representación tipada y versionada de los tokens efectivos.

Debe poder utilizarse desde el compositor, Shell, aplicaciones y herramientas de validación sin depender de Smithay ni del toolkit.

**px-ui**

Responsabilidades:

- Consumir los tokens efectivos.
- Aplicarlos a los componentes.
- Gestionar estados visuales.
- Aplicar transiciones de color.
- Respetar accesibilidad y movimiento reducido.

No debe implementar su propio sistema independiente de temas.

**parallax-compositor**

Consume exclusivamente los tokens necesarios para:

- Decoraciones.
- Bordes.
- Sombras.
- Radios.
- Estados de ventana.
- Efectos de composición.
- Overlays propios.

No debe depender del Shell para conocer el tema actual.

**Adaptadores externos**

Generan configuraciones compatibles con aplicaciones de terceros:

- GTK.
- Qt.
- Terminales.
- XDG Desktop Portal.

No forman parte del núcleo del motor de temas.

### 2.2 Flujo de configuración

```text
Preferencias del usuario
          |
          v
       settingsd
          |
          v
 Validación de preferencias
          |
          v
      px-theme
          |
          v
 Resolución de tokens efectivos
          |
          v
 Snapshot versionado
          |
          +----------> parallax-compositor
          |
          +----------> parallax-shell / px-ui
          |
          +----------> aplicaciones PARALLAX
          |
          +----------> adaptadores externos
```

`settingsd` es propietario de las preferencias persistentes.

`px-theme` es responsable de la resolución determinista de tokens.

Los consumidores no deben modificar las preferencias ni escribir directamente la configuración de otros componentes.

### 2.3 Actualización en caliente

Al cambiar un tema:

1. El usuario modifica una preferencia.
2. `settingsd` valida la solicitud.
3. `px-theme` calcula los tokens efectivos.
4. Se valida la configuración resultante.
5. Se persiste la preferencia de forma atómica.
6. Se incrementa la revisión de configuración.
7. Se publica el cambio mediante `Settings1`.
8. Los consumidores reciben el nuevo snapshot o delta.
9. Shell y compositor aplican los tokens.
10. Se ejecuta la transición visual correspondiente.

La actualización debe ser coherente entre consumidores.

Cada snapshot debe incluir:

- `schema_version`.
- `theme_id`.
- `revision`.
- `effective_tokens`.
- `accessibility_settings`.

Los consumidores deben ignorar revisiones antiguas y solicitar un snapshot actualizado si detectan una discontinuidad.

Un fallo de aplicación no debe dejar permanentemente al Shell y al compositor utilizando configuraciones incompatibles. Debe existir una ruta de resincronización.

## 3. Temas base

### 3.1 Oscuro

<badge color="success">Predeterminado</badge>

Utiliza los tokens establecidos en `DESIGN_TOKENS`.

Características:

- Fondo oscuro.
- Superficies elevadas.
- Transparencia controlada.
- Acento configurable.
- Sombras y profundidad espacial.
- Contraste legible sin oscurecer artificialmente el contenido de las aplicaciones.

### 3.2 Claro

Tokens iniciales:

| Token | Valor |
|---|---|
| `bg.base` | `#F6F7F9` |
| `bg.surface` | `#FFFFFF` |
| `bg.elevated` | `#FFFFFF` |
| `bg.glass` | `rgba(250,250,252,0.72)` |
| `fg.primary` | `#14161B` |
| `fg.secondary` | `#4E5360` |

Características:

- Superficies claras.
- Elevación mediante sombras y diferencias de superficie.
- Transparencia moderada.
- Contraste equivalente al tema oscuro.
- Acento ajustado mediante OKLCH cuando sea necesario.

Las sombras no deben depender exclusivamente de aumentar su opacidad. Se deben validar visualmente sobre los fondos correspondientes.

### 3.3 Automático

Modos:

- Horario personalizado.
- Amanecer y atardecer.
- Cambio automático según configuración del usuario.

Para amanecer y atardecer se requiere:

- Latitud y longitud configuradas por el usuario.
- Fecha local.
- Zona horaria.
- Algoritmo astronómico ejecutado localmente.

No solicitar ubicación precisa ni conectividad obligatoria.

Si no existe una ubicación configurada, utilizar un horario personalizado o una alternativa explícita.

Definir:

- Hora de transición.
- Comportamiento durante días polares.
- Comportamiento durante noches polares.
- Zona horaria y cambios de horario estacional.
- Preferencia manual que temporalmente prevalece sobre el modo automático.

### 3.4 Alto contraste

Variantes:

- Alto contraste oscuro.
- Alto contraste claro.

Características:

- Separación visual clara de superficies.
- Bordes perceptibles.
- Texto legible.
- Indicadores de foco prominentes.
- Transparencias deshabilitadas o reducidas cuando perjudiquen la legibilidad.
- Efectos visuales secundarios minimizados.
- No depender exclusivamente del color para comunicar estados.

El modo de alto contraste debe tener prioridad sobre las preferencias decorativas que puedan reducir la accesibilidad.

## 4. Sistema de tokens

Los tokens se organizan en categorías semánticas:

```text
color/
├── background
├── surface
├── foreground
├── accent
├── border
├── status
└── overlay

typography/
├── family
├── size
├── weight
├── line-height
└── tracking

spacing/
├── component
├── layout
└── window

radius/
├── control
├── panel
└── window

elevation/
├── shadow
├── blur
└── depth

motion/
├── duration
├── easing
└── reduced-motion
```

Los componentes consumen tokens semánticos, no valores arbitrarios dispersos.

Ejemplos:

- `fg.primary`.
- `fg.secondary`.
- `bg.surface`.
- `bg.elevated`.
- `accent.primary`.
- `border.subtle`.
- `status.danger`.
- `elevation.window.active`.
- `motion.base`.

El esquema debe definir tipos y unidades permitidas.

No permitir expresiones, referencias dinámicas arbitrarias ni funciones ejecutables en los archivos de temas.

## 5. Personalización permitida

### 5.1 Color de acento

- Nueve colores predefinidos.
- Color personalizado.
- Ajuste automático de luminosidad.
- Conversión y manipulación mediante OKLCH.
- Validación del contraste.
- Vista previa antes de aplicar.

El sistema no debe modificar silenciosamente el color seleccionado.

Cuando el color no cumpla los requisitos, debe mostrar el ajuste aplicado o permitir elegir otra variante.

### 5.2 Acento desde fondo de pantalla

Función opcional.

Proceso:

1. Analizar el fondo localmente.
2. Extraer colores dominantes.
3. Seleccionar un candidato.
4. Ajustar luminosidad y cromaticidad.
5. Validar contraste.
6. Presentar una vista previa.
7. Aplicar únicamente tras aceptación del usuario.

No enviar el fondo de pantalla a servicios externos.

La extracción de color debe ejecutarse fuera del hilo de renderizado.

### 5.3 Opacidad del vidrio

Rango: 0,5–1,0.

- Valor inicial definido por el tema.
- Configuración del usuario.
- Contraste validado después de componer el fondo.
- Compatibilidad con reducción de transparencia.

La validación debe considerar que el contenido detrás de una superficie translúcida puede cambiar.

Si no puede garantizarse un contraste adecuado, utilizar una superficie más opaca.

### 5.4 Transparencia reducida

Valores:

- Activada.
- Desactivada.
- Automática.

La modalidad automática puede considerar batería y capacidades del hardware.

La reducción de transparencia debe modificar el material visual, no solamente reducir el alpha.

### 5.5 Radios

- Compacto: ×0,6.
- Normal: ×1,0.
- Redondeado: ×1,3.

Aplicar factores sobre tokens base.

Respetar mínimos y máximos por componente.

No permitir que un override genere controles inutilizables o áreas interactivas incoherentes.

### 5.6 Densidad

- Compact.
- Normal.
- Comfortable.

Afecta a:

- Espaciado.
- Alturas de controles.
- Separación entre elementos.
- Densidad de listas.
- Dimensiones de paneles.

No debe reducir las áreas de interacción por debajo de los mínimos de accesibilidad definidos por PARALLAX.

### 5.7 Velocidad de animación

Rango: 0,5×–1,5×.

Opciones:

- Reducida.
- Normal.
- Rápida.

La configuración de movimiento reducido tiene prioridad sobre cualquier multiplicador.

No acelerar animaciones críticas de accesibilidad hasta hacerlas imperceptibles o difíciles de seguir.

### 5.8 Tipografía

Permitir fuentes UI y monoespaciadas instaladas.

- Fuente principal.
- Fuente monoespaciada.
- Fallback automático.
- Fallback para CJK.
- Fallback para árabe.
- Fallback para emoji.
- Fallback para símbolos.

Utilizar Fontconfig y el mecanismo de fallback del toolkit.

La selección de una fuente no garantiza que contenga todos los glifos.

Evitar que una fuente no disponible rompa la disposición de la interfaz.

### 5.9 Tamaño de texto

Rango: 100–200 %.

Debe afectar coherentemente:

- Texto de aplicaciones PARALLAX.
- Etiquetas.
- Menús.
- Diálogos.
- Decoraciones.
- Widgets.
- Componentes accesibles.

El cambio de tamaño debe recalcular geometrías sin producir recortes ni superposiciones.

### 5.10 Dock y topbar

Permitir personalizar:

- Posición del Dock.
- Tamaño del Dock.
- Comportamiento de ocultación.
- Topbar flotante o pegada.
- Transparencia permitida.
- Densidad.

La especificación de geometría y comportamiento pertenece a los documentos de Dock y Shell.

Este documento define únicamente la exposición de las preferencias al sistema de temas.

## 6. Accesibilidad y contraste

Aplicar WCAG 2.2 AA como referencia.

### 6.1 Contraste de texto

- Texto normal: mínimo 4,5:1.
- Texto grande: mínimo 3:1.

### 6.2 Contraste no textual

- Componentes e indicadores visuales esenciales: mínimo 3:1 respecto a colores adyacentes pertinentes, cuando aplique el criterio.

### 6.3 Transparencias

El contraste se calcula después de componer el color de primer plano, la superficie y el fondo.

No validar únicamente los valores RGBA originales.

### 6.4 Colores derivados

Validar combinaciones para:

- Texto sobre superficies.
- Texto sobre acento.
- Estados de error.
- Estados de advertencia.
- Estados de éxito.
- Controles deshabilitados.
- Indicadores de foco.
- Bordes relevantes.
- Selecciones.
- Notificaciones.

Los elementos deshabilitados tienen excepciones en algunos criterios WCAG, pero no deben perder legibilidad innecesariamente.

### 6.5 Validación

La validación debe realizarse automáticamente mediante `px-theme` y complementarse con pruebas visuales.

Si un tema no cumple las restricciones obligatorias:

- Rechazar la importación.
- Informar del token incompatible.
- Identificar las combinaciones afectadas.
- Ofrecer un ajuste seguro.

No modificar arbitrariamente el tema sin informar al usuario.

## 7. Temas comunitarios

Los temas comunitarios son archivos TOML declarativos.

No pueden contener:

- CSS.
- JavaScript.
- Scripts.
- Ejecutables.
- Plugins.
- Componentes personalizados.
- Código Rust.
- Referencias externas ejecutables.
- Carga dinámica de recursos remotos.

Ejemplo conceptual:

```toml
[theme]
id = "community.midnight"
name = "Midnight"
schema_version = 1

[colors]
bg_base = "#0E0F13"
bg_surface = "#16181E"
fg_primary = "#ECEDF1"
accent = "#7C8CFF"

[radius]
window = 12

[motion]
base_ms = 200
```

El esquema definitivo será tipado y versionado.

### 7.1 Validaciones

Al importar:

- Validar sintaxis.
- Validar versión del esquema.
- Validar identificador.
- Validar tipos.
- Validar rangos.
- Validar colores.
- Validar contraste.
- Validar tokens obligatorios.
- Rechazar campos desconocidos cuando puedan alterar el comportamiento.
- Limitar tamaño y complejidad del archivo.

### 7.2 Herencia

Permitir heredar de temas base aprobados.

La herencia debe ser determinista.

Orden:

1. Tema base.
2. Overrides del tema.
3. Preferencias del usuario.
4. Reglas de accesibilidad obligatorias.

Los overrides del usuario no pueden desactivar restricciones de accesibilidad.

### 7.3 Seguridad

Los temas no ejecutan código y no pueden:

- Leer archivos.
- Acceder a red.
- Lanzar procesos.
- Cambiar permisos.
- Modificar políticas de seguridad.

La instalación debe poder revertirse y no modificar archivos del sistema fuera de los directorios autorizados.

## 8. Aplicaciones de terceros

La compatibilidad visual con aplicaciones externas es best-effort.

No se garantiza que todas las aplicaciones reproduzcan exactamente el aspecto de PARALLAX.

### 8.1 GTK4

Generar una configuración compatible a partir de los tokens.

- Paleta.
- Preferencias de esquema claro/oscuro.
- CSS generado cuando corresponda.
- Estados de controles.

El CSS generado es un artefacto de compatibilidad, no un mecanismo para modificar el Shell ni sustituir sus componentes.

No inyectar reglas arbitrarias en aplicaciones.

### 8.2 Qt

Generar:

- Paleta.
- Preferencias de color.
- Configuración de estilo cuando sea posible.
- Integración opcional con Kvantum.

Kvantum será opcional y no una dependencia del núcleo.

### 8.3 Terminales

Generar paletas compatibles con los formatos de configuración soportados por las terminales instaladas.

No sobrescribir configuraciones personalizadas sin consentimiento.

### 8.4 XDG Desktop Portal

Publicar las preferencias correspondientes mediante `org.freedesktop.portal.Settings`.

Utilizar `org.freedesktop.appearance` para comunicar las preferencias de apariencia compatibles, incluido el esquema de color cuando corresponda.

Los valores y tipos deben respetar la especificación del portal.

No asumir que todas las aplicaciones consultan o reaccionan inmediatamente a los cambios.

### 8.5 Decoraciones de servidor

Las aplicaciones que acepten SSD mediante `xdg-decoration` reciben la apariencia definida en `WINDOW_STATES`.

Las aplicaciones CSD mantienen su propia decoración.

No garantizar uniformidad visual absoluta entre aplicaciones de terceros.

## 9. Persistencia

Las preferencias se almacenan mediante `settingsd`.

Ubicación conceptual:

```text
$XDG_CONFIG_HOME/parallax/
├── settings.toml
└── themes/
    ├── installed/
    └── imported/
```

Requisitos:

- Escritura atómica.
- Validación antes de persistir.
- Versionado del esquema.
- Migraciones.
- Recuperación ante archivos corruptos.
- Backup de la configuración anterior.
- No modificar directamente archivos administrados por otros componentes.

Los tokens efectivos pueden almacenarse en memoria y distribuirse mediante snapshots.

No es necesario persistir todos los tokens derivados si pueden calcularse determinísticamente.

## 10. Rendimiento

El sistema de temas debe evitar:

- Recalcular tokens por frame.
- Leer archivos de configuración durante el renderizado.
- Consultar fuentes instaladas repetidamente.
- Ejecutar extracción de colores en el hilo de UI.
- Reconstruir recursos gráficos innecesariamente.
- Generar eventos duplicados.

Los tokens se calculan al cambiar la configuración.

Las actualizaciones se aplican mediante snapshots y eventos.

Los cambios de color pueden interpolarse utilizando `motion.base`.

El cambio de tema no debe bloquear el compositor ni provocar recompilaciones innecesarias de shaders.

## 11. Pruebas obligatorias

### Funcionales

- Cambiar entre temas.
- Aplicar overrides.
- Cambiar acento.
- Cambiar fuentes.
- Modificar densidad.
- Cambiar tamaño de texto.
- Programar modo automático.
- Importar y exportar temas.
- Rechazar archivos inválidos.
- Restaurar configuración anterior.
- Reiniciar consumidores y recuperar snapshots.

### Accesibilidad

- Contraste de texto.
- Contraste de componentes.
- Indicadores de foco.
- Alto contraste.
- Movimiento reducido.
- Escalado al 200 %.
- Fuentes alternativas.
- CJK.
- Árabe RTL.
- Emoji.

### Integración

- Shell.
- Compositor.
- Aplicaciones PARALLAX.
- GTK4.
- Qt.
- Terminal.
- XDG Desktop Portal.

### Robustez

- Tema corrupto.
- Tema con campos desconocidos.
- Token fuera de rango.
- Cambio simultáneo de preferencias.
- Consumidor desconectado.
- Snapshot obsoleto.
- Reinicio de `settingsd`.
- Fuente desinstalada.
- Fondo de pantalla inaccesible.

## 12. Criterios de aceptación

El sistema se considera implementado cuando:

- Existe una fuente de verdad para las preferencias.
- Los tokens se resuelven de manera determinista.
- Los consumidores reciben snapshots versionados.
- Los cambios se aplican en caliente.
- No se reinician aplicaciones propias para cambiar de tema.
- Se validan los contrastes y transparencias.
- Alto contraste tiene prioridad.
- Los temas comunitarios no ejecutan código.
- La selección de fuentes tiene fallback.
- GTK y Qt reciben adaptaciones cuando son compatibles.
- El portal publica las preferencias soportadas.
- No se producen bloqueos ni regresiones importantes de rendimiento.

**Estado: Propuesto. Pendiente de aprobación arquitectónica.**