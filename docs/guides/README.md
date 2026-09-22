# docs/guides/ — Guías de instalación e implementación

Guías operativas paso a paso: **el CÓMO se instala, actualiza, migra y opera** una instalación
del ERP en un cliente. Convención completa y plantilla: [`../README.md`](../README.md) (sección
"Convenciones de las guías").

Diferencia con `requirements/`: un REQ define *qué debe hacer el sistema y por qué*; una guía
define *cómo lo operás vos, con las manos*. Diferencia con `conductor/`: una guía es permanente
y se actualiza cuando cambia el procedimiento; un plan es efímero y se borra al cerrarse.

**Regla de creación:** una guía se escribe cuando el procedimiento que documenta ya existe e
implementó — documenta lo real, no lo imaginado.

---

## Índice de guías

Las guías del módulo deploy nacen del plan `conductor/PLAN_DEPLOY_ESCALADO_ONPREMISE_2026-09-22.md`
(fase F5, contra la implementación real de F2–F4).

| Guía | Qué cubre | Estado |
|:--|:--|:--|
| `GUIDE_INSTALACION_NUEVO_CLIENTE.md` | Instalación desde el bundle con perfiles `laptop` y `server` (RF-DEPLOY-001..004) | ⏳ Pendiente (F5) |
| `GUIDE_TERMINALES_LOCAL.md` | Alta de una terminal: navegador, URL, usuario; confianza de la CA interna | ⏳ Pendiente (F5) |
| `GUIDE_MIGRACION_LAPTOP_A_SERVIDOR.md` | Traslado completo con backup previo, restore y smoke test (RF-DEPLOY-006..007) | ⏳ Pendiente (F5) |
| `GUIDE_SUCURSAL_NUEVA.md` | Alta de sucursal: entidad en el ERP + VPN + usuarios/roles + verificación (RF-DEPLOY-008..009) | ⏳ Pendiente (F5) |
| [`GUIDE_ENTORNO_DEV_WINDOWS.md`](GUIDE_ENTORNO_DEV_WINDOWS.md) | Máquina de **desarrollo** en Windows: loop nativo (:5050/:5173), Docker perfil `laptop`, recetas PG nativo/WSL | ✅ Vigente (consolidación F1) |

---

## Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v0.2 | 2026-09-22 | F1: consolidación del material de deploy disperso — `GUIDE_ENTORNO_DEV_WINDOWS.md` absorbe `WINDOWS_SETUP.md` (BE), `DOCKER_WINDOWS_POSTGRES_GUIDE.md` y `DOCKER_WSL_GUIDE.md` (FE); el resto obsoleto borrado | renpereiradx + agente |
| v0.1 | 2026-09-22 | Directorio creado con índice de guías planificadas (estructura de docs/ reorganizada) | renpereiradx + agente |
