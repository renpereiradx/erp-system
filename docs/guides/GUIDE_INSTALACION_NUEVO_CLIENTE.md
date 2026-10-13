# GUIDE_INSTALACION_NUEVO_CLIENTE — Instalar el ERP en un cliente nuevo

## 1. Ficha

| Campo | Valor |
|:--|:--|
| Módulo | Instalación on-premise de clientes |
| Código | GUIDE-INST-CLIENTE |
| Versión | v1.9 |
| Estado | Vigente (probada contra el ensayo E2E del plan PLAN_DEPLOY_ESCALADO_ONPREMISE) |
| Fecha | 2026-10-13 |
| Audiencia | Implementación / soporte |
| Requerimiento | [`REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md`](../requirements/REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md) (RF-DEPLOY-001..003, 011..023) · [`REQ_BIPACK_LICENCIAMIENTO_BI.md`](../requirements/REQ_BIPACK_LICENCIAMIENTO_BI.md) (RN-08/09) |

---

## 2. Qué necesitás

**Hardware del servidor** (perfil `server`) o laptop (perfil `laptop`):

| Requisito | Mínimo |
|:--|:--|
| SO | Linux x64 (Debian/Ubuntu recomendado). Máquina Windows: Linux en WSL2, solo perfil `laptop` → [`GUIDE_INSTALACION_WINDOWS.md`](GUIDE_INSTALACION_WINDOWS.md) |
| RAM | 4 GB (8 GB recomendado) |
| Disco | 20 GB libres (más el crecimiento de la base) |
| Red | Perfil `server`: IP fija en la red del local (LAN). Perfil `laptop`: sin requisitos |
| Software | Docker Engine + plugin compose (`docker compose version`). **Nada más**: PostgreSQL y Redis corren dentro del stack |

**Del lado del vendor:**

- El **bundle de release**: `erp-release-<versión>.tar.gz` (lo genera `business_management/deploy/scripts/make-release-bundle.sh`).
- La **licencia del cliente** si ya compró: `license.json` + `licensing-public.key` (generadas con `cmd/licensegen`; ver [`RUNBOOK_LICENCIAS.md`](RUNBOOK_LICENCIAS.md)). Si aún no compró, **no hace falta nada**: la instalación arranca en evaluación full (Core+BI) por 30 días desde el primer arranque; al vencer se bloquea y se activa cargando la licencia (rescate web — REQ_BIPACK v2.0).

### 2.1 Cargar licencia + clave pública en el bundle (lado vendedor)

Para que el instalador **autodetecte** la pareja, ambos archivos viajan DENTRO
del bundle, en `business_management/` y con esos nombres exactos. El `.tar.gz`
generado por `make-release-bundle.sh` **no los incluye**: copiarlos y
re-empaquetar. Reemplazar `<v>` por la versión del bundle.

```bash
cd ~/dev/erp-system/business_management/dist/erp-release-<v>/

# renombrar la pública al nombre que el instalador busca + traer la licencia:
cp ~/licencias/custodia/public.key business_management/licensing-public.key
cp ~/licencias/entregas/<cliente>/license.json business_management/license.json

# re-empaquetar (el .tar.gz viejo no contiene los archivos agregados):
cd ..
tar czf erp-release-<v>.tar.gz erp-release-<v>/
```

Notas:

- `sha256sum -c SHA256SUMS` en el cliente sigue pasando: el manifiesto verifica
  solo los archivos que lista; los agregados se ignoran.
- ¿El cliente aún no compró? Se puede omitir el `license.json` (evaluación de
  30 días), pero **incluir igual la clave pública**: el rescate web del día del
  bloqueo la necesita.
- Los flags `--license` / `--license-public-key` (§5) son equivalentes; meterlos
  en el bundle solo ahorra tipearlos.

---

## 3. El bundle: qué es y cómo verificarlo

