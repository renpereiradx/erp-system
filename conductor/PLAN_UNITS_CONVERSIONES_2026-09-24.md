# PLAN_UNITS — Conversiones de unidad por producto (BE + FE)

**Fecha:** 2026-09-24 · **Estado:** ✅ Ejecutado y verificado
**Planes de detalle:** `business_management/conductor/PLAN_UNITS_BACKEND.md` ·
`erp-webapp/conductor/PLAN_UNITS_FRONTEND.md`
**Commits:** BE dev `6d759a5` (2 migraciones 20260924105743/44) · FE dev `4d52a85`

## Qué se ejecutó

1. **Conversiones por producto** (deuda estructural): `unit_conversions.product_id`
   nullable (NULL = global), resolución en cascada específico→global en
   `convert_units_for_product`, adoptada por trigger de venta, procedures de
   venta/compra y reversa.
2. **Bug de inventario en `revert_sale`**: restauraba la cantidad en unidad de venta
   contra stock en unidad base, sin variante (cancelar 1 caja = 20 kg sumaba +1 al
   stock base del producto). Ahora restaura la cantidad exacta del movimiento SALE
   original. Diagnóstico en dev: 0 casos históricos → sin backfill.
3. **Validación Go de stock** convertía mal: 500 g contra 1 kg era rechazado. Ahora
   convierte antes de comparar (misma precedencia que SQL).
4. **Derivación de precio al vender** (paridad con compra): sin precio exacto para la
   unidad, deriva del precio base con el factor (productos y variantes).
5. **GET /units**: catálogo canónico expuesto (antes función SQL rota y sin
   consumidores; arreglada en la misma migración).
6. **Frontend**: selector unificado (`UnitSelect`) en ventas y compras (adiós a los
   datalists divergentes); el POS recalcula precio por unidad (`unit_prices`) y
   advierte unidades sin precio; presupuestos/requisiciones envían la unidad real;
   transferencias aceptan decimales para pesables; página admin de conversiones con
   alcance por producto.

## Verificación

Smoke E2E SQL en instancia efímera PG16 (cascada, conversión venta, precio derivado,
reversa exacta incl. variantes, rechazo por stock convertido) · smoke HTTP :5050
(GET /units, CRUD con product_id, rechazo de "caja") · gates BE (go test 40 pkgs,
build) y FE (vitest 1132/1132, tsc, build, lint:design) · migraciones aplicadas a dev.

## Deuda registrada

Manufacturing sin conversiones (vertical propia) · cleanup de
`process_sale_with_reserve` viola FK si falla post-INSERT de details (preexistente) ·
`stock_transactions` sin columna unit · UnitSelect aún usa catálogo local FE (migrar
a GET /units) · página de conversiones sin migrar a i18n/Design full.
