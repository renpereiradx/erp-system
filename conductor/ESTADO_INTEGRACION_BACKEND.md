# Estado de Integración — Frente BACKEND

**Repo:** `business_management/` (Go, Clean Architecture pragmática, FASE 0–13 cerrada)
**Fecha de inicio:** 2026-08-03 · **Última actualización:** 2026-08-05 (auditoría + corrección de atribución §H1)
**Auditoría:** 2026-08-05 — verificadas todas las afirmaciones contra el código. El trabajo descrito es exacto.
Salvo una **corrección de atribución documental (H1)** —el fix de error-shape de Category se ejecutó en el commit
de B1 (`df3e32b`), no en B4 (`e39641e`) como decía el texto original; ver §B1 y §B4— el resto (commits, código,
tests, builds, rutas) se reproduce sin discrepancias.
**Auditoría base:** `../../AUDITORIA_INTEGRACION_FRONTEND_BACKEND.md`
**Rama de trabajo:** `business_management` repo → `fix/integracion-backend`
**Criterio de priorización:** primero los 🔴 que rompen runtime en el frontend, sin romper contrato
para clientes ya correctos. Documentar cada cambio y la decisión detrás.

---

## Resumen ejecutivo

El refactor hexagonal está **cerrado** (FASE 0–13). La deuda pendiente **no** es estructural sino de
**contrato HTTP**: el backend quedó con inconsistencias pre-existentes (5 wrappers de lista, PascalCase
en category, hash de password expuesto, shapes de error heterogéneos, asimetría de fechas en tax).
Este documento rastrea los fixes backend que **desbloquean** la integración con el frontend.

**Estado del frente backend tras este pase (TODO commiteado y testeado):**

| Tarea | Estado | Commit | Impacto |
|:------|:-------|:-------|:--------|
| B1 — Category PascalCase → snake_case (DTO) | ✅ Commiteado | `df3e32b` | Pantallas de categorías del FE dejan de recibir `{"ID":1,"Name":...}` |
| B2 — Eliminar hash de password de responses | ✅ Commiteado | `f2d1449` | Cierre de fuga de información en `/api/v1/users` |
| B3 — Clasificar endpoints inexistentes | ✅ Investigado | — | **Veredicto: 0 endpoints nuevos.** Los 13 son código muerto FE → limpieza en FE (F3) |
| B4 — Unificar shapes de error (2 outliers) | ✅ Commiteado | `e39641e` | Manufacturing/currency-convert ahora usan `{success,error:{code,message}}`. Category **ya estaba** corregido en B1 (`df3e32b`) — ver §H1 |
| **B5 — Tax `effective_start` acepta RFC3339** | ✅ Commiteado | `466eb7f` | Asimetría read/write de fechas (§4.3) resuelta; round-trip sin reformateo |
| Tests guardianes B1/B2/B4 | ✅ Commiteado | `7b1baab` | Cada fix ahora tiene test que blinda el contrato (antes solo status/length) |

**El backend queda en verde y commiteado.** Suite completa: `go test ./...` → **24 paquetes ok / 0 FAIL**
(antes 22; +`internal/currency/http`, +`internal/manufacturing`). `go build ./...`, `go vet ./...` y
`golangci-lint run` (incluye `depguard`) limpios.

El principio rector aplicado: **cada cambio de contrato debe ser backward-compatible salvo que se
documenten explícitamente como breaking**. Los fixes 🔴 corrigen bugs que ya rompen a clientes
existentes, así que enderezar el contrato *es* la compatibilidad.

**Ítems backend-actionable de la auditoría (§4.3–§4.10) — TODOS resueltos o clasificados:**

| Audit § | Ítem | Veredicto | Dónde |
|:--------|:-----|:----------|:------|
| §4.1 | Category PascalCase | ✅ FIX (B1) | §B1 |
| §4.3 | tax `effective_start` string-in/time-out | ✅ FIX aditivo (B5) | §B5 |
| §4.4 | `currency_code` alias duplicado | ⏸️ DEFER (breaking, blast-radius FE ~15+ archivos) | §Diferidos |
| §4.5 | `UpdateProductRequest.State` no-puntero | ⛔ NO_CHANGE (PUT full-replace intencional) | §Diferidos |
| §4.6 | 4 shapes de error | ✅ FIX (B4 + B1) | §B4 (3 outliers: manufacturing/currency-convert corregidos en `e39641e`; category ya corregido en `df3e32b`/B1) |
| §4.7 | hash de password en response | ✅ FIX (B2) | §B2 |
| §4.8 | login username+email sin validar formato | ⛔ NO_CHANGE (latente; arreglarlo rompería el FE) | §Diferidos |
| §4.10 | forecast dual-mount | ⛔ NO_CHANGE (decisión documentada D10-2) | §Diferidos |

