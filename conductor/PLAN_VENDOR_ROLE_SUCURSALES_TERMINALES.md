# PLAN: Rol Vendor, Acceso a Sucursales y Terminales Vinculadas

**Fecha:** 2026-09-05
**Estado:** ✅ **PLAN COMPLETO (2026-09-06): todas las fases implementadas, incluidas F.6 y FASE E.** Las 5 decisiones abiertas de §5 quedaron resueltas según la recomendación (ver nota de cierre en §5).
**Progreso de implementación (2026-09-05):**
- ✅ **FASE A completa** (A.1-A.5) — backend commit `fe7a61c`, frontend commit `1a90f40`. Migraciones `20260905121820` (backfill) + `20260905121821` (F.0). Gates verdes: `make build`, `go test ./...`, `golangci-lint`, `vitest` 501/501, `tsc --noEmit`, `pnpm build`, `pnpm lint:design`.
- ✅ **F.0 fix de ENCR01** — mismo deploy que FASE A (grants por nombre, set Encargado completo).
- ✅ **FASE B completa** (B.1-B.5) — backend commit `395993d`, frontend commit `76f27b7`. Migración `20260905122845` (permisos 186-188 + grants F2VLso/ENCR01 + perfil VNDR01 sin reports/analytics). B.3 incluye `apperr.Forbidden` nuevo + `MapErrorToHTTP` mapeando apperr (wrappers incluidos) + puerto `sale.PermissionChecker` (bridge sobre el cache de permisos compartido) + `AllowPriceModifications` forzado en `ProcessSale` Y `ConvertToSale` (presupuestos). Nota: los permisos nuevos NO van en `config/module_permissions.go` porque no son perms de path-prefix — `sales:cancel` se aplica por-ruta (B.2) y los otros dos a nivel servicio/UI.
- ✅ **F.1/F.2/F.3 completos** — backend commit `680e163` (+ `e9e5797` constantes). Migraciones `20260905124005` (CAJA01) y `20260905124006` (DEPO01, incluye `branches:switch`). F.3: `canRole*` eliminados → `BranchService.canManageTransfers` (bypass admin por rol, resto `transfers:write` vía reader compartido, nil = fail-closed). `constants/roles.go` registra ambos roles.
- ✅ **FASE C completa** (C.1-C.3) — backend commit `794e624`, frontend commit `91d29f8`. Migración `20260905130719` (`transactions.sale_cancellation_requests`, índice parcial único por venta). Backend: entidad+puertos (`CancellationRequestRepository`, `SaleCanceller`) + `CancellationRequestService` (validación fail-fast: motivo, venta existente/no anulada, ventana fiscal SIFEN, sin pendiente duplicado; approve corre `CancelSaleWithReason` DENTRO de la tx de la aprobación vía `txm`; reject con motivo obligatorio; race de doble revisión cubierta por `WHERE status='pending'` + rowsAffected) + repo adapter + 4 rutas (crear con grupo `sales:*`, list/approve/reject + `sales:cancel` por ruta, literales antes de `/{id}`) + tests table-driven. Frontend: `RequestCancellationModal` (vendor), rama "Solicitar Anulación" en `SalesHistoryView` (dropdown + mobile), `CancellationRequestsPanel` + pestaña "Anulaciones" gated `sales:cancel` con badge de pendientes, hook `useCancellationRequests` (guard de carrera + contador con debounce), endpoints en `saleService`/`BusinessManagementAPI`, i18n es completo, 11 tests nuevos. Auditoría `vercel-react-best-practices` en `erp-webapp/conductor/AUDIT_REACT_PERFORMANCE_CANCELLATIONS_FRONTEND.md`.
- ✅ **§7 fixes menores completos** (1-3; el 4 ya venía de B.5) — backend commit `bcf2885` (migración `20260905134351` SUPLR1→SUPLR01, patrón copy-forward idempotente y FK-safe), frontend commit `b5d44ee` (`src/constants/roles.ts` espejo de `constants/roles.go`; `RoleGuard` sin map hardcodeado + i18n; `BranchSelection.tsx` pasada i18n completa con keys `branchSelection.*` en `locales/es/other.js`, error traducido en render para no meter `t` en las deps del effect).
- ✅ **FASE D completa** (D.1-D.5) — backend commit `97e4766` (ride-along D.1: puerto `ClearDefaultUserBranch` + SQL, swap clear-then-set en create/update de accesos para no chocar con `uk_user_default_branch`; tests con fake del puerto), frontend commit `6b7de78`. D.1: checkbox `is_default_branch` al asignar + badge y acción "marcar default" en la tabla de accesos (`BranchModal.tsx`, mutation `updateAccess`). D.2: sección read-only "Sucursales Asignadas" en `UserFormModal` (GET `/users/{id}/branches` + nombres) con link a Configuración → Sucursales; en create, nota del seed automático a MAIN. D.3: pantalla de emparejamiento `/configuracion/terminal` (`TerminalPairing.tsx`, gated `branches:switch`, entrada en Settings), helpers de `device.defaultBranch` centralizados en `src/utils/deviceBranch.ts`. D.4: `BranchContext` fuerza la sucursal de la terminal por encima de URL/localStorage/JWT **solo** para usuarios sin `branches:switch`; emparejamiento stale (fuera de allowed) cae a la cadena normal; app load aplica (no reacciona a eventos a mitad de sesión). D.5: `BranchSwitcher` se renderiza solo con `branches:switch` (bypass admin intacto).
- ✅ **F.4 completo** — frontend commit `e080d7a`. Feature `src/features/transfers/` (Feature-Sliced): bandeja con filtros por estado + badge de pendientes (`usePendingTransfersCount`), modal de detalle con acciones del workflow (approve/reject con motivo obligatorio, ship con tracking obligatorio, in-transit, receive — mapea las reglas de `canUserUpdateTransferStatus`), modal de creación (origen = sucursal activa fija, destino selector, búsqueda de productos con debounce 300 ms, variantes opcionales). `branchTransferService` tiene su primer consumidor y su tipo de listado corregido a la forma real `{transfers,total,page,page_size}`. Ruta `/transferencias` guarded `transfers:read` + nav bajo Logística. i18n `transfers.*` (es, en hereda). 10 tests.
- ✅ **F.5 completo** — frontend commit `67ba338`. (a) Banner "Sucursal de carga" (`PurchaseBranchBanner`) en todos los pasos del wizard, sin selector libre (modelo depósito puro). (b) `PurchaseConfirmationModal` vuelve a abrirse post-checkout (`onConfirmWizard` lo setea en `finally`) con CTA "Enviar a sucursal…" gated `transfers:write` que abre el modal de F.4 precargado con snapshot transferible de los ítems (`transferable_items` capturado en `handleSavePurchase` del closure antes de limpiar el carrito: producto/variante/cantidad/costo; origen = `branch_id` de la compra). No se encadenan `register`/`process`. 3 tests del CTA.
- ✅ **F.7 completo** — frontend commit `6b7de78` (mismo diff que FASE D). `PermissionGuard` en `/compras`+`/pagos-compras*`→`purchases:read`, `/caja-registradora`+`/movimientos-caja*`→`cash:read`, `/movimientos-stock`→`inventory:read`, `/ajustes-precios*`→`products:read`, `/configuracion/tipos-cambio`→`payments:read`.
- ✅ **F.6 completo** (2026-09-06) — backend commit `5f5f5ea`, frontend commit `f945d8e`. Migración `20260906105550` (`purchase_order_id INTEGER NULL` en `products.branch_transfer_items`, FK a `transactions.purchase_orders` ON DELETE SET NULL + índice parcial). Propagación punta a punta: request DTO → servicio (construcción de ítems) → repo INSERT/SELECT → respuesta HTTP (`omitempty`). Frontend: el snapshot del CTA F.5 estampa `purchase_order_id` en cada ítem (solo los precargados, no las líneas manuales), `CreateTransferModal` lo reenvía por ítem y `TransferDetailModal` muestra "Compra de origen: #N" con link a `/compras` cuando los ítems lo registran. Test de mapeo backend (omit/present) + 3 tests frontend.
- ✅ **FASE E completa** (2026-09-06) — backend commit `c2d30a7`, frontend commit `dd87531`. Migración `20260906110923` (`public.devices`: name, `branch_id` FK nullable, `pairing_code` UNIQUE, `is_active`, `last_seen_at/by`). Contexto nuevo `internal/devices/` (domain/port/service/dto/handler/repo + tests, patrón brand): CRUD de terminales con código de emparejamiento generado (8 chars sin ambigüedad, crypto/rand), `POST /api/v1/devices/pair` (solo auth — el código es el secreto), `TouchLastSeen`. **Middleware global `DeviceBranchMiddleware`** (registrado tras CheckAuth en `routes/routes.go`): con `X-Device-ID`, fuerza la sucursal del device (stripping `?branch_id=` + header + claim `ActiveBranch`) SOLO para usuarios sin `branches:switch` (admin bypass intacto), solo si la sucursal ∈ `allowed_branches` (stale pairing cae a la cadena normal, espejo de D.4); error de lookup de permisos degrada a "no puede cambiar" y fuerza igual (restrictivo); cada resolución toca `last_seen` (auditoría por terminal). Puerto `devices.PermissionReader` satisfecho por `PermissionCache.PermissionNames` (método nuevo en identity). Rutas admin gating `branches:read` (GET) / `branches:write` (escritura), mismo split por método que branches. Frontend: workspace `src/features/devices/` (maestro-detalle patrón printers) en `/configuracion/terminales` gated `branches:switch` + entrada en Settings; `TerminalPairing` registra la terminal contra el backend con el código (guarda `device.id` + `device.defaultBranch` como fallback) y muestra estado "Registrada (#id)"; `apiClient` envía `X-Device-ID` en todo request autenticado (`BusinessManagementAPI.getAuthHeaders`); i18n `devices.*` + `terminal.register*` (es, en hereda); ResizeObserver polyfill en `vitest.setup.ts` (Checkbox Radix). Tests: 5 casos backend middleware + service table-driven; 11 frontend nuevos.
  - **Nota de alcance:** la auditoría por-terminal de VENTAS individuales (columna `device_id` en `transactions.sales`) queda diferida a FASE 8 (migración del contexto sale): el INSERT de ventas corre por funciones SQL legacy que este plan no toca. La auditoría a nivel terminal está disponible vía `devices.last_seen_at/last_seen_by`.
