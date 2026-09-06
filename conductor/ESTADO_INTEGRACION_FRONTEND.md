# Estado de Integración — Frente FRONTEND

**Repo:** `erp-webapp/` (React 19 + Vite 6 + TypeScript, Feature-Sliced en curso)
**Fecha de inicio:** 2026-08-03 · **Última actualización:** 2026-08-05
**Auditoría base:** `../AUDITORIA_INTEGRACION_FRONTEND_BACKEND.md`
**Rama de trabajo:** `erp-webapp` repo → `main` (working tree limpio; el pase de sales está commiteado en `2005fd9` y posteriores)
**Criterio de priorización:** primero los 🔴 que rompen runtime cuando el FE se reintegre contra el
backend ya cerrado (FASE 0–13 ✅, ver `../conductor/ESTADO_INTEGRACION_BACKEND.md`). Nada de esto
requiere cambios backend: el contrato está estable y documentado.

---

## Resumen ejecutivo

El backend quedó **cerrado y commiteado** (B1–B5, suite 24 paquetes ok / 0 FAIL). Todos los 🔴 que
quedaban del inventario son **FE puro**: paths incorrectos, métodos de cliente inexistentes, fachadas
rotas y casing sin mapear. Este documento traduce los hallazgos de la auditoría (§2–§4) a tareas
ordenadas y las trackea.

**Regla heredada del frente backend (D-B3):** no implementar endpoints backend nuevos hasta
confirmar uso FE — ya verificado: los 13 endpoints "faltantes" son código muerto FE (tarea F5).

**Estado del frente frontend tras este pase:**

| Tarea | Estado | Impacto |
|:------|:-------|:--------|
| F1 — `processSalePayment` path `/sale/payment`→`/payment/process` | ✅ Corregido | Desbloquea `saleService.processPayment` y `salePaymentService.processPayment` (pago de venta) |
| F2 — `getPurchaseByIdAndSupplier` inexistente | ✅ Corregido | `getPurchaseOrderWithSupplierValidation` deja de tirar `TypeError`; `fetchPurchaseOrderById` ya no manda `supplier_name` vacío (400 backend) |
| F3 — Fachada `apiService` delegando a métodos inexistentes | ✅ Corregido | `getProducts`/`getBeachTennisCourts`/`searchProductsByName` apuntan a métodos reales |
| F4 — `brandService` casing camelCase | ✅ Corregido | `logoUrl`→`logo_url`, `isActive`→`is_active` (bidireccional); las marcas por fin se guardan |
| F5 — Dead code: 13 endpoints huérfanos + `usePurchasePaymentStore.js` + wrappers `useSaleStore.ts` | ✅ Completado | −1 store huérfano, −9 métodos, −5 wrappers, −8 endpoints muertos |
| F6 — `updateSale` cancela en vez de actualizar | ✅ **Decidido:** venta inmutable post-create → `updateSale` **eliminado** del FE (se une a F5) | El backend es cancel-only por diseño (ver §F6) |
| F7 — Estructural: 5 wrappers de lista, `currency_code`, tipos duplicados, JS→TS, `as any` | ✅ Completo (F7.1–F7.5 commiteadas; F7.6 resuelto por ADR `ADR-F7.6-DOBLE_API.md`) | Plan detallado en `PLAN_F7_CONTRATO_HTTP.md` |
| F8 — Pase de errores TypeScript pre-existentes | ✅ Completo (2026-08-05) | `tsc --noEmit` 484 → 0 (`227f256`→`99e6ee4`); plan en `PLAN_F8_ERRORES_TYPESCRIPT.md` |
| F7.7 — Contrato HTTP explícito (elimina doble-`/api` de raíz) | ✅ Ejecutado (2026-08-05, sin commitear) | URL directa al backend; proxy rewrite eliminado en Vite + Nginx; 11 puntos coordinados. Gate verde (tsc 0, build 0, tests 60=baseline). Smoke pendiente de backend levantado. Plan en `PLAN_F7.7_BASEURL_EXPLICITO.md` |

**Verificación:** `pnpm build` ✅ (sin errores TS) · `pnpm test` ✅ (suite unit en verde).

