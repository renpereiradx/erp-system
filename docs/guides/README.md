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
| [`GUIDE_PAQUETES_CLIENTE.md`](GUIDE_PAQUETES_CLIENTE.md) | Catálogo de paquetes para el cliente: Evaluación 90 días / ERP Core / Core+BI Premium, límites y cómo activar o subir de edición (REQ_BIPACK RN-07/08/09/10) | ✅ Vigente (v1.0, 2026-09-29) |
| [`GUIDE_INSTALACION_NUEVO_CLIENTE.md`](GUIDE_INSTALACION_NUEVO_CLIENTE.md) | Instalación/actualización desde el bundle con perfiles `laptop` y `server` (RF-DEPLOY-001..003, 011..014) | ✅ Vigente (contra el instalador real, F5) |
| [`GUIDE_TERMINALES_LOCAL.md`](GUIDE_TERMINALES_LOCAL.md) | Alta de una terminal: navegador, URL, usuario; confianza de la CA interna (RF-DEPLOY-004/005); móviles y tablets: WiFi del local o remoto por Tailscale (RF-DEPLOY-008) | ✅ Vigente (F5; v1.2 móviles) |
| [`GUIDE_MIGRACION_LAPTOP_A_SERVIDOR.md`](GUIDE_MIGRACION_LAPTOP_A_SERVIDOR.md) | Traslado completo con backup etiquetado, restore, verificación y baja de la laptop (RF-DEPLOY-006/007) | ✅ Vigente (F5, recorrido E2E del plan) |
| [`GUIDE_SUCURSAL_NUEVA.md`](GUIDE_SUCURSAL_NUEVA.md) | Alta de sucursal: VPN (tailnet por cliente) + entidad + accesos + soporte vendor (RF-DEPLOY-008/009) | ✅ Vigente (F5; join Tailscale en sitio) |
| [`GUIDE_ENTORNO_DEV_WINDOWS.md`](GUIDE_ENTORNO_DEV_WINDOWS.md) | Máquina de **desarrollo** en Windows: loop nativo (:5050/:5173), Docker perfil `laptop`, recetas PG nativo/WSL | ✅ Vigente (consolidación F1) |
| [`GUIDE_HOSTING_NUBE.md`](GUIDE_HOSTING_NUBE.md) | Catálogo de opciones para alojar el ERP en la nube (gratis → pago), con consenso de la comunidad y latencias desde Paraguay | 🟡 **Propuesta** (v0.1, 2026-09-30) — evaluación de opciones; sin procedimiento implementado (ver su §10) |

---

## Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v0.4 | 2026-09-30 | Nueva `GUIDE_HOSTING_NUBE.md` (v0.1): catálogo de alojamiento en nube, estado Propuesta — excepción documentada a la regla de "lo implementado": es evaluación para el cliente, el procedimiento real queda pendiente en su §10 | renpereiradx + agente |
| v0.3 | 2026-09-22 | F5: las 4 guías del módulo deploy escritas contra la implementación real (F2–F4) | renpereiradx + agente |
| v0.2 | 2026-09-22 | F1: consolidación del material de deploy disperso — `GUIDE_ENTORNO_DEV_WINDOWS.md` absorbe `WINDOWS_SETUP.md` (BE), `DOCKER_WINDOWS_POSTGRES_GUIDE.md` y `DOCKER_WSL_GUIDE.md` (FE); el resto obsoleto borrado | renpereiradx + agente |
| v0.1 | 2026-09-22 | Directorio creado con índice de guías planificadas (estructura de docs/ reorganizada) | renpereiradx + agente |
