# PLAN — Fix bugs de flujo Venta → Cobro → Reserva

**Fecha**: 2026-08-15
**Estado**: 🔄 En curso (investigación + fixes por agentes paralelos)
**Contexto**: Bug reportado en el flujo reservas → venta → cobro (feature walk-in checkout, WIP en erp-webapp) + bug en cobros de ventas pendientes.

---

## Bugs reportados

| # | Bug | Repo(s) | Severidad |
|:-:|:----|:--------|:----------|
| 1 | Al confirmar el cobro al final del wizard de ventas, la venta queda **PENDING** en el registro de ventas (el cobro no se registra) | BE + FE | 🔴 runtime |
| 2 | En el flujo reserva → facturar → ventas, con la reserva ya cargada en el carrito, el wizard **sigue mostrando la pestaña "Reservas"** aunque el backend solo permite una reserva por venta | FE | 🟠 UX/consistencia |
| 3 | En Cobros de ventas, al procesar el pago de una venta pendiente: **`(intermediate value).confirmSalePayment is not a function`** | FE | 🔴 runtime |

## Pregunta de investigación

- **¿Por qué el backend limita a UNA reserva por venta?** Rastrear el diseño (SQL `process_sale_with_reserve` con un único `p_reserve_id`, `SaleService.ProcessSale`/`resolveReserveID`, `ValidateProductsForSale`) y documentar la decisión. NO expandir a multi-reserva sin ADR.

## Hallazgos iniciales (mapa de código)

### Bug 3 — `confirmSalePayment` inexistente
- `erp-webapp/src/pages/SalePayment.jsx:634` llama `salePaymentService.confirmSalePayment(sales_order_id, confirmPayload)`.
- `src/services/salePaymentService.ts` **NO tiene** ese método.
- El API client SÍ tiene `confirmPayment(saleId, data)` → `PUT /sale/{id}/confirm-payment` (`BusinessManagementAPI.ts:506`).
- Backend: ruta registrada `sale.MakeConfirmPaymentHandler` (`internal/sale/http/routes.go:62`, `http.go:298`) — **pero el handler solo lee `payment_reference`/`payment_notes`**; ignora `payment_methods` y `caja_id` que envía `SalePayment.jsx:622-632`. Verificar qué hace `svc.ConfirmPayment` realmente (¿registra pago con método/monto o solo cambia estado?).

### Bug 2 — Pestaña Reservas con reserva ya cargada
- `src/features/sales/components/SaleCheckoutWizard.tsx:129-135` (WIP walk-in):
  ```ts
  const list: StepId[] = ['client']
  if (activeSales.length > 0) list.push('pending')
  if (pendingReservations.length > 0 || !!client) list.push('reservations')  // ← `|| !!client` nuevo en WIP
  list.push('payment', 'collection')
  ```
- El flujo "Facturar/Cobrar ahora" desde Reservas navega a `/ventas` con `location.state { reserve_id, client_id, product_id }` → `SalesNew.handleLoadSpecificReservation` agrega el ítem con `reserve_id` al carrito (`SalesNew.tsx:347-419`).
- El wizard recibe `items` (carrito). **Fix esperado**: no mostrar el paso 'reservations' cuando `items.some(i => i.reserve_id)` (una reserva por venta → no hay nada que agregar). Mantener walk-in disponible cuando NO hay reserva cargada.
- Guard de walk-in ya existe en `SalesNew.handleRegisterWalkIn` (`items.some(i => i.reserve_id)` → toast), pero la pestaña no debería siquiera aparecer.