**Lo que NO se hizo aquí (y por qué):**
- **No se tocaron los paths `/api/v1/`**: funcionaban por un doble-`/api` accidental (proxy + baseURL
  + endpoints). **Eliminado de raíz el 2026-08-05 por F7.7** (refactor FE: URL directa al backend,
  sin proxy rewrite en Vite/Nginx). El ADR F7.6 (opción C: documentar + guardia) fue reaperturado
  cuando el proyecto pasó a estar fuera de producción. Ver `PLAN_F7.7_BASEURL_EXPLICITO.md` y
  `ADR-F7.6-DOBLE_API.md` (historial). El backend no requiere cambios — los paths que registra son
  los que el FE ahora pasa directos.
  endpoints). Es una bomba de tiempo arquitectónica, no un bug activo. Requiere ADR cross-repo (ver
  §"Hallazgo crítico surgido durante B4").
- **No se unificaron los 5 wrappers de lista**: es deuda estructural mayor, rompe compatibilidad y
  requiere coordinación FE/BE simultánea. Candidato a fase posterior ("contrato HTTP unificado").
- **No se implementaron endpoints nuevos**: la investigación B3 demostró que todos los "faltantes"
  referenciados por el FE son código muerto.
- **No se quitó el alias `currency_code`** (§4.4): ~15+ archivos FE lo consumen; varios como key de
  matching exclusiva. Requiere migración FE-first. Ver §Diferidos.

---

## Tareas

### B1 — Category: crear Response DTO con JSON tags (PascalCase → snake_case) 🔴

**Estado:** ✅ Completado y commiteado (`df3e32b`)
**Tipo:** bug de contrato (rotura runtime en frontend)
**Refs:** `internal/category/domain.go`, `internal/category/dto.go`, `internal/category/http.go`

**Problema:** los handlers `GetCategoryByIdHandler`, `GetCategoriesHandler` y `GetCategoryTreeHandler`
codificaban directamente el struct de dominio `Category`, que **no tiene tags `json:`** → la respuesta
HTTP salía en PascalCase (`{"ID":1,"Name":"..."}`) rompiendo el contrato snake_case del resto del
sistema. Es el único contexto con este problema.

**Decisión:** NO meter tags `json:` en `domain.go` (rompería la separación dominio/transporte que el
refactor hexagonal estableció como regla). En cambio, crear un **`Response` DTO** en `dto.go` y
**mappers** `toCategoryResponse` / `toCategoryResponseList`, siguiendo el mismo patrón que `tax`
(`ToRateResponse`) y los demás contextos migrados.

**Detalles del cambio:**
- `internal/category/dto.go`: añadido `CategoryResponse` (snake_case), con campos `parent_id`,
  `children` (recursivo, `[]CategoryResponse`) y `default_tax_rate` anidado opcional reutilizando el
  *valor*, no el tipo `tax.RateResponse` —para evitar acoplar el **transport layer** de category al de
  tax, copiando los campos relevantes en un struct local `categoryTaxSummary`. Nótese que el **paquete**
  category sí importa `tax` a nivel dominio (`domain.go:6`, entidad `*tax.Rate`); lo que se preserva es
  el desacoplamiento del DTO/response de `tax.RateResponse`.
- `internal/category/dto.go`: añadidos mappers `toCategoryResponse(*Category) CategoryResponse` y
  `toCategoryResponseList([]*Category) []CategoryResponse`. Renombrado el wrapper de lista a usar el
  nuevo DTO.
- `internal/category/http.go`: los 3 handlers de lectura ahora codifican el DTO en vez de la entidad.
  Los handlers de escritura (Create/Update/Delete) ya usaban DTOs correctos (MessageResponse) → sin cambios.
