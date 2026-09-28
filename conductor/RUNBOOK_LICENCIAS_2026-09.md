# RUNBOOK — Licencias BI Pack Premium (2026-09 · v2.0 trial + rescate)

**Plan:** [PLAN_BI_PACK_PREMIUM.md](./PLAN_BI_PACK_PREMIUM.md) (ADR-1/ADR-4, F6) · **v2.0:** REQ_BIPACK (RF-025..029, RN-08/09)
**Implementación:** BE `9dd9942` (licencias) + v2.0 2026-09-28 (trial/bloqueo/rescate) · FE `b3ab4eb` + v2.0 (gate de licencia)
**Audiencia:** vendedor / ops — emisión, instalación, evaluación, bloqueo, rescate web, renovación y deploy prod.

---

## 1. Modelo en 60 segundos

- **Sin licencia:** la instalación corre COMPLETA (Core+BI) en **evaluación por 90 días desde
  el primer arranque** (ancla en `system_installation.first_boot_at`, sembrada por migración /
  baseline). Agotados: **bloqueo total** — solo login, refresh, logout y `/system/license`
  (ver + cargar) responden; el resto devuelve 403 `LICENSE_EXPIRED` y el FE muestra la
  pantalla "licencia requerida".
- **Licencia Core:** perpetua (`expires_at: null`, `-modules core`) — BI apagado, ERP Core 100%.
- **Licencia Core+BI:** perpetua (`expires_at: null`, `-modules bi`) — todo el pack.
- La licencia es un **archivo JSON firmado (ed25519)**; el backend la lee en boot con
  **hot-reload por mtime**: instalar, renovar o revocar no requiere reinicio.
- `LICENSE_ENFORCE=true` (default en installs del instalador) activa el control; **ni el admin
  lo bypasea**. `false` solo en dev (fail-open, sin trial ni bloqueos).
- El rescate es **web**: desde la pantalla de bloqueo se carga el `license.json` (PUT
  `/api/v1/system/license`), el backend verifica la firma antes de persistir y el sistema
  desbloquea en caliente. No hace falta SSH.
- El trial NO se reinicia: una licencia vencida (demo) cae al remanente de evaluación; el
  reloj siempre corre desde el primer arranque.

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
#   Core+BI (todo el pack):
/tmp/licensegen issue \
  -key ~/custodia-licencias/vendor.key \
  -customer "Cliente Demo Sucursal Centro" \
  -edition core+bi -modules bi \
  -out ~/entregas/cliente-demo-centro/license.json
#   Core SIN BI (perpetua, BI apagado — REQ_BIPACK v2.0 RF-029):
# /tmp/licensegen issue -key ~/custodia-licencias/vendor.key \
#   -customer "Cliente Core" -edition core -modules core \
#   -out ~/entregas/cliente-core/license.json
# Casos especiales ÚNICAMENTE a pedido (demo/trial con plazo):
#   -days 365            → vence al año
#   -expires 2027-09-21T12:00:00Z → fecha fija (gana sobre -days).
```

Entrega al cliente: **`license.json` + la clave pública (base64)**. Nunca la privada.

## 3. Instalación (servidor del CLIENTE)

**Vía instalador (v2.0, recomendada):** `install.sh` acepta `--license <archivo.json>`
y `--license-public-key <public.key>` (o los pide en el wizard; con licencia la clave es
obligatoria). Ambos archivos quedan copiados en `$ERP_ROOT/business_management/license/`,
el mount del contenedor, y el `.env.docker` se genera con `LICENSE_ENFORCE=true` +
`LICENSE_PUBLIC_KEY` + `LICENSE_TRIAL_DAYS=90`. Sin `--license`, la instalación arranca en
evaluación (90 días) y ya puede rescatar por web (la pública, si se pasó, hace el resto).

**Manual (servidor existente):**

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

## 4b. Activación / rescate web (v2.0 — post-evaluación o post-venta)

Cuando la instalación está bloqueada (evaluación agotada sin licencia) o el cliente compró
después de probar:

1. El cliente entra con su usuario (el login NO está bloqueado): verá la pantalla
   **"Licencia requerida"** con el estado y el botón **Cargar licencia**.
2. Entregarle el `license.json` (email/pendrive) y que lo seleccione ahí. Alternativa
   programática:
   ```bash
   curl -s -X PUT -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
        --data-binary @license.json http://localhost:5050/api/v1/system/license
   ```
3. Respuesta 200 + snapshot `"status":"active"` → el sistema desbloquea **en caliente** (el
   FE reinicia la app al segundo). Un candidato inválido responde 400 `LICENSE_INVALID` con el
   motivo y NO pisa la licencia existente.
4. Si el bloqueo aparece SIN poder loguear (olvidos de contraseña aparte), el path manual
   sigue siendo copiar el archivo a `$ERP_ROOT/business_management/license/license.json` — el
   hot-reload lo aplica igual.

Requisito: `LICENSE_PUBLIC_KEY` configurada (el instalador la pide; sin ella el backend no
puede verificar NINGUNA licencia ni rescatar).

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
| Pantalla "Licencia requerida" (todo bloqueado) | Evaluación agotada sin licencia | Rescate web (§4b): cargar `license.json` desde la propia pantalla |
| PUT de licencia → 400 `LICENSE_INVALID` | Firma inválida / clave pública incorrecta / archivo corrupto | Verificar que la pública sea la del par vendedor; re-emitir (§2b) |
| PUT de licencia → 400 "public key not configured" | `LICENSE_PUBLIC_KEY` vacía | Cargar la pública en `.env.docker` y recrear el contenedor backend |
| Evaluación ya vencida en un install recién actualizado | El ancla `system_installation` se sembró al aplicar la migración | Normal: 90 días desde el update. Activación adelantada vía §4b |

## 9. Verificación E2E ejecutada (2026-09-21, dev)

Con licencia real emitida por `licensegen`, backend efímero con `LICENSE_ENFORCE=true`, JWT de
admin real: sin archivo → 403 `MODULE_NOT_LICENSED` en toda la superficie BI (admin incluido) y
`/me` con `entitlements` core; instalación del archivo → 200 en caliente, `/system/license`
`active` (364 días), `/me` `core+bi` con `bi_expires_at`; vencida → 403 `expired`; renovada →
200; borrada → 403. Core intacto (`/api/v1/settings` → 200). Login de cliente real: verificado
por contrato en tests (el usuario dev de prueba fue purgado de la BD).
