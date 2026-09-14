# PLAN: Pedidos de Mostrador — FASE 5 ola 2 (ticket QR, métricas, stock en caja)

**Fecha:** 2026-09-13
**Estado:** ✅ IMPLEMENTADO (2026-09-13, pendiente commit; gates verdes + smoke API)
**Padre:** `PLAN_PEDIDOS_FASE5_MEJORAS.md` (5A TTL + 5B convert in-tx ✅ 2026-09-13) → `PLAN_PEDIDOS_MOSTRADOR_VENDEDOR_CAJA.md`
**Alcance:** Cross-repo → plan en `conductor/` raíz

### Registro de implementación (2026-09-13)

- **Ticket QR (FE render)**: `OrderTicketModal` + `OrderTicketContent` (estilos inline: viajan al iframe vía innerHTML — el iframe no hereda el CSS de la SPA) + `utils/printTicket.ts` (iframe efímero con `@page 80mm`, aislado para mockear en tests; el `window.print()` legacy de SalesPaymentHistory imprimía la página entera). QR con `qrcode.react` (ya estaba en package.json), value = código PED (la búsqueda de la bandeja matchea por ILIKE → escaneable). Entradas: botón en el footer de `OrderDetailModal` + apertura automática al guardar pedido NUEVO (`handleSaved` create-mode; editar sigue abriendo el detalle). Sin endpoint ni permiso nuevo.
- **Métricas**: `CounterOrderMetrics` (created/converted/cancelled/expired/active/avg_minutes/conversion_rate); SQL con `COUNT(*) FILTER` + `AVG(EXTRACT(EPOCH…))` en una pasada; servicio normaliza days (default 30, cap 365) y redondea; ruta literal `/counter-orders/metrics` ANTES del subrouter con `RequireModulePermission(reports:read)` (patrón discounts/sales-analytics). FE: `CounterOrdersMetricsPanel` (react-query `['counter-orders','metrics',days]`, rango 7/30/90, 6 tarjetas) montado en `/pedidos` SOLO con `hasPermission('reports:read')` — el query no se dispara sin permiso.
- **Stock en caja** (modelo "reserva solo si está pagado" — sin cambio de motor, documentado): (1) toast de advertencia al reclamar si el detalle trae ítems con `stock_warning` (`useCounterOrderCheckout.continueOrder`); (2) badge destructivo "Sin stock"/"Stock insuficiente: N" en `SalesCartGrid` (desktop + móvil, labels calculados una vez por render en un Map); (3) rama `INSUFFICIENT_STOCK`/`stock insuficiente` en el catch de `onConfirmWizard` → toast con el detalle del SQL (producto, disponible, solicitado); la venta ya se revierte entera server-side.
- **i18n**: keys `counterorders.metrics.*`, `counterorders.ticket.*`, `counterorders.checkout.stock_warning` (es/counterorders.js) + `sales.errors.insufficientStock`, `sales.cart.outOfStock/lowStock` (es/sales.js); `en` hereda `es`.
- **Auditoría vercel-react-best-practices** (componentes nuevos): imports directos de `qrcode.react`/`lucide` (bundle-barrel), sin barrel nuevo; query gated en la página (no fetch sin permiso); labels de stock en un Map calculado 1×/render (js-index-maps); sin efectos derivados ni estado espejo; `printTicketHtml` mockeable en la frontera del módulo. Sin hallazgos pendientes.
- **Tests**: BE `TestCounterOrderMetrics` (default/cap/branch/tasa 0, redondeo). FE: `CounterOrdersMetricsPanel.test` (valores + reconsulta por rango), `OrderTicketModal.test` (contenido, QR svg, impresión serializa HTML con código+ítems+svg, estado vacío), `SalesCartGrid.test` nuevo (badge 0/insuficiente/OK; el grid renderiza desktop+móvil → `getAllByTestId`), hook test +2 casos (toast con/sin stock), mock factory de la página con `metrics`.
- **Smoke API (dev)**: backend relanzado con bin nuevo → `GET /counter-orders/metrics?days=30` con JWT admin minteado (HS256 + JWT_SECRET local) → 200 `{"created":7,"converted":6,"expired":1,"avg_minutes_to_convert":1439.6,"conversion_rate":85.7}` (agregación real del dev DB); con token CAJA01 → **403** (gate); listado original sigue 200. El puerto 5050 lo retenía el proceso viejo (pid distinto, nombre "main") — matar por `ss -tlnp`, no por nombre.
- **Gates**: BE `make build` + `go test ./...` 0 fail + `golangci-lint` verde. FE: `tsc --noEmit` 0, vitest **640/640** (94 archivos), `pnpm build` + `lint:design` ("Código nuevo limpio"). Sin migraciones.

## 0. Decisiones del owner (2026-09-13)

| Ítem | Decisión |
|:-----|:---------|
| Ticket/QR (§5.5) | **FE con render** — impresión local, sin endpoint ni permiso nuevo (no toca `documents:*`) |
| Métricas | Hacerlas ahora. Gate `reports:read` (hoy solo F2VLso; alineado al reporte de descuentos. ENCR01 quedaría fuera — un grant lo agrega si se quiere) |
| Reserva de stock (§5.2) | **"Solo si está pagado"**: NO se reserva stock al crear/reclamar el pedido; el stock se compromete únicamente vía la venta pagada (`process_sale_with_reserve`, ya existente). Lo que falta es la **superficie en caja**: al procesar, mostrar claramente que ya no hay stock disponible |

