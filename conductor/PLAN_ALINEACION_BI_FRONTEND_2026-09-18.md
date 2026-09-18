# PLAN — Alineación completa del módulo BI del frontend con DESIGN.md + AGENTS.md

**Fecha**: 2026-09-18 · **Repo afectado**: `erp-webapp` (rama `dev`) · **Tipo**: 100% frontend, sin cambios de BE ni de contratos API.
**Contexto**: sucesor directo de `PLAN_AUDIT_BI_INTELIGENCIA_NEGOCIOS_2026-09-14` (cerrado 5/5 y purgado el 2026-09-17). Ese plan dejó los **datos honestos** (fixes P0/P1/P2), migró a `.tsx`/FSD **solo 6 grupos** (auditoría, financial-summary, cash-flow, supplier-analysis, sales-analytics, pronósticos demanda/ingresos) y añadió 78 tests + retro-i18n `bi.*` (296 keys). Este plan cierra la **deuda restante**: todo lo que quedó en `.jsx`/`.js` sin FSD, sin i18n y con tokens fuera de DESIGN.md.

**Objetivo**: todas las páginas del módulo BI cumplen `erp-webapp/DESIGN.md` (tokens, tipografía semántica, componentes `ui/`, 3 estados de datos) y `AGENTS.md` (TypeScript, Feature-Sliced, lógica de negocio en `src/domain/`, i18n sin strings hardcoded, tests), sin cambiar comportamiento funcional.

**Fuera de alcance**: backend, URLs/contratos de endpoints, permisos RBAC (ya cubiertos: todas las rutas BI tienen guard), cambios de funcionalidad de negocio, nuevas features.

---

## §1 Diagnóstico — estado actual (verificado 2026-09-18, rama `dev`)

### 1.1 Métricas globales del scope BI legacy

| Métrica | Valor |
|:--------|:------|
| Páginas `.jsx` legacy del módulo | **19** (~6.800 LOC) |
| Features legacy sin migrar | 4 (receivables, accounts-payable†, bi-forecasting‡, profitability) |
| Código muerto confirmado | **1.197 LOC** (ver 1.4) |
| Clases de color genéricas (`gray/slate/blue-NNN`…) | **2.977** |
| Escalas de texto genéricas (`text-xs/sm/lg…`) | **894** |
| hex/rgba literales | **695** |
| Sombras genéricas (`shadow-md/lg/xl`) | **312** |
| `font-bold/semibold` sueltos | **790** |
| Gradientes | 12 |
| Tests de las 19 páginas | **0** (baseline global: vitest 734/734, tsc 0, build index ~940 kB) |
| Módulos `src/domain/` de BI | **0** (no existe nada) |

† accounts-payable: `SupplierAnalysis/*` ya migrado (.tsx, con tests) — es el patrón de referencia interno.
‡ bi-forecasting: `PronosticoDemanda.tsx` / `PronosticoIngresos.tsx` ya migrados (patrón de referencia interno).

### 1.2 Inventario de páginas legacy (19)

