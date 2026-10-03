# PLAN_UNITS_V2_COMPLETACION — Unidades de medida: análisis de viabilidad y fases de completación

**Fecha:** 2026-09-30 · **Estado:** ✅ **F1-F5 CERRADOS** (ejecutados 2026-10-02, auditados 2026-10-03, **reparos de auditoría A1-A3 ejecutados y verificados en vivo 2026-10-03** — BE `fd4e7bf`, FE `310bcaa`, ver §10). **F6 rediseñado (2026-10-03) tras revisión de diseño del módulo de manufactura** (§11: producción rota desde 2026-04 + UI inexistente + cero uso → F6.0-F6.3; puntos de decisión del owner en §11.4).
**Scope:** Cross-repo — Backend Go + SQL (`business_management`) y Frontend (`erp-webapp`)
**Antecedentes:**
- `conductor/PLAN_UNITS_CONVERSIONES_2026-09-24.md` (registro v1 de este dir)
- `business_management/conductor/PLAN_UNITS_BACKEND.md` (+ addendum fix derivación 09-26)
- `erp-webapp/conductor/PLAN_UNITS_FRONTEND.md` (+ addendum originalPrice unit-aware 09-26)

---

## 1. Resumen ejecutivo

**Las unidades de medida ya están implementadas en el núcleo del sistema.** El maestro de
productos tiene `base_unit` obligatoria, existe la tabla de conversiones globales **y por
producto**, y los flujos de dinero (vender, comprar, revertir, cobrar, imprimir) ya convierten
unidades correctamente. Esto fue construido en v1 (2026-09-24) y auditado/corregido (09-26).

Lo que queda es **completar la adopción** en módulos periféricos y **consolidar
inconsistencias**. Veredicto: **viabilidad alta** — el patrón (cascada específico→global,
precios por unidad) ya está probado; extenderlo es trabajo incremental, no un rediseño. El
riesgo principal no es técnico sino de **consistencia**: la lista de unidades válidas vive
replicada en 5+ lugares que ya divergieron, y la cascada de conversión existe en 3
implementaciones paralelas (SQL, Go, TS).

**Prioridad P0 (bugs, no mejoras):** `update_sale_item` descuenta con factor global ignorando
la conversión específica, y el modal de compras no reacciona al cambio de unidad (pendiente 🟡
de la auditoría 09-26).

---

## 2. Línea base: qué existe y funciona (verificado 2026-09-30)

| Pieza | Estado | Referencia |
|:---|:---|:---|
| Unidad base del producto | ✅ `products.base_unit` NOT NULL, validada contra catálogo cerrado; editable solo al crear (inmutable después) | `internal/catalog/service.go:72-77,116,130-147`; migración `20260605000001_base_unit_required` |
| Catálogo de unidades | ✅ `GET /units` canónico (33 unidades, categorías) | `internal/measure/http/routes.go:26-28` → `products.get_available_unit_values()` |
| Conversiones | ✅ Globales + por producto (cascada específico-directo → específico-inverso → global-directo → global-inverso) | `unit_conversions` + `convert_units_for_product` (migración `20260924105743`) |
| Ventas | ✅ `sales_order_details.unit`; validación Go temprana (convierte a base antes del stock check); procedure deriva precio base × factor; trigger descuenta convertido; reversa exacta; ticket/factura imprimen unidad | `internal/sale/sale_service.go:310-479`; `20260926164457_fix_sale_unit_price_derivation`; `internal/documents/ticket_renderer.go:120-121` |
| Compras | ✅ `purchase_order_details.unit`; recepción convierte a base, deriva costos, registra `unit_costs` por producto+unidad+variante | `transactions.process_complete_purchase_order` (`20260924105744:643+`); `internal/platform/postgres/cost_repo.go:29-190` |
| Precios/costos | ✅ Por (producto, unidad, variante): `unit_prices` / `unit_costs` | `internal/pricing/service.go:186` (GET /products/{id}/units) |
| POS frontend | ✅ Precios por unidad, recálculo al cambiar unidad, hint sin-conversión, decimales solo pesables | `src/pages/SalesNew.tsx:1020-1096`; `src/domain/units/resolveUnitFactor.ts` |
| Presupuestos / requisiciones / transferencias | 🟡 Envían `base_unit` real (sin selector de otra unidad); transferencias permiten decimales | `BudgetCreate.tsx:156-160`; `PurchaseRequisitionCreate.tsx:123-125`; `CreateTransferModal.tsx:135,173-175` |

---

## 3. Hallazgos backend