**Lo que quedó fuera de estos pases (y por qué):**
- **F6** se resolvió (2026-08-03): la venta es inmutable post-create → `updateSale` eliminado.
- **F5** (limpieza de código muerto) se completó (2026-08-03) con diff dedicado; su residual
  `currentSession`/`sessionActive` se purgó el 2026-08-05 (ver §F5).
- **F7.5** (JS→TS de servicios) se completó (2026-08-04): 0 `.js` en la raíz de `src/services/`.
- **Doble-`/api`** (F7.6, hallazgo D-B4): **eliminado de raíz por F7.7** (2026-08-05). El ADR
  F7.6 original (opción C: documentar + guardia) fue **reaperturado** cuando el proyecto pasó a
  estar fuera de producción, permitiendo asumir el riesgo del refactor. F7.7 ejecuta "URL directa
  al backend" (sin proxy rewrite en Vite ni Nginx), alineando dev/prod al modelo que cloud/demo ya
  usaban. Gate verde; smoke pendiente de backend. Ver `PLAN_F7.7_BASEURL_EXPLICITO.md` y
  `ADR-F7.6-DOBLE_API.md` (historial completo de la decisión).

---

## Tareas

### F1 — `processSalePayment`: path incorrecto `/sale/payment` → `/payment/process` 🔴

**Estado:** ✅ Corregido
**Tipo:** bug de contrato (rompe runtime el pago de venta)
**Refs:** `src/services/BusinessManagementAPI.ts:496-498`

**Problema:** `BusinessManagementAPI.processSalePayment` hacía `POST /sale/payment`, que no existe
en el backend. El endpoint real es `POST /payment/process` (`internal/sale/http/routes.go:77`).
Consumidores afectados:
- `saleService.processPayment` (`saleService.ts:190`)
- `salePaymentService.processPayment` (`salePaymentService.ts:107`, con retry + telemetry)

**Fix:** cambiar el path a `/payment/process` (coincide con `API_ENDPOINTS.SALE_PAYMENT_PROCESS` en
`src/types.ts:2048`). Un cambio de string, cero riesgo de contrato — es el endpoint que el backend ya
sirve y que los clientes correctos usan.

---

### F2 — `getPurchaseByIdAndSupplier`: método inexistente invocado desde el store 🔴

**Estado:** ✅ Corregido
**Tipo:** bug de runtime (`TypeError: apiClient.getPurchaseByIdAndSupplier is not a function`)
**Refs:** `src/services/purchaseService.ts:296`, `src/store/usePurchaseStore.ts:157,246-253`

**Problema:** `purchaseService.getPurchaseOrderWithSupplierValidation` llama a
`apiClient.getPurchaseByIdAndSupplier(id, supplierName)`, que **no existe** en
`BusinessManagementAPI`. El cast `(apiClient as any)` esconde el error de compilación; en runtime
(lado no-demo) tira `TypeError`. Además, `usePurchaseStore.fetchPurchaseOrderById` (línea 157)
llamaba a ese método con `supplierName: ''` — que el backend rechaza con 400 `INVALID_INPUT`
("Missing supplier_name parameter", `internal/purchase/http_order_read.go:51-54`).

**Fix (doble):**
1. **Implementar el método** en `BusinessManagementAPI` → `GET /purchase/{id}/supplier/{supplier_name}`
   (ruta real, `internal/purchase/http/routes.go:45`), con `encodeURIComponent` en el nombre. El
   flujo de validación con nombre real queda operativo.
2. **Corregir `fetchPurchaseOrderById`** del store para que use `purchaseService.getPurchaseById`
   (fetch plano por id, que ya existía y funciona): su semántica es "traer orden por id", no
   "validar proveedor"; pasar `''` violaba el contrato del endpoint.

---

### F3 — Fachada `apiService`: delegaciones a métodos inexistentes 🔴

**Estado:** ✅ Corregido
**Tipo:** bug latente de runtime (TypeError si alguien invoca)
**Refs:** `src/services/api.ts:40,43,45`

