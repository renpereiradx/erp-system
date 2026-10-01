# EXEC_UUID_SKU_ULID — Ejecución del combo C+D (ULID + SKU universal)

Fecha: 2026-10-01 · Ejecuta: `ANALYSIS_UUID_SKU_PRODUCTS_2026-09-30.md` (fases F1–F3 + F5)
Commits: BE `788b8cc` (business_management dev) · FE `3578e41` (erp-webapp dev)

---

## 1. Resumen

Ejecutado el combo recomendado por el análisis (**C + D**, descartadas A/B por ausencia de driver externo):

- **F1 — ULID**: los IDs de producto/variante (y los sufijos de SKU de variante) se generan ahora con
  ULID monotónico (`oklog/ulid/v2`) vía `internal/platform/idgen`. 26 chars → caben en `varchar(27)`
  existente: **cero DDL por IDs**, coexisten con shortids ya emitidos. `teris-io/shortid` sigue en
  go.mod SOLO para users/parties/setup (fuera de alcance, fase propia si algún día aplica).
- **F2 — SKU universal**: nueva columna `products.products.sku varchar(50)` + `uq_products_sku` +
  backfill determinista (`'P-' || md5(id)[0:12]`, sin colisiones). Todo producto nace con código de
  negocio; las variantes conservan su propio SKU.
- **F3 — FE display**: SKU-first con fallback a ID en ~12 sitios; formulario de producto permite
  fijar/corregir el SKU; i18n es+en; fixtures demo normalizados.

## 2. Decisión de modelo (la que el análisis dejaba al owner)

**Se eligió (a): columna `products.sku` universal** — no variante implícita (b).

Razones: (b) obligaría a mover stock de filas `variant NULL` a variantes implícitas, tocar los uniques
compuestos con `COALESCE(variant_id,'__NULL__')` y re-mapear datos existentes — costo alto y riesgo de
descuadre; (a) es aditiva, reversible (drop column) y resuelve el problema humano directamente.
Es aditiva y reversible: si el owner prefiere (b) algún día, descartar la columna no deja deudas.

Semántica: `products.sku` = código del producto (para productos simples ES el código vendible; para
productos variables actúa como código de familia). La UI muestra: fila base → `products.sku`; fila de
variante → `variant.sku`.

## 3. Qué cambió (BE)

| Pieza | Detalle |
|---|---|
| Generador | `internal/platform/idgen.NewULID()` — entropy monotónica por proceso con mutex; test propio (formato/unicidad/concurrencia) |
| Call sites | `catalog/service.go` (producto), `platform/postgres/variant_repo.go`, `database/postgres/variant.go` (variante); sufijos de SKU de variante también a ULID |
| Migración | `20261001131356_add_products_sku` (up: column + comment + backfill DO-block + unique; down: drop) |
| Write path | `CreateProductRequest.sku` / `UpdateProductRequest.sku` (`*string`; nil en update = sin cambio; blank rechazado; ≤50). Sin sku en create → auto `"{slug nombre 3 ASCII}-{cola ULID 8}"` (ej. `SMO-JBQXWVWJ`). Pre-check de unicidad en tx + mapeo `uq_products_sku` → 409 amigable |
| Read path | `ProductEnriched.sku` (select base + scanner + DTO `sku,omitempty`); búsqueda plana (`granularity=variant`): legs B/C proyectan `psku.sku` (join `products.products`) y el predicado de búsqueda matchea `psku.sku ILIKE` |
| Detalles | presupuestos (`product_sku`), requisiciones (`product_sku`), transferencias (`product_sku`+`variant_sku` en `branchTransferItemResponse`), catálogo balanza (`product_sku`) |
| NO tocado | funciones DB financieras (`get_product_enriched_by_*`), search avanzado por producto (`granularity=product`), MV `mv_product_catalog` (sin churn DDL), BI/read models, F4 (requisiciones varchar(50)) — ver §5 |

## 4. Qué cambió (FE)

- Tipos: `Product.sku?`, `product_sku?` en BudgetItem / PurchaseRequisitionDetail / ScaleCatalogItem /
  BranchTransferItem (+`variant_sku?`).
