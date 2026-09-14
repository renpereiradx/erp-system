# PLAN: Pedidos de Mostrador — FASE 5 (mejoras post-v1)

**Fecha:** 2026-09-13
**Estado:** ✅ IMPLEMENTADO (5A + 5B, 2026-09-13, pendiente commit; E2E por API en dev verde)
**Padre:** `PLAN_PEDIDOS_MOSTRADOR_VENDEDOR_CAJA.md` (FASES 1–4 ✅ 2026-09-08)
**Alcance:** Cross-repo (backend `business_management` + frontend `erp-webapp`) → plan en `conductor/` raíz

### Registro de implementación (2026-09-13)

- **5A TTL**: puerto `SweepExpired` en `CounterOrderRepository` + repo SQL (`UPDATE ... SET status='EXPIRED' WHERE status='OPEN' AND created_at < now()-make_interval(mins=>$1) RETURNING id`, usa `idx_counter_orders_status_branch`) + const `counterOrderExpiryTTL = 72h` en el servicio; sweep junto al de claims en `List` y `Claim`. Test `TestCounterOrderList_SweepsExpiredWithTTL` fija los TTLs exactos; mocks actualizados en los 5 sitios de expectativa. FE: chip "Vencidos" en `STATUS_OPTIONS` (badge/i18n ya existían future-proof de FASE 2) + test del filtro. EXPIRED queda sin acciones en la bandeja (los gates de acciones ya eran OPEN/CLAIMED).
- **5B Convert in-tx**: puerto nuevo `CounterOrderCloser` (`MarkConverted` + `GetByID`, ya implementados por `CounterOrderRepo`); `POSCheckoutRequest.counter_order_id` y `AddProductsToSaleRequest.counter_order_id` (DTOs, cero cambios de handler); `POSService.SetCounterOrders` + in-tx `MarkConverted` después del pago exitoso en `Checkout`; `SaleService.SetCounterOrderConversion(tx, closer)` envuelve `AddProducts` + `MarkConverted` en `txm.Run` (el SQL del repo se une vía `txm.TxFromContext`). Wiring: un `CounterOrderRepo` compartido inyectado a ambos servicios dentro del lazy-init de `SaleService`. Guard estricto: 0 filas → 409 con el código del pedido y rollback total (venta + pago + stock). Tests: 3 en `pos_service_test.go` (convert en tx vía marcador de ctx, no-claim → 409 con código, not-wired → error duro) + 3 en `sale_service_test.go` (ídem para el merge). FE: `counter_order_id` en `types.ts`, propagado en `salePaymentService.posCheckout` (normalized) y en los 3 sitios de merge + pos-checkout de `SalesNew.tsx` vía `counterOrderFlow.getClaimedOrderId()`; `convertAfterCheckout` post-hoc se conserva como cinturón idempotente (no-op 200) y path de deploy-skew.
- **E2E por API (dev, token CAJA01 `e2ecaja01`)**: EXPLAIN del SQL de sweep (usa índice de status) → list 200 (sweep en camino real) → create `PED-CK8SRP` OPEN total 110500 → claim CLAIMED → `pos-checkout` con `counter_order_id` crea `SALE-1789314971-271` → pedido **CONVERTED con converted_sale_id sin llamada post-hoc** → `/convert` idempotente 200 → pedido OPEN con `created_at` -8d queda **EXPIRED** al listar y aparece en el filtro EXPIRED.
- **Path que conserva el two-step**: "Dejar pendiente" de venta NUEVA con ítems de pedido (`createSale`, C3 path) sigue usando el convert post-hoc — `POST /sale/` legacy no llevó el parámetro (fuera del alcance §2 del plan padre); el `/convert` idempotente lo cubre.
- **Gates**: BE `make build` + `go test ./...` (0 fail) + `golangci-lint run` verde; FE vitest 623/623 (90 archivos), `tsc --noEmit` 0, `pnpm build` + `pnpm lint:design` ("Código nuevo limpio"). Sin migraciones de DB.
- **Datos de prueba dejados en dev DB**: venta `SALE-1789314971-271` + pedidos `PED-CK8SRP` (CONVERTED) y 1 EXPIRED del test TTL (usuario e2ecaja01, cliente Fernando Maciel, producto Mouse Protek).