| # | Página (src/pages/) | Ruta | LOC | i18n | Riesgos notables para la migración |
|:--|:--------------------|:-----|:----|:-----|:-----------------------------------|
| 1 | `Dashboard.jsx` | `/dashboard` | 519 | parcial | **import eager en App.tsx L28** (rompe el lazy, arrastra recharts al chunk inicial); store con mock demo en catch; Exportar sin handler; `onClick` en divs |
| 2 | `DetailedKPIs.jsx` | `/dashboard/kpis` | 370 | parcial | `getTimeAgo` duplicado de Dashboard; barra de salud inventada (`turnover*10` con TODO) |
| 3 | `SalesHeatmap.jsx` | `/dashboard/sales-heatmap` | 341 | **hardcode total** | builder de matriz heatmap entero inline; widget "Sucursales Activas" muerto; hex crudos |
| 4 | `ConsolidatedAlerts.jsx` | `/dashboard/alerts` | 504 | **hardcode total** | god component; severity mapping triplicado; botones "Marcar todo como leído"/"Silenciar" muertos |
| 5 | `TopProductsOverview.jsx` | `/dashboard/top-products` | 369 | parcial | checkboxes sin estado; Exportar sin handler; paginación dummy |
| 6 | `ReceivablesDashboard.jsx` | `/dashboard/receivables` **y** `/receivables` | 179 | parcial | **doble montaje con guards distintos** (D3); gráfico SVG artesanal; posible crash si summary null |
| 7 | `PayablesDashboard.jsx` | `/dashboard/payables` | 218 | parcial | consume `src/hooks/usePayables.js` (fuera del feature); 5 useMemo de transformación → domain |
| 8 | `PayablesAgingReport.jsx` | `/payables/aging-report` | 618 | **hardcode total** | god component; **clases Tailwind dinámicas `h-[${h}%]` que no compilan** (dead visual); mini-chart DPO con alturas hardcodeadas; gauge con clipPath hack; 4 botones muertos |
| 9 | `InvoicesMasterList.jsx` | `/payables/invoices` | 533 | **hardcode total** | debounce `setTimeout` inline (existe `useDebounce`); listener resize manual (existe `use-mobile`); sin empty state |
| 10 | `InvoiceDetail.jsx` | `/payables/detail/:id` | 373 | **hardcode total** | botones PDF/Registrar Pago muertos; `return null` sin invoice |
| 11 | `OverdueAccounts.jsx` | `/receivables/overdue` | 119 | parcial | **`window.location.reload()` como retry**; botones stub |
| 12 | `ReceivablesMasterList.jsx` | `/receivables/list` | 110 | parcial | bajo riesgo |
| 13 | `ReceivableDetail.jsx` | `/receivables/detail/:id` | 136 | parcial | **`RegisterSalePaymentModal.tsx` (927 LOC) importado estático dentro de página lazy**; `window.location.reload()` retry |
| 14 | `AgingReport.jsx` | `/receivables/aging-report` | 105 | parcial | error no manejado (hook no lo expone) |
| 15 | `CashFlowAnalysisDashboard.jsx` | `/finance/analytical-cash-flow` | 632 | **hardcode total** | **useMemo de 140 líneas** con fallbacks/cálculos → domain (prioridad máxima); flag `VITE_USE_DEMO` residual |
| 16 | `TaxManagementDashboard.jsx` | `/finance/tax-management` | 391 | **hardcode total** | merge/dedup/sort de series mensuales inline → domain; botón "Formulario 120" muerto |
| 17 | `ProfitAndLoss.jsx` | `/finance/profit-and-loss` | 285 | **hardcode total** | **donut chart estático fake** (no refleja datos); empty = `return null` |
| 18 | `AuditLogDetail.jsx` | `/auditoria/logs/:id` | 190 | **hardcode total** | **datos fake hardcodeados** ("POST /api/v1/sales", correlation-id "f9a2-55d1…"); fetch manual sin hook; locale es-ES |
| 19 | `sales-analytics/ProductsCategories.jsx` | `/sales-analytics/products-categories` | 351 | **hardcode total** | única `.jsx` de su grupo; **"+12% vs mes anterior" falso**; fetch sin cancelación (carrera al cambiar período) |

Ninguna de las 19 usa `PageHeader`, `EmptyState`, `ErrorState`, `GenericSkeletonList` ni `text-data-mono` (los 5 existen en `src/components/ui/` con tests). Ninguna tiene page-test.

### 1.3 Inventario de features legacy

