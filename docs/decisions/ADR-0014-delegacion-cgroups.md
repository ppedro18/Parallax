# ADR-0014: Delegación de cgroups y acciones de resourced sobre procesos
- Estado: Propuesto · Fecha: 2026-11-19 · Dueños: ingeniero-recursos, ingeniero-linux · Revisa: ingeniero-seguridad
## Contexto
resourced quiere ajustar pesos, límites de memoria y congelar apps. Escribir directamente en cgroupfs choca con systemd (que es el dueño de la jerarquía) y, sin límites, permite afectar a componentes críticos o a procesos que el usuario no espera.
## Opciones
1. Escritura directa en cgroupfs con `Delegate=yes` sobre un subárbol propio.
2. API D-Bus de `systemd --user` (`SetUnitProperties` en runtime, `FreezeUnit`/`ThawUnit`).
3. Solo observación, sin actuar.
## Decisión
Opción 2, con las reglas de ámbito, origen, consentimiento, reversibilidad y no-matar definidas en `.claude/agents/ingeniero-recursos.md`. Modo observación (opción 3) por defecto hasta validar la política con pruebas de presión en Fase 5.
## Consecuencias
+ systemd sigue siendo el único dueño de la jerarquía; los cambios son visibles con `systemctl --user` y se pueden auditar y revertir.
− Latencia de D-Bus (~ms) por acción: aceptable porque la política reacciona con histéresis de segundos.
− Solo controla apps lanzadas como unidades systemd; procesos lanzados de otra forma quedan fuera.
## Revisión
Si las pruebas muestran que congelar causa defectos visibles (audio, red, IPC roto) en más del 1 % de apps probadas.
