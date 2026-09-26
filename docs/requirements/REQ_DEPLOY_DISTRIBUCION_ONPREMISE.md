# REQ_DEPLOY — Distribución y despliegue on-premise (perfiles de instalación y crecimiento)

## 1. Ficha

| Campo | Valor |
|:--|:--|
| Módulo | Distribución, instalación y escalado del producto en clientes (on-premise) |
| Código | DEPLOY |
| Versión | v0.3 |
| Estado | Implementado (reservas de sitio: join Tailscale real, endurecimiento SSH y prueba de 31 días de retención se ejecutan en el primer cliente) |
| Fecha | 2026-09-22 |
| Fuentes | Apuntes del dueño (2026-09-20, cuaderno: "Distribución y mantenimiento de Software"), evaluación técnica de sesión 2026-09-22 (auditoría de compose/nginx/CORS/licencias/modelo de sucursales de ambos repos) |
| Implementación | ✅ Implementado (F1–F4 del plan ejecutado el mismo día 2026-09-22): BE commits bd2751b→51438f2 (compose canónico con perfiles, `/version`, CA interna, instalador, bundle, migración, add-branch, VPN soporte), FE 0fdc04c/312ec6d (docs consolidation + proxy `/api` TLS), wrapper 25793c2+ (docs/guides). Plan: `conductor/PLAN_DEPLOY_ESCALADO_ONPREMISE_2026-09-22.md` |

## 2. Resumen ejecutivo (para todos)

El ERP se vende por separado a cada cliente (licencia + instalación propia, modelo on-premise, precio en guaraníes). Hoy la instalación es artesanal y está pensada para una sola máquina. Este requerimiento define cómo debe **instalarse, crecer y operarse** una instalación del producto a lo largo de la vida del cliente: empieza en una laptop, puede terminar en un servidor dedicado con varias terminales en el local y una o más sucursales conectadas por internet.

La pieza central es el **bundle de release**: un paquete cerrado y versionado (imágenes Docker + compose + migraciones + instalador) que sirve tanto para instalar un cliente nuevo como para actualizar uno existente. Sobre ese bundle se definen dos **perfiles de instalación** — `laptop` y `server` — y tres **procedimientos guiados**: agregar una terminal, migrar de laptop a servidor, y conectar una sucursal nueva.

El objetivo de negocio es que vender, instalar y hacer crecer una instalación sea un trámite repetible y seguro (con backup y rollback), no un proyecto artesanal por cliente.

## 3. Cómo funciona — guía no técnica

Imaginá el producto como un negocio con tres etapas de crecimiento:

**Etapa 1 — Todo en una laptop (perfil `laptop`).** El cliente arranca con una sola máquina: ahí viven la base de datos, el sistema y el navegador con el que se usa. Es como un comercio chico que atiende desde un solo mostrador. El riesgo no es técnico sino físico: si la laptop se pierde o se roba, se pierde el negocio. Por eso el perfil laptop exige dos cuidados que el instalador configura solo: el disco va **cifrado** (si se roban la máquina, los datos son ilegibles) y cada noche se hace una **copia de seguridad** fuera de la máquina (en la nube o en otro disco).

**Etapa 2 — Un servidor y varias terminales (perfil `server`).** El negocio crece y quiere cajas y vendedores trabajando a la vez. Se pasa todo a un servidor dedicado y cada persona usa su propia máquina **solo con un navegador y una dirección** (ej. `https://erp.local`). No se instala nada en las terminales — igual que entrás a cualquier sitio web. La única pieza técnica importante: el servidor entrega la web y las llamadas al sistema **por una sola puerta con candado** (un único origen con cifrado TLS), para que contraseñas y sesiones no viajen en claro por la red del local. Migrar de la laptop al servidor es un trámite guiado: se respalda, se instala en el servidor, se restaura, se verifica, y recién ahí se apaga la laptop. La licencia **no** hay que re-comprarla ni re-emitirse: viaja con el sistema (no está amarrada a la máquina).

**Etapa 3 — Una sucursal más.** El cliente abre otro local. Lo primero que hay que saber: **el software ya es multi-sucursal** (ventas por sucursal, transferencias de stock entre locales, usuarios y roles propios). Lo único nuevo es la conexión: las máquinas de la sucursal llegan al servidor central a través de un **túnel privado por internet (VPN)** — nadie expone el servidor al público. Para el empleado de la sucursal es transparente: abre el navegador y el sistema funciona como si estuviera en el local principal. Limitación honesta que se le informa al cliente: **si se corta el internet, la sucursal queda sin sistema** hasta que vuelva (ver RN-DEPLOY-004).

