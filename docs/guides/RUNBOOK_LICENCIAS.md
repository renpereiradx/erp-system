# RUNBOOK — Licencias BI Pack Premium (2026-09 · v2.3 rutas unificadas ~/licencias)

**Plan:** [PLAN_BI_PACK_PREMIUM.md](../../conductor/PLAN_BI_PACK_PREMIUM.md) (ADR-1/ADR-4, F6) · **v2.0:** REQ_BIPACK (RF-025..029, RN-08/09)
**Implementación:** BE `9dd9942` (licencias) + v2.0 2026-09-28 `441046e` (trial/bloqueo/rescate) + v2.1 2026-09-29 (auditoría: permiso `license:write`, licencia vencida→Core, ancla fail-open) · FE `b3ab4eb` + v2.0 `91b41de` (gate de licencia) + v2.1 (instalación desde Configuración, auto-recuperación del gate)
**Audiencia:** vendedor / ops — emisión, instalación, evaluación, bloqueo, rescate web, renovación y deploy prod.

---

## 1. Modelo en 60 segundos

- **Sin licencia:** la instalación corre COMPLETA (Core+BI) en **evaluación por 30 días desde
  el primer arranque** (ancla en `system_installation.first_boot_at`, sembrada por migración /
  baseline). Agotados: **bloqueo total** — solo login, refresh, logout y `/system/license`
  (ver + cargar) responden; el resto devuelve 403 `LICENSE_EXPIRED` y el FE muestra la
  pantalla "licencia requerida".
- **Licencia Core:** perpetua (`expires_at: null`, `-modules core`) — BI apagado, ERP Core 100%,
  **1 terminal POS activa por sucursal** (RN-10: las terminales adicionales son del Premium).
- **Licencia Core+BI:** perpetua (`expires_at: null`, `-modules bi`) — todo el pack, terminales
  sin límite. La evaluación vigente también cuenta como Premium para las terminales.
- Catálogo de paquetes para el cliente (comercial/soporte): `docs/guides/GUIDE_PAQUETES_CLIENTE.md`.
- La licencia es un **archivo JSON firmado (ed25519)**; el backend la lee en boot con
  **hot-reload por mtime**: instalar, renovar o revocar no requiere reinicio.
- `LICENSE_ENFORCE=true` (default en installs del instalador) activa el control; **ni el admin
  lo bypasea**. `false` solo en dev (fail-open, sin trial ni bloqueos).
- El rescate es **web** y requiere el permiso `license:write` (administrador — v2.1): desde
  la pantalla de bloqueo **o Configuración → Licencia → "Instalar licencia…"** se carga el
  `license.json` (PUT `/api/v1/system/license`), el backend verifica la firma antes de
  persistir (un archivo vencido se rechaza con 400) y el sistema desbloquea en caliente. No
  hace falta SSH.
- Una licencia firmada pero VENCIDA (demo) **degrada a Core**: BI apagado, ERP operativo —
  no bloquea el sistema ni re-regala la evaluación (v2.1). Solo la evaluación agotada sin
  licencia bloquea; el reloj del trial siempre corre desde el primer arranque.

## 2. Emisión (máquina del VENDEDOR — la clave privada jamás viaja)

> **Cómo leer esta sección si no sos desarrollador:** cada recuadro gris es una
> lista de comandos para la **terminal** de la computadora del vendedor; se
> tipea **una línea por vez** y se aprieta Enter. `~/` significa "mi carpeta
> personal" (ej. `/home/miusuario`), y las carpetas que falten se crean solas.
> Analogía de las dos claves: la **privada** es tu firma manuscrita — con ella
> se "firma" cada licencia y NADIE más debe verla; la **pública** es la forma
> de comprobar que una firma es tuya — viaja al cliente y no es secreto.