| # | Hallazgo | Referencia | Gravedad |
|:--|:---|:---|:---|
| B1 | **`update_sale_item` usa `convert_units` global**, no la por producto: editar una línea de venta en unidad ≠ base descuenta stock con el factor global aunque exista específico → descuadre silencioso. Mismo tipo de bug ya corregido en `add_products_to_sale` (addendum 09-26) | Última definición: migración `20260909214752_sales_discount_columns.up.sql:1132` (uso global en `:1225-1226`) | 🔴 Bug latente |
| B2 | **Compras no validan unidad temprano**: `validateCompleteRequest` valida producto/cantidad/precio pero no `unit` — una unidad sin conversión falla adentro del SQL, tarde y poco accionable | `internal/purchase/order_service.go:82-139` | 🟠 |
| B3 | **CHECK de `purchase_requisition_details` más angosto que el canónico** (faltan ~15 valores: case, carton, roll, pair, set, bundle, sqft, month, tray, bottle, can, jar, stick, slice, portion): una unidad válida en el resto del sistema puede reventar ese CHECK | Migración `20260319000007_add_purchase_requisition_system.up.sql:24-28` | 🟠 |
| B4 | **`budget_order_details` y `purchase_requisition_details` sin `variant_id`**: convertir un presupuesto a venta vende el producto base aunque el ítem sea una variante (GAP conocido de flat-search; el payload FE ya envía variant_id) | `budget_repo.go:55,122-138`; `requisition_repo.go:127`; `convert_budget_to_sale` en `20260817145150:63-64` | 🟠 |
| B5 | **Ajustes de inventario sin unidad**: `manage_stock_adjustment` sin parámetro unit (quien cuenta "5 cajas" calcula los kg mentalmente); `stock_transactions` sin columna unit (trazabilidad) | Definición vigente `20260925131040:118`; `internal/inventory/service.go:178-191`; `database/schema/tables/stock_transactions_table.sql:4-33` | 🟡 |
| B6 | **Transferencias sin unidad** en `branch_transfer_items` (todo en base_unit): consistente pero limitado | `20260402203000:44-66`; `identity_branch_repo.go:691-737` | 🟡 |
| B7 | **BI agrega cantidades sin normalizar**: "unidades vendidas" suma `sod.quantity` mezclando kg/cajas/unidades | `analytics_sales_repo.go:75,208,293`; `analytics_purchase_repo.go:125`; `analytics_inventory_repo.go:36,264,313,401,719` | 🟡 |
| B8 | **Manufacturing desconectado**: unidad texto libre (default "kg"), lista CHECK propia de 14 valores, sin conversiones ni `base_unit` | `internal/manufacturing/service.go:25-27`; `baseline_schema.sql:15008-15019` | ⚪ Vertical propia |
| B9 | Menores: repo legacy de conversiones global-only sin rewire (`database/postgres/unit_conversion.go:11-75`); `resolve_tax_rate_for_transaction` ignora `p_unit`; variantes sin unidad propia (heredan base del padre — decisión razonable, documentar) | | ⚪ |

## 4. Hallazgos frontend

| # | Hallazgo | Referencia | Gravedad |
|:--|:---|:---|:---|
| F1 | **Modal de compras no reacciona al cambio de unidad** (pendiente 🟡 auditoría 09-26): costo prefill queda el de la unidad anterior, margen hereda ese costo sin aviso, sin hint preventivo de unidad sin conversión (solo toast post-submit); cantidad siempre acepta decimales sin gating | `PurchaseProductModal.tsx:571-578,553-562`; `usePurchasesLogic.ts:622-718` (costo `:680`, precio `:684`), toast `:933-938` | 🟠 Gap UX más visible |
| F2 | **Presupuestos fuerzan cantidad entera** (`parseInt` + `min=1`): imposible presupuestar 0,5 kg | `BudgetCreate.tsx:330-333` | 🟠 |
| F3 | **`UnitSelect` solo llegó a ventas y compras**. Siguen sin usarlo: formulario de producto (Select duplicado, `ProductFormModal.tsx:545-585`), modal de conversiones (**inputs de texto libre** para from/to, `UnitConversionsPage.tsx:301-320`), ajustes de stock (`MovementForm.tsx`), ajustes de precio/costo (unidad fija de fila, `ProductHistoryModals.tsx:350,514`) | | 🟡 |
| F4 | **Catálogo local `constants/units.js` vs `GET /units`**: además `validMeasurableUnits` incluye `sqft` (ausente del catálogo FE) y excluye `ton` — ya divergieron | `useProductForm.ts:38-45`; cabecera de `UnitSelect.tsx:2-8` declara la migración como pendiente | 🟡 |
| F5 | **Ajuste masivo de precios ignora unidades** | `PriceAdjustmentNew.jsx` sin referencia a unit | 🟡 |
| F6 | Tickets client-side (comanda de pedidos, ticket de traslado) imprimen cantidad **sin unidad**; el ticket fiscal/receipt server-side sí la incluye | `counterorders/utils/printTicket.ts`; `TransferTicketModal.tsx:95-98` | ⚪ |
| F7 | `UnitConversionsPage` con i18n incompleta (títulos, headers, toasts hardcoded en español) | | ⚪ |
| F8 | BI muestra contadores "Unidades"/"uds." sin UOM por línea | `ProductsCategories.tsx:233+`, `CustomerSellerInsights.tsx:257-275`, `PeriodComparison.tsx:301-305` | ⚪ |

## 5. Riesgos transversales

1. **Lista cerrada de unidades replicada en 5+ lugares**: `internal/shared/validate_unit.go`
   (32 códigos), CHECK de `products.base_unit` (33 con sqft), CHECK de requisitions (angosto),
   CHECK de `manufacturing_supplies` (14), `constants/units.js` FE (~30) +
   `validMeasurableUnits`. Agregar/corregir una unidad toca 5 puntos en 2 repos. Consolidar
   (tabla `units` física que alimente CHECKs y FE vía `GET /units`) es la inversión de mejor
   retorno antes de que la divergencia crezca.
2. **Cascada de conversión implementada 3 veces**: SQL `convert_units_for_product`, Go
   (`internal/sale/sale_service.go:424-479`), TS (`src/domain/units/resolveUnitFactor.ts`).
   Cada módulo nuevo debe reutilizar una de estas, no escribir una cuarta.
3. **Migraciones de funciones SQL por dump+anclas**: método ya probado 2 veces pero frágil;
   cada función tocada exige smoke en instancia efímera.
4. **BI normalizado cambia números históricos** de reportes que el cliente ya conoce:
   comunicar antes de desplegar.

---

## 6. Fases propuestas

> Orden sugerido: F1 y F2 cierran bugs y gaps de contrato (baratos, alto valor). F3 baja el
> costo de todo lo que venga después. F4-F6 son decisiones de negocio: posponerlas no rompe
> nada.

### F1 — Cerrar bugs del núcleo (P0)

1. **BE**: migración que parchee `update_sale_item` → `convert_units_for_product` + guard
   EXISTS (mismo método dump+anclas de v1). Smoke: editar línea a unidad con factor específico
   ≠ global → stock descuenta con el específico.
2. **BE**: `validateCompleteRequest` valida `unit` contra `shared.ValidateUnit` y verifica
   factor de conversión temprano (reutilizar `findConversionFactor` / `UnitConversionReader`,
   mismo patrón que ventas).
