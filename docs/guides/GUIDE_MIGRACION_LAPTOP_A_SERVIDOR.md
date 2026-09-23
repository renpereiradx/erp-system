# GUIDE_MIGRACION_LAPTOP_A_SERVIDOR — Traslado completo de la instalación

## 1. Ficha

| Campo | Valor |
|:--|:--|
| Módulo | Migración laptop → servidor |
| Código | GUIDE-MIGRA-LAPTOP |
| Versión | v1.0 |
| Estado | Vigente (recorrido E2E verificado en el ensayo del plan: dump 2.5 MB → install `--restore` → datos completos verificados) |
| Fecha | 2026-09-22 |
| Audiencia | Implementación / soporte |
| Requerimiento | [`REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md`](../requirements/REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md) (RF-DEPLOY-006, 007, 015; RN-DEPLOY-007) |

---

## 2. La idea en una frase

Respaldar la laptop → instalar el servidor desde un bundle → restaurar el backup
en el servidor → verificar → **recién ahí** planear la baja de la laptop (que no
se toca durante toda la migración).

---

## 3. Antes de empezar

- [ ] Servidor preparado: Linux + Docker (ver `GUIDE_INSTALACION_NUEVO_CLIENTE.md` §2).
- [ ] Bundle de release en la laptop (o generarlo: `make-release-bundle.sh`).
- [ ] **Laptop con sus migraciones al día** (el dump viaja con su versión; el
      servidor aplicará el delta — no migrar en el servidor lo que falta en la laptop).
- [ ] Ventana acordada con el cliente (no operar en la laptop durante el backup).

---

## 4. Ejecutar (un solo comando, o paso a paso)

**Todo en uno** (desde la laptop, con SSH al servidor):

```bash
business_management/deploy/scripts/migrate-to-server.sh \
    --remote usuario@ip-del-servidor \
    --license ~/ruta/license.json
```

El script:

1. **Backup etiquetado** en la laptop: `erp-migration-<fecha>.sql.gz` + checksum
   (guardado en `~/erp-migration/`; verificar que el tamaño sea razonable).
2. **Bundle de release** (si no se pasó `--bundle`).
3. **Transfiere** backup + bundle al servidor (rsync/SSH).
4. **Instala en el servidor**: `install.sh --profile server --restore <backup>`
   (restore → migraciones delta → smoke; si algo falla, muere visible).

**Modo manual** (sin SSH directo): correr el script sin `--remote` — deja backup y
bundle listos e imprime los pasos del servidor; allí:

```bash
sudo ./business_management/deploy/scripts/install.sh --profile server \
    --source ~/erp-migration/bundle \
    --restore ~/erp-migration/erp-migration-<fecha>.sql.gz \
    --hostname erp.miempresa.local \
    --license ~/erp-migration/license.json \
    --yes
```

---

## 5. Verificación en el servidor (antes de tocar la laptop)

```
[ ] Login del admin funciona (el usuario viajó con el backup)
[ ] Ventas/recibos de la fecha del backup visibles (mismo total que la laptop)
[ ] Stock de 2-3 productos de control coincide
[ ] Usuarios y roles: los vendedores entran con su clave de siempre
[ ] Impresión de ticket de prueba desde una terminal
[ ] Backup del servidor activo: ls /opt/erp/backups/
[ ] CA instalada en las terminales (GUIDE_TERMINALES_LOCAL) — es una instalación nueva
[ ] /version refleja la versión del bundle
```

---

## 6. Baja de la laptop (solo después de verificar todo)

```
[ ] Terminales ya operando contra el servidor (URL nueva)
[ ] Cancelar autostart del ERP en la laptop
[ ] Detener contenedores: docker compose --profile laptop down
[ ] CONSERVAR el backup de la laptop 30 días (no borrar ~/erp-migration/)
[ ] A los 30 días sin incidentes: dar de baja la instalación local
```

---

## 7. Rollback

- **Durante la migración**: la laptop nunca se tocó — seguir operando en ella y
  descartar el intento en el servidor (reinstalar desde cero cuando se quiera).
- **Después del corte** (ya operando en el servidor y algo explota): restaurar el
  backup más reciente del servidor (`/opt/erp/backups/` o el de la laptop) con
  `install.sh --restore`, o re-apuntar las terminales a la laptop si aún está
  viva y volver a §4.

La licencia **viaja sola**: no está amarrada a hardware, no se re-emite (RF-DEPLOY-015).

---

## 8. Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v1.0 | 2026-09-22 | Primera versión contra `migrate-to-server.sh` real (con el ensayo E2E del plan) | renpereiradx + agente |
