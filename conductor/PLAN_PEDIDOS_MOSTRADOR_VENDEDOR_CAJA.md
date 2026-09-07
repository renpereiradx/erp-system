# PLAN: Pedidos de Mostrador — Carrito del Vendedor → Cobro en Caja

**Fecha:** 2026-09-07
**Estado:** 📝 PLAN (pendiente de implementación)
**Alcance:** Cross-repo (backend `business_management` + frontend `erp-webapp`) → este plan vive en `conductor/` raíz
**Origen:** Pedido del owner: el rol Vendor necesita una UI para (1) cargar productos en un carrito, (2) asociar un cliente y guardarlo. El cliente pasa por caja, el cajero busca su nombre/orden y procesa el carrito — como venta nueva o agregándolo a una venta pendiente. "Con esto, el vendedor es solo vendedor."
**Verificación:** Todas las afirmaciones de §2 fueron verificadas contra el código el 2026-09-07 (exploración de ambos repos con evidencia file:line). Los 3 hallazgos de integración de presupuestos de `HALLAZGOS_PRESUPUESTOS_BACKEND_2026-08-17.md` (H1 aridad SQL, H2 URLs, H3 shape) **ya están corregidos** — el módulo presupuestos funciona hoy con contrato canónico v2.0.

---

## 1. Problema y objetivo

### 1.1 Flujo de facto hoy (y por qué no alcanza)

Hoy un vendedor "deja un carrito" creando una venta en estado `PENDING` desde el POS (`/ventas` → "Dejar pendiente"). Eso tiene tres problemas:

1. **La venta PENDING descuenta stock al crearse** (`transactions.process_sale_with_reserve` hace el check + descuento dentro de la tx): un carrito que quizá nunca se cobre compromete inventario desde el minuto cero.
2. **El vendedor puede cobrar**: tiene `sales:write` y `POST /sale/pos-checkout` solo exige `sales:write` (`internal/sale/http/routes.go:51-53`). El addendum v2 de `PLAN_VENDOR_ROLE_SUCURSALES_TERMINALES.md` lo documenta: "el checkout del vendedor usa pos-checkout sin caja". contradice "el vendedor es solo vendedor".
3. **Los borradores contaminan la lista de pendientes**: el paso de pendientes del wizard muestra todas las `PENDING` del cliente, sin distinguir "venta real con pago pendiente" de "borrador del vendedor".

### 1.2 Flujo objetivo

```
VENDEDOR (VNDR01)                          CAJERO (CAJA01/ENCR01)
───────────────────                        ────────────────────────
1. /pedidos → arma carrito                 3. Cliente llega a caja
   (busca producto, cantidades,              → /ventas → wizard → busca
    variante, código de barras)                cliente por nombre
2. Asocia cliente (o alta rápida             → el paso de pendientes muestra
    QuickClientModal) + nota                    "Pedidos del vendedor" + pendientes
   → GUARDAR (estado OPEN)                  4. Elige pedido (claim OPEN→CLAIMED)
   → imprime/da código PED-XXXXXX              → ítems cargados con precios vigentes
                                                → destino: venta nueva o pendiente #N
                                              5. Wizard pago+cobro (existente)
                                              6. pos-checkout / addProductsToSale
                                                 → marca CONVERTED + sale_id
```

El carrito **no toca stock ni pagos**: es un pedido pre-venta. El stock se compromete recién al procesar la venta (validación hard ya existente). El vendedor jamás cobra.

> **Clarificación del owner (2026-09-07):** el cajero no requiere cambio alguno — usa `/ventas`
> directamente con el flujo de siempre, y que una venta quede pendiente es decisión exclusiva del
> cajero. El vendedor solo registra la pre-venta (análoga a un presupuesto). El plan ya está alineado
> con esto: FASE 3 no modifica el flujo de caja (solo agrega la sección de pedidos al paso de
> pendientes del wizard) y FASE 4 no restringe a CAJA01/ENCR01, que conservan `sales:write` +
> `cash:write` — pueden crear, cobrar y dejar pendientes igual que hoy.

## 2. Estado actual verificado (referencia)

### Backend (`business_management`)

