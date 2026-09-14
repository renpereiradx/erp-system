# FIX AJUSTES DE PRECIO VARIANT-AWARE + HISTORIAL DE PRECIOS — 2026-09-14

Owner report: (1) en la página de Ajustes de Precios, cambiar el precio de una
variante cambiaba el del padre. (2) En el modal de detalles de productos, el
historial de precios estaba incompleto. Ambos verificados contra BD y API en
vivo; fixes en FE + BE. Gates verdes.

## Causas raíz

### Bug 1 — ajuste de variante caía al padre
1. **FE `priceAdjustmentService.createPriceAdjustment`**: reconstruía el body
   del POST `/manual_adjustment/price` y **descartaba `variant_id`** que la
   página de detalle sí incluía. El backend (SQL variant-aware desde
   20260620132118) recibía NULL → upsert de la fila del padre.
   Evidencia: ajustes 13-15 en `products.manual_price_adjustments` del
   2026-09-14 con `variant_id = NULL` (la fila 30 del padre saltó 75.000→80.000
   a las 12:26, minuto de la captura del owner).
2. **FE `PriceAdjustmentDetail.jsx`**: mapeaba variantes enriquecidas por
   `v.variant_id` — no existe: traen `id`. El `<select>` mostraba
   "Producto Principal" y el `old_price` se resolvía contra filas del padre
   (unit_prices del padre enriquecido viaja mezclado padre+variantes).

### Bug 2 — historial incompleto
1. **BE**: `GET /price-transactions/product/{id}/history` devolvía **500**:
   `GetProductPriceHistory` (Go) llama `products.get_product_price_history`
   con 4 args y escanea `branch_id/unit_price_id/unit/price_type`, pero en BD
   solo existía el overload de 3 args / 20 columnas. Drift repo↔DB: la
   función actualizada jamás se migró.
2. **FE**: `priceTransactionService.getProductHistory` enmascaraba el 500 con
   **fallback demo** (1 fila falsa) en development → el "historial incompleto".
3. **FE**: el endpoint que consumía "Historial de Ajustes" de la página de
   ajustes (`/manual_adjustment/product/{id}/history`) es de **stock**
   (inventory) — filtraba `adjustment_type==='price'` y quedaba vacío siempre.
   Además `GetPriceAdjustmentsByDateRange` no seleccionaba `variant_id` y su
   filtro estricto por `metadata->>'branch_id'` vaciaba la lista con sucursal
   activa (0/15 filas la traen).

## Fixes

**Backend (migración 20260914125728 + Go)**
- `price_transactions.branch_id` (INTEGER, índice parcial) + backfill desde
  `metadata->>'branch_id'`. Sin ella, `GetPriceTransactionById` y
  `GetPriceTransactionsByDateRange` también 500eaban por columna faltante.
- Nuevo overload `get_product_price_history(varchar, int, int, int)`: 25
  columnas (agrega `branch_id`, `variant_id`, `unit_price_id`, `unit`,
  `price_type`), filtro de sucursal **tolerante a filas legacy** (`branch_id
  IS NULL OR = $4`), join a `users.parties` (los nombres/email viven ahí, no
  en `users.users` — bug latente también en la función vieja).
- `PriceTransactionHistory.VariantID` + SELECT/scan; `GetPriceAdjustmentsByDateRange`
  selecciona/escanea `variant_id` y su filtro de branch tolera NULL.
- Enum de `adjustment_type` en el schema de metadata valida ahora
  `MANUAL_ADJUSTMENT` (el modal de producto lo envía; si no, 400).

**Frontend**
- `priceAdjustmentService.createPriceAdjustment`: forward `variant_id`.
- `PriceAdjustmentDetail.jsx`: `getVariantId(v)` normaliza `variant_id ?? id`
  (select, find, chips de historial); `getUnitPriceFor(..., requireParent)`
  ignora filas de variantes al mostrar el precio del padre; `loadHistory` usa
  `/manual_adjustment/price/date-range?product_id=` (endpoint de precios) en
  vez del de stock; chip con nombre de variante en cada fila del historial.
- `priceTransactionService.getProductHistory`: **fallback demo eliminado**
  (los errores se propagan; los datos falsos ocultaron el 500 por semanas).
- `ProductDetailsModal` + `ProductHistoryModals`: chip de variante en la tabla
  "Precios por Unidad" (filas `variant_id` mezcladas con el padre), mapa
  `variantNameById` al historial (tag por fila), y "Ajustar Precio" de una fila
  de variante ahora ajusta ESA variante (el diálogo usa
  `/manual_adjustment/price` con `variant_id`, estado de error local para no
  cerrarse en fallo). i18n: `products.details.table.variant_tag` (es/en).