El bundle es un paquete cerrado con todo: imágenes Docker (backend + frontend
**+ las tres de terceros del stack**: postgres, redis y migrate), compose
canónico, migraciones, baseline de schema, seeds RBAC, instalador y config
nginx del frontend — desde el fix BE `428a6ff` el arranque del cliente no
toca internet.

> **Bundles emitidos antes del 2026-10-10** no traen las imágenes de
> terceros: `postgres:16-alpine`, `redis:7-alpine` y `migrate/migrate:v4.18.1`
> las baja Docker la primera vez que las necesita. Con internet en la fase de
> instalación no hace falta nada (el instalador avisa cuáles faltan); cliente
> 100% offline con un bundle viejo: llevarlas en el pendrive (`docker save`
> en la oficina → `docker load` en el cliente — receta en
> [`GUIDE_INSTALACION_WINDOWS.md`](GUIDE_INSTALACION_WINDOWS.md) §4 paso 4).

```bash
# dentro del bundle descomprimido, verificar integridad:
sha256sum -c SHA256SUMS        # debe terminar sin "FAILED"
```

Estructura del bundle:

```
erp-release-<v>/
├── business_management/   # compose, migraciones, seeds, baseline, instalador
├── erp-webapp/nginx/      # config nginx del frontend (template + tls.conf)
└── images/*.tar.gz        # imágenes Docker del release (backend, frontend
                           #  + postgres/redis/migrate desde el 2026-10-10)
```

---

## 4. Elegir perfil

| | `laptop` | `server` |
|:--|:--|:--|
| Para | Cliente chico, todo en una máquina | Local con varias terminales / sucursales |
| Frontend | `http://localhost:18081` **solo en esa máquina** | `https://<hostname>/` (80 → 443) para toda la red |
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
    --license-public-key ~/ruta/licensing-public.key \
    --yes