- Nota D.4: la resolución por terminal corre en el mount de `BranchContext` (deps auth); re-emparejar aplica en el próximo login/carga. Auditoría `vercel-react-best-practices` de D+F.4/F.5 en `erp-webapp/conductor/AUDIT_REACT_PERFORMANCE_TRANSFERS_TERMINAL_FRONTEND.md`. Gates finales: backend `go build`/`go test ./...`/`golangci-lint` verdes; frontend tsc 0, vitest 546/546 (532 en FASE D + 3 de F.6 + 11 de FASE E), `pnpm build`, `lint:design` sin violaciones nuevas.
- Nota A.2: el seed se implementó como flag `SeedMainBranchAccess` computado por el servicio e insertado por el repo DENTRO de la TX del create (testeable con mock del puerto, como pedía el plan). A.4: nuevo puerto `BranchCacheInvalidator` satisfecho por `*PermissionCache` (método `InvalidateBranchAccess`, key `user:branches:` compartida con el middleware).
**Alcance:** Cross-repo (backend `business_management` + frontend `erp-webapp`) → este plan vive en `conductor/` raíz
**Origen:** Bug reportado por el usuario: vendor "Hernan Cortes" ve "No tienes sucursales asignadas" en `/select-branch`. La auditoría derivó hacia diseño de rol vendor, permisos de venta y vinculación terminal→sucursal. **Extensión (misma fecha):** auditoría de roles operativos (Gerente/Cajero/Encargado/Depósito) → FASE F: fix de `ENCR01` (hoy roto), alta de `CAJA01` y `DEPO01`, refactor del gate por rol de transfers y UI de transferencias.
**Verificación:** Todas las afirmaciones del análisis original fueron re-verificadas contra el código el 2026-09-05 (evidencia file:line abajo). Correcciones al análisis original en §2. **Auditoría post-implementación (2026-09-06):** todos los claims verificados contra el código y gates verdes — ver `AUDIT_VENDOR_ROLE_SUCURSALES_TERMINALES.md`.

