# REQ_BIPACK — Licenciamiento: BI Pack Premium

## 1. Ficha

| Campo | Valor |
|:------|:------|
| Módulo | Licenciamiento de módulos — pack "BI Premium" |
| Código | BIPACK |
| Versión del documento | v1.0 |
| Estado | Implementado (F0-F6 completos, 2026-09-21; F7 opcional no incluido) |
| Fecha | 2026-09-21 |
| Fuentes | [conductor/PLAN_BI_PACK_PREMIUM.md](../conductor/PLAN_BI_PACK_PREMIUM.md) (plan + decisiones ADR 1-7) · [conductor/RUNBOOK_LICENCIAS_2026-09.md](../conductor/RUNBOOK_LICENCIAS_2026-09.md) (operación) |
| Implementación | Backend `9dd9942` + `146172c` · Frontend `b3ab4eb` + `26178ff` · Wrapper `809b549` |
| Audiencia | Dirección, comerciales, soporte, ingeniería |

---

## 2. Resumen ejecutivo (para todos)

El ERP Vista Bar se vende en **dos ediciones**: **ERP Core** (todo el sistema de gestión:
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
| Pronósticos (ventas, inventario, demanda) | ❌ | ✅ |
| Cuentas por cobrar y por pagar analíticas + auditoría | ❌ | ✅ |

> Nota: el módulo de Auditoría forma parte del pack BI. Los dashboards fiscales de SIFEN
> **no** — monitorear que la facturación electrónica funcione es parte del día a día
> operativo, no un extra analítico (ver RN-01 y RN-02).

### 3.2 La licencia es como un candado con llave única

- El proveedor genera **licencias** con una herramienta propia (`licensegen`). Cada licencia
  es un archivo que dice: para quién es (cliente), qué compró (edición y módulos), y hasta
  cuándo vale (fecha de vencimiento).
- El archivo va **firmado digitalmente**: el proveedor firma con una clave privada que nunca
  sale de su poder — igual que la llave maestra de un candado. El sistema del cliente solo
  tiene la "serrucha" para verificar esa firma, no la llave para fabricarla.
- Consecuencias prácticas: **nadie en la instalación del cliente puede fabricarse o alterar
  una licencia** — ni el administrador del sistema. Si alguien abre el archivo y le cambia la
  fecha de vencimiento, la firma deja de coincidir y el sistema lo rechaza. Si solo lo
  "reformatea" (espacios, saltos de línea), sigue funcionando: la firma cubre el contenido.

### 3.3 Qué ve el cliente en cada caso

| Situación | Qué pasa |
|:----------|:---------|
| Instalación **sin** pack BI | El menú "Inteligencia de Negocios" directamente no existe. Si alguien entra por un link viejo a una página BI, se redirige a Pedidos con un aviso claro. La página de inicio es Pedidos. El resto del ERP funciona al 100%. |
| Instalación **con** pack vigente | El sistema es exactamente el actual: menú BI completo, todas las pantallas. |
| Licencia por vencer | Un banner avisa durante los **30 días previos** ("su licencia vence en N días"). |
| Licencia vencida | El BI se apaga ese mismo día: mismo comportamiento que no tener pack. **Ningún dato se pierde** — solo se oculta el acceso. |
| Renovación | Se instala la nueva licencia y el BI reaparece en caliente, al segundo, sin reiniciar ni reinstalar. |
| Revocación (licencia retirada) | Borrar el archivo del servidor bloquea el BI de inmediato, incluso para usuarios con la sesión abierta: a la consulta siguiente, el sistema los saca de la pantalla BI. |

### 3.4 El ciclo comercial completo

1. **Venta**: el cliente compra Core o Core+BI (precios fuera de este documento — ver RN-05).
2. **Emisión**: el proveedor genera la licencia con `licensegen` (del lado suyo, con su clave
   privada) y se la entrega al cliente.
3. **Instalación**: se copia el archivo al servidor y se activa el control
   (`LICENSE_ENFORCE=true`, una sola vez, siguiendo el runbook).
4. **Uso**: el sistema verifica la licencia en cada consulta BI. No necesita internet: todo
   es local, no hay "llamadas a casa" ni telemetría.
5. **Renovación / upgrade**: nueva licencia sobre la anterior — en caliente.
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
- Entrega de la información de licencia a la sesión del usuario y pantalla/banner de estado.

**Fuera:**

- Precios, SKUs y condiciones comerciales (decisión de negocio — RN-05).
- Sistema de pagos, facturación del proveedor, portal de clientes.
- Exclusión del código BI del binario entregado (F7 "strip de build": opcional, endurecimiento
  extra; el control runtime ya es suficiente para vender — RNF-03).
- Protección contra un técnico del cliente con acceso físico al servidor (fuera de modelo).

## 5. Actores

| Actor | Rol frente al módulo |
|:------|:---------------------|
| Proveedor (Vista Bar) | Genera, firma, entrega y revoca licencias. Único dueño de la clave privada. |
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

## 7. Requerimientos no funcionales

| ID | Requerimiento | Prioridad | Estado | Verificación |
|:---|:--------------|:----------|:-------|:-------------|
| RNF-BIPACK-001 | La licencia debe ser criptográficamente infalsificable: ninguna modificación del contenido debe poder validar una firma existente. | Obligatorio | Implementado | Tests ed25519 (edición/ausencia/clave incorrecta) |
| RNF-BIPACK-002 | En desarrollo el control debe fallar abierto (todo disponible) y en producción cerrado (nada disponible ante la duda), controlado por `LICENSE_ENFORCE`. | Obligatorio | Implementado | Tests fail-open/fail-closed |
| RNF-BIPACK-003 | Debe existir un único binario/build para ambas ediciones; el gating es exclusivamente de tiempo de ejecución. | Obligatorio | Implementado | Un solo build; F7 opcional no requerido |
| RNF-BIPACK-004 | La implementación no debe introducir migraciones de base de datos ni estado persistido en DB. | Obligatorio | Implementado | Cero migraciones nuevas |
| RNF-BIPACK-005 | El contrato con el frontend debe ser aditivo: un backend sin el módulo de licencias debe seguir funcionando con frontends nuevos y viceversa. | Obligatorio | Implementado | Campo `entitlements` omitible; FE fail-open sin dato |
| RNF-BIPACK-006 | La verificación de licencia no debe agregar costo perceptible a las consultas (verificación de metadatos del archivo, no re-verificación criptográfica por consulta). | Obligatorio | Implementado | Re-chequeo por mtime; benchmark en plan |
| RNF-BIPACK-007 | La operación del proveedor (emisión, renovación, revocación, alta de instalaciones existentes) debe estar documentada paso a paso en un runbook. | Obligatorio | Implementado | `conductor/RUNBOOK_LICENCIAS_2026-09.md` |
| RNF-BIPACK-008 | El módulo no debe depender de conectividad a internet ni realizar reportes externos (licenciamiento local, sin telemetría). | Obligatorio | Implementado | Revisión de código: sin llamadas de red en licensing |

## 8. Reglas de negocio

| ID | Regla | Motivo | Estado |
|:---|:------|:-------|:-------|
| RN-BIPACK-01 | El pack BI incluye toda la superficie analítica: dashboards, receivables/payables, reportes financieros, analíticas de ventas/inventario, rentabilidad, pronósticos y auditoría. | El valor vendido es "todo el BI"; venderlo por piezas multiplica la complejidad comercial y de soporte. | Vigente |
| RN-BIPACK-02 | Los dashboards fiscales SIFEN quedan fuera del pack (siempre en Core). | Monitorear la facturación electrónica es operación diaria, no analítica premium; bloquearlo pondría en riesgo el cumplimiento fiscal del cliente. | Vigente |
| RN-BIPACK-03 | Ventana de gracia post-vencimiento: 0 días por defecto. | Cobro estricto; la gracia existe como opción negociable por cliente (configurable), no como regalo automático. | Vigente |
| RN-BIPACK-04 | Sin pack, el menú BI se oculta por completo (sin candados ni teasers). | Decisión de simplicidad; un teaser es posición comercial a futuro pero ensucia la UI diaria. | Vigente |
| RN-BIPACK-05 | Precios, SKUs y condiciones (Core / add-on BI / bundle) quedan pendientes de definición comercial, fuera del alcance del sistema. | Decisión de negocio; el sistema no condiciona el modelo de precios. | Pendiente (comercial) |
| RN-BIPACK-06 | La clave privada de firma jamás se versiona en los repositorios del producto; su custodia es exclusiva del proveedor. | La clave en el repo equivale a regalar el producto. | Vigente (verificado en auditoría 2026-09-21) |

## 9. Preguntas frecuentes

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

## 11. Trazabilidad

| Artefacto | Ubicación |
|:----------|:----------|
| Plan con decisiones (ADR-1..7) | [conductor/PLAN_BI_PACK_PREMIUM.md](../conductor/PLAN_BI_PACK_PREMIUM.md) |
| Runbook de operación | [conductor/RUNBOOK_LICENCIAS_2026-09.md](../conductor/RUNBOOK_LICENCIAS_2026-09.md) |
| Implementación backend | `business_management` · `9dd9942` (F1-F2) · `146172c` (F5-F6 docs) |
| Implementación frontend | `erp-webapp` · `b3ab4eb` (F3-F4) · `26178ff` (docs) |
| Registro de cierre | wrapper `809b549` |
| Tests backend | `business_management/tests/integration/bi_licensing_test.go` · `internal/licensing/service_test.go` · `cmd/licensegen/main_test.go` |
| Tests frontend | `src/components/auth/__tests__/bi-pack-routes.test.tsx` · `src/contexts/__tests__/AuthContext.entitlements.test.tsx` · `src/layouts/main/__tests__/navigation.bi-pack.test.ts` · `src/components/license/__tests__/LicenseBanner.test.tsx` · `src/utils/entitlements.test.ts` · `src/services/__tests__/BusinessManagementAPI.license.test.ts` |
| Auditoría de implementación | Sesión 2026-09-21: gates re-ejecutados (BE build/tests/lint; FE 1079/1079, tsc 0, build, lint:design), sin claves privadas en repos |

## 12. Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--------|:------|:-------|:------|
| v1.0 | 2026-09-21 | Versión inicial: requerimientos del módulo extraídos del plan y de la implementación auditada. | Ingeniería (ZCode) |
