# GUIDE_SUCURSAL_NUEVA — Conectar una sucursal por VPN

## 1. Ficha

| Campo | Valor |
|:--|:--|
| Módulo | Sucursales (VPN + alta de entidad) |
| Código | GUIDE-SUCURSAL |
| Versión | v1.0 |
| Estado | Vigente (add-branch.sh verificado E2E; el join real de Tailscale se ejecuta en sitio del cliente) |
| Fecha | 2026-09-22 |
| Audiencia | Implementación / soporte |
| Requerimiento | [`REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md`](../requirements/REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md) (RF-DEPLOY-008, 009; RN-DEPLOY-003/004) |

---

## 2. La idea en una frase

El software **ya es multi-sucursal** (ventas por sucursal, transferencias de stock
entre locales, usuarios con acceso por sucursal): abrir una sucursal es alta de
**entidad + usuarios** en el ERP y una **red privada (VPN)** para que las máquinas
de ese local lleguen al servidor central — sin exponer nada a internet.

**Limitación que se informa al cliente**: si se corta el internet de la sucursal,
esa sucursal queda sin sistema hasta que vuelva (modo offline fuera de alcance v1,
RN-DEPLOY-004). El local principal no se afecta.

---

## 3. Red: Tailscale con una tailnet por cliente

**Decisión de titularidad (F0): una tailnet POR CLIENTE** — la red privada de un
cliente jamás ve la de otro, y el acceso del vendor se concede dentro de la tailnet
del cliente que pide soporte.

### 3.1 En el servidor central (una vez)

```bash
# instalar Tailscale (el script de soporte no instala paquetes por su cuenta)
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --hostname erp-srv-principal
# autenticar con la cuenta del CLIENTE (se crea su tailnet en el momento)
```

### 3.2 Claves pre-aprobadas para las sucursales

En la consola admin de Tailscale **del cliente** (login del cliente):
Settings → Keys → **Auth key**: expiración corta, "pre-approved", reusable según
política. Con esa clave, cada máquina de la sucursal se une:

```bash
# en la máquina/terminal-server de la sucursal:
curl -fsSL https://tailscale.com/install.sh | sh   # o instalador Windows de tailscale.com
sudo tailscale up --auth-key tskey-auth-... --hostname erp-suc-centro
```

Tailscale atraviesa CGNAT (común en ISPs de Paraguay) sin abrir puertos del router.
Alternativa aceptada: WireGuard puro con port-forward explícito (solo si el cliente
rechaza terceros — documentar la excepción).

### 3.3 Verificar la red

```bash
# desde la máquina de la sucursal:
ping <ip-tailscale-del-servidor>                 # debe responder
bash business_management/deploy/scripts/add-branch.sh \
    --server https://<ip-tailscale-del-servidor> --check-url   # debe imprimir la versión
```

---

## 4. Alta de la sucursal en el ERP

Desde una máquina que alcance el servidor (la de la sucursal sirve — valida la VPN):

```bash
# 1) JWT de un ADMINISTRADOR (login normal)
TOKEN=$(curl -sk -X POST https://<servidor>/api/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"..."}' | jq -r .access_token)

# 2) crear la sucursal + dar acceso a usuarios existentes
business_management/deploy/scripts/add-branch.sh \
    --server https://<servidor> --token "$TOKEN" \
    --code SUC2 --name "Sucursal Centro" --type POINT_OF_SALE \
    --grant-user <user_id_vendedor>:full
```

El script verifica alcance → crea la entidad (`POST /branches`) → otorga accesos
(`POST /branches/{id}/access`) → verifica `/version`. Los tipos válidos:
`POINT_OF_SALE`, `WAREHOUSE`, `HEADQUARTERS`, `DISTRIBUTION_CENTER`.

---

## 5. Verificación final (checklist)

```
[ ] Máquina de la sucursal: candado sin advertencia en https://<servidor> (CA ya instalada)
[ ] Usuario de la sucursal hace login y su sucursal queda como default
[ ] Venta de prueba en la sucursal queda registrada con su sucursal
[ ] Stock de la sucursal: carga inicial de inventario (transferencia desde el principal)
[ ] Escaneo de puertos públicos del servidor: solo 80/443 responden (nada de DB/Redis/SSH)
[ ] El internet de la sucursal se corta → se informa la limitación (ya sabida, RN-DEPLOY-004)
```

---

## 6. Soporte del vendor (con consentimiento)

```bash
# EN EL SERVIDOR DEL CLIENTE, solo con autorización explícita del cliente:
sudo ./business_management/deploy/scripts/support-vpn.sh --consent \
    --auth-key tskey-auth-...   # clave de la tailnet DEL CLIENTE, expiración corta
```

- El vendor ve el servidor dentro de la tailnet del cliente y lee `/version`
  (inventario de instalaciones: cliente, versión, IP tailnet → `$ERP_ROOT/support-vpn-info.txt`).
- Al terminar: `sudo ./business_management/deploy/scripts/support-vpn.sh --revoke`.
- Cada acceso se registra en el acta de soporte del cliente.

---

## 7. Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v1.0 | 2026-09-22 | Primera versión contra `add-branch.sh` y `support-vpn.sh` reales; decisión tailnet-por-cliente documentada | renpereiradx + agente |
