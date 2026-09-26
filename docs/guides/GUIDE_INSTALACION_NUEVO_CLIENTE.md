# GUIDE_INSTALACION_NUEVO_CLIENTE — Instalar el ERP en un cliente nuevo

## 1. Ficha

| Campo | Valor |
|:--|:--|
| Módulo | Instalación on-premise de clientes |
| Código | GUIDE-INST-CLIENTE |
| Versión | v1.0 |
| Estado | Vigente (probada contra el ensayo E2E del plan PLAN_DEPLOY_ESCALADO_ONPREMISE) |
| Fecha | 2026-09-22 |
| Audiencia | Implementación / soporte |
| Requerimiento | [`REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md`](../requirements/REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md) (RF-DEPLOY-001..003, 011..014) |

---

## 2. Qué necesitás

**Hardware del servidor** (perfil `server`) o laptop (perfil `laptop`):

| Requisito | Mínimo |
|:--|:--|
| SO | Linux x64 (Debian/Ubuntu recomendado). Perfil `laptop` admite cualquier SO con Docker Desktop |
| RAM | 4 GB (8 GB recomendado) |
| Disco | 20 GB libres (más el crecimiento de la base) |
| Red | Perfil `server`: IP fija en la red del local (LAN). Perfil `laptop`: sin requisitos |
| Software | Docker Engine + plugin compose (`docker compose version`). **Nada más**: PostgreSQL y Redis corren dentro del stack |

**Del lado del vendor:**

- El **bundle de release**: `erp-release-<versión>.tar.gz` (lo genera `business_management/deploy/scripts/make-release-bundle.sh`).
- La **licencia del cliente**: `license.json` (generada con `cmd/licensegen`; ver REQ_BIPACK si el cliente compra el pack BI).

---

## 3. El bundle: qué es y cómo verificarlo

El bundle es un paquete cerrado con todo: imágenes Docker (backend + frontend),
compose canónico, migraciones, baseline de schema, seeds RBAC, instalador y
config nginx del frontend. No necesita internet en el cliente.

```bash
# dentro del bundle descomprimido, verificar integridad:
sha256sum -c SHA256SUMS        # debe terminar sin "FAILED"
```

Estructura del bundle:

```
erp-release-<v>/
├── business_management/   # compose, migraciones, seeds, baseline, instalador
├── erp-webapp/nginx/      # config nginx del frontend (template + tls.conf)
└── images/*.tar.gz        # imágenes Docker del release
```

---

## 4. Elegir perfil

| | `laptop` | `server` |
|:--|:--|:--|
| Para | Cliente chico, todo en una máquina | Local con varias terminales / sucursales |
| Frontend | `http://localhost:8081` **solo en esa máquina** | `https://<hostname>/` (80 → 443) para toda la red |
| TLS | No | Sí (CA interna, el instalador la genera) |
| Exposición | Ninguna (todo loopback) | Solo 80/443; backend/DB loopback; Redis interno |
| Backup diario | ✅ (script + cron o systemd) | ✅ (script + timer systemd) |

> Regla: si hoy es laptop pero el cliente va a crecer, instalar `server` directo
> en una PC dedicada — migrar después es un trámite (`GUIDE_MIGRACION_LAPTOP_A_SERVIDOR`),
> pero empezar con `laptop` en una máquina compartida termina mal.

---

## 5. Instalar

```bash
tar xzf erp-release-<v>.tar.gz
cd erp-release-<v>/
sudo ./business_management/deploy/scripts/install.sh \
    --profile server \
    --hostname erp.miempresa.local \
    --license ~/ruta/license.json \
    --yes
```

El instalador es **idempotente**: se puede re-ejecutar sin romper nada (útil si
algo falla a mitad de camino, y para **actualizar** una instalación: bundle nuevo
→ mismo comando → migraciones de delta aplicadas, imágenes reemplazadas, datos
y secretos intactos).

Qué hace (12 pasos, todos con salida visible):

