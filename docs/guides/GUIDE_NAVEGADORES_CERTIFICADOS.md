# GUIDE_NAVEGADORES_CERTIFICADOS — El aviso de seguridad del navegador y la CA interna

## 1. Ficha

| Campo | Valor |
|:--|:--|
| Módulo | Terminales / deploy on-premise |
| Código | GUIDE-NAVcert |
| Versión | v1.0 |
| Estado | Vigente (escrita contra el instalador real: `gen-ca.sh`, `install-ca-linux.sh`, `install-ca-windows.ps1`, nginx TLS del perfil `server`) |
| Fecha | 2026-10-10 |
| Audiencia | Implementación / soporte (y quien se pregunte "¿por qué a mí me advierte?") |
| Requerimiento | [`REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md`](../requirements/REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md) (RN-DEPLOY-002, RF-DEPLOY-004/005) |
| Guía hermanas | [`GUIDE_TERMINALES_LOCAL.md`](GUIDE_TERMINALES_LOCAL.md) (alta de terminal paso a paso; esta guía explica el POR QUÉ de su §3.2) |

---

## 2. La idea en una frase

El aviso de "riesgo de seguridad" **no significa que el sistema esté inseguro ni
que el navegador detectara un ataque**: significa que hay HTTPS funcionando y el
navegador no reconoce a la autoridad que firmó el certificado — y eso se arregla
haciendo que cada navegador conozca la CA interna una sola vez.

---

## 3. Por qué pasa: los tres escenarios

El navegador solo advierte cuando hay **HTTPS que validar**. Dónde está HTTPS y
dónde no, según cómo se accede al sistema:

| Escenario | URL típica | Protocolo | ¿Advierte? |
|:--|:--|:--|:--|
| Desarrollo local (Vite) | `http://localhost:5173` | HTTP puro | **No** — no hay nada que validar; además los navegadores tratan `localhost` como confiable por definición |
| Cliente, perfil `laptop` | `http://<hostname>:18081` | HTTP puro | **No** — ídem |
| Cliente, perfil `server` | `https://<hostname>/` (443, o `ERP_HTTPS_PORT` custom, ej. `https://localhost:28443` en installs de prueba) | **HTTPS** | **Sí**, hasta que la CA interna se instala en ese navegador (§5) |

El perfil `server` sirve con TLS (nginx, `tls.conf`) usando un certificado que
el instalador genera con `deploy/scripts/gen-ca.sh`: una **CA propia** firma el
certificado del servidor. El certificado **sí cubre el nombre** que se usa
(los SAN incluyen el hostname, `localhost`, `127.0.0.1` y las IPs de la
máquina), así que la advertencia nunca es por el nombre: es solo de confianza
en la firma.

> **"Sin advertencia" ≠ "seguro".** El dev local y el perfil `laptop` no
> advierten porque viajan **sin cifrado** — no hay chequeo que falle, no porque
> el navegador certifique nada. El que *más* protege es el perfil `server`
> (HTTPS): su única "deuda" es hacerle conocer la CA a cada navegador, que es
> exactamente lo que esta guía resuelve.

### Glosario mínimo

- **CA (autoridad certificadora):** una "firma" que avala que un servidor es
  quien dice ser. Las CAs públicas (conocidas por los navegadores de fábrica)
  avalan dominios de internet; para una red interna sin dominio público se usa
  una CA propia (RN-DEPLOY-002).
- **Certificado autofirmado / CA interna:** el aval lo da el instalador del
  ERP, no una empresa externa. Es el diseño correcto para installs
  on-premise: el cliente no tiene ni necesita un dominio en internet.
- **Trust store (almacén de confianza):** la "libreta de firmas" que cada
  navegador consulta. Chrome, Edge y Brave en Windows y Linux leen la del
  **sistema operativo**; Firefox lleva la **suya propia** (y por eso es el
  único que sigue advirtiendo después de instalar la CA en el sistema).
- **SAN (Subject Alternative Name):** la lista de nombres/IPs para los que un
  certificado es válido. La del ERP incluye hostname, `localhost`,
  `127.0.0.1` y las IPs del host (`gen-ca.sh`).

---

## 4. Qué almacén lee cada navegador

| Navegador | Motor | Almacén de confianza | ¿Basta `install-ca-windows.ps1` / `install-ca-linux.sh`? |
|:--|:--|:--|:--|
| Chrome | Chromium | Del sistema (Win: CryptoAPI / Linux: trust store + p11-kit) | ✅ Sí |
| Edge | Chromium | Del sistema | ✅ Sí — idéntico a Chrome |
| Brave | Chromium | Del sistema (no tiene gestión de CA propia en su configuración) | ✅ Sí — idéntico a Chrome |
| Firefox | Gecko | **Propio** (NSS, dentro del perfil del navegador) en **cualquier SO** | ❌ No — importar a mano o activar `security.enterprise_roots.enabled` (§5.4) |

En Linux, `install-ca-linux.sh` copia la CA a
`/usr/local/share/ca-certificates/` (Debian/Ubuntu) o
`/etc/pki/ca-trust/source/anchors/` (Fedora/RHEL) y reconstruye el trust store:
Chrome, Edge y Brave la heredan de ahí. Firefox ignora ese lugar por diseño.

---

## 5. Dejar el candado sin advertencia, por navegador

En todos los casos: la CA está en `deploy/certs/ca/ca.crt` del servidor (es la
parte **pública** del sello — puede copiarse por red, mail o pendrive sin
problema). Después de instalarla, **cerrar y reabrir el navegador**: si estaba
abierto durante la instalación puede no recargar el almacén.

