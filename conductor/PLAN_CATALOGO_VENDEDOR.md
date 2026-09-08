# PLAN: Catálogo del Vendedor — Productos sin Información Sensible

**Fecha:** 2026-09-07
**Estado:** 🚧 FASES 1+2 IMPLEMENTADAS (2026-09-07; backend commit `676509d`, ver §9). FASE 3 diferida hasta que exista `/pedidos`.
**Alcance:** Cross-repo (backend `business_management` + frontend `erp-webapp`) → este plan vive en `conductor/` raíz
**Origen:** Pedido del owner: nueva UI para el usuario vendedor que NO lo exponga a la página de productos (que muestra precio de costo); el vendedor solo debe ver precio de venta y detalles no sensibles.
**Verificación:** Todas las afirmaciones de §2 fueron verificadas contra el código el 2026-09-07 (exploración de ambos repos con evidencia file:line).
**Relación:** complementa a `PLAN_PEDIDOS_MOSTRADOR_VENDEDOR_CAJA.md` (el buscador de productos del armador de pedidos hereda estas garantías).

---

## 1. Problema y objetivo

El vendedor (VNDR01) tiene `products:read` (necesario para buscar productos en el POS). Con eso hoy puede:

1. **Entrar por URL a `/productos`** (route guard `products:read`, `src/App.tsx:319`) — el ítem de nav está oculto para él (la sección "Logística e Inventario" gatea `inventory:read`, `navigation.ts:230`, que no tiene), pero la ruta funciona igual. La página muestra **columna "Costo de Compra"** y **chip de margen %** (`ProductsTable.tsx:102-106,167-199`), y el modal de detalle muestra **resumen de costos** (`last_cost`, `weighted_avg_cost_6m`, `cost_variance_percent`), **historial de costos**, **"Mejor Margen"** (`ProductDetailsModal.tsx:282-337,483-498`).
2. **Recibir el costo por API aunque la UI no lo pinte**: la búsqueda del POS (`GET /products/info/search/{name}`, `internal/catalog/http.go:485-508`) serializa `ProductFinancialEnrichedResponse` con `unit_costs_summary`, `financial_health` y `best_margin_percent` (`internal/catalog/dto.go:145-174`). Ídem `/products/{id}/info`, `/products/info/barcode/{barcode}`, `/products/{id}/financial`, `/products/financial/search/{name}`, `/products/{id}/sale`, `/{id}/purchase`.
3. **Consultar las rutas de costo del contexto pricing con solo `products:read`** (`internal/pricing/http/routes.go:35-67`): `/products/{id}/costs/*`, `/margin-alert`, `/margin-report`, `/cost-trends`, `/weighted-average` y —peor— **`/products/{id}/supplier-comparison`**, que devuelve `supplier_name` + `cost_per_unit` (`models/cost.go:88-100`). Bonus: el rol `CLNT01` (CLIENT) también tiene `products:read` (`20260519112753:116`) — la fuga no es solo del vendedor.

Además, las rutas de **Rentabilidad** (`/profitability/*`, márgenes por producto/vendedor) no tienen `PermissionGuard` (`App.tsx:302-307`; solo el nav las gatea con `analytics:read`).

**Objetivo:** (a) el vendedor no accede a la página de gestión de productos; (b) dispone de una **UI de catálogo** (solo venta + detalles no sensibles); (c) el costo/margen/proveedor deja de viajar por la API para quien no debe verlo — la UI es cosmética, la garantía real es server-side (patrón `sales:apply_discount` de FASE B del plan vendor-role).

## 2. Estado actual verificado (referencia)

### Qué es sensible y dónde

| Dato | UI hoy | API hoy |
|:--|:--|:--|
| `last_cost` / `weighted_avg_cost_6m` / `cost_variance_percent` | `ProductsTable` columna costo; `ProductDetailsModal` "Resumen de Costos" | `unit_costs_summary` en `/products/info/search/{name}` (la del POS), `/{id}/info`, `/info/barcode/{barcode}`, `/{id}/financial`, `/financial/search/{name}`, `/{id}/sale`, `/{id}/purchase` |
| Margen | chip calculado en cliente (`ProductsTable.tsx:187-199`); card "Mejor Margen" (`ProductDetailsModal.tsx:483-498`); **Rentabilidad** (`/profitability/*` sin guard) | `best_margin_unit/percent` (mismo DTO financiero); `/margin-alert(s)`, `/margin-report` |
| Proveedor + costo juntos | — (sin uso en UI) | `/products/{id}/supplier-comparison`, `/suppliers/{id}/cost-analysis` (pricing, gate `products:read`) |

