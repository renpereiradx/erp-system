# Auditoría de Integración Frontend ↔ Backend (post-Refactor Hexagonal)

**Fecha:** 2026-08-03
**Alcance:** revisión del contrato API entre `business_management/` (backend Go, 14 contextos migrados a `internal/`) y `erp-webapp/` (frontend React/TS), sobre la base del `PLAN_HEXAGONAL_REFACTOR.md` ya ejecutado.
**Propósito:** servir de base sólida para re-evaluar la situación de integración. No es un plan de acción; es un inventario de hallazgos, observaciones sutiles y riesgos, con referencias `file:line` verificadas.

---

## Cómo leer este documento

- **Hallazgo**: cada ítem describe una situación concreta verificada en el código.
- **Severidad**: 🔴 rompe funcionalidad en runtime · 🟠 inconsistencia de contrato / bug latente · 🟡 deuda técnica / sutileza · ⚪ observación contextual.
- Las referencias `file:line` están verificadas contra el estado actual del repo (agosto 2026).
- Los endpoints se reportan **tal como los recibe el backend** (es decir, **después** del reescribe del proxy `/api/` → `/`). Ver §1.1 para entender esta transformación.

---

## 0. Estado real de la migración (discrepancia con AGENTS.md)

| Fuente | FASE 5 (Catalog) | FASE 6–13 | `handlers/` legacy |
|:-------|:-----------------|:----------|:-------------------|
| `AGENTS.md` (2026-07-21) | 🔄 2/7 | ⏳ Pendiente | "se migran por fase" |
| Estado verificado en código (2026-08-03) | ✅ **Migrado** (catalog con variants/tags/search) | ✅ **Completo hasta FASE 13** | Solo quedan helpers (`helpers.go`, `bi_helpers.go`, `branch_context.go`, `user.go` deshabilitado) |

**El refactor hexagonal está esencialmente cerrado.** Todos los contextos (catalog, currency, paymentmethod, tax, brand, category, analytics, inventory, purchase, party, measure, sale, manufacturing, identity) viven en `internal/<ctx>/`. **No hay handlers legacy activos de sale/purchase/product/inventory/analytics.** `routes/routes.go:33-99` quedó como orquestador thin (≤100 líneas, DO-11 cumplido).

**Implicación:** el AGENTS.md está desactualizado respecto al avance real. Las "reglas críticas para agentes" siguen siendo válidas, pero la tabla de estado de fases ya no refleja la realidad. Esto importa porque cualquier agente que lea el AGENTS.md asumirá que quedan fases por migrar y podría "migrar" contexto que ya está migrado.

---

## 1. Hallazgos del entorno / infraestructura de comunicación

### 1.1 ⚪ El proxy reescribe `/api/` → `/` — hay DOS familias de rutas en el backend

- Vite (`erp-webapp/vite.config.js:78-85`): `rewrite: path => path.replace(/^\/api/, '')`.
- Nginx (`erp-webapp/nginx.conf:22-28`): `rewrite ^/api/(.*)$ /$1 break;`.
- Base URL del frontend: `/api` (`erp-webapp/src/config/api.config.js:9`).

**Consecuencia:** cuando un servicio del frontend escribe `/api/v1/brands`, el backend recibe `/v1/brands`. Coexisten dos convenciones:

| Familia | Servicios del frontend | Path que recibe el backend |
|:--------|:-----------------------|:---------------------------|
| **Plana** (sin prefijo) | `BusinessManagementAPI`, `productService`, `saleService`, `purchaseService` (parcial), `categoryService`, `taxRateService`, `cashRegisterService`, `inventoryService`, `reservationService`, `budgetService`, `branchTransferService`, `purchaseRequisitionService`, `sessionService`, `exchangeRateService`, `paymentMethodService`, `currencyService`, `branchService`, `bi/*` | `/products`, `/sale`, `/currencies`, `/category/`, `/tax_rate/`, ... |
| **`/v1/...`** | `brandService`, `clientService`, `supplierService`, `userService`, `roleService`, `tagService`, `auditService`, `attributeService` (parcial), `variantService` (parcial) | `/v1/brands`, `/v1/parties`, `/v1/users`, `/v1/roles`, `/v1/tags`, `/v1/audit`, `/v1/attributes/definitions`, `/v1/variants` |

**Riesgo 🟡:** ambos prefijos funcionan hoy, pero es una carga cognitiva alta y un caldo de cultivo para bugs (ver §3.1). No rompe nada mientras el proxy esté activo; rompe en cualquier entorno donde el proxy no reescriba (ej: `.env.cloud` apunta directo al backend sin `/api`).

### 1.2 ⚪ No hay `/api/v1` como prefijo global en el backend

El backend **no** tiene un prefijo de versión global. Los `/api/v1/...` que el frontend asume se traducen a `/v1/...` reales, y el backend los sirve **solo porque cada `RegisterRoutes` de contexto monta su router en la raíz y usa paths que empiezan con `/v1/...` internamente** (ej: `internal/brand/http/routes.go:36` registra `/api/v1/brands` — notar que el backend sí espera `/api/v1/...` porque la reescritura del proxy solo quita `/api`, dejando `/v1/brands`... **excepto que el backend registra `/api/v1/brands`**, no `/v1/brands`).

> ⚠️ **Verificar esto con cuidado.** Hay una ambigüedad: el proxy quita `/api`, pero el backend registra `/api/v1/brands`. Si el frontend envía `/api/v1/brands` → el proxy lo convierte en `/v1/brands` → pero el backend espera `/api/v1/brands`. **Esto podría estar roto para los servicios de la familia `/v1/`, a menos que** (a) el proxy no aplique la reescritura a esos paths, o (b) el backend también registre alias sin `/api`. Este es uno de los hallazgos sutiles más importantes: **revisar si `brandService`, `clientService`, `userService`, etc. funcionan de verdad end-to-end o si fallan silenciosamente** (los servicios normalizan defensivamente y podrían no estar surfando el error). Confirmar con un curl real.