| Feature | LOC | Estado | Deuda principal |
|:--------|:----|:-------|:-----------------|
| `features/profitability/` | 2.129 | 6 monolitos page-component (302–384 LOC c/u) | **0 i18n**, **286 hex inline** (peor área), `formatPYG`/`formatPercent` re-definidos en los 6 archivos, hook `useProfitability.js` sin guard anti-carrera con dispatch por string, service feature-local fuera de `services/bi/`, 0 tests, 0 types |
| `features/receivables/` | 2.995 | 22 componentes + 6 hooks | 11/22 componentes sin i18n, 104 hex, 4/6 hooks sin guard anti-carrera, 5 `transform*` inline con tolerancias drift, sin barrel/types/tests; error del hook de aging no expuesto |
| `features/accounts-payable/` (legacy vivo) | ~880 + `usePayables` 370 | 6 componentes vivos | **0 i18n**, 30 hex, `usePayables.js` vive en `src/hooks/` con `translateStatus/translatePriority` ES inline y `formatDate` duplicado; sin guard; paginación simulada |
| `features/bi-forecasting/` (legacy) | 561 + hook 88 | 4 archivos | 0 i18n en los 4; hook exporta `formatCurrency/formatNumber` duplicando `utils/currencyUtils`; `deps` con `JSON.stringify(params)`; `KpiCard` triplicado; barrel no exporta el Nav |
| `pages/InventoryAnalytics/` + `components/InventoryAnalytics/` | ~2.100 | ya `.tsx` | 0 i18n (~40+ strings), 13 `any`, cálculo ABC inline en `InventoryDashboard.tsx:90-106`, fetch con `useEffect` directo sin hooks de feature, 0 tests |

### 1.4 Código muerto confirmado (eliminar, no migrar)

| Archivo/directorio | LOC | Evidencia |
|:-------------------|:----|:----------|
| `components/business-intelligence/` completo (6 archivos) | 469 | grep de `business-intelligence` en todo `src/`: 0 consumidores externos; contiene mock Stitch residual (`'$1,240,500'`) |
| `features/accounts-payable/components/AgingReport/*` (4) | 552 | nadie los importa; la ruta real `/receivables/aging-report` usa componentes de **receivables** |
| `features/accounts-payable/hooks/useAgingReport.js` | 176 | sin consumidores; encima llama a `receivablesService` (dominio equivocado) y fabrica `dpo = 42` hardcodeado |
| `services/mocks/receivablesMock.js` | — | sin importadores directos (verificar en F0 antes de borrar) |

### 1.5 Transversal

- **Hooks raíz fuera de feature**: `src/hooks/usePayables.js` (~370) y `src/hooks/useFinancialReports.js` (~340, 9 fetchers con un único `loading` compartido, sin guard, toasts ES). `useFinancialReports` lo consumen las 3 páginas `/finance/*` legacy **y** `FinancialSummaryDashboard.tsx` ya migrada.
- **Servicios**: `dashboardService.ts` es el modelo a seguir. `receivablesService.ts` (28 `any`) y `payablesService.ts` duplican aliases de compatibilidad (`getSummary`/`getStatistics`→`getOverview`, `getDetailedAging`/`getAging`→`getAgingReport`, `getOverdueAccounts`→`getOverdue`, `getClientRiskAnalysis`→`getClientRisk`) usados arbitrariamente por los hooks. `financialReportsService.ts` casi todo `Promise<any>`. `profitabilityService.js` vive dentro del feature y `biForecastingService.ts` (615 LOC, TS sin `any`) vive en `services/` raíz — ambos fuera de `services/bi/`.
- **Store** (`src/store/useDashboardStore.ts`): selectores atómicos OK (H2) y guard anti-carrera por seq-counters OK (H3) — conservar. Deuda: `loading`/`error` **globales compartidos por 4 fetchers** (interferencia entre vistas) y **mock demo en el catch de los 4 fetchers** (`DEMO_MODE` + `getDemoDashboardData`).
- **i18n**: namespace `bi.*` con 296 keys es/en (post-auditoría) — extenderlo para las migraciones nuevas; namespaces legacy `dashboard.*`/`receivables.*`/`payables.*` conviven.
- **Guards RBAC**: todas las rutas BI tienen `PermissionGuard` con el permiso correcto (gap del audit ya cerrado). Única inconsistencia: `ReceivablesDashboard` montado en 2 rutas con guards distintos (D3).
- **`utils/currencyUtils.js` sigue siendo `.js`** siendo el formatter compartido de todo BI.