## 0. Alcance elegido

La FASE 5 del plan padre define 5 mejoras "post-v1, on demand". De las 5:

| Ítem | ¿En este plan? | Motivo |
|:-----|:---------------|:-------|
| **5A. Auto-expiración TTL de OPEN** (§5.3) | ✅ Sí | El más barato: el estado `EXPIRED` ya existe en el CHECK y en el dominio (reservado). Mismo patrón sweep-al-listar del auto-release de claims. |
| **5B. Convert atómico in-tx** (§6 FASE 5) | ✅ Sí | Elimina el riesgo principal del diseño v1 (§7: "caja muere entre checkout y convert" → pedido CLAIMED huérfano + toast de reintento). |
| Ticket/QR imprimible (§5.5) | ⏸ Bloqueado por decisión del owner | Requiere decidir el permiso de impresión del vendor (`documents:*` revocado a VNDR01): FE imprime local vs. render gated a `counterorders:write`. La infraestructura de impresión existe (módulo documents, 2026-09-05). |
| Reservas de stock (§5.2) | ⏸ On demand | "Solo si la operación lo pide" (plan padre). El esquema resolve-on-read + validación hard sigue vigente. |
| Métricas | ⏸ On demand | Sin señal de la operación; ojo: el vendor perdió `dashboard:read` en v3 — las métricas serían para admin/encargado. |

Sin migraciones de DB: el TTL es app-level (const, como los 20 min del claim) y el convert in-tx solo agrega un parámetro opcional a DTOs existentes.

## 1. 5A — Auto-expiración TTL (backend)

- **Constante**: `counterOrderExpiryTTL = 72 * time.Hour` en `counter_order_service.go` (mismo patrón que `counterOrderStaleClaimTTL = 20m`). 72 h cubre findes; constante fácil de ajustar.
- **Repo** (`counter_order_repo.go`): `SweepExpired(ctx, olderThan) ([]string, error)` —
  `UPDATE ... SET status='EXPIRED', updated_at=now() WHERE status='OPEN' AND created_at < now() - make_interval(mins => $1) RETURNING id`.
  OPEN no tiene `claimed_by` (CHECK `ck_counter_orders_claim_present`), no hay nada más que limpiar.
- **Puerto**: `SweepExpired` se agrega a `CounterOrderRepository` (`port.go`).
- **Servicio**: sweep junto al de claims en `List` y `Claim` (mismo "sin cron, sweep barato al listar").
- **Ciclo de vida**: `EXPIRED` es terminal (como CANCELLED): sin re-apertura — el vendedor crea un pedido nuevo. `IsActive()`/`IsEditable()` ya lo excluyen; `MarkCancelled` solo acepta OPEN/CLAIMED (un EXPIRED no se cancela, ya está muerto).
- **Tests**: mock del repo gana `SweepExpired`; tabla de casos: List/Claim invocan el sweep con el TTL correcto; fallo del sweep no bloquea… (decisión: sí propaga, igual que SweepStaleClaims hoy).

### Frontend (mínimo)

`EXPIRED` ya existe en `types.ts` (`COUNTER_ORDER_STATUSES`), `OrderStatusBadge` (icono Archive) e i18n (`counterorders.status.EXPIRED: 'Vencido'`) — FASE 2 lo dejó future-proof. Falta solo:
- `CounterOrdersPage.tsx`: opción `EXPIRED` en `STATUS_OPTIONS` (chip "Vencidos").
- Test: el chip aparece en la bandeja.

## 2. 5B — Convert atómico in-tx (backend)

**Principio**: el marcaje `CONVERTED` entra en la MISMA tx que crea/cobra la venta. Los repos ya se unen a la tx activa vía `txm.TxFromContext`, así que basta llamar a `MarkConverted` con el txCtx.