```bash
# 0) Construir la herramienta de licencias (una vez por máquina; necesita Go
#    y el repo del ERP — pedir a desarrollo si no los tenés):
cd ~/dev/erp-system/business_management     # ajustar a dónde esté el repo
go build -o /tmp/licensegen ./cmd/licensegen

# 2a. Par de claves — UNA sola vez en la vida del negocio (la "llave maestra"
#     con la que se firmarán TODAS las licencias de TODOS los clientes):
/tmp/licensegen keygen -private ~/licencias/custodia/vendor.key -public ~/licencias/custodia/public.key
chmod 600 ~/licencias/custodia/vendor.key   # solo tu usuario puede leerla

# 2a-bis. BACKUP YA: copiar la carpeta ~/licencias/ COMPLETA (custodia +
#     entregas) a un pendrive o bóveda FUERA de esta máquina — nunca a la
#     nube (Dropbox/Drive sync): la privada saldría de tu control. Si se
#     pierde la privada, no se pueden emitir licencias nuevas para los
#     clientes que ya la usan; si se filtra, cualquiera podría fabricarse
#     licencias propias.
#   → private: hex seed, chmod 600. CUSTODIA FUERA DEL REPO (backup offline del vendedor).
#   → public:  base64; este valor es el que recibe el cliente.

# 2b. Emitir la licencia de un cliente (MODELO NORMAL: compra única, PERPETUA —
#     sin -days la licencia no vence jamás; decisión de negocio 2026-09-22):
#   Core+BI (todo el pack):
/tmp/licensegen issue \
  -key ~/licencias/custodia/vendor.key \
  -customer "Cliente Demo Sucursal Centro" \
  -edition core+bi -modules bi \
  -out ~/licencias/entregas/cliente-demo-centro/license.json
#   Core SIN BI (perpetua, BI apagado — REQ_BIPACK v2.0 RF-029):
# /tmp/licensegen issue -key ~/licencias/custodia/vendor.key \
#   -customer "Cliente Core" -edition core -modules core \
#   -out ~/licencias/entregas/cliente-core/license.json
# Casos especiales ÚNICAMENTE a pedido (demo/trial con plazo):
#   -days 365            → vence al año
#   -expires 2027-09-21T12:00:00Z → fecha fija (gana sobre -days).
```

**Flags de edición y módulos (qué significa cada uno):**

| Flag | Valores | Qué hace |
|:--|:--|:--|
| `-edition` | `core` · `core+bi` (default) | Etiqueta **comercial**: es el texto que muestra *Configuración → Licencia* del cliente. Es **informativo** — no prende ni apaga nada. El backend solo exige que no venga vacío, pero usar únicamente estos dos valores del catálogo ([`GUIDE_PAQUETES_CLIENTE.md`](GUIDE_PAQUETES_CLIENTE.md)): cualquier otro string se mostraría tal cual en la pantalla del cliente. |
| `-modules` | `bi` · `core` | El que **manda** de verdad: con `bi` el pack BI queda habilitado y las terminales POS no tienen límite; sin `bi` la instalación corre como ERP Core con 1 terminal POS activa por sucursal (RN-10). |

Regla práctica: `-edition core+bi -modules bi` para Premium, `-edition core
-modules core` para Core — siempre ambos alineados con lo que compró el cliente.

> **Varios clientes / correr `issue` varias veces:** es el flujo normal.
> `keygen` se corre UNA sola vez en la vida del negocio (se niega a pisar una
> clave privada existente; correrlo de nuevo con otras rutas crearía un segundo
> par = otra "marca" — no hacerlo sin decisión explícita). `issue` se corre una
> vez por cliente: cada llamada escribe un archivo **independiente** firmado
> con el **mismo** par de claves, y el comando NO lleva registro de lo emitido
> — la carpeta `~/licencias/entregas/<cliente>/` ES ese registro. Ojo: `-out` sobre un
> archivo existente lo **sobrescribe sin aviso**; un subfolder por cliente lo
> evita.

> **Roles de las dos carpetas:** todo vive bajo `~/licencias/`.
> `~/licencias/custodia/` es la **bóveda** — solo el par maestro; la privada
> jamás sale de esta máquina (salvo backup offline) y nada más entra.
> `~/licencias/entregas/<cliente>/` es la **salida** — las licencias que
> viajan a cada cliente. Regla corta: `-key` siempre apunta a la custodia
> (lee), `-out` siempre a entregas (escribe). El padre `~/licencias/` es la
> unidad de backup: pendrive/bóveda física, **nunca sync de nube** (§2a-bis).

Entrega al cliente: **`license.json` + la clave pública**. Nunca la privada. La forma más simple de entregarlos es meter ambos DENTRO del bundle (§2c): el instalador los detecta solo.

### 2c. Cargar licencia + clave en el bundle (flujo normal — autodetección)

El instalador busca la pareja dentro del bundle, en `business_management/`,
con esos nombres exactos (`license.json` y `licensing-public.key`), y la
detecta solo **sin ningún flag**: en el wizard pregunta por cada archivo
detectado ([S/n], Enter acepta) y en modo `--yes`/sin terminal la toma en
silencio. Mismo comportamiento en Linux y en Windows/WSL2 (es el mismo
instalador). Además valida la forma de la pareja antes de instalar — corta
con mensaje claro si la "clave pública" es en realidad la privada (hex), si
la licencia no tiene payload/signature o si la clave no es base64 de 32
bytes; y si la clave llegó con OTRO nombre (ej. `public.key` de la
custodia), el wizard lo avisa con el nombre exacto que busca. La
verificación de la FIRMA sigue siendo del backend al arrancar.