No sensibles (lo que el catálogo SÍ muestra): nombre, imagen, categoría, marca, SKU/código de barras, variantes (nombre/SKU/precio/stock), precio de venta con IVA, stock, unidad base, origen, estado. El costo **no** vive en `products.products` (está en `products.unit_costs`); las variantes no llevan costo.

### Permisos (migraciones)

- `products:read`=106 / `products:write`=107. Con solo read: **VNDR01, CAJA01, SUPL01 (legacy), CLNT01**. Con ambos: F2VLso (además bypass), BUYR01, DEPO01, ENCR01.
- Último ID de permiso usado: **188**; `PLAN_PEDIDOS_MOSTRADOR` reserva 189-190 (`counterorders:read/write`).
- Precedente de filtrado por permiso en backend: solo `internal/sale` (`PermissionChecker` inyectado + `enforcePriceModificationPolicy`, `sale_service.go:87-121`, wiring `wiring/sale.go:52-69`). En catalog/pricing **no existe nada** — handlers solo resuelven sucursal.
- **Cache con trampa**: `ProductService` cachea por (name, branch) vía `CachePort.CachedLookup` (`internal/catalog/service.go:246-258,379`) **sin dimensión de rol** → cualquier strip por permiso debe aplicarse **post-cache** (a nivel handler/DTO). Si se filtrara antes o se cachease el resultado filtrado, hay contaminación entre roles.
- **ETag con trampa**: el subrouter `/products` usa `ETagMiddleware` (`internal/catalog/http/routes.go:38`) — mismo URL con bodies distintos según permiso rompe el ETag; la dimensión de permiso debe entrar al cálculo (o excluir las rutas sanitizadas).

### Frontend (activos)

- Feature products migrada a TSX (`src/features/products/`), contenedor `src/pages/Products.tsx`. Reutilizable: `ProductsFilters`, `ProductsPagination`, `useProductsLogic` (list+search vía `productService`).
- Buscadores existentes no pintan costo (`ProductSearchPanel` del POS muestra nombre/SKU/precio/stock; dropdown de presupuestos solo nombre+precio) — pero consumen endpoints que sí lo devuelven.
- No existe ninguna vista de catálogo read-only ni componentes de tarjeta (`viewMode` es solo `paginated|search`).
- Gap extra hallado: botón "Nuevo Producto" de `ProductsHeader.tsx:20-23` sin gate de permiso.

## 3. Decisiones de diseño

### 3.1 Permiso nuevo `products:cost` (visibilidad de costo/margen/proveedor)

- Un solo permiso binario que habilita ver datos de costo, margen y comparativas de proveedores. Se toma el **siguiente ID libre** al crear la migración: **189** si `PLAN_PEDIDOS_MOSTRADOR` aún no migró, **191** si ya (189-190 reservados) — coordinar entre ambos planes.
- **Grants por nombre**: F2VLso (por consistencia; ya bypasea), ENCR01, DEPO01, BUYR01 (compras negocian costos). **Sin**: VNDR01, CAJA01 (ver 5.2), SUPL01/CLNT01 (legacy).
- Es un permiso de **datos**, no de path: las rutas de producto siguen gateadas por `products:read/write`; `products:cost` decide el *contenido* de la respuesta.

### 3.2 Strip server-side post-cache (la garantía real)

