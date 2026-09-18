# conductor/ (workspace-level)

Este directorio **raíz** (no el de `business_management/conductor/` ni `erp-webapp/conductor/`)
centraliza la **coordinación cross-repo** del ERP Vista Bar: planes y fixes que tocan backend y
frontend a la vez.

> **Limpieza 2026-09-11**: los planes/registros de trabajo completado fueron eliminados de los
> tres `conductor/`. Todo queda recuperable en el historial git de cada repo. Aquí viven solo
> planes con trabajo pendiente y el plan activo.

## Contenido

| Archivo | Rol |
|:--------|:----|
| `PLAN_BUSQUEDA_VARIANTES_PLANAS.md` | 📋 **Activo**: variantes como productos independientes en la búsqueda de compras, ventas, pedidos y catálogo. |
| `PLAN_PEDIDOS_MOSTRADOR_VENDEDOR_CAJA.md` | 🔄 FASE 5 pendiente (ver plan). |
| `PLAN_CATALOGO_VENDEDOR.md` | 🔄 FASE 3 opcional (integración con `/pedidos`). |
| `PLAN_MONOROL_DESCUENTOS_CREDITO_CLIENTE.md` | 🔄 A0 prod audit pendiente (deploy blocker). |

> **Cierre 2026-09-17**: `PLAN_AUDIT_BI_INTELIGENCIA_NEGOCIOS_2026-09-14.md` COMPLETADO (5/5
> fases: backend gate duro, auditoría 50 páginas, fixes P0/P1/P2, migración .tsx/FSD + tokens,
> tests FE + H7) y purgado — veredicto, hallazgos y gates en el historial git de este repo y en
> `erp-webapp/conductor/audits/bi-2026-09/` + `erp-webapp/conductor/AUDIT_REACT_PERFORMANCE_BI_FRONTEND.md`
> (H1-H5 de performance quedan como propuesta pendiente de sesión dedicada).

## Convención de limpieza

- Plan/registro cuyo trabajo está completo → se borra (git history es el archivo).
- Quedan: planes con fases pendientes, docs vivos referenciados por AGENTS.md
  (ej. `erp-webapp/conductor/PLAN_TEST_DESIGN_FRONTEND.md`, `business_management/conductor/fases/`)
  y registros de fixes de los últimos días.

## Planes cerrados (en git history)

- **PLAN_ALINEACION_BI_FRONTEND_2026-09-18** — ✅ CERRADO 5/5 fases (2026-09-19, rama `feat/bi-alignment` mergeada a `dev`). 19/19 páginas BI en .tsx, 6 features migrados, 14 módulos domain/, ~380 keys i18n `bi.*`, suite 734→1007, index 875 kB. Detalle: commits 7f87012..65b53ed + purge 7eba2a8.