### 1.3 ⚪ Autenticación con headers custom (no solo JWT)

`BusinessManagementAPI.getAuthHeaders()` (`BusinessManagementAPI.ts:84-115`) inyecta, además del `Authorization: Bearer`:
- `X-Role-ID`, `X-Role-Name` (de `localStorage.roleId`/`roleName`).
- `X-Branch-ID` (de `localStorage.activeBranch`) — **solo si** no hay `branch_id` en query, no se pide `skipBranchContext`, y el endpoint no está en una lista negra de admin global (`/branches`, `/client`, `/auth`, `/users`, `/api/v1/users`).
- En POST/PUT/PATCH: si el body tiene `branch_id`, también se inyecta `X-Branch-ID` desde ahí (`BusinessManagementAPI.ts:592-627`).

**Observación 🟡:** el contexto multi-sucursal vive en headers, no en el path. Cualquier servicio que se salte `BusinessManagementAPI` (uso directo de `fetch`) pierde el contexto de branch. Hoy solo `BusinessManagementAPI.performTokenRefresh` usa `fetch` crudo (`BusinessManagementAPI.ts:30`), lo cual es correcto (no necesita branch context), pero es un patrón frágil.

### 1.4 ⚪ Refresh de token duplicado

Dos implementaciones del refresh:
- `BusinessManagementAPI.performTokenRefresh` (`BusinessManagementAPI.ts:22-68`) — con single-flight.
- `authService.refreshToken` (`authService.ts:110`) — `POST /auth/refresh` con `{ refresh_token }`.

**Riesgo 🟡:** si ambos corren en paralelo pueden generar refreshes duplicados o desincronizar el single-flight. El de `BusinessManagementAPI` es el que tiene la lógica correcta; el de `authService` parece legacy. Verificar si `authService.refreshToken` todavía se invoca desde algún lugar.

---

## 2. Hallazgos 🔴 — rompen funcionalidad en runtime

### 2.1 Endpoints invocados por el frontend que NO existen en el backend

Verificado contra `internal/sale/http/routes.go` y `internal/purchase/http/routes.go` completos. **Estos endpoints no están registrados en ningún `RegisterRoutes`.**

#### Contexto SALE
| Endpoint (frontend) | Invocado desde | Estado backend | Consecuencia |
|:---------------------|:---------------|:---------------|:-------------|
| `POST /sale/session/start` | `saleService.ts:158` (`startSaleSession`) | ❌ No existe | 404 si se invoca. **Confirmar si la UI lo llama.** |
| `POST /sale/session/{id}/end` | `saleService.ts:171` (`endSaleSession`) | ❌ No existe | 404. Confirmar invocación. |
| `POST /sale/{id}/refund` | `saleService.ts:557` (`refundSale`) | ❌ No existe | 404. `useSaleStore` lo referencia (`refundSale`). |
| `GET /sale/stats` | `saleService.ts:622` (`getSalesStats`) | ❌ No existe | 404. Las stats de venta viven en `analytics` (`/sales-analytics/*`). |
| `GET /sale/top-products` | `saleService.ts:635` (`getTopSellingProducts`) | ❌ No existe | 404. Existe `/dashboard/top-products` en analytics. |
| `POST /sale/payment` | `BusinessManagementAPI.processSalePayment:497` | ❌ No existe | **Crítico.** El endpoint real es `POST /payment/process` (`sale/http/routes.go:77`). El método `processSalePayment` usa path `/sale/payment` — **rompe el pago de ventas**. |

> **Sobre `PUT /sale/{id}`:** existe, pero **NO es update genérico**. El handler es `MakeCancelSaleHandler` (`sale/http.go:67-105`) que solo decodifica `{reason}` y cancela. `saleService.updateSale` (`saleService.ts:540`) lo usa como si fuera update → en realidad **cancelaría la venta**. 🟠 Bug semántico grave.

#### Contexto PURCHASE
| Endpoint (frontend) | Invocado desde | Estado backend | Consecuencia |
|:---------------------|:---------------|:---------------|:-------------|
| `GET /purchase-orders/{id}/detailed-analysis` | `purchaseService.ts:499` | ❌ No existe | El prefijo correcto es `/purchase/` (no `/purchase-orders/`). |
| `GET /purchase-orders/{id}/integrity-validation` | `purchaseService.ts:519` | ❌ No existe | Igual: prefijo incorrecto. |
| `PUT /purchase/{id}/status` | `purchaseService.ts:539` (`updatePurchaseOrderStatus`) | ❌ No existe | No hay cambio genérico de estado de compra. Solo cancelación. |
| `GET /purchase/verify-integration` | `purchasePaymentService.js:470` | ❌ No existe | 404. |
| `POST /purchase/enhanced` | `purchasePaymentService.js:83` | ❌ No existe | El real es `/purchase/complete`. **Dos servicios pelean por crear compras con endpoints distintos** (ver §3.2). |
| `GET /purchase/1/20` (paginación) | `purchasePaymentService.js:118` | ❌ No existe | El pattern backend es `/purchase/` + query, no `/purchase/{page}/{pageSize}`. |

> **Confirmado que SÍ existen** (para no confundir): `POST /purchase/{id}/cancel` (routes.go:42), `PUT /purchase/cancel/{id}` (routes.go:41), `GET /purchase/{id}/supplier/{supplier_name}` (routes.go:45), `POST /purchase/payment/process` (routes.go:53).

### 2.2 Método del cliente API inexistente, invocado desde un store

- `purchaseService.getPurchaseOrderWithSupplierValidation` (`purchaseService.ts:285-296`) llama a `apiClient.getPurchaseByIdAndSupplier(...)`.
- **Ese método NO existe en `BusinessManagementAPI`** (verificado: la clase no lo define).
- Pero `usePurchaseStore` **SÍ lo invoca** (`usePurchaseStore.ts:157, 250`).
- El cast `(apiClient as any)` hace que TypeScript no lo detecte en compilación.
- **En runtime:** `TypeError: apiClient.getPurchaseByIdAndSupplier is not a function` en la rama no-demo del store.

