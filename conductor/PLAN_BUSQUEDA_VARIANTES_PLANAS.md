# PLAN — Búsqueda de productos con variantes planas (variantes como productos independientes)

**Fecha**: 2026-09-11
**Estado**: ✅ Implementado (F1–F6 completas, 2026-09-14)
**Alcance**: Cross-repo — backend `business_management/` + frontend `erp-webapp/`
**Superficies**: compras (`/compras`), ventas (`/ventas/nueva`), pedidos (`/pedidos`), catálogo (`/catalogo`)
**Origen**: Pedido del owner: cuando se busquen productos, mostrar las variantes como productos
independientes en compras, ventas, pedidos y catálogos.

---

## 1. Veredicto de factibilidad

**SÍ es posible, y la deuda es acotada: el modelo transaccional ya es "variant-first"; solo falta
la capa de búsqueda.** Verificado contra el código el 2026-09-11:

| Capa | Estado hoy | Conclusión |
|:-----|:-----------|:-----------|
| Líneas de venta/compra/pedido | Ya aceptan `variant_id` por línea (`models/sale.go:246-259`, `models/purchase.go:361-371`, `internal/sale/counter_order.go:59-65`) | ✅ Nada que migrar |
| Stock por variante | `products.stock.variant_id` (`20260613000002`) | ✅ Ya existe |
| Precios por variante | `products.unit_prices.variant_id`, resolución variante-primero con fallback al padre (`20260613000003`) | ✅ Ya existe |
| Costos por variante | `products.unit_costs.variant_id` (`20260707185244`) | ✅ Ya existe |
| Búsqueda de productos | Todos los endpoints devuelven **una fila por producto padre** con stock agregado; ningún search matchea SKU de variante | ❌ **Gap principal** |
| Pickers FE | Selección de variante es un **segundo paso** (modal/chips/expand) tras elegir el producto base | ❌ **Gap UX** |

El único trabajo real es: (1) un modo de búsqueda "plana" en el backend que emita una fila por
unidad vendible (variante o producto), y (2) que los 4 pickers consuman ese modo y agreguen
directamente la línea `product_id + variant_id`.

---

## 2. Estado actual (evidencia verificada)

### 2.1 Backend — búsqueda

Todos los endpoints viven en `internal/catalog/` (FASE 5 completa hasta 5.7; no queda handler
legacy de productos):

| Endpoint | Uso | Devuelve |
|:---------|:----|:---------|
| `POST /products/search/advanced` (`http.go:1545`, repo `product_search_repo.go` sobre `mv_product_catalog`) | Pedidos + Catálogo (`useCatalogProducts`) | 1 fila por producto; `has_variant`/`variant_count`; `stock_quantity` agregado (base + variantes activas) |
| `GET /products/search/{name}` (`http.go:381`, `product_repo.go:520-523`) | Compras + Ventas | `ProductEnriched`; matchea SOLO `p.name ILIKE` |
| `GET /products/{id}/sale` · `/{id}/purchase` (`http.go:795-849`) | Post-selección en ventas/compras | Producto + variantes **anidadas** (sin stock por variante) |
| `POST /sales/scan` (`internal/measure/service.go:343-355, 462-476`) | Escáner de código en ventas/pedidos | **Único precedente variante-plana**: barcode de variante → `{product_id, variant_id}` + precio variante-primero |
| `GET /products/{id}/variants`, `/api/v1/variants/{id}/stock`, `/{id}/stock-summary` | Enriquecimiento FE | Variantes del producto, nunca cross-product |

- Término de búsqueda: `mpc.name ILIKE` OR barcode padre exacto (`product_search_repo.go:219-225`).
  **Ningún path de búsqueda toca `product_variants.sku`/`barcode`/`variant_name`.**
- `GetBySKU`/`GetByBarcode` existen en `variant_repo.go:235-283` como lookups exactos sin ruta HTTP.
- `product_variants`: `sku UNIQUE`, `barcode` (unique parcial), `variant_attributes JSONB`, índices
  GIN + parent + sku (`20260613000001`).

### 2.2 Frontend — los 4 pickers

