# PLAN — Auditoría y remediación del módulo BI "Inteligencia de Negocios"

**Fecha**: 2026-09-14 (v2: auditoría profunda backend integrada) · **Tipo**: Cross-repo (backend Go + frontend React) · **Estado**: ✅ **FASE 1 COMPLETA — GATE FINAL SUPERADO (2026-09-16)** — FASE 0 ✅ + 1A ✅ + 1B ✅ + 1C ✅ (T10-T16: 8298583…19a69ea) + 1D ✅ (T17-T18: 251e69b) + 1E ✅ (T19-T23: 79bd447, d372437, 6914c40, 63c7cf4, 892aac3); cierre en `business_management/conductor/AUDIT_BI_BACKEND_2026-09.md`. Recién aquí arranca FASE 2 (frontend)

> **Orden de ejecución (v2)**: FASE 0 → **FASE 1 completa (backend BI, sub-fases 1A-1E)** → FASE 2-5
> (frontend). No se inicia ninguna fase de frontend hasta que el gate de FASE 1 esté en verde:
> las páginas FE auditarán contra una API correcta, rápida y consistente, no contra los bugs
> que hoy tiene el backend.

## 0. Contexto y objetivo

El módulo "Inteligencia de Negocios" del sidebar (9 grupos, ~36 páginas) acumuló deuda de
implementación: páginas que muestran datos mock/hardcodeados como si fueran reales, secciones
inventadas, llamadas a métodos de servicio que no existen, tablas sin paginación y redirecciones
rotas. El backend BI (`internal/analytics`, 11 servicios + 11 repos Postgres, SQL real) está
migrado a la arquitectura hexagonal pero la auditoría profunda (2026-09-14) encontró defectos
de corrección (timezone corrido 3h, compras canceladas en el PyG), de performance (índices
faltantes en las tablas más calientes) y de robustez (errores tipados sin usar, not-found → 500).

**Objetivo**: (1) remediar por completo el backend BI — corrección, robustez, performance,
auditoría activable, tests — y (2) auditar página por página el frontend (con runtime en
browser) hasta que: nada esté hardcodeado, toda interacción anunciada por la UI funcione, las
tablas paginen server-side (≤10 filas visibles) y las redirecciones lleven a rutas existentes
con el permiso correcto. Todo alineado a `erp-webapp/DESIGN.md` y `AGENTS.md` (ambos repos).

**No-escope**: migrar el forecast a ML externo (se mejora la estadística actual, no se agrega
infra de ML), reescribir `internal/analytics` completo, cambiar la librería de gráficos
(recharts se mantiene).

---

## 1. Alcance — inventario verificado

### 1.1 Frontend (erp-webapp)

Menú: `src/layouts/main/navigation.ts:51-171` (gated `analytics:read` a nivel de sección).
Router: `src/App.tsx:298-360, 580-599`.

| # | Grupo | Página | Ruta | Archivo | Migración |
|:--|:------|:-------|:-----|:--------|:----------|
| 1 | Dashboard | Resumen Ejecutivo | `/dashboard` | `src/pages/Dashboard.jsx` | legacy |
| 1 | | KPIs y Rendimiento | `/dashboard/kpis` | `src/pages/DetailedKPIs.jsx` | legacy |
| 1 | | Análisis de Ventas (heatmap) | `/dashboard/sales-heatmap` | `src/pages/SalesHeatmap.jsx` | legacy |
| 1 | | Alertas de Negocio | `/dashboard/alerts` | `src/pages/ConsolidatedAlerts.jsx` | legacy |
| 1 | | Top de Productos | `/dashboard/top-products` | `src/pages/TopProductsOverview.jsx` | legacy |
| 2 | Analítica de Ventas | Dashboard Ejecutivo | `/sales-analytics/dashboard` | `src/pages/sales-analytics/Dashboard.jsx` | legacy |
| 2 | | Productos y Categorías | `/sales-analytics/products-categories` | `.../ProductsCategories.jsx` | legacy |
| 2 | | Insights Clientes/Vend. | `/sales-analytics/insights` | `.../CustomerSellerInsights.jsx` | legacy |
| 2 | | Tendencias y Velocidad | `/sales-analytics/trends-velocity` | `.../TrendsVelocity.jsx` | legacy |
| 2 | | Comparativa Períodos | `/sales-analytics/period-comparison` | `.../PeriodComparison.jsx` | legacy |
| 2 | | Descuentos | `/sales-analytics/discounts` | `.../Discounts.tsx` | ✅ tsx |
| 3 | Pronósticos BI (IA) | Dashboard Pronósticos | `/bi/pronosticos/dashboard` | `src/features/bi-forecasting/.../DashboardPronosticos.jsx` | FSD/legacy-jsx |
| 3 | | Salud Inventario | `/bi/pronosticos/inventario` | `.../SaludInventario.jsx` | FSD/legacy-jsx |
| 3 | | Pronóstico Ventas | `/bi/pronosticos/ventas` | `.../PronosticoVentas.jsx` | FSD/legacy-jsx |
| 3 | | Pronóstico Demanda | `/bi/pronosticos/demanda` | `.../PronosticoDemanda.jsx` | FSD/legacy-jsx |
| 3 | | Pronóstico Ingresos | `/bi/pronosticos/ingresos` | `.../PronosticoIngresos.jsx` | FSD/legacy-jsx |
| 4 | Analítica de Rentabilidad | Dashboard | `/profitability/dashboard` | `src/features/profitability/.../ProfitabilityDashboard.jsx` | FSD/legacy-jsx |
| 4 | | Productos | `/profitability/products` | `.../ProductProfitability.jsx` | FSD/legacy-jsx |
| 4 | | Clientes | `/profitability/customers` | `.../CustomerProfitability.jsx` | FSD/legacy-jsx |
| 4 | | Categorías | `/profitability/categories` | `.../CategoryProfitability.jsx` | FSD/legacy-jsx |
| 4 | | Tendencias | `/profitability/trends` | `.../ProfitabilityTrends.jsx` | FSD/legacy-jsx |
| 4 | | Vendedores | `/profitability/sellers` | `.../SellerProfitability.jsx` | FSD/legacy-jsx |
| 5 | Analítica de Inventario | Dashboard | `/inventory-analytics/dashboard` | `src/pages/InventoryAnalytics/InventoryDashboard.tsx` | ✅ tsx |
| 5 | | Rotación y ABC | `/inventory-analytics/turnover-abc` | `.../InventoryTurnoverABC.tsx` | ✅ tsx |
| 5 | | Stock y Reposición | `/inventory-analytics/stock-levels` | `.../StockLevelsReorder.tsx` | ✅ tsx |
| 5 | | Riesgo de Inventario | `/inventory-analytics/risk` | `.../InventoryRisk.tsx` | ✅ tsx |
| 6 | Cuentas por Cobrar | Dashboard | `/receivables` | `src/pages/ReceivablesDashboard.jsx` | legacy |
| 6 | | Lista Maestra | `/receivables/list` | `src/pages/ReceivablesMasterList.jsx` | legacy |
| 6 | | Detalle | `/receivables/detail/:id` | `src/pages/ReceivableDetail.jsx` | legacy |
| 6 | | Vencidas | `/receivables/overdue` | `src/pages/OverdueAccounts.jsx` | legacy |
| 6 | | Aging Report | `/receivables/aging-report` | `src/pages/AgingReport.jsx` | legacy |
| 6 | | Perfil Crédito Cliente | `/receivables/client-profile/:id` | `src/pages/ClientCreditProfile.jsx` | legacy |
| 7 | Cuentas por Pagar | Dashboard | `/dashboard/payables` | `src/pages/PayablesDashboard.jsx` | legacy |
| 7 | | Facturas | `/payables/invoices` | `src/pages/InvoicesMasterList.jsx` | legacy |
| 7 | | Detalle | `/payables/detail/:id` | `src/pages/PayableDetail.jsx` | legacy |
| 7 | | Proyección Cash Flow | `/payables/cash-flow` | `src/pages/CashFlowProjection.jsx` | legacy |
| 7 | | Aging Report | `/payables/aging-report` | `src/pages/PayablesAgingReport.jsx` | legacy |
| 8 | Reportes Financieros | Resumen Financiero | `/dashboard/financial-summary` | `src/pages/FinancialSummaryDashboard.jsx` | legacy |
| 8 | | Cash Flow Analítico | `/finance/analytical-cash-flow` | `src/pages/CashFlowAnalysisDashboard.jsx` | legacy |
| 8 | | Gestión de Impuestos | `/finance/tax-management` | `src/pages/TaxManagementDashboard.jsx` | legacy |
| 8 | | Inutilización SIFEN | `/finance/sifen-inutilizacion` | `src/features/fiscal/pages/SkippedNumbersPage.tsx` | ✅ tsx |
| 8 | | Ops SIFEN | `/finance/sifen-ops` | `src/features/fiscal/pages/FiscalOpsDashboard.tsx` | ✅ tsx |
| 8 | | Libros Legales | `/finance/legal-books` | `src/pages/LegalBooks.jsx` | legacy |
| 8 | | PyG | `/finance/profit-and-loss` | `src/pages/ProfitAndLoss.jsx` | legacy |
| 9 | Auditoría de Sistema | Dashboard | `/auditoria` | `src/pages/AuditDashboard.jsx` | legacy |
| 9 | | Logs | `/auditoria/logs` | `src/pages/AuditLogs.jsx` | legacy |
| 9 | | Detalle log | `/auditoria/logs/:id` | `src/pages/AuditLogDetail.jsx` | legacy |
| 9 | | Actividad usuario | `/auditoria/usuarios/:id` | `src/pages/AuditUserActivity.tsx` | ✅ tsx |

**Rutas huérfanas** (no están en el sidebar, viven bajo BI): `/bi/inventory/stock-levels`
(`StockManagement.jsx`), `/bi/inventory/risk-analysis` (`InventoryRisk.jsx` de profitability),
`/finance/profitability` (`ProfitabilityAnalysis.jsx`). Se decide en FASE 2: eliminar o re-enlazar.

### 1.2 Backend (business_management)

- Núcleo: `internal/analytics/` — `domain.go` (value object `Period`), `port.go` (10 puertos
  reader), `service.go` (11 servicios, pass-through), `http*.go` (11 handlers), rutas en
  `internal/analytics/http/routes.go` (wiring en `routes/routes.go:79-86`, construcción lazy
  en `internal/platform/wiring/container.go:722-820`).
- Persistencia: `internal/platform/postgres/analytics_*.go` + `audit_repo.go` (11 repos,
  ~11.800 líneas). SQL real; cero mocks en backend.
- Infra disponible (no usada por BI hoy): Redis (`config/redis.go`, `internal/platform/redis/`
  con ProductCache + token blacklist), middleware de observabilidad (log+Prometheus por
  request), rate-limit Redis (solo `/setup/initialize`), ETag middleware (sin aplicar),
  `internal/shared/apperr` (errores tipados con `Kind`→`HTTPStatus`, usado por sale, purchase,
  party… pero NO por analytics).
- BI-adjacent: `internal/sale` (price-changes, caja, counter-orders/metrics, presupuestos),
  `internal/pricing` (cost-trends, margin-alerts), `internal/sifen` (dashboard fiscal).
- Scope de sucursal: todos los handlers llaman `ResolveBIContext`
  (`internal/analytics/branchctx.go` → `identity.ResolveBIContextFromAuth`); no-admin requiere
  branch; admin sin `X-Branch-ID` ve consolidado.
- RBAC por subrouter: `dashboard:read`, `receivables:read`, `payables:read`, `reports:read`,
  `analytics:read`, `audit:read`; financial-reports agrega gate de rol Admin/Buyer.

---

## 2. Hallazgos preliminares (verificados a nivel código, 2026-09-14)

### 2.1 P0 — Funcionalidad rota FE (páginas que pierden secciones o fallan)

