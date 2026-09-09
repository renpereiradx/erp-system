# Auditoría del flujo de pedidos de mostrador — registro de fixes (2026-09-09)

> Plan madre: `PLAN_PEDIDOS_MOSTRADOR_VENDEDOR_CAJA.md` (FASES 1-4 implementadas).
> Auditoría E2E previa (sesión 2026-09-09) encontró 3 críticos + 3 altos; este
> documento registra los arreglos implementados y verificados, y lo que queda
> pendiente de la lista media/baja.

## Implementado (críticos + altos + 1 medio)

### C1 — `Create` duplicaba pedidos si fallaba la resolución post-guardado ✅

El loop de reintento por colisión de código (`for attempt < 3 && detail == nil`)
no distinguía por qué `detail` quedó nil: si `repo.Create` succeeded pero el
`Get` posterior fallaba (p. ej. producto sin precio → 404 del resolver),
reintentaba **creando otro pedido**. Repro E2E: 1 llamada → 3 pedidos huérfanos
OPEN + 404 al cliente.

**Fix** (`internal/sale/counter_order_service.go`): el loop reintenta SOLO ante
`isCounterOrderCodeCollision`. Con create exitoso se llama `Get` una vez y se
retorna (degradado si hace falta, ver C2). Test:
`TestCounterOrderCreate_GetFailureAfterCreateDoesNotDuplicate`.

### C2 — Fail-loud de resolve-on-read tumbaba la bandeja completa ✅

`ResolveLine` devolvía 404 sin precio vigente y 500 si fallaba
`liquidate_sale_line_tax`; `resolveDetail` lo propagaba y `List`/`Get` fallaban
enteros (el vendor ni siquiera podía cancelar el pedido roto desde la UI).

**Fix** (degradación por ítem): `resolveDetail` ya no falla por resolución de
línea. Cada ítem degrada con `price_warning` / `tax_warning` (texto
informativo del backend), montos en cero y **excluido del total**; el nombre
de producto irrecuperable degrada al id. Los outcomes se cachean por line-key
(el fallo se resuelve una vez, no una por pedido). El precio real sigue
siendo hard-gate en checkout (`process_sale_with_reserve`). El stock degradado
devuelve `stock_available=nil` sin warning falso.

- Tipos: `ResolvedCounterOrderItem.PriceWarning/TaxWarning` (`counter_order.go`,
  `types.ts`).
- FE: `OrderDetailModal` renderiza la advertencia por ítem y cambia la etiqueta
  del total a "excluye ítems con advertencia" cuando hay líneas degradadas.
- Tests BE: `TestCounterOrderGet_UnpricedItemDegradesWithWarning`,
  `TestCounterOrderGet_TaxFailureDegradesWithWarning`,
  `TestCounterOrderList_DegradedItemDoesNotFailListing`.
- Tests FE: `OrderDetailModal.test.tsx` (3 casos).

### C3 — CTA "Procesar en caja" / "Dejar pendiente" dead-end en el confirm ✅

`resumePreload` abría el wizard sin armar `pendingSaleData`; `onConfirmWizard`
y `onLeavePendingWizard` exigían `pendingSaleData` → toast "No hay datos de
venta para procesar" y, al cerrar, el cleanup liberaba el claim y removía los
ítems (flujo sin salida).

**Fix** (`SalesNew.tsx`): la condición pasa a `pendingSaleData || items.length > 0`
en ambos callbacks — `buildNewSaleData()` reconstruye el payload desde el
carrito vigente (mismo criterio que el path manual). Guard `if (!salePayload)
return` para el caso de regla de reservas (ya informado por el builder).
`items` agregado a las deps de `onLeavePendingWizard` (stale-closure latente
que ya existía).

### A1 — El vendor perdió el botón "Solicitar Anulación" ✅

FASE 4 revocó `sales:write` a VNDR01 y el backend re-gateó POST
`/sale/{id}/cancellation-requests` a `sales:read`; el FE seguía con
`canRequestCancellation = canWrite`.

**Fix**: `canRequestCancellation = hasPermission('sales:read')` (`SalesNew.tsx`),
alineado con `salehttp.RegisterRoutes` (compat root con `sales:read`).

### A2 — `stock_available` usaba una fila de sucursal arbitraria ✅

`stockFor` pasaba `branchID = nil` al StockReader; con stock por sucursal,
`queryRow` devolvía una fila indeterminada mientras la venta valida hard contra
la sucursal que cobra.

**Fix**: `stockFor` recibe `rec.Order.BranchID` y lo pasa al lector. Los mocks
de test fueron actualizados (`expectResolvedDetail` espera `&branch`);
`TestCounterOrderGet_StockWarning` assertion explícita de `*b == branch`.