### 2.3 Fachada `apiService` con métodos que delegan a métodos inexistentes

`src/services/api.ts` expone métodos que delegan a `apiClient.<método>` que **no existen**:

| `apiService` (api.ts) | Delega a (inexistente) | Equivalente real |
|:----------------------|:----------------------|:-----------------|
| `getProducts` (`api.ts:40`) | `apiClient.getProducts(page,pageSize)` | `getProductsPaginated` |
| `getBeachTennisCourts` (`api.ts:43`) | `apiClient.getBeachTennisCourts()` | `getServiceProducts` |
| `searchProductsByName` (`api.ts:45`) | `apiClient.searchProductsByName(name)` | `searchProducts` |

Cast a `any` → sin error TS. **Runtime error si alguien los invoca.** Confirmar qué componentes todavía usan `apiService` (legacy) en vez de los servicios específicos.

### 2.4 `brandService` envía camelCase sin transformar

- Interfaz `Brand` (`features/brands/types/brand.ts:1-9`): `logoUrl`, `isActive` (camelCase).
- `brandService.create` (`brandService.ts:22-30`) y `update` (`:31-39`): pasan `data` **tal cual** al backend, sin mapear a `logo_url`/`is_active`.
- Backend (`internal/brand/dto.go:8`): `CreateRequest` espera `logo_url,omitempty` y `is_active` no está en create (sí en update, `dto.go:14`).
- **Resultado:** crear/actualizar marca con `logoUrl`/`isActive` → el backend **ignora esos campos** (los decodifica como ausentes). Bug silencioso de "datos que no se guardan".

---

## 3. Hallazgos 🟠 — inconsistencias de contrato / bugs latentes

### 3.1 Cinco formatos de envoltorio de lista distintos en el backend

El backend no tiene un wrapper de respuesta de lista unificado. Cada contexto eligió el suyo:

| Contexto | Shape | file:line |
|:---------|:------|:----------|
| Catalog, PaymentMethod, Tax, Brand, Inventory | **Array directo** `[{...}]` | ej. `catalog/http.go:304` |
| Currency, Sale (exchange-rates) | `{data, total, page, page_size, total_pages}` | `currency/dto.go:64`, `sale/domain.go:299` |
| Identity (users) | `{success, data, pagination:{page,page_size,total_items,total_pages,has_next,has_prev}}` | `identity/http.go:34` |
| Party | `{items, total, page, page_size}` | `party/domain.go:121` |
| Analytics/BI | `{success, data, metadata:{generated_at, scope}}` | `analytics/http.go:107` |

**Síntoma en el frontend:** `saleService.extractSalesAndPagination` (`saleService.ts:17-152`) y `purchaseService.extractPurchasesAndPagination` (`purchaseService.ts:18-149`) intentan ~10 claves distintas para el array (`data`, `data.data`, `sales`, `purchases`, `results`, `items`, `rows`) y ~12 claves de paginación (`page`, `page_size`, `pageSize`, `per_page`, `total_pages`, `totalRecords`, `total_items`, `total_count`, `has_next`, `hasNext`, `has_more`...). **Esta normalización defensiva masiva es síntoma directo del problema.** Mientras el backend no unifique, el frontend seguirá acumulando fallbacks.

### 3.2 Dos servicios frontend compiten por la misma operación (purchase)

| Operación | `purchaseService.ts` (TS) | `purchasePaymentService.js` (JS) |
|:----------|:--------------------------|:--------------------------------|
| Crear compra | `POST /purchase/complete` con `order_details` (`:166`) | `POST /purchase/enhanced` con `product_details` (`:83`) — **endpoint inexistente** |
| Cancelar compra | `POST /purchase/{id}/cancel` (via `apiClient`) | `PUT /purchase/cancel/{id}?branch_id=` (`:401`) |
| Obtener lista | via `apiClient.getPurchasesByDateRange` | `GET /purchase/1/20` hardcoded (`:118`) — **pattern inexistente** |

**Riesgo:** depende de qué servicio consuma cada pantalla, una funcionalidad funciona y la otra no. `usePurchasesLogic.ts` usa `purchaseService`; si alguna pantalla usa `purchasePaymentService`, falla. Ambos no deberían coexistir.

### 3.3 Cuatro paths distintos para "categorías"

| Path | Servicio | Comentario |
|:-----|:---------|:-----------|
| `GET /category/` | `categoryService.js:30` | El correcto (backend `internal/category/`). ⚠️ Pero devuelve PascalCase (ver §4.1). |
| `GET /products/all-categories` | `apiClient.getAllCategories` (`BusinessManagementAPI.ts:430`) | ¿Existe en backend? No encontrado en catalog routes. **Verificar.** |
| `GET /products/categories` | `apiClient.getCategories` (`:434`) | ¿Existe? **Verificar.** |
| `GET /categories` | `productService.getCategories` (`productService.ts:373`) | **No existe** en backend. |

### 3.4 Tres paths distintos para "clientes" y dos para "proveedores"

- `clientService.js`: `/api/v1/parties?party_type=CLIENT` (unificado).
- `reservationService.ts:116`: `/client/1/20` (legacy).
- `apiService` legacy: `/client/...`.

El backend (`internal/party/http/routes.go:22-60`) monta **triple alias**: `/client/*` (legacy), `/api/v1/parties`, `/api/v1/clients`, `/api/v1/suppliers`. Funciona pero es confuso.

### 3.5 `payment/process` (sale) — el contrato correcto

- `types.ts:2042` define `SALE_PAYMENT_PROCESS = '/payment/process'` (correcto).
- Pero `BusinessManagementAPI.processSalePayment` (`:497`) usa `/sale/payment` (incorrecto).
- **El endpoint real es `POST /payment/process`** (`sale/http/routes.go:77`).
- 🟠 La constante en `types.ts` está bien, el método en `BusinessManagementAPI` está mal. Si algún consumidor usa `API_ENDPOINTS.SALE_PAYMENT_PROCESS` directamente, funciona; si usa `apiClient.processSalePayment`, falla.

