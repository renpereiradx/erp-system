# GUIDE_TERMINALES_LOCAL — Alta de una terminal en el local

## 1. Ficha

| Campo | Valor |
|:--|:--|
| Módulo | Terminales del local (cajas, vendedores) |
| Código | GUIDE-TERMINAL |
| Versión | v1.4 |
| Estado | Vigente (escrita contra el perfil `server` real: proxy `/api` + CA interna; §5 móviles recién documentado — probar en el primer dispositivo real) |
| Fecha | 2026-10-15 |
| Audiencia | Implementación / soporte |
| Requerimiento | [`REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md`](../requirements/REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md) (RF-DEPLOY-004, 005, 008; RNF-DEPLOY-001) |

---

## 2. La idea en una frase

Una terminal es **un navegador y una dirección** — una PC del local, una tablet
en el mostrador o el teléfono del dueño (§5). No se instala el sistema: se
instala la **confianza** (el certificado de la CA interna) y se crea el **usuario**.

---

## 3. Alta de una terminal nueva (por cada máquina)

### 3.1 Requisitos de la máquina

- Windows 10/11 o Linux de escritorio (cualquier SO con Chrome/Edge actualizado;
  ver §4 — navegador recomendado). Teléfonos y tablets: §5.
- Red del local (cable o WiFi) con salida al servidor.
- Impresora de tickets si la caja imprime (la impresión sale del navegador:
  ver el módulo de impresión de documentos).

### 3.2 Confiar en la CA interna

Sin este paso el navegador muestra advertencia de "conexión no segura". El por
qué de esa advertencia (y por qué el dev local no la muestra) está explicado en
[`GUIDE_NAVEGADORES_CERTIFICADOS.md`](GUIDE_NAVEGADORES_CERTIFICADOS.md), con
los pasos por navegador (Chrome, Edge, Brave, Firefox).

**Windows (Chrome/Edge — sirve para toda la máquina):**

```powershell
# PowerShell COMO ADMINISTRADOR, con la CA copiada del servidor o compartida por red:
.\install-ca-windows.ps1 -CaCertPath \\servidor\erp\deploy\certs\ca\ca.crt
```

El script importa la CA al store de la máquina y verifica que quedó instalada.

**Linux (Chrome/Edge heredan el trust store del sistema):**

```bash
sudo ./install-ca-linux.sh /ruta/ca.crt
# Debian/Ubuntu: copia a /usr/local/share/ca-certificates/ + update-ca-certificates
# Fedora/RHEL:   copia a /etc/pki/ca-trust/source/anchors/ + update-ca-trust
```

**Firefox** (usa su propio almacén de certificados, en cualquier SO; ignorar si
el local no usa Firefox):
Configuración → Privacidad y seguridad → Certificados → Ver certificados →
Autoridades → Importar → elegir `ca.crt` → marcar "sitios web".

**Verificación:** abrir `https://<hostname>/` — el candado debe aparecer **sin
advertencias**.

### 3.3 Acceso y usuario

1. Abrir `https://<hostname>/` y crear acceso directo en el escritorio.
2. Login con el usuario que le creó el administrador (los usuarios se crean desde
   Configuración → Usuarios por un admin; los permisos dependen del rol:
   ver los 9 roles canónicos en Configuración).
3. Si el usuario es de una sucursal concreta, el admin le asignó acceso a esa
   sucursal (`GUIDE_SUCURSAL_NUEVA.md` §4) — el usuario trabaja su sucursal.

### 3.4 Registro en el sistema (código de vinculación)

Con varias cajas en el local conviene que cada máquina quede **registrada** en
el sistema: el servidor sabe a qué sucursal pertenece y audita su última
actividad. El trámite es una vez por máquina y son dos pasos:

1. **Administrador** → Configuración → **Terminales** → "Nueva terminal":
   nombre (ej. "Caja 1"), sucursal, y copiar el **código de vinculación**
   (8 caracteres).
2. **En la máquina de la caja** → Configuración → **Este dispositivo** →
   pegar el código → **Registrar**. Listo: quienes la usen sin permiso para
   cambiar de sucursal entran directo a la sucursal asignada.

