# Criterios de aceptación del sistema de diseño — PARALLAX

**Fase:** 1 — Foundations & Design System  
**Estado:** Propuesto · Pendiente de aprobación.  
**Objetivo:** Establecer las condiciones objetivas para considerar terminado e integrado el sistema de diseño de PARALLAX.

## 1. Principios de aceptación

El sistema de diseño no se considera terminado por tener componentes visualmente implementados.

Debe demostrar:

- Consistencia visual.
- Accesibilidad funcional.
- Comportamiento predecible.
- Integración con el sistema de temas.
- Rendimiento medido.
- Compatibilidad con diferentes escalas y direcciones de escritura.
- Integración con el compositor.
- Trazabilidad de assets y licencias.
- Ausencia de regresiones críticas.

Cada criterio debe contar con evidencia verificable.

Un criterio no se marca como completado únicamente porque exista código o documentación.

La evidencia debe incluir pruebas automatizadas, resultados de ejecución y, cuando corresponda, revisión manual documentada.

## 2. Tokens y sistema de temas

### 2.1 Fuente de verdad

- [ ] Tokens implementados en `design-system/tokens/*.toml`.
- [ ] Esquema tipado y versionado.
- [ ] Validación de sintaxis, tipos, rangos y referencias.
- [ ] Generación reproducible hacia `px-ui::tokens`.
- [ ] Generación de los artefactos necesarios para compositor y aplicaciones.
- [ ] Los artefactos generados no se editan manualmente.
- [ ] CI comprueba que los artefactos generados coinciden con las fuentes.
- [ ] Los cambios incompatibles requieren actualización explícita del esquema.

### 2.2 Valores visuales

- [ ] No existen colores, radios, espaciados, sombras o tipografías hardcodeados en los componentes de Shell y aplicaciones propias cuando existe un token equivalente.
- [ ] Lint de búsqueda en CI.
- [ ] Lista explícita de excepciones justificadas.
- [ ] Los valores excepcionales tienen documentación y responsable.
- [ ] Los componentes consumen tokens semánticos, no valores directos de los temas.

El lint no debe rechazar indiscriminadamente valores que no sean tokens, por ejemplo:

- Dimensiones estructurales específicas.
- Coordenadas y geometrías calculadas.
- Valores de algoritmos gráficos.
- Valores internos de shaders.
- Datos de pruebas.
- Recursos gráficos con especificaciones propias.

La excepción no puede convertirse en un mecanismo para evadir el sistema de diseño.

### 2.3 Contraste

- [ ] Test automático de combinaciones permitidas `fg`/`bg`.
- [ ] Cobertura de temas oscuro, claro y alto contraste.
- [ ] Validación de estados normales, seleccionados, activos y de error.
- [ ] Validación de texto normal y grande.
- [ ] Validación de componentes e indicadores visuales.
- [ ] Validación de foco.
- [ ] Validación de colores derivados y overrides.
- [ ] Validación de superficies translúcidas después de composición.
- [ ] Pruebas de peor caso para vidrio sobre fondos variables.
- [ ] Rechazo de temas comunitarios que incumplan restricciones obligatorias.

Umbrales de referencia:

| Elemento | Contraste mínimo |
|---|---:|
| Texto normal | 4,5:1 |
| Texto grande | 3:1 |
| Componentes e indicadores visuales esenciales | 3:1 cuando aplique |

Para el vidrio no basta con comprobar el color de `bg.glass`.

Se deben probar diferentes fondos representativos:

- Superficie clara.
- Superficie oscura.
- Imagen.
- Vídeo.
- Fondo con colores extremos.
- Contenido dinámico.

El contraste debe calcularse sobre el resultado compuesto.

Si una superficie no puede garantizar el contraste necesario, debe utilizar el fallback opaco.

### 2.4 Acento personalizado

- [ ] Validador de colores personalizados.
- [ ] Conversión y ajuste mediante OKLCH.
- [ ] Rechazo o ajuste informado cuando no se cumplan los requisitos.
- [ ] Tests con un mínimo de 1.000 colores generados aleatoriamente.
- [ ] Semilla reproducible para los tests.
- [ ] Casos límite para luminosidad, saturación y colores extremos.
- [ ] Validación sobre todas las combinaciones de superficie permitidas.
- [ ] Verificación de que el ajuste no rompe otros estados del tema.

