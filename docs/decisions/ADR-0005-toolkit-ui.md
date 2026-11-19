# ADR-0005: Toolkit de interfaz gráfica nativo Rust

- **Estado:** Propuesto — requiere benchmark (Fase 1)
- **Fecha:** 2026-11-19
- **Ámbito:** Shell, superficies gráficas, componentes visuales y aplicaciones propias de PARALLAX.
- **Especificación complementaria:** `specifications/framework-ui.md`

## 1. Contexto

PARALLAX necesita una infraestructura gráfica para construir su entorno de escritorio.

El toolkit seleccionado deberá permitir desarrollar:

- Dock.
- Launcher.
- Control Center.
- Topbar.
- Notificaciones.
- Menús contextuales.
- Widgets.
- Paneles flotantes.
- Diálogos del sistema.
- Componentes de configuración.
- Interfaces de administración del entorno.

El toolkit deberá integrarse con la arquitectura Wayland y Smithay, respetar el sistema de diseño PARALLAX y ofrecer rendimiento adecuado en un entorno de escritorio de larga duración.

La elección no debe basarse exclusivamente en facilidad de desarrollo o apariencia visual.

Debe evaluarse su rendimiento, consumo de memoria, accesibilidad, integración gráfica y capacidad de mantenimiento.

## 2. Requisitos fundamentales

El toolkit seleccionado deberá cumplir:

### Arquitectura
- Implementación nativa en Rust.
- Integración con Wayland.
- Compatibilidad con la arquitectura del compositor.
- Posibilidad de crear superficies independientes.
- Separación entre lógica de interfaz y presentación.
- Capacidad de reutilizar componentes.

### Renderizado
- Renderizado acelerado por GPU cuando corresponda.
- Actualizaciones parciales de interfaz.
- Gestión eficiente de superficies.
- Evitar renderizados innecesarios.
- Soporte para animaciones.
- Comportamiento eficiente con interfaces grandes.

### Accesibilidad
- Integración con AccessKit cuando sea compatible.
- Exposición correcta de roles y propiedades accesibles.
- Navegación por teclado.
- Gestión de foco.
- Compatibilidad con lectores de pantalla mediante AT-SPI.
- Soporte de escalado tipográfico.

### Integración visual
- Soporte para temas.
- Sistema de componentes reutilizables.
- Animaciones y transiciones.
- Transparencia cuando la plataforma lo permita.
- Integración con el sistema de profundidad PARALLAX.
- Adaptación a diferentes densidades y resoluciones.

## 3. Opciones evaluadas

| Alternativa | Ventajas | Consideraciones |
|---|---|---|
| Iced | Rust nativo, licencia MIT, arquitectura declarativa | Madurez de APIs e integración gráfica |
| libcosmic | Integración con el ecosistema COSMIC y experiencia de escritorio | Mayor dependencia del ecosistema COSMIC |
| Slint | Componentes declarativos y herramientas de diseño | Modelo de licencia y evaluación de integración |
| Toolkit propio | Control completo de arquitectura y comportamiento | Coste de desarrollo y mantenimiento elevado |

## 4. Decisión provisional

Se selecciona **Iced como candidato principal**, encapsulado mediante una biblioteca propia denominada `px-ui`.

Esta decisión no representa una aprobación definitiva del toolkit.

La selección final dependerá de los resultados del benchmark de Fase 1.

### 4.1 Iced

Iced será evaluado como infraestructura principal para las interfaces del Shell.

Se priorizará:

- Su implementación Rust.
- Su licencia MIT.
- Su modelo de componentes declarativos.
- Su integración con wgpu.
- Su arquitectura de actualización y renderizado.
- Su capacidad de integración con Wayland.
- Su comportamiento con AccessKit.
- Su rendimiento en interfaces de larga duración.

No se asumirá que todas las funcionalidades necesarias están disponibles directamente en Iced.

Las capacidades específicas de Wayland, layer-shell, accesibilidad y superficies independientes deberán verificarse mediante pruebas reales.

### 4.2 libcosmic

