# REQ_DEPLOY — Distribución y despliegue on-premise (perfiles de instalación y crecimiento)

## 1. Ficha

| Campo | Valor |
|:--|:--|
| Módulo | Distribución, instalación y escalado del producto en clientes (on-premise) |
| Código | DEPLOY |
| Versión | v0.1 |
| Estado | Borrador |
| Fecha | 2026-09-22 |
| Fuentes | Apuntes del dueño (2026-09-20, cuaderno: "Distribución y mantenimiento de Software"), evaluación técnica de sesión 2026-09-22 (auditoría de compose/nginx/CORS/licencias/modelo de sucursales de ambos repos) |
| Implementación | ⏳ No implementado. Plan aprobado sin iniciar: `conductor/PLAN_DEPLOY_ESCALADO_ONPREMISE_2026-09-22.md` |

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
| RF-DEPLOY-001 | El sistema debe instalarse desde un único bundle de release versionado (SemVer) que contiene imágenes, compose, migraciones e instalador. | Obligatorio | Pendiente | Instalar un cliente nuevo desde el bundle sin pasos manuales fuera de la guía |
| RF-DEPLOY-002 | El instalador debe admitir los perfiles `laptop` y `server` seleccionables por parámetro. | Obligatorio | Pendiente | `install.sh --profile laptop` y `--profile server` producen las diferencias de exposición y TLS definidas en este REQ |
| RF-DEPLOY-003 | El perfil `laptop` debe dejar todos los servicios escuchando únicamente en `localhost` (ninguna exposición a la red). | Obligatorio | Pendiente | Desde otra máquina de la misma red, ningún puerto del sistema responde |
| RF-DEPLOY-004 | El perfil `server` debe publicar la web y la API bajo un único origen con proxy inverso `/api`. | Obligatorio | Pendiente | La terminal usa una sola URL; no hay CORS ni dirección de backend horneada en el frontend |
| RF-DEPLOY-005 | Una terminal nueva debe poder operar el sistema usando exclusivamente un navegador y la URL del servidor. | Obligatorio | Pendiente | Alta de terminal en una máquina limpia sin instalar software |
| RF-DEPLOY-006 | El sistema debe ofrecer un procedimiento guiado de migración de laptop a servidor que preserva los datos. | Obligatorio | Pendiente | Migración E2E en VM de prueba: mismos datos, licencia y usuarios antes y después |
| RF-DEPLOY-007 | El procedimiento de migración debe exigir un backup previo automatizado y validar la instalación destino con smoke tests antes de dar por terminada la migración. | Obligatorio | Pendiente | El script aborta sin backup; falla visiblemente si el smoke test no pasa |
| RF-DEPLOY-008 | Una sucursal debe poder operar el sistema conectándose al servidor central por VPN sin que ningún puerto del servidor quede expuesto a internet pública. | Obligatorio | Pendiente | Escaneo de puertos pública del servidor: solo lo que el firewall declara; sucursal opera normal por túnel |
| RF-DEPLOY-009 | El alta de una sucursal debe incluir su creación como entidad en el ERP, sus usuarios y roles, y la verificación de que opera en su sucursal y no ve ventas ajenas. | Obligatorio | Pendiente | Caso de prueba con usuario de sucursal (scoping por sucursal ya existente en el producto) |
| RF-DEPLOY-010 | El backend debe exponer la versión instalada en `GET /version` sin autenticación. | Obligatorio | Pendiente | `curl https://<servidor>/api/v1/version` (o ruta que se fije) responde la versión del build sin sesión |
| RF-DEPLOY-011 | El instalador debe ser idempotente: re-ejecutarlo sobre una instalación existente no corrompe datos ni configuración. | Obligatorio | Pendiente | Re-ejecutar `install.sh` 2 veces seguidas; smoke test verde en ambas |
| RF-DEPLOY-012 | La instalación nueva debe crear el esquema (migraciones), los roles base (seeds RBAC) y el usuario administrador inicial sin pasos manuales en la base de datos. | Obligatorio | Pendiente | Instalación limpia → login de admin funciona y los 9 roles están presentes |
| RF-DEPLOY-013 | Cada instalación debe realizar un backup automático diario de la base de datos con retención de 30 días. | Obligatorio | Pendiente | Backup presente y restaurable tras 31 días de operación |
| RF-DEPLOY-014 | El instalador debe ejecutar un smoke test automático al finalizar (health, versión, login admin) y reportar resultado. | Obligatorio | Pendiente | Instalación con error inyectado → reporte marca fallo |
| RF-DEPLOY-015 | La licencia debe poder trasladarse de una máquina a otra (laptop → servidor) sin re-emisión. | Obligatorio | Congelado (ya se comporta así) | Payload de licencia sin amarra a hardware (verificado 2026-09-22); migración la mueve tal cual |