Los 1.000 colores son una prueba mínima de robustez, no una demostración exhaustiva de corrección.

El validador debe tener también pruebas unitarias de propiedades e invariantes.

## 3. Iconografía

- [ ] Set de iconos v1 completo según `ICONOGRAPHY § Set inicial`.
- [ ] Versiones de 16, 20 y 24 px.
- [ ] Revisión manual a tamaño real de 16 px.
- [ ] Alineación visual y consistencia de trazos.
- [ ] Consistencia de esquinas y proporciones.
- [ ] Estados activos, seleccionados y deshabilitados.
- [ ] Contraste suficiente en temas claro y oscuro.
- [ ] Fallback de iconos para recursos no disponibles.
- [ ] Licencia y procedencia documentadas para cada conjunto.
- [ ] Nombres semánticos y correspondencia con las acciones.

No asumir que escalar automáticamente un SVG garantiza legibilidad a 16 px.

Los iconos que lo necesiten deben disponer de ajustes ópticos específicos.

## 4. Galería de componentes

Crear una aplicación de referencia:

`px-ui/examples/component-gallery`

Debe permitir inspeccionar cada componente de manera independiente.

### 4.1 Cobertura visual

- [ ] Todos los componentes del catálogo.
- [ ] Todos sus estados universales aplicables.
- [ ] Variantes disponibles.
- [ ] Tamaños.
- [ ] Tema oscuro.
- [ ] Tema claro.
- [ ] Alto contraste.
- [ ] Densidad compacta.
- [ ] Densidad normal.
- [ ] Densidad comfortable.
- [ ] Dirección LTR.
- [ ] Dirección RTL.
- [ ] Tamaño de texto de 100 % y 200 %.

La galería debe permitir seleccionar combinaciones sin recompilar.

### 4.2 Estados universales

Como mínimo:

- Reposo.
- Hover.
- Presionado.
- Foco.
- Seleccionado.
- Deshabilitado.
- Cargando.
- Error.

Los estados no aplicables deben documentarse, no omitirse silenciosamente.

### 4.3 Estados de patrón

La galería debe incluir ejemplos de:

- Vacío.
- Cargando.
- Error.
- Sin permiso.
- Sin conexión.
- Operación en progreso.
- Operación completada.
- Operación cancelada.

## 5. Pruebas visuales

### 5.1 Capturas de referencia

- [ ] Captura de cada componente y variante.
- [ ] Captura de cada estado universal aplicable.
- [ ] Captura de temas claro y oscuro.
- [ ] Captura de alto contraste.
- [ ] Capturas representativas de RTL.
- [ ] Capturas a 100 % y 200 % de texto.
- [ ] Comparación automática contra referencias aprobadas.
- [ ] Detección de diferencias visuales.
- [ ] Aprobación manual de cambios intencionales.

### 5.2 Estrategia de cobertura

No ejecutar obligatoriamente todas las combinaciones posibles de componentes, estados, temas, densidades, escalas e idiomas en cada prueba.

Utilizar dos niveles:

**Cobertura exhaustiva:**

- Cada componente.
- Cada variante.
- Cada estado.
- Cada tema base.
- Cada caso de accesibilidad relevante.

**Cobertura combinatoria:**

- Densidades.
- Escalado.
- RTL.
- Tipografía.
- Transparencia.
- Estados simultáneos.
- Combinaciones visuales complejas.

La matriz combinatoria debe cubrir las interacciones de mayor riesgo y los casos límite.

### 5.3 Entorno de captura

Las capturas headless deben utilizar:

- Backend determinista.
- Resolución fija.
- Escala fija.
- Fuentes controladas.
- Animaciones deshabilitadas o congeladas cuando corresponda.
- Datos de prueba estables.
- Reloj controlado.
- Tema y tokens identificados.

Las capturas headless validan regresiones visuales, pero no sustituyen la validación del compositor sobre DRM/KMS y hardware real.

## 6. Accesibilidad por componente y pantalla

### 6.1 Estados y patrones