### 3.6 Casing conflictivo en tipos duplicados

#### Currency — 3 definiciones inconsistentes
| Origen | Campos clave |
|:-------|:-------------|
| `types.ts:626` (global) | `code`, `is_base`, `is_active` |
| `features/purchases/types/index.ts:47` (local) | `currency_code`, `is_base_currency`, `currency_name` |
| `currencyService.js` (normalización) | lee `currency_code || code`, `is_base || is_base_currency` |

El backend devuelve AMBOS: `code` Y `currency_code` (alias en `currency/dto.go:48`), `is_base` y `is_active`. Pero `features/purchases` solo conoce los alias legacy. Si se cambia el backend para limpiar el alias, el feature purchases se rompe.

#### PaymentMethod — 2 definiciones
| Origen | Campos |
|:-------|:-------|
| `types.ts:608` (global) | `id, method_code, description?, is_active, created_at?, updated_at?` |
| `features/purchases/types/index.ts:40` (local) | `id, method_code, name, description?` |

La local añade `name` (que el backend no envía) y omite `is_active`/timestamps. Componentes del feature purchases que lean `paymentMethod.name` obtienen `undefined`.

#### Brand — camelCase vs snake_case
Ver §2.4. El único modelo del frontend en camelCase (`logoUrl`, `isActive`).

### 3.7 Trailing slashes inconsistentes (Gorilla Mux los distingue)

**Requieren trailing slash** (POST/GET inicial): `/category/`, `/supplier/`, `/purchase/`, `/inventory/`, `/manual_adjustment/`, `/stock-transactions/`, `/cost-transactions/`, `/tax_rate/`, `/cash-movements/`, `/sale/`.

**Sin trailing slash**: `/products`, `/currencies`, `/payment-methods`, `/budgets`, `/branches`.

**Frontend:** `categoryService`, `taxRateService`, `saleService` (create) envían la trailing slash; otros servicios a veces no. Gorilla Mux con `StrictSlash` desactivado devuelve 404 si no coincide exactamente. **Verificar caso por caso** que el frontend coincida con el registro backend (la mayoría sí, pero es frágil).

### 3.8 `inventoryService` mezcla guion y underscore

- `GET /stock-transactions/product/{id}` (guion) ✓ backend.
- `POST /manual_adjustment` (underscore) en `inventoryService.js`, pero `API_ENDPOINTS.MANUAL_ADJUSTMENT_CREATE = '/manual-adjustment'` (guion) en `types.ts`.
- Backend: `POST /manual_adjustment/` (`inventory/http/routes.go` — underscore con slash).
- 🟠 El servicio JS usa underscore (coincide con backend), la constante TS usa guion (no coincide). Quien use la constante falla.

### 3.9 `reservationService.getInitialClients` usa path legacy

`reservationService.ts:116`: `GET /client/1/20`. El backend lo sirve (alias legacy de party), pero mezcla estilos dentro del mismo servicio (usa `/api/v1/parties` en `clientService`). Funciona por el triple-alias del backend, pero es deuda.

### 3.10 `purchasePaymentService.getPaymentHistory` devuelve siempre `[]`

`purchasePaymentService.js:188-211`: retorna `[]` hardcodeado con comentario de que el endpoint no existe. **Función zombie** — quien la consume cree que no hay historial cuando en realidad nunca se consulta al backend.

---

## 4. Hallazgos 🟡 — sutiles / deuda técnica

### 4.1 🟠 Category se serializa en PascalCase (bug de contrato)

- `internal/category/domain.go:9-22`: struct `Category` **sin tags `json:`** en ningún campo.
- `GetCategoryByIdHandler` (`category/http.go:75`) y `GetCategoryTreeHandler` codifican el struct de dominio directo.
- **Resultado HTTP:** `{"ID":1,"Name":"...","DefaultTaxRateID":2,"IsActive":true,"CreatedAt":"..."}` en vez de `{"id":1,"name":"...",...}`.
- `GetCategoriesHandler` usa el DTO `CategoriesResponse` (snake_case en el wrapper), pero los items internos siguen siendo `Category` → PascalCase.
- **Único contexto del sistema que rompe el snake_case.** El frontend `categoryService` normaliza `response.categories || response.data || response` pero no renombra `Name`→`name`, así que los componentes que lean `category.name` reciben `undefined` y solo funciona si leen `category.Name`.
- **Causa raíz:** el refactor hexagonal separó `domain.go` (sin tags) pero el handler expone la entidad en vez de un DTO. Falta el paso "k.6 adaptador HTTP → DTO de respuesta" para category.

### 4.2 🟡 Currency: respuesta inconsistente dentro del mismo endpoint

`GET /currencies` (`currency/http.go:15-60`):
- Sin filtros → `{data, total, page, page_size, total_pages}` (envoltorio paginado).
- Con `?id=` o `?code=` → `{success:true, data:{...}}` (envoltorio single).
- `?enriched=true` → otro comportamiento.

**Un endpoint, tres shapes.** El frontend `currencyService.getAll` (`:109`) y `getAllEnriched` (`:236`) tienen que manejar todos los casos.

### 4.3 🟡 Tax: `effective_start` string en request, `time.Time` en response

- `CreateRateRequest.effective_start` (`tax/dto.go:18`): `string` formato "YYYY-MM-DD".
- `RateResponse.effective_start` (`tax/dto.go:82`): `time.Time` → serializa como RFC3339 `"2026-08-03T00:00:00Z"`.
- **Asimetría read/write.** El frontend que reenvía el mismo valor que recibió en un GET para un PUT puede fallar el parseo de fecha. `taxRateService` no normaliza el formato.