- **Puerto nuevo** (`pos_service.go`): `CounterOrderCloser { MarkConverted(ctx, id, saleID) (bool, error); GetByID(ctx, id) (*CounterOrderRecord, error) }` — ya implementado por `CounterOrderRepo` (mismas firmas).
- **pos-checkout**: `POSCheckoutRequest` gana `counter_order_id *string` (json `omitempty`). En `POSService.Checkout`, dentro del `tx.Run`, después del pago exitoso: `MarkConverted(txCtx, id, sr.SaleID)`. Inyección por setter `SetCounterOrders` (patrón `SetPermissionChecker`; nil = deshabilitado en tests).
- **Merge path**: `AddProductsToSaleRequest` (domain.go) gana `counter_order_id *string`. `SaleService` gana `SetCounterOrderConversion(tx txm.Manager, closer CounterOrderCloser)`; en `AddProductsToSale`, si hay pedido vinculado, `processor.AddProducts` + `MarkConverted` se envuelven en `tx.Run` (el SQL de `SaleRepo.AddProducts` ya se une a la tx externa).
- **Semántica estricta (decisión de diseño)**: `MarkConverted` conserva su guard `WHERE status='CLAIMED'`. Si devuelve 0 filas → 409 con el código del pedido y **rollback completo** (venta + pago + stock). Motivo: el guard estricto es la última línea de defensa anti doble-caja — si el claim fue liberado por el sweep (20 min) y dos cajas intentan procesar, la segunda venta se revuelve entera en vez de concretar el doble cobro. El caso "claim venció mientras el cajero cobraba" es raro y tiene salida clara: re-abrir desde /pedidos (el pedido vuelve a OPEN). El `/convert` post-hoc **se conserva** como path de recuperación (y de deploy-skew BE/FE).
- **Semántica del merge**: el convert ocurre al aterrizar los ítems en la pendiente (no tras `confirm-payment` como hoy). Si el cobro luego falla, el pedido queda CONVERTED con `converted_sale_id` = la pendiente que YA tiene los ítems — trazabilidad correcta; hoy en cambio queda CLAIMED con los ítems ya mezclados (la ventana de crash que 5B mata).
- **Idempotencia con la FE**: la FE sigue llamando `convertAfterCheckout(saleId)` tras el checkout — con el convert in-tx ya hecho es un no-op 200 (mismo sale_id); cubre deploy-skew si el BE viejo ignora el parámetro.
- **Wiring** (`internal/platform/wiring/sale.go`): reusar el repo de `CounterOrderService()` → `saleSvc.SetCounterOrderConversion(c.TxManager(), repo)` + `posSvc.SetCounterOrders(repo)`.
- **Handlers**: cero cambios (los DTOs viajan por json).

### Frontend

- `types.ts`: `POSCheckoutRequest.counter_order_id?: string`; `AddProductsToSaleRequest.counter_order_id?: string`.
- `salePaymentService.ts` (`posCheckout`): el `normalized` reconstruye sale+payment — propagar `counter_order_id`.
- `SalesNew.tsx`: en el payload de `posCheckout` y en los 3 `addProductsToSale` del flujo de merge, agregar `counter_order_id: counterOrderFlow.getClaimedOrderId() ?? undefined` (si no hay pedido reclamado queda `undefined` y el backend no hace nada). `useCounterOrderCheckout.ts` sin cambios funcionales.

## 3. Gates

- **Backend**: `make build` + `go test ./...` + `golangci-lint run` (sin migraciones → sin `migrate-up`).
- **Frontend**: `npx vitest --run` 0 fallos + `npx tsc --noEmit` + `pnpm build` + `pnpm lint:design`.

## 4. Riesgos

| Riesgo | Mitigación |
|:-------|:-----------|
| 409 del convert in-tx revienta un checkout legítimo (claim vencido por el sweep) | Mensaje con el código del pedido; el pedido sigue OPEN y se re-abre desde /pedidos. Alternativa descartada (convert laxo desde OPEN) debilitaría la defensa anti doble-caja. |
| Deploy-skew FE nuevo / BE viejo | El parámetro extra se ignora en BE viejo; la FE conserva el convert post-hoc idempotente. |
| `EXPIRED` masivo si el TTL es muy corto | 72 h + sweep solo al listar (costo O(1 filas afectadas)); constante visible para ajuste. |
| Merge marcado CONVERTED antes del cobro | `converted_sale_id` apunta a la pendiente que ya contiene los ítems; si el cobro falla, la pendiente sigue y el flujo de cobro es el de siempre. |