---

## §2 Contrato de migración — checklist binaria por archivo tocado

Toda página/componente/hook migrado en las FASES 2–6 DEBE cumplir (sin excepciones):

1. **TypeScript**: archivo renombrado a `.tsx`/`.ts`; props y contratos de API tipados con interfaces locales documentando el envelope del endpoint (patrón `features/cash-flow/hooks/useCashFlow.ts`). Si un consumidor sigue siendo `.jsx`, los tipos se aportan vía JSDoc (load-bearing para tsc).
2. **i18n**: todo string visible vía `t(key, fallbackEspañol)`; keys nuevas bajo `bi.*` agregadas a `locales/es` y `locales/en` **en el mismo diff**. Errores de hooks con `tRaw`.
3. **Tokens DESIGN (§2–§5 de DESIGN.md)**: 0 hex/rgba, 0 `gray/slate/…-NNN`, 0 escala genérica (`text-xs/sm/2xl…`), 0 `shadow-md/lg/xl`, 0 gradientes, 0 `font-bold` suelto. Solo clases semánticas (`bg-surface`, `text-on-surface-deep`, `text-label-caps`, `shadow-whisper`…).
4. **Estructura de página**: `PageHeader` + contenedor `max-w-container-max`; números/fechas/IDs con `text-data-mono font-data-mono` (tablas financieras densas: `text-data-tabular`).
5. **3 estados de datos**: loading con `GenericSkeletonList` (o skeleton que imite la forma final), empty con `EmptyState` (con acción si aplica), error con `ErrorState` + `onRetry`. Prohibido `return null` como empty y `console.error` sin UI.
6. **1 solo `variant="primary"`** por vista; **botones/widgets sin handler se eliminan** (criterio H7 del plan anterior: cablear o quitar — aquí: quitar salvo decisión contraria explícita).
7. **Lógica de negocio fuera del componente**: cálculos, mapeos API→vista, formatters → `src/domain/<feature>/` (§4 FASE 1). El componente solo presenta. Reutilizar `utils/currencyUtils` — prohibido re-definir `formatPYG`/`formatCurrency`/`formatPercent` inline.
8. **Hooks**: guard anti-carrera (patrón seq-counter del store o `isMounted`); sin mock/fallback demo en catch; refetch expuesto para `onRetry`; contracts con tolerancias mínimas documentadas.
9. **Charts**: recharts (ya en deps) bajo guard `isClient`. Prohibido SVG artesanal con datos fabricados y clases Tailwind dinámicas (`h-[${…}%]` no compila).
10. **Patterns**: debounce → `useDebounce`; mobile → `use-mobile`; retry → refetch (prohibido `window.location.reload()`); modales pesados → `React.lazy` dentro de la página lazy.
11. **Tests**: page-test en `src/pages/__tests__/` por página migrada (patrón `SalesAnalyticsDashboard.page.test.tsx`), hook-test si el hook tiene lógica, domain-test en FASE 1. **Componente retirado → DELETE de su test** (nunca skip).
12. **Feature slice**: `components/` + `hooks/` + `types.ts` + `__tests__/` + barrel `index.ts` con exports nombrados; App.tsx importa del barrel vía lazy.

**Gates por fase** (todos obligatorios antes del commit de cierre de fase):
`npx vitest --run` 0 fallos · `npx tsc --noEmit` 0 errores · `pnpm build` verde · `pnpm lint:design` verde · `pnpm tokens:check` verde. Si `main` avanzó mid-session: rebase antes de correr los gates.

---

## §3 Decisiones (recomendación marcada; ejecutar la recomendación salvo indicación contraria)

