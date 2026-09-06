# PLAN F7 — Contrato HTTP unificado (fase estructural FE/BE)

**Repo(s):** `erp-webapp/` (principal) + `business_management/` (solo ítems coordinados)
**Fecha:** 2026-08-03 · **Estado:** 🟢 F7.1–F7.5 completadas · ✅ F7.6 eliminado de raíz por **F7.7** (refactor; el ADR F7.6 original fue reaperturado)
**Base:** `AUDITORIA_INTEGRACION_FRONTEND_BACKEND.md` §3.1/§3.6/§4.4/§6.1/§6.3/§6.4 y `ESTADO_INTEGRACION_FRONTEND.md` §F7
**Principio rector:** FE-first. Nada de cambios BE hasta que el FE lea el contrato canónico.
Cada sub-fase es independiente y revisable; se mide alcance real antes de tocar.

---

## Por qué existe F7

El refactor hexagonal cerró la deuda estructural del BE pero **no tocó el contrato HTTP**
(DO-12). Resultado: 5 wrappers de lista, alias duplicados (`currency_code`), tipos FE que mienten
sobre el BE, capa JS sin tipos y casts `as any` que esconden bugs. F7 ordena esa limpieza sin
romper compatibilidad, en sub-fases independientes.

## Alcance medido (2026-08-03)

| Ítem | Métrica |
|:-----|:--------|
| `currency_code` en FE | **24 archivos** (15 lectores, 3 emisores, 3 demos, 3 tipos/JSDoc) |
| `(apiClient as any)` | **31** (api.ts 23, brandService 5, productService 2, purchaseService 1) |
| Tipos duplicados Currency | **3** (`types.ts:626` canónico · `features/purchases/types/index.ts:47` legacy · `types/payment.js` JSDoc) |
| Tipos duplicados PaymentMethod | **2** (`types.ts:608` canónico · `features/purchases/types/index.ts:40` legacy con `name` que el BE no envía) |
| Extractores de lista duplicados | **2** (`saleService.extractSalesAndPagination` ~135 líneas + `purchaseService.extractPurchasesAndPagination` ~132 líneas) |
| Servicios legacy JS | ~20 (sin tipos, sin protección de contrato) |
| Doble-`/api` | 3 puntos acoplados (baseURL `/api` + endpoints `/api/v1/...` + proxy que quita un `/api`) |

---

## Sub-fases

### F7.1 — Desduplicar tipos (Currency ×3, PaymentMethod ×2) ✅ COMPLETADA (2026-08-03)

**Objetivo:** un solo tipo `Currency` y un solo `PaymentMethod` canónicos en `src/types.ts`;
eliminar las definiciones legacy de `features/purchases/types/index.ts` y el JSDoc de
`types/payment.js` (convertir a referencia del canónico).

**Dependencia:** los tipos locales de purchases usaban `currency_code`/`is_base_currency`/`name`
(alias legacy). **F7.2 migró los lectores primero** (el error TS2339 en `usePurchasesLogic.ts:343`
confirmó la dependencia en vivo).

**Ejecutado:**
- `features/purchases/types/index.ts`: eliminadas las interfaces locales `Currency` y
  `PaymentMethod` → re-exportan los canónicos (`export type { Currency, PaymentMethod } from '@/types'`).
  El `name` de PaymentMethod desapareció (el BE no lo emite; solo description + method_code).
- `types.ts`: el canónico `Currency` ahora declara los alias legacy que el BE aún emite
  (`currency_code?`, `currency_name?`, `is_base_currency?`) marcados para el pase final.
- `types/payment.js`: JSDoc de `Currency`/`CurrencyEnriched` con `code` canónico + alias legacy.

**Criterio de completitud:** `grep -rn "interface Currency\|interface PaymentMethod" src/` → solo
`types.ts` ✅ (verificado).

### F7.2 — Migrar `currency_code` → `code` (FE-first) ✅ COMPLETADA la parte FE (2026-08-03)

**Objetivo (D-§4.4):** desbloquear la eliminación del alias `currency_code` en el BE
(`internal/currency/dto.go:48`). El FE debe leer `code` (canónico) con fallback defensivo
`code || currency_code` en los sitios que aún puedan recibir objetos legacy/demo.

**Pasos ejecutados (FE-first, en orden):**
1. `currencyService.normalizeCurrency` emite `code` (de `payload.code || payload.currency_code`)
   **y mantiene** `currency_code` como alias legacy.
2. `demoData.js`: `DEMO_CURRENCIES_DATA` (8 entradas) y los 2 mappers (`createDemoCurrency`,
   `updateDemoCurrency`) emiten `code` además de `currency_code`.
