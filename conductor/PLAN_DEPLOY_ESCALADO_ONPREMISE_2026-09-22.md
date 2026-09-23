# PLAN_DEPLOY_ESCALADO_ONPREMISE — Distribución y escalado on-premise (laptop → servidor → sucursales)

| Campo | Valor |
|:--|:--|
| Estado | ✅ **EJECUTADO COMPLETO F1–F6 (2026-09-22)** · listo para purga |
| Fecha | 2026-09-22 |
| REQ asociado | `docs/requirements/REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md` (Borrador v0.1) |
| Repos afectados | wrapper (docs/, conductor/), business_management (compose, Dockerfile, nginx, /version, scripts), erp-webapp (nginx.conf, .env.production, VITE_API_URL) |
| Fuente | Evaluación de sesión 2026-09-22 sobre apuntes del dueño (2026-09-20) |

---

## 1. Contexto y hallazgos verificados (2026-09-22)

El producto se vende por separado (on-premise, guaraníes). La instalación hoy es artesanal y
mono-máquina. Este plan convierte la instalación en un proceso repetible con dos perfiles
(`laptop`, `server`) y tres procedimientos (terminal nueva, migración laptop→servidor,
sucursal nueva). Hallazgos que motivan las fases:

- `business_management/docker-compose.yml` construye el frontend desde `../../web-project/erp-webapp` — **ruta inexistente** (hoy es `../erp-webapp`). El compose está roto y no se notó porque el dev loop no lo usa.
- El encabezado del mismo compose documenta "API via Nginx proxy :8081/api", pero `erp-webapp/nginx.conf` **eliminó** ese proxy en F7.7 — el comentario miente.
- `docker-compose.server-local.yml` **publica Redis (6379) al host**; el compose principal lo mantiene interno. Drift y peor seguridad.
- `VITE_API_URL` se hornea al compilar el frontend (`erp-webapp/.env.example`: `http://localhost:5050`): **el escenario multi-máquina no funciona hoy** (el navegador de una terminal llamaría a su propio localhost).
- La licencia **no está amarrada a hardware** (`internal/licensing/domain.go`: edición+cliente+módulos+vencimiento) → portátil entre máquinas (congelar, RF-DEPLOY-015).
- CORS ya es por entorno (`main.go:271`), `/health` existe y está exento de auth (`internal/platform/health/routes.go:22`, `middleware/auth.go:29`); `/version` **no existe**.
- Material de deploy disperso y con drift: BE raíz (`WINDOWS_SETUP.md`, `DOCKER_WINDOWS_GUIDE.md`, `DOCKER_SETUP_COMPLETE.md`, `apply_migration_quick.ps1`, `verify-postgres-windows.ps1`), BE `docs/` (`DEPLOYMENT_DOCKER.md`, `docs/deployment/`, `docs/guides/`), FE (`DOCKER_BACKEND_PATH_GUIDE.md`, `scripts/docker-compose-*.ps1`), 2 compose + 4 Dockerfiles.
- El software ya es multi-sucursal como dato (ventas por sucursal, transferencias, RBAC 9 roles): la sucursal es un problema de red, no de producto.
- Ya existen `cmd/create_initial_admin`, `cmd/licensegen`, seeds RBAC canónicos (`database/seeds/`) y `make db-setup` verificado — piezas reutilizables del instalador.

## 2. Decisiones F0 (aprobadas por el dueño, 2026-09-22)

| # | Decisión | Resolución | Queda como |
|:--|:--|:--|:--|
| D1 | Contrato HTTP del perfil servidor | Restaurar **proxy inverso `/api` de origen único** en nginx de producción. Regla F7.7 de AGENTS.md: `VITE_API_URL=/api` + rewrite en `vite.config.js` + `nginx.conf` **en el mismo diff** | RN-DEPLOY-001 |
| D2 | TLS | **CA interna auto-firmada** generada y distribuida por el instalador; alternativa Tailscale-cert si ya hay VPN | RN-DEPLOY-002 |
| D3 | VPN sucursales | **Tailscale** por defecto (claves pre-aprobadas, atraviesa CGNAT); WireGuard puro como alternativa aceptada | RN-DEPLOY-003 |
| D4 | Modo offline sucursales | **Fuera de alcance v1**; limitación documentada en guía y REQ | RN-DEPLOY-004 |
| D5 | Organización de docs | `docs/requirements/` + `docs/guides/` como categorías permanentes; backfill REQ incremental, sin árbol paralelo de módulos | RN-DEPLOY-005 |
| D6 | Puertos perfil servidor | Estándar **80/443** (dejar 8080/8081 como herencia dev fuera del perfil server) | RN-DEPLOY-006 |