- Render SKU-first (fallback ID) en: carrito ventas (columna ID→SKU, `CartItem.sku` copiado en
  `addProductToCart`), VariantSelectorModal, MovementForm, ProductSearchInput (dropdown+seleccionado),
  PriceTransactions, PriceAdjustmentDetail, BudgetDetail (línea Cód: sku→barcode→id),
  PurchaseRequisitionDetail, TransferDetailModal (tooltip conserva IDs crudos), ScaleConfigPage,
  ProductDetailsModal (subtítulo). Los sitios ya sku-first (PurchaseProductSearchInput, BudgetCreate,
  ProductSearchModal, BudgetCreate dropdown) quedaron como estaban: su fallback a ID quedó muerto.
- Formulario de producto: campo SKU editable (Zod passthrough; vacío en create = autogenera, en
  update = sin cambio).
- i18n: `products.details.sku`, `products.modal.field.sku`, `products.modal.placeholder.sku` (es+en);
  `sales.cart.col.id` → 'SKU'.
- Fixtures demo: `demoData.js` (4 productos + 7 sale items con sku, ids numéricos → string),
  `demoPurchasePayments.js` (skus). Reservas demo sin tocar (dominio diferido).

## 5. Verificación (gates + smoke)

- **BE**: `go vet ./...` limpio · `golangci-lint run` limpio · tests unitarios ok (catalog, idgen,
  identity, sale, purchase, redis — `-count=1`). Fallo preexistente de `internal/platform/postgres`
  (necesita DB; falla igual en HEAD limpio) y gofmt de `ephemeral_test.go` preexistente.
- **Smoke vivo (:5050, JWT dev)**: migración aplicada (103/103 productos con sku);
  crear producto → id ULID 26 chars + sku `SMO-JBQXWVWJ`; sku explícito honrado; duplicado →
  `409 CONFLICT "Ya existe un producto con el SKU"`; búsqueda plana por `SMOKE-EXPL` matchea y trae
  `sku` en la fila; GET enriquecido trae `sku`; variante → id ULID + sku `01M3W5-L`. Datos de smoke
  purgados de la DB dev.
- **FE**: vitest 1357/1357 (3 tests nuevos/actualizados: columna SKU del carrito, BudgetDetail
  sku-first, PriceAdjustmentDetail sku) · `tsc --noEmit` 0 · `pnpm build` ✓ · `lint:design` ✓.
- **Baseline**: regenerado con `make db-baseline` (columna + constraint presentes para fresh-install).
  OJO: `baseline_schema.sql` es gitignoreado — viaja en el bundle al armar release.

## 6. Deuda y diferidos (explícitos)

1. **BI / read models**: TopProductsOverview y dashboards BI siguen mostrando `product_id` (los
   payloads de analítica no traen sku; tocaría los 41 read models — fuera de alcance).
2. **Reservas** (ReservationHistory/Dashboard): dominio reservas sin sku en payload.
3. **Búsqueda por producto** (`granularity=product`, /catalogo): no matchea ni expone `products.sku`
   aún (los pickers del flujo comercial sí, vía búsqueda plana). Si el catálogo necesita buscar por
   SKU, tocar el predicado del search repo (o añadir sku a la MV).
4. **F4** (normalizar `purchase_requisition_details.product_id` varchar(50)→27): omitido — ULID cabe
   en varchar(50); riesgo sin beneficio.
5. **Rutas financieras** `get_product_enriched_by_*` (DB functions) no exponen sku — solo afecta
   `/products/info/search` (fallback legacy de los pickers).
6. **shortid** sigue vivo para users/parties/setup/admin-init (13 call sites) — misma deuda de lib
   archivada, alcance distinto.

## 7. Gotchas para el futuro

- Al actualizar un cliente on-premise: correr `make migrate-up` (o el bundle), que hace ADD COLUMN +
  backfill + unique. Los SKUs viejos quedan como `P-XXXXXXXXXXXX` (md5) — el usuario puede
  corregirlos desde el formulario de producto.
- No renombrar `sku`/`product_sku` en los contratos: los caches Zustand y las cadenas de fallback FE
  dependen de esos nombres.
- El update de producto envía `sku: undefined` cuando el campo va vacío (no borra el SKU; el BE no
  soporta borrarlo).
