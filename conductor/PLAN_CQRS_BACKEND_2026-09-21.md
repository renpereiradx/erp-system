# PLAN — CQRS en business_management (evaluación de factibilidad + adopción CQRS-lite)

**Fecha**: 2026-09-21
**Estado**: 📋 F0 — veredicto emitido; F1 pendiente de ejecución, F2/F3 condicionales
**Alcance**: solo backend `business_management/`. **Sin impacto en `erp-webapp`**: el contrato
REST no cambia en ninguna fase. Hosteado en el conductor raíz por decisión del owner; los
registros por fase van a `business_management/conductor/` como es costumbre.
**Origen**: pedido del owner: "revisa si es factible implementar un patrón de arquitectura CQRS".

---

## 1. Veredicto de factibilidad

**CQRS sí es factible, pero solo en su variante liviana ("CQRS-lite"), y de hecho el sistema ya
la implementa a medias sin llamarla así.** CQRS completo (base de lectura separada, eventos,
consistencia eventual) **se rechaza** con criterios explícitos (§4.1).

| Nivel | Qué implica | Veredicto |
|:------|:------------|:----------|
| **1. CQRS-lite (código)** | Comandos y queries en interfaces/servicios separados, misma DB, mismos endpoints | ✅ **Viable y recomendado** — F1/F2 de este plan |
| **2. Read models (misma DB)** | Vistas SQL / matviews para analítica pesada, mantenidas por triggers + NOTIFY | 🟡 **Viable condicional** — solo con evidencia de coste medido (F3) |
| **3. CQRS completo** | Store de lectura separado, event sourcing / bus de eventos, consistencia eventual | ❌ **Rechazado** — ver §4.1 y R6 |

El hallazgo central de la evaluación es que **la separación Reader/Writer ya existe a nivel de
puertos en los agregados núcleo** y que **el lado lectura pesado ya está físicamente aislado**
(`internal/analytics` con 10 repos read-only). Lo que falta es acotado: 12 interfaces mezcladas
y 2-3 services que concentran comandos y queries. Completar eso es deuda mecánica, no un
refactor arquitectónico.

---

## 2. Qué es CQRS (glosario en lenguaje llano)

- **Comando**: una operación que cambia el estado (crear una venta, ajustar stock, abrir caja).
- **Query**: una operación que solo lee (listar ventas del día, dashboard, informe de giro de
  inventario). No cambia nada.
- **CQRS** (*Command Query Responsibility Segregation*): separar los caminos de escritura y de
  lectura para que evolucionen por separado, porque tienen necesidades distintas (la escritura
  pide consistencia y transacciones; la lectura pide velocidad y forma conveniente de los datos).

**Analogía**: un restaurante. Los mozos que toman pedidos y mandan a la cocina son los
*comandos*: cada uno cambia el estado de la cocina, y las reglas son estrictas (no se puede
cobrar dos veces la misma mesa). El contador que a fin de mes arma el resumen de qué se vendió
más es la *query*: lee lo mismo que los mozos fueron escribiendo, pero lo hace por su propio
camino (planillas, totales) y no necesita bloquear la cocina para calcular.

Los tres niveles de la tabla anterior, en la analogía: **(1)** separar en el organigrama a mozos
del contador; **(2)** darle al contador una planilla pre-armada que se va llenando sola durante
el mes; **(3)** contratar una oficina aparte que recibe copias de cada ticket y arma sus propios
informes — siempre un poco atrasados respecto de la cocina.

---

## 3. Estado actual (evidencia verificada 2026-09-21)

### 3.1 Lo que YA está separado — el sistema es CQRS-lite a medias

| Pieza | Evidencia |
|:------|:----------|
| Puertos Reader/Writer en agregados núcleo | `catalog/port.go` (Product :56/:73, Variant :86/:108, Attribute :124-137/:137, Tag :153/:166, Template :181/:195 + unión `ProductRepository` :80-83), `tax` (Rate, Classification), `brand`, `paymentmethod`, `pricing` (Price), `party` (Party, Client + `CreditReader` credit.go:121), `inventory` (Stock :20/:35, Transaction :71/:87), `purchase` (Supplier :40/:47, Order :56/:71, Requisition :85/:94), `devices` (PermissionReader :37) |
| Query side físico para BI | `internal/analytics`: 9 servicios read-only (`service.go`), 10 repos read-only en `internal/platform/postgres/` (**12.093 líneas** de SQL), 135 rutas GET (`http/routes.go`), puertos `*Reader` puros (sales 28 métodos, inventory, financial, payables, receivables, profitability, dashboard, forecast) |
| Separación física read/write ya hecha a mano | `purchase/http_order_read.go` (166 líneas) vs `http_order_write.go` (110); `order_read_repo.go` (1554 líneas, solo lecturas enriquecidas "FASE 9.4") |
| Proyección desacoplada (precedente de read model) | `products.mv_product_catalog` + trigger `pg_notify('catalog_change')` (`20260616202841`) + worker `internal/catalog/refresh/worker.go` (debounce 2 s, invalida cache Redis al refrescar) |
| Vistas SQL de reporte | `v_sales_ledger_sifen`, `v_purchases_ledger_sifen` (libros IVA), `v_monthly_vat_liquidation`, `analytics.v_forecast_method_accuracy`, vistas de auditoría |
| Unit of Work | `internal/platform/txm`: `Manager.Run(ctx, fn)` inyecta la tx en el contexto; los adapters eligen `tx` vs `db` con `txm.TxFromContext` — agnóstico de repos, sirve al lado comando tal cual está |
| Caches de lectura | Redis: `product_cache.go` (read-through + singleflight + invalidación), `bi_cache.go` (read-through TTL-only para agregados BI, decisión documentada), `token_blacklist.go` |