| ID | Hallazgo | Evidencia |
|:---|:---------|:----------|
| P0-1 | **CxP: `usePayables.js` llama 6 métodos inexistentes** en `payablesService.ts` (`getPayables`, `getPayableById`, `getTopSuppliers`, `getSchedule`, `getAgingSummary`, `getStatistics`) → TypeError capturado con toast. Afecta: PayablesDashboard (top proveedores + schedule), InvoicesMasterList (tabla completa), InvoiceDetail (página rota), PayablesAgingReport (KPIs DPO en '---') | `src/hooks/usePayables.js:53,144,232,251,270,308` vs `src/services/bi/payablesService.ts` (solo define 5 métodos) |
| P0-2 | **CxC: `receivablesService.getMasterList` no existe** → la tabla de facturas de ReceivablesDashboard y toda ReceivablesMasterList caen al catch | `src/features/receivables/hooks/useReceivablesDashboard.js:82`, `useReceivablesMasterList.js:97` |
| P0-3 | **Alertas con `action_url` muertas**: el backend envía `/products/out-of-stock`, `/analytics/inventory/low-stock`, `/margin-alerts`, `/budgets?status=expiring` — ninguna existe en el router FE (solo `/receivables/overdue` es real) | `analytics_dashboard_repo.go:1369,1387,1408,1429,1450` → `App.tsx` |
| P0-4 | **Chip de cliente navega a `/clientes?search=...` — ruta inexistente** (la real es `/parties`) | `src/pages/ConsolidatedAlerts.jsx:37,447` vs `App.tsx:344` |
| P0-5 | **Auditoría lee una tabla perpetuamente vacía**: `audit_logs` solo se escribe vía `AuditService.LogAction`, que no tiene ningún caller en producción (ni middleware ni handlers) | `internal/analytics/audit.go:85` (grep de callers vacío) |
| P0-6 | CashFlowProjection importa `bankPositions` de mockData ("Santander Corporate", "BBVA", "HSBC") y `useCashFlow.js` fabrica insights si la API no los trae | `CashFlowProjection.jsx:17`, `src/features/cash-flow/data/mockData.js:30-34` |

### 2.2 P1 — Hardcode FE / datos inventados que se muestran como reales

| ID | Hallazgo | Evidencia |
|:---|:---------|:----------|
| P1-1 | **Fallback demo masivo en `useDashboardStore.ts`**: KPIs inventados (average_ticket 350.000, net_margin 32.5…), heatmap con `Math.random()`, top products con profit `revenue*0.3` y "Marca Demo", profitabilityTrends con "Jan 1…Jan 30" literales | `src/store/useDashboardStore.ts:187-501` |
| P1-2 | **Sales Analytics arranca con MOCK y lo mantiene si la API falla**: `MOCK_DASHBOARD` (KPIs falsos 15M, alertas inventadas), `MOCK_BY_CUSTOMER/SELLER` | `sales-analytics/Dashboard.jsx:24-27`, `CustomerSellerInsights.jsx:16-20` |
| P1-3 | **Growth % literales aunque la API responda**: `growth={12}`, `{5.4}`, `{-2.1}`, `{15}`, "Churn rate: 13.9%" | `CustomerSellerInsights.jsx:77-98` |
| P1-4 | **Textos de estado inventados** en TrendsVelocity ("Hora pico: 14:00-15:00") y **max fijo `1000000` "consistent with mock max"** que distorsiona el heatmap real | `TrendsVelocity.jsx:69,117,124,131` |
| P1-5 | **Dos pantallas 100% hardcodeadas** (rutas huérfanas de profitability): StockManagement ("17 SKU Críticos", ₫ 1.840.500.000 fijos) e InventoryRisk (deadStock/stockoutRisk literales) | `StockManagement.jsx:51-116`, `InventoryRisk.jsx:41-108` |
| P1-6 | **Forecast en modo demo nunca llama a la API** si `VITE_USE_DEMO=true` (default de `pnpm dev`) | `biForecastingService.ts:528-535` |
| P1-7 | **"+12.5% vs prev."** hardcodeado en AuditDashboard | `AuditDashboard.jsx:109` |
| P1-8 | **UI decorativa**: chips bloqueados en DetailedKPIs, selects no cableados en SalesHeatmap, botones sin onClick (Exportar/Compartir/Silenciar…), "Índice de Salud" = `turnoverRate*10` con TODO | `DetailedKPIs.jsx:106-144,232-238`, `SalesHeatmap.jsx:24-25`, `sales-analytics/Dashboard.jsx:106,290,385`, `ConsolidatedAlerts.jsx:187-190,459-477` |
| P1-9 | **Código muerto con mocks** (eliminar): `src/components/business-intelligence/receivables/*`, `services/mocks/receivablesMock.js`, `services/auditService.ts` + `mocks/auditMocks.js`, `features/accounts-payable/data/*`, `handlers/bi_helpers.go` BE | grep de imports vacío |

### 2.3 P2 — Tablas sin paginación / filtros server-side

Regla objetivo: **≤10 filas visibles + paginación server-side + filtros**.

**FE sin paginar**: Top Productos (sales-analytics Dashboard), tablas de Insights (clientes y
vendedores), SaludInventario (contador sin controles), PronosticoVentas/Ingresos, stock-levels
y risk de Inventory Analytics (servicio no envía limit/page), OverdueAccounts, AgingReport CxC,
AuditLogs (el BE soporta page/page_size y no se usan). PayablesAgingReport pagina client-side
sobre el array completo.

**BE sin LIMIT** (slices potencialmente grandes): `DiscountLines` (todas las líneas del rango),
`ReceivablesRepo.AgingReport` (todos los clientes), `PayablesRepo.AgingReport`,
`InventoryRepo` ABC/DeadStock/Reorder/StockAging/StockForecast (catálogo completo), y
**`getLimit` sin techo** (`http.go:87-97`: `limit=1000000` pasa; `parsePagination` sí capa a
100 pero solo lo usa audit).

**Referencia buena a replicar**: ProductsCategories, ProductProfitability, LegalBooks
(page_size 50 + filtros SIFEN), List de receivables/payables (LIMIT/OFFSET + COUNT).

### 2.4 P3 — Redirecciones y navegación (verificación runtime en FASE 2)

Confirmado a nivel código: destino existente ✅ (`/dashboard/kpis`, `/ventas`, `/compras`,
`/movimientos-stock`, `/caja-registradora`, `/receivables`, `/receivables/overdue`,
`/pagos-compras`, `/cobros-ventas/:id[/pagos]`, `/movimientos-caja`, `/dashboard/alerts`,
`/productos`) y roto ❌ (`/clientes`×2, action_url backend ×4). Pendiente runtime: ¿"Reponer
Stock" → `/productos` pasa `PermissionGuard products:write` para vendedor? ¿`window.location.href`
en AuditLogs (`:135`, full reload) → `navigate()`?

### 2.5 Auditoría profunda BACKEND (2026-09-14) — alimenta la FASE 1

> Fuente: lectura completa de los 11 repos (~11.800 líneas), services, handlers, wiring,
> migraciones y docker-compose. Cada ítem cita evidencia. IDs `BC-x` (corrección),
> `BR-x` (robustez), `BP-x` (performance/arquitectura), `BQ-x` (calidad/tests).

#### 2.5.1 Corrección de datos — BC

| ID | Severidad | Hallazgo | Evidencia |
|:---|:----------|:---------|:----------|
| BC-1 | **CRÍTICO** | **Timezone: el backend corre en UTC y todas las métricas diarias están corridas 3 horas.** Solo el contenedor frontend tiene `TZ=America/Asuncion` (docker-compose.yml:91); el backend no define TZ ni hace `LoadLocation` (grep vacío en `cmd/`+`internal/`). `Period.Resolve` usa `now.Location()` (domain.go:94-123) y todos los repos pasan `time.Now()` crudo. "Hoy" arranza 00:00 UTC = 21:00 del día anterior en Asunción. Afecta: summary/kpis/trends de dashboard, by-hour, heatmap (`EXTRACT(HOUR/DOW FROM sale_date)` sobre hora UTC), by-day, ledgers IVA (MonthlyVAT, AnnualTaxSummary), inventory aging, audit trends | domain.go:94-123, `analytics_dashboard_repo.go:27`, `analytics_sales_repo.go:28,838,891,946`, `analytics_financial_repo.go:27,431-433,827-829`, docker-compose.yml:40-64 vs :91 — **F0 (2026-09-14) corrige la premisa: la dev DB escribe en Asuncion (el escritor es `NOW()` de PL/pgSQL con `TimeZone` de sesión = Asuncion); el shift 3h aplica a deploys con proceso/DB en UTC. Resuelto en T1 con `Timebase` config-driven.** |
| BC-2 | **CRÍTICO** | **Índices faltantes en las tablas más calientes del BI**: no existe `sales_orders(sale_date)` standalone (solo el compuesto `(branch_id, sale_date DESC)`, que el patrón `($n IS NULL OR branch_id = $n)` no garantiza usar); **no existe `sales_order_details(sales_order_id)`** (todo el BI de productos joinea por ese FK: decenas de sitios); no existe `purchase_orders(order_date)` ni `purchase_order_details(purchase_order_id)`; no existe `budget_orders(created_at)` (el DDL indexa `budget_date` pero BI filtra por `created_at`) | migraciones + `backups/full_schema.sql` vs `analytics_dashboard_repo.go:42-43,862-864`, `analytics_sales_repo.go:75,205`, `analytics_financial_repo.go:608,640` |
| BC-3 | **CRÍTICO** | **JOIN 1:N con `products.stock` en scope consolidado del forecast de inventario**: un producto con stock en N sucursales aparece N veces; `LIMIT 100` trunca arbitrario y el `SUM` de stock se contabiliza mal | `analytics_forecast_repo.go:455-462` |
| BC-4 | **ALTO** | **SQLi latente**: `filter.SortOrder` se interpola en ORDER BY sin whitelist en receivables y payables List. Hoy el handler no bindea `sort_order` (no explotable por HTTP), pero es defensa-en-profundidad rota — y la FASE 3-FE planea exponer orden | `analytics_receivables_repo.go:286-300`, `analytics_payables_repo.go:282-294`, `http_receivables.go`/`http_payables.go` sin bind |
| BC-5 | **ALTO** | **IncomeStatement cuenta compras CANCELLED**: `purchases_data` no filtra `po.status NOT IN ('CANCELLED')` (las ventas sí lo filtran) → PyG inflado | `analytics_financial_repo.go:63-71` vs `:51` |
| BC-6 | **ALTO** | **Summary de List escoped a la página**: los totales (TotalAmount/TotalPending/TotalPaid) de receivables/payables se acumulan solo sobre las filas de la página actual, no del result set completo | `analytics_receivables_repo.go:448-468`, `analytics_payables_repo.go:460-480` |
| BC-7 | **ALTO** | **`repeat_customers` cruza sucursales** (subquery sin branch filter mientras el resto de la query está scopeada) y `getCustomerKPIs` mezcla scopes (new/total customers sobre parties sin branch vs active con branch) | `analytics_dashboard_repo.go:843-855, 1006-1028` |
| BC-8 | **MEDIO** | **Pagos de ventas CANCELLED en RecentActivity** (no filtra `so.status`) | `analytics_dashboard_repo.go:546-551` |
| BC-9 | **MEDIO** | **LTV full-history scan**: `lifetime_value` de ByCustomer escanea toda la historia de ventas sin rango ni cota | `analytics_sales_repo.go:649-655` |
| BC-10 | **MEDIO** | **Flujo de caja aproximado**: usa `payment_method_id='cash'` como proxy de cobro (ignora `sale_payments` parciales/múltiples); `cash_from_receivables` solo con status='PAID'; `beginning_cash` solo cuenta cajas OPEN | `analytics_financial_repo.go:229-239, 860-876` |
| BC-11 | **MEDIO** | **`StatisticsDateRange` receivables**: total_collected omite cobranzas de ventas previas al período | `analytics_receivables_repo.go:929-936` |
| BC-12 | **BAJO** | Labels de fecha sin año (`'DD Mon'`, `'Semana' IW`) que fusionan/repiten en rangos >1 año; ORDER BY sin tie-breaker → páginas inestables; `COALESCE(get_cost_for_variant(...), unit_price*0.7)` inventa costo con margen 30%; `TopCustomers` = top 5 de la página actual | `analytics_sales_repo.go:191-192`, `analytics_financial_repo.go:531,645`, `analytics_dashboard_repo.go:913,960`, `analytics_profitability_repo.go:577-583` |

#### 2.5.2 Robustez de API — BR