1. Pre-chequeos (SO Linux, Docker corriendo, disco ≥ 5 GB, puertos libres)
2. Copia el stack a `/opt/erp/` (la instalación vive ahí; el repo queda fuera)
3. Genera `.env.docker` con **secrets nuevos** (JWT + password de DB aleatorios)
4. Levanta PostgreSQL (servicio propio con volumen dedicado, solo loopback)
5. Carga las imágenes del bundle (`docker load`) — sin internet
6. Genera la CA interna y el cert del servidor (solo perfil `server`)
7. Migraciones: base nueva → baseline de schema + estampado de versión; base existente → delta con golang-migrate
8. Seeds RBAC solo si la base está vacía (9 roles canónicos)
9. Usuario admin inicial con **contraseña aleatoria** guardada en `/opt/erp/credentials/`
10. Licencia: si está, `LICENSE_ENFORCE=true`; si no, modo gracia con aviso
11. Levanta el stack completo y espera healthchecks
12. **Smoke test automático**: health + `/version` + frontend + proxy `/api` (+ redirect HTTPS en server). Si algo falla, el instalador muere con el error visible

Al terminar imprime un resumen con URLs, ubicación de la CA y próximos pasos.

### Overrides útiles

| Situación | Variable / flag |
|:--|:--|
| Puerto 80/443 ocupado | `ERP_HTTP_PORT=8080 ERP_HTTPS_PORT=8443` antes del comando |
| Otro PostgreSQL ya existe en el servidor | mover el puerto del stack: `ERP_DB_PORT=5433` |
| Instalar sin licencia por ahora | omitir `--license` (queda en modo gracia) |
| Probar sin tocar `/opt` | `ERP_ROOT=/tmp/erp-test ./install.sh ...` |

---

## 6. Post-instalación (orden obligatorio)

1. **Guardar la contraseña del admin** (`/opt/erp/credentials/admin-inicial.txt`)
   en el gestor del cliente y **borrar el archivo**.
2. **Probar login** desde el navegador del servidor: `https://<hostname>/`.
   La instalación limpia siembra la sucursal **MAIN "Sucursal Principal"**
   (datos placeholder, con config fiscal `timbrado 00000000`): elegir esa
   sucursal al entrar y **cargar los datos reales del negocio** (nombre,
   RUC, timbrado vigente) en *Configuración → Sucursales* antes de facturar.
3. **Distribuir la CA a las terminales** → [`GUIDE_TERMINALES_LOCAL.md`](GUIDE_TERMINALES_LOCAL.md).
4. **Configurar backup externo**: el backup diario vive en `/opt/erp/backups/`
   (30 días de retención). Copiarlo fuera de la máquina (disco externo/nube) —
   el backup en el mismo disco no salva ante robo/fuego.
5. Si es perfil `laptop`: **cifrado de disco activado** (BitLocker/LUKS) — exigirlo
   al cliente antes de cargar datos reales.
6. Registrar el cliente en el **inventario de instalaciones**: cliente, versión
   (`curl -k https://<host>/api/version`), fecha, hostname.

---

## 7. Actualizar un cliente existente

```bash
# con un bundle más nuevo:
sudo ./business_management/deploy/scripts/install.sh --profile server \
    --hostname erp.miempresa.local --yes
```

El `.env.docker` existente (secretos) se conserva; las imágenes nuevas del bundle
se cargan y el stack se recrea con ellas; las migraciones nuevas se aplican como
delta. **Antes de actualizar: verificar que existe el backup de anoche**
(`ls /opt/erp/backups/`) — es la única reversión real (RN-DEPLOY-007).

---

## 8. Problemas frecuentes

| Síntoma | Causa probable | Qué hacer |
|:--|:--|:--|
| `docker compose: command not found` | Falta el plugin compose | `apt install docker-compose-plugin` |
| El instalador muere en "frontend HTTPS no responde" | Otra cosa ocupa 80/443 | Revisar `ss -tln` y usar `ERP_HTTP_PORT/ERP_HTTPS_PORT` |
| Login OK pero sin datos y sin roles | Bundle sin baseline (muy viejo) | Regenerar bundle (`make-release-bundle.sh` lo exige) |
| El navegador de las terminales da advertencia TLS | CA no instalada en esa terminal | `GUIDE_TERMINALES_LOCAL.md` |
| Advertencia "LICENSE_ENFORCE=false" en el instalador | Falta license.json | Copiarla a `/opt/erp/business_management/license/` y re-ejecutar |

---

## 9. Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v1.0 | 2026-09-22 | Primera versión, escrita contra el instalador real (F2-F3 del plan) y el ensayo E2E | renpereiradx + agente |