---

## 1. Problema y contexto

Un usuario con rol VENDOR (`VNDR01`) creado después de la migración multi-sucursal queda bloqueado en la pantalla de selección de sucursal porque su lista `allowed_branches` viene vacía. La auditoría del flujo completo (login → selección de sucursal → ventas → anulación) encontró además tres huecos de control: descuentos sin gate de permiso, anulación sin supervisión, y fail-open cuando la lista de sucursales está vacía.

## 2. Correcciones al análisis original (importantes)

El análisis original era correcto en ~90%. Estas son las correcciones que cambia el plan:

| # | Afirmación original | Realidad verificada | Impacto en el plan |
|:-:|:--------------------|:--------------------|:--------------------|
| C1 | "`is_default_branch` existe y no se usa" | **Sí se usa de punta a punta**: backend lo usa para el claim `active_branch` del JWT (`internal/identity/domain.go:677-680`), para `get_user_default_branch_id()` en SQL y tiene índice único parcial; el frontend lo consume en login (`authService.ts:81-88` → localStorage `activeBranch`) y como fallback en `BranchContext.tsx:63-65` | La maquinaria de "sucursal default" ya está cableada. Lo que falta es **UI para configurarla** (C2) y el seeding |
| C2 | "Asignar Usuario tiene checkbox `is_default_branch`" | **El checkbox no existe**: está solo en el state default (`BranchModal.tsx:83-84`, siempre `false`), nunca se renderiza | Hay que construir el checkbox — sin él, nadie puede marcar una default desde la UI |
| C3 | "`allow_price_modifications` es un permiso (precedente para descuentos)" | **No es un permiso: es un boolean del body del request** (`models/sale.go:196`, `internal/sale/domain.go:188`), controlado por el cliente. Hoy la modificación de precio NO está realmente gated server-side | El gate de `sales:apply_discount` debe cubrir **también** `allow_price_modifications`/precios custom, forzándolo a `false` server-side sin el permiso |
| C4 | Perfil VNDR01 = lista de 15 permisos | La lista real es un superconjunto: también tiene `payments:read` (135), `documents:read/write` (141-142), `clients:read/write` (143-144), `settings:read` y legacy `1/5/6` | El perfil vendor (§4.1) parte del set real; `clients:write` ya está concedido (el vendor ya puede crear clientes) |
| C5 | "Dos strings hardcodeadas en BranchSelection.tsx" | **El archivo entero viola i18n**: no importa `useI18n` en absoluto — todos los strings son español hardcodeado | La tarea i18n es una pasada completa al archivo, no 2 strings |
| C6 | "ProtectedRoute redirige si no hay sucursal seleccionada" | Redirige solo si: `allowedBranches.length > 1` AND sin localStorage AND usuario no-admin (`App.tsx:154-159`; admin = `admin`/`F2VLso` se salta el selector) | Matiz documentado; no cambia tareas |

Confirmados sin cambios: backfill a MAIN solo para usuarios pre-migración; `POST /api/v1/users` no crea accesos ni hay trigger; Redis `user:branches:<id>` TTL 5 min jamás invalidado (`middleware/auth.go:19,184,208`; grant/revoke en `internal/identity/service.go:1651,1706,1744` sin Redis); fail-open en 3 puntos (`branchctx.go:193-196,265-267`, `http_branch.go:1371-1374`); 403 out-of-list para no-admin (`branchctx.go:226-231`, bypass admin `:182-191`); `PUT /sale/{id}` anulable por cualquier `sales:write` sin check de propiedad (`internal/sale/http/routes.go:49,67`, `database/postgres/sale.go:46-50` → `transactions.revert_sale`; SIFEN `sale_service.go:277-287,311-321`); descuento solo exige `discount_reason` en SQL (`20260826140732_iva_debt1_sale_create_stock_branch.up.sql:319-322`); `SUPLR1` (migración) vs `SUPLR01` (`constants/roles.go:21`); `printers.branch_id` como patrón para devices (`20260814202936_create_documents_schema_and_printers.up.sql:11-13`); los 3 permisos nuevos no existen en ningún repo.

