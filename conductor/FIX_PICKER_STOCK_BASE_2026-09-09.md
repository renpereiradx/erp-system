# Fix picker de productos: stock de variantes en 0 + producto base no seleccionable (2026-09-09)

> Reporte del usuario con capturas: en el catálogo (`CatalogCard`) y en el
> builder de pedidos (`OrderBuilder`), las variantes mostraban `STOCK: 0` fijo
> y no se podía agregar el producto base, solo sus variantes.
> Commit FE: `280b4ad` (repo erp-webapp, branch dev).

## Causa raíz

1. **Stock de variantes en 0**: ambos componentes leen
   `variant.stock_quantity ?? 0`, pero `useCatalogVariants` traía el listado
   crudo `GET /products/{id}/variants`, que **nunca** llena `stock_quantity` ni
   `current_price` (son campos `omitempty` que solo puebla el path enriquecido).
   El modelo real: `products.stock` tiene filas por `(id_product, variant_id
   nullable, branch_id)` — p. ej. CAMISETA ADIDAS tiene base 41@b1+3@b3 y
   variantes 1/2@b1-b3 y 20@b3; el "67" de la tarjeta es el total de TODAS las
   filas (variante+sucursal) que calcula el advanced-search.
2. **Base no seleccionable**: `ProductPickCard` solo renderizaba botón Agregar
   para productos SIN variantes; con `has_variant` exponía únicamente las filas
   de variantes. La línea base (variant NULL) es legítima: el backend la acepta
   (precio por `get_active_price`, stock propio en filas variant NULL), y hay
   stock base real (44 u. de ADIDAS base).

## Fix

- `useCatalogVariants` ahora usa la MISMA fuente que el selector de /ventas:
  `variantService.getEnrichedVariants(productId, currentBranchId)` — por
  variante pide `GET /api/v1/variants/{id}/stock?branch_id=` (permiso
  products:read; la sucursal activa del JWT siempre está permitida, otras dan
  403 — verificado E2E) y precios desde `/products/{id}/units`.
  De paso llena `current_price` de la variante en `CatalogCard` (antes "—").
- `ProductPickCard`: los productos con variantes tienen una fila
  **"Producto base"** con su stock y botón [+] (mismo `data-testid`
  `counterorder-add-<productId>` que los simples), además de "Elegir variante".
- La línea del carrito ahora distingue `PRODUCTO · VARIANTE` en su nombre (el
  backend solo persiste `variant_id` crudo — hallazgo medio de la auditoría).

## Semántica del stock (para leer las capturas después del fix)

- Chip de la tarjeta y fila base: `stock_quantity` del advanced-search = TOTAL
  de todas las filas del producto (variantes + sucursales). Semántica
  preexistente del catálogo, sin cambiar.
- Filas de variante: stock de la **sucursal activa** — el número comparable con
  lo que el detalle del pedido resuelve (`stockFor` usa la sucursal del pedido,
  fix A2) y con la validación hard del checkout. Una variante con stock solo en
  otra sucursal sigue mostrando 0 en la activa: es real.

## Tests y gates

- `CounterOrdersPage.test.tsx`: mock del catálogo controlable por test; nuevos
  casos: stock de variantes enriquecido (5/0, no 0 fijo), agregar producto base
  + variante (líneas con keys distintas, "2 unidades"), producto simple sin
  regresión.
- `CatalogBoard.test.tsx`: mockea la frontera nueva (`@/contexts/BranchContext`)
  y `getEnrichedVariants`; el caso "expande variantes" assertion de
  `Stock: 10` + llamada con sucursal.
- Gates: tsc 0 · vitest 613/613 · `pnpm build` ✓ · `lint:design` ✓.
- Verificado en vivo contra el backend corriendo (:5050): stock por variante
  devuelve `total_stock: 1` para NEGRO-M @b1 (antes 0 fijo en la UI). El fix es
  FE-only: con el HMR del vite dev ya está visible en :5173.