### 5.1 Chrome (Windows y Linux)

**Windows (PowerShell como administrador):**

```powershell
.\install-ca-windows.ps1 -CaCertPath \\servidor\erp\deploy\certs\ca\ca.crt
```

**Linux:**

```bash
sudo ./install-ca-linux.sh /ruta/ca.crt
```

**Verificación:** abrir `https://<hostname>/` (o `https://localhost:28443` en
una install de prueba) → candado en la barra, sin advertencia. Si advierte,
reiniciar Chrome y reintentar.

### 5.2 Edge

Idéntico a Chrome: mismo motor Chromium, mismo almacén del sistema. Correr el
mismo script de §5.1 (sirve para ambos a la vez) y verificar igual. No hay paso
propio de Edge.

### 5.3 Brave

Idéntico a Chrome: Brave es Chromium y **no ofrece** importar certificados
desde su configuración — siempre lee el trust store del sistema. Correr el
mismo script de §5.1 y verificar igual. Recordar reiniciar Brave por completo
(si usa varias ventanas, cerrarlas todas).

> Para terminales del cliente, Chrome o Edge siguen siendo los recomendados
> (`GUIDE_TERMINALES_LOCAL.md` §4.1: navegador moderno sin auto-actualización +
> perfil aislado + kiosk). Brave funciona igual de bien si el local ya lo usa.

### 5.4 Firefox (el caso que advierte aunque la CA esté en el sistema)

Firefox usa su propio almacén en cualquier sistema operativo. Dos caminos — el
(a) es el recomendado porque es determinístico:

**a) Importar la CA a mano (una vez, por perfil de Firefox):**

1. Menú ≡ → **Ajustes** → **Privacidad y seguridad** → bajar a
   **Certificados** → **Ver certificados**.
2. Pestaña **Autoridades** → **Importar…** → elegir `ca.crt`.
3. En el diálogo, marcar **"Confiar en esta autoridad para identificar
   sitios web"** → Aceptar.
4. Reiniciar Firefox.

**Verificación:** abrir `https://<hostname>/` → candado sin advertencia.

**b) Que Firefox lea el almacén del sistema (Linux):**

1. En la barra de dirección escribir `about:config` → aceptar el aviso.
2. Buscar `security.enterprise_roots.enabled` → ponerlo en `true`.
3. Reiniciar Firefox. (Requiere que la CA ya esté en el sistema por
   `install-ca-linux.sh`, §5.1.)

En Windows los Firefox recientes ya leen el almacén del sistema por defecto,
pero el camino (a) funciona siempre y no depende de la versión.

**⚠️ El camino NO recomendado:** en la propia página de advertencia,
**Advanced… → Accept the Risk and Continue**. Es por sitio y temporal: vuelve
con cada advertencia nueva y entrena a los usuarios a aceptar alertas de
seguridad sin leerlas. Usarlo solo para mirar una vez desde una máquina
ajena al local.

---

## 6. Verificación final (smoke test)

Por máquina y por navegador que se use para operar:

```
[ ] https://<hostname>/ abre con candado y SIN advertencia
[ ] (server) http://<hostname>/ redirige solo a HTTPS (301) — no queda en HTTP
[ ] Login visible (si advierte en otro navegador, repetir §5 para ese navegador)
```

---

## 7. Problemas frecuentes

**Firefox advierte pero Chrome no.**
Causa: la clásica. Firefox no lee el trust store del sistema (§4).
Solución: §5.4 — importar la CA a mano o activar `security.enterprise_roots.enabled`.

**La advertencia dice que el certificado no es válido para el nombre que tipeé.**
Causa: se entra por un nombre o IP fuera de los SAN del certificado (ej. una
IP nueva de la red, un alias de DNS agregado después).
Solución: regenerar certificado con `gen-ca.sh` pasando el hostname extra o la
IP como argumento (los suma a los SAN), o entrar por el nombre original.
Ver el detalle de SAN en §3.

**Instalé la CA y el navegador sigue advirtiendo.**
Causas en orden de frecuencia: (1) el navegador estaba abierto durante la
instalación → cerrarlo por completo y reabrir; (2) el script falló sin sudo /
sin administrador → correrlo de nuevo y leer su salida; (3) Firefox → §5.4.
Verificar en Linux: `ls /usr/local/share/ca-certificates/` debe listar
`erp-internal-ca.crt`.

**En la barra de dirección un sitio HTTP aparece como "No es seguro" (Chrome/Edge/Brave).**
Es solo la etiqueta informativa de HTTP plano — en `localhost` ni eso aparece.
No es el aviso de certificado de esta guía; el HTTP plano solo existe en dev
y en el perfil `laptop` (§3).

**¿Por qué no usar un certificado "de verdad" (Let's Encrypt, etc.)?**
Requiere un dominio público y renovación online; las installs del cliente son
offline y sin dominio (RN-DEPLOY-002). La CA interna es el diseño correcto:
instalarla cuesta un comando por máquina (§5.1) y no depende de internet.

---

## 8. Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v1.0 | 2026-10-10 | Primera versión: por qué el dev local (HTTP) no advierte y el deploy server (HTTPS + CA interna) sí; qué almacén lee cada navegador (Chrome/Edge/Brave = sistema, Firefox = propio); pasos de confianza por navegador y troubleshooting | renpereiradx + agente |