- `internal/category/http.go`: **además**, en este mismo commit se reescribió el helper `respondWithError`
  de `{error: msg}` (string plano) → `{success:false, error:{code:"REQUEST_ERROR", message}}`, propagando a
  sus ~16 invocaciones. Este cambio resuelve el outlier de §4.6 para category. *(Originalmente atribuido a B4
  en el texto de este doc y del mensaje del commit `e39641e`; corregido en la auditoría 2026-08-05 — ver §H1:
  el diff de `e39641e` no contiene `internal/category/http.go`; el cambio real está en `df3e32b`.)*

**Riesgo de breaking:** NINGUNO para clientes correctos. Solo cambia de PascalCase→snake_case, que es
lo que el frontend ya espera. Si algún cliente dependía del PascalCase (no encontrado), era un bug
del cliente. Aditivo en el lado de error-shape (añade `success:false` y anida `message` bajo `error`).

**Verificación:** `go build ./...`, `go vet ./...`, `go test ./internal/category/...`. **Tests guardianes
añadidos** (`7b1baab`): `TestGetCategoriesHandler_FieldNames`, `TestGetCategoryByIdHandler_JSONContract`,
`TestGetCategoryTreeHandler_SnakeCase` (asertan snake presente Y Pascal ausente en cada nivel del árbol),
`TestToCategoryResponse_WithTaxSummary` (rama del mapper con `DefaultTaxRate` poblado, inalcanzable vía
los read handlers porque no enriquecen la tasa), y `TestRespondWithError_StandardShape` (guarda el cambio
de error-shape incluido en este mismo commit — ver §H1).

---

### B2 — Identity: eliminar hash de password de las respuestas de users 🔴

**Estado:** ✅ Completado y commiteado (`f2d1449`)
**Tipo:** bug de seguridad (fuga de información)
**Refs:** `internal/identity/http.go:989-1009` (struct), `:1025-1047` (mapper)

**Problema:** `userAdminResponse` incluía el campo `Password string \`json:"password,omitempty"\``, y el
mapper `toUserAdminResponse` lo copiaba desde `u.Password` (el hash bcrypt). Aunque `omitempty` lo
oculta si está vacío, el mapper lo setea siempre → **el hash viajaba en GET/POST `/api/v1/users`**.

**Decisión:** eliminar el campo `Password` del response DTO y del mapper. La contraseña NUNCA debe
serializarse en una respuesta HTTP, ni siquiera hasheada. La creación/actualización de usuario recibe
password por el request DTO (que sí debe tener el campo) — pero la respuesta no.

**Detalles del cambio:**
- `internal/identity/http.go`: eliminado el campo `Password` del struct `userAdminResponse` (struct en `:989-1009`; el campo `Password` estaba hacia `:993`).
- `internal/identity/http.go`: eliminada la línea `Password: u.Password` del mapper `toUserAdminResponse` (`:1025-1047`).
- El request DTO `userCreateRequest` (`:1060`; campo `Password` en `:1065`) y `changePasswordRequest`
  (`:1082`) **conservan** sus
  campos password — son entradas, no salidas.

**Riesgo de breaking:** NINGUNO legítimo. Si el frontend leía `password` del response (verificado: no
lo hace), era igualmente un bug. El frontend `userService.ts` no referencia ese campo.

**Verificación:** `go build ./...`, `go vet ./...`. **Tests guardianes añadidos** (`7b1baab`):
`TestGetUserHandler_JSONContract` ahora siembra un hash en el modelo y aserta su ausencia;
`TestListUsersHandler_PasswordAbsent` (lista con hashes, ninguno filtra);
`TestCreateUserHandler_PasswordNotInResponse` (POST con password de input, response no lo repite).

---

### B3 — Investigar endpoints inexistentes referenciados por el frontend 🟠

**Estado:** ✅ Investigado — **todos son código muerto en el frontend (desde la UI)**
**Tipo:** clasificación (¿implementar backend o eliminar código frontend muerto?)
**Veredicto global:** el backend **NO** implementa ningún endpoint nuevo. Toda la lista es código
muerto FE → se elimina en el frente FE (tarea F3).