### 4.4 🟡 `currency_code` alias duplicado (deuda de back-compat)

`currency/dto.go:48`: la respuesta incluye `code` Y `currency_code` (ambos con el mismo valor). Comentario indica "backward compat". Esto alimenta la proliferación de aliases en el frontend (§3.6). Cuando se limpie, romperá `features/purchases` que solo conoce `currency_code`.

### 4.5 🟡 `UpdateProductRequest.state` no es puntero

`catalog/dto.go:37`: `State bool` (no `*bool`). En un PUT full-replace, `false` es indistinguible de "no enviado". Coherente con semántica PUT, pero inconsistente con el resto de updates del sistema que usan punteros + omitempty (PATCH parcial). Si el frontend envía PUT sin `state`, este se interpreta como `false` (desactiva el producto). Confirmar que `productService` siempre envía `state` explícito.

### 4.6 🟡 Shapes de error distintos (4 variantes)

| Contexto | Shape de error |
|:---------|:---------------|
| Estándar (mayoría) | `{success:false, error:{code, message}}` |
| Manufacturing | `{error: msg}` (string plano, sin `success`/`code`) — `manufacturing/http.go:21` |
| Currency convert | `{error_code, message}` — `currency/http/convert.go:29` |
| Identity | `{success:false, error:..., ...}` |

El frontend `ApiError.toApiError` (`utils/ApiError.js`) normaliza 4 estructuras distintas. Funciona, pero es frágil.

### 4.7 🟡 `userAdminResponse.password` expone el hash

`identity/http.go:993`: el campo `password` (con `omitempty`) se copia del modelo en el mapper (`:1031`). Aunque `omitempty` lo oculta si está vacío, el mapper lo setea desde `u.Password` → **el hash viaja en GET/POST `/api/v1/users`**. Riesgo de fuga de información. No es un problema de integración FE/BE per se, pero conviene corregirlo.

### 4.8 🟡 Login envía `username` y `email` con el mismo valor

`BusinessManagementAPI.login` (`:648-662`) construye `{username: X, email: X, password}` — envía ambos campos con el mismo string. Comentario lo justifica como "máxima compatibilidad". El backend `LoginRequest` (`identity/http.go:16`) acepta ambos. Funciona, pero es un smell: si el backend alguna vez valida que `email` sea email y `username` no, falla para entradas que no son email.

### 4.9 🟡 `reportService` con fallback a datos simulados

`reportService.js:174-243`: `fetchWithFallback` retorna datos simulados si el fetch falla. **Silencia errores de integración:** si un endpoint de reportes no existe o falla, la UI muestra datos falsos sin alarma. Peligroso para la fase de "integración a la carrera" porque puede enmascarar 404s.

### 4.10 🟡 Forecast dual-mount

`analytics/http/routes.go:221-234`: `/forecast/*` y `/api/v1/forecast/*` ambos registrados con el mismo handler. Dos URLs para el mismo recurso. Inocuo pero añade ruido al contrato.

### 4.11 🟡 Hardcodes en servicios

- `paymentMethodService.js:8-13`: IDs 1-4 con códigos `CASH/CARD/TRANSFER/CHECK` hardcoded como demo.
- `purchasePaymentService.js:69-70`: `tax_rate_id: 1` y `profit_pct: 0.15` hardcoded al crear compra.
- Si el backend cambia los IDs seed (migración fresh), estos valores dejan de corresponder.

---

## 5. Mapeo de contextos — referencia rápida

Para cada contexto: endpoint base (lo que el **frontend** envía, pre-proxy), estado de integración, y notas.

| Contexto | Base (FE) | Shape lista (BE) | Notas de integración |
|:---------|:----------|:-----------------|:---------------------|
| **Catalog/Product** | `/products` | Array directo | ✅ Bien integrado. `/products/all-categories` y `/products/categories` (apiClient) — verificar existencia. |
| **Currency** | `/currencies` | Paginado + single inconsistente | 🟠 Alias `currency_code`/`is_base` alimentan duplicación de tipos. Endpoint `?enriched` y `?id=`/`?code=` con shapes distintos. |
| **PaymentMethod** | `/payment-methods` | Array directo | ✅ OK. Tipo local en `features/purchases` con `name` (no existe en BE). |
| **Tax Rate** | `/tax_rate/` | Array directo | 🟡 Asimetría fecha string→time. Trailing slash requerido. |
| **Tax Classification** | `/tax-classification` | — | ✅ OK. `API_ENDPOINTS.TAX_CLASSIFICATION_INFO` no definido (usa fallback). |
| **Brand** | `/api/v1/brands` | Array directo | 🔴 Casing camelCase en FE no se transforma (§2.4). |
| **Category** | `/category/` | PascalCase (roto) | 🔴 Struct sin tags json (§4.1). |
| **Identity/Auth** | `/login`, `/auth/*`, `/api/v1/users`, `/api/v1/roles`, `/branches`, `/sessions` | Variado (paginado users) | 🟡 `userAdminResponse.password` expone hash. 3 formatos de envoltorio distintos dentro del mismo contexto. |
| **Party/Client/Supplier** | `/api/v1/parties`, `/client`, `/api/v1/clients`, `/api/v1/suppliers` | `{items, total, page, page_size}` | 🟠 Triple alias. `reservationService` usa legacy `/client/1/20`. |
| **Inventory/Stock** | `/stock`, `/inventory`, `/manual_adjustment`, `/stock-transactions` | Objetos directos | 🟡 Mix guion/underscore. |
| **Purchase** | `/purchase`, `/supplier`, `/purchase/payment`, `/purchase-requisitions` | Variado | 🔴 `purchasePaymentService` usa endpoints inexistentes (§2.1, §3.2). `getPurchaseByIdAndSupplier` método inexistente invocado desde store (§2.2). |
| **Sale** | `/sale`, `/payment`, `/cash-registers`, `/cash-movements`, `/cash-audits`, `/budgets`, `/exchange-rates` | Variado | 🔴 `processSalePayment` path incorrecto (§3.5). Endpoints `/sale/session/*`, `/sale/stats`, `/sale/top-products`, `/sale/{id}/refund` inexistentes. `updateSale` en realidad cancela (§2.1). |
| **Analytics/BI** | `/dashboard`, `/sales-analytics`, `/inventory-analytics`, `/profitability`, `/financial-reports`, `/receivables`, `/payables`, `/audit` | `{success, data, metadata}` | ✅ BI unificado y consistente. `reportService` con fallback a datos falsos (§4.9). |
| **Measure** | `/unit-conversions`, `/barcode`, `/scale`, `/scales`, `/label-formats` | Objetos directos | ✅ OK. |
| **Manufacturing** | `/manufacturing` | — | 🟡 Shape error distinto `{error: msg}`. |
| **Cost/Pricing** | `/products/{id}/pricing-info`, `/cost-transactions`, `/suppliers/{id}/cost-analysis` | — | No auditado a fondo — **pendiente**. |

