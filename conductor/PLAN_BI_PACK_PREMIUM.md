# PLAN — BI Pack Premium (licenciamiento del módulo Inteligencia de Negocios)

**Estado:** ✅ F0–F6 completas (2026-09-21) — vendible con gating runtime; F7 opcional post-cierre
**Fecha:** 2026-09-21
**Alcance:** cross-repo (business_management + erp-webapp). Coordinación en este `conductor/` raíz.
**Precedentes de patrón:** toggle de módulo reservations (setting BE + `ReservationsModuleRoute` FE +
nav condicional) y permisos por módulo (`RequireModulePermission`).

---

## 1. Objetivo

Que el sistema se pueda **comercializar en dos ediciones**:

| Edición | Contenido |
|:--------|:----------|
| **ERP Core** | Ventas, compras, inventario, caja, pedidos, SIFEN, documentos, usuarios/roles — **sin BI** |
| **ERP Core + BI Premium** (bundle) | Todo lo anterior **+** módulo BI completo |

El módulo BI es un **add-on licenciado**: sin licencia vigente, toda la superficie BI queda
inaccesible (backend responde 403 y el frontend oculta nav/rutas), sin afectar en nada el resto
del ERP. Con licencia, el comportamiento es exactamente el actual.

Principios:

1. **Un solo binario / un solo build** para ambas ediciones: el gating es runtime (entitlement),
   no compile-time. (Ver F7 opcional para strip de build.)
2. **El enforcement es server-side y no negociable**; el gating FE es solo UX (ocultar/teaser).
3. **Entitlement ≠ permiso.** La licencia es a nivel de instalación (pack comprado); los
   permisos (`dashboard:read`, `analytics:read`, …) siguen siendo el control fino por usuario
   DENTRO de una instalación que sí tiene el pack. No se agregan permisos nuevos ni migraciones.
4. **Costo de revocación bajo:** apagar el pack no rompe nada del core (nav, landing, rutas
   alternativas ya resueltas — ver ADR-6).

---

## 2. Inventario de la superficie BI (lo que se licencia)

### 2.1 Backend — todo `internal/analytics`, un solo choke point

Todas las rutas BI se registran en
`business_management/internal/analytics/http/routes.go` (`RegisterRoutes`, línea 63) y pasan por
la cadena compartida **`biChain(perm, group)` (línea 92)**. Meter el middleware de licencia ahí
cubre ~120 endpoints de una vez:

| Grupo | Prefijo (línea) | Permiso actual | Endpoints |
|:------|:----------------|:---------------|:---------|
| Dashboard | `/dashboard` (103) | `dashboard:read` | 7 |
| Receivables | `/receivables` (114) | `receivables:read` | 13 |
| Payables | `/payables` (131) | `payables:read` | 17 |
| Financial Reports | `/financial-reports` (152) | `reports:read` | 19 |
| VAT Prorrateo | `/vat-prorrateo` (174) | `reports:read` | 3 |
| Discount reports | `/sales-analytics/discounts` (185) | `reports:read` | 2 |
| Sales Analytics | `/sales-analytics` (190) | `analytics:read` | 28 |
| Inventory Analytics | `/inventory-analytics` (232) | `analytics:read` | 14 |
| Profitability | `/profitability` (259) | `analytics:read` | 14 |
| Audit | `/api/v1/audit` (277) | `audit:read` | 17 |
| Forecast | `/forecast` (299) | `analytics:read` | 5 |

No queda lógica BI fuera de `internal/analytics` (FASE 10 completa; `handlers/` ya no tiene BI).
**OJO:** los endpoints de SIFEN (`internal/sifen`) NO pasan por `biChain` → quedan en Core
(consistente con ADR-3).

### 2.2 Frontend — bloques contiguos ya delimitados

`erp-webapp/src/App.tsx` (routing único, lazy en todo):