3. **FE compras**: `PurchaseProductModal` reacciona al cambio de unidad —
   `handleModalUnitChange` que recalcule costo sugerido (costo de la unidad si existe en
   `unit_costs`; si no, costo base ÷ factor, dirección compra), precio/margen sugerido, gating
   de decimales con `isDecimalUnit`, y hint preventivo cuando no hay NI costo NI conversión
   (reusar `resolveUnitFactor`; fetch de conversiones igual que `SalesNew.tsx:1045-1059`).
4. **Tests**: smoke SQL del ítem 1; vitest del modal reactivo (costo cambia con la unidad,
   hint sin conversión, kg decimal no se corrompe).

Esfuerzo: bajo-medio. Gates: BE `go test ./...` + `golangci-lint` + `make build`; FE vitest +
`tsc --noEmit` + `pnpm build` + `lint:design`.

### F2 — Presupuestos y requisiciones completos

1. **FE**: presupuestos con decimales para pesables (gating `isDecimalUnit`, mismo patrón que
   requisiciones `PurchaseRequisitionCreate.tsx:273-278`).
2. **BE**: migración alineando el CHECK de `purchase_requisition_details.unit` al catálogo
   canónico (cierra B3).
3. **BE**: `variant_id` en `budget_order_details` + `purchase_requisition_details` (migración
   de columnas + procedures `create_budget_order`/requisición/`convert_budget_to_sale` + repos)
   — cierra B4; el payload FE ya lo envía (deuda flat-search).
4. **Decidir**: selector de unidad en presupuestos/requisiciones (hoy base fija). Si el
   workflow no lo pide, documentar como decisión.

Esfuerzo: medio.

### F3 — Catálogo único de unidades (consolidación)

1. **BE**: evaluar tabla `products.units` física como fuente única:
   `get_available_unit_values()` la lee; CHECKs de base_unit/requisiciones derivan de ella.
   (Si el refactor de CHECKs históricos resulta caro, mínimo vital: alinear las listas.)
2. **FE**: `UnitSelect` consume `GET /units` (cache por sesión) con fallback al catálogo local;
   reemplaza el Select duplicado de `ProductFormModal` y el texto libre del modal de
   conversiones.
3. **FE**: sincronizar `validMeasurableUnits` (sqft/ton) con el catálogo.
4. **FE**: i18n completa de `UnitConversionsPage`.
5. Opcional: selector de unidad en ajustes de precio/costo por producto (hoy unidad fija de
   fila).

Esfuerzo: medio. Es prerrequisito blando de F4/F5.

### F4 — Inventario con unidades

1. **BE**: `manage_stock_adjustment` con `p_unit` opcional (default base) + conversión;
   columna `stock_transactions.unit` (+ backfill inferido donde se pueda).
2. **FE**: `MovementForm` con selector de unidad (base default) + hint del factor aplicado.
3. **BE+FE**: transferencias con `unit` en `branch_transfer_items` + conversión a base; el
   modal ya muestra stock con unidad.
4. Smoke de descuadres sobre instancia efímera (receta v1: backup + migraciones + BEGIN/ROLLBACK).

Esfuerzo: medio-alto (toca el corazón del inventario).

### F5 — BI con cantidades normalizadas

1. **BE**: analytics suman `quantity` convertida a base_unit (o exponen la unidad por fila).
2. **FE**: mostrar unidad en los reportes clave.
3. ⚠️ Comunicar el cambio de números históricos antes de desplegar.

Esfuerzo: medio.

### F6 — Manufacturing (rediseñado 2026-10-03 tras revisión de diseño — ver §11)

No es solo integrar unidades: la revisión pre-F6 encontró la producción **rota desde
2026-04** (bug `branch_id` probado en vivo), la UI **inexistente** (eliminada en la migración
JS→TS y nunca reconstruida) y **cero uso** del módulo. F6 queda redefinido en 4 sub-fases:
F6.0 reparación del bug → F6.1 unidades → F6.2 reconstrucción FE → F6.3 limpieza.
Detalle completo y puntos de decisión en §11.

Esfuerzo: F6.0+F6.1 medio; F6.2 alto (feature nueva); F6.3 bajo.

### Decisiones explícitas diferidas (documentar, no implementar)

- **`base_unit` editable tras crear**: exigiría convertir stock, precios y costos históricos
  de forma consistente (alto riesgo). Hoy se mitiga creando el producto bien.
- **Unidad por variante**: heredan la base del padre — razonable; los precios/costos sí son
  por variante+unidad.
- **Doble expresión de stock en pickers** ("X cajas / Y kg"): requiere exponer
  conversión+stock por unidad en la búsqueda plana (deuda FE v1 §3).

---

## 7. Verificación (por fase)

- Migraciones de funciones SQL: smoke en instancia efímera (initdb + backup restore +
  migraciones CLI), asserts por presencia de filas, no interpolación de variables.
- Gates estándar: BE `go test ./...` 0 FAIL + `golangci-lint run` + `make build`; FE
  `npx vitest --run` 0 fallos + `npx tsc --noEmit` + `pnpm build` + `pnpm lint:design`.
- Smoke HTTP con dev-JWT (recordar: `role_id` es STRING, `session_id` real de
  `users.user_sessions`).

## 8. Gotchas heredados de v1 (aplican a la ejecución)

- Dumps de `pg_get_functiondef` NO traen `;` — concatenar antes de ejecutar.
- psql NO interpola `:'var` dentro de dollar-quotes.
- **Smokes de unidades previos al fix 09-26 tienen aserciones de precio derivado INVERTIDAS**
  ("5000/20=250 ✓" consagraba el bug): no re-correr sus asserts de derivación.
- SQL puro no requiere recompilar el binario detached; cambios Go sí.
- `unit` en payloads siempre lowercase (CHECKs SQL case-sensitive).
- Repos con WIP ajeno uncommitted → stage por rutas explícitas.
- Tests de Radix Select requieren los stubs de PointerEvent del vitest.setup (ya presentes).