---

## 6. Observaciones para la re-evaluación

Estos son puntos sutiles que conviene tener en cuenta al volver a evaluar la situación:

### 6.1 La "normalización defensiva" del frontend es métrica de deuda backend

`saleService.extractSalesAndPagination` y `purchaseService.extractPurchasesAndPagination` (juntos, ~270 líneas de heurística de claves) existen **porque el backend devuelve 5 formatos de lista distintos**. Medir el progreso de la integración por "cuánto se puede simplificar estos extractores" es una métrica buena: cuando el backend unifique wrappers, estos extractores colapsan a 5 líneas.

### 6.2 El refactor hexagonal cerró la deuda estructural, no la de contrato

El `PLAN_HEXAGONAL_REFACTOR.md` era explícito: "Sin cambio de contrato HTTP ni esquema DB" (DO-12). El resultado es que la arquitectura interna está limpia, pero **el contrato HTTP quedó con todas sus inconsistencias pre-existentes intactas** (5 wrappers, PascalCase en category, aliases, trailing slashes). La próxima fase natural sería un **"contrato HTTP unificado"** — pero eso sí rompe compatibilidad y requiere coordinación FE/BE simultánea.

### 6.3 Hay dos capas de servicios frontend compitiendo

- Capa nueva (TS, tipada): `productService.ts`, `saleService.ts`, `purchaseService.ts`, `brandService.ts`, etc.
- Capa legacy (JS, sin tipar): `purchasePaymentService.js`, `paymentMethodService.js`, `currencyService.js`, `cashRegisterService.js`, `inventoryService.js`, `clientService.js`, `supplierService.js`, `categoryService.js`, `manufacturingService.js`, `reportService.js`, `exchangeRateService.js`.
- Fachada `apiService` (`api.ts`) que mezcla delegaciones a `apiClient` con bugs latentes.

**Riesgo:** la migración FE a TS está a medio camino. Los servicios JS no tienen tipos → no hay protección de contrato. Priorizar la migración de los servicios que tocan los endpoints 🔴 (purchase, sale) antes de depurar integration bugs a ciegas.

### 6.4 El `(apiClient as any)` esconde bugs de tipo

Varios servicios usan `(apiClient as any).post(...)` / `.put(...)` en vez de métodos tipados. Esto:
- Desactiva la verificación de tipos (§2.2, §2.3, §2.4 no se detectan en compilación).
- Permite inventar métodos/paths sin que TS se queje.
- Es el patrón que dejó pasar los bugs de `getPurchaseByIdAndSupplier`, `getProducts`, `getBeachTennisCourts`, y el casing de Brand.

**Acción de bajo costo:** eliminar los `as any` y añadir los métodos faltantes a `BusinessManagementAPI` (o usar los verbos genéricos `get/post/put/delete` tipados). TS empezará a marcar todos los bugs latentes de una vez.

### 6.5 Hay endpoints "confirmados inexistentes" que quizá sí se usan en la UI

La auditoría confirma que los endpoints NO existen en el backend y que los servicios del frontend los referencian. **Falta confirmar si la UI realmente dispara esos flujos.** Algunos (sale/session, sale/stats, sale/top-products) podrían ser código muerto en servicios que ninguna pantalla invoca. Recomendación: antes de "arreglar" creando el endpoint en el backend, confirmar con un `grep` de uso en componentes/hooks/stores si el método del servicio realmente se invoca. Quizá baste con eliminar el método del servicio.

### 6.6 Order de resolución de ambigüedades para no ahogarse

Si se aborda la integración "a la carrera", priorizar así:

1. **🔴 `processSalePayment` path incorrecto** (§3.5) — rompe todo pago de venta. Fix de 1 línea en `BusinessManagementAPI.ts:497`.
2. **🔴 `updateSale` cancela en vez de actualizar** (§2.1) — bug semántico crítico. Decidir: ¿venta es inmutable post-create (entonces `updateSale` debería no existir en el FE) o falta endpoint?
3. **🔴 `getPurchaseByIdAndSupplier` inexistente invocado desde store** (§2.2) — rompe flujo de validación de compra. Fix: implementar el método en `BusinessManagementAPI` o eliminar la llamada del store.
4. **🔴 `brandService` casing** (§2.4) — las marcas no se guardan bien. Fix: mapeo camelCase→snake_case en el servicio o cambiar la interfaz a snake_case.
5. **🔴 `purchasePaymentService` endpoints inexistentes** (§3.2) — decidir si se elimina el servicio (a favor de `purchaseService`) o se corrige.
6. **🟠 Category PascalCase** (§4.1) — cualquier pantalla de categorías que lea `category.name` está rota. Fix backend: añadir tags `json:` al struct o crear DTO.
7. El resto (wrappers, aliases, trailing slashes) es deuda estructural que se aborda con un contrato unificado.

### 6.7 Tests E2E como red de seguridad