| Bloque | Rutas | Líneas App.tsx |
|:-------|:------|:---------------|
| Dashboard (+payables detail) | `/dashboard*`, `/payables/*` | 319–352 |
| Finance / reports | `/finance/*` (cash-flow, tax, P&L, legal books, SIFEN dashboards) | 353–382 |
| BI Forecasting | `/bi/pronosticos/*` (5) | 384–409 |
| Profitability | `/profitability/*` (6) | 411–417 |
| Receivables | `/receivables/*` (6) | 419–448 |
| Sales Analytics | `/sales-analytics/*` (6) | 458–484 |
| Inventory Analytics | `/inventory-analytics/*` (4) | 486–506 |
| Audit | `/auditoria*` (4) | 726–745 |

Nav: el BI entero es **un grupo top-level** en `src/layouts/main/navigation.ts:51-171`
(`common.bi`, gate hoy `analytics:read`), filtrado por `filterNavigationItems` (379-391).
Servicios: `src/services/bi/*` (9 módulos, un solo `apiClient`). i18n: namespace `bi.*`
(778 keys es/en) ya se carga lazy (`loadBiNamespace.js`).

Gating existente que se conserva intacto: `PermissionGuard`, `DashboardRoute`, perm-gating de
llamadas en `useDashboardStore` (4 de 8), `utils/userPermissions.ts` (localStorage, fail-open).

---

## 3. Decisiones de diseño (ADR)

### ADR-1 — Mecanismo: archivo de licencia firmado (ed25519), no setting ni env var

- **Rechazado: setting** (`modules.bi_premium.enabled` vía `/api/v1/settings`, patrón
  reservations): el cliente tiene usuarios admin en su propia instalación; un flag editable con
  `settings:write` no es un mecanismo de venta.
- **Rechazado: env var sola** (`BI_LICENSED=true`): trivial de activar; solo sirve como override
  de desarrollo.
- **Elegido:** licencia JSON **firmada con ed25519** (stdLib `crypto/ed25519`, cero dependencias):
  `{edition, customer, modules:["bi"], issued_at, expires_at}` + firma. La clave pública va
  embebida/configurada en el binario; la privada la conserva exclusivamente el vendedor (jamás en
  el repo). Sin archivo y con enforcement activo → Core puro.
- Env vars (patrón `BI_RATE_LIMIT_ENFORCE`): `LICENSE_FILE` (path, default `./license.json`),
  `LICENSE_ENFORCE` (default `false` en dev = fail-open sin archivo; **`true` en deploy prod**),
  `LICENSE_GRACE_DAYS` (default 0, ver ADR-4).

### ADR-2 — El licensing no tiene bypass por rol

El admin (`F2VLso`) hoy bypasea los permisos de módulo (`middleware/module_permission.go:41`).
Eso se mantiene para permisos, pero **no** para licencia: sin pack, ni el admin ve BI. Es
justamente el punto comercial.

### ADR-3 — Alcance del pack: todo `internal/analytics` + dashboards fiscales fuera

- **Dentro del pack:** las 11 rutas-grupo de §2.1 y los bloques FE de §2.2 (incluida Auditoría,
  que hoy vive dentro del grupo nav BI).
- **Carve-out SIFEN:** los dashboards fiscales (`FiscalOpsDashboard`, `SkippedNumbersPage` bajo
  `/finance/*`) monitorean la operación de e-factura (core) y consumen `/sifen`, no `analytics`.
  Quedan en **Core** → en FE requieren un wrapper aparte de las páginas contiguas de
  `/finance/*` (dos wrappers, no uno). **Confirmar en F0** (alternativa: todo `/finance` en el
  pack, más simple).

### ADR-4 — Expiración: lock directo + banner, grace opcional

`expires_at` vencido → entitlement inactivo → igual que sin licencia (403 + UI oculta), más un
banner FE "licencia vencida, contacte a su proveedor". `LICENSE_GRACE_DAYS` permite días de gracia
post-vencimiento con el pack aún activo y banner de aviso. Sin modos read-only intermedios (YAGNI).

### ADR-5 — Entitlement en la sesión