**Problema:** tres métodos de la fachada legacy delegaban a métodos que no existen en
`BusinessManagementAPI` (el cast `as any` lo ocultaba en compilación):

| Método fachada | Delegaba a (inexistente) | Ahora delega a (real) |
|:---------------|:-------------------------|:----------------------|
| `getProducts(page, pageSize)` | `apiClient.getProducts` | `apiClient.getProductsPaginated` (`/products/list/{page}/{pageSize}`) |
| `getBeachTennisCourts()` | `apiClient.getBeachTennisCourts` | `apiClient.getServiceProducts` (`/products/service-courts`) |
| `searchProductsByName(name)` | `apiClient.searchProductsByName` | `apiClient.searchProducts` (`/products/search/{name}`) |

**Nota:** verificado con grep — **0 consumidores** de estos tres métodos en `src/` (la fachada
legacy está en extinción). El fix es higiene: si algo legacy los invoca, deja de tirar TypeError.
Eliminarlos por completo es parte de F5/F7.

---

### F4 — `brandService`: casing camelCase sin transformar 🔴

**Estado:** ✅ Corregido
**Tipo:** bug silencioso de datos (el backend ignora campos que no reconoce)
**Refs:** `src/services/brandService.ts`, `src/features/brands/types/brand.ts:1-9`,
`internal/brand/dto.go:6-19`

**Problema:** la interfaz FE `Brand` usa `logoUrl`/`isActive` (camelCase) y el formulario
(`BrandDetailForm.tsx`) envía esos campos tal cual. El backend espera `logo_url` (create y update)
e `is_active` (solo update). Resultado: crear/actualizar una marca con logo o toggle de activo → el
backend decodifica esos campos como ausentes y **los datos se pierden silenciosamente**. Además, el
backend devuelve `logo_url`/`is_active`, que el FE nunca mapeó a `logoUrl`/`isActive` → la UI leía
`undefined`.

**Fix (bidireccional en el servicio, sin tocar la UI ni el backend):**
- **Request** (`create`/`update`): mapear `logoUrl → logo_url`, `isActive → is_active`; no enviar
  campos que el backend no conoce (`slug` es derivado server-side, `icon` no existe en el BE) y
  omitir `logo_url` vacío (el validator `omitempty,url` rechazaría `""`).
- **Response** (`getAll`/`getById`): mapear `logo_url → logoUrl`, `is_active → isActive`.

---

### F5 — Dead code: 13 endpoints huérfanos + `usePurchasePaymentStore.js` + wrappers de `useSaleStore.ts` ✅

**Estado:** ✅ Completado (2026-08-03)
**Tipo:** limpieza — reduce superficie de bugs, no cambia comportamiento
**Clasificación fuente:** B3 del frente backend (ver `ESTADO_INTEGRACION_BACKEND.md` §B3)

**⚠️ Corrección a la clasificación B3 durante la ejecución:** B3 concluyó que `purchasePaymentService.js`
+ `usePurchasePaymentStore.js` eran "candidatos a eliminación" completos. **Solo el store era
huérfano.** El servicio tiene 3 métodos **vivos** consumidos por el stack real de pagos
(`purchasePaymentsMvpService.js`: `processPayment`, `getPurchaseOrderById`; `useCancellationPreview.js`:
`getCancellationPreview`). Se eliminaron únicamente los 9 métodos que solo usaba el store huérfano.
Esto corrige `ESTADO_INTEGRACION_BACKEND.md` §B3 y la nota §3.10 de la auditoría (la "función zombie"
`getPaymentHistory` → `return []` desapareció).

**Eliminado (verificado 0 consumidores en `src/` antes de borrar):**