---

## 9. Registro de ejecución (2026-10-02)

### F1 — Cerrado ✅ (BE + FE)

1. **B1 `update_sale_item`**: migración `20261002103000_update_sale_item_per_product_conversion`
   (dump+anclas): guard EXISTS + `convert_units_for_product` en las dos conversiones.
   Smoke data-driven en dev DB (BEGIN/ROLLBACK): conversión GLOBAL box→kg ×10 vs
   ESPECÍFICA ×20 — editar la línea 1→2 box descontó **20 kg** (con la función vieja
   habría descontado 10). Bonus: el trigger de INSERT de `sales_order_details` también
   descontó con el factor específico.
2. **B2 validación temprana compras**: la cascada de conversión Go se movió a
   `internal/shared` (`ScopedConversion` + `FindConversionFactor`, única implementación Go —
   sale y purchase delegan); `purchase.UnitConversion` gana `ProductID`;
   `OrderService.SetConversionReader` + validación en `validateCompleteRequest`
   (`shared.ValidateUnit` + factor elegida→base, fetch lazy, modos degradados nil-safe);
   wiring: `NewPurchaseConversionBridge` (measure→purchase) + `Conversions` en
   `PurchaseDeps`. Tests: 4 casos nuevos (código inválido, sin conversión,
   específica+global, degradado).
3. **Hallazgo P0 NUEVO (F9): la derivación de precio del POS estaba INVERTIDA** —
   `SalesNew.resolveModalUnitOriginalPrice` multiplicaba por `resolveUnitFactor(base→elegida)`
   (recíproco del SQL): 1 ton con base kg a 5.000 mostraba **5** en vez de **5.000.000**;
   es exactamente el bug "417 vs 60.000" que el SQL corrigió el 09-26 pero que quedó
   invertido en FE (la dirección solo se nota con filas de una sola dirección: globals
   están sembradas en ambos sentidos y lo enmascaraba). Fix: dominio nuevo
   `src/domain/units/deriveUnitPrice.ts` (precio registrado → base × factor(elegida→base)
   → base + `matchesUnit=false`), SalesNew delega; 11 tests de regresión.
4. **F1-FE modal compras reactivo**: `usePurchasesLogic` fetchea conversiones al abrir el
   modal (fail-silent) + guarda `unit_costs_summary` por unidad; `handleModalUnitChange`
   recalcula costo (registrado → base × factor vía deriveUnitPrice) y el hint preventivo
   `modalUnitConversionHint` (sin NI costo NI conversión); gating de decimales con
   `isDecimalUnit` al confirmar (floor para enteras, 0 se bloquea); `PurchaseProductModal`
   consume `handleModalUnitChange` + hint (`purchases.product_modal.unit_no_conversion`,
   es/en). Modo edición también carga costos por unidad. Tests: 7 del hook real (mocks en
   frontera de servicios) + 3 del modal (combobox Radix + hint).

### F2 — Cerrado ✅

1. **Presupuestos decimales** (`BudgetCreate.tsx`): gating `isDecimalUnit` igual que
   requisiciones (min/step por unidad + floor onBlur para enteras + mínimo 0,01/1).
   Tests: 3 (0,5 kg sobrevive; 1,5 box → 1; 0,5 box bloqueado).
2. **B3 CHECK requisiciones**: migración `20261002113000_requisition_unit_check_canonical`
   — CHECK alineado a los 33 valores canónicos (mismo orden que `shared.AllowedUnits`).
   Smoke: `pair`/`sqft` entran, `bushel` sigue rechazado.
3. **B4 variant_id**: migración `20261002114000_budget_requisition_variant_id` — columnas
   `variant_id` en `budget_order_details` + `purchase_requisition_details` (FK a
   product_variants); `create_budget_order` persiste variant (guard FIX-2) **y también la
   columna `unit` que descartaba**; `convert_budget_to_sale` incluye variant_id en el
   payload (process_sale_with_reserve ya lo consumía). Go: `BudgetOrderDetail`/
   `RequisitionDetail`/`CreateRequisitionDetailItem` ganan `VariantID` + repos. Smoke E2E:
   presupuesto con variante → venta vende la VARIANTE (antes: producto base).
4. **Decisión documentada**: presupuestos/requisiciones siguen en `base_unit` fija (sin
   selector de unidad) — el workflow de cotización/requerimiento no exige multi-unidad;
   la conversión real ocurre al vender/comprar. Si un módulo futuro la pide, el selector
   es `UnitSelect` + factor (patrón ya probado).

### F3 — Cerrado ✅ (mínimo vital según el propio plan)

1. **Tabla `products.units` física: DIFERIDA.** Los CHECKs SQL no pueden subqueeryar:
   derivarlos de una tabla exige triggers de validación (refactor caro sobre CHECKs
   históricos). Mínimo vital ejecutado: las listas quedaron alineadas — `shared.AllowedUnits`
   (33) = CHECK `products.base_unit` (33) = CHECK requisiciones (33, fix B3) = catálogo FE
   (32+sqft=33). Manufacturing (14) sigue siendo vertical propia (F6).
2. **`UnitSelect` consume `GET /units`** (`{value,description,category}`, cache por sesión
   vía `src/services/unitsService.ts`): la LISTA de valores es canónica del backend; los
   labels/categorías siguen viniendo del catálogo local (es-PY curado); fail-silent →
   catálogo local. Mock global en `vitest.setup.ts` (los tests re-moquean si necesitan filas).
3. **`validMeasurableUnits` sincronizada**: +`ton` (pesable que estaba excluido).
4. **Catálogo FE sincronizado**: +`sqft` (faltaba; el canónico lo tiene).
5. **`ProductFormModal`**: Select duplicado reemplazado por `UnitSelect` (mismo selector que
   ventas/compras; respeta inmutabilidad en edición y error styling).
