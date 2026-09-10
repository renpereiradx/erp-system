# PLAN: Monorol explícito, descuentos auditables y cuenta corriente de cliente

**Fecha:** 2026-09-09
**Estado:** ✅ IMPLEMENTADO (fases A, B, C) — ver §6. Pendientes: A0 en PROD (SQL para el owner), C7 opcional
**Alcance:** Cross-repo (backend `business_management` + frontend `erp-webapp`) → este plan vive en `conductor/` raíz
**Origen:** Tres hallazgos colaterales de la sesión dual-role F2VLso (ver `FIX_DUAL_ROLE_ADMIN_2026-09-09.md`), planteados por el owner:
1. El JWT lleva `roles[0]` sin regla visible para el usuario → cualquier dual-role futuro es un bypass invisible.
2. El rol vendedor puede aplicar descuentos por ítem; la DB debe quedar bien normalizada para estadística de ventas con descuento y **quién lo aplicó**.
3. `PENDING` en ventas: ¿no sería mejor tratarlo como **crédito** que suma a la deuda total del cliente? ¿Cómo lo implementamos en BD? ¿Cómo sabemos qué monto de esa deuda corresponde a un rango de vencimiento?
**Verificación:** Todas las afirmaciones de §1 fueron verificadas contra código y DB de dev el 2026-09-09 (evidencia `file:line` y queries sobre `erp_db`).

---

## 1. Hallazgos (estado actual, con evidencia)

### H1 — Multi-role: el JWT toma `roles[0]` con una regla que nadie ve

- `internal/identity/service.go:147-156` (Login) y `:288-293` (Refresh): si el usuario tiene varios roles, el token se emite con `roles[0]`. El TODO existe desde la migración FASE 11 (`D11-5: mono-role preserved, TODO legacy`).
- `internal/platform/postgres/identity_repo.go:110-117`: `GetUserRoles` hace `ORDER BY ur.assigned_at` → **`roles[0]` = el rol asignado más antiguo**. Esa es la regla invisible: no está en ninguna UI ni doc; cambiar el "orden" de asignación cambia los permisos efectivos del usuario sin que nadie mueva nada.
- `users.user_roles` (dev): `UNIQUE(user_id, role_id)`, multi-role permitido, **sin** `is_primary` ni constraint de rol único. El FE crea usuarios con un rol (`UserFormModal.tsx:141` manda `role_ids: [values.role]`), pero `ManageRolesPanel` + `POST /users/{id}/roles` permiten agregar un segundo rol (el backend solo protege el último rol restante, `ErrCannotRemoveLastRole`).
- Autorización real hoy = **unión de dos fuentes que pueden discrepar**: los permisos se resuelven por unión de TODOS los roles (`PermissionService.GetUserPermissions`), pero el claim `role` del JWT es uno solo — y el bypass de admin lee ese claim: `middleware/module_permission.go:41,105` (`claims.RoleID == constants.RoleAdmin` → bypass total del gate de permisos). También `internal/identity/service.go:1330` (transfers) y `http_branch.go:40,1380`.
- Caso real ya ocurrido: F2VLso dual-role (ADMIN asignado primero + VNDR) → JWT admin → "el vendor puede descontar". Limpieza registrada en `FIX_DUAL_ROLE_ADMIN_2026-09-09.md` (3656 usuarios de prueba).

**Conclusión H1:** mientras ambas cosas sean ciertas (multi-role permitido en DB + JWT mono-rol), todo dual-role es una bomba. Hay que elegir: mono-rol por constraint, o rol primario explícito. Ver D1.

### H2 — Descuentos: la estadística "quién descontó cuánto" solo vive en JSONB