| # | Decisión | Recomendación | Justificación |
|:--|:---------|:--------------|:--------------|
| D1 | Mock demo en el catch de los 4 fetchers de `useDashboardStore` | **Eliminar** | Regla FE-1 del plan anterior ("sin endpoint no se muestra") y criterio F3H ya aplicado al resto del BI. En modo API sana no dispara, pero es una trampa en prod. |
| D2 | `loading`/`error` globales del store | **Dividir por recurso** (`loadingBySlice`/`errorBySlice` o stores separados por vista) | Hoy 4 fetchers se pisan el estado entre sí; las 5 páginas dashboard lo sufren. |
| D3 | `ReceivablesDashboard` doble montaje (`/dashboard/receivables`→`dashboard:read` vs `/receivables`→`receivables:read`) | **Unificar guard a `receivables:read` en ambas rutas**, componente único | Es contenido CxC; `dashboard:read` deja pasar a ver cobranzas sin el permiso específico. Sin redirect (mantiene ambas URLs). |
| D4 | `Dashboard.jsx` import eager en App.tsx L28 | **Convertir a lazy** como sus hermanas | Continúa H1 (index 3,4 MB→940 kB); Dashboard arrastra recharts al chunk inicial. Verificar que la landing siga correcta. |
| D5 | `pages/InventoryAnalytics/` + `components/InventoryAnalytics/` | **Pulir in situ** (i18n, `any`, ABC→domain, tests) **sin mover directorios** | Consistente con sales-analytics post-FASE 4 (páginas en `pages/<Grupo>/`); el churn de rutas/imports no aporta valor funcional. |
| D6 | Namespaces i18n | **Strings nuevos bajo `bi.*`**; namespaces legacy (`dashboard.*`, `receivables.*`, `payables.*`) se migran solo en archivos ya reescritos | Evita re-traducción masiva; consistente con la convención post-auditoría. |

---

## §4 Fases

> Orden elegido: primero limpiar y poner cimientos (F0–F1), luego dominios de mayor deuda y menor acople (rentabilidad no comparte nada), después las dos features de cuentas (CxP antes que CxC por tamaño), dashboard core (store) y finanzas + remanentes. Cada fase deja el sistema shipeable.

### FASE 0 — Limpieza y cimientos estructurales (sin cambios visibles de UI)

1. **Eliminar código muerto** (§1.4): `components/business-intelligence/` (6 archivos, 469 LOC), `features/accounts-payable/components/AgingReport/*` (4, 552 LOC), `features/accounts-payable/hooks/useAgingReport.js` (176), `services/mocks/receivablesMock.js` y `services/mocks/salesAnalyticsMock.js` (ambos con 0 importadores — ver `audits/bi-2026-09/VERIFICACION_POST_CIERRE_2026-09-18.md` pendiente 2). Commits separados por grupo con la evidencia del grep en el mensaje.
2. **D4**: `Dashboard.jsx` eager → `lazy()` en App.tsx (mantener `DashboardRoute`). Medir chunk index antes/después (esperado < 940 kB).
3. **D3**: unificar guard de `/dashboard/receivables` a `receivables:read`.
4. `utils/currencyUtils.js` → `currencyUtils.ts` tipado (firmas `formatPYG`, `formatNumber`…). Ajustar consumidores si tsc lo exige.
5. **Consolidación de servicios**: mover `features/profitability/services/profitabilityService.js` → `services/bi/profitabilityService.ts` (tipado) y `services/biForecastingService.ts` → `services/bi/`. Actualizar imports.
6. **Unificación de aliases** `receivablesService`/`payablesService`: fijar método canónico por endpoint (`getOverview`, `getAgingReport`, `getOverdue`, `getClientRisk`), marcar aliases `@deprecated` y migrar los consumers en el mismo diff; reducir `any` de los métodos tocados.
7. Purga de residuos señalados: flag `VITE_USE_DEMO` de CashFlow/TaxManagement (decisión D1-adjacente: fuera), strings fake de `AuditLogDetail` (POST hardcodeado, correlation-id) — se eliminan al migrar esa página en F6; aquí solo documentar.

**Gate F0**: vitest 734/734 (sin nuevos tests) · tsc 0 · build · lint:design · bundle index ↓ documentado.