Para el dueño del producto (vos), cada instalación queda registrada en un **inventario** (qué cliente, qué versión, fecha) y el sistema expone su versión en una dirección interna (`/version`) para poder auditarla sin preguntarle a nadie.

## 4. Alcance

**DENTRO:**

- Bundle de release versionado (imágenes + compose + migraciones + instalador + manifiesto).
- Dos perfiles de instalación: `laptop` (todo local, sin exposición de red) y `server` (servidor dedicado con TLS y firewall).
- Procedimiento guiado de migración laptop → servidor con backup previo obligatorio y verificación posterior.
- Terminales operando sin instalación (navegador + URL).
- Conexión de sucursales vía VPN, con alta de la sucursal como entidad en el ERP (ya existente) y usuarios/roles propios.
- Instalador idempotente (re-ejecutable sin romper) con smoke test automático al terminar.
- Backup automático diario en ambos perfiles, con retención definida.
- Endpoint operativo `/version` público (sin autenticación, como `/health`).
- Guías operativas (runbooks) por escenario en `docs/guides/`.
- Portabilidad de la licencia entre máquinas (congelar el comportamiento actual: sin amarra a hardware).

**FUERA:**

- Modo offline para sucursales (operar sin internet y sincronizar después) — ver RN-DEPLOY-004.
- Reescritura a SaaS multi-tenant (un solo servidor central para todos los clientes).
- Orquestadores tipo Kubernetes; el stack sigue siendo Docker Compose de un nodo.
- Exponer el servidor de un cliente directamente a internet pública sin VPN.
- Servidores Windows como perfil `server` (el perfil servidor requiere Linux de 64 bits; la laptop admite cualquier SO con Docker).
- Dominio público y certificado de internet obligatorios (se usa CA interna; ver RN-DEPLOY-002).

## 5. Actores

| Actor | Rol en este módulo |
|:--|:--|
| Dueño del producto (vendor) | Genera releases, instala/migra/actualiza instalaciones, mantiene el inventario de clientes y versiones |
| Cliente (dueño del negocio) | Decide cuándo actualizar, provee el hardware y el internet, recibe las guías |
| Empleado de local (cajero/vendedor) | Usa una terminal: navegador + URL + usuario propio |
| Empleado de sucursal | Ídem, conectado vía VPN |
| Soporte técnico | Usa las guías y el túnel de soporte para operar de forma remota |

## 6. Requerimientos funcionales (RF-…)

