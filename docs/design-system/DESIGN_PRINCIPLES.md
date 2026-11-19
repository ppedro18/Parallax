# Principios de diseño

1. **Función antes que forma.** Cada elemento visible tiene un comportamiento definido, estados completos y una razón para existir. Nada decorativo sin función.
2. **Profundidad como estructura.** La jerarquía se expresa con elevación y separación de planos (ver DEPTH_SYSTEM), no con bordes ni colores fuertes. Es la identidad "parallax".
3. **Calma.** Fondo oscuro neutro, un único color de acento, movimiento breve. La UI no compite con el contenido.
4. **Consistencia absoluta.** Un mismo patrón resuelve un mismo problema en todo el sistema. Si dos apps necesitan algo parecido, es un componente de `px-ui`.
5. **Teclado primero, ratón cómodo.** Todo es alcanzable y operable por teclado; el foco siempre es visible.
6. **Rápido se siente premium.** Una respuesta en < 100 ms vale más que cualquier efecto. Ante conflicto, gana la latencia.
7. **Efectos con presupuesto.** Translucidez, blur y sombras tienen coste medido y se degradan con elegancia (RENDERING_PERFORMANCE).
8. **Personalizable sin romperse.** Se personalizan tokens y comportamientos, no se inyecta código ni CSS. Toda combinación permitida cumple accesibilidad.
9. **Honestidad del estado.** La UI nunca miente: si un servicio no responde, se muestra "no disponible", no un valor viejo sin marcar.
10. **Original.** Inspirado en macOS y Caffyne en calidad, no en copia: sin semáforos de ventana clonados, sin assets ajenos.

## Reglas de decisión ante conflicto
Accesibilidad > latencia > claridad > coherencia > estética > novedad.
