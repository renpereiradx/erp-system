# docs/ — Ingeniería de Requerimientos

Este directorio registra **textualmente los requerimientos** de cada módulo o sistema del ERP
System: *qué debe hacer el sistema y por qué*, escrito para que tanto ingeniería como
personas sin formación técnica (dirección, comerciales, soporte) puedan revisarlo y entenderlo.

Última actualización: 2026-09-22 (auditoría post-implementación: REQ_DEPLOY v0.3)

---

## Qué es y qué NO es este directorio

| Directorio | Pregunta que responde | Audiencia | Ciclo de vida |
|:-----------|:----------------------|:----------|:--------------|
| **`docs/requirements/`** | **¿QUÉ debe hacer el sistema y POR QUÉ?** (requerimientos por módulo) | Todas (híbrido: informativo + formal) | Vivo: se actualiza cuando cambia el alcance |
| **`docs/guides/`** | **¿CÓMO se instala, actualiza y opera una instalación?** (guías operativas paso a paso) | Implementación / soporte (híbrido) | Vivo: se actualiza cuando cambia el procedimiento |
| `conductor/` | ¿CÓMO y CUÁNDO se ejecutó el trabajo? (planes, fases, commits) | Ingeniería | Efímero: el plan se borra al cerrarse |
| Historial git | ¿Qué código exacto se cambió? | Ingeniería | Permanente |

Un requerimiento **no caduca cuando el trabajo termina** (a diferencia de un plan de
`conductor/`): queda como la definición vigente de cómo debe comportarse el módulo, y se
actualiza solo si el alcance cambia.

```
docs/
├── README.md           ← esta convención
├── requirements/       ← REQ_<CÓDIGO>_<NOMBRE>.md — el QUÉ y el POR QUÉ de cada módulo
└── guides/             ← GUIDE_<TEMA>.md — el CÓMO operativa (instalar, actualizar, migrar)
```

## Cuándo se crea un documento

- Un **REQ** (`requirements/`): al planificar un módulo o sistema nuevo (puede nacer como *Borrador* antes de existir código), o al formalizar un módulo existente que nunca tuvo requerimientos escritos. Nunca "post facto" para trabajo trivial (un hotfix no necesita un REQ).
- Una **guía** (`guides/`): al definir un procedimiento operativo que se repetirá en el mundo real (instalar un cliente, actualizar una instalación, migrar de laptop a servidor, conectar una sucursal). No antes de que el procedimiento exista: la guía documenta lo implementado, no lo imaginado.

---

## Estructura obligatoria de cada REQ (en `requirements/`)

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

## Convenciones de las guías (en `guides/`)

Nombre de archivo: `GUIDE_<TEMA>.md`, registrado en el índice de `guides/README.md`.

```markdown
# GUIDE_<TEMA>

## 1. Objetivo
Una frase: qué logra quien complete esta guía.

## 2. Alcance
Qué escenarios cubre y cuáles NO (derivados de un REQ cuando exista).

## 3. Prerrequisitos
Qué debe existir antes de empezar (hardware, accesos, bundle y su versión).

## 4. Pasos
Numerados; cada comando copiable tal cual; variables marcadas como <ASI>.
Cada paso o grupo lleva su verificación ("deberías ver…").

## 5. Verificación final (smoke test)
La lista corta que confirma que todo quedó funcionando.

## 6. Rollback
Cómo volver al estado anterior, paso a paso. Obligatorio si algún paso es destructivo.

## 7. Problemas frecuentes
Síntoma → causa → solución.

## 8. Historia de cambios
| Versión | Fecha | Cambio | Autor |
```

Reglas de las guías:

1. **Todo comando es copiable tal cual** — nada de "ajustá según tu caso" sin decir qué ajustar.
2. **Cada paso tiene su verificación**; si no se puede verificar, no es un paso, es una intención.
3. **Rollback obligatorio** para cualquier procedimiento que toque datos o esquema.
4. **Audiencia híbrida**: arranca con 2-3 frases de contexto no técnico; la parte ejecutable puede ser técnica.
5. La guía describe el procedimiento **tal como está implementado**; si el procedimiento cambia, la guía cambia en el mismo diff.
6. Estado y pendientes de cada guía se registran en el índice de `guides/README.md`, nunca como secciones vacías dentro de la guía.

---

## Índice de documentos

| Documento | Código | Módulo | Estado | Versión |
|:----------|:-------|:-------|:-------|:--------|
| [REQ_BIPACK_LICENCIAMIENTO_BI.md](requirements/REQ_BIPACK_LICENCIAMIENTO_BI.md) | BIPACK | Licenciamiento — BI Pack Premium (Core vs Core+BI) | Implementado · v1.2: compra única sin vencimiento (RN-07) | v1.2 |
| [REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md](requirements/REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md) | DEPLOY | Distribución y despliegue on-premise (perfiles, terminales, sucursales) | Implementado (reservas de sitio: Tailscale/SSH) · auditado v0.3 | v0.3 |
| [REQ_VENTAS.md](requirements/REQ_VENTAS.md) | VENTAS | Ventas — POS, pedidos de mostrador, cuenta corriente, anulaciones | Borrador (backfill módulo existente) | v0.1 |

Índice de guías operativas: [`guides/README.md`](guides/README.md).

> Para crear un documento nuevo: copiar la estructura de arriba, elegir un código de módulo no
> usado (ver índice) y agregar la fila correspondiente a esta tabla en el mismo diff.
