# REQ_BIPACK — Licenciamiento: BI Pack Premium

## 1. Ficha

| Campo | Valor |
|:------|:------|
| Módulo | Licenciamiento de módulos — pack "BI Premium" |
| Código | BIPACK |
| Versión del documento | v2.4 |
| Estado | Implementado (F0-F6, 2026-09-21) · v1.1 estadísticas relacionales (2026-09-22) · v2.0 modelo de evaluación y rescate (2026-09-28) · v2.1 correcciones de auditoría (2026-09-29) · v2.2 límite de terminales sin pack (RN-10) · v2.3 evaluación acortada a 30 días (2026-10-07) · v2.4 analítica de compras agregada (PLAN_ANALITICA_COMPRAS, 2026-10-22) — trazabilidad en las tablas RF/RN |
| Fecha | 2026-10-22 (v2.4) |
| Fuentes | [conductor/PLAN_BI_PACK_PREMIUM.md](../conductor/PLAN_BI_PACK_PREMIUM.md) (plan + decisiones ADR 1-7) · [guides/RUNBOOK_LICENCIAS.md](../guides/RUNBOOK_LICENCIAS.md) (operación) |
| Implementación | Backend `9dd9942` + `146172c` · v2.0: `441046e` · Frontend `b3ab4eb` + `26178ff` · v2.0: `91b41de` · Wrapper `809b549` + v2.0 `d95489b` · v2.1 (auditoría): correcciones del 2026-09-29 |
| Audiencia | Dirección, comerciales, soporte, ingeniería |

---

## 2. Resumen ejecutivo (para todos)

El ERP System se vende en **dos ediciones**: **ERP Core** (todo el sistema de gestión:
ventas, compras, inventario, caja, pedidos, facturación electrónica SIFEN, usuarios) y
**ERP Core + BI Premium**, que agrega el módulo completo de **Inteligencia de Negocios**
(dashboards, reportes financieros, analíticas de ventas e inventario, pronósticos,
rentabilidad y auditoría).

La diferencia entre ambas la define un **archivo de licencia**: un pequeño archivo que se
instala en el servidor del cliente y que **solo el proveedor puede generar**, porque va
firmado digitalmente — como un holograma en un billete: copiarlo o editarlo lo invalida.
Si el archivo está presente y vigente, el BI funciona; si no lo está, el BI desaparece del
sistema (menú, pantallas y consultas), sin afectar absolutamente nada del resto del ERP.

Esto permite **vender el sistema sin BI a un precio menor y cobrar el BI como extra**,
agregándolo después sin reinstalar nada: alcanza con instalar la licencia, y el módulo
aparece en caliente, al segundo.

## 3. Cómo funciona — guía no técnica

### 3.1 Las dos ediciones

| | **ERP Core** | **ERP Core + BI Premium** |
|:--|:-------------|:--------------------------|
| Ventas, compras, inventario, caja, pedidos | ✅ | ✅ |
| Facturación electrónica SIFEN y sus dashboards fiscales | ✅ | ✅ |
| Usuarios, roles y permisos | ✅ | ✅ |
| Dashboards e indicadores (KPIs, alertas, tendencias) | ❌ | ✅ |
| Reportes financieros (resultados, flujo de caja, IVA, libros) | ❌ | ✅ |
| Analíticas de ventas, inventario y rentabilidad | ❌ | ✅ |
| Estadísticas relacionales producto↔cliente↔proveedor (v1.1) | ❌ | ✅ |
| Pronósticos (ventas, inventario, demanda) | ❌ | ✅ |
| Cuentas por cobrar y por pagar analíticas + auditoría | ❌ | ✅ |

> Nota: el módulo de Auditoría forma parte del pack BI. Los dashboards fiscales de SIFEN
> **no** — monitorear que la facturación electrónica funcione es parte del día a día
> operativo, no un extra analítico (ver RN-01 y RN-02).

### 3.2 La licencia es como un candado con llave única

- El proveedor genera **licencias** con una herramienta propia (`licensegen`). Cada licencia
  es un archivo que dice: para quién es (cliente), qué compró (edición y módulos) y — solo en
  los casos con plazo (p. ej. una demo) — hasta cuándo vale. **La venta normal no vence**:
  Core y pack BI se compran una vez y no caducan (RN-07).
- El archivo va **firmado digitalmente**: el proveedor firma con una clave privada que nunca
  sale de su poder — igual que la llave maestra de un candado. El sistema del cliente solo
  tiene la "serrucha" para verificar esa firma, no la llave para fabricarla.
- Consecuencias prácticas: **nadie en la instalación del cliente puede fabricarse o alterar
  una licencia** — ni el administrador del sistema. Si alguien abre el archivo y le cambia la
  fecha de vencimiento, la firma deja de coincidir y el sistema lo rechaza. Si solo lo
  "reformatea" (espacios, saltos de línea), sigue funcionando: la firma cubre el contenido.

### 3.3 Estados de la instalación (v2.0)

| Estado | Comportamiento |
|:-------|:---------------|
| **Sin licencia** | Evaluación full (Core+BI) por **30 días desde el primer arranque**. Al vencer: **bloqueo total** — solo login, `/system/license` (ver + cargar) y logout siguen vivos. El FE muestra una pantalla de "licencia requerida" con carga de licencia. |
| **Licencia Core (sin BI)** | **Perpetua** (`expires_at: null`, `-modules core`). BI apagado, ERP Core 100% operativo. |
| **Licencia Core+BI** | **Perpetua**. Todo el pack BI. |