6. **`UnitConversionsPage`**: inputs de TEXTO LIBRE reemplazados por `UnitSelect`
   (`extraUnits` preserva valores legacy registrados) + **i18n completa** (34 keys es/en;
   fin de los títulos/headers/toasts hardcoded).

### Gates y verificación (2026-10-02)

- **BE**: `go test ./...` 0 FAIL (incl. `tests/integration`), `golangci-lint run` limpio,
  `make build` OK. Migraciones aplicadas a dev DB (`make migrate-up`, versión
  `20261002114000`); objetos verificados (CHECK 33, 2 columnas variant, funciones parcheadas).
  Backend detached relanzado (`server_starting port=:5050`, 0 errores frescos, /health OK).
- **FE**: `npx vitest --run` **1396/1396**, `tsc --noEmit` 0, `pnpm build` OK,
  `pnpm lint:design` "Código nuevo limpio".
- Smokes SQL con BEGIN/ROLLBACK sobre dev DB (sin fixtures persistidos).
- **Fix colateral**: `internal/category/http_test.go` — `mockTax` sin `AutoClassifyCategory`
  (rotura preexistente de c7662a5 que dejaba `go test ./...` sin compilar).

### F4 — Cerrado ✅ (2026-10-02, BE + FE)

1. **Migración `20261002123000_inventory_units_f4`** (dump+anclas desde dev DB):
   - `stock_transactions.unit VARCHAR(20)` — unidad en la que el ORIGEN expresó la
     cantidad; `quantity_change`/`before`/`after` siguen SIEMPRE base. Backfill inferido:
     todo el histórico se expresó en base → `unit = products.base_unit` (0 filas quedan
     NULL). Sin CHECK de catálogo a propósito: no replicar la lista en un sexto lugar
     (riesgo transversal #1); valida Go + guards SQL.
   - `branch_transfer_items.unit VARCHAR(20)` (NULL = base): unidad de TODAS las
     `quantity_*` del ítem (el CHECK de progresión sigue consistente).
   - `register_stock_movement` gana `p_unit` (12° param): convierte el delta a base con
     `convert_units_for_product` y persiste `unit = COALESCE(p_unit, base)`.
   - `manage_stock_adjustment` gana `p_unit` (8°): `p_new_quantity` viene en `p_unit`, se
     convierte a base ANTES del diff; old/new persisten base y el original viaja en
     metadata (`requested_unit`/`requested_quantity`).
   - Trigger `trg_apply_transfer_stock_movements`: convierte `item.unit → base` al
     registrar TRANSFER_OUT/TRANSFER_IN.
   - **Hallazgo del smoke: ambigüedad de overloads** — crear la firma de 12 params sin
     soltar la de 11 deja los callers con NULLs sin tipo sin mejor candidato ("could not
     choose a best candidate", el 12° con DEFAULT hace viable la firma nueva para esas
     llamadas). Fix: `DROP FUNCTION` previo de la firma corta en up y down (mismo
     conflicto de aridad que ya resolvió 20260715162522). Smoke lo cazó antes del commit.
2. **Validación temprana Go (patrón B2/F1)**: `shared.ValidateUnit` + factor con
   `shared.FindConversionFactor` (fetch lazy, degradado nil-safe) en
   `AdjustmentService.Insert`, `TransactionService.Register/RegisterBatch` (fail-fast
   all-or-nothing) y `BranchService.CreateBranchTransfer` (puerto `UnitConversionLookup`
   + bridge measure→identity; `ProductTypeInfo` gana `BaseUnit` via bridge de catalog).
   Wiring perezoso: los accessors del container resuelven el bridge en el primer request
   (SetInventoryDeps corre ANTES que SetMeasureDeps → llamar `MeasureService()` ahí
   panea el arranque; lo demostró el primer relanzamiento).
3. **Repos/HTTP**: `POST /manual_adjustment` y `/stock-transactions` aceptan `unit`
   (el delta/`new_quantity` viaja en ESA unidad); `StockTransaction`/history devuelven
   `Unit`; transfers create/detail persisten y devuelven `unit`.
4. **FE movimientos** (`/movimientos-stock`): fila con `UnitSelect` (base default) +
   hint del factor (`1 Caja = 10 Kilogramo (kg)`) o hint de error sin conversión (bloquea
   Guardar y el submit); gating de decimales por la unidad ELEGIDA; el delta viaja en la
   unidad elegida y el backend convierte. Dominio puro: `buildPayloadFromForm` con
   `unitContext {unit, unitFactor}` (preview y payload comparten el MISMO
   `resolveUnitFactor` — no hay cuarta cascada). Cambiar unidad resetea el valor.
5. **FE transferencias** (`CreateTransferModal`): columna Unidad con `UnitSelect` por
   fila + hint del factor / error sin conversión (bloquea el submit con toast accionable
   en vez de NO_CONVERSION a mitad del workflow); payload lleva `unit` (omitida = base);
   `TransferDetailModal` muestra la unidad del ítem cuando existe;
   `PreloadedTransferItem` gana `base_unit` opcional.
6. **Tests**: BE `service_unit_validation_test.go` (9 casos ajustes/movimientos/batch) +
   `branch_transfer_units_test.go` (validación directa + integración CreateBranchTransfer,
   fake retiene ítems). FE: `movements.test.ts` (6: payload con unidad, target vs base,
   precisión g→kg), `MovementFormUnit.test.tsx` (3: hint error bloquea, hint factor guarda,
   submit entrega unitFactor), `BranchTransfers.test.tsx` +2 (hint+payload con unit,
   sin conversión bloquea el POST).

#### Verificación F4 (2026-10-02)

- Smokes SQL (BEGIN/ROLLBACK, dev DB): ajuste en g (500 g → old 37.20 / new 0.50 +
  metadata requested_unit=g), movimiento 12 args (250 g → +0.25 kg, unit=g), legacy 11
  args (unit=kg), NO_CONVERSION en movimiento y en SHIP de transferencia; transferencia
  g→SHIP/RECEIVE movió ±0.5 kg entre sucursales.
- Smokes HTTP con dev-JWT: ajuste `unit:g` (37.5 ✓), `bushel` → 400 INVALID_INPUT
  temprano, `/stock-transactions` con `unit:g` (delta 0.25 + read-back Unit=g), batch con
  unidad inválida all-or-nothing, transferencia creada con `unit:"g"` (id 231, cancelada
  para cerrar; stock restaurado a 37.2). Nits documentados: `/manual_adjustment` mantiene
  `new_quantity: int` (contrato legacy; decimales de base van por /stock-transactions,
  camino que ya usa el FE) y el rechazo temprano de transfers responde 500 (mapeo de
  errores de CreateBranchTransfer es preexistente para todas sus validaciones).
- Gates BE: `go test ./...` 0 FAIL, `golangci-lint run` limpio (fix colateral: gofmt
  preexistente en `internal/sale/domain.go`, `internal/purchase/domain.go`,
  `sale_service_test.go`, entró con c315ade), `make build` OK; backend detached
  relanzado y sano (`server_starting port=:5050`, /health OK).
- Gates FE: vitest **1407/1407**, `tsc --noEmit` 0, `pnpm build` OK,
  `pnpm lint:design` "Código nuevo limpio". Audited contra vercel-react-best-practices
  (fetch único al montar, useQuery staleTime, sin hallazgos).

### F5 — Cerrado ✅ (2026-10-02, BE + FE)

1. **Migración `20261002150000_bi_normalized_units` (ADITIVA, sin dump+anclas):**
   - Función `products.convert_qty_to_base(product_id, quantity, unit, base_unit)`: fast-path
     identidad (unit NULL o = base, el caso dominante — sin plpgsql ni lookups por fila),
     delega en `convert_units_for_product` (la cascada canónica; NO hay cuarta implementación)
     y **devuelve NULL en NO_CONVERSION** → los SUM/AVG saltan la fila en vez de tumbar el
     reporte por datos históricos sin factor (el FE bloquea ese alta desde F1/F4).
   - `v_pending_transfers` (deuda F4/B7): suma `quantity_*` por ítem convertido a base
     (join products; los ítems pueden traer unidad elegida desde F4). Sin consumidores Go/FE
     hoy — cambio sin riesgo.
2. **BE normalización (~35 sitios en 7 repos)**: helper Go `biBaseQtyExpr`/`sdBaseQty`
   (`analytics_sql.go`, junto a `sdRevenue`/`sdCost`) inyecta la expresión; los
   `quantity * unit_price` (dinero) quedan intactos por ser agnósticos a unidad. Alcance:
   `analytics_sales_repo` (13: performance, trends, by-category/product, top-bottom+HAVING,
   by-seller, velocity×3+HAVING, compare, dashboard×3), `analytics_sales_relational_repo`
   (top-products compartido + buyers; **exponen `base_unit` por fila**), `analytics_purchase_repo`
   (ProductSuppliers: units + denominador de avg_unit_price → plata por unidad base),
   `analytics_inventory_repo` (turnover×2, ABC, daily_demand de reorder/forecast — normalizar
   la demanda CORRIGE `days_until_stockout`, que comparaba stock base contra demanda en unidad
   elegida —, slow movers), `analytics_dashboard_repo` (top products), `analytics_financial_repo`
   (margins ×4), `analytics_forecast_repo` (sales_velocity, recent_trend, demand por
   categoría/producto) y `analytics_profitability_repo` (×5).
3. **Modelos int→float64** solo en campos alimentados por las sumas convertidas (13 campos:
   SalesPerformance.TotalUnits, SalesTrendPoint.Units, CategorySalesData/ProductSalesData/
   SellerSalesData.UnitsSold, SalesComparisonPeriod.TotalUnits + Diff.UnitsChange,
   CategoryTurnover/ProductTurnover/ABCProduct.UnitsSold, CategoryMargins/ProductMargins.UnitsSold,
   ProfitabilitySummary.TotalUnitsSold + ProductProfitabilityDetail/CategoryProfitability/
   TrendPoint.UnitsSold, CategoryDemand/ProductDemand.HistoricalUnits). Go serializa 37 como
   `37` — el JSON solo cambia con decimales pesables. ProductContribution queda int (sin
   consumidor). Filas relacionales ya eran float64: solo ganan `base_unit` (`json:"base_unit,omitempty"`).
4. **FE**: celda `units` de los 4 drill-downs muestra la UOM real ("40 kg") vía helper puro
   `formatDrilldownUnits` (fallback "N uds." sin base_unit — fixtures viejos siguen verdes);
   menú de fila con la misma UOM (`bi.relational.menu.summary` pierde el "uds." horneado);
   contadores AGREGADOS relabelados a "(base)": ProductsCategories KPI (`bi.sales.col.unitsBase`),
   CustomerSellerInsights columna vendedores + PeriodComparison card (`bi.insights.col.unitsSoldBase`,
   celda pierde el sufijo " uds." que mentía con kg). Tablas por-producto quedan "Unidades"
   (cada fila es la base de ESE producto). Keys nuevas es/en.
5. **⚠️ Números históricos cambian** (F5.3): los contadores de unidades dejaron de mezclar
   kg/cajas/unidades — comunicar al cliente antes de desplegar. El A/B del ABC lo cazó en
   dev: productos con ventas reales en unidad ≠ base difieren (17 → 17,4).

#### Verificación F5 (2026-10-02)

- Smoke SQL (dev DB, BEGIN/ROLLBACK): conversión específica carton→kg ×20 → 2 carton + 3 kg
  = **43 kg**; sin conversión la fila carton se EXCLUYE (sum=3); fast-paths identidad; vista
  v_pending_transfers ejecuta (4 filas, TRF-2026 fixtures con decimales).
- Smoke HTTP (dev-JWT real user/role): **30 endpoints 200** (performance, trends, by-*,
  top-bottom, velocity, compare, dashboard sales, inventory ×8, profitability ×6,
  financial profit-margins, forecast ×4, dashboard top-products) + 4 drill-downs devolviendo
  `base_unit` por fila (ej. "PAPA KG PY" units 25 base_unit "kg").
- Gates BE: `go test ./...` 42 paquetes OK 0 FAIL, `golangci-lint run` limpio (gofmt
  colateral en sales_repo/relational.go), `make build` OK; backend detached relanzado y sano
  (`server_starting port=:5050`, /health OK). Fix de tests: `nonsargable_ab_test` alinea el
  lado "legacy" del A/B al cálculo normalizado (el test compara la EQUIVALENCIA del refactor
  CTE, ya no la semántica de unidades) y escanea float.
- Gates FE: vitest **1408/1408** (+1 test de UOM en celda y menú), `tsc --noEmit` 0,
  `pnpm build` OK, `pnpm lint:design` "Código nuevo limpio".

### Pendiente (decisiones de negocio, no rompe nada posponer)

- **F6** manufacturing (vertical propia).
- Deuda menor heredada: B8, B9 (`resolve_tax_rate_for_transaction` ignora `p_unit`),
  F6-FE (tickets client-side sin unidad), selector de unidad en ajustes de precio/costo
  (opcional F3.5), `new_quantity` int en `/manual_adjustment` (decimales de base →
  /stock-transactions), casts `::int` pre-existentes en contadores de `stock_movements`
  (ya base; truncan decimales kg — fuera del alcance B7), fallback "N uds." hardcodeado en
  `formatDrilldownUnits` (i18n pendiente si algún día se localiza el sufijo).
- Nota de operación: `TestAuditBranchScopeIntegracion` es estado-acumulativo contra dev DB
  (falla con N×corridas, pasa aislado con `-count=1`) — flakiness PRE-EXISTENTE detectado
  en esta sesión, no relacionado con F5.

---

## 10. Cierre — reparos de auditoría ejecutados (2026-10-03)

Los 2 reparos de F4 hallados por `AUDIT_UNITS_V2_F1_F5_2026-10-03.md` fueron ejecutados
y verificados, completando el cierre de F1-F5:

1. **A1+A2 — ajustes manuales float64 end-to-end** (BE `fd4e7bf`):
   `ManualAdjustment.New/OldQuantity` y el payload de `POST /manual_adjustment` pasaron de
   `int` a `float64` (el parámetro SQL `p_new_quantity` ya era `numeric` — Go era el único
   clamp); `AdjustmentRepo.List` escanea las columnas `numeric` a `sql.NullFloat64` (antes
   `NullInt64` → 500 con filas fraccionales); el listado ahora expone `Unit` recuperado de
   `metadata.requested_unit` (`null` en filas pre-F4). **Smoke en vivo contra la fila
   disparadora**: `GET /manual_adjustment/1/10` con binario relanzado → 200 con
   `OldQuantity:37.2, NewQuantity:37.5, Unit:"g"` (antes: 500). La fila id=9 de smoke se
   PRESERVA conscientemente: documenta un cambio de stock real (borrarla descuadraría
   historia vs stock).
2. **A3 — gating decimal en movimientos de stock** (FE `310bcaa`): `truncateForUnit`
   (`Math.trunc`, hacia cero — nunca mueve más stock del tipeado, también en deltas
   negativos) aplicado al guardar la fila y como guardia en el submit; "0,5" en unidad
   entera NO se pierde silenciosamente: bloquea con `stockMovements.errors.integer_unit_zero`
   (es; en hereda por spread). Misma política que POS/compras/presupuestos/requisiciones.
   +2 tests (trunca 1,5→1 al guardar; 0,5 bloquea el registro).
3. **Gates de cierre**: BE `go test ./...` 0 FAIL + `golangci-lint` limpio + `make build`;
   FE vitest **1410/1410** (+2), `tsc --noEmit` 0, `pnpm build`, `lint:design` limpio.
   Backend detached relanzado (:5050, `server_starting` fresco, /health OK).

**Queda pendiente solo F6 (manufacturing)** + la deuda menor documentada en la auditoría
(A4-A8: silent bump transferencias, avg filas sin factor, fallbacks "uds." stale, snapshot
schema, sugerencia de precio no re-derivada) y la nota operativa de comunicar al cliente el
cambio de números históricos de BI antes de desplegar.

---

## 11. F6 — Manufactura: revisión de diseño y alcance rediseñado (2026-10-03)

Verificación pedida por el owner antes de ejecutar F6: propósito, diseño actual y mejora.
Método: 2 exploraciones (BE+SQL, FE) con file:line + verificación en vivo en dev DB.

### 11.1 Propósito y arquitectura actual

El módulo responde a **producción propia con costeo**: comprar insumos (harina, azúcar),
registrar una receta por producto fabricado, registrar producción manual que **incrementa el
stock del producto terminado** vía la maquinaria central (`register_stock_movement` tipo
`PRODUCTION`), y reportar rentabilidad (costo de producción vs precio de venta).

**La decisión arquitectónica central es BUENA y conviene conservarla**: los insumos son un
**canal de costos deliberadamente aislado** del inventario (comentarios en el schema: "NO
forman parte del inventario", supply_purchases "solo auditoría y control de costos"), y la
producción es una envoltura fina sobre el stock core. Eso evita duplicar la lógica de
inventario y deja el producto terminado vendible por el flujo normal (`product_type
'PRODUCTION'`). Piezas: HTTP fino → service de validación mínima → repo SQL + 2 funciones
plpgsql (`register_supply_purchase`, `register_manual_production`) → vistas branch-aware
para reportes. Sin triggers, sin lifecycle de lotes, sin consumo de insumos al producir
(la receta es "informativa, NO se valida" — comentario del propio schema).

### 11.2 Hallazgos de la revisión (verificados)

| # | Hallazgo | Evidencia | Severidad |
|:--|:---|:---|:---|
| M1 | **La producción está ROTA desde 2026-04-02**: `register_manual_production` inserta `manufacturing_batches` SIN `branch_id`, columna NOT NULL desde `20260402154500` — todo llamado falla (excepción → `success:false`). Además su SELECT de stock nuevo no filtra por sucursal. Verificado en vivo: función sin `branch_id` + tabla `is_nullable=NO`. | baseline `:3336-3342`; INSERT vivo sin branch_id; 0 filas en batches | 🔴 |
| M2 | **La UI NO existe**: el panel JSX (2025, tab del modal de producto) se eliminó en la migración JS→TS (9d225b6c jun-2026, 4a0f05d8 ago-2026) y nunca se reconstruyó — 0 rutas, 0 servicios. El usuario hoy solo puede crear productos PRODUCTION y ajustarles stock manualmente. | FE: grep manufactur en App.tsx/navigation = 0 | 🔴 |
| M3 | **Cero uso**: dev DB con 0 batches, 0 supplies, 0 supply_purchases, 0 recipes; 1 producto PRODUCTION. Nadie extrañó M1/M2 → confirma módulo sin usuarios. | DB en vivo 2026-10-03 | 🔵 contexto |
| M4 | **Unidades aisladas (6ª copia de la lista)**: CHECK de `manufacturing_supplies` con 14 valores propio; unit texto libre en Go (default "kg" SOLO al crear — update con vacío revienta el CHECK); sin `shared.ValidateUnit`, sin conversiones; `quantity_per_unit` de la receta mezcla unidad de insumo vs base del producto sin conversión. | `service.go:25-27`; baseline `:15146` | 🟠 (era el F6 original) |
| M5 | **Costeo estimado huérfano**: `calculate_estimated_production_cost` (Σ receta × AVG costo de compras) existe en SQL sin caller Go ni ruta — la pieza más valiosa del costeo nunca se expuso. | `manufacturing_system.sql:516-578` | 🟠 |
| M6 | Código muerto/drift: 3 métodos del servicio sin ruta (GetSupplyByID, GetSupplyPurchaseByID, GetManufacturingBatchByID); permisos duplicados 125/126 vs 150/151; drift `MANUFACTURED` (Go/vistas) vs `PRODUCTION` (CHECK vivo); `products.tabs.manufacturing` key i18n muerta; script de diseño con vistas reemplazadas. | ver informes de exploración | 🟡 |
| M7 | Tests: 3 tests de shape de error solamente; sin tests de compras/producción/recetas/repo. | `http_test.go` | 🟡 |

### 11.3 F6 rediseñado

> F6 original ("integrar unidades, vertical propia") puliría un módulo muerto. El rediseño
> ataca primero lo que lo mantiene muerto (bug + UI) y después integra unidades.

- **F6.0 — Reparación (pre-requisito, sin esto nada del módulo funciona)**: migración que
  reescriba `register_manual_production`: pasar `p_branch_id` (o derivarlo del usuario como
  las vistas branch-aware), incluirlo en el INSERT, y filtrar por sucursal el SELECT de
  stock. Smoke BEGIN/ROLLBACK en dev (producción OK + lote con branch + stock del producto
  en su sucursal). Esfuerzo: bajo.
- **F6.1 — Unidades (scope original de F6)**: CHECK de `manufacturing_supplies` → 33
  canónicos (cierra la 6ª copia); `shared.ValidateUnit` en Go para supply/purchase/recipe +
  default "kg" también en update (o unit requerida); producción acepta `p_unit` → convierte
  a base con `convert_units_for_product` (producto terminado sigue en base); receta guarda
  la unidad del insumo y el costeo convierte (insumo→base del producto) usando la cascada
  canónica; exponer `calculate_estimated_production_cost` (cierra M5). Esfuerzo: medio.
- **F6.2 — Reconstrucción FE (la razón real de que el módulo esté muerto)**: vertical nueva
  en "Logística e Inventario" (navigation.ts:316+), patrón card-con-tabla §6.3 +
  PageHeader/EnhancedModal + i18n es/en + `UnitSelect` con hints de factor (patrón F4 ya
  probado en MovementForm). Scope propuesto (MVP→completo): listado de insumos con CRUD +
  compras de insumos; receta por producto (editor — nunca existió) + costo estimado;
  registrar producción (cantidad + unidad + costo + notas → batch); resumen de producción y
  rentabilidad. Gated `manufacturing:read/write`. Esfuerzo: alto (feature nueva).
- **F6.3 — Limpieza**: rutas para los 3 métodos muertos o DELETE de ellos (decidir);
  permisos duplicados 125/126 fuera; unificar `MANUFACTURED`→`PRODUCTION` en Go/vistas;
  key muerta `products.tabs.manufacturing`; actualizar docs API (MANUFACTURING_API.md);
  tests BE (service con mocks por puerto — hoy casi inexistentes) + FE del feature.
  Esfuerzo: bajo.

### 11.4 Puntos de decisión del owner (bloquean F6.2, no F6.0/F6.1)

1. **¿Scope del FE?** MVP (insumos + producción + resumen) vs completo (+ receta editable +
   compras de insumos + rentabilidad). Recomendado: completo — sin receta y sin compras, el
   costeo estimado (M5) no tiene datos de dónde salir.
2. **¿Los insumos deben descontar stock real del inventario?** Diseño actual: NO (canal de
   costos). Recomendado: mantener — convertir insumos en `products` con stock sería otro
   proyecto (inventario doble, compras, ventas de insumos).
3. **¿La receta se valida/consume al producir?** Actual: informativa. Recomendado:
   mantener informativo en esta fase; el consumo real exige decisión 2.

Orden de ejecución sugerido: F6.0 → F6.1 → F6.3 (barato, mientras el FE no existe) → F6.2.