- `transactions.sales_order_details.discount_amount numeric(12,2) DEFAULT 0` **existe pero está muerto en la práctica**: 0 de 121 líneas en dev con valor > 0. `process_sale_with_reserve` NO lo escribe en el INSERT — pliega el descuento al `unit_price` (semántica canónica C-3, migración `20260821145151_iva_audit2_fix_discount_amount_per_line`).
- La única traza de descuento con usuario y razón es `sales_orders.metadata->'discounts'` (JSONB), escrito por `create_discount_metadata(... p_user_id, v_discount_reason)` desde `process_sale_with_reserve`: 11 ventas en dev la tienen. Es auditoría de texto, no datos consultables.
- `public.v_sales_ledger_sifen` suma `total_line_with_tax` (ya viene descontado) → no expone el descuento para reportes.
- **Paridad rota con presupuestos:** `transactions.budget_order_details` SÍ tiene `discount_amount` + `discount_percent` normalizados; la venta los pierde.
- No existe columna alguna de "quién autorizó/aplicó el descuento" (`sales:apply_discount` es permiso 186, gate server-side en `internal/sale` — pero el sujeto queda solo en metadata).
- Consecuencia: reportar "descuentos por vendedor/aplicador/sucursal/período" exige parsear JSONB línea por línea. Es exactamente lo que el owner pide evitar.

### H3 — `PENDING` no es crédito: sin deuda por cliente, sin vencimiento real, cobro por venta suelto

- `transactions.sales_orders.status` CHECK: `PAID | PENDING | PARTIAL_PAYMENT | CANCELLED`. `PENDING` = venta creada sin pago; `PARTIAL_PAYMENT` = abonada parcial. El "saldo abierto" de un cliente = suma de `total_amount − pagos` de esas dos — hoy no existe materializado en ningún lado (ni columna ni vista ni endpoint).
- **Cobros anclados a UNA venta:** `sale_payments.sales_order_id NOT NULL`; `process_sale_partial_payment` limita cada pago al saldo de esa venta (`AMOUNT_EXCEEDS_BALANCE`). El flujo del owner — cliente debe 2×250.000 y paga 400.000 — exige registrar 2 cobros manuales (250.000 + 150.000). Funciona, pero el "pago de 400.000 del cliente" no existe como entidad.
- **Sin vencimiento:** `sales_orders` no tiene `due_date`; `users.parties` no tiene `credit_terms_days` ni `credit_limit` (22 columnas, ninguna de crédito).
- **El aging actual es una aproximación:** `internal/platform/postgres/analytics_receivables_repo.go:113,317` define "vencido" como `sale_date < NOW() - INTERVAL '30 days'` — 30 días hardcodeados sobre la fecha de venta, igual para todos los clientes. El módulo receivables (overview/aging/top-debtors/collection, `internal/analytics/http/routes.go:46-54`) ya existe y es la base natural, pero descansa sobre esa aproximación.

---

## 2. Decisiones de diseño

### D1 (H1) — Mono-rol por constraint de DB (recomendado), no rol primario

| Criterio | A. Rol único (constraint) ✅ | B. Rol primario explícito |
| :-- | :-- | :-- |
| Elimina el bypass | Sí, la clase entera de bugs desaparece | Parcial: el JWT sigue siendo 1 rol, hay que confiar en que el primario esté bien elegido y sincronizado |
| Superficie de cambio | 1 migración + 2 guards en identity + 1 panel FE | Migración + columna + UI de "elegir primario" + reglas en login/refresh + docs de por qué |
| Ajuste al modelo vigente | Los roles ya son perfiles puros (VNDR01, CAJA01, DEPO01, ENCR01…); la FASE B del plan vendor los diseñó mono-rol | Introduce un concepto nuevo que nadie pidió operativamente |
| Necesidad futura de capacidades combinadas | Se resuelve con un rol nuevo o grants puntuales (el grano fino ya es el permiso, no el rol) | Se resuelve con multi-role, que es lo que genera el bug |

**Decisión:** constraint de rol único. La única excepción histórica (CLIENT + algo) no aplica: las cuentas CLIENT son de mostrador y los operadores no necesitan ese rol.

**Bloqueante de producción (pedido explícito del owner):** auditar `user_roles` en la DB de **prod** ANTES de aplicar la migración — si allá hay dual-roles reales, el índice único revienta y hay que resolverlos caso a caso primero:

```sql
-- AUDITORÍA PRE-PROD (solo lectura)
SELECT ur.user_id, u.email, u.status,
       string_agg(ur.role_id, ', ' ORDER BY ur.assigned_at) AS roles,
       min(ur.assigned_at) AS first_assigned
FROM users.user_roles ur
JOIN users.users u ON u.id = ur.user_id
WHERE u.deleted_at IS NULL
GROUP BY ur.user_id, u.email, u.status
HAVING count(*) > 1
ORDER BY first_assigned;
```

El resultado define el trabajo manual: para cada dual-role, decidir qué rol conserva el usuario (el más reciente suele ser el deseado — el viejo F2VLso era ADMIN por ser el más antiguo, justo al revés de lo que el usuario esperaba).

### D2 (H2) — Normalizar el descuento en la línea de venta (columnas, no JSONB)

Proyección consultable de lo que hoy solo vive en `metadata`. El JSONB se conserva (detalle rico de la trazabilidad); las columnas son para estadística.

`transactions.sales_order_details` (nuevas columnas):

```sql
unit_price_original  numeric(12,2) NULL,  -- precio ANTES del descuento (hoy se pierde: el precio plegado no permite reconstruir el % ni comparar contra catálogo vigente a la fecha)
discount_amount      numeric(12,2) NOT NULL DEFAULT 0,  -- monto TOTAL DE LÍNEA (semántica canónica C-3; hoy es columna muerta)
discount_percent     numeric(5,2)  NULL,  -- solo si vino por % (mirar budget_order_details)
discount_reason      varchar(255)  NULL,
discount_applied_by  varchar(27)   NULL REFERENCES users.users(id)  -- QUIÉN
```

`transactions.sales_orders`:

```sql
discount_total numeric(12,2) NOT NULL DEFAULT 0  -- suma de líneas, la mantiene la función al cerrar la venta (para stats rápidas sin JOIN)
```

- Escriben estas columnas las 4 rutas SQL que hoy pliegan descuento: `process_sale_with_reserve`, `add_products_to_sale`, `update_sale_item`, `convert_budget_to_sale`.
- La lectura actual (`saleDetailBaseQuery` deriva `(precio catálogo − unit_price) × qty`) sigue funcionando igual; `unit_price_original` la vuelve exacta.
- **Backfill idempotente** desde `metadata->'discounts'` (dev: 11 ventas) → columnas. El `p_user_id` del JSONB es la fuente de `discount_applied_by`.
- Vista de estadística:

```sql
CREATE VIEW analytics.v_sales_discounts AS
SELECT so.id AS sale_id, so.sale_date, so.branch_id, so.id_user AS seller_user_id,
       sod.product_id, sod.variant_id,
       sod.discount_amount, sod.discount_percent, sod.discount_reason,
       sod.discount_applied_by, u.first_name || ' ' || u.last_name AS applied_by_name,
       sod.unit_price_original, sod.unit_price, sod.quantity
FROM transactions.sales_orders so
JOIN transactions.sales_order_details sod ON sod.sales_order_id = so.id
JOIN users.users u ON u.id = sod.discount_applied_by
WHERE sod.discount_amount > 0;
```

Con eso, "ventas con descuento por vendedor / aplicador / sucursal / período" es un SELECT, no un parser JSONB.

### D3 (H3) — Cuenta corriente (open-item) sobre las ventas, sin renombrar statuses

**No cambiamos el CHECK de status** (`PAID|PENDING|PARTIAL_PAYMENT|CANCELLED` está cosido a app, FE, funciones SQL, analytics y checks). En cambio: `PENDING`/`PARTIAL_PAYMENT` **significan** saldo abierto, y "crédito" pasa a ser el par *(venta con saldo abierto, vencimiento)*. En el FE se cambia solo la presentación ("Pendiente" → "A crédito", con fecha de vencimiento cuando corresponda).

**1) Vencimiento real.** Columnas nuevas:

```sql
ALTER TABLE transactions.sales_orders ADD COLUMN due_date date NULL;
ALTER TABLE users.parties
  ADD COLUMN credit_terms_days int NULL,          -- ej. 30; NULL = usa el default del sistema
  ADD COLUMN credit_limit numeric(15,2) NULL;     -- NULL = sin límite
-- settings: default_credit_terms_days (global, ej. 0 = contado)
```

Regla al crear la venta: `due_date = (sale_date + COALESCE(credit_terms_days del cliente, default_settings))::date`. Se asigna siempre (aunque se cobre al momento); el aging solo cuenta saldos abiertos, así que las ventas contado no contaminan reportes.

**2) Saldo por venta materializado.** Hoy cada query deriva `pagado = SUM(sale_payments)`; con allocations eso se multiplica. Columna mantenida:

```sql
ALTER TABLE transactions.sales_orders ADD COLUMN paid_amount numeric(15,2) NOT NULL DEFAULT 0;
-- trigger AFTER INSERT/UPDATE/DELETE sobre sale_payments Y account_payment_allocations:
--   paid_amount = SUM(sale_payments aplicables) + SUM(allocations)
--   status := PAID cuando cubre total_amount; PARTIAL_PAYMENT si > 0; PENDING si 0
-- CHECK (paid_amount <= total_amount)
```

Touchpoints que hoy recalculan pagos y deben pasar a leer `paid_amount` (inventario de los encontrados): `process_sale_partial_payment` (`v_existing_payments`), `get_sale_payment_totals_by_date_range`, `preview_sale_reversion` / revert, `analytics_receivables_repo.go`, `internal/sale/pos_service.go`.

**3) Deuda total por cliente (respuesta directa a la pregunta del owner):**

```sql
CREATE VIEW transactions.v_client_credit_balance AS
WITH open_sales AS (
  SELECT so.client_id, so.currency_id, so.id, so.due_date, (so.total_amount - so.paid_amount) AS balance
  FROM transactions.sales_orders so
  WHERE so.status IN ('PENDING','PARTIAL_PAYMENT') AND so.total_amount > so.paid_amount
)
SELECT client_id, currency_id,
       count(*) AS open_sales,
       sum(balance) AS total_debt,
       -- "¿qué monto corresponde a qué rango de vencimiento?" → buckets de aging
       sum(balance) FILTER (WHERE due_date >= CURRENT_DATE)                        AS bucket_current,
       sum(balance) FILTER (WHERE due_date <  CURRENT_DATE AND due_date >= CURRENT_DATE - 30) AS bucket_1_30,
       sum(balance) FILTER (WHERE due_date <  CURRENT_DATE - 30 AND due_date >= CURRENT_DATE - 60) AS bucket_31_60,
       sum(balance) FILTER (WHERE due_date <  CURRENT_DATE - 60 AND due_date >= CURRENT_DATE - 90) AS bucket_61_90,
       sum(balance) FILTER (WHERE due_date <  CURRENT_DATE - 90)                   AS bucket_over_90,
       min(due_date) AS oldest_due_date
FROM open_sales
GROUP BY client_id, currency_id;
```

Agrupado por `currency_id` — la deuda en Gs y en USD no se suma (las ventas ya son multi-moneda). Para aging a fecha arbitraria (reporte histórico "¿cómo estaba la cartera al 31/08?"), la misma lógica con `AS OF` parametriza en el endpoint, no en la vista.

**4) Pago a cuenta — el flujo de los 400.000 en una operación:**