| ID | Requerimiento | Prioridad | Estado | Verificación |
|:--|:--|:--|:--|:--|
| RF-DEPLOY-001 | El sistema debe instalarse desde un único bundle de release versionado (SemVer) que contiene imágenes, compose, migraciones e instalador. | Obligatorio | ✅ Verificada | Instalación offline desde bundle (docker load, sin build) EXIT 0 en el ensayo; `make-release-bundle.sh` exige baseline y emite SHA256SUMS |
| RF-DEPLOY-002 | El instalador debe admitir los perfiles `laptop` y `server` seleccionables por parámetro. | Obligatorio | ✅ Verificada | Ambos perfiles instalados E2E: server (TLS 443 + proxy /api) y laptop (todo loopback) |
| RF-DEPLOY-003 | El perfil `laptop` debe dejar todos los servicios escuchando únicamente en `localhost` (ninguna exposición a la red). | Obligatorio | ✅ Verificada | Instalación laptop: FE/BE/PG en 127.0.0.1; `echo > /dev/tcp/<lan-ip>/8081` inalcanzable desde otra máquina |
| RF-DEPLOY-004 | El perfil `server` debe publicar la web y la API bajo un único origen con proxy inverso `/api`. | Obligatorio | ✅ Verificada | Smoke: `/api/version` y SPA bajo el mismo origen TLS; `VITE_API_URL=/api` horneada; sin CORS en el camino del navegador |
| RF-DEPLOY-005 | Una terminal nueva debe poder operar el sistema usando exclusivamente un navegador y la URL del servidor. | Obligatorio | ✅ Verificada | Curl/login desde IP de LAN (simulación de terminal) sin nada instalado; guía GUIDE_TERMINALES_LOCAL cubre el navegador + CA |
| RF-DEPLOY-006 | El sistema debe ofrecer un procedimiento guiado de migración de laptop a servidor que preserva los datos. | Obligatorio | ✅ Verificada | E2E: dump 2.5 MB del dev DB → `install --restore` → `users=7932` y 10 schemas idénticos a origen |
| RF-DEPLOY-007 | El procedimiento de migración debe exigir un backup previo automatizado y validar la instalación destino con smoke tests antes de dar por terminada la migración. | Obligatorio | ✅ Verificada | `migrate-to-server.sh` genera el backup como paso 1 (con guard anti-dump-vacío — hallazgo del ensayo); `install.sh` muere visible si el smoke falla (observado en corridas fallidas) |
| RF-DEPLOY-008 | Una sucursal debe poder operar el sistema conectándose al servidor central por VPN sin que ningún puerto del servidor quede expuesto a internet pública. | Obligatorio | ⚠️ Parcial (sitio) | Escaneo desde LAN + IP Tailscale: solo puertos del frontend publicados ✅; el join real Tailscale requiere ejecutarse en el cliente (guía §3) |
| RF-DEPLOY-009 | El alta de una sucursal debe incluir su creación como entidad en el ERP, sus usuarios y roles, y la verificación de que opera en su sucursal y no ve ventas ajenas. | Obligatorio | ✅ Verificada (entidad+acceso+login; scoping interno pre-auditado 2026-09) | `add-branch.sh` E2E: sucursal SUC2 creada, branchadmin con acceso FULL (fila en `user_branch_access`), login OK |
| RF-DEPLOY-010 | El backend debe exponer la versión instalada en `GET /version` sin autenticación. | Obligatorio | ✅ Verificada | Ruta fijada: backend `GET /version` (whitelist auth) y `https://<servidor>/api/version` vía proxy; devuelve `{"version","commit","build_date","go"}` con ldflags |
| RF-DEPLOY-011 | El instalador debe ser idempotente: re-ejecutarlo sobre una instalación existente no corrompe datos ni configuración. | Obligatorio | ✅ Verificada | Segunda corrida EXIT 0 conservando secrets/certs/seed/admin (y una tercera con restore); login del admin original sigue funcionando. Auditoría 2026-09-22: el rsync de re-instalación borraba `license/` (el ensayo corrió sin licencia y no lo vio); corregido con exclude — licencia y CA sobreviven re-runs |
| RF-DEPLOY-012 | La instalación nueva debe crear el esquema (migraciones), los roles base (seeds RBAC), la sucursal inicial y el usuario administrador inicial sin pasos manuales en la base de datos. | Obligatorio | ✅ Verificada | Instalación limpia: baseline + `migrate force` + seeds (9 roles) + sucursal MAIN (seed 002, agregado 2026-09-26: el baseline salta la migración que la sembraba y el primer login quedaba trabado sin sucursal — hallazgo de VM real) + admin con acceso FULL a MAIN; login real OK (hallazgo: la cadena de migraciones nunca fue corrible desde cero — resuelto con baseline, `make db-baseline`) |
| RF-DEPLOY-013 | Cada instalación debe realizar un backup automático diario de la base de datos con retención 30 días. | Obligatorio | ⚠️ Parcial (tiempo) | `backup.sh` probado (dump gzip íntegro con checksum); timer systemd lo instala el instalador; la retención `-mtime +30` es trivialmente correcta pero los 31 días reales se observan en operación |
| RF-DEPLOY-014 | El instalador debe ejecutar un smoke test automático al finalizar (health, versión, login admin) y reportar resultado. | Obligatorio | ✅ Verificada | Camino verde y camino con falla ambos observados (el instalador muere con el error exacto del componente) |
| RF-DEPLOY-015 | La licencia debe poder trasladarse de una máquina a otra (laptop → servidor) sin re-emisión. | Obligatorio | ✅ Verificada | El instalador la copia a `license/` y activa `LICENSE_ENFORCE=true`; payload sin amarra a hardware (congelado, verificado 2026-09-22) |
| RF-DEPLOY-016 | El instalador debe poder ejecutarse de forma interactiva (wizard paso a paso) sin conocer los flags, manteniendo el modo flags para automatización. | Obligatorio | ✅ Verificada | `install.sh` sin flags y en terminal pregunta perfil → hostname → licencia (autodetectada junto al bundle) → restore, con defaults entre corchetes; los flags pasados no se re-preguntan y `--yes` desactiva el wizard; sin TTY exige flags como antes. Bash puro, sin dependencias nuevas (el cliente está offline) |

