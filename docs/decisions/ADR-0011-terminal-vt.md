# ADR-0011: Motor de emulación de terminal

- **Estado:** Propuesto — requiere spike de integración (Fase 4)
- **Fecha:** 2026-11-19
- **Ámbito:** Emulación VT, terminal gráfica, sesiones PTY y renderizado.
- **Relaciones:** ADR-0001 (Rust), ADR-0005 (Toolkit UI), ADR-0006 (Procesos del Shell).

## 1. Contexto

PARALLAX necesita una terminal nativa integrada con su entorno de escritorio.

La terminal deberá proporcionar:

- Emulación de terminal compatible con aplicaciones CLI.
- Interpretación de secuencias ANSI y VT.
- Gestión de buffers y scrollback.
- Renderizado eficiente de texto.
- Soporte para Unicode.
- Entrada de teclado.
- Selección y copia de texto.
- Portapapeles.
- Redimensionamiento dinámico.
- Colores y temas.
- Soporte para múltiples sesiones.
- Integración con shells y herramientas existentes.

Implementar completamente un motor de terminal desde cero supondría mantener un parser VT, una máquina de estados, un grid, un sistema de selección y una amplia superficie de compatibilidad.

Se prioriza reutilizar una implementación madura y concentrar el desarrollo propio en la experiencia visual y la integración con PARALLAX.

## 2. Opciones evaluadas

| Alternativa | Ventajas | Consideraciones |
|---|---|---|
| `alacritty_terminal` | Motor Rust maduro, parser y grid integrados | API y evolución upstream |
| `vte` + grid propio | Control del grid y reutilización del parser | Mayor responsabilidad de implementación |
| Motor propio completo | Control absoluto | Coste, compatibilidad y mantenimiento elevados |

## 3. Decisión provisional

Se selecciona `alacritty_terminal` como candidato principal para proporcionar:

- Parser VT.
- Máquina de estados.
- Grid de terminal.
- Gestión de líneas y celdas.
- Scrollback.
- Interpretación de secuencias de control.
- Estado de pantalla.

PARALLAX desarrollará su propia interfaz, integración con PTY y sistema de renderizado.

La selección definitiva dependerá de un spike técnico durante la Fase 4.

La licencia Apache-2.0 deberá verificarse para la versión concreta incorporada y sus dependencias.

## 4. Arquitectura

La terminal se dividirá en capas independientes.

```text
┌──────────────────────────────────────┐
│          PARALLAX TERMINAL           │
│                                      │
│  Tabs · Panels · Search · Settings   │
└──────────────────┬───────────────────┘
                   │
                   v
┌──────────────────────────────────────┐
│             px-terminal              │
│                                      │
│ Sessions · Input · PTY · Clipboard   │
│ Resize · Configuration · Lifecycle   │
└──────────────────┬───────────────────┘
                   │
                   v
┌──────────────────────────────────────┐
│         Terminal Engine Adapter      │
│                                      │
│       alacritty_terminal              │
│                                      │
│       Parser · Grid · State           │
└──────────────────┬───────────────────┘
                   │
                   v
┌──────────────────────────────────────┐
│              px-ui                   │
│                                      │
│ Glyphs · Selection · Cursor · UI     │
│ Rendering · Themes · Accessibility  │
└──────────────────────────────────────┘
```

La arquitectura es conceptual: `px-ui` proporciona los mecanismos de presentación, mientras que el motor de terminal expone el estado necesario para renderizarlo.

## 5. Motor de emulación

`alacritty_terminal` será responsable de interpretar el comportamiento del terminal.

Se utilizará como infraestructura para:

- Parser VT.
- Secuencias ANSI.
- Modos de terminal.
- Pantalla principal y alternativa.
- Grid de caracteres.
- Atributos de celdas.
- Colores.
- Scrollback.
- Cursor.
- Actualizaciones de pantalla.

PARALLAX no deberá modificar innecesariamente el motor upstream.

