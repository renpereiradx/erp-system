# docs/ — Ingeniería de Requerimientos

Este directorio registra **textualmente los requerimientos** de cada módulo o sistema del ERP
Vista Bar: *qué debe hacer el sistema y por qué*, escrito para que tanto ingeniería como
personas sin formación técnica (dirección, comerciales, soporte) puedan revisarlo y entenderlo.

Última actualización: 2026-09-21

---

## Qué es y qué NO es este directorio

| Directorio | Pregunta que responde | Audiencia | Ciclo de vida |
|:-----------|:----------------------|:----------|:--------------|
| **`docs/`** (este) | **¿QUÉ debe hacer el sistema y POR QUÉ?** | Todas (híbrido: informativo + formal) | Vivo: se actualiza cuando cambia el alcance |
| `conductor/` | ¿CÓMO y CUÁNDO se ejecutó el trabajo? (planes, fases, commits) | Ingeniería | Efímero: el plan se borra al cerrarse |
| Historial git | ¿Qué código exacto se cambió? | Ingeniería | Permanente |

Un requerimiento **no caduca cuando el trabajo termina** (a diferencia de un plan de
`conductor/`): queda como la definición vigente de cómo debe comportarse el módulo, y se
actualiza solo si el alcance cambia.

## Cuándo se crea un documento

- Al planificar un módulo o sistema nuevo (puede nacer como *Borrador* antes de existir código).
- Al formalizar un módulo existente que nunca tuvo requerimientos escritos.
- Nunca se crea "post facto" para trabajo trivial (un hotfix no necesita un REQ).

---

## Estructura obligatoria de cada documento

Nombre de archivo: `REQ_<CÓDIGO>_<NOMBRE_CORTO>.md` (código registrado en el índice de abajo).

```markdown
# REQ_<CÓDIGO> — <Nombre del módulo>

## 1. Ficha
| Campo | Valor |  → módulo, código, versión, estado, fecha, fuentes, implementación

## 2. Resumen ejecutivo (para todos)
2-3 párrafos sin tecnicismos: qué es y para qué existe.

## 3. Cómo funciona — guía no técnica
Lo que una persona sin formación técnica necesita para entender el módulo:
qué ve el usuario, escenarios paso a paso, analogías. Tablas y ejemplos bienvenidos.

## 4. Alcance
Listas explícitas: qué queda DENTRO y qué queda FUERA.

## 5. Actores
Quiénes interactúan o se ven afectados (personas y otros sistemas).

## 6. Requerimientos funcionales (RF-…)
Qué conductas observables debe tener el sistema.

## 7. Requerimientos no funcionales (RNF-…)
Restricciones de calidad: seguridad, performance, compatibilidad, operación.

## 8. Reglas de negocio (RN-…)
Decisiones de negocio tomadas sobre el módulo (el "por qué" de cada una).

## 9. Preguntas frecuentes
Las dudas reales que preguntaría un cliente/soporte, respondidas en lenguaje simple.

## 10. Glosario
Todo término técnico usado arriba, explicado en una línea.

## 11. Trazabilidad
Plan en conductor/, runbooks, commits que lo implementaron, tests que lo verifican.

## 12. Historia de cambios
| Versión | Fecha | Cambio | Autor |
```

Las tablas de requerimientos (§6-§8) usan siempre este formato:

```markdown
| ID | Requerimiento | Prioridad | Estado | Verificación |
```

---

## Convenciones

### IDs de requerimiento

- `RF-<CÓDIGO>-NNN` — requerimiento funcional (conducta observable).
- `RNF-<CÓDIGO>-NNN` — requerimiento no funcional (restricción de calidad).
- `RN-<CÓDIGO>-NNN` — regla de negocio (decisión, no conducta).
- Numeración correlativa desde 001. **Los IDs nunca se renumeran, reutilizan ni borran**:
  un requerimiento anulado se marca `Deprecado` con su motivo, y sigue ocupando su ID.

### Prioridad

- **Obligatorio** — sin esto el módulo no sirve o no se puede vender.
- **Deseable** — agrega valor, se acepta postergarlo.
- **Opcional** — registrado para el futuro, sin compromiso.

### Estados (del documento y de cada requerimiento)

`Borrador → En revisión → Aprobado → Implementado → (En cambio | Deprecado)`

### Versiones del documento

- `v1.0` al aprobarse. `+0.1` por ajustes de redacción o requerimientos nuevos compatibles.
- `+1.0` cuando un cambio rompe alcance ya aprobado (algo deja de valer).

---

## Reglas de redacción

1. **Español.** El término técnico inevitable se define en el Glosario en la primera uses.
2. **Un requerimiento = una oración = una sola conducta.** Si hay dos verbos, son dos
   requerimientos (prohibido encadenar con "y", "además", "también").
3. **Presente:** "El sistema debe…". Nunca futuro ("podrá", "va a").
4. **Verificable:** si no se te ocurre una prueba que lo confirme o desmienta, todavía no es
   un requerimiento — es una intención.
5. **QUÉ, no CÓMO:** la solución técnica (clases, middlewares, archivos) vive en `conductor/`
   y en los commits, y se referencia en Trazabilidad. Excepción: RNF que fijan una restricción
   técnica como exigencia (ej. "firma digital ed25519").
6. **Números explícitos** donde haya umbrales ("30 días antes", "0 días por defecto").
   Prohibido "pronto", "adecuado", "rápido", "eficiente".
7. **La sección 3 se escribe para alguien que no programa.** Analogías permitidas y
   bienvenidas; sin vergüenza por repetir lo que las secciones formales dicen con precisión.
8. **Cada cambio suma una fila** en Historia de cambios y sube la versión del documento.
9. **Nada de requisitos ocultos:** si una decisión condiciona el módulo (aunque venga de una
   conversación), se registra como RN o queda en el plan de conductor/, pero no se deja implícita.
10. **El documento describe el comportamiento acordado**, no la historia de su construcción:
    esa historia vive en conductor/ y en el historial git (ver Trazabilidad).

---

## Índice de documentos

| Documento | Código | Módulo | Estado | Versión |
|:----------|:-------|:-------|:-------|:--------|
| [REQ_BIPACK_LICENCIAMIENTO_BI.md](REQ_BIPACK_LICENCIAMIENTO_BI.md) | BIPACK | Licenciamiento — BI Pack Premium (Core vs Core+BI) | Implementado | v1.0 |

> Para crear un documento nuevo: copiar la estructura de arriba, elegir un código de módulo no
> usado (ver índice) y agregar la fila correspondiente a esta tabla en el mismo diff.