Sin el paso 2 también funciona: en "Este dispositivo" se puede tocar una
sucursal para vincular **solo ese navegador** (se pierde si se limpian los
datos del navegador). El registro con código, en cambio, queda en el servidor.
Una máquina ya registrada administra su sucursal desde Terminales: "Este
dispositivo" muestra el estado y permite desvincularla, no cambiarla.

> Sin el pack Premium la edición opera **1 terminal activa por sucursal**
> (RN-BIPACK-10): la segunda caja de una misma sucursal pide el pack.

### 3.5 Checklist final por terminal

```
[ ] Candado sin advertencia en https://<hostname>/
[ ] Navegador con perfil aislado del ERP (§4) o políticas aplicadas
[ ] Login con usuario propio (no compartir usuarios entre cajas)
[ ] Impresora de tickets imprime página de prueba (cajas)
[ ] Terminal registrada en el sistema (§3.4) si la caja va fija a una sucursal
[ ] Queda registrada qué máquina es (inventario simple del local)
```

---

## 4. El navegador: recomendación y perfil aislado del ERP

### 4.1 Recomendación

**Chrome (estándar del cliente) o Edge**, en cualquier SO de la terminal. Razones:

- El frontend usa tecnología moderna (React 19, Tailwind 4, efectos de vidrio con
  `backdrop-filter`): exige navegadores recientes. Como las terminales no tienen
  internet, instalar el navegador reciente y dejarlo — no se auto-actualiza.
- Chrome y Edge leen certificados del almacén del sistema (Windows y Linux): la CA
  se instala **una vez por máquina** (§3.2). Firefox usa su propio almacén: paso
  manual extra por terminal.
- Para cajas, el modo kiosk deja el navegador a pantalla completa encerrado en el ERP.

### 4.2 Perfil aislado: lo tipeado en el ERP no se mezcla con el Chrome personal

El navegador guarda en su autofill lo que se tipea en formularios (clientes, RUCs,
teléfonos) y lo vuelve a sugerir **en cualquier sitio**. Para que los datos del
cliente del ERP no aparezcan en el navegador personal de nadie, el ERP se abre con
un **perfil propio** (carpeta separada del navegador normal):

**Windows:**

```powershell
.\setup-terminal-chrome.ps1 -Url https://<hostname>            # ventana normal
.\setup-terminal-chrome.ps1 -Url https://<hostname> -Kiosk     # caja: pantalla completa
```

Crea el acceso directo "ERP" en el escritorio (usa Chrome con
`--user-data-dir=%LOCALAPPDATA%\ERP\chrome-erp`).

**Linux:**

```bash
./setup-terminal-chrome.sh --url https://<hostname>            # ventana normal
./setup-terminal-chrome.sh --url https://<hostname> --kiosk    # caja: pantalla completa
# Chromium en vez de Chrome: sumar --chromium
```

Crea el lanzador "ERP" (menú de aplicaciones y escritorio).

El uso diario del empleado (YouTube, mail personal, etc.) va en el navegador
normal: ni lo tipeado en el ERP aparece ahí, ni lo del navegador personal entra al
ERP. Todo lo que viajó en el bundle incluye estos scripts (`deploy/scripts/`).

### 4.3 Políticas (opcional, clientes estrictos o cajas compartidas)

Si además se quiere apagar el guardado de datos del navegador en esa máquina,
ambos scripts aplican políticas con un flag (sin dominio ni servidor: el flag
escribe el registro/JSON local). Requiere administrador:

```powershell
.\setup-terminal-chrome.ps1 -Url ... -ApplyPolicies   # Windows (HKLM, como admin)
sudo ./setup-terminal-chrome.sh --url ... --policies   # Linux (/etc/opt/chrome/policies/managed)
```

Apaga: autofill de formularios y tarjetas, inicio de sesión con cuenta Google +
sync (evita que datos del local se sincronicen a una cuenta personal que después
se usa en casa), guardado de contraseñas del ERP y historial.

> Chrome/Edge mostrarán "administrado por tu organización": es esperable, avisar
> al cliente.

