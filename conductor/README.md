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

- **PLAN_ALINEACION_BI_FRONTEND_2026-09-18** — ✅ CERRADO 5/5 fases (2026-09-19, rama `feat/bi-alignment` mergeada a `dev`). 19/19 páginas BI en .tsx, 6 features migrados, 14 módulos domain/, ~380 keys i18n `bi.*`, suite 734→1007, index 875 kB. Detalle: commits 7f87012..65b53ed (FE) + purge 7eba2a8.
Addendum §7 (ec4e282) absorbido: regla ≤10 filas server-side para CustomerSellerInsights,
sales-analytics/Dashboard, SaludInventario, PronosticoVentas/Ingresos y OverdueAccounts
queda como deuda puntual.
Reverificación 2026-09-21 (erp-webapp 890c539, addendum §5 de VERIFICACION_POST_CIERRE):
salesAnalyticsMock se borró recién ahí (FE c585f08) — el `7f43e9b` citado antes como
absorbido quedó colgante, sin rama que lo contenga; release note BC-1 hecha
(business_management e044750, CHANGELOG [Unreleased]); fabricaciones residuales de
ProductsCategories y AuditUserActivity eliminadas.
Segunda pasada del mismo día (addendum §5.1): la deuda que quedaba quedó cerrada —
paginación server-side en CustomerSellerInsights/SaludInventario/OverdueAccounts con
nuevo patrón T9 en `by-seller` y `/forecast/inventory` + tope de ventanas de forecast
(BE fab1e0e/d37a957, FE 9d56df6), y migración full de AuditUserActivity (FE 88d5ca0).
Deuda menor de §5.1 cerrada en tercera pasada (addendum §5.1): PronosticoVentas reescrito y
convención ui_labels eliminada end-to-end (FE 69b400c); runbooks de prod consolidados en un
checklist ejecutable único (business_management 7764a22, RUNBOOK_PROD_PENDIENTES_BI_2026-09-21) —
pendiente solo su EJECUCIÓN en prod (requiere acceso: A0 dual-roles, índices CONCURRENTLY,
validación TZ, flip rate-limit), trackeada en la tabla del propio runbook.
