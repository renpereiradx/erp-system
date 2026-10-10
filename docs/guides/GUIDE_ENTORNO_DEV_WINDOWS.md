# GUIDE_ENTORNO_DEV_WINDOWS — Entorno de desarrollo del ERP en Windows

## 1. Ficha

| Campo | Valor |
|:--|:--|
| Módulo | Entorno de desarrollo (laptop Windows del equipo) |
| Código | GUIDE-DEV-WIN |
| Versión | v1.0 |
| Estado | Vigente |
| Fecha | 2026-09-22 |
| Audiencia | Desarrollo / implementación |
| Origen | Consolidación F1 de `PLAN_DEPLOY_ESCALADO_ONPREMISE_2026-09-22`: absorbe `business_management/WINDOWS_SETUP.md`, `erp-webapp/docs/deployment/DOCKER_WINDOWS_POSTGRES_GUIDE.md` y `DOCKER_WSL_GUIDE.md` (borrados), conservando solo lo verificado en esta máquina |

> Esta guía cubre la **máquina de desarrollo** (correr el ERP para programar/probar).
> Para instalar un **cliente** usar `GUIDE_INSTALACION_NUEVO_CLIENTE.md` (instalador con
> perfiles `laptop`/`server`); si la máquina del cliente es **Windows**, el camino
> soportado es WSL2 + el instalador de siempre: `GUIDE_INSTALACION_WINDOWS.md`.
> Los dos caminos comparten PostgreSQL y Docker, pero no son lo mismo.

---

## 2. Requisitos de la máquina

- Windows 10/11 (2004+), WSL2 habilitado, Docker Desktop corriendo.
- PostgreSQL 16 (nativo o dentro de WSL — ver §6/§7).
- Node.js 20+ con **pnpm** (gestor exclusivo del frontend; nunca npm/yarn).
- Go 1.24+ (backend).
- Git.

---

## 3. Loop de desarrollo nativo (el de todos los días)

```
Backend  → :5050   (go run . / bin/business_management)
Frontend → :5173   (pnpm dev)
DB       → :5432   (PostgreSQL nativo del host)
```

1. **Base de datos** (una vez): crear usuario y base:

   ```sql
   CREATE USER dev_user WITH PASSWORD '<password_local>';
   CREATE DATABASE erp_db OWNER dev_user;
   GRANT ALL PRIVILEGES ON DATABASE erp_db TO dev_user;
   ```

2. **Esquema + migraciones**: `make db-setup` desde `business_management/` (crea schema,
   aplica migraciones con golang-migrate y estampa la versión — verificado 2026-09-21).
   Migraciones nuevas SOLO con `make migrate-create`; nunca editar una existente.

3. **Semillas RBAC canónicas**: `database/seeds/` (9 roles / 66 permisos). Sin seeds,
   el login no tiene roles a quienes colgar permisos.

4. **Usuario admin inicial**: `cmd/create_initial_admin` (no insertar usuarios a mano).

5. **Backend**: `.env` en la raíz de `business_management/` (plantilla `.env.example`;
   `PORT=:5050`). Levantar con `go run .` o el binario compilado.

6. **Frontend**: `.env` en la raíz de `erp-webapp/` con `VITE_API_URL=http://localhost:5050`
   (contrato F7.7: URL directa; el backend ya tiene CORS para `:5173`). `pnpm install` +
   `pnpm dev`.

7. **Licencias (pack BI)**: en dev, `LICENSE_ENFORCE=false` y claves de prueba en
   `~/erp-licensing-dev/`.

---

## 4. Docker Desktop + WSL2

1. Instalar Docker Desktop (instalador como administrador; reiniciar si lo pide).
2. `wsl --install` + `wsl --set-default-version 2` si WSL2 no está.
3. Docker Desktop → Settings → Resources → WSL Integration: habilitar la distro y asignar
   **≥ 4 GB** de RAM.
4. Verificar: `docker --version` y `docker compose version` (Compose v2, plugin `docker compose`;
   el binario viejo `docker-compose` ya no se usa).

---

## 5. Docker con perfil `laptop` (stack completo sin instalar Go/Node)

