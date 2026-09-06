# REF — Handoff F8.5 (pase de errores TypeScript) — 2026-08-05 · CERRADO ✅

Archivo de referencia del epic F8. **F8 completo el 2026-08-05: 0 errores reales.**

## Estado final

- **Repo:** `/home/darthrpm/dev/erp-system/erp-webapp` (React 19 + Vite 6 + TS, pnpm)
- **HEAD:** `99e6ee4` — F8.5e. Commits del epic: `227f256` (F8.1), `969df21` (F8.2),
  `1f2e14b` (F8.3), `a1c9e7a` (F8.4), `710442c` (F8.5a), `4d093ab` (F8.5b),
  `8fef546` (F8.5c), `eac0231` (F8.5d), `99e6ee4` (F8.5e)
- **Contador tsc:** 484 (baseline) → **28 total (0 reales)** — los 28 restantes son
  exclusivamente el WIP del pase de sales (SalesNew.tsx 26 + useBarcodeScanner.ts 2)
- **Working tree:** SOLO WIP del pase de sales sin commitear (SalesNew.tsx, VariantSelectorModal.tsx,
  useBarcodeScanner.ts, purchasePaymentService.ts, purchasePaymentsMvpService.ts, SALES_FLOWS_GUIDE.md,
  docs/archive/ borrados) → **NO TOCAR NI COMMITEAR** (se resuelve cuando se cierre ese pase)
- **Tests baseline:** 22 files / 60 fallidos (pre-existentes) — `pnpm test`
- **Build:** `pnpm build` ✅ (transpile only; gate real: `pnpm exec tsc --noEmit`)

## Invariantes por batch

1. tsc global flat-o-baja, sin errores NUEVOS en archivos que no se tocaron
2. tests idénticos al baseline (22 files / 60 fallidos)
3. `pnpm build` ✅
4. Commits conventional (`fix:`/`refactor:`) con mensaje que documenta bugs reales encontrados
5. No tocar el WIP de sales

## Catálogo de patrones de fix (validados)

- `catch (error: any)` — catch en TS strict infiere `unknown`; si el código usa `error?.name`/`error.message` → TS18046/TS2339
- params `= {}` → tipar explícitamente (`{ signal?: AbortSignal } = {}` etc.)
- Arrays `const x = []` → `const x: any[] = []` (si no, `never[]` → TS2345)
- Unwraps vestigiales `response.data || response` → `response` cuando el service ya mapea (verificar firma del service antes)
- `_params` / `_clientId` para noUnusedParameters (TS6133 en params)
- JSDoc `/** @param {{ ... }} props */` en componentes `.jsx` sin tipos: TS infiere props destructuradas sin default como REQUERIDAS (className/inset → TS2741 en call-sites). Fix de clase: JSDoc en el componente, no en cada call-site
- Tipos canónicos: `@/types` (Currency, PaymentMethod, ProductEnriched, BIParams...) — los features re-exportan, no duplican
- `'period' not in BIParams` → agregar `period?: string` al tipo (ya hecho en `src/types/bi.ts`)
- Bugs runtime reales se CORRIGEN (no se silencian): p.ej. `initialTab` 'general'→'info', `offset` fuera del try (F7.5), `url` de getByDate

## Inventario restante — ✅ CERRADO (0 errores reales)

El inventario de 91 errores (44 archivos) quedó resuelto en F8.5b–e. Únicos errores
restantes del baseline: 26 en `SalesNew.tsx` + 2 en `useBarcodeScanner.ts` (WIP sales).

## Clase pendiente (resuelta en F8.5c)

- **ProductsTable y similares**: `Type 'ProductEnriched' is not assignable to type 'Record<string, unknown>'` —
  resuelto tipando filter/map con `ProductEnriched` (import desde `@/types`, el mismo que usa el service)
  y `getProductDisplay` con el patrón `any` del mapper defensivo; filtros legacy `status/is_active`
  simplificados al campo canónico `state`.
- **TS2367/TS2304/TS2571 tratados como bugs reales** (F8.5c–e): `DEMO_SALES_PAYMENTS` import faltante
  (ReferenceError en demo mode), `<Badge>` sin importar, `budget_number` inexistente → `budget.id`
  (el backend genera BUD-&lt;millis&gt;), `verifyIntegration` muerto removido, `getPurchaseOrderById`→`getPurchaseById`,
  `getReorderAnalysis` añadido al service (ruta verificada), `TAX_CLASSIFICATION_INFO`→`/tax-classification/info`.

## Pitfalls (del skill erp-integration-pass + vividos en vivo)

1. **patch() fuzzy match puede "SUCCEEDER" con contenido equivocado** (fusionó líneas 2 veces:
   ProductsHeader.tsx, financialReportsService.ts). SIEMPRE re-leer el diff; si el resultado no es el esperado,
   re-leer el archivo y re-patchear con un span único más grande.
2. **patch() "no_change" FALSO**: reporta "already contains target text" cuando el old_string NO matchea
   (p.ej. orden de imports distinto al asumido). Verificar con `grep -n` el contenido real antes de insistir.
3. **Commits con paths parciales parten renames** (lección F7.5): `git add` del archivo completo o ambos paths.
4. **Los LSP diagnostics intermedios entre patches encadenados son engañosos** — el estado real lo da tsc completo.
5. **Remover un destructure puede huérfanar el import del hook** (useI18n/useBranch): `grep -c` antes de eliminar.

## Comandos de verificación

```bash
cd /home/darthrpm/dev/erp-system/erp-webapp
pnpm exec tsc --noEmit 2>&1 | tee /tmp/tsc_next.txt | grep -cE "error TS"   # contar
grep -E "error TS" /tmp/tsc_next.txt | grep -vE "SalesNew|useBarcodeScanner"  # resto real
pnpm build && pnpm test 2>&1 | grep -E "^ (Test Files|Tests)"
```

## Docs de coordinación (padre, no versionado)

- `../conductor/PLAN_F8_ERRORES_TYPESCRIPT.md` — plan maestro del epic (actualizar tras cada batch)
- `../conductor/ESTADO_INTEGRACION_FRONTEND.md` — estado global del frente
- Skill: `~/.hermes/skills/software-development/erp-integration-pass/` (referencia js-to-ts-migration.md)
