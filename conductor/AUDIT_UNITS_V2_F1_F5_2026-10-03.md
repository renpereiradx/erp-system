# AUDIT_UNITS_V2_F1_F5 — Auditoría independiente de la ejecución F1-F5

**Fecha:** 2026-10-03 · **Auditado:** PLAN_UNITS_V2_COMPLETACION_2026-09-30.md, fases F1-F5
**Commits auditados:** BE `1155f0e` (F1), `58929f5`+`c315ade` (F2), `433ffca` (F4), `080c388` (F5) · FE `d304928` (F1.3a), `e3a1f1c` (F1.3b), `b6eabf7` (F2.1), `fc2d97f` (F3), `bf21f30` (F4), `5512f80` (F5)
**Método:** verificación independiente de cada claim contra código (2 auditorías BE/FE con file:line), re-ejecución de los gates, y chequeo en vivo de la DB dev (migraciones, constraints, funciones, datos).

---

## 1. Resumen ejecutivo

| Fase | Veredicto |
|:---|:---|
| F1 — bugs del núcleo (`update_sale_item`, validación compra, modal compras, POS derivación) | ✅ **APROBADA** |
| F2 — presupuestos decimales, CHECK requisitions, `variant_id` | ✅ **APROBADA** |
| F3 — catálogo canónico `GET /units`, UnitSelect unificado, i18n | ✅ **APROBADA** |
| F4 — inventario con unidades (ajustes, movimientos, transferencias) | ⚠️ **APROBADA CON REPAROS** — 1 bug 🔴 activo + 1 🟠 |
| F5 — BI normalizado a base | ✅ **APROBADA** |

La ejecución coincide con lo alegado en la gran mayoría de los claims; la matemática de
conversiones (la familia de bugs histórica) está **correcta y verificada en ambas direcciones**,
no existe una cuarta cascada, y no hay regresiones de firmas SQL. **Se encontró 1 bug activo**
en el listado de ajustes manuales (endpoint 500 con datos que ya existen en dev) y 1
inconsistencia de gating en el FE de movimientos. Ambos tienen fix trivial. En varios puntos
la cobertura real es MAYOR que la alegada (ver §5).

## 2. Gates re-ejecutados (2026-10-03)

- **BE**: `go test ./...` 0 FAIL (incl. `tests/integration`), `golangci-lint run` limpio,
  `make build` OK. Nota: el binario compila como `dirty` por WIP de deploy sin commitear en
  el repo BE (4 archivos `deploy/*` + `docker-compose.yml`, +81 líneas — ajeno a unidades).
- **FE**: `npx vitest --run` **1408/1408**, `npx tsc --noEmit` 0 errores, `pnpm build` OK,
  `pnpm lint:design` "Código nuevo limpio".

## 3. Verificación en vivo de la DB dev

| Objeto | Resultado |
|:---|:---|
| `schema_migrations` | versión `20261002150000`, `dirty=false` ✅ |
| CHECK `transactions.purchase_requisition_details.unit` | **33 valores canónicos** contados uno a uno (incluye los 15 que faltaban: pair, set, case, bundle, sqft, roll, month, tray, bottle, can, jar, carton, stick, slice, portion) ✅ |
| `variant_id` en `budget_order_details` + `purchase_requisition_details` | presentes ✅ |
| `stock_transactions.unit` | columna presente, backfill completo: **0 NULLs** ✅ |
| `branch_transfer_items.unit` | presente ✅ |
| `register_stock_movement` | firma de **12 params** (p_unit) ✅ |
| `manage_stock_adjustment` | firma de **8 params** (p_unit) ✅ |
| `update_sale_item` | definición vigente contiene `convert_units_for_product` (patch F1 vivo) ✅ |
| `convert_qty_to_base` / `v_pending_transfers` | presentes ✅ |

## 4. Hallazgos

### 🔴 A1 (ACTIVO) — `GET /manual_adjustment/{page}/{pageSize}` devuelve 500 con filas fraccionales

Cadena completa verificada:

1. Desde F4, `POST /manual_adjustment` acepta `unit` — el payload exige `new_quantity: int`
   (🟠 A2 abajo), pero **el ajuste en unidad ≠ base produce valores base fraccionales**: ajustar
   "37500 g" con base kg persiste `old_quantity=37.20`, `new_quantity=37.50`.
2. Esas columnas son `numeric` en la DB (verificado).
3. `AdjustmentRepo.List` escanea `old_quantity`/`new_quantity` a `sql.NullInt64`
   (`internal/platform/postgres/adjustment_repo.go:80-83`) → el scan de `"37.50"` a int64
   falla → `GET /manual_adjustment/{page}/{pageSize}` (`internal/inventory/http.go:354`)
   responde 500 **con esa fila en la página**.
4. **La fila ya existe en dev**: `manual_adjustments.id=9` (producto `zveesdXvR`, 37.20→37.50,
   reason "smoke F4 ajuste en gramos", metadata `requested_unit=g` — residuo del smoke HTTP de
   F4 que no se limpió).

Mitigantes verificados: el FE **no consume** ese endpoint hoy
(`stockMovementsService` declara explícitamente no usar `/manual_adjustment/*`), y
`ProductHistory` está a salvo: escanea `OldValue/NewValue/ValueChange` como `float64`
(`internal/inventory/domain.go:94-96`) sobre `get_adjustment_history()` que devuelve `numeric`.
Aun así el bug es alcanzable por cualquier consumidor API/test, y la condición disparadora es
un workflow normal (contar stock en gramos de un producto con base kg).