### FASE 1 — Dominio BI (`src/domain/`)

Crear los módulos con tests unitarios puros (sin React), extrayendo **del código actual** (los hooks/components legacy pasan a importarlos — sin cambio de comportamiento):

| Módulo | Extracciones (origen) |
|:-------|:----------------------|
| `domain/receivables/aging.ts` | buckets 0/30/60/90 y totales por cliente (`AgingByClientTable.jsx:64-76`); buckets derivados de facturas (`useClientCreditProfile.js:99-125`); `getRiskLevel` 30/60/90 (rescatado del AP muerto) |
| `domain/receivables/mappers.ts` | `transformReceivableItem`, `transformApiResponse`, `transformDetailData`, `transformSummary` (5 hooks distintos) |
| `domain/receivables/risk.ts` | `getPriority` (>60/>30), `calculateStats` (eficiencia, riesgo) de `useOverdueAccounts.js:9-99`; `getRiskStyles` de `RiskGauge.jsx` |
| `domain/payables/statusLabels.ts` | `translateStatus`/`translatePriority` de `usePayables.js` (→ claves i18n, no ES hardcodeado) |
| `domain/payables/aging.ts` | `agingKpis`, `distributionSegments`, clasificación de riesgo Crítico/Moderado/Mínimo de `PayablesAgingReport.jsx` |
| `domain/payables/dashboard.ts` | los 5 useMemo de transformación de `PayablesDashboard.jsx` (KPIs, aging, pagos próximos, proveedores, fechas es-PY) |
| `domain/profitability/margins.ts` | margen %, contribution share, ranking, formato compacto `(x/1_000_000).toFixed(1)M` (`SellerProfitability.jsx:150,323`) |
| `domain/finance/cashFlow.ts` | el useMemo de 140 líneas de `CashFlowAnalysisDashboard.jsx` (fallbacks, nets por categoría, min/max balance, `getBalancePosition`, `formatSignedPYG`) — prioridad máxima |
| `domain/finance/vat.ts` | merge/dedup/sort mensual + deltas `pct(current,prev)` de `TaxManagementDashboard.jsx` |
| `domain/dashboard/heatmap.ts` | `uiIndexToApiDay`, `getIntensity*`, builder de matriz de `SalesHeatmap.jsx` |
| `domain/dashboard/alerts.ts` | `getCategoryIcon/Label`, heurísticas de claves financieras, filtro combinado de `ConsolidatedAlerts.jsx` |
| `domain/dashboard/shared.ts` | `getTimeAgo` (hoy dup en Dashboard + DetailedKPIs), `getActivityRoute` |
| `domain/inventory-analytics/abc.ts` | cálculo de valor ABC de `InventoryDashboard.tsx:90-106` |
| `domain/sales-analytics/format.ts` | `roundPct1`, paginación manual de `ProductsCategories.jsx` |

**Gate F1**: tests de cada módulo de domain (table-driven, casos borde: null, división por cero, buckets límite) · vitest total crece y 0 fallos · tsc 0.

### FASE 2 — Rentabilidad (6 páginas + feature completo)

Migrar `features/profitability/` completo con el contrato §2:
- 6 monolitos → componentes divididos por página (KPI grid / tabla / chart), `KpiCard` único del feature (hoy triplicado), skeletons compartidos.
- `hooks/useProfitability.js` → `hooks/useProfitability.ts` tipado con guard anti-carrera; quitar dispatch por string (hooks específicos por recurso o unión tipada).
- i18n: `bi.profitability.*` nuevo (hoy 0%); 286 hex → tokens semánticos según tablas DESIGN §2.
- 6 page-tests + hook-test + domain ya cubierto por F1.

### FASE 3 — CxC (feature receivables + 6 páginas)

