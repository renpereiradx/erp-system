# PLAN F8 — Pase de errores TypeScript pre-existentes (484 → 0)

**Repo:** `erp-webapp/` (React 19 + Vite 6 + TS)
**Fecha:** 2026-08-04 · **Estado:** 🟢 F8 completo — 0 errores reales (2026-08-05)
**Base:** baseline medido en F7.5 (`pnpm exec tsc --noEmit` = 484 errores tras completar JS→TS)
**Principio rector:** F7.5 dejó los errores pre-existentes INTENCIONALMENTE intactos (pase aparte).
Este pase los ataca. Cada fase = un commit conventional; el gate es el mismo de F7.5:
**tsc global decrece o se mantiene**, 0 errores nuevos, tests 22 files / 60 = baseline, `pnpm build` ✅.

---

## Medición (2026-08-04)

**484 errores en 126 archivos.** Distribución por código:

| Código | Cant | Significado |
|:-------|:-----|:------------|
| TS6133 | 230 | Declaración/import sin usar (47% — mecánico) |
| TS2339 | 82 | Propiedad inexistente sobre tipo |
| TS2741 | 48 | Faltan propiedades en object literal |
| TS2322 | 29 | Tipo no asignable |
| TS2345 | 24 | Argumento no asignable |
| TS18046 | 14 | `error` es `unknown` en catch |
| TS2353 | 12 | Propiedad inexistente en object literal |
| TS2739 | 7 | Tipo no asignable (faltan props) |
| TS6192 | 6 | Todas las imports de una declaración sin usar |
| TS2769 | 6 | Overload no matchea |
| TS2367 | 6 | Comparación aparentemente no intencional (posible bug) |
| TS2554 | 4 | Argumentos de más |
| TS2307 | 4 | Módulo no encontrado (imports rotos) |
| TS2304 | 4 | Nombre no encontrado (posible bug runtime) |
| TS2571 | 2 | Objeto de tipo `unknown` |
| TS2531 | 2 | Posible `null` |
| TS2698/TS2551/TS2352/TS18048 | 5 | Varios |

**Por archivo (top):**

| Archivo | Errores | Nota |
|:--------|:--------|:-----|
| `src/pages/Purchases.tsx` | 119 | 100% TS6133/TS6192 (página semi-migrada; destructurea el hook entero y usa ~10 nombres) |
| `src/store/useProductStore.ts` | 83 | Tipos reales (TS2339 ×42, TS18046 ×14, TS2345 ×12, TS2322 ×7...) |
| `src/pages/SalesNew.tsx` | 30 | ⛔ **DIFERIDO** — archivo en WIP del pase de sales (no commiteable sin mezclar el pase) |
| `src/store/useDashboardStore.ts` | 15 | TS2353 ×7, TS2339 ×5 |
| `src/pages/BudgetManagement.tsx` | 15 | TS6133 ×6, TS2741 ×4 |
| `src/pages/PurchaseRequisitionList.tsx` | 14 | — |
| `src/pages/UserManagementList.tsx` | 13 | — |
| `src/services/productService.ts` | 10 | TS2339 ×6, TS6133 ×4 |
| `src/store/useSaleStore.ts` | 9 | TS6133 ×6, TS2339 ×3 |
| ... | ... | ~116 archivos más con 1–9 errores |

---

## Fases (commit por fase)

| Fase | Alcance | Errores | Estado |
|:-----|:--------|:--------|:-------|
| **F8.1** | Imports rotos TS2307 (brand ×3): `data/mockData.ts`, `features/brands/components/BrandDetailForm.tsx`, `BrandList.tsx` → path correcto a `features/brands/types/brand.ts` | −3 | ✅ `227f256` |
| **F8.2** | `Purchases.tsx`: eliminar ~110 nombres del destructuring (se pasan por `{...logic}`) + imports sin usar (16) | −119 | ✅ `969df21` |
| **F8.3** | `useProductStore.ts`: 83 errores de tipos reales (narrowing, tipos de Product/ProductEnriched, catches) | −83 | ✅ `1f2e14b` |
| **F8.4** | Barrido mecánico TS6133/TS6192 en cola larga (46 archivos): imports/vars sin usar | −104 | ✅ `a1c9e7a` |
| **F8.5** | Errores de tipos reales (TS2339/2741/2322/2345/2353/2367/2554/2304/2571/2531/2769) — por lotes | ~−93 ✅ | `710442c` (batch A+B) · `4d093ab` (F8.5b) · `8fef546` (F8.5c) · `eac0231` (F8.5d) · `99e6ee4` (F8.5e) — **completa: 0 errores reales** |
| **F8.x** | Re-verificación: TS2367/TS2304/TS2571 (posibles bugs runtime reales) tratados como bugs, no silenciados | — ✅ | Hecho en F8.5b–e (ver registro) |

