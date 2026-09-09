# Hallazgo: "el vendedor puede ajustar los descuentos" — dual-role F2VLso en usuarios de prueba (2026-09-09)

> Reporte del usuario tras la auditoría de pedidos. Verificación E2E (API + UI
> real vía navegador) como usuario vendor. **No hubo bug de código**: los gates
> B.3/FASE 4 funcionan; la causa fue contaminación de datos en la DB dev.

## Síntoma

Logueado como vendor, /ventas mostraba la pestaña "Nueva Venta" con el POS
completo, incluido "Precio Final de Venta" (ajustador proporcional de precios)
y los editores de descuento.

## Causa raíz

El usuario de prueba tenía **dos roles: F2VLso (ADMIN) + VNDR01**. El login
toma `roles[0]` (`identity/service.go:147`, con TODO "Handle multiple roles
properly") → el JWT salía con `role_id=F2VLso` → bypass admin en middleware y
en `hasPermission` del FE. Cualquier descuento que "aplicaba el vendedor" era
un admin aplicándolo.

Alcance de la contaminación: **3.656 usuarios** con F2VLso + rol operativo
(1.828 `buyer_*`, 1.828 `vendor_*`, y restos `supplies_*`/`scoped_*`/`caja_*`
del seeding de carga del 11/08). **Cero usuarios con nombre real**; `hercortes`
estaba limpio (VNDR01 only).

## Limpieza aplicada (DB dev, SQL directo)

```sql
DELETE FROM users.user_roles ur
WHERE ur.role_id='F2VLso' AND ur.user_id IN (
  SELECT user_id FROM users.user_roles
  GROUP BY user_id HAVING COUNT(*) > 1 AND bool_or(role_id='F2VLso'));
-- DELETE 3656; dual-role con ADMIN restantes: 0; usuarios sin roles: 0
```

Los 917 holders de F2VLso restantes son single-role (`admin` real + `admin_*`
de prueba). Sin migración: es limpieza de datos dev; para producción, auditar
`user_roles` antes del deploy (ver recomendaciones).

## Verificación post-limpieza (vendor VNDR01-only)

- `/me`: roles `[VNDR01]`; sin `sales:write`, `sales:apply_discount`, `cash:write`.
- `POST /sale/` con `sale_price`+descuento → **403** (policy B.3).
- `POST /sale/pos-checkout` → **403** (gate `cash:write` de FASE 4).
- UI: /ventas muestra **solo Historial**; sin POS ni ajustador de precios;
  nav acotada a módulos vendor.

## Notas de diseño (pendientes de decisión)

1. **Multi-role silencioso**: con 2+ roles el JWT lleva `roles[0]` sin regla
   determinista para el usuario. Cualquier dual-role futuro vuelve a ser un
   bypass invisible. Sugerencia: constraint de rol único por usuario, o rol
   primario explícito + error de login si hay ambigüedad.
2. **`GetByUsername` falla en silencio** si cualquier columna escaneada de
   `parties` es NULL (lección E2E ya conocida por `email`; esta vez fue
   `last_name` NULL en el usuario de prueba — corregido a `''` para esa fila).
   Candidato a hardening: COALESCE o scan nullable.

## Artefactos de la verificación (limpiados)

- Venta de prueba `SALE-1788994713-572` (+ pago 80 + stock_tx 762) eliminada;
  stock de `Cc2y5JnvR` base@b1 restaurado a 41.
- Password temporal del usuario de prueba: el hash original no se preservó
  (error de procedencia mío); quedó con password vacío — la cuenta ya era
  inaccesible antes de la sesión (login roto por `last_name` NULL).
- Servidor temporal :5051 apagado; los de :3001/:5050 no se tocaron.