## 7. Requerimientos no funcionales (RNF-…)

| ID | Requerimiento | Prioridad | Estado | Verificación |
|:--|:--|:--|:--|:--|
| RNF-DEPLOY-001 | El perfil `server` debe cifrar todo el tráfico navegador↔servidor con TLS. | Obligatorio | ✅ Verificada | TLS 1.2/1.3 + redirect 80→443 + HSTS; `curl --cacert ca.crt` sin `-k` pasa (CA interna verificada); terminales instalan la CA con `install-ca-windows.ps1` |
| RNF-DEPLOY-002 | PostgreSQL debe escuchar solo en el servidor (localhost o red interna de Docker); Redis no debe publicar puertos al host. | Obligatorio | ✅ Verificada | PG como servicio compose publicado en `127.0.0.1` únicamente (cambio de arquitectura del ensayo F3: nativo-loopback + contenedores es imposible en Linux); Redis sin `ports` |
| RNF-DEPLOY-003 | Debe existir una única imagen de frontend para todos los clientes (sin direcciones horneadas por cliente). | Obligatorio | ✅ Verificada | `VITE_API_URL=/api` (relativo); la misma imagen sirve laptop (template HTTP) y server (tls.conf montada por compose) |
| RNF-DEPLOY-004 | Los orígenes permitidos (CORS) deben ser configurables por variables de entorno, sin cambios de código. | Obligatorio | Congelado (ya se comporta así) | `CORS_ALLOWED_ORIGINS` ya leído de entorno (`main.go:271`, verificado 2026-09-22); con proxy `/api` el navegador ni siquiera cruza origen |
| RNF-DEPLOY-005 | Los secretos (JWT, contraseñas de BD, claves de licencia) deben vivir solo en `.env` por instalación, jamás dentro de imágenes. | Obligatorio | ✅ Verificada | `.env.docker` generado por el instalador con secrets aleatorios, fuera del rsync de imágenes; imágenes solo contienen binarios/SPA |
| RNF-DEPLOY-006 | El acceso administrativo por SSH al servidor debe estar restringido (firewall y/o solo vía VPN). | Deseable | ⏳ En sitio | Regla documentada en `deploy/README.md` y guías; el escaneo del ensayo mostró 22 filtrado, el endurecimiento concreto es por cliente |
| RNF-DEPLOY-007 | Una instalación nueva en hardware preparado debe completarse en 30 minutos o menos, excluyendo descargas de imágenes. | Deseable | ⚠️ Parcial | Ensayo con builds: los installs con imágenes cargadas corren en minutos (el build local cacheado añadió ~2 min); cronometrar en el primer cliente |

## 8. Reglas de negocio (RN-…)

| ID | Regla | Motivo (el por qué) |
|:--|:--|:--|
| RN-DEPLOY-001 | En el perfil `server`, el frontend y la API se sirven bajo un único origen con proxy inverso `/api` (contrato F7.7 invertido solo para producción). | Una imagen de frontend sirve a todos los clientes; sin CORS, sin IP horneada, sin builds por cliente. La regla F7.7 de AGENTS.md exige cambiar `VITE_API_URL` y el rewrite de nginx **en el mismo diff**. |
| RN-DEPLOY-002 | El TLS del perfil servidor usa una CA interna auto-firmada, generada y distribuida por el instalador. | El servidor no tiene dominio público; la CA interna elimina la advertencia del navegador de forma automatizable. Si el cliente ya usa Tailscale, se admiten sus certificados como alternativa. |
| RN-DEPLOY-003 | Las sucursales se conectan por VPN, con Tailscale como herramienta por defecto. | Atraviesa redes difíciles (CGNAT, común en ISPs de Paraguay), se automatiza con claves pre-aprobadas y no expone puertos. Alternativa aceptada: WireGuard puro cuando el cliente rechace terceros. |
| RN-DEPLOY-004 | El modo offline de sucursales queda fuera del alcance v1; la limitación se informa al cliente y se documenta en la guía. | Sincronización offline con resolución de conflictos es una feature de producto de gran costo; se registra como posible ampliación futura, no como promesa. |
| RN-DEPLOY-005 | La documentación de requerimientos vive en `docs/requirements/` y las guías operativas en `docs/guides/` (una sola fuente de verdad por tema). | Evita árboles paralelos que duplican contenido y se desactualizan juntos. |
| RN-DEPLOY-006 | El perfil `server` publica puertos estándar 80/443 (no 8080/8081, herencia de desarrollo). | Terminales sin configuración rara; el cortafuegos del cliente se comporta como en cualquier web. |
| RN-DEPLOY-007 | Ninguna actualización ni migración se ejecuta sin backup automático previo con etiqueta de versión de origen. | Es la única reversión real ante una migración fallida (las migraciones son forward-only). |