- **Venta**: `POST /sale/` (crea, descuente stock) y `POST /sale/pos-checkout` (venta+pago+numeración fiscal SIFEN atómicos, `POSCheckoutRequest` en `internal/sale/pos_service.go:45-76`). Payload `ProcessSaleEnhancedRequest` (`models/sale.go:235-259`): `client_id` requerido, `product_details[]` (`product_id`, `variant_id?`, `quantity`, `unit?`, `sale_price?`, descuentos...), `allow_price_modifications`. Validaciones: `ValidateProductsForSale` (existencia, stock, unidades) y `enforcePriceModificationPolicy` (`sales:apply_discount` → 403 si descuentos/precio custom) en `internal/sale/sale_service.go`.
- **Pendientes**: `sales_orders.status ∈ {PAID, PENDING, PARTIAL_PAYMENT, CANCELLED}`. `GET /sale/client_id/{id}/pending` lista; `POST /sale/{id}/products` agrega ítems a una venta existente (descuenta stock del delta); `PUT /sale/{id}/confirm-payment` cobra multi-pago. Edición de ítems rechazada para PAID/CANCELLED.
- **Presupuestos** (`transactions.budget_orders`): cotización con cliente+ítems+estados (`PENDING|APPROVED|REJECTED|EXPIRED|CONVERTED|CANCELLED`), `convert_budget_to_sale` (convierte directo a venta, **borde el wizard de caja**), sin edición incremental de ítems, **sin columna `branch_id`**, sin semántica de claim. Contrato FE canónico v2.0 OK.
- **RBAC**: permisos en DB; último ID usado **188** (`20260905122845`). Patrón de subrouter: `RequireModulePermission(s, readPerm, writePerm)` + registro path-prefix en `config/module_permissions.go:37` (presupuestos: `"/budgets"`). Excepciones por ruta: `RequireModulePermissionOnPath` (`middleware/module_permission.go:92`). VNDR01 v2 = 12 perms (`sales:read/write`, `budgets:*`, `clients:*`, `products:read`, `payments:read`, `tax:read`, `branches:read`, `dashboard:read`, `settings:read`) — **sin** `cash:*`, `sales:apply_discount`, `sales:cancel`. **Lección aprendida** (post-auditoría 2026-09-07): los grants van **por nombre**, no por ID (topologías divergen); gates de plan incluyen `make migrate-up`.
- **Clientes**: `/api/v1/clients` (`clients:read/write`, VNDR01 lo tiene) — list con `?search=`. Búsqueda por nombre de venta: `GET /sale/client_name/{name}`.
- **Sucursales**: `X-Branch-ID` + `ResolveBranchContext`; `DeviceBranchMiddleware` fuerza la sucursal de la terminal para usuarios sin `branches:switch`. `sales_orders.branch_id` existe por trigger.

### Frontend (`erp-webapp`)

- **POS/wizard**: `/ventas` (`src/pages/SalesNew.tsx`, 2025 líneas) con tabs `new-sale|history|cancellations`. Wizard `SaleCheckoutWizard` (`src/features/sales/`): pasos `client → (pending) → (reservations) → payment → collection` (`checkoutSteps.ts:35-47`). Carrito = `useState<CartItem[]>` local en SalesNew (agrupa por producto+variante). Merge a pendiente: `handleContinueSale` carga la PENDING como CartItems (`isFromPendingSale`+`detailId`, setea `currentSaleId`); ítems nuevos van por `saleService.addProductsToSale`; confirmación por `posCheckout` (venta nueva) o `addProductsToSale`+`confirmSalePayment` (merge) (`SalesNew.tsx:1363`).
- **Selección de cliente**: `ClientStep` (`SearchableDropdown` + `useClientStore.searchClients`, min 3 chars) + `QuickClientModal` (alta rápida, `clients:write`). Al elegir cliente se buscan sus pendientes y se avisa "tiene pendientes".
- **Presupuestos**: páginas legacy `src/pages/Budget{Management,Create,Detail}.tsx` en `/comercial/presupuestos` (sin PermissionGuard hoy).
- **Referencia feature-sliced**: `src/features/transfers/` (components + hooks react-query + types + `__tests__`), patrón a seguir.
- **i18n**: módulos `src/lib/i18n/locales/es/<feature>.js` con spread en `index.js`; `en` hereda de `es`. Patrón `t('key', 'fallback español')`.
- **Gating**: `hasPermission` (AuthContext) + `hasStoredPermission` (`src/utils/userPermissions.ts`) para lógica fuera de React.

## 3. Decisiones de diseño

