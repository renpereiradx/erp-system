# PLAN — Servicios vendibles sin stock + módulo de Reservas opcional por negocio

**Fecha**: 2026-08-31
**Estado**: ✅ Implementado (S1–S5 completas, 2026-09-01 — ver §7) · ✅ Pendientes post-auditoría resueltos (2026-09-01 — ver §9)
**Tipo**: Cross-repo (backend `business_management/` + frontend `erp-webapp/`)
**Alcance**: Mecanismo para (a) vender productos de tipo SERVICE sin stock, y (b) activar/desactivar el módulo de reservas por negocio, sin forzarlo en menú ni en el flujo de venta.

---

## 1. Problemas (con evidencia verificada)

| # | Problema | Evidencia | Repo |
|:-:|:---------|:----------|:-----|
| P1 | Un producto SERVICE genérico (ej. "Mantenimiento", categoría SERVICIOS) **no se puede vender** desde la búsqueda del POS: el panel interpreta que requiere stock | `erp-webapp/src/features/sales/components/ProductSearchPanel.tsx:137` — `isOutOfStock = availableStock <= 0 && !product.has_variants` **sin chequear `product_type`**; el botón hace `return` temprano con toast de sin stock (`:153-159`) | FE |
| P2 | Inconsistencia: los otros 3 puntos de entrada al carrito SÍ excluyen SERVICE del chequeo de stock | `SalesNew.tsx:624` (`product.product_type !== 'SERVICE' && ... stock <= 0`), `useBarcodeScanner.ts:41`, `VariantSelectorModal.tsx:356` — la regla vive duplicada con formas distintas | FE |
| P3 | El backend **ya permite** vender SERVICE sin stock (el bloqueo es 100% FE), pero marca servicios como `out_of_stock` en `stock_status` | `internal/sale/sale_service.go:189` — chequeo de stock solo si `product.ProductType == "PHYSICAL"`; `models/product.go:75-84` — `getStockStatus()` devuelve `out_of_stock` cuando `StockQuantity == nil` (típico en servicios) | BE |
| P4 | "Servicio" y "reservable" están **fusionados**: el endpoint que alimenta el módulo de reservas usa una heurística hardcodeada de nombres/categorías | `internal/platform/postgres/product_repo.go:619-631` — `ListServiceCourts`: `id_category IN (3, 38) OR name LIKE '%cancha%'/'%court%'/'%field%'` | BE |
| P5 | El módulo de reservas está **forzado** para cualquier negocio: el paso de Reservas del checkout aparece siempre que hay cliente seleccionado | `erp-webapp/src/features/sales/checkoutSteps.ts:35-37` — `reservationsVisible = (pendingReservationsCount > 0 || hasClient) && !hasReserveInCart` | FE |
| P6 | El menú de Agenda/Reservas siempre visible (solo gated por permiso `reserves:read`, no por configuración del negocio) | `layouts/MainLayout.jsx:403` y `:499-509` (dos entradas al mismo route `/gestion-agenda`) | FE |
| P7 | No existe mecanismo de settings del negocio / feature flags en el backend | Búsqueda `feature_flag|app_settings|system_settings|tenant_config` en `**/*.go`: 0 resultados relevantes | BE |

**Hecho clave**: el backend ya tiene el modelo correcto — `product_type` (`PHYSICAL` | `SERVICE` | `PRODUCTION`) discrimina stock, y la reservabilidad ya existe como concepto per-producto (`ScheduleGenerationConfig` por producto, `internal/inventory/domain.go:343`). Lo que falta es (1) alinear el FE, (2) un flag de negocio, y (3) hacer explícita la reservabilidad en vez de inferirla por nombre.

---

## 2. Principio de diseño: desacoplar 3 conceptos

Hoy el sistema asume `SERVICE ⇒ cancha ⇒ reservable ⇒ módulo visible`. El plan los separa en tres ejes ortogonales:

| Eje | Pregunta que responde | Mecanismo | Dónde vive |
|:----|:----------------------|:----------|:-----------|
| **Tipo de producto** | ¿El producto descuenta stock al venderse? | `product_type` (ya existe): `PHYSICAL` descuenta, `SERVICE` no | BD, catálogo |
| **Reservabilidad** | ¿El producto se vende por franja horaria (agenda)? | `is_bookable` (nuevo, explícito por producto) — reemplaza la heurística de nombres | BD, catálogo |
| **Módulo activo** | ¿ESTE negocio usa reservas? | `business_settings` → `modules.reservations.enabled` (nuevo, nivel negocio) | BD, settings |