#### Contexto SALE
| Endpoint | Servicio FE | Invocación UI | Veredicto |
|:---------|:-----------|:--------------|:----------|
| `POST /sale/session/start` | `saleService.startSaleSession` | solo `useSaleStore.ts:244`; **NO** consumido por componente (SalesNew.tsx no lo desestructura) | CÓDIGO MUERTO |
| `POST /sale/session/{id}/end` | `saleService.endSaleSession` | solo `useSaleStore.ts:280`; no consumido | CÓDIGO MUERTO |
| `POST /sale/{id}/refund` | `saleService.refundSale` | solo `useSaleStore.ts:615`; no consumido | CÓDIGO MUERTO |
| `GET /sale/stats` | `saleService.getSalesStats` | solo `useSaleStore.ts:642` (`fetchSalesStats`); no consumido | CÓDIGO MUERTO |
| `GET /sale/top-products` | `saleService.getTopSellingProducts` | solo `useSaleStore.ts:653` (`fetchTopSellingProducts`); no consumido | CÓDIGO MUERTO |
| `PUT /sale/{id}` (updateSale) | `saleService.updateSale` | solo `useSaleStore.ts:568`; no consumido. El flujo real de cancelación usa `revertSale` | CÓDIGO MUERTO |

#### Contexto PURCHASE
| Endpoint | Servicio FE | Invocación UI | Veredicto |
|:---------|:-----------|:--------------|:----------|
| `GET /purchase-orders/{id}/detailed-analysis` | `purchaseService.getPurchaseOrderAnalysis` | **0 invocaciones** en src/ | CÓDIGO MUERTO |
| `GET /purchase-orders/{id}/integrity-validation` | `purchaseService.validatePurchaseOrderIntegrity` | **0 invocaciones** en src/ | CÓDIGO MUERTO |
| `PUT /purchase/{id}/status` | `purchaseService.updatePurchaseOrderStatus` | 1 invocación **comentada** en `usePurchasesLogic.ts:1009` con TODO "endpoint no existe" | CÓDIGO MUERTO |
| `GET /purchase/verify-integration` | `purchasePaymentService.verifyIntegration` | solo `usePurchasePaymentStore.js:319` (store huérfano) | CÓDIGO MUERTO |
| `POST /purchase/enhanced` | `purchasePaymentService.createPurchaseOrder` | solo `usePurchasePaymentStore.js:73` (store huérfano) | CÓDIGO MUERTO |
| `GET /purchase/1/20` | `purchasePaymentService.getPurchaseOrders` | solo `usePurchasePaymentStore.js` (store huérfano) | CÓDIGO MUERTO |
| `—` (retorna []) | `purchasePaymentService.getPaymentHistory` | solo `usePurchasePaymentStore.js` (store huérfano); además `return []` hardcodeado | CÓDIGO MUERTO |

#### Hallazgo adicional de la investigación
- **`usePurchasePaymentStore.js` entero es un store huérfano**: ningún componente/hook/página lo importa.
  La UI real de pagos de compra usa un stack distinto: `usePurchasePaymentsMvpStore` +
  `purchasePaymentsMvpService.js`. ⇒ Todo `purchasePaymentService.js` + `usePurchasePaymentStore.js`
  son candidatos a eliminación (tarea FE F3).
  > **Corrección 2026-08-03 (ejecución F5 en el frente FE):** solo el **store** era huérfano.
  > `purchasePaymentService.js` conserva 3 métodos vivos (`processPayment`, `getPurchaseOrderById`,
  > `getCancellationPreview`) usados por el stack MVP real; se eliminaron sus 9 métodos muertos.
  > Detalle en `../conductor/ESTADO_INTEGRACION_FRONTEND.md` §F5.
- **`useSaleStore.ts` tiene 6 wrappers huérfanos**: `startSaleSession`, `endSaleSession`,
  `refundSale`, `updateSale`, `fetchSalesStats`, `fetchTopSellingProducts` — sus métodos
  subyacentes apuntan a endpoints inexistentes y nadie los consume.

**Decisión D-B3 confirmada:** el principio "no implementar backend hasta confirmar uso FE" dio su
fruto — **cero endpoints nuevos necesarios**. Toda la limpieza es FE (tarea F3 del frente frontend).

---