## 9. Preguntas frecuentes

**¿Si se corta el internet, la sucursal sigue funcionando?**
No. En v1 la sucursal necesita internet para llegar al servidor central por el túnel. Está fuera de alcance el modo offline (RN-DEPLOY-004); el local principal sigue funcionando normal porque el servidor está ahí.

**¿Migrar de la laptop al servidor obliga a pagar o re-emitir la licencia?**
No. La licencia no está amarrada a ninguna máquina: se copia tal cual durante la migración (RF-DEPLOY-015).

**¿Qué hay que instalar en las cajas/terminales?**
Nada. Un navegador y la dirección del servidor. Las impresoras de tickets se conectan a cada terminal porque la impresión sale del navegador.

**¿Si roban la laptop con todo el sistema?**
El disco está cifrado (ilegible sin la contraseña) y los backups diarios están fuera de la máquina: se restaura en otra computadora y el negocio sigue.

**¿El cliente puede negarse a actualizar?**
Sí — el modelo es "actualización cuando el cliente lo considera necesario". La política comercial (qué versiones reciben soporte, qué cuesta actualizar tarde) es parte del modelo de mantenimiento, no de este requerimiento.

**¿Hay que abrir puertos del router del local para la sucursal?**
Con Tailscale, no. Es su razón de ser (RN-DEPLOY-003). Con WireGuard puro sí haría falta redirigir un puerto en el router principal.

**¿Dónde queda registrado qué versión tiene cada cliente?**
En el inventario de instalaciones del vendor, verificable contra el endpoint `/version` de cada instalación (RF-DEPLOY-010).

## 10. Glosario

| Término | Explicación en una línea |
|:--|:--|
| On-premise | Instalado en las máquinas del cliente, no en la nube del proveedor |
| Bundle de release | Paquete cerrado y versionado con todo lo necesario para instalar/actualizar |
| Perfil de instalación | Misma aplicación, distinta forma de exponerse: `laptop` (todo local) o `server` (servidor con TLS) |
| Proxy inverso | Un solo "mostrador" (nginx) que entrega la web y la API bajo una misma dirección |
| TLS | El candado que cifra lo que viaja entre navegador y servidor |
| CA interna | Un "sello de confianza" propio que se instala en las terminales para que el candado no dé advertencia |
| VPN | Túnel privado por internet entre la sucursal y el servidor |
| Tailscale / WireGuard | Herramientas para armar ese túnel; Tailscale es más automática, WireGuard más "hágalo usted mismo" |
| CGNAT | Tipo de red de ISP que complica conexiones entrantes caseras (y que Tailscale atraviesa) |
| Idempotente | Que se puede re-ejecutar sin romper nada |
| Runbook / guía operativa | Receta paso a paso para humanos, con comandos copiables y verificaciones |
| Smoke test | Verificación rápida de que lo esencial funciona (health, login, versión) |
| SemVer | Versionado semántico `mayor.menor.parche` |

## 11. Trazabilidad