La heterogeneidad del contrato hace que los tests unitarios (FE mockeando BE, o viceversa) sean poco fiables: el mock puede asumir un shape que el BE no produce. **Tests E2E (browser → backend real)** son la única forma fiable de detectar los 🔴. Si existe infraestructura para ello (el skill `web-gui-tester` está disponible), conviene levantar una batería mínima de smoke tests por flujo crítico: login, crear producto, crear venta + pago, crear compra + pago, CRUD marca, CRUD categoría.

---

## 7. Endpoints backend — referencia completa (post-proxy)

Lista de todos los endpoints registrados, agrupados por contexto, **tal como el backend los sirve** (es decir, lo que llega después del proxy). Útil para verificar cualquier path del frontend.

### Identity
```
GET  /                            (home, pre-auth)
POST /login                       (pre-auth)
GET  /setup/status                (pre-auth)
POST /setup/initialize            (pre-auth, rate-limited)
POST /auth/refresh  /auth/logout  /auth/logout-all
CRUD /branches                    + /branches/{id}/fiscal-config, /branches/fiscal-config/{id}
                                  + /branches/{branch_id}/access (CRUD), /users/{user_id}/branches
CRUD /branch-transfers            + /branch-transfers/{id}/status
GET  /sessions/active  /history  /activity  /config
POST /sessions/{id}/revoke  /sessions/revoke-all
POST /admin/sessions/cleanup  /{id}/revoke  GET /admin/sessions/all        (perm sessions:admin)
CRUD /api/v1/users                + /me, /me/change-password, /{id}/{activate,deactivate,change-password}, /{id}/roles
CRUD /api/v1/roles                + /{id}/permissions
GET  /api/v1/permissions
```

### Catalog
```
POST /products                              GET /products/list/{page}/{pageSize}
GET  /products/all  /products/service-courts
GET  /products/search/{name}  /products/search/facets
POST /products/search/advanced
GET  /products/barcode/{barcode}
GET  /products/info/barcode/{barcode}  /products/info/search/{name}
GET  /products/financial/barcode/{barcode}  /products/financial/search/{name}
GET  /products/{id}/info  /sale  /purchase  /financial
CRUD /products/{id}/attributes              + POST /products/{id}/attributes/bulk
CRUD /products/{id}/variants                + GET /products/{id}/total-stock, POST /products/{id}/reconcile-base-stock
GET  /products/{id}/tags                    POST/DELETE /products/{id}/tags/{tagId}
GET  /products/{id}/applicable-attributes
GET/PUT/DELETE /products/{id}
CRUD /api/v1/attributes/definitions         + /category/{categoryId}
CRUD /api/v1/tags
CRUD /api/v1/attributes/templates           + POST /api/v1/attributes/templates/{businessType}/apply
GET/PUT/DELETE /api/v1/variants/{id}        + GET /api/v1/variants/{id}/stock
```

### Currency
```
GET  /currencies                  (?id=, ?code=, ?enriched=) — shapes distintos
POST /currencies
GET  /currencies/convert          (?from=&to=&amount=&date=)
PUT/DELETE /currencies/{id}
```

### PaymentMethod
```
GET  /payments/bootstrap          (sin auth)
CRUD /payment-methods             + GET /payment-methods/code/{code}
```

### Tax
```
POST /tax_rate/                   (trailing slash)
GET  /tax_rate/name/{name}  /code/{code}  /default  /{id}  /{page}/{pageSize}
PUT/DELETE /tax_rate/{id}
GET  /tax-classification/info  /defaults  /product/{product_id}  /code/{code}  /{id}
POST /tax-classification/assign  /bulk-assign  /auto-classify
PUT/DELETE /tax-classification/{id}
```

### Brand
```
GET  /products/brands             (alias legacy)
CRUD /api/v1/brands
```

### Category
```
GET  /category/tree               (⚠️ PascalCase)
POST /category/                   (trailing slash)
GET  /category/  /{id}
PUT/DELETE /category/{id}
```

### Inventory
```
POST /stock/{product_id}          GET /stock/{id}  /stock/product_id/{product_id}
PUT  /stock/{id}  /stock/product_id/{product_id}
POST /inventory/                  GET/PUT /inventory/{id}  GET /inventory/{page}/{pageSize}
POST /manual_adjustment/          GET /manual_adjustment/integration/verify  /product/{productId}/history  /{page}/{pageSize}
GET  /schedules/available  /today  /product/{productId}/...  /generate/{daily|date|date/custom-range}  /{id}
POST /reserve/manage              GET /reserve/{available-schedules|report|consistency/check|all|date-range|client/name/{name}|product/{product_id}|client/{client_id}|{id}}
POST /stock-transactions/         GET /stock-transactions/{product/{product_id}|validate-consistency|discrepancy-report|movement-summary|by-date|types|{id}}
```

### Purchase
```
POST /supplier/                   GET /supplier/{id}  /name/{name}  /{page}/{pageSize}  PUT /supplier/{id}  /supplier/delete/{id}
POST /purchase/  /purchase/complete
PUT  /purchase/cancel/{id}        POST /purchase/{id}/cancel
GET  /purchase/{id}/preview-cancellation  /purchase/{id}  /purchase/{id}/supplier/{supplier_name}
GET  /purchase/supplier_id/{supplier_id}  /supplier_name/{name}  /date_range/
POST /purchase/payment/process    GET /purchase/payment/statistics
CRUD /purchase-requisitions       + /status/{status}  /user/{user_id}  /my  /{id}/status
```

### Party
```
POST /client/                     GET /client/{id}  /name/{name}  /{page}/{pageSize}  PUT /client/{id}  /client/delete/{id}
CRUD /api/v1/parties              + PATCH /api/v1/parties/{id}/type
CRUD /api/v1/clients  /api/v1/suppliers
```

### Measure
```
GET /unit-conversions/template    GET/POST /unit-conversions/   DELETE /unit-conversions/{from_unit}/{to_unit}
POST /barcode/decode  /barcode/generate
POST /scale/weigh-item  /scale/generate-label   GET /scale/catalog
CRUD /scales  /label-formats
```

