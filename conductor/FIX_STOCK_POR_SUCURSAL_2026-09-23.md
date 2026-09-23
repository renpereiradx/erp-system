# FIX: Stock de productos por sucursal en búsqueda avanzada — 2026-09-23

**Síntoma reportado:** al cambiar de sucursal en el menú superior y revisar el
stock de productos, los números no cambiaban nunca.

**Severidad real:** el instinto era correcto — no era un bug del módulo
productos solamente. Todo consumidor de `POST /products/search/advanced`
(página de productos con filas planas, pickers de ventas, presupuestos y
requisiciones vía `sellableUnitSearch`) veía **stock global sumado** en lugar
del stock de su sucursal activa.

## Causa raíz (2 capas + 3 bugs hermanos)

### 1. Backend — el handler descartaba la sucursal (causa principal)

`MakeAdvancedProductSearchHandler` (`internal/catalog/http.go`) llamaba
`ResolveBranchContext` pero **descartaba** el `branchID` resuelto: solo honraba
`filter.branch_id` del body JSON, que el frontend nunca envía (envía el header
`X-Branch-ID`). Resultado: el SQL sumaba `products.stock` de **todas** las
sucursales. Encima la caché de búsqueda (TTL 3 min) hashea solo el body → misma
key para todas las sucursales.

Reproducido en vivo (:5050, pre-fix): `X-Branch-ID: 1` y `X-Branch-ID: 3`
devolvían ambos stock 209 para "Coca Cola 500ml" (realidad: 10 y 11; global 209).

**Fix:** el handler ahora defaultea `filter.BranchID` desde la sucursal
resuelta **solo cuando hay selección explícita** (header o `?branch_id=`) —
la mutación del filter alimenta además la caché (`hashFilter`), que ahora
distingue sucursales. Sin selección explícita (vista global del admin) el
filtro queda nil = suma global, preservando la semántica de "Todas las
Sucursales". El `branch_id` del body, si viene, sigue ganando (con su check
de acceso).

Clave del diseño: **todo usuario con acceso a sucursales tiene `active_branch`
en el JWT** (`ResolveBranchClaimsFromAccess`), así que "sin header" es la
ÚNICA señal de vista global — no se puede defaultear desde claims sin matarla.

### 2. Frontend — la página productos no refetcheaba al cambiar sucursal

`useProductStore` (Zustand) tiene cachés propios (`pageCache`/`searchCache`,
TTL 120s) sin sucursal en la key, y `useProductsLogic` fetcheaba solo en el
mount. `MainLayout` limpia la caché de **react-query** al cambiar sucursal,
pero el módulo productos no usa react-query → seguía mostrando el stock viejo
aunque el backend estuviera arreglado.

**Fix:** (a) el store escucha el evento `branch:changed` (que emite
`BranchContext.changeBranch` — era código muerto) y vacía ambas cachés;
(b) `useProductsLogic` consume `useBranch()` y refethea la vista vigente
(paginada o búsqueda/filtros avanzados) al cambiar `currentBranchId`.

### 3. Bugs hermanos: la selección de sucursal se revertía sola

La selección de sucursal es client-side (localStorage + header); el backend
siempre reporta la default del JWT. Tres sitios la pisaban:

| Sitio | Síntoma | Fix |
|---|---|---|
| `/auth/refresh` (API client + authService) | refresh silencioso revertía la sucursal | solo sembrar si no hay selección |
| `/me` en `initializeAuth` (AuthContext:105) | **cada F5** revertía la sucursal | mismo guard |
| Efecto init de `BranchContext` | re-anclaba la rama del JWT al re-ejecutarse (refresh de token/usuario), matando la vista global | gate `initializedRef` (solo ancla en la primera inicialización) |

Además la vista global se representaba como "key ausente" — indistinguible de
"sin selección aún". Ahora `changeBranch(null)` escribe `branchView=global`
(marcador que sobrevive F5, se limpia al elegir sucursal y en logout para no
filtrar al siguiente usuario).

## Verificación

- BE: `go build`, `go test ./...`, `golangci-lint` en verde; tests nuevos
  (`TestAdvancedSearchHandler_BranchScoping`: header scopea, body gana, sin
  header queda global; `TestAdvancedSearchCacheKey_DistinguishesBranches`).
- FE: vitest 1124/1124 (10 tests nuevos: store invalida por evento, hook
  refetchea por sucursal incl. global, BranchContext persiste vista global,
  refresh no pisa selección), `tsc` 0, `pnpm build` + `lint:design` verdes.
- Smoke E2E post-fix (dev DB real): header 1 → stock 10, header 3 → 11,
  header 1 repetido → 10 (caché no filtra), sin header → 209 (global).
  Backend del usuario relanzado en `:5050` con el binario nuevo.

## Archivos

**Backend** (`business_management`, dev): `internal/catalog/http.go`,
`internal/catalog/branchctx.go`, `internal/catalog/http_search_units_test.go`,
`internal/catalog/equiv_test.go`.

**Frontend** (`erp-webapp`, dev): `src/features/products/hooks/useProductsLogic.ts`,
`src/store/useProductStore.ts`, `src/contexts/BranchContext.tsx`,
`src/contexts/AuthContext.tsx`, `src/services/BusinessManagementAPI.ts`,
`src/services/authService.ts` + 3 tests nuevos + 4 tests legacy con mock de
`useBranch`.

## Deuda observada (no bloqueante, documentada)

- `GET /products/{id}/stock-summary`, `/total-stock`, `/api/v1/variants/{id}/stock`
  leen solo `?branch_id=` (ignoran header) — el FE les pasa el param explícito,
  funcionan, pero no uniformly header-aware.
- `GET /products/search/facets` igual (solo query param; el FE no lo envía →
  facetas globales).
- El modo `granularity=product` con variantes muestra el stock de la fila base
  del padre (preexistente), no la suma de variantes de la sucursal.