Complementos del modelo de estados:

| Situación | Qué pasa |
|:----------|:---------|
| Evaluación por terminar | Un banner avisa durante los **30 días previos** al fin de la evaluación. La card de Configuración muestra la fecha de fin. |
| Instalación con licencia vigente | El sistema es exactamente el actual: menú BI completo según la edición comprada. |
| Licencia con plazo por vencer (demos) | Banner durante los 30 días previos; vencida, la instalación **degrada a Core** (pack BI apagado, ERP 100% operativo) — v2.1: una licencia firmada vencida nunca bloquea el ERP ni re-regala la evaluación. |
| Renovación / upgrade | Se instala la nueva licencia y el pack reaparece en caliente, al segundo, sin reiniciar ni reinstalar. |
| Revocación (borrar el archivo) | Con evaluación vigente, cae a evaluación; con la ventana agotada, bloqueo total de inmediato, incluso con sesiones abiertas. |
| Rescate web (v2.0) | Desde la pantalla de bloqueo **o Configuración → Licencia**, un usuario con el permiso `license:write` (administrador) puede **cargar el archivo de licencia** (PUT `/system/license`): el backend verifica la firma antes de persistir y el sistema se desbloquea en caliente. v2.1: también desde la card de Configuración, antes del vencimiento. |

> El bloqueo total NO borra ni oculta datos: es una compuerta de acceso. Al cargar una
> licencia válida, todo vuelve con su historia completa (mismo principio que v1.0).

### 3.4 El ciclo comercial completo

1. **Venta**: el cliente compra Core o Core+BI (precios fuera de este documento — ver RN-05).
2. **Emisión**: el proveedor genera la licencia con `licensegen` (del lado suyo, con su clave
   privada) y se la entrega al cliente.
3. **Instalación**: se copia el archivo al servidor y se activa el control
   (`LICENSE_ENFORCE=true`, una sola vez, siguiendo el runbook).
4. **Uso**: el sistema verifica la licencia en cada consulta BI. No necesita internet: todo
   es local, no hay "llamadas a casa" ni telemetría.
5. **Renovación / upgrade**: nueva licencia sobre la anterior — en caliente (solo para
   licencias con plazo; la venta normal no vence, RN-07).
6. **Baja**: retirar el archivo bloquea el BI al instante; el Core sigue igual para siempre.

### 3.5 ¿Dónde vive la licencia y quién la controla?

- Es **un archivo aparte** en el servidor (por defecto `license.json`). No está en la base de
  datos: no hay nada que un usuario con acceso al sistema pueda tocar desde la interfaz.
- La pantalla **Configuración → Licencia** muestra el estado (edición, módulos, vencimiento,
  días restantes) **solo lectura** para cualquier usuario logueado — informativa, no editable.
- Un técnico del cliente con acceso al servidor podría apagar el control general
  (`LICENSE_ENFORCE`), igual que podría tocar la base de datos: eso escapa al alcance
  (ver RNF-02 y la decisión en el plan). La firma digital cubre el caso normal: nadie puede
  *fabricar* acceso desde dentro del sistema.

---

## 4. Alcance

**Dentro:**

- Definición de las dos ediciones y del pack BI como unidad vendible.
- Mecanismo de licencia firmada: emisión, instalación, verificación, vencimiento, gracia,
  renovación y revocación — todo en caliente.
- Bloqueo de toda la superficie BI del backend (~120 consultas en 11 grupos) y ocultamiento
  de toda la superficie BI del frontend (41 rutas + menú).
- Estadísticas relacionales producto↔cliente↔proveedor (ampliación v1.1, plan aprobado):
  compradores de un producto, productos por cliente, productos por proveedor y comparación
  de precios entre proveedores.
- Entrega de la información de licencia a la sesión del usuario y pantalla/banner de estado.
- **Modelo de evaluación y rescate (v2.0):** sin licencia válida, la instalación corre en
  evaluación full (Core+BI) por 30 días desde el primer arranque (ancla persistida en
  `system_installation`); agotada, bloqueo total salvo login/logout/refresh y
  `/system/license` (ver + **carga de licencia vía PUT** = rescate web sin tocar el
  servidor). El instalador fija `LICENSE_ENFORCE=true` por defecto.

**Fuera:**

- Precios, SKUs y condiciones comerciales (decisión de negocio — RN-05).
- Sistema de pagos, facturación del proveedor, portal de clientes.
- Exclusión del código BI del binario entregado (F7 "strip de build": opcional, endurecimiento
  extra; el control runtime ya es suficiente para vender — RNF-03).
- Protección contra un técnico del cliente con acceso físico al servidor (fuera de modelo).

## 5. Actores

| Actor | Rol frente al módulo |
|:------|:---------------------|
| Proveedor | Genera, firma, entrega y revoca licencias. Único dueño de la clave privada. |
| Cliente (dirección) | Decide qué edición compra; ve estado y vencimiento en Configuración → Licencia. |
| Usuario del ERP (cualquier rol) | Ve o no ve BI según la licencia de la instalación; dentro del BI, sus permisos habituales siguen aplicando. |
| Soporte técnico | Instala/renueva licencias siguiendo el runbook; diagnostica con la pantalla de estado. |
| Sistema (backend) | Verifica la licencia en cada consulta BI y la expone en la sesión. |