```

> `--license` y `--license-public-key` van SIEMPRE juntos (el instalador lo
> exige en todos los modos): sin la clave pública el backend no puede verificar
> la licencia y la instalación correría en evaluación sin avisar.

El instalador es **idempotente**: se puede re-ejecutar sin romper nada (útil si
algo falla a mitad de camino, y para **actualizar** una instalación: bundle nuevo
→ mismo comando → migraciones de delta aplicadas, imágenes reemplazadas, datos
y secretos intactos).

Qué hace (12 pasos, todos con salida visible):

1. Pre-chequeos (SO Linux, Docker corriendo, disco ≥ 5 GB)
1b. **Stacks ERP preexistentes**: si en el motor Docker hay contenedores de
   una instalación vieja u otra instalación (no de esta corrida), pregunta
   qué hacer — ver §7.1. En máquina limpia ni se nota
2. Copia el stack a `/opt/erp/` (la instalación vive ahí; el repo queda fuera)
3. Genera `.env.docker` con **secrets nuevos** (JWT + password de DB aleatorios)
4. Levanta PostgreSQL (servicio propio con volumen dedicado, solo loopback)
5. Carga las imágenes del bundle (`docker load`) — sin internet
6. Genera la CA interna y el cert del servidor (solo perfil `server`)
7. Migraciones: base nueva → baseline de schema + estampado de versión; base existente → delta con golang-migrate
8. Seeds RBAC solo si la base está vacía (9 roles canónicos)
9. Usuario admin inicial con **contraseña aleatoria** guardada en `/opt/erp/credentials/`
10. Licencia: si está, `LICENSE_ENFORCE=true` y licencia activa; si no, evaluación de 30 días con aviso (REQ_BIPACK v2.0 — el bloqueo al vencer solo se levanta con licencia, vía rescate web)
11. Levanta el stack completo y espera healthchecks
12. **Smoke test automático**: health + `/version` + frontend + proxy `/api` (+ redirect HTTPS en server). Si algo falla, el instalador muere con el error visible

(El chequeo de puertos libres corre entre 1b y 2: si el operador elige
reemplazar un stack viejo en 1b, los puertos que este retenía quedan libres
para el chequeo.)

Al terminar imprime un resumen con URLs, ubicación de la CA y próximos pasos.

### Overrides útiles

| Situación | Variable / flag |
|:--|:--|
| Puerto 80/443 ocupado | `ERP_HTTP_PORT=8080 ERP_HTTPS_PORT=8443` antes del comando |
| Otro PostgreSQL ya existe en el servidor | mover el puerto del stack: `ERP_DB_PORT=5433` |
| **Máquina con un sistema ERP viejo** | el instalador lo detecta y pregunta (§7.1) — o `--replace-stack` directo |
| **Instalar en paralelo con otro stack ERP** | wizard opción 2, o a mano: `COMPOSE_PROJECT_NAME=erp-2 ERP_ROOT=/opt/erp-2 ERP_DB_PORT=6433 ERP_BACKEND_PORT=19080 ERP_FRONTEND_PORT=19081` (§7.1) |
| Instalar sin licencia por ahora | omitir `--license` (queda en evaluación: 30 días full, luego pantalla de rescate) |
| Activar un cliente que ya pagó (post-evaluación) | cargar el `license.json` desde la pantalla "Licencia requerida" o `curl -X PUT .../api/v1/system/license` ([RUNBOOK_LICENCIAS.md](RUNBOOK_LICENCIAS.md) §4b) |
| Probar sin tocar `/opt` | `ERP_ROOT=/tmp/erp-test ./install.sh ...` |

---

## 6. Post-instalación (orden obligatorio)

1. **Guardar la contraseña del admin** (`/opt/erp/credentials/admin-inicial.txt`)
   en el gestor del cliente y **borrar el archivo**.
2. **Probar login** desde el navegador del servidor: `https://<hostname>/`.
   La instalación limpia siembra la sucursal **MAIN "Nombre Sucursal"**
   (todo placeholder: razón social `Nombre Empresa`, nombre comercial
   `Nombre Comercial`, RUC `00000000-0`, config fiscal con `timbrado 00000000`):
   elegir esa sucursal al entrar y **cargar los datos reales del negocio** (nombre,
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
# con un bundle más nuevo, desde el directorio del bundle NUEVO:
tar xzf erp-release-<nueva>.tar.gz
sudo ./erp-release-<nueva>/business_management/deploy/scripts/install.sh
```

El instalador detecta la instalación previa y **Enter hereda sus decisiones**:
perfil instalado, hostname del certificado, licencia y overrides de puertos
(puertos custom como `ERP_DB_PORT` se pierden si se re-genera el `.env` sin
ellos — el instalador los recupera del `.env` existente). El prompt de backup
a restaurar se deja **vacío** en una actualización normal (es solo para
migración laptop→servidor).

Cambiar de perfil sobre una instalación existente (server↔laptop) pide
confirmación explícita: reconfigura puertos/TLS y **los contenedores del perfil
anterior no se detienen solos** (`docker compose --profile <viejo> down` a mano).

Para automatizar (o sin terminal): los flags de siempre —

```bash
sudo ./erp-release-<nueva>/business_management/deploy/scripts/install.sh \
    --profile server --hostname erp.miempresa.local --yes
```

El `.env.docker` existente (secretos) se conserva; las imágenes nuevas del bundle
se cargan y el stack se recrea con ellas; las migraciones nuevas se aplican como
delta. **Antes de actualizar: verificar que existe el backup de anoche**
(`ls /opt/erp/backups/`) — es la única reversión real (RN-DEPLOY-007).

### 7.1 Máquina con un sistema ERP previo (reemplazo protegido o paralelo)

Escenario: la máquina ya corre un ERP de **otra época** (versión vieja, otra
instalación, contenedores que quedaron de antes). Instalar el bundle nuevo
"parece un update", pero no lo es: es una instalación nueva sobre una máquina
ocupada — y hasta el 2026-10-13 el instalador ni se enteraba: si el stack
viejo compartía nombre de proyecto (`erp`), `docker compose up` lo
**reemplazaba en silencio** (así desapareció el stack viejo de la laptop del
negocio; su DB sobrevivió solo porque vivía en un Postgres nativo fuera de
Docker).

Desde REQ_DEPLOY v0.13 el paso 1b del instalador inventaría los contenedores
ERP del motor Docker y, si encuentra alguno que no pertenezca a esta
instalación, **pregunta**:

1. **Reemplazar** — antes de tocar nada archiva un resguardo en
   `/opt/erp/backups/pre-replace-<fecha>/` (inventario Docker, `inspect` de
   los contenedores, config compose si queda su directorio, y `pg_dump`
   best-effort de su Postgres) y recién ahí retira los contenedores viejos.
   El volumen del stack viejo solo se borra si su DB quedó dumpeada — sin
   dump el instalador corta (mejor pararse a mano que pisar datos).
   **Copiar el resguardo fuera de la máquina**, como cualquier backup.
2. **Instalar en paralelo** — el stack viejo sigue corriendo; el nuevo usa
   proyecto (`erp-<sufijo>`), raíz (`/opt/erp-<sufijo>`), puertos (DB 6433,
   backend 19080, frontend 19081 — en server 28080/28443) y timer de backup
   (`erp-backup-<sufijo>`) propios. Típico para capacitación con el sistema
   viejo aún facturando. Cada instalación recuerda su identidad en su
   `.env` (`COMPOSE_PROJECT_NAME`): `docker compose`, backups y migraciones
   de cada una apuntan a su propio stack.
3. **Cancelar**.

En modo automatizado (`--yes` / sin terminal) no hay pregunta: o se pasa
`--replace-stack` (resguardo + baja de los viejos, igual que la opción 1) o
el instalador **corta** listando lo detectado. El paralelo automatizado es
por variables (ver tabla de overrides del §5).

> ¿Reemplazar o empezar de cero con los datos del viejo? Reemplazar instala
> LIMPIO (sin datos viejos). Para traer clientes/proveedores (o todo) desde
> el sistema viejo: [`GUIDE_MIGRACION_DATOS_VIEJOS.md`](GUIDE_MIGRACION_DATOS_VIEJOS.md) —
> export del lado viejo + import idempotente en el nuevo.

---

## 8. Problemas frecuentes

| Síntoma | Causa probable | Qué hacer |
|:--|:--|:--|
| `docker compose: command not found` | Falta el plugin compose | `apt install docker-compose-plugin` |
| El instalador corta en el paso 1: "puerto X ya está en uso" | Otra cosa ocupa ese puerto (apache, Tailscale, otro stack) | Es intencional: corta ANTES de escribir nada en disco. Liberar el puerto o correr con el override indicado (`ERP_DB_PORT`, `ERP_BACKEND_PORT`, `ERP_HTTP_PORT`, `ERP_HTTPS_PORT`, `ERP_FRONTEND_PORT`). Si el que retiene el puerto es el **propio stack ERP** (update o re-corrida con los contenedores vivos), el instalador lo detecta y sigue: compose recrea sus contenedores (BE `616d551`) |
| El instalador muere en "frontend HTTPS no responde" (paso 12) | El frontend levantó mal por otra causa (los puertos ya se validan en el paso 1) | `docker compose --profile server logs frontend-server` |
| Instalación cortada a mitad (luz, ctrl-c, o corridas previas a 2026-10-10) | El script es lineal: lo hecho queda hecho | Re-correr `install.sh` (update-aware: hereda perfil/puertos del `.env` previo). Verificar al final `systemctl status erp-backup.timer` — el backup se instala en el paso 2b, temprano, así un corte tardío ya lo dejó programado |
| Login OK pero sin datos y sin roles | Bundle sin baseline (muy viejo) | Regenerar bundle (`make-release-bundle.sh` lo exige) |
| El navegador de las terminales da advertencia TLS | CA no instalada en esa terminal | `GUIDE_TERMINALES_LOCAL.md` |
| Pantalla "Licencia requerida" tras instalar | Evaluación agotada sin licencia | Cargar la licencia desde esa misma pantalla ([RUNBOOK_LICENCIAS.md](RUNBOOK_LICENCIAS.md) §4b); no hace falta SSH |
| El rescate web rechaza la licencia (LICENSE_INVALID) | `LICENSE_PUBLIC_KEY` vacía o distinta a la del par vendedor | Cargar la pública correcta en `.env.docker` y recrear el backend |
| `error: Dirty database version <X>` en el paso 7 | Una migración quedó interrumpida a mitad (terminal cerrada, corte, o un bug de migración — el `20261004190603` con `OWNER TO dev_user` lo causaba; corregido en BE `9a91a32`) | Recuperación *dirty* (abajo) y re-ejecutar el instalador |
| El paso 1b corta: "stacks ERP ajenos presentes" | En el motor Docker hay contenedores de un ERP viejo/otra instalación y el modo automatizado no decide | Es intencional (RF-DEPLOY-021): usar `--replace-stack` (resguardo + baja), paralelo por variables (§7.1), o retirarlos a mano |
| Docker Desktop muestra un stack ERP que "ya no existe" en el disco | Instalación paralela desarmada a medias o restos de un reemplazo | `docker compose -p <proyecto> down -v` con el nombre de proyecto explícito (los volúmenes sobreviven a los contenedores); los resguardos de reemplazo viven en `/opt/erp/backups/pre-replace-*/` |

**Recuperación de base *dirty* (paso 7):** forzar la última versión que
completó y dejar que el instalador re-aplique el delta (las migraciones usan
`CREATE OR REPLACE` / idempotencia, re-aplicar es seguro):

```bash
cd /opt/erp/business_management
PG_PASSWORD="$(sudo grep -E '^DB_PASSWORD=' .env.docker | cut -d= -f2-)"
sudo docker compose --profile laptop run --rm --entrypoint sh migrate -c \
  "migrate -path /migrations -database 'postgresql://erp_app:$PG_PASSWORD@postgres:5432/erp_db?sslmode=disable' force <versión_limpia>"
# <versión_limpia> = la migración ANTERIOR a la que quedó dirty (si dirty es N, forzar N-1)
# luego re-ejecutar install.sh: aplica el delta desde esa versión
```

---

## 9. Desinstalar (empezar de cero)

Para corregir una instalación donde se eligió mal algo de la configuración
inicial (perfil, hostname, nombre del cliente), para repetir un ensayo, o para
retirar el sistema de la máquina. **Borra la base de datos completa**: si hay
datos que valgan, rescatar antes los backups (`/opt/erp/backups/`).

Borrar el contenedor Docker **no alcanza**. La instalación deja cosas en tres
lugares y hay que limpiar los tres:

1. **Volúmenes Docker** — la base de datos vive en el volumen `erp_erp_pg_data`
   (y el cache de Redis en `erp_redis_data`), que sobrevive a los contenedores:
   es un disco aparte que Docker le presta al contenedor. Si solo se borran
   contenedores, la reinstalación reusa la base vieja: el instalador la detecta
   como "existente", **no re-seede** roles/sucursal/admin (pasos 8-9) y arrastra
   justo la configuración que se quería descartar.
2. **`/opt/erp/`** (o el `ERP_ROOT` que se haya usado) — el stack, los secretos
   (`.env.docker`), la licencia (`license/`), la CA (`deploy/certs/`), los
   backups, las credenciales del admin y el `.env` con las decisiones que el
   instalador **hereda** en una re-corrida: perfil, hostname del certificado,
   nombre del cliente y puertos (§7). Re-correir `install.sh` sin borrar esto
   reproduce las mismas elecciones, con Enter las confirma.
3. **systemd** — el timer del backup diario (`erp-backup.timer`, 02:30).

### 9.1 Borrón y cuenta nueva

```bash
# 0) SOLO si hay datos que rescatar: llevar los backups a otro lado
sudo cp -r /opt/erp/backups ~/respaldo-erp/

# 1) bajar TODOS los contenedores del stack y BORRAR los volúmenes
cd /opt/erp/business_management
sudo docker compose --profile laptop --profile server down -v --remove-orphans

# 2) borrar la instalación completa (stack, secretos, licencia, CA, backups, credenciales)
sudo rm -rf /opt/erp

# 3) quitar el backup diario programado (instalación paralela: erp-backup-<sufijo>)
sudo systemctl disable --now erp-backup.timer 2>/dev/null || true
sudo rm -f /etc/systemd/system/erp-backup.service /etc/systemd/system/erp-backup.timer
sudo systemctl daemon-reload
```

Notas:

- El `down` lleva **ambos perfiles** porque compose solo detiene los servicios
  del perfil activo: si la instalación previa fue del otro perfil, sus
  contenedores quedan vivos (mismo caso documentado en §7).
- El `-v` es lo que borra los volúmenes — la base de datos. Sin `-v` es solo un
  apagado con eliminación de contenedores.
- Si el `.env` de la instalación no se puede leer no importa: correr el
  `down` con `-p` explícito (`docker compose -p erp down -v`) o desde
  cualquier compose canónico pasando `COMPOSE_PROJECT_NAME`. El nombre de
  proyecto por defecto sigue siendo `erp` (`name: ${COMPOSE_PROJECT_NAME:-erp}`
  en el compose); una instalación en PARALELO usa el suyo (guardado en su
  `.env`) y su borrón es igual pero en su directorio y con su proyecto —
  **un `down -v` suelto en otro directorio igual apunta al stack cuyo
  proyecto resuelva**: para desarmar una prueba, siempre `-p` explícito.

Verificar que no quedó nada:

```bash
sudo docker volume ls | grep erp_    # no debe listar erp_erp_pg_data ni erp_redis_data
sudo docker ps -a | grep erp         # sin contenedores del stack
ls /opt/erp                          # no debe existir
```

Reinstalar: seguir §5 normalmente. La instalación limpia genera **todo nuevo**:
secretos (JWT + password de DB), admin con otra contraseña (`credentials/`),
CA con otra clave y evaluación de 30 días desde el primer arranque si el bundle
no trae licencia. Dos consecuencias a tener en cuenta:

- **Perfil `server` con terminales ya configuradas**: la CA nueva NO es la que
  las terminales tienen instalada → volver a instalar la CA en cada terminal
  ([GUIDE_TERMINALES_LOCAL.md](GUIDE_TERMINALES_LOCAL.md)).
- **Licencia instalada a mano** (flag `--license`, no §2.1): el archivo no está
  en el bundle → tener la ruta a mano para pasarla de nuevo en la reinstalación.

### 9.2 Antes del borrón: ¿se puede corregir sin desinstalar?

| Qué quedó mal | Alternativa sin borrón |
|:--|:--|
| Nombre del cliente | re-corrida con `--client-name "Nombre correcto"` (el instalador lo re-guarda) |
| Hostname del certificado (server) | re-emitir desde `$ERP_ROOT/business_management`: `bash deploy/scripts/gen-ca.sh --force-server --org "Cliente" nuevo.hostname` (y reinstalar la CA en terminales) |
| Datos del negocio (razón social, RUC, timbrado, nombre de sucursal) | **NO desinstalar**: cargarlos en *Configuración → Sucursales* (§6 paso 2) |
| Perfil laptop↔server | posible en caliente con confirmación (§7), pero el borrón es más simple: no deja contenedores del perfil viejo ni CA/puertos a re-crear a mano |

---

## 10. Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v1.9 | 2026-10-13 | §7.1 nuevo: máquina con un sistema ERP previo — paso 1b del instalador (detección de stacks ajenos, reemplazo con resguardo pre-reemplazo, instalación en paralelo con proyecto/raíz/puertos/timer propios, `--replace-stack`). §5: overrides de reemplazo/paralelo; pasos del instalador actualizados (1b). §8: filas de 1b y restos de stacks. §9.1: proyecto parametrizado `name: ${COMPOSE_PROJECT_NAME:-erp}` y timer por instalación. Contra REQ_DEPLOY v0.13 (RF-DEPLOY-021/022) + hallazgo laptop del negocio | renpereiradx + agente |
| v1.8 | 2026-10-10 | §3: el bundle ahora trae TODAS las imágenes — postgres/redis/migrate incluidas, derivadas del compose (BE `428a6ff`); el instalador las carga antes del primer `up` y avisa si falta alguna (bundle viejo). §9.1: advertencia sobre `down -v` — `name: erp` apunta al stack real desde cualquier directorio; las pruebas se desarman con `-p` explícito | renpereiradx + agente |
| v1.7 | 2026-10-10 | §2: Windows por WSL2 (GUIDE_INSTALACION_WINDOWS nuevo, perfil laptop). §3: las imágenes base postgres/redis NO viajan en el bundle — matiz sobre el offline. §4: puerto del frontend laptop corregido 8081 → 18081 (era pre-BE `9e998e8`). §8: el chequeo de puertos ya no corta cuando el propio stack ERP retiene el puerto en un update (BE `616d551`) | renpereiradx + agente |
| v1.6 | 2026-10-10 | §2.1: rutas vendor — custodia y entregas ahora viven bajo `~/licencias/` (`custodia/` + `entregas/`; RUNBOOK v2.3). Backup del vendor = carpeta `~/licencias/` completa a pendrive/bóveda, nunca nube | renpereiradx + agente |
| v1.5 | 2026-10-10 | §8: instalación interrumpida deja backup sin instalar (hallazgo real erp-test 2026-10-08, corte en 11/12 por puertos 80/443 ocupados — el "Extra: backup" era lo último tras el smoke). Corregido en `install.sh` (REQ_DEPLOY v0.11): puertos fatales en el paso 1 y backup en el paso 2b | renpereiradx + agente |
| v1.4 | 2026-10-08 | §9 nuevo: desinstalación completa para empezar de cero (los 3 lugares: volúmenes Docker + `/opt/erp` + timer systemd), verificación, consecuencias de la reinstalación limpia (CA/admin/evaluación) y alternativas al borrón (§9.2). Escrito contra `install.sh` y `docker-compose.yml` | renpereiradx + agente |
| v1.3 | 2026-10-06 | §8: recuperación de base *dirty* del paso 7 (force última versión limpia + re-run) — reproducido y verificado en la VM de prueba | renpereiradx + agente |
| v1.2 | 2026-10-06 | §2.1 nuevo: cargar licencia + clave pública dentro del bundle (autodetección del wizard) y re-empaquetar el `.tar.gz` | renpereiradx + agente |
| v1.1 | 2026-09-29 | Alineación con REQ_BIPACK v2.1 / REQ_DEPLOY v0.8 (auditoría): `--license` exige SIEMPRE `--license-public-key` (el ejemplo canónico ahora los lleva juntos), los updates sin licencia activan la evaluación (30 días desde el update), `license/` se entrega con permisos para el rescate web y el smoke lo verifica, instalación de licencia post-pago también desde Configuración → Licencia | renpereiradx + agente (auditoría ZCode) |
| v1.0 | 2026-09-22 | Primera versión, escrita contra el instalador real (F2-F3 del plan) y el ensayo E2E | renpereiradx + agente |
