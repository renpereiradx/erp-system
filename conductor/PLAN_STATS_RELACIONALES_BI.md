# PLAN — Estadísticas relacionales BI (producto ↔ cliente ↔ proveedor)

**Estado:** 📋 Activo — aprobado para implementación (F0 cerrado con este documento)
**Fecha:** 2026-09-21
**Alcance:** cross-repo (business_management + erp-webapp). Coordinación en este `conductor/` raíz.
**Origen:** pedido del owner + evaluación del mismo día verificada contra código.
**Requerimientos:** `docs/REQ_BIPACK_LICENCIAMIENTO_BI.md` v1.1 — RF-BIPACK-021..024 (este plan
los implementa).
**Precedentes de patrón:** T9 paginación server-side (`by-seller`, `COUNT OVER`, caps), drift test
RBAC, FASE 10 (repos read-only en `internal/analytics`).

---

## 1. Objetivo

Cuatro familias de consultas **drill-down relacionales** que hoy no existen en ninguna forma:

1. **Producto → compradores:** qué clientes compraron un producto (filtros + paginación).
2. **Cliente → productos:** qué productos compra más un cliente (filtros + paginación).
3. **Proveedor → productos:** qué productos se compran más de un proveedor (ídem).
4. **Producto → proveedores / mejor precio:** qué proveedores ofrecen un producto y a qué
   precio (último, promedio, mínimo histórico), ordenable.

Todas quedan **dentro del módulo BI** (= pack premium, ver D1): heredan el gate de licencia,
`analytics:read`, rate-limit, ETag y métricas de `biChain` sin tocar una línea de licenciamiento.

## 2. Antecedentes verificados contra código (2026-09-21)

- `by-product` / `by-customer` (`internal/analytics/http/routes.go:212-217`) son **rankings
  agregados**, no drill-downs. `payables/supplier/{id}/analysis` (línea 156) es análisis
  financiero de deudas. **No existe ningún analytics de compras por producto.**
- Fuentes de datos: `transactions.sales_orders` + `sales_order_details`;
  `transactions.purchase_orders` + `purchase_order_details`; clientes/proveedores unificados en
  `users.parties`. El join canónico que excluye ventas anuladas vive en
  `internal/platform/postgres/analytics_sql.go` (`salePaymentsPaidJoin`).
- Estados de compra: `PENDING` / `COMPLETED` / `CANCELLED`
  (`internal/purchase/domain.go:14-16`) → "comprado" = `COMPLETED`.
- Patrón T9 de paginación server-side ya probado (lecciones: `COUNT OVER` con CTE bien formado —
  hotfix `135652b`; tests de handler con fake reader **no** ejecutan SQL → la integración debe
  correr contra DB real).
- `config.ModulePermissions` (`config/module_permissions.go:23+`) debe reflejar el prefijo nuevo
  o `tests/integration/rbac_catalog_drift_test.go` falla.
- FE: `TablePagination` compartido, bloques de rutas BI contiguos en `App.tsx` detrás de
  `BiPermissionGuard`, grupo nav BI condicionado por `biEnabled`.

## 3. Decisiones (cerradas en F0)

### D1 — Ubicación: todo dentro del módulo BI (= pack premium)

- 2 subrutas nuevas en el grupo `/sales-analytics` existente (endpoints 1 y 2).
- 1 grupo nuevo `/purchase-analytics` con `biChain("analytics:read", "purchase-analytics")`
  (endpoints 3 y 4) — completa el hueco de FASE 10 (nunca hubo analytics de compras).
- **Justificación:** "endpoint independiente pero premium" exigiría re-cablear a mano el mismo
  `RequireModule("bi")` + cadena de middlewares, inventar un permiso nuevo (migración + grants)
  y un menú aparte que igual se oculta sin pack: mismo licenciamiento, más trabajo y dos
  convenciones. Dentro de BI: cero migraciones, cero permisos nuevos, gating automático.