## 6. Requerimientos funcionales

| ID | Requerimiento | Prioridad | Estado | Verificación |
|:---|:--------------|:----------|:-------|:-------------|
| RF-BIPACK-001 | El sistema debe comercializarse en dos ediciones: Core y Core+BI, diferenciadas exclusivamente por la presencia de una licencia vigente. | Obligatorio | Implementado | Diseño §3; smoke con/sin licencia |
| RF-BIPACK-002 | Ante una licencia inválida, ausente o vencida, el backend debe rechazar toda consulta del módulo BI con el error explícito `MODULE_NOT_LICENSED` (403). | Obligatorio | Implementado | `tests/integration/bi_licensing_test.go` (11 grupos) |
| RF-BIPACK-003 | El gate de licencia debe aplicar a todos los usuarios sin excepción de rol, incluido el administrador. | Obligatorio | Implementado | Test integración: admin sin licencia → 403 |
| RF-BIPACK-004 | La licencia debe ser un archivo firmado digitalmente (ed25519) que declare como mínimo: cliente, edición, módulos comprados y fecha de vencimiento. | Obligatorio | Implementado | `internal/licensing/service_test.go` (firma inválida rechazada) |
| RF-BIPACK-005 | La edición/reformat del archivo (sin cambiar su contenido semántico) no debe invalidar la licencia. | Obligatorio | Implementado | Test canonical-payload (reformat aceptado, edición de campos rechazada) |
| RF-BIPACK-006 | El sistema debe releer la licencia en caliente ante cualquier cambio del archivo (instalación, renovación o borrado) sin reiniciar el servicio. | Obligatorio | Implementado | Test hot-reload install/remove |
| RF-BIPACK-007 | Vencida la licencia, el pack debe desactivarse el mismo día; el sistema debe soportar una ventana de gracia configurable en días (valor por defecto: 0). | Obligatorio | Implementado | `TestBILicensing_GraceKeepsPackOn` |
| RF-BIPACK-008 | La sesión del usuario debe exponer los entitlements de la instalación (edición, módulos, vencimiento) en el login y en la consulta de perfil. | Obligatorio | Implementado | `AuthContext.entitlements.test.tsx`; contrato BE aditivo |
| RF-BIPACK-009 | El sistema debe exponer el estado de la licencia (edición, módulos, vencimiento, días restantes, modo del control) a todo usuario autenticado, sin requerir permiso especial. | Obligatorio | Implementado | `TestBILicensing_SystemLicenseIsAuthOnly` |
| RF-BIPACK-010 | Sin pack, el frontend debe ocultar por completo el grupo de menú "Inteligencia de Negocios" (sin candados ni teasers — RN-04). | Obligatorio | Implementado | `navigation.bi-pack.test.ts` (3 perfiles) |
| RF-BIPACK-011 | Sin pack, el ingreso directo a una página BI (link viejo, marcador) debe redirigir a Pedidos mostrando un aviso de módulo no incluido. | Obligatorio | Implementado | `bi-pack-routes.test.tsx` |
| RF-BIPACK-012 | Sin pack, la página de inicio debe ser Pedidos para todos los usuarios. | Obligatorio | Implementado | Test HomeRedirect (regla ADR-6) |
| RF-BIPACK-013 | Los dashboards fiscales SIFEN deben permanecer operativos sin pack, con entrada de menú propia cuando el grupo BI está oculto. | Obligatorio | Implementado | Test nav: grupo "Fiscal (SIFEN)" sin pack |
| RF-BIPACK-014 | Si la licencia deja de ser válida con sesiones abiertas, el siguiente rechazo debe refrescar el estado y expulsar al usuario de la pantalla BI. | Obligatorio | Implementado | Test dispatcher 403 + revocación (AuthContext) |
| RF-BIPACK-015 | El rechazo por licencia debe distinguirse del rechazo por falta de permisos en toda la cadena (código de error propio, orden de evaluación licencia-antes-permiso, aviso propio en la UI). | Obligatorio | Implementado | `BusinessManagementAPI.license.test.ts`; orden en `biChain` |
| RF-BIPACK-016 | El sistema debe mostrar una pantalla de solo lectura "Configuración → Licencia" con edición, módulos, vencimiento y días restantes. | Obligatorio | Implementado | `LicenseStatusPage`; smoke en vivo |
| RF-BIPACK-017 | El sistema debe avisar el vencimiento próximo con un banner global visible desde 30 días antes, y durante la ventana de gracia si la hubiera. | Obligatorio | Implementado | `LicenseBanner.test.tsx` |
| RF-BIPACK-018 | El modo demostración del frontend debe comportarse como una instalación con el pack completo. | Obligatorio | Implementado | Test demo entitlements |
| RF-BIPACK-019 | El proveedor debe contar con una herramienta (`licensegen`) para generar pares de claves y emitir licencias firmadas desde su lado. | Obligatorio | Implementado | `cmd/licensegen` + tests |
| RF-BIPACK-020 | Con el control desactivado (desarrollo), el sistema debe comportarse íntegramente como instalación con pack. | Obligatorio | Implementado | `TestBILicensing_FailOpenWithoutEnforcement` |
| RF-BIPACK-021 | El sistema debe permitir consultar qué clientes compraron un producto dado, con filtros (rango de fechas, sucursal, variante, método de pago, unidades mínimas, búsqueda) y paginación del lado del servidor. | Obligatorio | Implementado | `GET /sales-analytics/products/{id}/buyers` (BE 38d873b, fix sort 24a4c9f; FE 7d68011); integración real-DB + smoke en vivo |
| RF-BIPACK-022 | El sistema debe permitir consultar qué productos compra más un cliente dado, con filtros (fechas, sucursal, categoría, variante, búsqueda) y paginación del lado del servidor. | Obligatorio | Implementado | `GET /sales-analytics/customers/{id}/top-products` (BE 38d873b; FE 7d68011); integración real-DB + smoke |
| RF-BIPACK-023 | El sistema debe permitir consultar qué productos se compran más de un proveedor dado, con los mismos filtros y paginación que RF-BIPACK-022, sobre compras efectivamente completadas. | Obligatorio | Implementado | `GET /purchase-analytics/suppliers/{id}/top-products`, grupo nuevo gated `analytics:read` (BE 38d873b; FE 7d68011) |
| RF-BIPACK-024 | El sistema debe permitir comparar proveedores de un producto dado por precio (último, promedio ponderado y mínimo histórico), ordenable por cualquiera de ellos, considerando solo compras en moneda base e informando las exclusiones por moneda. | Obligatorio | Implementado | `GET /purchase-analytics/products/{id}/suppliers` + meta `excluded_other_currency` (BE 38d873b; FE 7d68011); banner FE y smoke 403 sin licencia |
| RF-BIPACK-025 | Sin licencia válida, la instalación debe operar en evaluación full (Core+BI) durante 30 días corridos desde el primer arranque (ventana configurable vía `LICENSE_TRIAL_DAYS`). | Obligatorio | Implementado | `TestServiceTrialWindow` + `TestBILicensing_Lifecycle` (trial → superficie BI 200); ancla `system_installation` |
| RF-BIPACK-026 | Agotada la evaluación sin licencia, el backend debe bloquear TODA ruta salvo login, refresh, logout, health/setup y `/system/license` (ver + carga), respondiendo 403 `LICENSE_EXPIRED`; el FE debe reemplazar la app completa por la pantalla de licencia requerida. | Obligatorio | Implementado | `TestRequireActiveLicense` (whitelist) + `TestBILicensing_TrialBlockAndRescue`; gate FE `LicenseRequiredGate` + dispatcher `api:license_expired` |
| RF-BIPACK-027 | El sistema debe permitir cargar una licencia desde la web (PUT `/api/v1/system/license`, **permiso `license:write`** — enmienda v2.1, antes auth-any): el candidato se verifica criptográficamente antes de persistir; inválido o **vencido** → 400 `LICENSE_INVALID` sin tocar el estado (v2.1: un vencido responde 200-falso que mantiene el bloqueo); error de persistencia (permisos/disco) → 500 `LICENSE_WRITE_FAILED`; válido → desbloqueo en caliente. La carga está disponible en la pantalla de bloqueo Y en Configuración → Licencia (v2.1). | Obligatorio | Implementado | `TestServiceInstallRescue` + `TestLicenseInstallHandler` (expired 400, WriteError 500) + `TestBILicensing_RescueRequiresLicenseWrite` + flujo E2E en `TestBILicensing_TrialBlockAndRescue` |
| RF-BIPACK-028 | El ancla del primer arranque debe persistirse en base de datos (`system_installation`, fila única sembrada por migración + baseline) y sobrevivir reinicios y actualizaciones. | Obligatorio | Implementado | Migración `20260928100000_system_installation_anchor` + sección en `baseline_schema.sql` |
| RF-BIPACK-029 | Licencia Core (sin BI): perpetua (`expires_at: null`, `-modules core`) con BI apagado; licencia Core+BI: perpetua con todo el pack. `licensegen` emite perpetuo por defecto. | Obligatorio | Implementado | `TestServiceHasModuleEnforcement` (perpetual) · runbook §2b (`-days 0` default, RN-07) |
| RF-BIPACK-030 | Sin el pack BI, la instalación debe limitar las terminales POS a **1 terminal ACTIVA por sucursal**: registrar (`POST /devices`) o mudar/reactivar una segunda terminal en la sucursal responde 403 `LICENSE_TERMINAL_LIMIT` con el mensaje comercial; emparejar (`POST /devices/pair`) una terminal que no es la primera activa de su sucursal se rechaza igual; y la operación (middleware `X-Device-ID`) de una terminal no incumbente responde 403 `LICENSE_TERMINAL_LIMIT`. Con el pack BI (Core+BI o evaluación vigente) no hay límite; desactivar una terminal libera su lugar. | Obligatorio | Implementado | `TestServiceTerminalLimit` + `TestDeviceBranchMiddleware_TerminalGate` + integración `TestDevices_TerminalLimitByEdition` (Core → 403, otra sucursal → 201, Premium en caliente → 201) |
| RF-BIPACK-031 | El sistema debe ofrecer KPIs agregados de compras del período (monto total, órdenes, orden promedio, unidades normalizadas a base, proveedores únicos) con comparación real contra la ventana previa equivalente, informando por separado las órdenes PENDIENTES (conteo y monto) y las compras COMPLETED excluidas por moneda no base. | Obligatorio | Implementado | `GET /purchase-analytics/performance(+/date-range, ?compare=true)` (PLAN_ANALITICA_COMPRAS BE 56778c0); pantalla FE Dashboard de Compras |
| RF-BIPACK-032 | El sistema debe ofrecer la serie temporal de compras con granularidad derivada del período (hoy→horaria, año→mensual, resto→diaria) y el perfil horario/semanal de las órdenes (hora pico y mejor día calculados por el backend). | Obligatorio | Implementado | `GET /purchase-analytics/trends(+dr)` · `by-hour(+dr)` · `by-day-of-week(+dr)`; pantalla FE Tendencias |
| RF-BIPACK-033 | El sistema debe desglosar las compras por categoría (participación, órdenes, unidades, producto principal) y por producto paginado y ordenable server-side (monto, unidades, nombre, última compra). | Obligatorio | Implementado | `GET /purchase-analytics/by-category(+dr)` · `by-product(+dr)` con sort validado 400 fuera de whitelist; pantalla FE Productos y Categorías |
| RF-BIPACK-034 | El sistema debe desglosar las compras por proveedor (monto, órdenes, orden promedio, última compra; paginado y ordenable server-side) y por usuario comprador. | Obligatorio | Implementado | `GET /purchase-analytics/by-supplier(+dr)` · `by-buyer(+dr)`; pantalla FE Proveedores y Compradores |
| RF-BIPACK-035 | El sistema debe comparar dos ventanas de compras (período vs su ventana previa equivalente, o dos rangos explícitos) en monto, órdenes, unidades, orden promedio y proveedores, con variaciones relativas y rangos mostrados inclusivos. | Obligatorio | Implementado | `GET /purchase-analytics/compare`; pantalla FE Comparativa; ventanas a medias son 400 (BR-4b) |