**Hallazgo nuevo (no estaba en el análisis original):** botón compacto/móvil de anular en `SalesHistoryView.tsx:328` visible sin `canWrite` (el del dropdown en `:264` sí está gated). Fix menor incluido en §7.

## 3. Estado actual verificado (referencia rápida)

- **Login/branch:** login → siempre `/select-branch` (`Login.tsx:37`). Resolución de sucursal activa (`BranchContext.tsx:57-73`): URL `?branch_id=` → localStorage `activeBranch` → JWT `active_branch` → auto-select si 1 sola → elección manual. El header lleva `X-Branch-ID`; backend 403 si fuera de `allowed_branches` (no-admin, lista no vacía).
- **Frontend gating:** patrón estándar `hasPermission` (`AuthContext.tsx:39-45`, bypass admin) + `PermissionGuard` (`components/auth/PermissionGuard.tsx`). `BranchSwitcher` en header sin gate de permiso, se auto-oculta si ≤1 sucursal (`BranchSwitcher.tsx:46-48`, `Header.tsx:115`).
- **Endpoint existente reutilizable:** `GET /users/{user_id}/branches` (`internal/identity/http/routes.go:69`, requiere `branches:read`) — para la sección de sucursales en el formulario de usuario.
- **Rutas de acceso a sucursales:** POST/GET `/branches/{id}/access`, PUT/DELETE `/branches/{id}/access/{user_id}` (`routes.go:65-68`).

## 4. Decisiones adoptadas

### 4.1 Perfil vendor (vendedor de mostrador)

Cambios de **datos** (migración), no de código:

| Permiso | Hoy VNDR01 | Destino | Motivo |
|:--------|:----------:|:-------:|:-------|
| `sales:read/write`, `cash:read/write`, `reserves:*`, `budgets:*`, `parties:read`, `clients:read/write`, `documents:read/write`, `payments:read`, `products:read`, `branches:read`, `settings:read` | ✅ | ✅ mantener | Trabajo del mostrador: vender, cotizar, caja de su turno, crear/consultar clientes, imprimir. `settings:read` queda bajo revisión post-implementación (verificar qué pantallas desbloquea) |
| `reports:read`, `analytics:read` | ✅ | ❌ quitar | No hay razón operativa: BI, márgenes y reportes financieros no son para vendedores |
| `dashboard:read`, `receivables:read` | ✅ | ✅ mantener* | *Decisión abierta §5.1 — no bloquea nada, se ajusta por datos |
| `sales:apply_discount`, `sales:cancel`, `branches:switch` | no existen | ❌ para VNDR01 | Nuevos permisos (FASE B); el vendor no descuenta, no anula, no elige sucursal |

### 4.2 Descuentos y precios modificados
Permiso nuevo `sales:apply_discount` (binario en esta iteración). Validación **server-side** en `internal/sale`: si algún ítem trae descuento o precio custom y el usuario no tiene el permiso → error tipado 403; además se fuerza `allow_price_modifications=false` cuando falta el permiso (corrige C3: hoy ese flag lo manda el cliente sin control).

### 4.3 Anulación con supervisión
Permiso nuevo `sales:cancel` solo para roles avanzados (`F2VLso`, `ENCR01`). `PUT /sale/{id}` pasa a exigirlo. El vendor solicita la anulación vía tabla `sale_cancellation_requests`; el approve ejecuta el `CancelSale` existente (así la validación/notify fiscal SIFEN corre en el momento de aprobación). Encaja en FASE 8 (contexto `internal/sale`).

### 4.4 Terminal vinculada a sucursal
- **Nivel 1 (esta iteración, frontend-only):** clave `device.defaultBranch` en localStorage, seteada una vez por un encargado en una pantalla de emparejamiento. Si existe y el usuario NO tiene `branches:switch`, se fuerza esa sucursal (por encima de localStorage de sesión y JWT), se saltea `/select-branch` y no se muestra `BranchSwitcher`.
- **Nivel 2 (futuro, §6 E):** tabla `devices` en backend con `branch_id` (patrón `documents.printers`), header `X-Device-ID`, server resuelve sucursal del device salvo `branches:switch` + header explícito.

### 4.5 Accesos confiables
- Seed de `user_branch_access` (sucursal MAIN, `FULL`, `is_default_branch=true`) al crear usuario no-CLIENT, en la misma TX del create.
- Backfill por migración para usuarios existentes sin filas.
- Fail-closed en los 3 puntos de `branchctx.go`/`http_branch.go`.
- Invalidación de `user:branches:<id>` en Redis al crear/actualizar/borrar accesos.

### 4.6 Roles operativos (Gerente, Cajero, Encargado, Depósito) — extensión

