# PLAN — Analítica de Compras (backend + frontend)

**Fecha:** 2026-10-22 · **Repos:** `business_management` (rama `dev`) + `erp-webapp` (rama `dev`)
**REQ:** `docs/requirements/REQ_BIPACK_LICENCIAMIENTO_BI.md` v2.4 (RF-BIPACK-031..035, RN-BIPACK-11)

## 1. Objetivo y contexto

Paridad del grupo "Analítica de Compras" con el de Ventas: la FASE 10 del BE
instrumentó ventas completas y compras solo recibió los 2 drill-downs
relacionales (RF-023/024). Este plan construye el módulo agregado de compras —
KPIs, tendencias, desgloses y comparativa — primero en el backend, después las
5 pantallas FE, siempre dentro del pack BI (`biChain("analytics:read",
"purchase-analytics")`).

## 2. Decisiones de borde (aplicadas)

1. **Status**: agregados sobre `status = 'COMPLETED'` (compras recibidas,
   mismo criterio que los drill-downs). Las PENDING viajan APARTE en
   performance (`pending_orders`/`pending_amount`) para no ocultar gasto
   comprometido. Descartado `NOT IN ('CANCELLED')` (criterio ventas):
   divergiría de los drill-downs ya shipped.
2. **Multi-moneda**: agregados en moneda base únicamente (`currency_id IS
   NULL OR = base`, patrón D3 de RF-024); el conteo de excluidas viaja en el
   payload (`excluded_other_currency`) y en metadata de performance; el FE lo
   muestra como banner.
3. **Sin `/dashboard` consolidado en BE**: el dashboard FE compone con
   endpoints atómicos (performance?compare + trends + by-payment-method +
   by-supplier + by-category). Evita el endpoint que en ventas quedó
   sub-usado por el corte a 30 días del consolidado (lección F4).
4. **Anti-duplicación multi-línea desde el día 1** (lección F4 §12.2): dinero
   a nivel orden en CTEs SIN join a detalles; unidades y plata de línea en
   CTEs propias unidas 1:1. Test invariante contra BD real.
5. **`week` = rolling 7 días** (`NewPeriodRolling`), igual que ventas.
6. **Sort whitelists validadas en el handler** (`parseSortParams`, BC-4):
   valor fuera de la lista → 400 (ventas cae silenciosa al default; compras
   endereza el rumbo). Lección F5: alias del SELECT siempre presentes.
7. **Ventanas de compare a medias → 400** (BR-4b): en ventas caen silenciosas
   a modo período.

## 3. Contrato BE (19 rutas GET nuevas, subrouter `/purchase-analytics`)

`performance(+/date-range, ?compare=true)` · `trends(+dr)` · `by-category(+dr)`
· `by-product(+dr)` (paginado+sort: purchases|units|name|last_purchase) ·
`by-supplier(+dr)` (amount|purchases|name|last_purchase) · `by-buyer(+dr)` ·
`by-payment-method(+dr)` · `by-hour(+dr)` · `by-day-of-week(+dr)` · `compare`.

Archivos: `models/purchase_analytics.go` (DTOs), port extendido en
`internal/analytics/port.go`, service en `service_purchase.go`, handlers en
`http_purchase.go`, repo en `internal/platform/postgres/
analytics_purchase_aggregates_repo.go` (Timebase nueva en el constructor),
rutas en `internal/analytics/http/routes.go`, wiring actualizado.

## 4. Pantallas FE (nav "Analítica de Compras": 2 → 7 hojas)

1. `/purchase-analytics/dashboard` — KPI ledger con comparación real, hints
   de pendientes/moneda excluida, serie, mix de pagos, top categorías/proveedores.
2. `/purchase-analytics/products-categories` — tabla por categoría + productos
   paginados/ordenados server-side (`aria-sort`).
3. `/purchase-analytics/insights` — proveedores (sort/paginación + RUC) y
   ranking de compradores.
4. `/purchase-analytics/trends` — serie con granularidad derivada + perfil
   horario/semanal (peak_hour/best_day del payload).
5. `/purchase-analytics/period-comparison` — modo período vs custom, rangos
   inclusivos, variaciones relativas (nunca "pp").

Estructura FSD espejo de F4: `services/bi/purchaseAnalyticsService.ts` ·
`domain/purchase-analytics/` (format reutiliza `domain/sales-analytics/format`,
precedente domain→domain core/rounding) · `features/purchase-analytics/`
(hooks react-query + UI pura; PurchaseKpiLedger feature-local como en
Payables/Profitability) · páginas delgadas · i18n `purchaseAnalytics` es/en
con gate de paridad.

## 5. Estado de ejecución (2026-10-22) — CERRADO

### Backend (commits `dev`: 56778c0 + 92574c3)

- 19 rutas registradas; las 2 relacionales intactas al inicio del subrouter.
- Tests: invariante anti-duplicación multi-línea vs BD real (performance =
  trends = by-supplier = by-payment = compare = SUM(total_amount) canónico),
  pending fuera de los agregados, sorts ASC/DESC completos, handlers con
  fakes (parsing/400s/envelope/compare/fechas a medias).
- Fix display: nombre de comprador deduplicado (`NULLIF(last_name,
  first_name)`) — mismo bug que F2 tuvo con actores del dashboard.
- Gates: gofmt/golangci-lint limpios, `make build` OK, tests OK
  (`tests/integration` falla preexistente por clúster efímero, documentado).
- Smoke curl :5050: 10 endpoints 200 con datos reales; 400s verificados
  (período inválido, sort fuera de whitelist, paginación basura, compare a
  medias, date-range sin fechas); `X-Branch-ID` scoping verificado contra el
  canónico de la sucursal (Gs. 9.229.040 = SUM branch 3 COMPLETED).

### Frontend (commit `dev`: feat(bi) purchase analytics screens)

- 5 pantallas + nav 7 hojas + rutas lazy con `AnalyticsRoute`; i18n es/en.
- Tests: 68 nuevos (dominio 29, servicio 11, hooks 6, 5 páginas, paridad
  i18n, nav actualizado). Suite completa: **365 archivos / 2382 tests OK**;
  `tsc` 0; `pnpm build` OK; `lint:design` limpio; `lint` 0 errores (132
  warnings legacy sin cambios).
- Smoke vivo (Vite :5173 + BE :5050, JWT dev, sucursal JUST STYLE): las 5
  pantallas con datos reales; KPI = canónico de la sucursal; sort server-side
  con `aria-sort`; paginación real (13 items, 2 páginas); deep-link
  `?period=year`; banner de moneda excluida condicionado al payload;
  responsive 390 px sin overflow del body; 0 errores de consola.
- Fix durante el smoke: key i18n `columns.orders` partida en `orders`
  (header) + `ordersWithCount` (texto con var) — el header mostraba
  "{count}" crudo; título de página Productos con key propia.

### Deuda declarada

- Descuentos de compra fuera del alcance (extensión futura si el owner la
  pide; `discount_amount` existe en los detalles).
- Agregados sin código de moneda por fila → FE formatea PYG (moneda base),
  igual que F3/F4; el banner de excluidas informa el alcance.
- `searchableRoutes` sin entradas del grupo (paridad con ventas, que tampoco
  las tiene).

## 6. Versionado

Módulo nuevo visible → MINOR en ambos repos al cerrar el hito (tag sobre
`dev`, decisión del owner). `package.json` FE a sincronizar en el release.