## 7. Requerimientos no funcionales

| ID | Requerimiento | Prioridad | Estado | Verificación |
|:---|:--------------|:----------|:-------|:-------------|
| RNF-BIPACK-001 | La licencia debe ser criptográficamente infalsificable: ninguna modificación del contenido debe poder validar una firma existente. | Obligatorio | Implementado | Tests ed25519 (edición/ausencia/clave incorrecta) |
| RNF-BIPACK-002 | En desarrollo el control debe fallar abierto (todo disponible) y en producción cerrado (nada disponible ante la duda), controlado por `LICENSE_ENFORCE`. | Obligatorio | Implementado | Tests fail-open/fail-closed |
| RNF-BIPACK-003 | Debe existir un único binario/build para ambas ediciones; el gating es exclusivamente de tiempo de ejecución. | Obligatorio | Implementado | Un solo build; F7 opcional no requerido |
| RNF-BIPACK-004 | La implementación no debe introducir estado de licenciamiento en DB **fuera del ancla del trial**: v2.0 enmienda esta regla y agrega exactamente una migración (`system_installation`, fila única). El archivo de licencia sigue siendo la única fuente de entitlements. | Obligatorio | Implementado (enmendada v2.0) | Migración `20260928100000` (única) · el gate sigue leyendo `license.json` |
| RNF-BIPACK-005 | El contrato con el frontend debe ser aditivo: un backend sin el módulo de licencias debe seguir funcionando con frontends nuevos y viceversa. | Obligatorio | Implementado | Campo `entitlements` omitible; FE fail-open sin dato |
| RNF-BIPACK-006 | La verificación de licencia no debe agregar costo perceptible a las consultas (verificación de metadatos del archivo, no re-verificación criptográfica por consulta). | Obligatorio | Implementado | Re-chequeo por mtime; benchmark en plan |
| RNF-BIPACK-007 | La operación del proveedor (emisión, renovación, revocación, alta de instalaciones existentes) debe estar documentada paso a paso en un runbook. | Obligatorio | Implementado | `docs/guides/RUNBOOK_LICENCIAS.md` |
| RNF-BIPACK-008 | El módulo no debe depender de conectividad a internet ni realizar reportes externos (licenciamiento local, sin telemetría). | Obligatorio | Implementado | Revisión de código: sin llamadas de red en licensing |