- Nota diferible: si algún día un perfil de compras sin `analytics:read` necesita "mejor
  precio" embebido en el flujo de compras, el endpoint ya es premium-gated y solo habría que
  replantear el permiso, no el licenciamiento.

### D2 — Granularidad: filas a nivel producto, filtro opcional `variant_id`

Lección de la búsqueda plana de variantes: el match por SKU de variante debe ser explícito.
Las filas agregan por producto; quien quiera el detalle de una variante filtra `variant_id`.

### D3 — Multi-moneda: comparación de precios solo en moneda base

El ranking de "mejor precio" compara exclusivamente compras en moneda base (PYG); las compras
en otras monedas quedan excluidas del ranking y se informan en un meta `excluded_other_currency`
(count). Los totales de los endpoints 1-3 no convierten moneda (columna informativa).

### D4 — Poblamiento: ventas no anuladas (join canónico); compras `COMPLETED`

### D5 — "Mejor precio": orden por defecto `last_unit_price` ascendente

Columnas `last_unit_price`, `avg_unit_price`, `min_unit_price` todas presentes y ordenables;
`avg` ponderado por unidades. Métrica de ranking por defecto: **último precio** (el promedio
histórico puede castigar a quien bajó precio recientemente).

### D6 — Índices solo si EXPLAIN lo exige

Verificar plan de consulta con EXPLAIN (convención de la casa); si hace falta, migración nueva
vía `make migrate-create` y en prod `CONCURRENTLY` siguiendo el patrón del runbook de BI.

## 4. Contrato de endpoints

Común a los cuatro (patrón T9): `GET`, respuesta `{rows, total, page, page_size}` con
`COUNT(*) OVER()`; filtros `date_from`/`date_to` (ISO date), `branch_id` (default: sucursal del
contexto), `q` (ILIKE nombre/identificación), `sort` (`+campo`/`-campo`), `page` (1-based,
default 1), `page_size` (default 10, **máx 100**).

| # | Endpoint | Params específicos | Fila (row) | Sort default |
|:--|:---------|:-------------------|:-----------|:-------------|
| 1 | `/sales-analytics/products/{product_id}/buyers` | `variant_id`, `payment_method_id`, `min_units` | `client_id, client_name, client_doc, purchases, units, total, avg_unit_price, last_purchase_at` | `-total` |
| 2 | `/sales-analytics/customers/{customer_id}/top-products` | `variant_id`, `category_id` | `product_id, product_name, product_sku, purchases, units, total, avg_unit_price, last_purchase_at` | `-units` |
| 3 | `/purchase-analytics/suppliers/{supplier_id}/top-products` | `variant_id`, `category_id` | ídem #2 sobre historial de compras `COMPLETED` | `-units` |
| 4 | `/purchase-analytics/products/{product_id}/suppliers` | `variant_id`, `min_purchases` | `supplier_id, supplier_name, supplier_doc, last_unit_price, avg_unit_price, min_unit_price, currency, units, purchases, last_purchase_at` + meta `excluded_other_currency` | `+last_unit_price` |

Errores: 404 si el recurso base (producto/cliente/proveedor) no existe (patrón `apperr` del
contexto); 400 en params inválidos.

## 5. Fases

> Gates de la casa: BE `make build` + `go test ./...` + `golangci-lint run`; FE
> `npx vitest --run` + `npx tsc --noEmit` + `pnpm build` + `pnpm lint:design`. Rebase previo si
> main/dev avanzaron a mitad de sesión.

### F0 — Decisiones y contrato (este documento)

- [x] D1-D6 cerradas, contrato congelado, RF-BIPACK-021..024 registrados en REQ v1.1.

### F1 — Backend

- [ ] Puertos en `internal/analytics/port.go`: extender el reader de sales-analytics (2
  drill-downs) + puerto nuevo `PurchaseAnalyticsReader`.
- [ ] Adaptador Postgres `analytics_purchase_repo.go` + extensión del repo de sales; joins sobre
  `transactions.*_orders` + `*_order_details` + `users.parties`; join canónico de anuladas;
  compras `COMPLETED`; moneda base en #4 (D3).