- Puerto `PermissionChecker` (patrón `internal/sale/port.go:150` + wiring) inyectado en los handlers de `internal/catalog/http.go`. Helper `sanitize`: sin `products:cost` → omitir en las respuestas `unit_costs_summary`, `financial_health`, `best_margin_unit/percent` (DTO financiero y `ProductOperationInfoResponse` de `/{id}/sale`+`/{id}/purchase`). Siempre **después** de `CachedLookup`, jamás cachear el resultado sanitizado.
- **Rutas de pricing movidas a `products:cost`**: `/products/{id}/costs*` (history/last/last-record/summary), `/margin-alert`, `/margin-report`, `/supplier-comparison`, `/cost-trends`, `/weighted-average`, `/cost-methods`, `/margin-alerts`, `/suppliers/{id}/cost-analysis` — vía `RequireModulePermissionOnPath` (read=write=`products:cost`). Auditar en implementación `/{id}/pricing-info` y `/{id}/units` (dejar fuera si no exponen costo).
- ETag: incorporar la dimensión de permiso relevante al hash del ETag en las rutas sanitizadas (o excluir esas rutas del ETagMiddleware) — decisión de implementación con test explícito.
- `target_margin_percent` (estrategia de precio, no costo real): se deja fuera del alcance v1 (ver 5.3).

### 3.3 Nueva UI: `/catalogo` (Catálogo comercial)

Feature-sliced `src/features/catalog/` (patrón `transfers/`), route `/catalogo` con `PermissionGuard permission="products:read"` + nav **"Catálogo"** en "Gestión Comercial" gate `products:read` (lo ven vendedor y cajero; es su fuente de consulta de precios).

- **Vista**: grilla de tarjetas (mobile-first) + búsqueda con debounce y soporte de código de barras (reusar patrón de `useBarcodeScanner`); filtros por categoría/marca (endpoints existentes); paginación.
- **Tarjeta de producto**: imagen, nombre, marca/categoría, SKU/código de barras, **precio de venta con IVA**, indicador de stock, variantes con su precio/stock (expandible). **Nada** de costo/margen/proveedor.
- **Datos**: los mismos endpoints de siempre (`/products/search/{name}`, `/products/list`, `/info/search`) — seguros una vez aplicado 3.2; el FE no necesita un modo especial.
- **Acciones**: ninguna de gestión (sin editar/crear); opcional CTA "Agregar al pedido" cuando exista `/pedidos` (FASE 3).
- i18n `locales/es/catalog.js`; componentes según `DESIGN.md` (Fluent 2, `glass-mica` en modales, elevaciones `shadow-fluent-*`).

### 3.4 Re-gating de las páginas de gestión (el vendedor no las ve)

| Ruta/elemento | Hoy | Destino |
|:--|:--|:--|
| `/productos` (route + nav) | `products:read` | **`products:write`** — gestión completa para quien administra catálogo |
| Columna costo + chip margen (`ProductsTable`) | siempre visible | render solo con `products:cost` (defensa en profundidad para roles write-sin-cost) |
| "Resumen de Costos"/historial/"Mejor Margen" (`ProductDetailsModal`) | siempre visible | solo con `products:cost`; "Ajustar Costo" ya gateado `products:write` |
| "Nuevo Producto" (`ProductsHeader`) | sin gate | `products:write` (fix de gap) |
| `/profitability/*` | **sin PermissionGuard** | guard `analytics:read` (fix de fuga de márgenes por URL) |
| `/configuracion/{categorias,marcas,atributos,balanzas,conversiones}` | `products:read` | ride-along opcional → `products:write` (CRUD admin; hoy ocultas de nav para VNDR01 pero accesibles por URL; sin datos sensibles) |

`/ajustes-precios` queda como está (no muestra costo; la escritura ya exige `products:write` server-side).

## 4. Mejoras propuestas

1. **La fuga se cierra para todos los roles**, no solo VNDR01: CLNT01 (rol de cliente) y CAJA01 dejan de recibir costos por API aunque nunca hayan visto la página.
2. **Rentabilidad blindada por fin** (guard que le faltaba).
3. El catálogo es también la página de consulta de precios del **cajero** (hoy su única opción era la tabla de gestión).
4. Base de componentes reutilizable para el buscador del armador de pedidos de `PLAN_PEDIDOS_MOSTRADOR` (FASE 2): tarjetas y picker comparten `src/features/catalog/`.
5. Coherencia con el patrón ya establecido: la UI oculta, el backend garantiza (idéntico a `sales:apply_discount` / detalle de venta acotado v2).

## 5. Decisiones abiertas

