# Propuesta: plantilla de ADR, estados y aprobaciones

- **Estado:** Propuesta — pendiente de aprobación de arquitecto-sistemas y del propietario. **No normativa.**
- **Fecha:** 2026-11-19
- **Resuelve (si se aprueba):** P-20, P-13, P-19 (base-audit-r3 §6).
- No modifica `decisions/ADR-0000-plantilla.md` ni ningún ADR. §4 lista los cambios a aplicar tras aprobación.

## 1. Situación actual

- Plantilla (`ADR-0000`): `Propuesto | Aceptado | Rechazado | Sustituido por ADR-XXXX`; un campo "Aprobado por".
- ADR-0001…0012: cabecera en negrita, sin "Aprobado por"; estados con texto libre ("Propuesto — requiere benchmark (Fase 1)", "— requiere decisión del propietario", "— decisión redactada…").
- ADR-0013…0015: formato corto; algunos con "Dueño/Revisa".
- AGENTS.md exige aprobación del arquitecto (§Cambios arquitectónicos) y del responsable (§Estado del proyecto). La plantilla solo tiene un campo.

## 2. Propuesta

### 2.1 Estados (conjunto cerrado)
`Propuesto` · `Aceptado` · `Rechazado` · `Sustituido por ADR-NNNN`

| Estado | Efecto |
|---|---|
| `Aceptado` | Normativo. Requiere las dos aprobaciones de §2.3. |
| `Propuesto` | Orienta el trabajo; no prevalece como decisión definitiva ni autoriza implementación. |
| `Rechazado` | Sin efecto; se conserva como registro. |
| `Sustituido por ADR-NNNN` | Sin efecto; prevalece el ADR sustituto si está `Aceptado`. |

Sin texto libre en el campo de estado. No se añaden `Obsoleto` ni `Provisional` (base-audit-r3 §6.1).

### 2.2 Plantilla propuesta

```markdown
# ADR-NNNN: Título

- **Estado:** Propuesto | Aceptado | Rechazado | Sustituido por ADR-NNNN
- **Fecha:** AAAA-MM-DD
- **Dueño técnico:** <agente/área>
- **Revisan:** <agentes>
- **Aprobación técnica (arquitecto-sistemas):** — | AAAA-MM-DD
- **Aprobación del propietario:** — | AAAA-MM-DD
- **Relacionado con:** ADR-…
- **Excepciones vigentes:** — | EXC-NNN

## 1. Contexto
## 2. Opciones evaluadas
## 3. Decisión
## 4. Consecuencias (positivas, negativas, riesgos)
## 5. Condiciones de aceptación
Lista verificable que debe cumplirse para pasar a `Aceptado`.
## 6. Evidencias
Enlaces a spikes (SPK-NNN), benchmarks, documentos. Una evidencia no existente no se enlaza.
## 7. Condiciones pendientes tras la aceptación
Solo si se acepta con condiciones: condición, fase, responsable, spike asociado.
## 8. Criterio de revisión
Qué dato haría cambiar la decisión.
```

### 2.3 Aprobaciones
- **Técnica:** arquitecto-sistemas (AGENTS.md §Cambios arquitectónicos).
- **Del propietario:** responsable del proyecto (AGENTS.md §Estado; roadmap §2).
- `Aceptado` solo con ambas fechas registradas. Ningún agente registra la aprobación del propietario sin instrucción expresa de este.

### 2.4 Reglas de alcance
1. Un ADR `Aceptado` con condiciones pendientes, o una excepción de cierre de fase, **no autoriza código de producción**; eso lo hace el gate de la fase (AGENTS.md §Estado).
2. Los spikes no son implementación: requieren su propia autorización ([registro de spikes](spikes.md)).
3. Las excepciones se registran en el [registro de excepciones](excepciones.md) con alcance y caducidad.

### 2.5 Prevalencia en documentos derivados
Un documento que aplica un ADR debe indicar su estado. Fórmula propuesta: "Refleja ADR-NNNN (estado: X). Normativo cuando ADR-NNNN esté `Aceptado`." Corrige P-19 en `security/dependencias.md`.

## 3. Migración propuesta de ADR existentes

| ADR | Estado normalizado | Texto que pasa a §5/§7 |
|---|---|---|
| 0005 | Propuesto | "requiere benchmark (Fase 1)" → Condiciones de aceptación |
| 0008 | Propuesto | "decisión redactada, pendiente de aprobación" → se elimina; ya implícito |
| 0011 | Propuesto | "requiere spike de integración (Fase 4)" → Condiciones |
| 0012 | Propuesto | "requiere validación de compatibilidad (Fase 2)" → Condiciones |
| 0013–0015 | Propuesto | Ampliar al formato completo (P-03, P-13) |
| Resto | Propuesto | Añadir campos de aprobación vacíos |

## 4. Cambios a aplicar tras aprobación (no aplicados)

| Archivo | Cambio |
|---|---|
| `decisions/ADR-0000-plantilla.md` | Sustituir por §2.2 |
| `decisions/ADR-0001…0015` | Cabecera según §3; sin cambiar estado |
| `security/dependencias.md` l.3 | Fórmula de §2.5 (P-19) |
| `AGENTS.md` §Cambios arquitectónicos | Opcional: citar las dos aprobaciones y §2.4 |