Con esto:
- "Mantenimiento" = `SERVICE` + `is_bookable=false` → se vende directo desde el POS, sin stock, sin agenda.
- "Cancha 01" = `SERVICE` + `is_bookable=true` → se vende vía reserva/agenda (comportamiento actual de Vista Bar).
- Un negocio 100% físico → toggle `modules.reservations.enabled=false` → sin menú Agenda, sin paso Reservas en el checkout, sin dashboard de reservas.

---

## 3. Fases (shippeable independiente, orden por valor)

> Cada fase = 1 o más commits lógicos (`feat:`/`fix:`), gates verdes, sin big-bang. S1 y S2/S3 son independientes entre sí (S1 ni siquiera requiere migraciones).

### S1 — Vender servicios sin stock (bugfix P1/P2/P3) 🔴 prioridad

**Frontend**
- [ ] Crear helper puro de dominio `src/domain/products/sellability.ts`:
  - `requiresStock(product): boolean` → `product.product_type !== 'SERVICE'`
  - `isBlockedByStock(product, qtyInCart): boolean` → combina stock disponible + variantes + `requiresStock`
  - `stockBadgeKind(product)` → `'stock' | 'service' | 'variants'` (para badges neutrales)
- [ ] `ProductSearchPanel.tsx`: usar `isBlockedByStock` (corrige P1); badge "Servicio" en vez de "Sin stock" cuando `!requiresStock` (i18n: `sales.search.serviceBadge`).
- [ ] Reemplazar las 3 reglas duplicadas por el helper: `SalesNew.tsx:624`, `useBarcodeScanner.ts:41`, `VariantSelectorModal.tsx:356` (mismo comportamiento, una sola fuente de verdad).
- [ ] Tests unitarios de `sellability` (FE `src/__tests__/`).

**Backend**
- [ ] `models/product.go` `getStockStatus()` (y su gemelo en `internal/catalog`): si `ProductType == "SERVICE"` → devolver `no_stock_tracking` (literal que el FE YA tiene tipado en `types.ts:328`) en vez de `out_of_stock`.
- [ ] Test table-driven del nuevo status.

**Gates**: FE `tsc --noEmit` 0 errores, `pnpm build`, `pnpm test` vs baseline (49 fallos preexistentes — comparar, no romper). BE `go build ./...`, `go test ./...`, `gofmt -l`, `golangci-lint run`.
**Commits sugeridos**: FE `fix(sales): allow selling SERVICE products without stock via shared sellability helper`; BE `fix(catalog): report no_stock_tracking for SERVICE products`.

### S2 — Settings del negocio (backend, P7)

- [ ] Migración (SOLO `make migrate-create`): tabla `business_settings (key TEXT PK, value JSONB NOT NULL, updated_at, updated_by)` + seed `modules.reservations.enabled = true` (default true = no cambia comportamiento actual de Vista Bar).
- [ ] Contexto acotado `internal/settings/` siguiendo el patrón FASE (domain.go / port.go / service.go / handler.go) + repo en `internal/platform/postgres/settings_repo.go` + wiring. Puerto mínimo: `Get(ctx, key)`, `Set(ctx, key, value, userID)` con allowlist de claves conocidas (rechazar claves desconocidas → no es un KV abierto).
- [ ] Rutas: `GET /settings` (auth + `settings:read`) y `PUT /settings/{key}` (auth + `settings:write`). Registrar en `internal/settings/http/routes.go` (patrón FASE 13).
- [ ] RBAC: delta migration de roles con `settings:read` (todos los roles operativos) / `settings:write` (admin/manager). Los roles canónicos se versionan vía migración delta (seeds gitignored).
- [ ] Tests: service con mock por puerto + handler.
- [ ] `make migrate-up` + `make migrate-verify-gaps`.

**Decisiones tomadas**: toggle **global del negocio** (no per-branch) — los branches comparten el modelo comercial; per-branch queda como extensión futura si aparece un caso real.
**Gates**: BE completos + up/down de la migración probados.
**Commit**: `feat(settings): business settings bounded context with reservations module toggle`.

### S3 — Frontend consume la configuración (P5/P6)

- [ ] `src/store/useBusinessConfigStore.ts` (Zustand): fetch de `GET /settings` post-login, expone `reservationsEnabled` (default `true` si el endpoint falla → fail-open conservador: el negocio que lo desactivó lo ve solo si hay error de red, aceptable y documentado).
- [ ] `checkoutSteps.ts`: nuevo campo `reservationsEnabled: boolean` en `CheckoutStepsInput`; `reservationsVisible = reservationsEnabled && (...)`. Actualizar `__tests__/checkout-steps.domain.test.ts`.
- [ ] `MainLayout.jsx`: gatear AMBAS entradas de menú (Ventas → "Agenda y Reservas" `:403` y sección "Planificación y Reservas" `:499`) con `reservationsEnabled && permission reserves:read`.
- [ ] Guard de ruta `/gestion-agenda` (y derivados) — redirect a dashboard + toast si desactivado.
- [ ] i18n completo de strings nuevas (es + en).

