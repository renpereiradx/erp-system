# conductor/ (workspace-level)

Este directorio **raíz** (no el de `business_management/conductor/` ni `erp-webapp/conductor/`)
centraliza la **coordinación cross-repo** del ERP Vista Bar: fixes que tocan backend y frontend a la
vez, estado de la integración, y planes que no viven en un solo repo.

## Contenido

| Archivo | Rol |
|:--------|:----|
| `ESTADO_INTEGRACION_BACKEND.md` | ✅ **Cerrado** (2026-08-03): fixes B1–B5 commiteados, suite 24 paquetes ok / 0 FAIL. |
| `ESTADO_INTEGRACION_FRONTEND.md` | Estado del frente frontend: tracking F1–F7 (F1–F4 🔴 corregidos, F5/F7 pendientes, F6 bloqueado por decisión de producto). |
| `PLAN_SERVICIOS_Y_RESERVAS_CONFIGURABLES.md` | ✅ **Implementado** (2026-09-01): S1–S5 completas — venta de SERVICE sin stock, `business_settings` con toggle de reservas, `is_bookable` explícito. Registro en §7 del plan. |
| `PLAN_INTEGRACION.md` | *(próximamente)* Plan ordenado de integración FE↔BE post-auditoría. |

## Siguiente frente

El **frontend** (`erp-webapp/`) es el frente activo. Arrancar desde
`ESTADO_INTEGRACION_FRONTEND.md`: F5 (dead code) es el siguiente pase de limpieza, y **F6
(`updateSale` cancela en vez de actualizar) requiere decisión de producto antes de tocarse**: ¿la
venta es inmutable post-create (solo se cancela) o necesita edición real?

## Referencias

- Auditoría base: `../AUDITORIA_INTEGRACION_FRONTEND_BACKEND.md` (inventario de hallazgos verificados).
- Plan hexagonal (cerrado): `../business_management/conductor/PLAN_HEXAGONAL_REFACTOR.md`.