```sql
CREATE TABLE transactions.account_payments (
  payment_id        integer PRIMARY KEY DEFAULT nextval('transactions.account_payments_payment_id_seq'),
  client_id         varchar(30) NOT NULL REFERENCES users.parties(id),
  amount            numeric(15,2) NOT NULL CHECK (amount > 0),   -- lo que el cliente entregó
  unapplied_amount  numeric(15,2) NOT NULL CHECK (unapplied_amount >= 0 AND unapplied_amount <= amount), -- saldo a favor
  payment_reference varchar(255), payment_notes text,
  payment_date      timestamp NOT NULL DEFAULT now(),
  processed_by      varchar(27) NOT NULL REFERENCES users.users(id),
  status            varchar(20) NOT NULL DEFAULT 'OPEN'
                    CHECK (status IN ('OPEN','APPLIED','PARTIAL','CANCELLED','REFUNDED')),
  payment_method_id integer REFERENCES payments.payment_methods(id),
  currency_id       integer REFERENCES payments.currencies(id),
  exchange_rate     numeric(15,6) DEFAULT 1.0,
  original_amount   numeric(15,2),
  cash_register_id  integer REFERENCES transactions.cash_registers(id),
  branch_id         integer NOT NULL REFERENCES branches(id),
  created_at/updated_at …
);

CREATE TABLE transactions.account_payment_allocations (
  id                 integer PRIMARY KEY DEFAULT nextval(...),
  account_payment_id integer NOT NULL REFERENCES transactions.account_payments(payment_id),
  sales_order_id     varchar(27) NOT NULL REFERENCES transactions.sales_orders(id),
  allocated_amount   numeric(15,2) NOT NULL CHECK (allocated_amount > 0),
  allocated_by       varchar(27) NOT NULL REFERENCES users.users(id),
  allocated_at       timestamptz NOT NULL DEFAULT now(),
  UNIQUE (account_payment_id, sales_order_id)
);
-- trigger: allocation no excede el saldo de la venta ni el unapplied del pago;
-- reinserta en paid_amount/status via el trigger de (2).
```

Flujo: se registra UN `account_payment` de 400.000 → `allocate_client_payment()` asigna FIFO por vencimiento (`ORDER BY due_date NULLS LAST, sale_date, id`, `FOR UPDATE` + advisory lock por `client_id` — dos cajas pueden cobrar al mismo cliente) → venta 1 queda PAID (250.000), venta 2 PARTIAL_PAYMENT (150.000). El sobrante queda como `unapplied_amount` (saldo a favor, aplicable a la próxima venta). Reversión: cancelar allocation/pago con los mismos guards de caja cerrada que ya tiene `sale_payments` (`prevent_payment_in_closed_cash_register`).

**Alternativa descartada** (documentada para la posteridad): endpoint "cobrar cliente" que internamente lance N `process_sale_partial_payment` en una tx. Menos schema, pero el pago real del cliente (400.000, un método, una caja) deja de existir como entidad — caja y auditoría pierden el evento. Con `account_payments` el evento es la fila y las allocations son contabilidad interna; caja/cash_movements consumen `account_payments` exactamente como hoy consumen `sale_payments`.

**5) Receivables sobre `due_date` real.** `analytics_receivables_repo.go` reemplaza `sale_date < NOW() - INTERVAL '30 days'` por `due_date < CURRENT_DATE` (fallback `COALESCE(due_date, sale_date + 30)` para ventas históricas pre-migración, así el módulo no cambia de significado con datos viejos).

**6) Límite de crédito (gate en la venta).** Si `parties.credit_limit IS NOT NULL` y `deuda abierta + venta nueva > credit_limit` → warning en la respuesta (bloqueo duro configurable por settings en una segunda iteración; arrancar suave para no frenar la operación el día del deploy).

**Fuera de alcance (notas):** fecha de vencimiento en la factura electrónica Sifen (campo fiscal propio, se evalúa aparte); auto-aplicar saldo a favor en el checkout del POS (fase opcional C7, el wizard ya lista pendientes del cliente desde el plan de pedidos).

---

## 3. Fases y tareas

### FASE A — Monorol explícito (backend + FE mínimo) — **prioridad 1, bloqueante pre-producción**