- [ ] Todos los estados universales implementados.
- [ ] Vacío.
- [ ] Cargando.
- [ ] Error.
- [ ] Sin permiso.
- [ ] Sin conexión cuando corresponda.
- [ ] Estados comunicados semánticamente.

### 6.2 Teclado

- [ ] Todas las acciones esenciales son operables mediante teclado.
- [ ] Orden de foco lógico.
- [ ] Foco visible.
- [ ] Sin trampas accidentales.
- [ ] Escape funciona según el contexto.
- [ ] Enter y Espacio funcionan según el rol.
- [ ] Navegación de menús.
- [ ] Navegación de listas.
- [ ] Navegación de tablas.
- [ ] Navegación de grids.
- [ ] Gestión de foco en diálogos y popovers.
- [ ] Tests scriptados de secuencias de teclado.

Los componentes puramente informativos no necesitan ser enfocables individualmente.

Las excepciones deben responder a su semántica, no a una limitación de implementación.

### 6.3 AccessKit

- [ ] Árbol accesible disponible.
- [ ] Roles correctos.
- [ ] Nombres accesibles correctos.
- [ ] Descripciones correctas.
- [ ] Estados correctos.
- [ ] Valores correctos.
- [ ] Relaciones entre controles.
- [ ] Anuncios de cambios relevantes.
- [ ] Tests automatizados del árbol accesible.

Validar también mediante integración real con AT-SPI y Orca.

La existencia de un árbol AccessKit no demuestra por sí sola que la experiencia de accesibilidad sea correcta.

### 6.4 Escalado

- [ ] Texto al 200 %.
- [ ] Escala 1,0.
- [ ] Escala 1,25.
- [ ] Escala 1,5.
- [ ] Escala 2,0.
- [ ] Sin solapamientos.
- [ ] Sin recortes de texto.
- [ ] Sin pérdida de controles.
- [ ] Sin áreas interactivas inaccesibles.
- [ ] Sin desbordamientos en diálogos, menús y paneles.

Distinguir entre:

- Escala de interfaz.
- Escala de texto.
- Escala del output.
- Escala fraccional del compositor.

No asumir que son equivalentes.

### 6.5 Preferencias de accesibilidad

- [ ] `reduced_motion` respetado.
- [ ] `reduced_transparency` respetado.
- [ ] Alto contraste respetado.
- [ ] Foco visible.
- [ ] Tamaño de texto respetado.
- [ ] Preferencias aplicadas en caliente.

Ninguna degradación automática de rendimiento puede desactivar funciones esenciales de accesibilidad.

## 7. Movimiento y profundidad

### 7.1 Coreografía de movimiento

- [ ] Cada animación definida en `MOTION` tiene duración y curva documentadas.
- [ ] Cada animación puede interrumpirse.
- [ ] Cada animación puede invertirse a mitad de ejecución.
- [ ] La inversión no produce saltos de posición.
- [ ] La inversión no produce estados inconsistentes.
- [ ] Las animaciones no bloquean input.
- [ ] El movimiento reducido tiene comportamiento alternativo.
- [ ] No existen animaciones infinitas innecesarias.

Test obligatorio:

1. Iniciar animación.
2. Interrumpir al 25 %.
3. Invertir dirección.
4. Comprobar continuidad.
5. Repetir al 50 % y 75 %.
6. Verificar estado final.
7. Verificar que el input sigue respondiendo.

### 7.2 Sistema de profundidad

- [ ] Planos definidos en `DEPTH_SYSTEM`.
- [ ] Orden de composición validado.
- [ ] Notificaciones sobre Launcher cuando corresponda.
- [ ] Dialogs y Popovers respetan sus relaciones de profundidad.
- [ ] Polkit sobre las superficies normales.
- [ ] Pantalla de bloqueo por encima de superficies ordinarias.
- [ ] No existe una vía visual que permita mostrar una solicitud privilegiada por encima de la pantalla de bloqueo.
- [ ] Overlays y scrims no interceptan entrada de forma incorrecta.

La capa de autenticación privilegiada no debe permitir eludir el estado de bloqueo.

Si el sistema está bloqueado, la autorización debe seguir la política de sesión segura definida por PARALLAX.

### 7.3 Estados de ventana