**Gates**: FE completos.
**Commits**: `feat(settings): business config store` + `feat(sales): make reservations checkout step conditional on business config` + `feat(nav): hide reservations menu when module disabled`.

### S4 — Reservabilidad explícita por producto (P4)

- [ ] Migración: `ALTER TABLE products ADD COLUMN is_bookable BOOLEAN NOT NULL DEFAULT FALSE` + backfill con la MISMA heurística actual (categorías 3/38, nombres cancha/court/field, `product_type='SERVICE'`) para no perder los datos vivos de Vista Bar.
- [ ] `ListServiceCourts` (`product_repo.go:619`): reemplazar heurística por `WHERE product_type='SERVICE' AND is_bookable AND state=true`. Contrato HTTP estable (mismo endpoint `/products/service-courts`).
- [ ] `internal/catalog`: exponer `is_bookable` en DTO/domain + validación `is_bookable=true ⇒ product_type='SERVICE'` en el service (error tipado `apperr`).
- [ ] FE `ProductFormModal.tsx`: switch "Reservable (agenda de horarios)" visible solo cuando tipo = SERVICE (Zod schema + i18n).
- [ ] Tests: BE validación + FE form.

**Gates**: ambos repos + migración up/down.
**Commits**: BE `feat(catalog): explicit is_bookable flag replacing name heuristics for bookable services`; FE `feat(products): bookable toggle for SERVICE products`.

### S5 — Limpieza y verificación

- [ ] Buscar restos de la heurística (`grep -ri cancha` en queries) y dejar la heurística SOLO en la migración de backfill (histórico).
- [ ] Smoke manual end-to-end con curl al dev server (:5050) + FE :5173: (1) crear servicio no-bookable y venderlo, (2) toggle off → menú y checkout sin reservas, (3) toggle on → flujo cancha actual intacto.
- [ ] Actualizar `conductor/README.md` de ambos repos si aplica.

---

## 4. Decisiones registradas

| ID | Fecha | Decisión |
|:---|:------|:---------|
| D-SR-1 | 2026-08-31 | La venta de SERVICE sin stock NO requiere cambios de backend: `ValidateProductsForSale` ya omite stock para no-PHYSICAL. El fix es FE (regla unificada en `sellability.ts`). |
| D-SR-2 | 2026-08-31 | `stock_status` para SERVICE = `no_stock_tracking` (literal ya tipado en FE), no `out_of_stock`. |
| D-SR-3 | 2026-08-31 | Toggle de módulo: **global del negocio** en tabla `business_settings` (key/value jsonb con allowlist), default `true` al migrar (preserva Vista Bar). Per-branch = extensión futura, no se diseña ahora. |
| D-SR-4 | 2026-08-31 | Reservabilidad = atributo explícito `products.is_bookable` (backfill con heurística actual); reservable ⇒ SERVICE. El endpoint `/products/service-courts` mantiene nombre y contrato. |
| D-SR-5 | 2026-08-31 | Fail-open en FE: si `GET /settings` falla, `reservationsEnabled=true` (comportamiento actual). Documentado en el store. |
| D-SR-6 | 2026-08-31 | Fuera de alcance: multi-reserva por venta (vigente D-Fx-5 en PLAN_FIX_VENTA_COBRO_RESERVA.md — requiere ADR), cambios SIFEN (los SERVICE ya se facturan hoy vía canchas), borrado de código de reservas (el módulo se oculta, no se elimina — siempre deployable). |

## 5. Decisiones abiertas (para el usuario)

| # | Pregunta | Recomendación |
|:-:|:---------|:--------------|
| Q1 | ¿Dónde vive el toggle en la UI? | Configuración y Sistema → nueva página "Preferencias del negocio" (permiso `settings:write`) |
| Q2 | ¿Default del toggle post-migración? | `true` (no cambia nada para Vista Bar; cada negocio lo apaga si no usa servicios) |
| Q3 | ¿S4 (is_bookable) en este pase o diferido? | Diferirlo NO rompe nada (la heurística sigue funcionando); hacerlo ahora elimina deuda. Recomendación: hacerlo — es la raíz del acoplamiento |

## 6. Riesgos