Punto operativo abierto dentro de F4 (no bloqueante): titularidad de la tailnet por cliente
(recomendado: una tailnet por cliente, aislamiento total) — se decide al ejecutar F4 y queda
documentado en la guía.

## 3. Fases

> Convención: cada fase cierra con sus gates en verde y commit(s) con conventional commits.
> **Ninguna fase fue iniciada** (salvo la estructura de docs marcada ✅ más abajo, creada por
> pedido explícito junto a este plan).

### F1 — Higiene de deploy (sin cambio de comportamiento)

- ✅ T1 (2026-09-22): `docker-compose.yml` — context `../erp-webapp`, encabezado reescrito sin el proxy mentiroso; compose canónico con perfiles `laptop`/`server` (T2 absorbió esto).
- ✅ T2 (2026-09-22): compose canónico único con perfiles Compose; `docker-compose.server-local.yml` eliminado (Redis ya interno en el principal; la variante moría). Backend publicado solo en loopback `127.0.0.1:8080` en ambos perfiles. Plantilla `docker/env.docker.example` commiteada. Makefile: `deploy-local`→`--profile laptop`, `deploy-server`→`--profile server`, `docker-down` sin referencia al compose muerto.
- ✅ T3 (2026-09-22): consolidación — borrados en BE: `DOCKER_WINDOWS_GUIDE.md`, `DOCKER_SETUP_COMPLETE.md`, `apply_migration_quick.ps1`, `verify-postgres-windows.ps1`, `docs/deployment/deployment-strategy.md`, `WINDOWS_SETUP.md`. Borrados en FE: `DOCKER_BACKEND_PATH_GUIDE.md`, `scripts/docker-compose-{up,down}.ps1`, `docs/deployment/` completa (16 archivos, entre ellos un clon de WINDOWS_SETUP con contraseña real hardcodeada). `docs/DEPLOYMENT_DOCKER.md` (BE) se conservó y se reescribió contra la realidad (matriz Cloud Run + perfiles). Material vigente de Windows consolidado en `docs/guides/GUIDE_ENTORNO_DEV_WINDOWS.md` del wrapper (PG nativo pg_hba/WSL/host.docker.internal + loop dev real). Referencias colgantes corregidas en 3 READMEs. Se conservaron los scripts FE `docker-{build,clean,logs,run,stop}.ps1` (funcionan contra el contenedor standalone).
- ✅ T4 (2026-09-22): `default.tar.gz` (13 MB, estado de herramienta agentica, no del ERP) movido a `~/erp-releases/`.
- Gates: BE `make build` ✅ · FE `pnpm build` ✅ · `docker compose config` ✅ (laptop, server y default).

### F2 — Perfil servidor (contrato HTTP + TLS + observabilidad)