- `LoginResponse` y `GET /api/v1/users/me` (`myProfileResponse`) agregan:
  `entitlements: { edition: string, modules: ["bi"], bi_expires_at: RFC3339|null }`.
  El FE ya usa `/me` como fuente canónica (AuthContext B4) y persiste a localStorage el espejo
  de permisos (`utils/userPermissions.ts`) → mismo patrón para `entitlements`
  (`persistEntitlements` / `hasStoredEntitlement`).
- Endpoint nuevo **`GET /api/v1/system/license`** (auth-only, sin permiso): estado de licencia
  para la card de Configuración (edición, módulos, vencimiento, días restantes).
- Modo demo FE (`VITE_USE_DEMO`): entitlements completos fake (los demos deben mostrar BI).

### ADR-6 — Instalaciones existentes y landing sin BI

- **Sin migraciones.** El estado de licencia es in-memory (leído del archivo en boot + hot-reload
  barato vía mtime). Nada que backfilllear en DB.
- Instalaciones existentes: paso de **provisioning en runbook** (F6) — emitir e instalar licencia
  con BI activo antes de flipear `LICENSE_ENFORCE=true` en prod.
- **Landing:** `HomeRedirect` (App.tsx:203-215) hoy aterriza en `/dashboard` si
  `dashboard:read`. Nueva regla: si hay pack BI **y** `dashboard:read` → `/dashboard`; si no →
  `/pedidos` (misma landing que vendedores). Sin BI, el ERP Core queda 100% navegable.
- Mid-session revocation: el listener global `api:forbidden` (App.tsx:235-258) detecta el código
  nuevo `MODULE_NOT_LICENSED`, refresca entitlements, toast + redirect fuera de BI.

### ADR-7 — Contrato de error: 403 con código explícito `MODULE_NOT_LICENSED`

`apperr` no cambia de `Kind` (Forbidden ya mapea 403); el middleware de licencia responde su
propio envelope **plano** `{"error":"…","code":"MODULE_NOT_LICENSED"}`. Ajuste sobre el boceto
original de este ADR: el FE parsea `code` desde el envelope plano de `helpers.ErrorResponse`
(`erp-webapp/src/utils/ApiError.js`, `toApiError` prioriza `error.code`) — un envelope anidado
`{success:false, error:{code}}` NO sería leído por el FE. El orden en `biChain` pone la licencia
**antes** del permiso, para que el 403 de licencia no se confunda con falta de permiso. Además,
el dispatcher FE (`BusinessManagementAPI.ts`) emite `api:module_not_licensed` para este código
(incluso en GET, que hoy silencia los 403 de permiso) y suprime el `api:forbidden` duplicado.

---

## 4. Fases

> Regla de gates de la casa: BE `make build` + `go test ./...` + `golangci-lint run`; FE
> `npx vitest --run` + `npx tsc --noEmit` + `pnpm build` + `pnpm lint:design`. Sin migraciones en
> este plan (nada que correr con `migrate-up`). Rebase antes de gates si main avanza a mitad de
> sesión.

### F0 — ADRs y comercial (este documento)

- [x] **Confirmado ADR-3** (2026-09-21): carve-out SIFEN — los dashboards fiscales
  `/finance/sifen-*` (consumen `/sifen`, operación core de e-factura) quedan en **Core**;
  el resto de `/finance/*` dentro del pack. Auditoría **dentro** del pack (vive en el grupo nav
  BI y en `biChain`).
- [x] **Confirmado ADR-4** (2026-09-21): `LICENSE_GRACE_DAYS` default **0**; nav sin licencia
  **oculta** (no teaser). Banner solo informativo.
- [ ] Definir SKUs comerciales (Core / BI add-on / bundle) y precios — negocio, fuera del repo.
  No bloquea implementación (el esquema ya soporta edition/modules arbitrarios).
- [x] Claves ed25519 (2026-09-21): se generan con `cmd/licensegen keygen`. La privada vive
  **fuera del repo** (custodia del vendedor); la pública viaja en `LICENSE_PUBLIC_KEY` (base64).
  Los tests generan pares efímeros in-test (ninguna clave en el repo, ni de test).