- [x] **A0. Auditoría dual-roles en prod** (SQL de §D1, solo lectura). Resolver caso a caso los que aparezcan. Gate: 0 dual-roles en prod antes de A1.
- [x] **A1. Migración** `make migrate-create`: índice único parcial `CREATE UNIQUE INDEX uk_user_roles_single_role ON users.user_roles(user_id);` (falla ruidosamente si quedan dual-roles — fail-closed intencional). Down: `DROP INDEX`.
- [x] **A2. Identity**: `Login` y `Refresh` — si `len(roles) > 1` → error explícito (`ErrUserMultipleRoles`, 403 con mensaje accionable) en vez de `roles[0]` silencioso. Defensa en profundidad: la DB ya lo impide, pero identity no debe callarse si alguien relaja el constraint. Elimina el TODO D11-5 de `service.go:70,148`. Tests: login dual-role rechazado; refresh dual-role rechazado; migración up con dual-role data revienta.
- [x] **A3. FE** `ManageRolesPanel`: al asignar un rol cuando el usuario ya tiene uno → error claro del backend renderizado (i18n `es/en`). `UserFormModal` ya es mono-rol, sin cambios.
- [ ] **A4. Docs:** nota en `AGENTS.md` backend (regla: "mono-rol por constraint; los permisos heredan del rol único") + actualizar `FRONTEND_RBAC_GUIDE.md` si menciona multi-role.

### FASE B — Descuentos auditables (backend + BI + FE) — independiente de A y C

- [x] **B1. Migración** columnas de §D2 en `sales_order_details` + `sales_orders.discount_total` + backfill idempotente desde `metadata->'discounts'`.
- [x] **B2. Funciones SQL** escriben las columnas en las 4 rutas (`process_sale_with_reserve`, `add_products_to_sale`, `update_sale_item`, `convert_budget_to_sale`) y el `discount_total` al cierre. `discount_applied_by = p_user_id` SIEMPRE (el que corre la función; ya llega como parámetro). Tests SQL de paridad presupuesto→venta se mantienen en verde (semántica C-3 intacta: los totales no cambian).
- [x] **B3. Vista** `analytics.v_sales_discounts` + endpoints de reporte (gate `reports:read`): por vendedor, por aplicador, por sucursal/período. Paridad con `budget_order_details` restaurada.
- [x] **B4. FE** detalle de venta (admin/reports:read): badge/línea "Descuento aplicado por X — motivo" por ítem; reporte BI simple "descuentos por vendedor".
- [ ] **B5. Gate:** `go test ./...` + `golangci-lint` + `make build` + `make migrate-up` (lección de la divergencia RBAC: migrate-up SIEMPRE en los gates) + E2E con descuento verificando columnas y JSONB consistentes.

### FASE C — Cuenta corriente de cliente — la más grande; usa `paid_amount`/`due_date` de C1 en receivables

- [x] **C1. Migración** §D3.1–2: `due_date`, `paid_amount` (+ triggers sobre `sale_payments` y allocations), `parties.credit_terms_days/credit_limit`, setting default terms, `account_payments` + `account_payment_allocations` (CHECKs de §D3.4). Backfill: `due_date = sale_date + 30` para ventas abiertas existentes (mismo criterio que el aging actual, así la cartera no "cambia" el día del deploy) y `paid_amount` desde `SUM(sale_payments)` por venta.
- [ ] **C2. `allocate_client_payment(payment_id, override?)`**: FIFO por vencimiento con override manual (lista de `{sale_id, monto}`), lock por cliente, actualiza `unapplied_amount`/status del pago y `paid_amount`/status de ventas vía triggers. Reversión de allocation y cancelación del pago con guards de caja cerrada. Tests de concurrencia (dos cajas).
- [x] **C3. Touchpoints** a `paid_amount`: `process_sale_partial_payment`, `get_sale_payment_totals_by_date_range`, `preview_sale_reversion`/revert, `analytics_receivables_repo`, `internal/sale/pos_service`. Vista `transactions.v_client_credit_balance`. Cash: `account_payments` entra a cash_movements/reportes de caja igual que `sale_payments` (sin doble conteo: el movimiento vive en el pago a cuenta, no en las allocations).
- [x] **C4. API** (contexto `internal/sale` o `internal/party`): `GET /clients/{id}/credit` (deuda por moneda, buckets, ventas abiertas, saldo a favor), `GET /clients/credit/aging?branch_id&as_of` (cartera), `POST /clients/{id}/account-payments` (+allocations auto FIFO o manual). Gates: `cash:write` para cobrar, `receivables:read`/`reports:read` para reportes.
- [ ] **C5. Receivables con `due_date` real** (§D3.5) — el aging del BI pasa de aproximación a contractual.
- [ ] **C6. FE:** ficha de cliente → bloque "Crédito" (terms, límite, deuda total con chips de aging, saldo a favor); vista "Cobros de cuenta" en caja (reusa CollectionStep: métodos, multi-moneda); presentación de ventas `PENDING/PARTIAL_PAYMENT` → "A crédito — vence DD/MM". i18n es/en.
- [ ] **C7 (opcional).** Auto-aplicar saldo a favor en el checkout del POS; gate de `credit_limit` duro por settings.
- [ ] **C8. Gate + E2E del escenario del owner:** 2 ventas de 250.000 → 1 pago a cuenta de 400.000 → venta 1 PAID, venta 2 PARTIAL 150.000 pendiente, deuda total 150.000, aging correcto por `due_date`; reversión del pago restaura todo; dos cajas concurrentes no sobreasignan.