### 3.2 La deuda — dónde se mezcla (el gap real)

Interfaces que mezclan comandos y queries en el mismo puerto:

| Puerto | Comandos mezclados con | Ubicación |
|:-------|:-----------------------|:----------|
| `ExchangeRateRepository` | `Create/Update/Delete` + 6 Gets | `internal/sale/port.go:161-171` |
| `PaymentRepository` | `ProcessSalePaymentUnified` + 4 Gets/estadísticas | `internal/sale/port.go:195-215` |
| `CashRegisterRepository` | `Open/Close/RegisterMovement/VoidMovement` + 5 Gets/reportes | `internal/sale/port.go:220-233` |
| `CashAuditRepository` | escritura + lectura | `internal/sale/port.go:236-242` |
| `BudgetRepository` | `Create/UpdateStatus/ConvertToSale/MarkExpired` + 3 Gets | `internal/sale/port.go:247-256` |
| `CounterOrderRepository` | `Create/ReplaceItems/Claim/Release/Mark*` + 3 Gets | `internal/sale/port.go:264-293` |
| `CancellationRequestRepository` | escritura + lectura | `internal/sale/port.go:306-315` |
| `CountRepository` | `Insert/Invalidate` + Gets/List | `internal/inventory/port.go:43-52` |
| `AdjustmentRepository` | `Insert` + List/History/Verify | `internal/inventory/port.go:57-66` |
| `ReserveRepository` | `ManageReserve` + 9 Gets/reportes | `internal/inventory/port.go:95-118` |
| `ScheduleRepository` | `GenerateDaily/UpdateAvailability/UpsertConfig` + 11 Gets | `internal/inventory/port.go:123-154` |
| `PaymentRepository` (purchase) | `ProcessPayment*` + estadísticas | `internal/purchase/port.go:125-137` |

Además, a nivel de services: `ProductService` mezcla `CreateProduct`/`UpdateProduct` con
`ListProducts`/`GetProductInfo` (`internal/catalog/service.go`, 1170 líneas);
`SaleServiceAPI` declara comandos y queries juntos (`internal/sale/port.go:327-345`); y el
wiring pasa el mismo repo tres veces (`wiring/sale.go:123-130`: `NewSaleService(repo, repo,
repo, …)`), escondiendo qué lado usa cada colaborador.

### 3.3 Infraestructura: qué hay y qué no hay

- **No hay** event bus, pub/sub genérico, outbox genérico ni broker en `go.mod` (el único
  outbox es el fiscal de SIFEN, específico del dominio; el único async in-proc es el audit
  trail por canal y los workers SIFEN/catalog-refresh).
- **Un solo PostgreSQL** transaccional (esquemas por dominio, 566 migraciones).
- Performance BI ya tratada por `PLAN_AUDIT_BI` F1 (índices T2, dedup SQL T11, SARGable T13,
  cache Redis T10, ETag + rate limit T14): **no hay dolor de escala sin tratar que justifique
  infraestructura nueva**.

---

## 4. Evaluación por nivel

### 4.1 CQRS completo — por qué NO

1. **La consistencia fuerte es innegociable en este ERP.** Stock disponible (no sobrevender),
   caja (arqueos), crédito FIFO con lock por cliente, conversión atómica de pedidos de
   mostrador vía `txm`, mono-rol de descuentos: todas esas invariantes las garantiza hoy una
   sola DB transaccional. Con un store de lectura separado y proyecciones asíncronas, una
   pantalla podría mostrar stock o saldo viejos y una decisión transaccional tomarse con datos
   atrasados — exactamente la clase de bug silencioso que este repo viene eliminando.