El `.tar.gz` que genera `make-release-bundle.sh` NO trae la pareja: copiarla
y re-empaquetar. Reemplazar `<v>` por la versión del bundle (ej. `v1.0.0-4-g01ed28a`).

```bash
cd ~/dev/erp-system/business_management/dist/erp-release-<v>/
cp ~/licencias/custodia/public.key business_management/licensing-public.key
cp ~/licencias/entregas/<cliente>/license.json business_management/license.json

cd ..    # re-empaquetar: el .tar.gz viejo no contiene los archivos nuevos
tar czf erp-release-<v>.tar.gz erp-release-<v>/
```

- `sha256sum -c SHA256SUMS` en el cliente sigue dando OK: el manifiesto
  verifica solo los archivos listados; los agregados se ignoran.
- Cliente sin licencia todavía: incluir **igual** la clave pública (habilita
  el rescate web cuando compre); sin `license.json` arranca la evaluación de
  30 días.
- Instalar desde la carpeta DESCOMPRIMIDA (pendrive ya extraído) no requiere
  re-tar; re-empaquetar solo si se distribuye el `.tar.gz`.
- Opcional (evita el re-tar): `make-release-bundle.sh --license <archivo>
  --license-public-key <clave>` embebe la pareja al emitir y la deja cubierta
  por `SHA256SUMS`; también acepta solo la clave pública. El copiado manual
  de arriba es el flujo normal.
- Detalle completo del lado del cliente: [`GUIDE_INSTALACION_NUEVO_CLIENTE.md`](GUIDE_INSTALACION_NUEVO_CLIENTE.md) §2.1 y §5.

## 3. Instalación (servidor del CLIENTE)

**Vía instalador (v2.0, recomendada):** `install.sh` acepta `--license <archivo.json>`
y `--license-public-key <public.key>` (o los pide en el wizard; con licencia la clave es
obligatoria). Ambos archivos quedan copiados en `$ERP_ROOT/business_management/license/`,
el mount del contenedor, y el `.env.docker` se genera con `LICENSE_ENFORCE=true` +
`LICENSE_PUBLIC_KEY` + `LICENSE_TRIAL_DAYS=30`. Sin `--license`, la instalación arranca en
evaluación (30 días) y ya puede rescatar por web (la pública, si se pasó, hace el resto).

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

## 6. Provisioning de instalaciones existentes (v2.1)

Desde REQ_DEPLOY v0.8 el instalador maneja el flip solo:

1. Emitir licencia con BI para el cliente (§2b).
2. Correr `install.sh --license <archivo> --license-public-key <clave>` (ambos flags,
   obligatorios juntos): copia la licencia, embebe/inyecta la clave pública en `.env.docker`
   y deja `LICENSE_ENFORCE=true`.
3. Verificar `GET /system/license` → `status: active`.

El camino manual (licencia + `LICENSE_PUBLIC_KEY` con `LICENSE_ENFORCE=false`, verificar
`active`, recién entonces flip a `true` + reinicio) queda para instalaciones NO gestionadas
por el instalador: el hot-reload cubre el archivo, no las env vars — cambiar
`LICENSE_ENFORCE` siempre requiere reinicio.

**Instalación vieja actualizada sin licencia:** el update activa `LICENSE_ENFORCE=true` y la
evaluación corre 30 días desde la migración del ancla (`system_installation`) — avisar al
cliente: al vencer, bloqueo con rescate web.

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
| Evaluación ya vencida en un install recién actualizado | El ancla `system_installation` se sembró al aplicar la migración | Normal: 30 días desde el update. Activación adelantada vía §4b |

## 9. Verificación E2E ejecutada (2026-09-21, dev)

Con licencia real emitida por `licensegen`, backend efímero con `LICENSE_ENFORCE=true`, JWT de
admin real: sin archivo → 403 `MODULE_NOT_LICENSED` en toda la superficie BI (admin incluido) y
`/me` con `entitlements` core; instalación del archivo → 200 en caliente, `/system/license`
`active` (364 días), `/me` `core+bi` con `bi_expires_at`; vencida → 403 `expired`; renovada →
200; borrada → 403. Core intacto (`/api/v1/settings` → 200). Login de cliente real: verificado
por contrato en tests (el usuario dev de prueba fue purgado de la BD).