- [x] **Esquema JSON de licencia congelado** (2026-09-21). El archivo es un envelope con la
  firma desprendida sobre la **forma canónica compacta** del payload (los bytes exactos de
  `json.Marshal` del struct `License` — orden de declaración de campos): el archivo puede ir
  pretty-printeado; solo editar campos del payload invalida la firma.

  ```json
  {
    "payload": {
      "edition": "core+bi",
      "customer": "Cliente Demo",
      "modules": ["bi"],
      "issued_at": "2026-09-21T12:00:00Z",
      "expires_at": "2027-09-21T12:00:00Z"
    },
    "signature": "<base64-std de la firma ed25519 sobre el JSON canónico del payload>"
  }
  ```

  Reglas: `modules` es lista (packs futuros sin migrar esquema); `expires_at: null` = perpetua;
  edition es string libre (`core`, `core+bi`). El FE solo consume la proyección `entitlements`
  y `/system/license`, nunca el archivo.

### F1 — Backend: contexto `internal/licensing`

- [x] `internal/licensing/`: `domain.go` (License, Entitlement, Status), `port.go`, `service.go`
  (Load en boot, verificación ed25519, hot-reload por mtime, `Has(module)`, `Status()`),
  `service_test.go` (licencia válida/vencida/firma inválida/módulo ausente).
- [x] Config: `LICENSE_FILE`, `LICENSE_ENFORCE`, `LICENSE_GRACE_DAYS`, clave pública
  (`server/server.go` + parse en `main.go`, patrón `BI_RATE_LIMIT_ENFORCE`).
- [x] Middleware `middleware.RequireModule(deps, "bi")`: fail-open si enforcement off; 403
  envelope `MODULE_NOT_LICENSED` si activo y sin entitlement (ADR-7).
- [x] Handler `GET /api/v1/system/license` (auth-only) + `entitlements` en login y `/users/me`
  (`identity/http.go`, `http_useradmin.go`; port `identity.EntitlementSource` con bridge en
  `routes/licensing_bridge.go`).
- [x] CLI vendor-only `cmd/licensegen` (emite y firma licencias; nunca commiteado con claves).
  Test de round-trip: archivo emitido ↔ verificación del servicio (firma sobre JSON canónico del
  payload, no sobre el formato del archivo — ver addendum §7).
- [x] **Gate:** build + `go test ./internal/licensing/...` + lint (BE completo: build ok,
  `go test ./...` 39 pkgs ok, golangci-lint limpio). Commit `9dd9942`.

### F2 — Backend: enforcement sobre toda la superficie BI

- [x] Insertar `RequireModule("bi")` al inicio de `biChain` (`internal/analytics/http/routes.go`).
- [x] Verificar `tests/integration/rbac_catalog_drift_test.go` y `config.ModulePermissions`
  (no cambian: el drift test escanea call-sites de `biChain(perm, group)`, que no se tocaron).
- [x] Tests de integración (`tests/integration/bi_licensing_test.go`, 4 tests): ciclo completo
  (sin licencia → instalada hot-reload → vencida → módulo ausente) por los 11 grupos de §2.1;
  grace → 200 + `status:"grace"`; fail-open dev → 200 sin archivo; `/system/license` auth-only.
- [x] **Gate:** `go test ./...` completo (39 pkgs) + lint en verde.

### F3 — Frontend: plumbing de entitlements

- [x] `utils/entitlements.ts` (espejo localStorage, patrón fail-open de `userPermissions`) +
  entitlements en `AuthContext` (login + `/me` + `refreshEntitlements` + demo) y hook
  `useBiPackEnabled()`.
- [x] `BiModuleRoute` (components/auth, clon de `ReservationsModuleRoute`): sin pack → redirect
  `/pedidos` + toast (`licensing.moduleNotLocked`), sin romper deep-links. Aplicado a 41 rutas
  de los bloques §2.2 vía `BiPermissionGuard` (licencia fuera, permiso dentro).
- [x] Carve-out SIFEN en `/finance/*` (ADR-3): `sifen-inutilizacion`/`sifen-ops` sin wrapper de
  licencia; sin pack, sus entradas de nav migran a un grupo Core "Fiscal (SIFEN)".