## 7. Requerimientos no funcionales (RNF-…)

| ID | Requerimiento | Prioridad | Estado | Verificación |
|:--|:--|:--|:--|:--|
| RNF-DEPLOY-001 | El perfil `server` debe cifrar todo el tráfico navegador↔servidor con TLS. | Obligatorio | Pendiente | Certificado interno aceptado en las terminales; sin avisos de conexión insegura |
| RNF-DEPLOY-002 | PostgreSQL debe escuchar solo en el servidor (localhost o red interna de Docker); Redis no debe publicar puertos al host. | Obligatorio | Pendiente | `docker compose` perfil server sin puertos de Redis; PG inaccesible desde fuera del servidor |
| RNF-DEPLOY-003 | Debe existir una única imagen de frontend para todos los clientes (sin direcciones horneadas por cliente). | Obligatorio | Pendiente | Misma imagen FE servida en dos instalaciones con distinta IP/dominio |
| RNF-DEPLOY-004 | Los orígenes permitidos (CORS) deben ser configurables por variables de entorno, sin cambios de código. | Obligatorio | Congelado (ya se comporta así) | `CORS_ALLOWED_ORIGINS` ya leído de entorno (`main.go:271`, verificado 2026-09-22) |
| RNF-DEPLOY-005 | Los secretos (JWT, contraseñas de BD, claves de licencia) deben vivir solo en `.env` por instalación, jamás dentro de imágenes. | Obligatorio | Pendiente | Inspección de imágenes sin secretos; `.env` en `.gitignore` |
| RNF-DEPLOY-006 | El acceso administrativo por SSH al servidor debe estar restringido (firewall y/o solo vía VPN). | Deseable | Pendiente | SSH no accesible desde internet pública |
| RNF-DEPLOY-007 | Una instalación nueva en hardware preparado debe completarse en 30 minutos o menos, excluyendo descargas de imágenes. | Deseable | Pendiente | Cronómetro en la guía de validación E2E |

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
| Plan de ejecución (aprobado, no iniciado) | `conductor/PLAN_DEPLOY_ESCALADO_ONPREMISE_2026-09-22.md` |
| Evaluación de origen (hallazgos verificados en código) | Sesión 2026-09-22: compose principal con ruta de frontend rota (`../../web-project/erp-webapp`), comentario de proxy `/api` desactualizado en `docker-compose.yml`, Redis expuesto en `docker-compose.server-local.yml`, `VITE_API_URL` horneado, licencia sin amarra a hardware (`internal/licensing/domain.go`), CORS por entorno (`main.go:271`), `/health` existente y `/version` inexistente |
| Material de deploy disperso a consolidar | BE raíz: `WINDOWS_SETUP.md`, `DOCKER_WINDOWS_GUIDE.md`, `DOCKER_SETUP_COMPLETE.md`, `apply_migration_quick.ps1`, `verify-postgres-windows.ps1`; BE `docs/`: `DEPLOYMENT_DOCKER.md`, `docs/deployment/`, `docs/guides/`; FE: `DOCKER_BACKEND_PATH_GUIDE.md`, `scripts/docker-compose-*.ps1` |
| Requisito relacionado | REQ_BIPACK (licenciamiento: el bundle debe portar la licencia del cliente) |

## 12. Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v0.1 | 2026-09-22 | Borrador inicial a partir de la evaluación; decisiones D1–D6 registradas como RN-DEPLOY-001..006 | renpereiradx + agente |