## Verificación (en vivo, dev :5050)
- POST ajuste con `variant_id=glxPj--vg` → `unit_price_id: 88` (fila de la
  variante), fila 30 del padre intacta; restore 70.000→67.990 (ajustes 16-17
  quedan como auditoría de la prueba).
- History endpoint: 200 con y sin `X-Branch-ID`; transacciones de variantes
  listadas con `variant_id`.
- Gates: `go test ./...` 36 ok / 0 fail; FE vitest 626/626, `tsc --noEmit` 0,
  `pnpm build` OK, `lint:design` "Código nuevo limpio".

## Deuda / notas
- Ningún writer de `price_transactions` puebla `branch_id` todavía (columna
  nueva, filas NULL = visibles en todas las sucursales). Cuando se agregue
  branch a `register_*_transaction`, el filtro ya está listo.
- El fallback demo persiste en otros métodos del service (validateConsistency,
  types) — mismo patrón de enmascaramiento, a revisar on-demand.
- Migración aplicada en dev (make migrate-up); prod la recibe con el deploy.

---

# SEGUIMIENTO (mismo día): FILTROS DEL HISTORIAL GLOBAL — BE 66c4049 / FE d2c5dda

Owner report: "mejora el manejo de rango de fechas, revisa los demás filtros,
haz que Aplicar Filtros funcione óptimo" (página /ajustes-precios/historial).

## Causas raíz (los filtros nunca llegaron a la API)

1. La página llamaba `getRecentAdjustments({page, limit, ...filters})` — la
   firma real es `(limit, days)`: pasaba un objeto, el backend devolvía
   `limit=50` default **de los últimos 7 días**, sin ningún filtro.
2. Los rango de fechas eran inputs de TEXTO con defaults hardcodeados
   01/10/2024–31/10/2024 que no se enviaban a ningún lado (y ni siquiera
   matchearían el formato `YYYY-MM-DD 00:00:00` del backend).
3. usuario/unidad/tipo eran inputs de texto que el endpoint ignora por diseño
   (solo soporta fechas + product_id exacto).
4. El header "Mostrando 1 a 0 de 0": total = length de página; sin rows → 0.
   Además `GetRecentPriceAdjustments` filtraba `metadata->>'branch_id'` de
   forma estricta: 0/15 filas lo traen → lista global vacía con sucursal
   activa (exactamente la captura del owner).
5. Fallback demo: si la API fallaba, la página pintaba 3 filas falsas con
   total simulado 150.

## Fixes

**Backend (66c4049)** — `GET /manual_adjustment/price/date-range`:
- Params nuevos: `user` (ILIKE user_id/username/nombre party), `unit`
  (exacto), `adjustment_type` (metadata). `product_id` pasó de igualdad
  exacta a búsqueda "nombre o SKU": ILIKE sobre p.name, p.id y **pv.sku**
  (LEFT JOIN product_variants).
- Rows enriquecidas: product_name, user_name (join users.parties), variant_name.
- `COUNT(*) OVER()` → `total` real en la respuesta para paginar.
- `GetRecentPriceAdjustments` ahora también es branch-tolerante a filas NULL.
- Port/service: filtro agrupado en `pricing.PriceAdjustmentFilter` (sin firma
  de 9 strings).

**Frontend (d2c5dda)** — página reescrita:
- `getByDateRange(dateFrom, dateTo, product, 25, offset, {user, unit,
  adjustment_type})` — todo server-side.
- Filtros draft vs applied: tipear no dispara requests; Aplicar/Enter aplican
  y resetean a página 1; rango invertido muestra error inline (ISO compara
  lexicográfico) y NO fetchea.
- Fechas: inputs nativos `type="date"` (ISO directo), sin defaults falsos —
  vacío = todo el historial.
- Unidad y Tipo: `<select>` reales (catálogo de unidades y enum de metadata).
- Total server-side; header colapsa a "resultados" cuando es 0.
- Fallback demo ELIMINADO (error state con reintento); chip de tipo humanizado
  desde metadata; chip de variante en la fila; user_name en vez del id crudo.
- i18n: dateRangeError, table.variant, adjustmentType.* (8 valores) en es.

## Verificación en vivo (instancia efímera :5099 — el :5050 era del usuario)

- Sin filtros: total 18, rows con product/variant/user_name.
- product_id=NEGRO-M (SKU de variante) → 3 rows; producto+user+tipo → 2;
  fechas 14/09 + unit → 6; tipo inexistente → 0.
- Nota: el proceso :5050 era un `go run` del USUARIO con binario viejo — mi
  instancia nueva murió por puerto ocupado (bind: address already in use).
  Regla nueva del usuario: al terminar, SIEMPRE liberar el puerto.
- Gates: go test ./... 36 ok; vitest 629/629 (page test reescrito con mocks
  estables de i18n — t nuevo por render refetchearía en bucle), tsc 0, build,
  lint:design.