### B4 — Unificar shapes de error inconsistentes 🟡

**Estado:** ✅ Completado y commiteado (`e39641e`)
**Tipo:** deuda de contrato (no rompe, pero alimenta normalización defensiva en FE)

Variantes detectadas y **corregidas**:
| Contexto | Shape antes | Shape ahora | file | commit |
|:---------|:------------|:------------|:-----|:-------|
| Manufacturing | `{error: msg}` (string) | `{success:false, error:{code,message}}` | `internal/manufacturing/http.go:23` (helper `writeError`, propaga a ~28 invocaciones) | `e39641e` (B4) |
| Currency convert | `{error_code, message}` | `{success:false, error:{code,message}}` | `internal/currency/http/convert.go:16` (helper `writeConvertError`, 4 sitios) | `e39641e` (B4) |
| Category | `{error: msg}` (string) | `{success:false, error:{code,message}}` | `internal/category/http.go:16` (helper `respondWithError`, ~16 invocaciones) | **`df3e32b` (B1)** — no `e39641e` |

> **Corrección de atribución (auditoría 2026-08-05, §H1):** el texto original de este doc y el mensaje del
> commit `e39641e` listaban category como parte de B4. El diff real de `e39641e` **no** incluye
> `internal/category/http.go` (solo `currency/http/convert.go` + `manufacturing/http.go`); el cambio funcional
> de `respondWithError` en category se realizó dentro del commit `df3e32b` (B1), junto con el DTO. Funcionalmente
> correcto; solo la atribución documental era errónea. El test guardián `TestRespondWithError_StandardShape`
> (commit `7b1baab`) cubre este cambio.

**Decisión:** alinear los outliers al shape estándar que ya usan currency/brand/catalog/etc. vía
`apperr`. Los códigos se mantienen semánticamente (`MISSING_PARAMS`, `INVALID_AMOUNT`,
`FROM_CURRENCY_NOT_FOUND`, etc. en convert) para no romper al FE si lee `error.code`. El cambio es
aditivo: añade `success:false` y anida `message` bajo `error`, pero el FE normaliza ambas formas.

**Riesgo de breaking:** bajo. El FE `ApiError.toApiError` ya soporta `{error:{message}}` y `{message}`.
Quien leyera `response.error_code` (string plano) deja de recibirlo, pero se verificó que ningún
servicio FE lee ese campo — `currencyService.convert` solo lee el body en éxito.

**Tests guardianes añadidos** (`7b1baab`): `internal/currency/http/convert_test.go` (4 tests cubren
`MISSING_PARAMS`, `INVALID_AMOUNT`, `FROM_CURRENCY_NOT_FOUND`, `TO_CURRENCY_NOT_FOUND` y asiertan que el
legacy `error_code` plano desapareció); `internal/manufacturing/http_test.go` (`writeError` directo +
dos handlers vía harness con claims); `TestRespondWithError_StandardShape` en category.

---

### B5 — Tax: aceptar RFC3339 en `effective_start` (asimetría read/write §4.3) 🟡

**Estado:** ✅ Completado y commiteado (`466eb7f`)
**Tipo:** bug de contrato latente (asimetría fecha input/output)
**Refs:** `internal/tax/service.go:448` (`parseDate`), `internal/tax/dto.go:18,34,82`

**Problema:** los request DTOs `CreateRateRequest.EffectiveStart` y `UpdateRateRequest.EffectiveStart`
son `string` parseados como `"YYYY-MM-DD"`, pero `RateResponse.EffectiveStart` es `time.Time` →
serializa como RFC3339 (`"2026-08-03T00:00:00Z"`). Un cliente que reenvíe el valor recibido en un GET
para un PUT debía reformatearlo, o el PUT fallaba el parseo.

**Decisión D-B5:** relajar **solo el lado input** (`parseDate`): aceptar `"YYYY-MM-DD"` primero
(contrato legacy, sin cambios) y caer a `time.RFC3339` como fallback. El lado output
(`RateResponse.EffectiveStart time.Time`) **NO se toca** — es el contrato wire pre-existente
(`models.tax_rate` ya era `time.Time`); cambiarlo a emitir `"YYYY-MM-DD"` sería el cambio breaking
real. El cambio es puramente aditivo: todo input previamente aceptado sigue parseando idéntico.