- [ ] Handlers: 2 en `http_sales.go` (o archivo nuevo del contexto) + `http_purchase.go`.
- [ ] Rutas: subrutas en el grupo `/sales-analytics`; grupo nuevo `/purchase-analytics` con
  `biChain("analytics:read", "purchase-analytics")`.
- [ ] `config.ModulePermissions`: entrada `"/purchase-analytics": {ReadPerm: "analytics:read", WritePerm: ""}`.
- [ ] Tests: unitarios de parseo/sort + **integración contra DB real** (lección `135652b`):
  paginación total consistente, filtros, anuladas/compras pendientes excluidas, moneda base en #4.
- [ ] EXPLAIN de las 4 consultas sobre datos de dev; índices vía `make migrate-create` solo si
  D6 lo pide.
- [ ] **Gate:** build + `go test ./...` + lint + drift test en verde.

### F2 — Frontend

- [ ] Feature `src/features/relational-analytics/` (componentes visuales + hooks) con dominio
  puro (sort/normalización de filas) en `src/domain/`; servicios en `src/services/bi/`.
- [ ] Rutas: #1/#2 dentro del bloque `/sales-analytics/*`, #3/#4 bloque nuevo — ambos tras
  `BiPermissionGuard permission='analytics:read'`.
- [ ] Nav: 2 entradas nuevas en el subgrupo "Analíticas de Ventas" + subgrupo nuevo "Analíticas
  de Compras" (2 entradas), todo dentro del grupo BI condicionado por `biEnabled`.
- [ ] Deep-links desde páginas Core: ficha de producto → "Ver compradores" (#1) y "Comparar
  proveedores" (#4); ficha de cliente → "Productos que compra" (#2); ficha de proveedor →
  "Productos suministrados" (#3). Sin pack, el gate redirige con aviso (ya resuelto).
- [ ] `TablePagination` compartido, estados 3 (loading/error/vacío), i18n `bi.relational.*`
  es/en (regla: la key existe en ambos locales en el mismo diff).
- [ ] Tests según `PLAN_TEST_DESIGN_FRONTEND` (pages/features ubicación canónica, mock en la
  frontera, sin mockear `lucide-react`).
- [ ] **Gate:** vitest + tsc + build + lint:design.

### F3 — Verificación en vivo y cierre de suites

- [ ] Smoke con dev-JWT (receta de la casa) sobre los 4 endpoints: 200, paginación consistente,
  filtros, y 403 `MODULE_NOT_LICENSED` con `LICENSE_ENFORCE=1` sin licencia (herencia del pack).
- [ ] Suites completas de ambos repos en verde.

### F4 — Cierre documental

- [ ] REQ_BIPACK: RF-021..024 → Implementado (mismos commits en trazabilidad).
- [ ] Registro de cierre en este README y purge del plan (convención de limpieza; los RF quedan
  como definición vigente en `docs/`).

## 6. Riesgos y mitigaciones

| Riesgo | Mitigación |
|:-------|:-----------|
| Semántica de "mejor precio" (monedas, unidades de medida, avg vs último) | D3/D5 cerradas en F0; `avg` ponderado por unidades; moneda base only |
| Perf sobre `*_order_details` grandes | EXPLAIN obligatorio en F1; índices solo si los pide (D6); prod CONCURRENTLY |
| Drift rutas ↔ `ModulePermissions` | Entrada nueva en F1 + drift test como gate |
| Tests con fake reader no ejecutan SQL (regresión real tipo `135652b`) | Integración contra DB real como gate de F1 |
| Granularidad variante confusa para el usuario | D2: filas por producto + filtro `variant_id` explícito |

## 7. Registro de progreso

| Fase | Estado | Notas |
|:-----|:-------|:------|
| F0 decisiones | ✅ Cerrada | Este doc + REQ_BIPACK v1.1 |
| F1 backend | ⏳ | |
| F2 frontend | ⏳ | |
| F3 smoke + suites | ⏳ | |
| F4 cierre documental | ⏳ | |