### 3.1 Entidad nueva `counter_orders` (NO reutilizar presupuestos)

| Criterio | Reusar `budget_orders` | Nueva `counter_orders` ✅ |
|:--|:--|:--|
| Semántica | Cotización comercial (validez, aprobación APPROVED→CONVERTED) | Pedido operativo de mostrador (ciclo carrito nativo) |
| Claim anti-doble-caja | No existe (habría que inventar estados) | `OPEN→CLAIMED` con `claimed_by/at` desde el día 1 |
| Edición de ítems | Solo create one-shot (no hay update de ítems) | Replace-all de ítems mientras OPEN |
| Sucursal | No hay columna (solo filtro) | `branch_id` columna |
| Conversión | `convert_budget_to_sale` crea la venta directo, salteando el wizard de caja (pagos, pendientes) | La caja procesa vía wizard existente; el pedido se marca CONVERTED con `sale_id` |
| Costo | Adaptar estados+endpoints+tabla ≈ mismo trabajo, con sobrecarga semántica | Migración limpia + contexto nuevo |

Los presupuestos siguen existiendo para cotizaciones (reuso de piezas: `clientService.searchByName`, `productService.search`, `calculateSaleTotals`). **Dónde vive**: archivos `counter_order_*.go` dentro de `internal/sale` siguiendo el patrón de budgets (`budget_service.go` + `http_budget.go` + repo en `internal/platform/postgres/`), con puertos propios. No un contexto aparte: es un artefacto pre-venta del dominio sale (coherente con FASE 8 futura).

### 3.2 Ciclo de vida y concurrencia

```
OPEN ──claim──▶ CLAIMED ──convert(sale_id)──▶ CONVERTED (final, guarda venta)
  ▲                │
  └────release─────┘
OPEN/CLAIMED ──cancel(motivo)──▶ CANCELLED (final)
```

- **Claim**: `UPDATE ... SET status='CLAIMED', claimed_by=$user, claimed_at=now() WHERE id=$id AND status='OPEN'` → si 0 filas, 409 con el estado actual (dos cajas no procesan el mismo pedido).
- **Auto-release de claims huérfanos**: antes de listar/leer, sweep barato `status='CLAIMED' AND claimed_at < now()-interval '20 minutes'` → vuelve a OPEN. Sin cron.
- **Convert**: idempotente por `sale_id`; `WHERE status='CLAIMED'`; el servicio verifica que la venta existe (puerto sale reader). Si la caja muere entre checkout y convert, el pedido queda CLAIMED (visible) y se recupera con re-intento o release.
- **Edición/cancelación**: solo mientras OPEN y sin claim (edición); cancelación con motivo obligatorio (auditoría).

### 3.3 Precios y stock: resolver al leer, no snapshot

- El detalle **no persiste precios**: `GET` de lista/detalle resuelve precio vigente + IVA + subtotal por ítem (misma resolución de precios que usa la venta) y devuelve `stock_available` + `stock_warning`. Totales siempre frescos, cero divergencia stale-price entre mostrador y caja.
- El precio definitivo lo fija `ProcessSale`/`pos-checkout` al procesar (con el gate de `sales:apply_discount` que el vendedor no tiene). El stock se valida hard al procesar (comportamiento ya existente); el warning en el detalle es informativo para que el cajero lo charle con el cliente antes.

### 3.4 Sucursal

- `branch_id` se estampa al crear (sucursal activa del vendedor vía `ResolveBranchContext`, mismo mecanismo que budgets). La bandeja de la caja filtra por defecto a su sucursal activa; toggle "ver todas" para `branches:switch`/admin. Pedido cross-sucursal: puede procesarse en otra sucursal (el stock de la venta es el de la sucursal que cobra).

### 3.5 Permisos y roles

