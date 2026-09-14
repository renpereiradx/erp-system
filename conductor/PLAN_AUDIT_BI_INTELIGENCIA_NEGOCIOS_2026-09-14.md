# PLAN — Auditoría y remediación del módulo BI "Inteligencia de Negocios"

**Fecha**: 2026-09-14 (v2: auditoría profunda backend integrada) · **Tipo**: Cross-repo (backend Go + frontend React) · **Estado**: 🔄 En progreso — **FASE 0 ✅ + 1A ✅ COMPLETA (T1-T5) + 1B: T6 ✅ (3cc04ab)**; T7-T9 + sub-fases 1C-1E pendientes

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
- [ ] **T7 · Timeouts + paralelismo (BR-3, BP-1)**: `context.WithTimeout` (10-15s) en los
  services BI o budget por query en repos; `errgroup` para las sub-queries de Summary/KPIs;
  `Comparison`/`TopBottomProducts` a query única con window functions en vez de 2/4 pasadas.
- [ ] **T8 · Errores parciales honestos (BR-4a)**: sub-summaries fallidas → log estructurado
  + `metadata.partial: true` + `metadata.partial_errors:[...]` (el FE podrá mostrar banner
  "datos parciales" en FASE 3).
- [ ] **T9 · Parseo unificado (BR-5, P2)**: un único `parsePeriod` validado (default month),
  un único `parsePagination` (default 1/20, cap 100) para TODOS los handlers; page/size
  inválidos → 400; `getLimit` con techo 100; LIMIT en AgingReport/DiscountLines/
  CollectionReminders + `COUNT OVER` para total. Extraer helpers de date-range y days.
- Gates 1B: ídem 1A + tests de mapeo de errores (404/400/500) y de parseo.

#### 1C — Performance y arquitectura (4-5 días)

- [ ] **T10 · Cache BI (BP-2)**: puerto `CachePort` en analytics (estilo pricing) + adapter
  Redis reutilizando `config/redis.go`; clave = endpoint+period+branch+hash(filters); TTL
  60-120s para dashboard summary/kpis/overview de receivables/payables; singleflight +
  fail-open (patrón ProductCache); cache solo en GET. Evaluar como alternativa/complemento
  una vista materializada del summary con refresh event-driven (patrón del worker de
  `mv_product_catalog` con pg_notify).
- [ ] **T11 · Deduplicación SQL (BP-3)**: helpers Go para el bloque `paid` (~30 copias) y el
  par revenue/cost; ledgers IVA pasan a leer las vistas `v_sales_ledger_sifen` /
  `v_purchases_ledger_sifen` / `v_monthly_vat_liquidation` con test A/B de equivalencia
  (misma fecha → mismo resultado que la query a mano antes de borrarla).
- [ ] **T12 · Audit scope (BP-4)**: denormalizar `branch_id` en `audit_logs` (migración +
  backfill desde el mapeo de entidades) y reemplazar el OR-de-11-EXISTS + `strings.Replace`
  por filtro directo; test anti-regresión de scope (usuario de una branch no ve logs de otra).
- [ ] **T13 · Queries non-sargable (BP-5)**: reescribir `EXTRACT(DAY FROM NOW()-x)` como
  rango de fechas calculado en Go; due-date de payables a columna calculada o rango; ABC/
  StockAging/FinancialOverview con CTEs pre-agregados en vez de subqueries por fila.
- [ ] **T14 · Protección (BP-6)**: rate-limit (Redis, fail-open) en los endpoints agregados
  pesados (dashboard, analytics, reports); ETag en GET de reports. Evaluar costo/beneficio
  con métricas del middleware antes de habilitar.
- [ ] **T15 · Branch scope estable (BP-8)**: reemplazar `($n IS NULL OR branch_id=$n)` por
  `branch_id = COALESCE($n, branch_id)` o query dual elegida en Go, verificado con EXPLAIN
  que usa el índice compuesto.
- [ ] **T16 · Legibilidad (BP-7)**: partir los 4 métodos gigantes y el SQL one-line de
  inventory (sin cambio funcional; test de regresión = mismas filas/totales antes/después);
  borrar `handlers/bi_helpers.go`.
- Gates 1C: EXPLAIN comparado + benchmark simple (p95 por endpoint con `hey`/`ab` contra el
  bin local) + `go test ./...` + lint + build.

#### 1D — Auditoría activable + RBAC (3 días)