- [x] Nav: `buildNavigation(t, reservationsEnabled, biEnabled = true)` — grupo `common.bi`
  condicional (MainLayout consume `useBiPackEnabled`).
- [x] `HomeRedirect` con la regla de ADR-6 (extraído a `components/auth/HomeRedirect.tsx`).
- [x] Dispatcher `api:module_not_licensed` (`BusinessManagementAPI.ts`, también en GET y sin
  duplicar `api:forbidden`) + listener en App: refresh entitlements + toast; el cambio de estado
  dispara el redirect de `BiModuleRoute` (revocación en caliente).
- [x] i18n `licensing.*` es/en (toast, banner, card) — firma `t(key, fallback, vars)`.
- [x] Tests: `entitlements.test.ts`, `navigation.bi-pack.test.ts` (3 perfiles), 
  `bi-pack-routes.test.tsx` (HomeRedirect + BiModuleRoute), `AuthContext.entitlements.test.tsx`
  (login/me/refresh/revocación/logout), `BusinessManagementAPI.license.test.ts` (dispatcher 403
  licencia), `LicenseBanner.test.tsx`.
- [x] **Gate:** vitest 1079/1079 + tsc 0 + `pnpm build` + `lint:design` limpio. Commit `b3ab4eb`.

### F4 — Frontend: UX de licencia

- [x] Card "Licencia" en Configuración (`LicenseStatusPage` en `/configuracion/licencia`, solo
  lectura: edición, cliente, módulos, vencimiento, días restantes) + fila en Settings.
- [x] Banner global (`LicenseBanner` en MainLayout) cuando falten ≤30 días o esté vencida/gracia
  (ADR-4); solo con enforcement ON.
- [x] Demo mode: `DEMO_ENTITLEMENTS` (pack completo fake, ADR-5).
- [x] **Gate:** idem F3 (mismo commit `b3ab4eb`).

### F5 — Verificación E2E y suite completa

- [x] Smoke con licencia real emitida por `licensegen` (backend efímero :5061,
  `LICENSE_ENFORCE=true`, JWT admin real): sin archivo → 403 `MODULE_NOT_LICENSED` en toda la
  superficie BI (admin incluido, ADR-2) + `/me` core; instalación → 200 en caliente +
  `/system/license` `active` (364 días) + `/me` `core+bi`; vencida → 403 `expired`; renovación →
  200; revocación (borrar archivo) → 403; core intacto (`/api/v1/settings` → 200). Detalle:
  `RUNBOOK_LICENCIAS_2026-09.md` §9. Verificación FE en navegador (nav sin BI, landing
  `/pedidos`, deep-link → redirect): cubierta por tests de nav/rutas; smoke manual pendiente
  para la demo con el cliente.
- [x] Suites completas de ambos repos en verde (BE 39 pkgs, FE 1079/1079) + registro de cierre
  en este conductor y en los `conductor/` de cada repo.

### F6 — Ops y release

- [x] `RUNBOOK_LICENCIAS_2026-09.md` (en este conductor): emitir, instalar, renovar, revocar,
  provisioning de instalaciones existentes, checklist de deploy prod (`LICENSE_ENFORCE=true`
  recién con licencia instalada) + troubleshooting.
- [x] CHANGELOG + release note (BE `CHANGELOG.md` §Unreleased: nuevas env vars, endpoint
  `/system/license`, campo `entitlements` en login/me, contrato 403).

### F7 — Opcional (post-cierre): strip de build por edición

- [ ] FE `VITE_EDITION=core` excluye chunks BI del bundle; BE build tag sin `internal/analytics`.
  Solo como anti-tamper/optimización: el gating runtime (F1-F2) ya es suficiente para vender.

---

## 5. Riesgos y mitigaciones

