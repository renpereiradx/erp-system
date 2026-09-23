# REQ_VENTAS — Venta de mostrador y pedidos (POS, crédito, anulación)

## 1. Ficha

| Campo | Valor |
|:--|:--|
| Módulo | Ventas (POS, pedidos de mostrador, cuenta corriente, anulaciones) |
| Código | VENTAS |
| Versión | v0.1 |
| Estado | Borrador (backfill: el módulo existe y opera; este REQ formaliza lo vigente) |
| Fecha | 2026-09-22 |
| Fuentes | Código y planes ejecutados (counter-orders 2026-09, monorol+descuentos+crédito 2026-09-09, vendor role 2026-09-06/07, BI drill-downs 2026-09-22), AGENTS.md |
| Implementación | ✅ Operativa (BE/FE en producción dev; ver trazabilidad §11) |

## 2. Resumen ejecutivo (para todos)

Ventas es el corazón del ERP: registra cada operación de venta del local — desde el
cliente que paga en efectivo en la caja hasta el pedido tomado en el mostrador que
después pasa a pagar, o la venta a cuenta corriente que el cliente liquida a fin de
mes. Todo queda con su sucursal, su usuario, sus impuestos y su comprobante.

El módulo contempla los tres caminos reales del negocio: **venta directa** (armás el
carrito y cobrás), **pedido de mostrador** (queda pendiente con vencimiento, se
cobra después en caja) y **cuenta corriente** (el cliente se lleva hoy y paga
después, con control de vencimientos y pagos aplicados alcliente más antiguo primero).
Cada venta puede tener descuentos con control por rol, varios medios de pago, y
puede **anularse con aprobación** dejando rastro.

## 3. Cómo funciona — guía no técnica

**Venta directa (caja).** El cajero busca productos (por nombre o escaneando el SKU
de una variante), suma al carrito, aplica un descuento si su rol lo permite, cobra
con uno o varios medios (efectivo, tarjeta, transferencia...) y cierra. El stock
descuenta, el ticket sale por la impresora del navegador y la venta queda ligada a
la sucursal donde se atendió.

**Pedido de mostrador (counter order).** Cuando el cliente pide algo pero va a
pagar después (o va a retirar), el vendedor crea un pedido con código QR. El pedido
vive 72 horas: se puede reclamar en caja (lo convierte en venta cobrada), liberar
(editarlo de nuevo) o dejar vencer automáticamente. Si al cobrar el stock cambió,
la caja lo avisa antes de completar.

**Cuenta corriente.** Una venta a crédito queda con fecha de vencimiento y saldo.
Los pagos del cliente se aplican **de lo más viejo a lo más nuevo** (FIFO) — como
una fila de deudas que se va atendiendo por orden de llegada. La ficha del cliente
muestra saldo, vencidos y pagos aplicados.

**Anulación.** Una venta mal hecha no se borra: se **anula**, pidiendo aprobación
(a quien su rol lo permita). La anulación corre en la sucursal de la venta y queda
auditada. El stock vuelve.

**Roles.** Lo que cada uno puede hacer depende de su rol (9 roles canónicos). El
vendedor, por ejemplo, no ve costos ni puede cancelar ventas ni otorgar descuentos
fuera de su política; sus permisos están podados en el menú y en la API.

## 4. Alcance

**DENTRO:** POS con variantes y SKU plano · descuentos con política por rol ·
múltiples medios de pago · pedidos de mostrador con TTL y QR · cuenta corriente
(vencimientos, pagos FIFO, ficha) · anulaciones con aprobación y auditoría ·
scoping por sucursal · historial de ventas con filtros server-side · ticket de
impresión desde el navegador · drill-downs de BI sobre ventas (pack BI).

**FUERA:** factura electrónica SIFEN (módulo propio; pendiente certificado del
cliente) · precios por volumen/promociones complejas · e-commerce · modo offline
del POS · multi-moneda (el precio opera en guaraníes).

## 5. Actores

| Actor | Interacción |
|:--|:--|
| Cajero | Cobra ventas directas y reclama pedidos en caja |
| Vendedor | Toma pedidos, consulta stock y precios (sin costos); descuento limitado por política |
| Encargado/Administrador | Anula ventas (aprobación), ve reportes, configura descuentos |
| Cliente | Recibe ticket/pedido; en cuenta corriente, paga y consulta su saldo |
| Sistema SIFEN | (futuro) recibe el comprobante electrónico cuando haya certificado |

## 6. Requerimientos funcionales (RF-…)

| ID | Requerimiento | Estado |
|:--|:--|:--|
| RF-VEN-001 | Crear ventas con productos/variantes (búsqueda por SKU de variante resuelve la fila de esa variante), cantidades y precios con impuestos | ✅ Operativa |
| RF-VEN-002 | Cobrar con uno o varios medios de pago por venta | ✅ Operativa |
| RF-VEN-003 | Aplicar descuentos con política por rol (vendedor con tope; administrador sin tope); registro del descuento por línea | ✅ Operativa |
| RF-VEN-004 | Crear pedidos de mostrador pendientes con QR, TTL 72 h y estados EN CAJA / pendiente / vencido | ✅ Operativa |
| RF-VEN-005 | Reclamar el pedido en caja convirtiéndolo en venta cobrada de forma atómica; guard 409 si ya fue reclamado; aviso si el stock cambió | ✅ Operativa |
| RF-VEN-006 | Vender a cuenta corriente con fecha de vencimiento y saldo por venta | ✅ Operativa |
| RF-VEN-007 | Registrar pagos de cuenta corriente aplicados FIFO (primero lo más viejo) con lock por cliente | ✅ Operativa |
| RF-VEN-008 | Anular venta con flujo de aprobación por rol, corre en la sucursal de la venta y queda auditada | ✅ Operativa |
| RF-VEN-009 | Toda venta y su historial quedan acotadas a la sucursal del usuario (scoping) | ✅ Operativa |
| RF-VEN-010 | Historial de ventas con filtros server-side (fecha, usuario, sucursal, estado) y paginación real | ✅ Operativa |
| RF-VEN-011 | Imprimir ticket desde el navegador (render FE, contrato renderTicket) | ✅ Operativa |
| RF-VEN-012 | Emitir comprobante electrónico SIFEN por venta | ⏳ Pendiente (certificado del cliente) |