- **Permisos nuevos**: `counterorders:read` (**189**), `counterorders:write` (**190**) — son perms de path-prefix → sí se registran en `config/module_permissions.go` (`"/counter-orders"`).
- **Grants por nombre** (lección de topologías): VNDR01, CAJA01, ENCR01 reciben ambos; F2VLso por consistencia; DEPO01 no.
- **"Vendedor solo vendedor" (v3, FASE 4)**: revocar `sales:write` a VNDR01 y blindar server-side el cobro:
  - `POST /sale/pos-checkout` y `PUT /sale/{id}/confirm-payment` pasan a exigir además `cash:write` por ruta (`RequireModulePermissionOnPath`; hoy solo `sales:write`, `routes.go:53,64`). Cobrar = manejar dinero.
  - `POST /sale/{id}/cancellation-requests` (FASE C, hoy `sales:write`) se re-gatea a `sales:read`: el vendedor conserva su "Solicitar Anulación" (la aprobación sigue siendo `sales:cancel`).
  - Con esto VNDR01 no puede: crear ventas (ni pendientes-borrador), cobrar. Sí ve historial/detalle acotado (`sales:read`, addendum v2 §A.2 intacto).
  - Verificación previa obligatoria: auditar qué otros roles tienen `sales:write` sin `cash:write` y usan `pos-checkout`/`confirm-payment` (esperado: solo VNDR01; CAJA01/ENCR01 tienen ambos, F2VLso bypasea).

### 3.6 API contract

Subrouter `/counter-orders` (`RequireModulePermission(Server, "counterorders:read", "counterorders:write")`, literales antes de `/{id}`):

| Método | Ruta | Descripción |
|:--|:--|:--|
| POST | `/counter-orders` | Crear `{client_id, items[{product_id, variant_id?, quantity, unit?, notes?}], notes?}` → detalle resuelto |
| GET | `/counter-orders` | Lista `?status=&client_id=&branch_id=&q=&page=&page_size=` (q = código o nombre de cliente); incluye totales resueltos y cuenta de ítems; ejecuta sweep de claims stale |
| GET | `/counter-orders/{id}` | Detalle con ítems resueltos (precio, IVA, subtotal, stock flag) |
| PUT | `/counter-orders/{id}` | Replace-all de ítems + notas (solo OPEN sin claim) |
| POST | `/counter-orders/{id}/claim` | OPEN→CLAIMED (409 con estado si no se puede); devuelve detalle para precargar wizard |
| POST | `/counter-orders/{id}/release` | CLAIMED→OPEN (quien clamó o admin) |
| POST | `/counter-orders/{id}/convert` | `{sale_id, mode: "new"\|"merge"}` → CONVERTED (idempotente por sale_id) |
| POST | `/counter-orders/{id}/cancel` | `{reason}` (obligatoria) → CANCELLED |

### 3.7 DDL (borrador — tipos copiados de `budget_orders`/`sales_order_details`)

```sql
CREATE TABLE transactions.counter_orders (
  id                 varchar(27) PRIMARY KEY,
  code               varchar(16) NOT NULL UNIQUE,        -- PED-XXXXXX (crypto/rand sin ambigüedad, patrón pairing_code)
  client_id          varchar(9)  NOT NULL REFERENCES users.parties(id),
  branch_id          integer     NOT NULL REFERENCES branches(id),
  status             varchar(12) NOT NULL CHECK (status IN ('OPEN','CLAIMED','CONVERTED','CANCELLED','EXPIRED')),
  notes              text,
  created_by         varchar(27) NOT NULL REFERENCES users.users(id),
  claimed_by         varchar(27) REFERENCES users.users(id),
  claimed_at         timestamptz,
  converted_sale_id  varchar(27) REFERENCES transactions.sales_orders(id),
  converted_at       timestamptz,
  cancelled_reason   text, cancelled_at timestamptz,
  metadata           jsonb NOT NULL DEFAULT '{}',
  created_at         timestamptz NOT NULL DEFAULT now(),
  updated_at         timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_counter_orders_status_branch ON transactions.counter_orders(status, branch_id);
CREATE INDEX idx_counter_orders_client ON transactions.counter_orders(client_id) WHERE status IN ('OPEN','CLAIMED');

CREATE TABLE transactions.counter_order_details (
  id                bigserial PRIMARY KEY,
  counter_order_id  varchar(27) NOT NULL REFERENCES transactions.counter_orders(id) ON DELETE CASCADE,
  product_id        varchar(...) NOT NULL REFERENCES products.products(id),
  variant_id        ...,
  quantity          numeric(10,2) NOT NULL CHECK (quantity > 0),
  unit              varchar(20) NOT NULL DEFAULT 'UND',
  notes             text
);
```

## 4. Mejoras propuestas al flujo (más allá de lo pedido)