- **S2 RBAC**: agregar permisos nuevos a roles existentes requiere delta migration cuidadosa (patrón ya probado). Si un rol no tiene `settings:read`, el FE store debe tratarlo como fail-open (D-SR-5) para no romper el arranque.
- **S4 backfill**: la heurística actual puede marcar como bookable algún SERVICE que no lo es (nombre con "court"). Mitigación: revisar el resultado del backfill contra `product_schedule_generation_config` (productos con schedule activo) antes de cerrar la fase.
- **Baseline FE**: 17 files / 49 tests fallidos preexistentes — comparar conteos por suite, nunca esperar verde total.

---

## 7. Registro de ejecución (2026-09-01)

| Fase | Estado | Commits | Notas |
|:-----|:-------|:--------|:------|
| S1 | ✅ | BE `1f2678b` · FE `6847b28` | Helper `domain/products/sellability.ts` unifica las 4 entradas al carrito (P1/P2); `no_stock_tracking` en los 5 puntos que computan `stock_status` (P3). Badge "Servicio" con i18n. |
| S2 | ✅ | BE `9db9ddd` | `settings.business_settings` + contexto `internal/settings` completo (allowlist, no KV abierto); rutas `GET/PUT /api/v1/settings`; permisos 184/185 con delta RBAC (`settings:read` operativos, `settings:write` ADMIN+ENCARGADO). Up/down verificados. |
| S3 | ✅ | FE `01332f7` `41b27fd` `8681b0b` | `useBusinessConfigStore` (fail-open D-SR-5); `reservationsEnabled` en `CheckoutStepsInput`; ambas entradas de menú gated + route guard `/gestion-agenda` con toast; i18n es+en. |
| S4 | ✅ | BE `7a3f3b3` · FE `d3e18e2` | `products.is_bookable` con backfill idéntico a la heurística; verificado contra schedules reales: 3 canchas bookable, MANTENIMIENTO no (riesgo R-S4 descartado). `ListServiceCourts` sin heurística, contrato estable. Validación `bookable ⇒ SERVICE` en service (BE) y Zod (FE). Switch en `ProductFormModal` solo para SERVICE. |
| S5 | ✅ | — | Sin restos de heurística en queries (solo en la migración de backfill, histórico). Smoke curl: settings default/PUT/allowlist 400/auditoría OK; `/products/service-courts` devuelve las 3 canchas con `is_bookable` y `no_stock_tracking`; alta de SERVICE no-bookable OK y rechazo 400 de PHYSICAL bookable OK. |

**Gates**: BE `go build`/`go test ./...`/`golangci-lint` verdes (gofmt preexistente sucio en `internal/sale/`: `pos_service.go`, `payment_service_test.go` — fuera de alcance, no introducido por este plan). FE `tsc --noEmit` 0 errores, `pnpm build` OK, suite 48 fallidos preexistentes en todas las fases (baseline estable: 413→429→433 passed).

**Pendiente (fuera de alcance de este pase)**:
- Página UI "Preferencias del negocio" (Q1: vive en Configuración y Sistema, permiso `settings:write`) — el toggle hoy se cambia vía `PUT /settings/{key}`.
- Smoke visual del FE (:5173) con el navegador — la lógica de gates está cubierta por tests de dominio.
- Comentario huero `GetServiceCourtsEnriched` en `database/postgres/product.go:612` (doc drift del adaptador legacy, sin SQL vivo).

## 8. Auditoría de implementación (2026-09-01, orquestador)

**Veredicto: ✅ CONFORME** — verificado contra código, DB viva y runtime (backend levantado en :5050), no solo contra mensajes de commit.

| Verificación | Resultado |
|:-------------|:----------|
| E2E venta SERVICE sin stock (MANTENIMIENTO, pos-checkout) | ✅ `SALE-...-18` PAID atómico; **0** filas de stock creadas para el servicio |
| `/products/service-courts` | ✅ 3 canchas bookable + `no_stock_tracking`; MANTENIMIENTO excluido |
| `stock_status` SERVICE (`/products/all`) | ✅ los 4 SERVICE → `no_stock_tracking` |
| Settings API | ✅ GET devuelve toggle `true`; PUT clave desconocida → 400; PUT tipo incorrecto → 400 (allowlist + validación de tipo efectivas) |
| 4 entradas al carrito vía `sellability.ts` | ✅ ProductSearchPanel:139, SalesNew:625, useBarcodeScanner:41, VariantSelectorModal:96 |
| Gates | ✅ BE build/test/lint verdes (gofmt limpio en tocados; suciedad preexistente en `internal/sale/` confirmada ajena — commit `831fa8e`); FE tsc 0, build OK, suite 48F/433P vs baseline 49F |
| RBAC | ✅ permisos 184/185; `ENCR01` verificado como rol canónico (migración 20260812) |
| Migraciones | ✅ `migrate-verify-gaps` limpio; down-migrations correctas |
| Walk-in/agenda consumen `service-courts` | ✅ `reservationService.getProducts` → `/products/service-courts` (respeta `is_bookable`) |