| # | Pregunta | Recomación | Bloquea |
|:-:|:---------|:-----------|:--------|
| 5.1 | ¿ID del permiso? | **RESUELTO: products:cost = 189** (se implementó primero; `PLAN_PEDIDOS_MOSTRADOR` pasa a 190/191 para counterorders:*) | Migración |
| 5.2 | ¿CAJA01 ve costos? | No (no los usa en ningún flujo; hoy tampoco los ve en UI). Reversible por datos si la operación lo pide | Grants |
| 5.3 | ¿Se strippea `target_margin_percent` (objetivo de margen configurado)? | No en v1 (es parámetro de estrategia, no costo real; lo consulta la UI de pricing). Revisar si se considera comercialmente sensible | Nada |
| 5.4 | ¿Las páginas de metadatos (categorías/marcas/atributos/balanzas/conversiones) pasan a `products:write`? | Sí como ride-along (§3.4); sin urgencia | Nada |

## 6. Fases

> Reglas AGENTS.md: migraciones SOLO via `make migrate-create`; handlers sin lógica de negocio; tests obligatorios; i18n siempre; pnpm exclusivo; skill `vercel-react-best-practices` con reporte en `conductor/` del frontend.

### FASE 1 — Backend: permiso + rutas + strip (sola)

1. **Migración**: alta de `products:cost` (ID según 5.1, documentado) + grants por nombre (F2VLso, ENCR01, DEPO01, BUYR01) + `.down.sql`. Gates incluyen `make migrate-up` + `make migrate-verify-gaps`.
2. **Puerto + wiring**: `PermissionChecker` para catalog (patrón sale, `wiring/sale.go:52-69`).
3. **Sanitización post-cache** en `internal/catalog/http.go`: DTOs financiero y `ProductOperationInfoResponse` omiten `unit_costs_summary`/`financial_health`/`best_margin_*` sin el permiso. Tests table-driven: con/sin permiso, admin bypass, **cache compartida entre roles (misma clave, dos respuestas)**, ETag por permiso.
4. **Pricing**: re-gate de las rutas de costo a `products:cost` (`RequireModulePermissionOnPath`) + test de middleware.

**Gates:** `make build` + `go test ./...` + `golangci-lint run` + migrate-up/verify en dev.

### FASE 2 — Frontend: catálogo + re-gating

1. **Feature** `src/features/catalog/` (components CatalogBoard/CatalogCard/VariantList + hooks react-query sobre `productService` + types) + página `/catalogo` (guard `products:read`) + nav "Catálogo" en "Gestión Comercial".
2. **Re-gating** §3.4: `/productos` → `products:write`; costo/margen de `ProductsTable`/`ProductDetailsModal` → `products:cost` (`hasPermission`); "Nuevo Producto" → `products:write`; guards `analytics:read` en `/profitability/*`; (ride-along 5.4 si se acepta).
3. **i18n** `locales/es/catalog.js` + keys `catalog.*`; ajustar keys de products si hace falta.
4. **Tests** según `PLAN_TEST_DESIGN_FRONTEND.md` (catálogo renderiza precio de venta y NO costo; gates por permiso en products) + **auditoría** `vercel-react-best-practices` → reporte en `erp-webapp/conductor/`.

**Gates:** `npx vitest --run` 0 fallos + `npx tsc --noEmit` + `pnpm build` + `pnpm lint:design`.

### FASE 3 — Integración con pedidos (cuando `/pedidos` exista)

El buscador del armador de pedidos (`PLAN_PEDIDOS_MOSTRADOR` FASE 2) reusa tarjetas/picker de `src/features/catalog/`; CTA "Agregar al pedido" desde el catálogo (abre `/pedidos` con el ítem precargado). Opcional según prioridad.

## 7. Riesgos