| Artefacto | Referencia |
|:--|:--|
| Plan de ejecución (EJECUTADO F1–F4 el 2026-09-22; purgado al cierre) | `conductor/PLAN_DEPLOY_ESCALADO_ONPREMISE_2026-09-22.md` (historial git) |
| Evaluación de origen (hallazgos verificados en código) | Sesión 2026-09-22: compose principal con ruta de frontend rota (`../../web-project/erp-webapp`), comentario de proxy `/api` desactualizado en `docker-compose.yml`, Redis expuesto en `docker-compose.server-local.yml`, `VITE_API_URL` horneado, licencia sin amarra a hardware (`internal/licensing/domain.go`), CORS por entorno (`main.go:271`), `/health` existente y `/version` inexistente |
| Material de deploy disperso a consolidar | BE raíz: `WINDOWS_SETUP.md`, `DOCKER_WINDOWS_GUIDE.md`, `DOCKER_SETUP_COMPLETE.md`, `apply_migration_quick.ps1`, `verify-postgres-windows.ps1`; BE `docs/`: `DEPLOYMENT_DOCKER.md`, `docs/deployment/`, `docs/guides/`; FE: `DOCKER_BACKEND_PATH_GUIDE.md`, `scripts/docker-compose-*.ps1` |
| Requisito relacionado | REQ_BIPACK (licenciamiento: el bundle debe portar la licencia del cliente) |
| Guías operativas (el CÓMO) | `docs/guides/GUIDE_INSTALACION_NUEVO_CLIENTE.md`, `GUIDE_TERMINALES_LOCAL.md`, `GUIDE_MIGRACION_LAPTOP_A_SERVIDOR.md`, `GUIDE_SUCURSAL_NUEVA.md` |
| Scripts implementados (BE repo) | `deploy/scripts/{install,migrate-to-server,make-release-bundle,gen-ca,add-branch,support-vpn}.sh`, `deploy/scripts/install-ca-windows.ps1`, `docker-compose.yml` (perfiles laptop\|server + postgres/migrate), `internal/platform/version` (`/version`), `database/baseline/` + `make db-baseline` |

## 12. Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v0.6 | 2026-09-26 | Hallazgo de instalación real en VM (2º): tras login en instalación limpia el admin quedaba trabado en la selección de sucursal — el baseline salta la migración que sembraba la sucursal MAIN y `create_initial_admin` no otorgaba acceso a sucursal. Fixes: seed `002_initial_branch.sql` (MAIN + config fiscal placeholder, solo si no hay sucursales) corrido por install.sh paso 8b; `create_initial_admin` otorga FULL a MAIN en la misma tx; FE: BranchSelection ofrece "Continuar en vista global" al admin cuando no hay sucursales (deja de ser callejón sin salida). Guía §6 actualizada (cargar datos reales de MAIN antes de facturar) | renpereiradx + agente |
| v0.5 | 2026-09-26 | Hallazgo de instalación real en VM: con las imágenes del bundle presentes, el paso 5 del instalador igual caía en "sin bundle de imágenes" y un `docker compose build` imposible offline ("failed to read dockerfile"). Dos bugs corregidos: `--source` con ruta relativa dejaba de resolver tras el `cd` interno del paso 2 (todos los paths de usuario se absolutizan antes), y corriendo desde el bundle sin `--source` no se miraba la raíz del bundle, donde viven las imágenes (se buscan en ambos layouts). El fallback de build ahora muere con mensaje accionable cuando el source no tiene Dockerfiles (un bundle nunca los trae) | renpereiradx + agente |
| v0.4 | 2026-09-26 | Wizard interactivo en `install.sh` (RF-DEPLOY-016): correrlo sin flags en una terminal pregunta paso a paso lo que falte con defaults; flags y `--yes` mantienen el modo automatizado intacto. Mensaje final de `make-release-bundle.sh` simplificado (el técnico ya no tipea flags) | renpereiradx + agente |
| v0.3 | 2026-09-22 | Auditoría post-implementación: 4 correcciones en scripts — (1) install.sh ya no borra `license/` al re-instalar (rsync exclude; RF-DEPLOY-011 ahora cubre licencia), (2) pipefail en pipelines de backup y restore dentro del contenedor (dump truncado ya no puede pasar por exitoso), (3) camino `--remote` de migrate-to-server.sh reparado (path real del instalador en el bundle + `sudo env`), (4) smoke con login del admin recién creado (RF-DEPLOY-014 completo en el script). Higiene FE: contraseña vieja de DB eliminada de 4 archivos (credencial ya rotada 2026-09-21) | renpereiradx + agente |
| v0.2 | 2026-09-22 | Estado Implementado: F1–F4 ejecutadas; §6/§7 verificaciones marcadas con resultados reales del ensayo (2 hallazgos de fondo resueltos: migraciones no corribles desde cero → baseline; PG nativo-loopback imposible en Linux → PG servicio compose). Reservas de sitio: join Tailscale, SSH, 31 días de retención | renpereiradx + agente |
| v0.1 | 2026-09-22 | Borrador inicial a partir de la evaluación; decisiones D1–D6 registradas como RN-DEPLOY-001..006 | renpereiradx + agente |