libcosmic será evaluado como alternativa arquitectónica.

Se tendrá en cuenta que pertenece al ecosistema COSMIC y está construido sobre tecnologías relacionadas con Iced.

Su evaluación deberá considerar:

- Dependencias del ecosistema COSMIC.
- Integración con Wayland.
- Madurez de sus componentes.
- Flexibilidad visual.
- Coste de adaptación a PARALLAX.
- Compatibilidad con el sistema de diseño propio.

No se adoptará únicamente por compartir tecnologías con el entorno COSMIC.

### 4.3 Slint

Slint permanecerá como alternativa si Iced no alcanza los requisitos de accesibilidad, memoria o integración gráfica.

Su evaluación deberá contemplar:

- Rendimiento.
- Integración con Wayland.
- Compatibilidad con el compositor.
- Herramientas de desarrollo.
- Modelo de componentes.
- Integración con Rust.
- Condiciones de licencia.

Antes de adoptarlo deberá resolverse formalmente su compatibilidad con la licencia elegida para PARALLAX.

### 4.4 Toolkit propio

Se descarta desarrollar un toolkit completo desde cero en la primera versión.

Esta decisión evita duplicar trabajo que ya resuelven proyectos especializados.

PARALLAX podrá desarrollar componentes, abstracciones y mecanismos gráficos propios sin necesidad de implementar todo un framework de interfaz.

La posibilidad de desarrollar un toolkit propio podrá revisarse en versiones futuras si existe una necesidad técnica demostrable.

## 5. Capa de abstracción px-ui

PARALLAX mantendrá una biblioteca propia denominada `px-ui`.

Su responsabilidad será proporcionar una API estable para las aplicaciones internas del Shell.

### Responsabilidades

- Componentes visuales.
- Tokens del sistema de diseño.
- Temas.
- Tipografía.
- Estados de componentes.
- Gestión de foco.
- Accesibilidad.
- Animaciones.
- Transiciones.
- Integración con el sistema de profundidad.
- Adaptadores de renderizado.
- Integración con superficies del compositor.

### Reglas arquitectónicas

- Las aplicaciones del Shell no deberán depender directamente de Iced cuando no sea necesario.
- Los tipos internos del toolkit no deberán propagarse por toda la arquitectura.
- Los componentes propios deberán consumir tokens oficiales.
- La lógica de negocio deberá permanecer separada de la presentación.
- Las interfaces deberán poder probarse independientemente del compositor.
- La capa de abstracción deberá ser delgada y evitar duplicar funcionalidades del toolkit.

`px-ui` no será un segundo framework completo ni una implementación paralela de Iced.

## 6. Integración con PARALLAX

El toolkit deberá integrarse con las siguientes capas:

```text
PARALLAX SHELL
      |
      v
   px-ui
      |
      v
  UI Adapter
      |
      v
  Iced / Toolkit elegido
      |
      v
 Wayland Surface
      |
      v
Smithay Compositor
      |
      v
 GPU / DRM / KMS
```

La implementación definitiva deberá respetar las responsabilidades de cada capa.

El toolkit no controlará directamente la política de gestión de ventanas.

El compositor mantendrá el control sobre la creación, posicionamiento, elevación y presentación de las superficies.

## 7. Integración con el sistema de diseño

`px-ui` será el punto de integración del sistema de diseño PARALLAX.

Deberá implementar:

- Tokens de color.
- Tipografía.
- Espaciado.
- Radios.
- Elevaciones.
- Estados.
- Temas.
- Sistema de profundidad.
- Movimiento.
- Transparencias.
- Accesibilidad.

Las aplicaciones internas no podrán introducir estilos arbitrarios que contradigan el sistema de diseño.

La personalización se realizará mediante propiedades y tokens oficialmente expuestos.

## 8. Benchmark obligatorio

La selección definitiva dependerá de un benchmark reproducible.

Se utilizará el mismo hardware, configuración, escenarios y condiciones de prueba para todas las alternativas.

### 8.1 Escenarios

