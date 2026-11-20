# Registro de excepciones

- **Estado del registro:** Propuesto (base-audit-r3 §6.3, P-24). Pendiente de aprobación.
- Una excepción solo es válida con aprobación expresa del propietario registrada aquí. Ninguna excepción autoriza código de producción.

## Campos
| Campo | Contenido |
|---|---|
| ID | `EXC-NNN` |
| Afecta a | ADR, criterio de fase o regla de AGENTS.md |
| Motivo | Por qué no puede cumplirse ahora |
| Permite | Alcance exacto |
| No permite | Lo que sigue prohibido |
| Responsable | Agente o persona |
| Caducidad | Fecha o condición de cierre |
| Aprobación del propietario | — / fecha |
| Estado | Solicitada · Aprobada · Rechazada · Cerrada |

## Registro

| ID | Afecta a | Motivo | Permite | No permite | Responsable | Caducidad | Aprobación | Estado |
|---|---|---|---|---|---|---|---|---|
| EXC-001 | `criterios-aceptacion.md` cierre de Fase 0; P-06 | Contratos D-Bus XML y `architecture/storage.md` dependen de decisiones de Fase 1 | Cerrar Fase 0 sin esos documentos | Implementar servicios D-Bus o el instalador antes de que existan | arquitecto-sistemas, ingeniero-linux | Inicio de Fase 1 + entrega de ambos documentos | — | Solicitada (propuesta de base-audit-r3 §7.1) |