| Archivo | Eliminado | Endpoints muertos que desaparecen |
|:--------|:----------|:----------------------------------|
| `src/services/saleService.ts` | `startSaleSession`, `endSaleSession`, `refundSale`, `getSalesStats`, `getTopSellingProducts` | `POST /sale/session/start`, `POST /sale/session/{id}/end`, `POST /sale/{id}/refund`, `GET /sale/stats`, `GET /sale/top-products` |
| `src/store/useSaleStore.ts` | wrappers `startSaleSession`, `endSaleSession`, `refundSale`, `fetchSalesStats`, `fetchTopSellingProducts` + declaraciones de interfaz de los wrappers eliminados | — |
| `src/services/purchaseService.ts` | `getPurchaseOrderAnalysis`, `validatePurchaseOrderIntegrity`, `updatePurchaseOrderStatus` | `GET /purchase-orders/{id}/detailed-analysis`, `GET /purchase-orders/{id}/integrity-validation`, `PUT /purchase/{id}/status` |
| `src/features/purchases/hooks/usePurchasesLogic.ts` | bloque comentado que referenciaba `updatePurchaseOrderStatus` | — |
| `src/services/purchasePaymentService.js` | `createPurchaseOrder`, `getPurchaseOrders`, `getPaymentHistory`, `cancelPayment`, `getPaymentStatistics`, `cancelPurchaseOrder`, `verifyIntegration`, `validateCreatePurchaseOrderData`, `validatePaymentData` + constantes de `API_ENDPOINTS` muertas | `POST /purchase/enhanced`, `GET /purchase/1/20`, `GET /purchase/verify-integration`, `GET /purchase/orders/{id}/payments` (zombie `return []`), `PUT /purchase/payment/{id}/cancel` |
| `src/store/usePurchasePaymentStore.js` | **archivo eliminado** (store huérfano, 0 imports) | — |

**Se conserva (vivo):** `purchasePaymentService.processPayment`, `getPurchaseOrderById`,
`getCancellationPreview` (consumidos por el stack MVP real); `useSaleStore.updateStats` (cálculo
local, no depende de los métodos muertos); `saleService.getTodaySales`.

**Nota:** el conteo de "13 endpoints" de B3 incluía `PUT /sale/{id}` (updateSale) — ya eliminado en
F6. Los 8 endpoints muertos restantes se eliminaron aquí. Los wrappers de sesión y **el estado muerto
`currentSession`/`sessionActive`** fueron eliminados de la interfaz `SaleState` en `useSaleStore.ts`
(verificado: 0 lectores, 0 inicializaciones, 0 asignaciones en `src/`; `clearCurrentSale` usa
`sessionId: null`). Purga completada el 2026-08-05 — gate cumplido: `tsc --noEmit` ✅ 0 errores,
`pnpm build` ✅, tests en baseline (60 fallos pre-existentes, 0 nuevos).

---

### F6 — `updateSale` cancela en vez de actualizar ✅ RESUELTO (decisión de producto)

**Estado:** ✅ Resuelto — la venta es **inmutable post-create** → `updateSale` **eliminado** del FE
(2026-08-03, decisión del usuario)
**Tipo:** semántica de producto
**Refs:** `saleService.ts` (método eliminado), `useSaleStore.ts` (wrapper + interfaz eliminados),
`PUT /sale/{id}` = `MakeCancelSaleHandler` (`internal/sale/http/routes.go:63`)

**Hecho verificado:** el backend `PUT /sale/{id}` es **cancel-only por diseño** (decodifica
`{reason}` y cancela). El FE `updateSale` lo invocaba como si fuera update genérico → si algo lo
llamara, **cancelaría la venta**. Nadie lo consumía desde la UI (código muerto), el bug estaba
latente.

**Decisión (usuario):** la venta es **inmutable post-create** — solo se cancela (`revertSale`/`cancelSale`).
**Acción:** se eliminaron `saleService.updateSale`, el wrapper `updateSale` de `useSaleStore.ts` y su
declaración de interfaz. El riesgo de "cancelar sin querer" desaparece con el código. La limpieza
del resto de wrappers huérfanos del store queda en F5.

---

### F7 — Estructural (defer): "contrato HTTP unificado" 🟡