### Sale
```
POST /sales/orders                (alias)   POST /sales/scan   (cross-context measure)
POST /sale/  /sale/with-units
GET  /sale/price-changes/report  /date_range/payment-status  /date_range
GET  /sale/client_id/{client_id}/pending  /client_id/{client_id}
GET  /sale/client_name/{name}/payment-status  /client_name/{name}
GET  /sale/{id}/with-metadata  /{id}/payment-status
PUT  /sale/{id}/confirm-payment  (← confirmar pago)
GET  /sale/{id}/preview-cancellation
POST /sale/{id}/products
PUT  /sale/{id}                   (← CANCELACIÓN, no update)
GET  /sale/{id}
CRUD /exchange-rates
POST /payment/process  /payment/process-partial
GET  /payment/details/{saleId}  /payment/{paymentId}  /payment/statistics/change  /payment/totals/sales  /payment/totals/purchases
POST /cash-registers/open         GET /cash-registers/active
POST /cash-registers/payments/sale  /cash-registers/payments/purchase   (cross-context)
CRUD /cash-registers              + /{id}/close  /{id}/movements  /{id}/movements/filter  /{id}/report  /{id}/audits  /{id}/balance-summary
POST /cash-movements/  /cash-movements/{id}/void
POST /cash-audits                 GET /cash-audits/denominations  /{id}   POST /cash-audits/{id}/resolve
CRUD /budgets                     + /client/{client_id}  /status/{status}  /mark-expired  /{id}/status  /{id}/convert
```

### Manufacturing
```
CRUD /manufacturing/supplies
POST/GET /manufacturing/purchases
POST /manufacturing/production     GET /manufacturing/production/batches
GET /manufacturing/recipes/{product_id}
POST /manufacturing/recipes/{product_id}/ingredients
DELETE /manufacturing/recipes/{product_id}/ingredients/{supply_id}
GET /manufacturing/reports/{supply-purchases|production|profitability}
```

### Analytics/BI (~140 endpoints, todos GET salvo export)
```
/dashboard/{summary,kpis,trends,alerts,top-products,sales-heatmap,recent-activity}
/sales-analytics/{performance,trends,by-category,by-product,top-bottom-products,by-customer,by-seller,by-payment-method,by-hour,by-day-of-week,heatmap,velocity,compare,dashboard}
/inventory-analytics/{overview,stock-levels,turnover,abc,dead-stock,reorder,aging,movements,forecast,dashboard}
/profitability/{overview,products,customers,categories,trends,sellers,dashboard}
/financial-reports/{income-statement,cash-flow,vat,sales-ledger,purchase-ledger,profit-margins,tax-summary,overview,health-score,compare-periods}
/receivables/{overview,overdue,top-debtors,aging/summary,aging/report,collection/reminders,statistics,client/{client_id}/risk,client/{client_id},{id},""}
/payables/{overview,overdue,urgent,top-suppliers,schedule,aging,reminders,statistics,cash-flow,payment-capacity,supplier/{supplier_id}/analysis,supplier/{supplier_id},{id},""}
/api/v1/audit/{logs,logs/{id},summary,summary/range,trends,trends/range,entity/{entity_type}/{entity_id}/history,users/top,users/{user_id}/activity,dashboard,categories,actions,levels,entity-types}
POST /api/v1/audit/export
/forecast/{sales,inventory,demand,revenue,dashboard}      (dual-mount con /api/v1/forecast/*)
Query params estándar: period, start_date+end_date, granularity, limit, page/page_size, sort_by/sort_order, compare
```

### Health
```
GET /health   GET /metrics   (Prometheus)
```

---

## 8. Glosario de archivos clave (referencia rápida)

### Backend
- Orquestador de rutas: `business_management/routes/routes.go:33-99`
- Por contexto: `business_management/internal/<ctx>/http/routes.go`
- Handlers: `business_management/internal/<ctx>/http.go`
- DTOs: `business_management/internal/<ctx>/dto.go` (o `domain.go` para los que no separaron DTO)

### Frontend
- Cliente HTTP: `erp-webapp/src/services/BusinessManagementAPI.ts`
- Fachada legacy: `erp-webapp/src/services/api.ts`
- Config URL base: `erp-webapp/src/config/api.config.js`
- Proxy: `erp-webapp/vite.config.js:78-85`, `erp-webapp/nginx.conf:22-28`
- Errores: `erp-webapp/src/utils/ApiError.js`
- Tipos: `erp-webapp/src/types.ts` (2360 líneas)
- Constantes endpoints: `erp-webapp/src/types.ts:1986-2187` (`API_ENDPOINTS`)
- Auth: `erp-webapp/src/services/authService.ts`, `erp-webapp/src/contexts/AuthContext.tsx`
- Stores: `erp-webapp/src/store/useSaleStore.ts`, `erp-webapp/src/store/usePurchaseStore.ts`
- Features: `erp-webapp/src/features/{products,purchases,sales,brands,branches}/`

---

## 9. Próximos pasos sugeridos (no es plan, es opciones)

Al re-evaluar sobre esta base, los caminos posibles son:

1. **Fix & go (mínimo):** abordar solo los 🔴 de §2 y el §6.6 (orden sugerido). Deja el sistema funcional para integración "a la carrera" sin tocar contrato.
2. **Contrato unificado (medio):** después de (1), unificar los 5 wrappers de lista en uno solo + corregir category PascalCase + eliminar aliases. Requiere coordinación FE/BE simultánea; rompe compatibilidad.
3. **Migración FE a TS (paralelo):** migrar los servicios JS legacy a TS, eliminar `apiService` y los `as any`. Reduce la superficie de bugs latentes.
4. **Smoke tests E2E (base):** levantar una batería mínima de tests browser→backend por flujo crítico como red de seguridad antes de cualquier cambio.

Este documento no elige; solo documenta para que la decisión se tome informada.