| Riesgo | Mitigación |
|:-------|:-----------|
| Contaminación de cache entre roles (clave sin permiso) | Strip **post-cache** obligatorio; test explícito de misma clave con dos roles |
| ETag colisiona cuando el body varía por permiso | Dimensión de permiso en el hash o exclusión de rutas sanitizadas; test |
| Alguna UI consume hoy `unit_costs_summary` del endpoint del POS y se rompe | Consumidores verificados: POS/presupuestos no lo pintan; `ProductDetailsModal` queda gateado `products:cost` (ENCR01/DEPO01/BUYR01 lo tienen) |
| Re-gate `/productos` a `products:write` deja fuera a un rol legítimo de solo lectura | Solo quedaban VNDR01/CAJA01/SUPL01/CLNT01 (todos con destino al catálogo o legacy); auditar usuarios reales antes del deploy |
| Conflicto de IDs de permiso con `PLAN_PEDIDOS_MOSTRADOR` | §3.1/5.1: siguiente libre + documento cruzado en ambos planes |
| JWTs sin el permiso nuevo | Sin problema: el default es *sin* costo (fail-closed); roles con grant necesitan re-login para ver costos — comunicar |
| Rollback | `.down.sql` (permiso) + re-gates revertibles en código |

## 8. Orden y dependencias

```
FASE 1 (backend, sola)  ──>  FASE 2 (frontend catálogo + re-gating)  ──>  FASE 3 (integración /pedidos)
```

Independiente del plan de pedidos (puede implementarse antes o después); solo comparten la numeración de permisos (§3.1) y la FASE 3. Deploy de FASE 1: fail-closed — tras aplicar, roles sin `products:cost` dejan de recibir costo por API de inmediato (efecto deseado; ninguna UI de esos roles lo pinta).

---

## 9. Registro de implementación (2026-09-07)

### FASE 1 — Backend (commit `676509d` en `business_management`, branch `dev`)

- **Migración `20260907202357_add_products_cost_permission`**: `products:cost` = **189**; grants por nombre a F2VLso, ENCR01, DEPO01, BUYR01. `.down.sql` revoca+elimina (nota: el strip vive en código, el rollback del permiso solo no restaura visibilidad). Verificado en dev: 4 grants, 0 para VNDR01/CAJA01/SUPL01/CLNT01.
- **Puerto + gate**: `catalog.PermissionChecker` (port.go) + `catalog.CostGate` (costgate.go, patrón sale B.3: nil=legacy, admin bypass, error→500). Bridge en `wiring/catalog.go` (`CatalogCostGate()` sobre el permission cache compartido).
- **Strip post-cache**: helpers `StripCostData()` sobre los 3 DTOs de salida — `ProductFinancialEnrichedResponse` (unit_costs_summary→null, financial_health→cero, has_valid_costs→false, best_margin_*→omitidos), `ProductOperationInfoResponse` (ídem) y `ProductEnrichedResponse` (**purchase_price→omitido**: la columna "Costo de Compra" del FE cae a este campo como fallback — hallazgo del §2 confirmado en implementación). Se aplica en 15 handlers de catalog DESPUÉS del servicio; la cache (sin dimensión de rol) nunca ve payloads sanitizados. `unit_costs_summary` serializa `null` (key estable; no distingue "sin permiso" de "producto sin costos" — intencional).
- **Pricing re-gate** (`internal/pricing/http/routes.go`): wrapper `costGate` (`RequireModulePermissionOnPath`, read=write=`products:cost`) sobre `/products/{id}/costs*`, `/margin-alert`, `/margin-report`, `/supplier-comparison`, `/cost-trends`, `/weighted-average`, `/cost-methods`, `/margin-alerts`, `/suppliers/{id}/cost-analysis`, más **`/cost-transactions`** (subrouter completo — estaba en products:read: fuga enumerada como riesgo en §7) y dos hallazgos de la auditoría §3.2: **`/products/by-category`** (devuelve cost+margin de ProductWithPricing) y **`/{id}/pricing-info`** (devuelve current_cost/weighted_avg_cost/margin_*). **`/{id}/units` queda fuera**: UnitPrice es solo precio de venta (auditado). Price-transactions y ajustes de precio quedan como estaban.
- **ETag**: resuelto *por construcción*: el hash es del body, que ya varía por permiso → la dimensión entra al hash sola. Test explícito: ETags distintos por rol, replay cruzado sin 304, replay propio con 304.
- **Tests**: `internal/catalog/http_costgate_test.go` (por familia de handler con/sin permiso/admin bypass, strip-no-contamina-cache con puntero compartido, resolver-error→500, ETag por permiso) y `internal/pricing/http/routes_test.go` (403 sin products:cost / 200 con él / price-routes siguen con products:read).
- **Gates**: `make build` ✅ · `go test ./...` 0 FAIL ✅ · `golangci-lint run` ✅ · `make migrate-up` + `migrate-backfill` + `migrate-verify-gaps` ✅ (268 aplicadas, 0 gaps — el backfill registró 6 migraciones del 2026-09-05 ya aplicadas pero sin fila en `migration_history`, condición preexistente del dev DB).

