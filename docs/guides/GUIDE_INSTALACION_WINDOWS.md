# GUIDE_INSTALACION_WINDOWS — Instalar el ERP en un cliente con Windows

## 1. Ficha

| Campo | Valor |
|:--|:--|
| Módulo | Instalación on-premise de clientes sobre Windows |
| Código | GUIDE-INST-WINDOWS |
| Versión | v1.3 |
| Estado | Vigente (instalación limpia + re-corrida verificadas E2E 12/12; desde el fix BE `428a6ff` el bundle trae TODAS las imágenes — base incluidas — y la Fase 1 es 100% offline; WSL2/Docker Desktop según GUIDE_ENTORNO_DEV_WINDOWS §4-5; desde REQ_DEPLOY v0.14 la licencia/clave embebidas en el bundle se autodetectan en TODOS los modos del instalador) |
| Fecha | 2026-10-19 |
| Audiencia | Implementación / soporte |
| Guía madre | [`GUIDE_INSTALACION_NUEVO_CLIENTE.md`](GUIDE_INSTALACION_NUEVO_CLIENTE.md) (misma licencia, mismos perfiles, mismo instalador) |

---

## 2. Para qué sirve (y para qué no)

El instalador (`install.sh`) exige Linux y corta en Windows nativo (`install.sh:315`).
La salida no es instalar "a mano": es montar **un Linux real dentro de Windows**
(WSL2) y correr ahí el instalador de siempre, intacto. Windows aporta la máquina;
todo lo demás — PostgreSQL, Redis, backend, frontend, migraciones, backup —
corre igual que en la guía madre.

**Alcance: solo perfil `laptop`** (el sistema se usa en esa misma máquina,
HTTP `http://localhost:18081`). Si el cliente va a tener **más de una terminal
o varias máquinas en red**, no es un trabajo para Windows: es el perfil `server`
sobre Linux — [`GUIDE_INSTALACION_NUEVO_CLIENTE.md`](GUIDE_INSTALACION_NUEVO_CLIENTE.md).

> Regla práctica: Windows + WSL2 = cliente de una sola máquina que ya tiene
> (o exige) una laptop con Windows. Cualquier otra cosa: PC dedicada con Linux.

---

## 3. Qué necesitás

| Requisito | Mínimo |
|:--|:--|
| Windows | 10 (2004+) u 11, con permisos de administrador |
| RAM | 8 GB recomendado (4 GB es el piso; con 4 capar WSL con `.wslconfig`, §5 paso 6) |
| Disco | 30 GB libres |
| Software | WSL2 + Ubuntu, Docker Desktop — nada más: PostgreSQL y Redis corren dentro del stack |
| Internet | Solo para la **Fase 0** (preparar la máquina). El ERP después funciona offline |
| Del vendor | El bundle `erp-release-<v>.tar.gz` en un pendrive (+ licencia si ya compró, §5 paso 3). Desde el fix BE `428a6ff` el bundle trae TODAS las imágenes del stack — las base incluidas; con un bundle emitido antes del 2026-10-10, ver §4 paso 4 |

---

## 4. Fase 0 — Preparar la máquina Windows (una vez, con internet)

PowerShell **como administrador** para lo de Windows; terminal Ubuntu para lo de Linux.

1. **WSL2 + Ubuntu**:

   ```powershell
   wsl --install          # instala WSL2 + Ubuntu; pide reiniciar
   wsl --update           # kernel al día (repetir tras el reinicio)
   ```

   Al reiniciar, Ubuntu pide crear un usuario y contraseña (los que sean; es
   el admin de ese Linux, no del ERP).

2. **Docker Desktop**: instalar (instalador como admin), arrancar, y en
   *Settings → Resources → WSL Integration* habilitar la distro Ubuntu.
   Dejar activado **"Start Docker Desktop when you sign in"** (§8: es lo que
   hace que el ERP vuelva solo tras un reinicio). Verificar **dentro de Ubuntu**:

   ```bash
   docker --version && docker compose version
   ```

3. **rsync**: el instalador lo exige y Ubuntu no siempre lo trae:

   ```bash
   sudo apt update && sudo apt install -y rsync
   ```