## 4. Riesgos y mitigaciones

| Riesgo | Mitigación |
| :-- | :-- |
| Prod tiene dual-roles reales → A1 revienta | A0 primero (el owner ya lo pidió); el fallo ruidoso es deliberado |
| Allocations + triggers de caja interactúan con `process_sale_partial_payment` | `paid_amount` como única fuente de verdad; C3 migra los lectores en el mismo diff |
| Doble conteo en reportes de caja (pago a cuenta + allocations) | Movimiento de caja vive SOLO en `account_payments`; test de suma de caja en E2E |
| Multi-moneda: sumar deudas Gs+USD | Vista agrupa por `(client_id, currency_id)`; FE muestra por moneda |
| Concurrencia (dos cajas, mismo cliente) | Advisory lock por `client_id` + `FOR UPDATE` en allocations (E2E C8) |
| Backfill H2 desde JSONB incompleto/edge cases | Idempotente + verificación: post-backfill, `COUNT(filas con descuento en JSONB) == COUNT(filas con discount_amount > 0)` |
| Renombre "pending"→"crédito" confunde a operadores | Solo presentación FE con fecha de vencimiento; status interno intacto |

## 5. Orden de ejecución recomendado

1. **FASE A** — pequeña, bloqueante de seguridad pre-producción, no depende de nada.
2. **FASE B** — independiente, valor inmediato para BI (los descuentos ya se aplican hoy sin traza consultable).
3. **FASE C** — la más grande; C1 es su migración base y C5 conecta con el módulo receivables existente.

Gates generales por repo (sin cambios): backend `go test ./...` + `golangci-lint` + `make build` + `make migrate-up`; frontend `npx vitest --run` + `npx tsc --noEmit` + `pnpm build` + `pnpm lint:design`. Commits: repos separados, conventional commits; registro de implementación en este archivo (sección al estilo de `PLAN_PEDIDOS_MOSTRADOR_VENDEDOR_CAJA.md`).

---

## 6. Registro de implementación (2026-09-09)

### Commits (repos separados, ambos en `dev`)

| Fase | Backend (`business_management`) | Frontend (`erp-webapp`) |
| :-- | :-- | :-- |
| A — Monorol | `8427719` feat(identity): mono-role | `006c8c1` feat(users): mono-role rejection UI |
| B — Descuentos | `a987a36` feat(analytics): discount columns + report | `387e413` feat(bi): discounts report page + badge |
| C — Cuenta corriente | `a93dc47` feat(party): credit account + `7d746b4` feat(sale): due_date API | `930b324` feat(clients): credit card + "a crédito" |

Migraciones (5): `20260909213019_mono_role_constraint`, `20260909214752_sales_discount_columns`,
`20260909221733_client_credit_account`, `20260909222634_paid_amount_source_of_truth`,
`20260909222841_register_account_payment_fn`.

### Verificación (dev)

