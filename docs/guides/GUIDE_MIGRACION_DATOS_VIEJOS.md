# GUIDE_MIGRACION_DATOS_VIEJOS — Traer datos de una instalación vieja del ERP

## 1. Ficha

| Campo | Valor |
|:--|:--|
| Módulo | Migración de datos desde instalaciones de otra época |
| Código | GUIDE-MIGR-VIEJOS |
| Versión | v1.0 |
| Estado | Vigente (herramientas `export-legacy-db.sh` / `import-legacy.sh` contra el instalador real; escenario origen: laptop del negocio con PG nativo + stack Docker viejo, 2026-10-13) |
| Fecha | 2026-10-13 |
| Audiencia | Implementación / soporte |
| Requerimiento | [`REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md`](../requirements/REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md) (RF-DEPLOY-021/023) |
| Guías hermanas | [`GUIDE_INSTALACION_NUEVO_CLIENTE.md`](GUIDE_INSTALACION_NUEVO_CLIENTE.md) (§7 update, §7.1 máquina con sistema previo) · [`GUIDE_MIGRACION_LAPTOP_A_SERVIDOR.md`](GUIDE_MIGRACION_LAPTOP_A_SERVIDOR.md) (misma línea de versión) |

---

## 2. Para qué sirve (y para qué NO)

Cuando en la máquina ya corrió una versión **de otra época** (pre-bundle,
pre party-model, Postgres nativo fuera de Docker) y se instala el sistema
nuevo **empezando de cero**, los datos viejos útiles (sobre todo clientes y
proveedores) quedan inaccesibles desde el sistema nuevo. Esta guía los trae.

NO es para:

- **Actualizar un cliente de la misma línea** (v1.x → v1.x): eso es
  re-correr el instalador (guía madre §7) — migraciones delta solas.
- **Migrar laptop → servidor de la misma versión**: `install.sh --restore`
  (GUIDE_MIGRACION_LAPTOP_A_SERVIDOR). Ojo: `--restore` asume un dump de la
  línea ACTUAL; un dump viejo restaurado por ahí queda con schema atrasado y
  el backend se rompe — para dumps viejos, el orden correcto está en §5.

El origen del problema (por si vuelve a pasar): instalar el stack nuevo
**reemplaza** los contenedores del viejo si comparten nombre de proyecto
(`erp`) — desde el 2026-10-13 el instalador lo detecta, pregunta y deja
resguardo (guía madre §7.1). Esta guía asume que la **base vieja sobrevivió**
(en Postgres nativo, en un volumen, o en un backup).

---

## 3. Las dos eras de esquema (qué se va a encontrar)

| Era | Marcas en la DB vieja | Clientes/proveedores viven en |
|:--|:--|:--|
| **Moderna** (desde 2026-04-21) | tabla `users.parties` | `users.parties` (`party_type` CLIENT / SUPPLIER / CLIENT_SUPPLIER) |
| **Pre party-model** (antes) | tablas `clients.clients` y `clients.suppliers` | `clients.clients` (name, last_name, ci, contact) y `clients.suppliers` (name, tax_id, contact_info) |

No hace falta saberlo de antemano: el export genera un **INVENTARIO** que lo
dice, y el import detecta la era solo y aplica el mapeo correspondiente (el
de la era vieja es el mismo de la migración histórica `20260421110700`,
probado en su momento).

---

## 4. Flujo A — solo clientes y proveedores (lo típico al empezar de cero)

El sistema nuevo queda tal cual (inventario/prices nuevos intactos) y solo se
copian los parties. **Idempotente**: re-correrlo no duplica (salta los que ya
existen por id o por cédula/RUC).

### 4.1 Exportar desde la máquina vieja

En la máquina donde vive la base vieja, con el cliente de Postgres instalado
(en WSL: `sudo apt install -y postgresql-client`):

```bash
# PG nativo de Windows alcanzado desde WSL (host = gateway de WSL; el script
# lo deduce solo si no se pasa --host):
./export-legacy-db.sh --user postgres --db erp_db --host "$(ip route show default | awk '{print $3}')"
```

Produce `erp-legacy-<db>-<fecha>.tar.gz` (dump completo + inventario) e
imprime el inventario en pantalla. Si la contraseña del PG viejo no está en
`~/.pgpass`: `export PGPASSWORD='...'` antes.

### 4.2 Importar en la instalación nueva

Copiar el `.tar.gz` a la máquina/WSL de la instalación nueva y:

```bash
cd /opt/erp          # (o el ERP_ROOT de la instalación)
sudo ./business_management/deploy/scripts/import-legacy.sh ~/erp-legacy-*.tar.gz --parties
```

Qué hace: restaura el dump en una DB de staging (`legacy_stage`) **dentro**
del Postgres del stack — `erp_db` no se toca — detecta la era, y copia los
parties a `erp_db` deduplicando. Si el dump es de era moderna y el delta de
migraciones corre, primero lleva el staging al schema actual.

