# PLAN_UNITS_V2_COMPLETACION — Unidades de medida: análisis de viabilidad y fases de completación

**Fecha:** 2026-09-30 · **Estado:** 📋 Propuesto (hallazgos verificados contra código al 2026-09-30; ejecución pendiente de aprobación)
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

### F6 — Manufacturing (opcional)

Integrar `base_unit` + conversiones en recetas/supplies; unificar su lista CHECK con el
catálogo. Vertical propia.

Esfuerzo: alto.

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
