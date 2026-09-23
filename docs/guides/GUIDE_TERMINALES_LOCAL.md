# GUIDE_TERMINALES_LOCAL — Alta de una terminal en el local

## 1. Ficha

| Campo | Valor |
|:--|:--|
| Módulo | Terminales del local (cajas, vendedores) |
| Código | GUIDE-TERMINAL |
| Versión | v1.1 |
| Estado | Vigente (escrita contra el perfil `server` real: proxy `/api` + CA interna) |
| Fecha | 2026-09-22 |
| Audiencia | Implementación / soporte |
| Requerimiento | [`REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md`](../requirements/REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md) (RF-DEPLOY-004, 005; RNF-DEPLOY-001) |

---

## 2. La idea en una frase

Una terminal es **un navegador y una dirección**. No se instala el sistema: se
instala la **confianza** (el certificado de la CA interna) y se crea el **usuario**.

---

## 3. Alta de una terminal nueva (por cada máquina)

### 3.1 Requisitos de la máquina

- Windows 10/11 o Linux de escritorio (cualquier SO con Chrome/Edge actualizado;
  ver §4 — navegador recomendado).
- Red del local (cable o WiFi) con salida al servidor.
- Impresora de tickets si la caja imprime (la impresión sale del navegador:
  ver el módulo de impresión de documentos).

### 3.2 Confiar en la CA interna

Sin este paso el navegador muestra advertencia de "conexión no segura".

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

### 3.4 Checklist final por terminal

```
[ ] Candado sin advertencia en https://<hostname>/
[ ] Navegador con perfil aislado del ERP (§4) o políticas aplicadas
[ ] Login con usuario propio (no compartir usuarios entre cajas)
[ ] Impresora de tickets imprime página de prueba (cajas)
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

## 5. Preguntas frecuentes

**¿Se puede usar desde casa?**
No por defecto: el servidor vive en la red del local. Acceso remoto = VPN
(`GUIDE_SUCURSAL_NUEVA.md` §5, con consentimiento del cliente).

**¿Se corta la luz/red del local, qué pasa con las terminales?**
Sin red no hay sistema en las terminales; el servidor (si tiene UPS) sigue
operativo. El modo offline de terminales está fuera del alcance v1
(RN-DEPLOY-004 del REQ).

**¿Cada actualización del sistema toca las terminales?**
No. Actualizar el servidor es todo (`GUIDE_INSTALACION_NUEVO_CLIENTE.md` §7);
las terminales solo refrescan el navegador.

---

## 6. Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v1.1 | 2026-09-23 | §4 navegador: perfil aislado del ERP + políticas opcionales (Windows **y Linux**, setup-terminal-chrome.ps1/.sh); CA en Linux (install-ca-linux.sh); SO de terminal sin asumir Windows | renpereiradx + agente |
| v1.0 | 2026-09-22 | Primera versión contra la implementación real (CA interna + proxy `/api` + install-ca-windows.ps1) | renpereiradx + agente |