### 4.3 Verificar y limpiar

1. En la UI: **Clientes** y **Proveedores** — revisar cantidad y unos cuantos
   nombres/cédulas conocidos.
2. El staging queda para reintentos; cuando la importación está validada:
   `sudo docker compose exec postgres dropdb -U erp_app legacy_stage`
   (parado en `/opt/erp/business_management`).
3. Guardar el `.tar.gz` de origen junto a los backups del cliente — es el
   único respaldo de la época vieja.

---

## 5. Flujo B — migración COMPLETA de la época vieja

Para llevarse TODO (ventas, compras, productos, usuarios). Solo si el delta
de migraciones corrió sin errores desde la versión del dump:

```bash
# 1) staging al schema actual (el import sin --parties ya lo intenta e informa)
sudo ./business_management/deploy/scripts/import-legacy.sh ~/erp-legacy-*.tar.gz

# 2) si informó "staging migrado al schema actual": dump del staging
cd /opt/erp/business_management
sudo docker compose exec -T postgres bash -c \
    'set -o pipefail; pg_dump -U erp_app -d legacy_stage --no-owner --no-acl | gzip' \
    > ~/erp-viejo-migrado.sql.gz

# 3) reinstalar desde cero con ese dump (guía madre §9.1 borrón + §5 con):
sudo ./deploy/scripts/install.sh --profile <perfil> --restore ~/erp-viejo-migrado.sql.gz ...
```

El paso 3 funciona porque el dump del paso 2 ya está **al schema actual**
(línea de la instalación nueva): el `--restore` del instalador está pensado
para dumps así. Restaurar el dump VIEJO directamente por `--restore` es el
error a evitar (§2).

**Si el delta no corrió** (versión del dump anterior a migraciones rotas o
intermedias insaltables): la migración completa requiere trabajo a mano del
vendor (mapear tabla por tabla). Los parties igual se pueden traer (Flujo A —
el mapeo de era vieja no depende de las migraciones).

---

## 6. Caso concreto: laptop del negocio (Windows + WSL2, PG nativo)

1. **Exportar** (dentro de Ubuntu/WSL de la laptop):
   ```bash
   sudo apt install -y postgresql-client
   export PGPASSWORD='<password del postgres nativo de Windows>'
   ~/erp-release-<v>/business_management/deploy/scripts/export-legacy-db.sh \
       --user postgres --db erp_db --host "$(ip route show default | awk '{print $3}')"
   ```
   (Los scripts viajan en el bundle; si la instalación ya se hizo, también
   quedaron en `/opt/erp/business_management/deploy/scripts/`.)
2. **Importar** (en la misma WSL, contra el stack nuevo):
   ```bash
   sudo /opt/erp/business_management/deploy/scripts/import-legacy.sh \
       ~/erp-legacy-*.tar.gz --parties
   ```
3. Verificar en `http://localhost:18081` (Clientes/Proveedores) y borrar el
   staging (§4.3).

Si el PG nativo de Windows no acepta conexiones desde WSL (`pg_hba.conf`
restringido o escuchando solo en 127.0.0.1 de Windows): dumpear desde
PowerShell con el pg_dump del PG nativo
(`pg_dump -U postgres -d erp_db --no-owner --no-acl | gzip > dump.sql.gz`),
empaquetar a mano (`tar czf erp-legacy-manual.tar.gz dump.sql.gz`) y seguir
en el paso 2 — el import solo necesita el `dump.sql.gz` dentro del tarball.

---

## 7. Límites y preguntas frecuentes

| Pregunta | Respuesta |
|:--|:--|
| ¿Migra usuarios/contraseñas? | No. Los parties ≠ cuentas: los usuarios se crean de cero en la instalación nueva (el dump de era vieja tampoco tiene `users.users` comparable) |
| ¿Duplica si lo corro dos veces? | No: salta por mismo id o misma cédula/RUC (`document_id`) |
| ¿Y los productos/precios del sistema viejo? | Flujo B (migración completa) — o carga manual: al empezar de cero suele ser preferible (catálogo nuevo, precios nuevos) |
| ¿El cliente en erp_test/laptop puede hacer esto solo? | No: lo corre el vendor — necesita acceso a la DB vieja y decisión de qué traer |
| ¿Sirve para un backup diario normal como origen? | Sí: cualquier `pg_dump` de una DB del producto (`.sql.gz`) empaquetado como en §6 sirve de tarball |
| ¿Riesgo para la DB nueva? | El restore va a `legacy_stage`, nunca a `erp_db`; la única escritura a `erp_db` es el INSERT de parties (deduplicado) |

---

## 8. Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v1.0 | 2026-10-13 | Primera versión: flujos A (parties) y B (completa), eras de esquema, caso laptop WSL2 + PG nativo. Herramientas `export-legacy-db.sh` e `import-legacy.sh` (REQ_DEPLOY v0.13) | renpereiradx + agente |