- Feature: 22 componentes → `.tsx` + i18n (11 sin i18n hoy), 104 hex → tokens, 6 hooks → `.ts` con guard (4 sin guard) importando `domain/receivables/*`, barrel + types.
- Páginas: `ReceivablesDashboard` (gráfico SVG artesanal → recharts; guard de summary null), `ReceivablesMasterList` (bajo riesgo), `ReceivableDetail` (modal 927 LOC → `React.lazy`; `window.location.reload()` → refetch), `OverdueAccounts` (ídem retry), `AgingReport` (exponer error del hook).
- El error del hook de aging debe exponerse a la página (hoy lo traga).

### FASE 4 — CxP (feature accounts-payable + usePayables + 4 páginas)

- `src/hooks/usePayables.js` → `features/accounts-payable/hooks/usePayables.ts` (guard, tipado, `formatDate` único, statusLabels vía domain).
- 6 componentes vivos → `.tsx` + i18n (hoy 0%).
- Páginas: `PayablesDashboard` (transformaciones ya en domain F1), `PayablesAgingReport` (god 618 LOC: dividir en KPIs/distribución/tabla/gauge; mini-chart DPO con clases dinámicas no compiladas → recharts o eliminar; 4 botones muertos fuera), `InvoicesMasterList` (debounce inline → `useDebounce`; resize manual → `use-mobile`; empty state), `InvoiceDetail` (botones PDF/Pago muertos fuera o cablear — decisión en fase: recomendación fuera).

### FASE 5 — Dashboard core (5 páginas + store)

- **Store (D1 + D2)**: quitar mock demo de los 4 catch; dividir `loading`/`error` por recurso manteniendo el guard seq-counter y los selectores atómicos existentes. Actualizar los 5 consumers a los nuevos selectores.
- Páginas (`Dashboard`, `DetailedKPIs`, `SalesHeatmap`, `ConsolidatedAlerts`, `TopProductsOverview`) → `.tsx` con contrato §2 completo: PageHeader, 3 estados, data-mono, tokens (hoy ~105–157 violaciones c/u).
- Limpiezas señaladas: widget "Sucursales Activas" muerto (SalesHeatmap) fuera; Exportar/marcar-leído/silenciar/checkboxes muertos fuera; barra de salud inventada (DetailedKPIs) fuera o con fórmula documentada en domain + key i18n.

### FASE 6 — Finanzas + remanentes

- `useFinancialReports.js` → tipar y dividir por recurso (`features/financial-reports/hooks/`: `useIncomeStatement`, `useVatReport`, `useCashFlowReport`…), cada uno con su `loading`/`error` + guard; toasts ES → `tRaw`/`bi.*`.
- Páginas: `CashFlowAnalysisDashboard` (usa `domain/finance/cashFlow.ts`), `TaxManagementDashboard`, `ProfitAndLoss` (**donut fake fuera**; empty `return null` → `EmptyState`).
- bi-forecasting legacy: `DashboardPronosticos`, `PronosticoVentas`, `SaludInventario`, `BIForecastingNav` → `.tsx` + i18n copiando el patrón de `PronosticoDemanda.tsx`; formatters dup fuera; `KpiCard` compartido; barrel completo.
- `ProductsCategories.jsx` → `.tsx` (cierra el grupo sales-analytics 6/6); **"+12%" falso fuera**; cancelación de fetch al cambiar período.
- `AuditLogDetail.jsx` → `.tsx` + hook de feature + datos fake hardcodeados fuera + locale es-PY.
- InventoryAnalytics (D5, pulido in situ): i18n completo, 13 `any` → tipos, ABC → `domain/inventory-analytics/abc.ts`, tests de las 4 páginas.

### FASE 7 — Cierre integrado