### Bug 1 — Venta queda PENDING tras cobrar en wizard
- FE: `onConfirmWizard` (`SalesNew.tsx:1303-1386`) → `salePaymentService.posCheckout({ sale, payment })` → `POST /sale/pos-checkout`. **El FE ignora `result.success` y `result.payment_error`** — si el backend responde `success:false` sin lanzar error HTTP, el FE muestra toast de éxito igual (`Venta #... cobrada exitosamente`).
- BE: `POSService.Checkout` (`internal/sale/pos_service.go:66-117`): `ProcessSale` + `ProcessSalePaymentWithRegister` en una sola tx. Si pago falla → rollback total (no habría venta). Como la venta SÍ aparece PENDING, hipótesis principal: **el INSERT de la venta la crea con status PENDING y el pago (o su leg de actualización de estado) no la pasa a PAID**, o el pago se registra sin actualizar `sales_orders.status`.
- SQL: `transactions.process_sale_with_reserve` (`database/migrations/20260613000012_*.up.sql:104+`): INSERT `sales_orders` (~línea 239) — verificar con qué status inserta; exige reserva `status='CONFIRMED'` (líneas 181, 468); marca reserva COMPLETED al final (línea 493).
- Verificar claim del WIP FE: "el backend auto-confirma la reserva dentro del pos-checkout" (`ReservationDetail.tsx` WIP: permite "Cobrar ahora" desde estado RESERVED) — el SQL visto exige CONFIRMED, el claim parece **falso**. Si es falso, es un mismatch de contrato a reportar (no romper el WIP, reportar + sugerir).

## División de trabajo (2 agentes paralelos)

- **Agente Backend** (`business_management/`): rastrear Bug 1 end-to-end contra SQL real (verificar contra DB viva `erp_db` vía `pg_proc` si es posible), investigar y documentar el porqué del límite 1-reserva-por-venta, definir contrato correcto de `PUT /sale/{id}/confirm-payment` para cobrar una venta pendiente (Bug 3 BE-side), implementar fixes.
- **Agente Frontend** (`erp-webapp/`): arreglar Bug 3 (método `confirmSalePayment` en el service, manejando la respuesta real), Bug 2 (ocultar pestaña Reservas con reserva cargada), Bug 1 FE-side (leer `success`/`payment_error` del posCheckout y no mentir con el toast; validar payload de pago completo).

## Gates (ambos repos)

- BE: `go build ./...`, `go test ./...` (excluir `tests/integration`), `gofmt -l .`, `golangci-lint run`. Migraciones solo vía `make migrate-create` + `migrate-up` + `migrate-verify-gaps`. NUNCA modificar migraciones aplicadas.
- FE: `pnpm exec tsc --noEmit` (0 errores en archivos tocados), `pnpm build`, `pnpm test` (baseline: ~22 files / ~60 fallos pre-existentes — comparar, no romper). pnpm exclusivo; strings UI vía `src/lib/i18n.js`.
- Ambos: **NO commitear**. Dejar cambios en working tree y reportar diffs. El usuario aprueba por fase.
- FE tiene WIP sin commitear (feature walk-in) — trabajar ENCIMA, no stash/dropear.

## Decisiones registradas