- [ ] Estados de `WINDOW_STATES` implementados.
- [ ] Foco.
- [ ] Hover.
- [ ] Maximizado.
- [ ] Fullscreen.
- [ ] Tiled.
- [ ] Snap.
- [ ] Minimizado.
- [ ] Moving.
- [ ] Resizing.
- [ ] Modal.
- [ ] Unresponsive.
- [ ] Urgent.
- [ ] Overview.

Las capturas headless verifican apariencia y geometría.

La integración real debe verificar comportamiento Wayland, input, clipping, stacking y presentación.

## 8. Rendimiento

### 8.1 Efectos

- [ ] Cada efecto tiene presupuesto.
- [ ] Cada efecto tiene fallback.
- [ ] Cada efecto cuenta con benchmark.
- [ ] Los costes se miden por separado.
- [ ] El coste agregado se mide en escenarios combinados.
- [ ] Se utilizan GPU timestamps cuando el backend lo permite.
- [ ] Se registra el estado de disponibilidad de medición.
- [ ] Resultados verificados en HW-A.
- [ ] Resultados repetidos en HW-B cuando esté disponible.

No utilizar capturas headless para certificar tiempos GPU de hardware real.

### 8.2 Reposo

Escenario de prueba:

- Shell visible.
- Topbar y Dock.
- Cuatro widgets sin segundos ni animaciones.
- Sin cambios de contenido.
- Sin entrada del usuario.
- Sin ventanas que actualicen contenido.
- Sin operaciones pendientes.
- Duración: 60 segundos.

Criterios:

- [ ] 0 frames nuevos solicitados por el compositor durante el intervalo estático.
- [ ] 0 renderizados vacíos.
- [ ] 0 invalidaciones periódicas innecesarias.
- [ ] 0 polling de widgets que no necesitan actualizarse.
- [ ] Sin wakeups periódicos de renderizado injustificados.

No confundir este criterio con actividad de otras aplicaciones, actualizaciones de hardware o eventos externos que requieran presentar una nueva imagen.

### 8.3 Degradación automática

- [ ] Se activa según los umbrales de `RENDERING_PERFORMANCE`.
- [ ] Se recupera según los umbrales de histéresis.
- [ ] No oscila continuamente.
- [ ] No modifica preferencias persistentes.
- [ ] No desactiva accesibilidad.
- [ ] Expone la causa de degradación a Observatory.

Prueba obligatoria:

1. Ejecutar escenario estable.
2. Introducir carga GPU sintética.
3. Verificar transición FULL → REDUCED.
4. Incrementar carga.
5. Verificar transición REDUCED → MINIMAL.
6. Retirar carga.
7. Verificar recuperación.
8. Comprobar histéresis.
9. Comprobar ausencia de oscilaciones.

La carga sintética debe producir presión medible y reproducible en la GPU real.

Una simulación puramente lógica puede validar la máquina de estados, pero no sustituye la prueba de rendimiento real.

## 9. Originalidad y licencias

### 9.1 Revisión

Revisión conjunta de:

- `disenador-producto`.
- `ingeniero-seguridad`.

### 9.2 Procedencia

- [ ] Cada asset tiene fuente identificada.
- [ ] Cada icono tiene licencia documentada.
- [ ] Cada fuente tipográfica tiene licencia documentada.
- [ ] Cada dependencia visual tiene licencia revisada.
- [ ] Se conserva atribución cuando sea obligatoria.
- [ ] Se documentan modificaciones y adaptaciones.
- [ ] No se incorporan recursos sin procedencia verificable.

### 9.3 Originalidad

- [ ] No se copian assets de Caffyne, Apple u otros proyectos.
- [ ] No se reproducen layouts protegidos o identidades visuales ajenas como si fueran propios.
- [ ] Las referencias visuales se utilizan como inspiración funcional, no como material para copiar.
- [ ] Se revisan similitudes visuales relevantes.
- [ ] Las decisiones de identidad visual están documentadas.

La revisión de originalidad no constituye una garantía legal absoluta de ausencia de similitudes.

Cuando exista una duda razonable sobre procedencia, licencia o similitud sustancial, el recurso debe sustituirse o someterse a revisión antes de distribuirse.

### 9.4 Registro de assets

