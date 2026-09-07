# AUDITORÍA: PLAN_VENDOR_ROLE_SUCURSALES_TERMINALES.md

**Fecha:** 2026-09-06
**Sujetos auditados:** backend `business_management` @ `c2d30a7` (dev) + frontend `erp-webapp` @ `dd87531` (dev), árboles limpios salvo `bin/business_management` (artefacto de build preexistente).
**Veredicto: el plan es fiel a la realidad.** La afirmación "PLAN COMPLETO (2026-09-06)" se sostiene: 18/18 commits citados existen, 9/9 migraciones existen con `.down.sql`, ~40 claims de código backend/frontend verificados con evidencia file:line, y los 4 gates siguen verdes sobre los HEADs que el plan declara como finales. **No se encontró ningún claim falso.**

---

## 1. Gates verificados (2026-09-06, sobre los HEADs citados)

| Gate | Resultado |
|:--|:--|
| `go build ./...` + `go test ./...` | ✅ exit 0 |
| `golangci-lint run` | ✅ exit 0 |
| `npx tsc --noEmit` | ✅ 0 errores |
| `npx vitest --run` | ✅ 546/546 (77 archivos) |

## 2. Claims verificados (evidencia destacada)

- **Commits:** los 9 commits backend (`fe7a61c`, `395993d`, `680e163`, `e9e5797`, `794e624`, `bcf2885`, `97e4766`, `5f5f5ea`, `c2d30a7`) y 9 frontend (`1a90f40`, `76f27b7`, `91d29f8`, `b5d44ee`, `6b7de78`, `e080d7a`, `67ba338`, `f945d8e`, `dd87531`) existen; los HEADs de `dev` son los finales del plan.
- **Migraciones:** `20260905121820`→`20260906110923` completas. Spot-checks: B.1 (`20260905122845`) crea perms 186-188, otorga a F2VLso/ENCR01 (comentario cita decisión 5.3) y quita `reports/analytics` a VNDR01; F.2 (`20260905124006`) da `branches:switch` a DEPO01.
- **FASE A:** fail-closed en `internal/identity/branchctx.go:193,272` + `http_branch.go:1375` (antes devolvía TODAS las sucursales con lista vacía); seed `SeedMainBranchAccess` (`identity/service.go:835,850`); `InvalidateBranchAccess` (`service.go:1243`, `port.go:156`, con test propio).
- **FASE B:** puerto `PermissionChecker` (`sale_service.go:31,66` + `budget_service.go:23` — cubre `ProcessSale` Y `ConvertToSale`); `AllowPriceModifications=false` forzado (`sale_service.go:121`); `MapErrorToHTTP` en `internal/sale/error_mapper.go` + tests; rutas `sales:cancel` por-ruta con literales antes de `/{id}` (`internal/sale/http/routes.go:73-81`); `apperr.Forbidden` (`internal/shared/apperr/errors.go:21-49`).
- **FASE C:** approve corre `CancelSaleWithReason` DENTRO de la tx (`cancellation_request_service.go:133`); `RequestCancellationModal`, `CancellationRequestsPanel`, `useCancellationRequests`, rama "Solicitar Anulación" en `SalesHistoryView.tsx:281,361`.
- **FASE D:** rutas `/configuracion/terminal` y `/configuracion/terminales` gated `branches:switch` (`App.tsx:483-493`); forcing de `device.defaultBranch` en `BranchContext.tsx:62-69`; `BranchSwitcher` auto-gateado (`BranchSwitcher.tsx:51-52`, bypass admin conservado); puerto `ClearDefaultUserBranch` (`identity/port.go:181-184` + tests).
- **F.0-F.3:** `constants/roles.go:15,21` (CAJA01/DEPO01); `canRole*` eliminados → `canManageTransfers` (`identity/service.go:1330`, confirmado en comentario `:1427`); rename SUPLR01 (`20260905134351`).
- **F.4/F.5/F.6:** feature `src/features/transfers/` completa (3 componentes + `useBranchTransfers` con `usePendingTransfersCount`); `PurchaseBranchBanner` en el wizard; CTA post-compra gated `transfers:write` + `transferable_items` (`PurchaseConfirmationModal.tsx:54-59`); "Compra de origen #N" con link a `/compras` (`TransferDetailModal.tsx:115-186`).
- **F.7:** las 7 rutas citadas wrapped con los permisos exactos (`App.tsx:407` compras/purchases:read, `:420-444` caja/cash:read, `:444-452` pagos-compras/purchases:read, `:380` movimientos-stock/inventory:read, `:336-371` ajustes-precios/products:read, `:553` tipos-cambio/payments:read).
- **FASE E:** contexto `internal/devices/` completo + repo en `internal/platform/postgres/device_repo.go`; middleware registrado DESPUÉS de `CheckAuth` (`routes/routes.go:56`→`:69`); stripga `?branch_id=` y lee `X-Device-ID` (`devicebranch.go:53,106-109`); `TouchLastSeen` (`service.go:137,146`); `PermissionNames` nuevo (`identity/service.go:1253`); header en `BusinessManagementAPI.ts:121`; `TerminalPairing` registra contra backend y persiste `device.id`; polyfill ResizeObserver en `vitest.setup.ts`.