| ID | Severidad | Hallazgo | Evidencia |
|:---|:----------|:---------|:----------|
| BR-1 | **ALTO** | **analytics no usa `apperr`** (Kind NotFound/InvalidInput/…→HTTPStatus; usado por sale/purchase/party): repos devuelven `fmt.Errorf` crudo y **todo error de servicio llega como 500**, incluidos not-found de `ByID` payables/receivables (deberían ser 404). `respondError` envía `err.Error()` crudo al cliente (posible leak de SQL) | `internal/shared/apperr/errors.go` sin referencias en `internal/analytics/`; `http_payables.go:83-85`, `http_receivables.go:78-80`, `http.go:121-126` |
| BR-2 | **ALTO** | **Not-found → 500**: `sql.ErrNoRows` se convierte en `fmt.Errorf` en payables/receivables ByID | `analytics_payables_repo.go:536-538`, `analytics_receivables_repo.go:524-526` (prorrateo y audit sí lo hacen bien) |
| BR-3 | **MEDIA-ALTA** | **Cero timeouts en queries BI**: ctx propagado correctamente, pero ningún `context.WithTimeout` en analytics — un query lento retiene conexión del pool hasta el WriteTimeout de 60s del server | grep `WithTimeout` en `internal/` = solo documents+cleanup |
| BR-4 | **MEDIO** | **Errores silenciados**: (a) sub-summaries de Dashboard `Summary`/`KPIs` con `if err == nil && x != nil` → fallo parcial = ceros sin log ni aviso; (b) `page, _ := strconv.Atoi` en 10+ handlers → valor inválido = 0 silencioso en vez de 400; (c) `data, _ := json.Marshal` en audit | `analytics_dashboard_repo.go:1171-1232`, `http.go:316-406`, `http_payables.go:97-115`, `http_profitability.go:74-139`, `audit.go:106-114` |
| BR-5 | **MEDIO** | **Parseo divergente**: 2 parsers de period (`getPeriod` sin validación default month — 39 call sites; `parsePeriod` validado default today — solo audit) y 3 patrones de paginación (crudo / `parsePagination` cap 100 / `ledgerPagination` sin cap). Date-range de 4 fechas duplicado; `days=30` triplicado; `branch_id` query solo lo parsea discounts | `http.go:78-84,129-145,148-161,674-700`, `http_financial.go:187-197,401-425`, `http_payables.go:147-150` |
| BR-6 | **MEDIO** | **Catálogo de permisos desincronizado del gate real**: el gating está hardcoded por subrouter en `internal/analytics/http/routes.go`; `config/module_permissions.go` (que solo consumen tests) no tiene entradas para `/vat-prorrateo` ni `/sales-analytics/discounts` | `config/module_permissions.go:11-65`, `routes.go:108,119` |

#### 2.5.3 Performance y arquitectura — BP

| ID | Severidad | Hallazgo | Evidencia |
|:---|:----------|:---------|:----------|
| BP-1 | **ALTO** | **Fan-out secuencial**: `DashboardRepo.Summary` ejecuta 7+ sub-queries en serie (KPIs ídem con 5 grupos); `Comparison` ejecuta la misma query 2 veces; `TopBottomProducts` 4 veces | `analytics_dashboard_repo.go:1164+`, `analytics_sales_repo.go:1157-1168, 478-512` |
| BP-2 | **MEDIO** | **Cero cache en BI**: cada request (incluido `/dashboard/summary` con 7 queries) golpea Postgres. Redis ya existe (ProductCache con singleflight+fail-open, token blacklist, permisos) — patrón a replicar | `grep cache internal/analytics/` = 0; `internal/platform/redis/product_cache.go` |
| BP-3 | **ALTO** | **Duplicación masiva de SQL**: el bloque `paid` (SUM de pagos por orden) está copiado ~30 veces (12× receivables, 10-13× payables, dashboard, financial); el par revenue/cost con `get_cost_for_variant` ~30 veces; el switch granularity→`to_char/DATE_TRUNC` 4 veces; **los ledgers IVA re-implementan en Go 4 vistas que ya existen** (`v_sales_ledger_sifen`, `v_purchases_ledger_sifen`, `v_monthly_vat_liquidation`) | ver detalle en auditoría; `migrations/20260313155000` |
| BP-4 | **ALTO** | **Audit scope = OR de 11 EXISTS correlacionados** inyectado por `strings.Replace` en 10+ queries: caro en runtime y frágil (un cambio de texto del SQL base rompe el scope silenciosamente) | `audit_repo.go:28-92, 506-1183` |
| BP-5 | **MEDIO** | **Queries non-sargable / pesadas**: `EXTRACT(DAY FROM (NOW()-sale_date)) >= $n`; `order_date + INTERVAL '30 days' < NOW()`; `cdc ILIKE '%…%'`; `DATE(reserve_date)=` ; subqueries correlacionadas por fila en StockAging (3/producto), FinancialOverview, ABC (LATERAL); `fiscal_documents` LATERAL por fila en summaries | `analytics_receivables_repo.go:350`, `analytics_payables_repo.go:310-327`, `analytics_financial_repo.go:470-526, 878-895`, `analytics_inventory_repo.go:236,369` |
| BP-6 | **MEDIO** | **Sin rate-limit ni ETag en BI**: queries agregadas pesadas sin protección; el middleware ETag existe y no está aplicado | `middleware/rate_limit.go` (solo /setup), `middleware/etag.go` sin uso |
| BP-7 | **MEDIO** | **Métodos gigantes** (top: `ProductProfitabilityDateRange` 267 líneas, `PayablesRepo.List` 219, `ReceivablesRepo.List` 202, `GetAuditLogs` 196) y SQL one-line de ~1500 chars en inventory | `analytics_profitability_repo.go:189-450`, `analytics_inventory_repo.go:124,174,198,532` |
| BP-8 | **MEDIO** | **Scope branch por el patrón `($n::int IS NULL OR branch_id = $n)`**: en planes genéricos Postgres no puede usar el índice compuesto — frágil; recomendado `branch_id = COALESCE($n, branch_id)` o query dual decidida en Go | patrón en todos los repos |

#### 2.5.4 Calidad y tests — BQ

| ID | Severidad | Hallazgo | Evidencia |
|:---|:----------|:---------|:----------|
| BQ-1 | **ALTA** | **Cobertura de tests desigual**: `go test ./internal/analytics/...` OK hoy, pero 4/11 servicios con tests parciales (1 handler de N) y **7 sin ningún test** (Inventory, Receivables-handlers, Profitability, Audit×17 handlers, Discounts-handlers); `analytics/http` (rutas+gates) sin tests; **los 11 repos Postgres sin tests**; 1 solo integration test BI (`tests/integration/bi_variant_integrity_test.go`) | `internal/analytics/*_test.go` |
| BQ-2 | **MEDIO** | **Contratos sin tipar**: 9 métodos de repo devuelven `map[string]interface{}` (GetLowStock/GetOutOfStock/GetNegativeMargin/GetExpiringBudgets, FinancialHealthScore, ComparePeriods, AnalyzeClientRisk, AnalyzePaymentCapacity, SupplierAnalysis) + `Details map[string]interface{}` en DashboardSummary/KPIs | `analytics_dashboard_repo.go:611-773`, `analytics_financial_repo.go:1038,1109`, etc. |
| BQ-3 | **MEDIO** | **Deuda de pureza del dominio**: domain.go declara pureza pero aliasa 4 tipos de `models/` y todo `port.go` firma con `models.*`; `discounts.go` tiene tags `json:`; `audit.go:41-64` mantiene un `getPeriodDates` propio (la "implementación 14" que `Period` decía reemplazar) | `domain.go:5-6,24-52`, `audit.go:41-64` |
| BQ-4 | **BAJO** | **Sin logging interno BI**: ni services ni repos loguean; un error de query solo deja la línea genérica del middleware. Sin métricas de duración de queries ni pprof | grep logging en `internal/analytics/` = 0 |
| BQ-5 | **BAJO** | **Forecast sin ML y con métodos fantasma**: LINEAR/MOVING_AVG implementados (OLS con R²/MAE/MAPE/RMSE, media móvil 3/6); EXPONENTIAL/SEASONAL declarados en `models/forecast.go:20-23` pero el switch los degrada a linear **silenciosamente**; bandas fijas ±15%; confidence cap 95%; seasonality naive (umbral 0.3) | `analytics_forecast_repo.go:29-114, 105-106, 248, 684-694` |
| BQ-6 | **BAJO** | **Dual-mount `/forecast` + `/api/v1/forecast`** (deuda D10-2) y `handlers/bi_helpers.go` muerto | `routes.go:231-244` |

### 2.6 Transversal

- **Tests FE: cero** para las ~36 páginas BI. Cada fix de FASE 3 lleva su test según
  `erp-webapp/conductor/PLAN_TEST_DESIGN_FRONTEND.md`.
- **Data sanity en dev DB**: ✅ RESUELTO 2026-09-14 — las ~85 ventas CANCELLED de pruebas fue
  purgadas con backup (`dev_purge_backup_20260914`); 62 ventas reales permanecen.
- **i18n**: todo string nuevo pasa por `src/lib/i18n.js`.
- **Recharts** es la librería del área — se mantiene.

---

## 3. Protocolo de auditoría por página FE (el corazón de la FASE 2)

Por cada página del inventario 1.1, ejecutar y documentar en la ficha de la página:

1. **Screenshot completo**: navegar con browser (skill `browser-use:web-gui-tester`),
   capturar full-page; si el screenshot automático no cubre todo, scroll manual y captura por
   tramos. Guardar en `erp-webapp/conductor/audits/bi-2026-09/screenshots/<ruta>.png`.
2. **Trazabilidad de cada card/sección**: para cada bloque visible (KPI, tabla, gráfico,
   insight, alerta): ¿desde dónde consume? (store → service → endpoint → repo SQL) ¿El endpoint
   existe y devuelve lo que la UI asume? ¿Hay contrato roto?
3. **Comprensión de usuario**: ¿un usuario de negocio entiende el número sin contexto?
   (unidades vs moneda, %-vs-pp, períodos comparados). Anotar ambigüedades.
4. **Afordancias reales**: todo lo que parece clicable se prueba en runtime y se registra.
5. **Hardcode scan**: grep dirigido página+store+hook+service: arrays literales, `MOCK_`,
   `mockData`, `Math.random`, porcentajes/fechas literales, fallbacks fabricados.
6. **Tabla**: contar filas renderizadas; si >10 → fix con paginación server-side + filtros.
7. **Redirecciones**: cada `navigate`/`Link`/`action_url` → ¿existe la ruta? ¿el rol tiene
   permiso en el destino? ¿preserva contexto?
8. **Estados de datos**: loading/empty/error presentes (receta DESIGN.md §6.7).
9. **DESIGN.md checklist** (§10) + anti-patrones (§8).
10. **AGENTS.md**: i18n, lógica fuera de componentes, performance
    (`vercel-react-best-practices` + reporte en `conductor/`).

**Ficha por página** en `erp-webapp/conductor/audits/bi-2026-09/pages/<Grupo>-<Pagina>.md`.

---

## 4. Fases de trabajo

### FASE 0 — Baseline y evidencia (0.5 día)

- [x] Entorno FE con `.env.api` (VITE_USE_DEMO=**false**); BE vivo en `:5050`; smoke dev-JWT
      (HS256 con JWT_SECRET local, `session_id:0`) de los 11 grupos de endpoints. ✅ 2026-09-14:
      `.env.api` + `pnpm dev:api` ya configurados; smoke completo registrado en el doc T1.
- [x] **Baseline de performance BE**: capturar `EXPLAIN (ANALYZE, BUFFERS)` de las queries hot
      (dashboard summary/kpis/trends, by-hour, ledgers, stock-levels) ANTES de índices — para
      comparar después de BC-2. ✅ 2026-09-14: 11 queries capturadas en DB post-purge —
      `business_management/conductor/FASE0_BASELINE_EXPLAIN_2026-09-14.md` (29 Seq Scans en el
      conjunto, sin índice standalone en `sales_orders(sale_date)` ni
      `sales_order_details(sales_order_id)` — confirma BC-2; script regenerable para el delta
      post-T2). **Hallazgo colateral corregido en BE `f5b1a38`**: la extracción de las queries
      reales destapó más drift del port legacy en receivables (`c.contact` → `contact_info`
      jsonb; `EXTRACT(DAY FROM date-date)` sobre integer) — el grupo de detalle CxC estaba
      roto (List, ByID, client risk/detail); corregido y smokeado en 200 con datos.
- [x] **Auditoría de zona horaria de los datos** (insumo de T1) — ✅ 2026-09-14: **los naive
      timestamps de la dev DB están en America/Asuncion, NO en UTC** (la `TimeZone` de sesión de
      la DB es Asuncion y `sale_date` lo escribe `NOW()` dentro de la función PL/pgSQL
      `process_sale_with_reserve`; logins y ventas en horario laboral local; el proceso backend
      arrancó 15:03 ↔ sesión guardada 15:03:53). El supuesto "backend corre UTC" de BC-1 solo
      aplica a deploys docker (proceso UTC) o DB con `TimeZone=UTC` — dev es correcta hoy por
      coincidencia de zonas (host -03 ≡ Asuncion). Detalle: `business_management/conductor/
      AUDIT_BI_T1_TIMEZONE_2026-09-14.md`.
- [x] Estado de la dev DB (ventas CANCELLED de prueba) + propuesta de purge (confirmación del
      usuario). ✅ 2026-09-14: **purge ejecutado con autorización del usuario** — 85 ventas
      CANCELLED (2025-10..2026-09) + 93 detalles + 39 pagos + 2 counter_orders eliminados en
      transacción; backup completo en schema `dev_purge_backup_20260914` (restaurable); 1
      cash_movement con referencia blanda a venta borrada se conserva y queda anotado. Quedan
      62 ventas reales (45 PAID / 3 PARTIAL / 14 PENDING); integridad verificada, 0 huérfanos.
- [x] Crear `erp-webapp/conductor/audits/bi-2026-09/`. ✅ 2026-09-14 (scaffold con README del
      protocolo §3; commit FE `9cc2509`).