## 7. Requerimientos no funcionales (RNF-…)

| ID | Requerimiento | Estado |
|:--|:--|:--|
| RNF-VEN-001 | El cobro del pedido y el descuento de stock son atómicos (una transacción; sin ventas con stock negativo invisible) | ✅ |
| RNF-VEN-002 | El cálculo de precios/impuestos/descuentos vive en dominio FE puro y lógica BE — nunca en el JSX | ✅ |
| RNF-VEN-003 | El vendedor no recibe en la API datos de costo de proveedor (strip server-side post-cache) | ✅ |
| RNF-VEN-004 | Pedidos vencidos se barren automáticamente (sweep TTL) sin intervención humana | ✅ |
| RNF-VEN-005 | Toda operación de venta/anulación queda auditada (usuario, sucursal, timestamp) | ✅ |

## 8. Reglas de negocio (RN-…)

| ID | Regla | Motivo |
|:--|:--|:--|
| RN-VEN-001 | El pedido de mostrador vence a las 72 horas y pasa a EXPIRED sin notificación crítica | Evita colas de pedidos fantasma; el TTL es la política acordada con el dueño |
| RN-VEN-002 | Los pagos de cuenta corriente se aplican de la deuda más vieja a la más nueva | Práctica comercial estándar; evita discusiones sobre qué factura se pagó |
| RN-VEN-003 | La anulación requiere aprobación de quien tenga el permiso; quien vendió no se autoaprueba | Control interno y auditoría |
| RN-VEN-004 | El vendedor no ve costos ni márgenes, en ninguna pantalla ni endpoint | Protección del dato comercial sensible |
| RN-VEN-005 | Las ventas son por sucursal: se atienden, anulan y reportan en su sucursal | Multi-sucursal nativo; reportes por local |
| RN-VEN-006 | La venta no se borra nunca: se anula (estado) | Trazabilidad contable y fiscal |

## 9. Preguntas frecuentes

**¿El cliente pagó solo parte de la cuenta corriente, qué se paga primero?**
La deuda más vieja (FIFO). La ficha del cliente muestra qué ventas quedaron con
saldo después de aplicar el pago.

**¿Se puede editar una venta cerrada?**
No: se anula y se hace una nueva. La anulación deja rastro y devuelve stock.

**¿Qué pasa si dos cajas reclaman el mismo pedido al mismo tiempo?**
El primero gana; el segundo recibe "ya reclamado" (409) sin romper nada.

**¿El vendedor puede dar descuentos?**
Según su política de rol (tope configurado); encima del tope, solo un administrador.

## 10. Glosario

| Término | Explicación en una línea |
|:--|:--|
| POS | Punto de venta: la pantalla de cobro del sistema |
| Pedido de mostrador (counter order) | Venta pendiente con código QR que se cobra después en caja |
| TTL | Tiempo de vida: cuánto dura algo antes de vencer (72 h para pedidos) |
| FIFO | Primero entra, primero sale: se paga primero la deuda más vieja |
| Cuenta corriente | Vender hoy y cobrar después, con vencimientos y saldo |
| Anulación | Dejar una venta sin efecto (sin borrarla), con aprobación y auditoría |
| Scoping | Cada usuario ve solo los datos de su sucursal |
| SKU de variante | El código de la versión concreta del producto (talle/color); escanearlo elige esa variante |
| SIFEN | Sistema fiscal paraguayo de comprobantes electrónicos |

## 11. Trazabilidad

| Artefacto | Referencia |
|:--|:--|
| Planes ejecutados (conductor/, en git history) | counter-orders (FASES 1-5B + auditoría, BE 9e4ad69/3b26ccf, FE c654d1b), monorol+descuentos+crédito (BE 8427719…, FE 006c8c1…), vendor role & branch (100% 2026-09-06..09-10), variant flat search + filtros historial (2026-09-14), BI drill-downs ventas (RF-BIPACK-021..024, 2026-09-22) |
| Dominio FE | `src/domain/sale/calculations/`, `src/domain/purchase/pricing/`; features `src/features/sales/`, `src/features/transfers/` |
| Tests | Suite FE 1104/1104 (2026-09-22); E2E counter-orders como gate de SQL real |
| REQ relacionados | REQ_BIPACK (BI sobre ventas) · REQ_DEPLOY (ventas por sucursal en la red de la instalación) |

## 12. Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v0.1 | 2026-09-22 | Borrador inicial de backfill (módulo existente): formaliza POS, pedidos de mostrador, crédito FIFO y anulaciones | renpereiradx + agente |