**Fix sugerido (trivial):** scan a `sql.NullFloat64` + `ManualAdjustment.Old/NewQuantity`
float64 + exponer `Unit`/`requested_*` de metadata en la respuesta. Limpieza: borrar/preservar
conscientemente la fila id=9 de smoke.

### 🟠 A2 — ajustes manuales siguen tratando la cantidad como `int` en Go/HTTP

`ManualAdjustment.NewQuantity int` (`internal/inventory/domain.go:82`) y payload `int`
(`http.go:313`). Inconsistente con movimientos (`float64`, `http.go:444`) y transferencias.
No bloquea el workflow en-unidad (el int viaja en la unidad elegida), pero impide ajustes
fraccionales en la unidad elegida ("0.5 box"). Fue anotado como nit en el registro F4, pero
combinado con A1 forma el mismo patrón de deuda. Fix: float64 en dominio+payload (+A1).

### 🟠 A3 (FE) — gating decimal no enforceado en movimientos de stock

`MovementForm.tsx`: el atributo `step` del input (línea 554) es la única barrera;
`handleSubmit` (221-256) NO aplica `isDecimalUnit`/floor/bloqueo sobre el valor — un delta
"0.5" en unidad entera se envía tal cual (el BE lo acepta como float). Inconsistente con la
política de todo el sistema (POS/compras floor+bloqueo, presupuestos floor, requisiciones
floor, transferencias round+clamp). Fix: mismo patrón que `usePurchasesLogic.ts:846-848`.

### 🟡 Menores

- **A4**: transferencias hacen `Math.max(1, Math.round(qty))` silencioso para enteras
  (`CreateTransferModal.tsx:228-230`): 0.4 → 1 sin aviso (compras bloquea, presupuestos
  rechaza) — riesgo de mover más stock del tipeado.
- **A5**: `avg_unit_price` (purchase `:125`, relational `:124/:214`) con numerador
  (todas las filas) y denominador (`sdBaseQty`, salta filas sin factor) desalineados → promedio
  inflado si existen filas históricas sin conversión. Aceptable documentado; ideal COALESCE
  por fila.
- **A6**: fallbacks stale de `t()` con "uds." horneado en las 4 páginas BI (inerte hoy, la
  clave existe es+en; produce doble UOM si la clave se pierde).
- **A7**: `database/schema/functions/stock_transaction_functions.sql` no refleja las firmas
  post-F1/F4 (snapshot de referencia desactualizado).
- **A8**: sugerencia de precio de venta (costo×1.3) no se re-deriva al cambiar unidad en el
  modal de compras (`usePurchasesLogic.ts:723`); solo reaccionan margen y `effectiveSalePrice`.
- ⚪ cosméticos: comentarios de `20261002113000` con conteos erróneos (15 valores, no 14/19);
  test vacuo `unit_conversion_test.go:28` (consulta `bag→kg`, resuelve por specific-direct y
  nunca evalúa el caso nombrado); `unitsService` re-fetcha por montaje si el backend devuelve
  lista vacía; NaN en cantidad de presupuestos al vaciar input (preexistente); smoke F4 dejó
  residuo en dev (fila id=9, ver A1).

### A favor (cobertura real > alegada)

- F5: **≈51 sitios** convertidos (registro decía "~35"); **18 campos** int→float64 (decía 13).
  Ningún sitio de suma de cantidades de detalles quedó sin convertir; expresiones de dinero
  intactas; HAVING/AVG convertidos; todos los queries tienen el `JOIN products` requerido.
- F3: **45 claves** i18n es/en exactas (decía ~34); cero español hardcodeado en
  `UnitConversionsPage`.
- F4-FE: **8** tests de dominio nuevos (decía ~6).

### Verificaciones negativas importantes (sin hallazgos)

- **Matemática de derivación/direcciones**: `deriveUnitPrice` multiplica por
  factor(elegida→base) — correcto para venta y costo; `update_sale_item` convierte
  venta→base antes del delta; `manage_stock_adjustment` convierte **antes** del diff;
  trigger de transferencias convierte item→base. Sin cuartas cascadas: única implementación
  Go en `internal/shared/unit_conversion.go:19` con precedencia idéntica al SQL; FE comparte
  `resolveUnitFactor`.
- **Sin regresiones de firmas SQL**: ~80 callers históricos de
  `register_stock_movement` (9/11 args) resuelven por DEFAULT del 12º param; el DROP de la
  firma corta está en up y down de `20261002123000`; `update_sale_item` mantiene firma de 5;
  ninguna migración posterior re-define las funciones.
- CHECK requisitions y `shared.AllowedUnits`: 33/33 idénticos en contenido y orden.
- `variant_id` end-to-end verificado: SQL (INSERT + guard de variante válida) → repos →
  dominios → HTTP request DTO; `convert_budget_to_sale` lo incluye en el payload.
- Contrato transferencias FE↔BE: unit omitida cuando es base; BE trata NULL/'' = base.
- i18n: todas las claves nuevas existen es+en (o heredan por spread declarado).

## 5. Recomendación

1. **Reparo rápido antes de cerrar F1-F5** (una sesión chica): A1+A2 (float64 end-to-end en
   ajustes + scan NullFloat64 + exponer unit) y A3 (gating en submit de movimientos). Con eso
   F4 pasa a ✅ plena.
2. A4-A8 quedan como deuda menor documentada (ninguna rompe flujo actual).
3. **F6 (manufacturing)** permanece pendiente, sin cambios en su evaluación.
4. Comunicar al cliente el cambio de números históricos de BI (F5) antes de desplegar.