### A3 — El toggle "Ver todas las sucursales" era un no-op ✅

Omitir `branch_id` no lograba nada: `ParseOptionalBranchID` cae al
`ActiveBranch` del JWT y el listado filtraba igual. Permiso de gate:
**`branches:switch`** (perm 188, FASE B.1 multi-sucursal — ya era el que usaba
el toggle FE y el `BranchSwitcher`; no `branches:write` como sugirió la
auditoría a priori).

**Fix**:
- `CounterOrderFilter.AllBranches` (nuevo campo). El handler parsea
  `all_branches=1`; el service lo honra SOLO con `branches:switch` (admin por
  rol, `PermissionChecker` por puerto con el mismo cache Redis del middleware
  — wiring `CounterOrderService()` + `SetPermissionChecker`). Sin el permiso
  el scope queda en la sucursal activa, silencioso (sin 403 por lista FE
  stale).
- FE `useCounterOrders`: con `allBranches` envía `all_branches: 1` y omite
  `branch_id`.
- Tests: `TestCounterOrderList_AllBranchesGate` (4 subtests) + 2 tests en
  `CounterOrdersPage.test.tsx`.

### Medio — Precisión de cantidad ✅

`quantity: 0.004` → `numeric(10,2)` redondeaba a 0.00 → CHECK violation → 500.
`validateItems` ahora rechaza >2 decimales con 400 informativo ("admite hasta
2 decimales"). Caso agregado a `TestCounterOrderCreate_Table`.

### C4 (nuevo, hallado en la re-verificación E2E) — Todo cancel por API devolvía 500 ✅

`MarkCancelled` pasaba `userID` como `$2` pero el SQL solo referenciaba `$1` y
`$3` → Postgres no puede inferir el tipo de un parámetro sin referenciar
("could not determine data type of parameter $2"). `POST /counter-orders/{id}/cancel`
fallaba 500 **para todos los usuarios** (los tests unitarios no lo vieron:
mockean el repo). La tabla no tiene `cancelled_by`, así que el fix elimina el
parámetro huérfano (`counter_order_repo.go`). Lección: las mutaciones guardadas
del repo no tienen cobertura de integración — el E2E por API es el único gate
que las ejercita.

## Re-verificación E2E (binario arreglado en :5051, sin tocar los dev :3001/:5050)

- Create con producto sin precio → **201 + `price_warning`** y EXACTAMENTE 1
  pedido nuevo (antes: 404 + 3 huérfanos). C1+C2 verificados en vivo.
- Bandeja y detalle del pedido degradado → 200 con el ítem avisado y total 0.
- Cancel del pedido degradado → 200 tras C4 (antes imposible, 500).
- `all_branches=1` sin `branches:switch` → 200 con scope en sucursal activa.
- Cantidad `0.004` → 400 "admite hasta 2 decimales".
- Datos de prueba eliminados (pedido + producto); DB dev quedó en su estado
  previo (4 pedidos activos).

## Gates

- Backend: `go build ./...` ✅ · `go test ./...` ✅ (sale: 40+ casos) ·
  `golangci-lint run` ✅
- Frontend: `npx tsc --noEmit` ✅ · `npx vitest --run` 610/610 ✅ ·
  `pnpm build` ✅ · `pnpm lint:design` ✅ ("código nuevo limpio")

## Pendiente (medios/bajos de la auditoría, sin cambiar)

- N+1 en el listado (3 queries por línea distinta por pedido).
- `variant_id` crudo en el detalle (joinear `product_variants` para el nombre).
- Merge a pendiente: el wizard no muestra el saldo previo de la venta
  (decisión "cobrar solo lo nuevo" sin documentar en UI).
- `useCounterOrderCheckout` retorna objeto no memoizado (rerender-memo).
- `convert`/`release` fuera de react-query (no invalidan bandeja hasta
  staleTime).
- Claim disponible para cualquier rol con `counterorders:write` por API
  (evaluar gate `cash:write`).
- Refresh mid-wizard deja el claim sin release (cubierto por sweep de 20 min).
- Estado a medias del carrito tras cerrar wizard en modo merge.

## Notas de despliegue

- Sin migraciones nuevas.
- Requiere **rebuild + restart del backend** para tomar efecto (los servidores
  dev :3001/:5050 siguen con el binario previo — el fix del cancel C4 en
  especial: hoy cancelar un pedido por API/UI da 500 con el binario viejo).
- Los roles ya granteados no cambian; `branches:switch` (188) es quien habilita
  "ver todas" — hoy solo admin + DEPO01.