## 8. Reglas de negocio

| ID | Regla | Motivo | Estado |
|:---|:------|:-------|:-------|
| RN-BIPACK-01 | El pack BI incluye toda la superficie analítica: dashboards, receivables/payables, reportes financieros, analíticas de ventas/inventario, rentabilidad, pronósticos, auditoría y (desde v1.1) analíticas relacionales producto↔cliente↔proveedor. | El valor vendido es "todo el BI"; venderlo por piezas multiplica la complejidad comercial y de soporte. | Vigente |
| RN-BIPACK-02 | Los dashboards fiscales SIFEN quedan fuera del pack (siempre en Core). | Monitorear la facturación electrónica es operación diaria, no analítica premium; bloquearlo pondría en riesgo el cumplimiento fiscal del cliente. | Vigente |
| RN-BIPACK-03 | Ventana de gracia post-vencimiento: 0 días por defecto. | Cobro estricto; la gracia existe como opción negociable por cliente (configurable), no como regalo automático. | Vigente |
| RN-BIPACK-04 | Sin pack, el menú BI se oculta por completo (sin candados ni teasers). | Decisión de simplicidad; un teaser es posición comercial a futuro pero ensucia la UI diaria. | Vigente |
| RN-BIPACK-05 | Precios, SKUs y condiciones (Core / add-on BI / bundle) quedan pendientes de definición comercial, fuera del alcance del sistema. | Decisión de negocio; el sistema no condiciona el modelo de precios. | Pendiente (comercial) |
| RN-BIPACK-06 | La clave privada de firma jamás se versiona en los repositorios del producto; su custodia es exclusiva del proveedor. | La clave en el repo equivale a regalar el producto. | Vigente (verificado en auditoría 2026-09-21) |
| RN-BIPACK-07 | La venta normal es **compra única sin vencimiento**: Core perpetuo y pack BI perpetuo. `licensegen` emite perpetuo por defecto (`-days 0`); el vencimiento, la gracia y la renovación quedan como capacidad técnica para casos especiales (demo, trial, acuerdo puntual), no como modelo de negocio. | Decisión del dueño (2026-09-22): el cliente compra una vez y el producto no caduca. Sin renovaciones que gestionar ni clientes "vencidos" por accidente — el default previo de `licensegen` (365 días) era una bomba de tiempo contraria al modelo. | Vigente |
| RN-BIPACK-08 | **Evaluación de 30 días** (v2.0; acortada a 30 en v2.3): una instalación sin licencia válida opera full (Core+BI) desde el primer arranque (ancla `system_installation`, sembrada en la migración / baseline); agotada la ventana, **bloqueo total** salvo login/logout/refresh y `/system/license` (ver + carga). La licencia Core (`-modules core`) y la Core+BI son perpetuas y apagan el trial al instalarse. La evaluación NO se reinicia al instalar/quitar licencias: el reloj corre desde el primer arranque. **Enmienda v2.1:** una licencia firmada vencida **degrada a Core** (pack BI apagado, ERP operativo) — no cae al remanente de ventana ni bloquea. | Decisión del dueño (2026-09-28): el cliente prueba el sistema completo 30 días antes de pagar; el bloqueo total al vencer (no solo el BI) es el incentivo de cierre, y el rescate web evita visitas técnicas. | Vigente |
| RN-BIPACK-09 | **Rescate web** (v2.0): PUT `/api/v1/system/license` (**permiso `license:write`**, enmienda v2.1 — el GET sigue auth-only) acepta el archivo de licencia textual, lo verifica criptográficamente ANTES de persistir (inválido → 400 `LICENSE_INVALID`, estado intacto) y lo escribe atómicamente en el path del mount (compose sin `:ro`). Es el procedimiento estándar de activación post-venta: el técnico carga la licencia desde el navegador, sin SSH. | El bloqueo total deja el sistema inusable pero el servidor vivo: la puerta de rescate es lo único que necesita un cliente que acaba de pagar. La verificación previa impide que un upload basura pise una licencia buena. | Vigente |
| RN-BIPACK-10 | **Terminales por sucursal (v2.2):** sin el pack BI la instalación opera **1 terminal POS activa por sucursal**; el pack Premium (Core+BI, o la evaluación vigente) habilita terminales adicionales. El límite aplica al registro, la mudanza/reactivación, el emparejamiento y la OPERACIÓN (la terminal incumbente —la primera activa de la sucursal— sigue; las demás responden 403 `LICENSE_TERMINAL_LIMIT`). Desactivar una terminal libera su lugar. El chequeo de operación es best-effort (un fallo de conteo deja pasar, como el touch de auditoría). | Decisión del dueño (2026-09-29): las terminales adicionales son un diferencial del pack Premium; cierra el hueco de registrar varias terminales durante la evaluación y seguir operándolas con Core. | Vigente |
| RN-BIPACK-11 | **Alcance de los agregados de compras** (v2.4): los KPIs y desgloses de `/purchase-analytics/*` se computan sobre órdenes **COMPLETED en moneda base** (mismo criterio que los drill-downs RF-023/024); las PENDING viajan aparte (`pending_orders`/`pending_amount`) y las compras en otra moneda se excluyen informando el conteo (`excluded_other_currency`). La semana es rolling (últimos 7 días), igual que ventas. | Paridad con el criterio ya vendido de los drill-downs: mezclar recibidas con pendientes o sumar monedas sin convertir produciría cifras que no cierran contra ningún reporte. | Vigente |