1. **Código escaneable `PED-XXXXXX`** en todo pedido: el cajero puede buscar por código exacto (además de por nombre de cliente). Base para ticket/QR futuro (fase 5).
2. **Claim anti doble-caja**: dos cajeros no pueden procesar el mismo pedido a la vez (guard en UPDATE + 409 informativo); pedidos "en caja con X" se ven deshabilitados en el paso del wizard.
3. **Auto-release de claims huérfanos** (20 min, sweep al listar): una caja abandonada no bloquea el pedido para siempre.
4. **Precios siempre vigentes + warning de stock** en el detalle: el cajero ve divergencias ANTES de cobrar, no un error seco en el checkout.
5. **Nota del vendedor → cajero** (campo `notes` visible en el paso del wizard): "cliente pidió facturar a...", "espera descuento del encargado", etc.
6. **Destino combinado en un solo paso**: el paso de pendientes del wizard pasa a listar "Pedidos del vendedor" + "Ventas pendientes" del cliente, y un pedido puede procesarse como venta nueva **o** agregarse a un pendiente existente (merge ya soportado por `addProductsToSale`) — exactamente el flujo pedido, sin pantalla nueva para el cajero.
7. **CTA "Procesar en caja"** en la bandeja `/pedidos`: navega a `/ventas` con cliente+pedido pre-cargados (store de precarga), para el caso "el cajero ya sabe qué pedido es".
8. **Trazabilidad**: pedido CONVERTED enlaza a la venta (`/cobros-ventas/:id` — el vendedor ve el detalle acotado v2); quien creó, quién clamó y quién procesó quedan auditados.
9. **Desbloqueo del cuello de botella**: los pendientes dejan de mezclarse con borradores (§1.1.3) y el stock deja de comprometerse por carritos incobrados (§1.1.1).

## 5. Decisiones abiertas

| # | Pregunta | Recomendación | Bloquea |
|:-:|:---------|:--------------|:--------|
| 5.1 | ¿Bloquear el cobro del vendor revocando `sales:write` (v3) o con permiso nuevo `sales:charge`? | Revocar `sales:write` + gates `cash:write` por ruta (§3.5) — menos superficie; requiere re-gatear cancellation-requests a `sales:read`. La alternativa `sales:charge` solo si la auditoría pre-FASE 4 encuentra un rol que cobra legítimamente sin caja | FASE 4 |
| 5.2 | ¿Reservar stock al crear el pedido? | No (v1): warning informativo + validación hard al procesar. Reservas reales (módulo reserves) como fase 5 si la operación lo pide | Nada |
| 5.3 | ¿Auto-expiración de OPEN (TTL)? | No en v1 (cancelación manual; estados ya prevén EXPIRED). Job/TTL en fase 5 | Nada |
| 5.4 | ¿El vendedor ve pedidos de otros vendedores? | Sí (equipo de sucursal): lista completa con filtro por creador; editable solo si OPEN sin claim. Fomenta que cualquier compañero ajuste a pedido del cliente | Nada |
| 5.5 | ¿Ticket/QR imprimible del pedido? | Fase 5: requiere decidir permiso de impresión para vendor (hoy `documents:*` está revocado a VNDR01 — o FE imprime local, o se expone render con `counterorders:write`) | Fase 5 |

## 6. Fases

> Reglas AGENTS.md: migraciones SOLO via `make migrate-create`; handlers sin lógica de negocio; tests obligatorios (table-driven + mocks por puerto); i18n siempre; cambio de API backend → frontend en el mismo ciclo; pnpm exclusivo; skill `vercel-react-best-practices` con reporte en `conductor/`.

### FASE 1 — Backend: entidad + API (sola)

1. **Migración** (`make migrate-create`): tablas §3.7 + permisos 189/190 (`counterorders:read/write`) + grants por nombre (VNDR01, CAJA01, ENCR01, F2VLso) + `.down.sql` (revoke + drop). Verificar con `make migrate-up` y `make migrate-verify-gaps` (lección del 2026-09-07: `migrate-up` va en los gates).
2. **Dominio y puertos** (`internal/sale/counter_order.go` + port): tipos `CounterOrder`/`CounterOrderItem`/estados; puertos `CounterOrderRepository` (CRUD + claim/release/convert con guards SQL), `ClientReader` (existe/activo), `ProductReader` (existencia + precio vigente + IVA + stock por sucursal — reusar los que ya consume el servicio de venta).
3. **Servicio** (`counter_order_service.go`): create (valida cliente/productos/unidades/cantidad>0, genera código `PED-`), update (solo OPEN sin claim), claim (guard + sweep stale), release, convert (verifica venta + idempotente), cancel (motivo), list/detail con resolución de precios/IVA/stock. Branch por `ResolveBranchContext`. Tests table-driven con mocks por puerto (incl. caso 409 de claim).
4. **HTTP** (`http_counter_order.go` + subrouter en `internal/sale/http/routes.go`): rutas §3.6, handlers solo parseo. Registro en `config/module_permissions.go`. `wiring/sale.go` conecta dependencias.