| ID | Fecha | Decisión |
|:---|:------|:---------|
| D-Fx-1 | 2026-08-15 | Bug 1: el `success:false` del SQL de pago se trata como ERROR tipado en `processPayment` (`checkPaymentEnvelope`) → el pos-checkout hace rollback en vez de commitear una venta PENDING sin pago. |
| D-Fx-2 | 2026-08-15 | Bug 3: `PUT /sale/{id}/confirm-payment` ahora implementa el cobro real (un pago por entrada de `payment_methods`, resolución de método por code/name/id, atómico vía txm, venta queda PAID/PARTIAL_PAYMENT). Se elimina el stub `ConfirmPayment` de SaleService. |
| D-Fx-3 | 2026-08-15 | Bug 2: paso 'reservations' suprimido cuando el carrito ya tiene un ítem con `reserve_id` (regla en `checkoutSteps.ts`, test unitario). |
| D-Fx-4 | 2026-08-15 | FE no festeja cobros fallidos: `onConfirmWizard` y `handlePaymentSubmit` chequean `success===false`/`payment_error` y no resetean carrito ni recargan la lista en fallo. |
| D-Fx-5 | 2026-08-15 | Límite una-reserva-por-venta: regla de negocio deliberada reforzada por el SQL (`p_reserve_id` único, consumo CONFIRMED→COMPLETED). Multi-reserva = ADR + rediseño; NO implementado. |
| D-Fx-6 | 2026-08-15 | ✅ DECIDIDO por el usuario: el FE **confirma la reserva (CONFIRM) antes de navegar** a /ventas cuando está RESERVED ("Cobrar ahora"). Sin cambios de backend. Implementado en `ReservationDetail.tsx` (`handleInvoice` async: `await onConfirm(...)` si `isReserved`, luego `onInvoice`; botón con estado de procesamiento). |
| D-Fx-7 | 2026-08-15 | ✅ DECIDIDO por el usuario: **la caja NO es obligatoria** para cobrar. (a) BE: política `CashRegisterExplicit` en pos-checkout y confirm-payment — solo se usa la caja enviada explícitamente (validada por SQL: abierta + misma sucursal); sin caja → pago sin caja, JAMÁS auto-resolver la caja activa del operador. (b) FE: el select de caja guía — solo seleccionables las cajas abiertas de la sucursal actual (las de otras sucursales se listan deshabilitadas), hint informativo al cobrar sin caja, aviso ámbar cuando no hay cajas de la sucursal, y la caja elegida ahora SÍ viaja al backend (`cash_register_id`). Errores de pago mapeados a guías accionables (BRANCH_MISMATCH, CASH_REGISTER_NOT_OPEN). Commits: BE `7dc9076`, FE `db02664`. |
| D-Fx-8 | 2026-08-15 | ✅ Fixes de coherencia del wizard: (a) Pendientes desmarcables (click en la seleccionada → -1); "Nueva venta" limpia ítems de la pendiente continuada + merge state y AVANZA (antes solo limpiaba y parecía muerta). (b) Reservas: radio — UNA reserva por venta en ReservationsStep + onToggleReservation (antes solo bloqueaba mismo producto). (c) `blockedProductIds` solo bloquea canchas vía reserva (los ítems de pendientes continuadas ya no bloquean el select walk-in); bloqueadas al final, deshabilitadas, "(ya en el carrito)". (d) i18n sin ICU: plurales parentéticos (`{count} venta(s) pendiente(s)`) — el motor no soporta ICU y filtraba `{count, plural...}` literal. (e) BE: `amount_to_apply` del pos-checkout se CAPEA en `min(recibido, total autoritativo)` — el exceso es vuelto; elimina el error AMOUNT_EXCEEDS_BALANCE por totales locales divergentes (p. ej. ítems de pendiente sin reserve_id). Commits: BE `078b4b8`, FE `46d0214` (incluye archivos del feature walk-in — inseparables de los fixes). |

## Resultado de agentes + verificación del orquestador — 2026-08-15

Ambos agentes completaron; hallazgos VERIFICADOS por el orquestador (no solo auto-reportes):

| Verificación | Resultado |
|:-------------|:----------|
| BE `go build ./...` | ✅ BUILD_OK |
| BE `go test` (excl. integration) + `-count=1` en sale/postgres | ✅ verde |
| BE `gofmt -l` archivos tocados | ✅ limpio |
| FE `pnpm exec tsc --noEmit` | ✅ 0 errores globales |
| FE `pnpm build` | ✅ OK |
| FE `pnpm test` post-cambios | 17 files/49 tests fallidos = baseline pre-cambios idéntico; +8 tests nuevos pasando (checkout-steps) — 0 regresiones |
| Causa raíz Bug 1 reproducida en DB viva | ✅ BRANCH_MISMATCH: caja abierta (rama 3) ≠ venta (rama 1) → SQL `success:false` → Go commiteaba igual |
| Claim auto-confirm RESERVED | ❌ FALSO — `process_sale_with_reserve` exige CONFIRMED (RAISE RESERVE_NOT_AVAILABLE) |

Nota de contrato: el backend ahora responde en confirm-payment el objeto completo `{success, sale_id, sale_status, total_paid, payments[]}`; el wrapper FE es forward-compatible. Edge case conocido (documentado, no bloqueante): `payment_methods` multi-entrada donde la primera cubre el total → la segunda pata devuelve SALE_ALREADY_PAID y revierte todo; el FE actual envía una sola entrada.