## 9. Preguntas frecuentes

**¿La licencia comprada vence algún día? ¿Hay que pagar algo periódicamente?**
No. La compra es única: Core y pack BI no caducan (RN-07). El vencimiento solo existe si se
entrega una licencia con plazo expreso (p. ej. una demo).

**¿Y si la instalación no tiene licencia? (v2.0)**
Funciona completa — Core+BI — por 30 días desde el primer arranque (evaluación, RN-08).
Agotados, el sistema se bloquea: solo login y la pantalla de licencia quedan vivos. Nada
se pierde; al cargar la licencia comprada todo vuelve.

**¿Cómo se activa un cliente que compró después de la evaluación? (v2.0)**
Rescate web (RN-09): iniciar sesión en la instalación bloqueada y cargar el archivo
`license.json` desde la pantalla de "licencia requerida" (o Configuración → Licencia).
El backend verifica la firma y el sistema se desbloquea al instante, sin reiniciar.

**¿Si la licencia vence, se pierden los datos del BI?**
No. El BI solo se *oculta*: los datos quedan intactos en la base. Al renovar, todo reaparece
con su historia completa.

**¿Puede el administrador del cliente activar el BI por su cuenta?**
No. La licencia se verifica contra una firma que solo el proveedor puede generar, y ningún rol
(incluido el administrador) tiene bypass.