- **A0:** 0 dual-roles en `erp_db` (incluye soft-deleted). **El gate de PROD sigue pendiente**: correr el SQL de §D1 (con `u.username`, no `u.email` — la tabla no tiene email) ANTES de desplegar la migración; el índice único falla ruidoso si hay dual-roles (fail-closed).
- **A2:** login/refresh con 2 roles → 403 `ErrUserMultipleRoles`; `AssignRole` → 409 `USER_ALREADY_HAS_ROLE`. Índice probado con INSERT de segundo rol → `unique_violation`.
- **B2 (E2E SQL):** venta con 10% en qty 2 → `unit_price_original` 66.300, `unit_price` 59.670, `discount_amount` 13.260 (línea), `discount_total` 13.260, JSONB rico intacto, total C-3 sin cambios. Backfill: `discount_total == SUM(líneas)` en las 11 ventas con JSONB (10 matcheadas; la 11ª es un 100% sin precio original — irreconstruible, queda 0, no se inventa).
- **C2 (E2E SQL):** escenario del owner exacto (2×250.000, pago a cuenta 400.000) → venta 1 PAID, venta 2 PARTIAL con saldo 100.000, buckets de aging correctos; cancelación del pago restaura ambas a PENDING/0. **Concurrencia:** dos cajas allocando al mismo cliente → 200.000 + 50.000 sin sobreasignación (advisory lock por cliente serializa; el guard de saldo validó contra el estado post-commit de la otra caja).

### Hallazgos y desviaciones documentadas

1. **Bug preexistente corregido (fase B):** `SaleRepo.updateSaleMetadata` SOBRESCRIBÍA todo el JSONB `metadata` después de cada venta, destruyendo la traza rica que `process_sale_with_reserve` ya había escrito (por eso 9 de 11 ventas tenían entradas sin `original_price/final_price`). Eliminado — la función SQL es el único escritor de metadata de descuentos.
2. **C1:** sin `CHECK (paid_amount <= total_amount)` — dev ya tiene una venta +0.50 por redondeo histórico; el guard real vive en las funciones de pago (`AMOUNT_EXCEEDS_BALANCE`).
3. **C3:** `get_sale_payment_totals_by_date_range` y `preview_sale_reversion` siguen leyendo `sale_payments` (reportes por FECHA / montos a devolver; `paid_amount` es estado actual, no serie temporal).
4. **C4 gates reales:** ficha de crédito `clients:read` (subrouter de clientes, va en la ficha que ya ve quien ve el cliente), cartera `/clients/credit/aging` → `receivables:read`, cobro `POST /clients/{id}/account-payments` → `cash:write` (wrapper `RequireModulePermissionOnPath` sobre el gate del subrouter).
5. **C6:** el FE registra cobros a cuenta con **FIFO automático** (default del backend); el override manual existe en la API (`allocations`) pero aún sin UI. La vista dedicada "Cobros de cuenta" en caja quedó absorbida por el bloque de la ficha; si se quiere un flujo de caja dedicado, reusar `CollectionStep` (pendiente opcional junto a C7).
6. **Bug del plan corregido en el camino (C2):** el guard de allocation restaba dos veces las allocations existentes (ya están dentro de `paid_amount`); la fórmula final valida `allocated ≤ total − paid_amount` directamente. Detectado por el E2E de concurrencia.
7. **Checksums de migración:** dos migraciones se corrigieron durante el desarrollo (pre-commit) y el hash quedó grabado del primer up (`ON CONFLICT DO NOTHING`); reconciliado con UPDATE sobre `migration_history` — `migrate-verify-gaps` en verde.

### Pendientes

- [ ] **A0 PROD (bloqueante de deploy):** correr la auditoría SQL en prod y resolver dual-roles caso a caso ANTES de desplegar `20260909213019`.
- [ ] **C7 (opcional):** auto-aplicar saldo a favor en el checkout del POS; gate duro de `credit_limit` por settings.
- [ ] Re-login de usuarios admin tras deploy (los gates de las nuevas rutas viajan en el JWT).
