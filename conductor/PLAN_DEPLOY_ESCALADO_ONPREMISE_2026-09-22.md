# PLAN_DEPLOY_ESCALADO_ONPREMISE — Distribución y escalado on-premise (laptop → servidor → sucursales)

| Campo | Valor |
|:--|:--|
| Estado | ✅ Aprobado (F0 decidido 2026-09-22) · ⏳ **NO INICIADO** |
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

- ⏳ T1: Restaurar proxy `/api` en `nginx.conf` de producción con TLS (443) + redirect 80→443; `VITE_API_URL=/api` en `.env.production` — **mismo diff** (regla F7.7).
- ⏳ T2: Endpoint `GET /version` en el BE: versión SemVer inyectada por build (variable `APP_VERSION`/ldflags), **sin autenticación** (patrón `/health`). Actualizar healthcheck/docs.
- ⏳ T3: CA interna: generación por instalador, instalación del certificado server en nginx, y script/directiva para terminales (guía en F5).
- ⏳ T4: Puertos perfil server 80/443 (D6) + endurecimiento: PG solo local, Redis interno (viene de F1-T2), SSH por firewall/VPN documentado.
- Gates: BE `go test ./...` + `golangci-lint run` + `make build`; FE `npx vitest --run` + `npx tsc --noEmit` + `pnpm build` + `pnpm lint:design`; smoke manual del proxy en local.

### F3 — Instalador y migración

- ⏳ T1: `install.sh` con `--profile laptop|server`: pre-checks (SO, Docker, PG, disco, puertos), estructura `/opt/erp/`, `.env` desde template (secrets generados), imágenes (pull o `docker load` offline), migraciones vía **contenedor efímero** (`docker compose run --rm migrate` con golang-migrate), seeds RBAC + `cmd/create_initial_admin`, colocación y verificación de licencia (`licensegen` documentado), smoke test final, backup systemd-timer diario con retención 30 días. Idempotente.
- ⏳ T2: `migrate-to-server.sh`: backup etiquetado en laptop → install profile server → restore → migraciones → smoke → checklist de baja de la laptop. Rollback = restaurar backup + imágenes anteriores.
- ⏳ T3: Bundle de release: script que empaqueta imágenes (`docker save`) + compose + migraciones + instalador + manifiesto con checksums (firma: reutilizar ed25519 del licenciamiento si encaja, sino al menos checksums).
- Gates: instalación limpia ×2 seguidas (idempotencia) en VM efímera; migración E2E con datos de prueba; smoke verde.

### F4 — Sucursales (VPN)

- ⏳ T1: Procedimiento Tailscale: instalación en servidor, claves pre-aprobadas, alta de máquinas de sucursal; decidir titularidad de tailnet (ver F0) y documentar.
- ⏳ T2: `add-branch` (script + runbook): crear sucursal como entidad en el ERP, usuarios/roles de sucursal, verificar scoping (ventas/transferencias de stock entre locales ya existen).
- ⏳ T3: VPN de soporte del vendor (Tailscale/WireGuard) con consentimiento del cliente, para lectura de `/version` y soporte remoto — alimenta el inventario de instalaciones.
- Gates: prueba real con 2 redes separadas (o 2 VM): escaneo de puertos públicos del servidor sin servicios expuestos; usuario de sucursal opera en scoping correcto.

### F5 — Documentación (docs/)

- ✅ T1 (2026-09-22, junto a este plan, por pedido del dueño): reorg `docs/` en `requirements/` + `guides/`, README actualizado con categoría de guías + plantilla, REQ_DEPLOY Borrador v0.1, índice de guías planificadas.
- ⏳ T2: Escribir las 4 guías **contra la implementación real** de F2–F4 (los nombres viven en `docs/guides/README.md`): instalación nuevo cliente, terminales, migración laptop→servidor, sucursal nueva.
- ⏳ T3: REQ_DEPLOY: Borrador → En revisión → Aprobado conforme F2–F4 aterricen; verificaciones de la tabla §6 del REQ ejecutadas y marcadas.
- ⏳ T4: Iniciar backfill REQ de módulos existentes (prioridad: ventas, compras, inventario, caja) — incremental, un módulo por vez, sin big-bang.

### F6 — Validación E2E en VMs efímeras

- ⏳ T1: Extender la receta de instancia efímera (`:5433`) a un ensayo completo de producto: VM1 instala perfil `laptop` con datos de prueba → `migrate-to-server` hacia VM2 (perfil `server`, TLS, CA interna) → terminal desde VM3 (navegador/curl, solo URL) → opcional VM4 como sucursal vía Tailscale.
- ⏳ T2: Checklist de cierre: RF/RNF del REQ verificados uno a uno (incluye idempotencia, backup restaurable, scoping de sucursal, `docker compose config` sin puertos de Redis, imagen FE única para dos instalaciones).
- Gates: checklist completo en verde + guías actualizadas con lo aprendido.

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
