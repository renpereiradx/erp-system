# GUIDE_TERMINALES_LOCAL — Alta de una terminal en el local

## 1. Ficha

| Campo | Valor |
|:--|:--|
| Módulo | Terminales del local (cajas, vendedores) |
| Código | GUIDE-TERMINAL |
| Versión | v1.0 |
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

- Windows 10/11 (o cualquier SO con navegador moderno: Chrome/Edge/Firefox).
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

**Firefox** (usa su propio almacén de certificados, ignorar si el local no usa Firefox):
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
[ ] Login con usuario propio (no compartir usuarios entre cajas)
[ ] Impresora de tickets imprime página de prueba (cajas)
[ ] Queda registrada qué máquina es (inventario simple del local)
```

---

## 4. Preguntas frecuentes

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

## 5. Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v1.0 | 2026-09-22 | Primera versión contra la implementación real (CA interna + proxy `/api` + install-ca-windows.ps1) | renpereiradx + agente |
