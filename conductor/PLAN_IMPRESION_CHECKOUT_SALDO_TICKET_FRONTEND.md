# PLAN — Impresión al finalizar el checkout de venta + saldo pendiente en documentos + fix print historial

Fecha: 2026-09-30 · Repos: business_management (BE) + erp-webapp (FE)

## Problema

1. Tras cobrar en el wizard de venta (`/ventas` → Concretar Venta) no se ofrece
   imprimir: el ticket vive solo en el detalle de venta y, para ventas NO
   fiscales, ni ahí (el panel fiscal muestra "Venta sin documento fiscal" sin
   acciones). Ir al detalle tras cada cobro es un flujo extra en caja.
2. **Pago parcial**: el ticket ESC/POS y el comprobante PDF muestran TOTAL +
   pagos + cambio, pero NUNCA el saldo pendiente — un cliente que paga una
   fracción recibe un comprobante que no dice que debe el resto.
3. `SalesPaymentHistory.jsx` ("Imprimir") usa `window.print()` crudo → imprime
   la SPA entera. El propio `printTicket` de counterorders lo documenta como
   anti-patrón.

## Diseño

### BE — documents (saldo pendiente)

- `ticket_renderer.go`: tras el bloque de pagos, si
  `data.Total − Σ AmountApplied > tol` (tol 0.5 en moneda de 0 decimales,
  0.005 en el resto) → línea en negrita `SALDO PENDIENTE … Gs. X`. Aplica a
  tickets fiscales (zona informativa, fuera del KuDE prescripto) y no-fiscales.
- `pdf_renderer.go` → `paymentsRows`: misma fila bold al final.
- Tests: fixture parcial (contiene SALDO PENDIENTE + monto) y completo (no lo
  contiene) en `ticket_renderer_test.go`; equivalente en `pdf_renderer_test.go`.

### FE — modal de éxito con impresión (wizard)

- Nuevo `features/sales/components/SaleSuccessPrintModal.tsx` (EnhancedModal):
  tras cobro exitoso (COMPLETO o PARCIAL — "Dejar pendiente" NO lo abre) muestra
  venta, total, cobrado y (si hay) saldo pendiente con badge "Cobro parcial".
  Acciones: **Imprimir ticket** (`fiscalService.printTicket`, deshabilitado con
  hint si no hay RECEIPT activa+default o sin `documents:read`), **Descargar
  PDF** (`downloadComprobantePdf`, blob con auth) y **Listo**. El resumen se
  consulta a `salePaymentService.getSalePaymentStatus` (verdad del BE; en modo
  merge el total del carrito NO es el total de la venta).
- `SalesNew.tsx`: `receiptSaleId` state; `onConfirmWizard` lo setea con
  `checkoutSaleId` antes de `resetSaleState()` (ambos modos: merge y venta
  nueva). Modal se cierra y limpia.
- VNDR01/CAJA01 ya tienen `documents:read` (mig 20260814203053) — funciona en
  caja sin cambios de permisos.

### FE — detalle de venta (no fiscales)

- `SaleFiscalPanel.tsx` rama `isNotFiscal`: agrega "Imprimir ticket" +
  "Descargar comprobante" reutilizando `reprintTicket`/`downloadPdf` del hook
  (el BE ya soporta ambos para ventas no fiscales). Gated a `documents:read`,
  hint de impresora igual que el branch fiscal.

### FE — fix print historial de cobros

- Nuevo util compartido `src/lib/printHtml.ts`: imprime HTML en iframe efímero
  (patrón counterorders, `@page` configurable). `printTicketHtml` de
  counterorders delega (misma firma exportada — transfers y tests no cambian).
- `SalesPaymentHistory.jsx`: "Imprimir" arma el reporte (venta, totales,
  estado, tabla de cobros) y lo manda por el util — ya no `window.print()`.

### i18n

Keys nuevas `sales.checkoutWizard.receipt.*`, `fiscal.panel.notFiscal.print*`,
`sales.paymentHistory.print.*` en `locales/{es,en}/sales.js` + `fiscal.js`.

## Fuera de alcance

- `handleDownload` (Exportar) de SalesPaymentHistory sigue muerto (console.log)
  — preexistente, no se toca.
- Migración completa .jsx→.tsx de SalesPaymentHistory (touch quirúrgico, no
  heavily-touched; queda documentado aquí).

## Hallazgos durante el smoke (preexistentes, corregidos en el mismo diff)

El smoke en vivo contra dev:5050 reveló que **el render de documentos estaba
roto en producción-de-facto**: ticket/render, comprobante.pdf, email — todo
500 para cualquier venta con pagos. Tres drifts acumulados (el módulo se
validó solo con tests unitarios, nunca end-to-end):

1. `payment_repo.go` GetPaymentDetails: `pm.code` → la columna real es
   `method_code` (baseline + resto del archivo).
2. Mismo query: `payment_reference`/`payment_notes` (VARCHAR) se escaneaban a
   `sql.NullInt32` → scan error con cualquier pago auto ("AUTO-PAY-SO-…");
   además el valor escaneado nunca llegaba al struct (se descartaba). Ahora
   `NullString` + asignación a `PaymentReference`/`PaymentNotes`.
3. `sale_repo.go` expone `curr.name AS currency` ("Guaraní") y el bridge de
   documentos lo pasaba a `GetByCode` → 404 "moneda no encontrada". El FE ya
   normaliza ese nombre (`normalizeCurrencyCode`), así que no se toca el repo:
   el bridge ahora resuelve el ISO code por `currency_id` (GetByID) y
   `loadCurrency` degrada a `DefaultCurrency` en 404 (principio documentado:
   el render no hard-falla por metadatos de formato).
4. Observabilidad: `writeDocsAppErr` se comía el error envuelto (500
   indepurable). Ahora loguea la causa (`documents_internal_error`).

Smoke final en dev:5050 con binario nuevo: venta PAID → 200 sin línea de
saldo; venta parcial (Gs. 1.000 de 3.630) → 200 con
`SALDO PENDIENTE 2.630` tras el pago; PDF 200 estructuralmente válido.

## Gates

`go test ./internal/documents/...` + `go build`; `npx vitest --run`,
`npx tsc --noEmit`, `pnpm build`, `pnpm lint:design` en verde.
