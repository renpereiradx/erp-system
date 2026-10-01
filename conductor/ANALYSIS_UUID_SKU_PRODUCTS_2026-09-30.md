# ANÁLISIS_UUID_SKU_PRODUCTS — Identificadores de producto: viabilidad de UUID + SKU

Fecha: 2026-09-30 · Tipo: análisis de viabilidad (sin código) · Repos: business_management (BE) + erp-webapp (FE)
Método: exploración verificada BE+FE (baseline_schema.sql, migraciones, internal/*, models/, services FE, stores FE).

---

## 1. Resumen ejecutivo

**La premisa "migrar de ID numérico a UUID" no aplica: los productos/variantes YA usan identificadores string
end-to-end** — `varchar(27)` en DB, `string` en Go (~170 archivos), `string` en TS (~162 archivos). El ID se
genera server-side con `teris-io/shortid` (7–14 caracteres). No hay ningún entero que convertir.

Los gaps reales que motivan la idea "UUID + SKU" son tres, y ninguno exige una migración masiva de datos:

1. **SKU no es universal**: existe SOLO en `product_variants.sku` (UNIQUE, auto-generado). Un producto sin
   variantes no tiene SKU visible → la UI muestra el ID crudo (`PurchaseProductSearchInput.tsx:146`
   `p.sku || ID: ${p.id}`, `BudgetCreate.tsx:324` `SKU: ... else ID: ...`).
2. **IDs crudos en pantalla** en ~8–15 componentes FE (carrito ventas, movimientos, detalle producto, escalas).
3. **El generador actual es deuda**: `teris-io/shortid` está archivada/congelada desde ~2017 y no es
   criptográficamente segura.

**Recomendación**: combo **ULID + SKU universal** (opciones C+D, §4) — ~90% del beneficio con <10% del costo,
**cero migración de datos**. La conversión a UUID (opciones A/B, §4) es viable pero solo se justifica con un
driver externo concreto (API pública a terceros, sincronización multi-master); hoy no existe ninguno.

---

## 2. Línea base: cómo está cableado hoy (verificado 2026-09-30)

### 2.1 Esquema DB

- `products.products.id` **varchar(27)** PK (baseline_schema.sql:15155). Sin columna `sku`; sí
  `barcode varchar(50)` (unique parcial) y `scale_code varchar(5)` (unique parcial). Sin soft-delete (`state boolean`).
- `products.product_variants` (migración 20260613000001): `id VARCHAR(27)` PK,
  `parent_product_id` FK **ON DELETE CASCADE**, **`sku VARCHAR(100) NOT NULL UNIQUE`** (auto-gen
  `{parent_barcode}-{attr1}-{attr2}`), `barcode` unique parcial.
- **30 columnas** `product_id`/`variant_id` en **27 tablas**, todas varchar(27) — EXCEPTO
  `transactions.purchase_requisition_details.product_id varchar(50)` (mismatch de ancho, FK igualmente válida).
  **37 FKs reales**; solo `audits.price_transactions_audit` y `audits.unit_costs_audit` quedan sueltas (sin FK).
- **41 índices secundarios** sobre esas columnas, incl. uniques compuestos con sentinel string:
  `uk_transfer_items_unique`, `uk_unit_prices_product_unit_variant`, `uq_stock_product_branch_variant` usan
  `COALESCE(variant_id, '__NULL__')` — **sensibles a conversión de tipo** (§7).
- SQL pesado: **~70 funciones + 5 procedures + 20+ triggers** reciben `p_product_id/p_variant_id varchar`
  (`register_stock_transaction` ×3 overloads, `add_products_to_sale`, `revert_sale(_improved)`,
  `cancel_purchase_order`, `create_budget_order`, `convert_units_for_product`, triggers
  `trg_auto_reduce_stock_on_sale`, 9× `tg_notify_catalog_on_*`…). **1 MV**: `products.mv_product_catalog`
  (compara ids con casts `::text`) + `refresh_product_catalog()`.
- Extensiones: `uuid-ossp` instalada (analytics la usa); PG del stack ≥16 → `gen_random_uuid()` nativo disponible.
  `analytics.*` (forecast/goals) ya usa columnas `uuid` — sin FK hacia products.

### 2.2 Backend Go

- Generación del ID: **única por entidad** — `internal/catalog/service.go:81` (producto),
  `internal/platform/postgres/variant_repo.go:483` y `database/postgres/variant.go:202` (variante), más sufijos
  SKU aleatorios `variant_repo.go:117` / `variant.go:108`. `shortid.Generate()` además genera ids de
  users/parties/setup/admin-init (13 call sites BE en total).
- Tipos: `ProductID/VariantID string | *string | sql.NullString` duplicados en ~170 archivos; **cero
  `strconv` sobre ids de producto** (los Atoi de `internal/catalog/http.go` son page/category/attr/tag —
  entidades genuinamente int).
- `github.com/google/uuid v1.6.0` ya está en go.mod — hoy solo correlation IDs, scheduler y tokens identity.
  No hay lib ULID (`grep -i ulid` → 0).
- `txm`/`wiring`: firmas string por todos lados; un cambio de FORMATO no los toca.

### 2.3 Frontend

- `Product.id` / `ProductVariant.id` / `product_id`/`variant_id` de ventas/compras/transferencias/presupuestos/
  requisiciones/pedidos-mostrador: **todos `string`**. Cero `parseInt`/`Number()`/`z.number()` sobre ids de
  producto; sin libs uuid; carrito POS usa id sintético propio (`SalesNew.tsx:737`) y dedupe por
  `(productId, variantId)`.
- Caches Zustand (`useProductStore.ts`) keyed por string con fallbacks `p.product_id || p.id` — mantener
  nombres de campo estables si se toca el contrato.
- SKU hoy: display con fallbacks `sku || barcode || code`; búsqueda por SKU es server-side
  (`granularity: 'variant'` matchea SKU/barcode/nombre); `VariantFormModal` SKU opcional (auto-gen BE);
  **no hay validación de unicidad client-side** (la tiene la DB por unique).

---

## 3. Qué problema resolvería cada pieza de "UUID + SKU"

| Motivación posible | ¿UUID la resuelve? | ¿SKU la resuelve? | Alternativa |
|---|---|---|---|
| Usuario ve códigos crípticos en UI | No (36 chars es peor de mostrar) | **Sí** — código de negocio humano | Opción D |
| Colisiones/entropía del ID | Sí, pero shortid ya es suficiente a este volumen | No | Opción C (ULID) |
| Generación distribuida / offline-first | Sí | No | No hay requisito hoy (on-premise single-node) |
| Lib de generación sin mantenimiento | Sí (google/uuid) | No | Opción C (ULID) — o swap directo |
| Índices fragmentados por inserts random | **Empeora** con UUIDv4 (mejora con v7/ULID) | No | Opción C |
| Integración externa que exige uuid | Sí | No | Driver no existente hoy |

---

## 4. Opciones evaluadas

### Opción A — UUID nativo PostgreSQL (tipo `uuid`) · ❌ costo máximo
Convertir las 30 columnas varchar→uuid: reescritura de todas las tablas + datos (mapeo old→new), drop/recreate
de 37 FKs, redeclarar ~70 funciones + 5 procedures + triggers, regenerar MV, mapear a mano las 2 tablas
`audits.*` sueltas, reescribir los uniques compuestos `COALESCE('__NULL__')` (uuid no coalescea con string;
PG15+ ofrece `NULLS NOT DISTINCT`), y auditar casts implícitos varchar↔uuid en ~313 archivos SQL
(`operator does not exist: uuid = character varying`). Con lib/pq, además, binds de strings contra columnas
uuid exigen cast explícito. Gana: índices de 16 bytes y validación de formato nativa.
En el modelo de distribución on-premise = una ventana de mantenimiento por cliente.

### Opción B — UUID como texto en varchar(36/50) · ⚠️ peor de ambos mundos
`ALTER TYPE varchar(27)→varchar(50)` es metadata-only (PG≥9.2, no reescribe tabla) pero reconstruye los 41
índices (lock ACCESS EXCLUSIVE breve); los shortids viejos conviven sin reescribir datos; solo cambia el
generador (3 call sites) a `uuid.NewString()`. Se pierde la validación nativa de A y se agranda el índice,
ganando esencialmente nada vs. shortid. De paso normalizaría el mismatch varchar(50) de requisiciones.

### Opción C — Swap de generador a **ULID** · ✅ costo mínimo, beneficio real
ULID = 26 chars **cabe exacto en varchar(27)**: **cero DDL, cero SQL, cero FE**. Solo `go.mod`
(`oklog/ulid/v2`) + 3 call sites de generación (producto/variante). 128 bits (48 timestamp + 80 random),
lexicográficamente ordenable → mejor localidad B-tree que UUIDv4, URL-safe, estándar Crockford base32.
Coexistencia con shortids existentes garantizada (ambos son strings ≤27). Resuelve la lib archivada y la
fragilidad del generador. Los shortids ya emitidos NUNCA se reescriben.

### Opción D — **SKU universal** · ✅ resuelve el problema humano
1. Decisión de modelo: (a) `sku` a nivel `products` para el caso simple, o (b) todo producto vendible tiene
   variante (implícita si es simple) — el modelo ya treata "variante = SKU" (comment de la tabla) y el FE ya
   abstrae "sellable unit" (`features/catalog/sellableUnitSearch.ts`).
2. Backfill de SKU para productos sin variantes + validación de unicidad (BE).
3. FE: dejar de mostrar IDs crudos en ~8–15 archivos (`SalesCartGrid`, `MovementForm`, `ProductDetailsModal`,
   `PurchaseProductSearchInput`, `ProductSearchModal`, `BudgetCreate`, `ScaleConfigPage`) y limpiar los 3
   fixtures demo con ids numéricos.

**Recomendación: C + D.** A/B solo ante driver externo explícito (§8).

---

## 5. Impacto por módulo (si se ejecutara A o B)

| Módulo | Impacto | Superficie |
|---|---|---|
| products/catalog | Alto | generación, MV + refresh, search repos, tags/attrs/tax-classif |
| inventory/stock | Alto | stock, unit_prices/costs, price_transactions, ~15 funciones de stock/costo |
| sales | Alto | `add_products_to_sale`, `revert_sale`, trigger auto-reduce-stock, reserves |
| purchases | Medio-alto | order details, `cancel_purchase_order`, requisiciones |
| budgets / counterorders | Medio | detail tables, `create_budget_order` |
| transfers | Medio | items (FK RESTRICT) + unique compuesto con sentinel `'__NULL__'` |
| manufacturing | Medio | batches, recipes, funciones de costo |
| pricing | Medio | 10 archivos, rutas `/product/{product_id}/history` |
| analytics/BI | Medio | 41 read models + drill-downs `?product_id=` (sin FK, solo display) |
| documents | **Ninguno** | cero referencias a product_id |
| licensing / identity | Ninguno | (products) |
| Frontend | ~0 tipos | string ya en todo; solo display (D) |
| Deploy | Alto | migración en bundle update por cliente + regen `make db-baseline` |

Para C+D el impacto de A/B desaparece: C no toca DB y D es BE (1 columna/backfill o variante implícita) + FE display.

---

## 6. Fases propuestas (C+D)

- **F1 — ULID swap (C)**: agregar `oklog/ulid/v2`, reemplazar los 3 call sites de shortid de producto/variante
  (dejar el resto de entidades para otra pasada), tests de formato/longitud, smoke crear producto+variante.
- **F2 — SKU universal (D-decisión)**: decidir columna `products.sku` vs variante implícita; migración
  (`make migrate-create`) + backfill + unique; BE valida y expone SKU en sellable units/flat search.
- **F3 — FE display**: reemplazar IDs crudos por SKU en los ~8–15 componentes; limpiar fixtures demo numéricos;
  i18n keys nuevas en es+en si hacen falta.
- **F4 — (opcional) normalizar** `purchase_requisition_details.product_id` varchar(50)→varchar(27).
- **F5 — Gates**: `go test ./...`, instancia efímera + smoke (crear producto simple y con variantes, verificar
  ULID 26 chars y SKU visible en POS/compras/presupuestos), FE `vitest --run`, `tsc --noEmit`, `pnpm build`,
  `lint:design`, regen baseline.

**Plan contingente (solo si el owner decide A/B)**: fase 0 de decisión de formato → script de mapeo
old→new transaccional (30 columnas + FKs + funciones + MV + audits) → dry-run con docker-shim → ventana de
mantenimiento por cliente → baseline regen → re-validación completa.

## 7. Gotchas (aplican a cualquier ejecución)

- Uniques compuestos con `COALESCE(variant_id,'__NULL__')` / `''`: explotan con tipo uuid nativo; con ULID/varchar ni se tocan.
- Widening varchar (B) no reescribe tabla pero sí reconstruye índices → lock breve; en cliente, coordinar ventana.
- `gen_random_uuid()` nativo PG13+ (sin pgcrypto); `uuid-ossp` ya instalada pero es la vía vieja.
- lib/pq + columnas uuid nativas: strings requieren cast explícito — auditar antes de prometer A.
- MV `mv_product_catalog` compara ids con `::text` — cualquier cambio de tipo exige regenerar MV + `refresh_product_catalog()`.
- `audits.*` sin FK: cualquier re-mapeo de IDs existentes es manual.
- FE: mantener nombres de campo (`id`, `product_id`) — las cadenas de fallback `p.product_id || p.id` y la
  invalidación de caché por igualdad string dependen de ellos.
- Los shortids de OTRAS entidades (users/parties/sales_orders) viven en el mismo patrón varchar — no arrastrar
  el swap más allá de producto/variante sin plan propio.
- Baseline + fresh-install (`baseline` + `migrate force`) debe quedar consistente tras cualquier DDL (README de baseline).

## 8. Decisiones diferidas al owner

1. **Driver real**: ¿mostrar códigos amigables (→D), robustez del generador (→C), o estándar externo (→A/B)?
2. **SKU simple**: columna `products.sku` vs variante implícita única para productos simples.
3. **¿Migrar IDs existentes?** Recomendado: NO — solo cambiar generación hacia adelante.
4. Si A/B: ventana de mantenimiento por cliente on-premise y orden de clientes (dev → piloto → resto).