**Gates:** `make build` + `go test ./...` + `golangci-lint run` + `make migrate-up` + `make migrate-verify-gaps` (en dev).

### FASE 2 — Frontend: feature del vendedor `/pedidos`

1. **Feature-sliced** `src/features/counterorders/` (patrón `transfers/`):
   - `components/OrdersBoard.tsx`: bandeja con filtros por estado (badge OPEN), búsqueda por cliente/código, sucursal activa + toggle "ver todas" (`branches:switch`), acciones por estado (editar/cancelar con motivo; "Procesar en caja" gated `sales:write` — hasta FASE 4, luego `cash:write`).
   - `components/OrderBuilder.tsx`: carrito del vendedor — búsqueda de productos (debounce, variantes, barcode vía `useBarcodeScanner`), cantidades/unidades, cliente (`useClientStore` + `QuickClientModal`), notas, totales **resueltos por el backend** (detalle preview), guardar. Guard con ítems sin salvar al salir. Si `PLAN_CATALOGO_VENDEDOR` ya está implementado, reusar sus tarjetas/picker (`src/features/catalog/`).
   - `components/OrderDetailModal.tsx` + `OrderStatusBadge.tsx`: detalle resuelto (precios, IVA, stock warnings) + link a venta si CONVERTED.
   - `hooks/useCounterOrders.ts` (react-query, invalidación `['counter-orders']`) y `hooks/useOrderCart.ts` (estado local: add/remove/qty, agrupación por producto+variante — espejo de `addProductToCart` de SalesNew).
   - `types.ts`: `CounterOrder`, `CounterOrderItem`, `COUNTER_ORDER_STATUSES`, DTOs resueltos.
2. **Servicio** `src/services/counterOrderService.ts` (apiClient, rutas §3.6).
3. **Ruta + nav**: `/pedidos` con `PermissionGuard permission="counterorders:read"`; nav "Gestión Comercial" → "Pedidos" (`navigation.ts`).
4. **i18n**: `locales/es/counterorders.js` (spread en `index.js`), keys `counterorders.*` con fallback español.
5. **Tests** (`__tests__/`, según `PLAN_TEST_DESIGN_FRONTEND.md`: mocks en la frontera del service, i18n mock, queries por rol/aria). **Auditoría** `vercel-react-best-practices` → reporte en `erp-webapp/conductor/`.

**Gates:** `npx vitest --run` 0 fallos + `npx tsc --noEmit` + `pnpm build` + `pnpm lint:design`.

### FASE 3 — Frontend: integración en el wizard de caja

1. **Paso del wizard**: extender `PendingSalesStep` (título "Pendientes y pedidos del cliente"): sección "Pedidos del vendedor" (código, ítems, total estimado, vendedor, hace cuánto, estado — CLAIMED por otro → deshabilitado "En caja con X") + sección "Ventas pendientes" (comportamiento actual). Al elegir pedido → destino inline: "Venta nueva" o "Agregar a pendiente #N" (dropdown). `checkoutSteps.ts` incluye el paso si hay pedidos OPEN o pendientes.
2. **Orquestación** en `src/features/counterorders/hooks/useCounterOrderCheckout.ts` (consumido por `SalesNew.tsx` — no engordar la página): claim al elegir, ítems → CartItems con flag `isFromCounterOrder` (junto al patrón `isFromPendingSale`), release al salir sin procesar, convert tras éxito de `posCheckout`/`addProductsToSale+confirmSalePayment` (con `sale_id` y mode). Error de convert → toast accionable "marcar como procesado" (reintento).
3. **Selección de cliente**: `handleSelectClient` también consulta pedidos OPEN del cliente (hook react-query) — aviso en `ClientStep` "Tiene N pedidos del vendedor" (patrón del aviso de pendientes existente).
4. **Precarga**: `src/store/useCounterOrderPreloadStore.ts` (zustand) — CTA "Procesar en caja" de la bandeja precarga cliente+pedido en `/ventas` (claim inmediato + apertura del wizard).
5. **Tests**: paso del wizard (selección, claim 409, merge a pendiente, convert tras checkout), store de precarga, gating. **Auditoría react** update.