**¿El cliente necesita internet para que se valide la licencia?**
No. Todo es local: el archivo en el servidor y la clave pública embebida. No hay llamadas
externas ni telemetría.

**¿Se puede agregar el BI a una instalación Core ya instalada?**
Sí, sin reinstalar: se instala la licencia y el módulo aparece en caliente, al segundo.

**¿Cómo sé cuánto le queda a una licencia?**
Pantalla Configuración → Licencia (días restantes) y, faltando 30 días o menos, un banner
global avisa a todos los usuarios.

**¿Qué pasa si borro el archivo por accidente?**
El BI se bloquea de inmediato, pero basta con restaurar el archivo: nada se pierde y el módulo
vuelve al segundo siguiente.

**¿Comprar el BI más adelante sale igual que comprarlo desde el inicio?**
Técnicamente es idéntico (misma instalación, misma licencia). La diferencia de precio, si la
hubiera, es política comercial (RN-05), no del sistema.

## 10. Glosario

| Término | Significado |
|:--------|:------------|
| Core | La edición base del ERP: todo el sistema de gestión sin el módulo BI. |
| Pack BI / add-on | El conjunto de funciones analíticas que se vende como extra. |
| Licencia | Archivo firmado que declara qué compró el cliente y hasta cuándo vale. |
| Firma digital (ed25519) | Mecanismo que hace que solo el poseedor de la clave privada pueda generar licencias válidas; editar una licencia firmada la invalida. |
| Clave privada / pública | La privada (nunca sale del proveedor) firma; la pública (viaja en el sistema) verifica. Como la llave y el candado: se reparten candados, nunca la llave. |
| Vencimiento | Fecha hasta la cual la licencia es válida; al día siguiente el pack se desactiva. |
| Ventana de gracia | Días extra post-vencimiento en que el pack sigue activo con aviso (por defecto: 0). |
| Entitlement | "Lo que esta instalación compró" (¿tiene el pack BI: sí/no?). Distinto de un permiso, que es lo que un usuario específico puede usar. |
| Revocación en caliente | Retirar la licencia con el sistema andando: el bloqueo aplica al segundo, sin reiniciar. |
| Fail-open / fail-closed | "Ante la duda, dejar pasar" (desarrollo) / "ante la duda, bloquear" (producción). |
| `LICENSE_ENFORCE` | Interruptor que prende/apaga el control de licencias (apagado en desarrollo, prendido en producción). |
| `licensegen` | Herramienta del proveedor para generar claves y emitir licencias firmadas. |
| Runbook | Manual paso a paso de operación (emisión, instalación, renovación, revocación). |
| ADR | *Architecture Decision Record*: decisión de diseño registrada y numerada en el plan, con su justificación. |
| Evaluación (trial) | Período de 30 días desde el primer arranque en que una instalación sin licencia opera full (Core+BI). Contado por `system_installation.first_boot_at`. |
| Bloqueo total | Estado post-evaluación sin licencia: solo login/logout/refresh y `/system/license` responden; el resto devuelve 403 `LICENSE_EXPIRED`. |
| Rescate web | Carga del archivo de licencia desde el navegador (pantalla de bloqueo o PUT `/system/license`), con verificación previa de firma. |
| `system_installation` | Tabla de fila única con `first_boot_at`: el ancla del trial, sembrada por migración y baseline. |

## 11. Trazabilidad

| Artefacto | Ubicación |
|:----------|:----------|
| Plan con decisiones (ADR-1..7) | [conductor/PLAN_BI_PACK_PREMIUM.md](../conductor/PLAN_BI_PACK_PREMIUM.md) |
| Runbook de operación | [guides/RUNBOOK_LICENCIAS.md](../guides/RUNBOOK_LICENCIAS.md) |
| Implementación backend | `business_management` · `9dd9942` (F1-F2) · `146172c` (F5-F6 docs) |
| Implementación frontend | `erp-webapp` · `b3ab4eb` (F3-F4) · `26178ff` (docs) |
| Registro de cierre | wrapper `809b549` |
| Tests backend | `business_management/tests/integration/bi_licensing_test.go` · `internal/licensing/service_test.go` · `cmd/licensegen/main_test.go` |
| Tests frontend | `src/components/auth/__tests__/bi-pack-routes.test.tsx` · `src/contexts/__tests__/AuthContext.entitlements.test.tsx` · `src/layouts/main/__tests__/navigation.bi-pack.test.ts` · `src/components/license/__tests__/LicenseBanner.test.tsx` · `src/utils/entitlements.test.ts` · `src/services/__tests__/BusinessManagementAPI.license.test.ts` |
| Auditoría de implementación | Sesión 2026-09-21: gates re-ejecutados (BE build/tests/lint; FE 1079/1079, tsc 0, build, lint:design), sin claves privadas en repos |