**Detalles del cambio:**
- `internal/tax/service.go:448`: `parseDate` ahora intenta `"2006-01-02"` y, si falla, `time.RFC3339`.
- Mensajes de error de validación (líneas ~125/131/149/155) actualizados a "usar YYYY-MM-DD o RFC3339".
- `internal/tax/service_test.go`: añadido `TestParseDate` table-driven (legacy date-only, RFC3339 UTC,
  RFC3339 con offset, y 4 formas rechazadas).

**Riesgo de breaking:** NINGUNO. Aditivo puro. Confirmado: los tests existentes que usan
`"2026-01-01"` siguen pasando (primera rama del parser).

---

## Ítems auditados diferidos o descartados explícitamente (§4.4, §4.5, §4.8, §4.10)

Estos 4 ítems de la auditoría son backend-side y **no se fixean en este pase**. Se documentan aquí con
veredicto y razón para cerrar la brecha de honestidad (antes el doc era silente sobre ellos).

### §4.4 — `currency_code` alias duplicado ⏸️ DEFER (breaking)

`internal/currency/dto.go:48` emite `code` Y `currency_code` (ambos con el mismo valor, marcado
"backward compat"). **Quitarlo es breaking:** ~15+ archivos FE lo consumen, y varios lo usan como
**key de matching exclusiva** (no como fallback):
- `services/currencyService.js:331,334` — `getCurrencyCode` muere sin el alias; `if (!currency.currency_code)`.
- `components/payment/CurrencySelector.jsx:82-84` — la lógica de selección (`c.currency_code.toUpperCase() === value`) dejaría de matchear.
- `store/useCurrencyStore.js`, `usePaymentBootstrapStore.js`, `useExchangeRateStore.js`,
  `features/purchases/hooks/usePurchasesLogic.ts`, `pages/Currencies.jsx`, `pages/ExchangeRates.jsx`,
  `config/demoData.js` (8+ monedas demo), y los modales `RegisterPaymentModal`/`RegisterSalePaymentModal`.

**Decisión:** la migración debe ser **FE-first**: un PR que cambie cada consumidor a leer `code` (o
`currency_code || code` defensivamente), desplegar, y luego un PR BE que quite el alias. Mientras tanto
el alias es barato (un string extra por fila) y debe quedar. Se documenta para que no se quite por
accidente. Candidato a la fase de "contrato HTTP unificado".

### §4.5 — `UpdateProductRequest.State` no es puntero ⛔ NO_CHANGE

`internal/catalog/dto.go:37`: `State bool` (no `*bool`). Investigado:
- `UpdateProduct` (`catalog/service.go:107-156`) es **PUT full-replace** intencional: llama
  `UpdateFull(...)` (línea 145), swagger annotated `PUT` (`http.go:125`), docstring en `dto.go:33`.
  El único merge puntual es `base_unit` (legacy invariant), no semántica PATCH.
- El flujo de edición real del FE (`features/products/hooks/useProductForm.ts:220`) **siempre** envía
  `state: true` explícito → el `bool` actual funciona, no hay pérdida de datos ni falsa desactivación.
- Los callers FE que omiten `state` (`useProductStore.ts:1985`/`:2030`, bulk activate/deactivate con
  `{ is_active }`) **ya están rotos por razones no relacionadas**: no envían `name` ni `category_id`
  (que el service exige, líneas 109/112) y mandan un campo `is_active` que el DTO no reconoce.
  Cambiar `bool → *bool` no rescata ninguno.

**Decisión:** nit de estilo, no bug activo. Si se quiere semántica PATCH real, es un epic mayor
(pointer-izar todos los campos + fetch-then-merge en el service), no un drive-by a un campo. Sin
cambio en este pase.

### §4.8 — login envía `username` y `email` con el mismo valor ⛔ NO_CHANGE (latente)

`BusinessManagementAPI.login` construye `{username: X, email: X, password}`. El backend
(`identity/service.go:114-119`) usa **username primero** y solo cae a email si username está vacío:
el `email` duplicado (que puede no ser email) es **inert** hoy. No hay validación de formato de email
en el handler (`http.go:16-54`).