- ✅ T1 (2026-09-22): proxy `/api` de origen único restaurado en FE. Nueva estructura `erp-webapp/nginx/`: `templates/default.conf.template` (HTTP, upstream inyectado por envsubst vía `ERP_API_UPSTREAM`, default `http://backend:8080`), `snippets/erp-app.conf` (SPA compartida, sin duplicación), `tls.conf` (443 ssl + redirect 80→443 + headers de endurecimiento). El compose del perfil server monta `tls.conf` sobre `conf.d/default.conf` (una sola imagen FE para ambos perfiles). `.env.production` → `VITE_API_URL=/api` y `vite.config.js` con proxy dev `/api→:5050` restaurado (dormant) — **mismo diff** (regla F7.7). Scripts `docker-build.ps1`/`docker-run.ps1` actualizados al nuevo layout.
- ✅ T2 (2026-09-22): `GET /version` sin auth (patrón `/health`): paquete nuevo `internal/platform/version` (Version/Commit/BuildDate por ldflags) + handler en `internal/platform/health` + whitelist `NO_AUTH_NEEDED` + tests (`routes_test.go`). `APP_VERSION` via `git describe` en Makefile (`make build` imprimió `phase8-testing-integral-501-gd4c1f66-dirty`) y `ARG APP_VERSION` en Dockerfile (el `-X main.Version` viejo era un no-op: el símbolo no existía). Smoke real devolvió `{"version":"smoke-f2",...}`.
- ✅ T3 (2026-09-22): CA interna — `deploy/scripts/gen-ca.sh` (CA 10 años + cert servidor 3 años, SANs automáticos: hostname + localhost + todas las IPs del host incluida Tailscale; `--force-server` reemite sin tocar la CA; probado: `openssl verify` OK) + `deploy/scripts/install-ca-windows.ps1` (LocalMachine\Root) + `deploy/README.md` con flujo de certs y reglas fijas.
- ✅ T4 (2026-09-22): puertos server 80/443 (overridables `ERP_HTTP_PORT`/`ERP_HTTPS_PORT`); endurecimiento verificado en el smoke: FE publicado 0.0.0.0 (única superficie), backend **loopback-only** (`curl` desde la IP de Tailscale a :8080 → inalcanzable), Redis sin publicación. PG local-only y SSH quedan documentados en `deploy/README.md` (guía completa en F5).
- Hallazgo del smoke: los 4 Dockerfiles BE usaban base `golang:1.25-alpine` contra `go.mod` ≥ 1.26.1 — cualquier build por imagen moría en `go mod download`. Bump a `golang:1.26-alpine` en los 4.
- Gates: BE `go test ./...` ✅ + `golangci-lint run` ✅ + `make build` ✅ · FE `npx vitest --run` 1104/1104 ✅ + `npx tsc --noEmit` ✅ + `pnpm build` ✅ + `pnpm lint:design` ✅ · smoke manual del proxy en local ✅ (imagen FE+BE, PG efímero :5433, perfil server up: `/api/version` con CA propia y con `-k`, `/api/health`, redirect 80→443, SPA fallback `/pedidos`, assets 200).

### F3 — Instalador y migración

- ✅ T1 (2026-09-22): `deploy/scripts/install.sh` — orquestador delgado e idempotente con `--profile laptop|server`: pre-checks (Linux, Docker, rsync, disco ≥5 GB, puertos libres), estructura `ERP_ROOT` (default `/opt/erp`; override de prueba), `.env.docker` con secrets generados (JWT + PG password; se conservan en re-runs), imágenes `docker load` (offline) o build, migraciones vía contenedor efímero `migrate` (servicio compose con perfil propio — `up --wait` jamás lo arranca), seeds RBAC solo si `users.roles` vacía, `create_initial_admin` no-interactivo (ADMIN_* env, nuevo) con credenciales en `credentials/` (600), licencia opcional con `LICENSE_ENFORCE` automático, smoke final (backend health+version, frontend HTTPS+proxy+redirect o laptop HTTP), backup diario `$ERP_ROOT/backup.sh` + timer systemd con retención 30 días.
- ✅ T2 (2026-09-22): `deploy/scripts/migrate-to-server.sh` — backup etiquetado de la laptop (`pg_dump --clean --if-exists` + sha256 + guard anti-dump-vacío) → bundle → rsync/ssh (`--remote`) o manual → `install.sh --profile server --restore` → checklist de baja de la laptop (roll-back documentado: la laptop no se toca hasta 30 días post-verificación).
- ✅ T3 (2026-09-22): `deploy/scripts/make-release-bundle.sh` — imágenes (`docker save|gzip`, `APP_VERSION` inyectada) + compose + plantilla env + migraciones + **baseline de schema** + seeds + instalador + nginx FE + `SHA256SUMS` (firma opcional `RELEASE_SIGNING_KEY` ed25519 dedicada; por defecto checksums, como el plan permite).
- **Hallazgo mayor del gate E2E**: la cadena de migraciones NO es corible desde cero (000003 hace `ALTER TABLE products.products` que ninguna migración crea — el dev DB nació antes de golang-migrate; el memory de db-setup ya lo insinuaba). Solución: **baseline de schema** (`database/baseline/baseline_schema.sql`, DDL completo sin datos, regenerable con `make db-baseline`) + `migrate force <última>` en instalaciones frescas; en DBs con datos, `migrate up` normal (delta). El guard del instalador detecta bundles sin baseline.
- **Cambio de arquitectura validado en E2E**: PG nativo en loopback + contenedores es IMPOSIBLE en Linux (host-gateway = bridge 172.17.x, no llega al 127.0.0.1 del host; el smoke de F2 no lo detectó porque `/health` no toca DB). El compose canónico ahora incluye **PostgreSQL como servicio** (volumen `erp_pg_data`, publicado solo en loopback `ERP_DB_PORT`); el path nativo queda documentado como override (Windows/Docker Desktop). Backup vía `docker compose exec postgres pg_dump`.
- Gates: **instalación limpia ×2 en el mismo destino (idempotencia)** ✅ (secrets/certs/seeds/admin conservados; segunda corrida EXIT 0) · **migración E2E con datos reales** ✅ (dump 2.5MB de la dev DB → instalación desde bundle offline (`docker load`) con `--restore` → `users=7932` verificado en ambos lados, 10 schemas restaurados) · smoke verde ✅ (incl. **login real** con el admin generado vía proxy TLS: JWT con rol `F2VLso`).

