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
| [`GUIDE_PAQUETES_CLIENTE.md`](GUIDE_PAQUETES_CLIENTE.md) | Catálogo de paquetes para el cliente: Evaluación 30 días / ERP Core / Core+BI Premium, límites y cómo activar o subir de edición (REQ_BIPACK RN-07/08/09/10) | ✅ Vigente (v1.0, 2026-09-29) |
| [`RUNBOOK_LICENCIAS.md`](RUNBOOK_LICENCIAS.md) | Emisión, instalación, evaluación, bloqueo, rescate web, renovación y revocación de licencias — paso a paso para vendedor/ops (RNF-BIPACK-007) | ✅ Vigente (v2.2; movido de `conductor/` 2026-10-06) |
| [`GUIDE_INSTALACION_NUEVO_CLIENTE.md`](GUIDE_INSTALACION_NUEVO_CLIENTE.md) | Instalación/actualización/desinstalación desde el bundle con perfiles `laptop` y `server` (RF-DEPLOY-001..003, 011..014; §9 borrón y cuenta nueva) | ✅ Vigente (contra el instalador real, F5) |
| [`GUIDE_TERMINALES_LOCAL.md`](GUIDE_TERMINALES_LOCAL.md) | Alta de una terminal: navegador, URL, usuario; confianza de la CA interna (RF-DEPLOY-004/005); móviles y tablets: WiFi del local o remoto por Tailscale (RF-DEPLOY-008) | ✅ Vigente (F5; v1.3 cross-ref navegadores) |
| [`GUIDE_NAVEGADORES_CERTIFICADOS.md`](GUIDE_NAVEGADORES_CERTIFICADOS.md) | Por qué el navegador advierte en deploy (HTTPS + CA interna) y no en dev/laptop (HTTP): el POR QUÉ de la confianza de la CA, paso a paso por navegador — Chrome, Edge, Brave, Firefox (RN-DEPLOY-002) | ✅ Vigente (v1.0, 2026-10-10) |
| [`GUIDE_MIGRACION_LAPTOP_A_SERVIDOR.md`](GUIDE_MIGRACION_LAPTOP_A_SERVIDOR.md) | Traslado completo con backup etiquetado, restore, verificación y baja de la laptop (RF-DEPLOY-006/007) | ✅ Vigente (F5, recorrido E2E del plan) |
| [`GUIDE_SUCURSAL_NUEVA.md`](GUIDE_SUCURSAL_NUEVA.md) | Alta de sucursal: VPN (tailnet por cliente) + entidad + accesos + soporte vendor (RF-DEPLOY-008/009) | ✅ Vigente (F5; join Tailscale en sitio) |
| [`GUIDE_ENTORNO_DEV_WINDOWS.md`](GUIDE_ENTORNO_DEV_WINDOWS.md) | Máquina de **desarrollo** en Windows: loop nativo (:5050/:5173), Docker perfil `laptop`, recetas PG nativo/WSL | ✅ Vigente (consolidación F1) |
| [`GUIDE_HOSTING_NUBE.md`](GUIDE_HOSTING_NUBE.md) | Catálogo de opciones para alojar el ERP en la nube (gratis → pago), con consenso de la comunidad y latencias desde Paraguay | 🟡 **Propuesta** (v0.1, 2026-09-30) — evaluación de opciones; sin procedimiento implementado (ver su §10) |

---

## Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v0.6 | 2026-10-10 | Nueva `GUIDE_NAVEGADORES_CERTIFICADOS.md` (v1.0): aviso de seguridad del navegador explicado (dev HTTP vs deploy HTTPS), almacén de confianza por navegador y pasos de la CA para Chrome/Edge/Brave/Firefox; GUIDE_TERMINALES v1.3 le agrega la referencia cruzada | renpereiradx + agente |
| v0.5 | 2026-10-06 | `RUNBOOK_LICENCIAS.md` movido de `conductor/` a `docs/guides/` (regla del owner: guías vivas en docs/, planes efímeros en conductor/); refs actualizadas en GUIDE_INSTALACION y REQ_BIPACK | renpereiradx + agente |
| v0.4 | 2026-09-30 | Nueva `GUIDE_HOSTING_NUBE.md` (v0.1): catálogo de alojamiento en nube, estado Propuesta — excepción documentada a la regla de "lo implementado": es evaluación para el cliente, el procedimiento real queda pendiente en su §10 | renpereiradx + agente |
| v0.3 | 2026-09-22 | F5: las 4 guías del módulo deploy escritas contra la implementación real (F2–F4) | renpereiradx + agente |
| v0.2 | 2026-09-22 | F1: consolidación del material de deploy disperso — `GUIDE_ENTORNO_DEV_WINDOWS.md` absorbe `WINDOWS_SETUP.md` (BE), `DOCKER_WINDOWS_POSTGRES_GUIDE.md` y `DOCKER_WSL_GUIDE.md` (FE); el resto obsoleto borrado | renpereiradx + agente |
| v0.1 | 2026-09-22 | Directorio creado con índice de guías planificadas (estructura de docs/ reorganizada) | renpereiradx + agente |