### FASE 2 — Frontend (branch `dev` de `erp-webapp`)

- **Feature `src/features/catalog/`** (feature-sliced, patrón transfers): `types.ts` (proyección `CatalogProduct` del DTO de búsqueda avanzada — no tipa ningún campo de costo), `hooks/useCatalogProducts.ts` (react-query sobre `productService.searchAdvanced` = POST `/products/search/advanced`: filtros server-side de categoría/marca, stock, sort, paginación; el backend matchea `name ILIKE OR barcode =` así que el buscador soporta código de barras sin endpoint extra), `hooks/useDebouncedValue.ts`, `components/CatalogCard.tsx` (imagen/ícono, nombre, marca·categoría, barcode/unidad, P.V.P. con IVA, badge de stock, variantes expandibles on-demand), `components/CatalogBoard.tsx` (PageHeader + toolbar + grilla mobile-first + estados §6.7 + paginación), barrel `index.ts`.
- **Ruta/nav**: `/catalogo` con `PermissionGuard products:read` (App.tsx) + "Catálogo" en "Gestión Comercial" (navigation.ts, ícono Store, products:read — lo ven vendedor y cajero).
- **Re-gating §3.4**: `/productos` → `products:write` (ruta+nav); ProductsTable columna costo + chip margen → `products:cost` (useAuth); ProductDetailsModal (tabla de costos, card Mejor Margen, ítem has_costs del checklist, banner Financial Health) → `products:cost`; "Nuevo Producto" (ProductsHeader) → `products:write` (fix del gap); `/profitability/*` (6 rutas) → `analytics:read`; ride-along §5.4: `/configuracion/{categorias,marcas,atributos,balanzas,conversiones}` → `products:write` (ruta+nav).
- **i18n**: `locales/es/catalog.js` (clave `catalog.*` + `nav.catalog`); `en` cae a fallback español (patrón transfers).
- **Tests**: `CatalogBoard.test.tsx` (7: precio sin costo, params de query, debounce, empty, error+retry, variantes on-demand, paginación) + ProductsTable.test (2 nuevos: columna costo oculta sin products:cost, chip de margen solo con él) + mock de AuthContext agregado a 5 tests legacy que renderizan los componentes ahora gateados + navigation.test actualizado (VNDR01 ahora ve Catálogo en Gestión Comercial — cambio de spec del plan).
- **Auditoría performance**: `erp-webapp/conductor/AUDIT_REACT_PERFORMANCE_CATALOGO_FRONTEND.md` — sin hallazgos bloqueantes (2 notas P3).
- **Gates**: `vitest --run` 578/578 ✅ · `tsc --noEmit` 0 ✅ · `pnpm build` ✅ · `lint:design` ✅.

### Hotfix post-UV (2026-09-08, backend `75b6518`)

- **`POST /products/search/advanced` daba 403 al vendedor**: el endpoint es una búsqueda (lectura) que viaja como POST, y en el subrouter `/products` el middleware de módulo mapea POST→`products:write`. Era una trampa legacy (los únicos consumidores previos eran roles write); el catálogo es el primer consumidor read-only. Fix: registrado a nivel raíz con `RequireModulePermissionOnPath(products:read)` + ETag (precedencia intacta; tests de ruta: read-only 200, admin 200, sin products:read 403). `/search/facets` queda en el subrouter (GET→read).

### Notas de deploy

- Fail-closed: al deployar FASE 1, roles sin `products:cost` dejan de recibir costos por API de inmediato (ninguna UI de esos roles los pinta). **F2VLso/ENCR01/DEPO01/BUYR01 necesitan re-login** para que el JWT refresque permisos y vean costo en la UI de gestión.
- FASE 3 (CTA "Agregar al pedido" + reuso en el armador de `/pedidos`) queda para `PLAN_PEDIDOS_MOSTRADOR`.
