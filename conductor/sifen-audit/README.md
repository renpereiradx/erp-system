# Auditoría de implementación SIFEN

Auditoría fase por fase de la implementación de facturación electrónica (SIFEN,
Paraguay) contra los planes:

- Plan maestro: `business_management/conductor/PLAN_SIFEN_FACTURACION_ELECTRONICA.md`
- Estado: `business_management/conductor/PLAN_SIFEN_FACTURACION_ELECTRONICA_STATUS.md`
- Plan frontend: `erp-webapp/conductor/PLAN_SIFEN_FRONTEND.md`

## Informes

| Informe | Fase auditada | Estado |
|:--|:--|:--|
| [S1.md](./S1.md) | S1 — Contexto `internal/sifen`: dominio puro + schema | ✅ 2026-08-21 — 🔧 remediado 2026-08-22 (H1/H4/H5) |
| [S2.md](./S2.md) | S2 — Adapters: firma XAdES, SOAP, TLS mutua | ✅ 2026-08-21 — ⚠️ 1 hallazgo crítico (envelope SOAP) · 🔧 remediado 2026-08-24 (H1/H2/H6/H7/H4) |
| [S3.md](./S3.md) | S3 — Integración ventas: numeración atómica + emisión | ✅ 2026-08-22 — ⚠️ 1 🔴 nuevo (ventana 72 h deslizante) + 3 🟡; arrastre S2-H1 |
| [S4.md](./S4.md) | S4 — Eventos: cancelación/inutilización; NCE/NDE | ✅ 2026-08-22 — ⚠️ 1 🔴 nuevo (envelope SOAP del evento) + 5 🟡; arrastres S2-H1/S3-H1/S3-H2 · 🔧 remediado 2026-08-24 (H1–H9, H10/H12; H11 registrado como decisión operativa) |
| [S5.md](./S5.md) | S5 — KuDE: PDF/ticket con QR oficial | ✅ 2026-08-22 — ⚠️ 0 🔴 nuevos + 4 🟡 (invoice_number no persistido — verificado contra BD —, estado del DE sin render, sin KuDE de NCE/NDE, QR solo post-envío); arrastres S2-H1/S4-H1 · 🔧 remediado 2026-08-24 (H1–H4 completos; H5 parcial —D131/D012/E644/D213-D216 pendientes—; H6/H7; H8/H9 sin acción) |
| [S6.md](./S6.md) | S6 — Frontend: FE1–FE5 | ✅ 2026-08-22 — ⚠️ 1 🔴 nuevo (botón KuDE PDF roto: window.open sin `/api/v1` ni Authorization) + 2 🟡 (retry del error = reenvío del DE; reprima/email sin gate por estado); i18n y contrato verificados 100 % · 🔧 remediado 2026-08-24 (H1–H8; H9 anotado para S7) |
| [S7.md](./S7.md) | S7 — Volumen, observabilidad, cierre | ✅ 2026-08-22 — ⚠️ 1 🔴 nuevo (`ConsultarCDC` con wire inventado vs MT §9.4: la reconciliación S7.3 no puede funcionar real) + 2 🟡 (curl `/skipped` del runbook → 400; cierre S7.5 con verdades vacuas/prematuras) + 5 🟢; alertas S7.2 verificadas contra BD real (anclaje correcto — mitiga detección de S3-H1) · 🔧 remediado 2026-08-25 (H1–H8, H10; H9-b FE de `/alerts` ✅, H9-a smoke funcional atado a S0.4) |

## Cierre de la serie (S1–S7)

La implementación SIFEN queda ✅ conforme en dominio y arquitectura (CDC/QR/firma/
numeración/KuDE/UI verificados independientemente) y, tras el pase de corrección
de wire, con el camino al DNIT alineado al MT en sus tres vías: ~~envelope SOAP
de `Emitir`~~ (S2-H1 ✅ remediado 2026-08-24), ~~envelope de eventos~~ (S4-H1 ✅
remediado 2026-08-24), ~~ventana del worker anclada a `updated_at`~~ (S3-H1 ✅
remediado 2026-08-24), ~~filtro de rechazos reenviables~~ (S3-H2 ✅ remediado
2026-08-24) y ~~cliente de consulta CDC contra el MT §9.4~~ (S7-H1 ✅ remediado
2026-08-25). Las siete auditorías (S1–S7) están remediadas. Queda como pendiente
real el **S0.4** (certificado del cliente) para los e2e contra `sifen-test`; el
smoke de entonces debe incluir emisión, evento, consulta y reconciliación — las
cuatro vías.
Ver conclusión de [S7.md](./S7.md) §6.

## Metodología

Cada informe audita **plan vs código real**:

1. **Trazabilidad**: cada tarea del plan → commit → archivo, con verificación de
   que lo prometido existe.
2. **Verificación independiente**: los algoritmos fiscales (CDC módulo 11, hash
   del QR) se recalculan fuera del código Go; la estructura XML se contrasta
   contra los XSD v150 vendoreados en `internal/sifen/testdata/xsd/`.
3. **Reglas críticas** del plan (depguard, god-interface, secretos, migraciones)
   y de AGENTS.md.
4. **Ejecución**: `go test ./internal/sifen/`, `golangci-lint`, `make
   migrate-verify-gaps`.

### Escala de severidad

| Nivel | Significado |
|:--|:--|
| 🔴 Alto | Incumplimiento del plan/MT con impacto funcional o fiscal |
| 🟡 Medio | Desviación o riesgo real que puede impactar en fases posteriores |
| 🟢 Bajo | Deuda menor, cosmético o riesgo residual despreciable |
| ℹ️ Info | Observación/contexto sin acción requerida |

> Regla 8 del plan: ante discrepancia plan ↔ MT oficial, gana el MT. La
> auditoría aplica el mismo criterio: el XSD v150 vendoreado es la fuente
> autoritativa de estructura.