2. **Coste de infra sin dolor que lo pida.** Requeriría broker de mensajes, proyecciones,
   replays, monitoreo de drift entre escritura y lectura. Operación actual: un dev + agentes;
   un solo Postgres; search avanzado en ~1.4 ms (EXPLAIN verificado en el plan de variantes
   planas) y BI ya optimizado y cacheado.
3. **Perdería un activo de tests.** Los tests de equivalencia byte-a-byte contra legacy
   (analytics, catalog, identity, pricing, inventory) y los de integración con Postgres real
   asumen una sola fuente de verdad; bajo dual-store eventual-consistent quedan obsoletos.
4. **Modos de fallo no deterministas** (proyecciones atrasadas, mensajes duplicados,
   reordenamiento) son difíciles de razonar y de gatear con reglas binarias, que es como este
   workspace opera con agentes.

**Decisión**: rechazado. Se registra como decisión negativa (ADR en F4) para que no se
reabra por inercia.

### 4.2 CQRS-lite (código) — por qué SÍ

- **Es la continuación natural de la migración hexagonal**, no una arquitectura nueva: los
  puertos ya lo hacen en los agregados núcleo; completarlo en las 12 interfaces rezagadas es
  deuda puntual y mecánica.
- **Beneficios concretos y presentes**:
  1. *Mocks más chicos*: 73 archivos de test con mocks hand-rolled por puerto; hoy testear un
     comando obliga a implementar todas las queries del mismo puerto (p.ej. `ReserveRepository`:
     1 comando + 9 lecturas en una interfaz).
  2. *Wiring honesto*: `NewSaleService(repo, repo, repo, …)` deja de esconder quién lee y quién
     escribe; el typo-del-futuro (inyectar un writer donde va un reader) pasa a ser error de
     compilación.
  3. *Los adapters de lectura ya divergieron* (`order_read_repo.go` vs write repo): la frontera
     existe en los hechos, falta declararla en los puertos que faltan.
  4. *Deja la puerta abierta* a read models medidos (nivel 2) sin big-bang.
- **Coste bajo**: sin cambio de contrato REST, sin migraciones SQL, sin dependencias nuevas; el
  adapter SQL puede seguir siendo un solo struct que implementa ambas mitades.
- **Riesgo principal**: churn cosmético. Mitigado: F1 es la única fase incondicional y es
  mecánica; F2/F3 tienen umbrales de entrada (§5 R3, §6).

### 4.3 Read models (misma DB) — condicional

- Solo para **analítica tolerante a staleness** (BI), nunca para lecturas transaccionales (R4).
- Escalera de coste: **vista SQL primero** (siempre fresca, cero infra nueva, planner-friendly
  desde PG12), **matview después** solo si `EXPLAIN`/`pg_stat_statements` muestran coste
  agregado repetido que la vista no evita. El mecanismo de refresh asíncrono ya existe y está
  probado (trigger NOTIFY + worker con debounce del catálogo).

---

## 5. Convención objetivo (R1–R6)

Estas reglas son el entregable duradero de F1/F4; van a parar a `business_management/AGENTS.md`.

- **R1** — Un puerto nunca mezcla comandos y queries: `XReader` / `XWriter` separados por
  agregado. El adapter SQL puede ser un solo struct que implementa ambos.
- **R2** — El tipo unión `XRepository = interface { XReader; XWriter }` se declara solo donde
  el wiring necesite pasar "el repo completo" (patrón existente `catalog/port.go:80-83`).
- **R3** — En services: los comandos quedan en el service actual; las queries enriquecidas de
  negocio se extraen a un `XQueryService` **solo si** el service supera el umbral (~600 líneas
  o >8 métodos de lectura enriquecida). No se parte un service sano por deporte.
- **R4** — Binaria: toda lectura que alimenta una decisión transaccional (stock para vender,
  saldo de crédito, efectivo en caja) consulta **tablas operacionales**, nunca proyecciones ni
  caches TTL-only. Las proyecciones son territorio exclusivo de analítica/BI.
- **R5** — Proyecciones SQL solo para analítica: preferir vistas; matview con evidencia de
  coste medida, refresh vía el patrón LISTEN/NOTIFY existente, y documentar el staleness máximo
  esperado en el propio registro de la fase.
- **R6** — Nada de command bus, event sourcing ni stores de lectura separados: las llamadas
  siguen siendo métodos directos sobre services. Decisión negativa deliberada (§4.1, ADR en F4).

---

## 6. Fases

### FASE F1 — Completar el split de puertos (única incondicional)