Un solo compose canónico en `business_management/docker-compose.yml` con perfiles
(absorbe al extinto `docker-compose.server-local.yml`):

```powershell
cd business_management
copy docker\env.docker.example .env.docker   # y editar credenciales
docker compose --profile laptop up -d --build
```

- Frontend: `http://localhost:18081` · Backend: `http://127.0.0.1:18080` (solo loopback; PostgreSQL del stack en `127.0.0.1:5433` si se levanta).
- PostgreSQL sigue siendo el nativo del host: `DB_HOST=host.docker.internal` funciona en
  Windows y en Linux (el compose añade `extra_hosts: host-gateway`).
- Redis corre interno de la red compose; **nunca** se publica al host.
- El perfil `server` (80/443 + TLS) es para instalación de clientes, no para dev.

---

## 6. PostgreSQL nativo en Windows + Docker

Receta validada en esta máquina (log 2025-10-12):

1. **Servicio**: `Get-Service postgresql-x64-16` · `net start/stop postgresql-x64-16`.
2. **`postgresql.conf`**: `listen_addresses = '*'` (o `localhost` si solo conecta el host).
3. **`pg_hba.conf`**: permitir la subred de Docker y localhost:

   ```
   # TYPE  DATABASE        USER            ADDRESS                 METHOD
   host    all             all             172.17.0.0/16           scram-sha-256
   host    all             all             127.0.0.1/32            scram-sha-256
   ```

4. **Desde contenedores**, usar `host.docker.internal` como `DB_HOST` (no `localhost`):
   dentro del contenedor, `localhost` es el propio contenedor.
5. Reiniciar el servicio tras tocar `pg_hba.conf`/`postgresql.conf`.
6. Diagnóstico rápido: `netstat -ano | findstr :5432` (¿escucha?) y
   `psql -h localhost -U dev_user -d erp_db` (¿autentica?).

---

## 7. PostgreSQL dentro de WSL

⚠️ **La IP de WSL cambia en cada reinicio** — si el backend (Windows) apunta a la IP vieja,
conecta nada. Flujo:

1. IP actual: `wsl hostname -I`.
2. Actualizar `DB_HOST` del `.env` automáticamente: `erp-webapp/scripts/update-wsl-ip.ps1`.
3. En WSL, `postgresql.conf` con `listen_addresses = '*'` y `pg_hba.conf` aceptando la
   subred de Docker (`172.17.0.0/16`) — igual que §6.
4. Probar conectividad desde Windows: `psql -h <ip_wsl> -U dev_user -d erp_db`.

---

## 8. Problemas conocidos (verificados)

| Síntoma | Causa | Solución |
|:--|:--|:--|
| Backend no conecta a PG desde Docker | `DB_HOST=localhost` dentro del contenedor | Usar `host.docker.internal` |
| `pg_hba.conf` rechaza conexión | Subred de Docker no permitida | Agregar `172.17.0.0/16` y reiniciar el servicio |
| Ayer conectaba, hoy no (WSL) | IP de WSL cambió al reiniciar | `update-wsl-ip.ps1` + revisar `DB_HOST` |
| "port already in use" al levantar compose | Otro proceso en 18080/18081 | `netstat -ano | findstr :18081` → `taskkill /PID <PID> /F`, o cambiar `ERP_BACKEND_PORT`/`ERP_FRONTEND_PORT` |
| Frontend levanta pero sin datos | `VITE_API_URL` horneada al build no apunta al backend | En dev: `http://localhost:5050`; en Docker laptop, la imagen se construye con `.env.production` — reconstruir con `--build` si cambió |

---

## 9. Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v1.2 | 2026-10-10 | Nota de la ficha: instalación de cliente sobre Windows → GUIDE_INSTALACION_WINDOWS (WSL2 + instalador) | renpereiradx + agente |
| v1.1 | 2026-10-06 | Puertos del compose laptop actualizados a los defaults nuevos: frontend `18081`, backend loopback `18080`, PG del stack `5433` (BE `9e998e8`) | renpereiradx + agente |
| v1.0 | 2026-09-22 | Consolidación desde 3 documentos dispersos (F1 del plan de deploy); borrados los originales | renpereiradx + agente |