4. **Imágenes base del stack — solo bundles emitidos antes del 2026-10-10**:
   los bundles actuales ya las traen (postgres, redis y migrate viajan dentro
   de `images/` junto a las del ERP; fix BE `428a6ff`) — no hay que hacer
   nada. Con un **bundle viejo** y un cliente que quedará 100% offline desde
   antes de instalar, bajarlas en la oficina, llevarlas en el pendrive y
   cargarlas ANTES de correr el instalador:

   ```bash
   # oficina (con internet):
   docker save postgres:16-alpine    | gzip > postgres-16-alpine.tar.gz
   docker save redis:7-alpine        | gzip > redis-7-alpine.tar.gz
   docker save migrate/migrate:v4.18.1 | gzip > migrate-v4.18.1.tar.gz
   # cliente (desde el pendrive, dentro de Ubuntu):
   docker load -i /mnt/d/postgres-16-alpine.tar.gz
   docker load -i /mnt/d/redis-7-alpine.tar.gz
   docker load -i /mnt/d/migrate-v4.18.1.tar.gz
   ```

   (Si la máquina tendrá internet durante la instalación, ni siquiera eso:
   el instalador avisa qué imágenes faltan y compose las baja solo.)

5. **(Opcional) Capar la RAM de WSL** en laptops de 8 GB — archivo
   `C:\Users\<usuario>\.wslconfig`:

   ```ini
   [wsl2]
   memory=4GB
   ```

   (`wsl --shutdown` para aplicarlo.)

---

## 5. Fase 1 — Instalar el ERP (offline, desde el pendrive)

Todo dentro de la terminal **Ubuntu**. El bundle se trabaja en el home de
Linux (`~/`), nunca descomprimido en `/mnt/c/...` (el puente Windows-Linux
es lentísimo para muchas archivos chicos).

```bash
# 1. copiar el bundle del pendrive al home y descomprimir
cp /mnt/d/erp-release-<v>.tar.gz ~/
cd ~ && tar xzf erp-release-<v>.tar.gz
cd ~/erp-release-<v>

# 2. integridad (debe terminar sin "FAILED")
sha256sum -c SHA256SUMS

# 3. instalar (wizard interactivo; licencia autodetectada si viaja en el
#    bundle — ver guía madre §2.1)
sudo ERP_SKIP_SYSTEMD=1 ./business_management/deploy/scripts/install.sh
```

- **`ERP_SKIP_SYSTEMD=1` es a propósito**: el backup diario se programa desde
  Windows (§7), no desde systemd dentro de WSL (la distro se duerme cuando no
  la usamos y un timer ahí no es confiable). Con la variable, el instalador
  avisa "timer omitido" y sigue — es lo esperado.
- Alternativa sin preguntas (los flags de la guía madre §5). Si el bundle
  trae la licencia embebida (guía madre §2.1), NO hace falta pasar los flags
  de licencia — el instalador la detecta solo también en este modo:

  ```bash
  sudo ERP_SKIP_SYSTEMD=1 ./business_management/deploy/scripts/install.sh \
      --profile laptop --client-name "<Cliente>" --yes
  ```

  (Con la licencia FUERA del bundle: agregar `--license ~/ruta/license.json
  --license-public-key ~/ruta/licensing-public.key`.)

El instalador corre sus 12 pasos idénticos a Linux (pre-chequeos, secrets
nuevos, PostgreSQL, imágenes del bundle, migraciones, seeds, admin, licencia,
stack, smoke). La instalación vive en `/opt/erp` **dentro de WSL** — a todos
los efectos, es un servidor Linux más.

> **Si la laptop ya tenía una versión vieja del sistema corriendo en Docker
> Desktop**, el paso 1b del instalador la va a detectar (WSL y Docker Desktop
> comparten el mismo motor): pregunta reemplazar (con resguardo de su DB) /
> instalar en paralelo / cancelar — guía madre §7.1. Para traer los
> clientes/proveedores del sistema viejo después:
> [`GUIDE_MIGRACION_DATOS_VIEJOS.md`](GUIDE_MIGRACION_DATOS_VIEJOS.md) §6.

Al terminar: abrir **en el navegador de Windows** `http://localhost:18081`
(Docker Desktop reenvía el localhost de WSL a Windows — no hay que configurar
nada). Con licencia quedó activa; sin licencia arranca la evaluación de 30
días (REQ_BIPACK v2.0).

### Post-instalación (igual que la guía madre §6, lo esencial)