### F4 — Sucursales (VPN)

- ✅ T1 (2026-09-22): decisión F0 resuelta y documentada en `deploy/scripts/support-vpn.sh`: **una tailnet por cliente** (aislamiento total; el vendor entra a la tailnet del cliente con clave pre-aprobada de expiración corta). El procedimiento Tailscale de sucursales vive en `GUIDE_SUCURSAL_NUEVA` (F5-T2). NOTA honesta: el join real de 2 redes Tailscale requiere estar en el sitio del cliente; lo ejecutable acá quedó verificado (escaneo de puertos).
- ✅ T2 (2026-09-22): `deploy/scripts/add-branch.sh` — verifica alcance (el check de VPN), crea la entidad vía API (`branch_type` whitelisted), otorga accesos de usuarios (`/branches/{id}/access`), verifica `/version`. E2E real contra el stack server: sucursal SUC2 creada (id=4), branchadmin con acceso FULL (fila `user_branch_access` verificada), login OK con rol admin. El scoping de datos por sucursal es el sistema preexistente auditado (vendor-role-branch 2026-09-06/09-10) — se verificó su llama (acceso+login+entidad), no se re-auditó su interior.
- ✅ T3 (2026-09-22): `deploy/scripts/support-vpn.sh` con `--consent` obligatorio, `--revoke`, inventario en `$ERP_ROOT/support-vpn-info.txt` (alimenta el inventario de instalaciones). Requiere Tailscale real instalado en sitio.
- Gates: escaneo de puertos desde 2 redes (LAN 192.168.100.x + Tailscale 100.x): **solo los puertos del frontend publicados** (backend/PG loopback, Redis interno) ✅; usuario de sucursal opera con scoping correcto ✅ (a nivel entidad/acceso/login; el interior del scoping ya estaba auditado). Hallazgo lateral: el backend DEV nativo de esta máquina escucha en `*:5050` (no es del stack productivo; igualmente conviene bindearlo a loopback en dev).

### F5 — Documentación (docs/)

- ✅ T1 (2026-09-22, junto a este plan, por pedido del dueño): reorg `docs/` en `requirements/` + `guides/`, README actualizado con categoría de guías + plantilla, REQ_DEPLOY Borrador v0.1, índice de guías planificadas.
- ✅ T2 (2026-09-22): las 4 guías escritas contra la implementación real de F2–F4: `GUIDE_INSTALACION_NUEVO_CLIENTE.md` (bundle, perfiles, 12 pasos del instalador, actualización de cliente, overrides), `GUIDE_TERMINALES_LOCAL.md` (CA interna con install-ca-windows.ps1, Firefox, checklist por terminal), `GUIDE_MIGRACION_LAPTOP_A_SERVIDOR.md` (migrate-to-server.sh, verificación, baja a 30 días, rollback), `GUIDE_SUCURSAL_NUEVA.md` (tailnet por cliente, add-branch.sh, soporte vendor, limitación offline).
- ✅ T3 (2026-09-22): REQ_DEPLOY → **v0.2 Implementado** con §6/§7 verificadas una a una con resultados reales del ensayo (13/15 RF ✅ verificadas, 2 ⚠️ parciales de sitio/tiempo: join Tailscale real, retención de 31 días observada; RNF-006 SSH = en sitio).
- ✅ T4 (2026-09-22): backfill iniciado con **REQ_VENTAS v0.1 Borrador** (POS, pedidos de mostrador con TTL, crédito FIFO, anulaciones con aprobación, scoping, rol vendedor; 12 RF + 5 RNF + 6 RN). Siguientes módulos (compras, inventario, caja) quedan como cola incremental.