### FASE 1 — **BACKEND BI COMPLETO** (~3 semanas; gate duro antes del frontend)

> Objetivo: una API BI correcta (números que el negocio puede firmar), robusta (errores
> honestos), rápida (índices + cache) y testeada. Cada tarea cita el hallazgo que resuelve.

#### 1A — Corrección crítica (4-5 días)

- [x] **T1 · Timezone America/Asuncion (BC-1)** — ✅ 2026-09-14, opción no destructiva
      **ajustada por la auditoría F0** (storage = Asuncion en dev, no UTC): `Timebase` en
      `internal/analytics/tz.go` con dos zonas por config (`BI_TIMEZONE` business default
      America/Asuncion; `BI_STORAGE_TIMEZONE` storage default = business). Boundaries resueltos
      en reloj de negocio y renderizados al wall clock de storage; `TZExpr(col)` envuelve las
      proyecciones wall-clock (heatmap/by-hour/dow/trends/audit-trends/VAT mensual/forecast
      histórico/movimientos diarios) y es **identidad byte-idéntica cuando las zonas coinciden**
      (dev: SQL y EXPLAIN sin cambios). `TZ=America/Asuncion` del proceso deja de importar.
      `getPeriodDates` de audit.go reemplazado por `Period`+`Timebase` (avanza BQ-3). Guard de
      startup `bi_storage_timezone_mismatch` + runbook de validación prod en el doc. Gates:
      go test 36 OK / lint / build / smoke dev-JWT de los 11 grupos (2×500 = drift SQL
      pre-existente, ver hallazgos nuevos en el doc T1).
- [x] **T2 · Índices (BC-2)** — ✅ 2026-09-14 (BE `884231b`, migración
      `20260914184232_bi_hot_indexes_bc2` aplicada en dev): `sales_orders(sale_date)`,
      `sales_order_details(sales_order_id)`, `purchase_orders(order_date)`,
      `purchase_order_details(purchase_order_id)`, `budget_orders(created_at)` + `ANALYZE`;
      `IF NOT EXISTS` para que el runbook prod (`CREATE INDEX CONCURRENTLY` fuera de tx +
      `migrate-force`) la deje como no-op — runbook en la cabecera de la migración y en
      `business_management/conductor/FASE1_T2_INDICES_2026-09-14.md`. Delta EXPLAIN honesto: en
      dev (62 ventas) el planner mantiene Seq Scan (correcto a esa escala); usabilidad probada
      con `enable_seqscan=off` (Bitmap Index Scan sobre `sale_date`, 7 vs 14 buffers; FK de
      detalles usado en el join del VAT). La medición real de planes va en prod. Colateral:
      `migrate-verify-gaps` detectó checksum vencido de `20260914125728` (price-history) —
      verificado con diff de `pg_get_functiondef` que dev == archivo commiteado, y re-sellado.
- [x] **T3 · Whitelist de orden (BC-4)** — ✅ 2026-09-14 (BE `514d856`): `parseSortParams`
      en `internal/analytics/http.go` bindea `sort_by`/`sort_order` con whitelist por endpoint y
      **400** listando los valores permitidos (activa el orden server-side para FASE 3-FE:
      receivables `date|amount|client|days_overdue` default `date desc`; payables
      `due_date|date|amount|supplier|priority` default `due_date asc`); defensa en profundidad en
      los 2 repos: `SortOrder` normalizado a `ASC`/`DESC` canónicos antes de interpolar.
      Barrido del resto del contexto — ya seguro sin cambios: inventory/sales/profitability
      (`validSortFields`/`validSorts`), dashboard TopProducts y audit (switches cerrados).
      Tests table-driven (`TestParseSortParams`); smoke: válidos 200, payloads de inyección 400.
- [x] **T4 · Bugs de corrección**: ✅ 2026-09-14 (BE `fca817b`, reporte
  `business_management/conductor/FASE1_T4_CORRECCIONES_2026-09-14.md`): forecast stock JOIN 1:N
  pre-agregado por producto (BC-3); compras CANCELLED fuera del IncomeStatement (BC-5); summary
  global de List con `SUM() OVER ()` (BC-6); repeat_customers con scope de branch + customer
  KPIs consistentes — `new_customers` redefinido a "primera compra del período" branch-scoped,
  misma definición que el summary de ByCustomer (BC-7); RecentActivity filtra `so.status`
  (BC-8); LTV acotado as-of fin de período con `sale_date < $2` (BC-9); tie-breaker
  determinista en todo ORDER BY paginado — CxC/CxP List, TopProducts, ledgers, profitability,
  audit (BC-12). Colateral del smoke: drift `parties.ruc`→`tax_id` en ByCustomer (500 crónico
  enmascarado por el mock del FE, P1-2; misma clase que f5b1a38). Verificado: smoke dev-JWT 200
  en todos los endpoints, invarianza del summary entre page_size 2 y 100 (API real), orden
  estable en requests idénticos, EXPLAIN delta de las 6 queries (sub-ms en dev).
  ~~hallazgos del smoke T1~~ → ✅ **resueltos aparte en BE `363393d` (2026-09-14)**: drift de
  `auditBranchScopeCondition` (`sales_payments`→`sale_payments` + PK `payment_id`,
  `reserve_orders`→`reserves`) y aliases faltantes en las 4 queries de turnover (Overview
  además mostraba rotación en cero silencioso).
- [x] **T5 · action_url reales (P0-3)**: ✅ 2026-09-14 (BE `d2d3607`, reporte
  `business_management/conductor/FASE1_T5_ALERT_ACTION_URLS_2026-09-14.md`):
  `internal/analytics/alert_routes.go` con `AlertActionURLs` (key→ruta) +
  `CanonicalFEAlertRoutes` (contrato compartido); repo deja de hardcodear. Destinos: inventario
  → `/inventory-analytics/stock-levels` (se descarta `/productos`: gated `products:write` =
  muro de permisos al click), margen → `/profitability/products`, vencidas mantiene
  `/receivables/overdue`, presupuestos → `/comercial/presupuestos`. 3 tests de contrato
  (catálogo ⊆ canónica, formato, no-vacíos). Smoke: `/dashboard/alerts` 200 con las URLs
  nuevas en las 2 alertas vivas de dev; 5/5 rutas verificadas contra `App.tsx`. Espejo TS del
  FE + test de router: FASE 3.
- Gates 1A: ✅ `go test ./...` + `golangci-lint run` + `make build` + smoke dev-JWT de los
  endpoints corregidos + EXPLAIN comparado (T2/T4) documentado en `conductor/` del BE.
  **Sub-fase 1A COMPLETA (T1-T5).**

#### 1B — Robustez de API (3 días)

- [x] **T6 · apperr en analytics (BR-1, BR-2)**: ✅ 2026-09-14 (BE `3cc04ab`, reporte
  `business_management/conductor/FASE1_T6_APPERR_2026-09-14.md`): `respondError` escribe el
  envelope unificado del middleware auth (`{success:false, error:{code,message}}`; el
  `toApiError` del FE ya lo parsea) y **sanitiza todo mensaje 5xx** a genérico; nuevo
  `respondAppError(w,r,err)` mapea `apperr` vía `HTTPStatus()` y loguea el error completo con
  request_id — los **132 call sites** `respondError(w,500,err.Error())` de los 11 http*.go
  convertidos mecánicamente (cero leak). 4 not-found tipados (payables/receivables ByID,
  supplier/client detail) → **404** con mensaje en español; audit/prorrateo ya lo hacían bien.
  Tests de mapeo (Kind→status, unwrap, 500 sin leak). Smoke: ByID falso 500→404, sort_by
  inválido 400 con `BAD_REQUEST`.
- [x] **T7 · Timeouts + paralelismo (BR-3, BP-1)**: ✅ 2026-09-14 (BE `8f4d2c3`, reporte
  `business_management/conductor/FASE1_T7_TIMEOUTS_PARALELISMO_2026-09-14.md`):
  `withQueryDeadline` instala `WithTimeout(15s)` en los 12 subrouters BI (por subrouter a
  propósito: RegisterRoutes recibe el router raíz compartido); `Summary` (7), `KPIs` (5) y
  `Trends` (4) en `errgroup` — Summary/KPIs preservan ceros-ante-parcial pero dejan de ser
  silenciosos (log; metadata.partial = T8); `Comparison` 2→1 query (CTE periods, A/B
  idéntico) y `TopBottomProducts` 4→1 (CTE materializada + 4 window rankings + tie-breaker
  product_id; el único delta vs viejo es el orden dentro de empates = la inestabilidad
  BC-12). Diff semántico pre/post refactor de summary/kpis/compare/topbottom: IDENTICAL.
- [x] **T8 · Errores parciales honestos (BR-4a)**: ✅ 2026-09-14 (BE `ea8de73`, reporte
  `business_management/conductor/FASE1_T8_METADATA_PARTIAL_2026-09-14.md`): payloads
  `DashboardSummary`/`DashboardKPIs` cargan `partialErrors` (no serializado); el repo lo
  llena con los fallos del fan-out T7 recolectados con mutex (`collect` en los 12 sitios —
  Wait solo da el primer error) y `respondSuccess` promueve a `metadata.partial: true` +
  `partial_errors:[...]` vía interface `partialReporter` (genérico). Respuesta sigue 200 —
  banner FE en FASE 3. Extra BR-4a(c): `json.Marshal` silenciosos de audit.go ahora
  propagan error. Tests de promoción; camino sano E2E sin clave `partial`.
- [x] **T9 · Parseo unificado (BR-5, P2)**: ✅ 2026-09-15 (BE `425e534`, reporte
  `business_management/conductor/FASE1_T9_PARSEO_UNIFICADO_2026-09-14.md`):
  `parsePeriodQuery` único (validado, default month, 400 con valores permitidos; reemplaza
  getPeriod ×39 + parsePeriod de audit ×5 con sus 5 bloques muertos);
  `parsePaginationErr(defaultSize)` única (cap 100 clamp, presente-inválido → 400; reemplaza
  15 sitios crudos + 2 parsePagination silenciosos + ledgerPagination sin cap, ambos viejos
  eliminados); `getLimit` clampea a 100; `parseDateRange` con envelope T6. P2: AgingReport
  CxC/CxP paginados con `COUNT(*) OVER()`/`SUM OVER()` → totales GLOBALES verificados por API
  (page_size 2 ≡ 100 en totales; lección BC-6 aplicada); `DiscountLines` LIMIT opcional +
  `metadata.total_count`; CollectionReminders ya tenía LIMIT 50. Campos aditivos para el FE.
  **Gates 1B ✅ (tests de mapeo T6 + parseo T9, go test, lint, build, smoke) — SUB-FASE 1B
  COMPLETA (T6-T9).**
- Gates 1B: ídem 1A + tests de mapeo de errores (404/400/500) y de parseo.

#### 1C — Performance y arquitectura (4-5 días)

- [x] **T10 · Cache BI (BP-2)**: ✅ 2026-09-15 (BE `8298583`, reporte
  `business_management/conductor/FASE1_T10_CACHE_BI_2026-09-15.md`): puerto driven `CachePort`
  en analytics (solo read-through; sin invalidación activa — el TTL es la estrategia de
  refresco) + adapter `redis.BICache` reutilizando el client de `config/redis.go` (fail-open +
  singleflight del patrón ProductCache, namespace propio `v1:bi:*`); clave =
  endpoint+period+day-bucket+branch+hash(filters); TTL 60s para dashboard summary/kpis y 120s
  para overviews de receivables/payables (solo GET — verificado por call sites); envelope
  `biEnvelope` preserva el `partialErrors` no exportado de T8 en el round-trip JSON (banner
  `metadata.partial` sobrevive a los hits). La vista materializada del summary con refresh
  event-driven (patrón `mv_product_catalog`) fue evaluada y **descartada por ahora**:
  granularidad sin branch/período, ~8 triggers en tablas calientes y modo de fallo peor (worker
  muerto = stale indefinido); criterio de revisit documentado (miss p95 > 500ms en prod).
  Smoke dev-JWT: 4 endpoints miss/hit 200 con `data` IDENTICAL (único delta `generated_at`
  por-request), claves con TTL exacto en Redis, 0 warnings. Colateral: panic preexistente en
  `TestParsePeriodQuery` de T9 (espacio crudo en el target de httptest mataba el paquete
  analytics) arreglado con `url.QueryEscape` + gofmt de `http_financial.go`.