**Diferido (no se toca aquí):** `SalesNew.tsx` (26) + `useBarcodeScanner.ts` (2) — WIP del pase de sales. Únicos errores restantes del baseline (28).

---

## Gates de verificación (cada fase)

1. `pnpm exec tsc --noEmit` → contador global **decrece** (o plano) y 0 errores nuevos.
2. `pnpm test` → 22 files / 60 fallos = baseline exacto.
3. `pnpm build` ✅.
4. Commit conventional por fase (`fix:` para bugs reales, `refactor:` para limpieza mecánica).

## Decisiones registradas

| # | Decisión | Razón |
|:--|:---------|:------|
| D-F8.1 | Corregir los 3 paths de `types/brand` (no borrar los imports) | El tipo canónico vive en `features/brands/types/brand.ts`; los archivos están vivos (`BrandsPage`, `useAttributes`) |
| D-F8.2 | `Purchases.tsx`: recortar destructuring a los ~10 nombres usados; el resto sigue llegando a los hijos vía `{...logic}` | Comportamiento idéntico (los hijos reciben el objeto completo); elimina 119 errores mecánicos de un golpe |
| D-F8.3 | No tocar SalesNew/useBarcodeScanner (WIP sales) | Committear ese archivo mezclaría el WIP del pase de sales; se resuelve cuando se cierre ese pase |
| D-F8.x | TS2367/TS2304/TS2571 = bugs reales → corregir semántica, no `as any` | Son la clase de bug que la migración a tipos destapa (ej. `offset` try-scoped en F7.5) |

## Registro de verificación

| Fecha | Fase | tsc | tests | build |
|:------|:-----|:----|:------|:------|
| 2026-08-04 | Baseline (fin F7.5) | 484 | 22/60 | ✅ |
| 2026-08-04 | F8.1 (`227f256`) | 484 → 481 (cascada de tipos Brand resuelta: 2 errores en BrandsPage + 2 en componentes aparecieron al unificar el tipo y se corrigieron) | 22/60 | ✅ |
| 2026-08-04 | F8.2 (`969df21`) | 481 → 362 | 22/60 | ✅ |
| 2026-08-04 | F8.3 (`1f2e14b`) | 362 → 279 (useProductStore: 83→0; `response.data \|\| response` vestigial eliminado — el service devuelve el producto mapeado directo) | 22/60 | ✅ |
| 2026-08-04 | F8.4 (`a1c9e7a`) | 279 → 175 (46 archivos; TS6133/TS6192 no-WIP → 0; patrón: React imports sin usar por el transform JSX, iconos lucide, destructures muertos, imports de hooks huérfanos) | 22/60 | ✅ |
| 2026-08-04 | F8.5a (`710442c`) | 175 → 121 (BI stores: BIParams.period, unwraps vestigiales, getTopProducts type; **fix de clase**: JSDoc en dropdown-menu/avatar .jsx hacía className/inset requeridos → 14 TS2741; bug latente initialTab 'general'→'info' corregido) | 22/60 | ✅ |
| 2026-08-05 | F8.5b (`4d093ab`) | 121 → 92 (fix de clase JSDoc en button/avatar/separator/popover/select/switch/ToastContainer: props opcionales → 29 TS2741/TS2739) | 22/60 | ✅ |
| 2026-08-05 | F8.5c (`8fef546`) | 92 → 56 (clase ProductEnriched vs Record: search inputs tipados; branches: contrato real {branches,...} vs PaginatedResponse; brandService create/update objeto; DEMO_SALES_PAYMENTS import faltante (ReferenceError real); Badge sin importar; apiService.post data opcional; userService envelope real; useVariants currentBranchId; tags sin slug; CategoryTree parent_id legacy; BranchModal is_active fuera del create) | 22/60 | ✅ |
| 2026-08-05 | F8.5d (`eac0231`) | 56 → 28 (verifyIntegration muerto removido; getPurchaseById real; BIParams.compare; getReorderAnalysis añadido (ruta verificada); /tax-classification/info; StockLevelsTable row unificado; RefObject null; unwraps vestigiales nunca-ejecutados; main.tsx root guard) | 22/60 | ✅ |
| 2026-08-05 | F8.5e (`99e6ee4`) | 28 → 28 (0 reales — budget_number = budget.id (backend genera BUD-<millis>); valid_until/tax_amount opcionales; BudgetCreate ProductOperationInfoResponse; InventoryRisk fallback) | 22/60 | ✅ |