**Gates:** ídem FASE 2.

### FASE 4 — "Vendedor solo vendedor" (v3, backend + frontend)

1. **Auditoría previa**: roles con `sales:write` sin `cash:write` que usan `pos-checkout`/`confirm-payment` (esperado: solo VNDR01).
2. **Backend**: `RequireModulePermissionOnPath` con `cash:write` en `POST /sale/pos-checkout` y `PUT /sale/{id}/confirm-payment`; re-gate `POST /sale/{id}/cancellation-requests` a `sales:read`; tests de middleware.
3. **Migración** (por nombre): revocar `sales:write` a VNDR01 (conserva `sales:read`). Documentar en el plan de migración que el flujo "vendedor deja PENDING" muere acá, reemplazado por pedidos.
4. **Frontend**: nav/UX del vendor — `/ventas` sin `sales:write` abre directo en Historial y oculta tab "Nueva venta" (gate `hasStoredPermission('sales:write')`); landing del vendor apunta a `/pedidos`; matriz de `docs/guides/frontend/FRONTEND_RBAC_GUIDE.md` actualizada. Este plan **supera** la frase del addendum v2 "el checkout del vendedor usa pos-checkout sin caja".
5. **Deploy**: migración + backend juntos; comunicar **re-login** (permisos en token/`/me`).

**Gates:** backend ídem FASE 1; frontend ídem FASE 2; además prueba manual E2E del flujo completo con usuarios vendedor/cajero.

### FASE 5 — Mejoras (post-v1, no calendarizadas)

Ticket/QR imprimible del pedido (decisión 5.5); marcaje CONVERTED **dentro de la tx** de `pos-checkout`/`addProductsToSale` (param `counter_order_id`, elimina el two-step); reservas de stock; auto-expiración TTL; métricas (pedidos creados/convertidos/tiempo medio mostrador→caja).

## 7. Riesgos

| Riesgo | Mitigación |
|:-------|:-----------|
| Doble procesamiento del mismo pedido | Claim con guard SQL (409) + auto-release 20 min + convert idempotente por sale_id |
| Caja muere entre checkout y convert | Pedido queda CLAIMED (visible); reintento/"marcar como procesado"; fase 5 lo lleva in-tx |
| Divergencia precio/stock entre mostrador y caja | Resolve-on-read (sin snapshot); warnings informativos; validación hard ya existente en ProcessSale |
| Re-gate `cash:write` en pos-checkout rompe a un rol legítimo | Auditoría previa obligatoria (FASE 4.1); alternativa permiso `sales:charge` (5.1) |
| Vendedor pierde "Solicitar Anulación" al revocar sales:write | Re-gate de cancellation-requests a `sales:read` en el MISMO deploy (FASE 4.2) |
| Crecimiento de `SalesNew.tsx` (2025 líneas, legacy) | Orquestación en `useCounterOrderCheckout` + componentes del feature; migración completa de la página es deuda del plan de Sales (no de este) |
| Topología de permisos divergente (IDs) | Grants por nombre + `migrate-up` en gates (lecciones 2026-09-07) |
| JWTs con permisos viejos | Re-login comunicado en deploy de FASE 4 (patrón anterior) |
| Rollback | Migraciones con `.down.sql`; FASE 4 reversible por datos (re-grant) |

## 8. Orden y dependencias

```
FASE 1 (backend, sola — desbloquea todo)
 └─> FASE 2 (FE vendor /pedidos)  ──> FASE 3 (FE caja wizard)  ──> FASE 4 (vendedor puro v3)
                                                                (2 y 3 pueden solaparse tras 1;
                                                                 4 requiere 2+3 para no dejar al vendor sin flujo)
FASE 5 (mejoras, on demand)
```

Orden de deploy de FASE 4 crítico: migración + re-gates backend en el mismo diff, frontend en el mismo ciclo. Nunca revocar `sales:write` antes de que `/pedidos` exista en producción.