| Pantalla | Componente | Fuente de búsqueda | Selección de variante hoy |
|:---------|:-----------|:--------------------|:--------------------------|
| Compras | `PurchaseProductModal.tsx` + `usePurchasesLogic.ts:476-505` (debounce 500ms) | `GET /products/search/{term}` | 2º paso: chips de atributos in-modal (423-558) o botón "producto principal" |
| Ventas | `ProductSearchPanel.tsx` + `SalesNew.tsx:508-534` (debounce 300ms) | `GET /products/search/{term}` | 2º paso: `VariantSelectorModal` (abre automáticamente si `has_variants`) |
| Pedidos | `OrderBuilder.tsx` + `ProductPickCard` (debounce 350ms) | `POST /products/search/advanced` (vía `useCatalogProducts`) | Semi-plano ya: fila "Producto base" + toggle "Elegir variante (n)" con [+] por variante (fix `280b4ad`) |
| Catálogo | `CatalogBoard.tsx` + `CatalogCard` (debounce 350ms) | `POST /products/search/advanced` (mismo hook) | Expandible "Variantes (n)" por tarjeta, solo lectura |

- Pedidos y catálogo **comparten** la capa de datos (`features/catalog/hooks/useCatalogProducts.ts`);
  compras y ventas tienen inputs propios.
- El stock/precio por variante se arma hoy con **N+1 llamadas** en `variantService.getEnrichedVariants`
  (variantes + un GET de stock por variante + `/units` para precios) — consumido por los 4 modales.
- Los carritos ya deduplican por `productId|variantId` (`domain/counterorders/cart.ts:27-39`,
  `SalesNew.tsx:700`, `usePurchasesLogic.ts:732`) y componen el nombre como `"Producto · Variante"`.

---

## 3. Diseño propuesto

### 3.1 Decisión central: extender `POST /products/search/advanced` con `granularity`

```
AdvancedProductFilter += granularity: "product" (default) | "variant"
```

- **Default `"product"` = cero regresión**: consumidores actuales (catálogo, pedidos, facets)
  no cambian en absoluto.
- `"variant"` devuelve **una fila por unidad vendible**.
- Compras y ventas **migran** de `GET /products/search/{term}` a este endpoint: una sola
  superficie de búsqueda para las 4 pantallas (con filtros, paginación y `branch_id` que el
  endpoint por nombre no tiene).
- El endpoint por nombre queda intocado para sus otros consumidores (autocompleters legacy,
  `ProductSearchInput`, admin productos).

### 3.2 Fila "unidad vendible" (`granularity: "variant"`)

```jsonc
{
  "id": "prod_x",              // producto padre
  "variant_id": "var_y",       // null en fila base / producto sin variantes
  "is_base_row": false,        // true solo en la fila "producto base" de un producto CON variantes
  "name": "Camisa Oxford",
  "variant_name": "Rojo / M",
  "sku": "CAM-ROJ-M",          // SKU de la variante (o del producto si no tiene)
  "barcode": "123456789",
  "variant_attributes": { "color": "rojo", "talla": "M" },
  "has_variant": true,
  "brand": "...", "category": "...", "product_type": "PHYSICAL",
  "current_price": 150000,     // precio EFECTIVO: variante-primero, fallback padre
  "stock_quantity": 7,         // stock de ESA unidad (branch-aware si viene branch_id)
  "stock_status": "in_stock"
}
```