**Escenario A: Ventana vacía**
- Crear una superficie.
- Mostrar una interfaz mínima.
- Medir memoria residente y PSS.
- Medir tiempo de inicialización.

**Escenario B: Lista de 10 000 elementos**
- Crear una lista con 10 000 registros.
- Implementar virtualización cuando corresponda.
- Medir desplazamiento.
- Medir tiempos de frame.
- Medir consumo de memoria.
- Comprobar respuesta de interacción.

**Escenario C: Componentes complejos**
- Tarjetas.
- Menús.
- Popovers.
- Controles.
- Iconografía.
- Imágenes.
- Actualizaciones de estado.

**Escenario D: Animaciones**
- Transiciones.
- Opacidad.
- Transformaciones.
- Profundidad.
- Apertura y cierre de paneles.

**Escenario E: Accesibilidad**
- Navegación por teclado.
- Foco.
- Lectores de pantalla.
- Árbol de accesibilidad.
- Escalado de texto.
- Roles y etiquetas.

**Escenario F: Larga duración**
- Ejecución prolongada.
- Apertura y cierre repetido de superficies.
- Actualizaciones frecuentes.
- Detección de fugas de memoria.
- Estabilidad del renderizado.

## 9. Criterios de aceptación

Los objetivos iniciales serán:

| Métrica | Objetivo |
|---|---|
| PSS de ventana vacía | ≤ 25 MB |
| Frame p99 en lista de 10 000 elementos | ≤ 8 ms |
| Accesibilidad | AccessKit funcional |
| Navegación | Teclado completamente operativo |
| Estabilidad | Sin crecimiento sostenido de memoria |
| Renderizado | GPU acelerada en hardware compatible |
| Integración | Wayland funcional |
| Componentes | Reutilizables mediante px-ui |

Los valores deberán medirse con procedimientos documentados en `benchmarks/metodologia.md`.

El objetivo de 8 ms corresponde a un presupuesto de frame de aproximadamente 120 Hz. Se registrarán también resultados para 60 Hz y otras frecuencias.

La medición de la lista deberá diferenciar el tiempo de actualización de la interfaz, el renderizado y la presentación del frame.

La virtualización no deberá considerarse una optimización opcional para listas grandes.

## 10. Gestión de versiones

El toolkit seleccionado tendrá una versión fijada explícitamente.

Se realizará seguimiento mensual de upstream.

Toda actualización deberá:

1. Revisar cambios incompatibles.
2. Ejecutar pruebas de integración.
3. Ejecutar benchmarks.
4. Validar accesibilidad.
5. Validar integración Wayland.
6. Comprobar compatibilidad con px-ui.
7. Documentar cambios relevantes.

Las versiones de producción no se actualizarán automáticamente.

## 11. Consecuencias

### Positivas

- Desarrollo nativo en Rust.
- Mayor reutilización de componentes.
- Separación entre aplicaciones y toolkit.
- Menor coste de mantenimiento que un framework completamente propio.
- Posibilidad de evolucionar la infraestructura gráfica.
- Integración con el sistema de diseño PARALLAX.

### Negativas

- Dependencia de un toolkit externo.
- Posibles limitaciones en accesibilidad.
- Evolución de APIs.
- Necesidad de mantener adaptadores.
- Posibles limitaciones de integración con Wayland.
- Coste de adaptación de componentes especializados.

## 12. Revisión

Este ADR deberá revisarse cuando:

- Iced no alcance los objetivos de memoria.
- Existan problemas de accesibilidad no solucionables.
- La integración Wayland resulte insuficiente.
- Slint o libcosmic demuestren ventajas técnicas significativas.
- El mantenimiento de `px-ui` se vuelva desproporcionado.
- Se justifique técnicamente la creación de un toolkit propio.

## 13. Principio rector

**PARALLAX controlará su lenguaje visual y su arquitectura de interfaz, independientemente del toolkit utilizado para renderizarla.**

El toolkit será infraestructura. La experiencia de usuario, los componentes y la identidad visual serán propiedad de PARALLAX.