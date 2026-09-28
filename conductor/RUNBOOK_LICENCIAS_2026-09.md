# RUNBOOK — Licencias BI Pack Premium (2026-09)

**Plan:** [PLAN_BI_PACK_PREMIUM.md](./PLAN_BI_PACK_PREMIUM.md) (ADR-1/ADR-4, F6)
**Implementación:** BE `9dd9942` (business_management), FE `b3ab4eb` (erp-webapp)
**Audiencia:** vendedor / ops — emisión, instalación, renovación, revocación y deploy prod.

---

## 1. Modelo en 60 segundos

- La licencia es un **archivo JSON firmado (ed25519)** que declara `edition`, `customer`,
  `modules` (`["bi"]`), `issued_at` y `expires_at` (`null` = perpetua).
- El backend la lee en boot y **hot-reload por mtime**: instalar, renovar o revocar no requiere
  reinicio.
- `LICENSE_ENFORCE=false` (default) = fail-open: todo se comporta como licenciado (dev).
  `LICENSE_ENFORCE=true` = el pack BI solo funciona con licencia vigente; **ni el admin la
  bypasea**.
- El FE consulta `entitlements` (login/`/me`) y `GET /api/v1/system/license` para ocultar nav,
  redirigir rutas BI a `/pedidos` y mostrar banners. El enforcement real es siempre server-side.

## 2. Emisión (máquina del VENDEDOR — la clave privada jamás viaja)

```bash
cd business_management
go build -o /tmp/licensegen ./cmd/licensegen

# 2a. Par de claves (una vez por "marca" / llave maestra):
/tmp/licensegen keygen -private ~/custodia-licencias/vendor.key -public ~/custodia-licencias/public.key
#   → private: hex seed, chmod 600. CUSTODIA FUERA DEL REPO (backup offline del vendedor).
#   → public:  base64; este valor es el que recibe el cliente.

# 2b. Emitir la licencia de un cliente (MODELO NORMAL: compra única, PERPETUA —
#     sin -days la licencia no vence jamás; decisión de negocio 2026-09-22):
/tmp/licensegen issue \
  -key ~/custodia-licencias/vendor.key \
  -customer "Cliente Demo Sucursal Centro" \
  -edition core+bi -modules bi \
  -out ~/entregas/cliente-demo-centro/license.json
# Casos especiales ÚNICAMENTE a pedido (demo/trial con plazo):
#   -days 365            → vence al año
#   -expires 2027-09-21T12:00:00Z → fecha fija (gana sobre -days).
```

Entrega al cliente: **`license.json` + la clave pública (base64)**. Nunca la privada.

## 3. Instalación (servidor del CLIENTE)

1. Copiar `license.json` al servidor (default: junto al binario, `./license.json`; cualquier ruta
   sirve configurando `LICENSE_FILE`).
2. Configurar en el `.env` (o environment del servicio):

   ```ini
   LICENSE_FILE=./license.json
   LICENSE_PUBLIC_KEY=<clave pública base64 que acompaña la licencia>
   LICENSE_GRACE_DAYS=0          # opcional: días de gracia post-vencimiento
   LICENSE_ENFORCE=true          # SOLO después de verificar el punto 3
   ```

3. Verificación pre-flip (con `LICENSE_ENFORCE=false` o antes de reiniciar):
   `curl -s -H "Authorization: Bearer <token>" http://localhost:5050/api/v1/system/license`
   → `"status":"active"` + `days_remaining` razonable.

## 4. Renovación

1. Emitir nueva licencia (§2b) con `-days` nuevo o `-expires` futuro (mismo par de claves —
   la pública del cliente no cambia).
2. Reemplazar `license.json` en el servidor (copy-over atómico o mv).
3. Al siguiente request el backend recarga por mtime: `system/license` debe reportar
   `active` con el nuevo vencimiento. **Sin reinicio.**

## 5. Revocación / baja del pack

- **Inmediata (remoto no posible por diseño):** pedir al cliente borrar `license.json` — o
  hacerlo en una visita/remoto acordado. Al desaparecer el archivo el pack se apaga en caliente
  (`403 MODULE_NOT_LICENSED`, nav BI desaparece, landing pasa a `/pedidos`).
- **Expiración natural:** dejar pasar `expires_at` (+ `LICENSE_GRACE_DAYS` si se pactó). El FE
  muestra banner de aviso desde 30 días antes.

## 6. Provisioning de instalaciones existentes (¡antes del flip!)

Una instalación que hoy usa BI **no debe perderlo** al activar enforcement:

1. Emitir licencia con BI para el cliente (§2b).
2. Instalar archivo + `LICENSE_PUBLIC_KEY` (§3, con `LICENSE_ENFORCE=false` todavía).
3. Verificar `system/license` → `active`.
4. Flip `LICENSE_ENFORCE=true` y reiniciar (o solo recargar el servicio). El hot-reload cubre el
   archivo, no las env vars: cambiar `LICENSE_ENFORCE` requiere reinicio.

## 7. Checklist de deploy prod

- [ ] `LICENSE_FILE` apunta a la licencia instalada y legible por el usuario del servicio.
- [ ] `LICENSE_PUBLIC_KEY` = la pública del par con el que se firmó.
- [ ] `GET /api/v1/system/license` → `active` (o `grace` si aplica).
- [ ] `GET /api/v1/users/me` → `entitlements.modules` incluye `"bi"` (si el cliente compró BI).
- [ ] `LICENSE_ENFORCE=true` recién ahora. Reiniciar.
- [ ] Smoke post-deploy: `GET /dashboard/summary` con token admin → 200 (con pack) o 403
      `MODULE_NOT_LICENSED` (ERP Core, esperado).
- [ ] Sin `LICENSE_PUBLIC_KEY` con enforcement ON la instalación queda **ERP Core por diseño**
      (fail-closed): el log muestra `license_enforce_without_public_key`.

## 8. Troubleshooting

| Síntoma | Causa probable | Acción |
|:--------|:---------------|:-------|
| `status:"none"` con archivo presente | `LICENSE_FILE` apunta a otra ruta / permisos | Verificar ruta y usuario del servicio |
| `status:"invalid"` — signature verification failed | Licencia de otro cliente, o archivo editado a mano | Re-emitir (§2b) |
| `status:"invalid"` — public key not configured | Falta `LICENSE_PUBLIC_KEY` | Cargar la pública y reiniciar |
| `status:"expired"` | Venció (+grace consumido) | Renovar (§4) |
| 403 `MODULE_NOT_LICENSED` con licencia `active` | `modules` de la licencia no incluye `"bi"` | Re-emitir con `-modules bi` |
| FE no muestra BI tras renovar | Entitlements en cache de sesión | Re-login o esperar al 403+refresh (revocación en caliente) |

## 9. Verificación E2E ejecutada (2026-09-21, dev)

Con licencia real emitida por `licensegen`, backend efímero con `LICENSE_ENFORCE=true`, JWT de
admin real: sin archivo → 403 `MODULE_NOT_LICENSED` en toda la superficie BI (admin incluido) y
`/me` con `entitlements` core; instalación del archivo → 200 en caliente, `/system/license`
`active` (364 días), `/me` `core+bi` con `bi_expires_at`; vencida → 403 `expired`; renovada →
200; borrada → 403. Core intacto (`/api/v1/settings` → 200). Login de cliente real: verificado
por contrato en tests (el usuario dev de prueba fue purgado de la BD).