### Cómo se maneja hoy el stock en caja (revisado, §"revisar como se maneja eso")

1. **Resolve-on-read**: el detalle del pedido trae `stock_available` + `stock_warning` por ítem (sucursal del pedido). Se muestra en `OrderDetailModal` (vista vendedor, `/pedidos`).
2. **Claim**: los ítems entran al carrito con `stock` (mapeo en `useCounterOrderCheckout.mapItems`) — pero `SalesCartGrid` **no muestra stock** y nadie avisa al reclamar.
3. **Al procesar**: `process_sale_with_reserve` hard-valida y levanta `INSUFFICIENT_STOCK: Stock insuficiente para producto X: disponible Y, solicitado Z`. El catch de `SalesNew.onConfirmWizard` NO tiene rama para stock → cae al toast genérico "No se pudo completar la venta y el cobro".
4. El paso de pendientes del wizard muestra pedidos sin stock (los summaries no traen stock; requeriría N fetches — se deja así, el aviso vive en claim + carrito + checkout).

## 1. Ticket QR del pedido (FE, render local)

- **`OrderTicketModal.tsx`** (`features/counterorders/components/`): preview estilo ticket 80mm (mono, líneas), con: título del local/branch, código `PED-XXXXXX` grande, **QR** (`qrcode.react` `QRCodeSVG`, value = código PED — escaneable en la búsqueda de la bandeja), fecha, vendedor, cliente, ítems (nombre, cant×unidad, total de línea), TOTAL, nota del vendedor y pie "Precios vigentes al emitir; el cobro se confirma en caja".
- **`utils/printTicket.ts`**: imprime vía iframe oculto (srcdoc con el HTML del ticket + CSS @page 80mm) — no imprime la página entera (el patrón `window.print()` de SalesPaymentHistory no sirve acá). Mockeable en tests.
- **Entradas**: botón "Imprimir ticket" en el footer de `OrderDetailModal` + apertura automática al **guardar un pedido nuevo** (`onSaved` en modo create — el vendedor se lo da al cliente).
- Gate: ninguno nuevo (la página ya es `counterorders:read`; impresión local no toca backend).

## 2. Métricas de pedidos (BE + FE)

- **BE**: `CounterOrderMetrics` (created, converted, cancelled, expired, active, avg_minutes_to_convert, conversion_rate). SQL de agregación sobre `counter_orders` con `COUNT(*) FILTER` + `AVG(EXTRACT(EPOCH ...))`, ventana `created_at >= now()-days`, opcional `branch_id`.
  - Puerto: `Metrics(ctx, since, branchID)` en `CounterOrderRepository`; servicio normaliza `days` (default 30, cap 365); handler parsea query.
  - **Ruta**: literal `GET /counter-orders/metrics` registrado **ANTES** del subrouter `/counter-orders` (mux matchea prefijos en orden de registro, patrón discounts/sales-analytics) con `RequireModulePermissionOnPath(reports:read)`. El subrouter del feature exige `counterorders:read` — que VNDR01 tiene — por eso la métrica va por fuera con gate propio.
- **FE**: `CounterOrdersMetricsPanel` arriba de la bandeja `/pedidos`, renderizado solo con `hasPermission('reports:read')`: tarjetas (creados, convertidos, tasa, tiempo medio, activos, cancelados+vencidos) + selector de rango 7/30/90 días. `counterOrderService.metrics(days)` con react-query (`['counter-orders','metrics',days]`).

## 3. Stock visible en caja

1. **Toast al reclamar** (`useCounterOrderCheckout.continueOrder`): si algún ítem trae `stock_warning`, toast de advertencia "el pedido tiene ítems sin stock suficiente" — el cajero lo sabe ANTES de cargar.
2. **Badge en el carrito** (`SalesCartGrid`, desktop + móvil): `Sin stock` cuando `stock === 0`, `Stock insuficiente` cuando `stock < quantity` (el `CartItem.stock` ya viaja; hoy nadie lo pinta).
3. **Error de checkout claro**: rama `INSUFFICIENT_STOCK` / "stock insuficiente" en el catch de `onConfirmWizard` → "No hay stock disponible para procesar la venta" + el detalle del SQL (producto, disponible, solicitado). La venta ya se revierte entera en el backend (tx atómica); el operador puede liberar el pedido o coordinar con el cliente.
4. Modelo de reserva queda documentado: **stock se compromete solo vía pago** (venta pagada); pedidos OPEN/CLAIMED no tocan stock.

## 4. Gates y tests

- BE: `make build` + `go test ./...` + `golangci-lint run`; tests del servicio de métricas (days default/cap, branch, SQL mockeado) + smoke API con token admin (EXPLAIN + respuesta real).
- FE: vitest (panel metrics gated, ticket modal con util mockeada, badge del carrito en test nuevo de SalesCartGrid, toast de claim en test del hook, mock factory con `metrics`), `tsc --noEmit`, `pnpm build`, `pnpm lint:design`.
- React: aplicar `vercel-react-best-practices` (import directo de `qrcode.react`, sin effects derivados, memo innecesario evitado); update del reporte de auditoría del feature.
- Sin migraciones de DB.