**Decisión:** arreglarlo (añadir validación de email) **rompería el FE actual**: cada login por
username pone un no-email en el campo `email` (`BusinessManagementAPI.ts:653`), y un
`mail.ParseAddress(email)` los rechazaría a todos. Es latente puro. La secuencia segura, si se quiere
endurecer, es FE-first (enviar solo el campo que corresponde) y luego validar backend. Sin cambio.

### §4.10 — forecast dual-mount (`/forecast/*` y `/api/v1/forecast/*`) ⛔ NO_CHANGE

`internal/analytics/http/routes.go:220-234` registra ambos prefijos con los mismos handlers, con
comentario "dual mount preserved per D10-2". Es una **decisión arquitectónica documentada**
(`conductor/fases/FASE_10_ANALYTICS.md:136`: "Conservar ambos mounts apuntando al nuevo handler; no
romper clientes"). El FE usa **exclusivamente** el legacy `/forecast/*` (`biForecastingService.js:561-593`);
cero referencias a `/api/v1/forecast` en el FE. Los goldens de rutas de FASE 13
(`tests/integration/route_walk_test.go`, comparando el set ordenado) protegen ambos mounts.

**Decisión:** intencional, documentado y testeado. Quitar el mount legacy rompería el FE; quitar el
`/api/v1/forecast` rompería los goldens y contradiría D10-2. Requiere decisión supersede (revocar
D10-2) + migración FE + actualización de goldens. Sin cambio.

---

## Hallazgo crítico surgido durante B4 — el "doble `/api`" (no es bug activo, es bomba de tiempo)

**Estado:** 🟡 Documentado (NO corregido — requiere decisión cross-repo)
**Severidad real:** baja hoy (todo funciona), alta si se toca un lado sin el otro.

**Qué pasa:** los servicios FE de la familia `/v1/` (brand, client, supplier, user, role, tag, audit,
attribute, variant) **funcionan por un accidente de doble prefijado**:
1. El servicio FE pasa el endpoint completo `/api/v1/brands` a `apiClient`.
2. `makeRequest` concatena `baseURL + endpoint` = `/api` + `/api/v1/brands` = **`/api/api/v1/brands`**.
3. El proxy (Vite/Nginx) quita **un solo** `/api` → el backend recibe `/api/v1/brands`.
4. El backend registra `/api/v1/brands` (brand/http/routes.go:36) → **match**.

**Por qué es frágil:** los tres puntos (baseURL `/api`, endpoints con `/api/v1/` completo, proxy que
quita un `/api`) están acoplados. Editar cualquiera en aislamiento rompe los 8+ servicios v1:
- Quitar el `rewrite` del proxy → 404.
- Quitar `/api/v1/` de los servicios FE → 404 (recibiría `/v1/brands`, backend espera `/api/v1/brands`).
- Cambiar baseURL a `''` → rompe los servicios no-v1 (products, sale, etc.).

**Decisión D-B4:** NO se toca en este pase. Es un refactor cross-repo (FE + proxy + posiblemente BE)
que merece su propio ADR. Se documenta aquí para que no pille por sorpresa. Candidato a la fase de
"contrato HTTP unificado" posterior.

---

## Pendientes NO backend (referencia, se tratan en el frente FE)

Estos surgen de la auditoría pero **no** se resuelven en el backend:

- **`processSalePayment` path incorrecto** (`/sale/payment` vs `/payment/process`): bug del
  `BusinessManagementAPI.ts:497`. El backend ya sirve `/payment/process` correctamente. Fix es FE puro.
- **`apiClient.getPurchaseByIdAndSupplier` inexistente**: el método falta en el cliente FE. El backend
  **sí** sirve `GET /purchase/{id}/supplier/{supplier_name}` (`purchase/http/routes.go:45`). Fix es FE.
- **`apiService.getProducts`/`getBeachTennisCourts`/`searchProductsByName`** delegan a métodos
  inexistentes: bug FE puro.
- **`brandService` casing camelCase**: bug FE (mapeo o cambio de interfaz).
- **`updateSale` cancela en vez de actualizar**: el backend `PUT /sale/{id}` es cancel-only por diseño.
  La decisión es de producto: ¿la venta es inmutable post-create? Si sí, eliminar `updateSale` del FE.
  Si no, diseñar un endpoint `PUT /sale/{id}` con semántica de update (sub-tarea nueva, requiere ADR).

---

## Registro de builds (evidencia)

| Fecha | Tarea | `go build ./...` | `go vet ./...` | `golangci-lint` | `go test` |
|:------|:------|:-----------------|:---------------|:----------------|:----------|
| 2026-08-03 | B1 (category DTO) | ✅ limpio | ✅ limpio | ✅ limpio | ✅ `./internal/category/...` ok |
| 2026-08-03 | B2 (identity password) | ✅ limpio | ✅ limpio | ✅ limpio | ✅ `./internal/identity/...` ok |
| 2026-08-03 | B4 (error shapes) | ✅ limpio | ✅ limpio | ✅ limpio | ✅ category/manufacturing/currency ok |
| 2026-08-03 | Tests guardianes B1/B2/B4 | ✅ limpio | ✅ limpio | ✅ limpio | ✅ **24 paquetes ok / 0 FAIL** |
| 2026-08-03 | B5 (tax RFC3339) | ✅ limpio | ✅ limpio | ✅ limpio | ✅ `./internal/tax/...` ok (TestParseDate + existentes) |

**Suite completa post-todos-los-fixes:** `go test ./...` → **24 paquetes ok, 0 FAIL**
(antes 22; +`internal/currency/http`, +`internal/manufacturing`). `go build ./...` → limpio.
`go vet ./...` → limpio. `golangci-lint run` → limpio (incluye `depguard`). Equivale a `make test` en verde.

**Limpieza de repo (commit aparte `e12c302`):** se eliminaron binarios commiteados por error
historicamente (`cloud-sql-proxy` 32MB, `server_bin` 11MB, ambos en `.gitignore`) y docs obsoletos
(`MIGRATION_*.md`, `README_DEPLOYMENT.md`). No mezclado con los fixes de contrato.

---

## Decisiones registradas

| # | Decisión | Razón |
|:---|:---------|:------|
| D-B1 | Crear DTO de respuesta para Category en vez de taggear el dominio | Preserva la regla del refactor hexagonal: el dominio no lleva tags de transporte. Sigue el patrón de `tax.RateResponse`. |
| D-B2 | Eliminar `Password` del response (no solo confiar en `omitempty`) | `omitempty` no protege: el mapper lo setea desde `u.Password`. La salida nunca debe contener el hash, ni siquiera hasheado. |
| D-B3 | No implementar endpoints backend nuevos hasta confirmar uso FE | Evita acumular superficie API basura. La verificación demostró que los 13 endpoints "faltantes" son código muerto FE → 0 endpoints nuevos. |
| D-B4a | Alinear los outliers de error al shape estándar (manufacturing/currency en B4; category ya en B1) | Cosmetic pero reduce la heurística de normalización del FE. Cambio aditivo (anida `message` bajo `error`, añade `success:false`). |
| D-B4b | NO tocar el doble-`/api` en este pase | Funciona por accidente acoplado (proxy + baseURL + endpoints). Editar un lado aislado rompe 8+ servicios. Requiere ADR cross-repo; defer a fase de "contrato HTTP unificado". |
| D-B5 | Relajar `parseDate` (input) a YYYY-MM-DD + RFC3339; NO tocar el output `time.Time` | El output es el contrato wire legacy (models.tax_rate ya era time.Time); cambiarlo sería el breaking real. Relajar el input es aditivo y resuelve el round-trip. |
| D-§4.4 | DEFER la eliminación del alias `currency_code` | Breaking: ~15+ archivos FE lo consumen, varios como key de matching exclusiva. Requiere migración FE-first. |
| D-§4.5 | NO_CHANGE en `State bool` | PUT full-replace intencional; el FE siempre envía `state`; los callers que lo omiten ya están rotos por otras razones. Nit de estilo. |
| D-§4.8 | NO_CHANGE en login username+email | `username` es autoritativo (`service.go:114-119`); el email duplicado es inert. Validarlo rompería el FE. Latente puro. |
| D-§4.10 | NO_CHANGE en forecast dual-mount | Decisión documentada D10-2; el FE usa el legacy `/forecast/*`; los goldens de rutas lo protegen. |