- **Gerente = `F2VLso`**: ya existe con todos los permisos + bypass por claim de rol en el middleware (`middleware/module_permission.go:40-44`). No se crea nada.
- **Cajero = rol nuevo `CAJA01`**: mostrador de plata — ventas/cobros, clientes, proveedores (lectura), tipo de cambio, caja, pago de compras desde caja. Diferencia con `VNDR01`: el vendedor hace presupuestos/reservas; el cajero opera dinero (tipo de cambio vía `payments:write`, pago de compras desde caja vía `cash:write`).
- **Encargado = fix de `ENCR01`**: hoy ROTO — se sembró (2026-08-12) solo con permisos legacy (IDs 1,2,3,5,6,7,8,9) + settings, y el middleware moderno nunca chequea esos nombres → un encargado recibe 403 en ventas, productos, stock, compras, todo. Se le concede el set completo: Cajero + compras + inventario + precios/productos + reservas + usuarios + transfers.
- **Depósito = rol nuevo `DEPO01`** (nombre definido en este plan, a pedido del usuario): compras + pago de compras + productos + precios + proveedores + inventario + transfers; **sin caja, sin ventas, sin usuarios**; acceso multi-sucursal.
- **Modelo depósito puro para compras**: la compra carga stock al depósito (sucursal activa del operador) y de ahí sale por **transferencia** a la sucursal destino. Las transferencias YA existen en backend end-to-end (`products.branch_transfers`, workflow `PENDING→APPROVED→SHIPPED→IN_TRANSIT→RECEIVED` con movimientos pareados `TRANSFER_OUT`/`TRANSFER_IN` por sucursal, `20260402203000` + `20260402212000`); lo que falta es UI y ajustes de permisos (FASE F). No se agrega selector libre de sucursal en el formulario de compras (permitiría saltarse el depósito).

## 5. Decisiones abiertas

> **Cierre (2026-09-06):** las 5 se resolvieron según la recomendación al implementar. 5.1: VNDR01 mantiene `dashboard:read` (B.1 solo quitó `reports:read`/`analytics:read`). 5.2: descuento binario (`sales:apply_discount`). 5.3: sí — `F2VLso` y `ENCR01` recibieron los tres permisos (migración `20260905122845`). 5.4: pago de compras solo desde caja (CAJA01 quedó sin `purchases:*`). 5.5: tipo de cambio cubierto por `payments:write` (CAJA01 lo tiene).

| # | Pregunta | Recomendación | Qué bloquea |
|:-:|:---------|:--------------|:------------|
| 5.1 | ¿Vendor ve `dashboard:read`? | Mantener por ahora (home útil del POS); si se quiere acotar, hacer después un dashboard por rol — no mezclarlo con este plan | Nada (ajuste de datos reversible) |
| 5.2 | ¿Descuento binario o con tope `%`? | Binario primero (`sales:apply_discount`); el tope `max_discount_percent` queda como iteración 2 si surge el rol "supervisador con tope" | Solo el diseño fino de B.3 — se implementa binario |
| 5.3 | ¿`ENCR01` recibe `sales:cancel` + `branches:switch`? | Sí (es "rol avanzado" según el usuario). Confirmar en implementación | Asignación de grants en la migración de B.1 |
| 5.4 | ¿Cajero paga compras **fuera** de caja (módulo `/pagos-compras`)? | Mantener solo pago desde caja: `cash:write` ya cubre `POST /cash-registers/payments/purchase`; si se necesita pago sin caja, crear permiso fino `purchases:pay` (id 186) en vez de dar `purchases:write` (que también dejaría registrar compras) | Set final de F.1 |
| 5.5 | ¿Tipo de cambio con permiso propio? | Queda `payments:write` por ahora (hoy lo tiene BUYR01); si fijar la tasa debe ser exclusivo de Cajero/Gerente, iteración 2 con `currency:write` (id 187) | Nada (reversible por datos) |

## 6. Fases

> Reglas AGENTS.md: migraciones SOLO via `make migrate-create`, jamás modificar existentes; handlers sin lógica de negocio; tests obligatorios; depguard activo en `internal/`; i18n siempre; cambios de API backend → actualizar frontend en el mismo diff; pnpm exclusivo.

### FASE A — P0: acceso a sucursales confiable (backend, sola)

**A.1 Migración backfill.** Usuarios sin filas en `users.user_branch_access` → INSERT a MAIN `FULL` `is_default_branch=true` (mismo patrón que `20260402153000:129-136`). Criterio: tras aplicar, `SELECT count(*) FROM users.users u WHERE NOT EXISTS (SELECT 1 FROM users.user_branch_access a WHERE a.user_id=u.id)` = 0.

**A.2 Seed en CreateUser.** `internal/identity/service.go:805-838` (`CreateUser`): si el rol asignado ≠ `CLNT01`, sembrar acceso a MAIN (`FULL`, default) en la misma transacción del create. Tests en `service_test.go` (mock del puerto de accesos). El admin puede reasignar después desde Configuración → Sucursales.

**A.3 Fail-closed.** Tres puntos:
- `internal/identity/branchctx.go:193-196`: lista vacía + branch solicitada → 403 (hoy pasa sin validar);
- `branchctx.go:265-267`: `HasBranchAccess` → `false` con lista vacía (hoy `true`);
- `internal/identity/http_branch.go:1371-1374`: `filterBranchesByClaims` → lista vacía para no-admin sin accesos (hoy devuelve TODAS).
El bypass de admin (`branchctx.go:182-191`, por rol, no por lista) queda intacto — los admins no dependen de la lista. Con A.1+A.2 deployados antes, ningún usuario legítimo queda con lista vacía.

**A.4 Invalidación Redis.** `BranchService` (`internal/identity/service.go:1253-1257`) no tiene dependencia Redis → inyectarla vía `wiring/` y hacer `Del("user:branches:<userID>")` en `CreateUserBranchAccess`/`UpdateUserBranchAccess`/`DeleteUserBranchAccess` (`service.go:1651,1706,1744`).

**A.5 Hardening frontend (ride-along).** `BranchContext.tsx:61-62`: validar `savedBranch ∈ finalAllowedBranches` antes de usarlo (hoy un localStorage stale de una sucursal revocada se setea igual y todos los requests caen en 403).