1. Guardar la contraseña del admin de `/opt/erp/credentials/admin-inicial.txt`
   en el gestor del cliente y borrar el archivo. Desde el Explorador de
   Windows esa ruta es `\\wsl$\Ubuntu\opt\erp\credentials\`.
2. Entrar con `admin`, cambiar la contraseña, y en *Configuración →
   Sucursales* cargar los datos reales del negocio (la instalación trae la
   sucursal MAIN placeholder).
3. Probar una venta de punta a punta antes de irse del cliente.

---

## 6. Cómo funciona por dentro (para soporte)

| Pieza | Dónde vive |
|:--|:--|
| Instalación completa | `/opt/erp` **dentro de WSL** (ext4 de la distro, rápido y con permisos Unix reales) |
| Contenedores | En la VM de Docker Desktop (WSL2); `restart: unless-stopped` los revive con Docker Desktop |
| Frontend | `http://localhost:18081` publicado en loopback de WSL y reenviado a Windows por Docker Desktop |
| Backend | `http://127.0.0.1:18080` (solo loopback; mismo contrato que Linux) |
| Backups | `/opt/erp/backups` (dentro de WSL) — ver §7 |
| Credenciales admin | `/opt/erp/credentials/` → `\\wsl$\Ubuntu\opt\erp\credentials\` desde Windows |

---

## 7. Backup diario en Windows (Programador de tareas)

El script de backup ya quedó instalado en `/opt/erp/backup.sh` (dump diario
con retención de 30 días). Lo que cambia respecto a Linux es **quién lo
despierta**: en Windows es el Programador de tareas, llamando a WSL.

1. **Programador de tareas → Crear tarea básica**: nombre "ERP backup diario".
2. **Desencadenador**: diario, a una hora con la máquina encendida (en un
   local comercial, el mediodía suele ser más realista que las 02:30).
   Marcar **"Iniciar la tarea lo antes posible si se pierde un inicio
   programado"** (lo recupera si la laptop estaba apagada).
3. **Acción**:
   - Programa: `wsl.exe`
   - Argumentos: `-d Ubuntu -u root --exec /opt/erp/backup.sh`
   (`-u root` porque `/opt/erp` es de root; `wsl.exe` despierta la distro sola,
   sin abrir terminal. Ajustar `-d` si la distro no se llama `Ubuntu`.)
4. Dejar el default **"Ejecutar solo cuando el usuario haya iniciado sesión"**:
   garantiza Docker Desktop arriba cuando el backup corra.

**Copiar los backups FUERA de la máquina** (misma regla que la guía madre §6 —
el backup en el mismo disco no salva ante robo/fuego): desde el Explorador,
`\\wsl$\Ubuntu\opt\erp\backups` → copiar a pendrive/carpeta de Windows que ya
se respalde. Prueba manual en cualquier momento:

```powershell
wsl -d Ubuntu -u root --exec /opt/erp/backup.sh
```

---

## 8. Ciclo de vida

**Reinicio / apagón**: al iniciar sesión, Docker Desktop arranca (si está
activado el autostart) y revive los contenedores solos → ERP disponible en
`http://localhost:18081` sin tocar nada. Mientras nadie inicie sesión en
Windows, el ERP no está sirviendo (es una laptop, no un server — otra razón
para no prometer multi-terminal sobre Windows).

**Actualizar** (guía madre §7, dentro de Ubuntu, con el bundle nuevo en `~/`):

```bash
cd ~/erp-release-<nueva>
sudo ERP_SKIP_SYSTEMD=1 ./business_management/deploy/scripts/install.sh \
    --profile laptop --client-name "<Cliente>" --yes
```

Se puede correr **con el stack levantado**: el instalador detecta que los
puertos los retiene el propio stack (fix BE `616d551`) y compose recrea los
contenedores. Antes de actualizar: verificar el backup de anoche en
`/opt/erp/backups/`.

**Desinstalar** (guía madre §9 dentro de Ubuntu: `compose down -v` + borrar
`/opt/erp`). Borrón total de la máquina: además `wsl --unregister Ubuntu`
(borra TODO el Linux) y desinstalar Docker Desktop.

---

## 9. Problemas frecuentes