1. Gates totales: `npx vitest --run` (suite ≥ 734 + ~80-100 nuevos) · `tsc --noEmit` 0 · `pnpm build` · `pnpm lint:design` · `pnpm tokens:check`.
2. **Gate de diseño BI**: re-correr los greps del §1.1 sobre el scope — esperado **0** en todos los contadores (colores genéricos, text-scale, hex/rgba, sombras, gradientes).
3. **Gate de completitud**: 0 archivos `.jsx`/`.js` vivos en el scope BI (páginas, features, hooks, components, services); 19/19 páginas con PageHeader + 3 estados + data-mono + i18n.
4. Verificación visual por screenshots (protocolo FASE 2 del plan anterior: admin + spot-check vendedor VNDR01) sobre una muestra por bloque.
5. Reporte de performance (vercel-react-best-practices) en `erp-webapp/conductor/` si se tocaron hot paths; purge de este plan del conductor.

**Estimación**: F0 1 sesión · F1 1–2 · F2 2 · F3 2–3 · F4 2–3 · F5 2 · F6 2–3 · F7 1 → **13–17 sesiones**.

---

## §5 Riesgos y mitigaciones

| Riesgo | Mitigación |
|:-------|:-----------|
| Regresión visual/funcional en 19 páginas | Una fase por dominio, gates por fase, cero cambios de comportamiento (solo estructura/visual), screenshots de cierre |
| Contratos API frágiles (tolerancias `response.data \|\| response` dispersas) | Interfaces de contrato documentadas en cada hook (patrón cash-flow) + smoke con dev-JWT antes de dar por buena una página |
| `main` avanza mid-session | Rebase antes de cada gate (lint:design compara contra ref) |
| Ciclos domain↔features | `src/domain/**` sin imports de React ni de features (regla binaria, se verifica en review) |
| JSDoc load-bearing | Al tipar `currencyUtils` y movers de servicios, correr tsc completo — los consumidores `.jsx` dependen de JSDoc para pasar |
| `useFinancialReports` compartido con `FinancialSummaryDashboard.tsx` ya migrada | Migración por-recurso con re-export temporal o actualización del consumer en el mismo diff |
| Bundle regressions | Medir chunk index en F0 (lazy Dashboard) y en F7; presupuesto ≤ 940 kB |

## §6 Criterios de cierre del plan

- [ ] 19/19 páginas migradas a `.tsx` con contrato §2 completo (checklist binaria por página en el commit de cierre de cada fase).
- [ ] 4 features legacy migrados + InventoryAnalytics pulido; 0 `.jsx`/`.js` vivos en scope BI.
- [ ] 1.197 LOC muertos eliminados; 0 botones/widgets decorativos en el módulo.
- [ ] `src/domain/` con los módulos BI del F1 y tests propios.
- [ ] Store sin mock demo y con loading/error por recurso.
- [ ] Gates de §4 FASE 7 en verde, incluido el gate de diseño BI en 0.
- [ ] Suite de tests global en verde con ≥ +80 tests nuevos de BI.

---

## §7 Adendum — verificación post-cierre del PLAN_AUDIT_BI (2026-09-18)

Verificación independiente de los compromisos del plan de auditoría BI cerrado: reporte completo
en `erp-webapp/conductor/audits/bi-2026-09/VERIFICACION_POST_CIERRE_2026-09-18.md`. Hallazgos
que este plan ya absorbe: fallback demo del store (F5/D1), código muerto P1-9 parcial —
`business-intelligence/` + `receivablesMock`/`salesAnalyticsMock` huérfanos (F0), botones muertos
en legacy (H7 a medias, contrato §2.6), "+12%" de ProductsCategories, fabricaciones de
AuditLogDetail y donut fake de PyG (F6), Dashboard eager (F0/D4), doble montaje receivables
(F0/D3), i18n legacy (todo el plan). **Ajustes al alcance por esa verificación**: la paginación
server-side pendiente se explicita — tablas de Insights (`CustomerSellerInsights`) y top-productos
de `sales-analytics/Dashboard` entran en F6 junto con `SaludInventario`, `PronosticoVentas`/
`PronosticoIngresos` y `OverdueAccounts` (regla ≤10 filas del plan anterior, nunca completada
para esas páginas). Fuera de este plan queda la **release note BC-1 (timezone)** — tarea suelta de
docs/BE registrada en el reporte de verificación.