**Gates:** `make build` + `go test ./...` + `golangci-lint run` en verde.

### FASE B — P1: permisos de venta (backend + frontend)

**B.1 Migración de permisos.** Alta de `sales:apply_discount`, `sales:cancel`, `branches:switch` como permisos en DB (ids 186-188) + grants: `F2VLso` y `ENCR01` reciben los tres; `VNDR01` ninguno. (Si a LNCR01/BUYR01 les corresponde `sales:cancel`, definirlo acá — ver 5.3.) **Nota de implementación:** NO se registran en `config/module_permissions.go` porque no son perms de path-prefix — `sales:cancel` se aplica por-ruta (B.2) y los otros dos a nivel servicio/UI.

**B.2 Gate de anulación.** Ruta `PUT /sale/{id}` (`internal/sale/http/routes.go:67`): agregar requisito `sales:cancel` además del group middleware existente (`sales:read`+`sales:write`, `routes.go:49`). Hoy el grupo deja pasar a cualquier `sales:write` — el vendor incluido.

**B.3 Gate de descuento/precio server-side.** En `internal/sale/sale_service.go` (`ValidateProductsForSale`, ~`:159-260`): obtener claims/permisos del contexto del request; si algún detalle tiene `discount_percent`/`discount_amount` o precio custom (`sale_price` ≠ precio de lista) y el usuario carece de `sales:apply_discount` → error tipado (apperr) 403. Además forzar `AllowPriceModifications=false` sin el permiso (corrige C3). Tests table-driven con el puerto de permisos moqueado.

**B.4 Frontend descuento.** `SalesNew.tsx` calcula `canApplyDiscount = hasPermission('sales:apply_discount')` y lo baja a `EditItemModal` (`features/sales/components/EditItemModal.tsx`, líneas de descuento `:244-294` y razón `:296-321`): sin permiso, los inputs de precio/descuento deshabilitados con hint (candado) — es cosmético, la validez real es B.3. i18n keys nuevas en `locales/es` + `locales/en`.

**B.5 Frontend anulación (transición).** `SalesHistoryView.tsx` y `CancelSaleModal.tsx`: el botón "Anular Venta" (dropdown `:264`, confirm `:55`) solo visible/habilitado con `sales:cancel`. Fix del botón compacto móvil (`:328`) que hoy no chequea `canWrite` (hallazgo nuevo §2). El botón "Solicitar anulación" del vendor llega en FASE C.

**Gates:** backend ídem A + frontend `npx vitest --run` 0 fallos, `npx tsc --noEmit` 0 errores, `pnpm build`, `pnpm lint:design`.

### FASE C — P2: flujo de solicitud de anulación (backend + frontend; coordinar con FASE 8 de `internal/sale`)

**C.1 Migración.** Tabla `transactions.sale_cancellation_requests`: `id`, `sale_id` (FK), `requested_by`, `reason` (obligatorio), `status` `pending|approved|rejected`, `reviewed_by`, `reviewed_at`, `created_at`. Índice parcial único: una solicitud `pending` por venta.

**C.2 Backend (contexto `internal/sale`, arquitectura hexagonal del repo).** `domain.go`/`port.go`/`service.go`/`repository.go`/`handler.go`:
- `POST /sale/{id}/cancellation-requests` — permiso `sales:write`; valida venta anulable y estado fiscal (`fiscalCancel.ValidateCancellation`) al crear la solicitud (fail-fast);
- `GET /sale/cancellation-requests?status=pending` — permiso `sales:cancel`;
- `POST /sale/cancellation-requests/{id}/approve` — permiso `sales:cancel`; llama a `CancelSaleWithReason` (variante justificada del `CancelSale` existente; SIFEN + `revert_sale` corren acá) y marca approved en la misma UoW (`txm`);
- `POST /sale/cancellation-requests/{id}/reject` — permiso `sales:cancel` + razón de rechazo.

**C.3 Frontend.** Vendor: botón "Solicitar Anulación" en `SalesHistoryView` (rama por permiso: `sales:cancel` → anular directo; si no → solicitar) + modal de razón. Aprobador: inbox de solicitudes pendientes (badge de conteo; sección dentro de Ventas). i18n completo es/en. Tests por `PLAN_TEST_DESIGN_FRONTEND.md`.

### FASE D — P3: sucursal default configurable + terminal vinculada (mostly frontend)

**D.1 Checkbox default en la UI (corrige C2).** `BranchModal.tsx` pestaña ACCESOS: render del checkbox `is_default_branch` (el state ya existe en `:83-84`) al asignar/editar acceso. Al marcar, el backend ya respeta el índice único (`uk_user_default_branch`).

**D.2 Sucursales en el formulario de usuario.** `UserFormModal.tsx`: sección de solo lectura "Sucursales asignadas" consumiendo `GET /users/{user_id}/branches` (ya existe, `routes.go:69`) + link a Configuración → Sucursales (fuente de verdad única: no duplicar la asignación en dos pantallas).

**D.3 Emparejamiento de terminal.** Pantalla one-time (accesible para `branches:switch`): "Esta terminal opera la sucursal X" → guarda `localStorage['device.defaultBranch']`. Entrada: Configuración → Terminal (y/o interceptor en primer arranque si no existe la clave).

**D.4 Resolución en BranchContext.** Nueva precedencia **si el usuario no tiene `branches:switch`**: `device.defaultBranch` (si ∈ allowed) > todo lo demás; saltear `/select-branch` por completo (extender la condición de `App.tsx:154-159`). Con `branches:switch`: jerarquía actual (URL → localStorage → JWT → única → elección).