**Regla de filas**: producto sin variantes → 1 fila (igual que hoy). Producto con variantes
activas → 1 fila por variante activa **+ 1 fila base** (`is_base_row: true`, stock base-only),
porque el producto base es agregable en los 4 flujos hoy (fix `280b4ad`, botón "producto
principal" en compras/ventas). Orden estable: base primero, luego variantes por `display_order`.

**Costo**: `current_cost` se incluye SOLO si el caller tiene `products:cost` (permiso 189),
con strip post-cache idéntico al de los 15 handlers ya cubiertos por
`PLAN_CATALOGO_VENDEDOR` — un vendedor jamás debe ver costo por esta vía nueva.

### 3.3 Matching del término (la mejora de UX central)

En modo `variant`, el término matchea:

- `name` del producto (como hoy) y barcode padre exacto;
- `pv.sku ILIKE '%term%'` — **buscar por SKU de variante pasa a funcionar**;
- `pv.barcode = term` (exacto);
- `pv.variant_name ILIKE '%term%'`;
- `pv.variant_attributes::text ILIKE '%term%'` (v1 simple; los atributos estructurados se
  filtran mejor con el filtro `attributes`).

### 3.4 Filtros, orden y facetas

| Filtro | Comportamiento en modo variant |
|:-------|:-------------------------------|
| `category_id(s)`, `brand_ids`, `tag_slugs`, `state`, `product_type` | Nivel producto padre (sin cambio) |
| `attributes` (`is_variant_attribute`) | Contra el `variant_attributes` jsonb **de la fila** (más preciso que el EXISTS actual, que matchea si CUALQUIER variante cumple) |
| `in_stock_only` | Stock de la unidad ≥ 1 |
| `price_min/max` | Contra el precio efectivo de la unidad |
| `sort_by` name/price | Name compone `nombre · variante` |
| `branch_id` | Stock de la unidad en esa sucursal; total si no viene |
| Facetas | Sin cambio en v1 (siguen a nivel producto) |

### 3.5 Estrategia SQL

- `mv_product_catalog` queda como está (nivel producto, modo `product` intocado).
- Modo `variant` = query-time: `UNION ALL` de (a) join `mv_product_catalog × product_variants`
  con agregación de `products.stock` por (producto, variante) vía LATERAL/CTE, (b) filas base de
  productos con variantes, (c) productos sin variantes. Los índices existentes (parent, sku,
  barcode parcial, GIN attributes) cubren el acceso.
- `page_size` ≤ 100 con paginación OFFSET sobre el UNION.
- **Plan B diferido**: si el EXPLAIN sobre la DB dev con dataset real muestra degradación, MV
  dedicada `mv_sellable_catalog` refrescada junto a la actual. No bloquea la v1.

### 3.6 Frontend

- **F2 (compartido)**: `CatalogSellableUnit` (types) + `useCatalogProducts`/`productService.searchAdvanced`
  ganan `granularity`; compras y ventas pasan a consumirlo. `getEnrichedVariants` (N+1) sigue
  existiendo para los modales admin de variantes — fuera de alcance, candidato a reuso de F1 después.
- **Selección**: la fila plana **resuelve la variante ya elegida**. En pedidos agrega directo
  (`product_id + variant_id + qty`); en catálogo es solo lectura; en ventas/compras el flujo
  mantiene su fetch composite (`getForSale`/`getForPurchase`) para impuestos/unidades/precios
  completos, pero **saltándose el paso de elegir variante** — la variante viene de la fila.
  `VariantSelectorModal` y los chips de compras quedan como fallback (botón "ver todas las
  variantes del producto"), no como camino principal.
- **Display**: nombre `"Producto · Variante"` (patrón ya usado por los carritos), chip de SKU,
  chip de atributos, stock propio de la unidad.
- **Densidad**: producto con muchas variantes puede inundar el grid → en pickers limitar filas
  por producto (top N por `display_order` + CTA "ver todas las variantes" que abre el modal
  existente); en catálogo, agrupación visual opcional por producto.
- i18n: claves nuevas en `locales/es` + `locales/en` en el mismo diff (regla de tests §3).
- UI según `DESIGN.md` (Fluent 2, chips, `glass-mica` en modales tocados).

---

## 4. Fases y gates

| Fase | Repo | Contenido | Gate de salida |
|:-----|:-----|:----------|:---------------|
| F1 | BE | Puerto `SearchReader` + repo SQL modo variant + param en handler + DTO | `go test ./...` (table-driven con mock de puerto) + **≥1 test que ejercite el SQL real** (lección pedidos: repos mockeados no prueban SQL) + `golangci-lint` + `make build` |
| F2 | FE | Types + `searchAdvanced(granularity)` + `useCatalogProducts` | `tsc --noEmit` 0 + vitest del hook |
| F3 | FE | Pedidos: `OrderBuilder`/`ProductPickCard` plano, [+] directo por variante | vitest (`CounterOrdersPage.test`, `CatalogBoard.test` ya existen) + `lint:design` |
| F4 | FE | Catálogo: `CatalogBoard`/`CatalogCard` por unidad vendible | vitest + revisión visual |
| F5 | FE | Ventas: `ProductSearchPanel` plano, add directo sin modal obligatorio | vitest + **auditoría `vercel-react-best-practices`** con reporte en `erp-webapp/conductor/` |
| F6 | FE | Compras: `PurchaseProductModal` plano (costo gated) | Gates finales: `npx vitest --run` 0 fallos, `tsc` 0, `pnpm build`, `pnpm lint:design`; BE: `go test ./...` + lint |

**Sin migraciones de DB**: todo es SQL de consulta + código. `make migrate-create` NO aplica.

---

## 5. Riesgos y mitigaciones

| Riesgo | Mitigación |
|:-------|:-----------|
| Perf del UNION query-time | Medir con EXPLAIN en dev antes de mergear F1; MV dedicada como plan B (§3.5) |
| Densidad UX (N variantes por producto) | Top-N por producto + CTA al modal existente (§3.6) |
| Regresión en consumidores actuales | `granularity` default `"product"`; los consumers no cambian su payload |
| Costo filtrado por permiso | Strip post-cache `products:cost` en el handler nuevo (§3.2) |
| Contrato BE→FE desincronizado | Regla AGENTS.md #2: cada fase BE+FE en el mismo flujo de trabajo |
| Búsqueda por nombre de variante igual a otro producto | Aceptable: el resultado muestra producto · variante, no es ambiguo para el usuario |

## 6. Fuera de alcance

- `/productos` (admin, `ProductsTable` 1 fila por producto): mismo patrón aplicable después.
- Optimizar el N+1 de `getEnrichedVariants` en modales admin (candidato post-F1).
- Edición/CRUD de variantes, printing, reportes.

## 7. Relación con otros planes vivos

- `PLAN_PEDIDOS_MOSTRADOR_VENDEDOR_CAJA.md` — FASE 5 pendiente; este plan no la bloquea
  (toca el mismo picker en F3; coordinar orden si se ejecutan juntos).
- `PLAN_CATALOGO_VENDEDOR.md` — FASE 3 opcional (CTA "Agregar al pedido" desde `/catalogo`);
  su garantía de ocultamiento de costo es requisito aquí (§3.2).
- `PLAN_MONOROL_DESCUENTOS_CREDITO_CLIENTE.md` — independiente (A0 prod audit pendiente).

## 8. Registro

- 2026-09-11 — Plan creado tras exploración verificada de ambos repos (agentes Explore BE/FE,
  evidencia file:line en §2). Limpieza de `conductor/` realizada el mismo día: los planes
  completados viven en el historial git de cada repo.
- 2026-09-14 — **Implementación completa F1–F6.** BE (branch `dev`): `SearchReader.
  AdvancedSearchUnits` + repo SQL `product_search_units_repo.go` (UNION ALL de 3 legs sobre
  `mv_product_catalog`, precio/costo variante-primero inline vía joins únicos
  `uk_unit_prices_product_unit_variant`/`uk_unit_costs_product_unit_variant`), filtro
  `granularity` validado en handler (400 si inválida), `SellableUnit`/`SellableUnitResponse`
  + strip `current_cost` post-cache por `products:cost`. Gates: `go test ./...` verde
  (unit + HTTP + integración contra BD real que ejercita el SQL: SKU/barcode match, fallback
  de precio, stock por unidad, branch, in_stock_only, rango de precio, variantes inactivas),
  `golangci-lint`, `make build`, **EXPLAIN ANALYZE peor caso: 1.4 ms** → plan B (MV dedicada)
  descartado. FE (branch `dev`): `CatalogSellableUnit` + `useCatalogSellableUnits`;
  pedidos picker plano con cap de densidad (top-3 variantes + "+n variantes más"); catálogo
  por unidad vendible; ventas dropdown plano (modal de variantes queda como fallback, stock
  virtual por unidad); compras con variante preseleccionada desde la fila y costo prefill
  de la unidad. Gates FE: vitest 626/626 (91 archivos), tsc 0, `pnpm build`, `lint:design`;
  auditoría react en `erp-webapp/conductor/AUDIT_REACT_PERFORMANCE_BUSQUEDA_VARIANTES_PLANAS_FRONTEND.md`.
  Smoke HTTP end-to-end en dev: filas planas ✓, matcheo por SKU de variante ✓, admin ve
  `current_cost` / vendedor no ✓, modo default sin cambios (87 productos, sin
  `variant_id`) ✓, granularidad inválida 400 ✓.
  Desviaciones menores del diseño original: (1) el matcheo del término es por unidad (§3.3):
  buscar por SKU de variante devuelve SOLO la fila de esa variante, no la base — es la UX
  buscada; (2) `sort_by=name` ordena por nombre del padre + base-primero + display_order
  (agrupa variantes bajo su producto) en vez del string compuesto; (3) el N+1 de
  `getEnrichedVariants` queda en los modales admin de variantes (fuera de alcance, §6).