Hallazgos menores (no bloqueantes):
1. **Código muerto con heurística**: `useProductStore.fetchServiceCourts` (:1373) llama a `/products/enriched` (ruta inexistente) y su fallback filtra client-side `SERVICE || nombre cancha` — pero **no tiene consumidores** (grep: 0 callers). Candidato a F5 dead-code cleanup.
2. **searchableRoutes.js:93** lista "Gestión de Agenda" sin gate — el command palette ofrecería una ruta que redirige con toast si el módulo está off. UX menor.
3. **i18n EN**: `sales.search.serviceBadge` solo existe en es (no hay `en/sales.js`; TODO preexistente en `en/index.js`). EN muestra el fallback español. Patrón preexistente del módulo sales.
4. **Higiene de branches**: los 3 commits BE del plan viven en `feat/cobro-en-divisa` (nombre ajeno al plan), FE en `main`. Al mergear, esa rama lleva mezcladas dos features.

Notas: el PUT exitoso del toggle quedó evidenciado por la fila en DB (`updated_by=smoke_admin`); la auditoría ejecutó PUTs de rechazo (400) y GETs. Ventas de prueba de la auditoría (3) canceladas y verificadas CANCELLED en DB; stock restaurado por el flujo de cancelación.

## 9. Pase post-auditoría: pendientes + hallazgos menores (2026-09-01)

Resolución de los 3 pendientes de §7 y los hallazgos 1 y 2 de §8. Los pendientes Q1 (página UI) y el smoke visual quedan cerrados; el hallazgo 4 (higiene de branches) queda para decisión de merge del usuario.

| Ítem | Estado | Commits | Notas |
|:-----|:-------|:--------|:------|
| Página UI "Preferencias del negocio" (Q1) | ✅ | FE `91933eb` `393c2a3` `e1f9963` | `features/settings/` (page + hook), ruta `/configuracion/preferencias` con `settings:write`, entrada en menú Configuración y Sistema + fila en hub `/configuracion`, i18n es+en, test de página (3 tests). Muestra metadata de auditoría (`updated_by`/`updated_at`). |
| Smoke visual FE :5173 | ✅ | — | Con backend real y navegador: login → toggle ON/OFF → menú Agenda aparece/desaparece → command palette sin "Gestión de Agenda" con módulo off (hallazgo 2 cerrado) → route guard `/gestion-agenda` redirige a dashboard con toast → restaurado a `true` en DB. Capturas verificadas (Fluent 2 conforme). |
| Comentario huero `GetServiceCourtsEnriched` | ✅ | BE `b82d8a5` | Eliminado (doc drift sobre `GetProductByScaleCode`). |
| Hallazgo 1: `fetchServiceCourts` muerto | ✅ | FE `84e0d4a` | Acción + campo `serviceCourts` eliminados (0 consumidores). |

**Bug latente corregido por el smoke**: `businessSettingsService` leía `response?.data` pero `apiClient` (fetch) ya devuelve el body parseado → `GET /settings` llegaba como `[]` al store y el fail-open (D-SR-5) lo enmascaraba (los gates parecían funcionar porque el default es `true`). Corregido en `91933eb`; la página nueva lo destapó porque muestra el listado real.

**Gates**: FE tsc 0, `pnpm build` OK, suite 48F/436P (mismo baseline 48F en los mismos 17 archivos legacy; 433→436 por los 3 tests nuevos), `pnpm lint:design` código nuevo limpio. BE gofmt/build OK en paquete tocado.

**Notas de entorno**:
- Haba un backend viejo (pid de `go run` del 07:30) ocupando :5050; el smoke corrió con backend actual en **:5051** (`make run PORT=5051`) y Vite con `VITE_API_URL=http://localhost:5051`.
- Usuario smoke creado para la UI: `smoke_ui_admin` / `SmokeUi2026!` (rol `F2VLso`, party `smokeuiparty01`) — queda en DB dev para futuros smokes; borrar si molesta.
- Quirk descubierto (no bloqueante, fuera de alcance): `GetByUsername` escanea `p.email` en `string`; un party con email NULL hace fallar el scan silenciosamente y el login responde "invalid credentials" sin trazabilidad. Candidato a fix en identity (puntero/COALESCE) si aparece un caso real.