### F6 — Validación E2E en VMs efímeras

- ✅ T1 (2026-09-22): ensayo completo ejecutado contra stacks efímeros (PG contenedorizado en puertos altos + ERP_ROOT temporal, sin tocar dev ni /opt): **instalación laptop** (perfil laptop, todo loopback, login admin OK) → **migración** (migrate-to-server.sh: dump etiquetado 2.5 MB → instalación **offline desde bundle** con --restore → 7.932 users y 10 schemas verificados en ambos lados) → **server** (TLS+CA+proxy /api) → **terminal simulada** (curl/login desde la IP de LAN, sin nada instalado) → **sucursal** (add-branch E2E + escaneo de puertos desde 2 redes). Tailscale real queda para el sitio del cliente (documentado honestamente).
- ✅ T2 (2026-09-22): checklist RF/RNF uno a uno — vive en el REQ §6/§7 (fuente permanente, RN-DEPLOY-005): 13/15 RF ✅, 2 ⚠️ parciales (Tailscale-sitio, retención 31 días); RNF: 5/7 ✅, SSH en sitio, 30-min a cronometrar en el primer cliente. Incluye idempotencia ×2 (secrets/certs/seeds/admin conservados), backup restaurable (gzip íntegro + restore E2E), compose config sin puertos de Redis, imagen FE única servida en dos perfiles.
- Gates: checklist en verde con reservas marcadas ✅ + guías actualizadas con lo aprendido ✅.

---

## 7. Hallazgos del ensayo (lecciones incorporadas)

1. **La cadena de migraciones nunca fue corible desde cero** (000003 asume tablas que ninguna migración crea): resuelto con `database/baseline/` + `migrate force`; `make db-baseline` para regenerar. Sin esto, ningún cliente nuevo podía instalarse.
2. **PG nativo en loopback + contenedores es imposible en Linux** (host-gateway = IP de bridge, no alcanza el 127.0.0.1 del host): PG pasó a servicio compose con volumen propio y publicación loopback-only; el smoke de F2 no lo detectó porque `/health` no toca la DB.
3. **El flujo offline ignoraba las imágenes del bundle** (nombres distintos → intentaba build sin source Go): tags pineados vía `APP_BE_IMAGE_TAG`/`APP_FE_IMAGE_TAG`.
4. **Backup "exitoso" de 20 bytes**: pg_dump fallaba dentro del contenedor y sin pipefail el gzip dejaba un dump vacío con exit 0. Guard de tamaño + pipefail dentro del contenedor.
5. Dockerfiles BE con base golang:1.25 contra go.mod >= 1.26.1: cualquier build por imagen moría; bump a 1.26 en los 4.
6. Lateral (fuera del stack productivo): el backend DEV nativo de la estación de trabajo escucha en *:5050 — conviene bindearlo a loopback.


## 4. Fuera de alcance (no proponer dentro de este plan)

Modo offline de sucursales (D4) · SaaS multi-tenant · Kubernetes/orquestadores · exponer
servidores de clientes a internet pública sin VPN · servidores Windows como perfil `server`.

## 5. Riesgos y mitigaciones

| Riesgo | Mitigación |
|:--|:--|
| Certificados auto-firmados generan desconfianza/advertencia | Distribución automatizada de la CA por el instalador + explicación en la guía de terminales |
| CGNAT del ISP complica la conectividad de sucursal | Tailscale (D3) atraviesa CGNAT; WireGuard solo como alternativa con port-forward |
| El instalador crece y se vuelve inmantenible | Mantenerlo como orquestador delgado que reusa piezas existentes (`db-setup`, seeds, `create_initial_admin`, `licensegen`) |
| Drift futuro entre guías y realidad | Regla: el procedimiento cambia → la guía cambia en el mismo diff (docs/README, reglas de guías #5) |
| Robo/pérdida de laptop con operación completa | Perfil laptop exige cifrado de disco + backup externo (guía de instalación, F5-T2) |

## 6. Criterios de cierre del plan

1. F6 con checklist completo en verde.
2. Las 4 guías publicadas en `docs/guides/` y probadas contra el ensayo E2E.
3. REQ_DEPLOY en estado Implementado (verificaciones de RF/RNF ejecutadas).
4. Purga de este plan de `conductor/` (convención: el plan se borra al cerrarse); el REQ y las guías quedan como fuente permanente.