> **Terminales compartidas entre empleados:** con solo el perfil aislado (§4.2),
> lo tipeado en el ERP se sugiere *dentro* del ERP entre turnos. Si eso no se
> acepta, aplicar también las políticas (§4.3), que apagan el autofill del todo.

---

## 5. Teléfonos y tablets (Android / iOS)

Un celular o tablet es **una terminal más**: nada del sistema se instala, solo
un navegador. Lo que cambia respecto de una PC son dos cosas — cómo se instala
la confianza (la CA) en Android/iOS, y cómo se llega desde afuera del local
(app de Tailscale, el mismo túnel privado de las sucursales).

### 5.1 Qué esperar (antes de prometerle algo al cliente)

- **Tablet**: sirve como terminal de consulta y venta ligera en el mostrador.
- **Teléfono**: para que el dueño **revise números, apruebe o consulte** desde
  cualquier lado. La interfaz está diseñada para escritorio (densidad de datos,
  atajos de teclado, modo kiosk): **no prometer el POS completo en un teléfono**.
- **Impresión de tickets**: sale del navegador de las PCs del local; desde móvil
  no está soportado ni probado. Las cajas siguen siendo PCs.
- Navegador actualizado (Chrome en Android, Safari en iOS): misma exigencia de
  §4.1.

### 5.2 Dentro del local (mismo WiFi que el servidor) — igual que una PC

**Requisitos:** el teléfono conectado al WiFi del local, y el archivo `ca.crt`
del servidor (`deploy/certs/ca/ca.crt`) pasado al teléfono. El `ca.crt` es la
parte **pública** del sello — puede mandarse por mail o cable sin problema (lo
que importa no es esconderlo, es que alguien decida confiar en él).

**Android (una vez por teléfono):**

1. Pasar `ca.crt` al teléfono (mail, cable USB, etc.) y descargarlo.
2. Ajustes → buscar **"instalar certificado"** — el camino exacto varía por
   marca y versión de Android; el buscador de Ajustes lo encuentra siempre →
   elegir **Certificado de CA** → `ca.crt` → aceptar.
3. Android va a avisar que "la red podría supervisar el tráfico": es el aviso
   estándar para CUALQUIER certificado, no un problema del nuestro. Continuar.
4. Verificar: abrir Chrome → `https://<hostname>/` → el candado debe aparecer
   **sin advertencia**.

**iPhone / iPad (una vez por equipo):**

1. Enviarse `ca.crt` por mail, abrir el adjunto → aviso "Perfil descargado".
2. Ajustes → General → **VPN y gestión de dispositivos** → el perfil de la CA
   del ERP → **Instalar** (pide el código del equipo).
3. **El paso que casi nadie hace y sin él no funciona:** Ajustes → General →
   Información → **Ajustes de confianza de certificados** → activar el
   interruptor de la CA del ERP. Sin esto, Safari sigue advirtiendo aunque el
   perfil ya esté instalado.
4. Verificar: abrir Safari → `https://<hostname>/` → candado sin advertencia.

> Firefox en Android usa su propio almacén de certificados (como en escritorio,
> §3.2). En móvil usar Chrome directamente.

La dirección es **la misma que usan las PCs del local**. Si en el teléfono el
nombre no resuelve, es un tema del router del local (DNS) — lo resuelve quien
configuró el servidor; no es un problema del teléfono ni del ERP.

Login: usuario propio creado por el admin, igual que §3.3.

### 5.3 Desde afuera del local (la casa del dueño, un depósito) — app Tailscale

Es el túnel privado de las sucursales (RN-DEPLOY-003) con el teléfono de
"bolsillo": nada del servidor queda expuesto a internet.

1. **Instalar la app de Tailscale** (gratis: Play Store / App Store).
2. **Unir el teléfono a la tailnet DEL CLIENTE** — cómo, según de quién sea el
   equipo:
   - **Teléfono personal del dueño (recomendado):** invitar su cuenta como
     usuario de la tailnet desde la consola de Tailscale del cliente; en la app
     se loguea con su propia cuenta Google/Apple. Sin claves que vencen.
   - **Tablet compartida del local:** unirla con clave pre-aprobada, igual que
     una máquina de sucursal (`GUIDE_SUCURSAL_NUEVA.md` §3.2).