Pendiente de commit (aprobación del usuario, unidades lógicas):
1. BE `fix(sale): treat SQL payment success:false as error so POS checkout rolls back` + `feat(sale): implement confirm-payment for pending sales` (o dos commits).
2. FE `fix(sales): hide Reservations step when a reservation is already in cart` + `fix(sales): don't celebrate failed POS checkouts` + `fix(payments): add confirmSalePayment service method`.
3. Decisión D-Fx-6 (auto-confirm RESERVED) antes de mergear el WIP de ReservationDetail.

## Commits realizados — 2026-08-15 ✅

**Backend (main):**
- `a3a087e` fix(sale): treat SQL payment success:false as error so POS checkout rolls back
- `33e50be` feat(sale): implement confirm-payment endpoint for pending sales

**Frontend (feat/purchase-checkout-wizard):**
- `5499982` fix(sales): hide Reservations step when a reservation is already in the cart
- `11b9bf0` fix(sales): don't celebrate failed POS checkouts
- `fa315d8` fix(payments): add confirmSalePayment service method for pending sales
- `f4d991a` fix(reservations): confirm reservation before invoicing from RESERVED status

Cada commit verificado standalone (tree extraído + tsc/build/tests). Queda SIN commitear (correctamente): el WIP walk-in del usuario (DESIGN.md, walk-in wiring en wizard/SalesNew, WalkInReservationForm, slotAvailability, claves i18n walkIn, PLAN_WALKIN_CHECKOUT). Gates finales: FE tsc 0 errores + `pnpm build` OK; BE build+tests verdes.

## Verificación contra DB viva (orquestador, read-only) — 2026-08-15

Hechos confirmados con `pg_proc` en `erp_db` (la DB es la fuente de verdad):

1. **`transactions.process_sale_with_reserve`** recibe UN único `p_reserve_id integer` (firma verificada) → la limitación "una reserva por venta" es estructural del SQL, no solo del Go.
2. **No existe auto-confirmación**: la función `RAISE EXCEPTION 'RESERVE_NOT_AVAILABLE: ... Debe estar en estado CONFIRMED ...'` si la reserva no es CONFIRMED. El claim del WIP FE ("el backend auto-confirma la reserva dentro del pos-checkout") es **FALSO** — "Cobrar ahora" desde estado RESERVED fallará con 400. Mismatch de contrato a reconciliar.
3. **INSERT de la venta con `status = 'PENDING'`** y `total_amount` actualizado al final de la función; reserva → `COMPLETED` al final.
4. **`transactions.process_sale_partial_payment`** (SQL unificado de pago, vía `PaymentRepo.ProcessSalePaymentUnified` → `payment_repo.go:59`): devuelve **JSON `success:FALSE` sin excepción** para SALE_NOT_FOUND / BRANCH_SCOPE_VIOLATION / SALE_CANCELLED / SALE_ALREADY_PAID / INVALID_AMOUNT (cash_received ≤ 0) / AMOUNT_EXCEEDS_CASH / AMOUNT_EXCEEDS_BALANCE / CASH_REGISTER_NOT_OPEN / BRANCH_MISMATCH. Si paga completo → `UPDATE sales_orders SET status='PAID'`; parcial → `PARTIAL_PAYMENT`. (Ojo: el FE espera `'PARTIAL'`, backend setea `'PARTIAL_PAYMENT'` — mismatch menor de status a revisar en SalePayment.jsx.)
5. **Causa raíz Bug 1 (backend)**: `POSService.Checkout` (`internal/sale/pos_service.go:91-116`) NO revisa `paymentResult["success"]` — ante un `success:false` del SQL (ej. `amount_received=0` por carrera del CollectionStep con el shortcut F12, caja cerrada, etc.) la tx **commitea igual** la venta PENDING sin pago y responde `Success: true` hardcodeado. El FE además ignora `result.success`/`payment_error` y muestra toast de éxito → el operador ve la venta pendiente "sin cobro". Fix: validar el resultado del pago en Checkout y devolver error (rollback), y corregir el `Success` hardcodeado.
6. Existen endpoints de pago completos ya: `POST /payment/process` (`ProcessPayment`, política caja opcional) y `POST /payment/process-partial` — candidatos naturales para "cobrar venta pendiente" desde Cobros de ventas (Bug 3), en vez de extender `PUT /sale/{id}/confirm-payment`.