| Riesgo | Mitigación |
|:-------|:-----------|
| El cliente con acceso al server flipea `LICENSE_ENFORCE` o edita binario | Fuera de modelo de amenaza razonable para on-prem; la firma cubre el caso estándar. F7 opcional endurece. |
| Cliente pierde BI por sorpresa al vencer licencia | Banner FE con ≥30 días de aviso + runbook de renovación; grace configurable. |
| Romper el core al tocar `biChain` | El cambio es add-first en la cadena; tests de integración por grupo (F2) + suite completa; sin migraciones. |
| Bloquear a quien hoy usa BI en instalaciones existentes | Provisioning previo al flip de `LICENSE_ENFORCE` (runbook F6); en dev siempre fail-open. |
| Drift entre rutas y `config.ModulePermissions` | No se agregan rutas ni permisos; correr `rbac_catalog_drift_test` en F2. |
| Regresión FE en nav/landing para usuarios sin `dashboard:read` (vendedores) | Tests de HomeRedirect/nav con tres perfiles: admin+BI, admin sin BI, vendedor. |

---

## 6. Registro de progreso

| Fase | Estado | Notas |
|:-----|:-------|:------|
| F0 ADRs | ✅ Completa | 2026-09-21: ADR-3/4 confirmados, esquema congelado; SKUs/precios quedan en negocio |
| F1 licensing BE | ✅ Completa | `9dd9942` (2026-09-21): internal/licensing + env + RequireModule + /system/license + entitlements + licensegen |
| F2 enforcement BE | ✅ Completa | `9dd9942`: RequireModule primero en biChain; 4 tests integración (11 grupos × ciclo); gate BE verde |
| F3 plumbing FE | ✅ Completa | `b3ab4eb` (2026-09-21): entitlements AuthContext + BiModuleRoute (41 rutas) + nav + HomeRedirect + dispatcher |
| F4 UX licencia FE | ✅ Completa | `b3ab4eb`: LicenseStatusPage + LicenseBanner + demo mode; gate FE verde (1079/1079) |
| F5 E2E + suites | ✅ Completa | Smoke live con licencia real (runbook §9); suites 39 pkgs BE / 1079 FE. Pendiente opcional: smoke FE en navegador con el cliente |
| F6 ops/release | ✅ Completa | RUNBOOK_LICENCIAS_2026-09.md + CHANGELOG BE |
| F7 strip build (opcional) | ⏳ | Post-cierre; el gating runtime ya es vendible |

## 7. Addendum de implementación (2026-09-21)

- **Firma sobre JSON canónico:** la verificación re-marshalea el payload y firma/verifica esos
  bytes (no el formato del archivo). `json.MarshalIndent` re-indenta los `json.RawMessage`, así
  que firmar los bytes literales del archivo habría acoplado la validez al pretty-print; ahora
  el envelope puede formatearse libremente y solo editar campos del payload invalida la firma.
  Contrato congelado igualmente: los bytes canónicos son los de `json.Marshal` del struct.
- **Envelope 403 plano (ajuste ADR-7):** `{"error":"…","code":"MODULE_NOT_LICENSED"}` — el FE
  (`ApiError.toApiError`) lee `code` del envelope plano de `helpers.ErrorResponse`; el dispatcher
  acepta además el anidado por tolerancia.
- **Puente identity ↔ licensing:** identidad define el wire shape (`identity.Entitlement`) y el
  port `EntitlementSource`; `routes/licensingEntitlementSource` adapta `*licensing.Service`
  (mismo patrón que `AuditEventRecorder`). El servicio de licensing se construye en
  `addRoutes` (lee `server.Config`, como `middleware.BuildDeps`).
- **Carve-out SIFEN en nav (ADR-3):** sin pack, `Fiscal (SIFEN)` aparece como grupo Core propio;
  con pack, las mismas entradas viven en Reportes Financieros (dentro del grupo BI), como hoy.
- **Fail-open end-to-end:** sin entitlements (`null` FE / enforcement OFF BE) todo se comporta
  como licenciado — instalaciones legacy y backend viejo no se rompen; el enforcement real es
  siempre el 403 del BE.
- Commits: BE `9dd9942`, FE `b3ab4eb` (rama `dev` de cada repo). Claves dev de prueba en
  `~/erp-licensing-dev/` (fuera de los repos).