3. Con la app conectada, abrir en el navegador `https://<hostname>/` — el mismo
   nombre que usan las PCs; dentro de la tailnet lo resuelve MagicDNS (por eso
   el servidor se une con `--hostname` igual al del certificado, §3.1 de la
   guía de sucursales). Mismo candado, mismo login: funciona como en el local.

**El candado en este camino:** con la CA ya instalada (§5.2) funciona tal cual.
Alternativa futura para **evitar instalar la CA en cada móvil**: los
certificados HTTPS de Tailscale (nombres `...ts.net`), aceptados por
RN-DEPLOY-002 — requieren una configuración única en el servidor que hoy no
está automatizada en el instalador; hasta que exista, el camino es CA
instalada + app Tailscale.

**Si se corta el internet**, el teléfono de afuera queda sin sistema hasta que
vuelva (RN-DEPLOY-004) — la misma limitación que se informa a las sucursales.

### 5.4 Datos del navegador en el teléfono

Los perfiles aislados de §4.2 no existen en móvil: el ERP corre en el navegador
personal. En el teléfono del dueño es su decisión — recomendarle **rechazar**
cuando el navegador ofrezca guardar la contraseña del ERP. En tablets
compartidas del local, crear un **usuario Android separado** (o el modo
invitado) y entrar al ERP siempre desde ahí.

### 5.5 Checklist por móvil

```
[ ] Candado sin advertencia en https://<hostname>/ (en el local y, si aplica, por Tailscale)
[ ] Login con usuario propio (no la cuenta del admin)
[ ] Se probó EN el móvil lo que se va a hacer desde ahí (ver / aprobar / vender)
[ ] (Remoto) App Tailscale queda logueada y el nombre del servidor responde
[ ] El equipo queda en el inventario del local (qué es, de quién, desde dónde entra)
```

---

## 6. Preguntas frecuentes

**¿Se puede usar desde casa?**
No por defecto: el servidor vive en la red del local. Desde un teléfono o
tablet: app de Tailscale (§5.3). Desde una computadora: la VPN de sucursal
(`GUIDE_SUCURSAL_NUEVA.md` §3).

**¿Se corta la luz/red del local, qué pasa con las terminales?**
Sin red no hay sistema en las terminales; el servidor (si tiene UPS) sigue
operativo. El modo offline de terminales está fuera del alcance v1
(RN-DEPLOY-004 del REQ).

**¿Cada actualización del sistema toca las terminales?**
No. Actualizar el servidor es todo (`GUIDE_INSTALACION_NUEVO_CLIENTE.md` §7);
las terminales solo refrescan el navegador.

---

## 7. Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v1.4 | 2026-10-15 | §3.4 nueva: registro de la terminal en el sistema (Configuración → Terminales + código de vinculación en "Este dispositivo"); checklist actualizado | renpereiradx + agente |
| v1.3 | 2026-10-10 | §3.2 referencia a la nueva GUIDE_NAVEGADORES_CERTIFICADOS (el por qué de la advertencia + pasos por navegador) — sin cambios de procedimiento | renpereiradx + agente |
| v1.2 | 2026-09-29 | §5 nuevo: terminales móviles — CA interna paso a paso en Android/iOS (con el paso de confianza de iOS explícito), acceso remoto por app Tailscale (cuenta invitada para el dueño / clave pre-aprobada para tablets), alternativa ts.net de RN-DEPLOY-002 y límites honestos (teléfono = consulta/aprobación, impresión no soportada). FAQ "¿desde casa?" reescrita; §2 y §3.1 mencionan móviles | renpereiradx + agente |
| v1.1 | 2026-09-23 | §4 navegador: perfil aislado del ERP + políticas opcionales (Windows **y Linux**, setup-terminal-chrome.ps1/.sh); CA en Linux (install-ca-linux.sh); SO de terminal sin asumir Windows | renpereiradx + agente |
| v1.0 | 2026-09-22 | Primera versión contra la implementación real (CA interna + proxy `/api` + install-ca-windows.ps1) | renpereiradx + agente |