**Estado:** ✅ F7.1–F7.5 completadas y commiteadas (`e1b9454`, `7c665f7`, `2f68977`, `0dcbf27`, y `4a0f05d`–`f0c6a83` para F7.5); F7.6 (doble-`/api`) **eliminado de raíz por F7.7** (sin proxy rewrite; URL directa al backend). Los 484 errores TS pre-existentes que F7.5 dejó intactos intencionalmente se cerraron en **F8** (ver sección siguiente).
**Tipo:** deuda estructural heredada del refactor hexagonal (DO-12: sin cambio de contrato)
**Detalle:** `PLAN_F7_CONTRATO_HTTP.md` (sub-fases F7.1–F7.6, decisiones D-F7.x, registro de verificación).

| Ítem | Estado | Notas |
|:-----|:-------|:------|
| F7.1 — Desduplicar tipos (Currency ×3, PaymentMethod ×2) | ✅ Commiteada | `7c665f7`; solo `types.ts` define `Currency`/`PaymentMethod`; features re-exportan |
| F7.2 — Migrar `currency_code` → `code` (FE-first) | ✅ Commiteada | `2f68977`; 16 lectores + emisores + demo; **bug destapado:** `preparePayload` enviaba `currency_code` (BE espera `code`) |
| F7.3 — Unificar extractores de lista | ✅ Commiteada | `0dcbf27`; `src/utils/extractListResponse.ts`; ~250 líneas eliminadas, shape de salida intacto |
| F7.4 — Eliminar `(apiClient as any)` | ✅ Commiteada | `e1b9454`; 31 casts eliminados; **bug destapado:** `getSalesByClientId` firmas; `apiService` ya no es `any` |
| F7.5 — JS→TS de servicios legacy | ✅ **Completada** (2026-08-04) | **Hallazgo de medición (2026-08-04): 4 servicios JS son dead code** (0 consumidores verificados): `reportService`, `productAdjustmentService`, `manufacturingService`, `currencyConversionService` (~1.150 líneas) → se ELIMINAN, no se migran. Resto: ~7.700 líneas en 20 archivos vivos, migrados por familias. **Progreso:** dead code ✅ `4a0f05d` · pago núcleo ✅ `4ffa41e` · exchange/tasas ✅ `ef807be` (bug `getByDate`/`url` corregido) · pagos de compra ✅ `6208a79` · maestros ✅ `382304f` (tsc migrados 30→0) · BI/misc ✅ `6c153fc` (23→0) · inventario/pricing ✅ `f0c6a83` (81→0; bug `offset` try-scoped → `ReferenceError` en el fallback demo de `getProductHistory`, corregido). **Resultado: 0 servicios `.js` en la raíz de `src/services/`** (quedan 5 fixtures `.js` bajo `src/services/mocks/`) |
| F7.6 — Doble-`/api` | ✅ Eliminado de raíz por F7.7 (2026-08-05) | ADR F7.6 (opción C) reaperturado → refactor ejecutado: URL directa al backend, sin proxy rewrite |

---

### F7.7 — Contrato HTTP explícito (elimina doble-`/api`) ✅

**Estado:** ✅ Ejecutado (2026-08-05, sin commitear) · 🟡 Smoke pendiente de backend levantado
**Tipo:** refactor estructural — elimina el acoplamiento "doble `/api`" de raíz
**Contexto:** proyecto fuera de producción → se asume riesgo de refactor cross-file
**Detalle:** `PLAN_F7.7_BASEURL_EXPLICITO.md` (11 puntos, decisiones D-F7.7a–f, gate de verificación)

**Qué hace:** alinea dev/prod al modelo que cloud/demo ya usaban — `baseURL` es siempre la URL real
del backend (`http://localhost:5050` en dev), sin proxy rewrite. El proxy `/api` se elimina en Vite
(dev) y Nginx (prod). Cada servicio pasa el path exacto que el backend registra.

**Alcance (7 planificados → 11 ejecutados):** 3 envs + `api.config.js` + `vite.config.js` (proxy) +
`nginx.conf` (proxy) + 4 consumers ad-hoc (`devAuth.js`, `ApiStatusIndicator.jsx`,
`paymentApiDebug.ts`, `PaymentApiDiagnostic.jsx`) + 1 test MSW (`useProductStore.cache.test.js`) +
`.env.example` (comentario). Los consumers extra surgieron al romper un test MSW y por barrido grep.

