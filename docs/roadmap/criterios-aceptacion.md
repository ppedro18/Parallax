# Criterios de aceptación globales

## Fase 0 (checklist de cierre)
- [x] Responsabilidades por componente — `architecture/arquitectura.md §2`, especificaciones
- [x] Relaciones entre componentes — diagrama de dependencias, `ipc.md`
- [x] Modelo de comunicación — `ipc.md`, `eventos.md`
- [x] Tecnologías justificadas — ADR-0001…0013
- [x] Arquitectura gráfica — `compositor.md`, `framework-ui.md`, ADR-0003/4/5
- [x] Sistema de diseño — `design-system/`
- [x] Especificaciones funcionales de apps
- [x] Arquitectura de recursos — `resource-intelligence.md`
- [x] Modelo de seguridad
- [x] Política de dependencias
- [x] Estrategia de recuperación
- [x] Criterios de rendimiento medibles — `benchmarks/metodologia.md`
- [x] Plan de pruebas
- [x] Instrucciones de agentes
- [x] Riesgos registrados
- [ ] **ADR críticos aceptados** — pendiente de tu aprobación (ver `fase0-revision.md`)

## Por componente
Cada especificación tiene §Aceptación; se completa al pasar a fase de implementación del componente.


## Verificaciones arquitectónicas transversales

Antes de aprobar Fase 0 deberán cumplirse las siguientes condiciones:

- [ ] No existen dependencias circulares entre componentes.
- [ ] Cada componente tiene un propietario arquitectónico definido.
- [ ] Los límites entre procesos están documentados.
- [ ] Los contratos IPC tienen versión, errores y políticas de compatibilidad.
- [ ] Los componentes críticos tienen estrategia de recuperación.
- [ ] Los procesos privilegiados están identificados.
- [ ] Las fronteras de confianza están documentadas.
- [ ] Las dependencias externas tienen licencia y mantenimiento evaluados.
- [ ] Los presupuestos de memoria tienen valores objetivo y métodos de medición.
- [ ] Los presupuestos de frame están definidos para 60, 120 y 144 Hz.
- [ ] Los escenarios de fallo tienen pruebas asociadas.
- [ ] Las decisiones provisionales tienen criterios objetivos de aceptación o sustitución.
- [ ] Las tecnologías seleccionadas tienen una versión o estrategia de versionado definida.
- [ ] Existe una matriz de compatibilidad entre compositor, Shell, toolkit y aplicaciones.

## Cierre formal de Fase 0

La Fase 0 podrá cerrarse cuando:

1. Todos los ADR críticos hayan sido aceptados o tengan una excepción documentada.
2. Las verificaciones arquitectónicas transversales estén completas.
3. No existan contradicciones críticas entre especificaciones.
4. Los riesgos de alta prioridad tengan mitigación definida.
5. La arquitectura permita iniciar los prototipos técnicos de Fase 1.

La aprobación de Fase 0 no significa que todas las tecnologías estén definitivamente validadas.

Los ADR que requieran benchmarks o spikes conservarán su estado provisional hasta completar sus respectivos criterios de aceptación.