3. Lectores migrados a `code || currency_code` (16 sitios): `usePurchasesLogic.ts` (156, 342, 726),
   `PurchaseCheckoutModal.tsx` (203-204), `PaymentConfigCard.tsx` (64-65), `currencyService.js`
   (search, `formatCurrencyCode`), `useCurrencyStore.js` (41, 86), `useExchangeRateStore.js` (128, 282),
   `usePaymentBootstrapStore.js` (151, 219), `paymentBootstrapService.js` (238), `CurrencySelector.jsx`
   (74, 82-84, 147, 206 — el "key de matching exclusiva" de D-§4.4), `ExchangeRateDisplay.jsx` (253),
   `CurrencyConverter.jsx` (217, 221), `Currencies.jsx` (603, 713), `ExchangeRates.jsx` (116, 177, 214).
4. Emisores internos emiten `code` además de `currency_code` (aditivo): `exchangeRateService.js:133`,
   `paymentBootstrapService.js:173`, `usePaymentBootstrapStore.js:145`.
5. JSDoc/tipos actualizados (`paymentBootstrapService.js`, `types/payment.js`, `purchases/types`).

**🐛 Bug destapado y corregido:** `currencyService.preparePayload` enviaba `currency_code` en el
request de crear/actualizar moneda, pero el BE espera `code` (`CreateRequest`, `currency/dto.go:7`)
→ **crear moneda desde el FE mandaba un campo que el BE ignoraba**. Corregido a `code` (acepta
`code || currency_code` de entrada). Esto es exactamente la clase de bug que F7.4/`as any` esconde.

**Quedan `currency_code` (intencional, documentado):** requests de pago
(`purchasePaymentService.js:122,127`, `purchasePaymentsMvpService.js:961` — el BE los acepta así),
emisores legacy con `code` ya añadido (`demoData.js:2133-2138`), form state de `Currencies.jsx`
(entrada de usuario, no respuesta BE), y los alias en normalizadores/emisiones (pase final).

**NO se hizo:** eliminar el alias `currency_code` del FE ni del BE. Eso es el **pase final**
(PR BE + limpieza FE simultánea) — ahora sí desbloqueado: todos los lectores usan
`code || currency_code`.

**Criterio de completitud:** lectores directos de `.currency_code` sobre respuestas BE → **0**
✅ (verificado con grep; los restantes son requests/emisores/form-state documentados).

### F7.3 — Unificar extractores de lista (DRY, sin cambio de comportamiento) ✅ COMPLETADA (2026-08-03)

**Objetivo (§3.1/§6.1):** `extractSalesAndPagination` + `extractPurchasesAndPagination` (~267
líneas casi idénticas) → un helper `src/utils/extractListResponse.ts` parametrizado, con la
**lógica exacta** de ambos (mismas claves, mismo orden de prioridad).

**Ejecutado:**
- `src/utils/extractListResponse.ts` creado con las opciones verificadas de ambos extractores
  (itemKeys, isSingleItem, pageSizeFallbackKeys, extraTotalKeys, includeLegacyPaginationKeys)
  y `firstDefined` replicando la semántica `??`.
- `saleService.extractSalesAndPagination` y `purchaseService.extractPurchasesAndPagination`
  delegan al helper (~250 líneas eliminadas; los wrappers conservan el shape de salida exacto:
  dual-key para sale, camelCase para purchase).

**Criterio de completitud:** los dos services delegan al helper ✅; comportamiento idéntico
verificado con `pnpm build` + suite de tests sin cambios vs baseline (22 archivos / 60 tests
pre-existentes, 0 nuevos).

**NOTA (deuda futura, §6.1):** el helper existe porque el BE tiene 5 wrappers. Cuando el BE
unifique, el helper colapsa a ~5 líneas — el punto de medición queda en un solo archivo.

### F7.4 — Eliminar `(apiClient as any)` ✅ COMPLETADA (2026-08-03)

**Alcance medido:** 31 casts (api.ts 23, brandService 5, productService 2, purchaseService 1).
**Objetivo (§6.4):** que TS marque los bugs latentes de contrato. `BusinessManagementAPI` ya tiene
`get/post/put/delete` públicos tipados — la mayoría de casts son innecesarios.

**Ejecutado:**
- Verificado contra la clase real: **todos** los métodos delegados existen tipados en
  `BusinessManagementAPI` → los 31 casts eran innecesarios y se eliminaron.
- **🐛 Bug de firma destapado y corregido:** `apiService.getSalesByClientId` pasaba
  `id: string | number` a un método que espera `clientId: string` — el cast lo ocultaba.
  Corregido con `String(id)`.