## 3. Fixes documentales aplicados en esta auditoría (2026-09-06)

1. **Línea 19 (bullet contradictorio):** eliminado el leftover "⏸️ F.6 (opcional) diferido — COMPLETADO".
2. **Nota de gates finales:** "vitest 532/532" → "546/546" con la aritmética aclarada (532 FASE D + 3 F.6 + 11 FASE E).
3. **Header de estado:** actualizado a "PLAN COMPLETO"; añadida nota de cierre de §5 con la resolución de las 5 decisiones.
4. **B.1 (§6):** corregido el cuerpo — los permisos NO van en `config/module_permissions.go` (no son path-prefix); desviación ahora documentada en la fase, no solo en el progreso.
5. **C.2 (§6):** "llama al `CancelSale` existente" → "`CancelSaleWithReason` (variante justificada…)".
6. **Typo §5.1:** eliminado el carácter CJK intruso ("no混 mezclarlo").

## 4. Observación de diseño pendiente de decisión — gates de Terminales (FASE E)

**Estado actual (verificado):**
- Frontend: ruta `/configuracion/terminales` y entrada en Settings gated `branches:switch` (`App.tsx:489-493`, `Settings.tsx:307-308`). La página NO tiene checks internos: los botones de escritura se renderizan para todo el que llega.
- Backend: GET gated `branches:read`, escrituras gated `branches:write` (`internal/devices/http/routes.go:35`); `POST /devices/pair` solo auth (el código es el secreto).
- Roles: `ENCR01` y `DEPO01` tienen `branches:switch` (+ `branches:read`) pero **ningún rol operativo tiene `branches:write`** (0 matches en las 3 migraciones de roles) — solo F2VLso (bypass/total).

**Consecuencia:** Encargado y Depósito ven la pantalla de Terminales desde Settings, pueden listar, pero toda escritura (crear/editar/desactivar) cae en 403 de API con error visible, no en un gate UI. La pantalla hermana Sucursales es admin-only por rol (`RoleGuard allowedRoles={['F2VLso']}`, `App.tsx:477-481`).

**Opciones:**
- **(a) Alinear con Sucursales:** `RoleGuard F2VLso` en la ruta + entrada de Settings → admin-only, consistente con su sibling. Mínimo cambio, cero migraciones.
- **(b) Gate por permiso `branches:write`:** mismo efecto práctico que (a) hoy, pero permission-based (más flexible si mañana alguien más debe administrar sucursales completas).
- **(c) Permiso fino `devices:write` (id 189):** permite delegar la gestión de terminales (p. ej. al Encargado) sin desbloquear el CRUD de sucursales. Requiere migración + cambio de gating backend (routes) + frontend. Es la opción correcta si la operación quiere que el encargado administre terminales.

**Nota:** la pantalla de emparejamiento `/configuracion/terminal` (D.3, gated `branches:switch`) es COHERENTE como está: es la que el Encargado usa físicamente en el POS para vincular la terminal; el endpoint `pair` es auth-only por diseño.

**✅ DECISIÓN (2026-09-06): opción (b)** — elegida por el usuario por consistencia con el patrón de permisos del sistema. Implementada en frontend `erp-webapp@dev d4cf88c`: guard de ruta `/configuracion/terminales` (`App.tsx`) y entrada en Settings (`Settings.tsx`) pasan a `branches:write`; comentario de cabecera de `DevicesPage.tsx` actualizado; pantalla de emparejamiento D.3 intacta. Gates verdes tras el cambio (vitest 546/546, tsc 0, build, lint:design limpio). Efecto: la administración de terminales queda solo para roles con `branches:write` (hoy, admin) — alineada con el split por método del backend; ENCR01/DEPO01 dejan de ver la pantalla con botones que caían en 403.

## 5. Nota de higiene (fuera de alcance del plan)

`bin/business_management` (binario de build) está trackeado en git y sucio en el working tree del backend. Convendría gitignoringarlo.

## 6. Registro

- Repos backend/frontend: sin cambios de código en esta auditoría (solo lectura + gates).
- Esta auditoría y los fixes documentales viven en `conductor/` raíz (repo wrapper del workspace, subrepos excluidos por `.gitignore`).