- [x] **T11 · Deduplicación SQL (BP-3)**: ✅ 2026-09-15 (BE `8b46bcf`, migración
  `20260915144941`, reporte `business_management/conductor/FASE1_T11_DEDUP_SQL_2026-09-15.md`):
  helpers Go con texto SQL canónico + **golden tests** que fijan la salida al texto
  pre-refactor (reemplazo provablemente idéntico): `salePaymentsPaidJoin`/`purchasePaymentsPaidJoin`
  reemplazan las 29 copias del bloque `paid` (3 variantes: branch `$N`, sin branch —el dashboard
  NUNCA filtró pagos por branch, preservado—, y rango payment_date DSO/DPO);
  `sdRevenue`/`sdCost`/`sdProfitSum`/`coalesce0` reemplazan 62 expresiones del par
  revenue/cost con las 3 políticas de coste NULL documentadas (skip / plegar a 0 / fallback
  0.7). **Hallazgo mayor que el plan:** las queries a mano de libros IVA sobredeclaraban el IVA
  — `bruto × rate/100` sobre precio CON-IVA en vez de `rate/(100+rate)`; las vistas SIFEN ya
  traían la base legal corregida (auditoría IVA 2026-08-19) pero nadie en Go las leía. Los
  libros (`VATReport`/`SalesLedger`/`PurchaseLedger`/`getMonthlyVAT`) ahora leen las vistas:
  migración con columnas nuevas aditivas (branch, fiscal FE5.1, payment_method, buckets brutos
  con-IVA, gross_lines/gross_amount), JOIN a detalles INNER y `NOT IN ('CANCELLED')` (corrige el
  Form 120: 6 compras CANCELLED sumaban IVA crédito en dev); down restaura 20260820152422
  verbatim. **Test A/B de integración** congela las queries viejas y verifica: counts/brutos/
  exentos/entries idénticos ±0.01, ratio IVA logueado 10/11 y 5/105 exactos; repo 1:1 con la
  vista. `supplier_ruc` del libro compras ahora trae `tax_id` (antes siempre `''`). Contrato FE
  sin cambios — solo bajan los montos de IVA/neto al valor legal. Smoke dev ✅ (libros, vat,
  vat/monthly = liquidación, filtros SIFEN).
- [x] **T12 · Audit scope (BP-4)**: ✅ 2026-09-15 (BE `c6d8c0a`, migración
  `20260915173844_t12_audit_logs_branch_backfill`, reporte
  `business_management/conductor/FASE1_T12_AUDIT_SCOPE_2026-09-15.md`): `audit_logs.branch_id` y
  su índice ya existían desde 20260429193353 pero el read-side nunca los usó — la condición era
  un OR de **11 EXISTS correlacionados** (65 líneas de SQL, PAYMENT con doble probe)
  inyectado en **18 sitios** de `audit_repo.go`, 12 de ellos vía `strings.Replace`/`ReplaceAll`
  (aliasización tardía, `GROUP BY X` → `AND (...) GROUP BY X`). Ahora:
  `auditBranchScopeCondition` = `alias.branch_id = $n` (O(1) sobre el índice) + los 12 sitios
  Replace reescritos a construcción directa (`branchCond` explícito; muere el
  `ReplaceAll("timestamp","al.timestamp")` innecesario). Semántica preservada: filas NULL-branch
  invisibles a lecturas de branch, visibles consolidadas. Migración backfill (10 UPDATEs
  guardados por `branch_id IS NULL` desde el mapeo de entidades; PRODUCT a NULL a propósito —
  multi-sucursal, T17 escribirá la branch del actor). **Bug latente arreglado**: `CreateAuditLog`
  siempre fallaba — lib/pq codifica `[]byte` tipado (incluido `json.RawMessage` nil) como cadena
  vacía y jsonb lo rechaza; `nullableJSON` pasa vacío como NULL (T17 lo necesita). Test de
  integración anti-regresión: siembra multi-branch y verifica aislamiento en logs/ByID
  (anti-leak), summary, dashboard KPIs, trends, top users, entity history, hourly, alerts.
  Smoke HTTP ✅ (consolidado 3 filas vs branch_id=1/2 con 1 fila cada uno). Nota para T17: el
  writer debe poblar `branch_id` con la branch activa del actor.
- [x] **T13 · Queries non-sargable (BP-5)**: ✅ 2026-09-15 (BE `7b506a4`, reporte
  `business_management/conductor/FASE1_T13_NONSARGABLE_2026-09-15.md`): predicated rewrites
  sargables — ventanas de due-date de payables List (`+INTERVAL '30 days'` pasa del lado de la
  columna al de la constante en Status OVERDUE + Priorities URGENT/HIGH/MEDIUM/LOW + DueBefore;
  con `enable_seqscan=off` la forma vieja no usa índice alguno y la nueva hace Bitmap Index
  Scan sobre `idx_purchase_orders_order_date` de T2) y `DaysOverdue` de receivables
  (`EXTRACT(DAY FROM NOW()-sale_date) >= n` → `sale_date <= NOW() - make_interval(days => n)`;
  inconsistencia preexistente documentada sin cambiar: filtra antigüedad de venta, no la
  columna mostrada basada en due_date). CTEs pre-agregados: StockAging (subqueries
  correlacionados, uno duplicado, → 2 CTEs agrupados), ABC (LATERAL por producto → un pase
  `sales_agg`), FinancialOverview (pagos por fila → paid joins de T11; caja → movimientos por
  registro agrupados). **HALLAZGO: StockAging, DeadStock y Reorder eran 500s crónicos desde
  FASE 10.2** — el CTE `product_stock` no aliasaba columnas y las queries referenciaban
  `pst.product_id`/`pst.current_stock`/`ls.last_sale_date`/`ORDER BY stock_value` inexistentes
  (SQL histórica congelada y corrida en psql para probarlo); alias explícitos en todos los CTEs
  y los 3 endpoints devuelven datos reales (aging 20 / dead-stock 9 / reorder 5 en dev). Tests
  A/B de integración: sets idénticos en las 6 ventanas, valores idénticos de FinancialOverview
  (consolidado + branch), ABC fila por fila. Smoke 8 endpoints 200 ✅.