- **`apiService` ya no es `any`**: quitado el tipo `: any` de la fachada — TS infiere el tipo
  del objeto y marcará cualquier método inexistente en el futuro (los 9 consumidores compilan).

**Criterio de completitud:** `grep -rn "(apiClient as any)" src/` → **0** ✅ (verificado).

**Nota:** quedan otros `as any` en el codebase (no relacionados con `apiClient`) — se tratan en la
migración JS→TS (F7.5), que ahora tiene el cliente tipado como base.

### F7.5 — JS→TS de servicios legacy ✅ COMPLETADA (2026-08-04)

**Alcance medido (2026-08-04):** 24 servicios JS (~8.900 líneas). **4 resultaron dead code**
(0 consumidores verificados en todo el repo) → **eliminados en `4a0f05d`** (no migrados):
`reportService`, `productAdjustmentService`, `manufacturingService`, `currencyConversionService`
(~1.150 líneas). Resto: ~7.700 líneas en 20 archivos vivos, migrados por familias, cada familia con
su commit.

**⚠️ Hallazgo de verificación (2026-08-04):** `pnpm build` es solo `vite build` — **NO ejecuta tsc**.
El repo tenía **590 errores TS pre-existentes** invisibles al build. Desde ahora: `pnpm exec tsc --noEmit`
es el gate de tipos real (contador global en cada sub-fase).

**Progreso:**

| Familia | Archivos | Estado | tsc migrados | tsc global |
|:--------|:---------|:-------|:-------------|:-----------|
| Dead code | reportService, productAdjustmentService, manufacturingService, currencyConversionService | ✅ `4a0f05d` | — | 590 → 590 |
| Pago núcleo | currencyService, paymentMethodService, cashRegisterService, paymentBootstrapService, paymentApiDebug | ✅ `4ffa41e` | 97 → 0 | 590 → 486 |
| Exchange/tasas | exchangeRateService (967), taxRateService (146) | ✅ `ef807be` | 62 → 0 | 486 → 484 |
| Pagos de compra | purchasePaymentsMvpService (1081), purchasePaymentService (219) | ✅ `6208a79` | 58 → 0 | 484 → 484 |
| Maestros | clientService, supplierService, categoryService, categoryCacheService | ✅ `382304f` | 30 → 0 | 484 → 484 |
| Inventario/pricing | inventoryService (920), priceAdjustmentService (287), priceTransactionService (603) | ✅ `f0c6a83` | 81 → 0 | 484 → 484 |
| BI/misc | biForecastingService (597), telemetryService (110), cashAuditService (88), roleService (115) | ✅ `6c153fc` | 23 → 0 | 484 → 484 |

**Mecánica de migración (probada en pago núcleo):**
1. `git mv X.js X.ts` — los imports sin extensión resuelven igual (0 imports `.js` explícitos).
2. `pnpm exec tsc --noEmit` — los errores del archivo migrado son la deuda a sanear (no tocar los
   486 pre-existentes de otros archivos: son baseline).
3. Patrones de saneamiento: `catch (error: any)` (comportamiento idéntico al JS sin tipar),
   tipar el 4º param del logger (`PaymentApiDebugOptions` en `paymentApiDebug.ts`),
   **los JSDoc `@param` se IGNORAN en `.ts`** → anotar parámetros con default inline,
   `const errors = []` → `string[]`, `String()` para URLSearchParams.
4. `src/vite-env.d.ts` creado (types de `import.meta.env`) — limpia también errores pre-existentes
   de `demoAuth.ts`.

**⚠️ Pitfall baseline-compare (2026-08-04):** `git stash push -- <servicio.ts>` sobre renames
staged (R .js→.ts) es **poco fiable**: al stashar, ni el `.js` ni el `.ts` quedan en el árbol →
los imports de consumidores se rompen → vitest cuenta módulos rotos como archivos fallidos y NO
ejecuta sus tests (conteos distorsionados: 54 vs 60 reales). Además el `pop` conflictúa
(rename/delete). **Gate fiable para familias migradas:** (a) tsc en migrados = 0, (b) tsc global no
sube, (c) `pnpm test` da el conteo documentado (22 files / 60 tests). NO usar stash para baseline
en F7.5. Para recuperar un stash dropeado por error: `git fsck --no-reflogs --unreachable` y buscar
el commit cuyo tree tenga los `.ts`.

**Verificación familia pago núcleo:** tsc migrados 0 errores · tests 60 fallos = baseline 60
(0 nuevos, 22 archivos fallidos vs 23 baseline) · `pnpm build` ✅.
**Verificación familia exchange/tasas:** tsc migrados 0 errores (62→0) · tests 22/60 = baseline ·
`pnpm build` ✅ · **bug JS real corregido:** `getByDate` referenciaba `url` indefinido
(ReferenceError en runtime) → URL construida desde `URLSearchParams`; `Date - Date` → `getTime()`.