**D.5 BranchSwitcher gated.** `Header.tsx:115` / `BranchSwitcher.tsx`: render solo con `branches:switch` (mantiene el auto-ocultar con ≤1 sucursal).

### FASE F — Roles operativos + transferencias (backend + frontend; extensión 2026-09-05)

> Contexto verificado: roles sembrados en `20260212123000` (F2VLso, BUYR01, CLNT01, SUPL01, VNDR01), `20260519112753` (SUPLR1) y `20260812151909` (ENCR01). Existe CRUD de roles en runtime (`/api/v1/roles` + `/roles/{id}/permissions`, `internal/identity/http/routes.go:129-143`) pero sin UI y generando IDs shortid — los roles operativos se siembran **por migración** para tener IDs fijos y entornos reproducibles. IDs de permiso libres: **186+** (147-181 reservados a seeds gitignored; 182-183 sifen, 184-185 settings).

**F.0 Fix de ENCR01 (migración — urgente, hoy el rol está roto).** `ENCR01` solo tiene permisos legacy + settings (`20260812151909:16-25`); `RequireModulePermission` nunca consulta esos nombres → 403 en todo. Migración con el set Encargado = set Cajero de F.1 **más**: `purchases:read/write` (110/111), `inventory:read/write` (112/113), `products:write` (107 — cubre precios), `reserves:read/write` (123/124), `users:read` (100) + `users:write` (140), `transfers:read/write` (129/130), `suppliers:write` (146). (`reports:read`/`analytics:read` fuera según spec; agregar por datos si se pide.) Comunicar en el deploy: los usuarios ENCR01 existentes pasan de bloqueados a operativos.

**F.1 Alta de CAJA01 'CAJERO' (migración).** `INSERT INTO users.roles (id, name) VALUES ('CAJA01','CAJERO')` + grants: `products:read` (106), `sales:read/write` (108/109), `dashboard:read` (114), `receivables:read` (115), `parties:read` (119), `clients:read/write` (143/144), `suppliers:read` (145), `cash:read/write` (121/122), `payments:read` (135) + `payments:write` (136 — fijar tipo de cambio: `/exchange-rates` es `payments:*`), `branches:read` (131), `documents:read/write` (141/142 — imprimir), `settings:read` (184). **Sin** `reserves/budgets` (del vendedor/encargado), sin `reports/analytics` (misma regla que §4.1), sin `purchases:*` (pago de compras desde caja cubierto por `cash:write` en `POST /cash-registers/payments/purchase`, `internal/sale/http/routes.go:96`; pago fuera de caja → decisión 5.4).

**F.2 Alta de DEPO01 'DEPOSITO' (migración).** `('DEPO01','DEPOSITO')` + grants: `products:read/write` (106/107), `purchases:read/write` (110/111 — incluye `POST /purchase/payment/process`, pago de compras), `suppliers:read/write` (145/146) + `parties:read` (119), `inventory:read/write` (112/113), `transfers:read/write` (129/130), `branches:read` (131), `payments:read` (135), `dashboard:read` (114), `settings:read` (184), `documents:read/write` (141/142 — remitos, junto a F.4). **Sin** `cash:*`, `sales:*`, `users:*`, `payments:write` (no fija tipo de cambio). Multi-sucursal: filas en `user_branch_access` (depósito + destinos) + `branches:switch` (FASE B.1) o terminal vinculada (FASE D). La sucursal depósito se marca con `branch_type='WAREHOUSE'` (o `DISTRIBUTION_CENTER`), `allows_purchases=true`, `allows_sales=false` — columnas que ya existen en `public.branches`.

**F.3 Refactor del gate por rol en transfers (código).** `internal/identity/service.go:1314-1352`: `canRoleCreateTransfer` (Admin/Inventory/Buyer) y `canRoleUpdateTransferStatus` (Admin/Inventory) chequean IDs de rol hardcodeados → `DEPO01` o cualquier rol runtime sería rechazado 403 aunque tenga `transfers:write`. Cambiar a chequeo por permiso (`transfers:write` desde los claims del contexto). Conservar las reglas de negocio por sucursal ya implementadas (ship exige acceso writable a la sucursal origen; receive a la destino). Tests table-driven: rol con permiso pasa; sin permiso → 403.

**F.4 UI de transferencias (frontend — la build grande).** `branchTransferService.ts` está hoy huérfano (cero consumidores). Nueva feature `src/features/transfers/` (Feature-Sliced): bandeja con estados y badge de pendientes, creación (origen = sucursal activa, destino selector, ítems producto+variante+cantidad), acciones approve/ship/receive gated con `PermissionGuard permission="transfers:write"`. Ruta `/transferencias` guarded `transfers:read` + ítem de nav. i18n es/en completo; tests según `PLAN_TEST_DESIGN_FRONTEND.md`; skill `vercel-react-best-practices` con reporte en `conductor/` del frontend.

**F.5 Compras → sucursal destino (modelo depósito puro, §4.6).** Sin selector libre de sucursal en el formulario. En su lugar: (a) el wizard de compras muestra la sucursal destino destacada desde el inicio (hoy solo aparece read-only en la confirmación, `PurchaseConfirmationModal.tsx:81-86`); (b) post-compra, CTA "Enviar a sucursal…" que abre la transferencia de F.4 precargada con los ítems de esa compra. Usar el mismo endpoint que usa hoy el wizard — `register_purchase_order` y `process_complete_purchase_order` son caminos **alternativos** que cada uno suma stock; no encadenarlos.