- [ ] **T1.1** `internal/sale/port.go`: dividir en Reader/Writer `ExchangeRate`, `Payment`,
      `CashRegister`, `CashAudit`, `Budget`, `CounterOrder`, `CancellationRequest`. Ojo:
      `ProcessSalePaymentUnified` y `ProcessPayment*` son **comandos** (van al Writer aunque
      devuelvan datos); `Metrics`, `GetReport`, `GetBalanceSummary` al Reader.
- [ ] **T1.2** `internal/inventory/port.go`: `Count`, `Adjustment`, `Reserve`, `Schedule`.
      Ojo: `ManageReserve` es comando; `CheckConsistency`/`VerifyStockIntegration` son queries
      de diagnóstico.
- [ ] **T1.3** `internal/purchase/port.go`: `Payment`.
- [ ] **T1.4** Re-grep de mezclas residuales al ejecutar (`rg -n "interface \{" internal/*/port.go`)
      y split de lo que aparezca.
- [ ] **T1.5** Reglas R1–R6 en `business_management/AGENTS.md`.
- Los adapters SQL y los repos compuestos siguen siendo un solo struct; solo cambian firmas de
  constructores y mocks (R1/R2).
- **Gate de salida**: `go build ./...`, `golangci-lint run`, `go test ./...` (incluye equiv
  tests) — todo verde; diff sin cambios de comportamiento (cero cambios en handlers/JSON).
- **Esfuerzo estimado**: 1 sesión.

### FASE F2 — Query services donde el umbral R3 lo pide (condicional)

- [ ] **T2.1** `purchase`: formalizar `OrderQueryService` (extraer `GetByIDFull`,
      `ListBySupplierIDFull`, `ListByDateRangeFull` de `OrderService`; el repo read-only ya
      existe).
- [ ] **T2.2** `catalog`: extraer `ProductQueryService` (`GetProduct`, `GetProductByBarcode`,
      `ListProducts`, `GetProductInfo`, search) de `ProductService` (1170 líneas).
- **Gate de salida**: build/lint/test + equiv tests + smoke en vivo de endpoints afectados con
  dev-JWT (receta de smoke del plan de variantes planas).
- **Nota**: `identity/service.go` (2169 líneas) **no entra** por CQRS: su tamaño se resuelve
  partiendo por los 6 servicios que ya están identificados ahí; es otro plan.
- **Esfuerzo estimado**: 1 sesión (si se activa).

### FASE F3 — Read models BI medidos (condicional: solo con evidencia)

- [ ] **T3.1** Medir antes de tocar nada: top de queries BI por `pg_stat_statements` +
      `EXPLAIN ANALYZE` (candidatos observables: `by-seller`, `turnover-abc`, dashboard
      summary). Documentar números en `business_management/conductor/`.
- [ ] **T3.2** Solo si hay coste agregado repetido alto: vista SQL primero; matview + NOTIFY
      worker después si la vista no alcanza (generalizar `internal/catalog/refresh/worker.go`).
- **Gate de salida**: migraciones solo con `make migrate-create`; EXPLAIN antes/después;
  tests de integración con Postgres real; staleness máximo documentado (R5).
- **Esfuerzo estimado**: 1 sesión (si se activa).

### FASE F4 — Cierre

- [ ] **T4.1** ADR-0001 en `business_management/conductor/`: decisión negativa de CQRS completo
      (criterios §4.1) + adopción de CQRS-lite con R1–R6.
- [ ] **T4.2** Purga de este plan del conductor raíz (el historial git es el archivo).

---

## 7. Riesgos y mitigaciones

| Riesgo | Mitigación |
|:-------|:-----------|
| Churn cosmético sin beneficio | F2/F3 condicionales con umbrales (R3, T3.1); F1 es mecánica y cierra deuda real de mocks/wiring |
| Romper mocks/tests al partir interfaces | El split preserva los métodos (solo cambian interfaces y constructores); equiv tests como red de seguridad |
| Que alguien "complete" CQRS después | R6 + ADR-0001 con criterios explícitos |
| Staleness en pantallas operativas | R4 binaria; `bi_cache` TTL-only ya está confinado a agregados BI por diseño |
| Mezclar con otros planes activos del BE | F1 no toca comportamiento; ejecutar en ventana sin plan BE activo |

## 8. Impacto / no-impacto

- **Contrato REST**: sin cambios en ninguna fase. **`erp-webapp`: cero cambios.**
- **Migraciones SQL**: ninguna en F1/F2; F3 solo si se activa (y con `make migrate-create`).
- **`go.mod`**: sin dependencias nuevas.
- **Lo que este plan NO hace**: no reescribe `internal/analytics` (los 10 repos read-only ya
  son el query side), no introduce asincronía nueva, no persigue performance (la performance BI
  ya quedó tratada por `PLAN_AUDIT_BI` F1 — este plan es higiene arquitectónica, no un fix).