Mantener un inventario:

```text
design-system/
└── assets/
    ├── manifest.toml
    ├── icons/
    ├── illustrations/
    └── fonts/
```

El manifiesto debe contener:

- Identificador.
- Archivo.
- Autor o fuente.
- Licencia.
- URL de procedencia cuando exista.
- Modificaciones realizadas.
- Fecha de incorporación.
- Estado de revisión.

## 10. Automatización y CI

Las pruebas deben ejecutarse en niveles.

### Nivel 1 — Cada cambio

- Validación de tokens.
- Generación de artefactos.
- Lint de valores literales.
- Tests unitarios.
- Tests de contraste.
- Tests de accesibilidad.
- Tests funcionales.
- Capturas visuales relevantes.

### Nivel 2 — Pull request

- Galería completa.
- Comparación visual.
- Matriz combinatoria.
- Tests de teclado.
- Tests de RTL.
- Tests de escalado.
- Verificación de artefactos generados.

### Nivel 3 — Hardware de referencia

- Benchmarks GPU y CPU.
- Damage tracking.
- Reposo.
- Degradación.
- Recuperación.
- Estados reales de ventana.
- Integración DRM/KMS.

### Nivel 4 — Revisión de versión

- Revisión visual del producto.
- Revisión de accesibilidad.
- Revisión de licencias.
- Revisión de riesgos.
- Aprobación de responsables.

## 11. Evidencias

Cada criterio debe poder vincularse a una evidencia.

Estructura conceptual:

```text
docs/
└── design-system/
    ├── acceptance-criteria.md
    ├── validation-report.md
    ├── accessibility-report.md
    ├── visual-regression-report.md
    ├── performance-report.md
    ├── asset-provenance.md
    └── approvals/
```

Cada informe debe incluir:

- Criterio evaluado.
- Resultado.
- Fecha.
- Commit.
- Entorno.
- Herramienta.
- Responsable.
- Evidencia.
- Incidencias.
- Estado.

## 12. Estados de aceptación

Utilizar los siguientes estados:

| Estado | Significado |
|---|---|
| PENDING | Sin iniciar |
| IN_PROGRESS | En implementación o pruebas |
| BLOCKED | Bloqueado por dependencia o problema |
| READY_FOR_REVIEW | Evidencia disponible, pendiente de revisión |
| APPROVED | Criterio aprobado |
| REJECTED | Criterio no aprobado |
| WAIVED | Excepción aprobada y documentada |

`WAIVED` requiere:

- Justificación.
- Riesgo asociado.
- Responsable de aprobación.
- Alcance.
- Fecha de revisión o expiración cuando corresponda.

No utilizar `WAIVED` para evitar resolver problemas críticos de accesibilidad, seguridad o estabilidad.

## 13. Gate de cierre de Fase 1

El sistema de diseño puede considerarse listo para aprobación cuando:

1. Los tokens tienen fuente de verdad y generación reproducible.
2. El motor de temas valida configuraciones.
3. El contraste cumple los criterios establecidos.
4. El set de iconos está revisado.
5. Existe una galería funcional de componentes.
6. Las pruebas visuales están automatizadas.
7. Los componentes cumplen los contratos de teclado y accesibilidad.
8. Se verifica escalado y RTL.
9. Movimiento y profundidad están integrados con el compositor.
10. Los efectos cumplen sus presupuestos en HW-A.
11. La degradación y recuperación están probadas.
12. Los assets tienen procedencia y licencias verificadas.
13. No quedan defectos críticos abiertos.
14. Los responsables técnicos han revisado las evidencias.

### Aprobaciones requeridas

- [ ] Arquitectura: consistencia con ADRs y dependencias.
- [ ] Diseño: identidad visual, tokens y componentes.
- [ ] Rendimiento: benchmarks y presupuestos.
- [ ] Seguridad: assets, licencias y superficies de privilegio.
- [ ] Accesibilidad: teclado, AccessKit, Orca y escalado.
- [ ] Responsable de fase: revisión integral.

**El cumplimiento de estos criterios habilita la solicitud de aprobación de Fase 1. No autoriza por sí mismo el inicio automático de la siguiente fase.**

**Estado: Propuesto. Pendiente de aprobación.**