**F.6 (opcional) Trazabilidad compra→transferencia — ✅ IMPLEMENTADA (2026-09-06, commits backend `5f5f5ea` / frontend `f945d8e`).** Migración: `purchase_order_id` en `products.branch_transfer_items` (ya tiene `unit_cost`) + link en la UI de F.5(b) para auditar "estos ítems salieron de la compra X".

**F.7 Guardas de rutas frontend (deuda colateral).** Rutas hoy sin guarda (el backend 403-ea igual, pero la UI entera se muestra a cualquier usuario logueado): `/compras` + `/pagos-compras` → `purchases:read`; `/caja-registradora` + `/movimientos-caja` → `cash:read`; `/movimientos-stock` → `inventory:read`; `/ajustes-precios` → `products:read`; `/configuracion/tipos-cambio` → `payments:read`. Wrapper `PermissionGuard` en `App.tsx`, mismo patrón que las rutas ya guarded.

**Gates:** backend `make build` + `go test ./...` + `golangci-lint run`; frontend `npx vitest --run` 0 fallos, `npx tsc --noEmit`, `pnpm build`, `pnpm lint:design`.

### FASE E — P4 (futuro, no calendarizada): registro de terminales en backend — ✅ IMPLEMENTADA (2026-09-06, commits backend `c2d30a7` / frontend `dd87531`)

Tabla `devices` (id, nombre, `branch_id` — patrón `documents.printers`, `20260814202936:11-13` —, código de emparejamiento, activo), header `X-Device-ID`, middleware resuelve sucursal del device salvo `branches:switch` + header explícito; audita qué terminal hizo cada venta. Implementación: contexto `internal/devices/` + middleware global `DeviceBranchMiddleware` + UI en `/configuracion/terminales` (detalle arriba en el progreso). La auditoría de ventas individuales queda diferida a FASE 8 (INSERT legacy de ventas); hoy la auditoría es por-terminal (`last_seen_at/by`). Dispara cuando: haya suficientes terminales para gestionar, se borre caché del navegador con frecuencia (nivel 1 se pierde), o se necesite auditoría por terminal.

## 7. Fixes menores (ride-along, cualquier fase)

1. **SUPLR1 vs SUPLR01:** migración `UPDATE users.roles SET id='SUPLR01' WHERE id='SUPLR1'` (+ FKs de asignaciones) para alinear datos con `constants/roles.go:21`.
2. **RoleGuard.tsx:52-58:** reemplazar map hardcodeado (`VENDEDOR`, `GESTOR` no matchean nada en el sistema) por IDs reales (`VNDR01`, `ENCR01`, `F2VLso`); idealmente crear `src/constants/roles.ts` único. Verificar IDs reales en DB antes de hardcodear.
3. **BranchSelection.tsx:** pasada i18n completa (hoy TODO el archivo está hardcodeado, sin `useI18n`), keys en `locales/es` + `locales/en`.
4. **SalesHistoryView.tsx:328:** gating `canWrite` del botón compacto (incluido en B.5).

## 8. Orden de ejecución y dependencias

```
A (P0, backend solo — desbloquea vendors YA)
 └─> B (P1, permisos — B.1 es prereq de B.2-B.5, C, D.3-D.5)
      ├─> C (P2, solicitudes de anulación — coordinar FASE 8)
      ├─> D (P3, terminal — D.1/D.2 independientes de B, pueden adelantarse)
      └─> F (P2/P3, roles operativos — F.0 fix de ENCR01 es urgente e independiente:
             puede correr en paralelo a A/B; F.1/F.2/F.3 tras B.1; F.4/F.5 tras F.3;
             el uso multi-sucursal de DEPO01 requiere A + B.1)
Nivel 2 (E) post-D/F, cuando la operación lo pida.
```

Deploy seguro de A: migración primero (A.1) → backend (A.2-A.4) → frontend (A.5). Así nunca convive "fail-closed" con "usuarios sin filas".

## 9. Riesgos

| Riesgo | Mitigación |
|:-------|:-----------|
| JWTs vigentes sin los permisos nuevos | Refresh de token re-resuelve claims (`internal/identity/token.go:149,176`); si aun así, re-login. Comunicar en el deploy |
| Fail-closed rompe a algún usuario legítimo con lista vacía | Orden §8: backfill A.1 y seed A.2 ANTES de A.3 |
| Encargado pierde anulación al cambiar el permiso de `PUT /sale/{id}` | B.1 asigna `sales:cancel` a F2VLso+ENCR01 en el mismo deploy |
| `device.defaultBranch` se borra (limpieza de navegador) | Nivel 1: re-emparejar (pantalla D.3 sigue accesible). Nivel 2 (E) lo elimina de raíz |
| Fix de ENCR01 "desbloquea" usuarios que hoy reciben 403 en todo | Intencional; revisar qué usuarios tienen el rol asignado antes del deploy y comunicarlo |
| Transferencia enviada a sucursal equivocada | El workflow exige receive explícito en destino; revertir con transferencia de vuelta (tipo RETURN) |
| Doble carga de stock en compras del depósito | `register_purchase_order` y `process_complete_purchase_order` cada uno suma stock (son caminos alternativos, no secuencia); F.5 usa el mismo endpoint que ya usa el wizard — no encadenar |
| Rollback | Migraciones nuevas tienen `.down.sql`; no se modifican existentes |