- [ ] **T17 · Write-path de auditoría (P0-5)**: middleware/hooks que capturen eventos y
  escriban async — worker con canal buffer + INSERT por lote (sin bloquear el request path),
  flush en graceful shutdown, sampling configurable. Eventos mínimos: login/logout (éxito y
  fallo), escrituras core (ventas, compras, pagos, anulaciones), ajustes de stock/precio,
  cambios de permisos/usuarios. Capturar request_id (ya en ctx), claims, status, duración.
  LogAction queda como API interna del middleware.
- [ ] **T18 · RBAC coherente (BR-6, B-8 viejo)**: completar `module_permissions.go`
  (`/vat-prorrateo`, `/sales-analytics/discounts`) y generar/testear el catálogo contra los
  gates reales (test que falle si drift); decidir la única fuente para auditoría FE
  (`audit:read` vs RoleGuard F2VLso) y documentarla.
- Gates 1D: integration test del middleware (escribe batch, no bloquea, samplea) + test de
  catálogo.

#### 1E — Calidad, contrato y tests (4-5 días)

- [ ] **T19 · Contratos tipados (BQ-2)**: eliminar los 9 `map[string]interface{}` → structs
  del dominio (LowStockProduct, OutOfStockProduct, NegativeMarginProduct, ExpiringBudget,
  HealthScore, PeriodComparison, ClientRisk, PaymentCapacity, SupplierAnalysis) +
  `Details map[string]interface{}` de DashboardSummary/KPIs → structs.
- [ ] **T20 · Forecast honesto (BQ-5)**: implementar EXPONENTIAL (suavizado simple/Holt) y
  SEASONAL (factores por período — la base ya existe en `detectSeasonality`), o devolver 400
  "method not supported" en vez de degradar a linear en silencio; exponer `method_effective`
  en la respuesta; documentar bandas ±15% y confidence cap; decidir canonical
  (`/forecast`) y deprecar el dual-mount `/api/v1/forecast` cuando el FE confirme (FE usa
  `/forecast` hoy → el BE queda con `/forecast` y remueve el alias).
- [ ] **T21 · Logging/métricas internas (BQ-4)**: log estructurado en repos (query lenta >
  umbral, con request_id), métrica Prometheus de duración por grupo de endpoint
  (`bi_query_duration_seconds{group=}`); pprof detrás de flag admin-only opcional.
- [ ] **T22 · Tests (BQ-1)**: service tests para los 7 servicios sin cobertura (mocks por
  puerto, estilo `equiv_test.go`); tests de rutas/gates de `analytics/http`; integration
  tests (DB real, docker) para las queries corregidas en T1/T2/T4/T11/T12/T13 — el E2E es el
  único gate que ejercita SQL real de repos mockeados (lección FASE 5 pedidos).
- [ ] **T23 · Pureza de dominio (BQ-3, incremental)**: en lo tocado, mover structs de
  `models/` al dominio analytics; quitar `json:` tags de discounts.go o mover al handler;
  eliminar `getPeriodDates` de audit.go usando `Period`.
- **GATE FINAl FASE 1 (duro)**: `go test ./...` verde con la cobertura nueva · `golangci-lint
  run` · `make build` · smoke dev-JWT de los 11 grupos · EXPLAIN/benchmark documentado ·
  reporte de cierre en `business_management/conductor/AUDIT_BI_BACKEND_2026-09.md`.
  ⛔ Recién aquí arranca la FASE 2 (frontend).

### FASE 2 — Auditoría por página FE (3-4 días; requiere FASE 1 ✅)

- [ ] Ejecutar protocolo §3 para las ~36 páginas, en orden de riesgo: CxP (6) → CxC (6) →
      Dashboard (5) → Sales Analytics (6) → Pronósticos (5) → Rentabilidad (6) → Inventario
      (4) → Reportes Financieros (8) → Auditoría (4)
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

- [ ] Cada página legacy tocada sustancialmente se migra a `.tsx` + FSD con hooks puros y
      domain fuera de componentes (expectativa full-alignment del workspace)
- [ ] Checklist DESIGN §10 + §6.7 + §6.3 por página migrada
- [ ] Auditoría `vercel-react-best-practices` del área → reporte en
      `erp-webapp/conductor/AUDIT_REACT_PERFORMANCE_BI_FRONTEND.md`
- [ ] i18n: sin strings nuevos hardcoded

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