## 12. Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--------|:------|:-------|:------|
| v2.4 | 2026-10-22 | **Analítica de compras agregada** (RF-BIPACK-031..035, RN-11): paridad del grupo de compras con ventas — KPIs con comparación real, tendencias con perfil horario/semanal, desgloses por categoría/producto/proveedor/comprador y comparativa de ventanas. 19 rutas GET nuevas bajo `/purchase-analytics` (mismo `biChain analytics:read`), anti-duplicación multi-línea y whitelists de sort desde el día uno; 5 pantallas FE (nav Compras 7 hojas). Plan: `conductor/PLAN_ANALITICA_COMPRAS_2026-10-22.md` | renpereiradx + agente |
| v2.3 | 2026-10-07 | **Evaluación acortada de 90 a 30 días** (RN-08 enmendada): decisión comercial del dueño. Default del routes bridge `90→30 días` (`routes/licensing_bridge.go`), `LICENSE_TRIAL_DAYS=30` en `.env.docker` generado por `install.sh`, umbral del aviso FE de fin de evaluación recalibrado (≤10 días — con 30 el banner sería permanente durante TODA la evaluación; licencias firmadas siguen avisando a ≤30). Instalaciones en evaluación al momento del cambio ven su ventana acortarse al vencer los 30 días desde SU primer arranque. Docs alineados: GUIDE_PAQUETES v1.1, RUNBOOK, GUIDE_INSTALACION, GUIDE_HOSTING, REQ_DEPLOY v0.10 | renpereiradx + agente |
| v2.2 | 2026-09-29 | **Límite de terminales por sucursal (RN-10, RF-030):** sin el pack BI la instalación opera 1 terminal activa por sucursal — gates en `devices.Service` (Create/Update/Pair con `CountActiveByBranch`/`HasActiveOlderInBranch`) y en `DeviceBranchMiddleware` (operación: la incumbente sigue, las demás 403 `LICENSE_TERMINAL_LIMIT`, best-effort). `licensing.Service.TerminalLimitPerBranch()` expone la allowance (0 = ilimitado; dev fail-open = ilimitado). FE: aviso comercial en Terminales sin pack BI (`devices.limit.notice`). Catálogo de paquetes para el cliente: `docs/guides/GUIDE_PAQUETES_CLIENTE.md`. | renpereiradx + agente |
| v2.1 | 2026-09-29 | **Correcciones de auditoría de la v2.0:** (1) PUT `/system/license` detrás del permiso `license:write` (migración `20260929105845`, grant solo a F2VLso) — antes cualquier usuario autenticado podía reemplazar la licencia; (2) licencia firmada vencida degrada a **Core** (RF-027/RN-08 enmendadas) — antes regalaba el trial remanente o bloqueaba todo; (3) ancla del trial: los errores transitorios de DB **fallan abierto** como trial con retry cada 30s (antes un glitch en el primer request bloqueaba el ERP hasta reiniciar el binario) + siembra lazy si falta la fila; (4) el PUT rechaza licencias ya vencidas (400 con la fecha) y responde 500 `LICENSE_WRITE_FAILED` en errores de persistencia; (5) `LICENSE_GRACE_DAYS` aplica también al trial; (6) `install.sh`: `chown 1000:1000 license/` (el rescate escribía EACCES como appuser en Docker), probe de escritura en el smoke, flip de `LICENSE_ENFORCE=true` en updates sin licencia, inyección de `LICENSE_PUBLIC_KEY` en `.env.docker` existente, pareja `--license`→`--license-public-key` obligatoria en todos los modos; (7) FE: botón "Instalar licencia…" en Configuración → Licencia, auto-recuperación del gate (sondeo 30s + botón de estado), badge dinámico, banner de fin de evaluación. | Auditoría ZCode |
| v2.0 | 2026-09-28 | **Modelo de evaluación y rescate:** sin licencia → trial full (Core+BI) 90 días desde el primer arranque (ancla `system_installation`, migración 20260928100000 + baseline); al vencer → bloqueo total con whitelist de rescate (403 `LICENSE_EXPIRED`) y pantalla FE "licencia requerida" con carga de licencia (PUT `/system/license`, verificación previa, desbloqueo en caliente); licencias Core y Core+BI perpetuas (RN-07/08/09, RF-025..029); `install.sh` fija `LICENSE_ENFORCE=true` por defecto y pide la clave pública; compose deja el mount de licencia escribible para el rescate. RNF-004 enmendada. | renpereiradx + agente |
| v1.0 | 2026-09-21 | Versión inicial: requerimientos del módulo extraídos del plan y de la implementación auditada. | Ingeniería (ZCode) |
| v1.2 | 2026-09-22 | RN-07: modelo de compra única sin vencimiento (core y pack BI perpetuos); `licensegen` cambia su default de `-days 365` a `-days 0` (perpetuo); §3 y FAQ alineados — vencimiento/gracia/renovación pasan a ser casos especiales (demo/trial). Runbook de licencias actualizado en el mismo cambio | renpereiradx + agente |
| v1.1 | 2026-09-21 | Ampliación aprobada: estadísticas relacionales producto↔cliente↔proveedor (RF-021..024, RN-01 ampliada). Implementación planificada en `conductor/PLAN_STATS_RELACIONALES_BI.md`. | Ingeniería (ZCode) |