| Síntoma | Causa probable | Qué hacer |
|:--|:--|:--|
| `wsl --install` falla / la distro no arranca | Virtualización desactivada | Activar Intel VT-x/AMD-V en BIOS/UEFI + "Plataforma de máquina virtual" en características de Windows |
| `docker: command not found` dentro de Ubuntu | Docker Desktop sin integración con la distro | Docker Desktop → Settings → Resources → WSL Integration → habilitar Ubuntu |
| Instalador corta en paso 1: "rsync no instalado" | Ubuntu base sin rsync | `sudo apt install -y rsync` (con internet; si el cliente es 100% offline, instalarlo en Fase 0 con el .deb en el pendrive) |
| Paso 4/12 avisa "imágenes ausentes en el daemon" (o `No such image` más adelante) | Bundle emitido antes del 2026-10-10 en un cliente sin internet: las imágenes base no viajaban en el bundle | §4 paso 4: `docker load` desde el pendrive (postgres/redis/migrate) y re-correr el instalador (es idempotente) |
| `localhost:18081` no abre en Windows | Docker Desktop parado o stack caído | Abrir Docker Desktop; en Ubuntu: `cd /opt/erp/business_management && docker compose --profile laptop ps` |
| `[aviso] systemd no disponible / timer omitido` en la instalación | Esperado con `ERP_SKIP_SYSTEMD=1` | No es un error: el backup va por el Programador de tareas (§7) |
| Paso 1b pregunta por "stacks ERP preexistentes" | La máquina ya tenía el sistema (u otro ERP del producto) corriendo en Docker Desktop — WSL y Docker Desktop ven el MISMO motor | Decir reemplazar (guarda resguardo de la DB vieja en `/opt/erp/backups/pre-replace-*/`) o paralelo — guía madre §7.1; nunca ignora el stack viejo |
| No hubo backup a la hora programada | Laptop apagada / sin sesión iniciada | Activar "iniciar la tarea lo antes posible si se pierde un inicio"; o correr el comando manual de §7 |
| Pantalla "Licencia requerida" tras instalar | Evaluación de 30 días agotada | Igual que Linux: cargar la licencia desde esa pantalla (guía madre §8 / RUNBOOK_LICENCIAS §4b) |
| Instaló en evaluación pese a que el bundle traía la licencia | Instalador anterior al 2026-10-19 (REQ_DEPLOY v0.14) corrido con `--yes`: ignoraba la pareja embebida — o los archivos llegaron con otros nombres (`license.json`/`licensing-public.key` exactos en `business_management/`) | Instalador del bundle nuevo, o `--license`/`--license-public-key` explícitos, o cargar la licencia desde *Configuración → Licencia* / pantalla de bloqueo |
| Puertos 18080/18081/5433 ocupados | Casi imposible en Windows limpio | Overrides `ERP_BACKEND_PORT` / `ERP_FRONTEND_PORT` / `ERP_DB_PORT` (guía madre §5). Si el que retiene el puerto es el propio stack ERP, el instalador sigue solo |

---

## 10. Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v1.3 | 2026-10-19 | §5: el modo flags/`--yes` también autodetecta la licencia/clave embebidas en el bundle (REQ_DEPLOY v0.14, RF-DEPLOY-024 — antes solo el wizard las veía y un `--yes` dejaba la instalación sin `LICENSE_PUBLIC_KEY`); flags de licencia solo si viajan fuera del bundle. §9 fila nueva para instalaciones viejas/nombres equivocados | renpereiradx + agente |
| v1.2 | 2026-10-13 | §5 nota + §9 fila: el paso 1b del instalador detecta stacks ERP viejos que vivan en Docker Desktop (WSL y Docker Desktop comparten motor) y pregunta reemplazo con resguardo / paralelo / cancelar (REQ_DEPLOY v0.13, guía madre §7.1); enlace a GUIDE_MIGRACION_DATOS_VIEJOS §6 para recuperar clientes/proveedores del sistema viejo | renpereiradx + agente |
| v1.1 | 2026-10-10 | Cerrado de raíz el hallazgo de imágenes base (BE `428a6ff`): el bundle ahora trae TODAS las imágenes (postgres/redis/migrate derivadas del compose) y el instalador las carga ANTES del primer `up`. §4 paso 4 queda como receta solo para bundles viejos (+migrate), §3 y §9 actualizados | renpereiradx + agente |
| v1.0 | 2026-10-10 | Primera versión: WSL2 como camino soportado para clientes Windows (perfil laptop). Fase 0 con internet + Fase 1 offline, backup por Programador de tareas, ciclo de vida y problemas frecuentes. Incluye el hallazgo de imágenes base (postgres/redis fuera del bundle) y el fix del instalador para updates con stack vivo (BE `616d551`) | renpereiradx + agente |