**NOTA (deuda futura):** los 486 errores TS pre-existentes (componentes, stores, config) son un
pase aparte — NO se tocan en F7.5.

### F7.6 — Doble-`/api` (baseURL + endpoints + proxy) ✅ ELIMINADO DE RAÍZ POR F7.7 (2026-08-05)

**Estado:** eliminado de raíz por refactor F7.7 (`conductor/PLAN_F7.7_BASEURL_EXPLICITO.md`).
**Historial:** el ADR F7.6 (`ADR-F7.6-DOBLE_API.md`) originalmente cerró esto como "opción C:
documentar + guardia" (proyecto en producción). Reaperturado cuando el proyecto pasó a estar fuera
de producción, permitiendo ejecutar el refactor de raíz: URL directa al backend, sin proxy rewrite
en Vite ni Nginx, alineando dev/prod al modelo que cloud/demo ya usaban. El backend no requiere
cambios — los paths que registra son los que el FE ahora pasa directos.
**Gate:** ✅ tsc 0, build 0, tests 60=baseline (0 nuevos). Smoke pendiente de backend levantado.

---

## Dependencias y orden recomendado

```
F7.2 (currency_code → code) ──► F7.1 (tipos)      ← ambas FE-only, F7.2 antes que F7.1
F7.3 (extractores)            ← independiente, DRY puro
F7.4 (as any) ──► F7.5 (JS→TS)                    ← secuenciales, pases dedicados
F7.6 (doble-/api)             ← ✅ eliminado de raíz por F7.7 (refactor, no ADR)
```

## Decisiones

| # | Decisión | Razón |
|:--|:---------|:------|
| D-F7.1 | El tipo canónico vive en `src/types.ts`; los feature-types re-exportan | Un solo dueño del contrato; el feature no define su propia verdad |
| D-F7.2 | Fallback defensivo `code \|\| currency_code` en los lectores (no `code` a secas) | Cubre objetos demo/legacy sin migrar; el propio D-§4.4 lo prescribe para el PR FE |
| D-F7.2b | `normalizeCurrency` mantiene `currency_code` como alias | El alias FE interno se limpia SOLO en el pase final junto al BE; quitarlo antes rompe lectores JS no migrados |
| D-F7.3 | Helper con parámetros, no forzar un shape único | Los 5 wrappers BE siguen existiendo; el helper debe replicar comportamiento exacto, no "mejorarlo" |
| D-F7.6 | NO tocar el doble-`/api` sin ADR | Confirmado por D-B4: editar un lado aislado rompe 8+ servicios |

## Registro de verificación

| Fecha | Ítem | `pnpm build` | `pnpm test` |
|:------|:-----|:-------------|:------------|
| 2026-08-03 | F7.2 + F7.3 + F7.1 | ✅ | ✅ sin cambios vs baseline (22 archivos / 60 tests pre-existentes) |
| 2026-08-04 | F7.5 dead code (`4a0f05d`) + pago núcleo (`4ffa41e`) | ✅ | ✅ 60 fallos = baseline 60 (0 nuevos; 22 archivos vs 23 baseline) — además `tsc --noEmit`: migrados 97→0, global 590→486 |
| 2026-08-04 | F7.5 exchange/tasas (`ef807be`) | ✅ | ✅ 22 files / 60 tests = baseline — además `tsc`: migrados 62→0, global 486→484; bug `url` de `getByDate` corregido |
| 2026-08-04 | F7.5 pagos de compra (`6208a79`) | ✅ | ✅ 22 files / 60 tests = baseline — además `tsc`: migrados 58→0, global 484 estable |
| 2026-08-04 | F7.5 maestros (`382304f`) | ✅ | ✅ 22 files / 60 tests = baseline — además `tsc`: migrados 30→0, global 484 estable |
| 2026-08-04 | F7.5 BI/misc (`6c153fc`) | ✅ | ✅ 22 files / 60 tests = baseline — además `tsc`: migrados 23→0, global 484 estable (TS6133 de `productService.ts:8` verificado pre-existente vía worktree en `6208a79`) |
| 2026-08-04 | F7.5 inventario/pricing (`f0c6a83`) | ✅ | ✅ 22 files / 60 tests = baseline — además `tsc`: migrados 81→0, global 484 estable; **bug JS real corregido:** `getProductHistory` usaba `offset` (try-scoped) dentro del fallback demo del `catch` → `ReferenceError` en runtime; `offset` movido al scope de la función |