**Gate de código:** ✅ `tsc --noEmit` 0 · `pnpm build` exit 0 · `pnpm test` 60 = baseline (verificado
con stash: 0 nuevos). **Smoke manual:** ⏳ pendiente de levantar backend en `localhost:5050`
(`GET /api/v1/brands` + `GET /products/list/1/10`).

---

### F8 — Pase de errores TypeScript pre-existentes (484 → 0) ✅

**Estado:** ✅ Completo (2026-08-05)
**Tipo:** deuda técnica — typecheck real del repo (distinto de `pnpm build`, que no ejecuta tsc)
**Base:** los 484 errores que F7.5 dejó **intencionalmente** intactos (pase aparte, mismo gate).
**Detalle:** `PLAN_F8_ERRORES_TYPESCRIPT.md` (F8.1–F8.5, 11 commits).

| Ítem | Estado | Commit |
|:-----|:-------|:-------|
| F8.1 — Reparar imports de `Brand` y unificar tipo | ✅ | `227f256` |
| F8.2 — Limpiar declaraciones no usadas (Purchases page) | ✅ | `969df21` |
| F8.3 — Resolver 83 errores de tipo en `useProductStore` | ✅ | `1f2e14b` |
| F8.4 — Eliminar imports/declaraciones no usadas (46 archivos) | ✅ | `a1c9e7a` |
| F8.5 — Contratos de tipos UI + bugs de contrato reales | ✅ | `710442c`→`99e6ee4` |
| Post-F8 — Sales: out-of-stock blocking, WIP TS, POS header | ✅ | `2005fd9`, `574a1c2` |

**Gate cumplido:** `pnpm exec tsc --noEmit` = **0 errores**, 0 tests nuevos rotos, `pnpm build` ✅.

---

## Verificación

| Ítem | Comando | Resultado |
|:-----|:--------|:----------|
| Build (transpile) | `pnpm build` | ✅ sin errores |
| **Typecheck TS real** | `pnpm exec tsc --noEmit` | ✅ **0 errores** tras F8 (2026-08-05). Baseline F7.5: 484 errores en 126 archivos; F8.1–F8.5 los cerraron (`227f256`→`99e6ee4`). Ver `PLAN_F8_ERRORES_TYPESCRIPT.md` |
| Tests unit | `pnpm test` | ✅ en verde relativo — 60 fallos pre-existentes = baseline (comparado con stash: 0 nuevos) |
| Lint | `pnpm lint` | ✅ sin errores nuevos |

---

## Decisiones registradas

| # | Decisión | Razón |
|:--|:---------|:------|
| D-F1 | Corregir el path en `BusinessManagementAPI` en vez de en los servicios | Es el único dueño del path; los dos consumidores (`saleService`, `salePaymentService`) quedan arreglados de una vez |
| D-F2a | Implementar `getPurchaseByIdAndSupplier` (no eliminar la llamada) | El backend sirve la ruta real; el flujo de validación por proveedor es legítimo cuando hay nombre |
| D-F2b | `fetchPurchaseOrderById` usa fetch plano por id | Pasar `''` como supplier_name viola el contrato (400 backend). La semántica del método es "traer por id"; la validación vive en su propio método |
| D-F3 | Apuntar los 3 delegates a métodos reales (no borrarlos) | 0 consumidores hoy, pero la fachada legacy está en extinción gradual; borrarla es F5/F7 |
| D-F4 | Mapeo bidireccional en el servicio, sin tocar UI ni backend | La UI (camelCase) y el contrato BE (snake_case) son ambos correctos en su capa; el adaptador es el servicio |
| D-F5 | Defer la limpieza de dead code a pase dedicado | Diff limpio y revisable, separado de los fixes de runtime; mismo patrón que la limpieza `e12c302` del backend |
| D-F6 | La venta es **inmutable post-create** (decisión del usuario, 2026-08-03) → `updateSale` eliminado del FE | El backend es cancel-only por diseño; mantener un "update" que cancela es una trampa. La cancelación real vive en `revertSale`/`cancelSale`. Si algún día se necesita edición real, es ADR + endpoint nuevo |