Se mantendrá una capa adaptadora que permita sustituir el motor en el futuro si aparecen limitaciones técnicas.

## 6. Capa px-terminal

`px-terminal` será responsable de la integración entre el motor y el sistema operativo.

### Responsabilidades

- Crear y administrar sesiones.
- Crear PTY.
- Gestionar procesos hijos.
- Iniciar shells configurados.
- Enviar entrada al proceso.
- Recibir salida del PTY.
- Gestionar redimensionamiento.
- Coordinar el ciclo de vida de las sesiones.
- Gestionar señales y terminación de procesos.
- Gestionar el cierre de sesiones.
- Integrar el portapapeles.
- Exponer una API independiente del motor.

### PTY

Se utilizará un pseudoterminal del sistema operativo.

El proceso hijo deberá disponer de un entorno de ejecución adecuado para aplicaciones interactivas.

Se contemplará:

- `TERM`.
- Tamaño inicial del terminal.
- Redimensionamiento mediante `TIOCSWINSZ`.
- Señales de cambio de tamaño.
- Gestión de procesos hijos.
- Cierre de descriptores.
- Limpieza de procesos al finalizar la sesión.

La terminal no asumirá que todos los shells utilizan las mismas configuraciones o capacidades.

## 7. Renderizado propio

El renderizado será responsabilidad de PARALLAX mediante la infraestructura gráfica de `px-ui`.

Deberá incluir:

- Renderizado de glifos.
- Gestión de fuentes.
- Caché de glifos.
- Colores ANSI.
- True Color.
- Cursor.
- Selección.
- Scrollback visual.
- Subrayado y estilos de texto.
- Negrita e itálica.
- Combinación de caracteres.
- Emoji y caracteres de ancho variable.

El renderizador deberá contemplar Unicode y las reglas de ancho de caracteres, incluyendo caracteres combinados y de doble ancho.

No se asumirá que un carácter Unicode corresponde siempre a un único glifo.

### Optimización

Se priorizarán:

- Caché de glifos.
- Actualización de celdas modificadas.
- Renderizado por regiones.
- Evitar reconstruir toda la superficie en cada actualización.
- Control de asignaciones de memoria.
- Reutilización de recursos gráficos.

Se evaluará el uso de atlas de glifos y mecanismos de renderizado acelerado.

## 8. Entrada e interacción

La terminal deberá soportar:

- Teclado.
- Modificadores.
- Atajos configurables.
- Selección de texto.
- Copiar y pegar.
- Scroll vertical.
- Búsqueda.
- Selección por palabras y líneas.
- Mouse reporting.
- Envío de eventos de ratón a aplicaciones TUI.
- Soporte para modos de teclado utilizados por aplicaciones modernas.

Se respetarán los modos de entrada y salida definidos por el protocolo del terminal.

El sistema no deberá interceptar indiscriminadamente combinaciones de teclas que las aplicaciones TUI necesitan recibir.

## 9. Compatibilidad

La terminal deberá probarse con herramientas habituales:

- Bash.
- Zsh.
- Fish.
- Nushell.
- Vim.
- Neovim.
- Tmux.
- Htop.
- Btop.
- Git.
- Herramientas interactivas basadas en TUI.

También deberá contemplar:

- Aplicaciones que utilizan pantalla alternativa.
- Aplicaciones que cambian el tamaño del terminal.
- Secuencias de control incompletas.
- Unicode.
- Colores de 256 entradas.
- True Color.
- Terminales con diferentes valores de `TERM`.

La compatibilidad no se basará únicamente en que una aplicación pueda iniciarse; deberá verificarse su comportamiento interactivo.

## 10. Seguridad

La terminal ejecutará procesos externos con los permisos del usuario.

El motor de emulación no deberá interpretar secuencias de control como instrucciones arbitrarias del sistema operativo.

Se prestará especial atención a:

- Secuencias OSC.
- Integración con el portapapeles.
- Hipervínculos.
- Solicitudes de información del terminal.
- Secuencias de control de títulos.
- Imágenes y protocolos gráficos.
- Datos malformados o excesivamente grandes.