- [x] **T14 · Protección (BP-6)**: ✅ 2026-09-15 (BE `e69f097`, reporte
  `business_management/conductor/FASE1_T14_RATE_LIMIT_ETAG_2026-09-15.md`): rate-limit Redis
  fail-open en los 12 subrouters BI vía `biChain` compartida (deadline → permiso → rate-limit →
  ETag; DI por `RouteDeps.RateLimiter`, sin tocar el global de `/setup`) con `config.BIRateLimit`
  (120 req/min por usuario — carga de dashboard ≈ 8 llamadas). **Ships observe-only** (el plan
  pide evaluar con métricas antes de habilitar): campo nuevo `RateLimitConfig.ObserveOnly`
  computa la decisión, cuenta `rate_limit_decisions_total{outcome=allowed|would_block|blocked|fail_open}`
  y loguea `rate_limit_would_block`, pero siempre pasa; flip a enforce (429 + Retry-After) con
  env `BI_RATE_LIMIT_ENFORCE` → `server.Config.BIRateLimitEnforce`. ETag middleware existente
  aplicado a los GET de BI, **habilitado por floorear `metadata.generated_at` al minuto** en
  `respondSuccess` — el timestamp por-request hacía cada respuesta byte-distinta y el 304 era
  imposible; ahora los hits del caché T10 son 100% byte-idénticos (colateral: muere el "único
  delta generated_at por-request" del smoke T10; FE solo lo usa como fallback/display).
  Métricas Prometheus nuevas en `/metrics` (puente a T21): rate limit por outcome +
  `etag_responses_total{hit,miss}` / `etag_bytes_saved_total`. Hallazgos: la semántica
  preexistente de `FailOpen=true` era "never block" (solo loguea; preservada, BI usa
  `ObserveOnly`); `CheckRateLimit` siempre fail-opens ante outage Redis (ahora visible con
  `Degraded`+`fail_open`; el comentario fail-closed de `SetupRateLimit` era falso, corregido).
  Smoke dev: 2º GET summary → **304**; 130 requests concurrentes → 130×200 con 12 would_block;
  barrido 12 grupos 200 (discounts 400 sin fechas = preexistente); legacy sin ETag. Gates:
  tests/lint/build ✅. Runbook del flip en el reporte.
- [x] **T15 · Branch scope estable (BP-8)**: ✅ 2026-09-15 (BE `9cc3a7d`, reporte
  `business_management/conductor/FASE1_T15_BRANCH_SCOPE_2026-09-15.md`): medido antes de
  elegir — COALESCE **no arregla nada** (columna dentro de la expresión = misma falta de
  condición de índice que el OR, comprobado con `force_generic_plan` + EXPLAIN), así que se
  implementó la **query dual** vía `biBranchScope(alias, n, branchID)` en `analytics_sql.go`:
  scoped `(alias.branch_id = $n)` (compuesto `idx_*_branch_date` alcanzable), consolidado
  `($n::int IS NULL)` (el brazo nil del OR verbatim — placeholder consumible ⇒ las 203 listas
  de args de los call sites no cambian de forma). Transformación mecánica de 203 sitios en los
  9 repos BI (variantes con/sin espacios, con/sin alias, SQL denso FASE 10.2 incluido) +
  `paymentsPaidJoin` T11 toma `branchID *int` (31 callers; variante sin-branch del dashboard
  preservada). Outliers manuales: `f.BranchID` discounts, 2 strings interpretados, 5 `const`
  → `:=`. Equivalencia exacta por construcción (nil ⇒ TRUE ambos; no-nil ⇒ `= $n` ambos;
  filas NULL-branch mismas visibilidades que T12). **A/B contra dev DB**: 3 shapes ×
  nil/branch/999 → sets idénticos vs texto histórico congelado; goldens T11 re-fijados a la
  forma dual; baseline pre-T15 congelada verbatim en el A/B de T13 (sigue verde). **EXPLAIN
  genérico con síntesis 300k filas**: shape agregado 4.1× más rápido y 3× menos filas tocadas;
  shape lista independiente del layout de datos. Gates + smoke ✅ (branch 999 → ceros;
  b1/b2/b3 con aritmética exacta). Hallazgos: (1) el "consolidado" por API no lo era —
  `middleware/auth.go:206` inyecta la branch activa del admin en los claims post-token
  (FASE E, preexistente); (2) **fan-out de `total_sales` en Performance** — `SUM(s.total_amount)`
  sobre LEFT JOIN con detalles infla el total por número de líneas (clase BC, preexistente,
  surfaceado al backlog); (3) los OR-branch de contextos no-BI quedan para su fase.
- [x] **T16 · Legibilidad (BP-7)**: ✅ 2026-09-15 (BE `19a69ea`, reporte
  `business_management/conductor/FASE1_T16_LEGIBILIDAD_2026-09-15.md`) — **cierra 1C**: los 4
  métodos gigantes partidos en orquestador + helpers privados (movimiento puro, SQL verbatim):
  ProductProfitabilityDateRange 267→59, PayablesRepo.List 219→85, ReceivablesRepo.List 202→83,
  GetAuditLogs 196→67 (builder de condiciones devuelve `(where, args, próximo índice)` que los
  LIMIT/OFFSET consumen). Las 14 líneas de SQL one-line de inventory reformateadas
  multi-línea, con verificación programática whitespace-normalizada línea a línea. Borrado
  `handlers/bi_helpers.go` (122 líneas muertas, 0 consumidores en el paquete). **HALLAZGO:
  2 endpoints más de la clase crónica T13** — `/inventory-analytics/forecast` (CTE
  `product_stock` sin aliases + ORDER BY sin alias) y `/inventory-analytics/movements` (CTE
  sin aliasar `quantity`/`value`; by-type ordenaba por `units` inexistente); probados
  preexistentes corriendo la query de HEAD en psql, arreglados con aliases explícitos,
  verificados 200 con datos reales y cubiertos por `TestInventoryResurrectedEndpoints`
  (ampliado) — los 5 endpoints crónicos de la clase quedan resucitados y con test. Gates +
  smoke ✅ (payables `total_pending` 6.414.250 idéntico al smoke T15 = regresión numérica;
  inventory 11/11; 10 grupos BI 200; 0 errores en log). El gate de cierre 1C (benchmark p95
  hey/ab) queda para el gate final de FASE 1.
- Gates 1C: EXPLAIN comparado + benchmark simple (p95 por endpoint con `hey`/`ab` contra el
  bin local) + `go test ./...` + lint + build.

#### 1D — Auditoría activable + RBAC (3 días)

- [x] **T17 · Write-path de auditoría (P0-5)**: ✅ 2026-09-16 (BE `251e69b`, reporte
  `business_management/conductor/FASE1_T17_T18_AUDIT_WRITE_PATH_RBAC_2026-09-16.md`):
  hasta T17 **nadie escribía** audit_logs (read-side only, `LogAction` sin callers). Pipeline
  async completo: `AuditTrail` worker (canal buffer 4096, Enqueue no-bloqueante con drop
  contado — jamás back-pressure del request path), batch INSERT multi-VALUES (chunks 200) vía
  puerto estrecho `AuditLogBatchWriter`, flush por tamaño (100) / ticker (2s) / drain en
  graceful shutdown; sampling configurable (env `AUDIT_SAMPLE_RATE`, default 1.0) que jamás
  suelta fallos, warnings ni AUTH. Middleware `AuditTrailMiddleware` para escrituras core
  autenticadas (ventas/anulaciones, compras, pagos, stock, precios, users/roles, caja;
  matching segment-aware — `/sale` no se traga `/sales-analytics`) con claims, request_id en
  metadata jsonb, status, duración y **branch activa denormalizada (nota T12 cumplida)**.
  Hooks AUTH en identity (login éxito/fallo con el intento como entity_id, logout, logout-all)
  porque `/login` es pre-auth y `/auth/refresh` contaminaría. Métricas
  `audit_trail_events_total{enqueued,sampled_out,dropped_full,written,write_error}` (T21).
  Ciclos de import resueltos: `auditIPAddress` local en identity (helpers importa identity
  vía jwt_helper), `logging.RequestIDMetadata` (analytics importa identity vía branchctx).
  **HALLAZGO (smoke, no tests): `Run` creaba el ticker y nadie consumía `ticker.C`** —
  lotes parciales no salían por tiempo; fix select-loop + test de regresión. El flush en
  shutdown graceful quedó validado EN VIVO (SIGTERM drenó un evento pre-fix). Test T12 hecho
  robusto por delta (los agregados absolutos rompían con filas concurrentes en la BD
  compartida — lo rompieron las filas del smoke). Smoke: login fallido/logout/escritura 400
  con branch=1/lectura admin/403 vendor ×3/métricas ✅. `LogAction` queda como API interna.
- [x] **T18 · RBAC coherente (BR-6, B-8 viejo)**: ✅ 2026-09-16 (mismo commit, mismo reporte):
  `module_permissions.go` completado contra los gates REALES — encabezados del plan:
  `/vat-prorrateo` y `/sales-analytics/discounts` son **reports:read** (no analytics:read);
  +17 prefijos faltantes (settings, devices, `sifen/{config,branch,inutilize,metrics,reconcile}`
  — reconcile es sifen:write, `/supplier`, `/purchase/payment`, `/counter-orders/metrics`
  (reports:read), measure/*, barcode read-only, `/cost-transactions` = **products:cost**) y
  stale corregidos: `/cost-trends`, `/margin-alerts`, `/suppliers/{id}/cost-analysis` gatean
  **products:cost**, no products:read; `/api/v1/permissions` era exact-path, no subrouter.
  Exact-path nuevos: pos-checkout/confirm-payment (cash:write), anulaciones (sales:cancel),
  aging crédito (receivables:read), account-payments (cash:write). **Test anti-drift**
  `rbac_catalog_drift_test.go`: escanea el fuente de registro de rutas y falla ante
  divergencia catálogo↔rutas en ambas direcciones (prefijos + `METHOD:/template` exactos),
  con excepciones documentadas (sifen sobre /sale; auth-only /auth y /sessions); verificado
  por mutación. **Decisión auditoría FE: `audit:read` es la fuente única** (gate BE real,
  nav FE ya lo usa); los 4 `RoleGuard F2VLso` de `/auditoria*` migran a PermissionGuard en
  FASE 2 — documentado en el reporte y en el test.
- Gates 1D: integration test del middleware (escribe batch 250 filas/2 chunks contra BD real,
  no bloquea con buffer lleno, samplea con excepciones AUTH/fallos) ✅ + test de catálogo
  (anti-drift bidireccional) ✅ + `go test ./...` 37 paquetes + lint + build + smoke ✅.

#### 1E — Calidad, contrato y tests (4-5 días)

- [x] **T19 · Contratos tipados (BQ-2)**: ✅ 2026-09-16 (79bd447) — 9 `map[string]interface{}` → structs tipados (LowStockProduct, OutOfStockProduct, NegativeMarginProduct, ExpiringBudget, HealthScore, PeriodComparison+Snapshot/Delta, ClientRisk, PaymentCapacity, SupplierAnalysis) + `Alert.Details`/`RecentActivity.Details` tipados con puntero+omitempty (contrato heterogéneo preservado); golden tests de claves JSON (regla 7). Nota: `DashboardSummary/KPIs.Details` ya no existían (T8 los reemplazó por `partialErrors`).
  del dominio (LowStockProduct, OutOfStockProduct, NegativeMarginProduct, ExpiringBudget,
  HealthScore, PeriodComparison, ClientRisk, PaymentCapacity, SupplierAnalysis) +
  `Details map[string]interface{}` de DashboardSummary/KPIs → structs.
- [x] **T20 · Forecast honesto (BQ-5)**: ✅ 2026-09-16 (d372437) — EXPONENTIAL (SES α=0.3, proyección plana) y SEASONAL (factores mensuales del trailing year, baseline media 12 puntos) implementados; método desconocido = 400 apperr (fail-fast pre-DB); `method_effective` expuesto; bandas ±15% y cap 95% documentados en `ForecastConfidence`; alias `/api/v1/forecast` removido (FE usa `/forecast`; el anti-drift T18 cazó la entrada stale del catálogo). Smoke en vivo: method_effective, 400 honesto de SEASONAL con <12 puntos, alias 404. (suavizado simple/Holt) y
  SEASONAL (factores por período — la base ya existe en `detectSeasonality`), o devolver 400
  "method not supported" en vez de degradar a linear en silencio; exponer `method_effective`
  en la respuesta; documentar bandas ±15% y confidence cap; decidir canonical
  (`/forecast`) y deprecar el dual-mount `/api/v1/forecast` cuando el FE confirme (FE usa
  `/forecast` hoy → el BE queda con `/forecast` y remueve el alias).
- [x] **T21 · Logging/métricas internas (BQ-4)**: ✅ 2026-09-16 (6914c40) — `bi_query_duration_seconds{group,op}` (op="http" en los 11 grupos vía biChain; op=<Método> en 53 agregados pesados) + `bi_slow_query` log estructurado con request_id sobre umbral `BI_SLOW_QUERY_MS` (default 500ms) + pprof en `/debug/pprof` tras `PPROF_ENABLED` (default false) y gate admin `users:write` sobre el auth global. Anti-drift T18 actualizado a la firma biChain(perm, group). (query lenta >
  umbral, con request_id), métrica Prometheus de duración por grupo de endpoint
  (`bi_query_duration_seconds{group=}`); pprof detrás de flag admin-only opcional.
- [x] **T22 · Tests (BQ-1)**: ✅ 2026-09-16 (63c7cf4) — handler tests estilo equiv para las superficies sin cobertura (inventory analytics con los 3 resucitados T13, profitability, receivables, audit, discount reports) + contratos T19 por el wire; `routes_inventory_test.go` fija la tabla de rutas via router.Walk (representante por grupo, resucitados, GET-only, 404-no-500, alias muerto); `tests/integration/bi_fixed_queries_test.go` con DB real: endpoints resucitados, contratos tipados financieros, análisis CxP/CxC (proveedor inexistente = 404 tipado; cliente inexistente = zeros), dashboard con metadata.partial y los 4 métodos de forecast. (mocks por
  puerto, estilo `equiv_test.go`); tests de rutas/gates de `analytics/http`; integration
  tests (DB real, docker) para las queries corregidas en T1/T2/T4/T11/T12/T13 — el E2E es el
  único gate que ejercita SQL real de repos mockeados (lección FASE 5 pedidos).
- [x] **T23 · Pureza de dominio (BQ-3, incremental)**: ✅ 2026-09-16 (892aac3) — `getPeriodDates` de audit.go eliminado (5 call sites → `tb.Boundaries(NewPeriodRolling(period))`); wire DTOs de discounts (`DiscountLine`, `DiscountSummaryRow`) movidos al handler; `internal/analytics/dto.go` creado (el archivo que el doc de paquete reserva para transporte) con los 10 contratos T19 dentro del contexto acotado. `OutOfStockProduct`/`AlertDetails`/`ActivityDetails` permanecen en models/ (models los embebe por valor; ciclo de imports). Alias restantes de domain.go: deuda documentada., mover structs de
  `models/` al dominio analytics; quitar `json:` tags de discounts.go o mover al handler;
  eliminar `getPeriodDates` de audit.go usando `Period`.
- [x] **GATE FINAL FASE 1 (duro)**: ✅ 2026-09-16 SUPERADO — `go test ./...` 37 paquetes OK
  · `golangci-lint run` verde · `make build` OK · smoke dev-JWT 11 grupos (13 endpoints 200) con
  contratos T19/T20/T21 verificados en vivo · benchmark p95 documentado (todos los grupos
  ≤ 11 ms en dev; caveat DB pequeña) · reporte de cierre en
  `business_management/conductor/AUDIT_BI_BACKEND_2026-09.md` (5be5ef2).
  ⛔ Recién aquí arranca la FASE 2 (frontend).

### FASE 2 — Auditoría por página FE (3-4 días; requiere FASE 1 ✅)

- [ ] Ejecutar protocolo §3 para las ~36 páginas, en orden de riesgo: **CxP (6) ✅ 2A** → **CxC (6) ✅ 2B** →
      **Dashboard (5) ✅ 2C** → Sales Analytics (6) → Pronósticos (5) → Rentabilidad (6) → Inventario
      (4) → Reportes Financieros (8) → Auditoría (4)
      - **[x] 2A CxP ✅ 2026-09-16** (FE `2a1cce6`): 6 fichas + 7 screenshots en
        `erp-webapp/conductor/audits/bi-2026-09/`. Veredictos: 6/6 FAIL — 4 páginas muertas por
        P0-1 (métodos inexistentes en `payablesService`), Cash Flow con drift de contrato total
        (API 200 → UI en cero) + bancos/insights fabricados (P0-6 ampliado), Supplier Analysis
        en blanco con API sana, "Nueva Factura" → `/payables/new` **404**, Aging funcional pero
        con 3 KPIs "---" y 4 botones decorativos. **Transversal nuevo**: `/payables/*` sin guard
        FE (vendedor accede por URL a 5 rutas; BE 403 correcto con `payables:read`); el gate del
        dashboard usa `dashboard:read` ≠ `payables:read` del BE. Corrección al plan: la página
        de detalle es `InvoiceDetail.jsx`, no `PayableDetail.jsx` (stale). Ambos roles probados
        (admin consolidado dev-JWT + VNDR01).
- [ ] Probar con 2 roles: admin (consolidado) y perfil acotado (vendedor / Buyer) — validar
      gates y redirecciones por rol
- [ ] Consolidar `SUMMARY.md` con matriz página × veredicto

### FASE 3 — Fixes FE priorizados (dependiente de F2)

- [ ] **Ronda P0** (2 días): completar `payablesService` (6 métodos contra los endpoints
      reales) · `getMasterList` CxC · redirecciones (usa el catálogo de action_url de T5;
      `/clientes`→`/parties`) · eliminar bankPositions/insights mock
- [ ] **Ronda P1** (2-3 días): eliminar fallbacks demo del store y mocks de inicialización
      (estado inicial `null` + estados loading/error/empty reales DESIGN §6.7; usar
      `metadata.partial` de T8 para banner de datos parciales) · growth literales → API ·
      TrendsVelocity max dinámico · decisión sobre las 2 pantallas 100% mock · "+12.5%" ·
      cablear o quitar UI decorativa · borrar código muerto P1-9
- [ ] **Ronda P2** (2-3 días): paginación server-side en tablas del bloque 2.3 (el BE ya
      pagina tras T9) · filtros · `window.location.href` → `navigate()`
- Gates por ronda: `npx vitest --run` 0 fallos, `npx tsc --noEmit` 0, `pnpm build`,
  `pnpm lint:design`, re-verificación browser.

### FASE 4 — Alineación DESIGN/AGENTS + migración (continua, por grupo)

- [x] Cada página legacy tocada sustancialmente se migra a `.tsx` + FSD con hooks puros y
      domain fuera de componentes (expectativa full-alignment del workspace) — ✅ 2026-09-16:
      12 páginas + 2 features completos (cash-flow, supplier-analysis) en 7 commits FE
      (`1fdb8d3` → `9675c5f`); alcance binario (diff FASE 3 sustancial) y checklist por
      página en `erp-webapp/conductor/PLAN_FASE4_MIGRACION_TSX_BI_2026-09-16.md`
- [x] Checklist DESIGN §10 + §6.7 + §6.3 por página migrada — ✅ las 11 páginas quedan
      limpias contra `lint:design --base main` (todo el color legacy slate/hex mapeado a
      tokens §2); desviaciones honestas anotadas (spinners legacy, i18n retro pendiente)
- [x] Auditoría `vercel-react-best-practices` del área → reporte en
      `erp-webapp/conductor/AUDIT_REACT_PERFORMANCE_BI_FRONTEND.md` — ✅ 7 hallazgos
      priorizados (H1 chunk único de 3.3 MB sin code-splitting, H2 suscripción sin selector
      al store en 5 páginas, H3 carrera de fetch en useDashboardStore, H4/H5/H6/H7) con
      propuesta de trabajo
- [x] i18n: sin strings nuevos hardcoded — ✅ (strings preexistentes se conservan verbatim;
      sweep retro-i18n anotado como tarea independiente)
- **Drift destapado por la migración y corregido**: `TrendsVelocity` crasheaba en dev HEAD
  (TDZ) y conservaba el max fijo del heatmap (P1-4); `PeriodComparison` fabricaba la serie
  "Período B" (×0.85/×1.15 → ahora `/trends/date-range` real con los límites que el propio
  compare devuelve, nuevo `getTrendsDateRange`); `sales-analytics/Dashboard` nunca enviaba
  `period` al BE ni renderizaba su estado error; `TrendChart` de cash-flow con series
  muertas (dataKey vs contrato del hook, resto del drift 2A); `PronosticoIngresos` con
  clase dinámica no compilable y fallbacks de período inventados.
- **Smoke en vivo (dev-JWT admin + modo API)**: 11 páginas migradas verificadas en navegador
  — auditoría (18 acciones reales), logs, resumen financiero (caja Gs. 386.360, salud 95),
  cash-flow (gráfico pintando series reales), proveedor BodyTech completo, sales dashboard,
  insights, trends-velocity (heatmap real, pico 12:00), comparativa, demanda, ingresos
  (estado vacío honesto de escenarios). Gates: vitest 656/656 · tsc 0 · build · lint:design.

### FASE 5 — Tests y cierre (1-2 días)

- [ ] Tests FE por página/componente tocado según PLAN_TEST_DESIGN_FRONTEND
- [ ] Gate final integrado (ambos repos) + registro del cierre y purge del plan según
  convención del README

---

## 5. Reglas de decisión (binarias)

**Backend**: 1) Todo error tipado con `apperr`; not-found es 404, filtro inválido es 400, y
jamás se envía `err.Error()` crudo al cliente. 2) Ninguna query BI sin deadline. 3) Ningún
slice SQL sin LIMIT declarado (o justificación written-down de acotamiento natural). 4) Toda
métrica diaria/horaria se computa en America/Asuncion (documentado en el repo). 5) ORDER BY
dinámico solo desde whitelist. 6) Toda vista materializada/cache con estrategia de
invalidación o TTL declarada. 7) Cambio de contrato BE ⇒ test que lo fije + actualización FE
en el mismo ciclo.

**Frontend**: 1) Nada hardcodeado en modo API: si un dato no viene de endpoint, la sección no
se muestra o muestra estado vacío honesto. 2) Toda afordancia funciona o desaparece. 3) >10
filas ⇒ paginación server-side + filtros + total count. 4) Redirecciones: destino existe, el
rol tiene permiso, preserva contexto. 5) No se salta DESIGN.md.

---

## 6. Entregables

1. Este plan (raíz, cross-repo) + avance por fase.
2. BE: migraciones de índices/audit-scope (solo `make migrate-create`), reporte
   `AUDIT_BI_BACKEND_2026-09.md` en `business_management/conductor/` con EXPLAIN antes/después
   y benchmark p95, tests nuevos (service + integration).
3. FE: `erp-webapp/conductor/audits/bi-2026-09/` (fichas + screenshots + SUMMARY.md), fixes
   en ramas `dev`, reporte performance React.
4. Al cierre: purge de este archivo según convención del README.

## 7. Riesgos y notas

- **BC-1 cambia números que el usuario conoce**: tras el fix de timezone, "hoy" incluirá
  21:00-24:00 locales y excluirá las del día anterior — los dashboards van a "moverse" 3h.
  Es correcto, pero hay que anunciarlo (nota de release).
- **T1 opción destructiva** (fijar TZ + backfill de naive timestamps) solo con confirmación
  explícita del usuario y backup; la opción recomendada (conversión en boundary) no toca datos.
- **Validación de zona en prod (pendiente de acceso, ejecutar en la auditoría A0)**: el
  runbook paso a paso está en `business_management/conductor/AUDIT_BI_T1_TIMEZONE_2026-09-14.md`
  §4 (SHOW timezone + distribución horaria de sesiones → decidir `BI_STORAGE_TIMEZONE`; si la
  DB corre UTC, setear `BI_STORAGE_TIMEZONE=UTC` + `TZ=America/Asuncion` en el contenedor
  backend y preferentemente `TimeZone=America/Asuncion` en la DB; verificar ausencia de
  `bi_storage_timezone_mismatch` en el log de arranque).
- **Índices en prod**: `CREATE INDEX CONCURRENTLY` fuera de tx (golang-migrate corre
  migraciones en tx) — runbook en la migración.
- **Cache TTL**: dashboards con 60-120s de staleness — aceptable para BI; documentar y no
  cachear endpoints de detalle.
- **Dev DB ruidosa** (~84 ventas CANCELLED): ✅ resuelto en F0 (purge con backup 2026-09-14).
- **`pnpm dev` defaultea a demo**: auditar siempre en modo API.
- **Auditoría BE (T17)** puede generar volumen: sampling configurable desde día 1.
- **Main se mueve entre sesiones**: rebase antes de cada gate.

> **Avance FASE 2 (2026-09-16)**
> - **2A CxP ✅** (FE `2a1cce6`, wrapper `deb1b5c`) — 6/6 FAIL; P0-1 confirmado; CashFlow drift total
>   + fabricaciones (P0-6 ampliado); SupplierAnalysis blank con API 200; "Nueva Factura" →
>   `/payables/new` 404; `/payables/*` sin guard FE (vendedor accede, BE 403 ok); plan stale:
>   detalle = `InvoiceDetail.jsx`.
> - **2B CxC ✅** (FE `adc32fa`) — Resumen FAIL (P0-2 envenena Promise.all → ceros + curva fabricada
>   + chip "Sincronizado" falso); Lista FAIL (vacío deshonesto + filtros stub); Detalle FAIL (2
>   métodos); Vencidas FAIL trivial (name-mismatch, ruta canónica T5 rota); Aging PASS condicional
>   (datos reales; "Oct 2023" hardcodeado; gráfico sin fuente); Perfil Crédito FAIL (score 50
>   fallback vs risk_level LOW real; límite 150M literal; barras literales; falsa alerta >90d).
>   **P0-2 ampliado a 4 métodos faltantes.** Doble montaje `/receivables` (sin guard) vs
>   `/dashboard/receivables` (gated). Vendedor accede por URL al grupo (NA-CXP-2).

> - **2C Dashboard ✅** (FE `9e6bd6c`) — **NA-DB-1**: `useDashboardStore` guarda el envelope sin
>   `.data` → Resumen Ejecutivo en Gs. 0 con trends/alerts/actividad reales (fix 1 línea, impacto
>   máximo); **NA-DB-2**: heatmap con "$0" en sistema PYG; KPIs Detallado lee bien el contrato
>   anidado (Índice de Salud = TODO turnover×10; chips candado decorativos); Alertas: **T5
>   "Gestionar" → /receivables/overdue verificado EN VIVO** (destino roto = 2B), 4 botones sin
>   handler, P0-4 no ejercitable (0 alertas cliente en dev); Top Productos PASS condicional
>   (floats crudos). Grupo 100% gated dashboard:read ✅. P1-1 fallback demo NO dispara con API
>   sana (solo catch).

> - **2D Sales Analytics ✅** (FE `60df285`) — datos reales en las 6 (mocks catch-only con API
>   sana); payment methods "UNKNOWN 100%"; floats crudos ×4 págs ("−77.0818…%"); growth/churn
>   literales P1-3 (retención 100% vs churn 13.9% contradictorios); heatmap de velocity SIN datos
>   + hora pico fija P1-4; comparativa con delta de margen mal calculado (−5.0pp mostrado para
>   +20.3pp reales); Descuentos PASS (única con guard reports:read, .tsx, 1 fila incoherente);
>   5/6 rutas sin guard.

> - **2E Pronósticos ✅** (FE commits de esta sesión) — Salud Inventario/Ventas/Demanda PASS
>   (datos reales; stats honestas T20 visibles; demanda 13 filas sin paginar); Ingresos FAIL
>   (escenarios y bandas en cero por mapping; tabla real); Dashboard FAIL P1 ("+33521439.14%"
>   = monto $ como %). 5 rutas sin guard.

> - **2F Rentabilidad ✅** — Productos PASS (patrón referencia); Clientes PASS cond (Pareto 0%);
>   Categorías FAIL (NaN en KPI); Dashboard FAIL (gráfico vacío, texto roto, **discrepancia de
>   profit del módulo vs summary global = política de costo 0,7 vs real — decisión de negocio**);
>   Tendencias FAIL (+-100%, pico ---, vacío); Vendedores FAIL (fan-out T15 BE + objetivo 0%).
>   Grupo 100% gated analytics:read ✅.

> - **2G Inventario ✅** — 4/4 PASS (único grupo .tsx completo; dashboard = mejor página BI;
>   stock-levels 20 filas sin paginar P2; endpoints resucitados T13/T16 verificados en vivo).

> - **2H Reportes Financieros ✅** (7 págs reales en sidebar, conteo "8" stale) — Resumen
>   Financiero FAIL P1 (caja "$1.2M" = mock Santander; pronóstico "$742k" fabricado); Gestión
>   IVA FAIL P0-trivial (getVat/getTaxSummary inexistentes; IVA legal post-T11 invisible);
>   CashFlow/PyG PASS cond; Libros Legales y SIFEN×2 PASS. 5/7 rutas sin guard.

> - **2I Auditoría + huérfanas ✅** (FE `7e30db9`, SUMMARY.md) — Dashboard Auditoría FAIL
>   (curva fabricada + donut 100% + "+12.5%"); **Logs FAIL grave (KPIs 100% mock 12.450/42/188/
>   3.120 junto a tabla vacía; "Página 1 de 30" sin filas)**; Detalle y Actividad PASS (honestos).
>   Huérfanas: StockManagement CRASHEA, decisión ELIMINAR ×3. **RoleGuard F2VLso sigue sin
>   migrar a audit:read (tarea T18 pendiente de ejecución en FASE 3).**
>
> ## ✅ FASE 2 COMPLETA (2026-09-16) — 50 páginas + 3 huérfanas auditadas en 9 bloques
>
> **Veredicto global: 10 PASS · 12 PASS condicional · 28 FAIL (56% no confiable)**.
> Entregable: `erp-webapp/conductor/audits/bi-2026-09/SUMMARY.md` (matriz completa, 12 hallazgos
> transversales, prioridades P0/P1/P2 para FASE 3) + 44 fichas + 63 screenshots. Commits FE:
> `2a1cce6` (2A) → `7e30db9` (2I). Roles probados: admin consolidado + vendedor VNDR01.
> **Next: FASE 3 — fixes FE priorizados** (P0: 12 métodos de service, NA-DB-1 unwrap, remapeos
> cash-flow/supplier, KPIs mock de auditoría, guards; P1: fabricaciones + honestidad; P2:
> paginaciones + pulido + eliminar huérfanas).

> ## ✅ FASE 3 — RONDA P0 COMPLETA (2026-09-16) — páginas muertas desbloqueadas + RBAC real
>
> 7 commits FE en `dev` (774314a → c3be837), gates verdes por commit (vitest 653/653 ·
> tsc 0 · build · lint:design) + **smoke de navegador verificado en vivo** (dev-JWT admin
> jJkV4F6HR + modo API):
>
> - **3A `774314a`** — 12 métodos de service completados contra endpoints BE reales
>   (payables 6 · receivables 4 · finreports 2) + mapeos `purchase_date`/`payment_history`
>   + fix prefijo audit. Test de contrato: `biServices.p0-methods.service.test.ts` (12 tests).
> - **3B `3f12c3c`** — **NA-DB-1**: store desenvuelve `.data` (summary + payablesOverview;
>   tipo de getSummary honesto). `useCashFlow` remapeado a `/payables/cash-flow` +
>   `/payables/schedule` (proyección y calendario reales; bancos Itaú/Sudameris e insights
>   fabricados FUERA con sus archivos: TreasuryInsights.jsx + mockData.js eliminados).
>   `useSupplierAnalysis` remapeado al dto plano + `/payables/supplier/{id}` (página pasó de
>   EN BLANCO a completa; fuera score/100, límite ×1.5, trend Ene-Feb-Mar, badge CRÍTICA fija).
> - **3C `922eb51`** — Auditoría: KPIs mock (12.450/42/188/3.120) → reales del dashboard
>   endpoint ('—' si no hay); paginación server-side real (total/total_pages/page_size);
>   filtros fechas/nivel/resultado cableados; navigate() en vez de location.href; curva y
>   donut reales desde /audit/trends y actions_by_category; "+12.5%" y "0% = Óptimo" fuera.
>   Bug hallado: el dashboard de auditoría leía el envelope (ceros estructurales) — unwrap.
> - **3D `152c772`** — **RBAC**: 32 rutas BI con PermissionGuard espejando el gate exacto
>   del BE (payables/receivables/reports/sifen/analytics:read) + **T18 ejecutada**: 4 RoleGuard
>   F2VLso → PermissionGuard audit:read.
> - **3E `2411723`** — "Nueva Factura" → 404 eliminada (botón+FAB+Exportar muerto);
>   /clientes → /parties?tab=clientes (chips de alertas + breadcrumb).
> - **P0 gate `c3be837`** — fixes del smoke: variable equivocada en fetchSummary;
>   contacto JSON crudo del proveedor parseado; avgDays 0 → sin frase.
>
> **Verificación en vivo (post-P0)**: /payables/invoices 18 facturas reales · /receivables
> Gs. 522.200 + facturas recientes · /receivables/overdue viva · /dashboard montos reales
> (CxC 81% cobrado, CxP 49,9%) · /auditoria ceros honestos · IVA Gs. 19.425 = tasa legal
> 10/110 T11 · cash-flow con calendario real · proveedor BodyTech completo (Crítica real,
> Gs. 3.009.450, 10 facturas) · /auditoria/logs sin mocks, paginación real.
>
> **Notas para siguientes rondas (P1/P2)**: curva de cobranza CxC sigue con fallback "-∞" y
> chip "Sincronizado" hardcodeado; tramos de antigüedad del dashboard CxC en 0% (revisar
> mapping vs /aging/summary); transform de Overdue genera refs "INV-2023-XXX" y contactos
> "Vía Email" ficticios; tabla nueva audit_logs tiene 0 filas (pipeline T17) mientras el
> legacy audits.audits conserva 8.816 sin backfill (decisión BE pendiente); filters
> search/priority/fechas del List de payables viajan pero el BE los ignora (gap contrato);
> POST /api/v1/audit/export existe sin botón FE (cablear en P2 o quitar).
> **Next: FASE 3 ronda P1** (fabricaciones del resumen financiero/insights/perfil crédito,
> NA-DB-2 $→Gs., redondeos, fallbacks demo de catches, delta margen pp).

> ## ✅ FASE 3 — RONDA P1 COMPLETA (2026-09-16) — honestidad de datos
>
> 5 commits FE (d0846f7 → dc31468), gates verdes + smoke:
>
> - **3F `d0846f7`** — Resumen Financiero: caja real del cash-flow (fuera "$1.2M"), salud
>   real de /health-score (fuera "84", ratios 2.45/1.82/42.5%, "Obj: 45%"), banner
>   "$742k Predictive BI" eliminado, Detalle → PyG real. Sales Insights: mocks fuera del
>   estado inicial, growths literales (12/5.4/-2.1/15 + churn 13.9%) fuera, "Ver todos los
>   clientes" → /parties. CxC: tramos reales (getAgingSummary apuntaba a /aging/report →
>   ahora /aging/summary), curva sin datos → empty state (los "-∞" venían de Math.max de
>   array vacío), chip "Sincronizado" fuera. Perfil Crédito: score 50 fuera (RiskGauge por
>   nivel), 150M fuera, utilization 80 fuera, aging computado desde facturas reales
>   (fuera "Gs. 250M" hardcodeado). Overdue: refs INV-2023-XXX y "Vía Email" reales.
> - **3G `c8b6851` + `dc31468`** — insights de pronósticos formateados por semántica del
>   Type BE (monto→PYG, conteo, % 1 decimal; fuera el "+33521439.14%"); delta de margen
>   en pp calculado de los márgenes reales (margin_change_pct es cambio RELATIVO, no pp);
>   floats crudos redondeados (comparativa, heatmap "$0"→"Gs. 0", margen 94.75…%, kpis
>   pronósticos); leak err.message de InvoiceDetail saneado.
> - **3H `a16874e`** — mocks fuera de estado inicial/catch (SalesDashboard, TrendsVelocity,
>   PeriodComparison, useAgingReport) + 4 mocks huérfanos de accounts-payable borrados +
>   **banner T8**: apiClient despacha `api:partial-data` con metadata.partial y App muestra
>   toast throttled 5s.
>
> Smoke post-P1: Resumen Financiero caja Gs. 386.360 real / salud 95 real / Quick Ratio
> 0.29 "Revisar" (honesto); CxC tramos 34,28%/65,72% reales; sales dashboard sin flash de
> mock. **Pendiente de decisión de negocio: unificación de política de costo (profit
> módulo vs summary global) — NO tocar sin el usuario.**
>
> **Next: FASE 3 ronda P2** (huérfanas eliminar, TrendsVelocity hora pico, "Oct 2023",
> bandas T20 ingresos, paginación stock-levels/demanda).

> ## ✅ FASE 3 — RONDA P2 + CIERRE (2026-09-16) — pulido y limpieza
>
> 3 commits FE (a8b0783 → 6dfb20f), gates verdes por commit:
>
> - **3I `a8b0783`** — decisión ELIMINAR de FASE 2 ejecutada: rutas huérfanas fuera
>   (/bi/inventory/stock-levels que CRASHEABA + mock, /bi/inventory/risk-analysis mock,
>   /finance/profitability duplicada) + componentes y exports borrados.
> - **3J `aed0566` + `8e8399f`** — "Hora pico 14:00" fija → real del hourly trend ("n/d" si
>   no hay datos); captions literales ("+5% unidades", "Promedio diario estable")
>   neutralizados; "Oct 2023" y stub "Exportar" del Aging CxC fuera; escenarios de
>   Pronóstico de Ingresos ya no se fabrican con el BE mandando scenarios=null (fuera el
>   bug `recomendado: || true`) — estado vacío honesto; las bandas T20 (lower/upper bound)
>   ya se muestran reales en la tabla mensual. Hex → tokens en chip de TrendsVelocity.
> - **3K `6dfb20f`** — stock-levels consume la paginación server-side de T9 (antes traía
>   solo la página default sin controles): page/page_size + Anterior/Siguiente + "Página X
>   de Y" desde metadata real, verificado en vivo (17 de 17, 1 página en dev).
>
> **Smoke final en vivo**: stock-levels paginado OK; resumen financiero, CxC, auditoría,
> IVA, cash-flow y proveedor verificados en la ronda P0/P1.
>
> ## ✅ FASE 3 COMPLETA (2026-09-16) — 15 commits FE (774314a → 6dfb20f) + 3 wrappers
>
> - P0: 12 métodos de service + NA-DB-1 + remapeos cash-flow/supplier + auditoría real +
>   RBAC (32 rutas, T18 audit:read) + redirecciones → **~10 páginas muertas resucitadas**.
> - P1: fabricaciones fuera (resumen financiero, insights, perfil crédito, curva CxC,
>   overdue), NA-DB-2, redondeos, delta margen pp, leak InvoiceDetail, fallbacks mock
>   fuera, banner T8 metadata.partial.
> - P2: huérfanas eliminadas, hora pico real, Oct 2023, bandas T20, paginación stock-levels.
> - **Aucas pendientes → RESUELTAS (2026-09-16, sesión pre-FASE 4)**: ① unificación de
>   política de costo ✅ **decisión del usuario: política A (costo real en todo)** — BE
>   `3e77d05`: `getFinancialKPIs`/`getInventoryKPIs` sin el fallback `unit_price*0.7`
>   (BC-12) y el bloque `profit` del Summary calculado sobre **COGS real** de las líneas
>   vendidas (nuevo `getRealCOGS`) en vez de `ventas − compras del período` (la razón
>   oculta por la que ni el 0.7 cuadraba); verificado en vivo: `/dashboard/summary` ==
>   `/profitability/overview` → 202.480 / 94,76%. El snapshot histórico de `unit_cost`
>   en la línea de venta queda como mejora futura (FASE 8) — el profit sigue resolviendo
>   costo actual, o sea cambios de costo "reescriben historia"; ② demanda server-side pagination ✅ (BE `41e708e`: parsePaginationErr +
>   COUNT separado + LIMIT/OFFSET con CTE compartido, `top_n` queda como alias de
>   page_size; FE `0f524b6`: page state + footer metadata, patrón F3K); ③ backfill
>   legacy ✅ (migraciones `20260916183127` + `20260916185600`: 8.818 filas → audit_logs
>   con `metadata.legacy_id` idempotente, TZ reconstruida con
>   `current_setting('TimeZone')`, triggers legacy CONGELADOS — audit-only, verificado
>   en pg_proc; SALE_PAYMENTS entity_id recuperado vía `payment_id` (la PK no es `id`);
>   branch mapeado para reserves/SALE_PAYMENTS según criterio T12 — users/PRODUCTS
>   quedan consolidado-only; 19 LEGACY visibles en branch 1 en dev, export CSV
>   verificado); ④ List payables ✅ (BE `ce2af93`: search/priority/fechas parseados,
>   priority enum-validada 400, end_date exclusivo +1d, ILIKE id/proveedor/RUC/CI,
>   parties JOIN agregado al COUNT; FE `fee054c` doc de contrato); ⑤ audit/export ✅
>   (FE `7c2baac`: `exportLogs` vía getBlob + botón "Exportar CSV" en /auditoria/logs
>   con filtros activos, search→search_term, success→bool, end_date exclusivo, test de
>   contrato). Smoke en vivo: payables 18→10 con filtros, demanda página 1/2 sin
>   solapamiento, CSV con usuario resuelto y TZ -03. Gates: BE go test/lint/build +
>   FE vitest 656/tsc/build/lint:design en verde.
> - **Next: FASE 4** — migración .tsx/FSD de páginas legacy sustancialmente tocadas +
>   checklist DESIGN + auditoría vercel-react-best-practices del módulo BI.

> ## ✅ FASE 4 COMPLETA (2026-09-16) — migración .tsx/FSD + DESIGN tokens + auditoría React
>
> 7 commits FE en `dev` (`1fdb8d3` → `9675c5f`), gates verdes por commit (vitest 656/656 ·
> tsc 0 · build · lint:design vs main) + smoke de navegador en vivo (dev-JWT admin
> jJkV4F6HR, modo API, Vite reiniciado tras los renombres):
>
> - **Migrados a .tsx/.ts** (criterio: diff FASE 3 sustancial — detalle y checklist por
>   página en `erp-webapp/conductor/PLAN_FASE4_MIGRACION_TSX_BI_2026-09-16.md`):
>   - A Auditoría: AuditDashboard, AuditLogs (`1fdb8d3`)
>   - B Financiero: FinancialSummaryDashboard + tokens (`bcc2280`)
>   - C Cash-flow: página + useCashFlow.ts + KpiSection/TrendChart/PaymentCalendar +
>     types.ts FSD + fix dataKeys del gráfico (`fbecb52`)
>   - D Supplier Analysis: página + hook + 4 componentes + types.ts + tokens (`eedad31`)
>   - E Sales Analytics: Dashboard, CustomerSellerInsights, PeriodComparison,
>     TrendsVelocity + fixes (`7ab6c85`)
>   - F Pronósticos: PronosticoDemanda, PronosticoIngresos + tokens (`46d1857`)
>   - Smoke fix: float crudo en unidades/día (`9675c5f`)
> - **Drift corregido** (la conversión a tipado lo destapó): TDZ crash de TrendsVelocity en
>   dev HEAD; serie fabricada "Período B" → `/trends/date-range` real (nuevo
>   `salesAnalyticsService.getTrendsDateRange`, límites del propio compare); `period` que
>   nunca viajaba en sales dashboard + error state; dataKeys muertos del TrendChart
>   (drift 2A remanente); `bg-${color}-500` no compilable; fallbacks de período inventados;
>   max fijo 1M del heatmap (P1-4 remanente).
> - **Entregables**: checklist DESIGN §10/§6.7/§6.3 por página (11/11 limpias contra
>   `lint:design --base main`) + `erp-webapp/conductor/AUDIT_REACT_PERFORMANCE_BI_FRONTEND.md`
>   (H1 chunk 3.3 MB sin lazy-routes, H2 store sin selectores, H3 carrera de fetch, H4-H7;
>   propuesta: code-splitting BI + selectores Zustand + guard anti-carrera → próxima sesión).
> - **Fuera de alcance documentado**: páginas con toques triviales de FASE 3
>   (InvoicesMasterList, InvoiceDetail, ReceivablesDashboard, AgingReport, ConsolidatedAlerts,
>   SalesHeatmap, ProductsCategories, ClientCreditProfile); sweep retro-i18n de las páginas
>   migradas; afordancias decorativas heredadas (H7, decisión en FASE 5).
>
> **Next: FASE 5** — tests FE por página/componente migrado (PLAN_TEST_DESIGN_FRONTEND) +
> gate final integrado + cierre y purge del plan según convención del README.