Las funciones que puedan producir efectos externos, como modificar el portapapeles o abrir enlaces, deberán tener políticas explícitas.

El contenido emitido por un proceso no deberá considerarse confiable.

## 11. Gestión de memoria

Se establecerán límites configurables para:

- Líneas de scrollback.
- Cantidad de sesiones.
- Tamaño de buffers.
- Caché de glifos.
- Recursos gráficos.
- Datos de búsqueda.
- Imágenes o contenido adicional, si se incorpora en el futuro.

La terminal deberá liberar recursos cuando se cierren sesiones.

El scrollback no deberá crecer ilimitadamente.

Se medirá el consumo con múltiples sesiones y cargas de salida intensivas.

## 12. Benchmark y spike de integración

Durante la Fase 4 se realizará un prototipo funcional.

### Escenario A: emulación

- Ejecutar shells interactivos.
- Ejecutar aplicaciones TUI.
- Probar secuencias VT.
- Validar pantalla alternativa.
- Probar redimensionamiento.

### Escenario B: renderizado

- Renderizar texto estático.
- Renderizar salida continua.
- Modificar regiones pequeñas.
- Medir tiempos de actualización.
- Evaluar caché de glifos.

### Escenario C: Unicode

- Caracteres ASCII.
- Caracteres combinados.
- Idiomas con escrituras complejas.
- Emoji.
- Caracteres de doble ancho.

### Escenario D: carga

- Salida de texto intensiva.
- Scrollback grande.
- Varias sesiones.
- Aplicaciones TUI.
- Redimensionamiento frecuente.

### Escenario E: seguridad

- Secuencias OSC no confiables.
- Solicitudes de portapapeles.
- Secuencias malformadas.
- Mensajes excesivamente grandes.
- Terminación inesperada del proceso hijo.

### Criterios de aceptación

- Emulación funcional con aplicaciones CLI y TUI habituales.
- Unicode y caracteres de ancho variable correctos.
- PTY y redimensionamiento funcionales.
- Scrollback con límites configurables.
- Renderizado incremental.
- Sin crecimiento sostenido de memoria.
- Sin dependencia directa de Alacritty en la interfaz pública de PARALLAX.
- Licencias y dependencias verificadas.

## 13. Gestión de versiones

La versión de `alacritty_terminal` se fijará explícitamente.

Se revisarán los cambios upstream antes de actualizar.

Toda actualización deberá ejecutar:

- Pruebas de emulación.
- Pruebas de compatibilidad.
- Pruebas de Unicode.
- Pruebas de seguridad.
- Benchmarks.
- Pruebas de memoria.

Se evitarán modificaciones locales al motor salvo que exista una necesidad demostrada.

Las modificaciones que no puedan mantenerse como adaptadores deberán documentarse.

## 14. Consecuencias

### Positivas

- Motor VT maduro.
- Implementación nativa Rust.
- Menor coste de mantenimiento.
- Renderizado visual completamente controlado por PARALLAX.
- Posibilidad de desarrollar una experiencia propia.
- Independencia entre interfaz y emulación.

### Negativas

- Dependencia de un proyecto externo.
- Necesidad de adaptar su API.
- Responsabilidad propia sobre PTY y sesiones.
- Trabajo adicional para el renderizador.
- Posibles limitaciones en protocolos de terminal avanzados.

## 15. Revisión

Revisar este ADR después del spike de integración de la Fase 4.

Cambiar la alternativa si:

- `alacritty_terminal` no permite implementar los requisitos.
- La API impide mantener un adaptador estable.
- Existen problemas graves de rendimiento.
- La gestión de Unicode resulta insuficiente.
- Las necesidades de integración superan las capacidades del motor.

## 16. Principio rector

**PARALLAX reutilizará un motor de emulación probado y construirá su propia experiencia de terminal.**

La compatibilidad con el ecosistema CLI tendrá prioridad sobre reinventar protocolos que ya llevan décadas de evolución.