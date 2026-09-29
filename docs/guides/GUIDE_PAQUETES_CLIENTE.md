# GUIDE_PAQUETES_CLIENTE — Paquetes disponibles para el cliente

## 1. Ficha

| Campo | Valor |
|:------|:------|
| Módulo | Catálogo de paquetes / ediciones del producto |
| Código | GUIDE-PAQUETES |
| Versión | v1.0 |
| Estado | Vigente |
| Fecha | 2026-09-29 |
| Audiencia | Comercial, soporte y clientes |
| Requerimiento | [`REQ_BIPACK_LICENCIAMIENTO_BI.md`](../requirements/REQ_BIPACK_LICENCIAMIENTO_BI.md) (v2.2 — RN-07/08/09/10) |

---

## 2. Los paquetes en una tabla

| | **Evaluación** | **ERP Core** | **ERP Core + BI Premium** |
|:--|:--|:--|:--|
| **Qué es** | El sistema completo, gratis, para conocerlo antes de pagar | La compra estándar: todo el ERP operativo | La compra completa: ERP + inteligencia de negocios |
| **Duración** | **90 días** desde la primera instalación | **Perpetua** (no vence nunca) | **Perpetua** (no vence nunca) |
| **Módulos ERP** (ventas, compras, inventario, producción, fiscal/SIFEN, caja, clientes, proveedores) | ✔ Todos | ✔ Todos | ✔ Todos |
| **Módulo BI** (dashboards, analíticas de ventas/compras/inventario, rentabilidad, pronósticos, auditoría) | ✔ Todos durante la evaluación | ✘ No incluido | ✔ Todos |
| **Terminales POS por sucursal** | Sin límite | **1 terminal activa por sucursal** | Sin límite |
| **Precio** | Gratis | Ver comercial | Ver comercial |
| **Requiere archivo de licencia** | No — arranca solo | Sí (`license.json` edición Core) | Sí (`license.json` edición Core+BI) |

> El límite de terminales es por SUCURSAL: cada sucursal de una instalación Core
> opera una terminal activa. Con el pack Premium se agregan todas las que el
> negocio necesite (RN-BIPACK-10).

## 3. Cómo funciona cada paquete

### 3.1 Evaluación (sin licencia)

- Se instala el sistema y **arranca solo**, sin licencia y sin llamar al
  proveedor: todo funciona (Core + BI) durante **90 días corridos** desde la
  primera instalación.
- El reloj no se reinicia: reinstalar o borrar la licencia no devuelve días.
- Desde 30 días antes del fin aparece un aviso en pantalla con los días
  restantes.
- **Al vencer:** el sistema se bloquea por completo (no se pierde ningún dato).
  Se desbloquea al cargar la licencia comprada — ver §4.
- En Configuración → Licencia se ve el estado y los días restantes.

### 3.2 ERP Core (licencia sin BI)

- La compra estándar. Se entrega un `license.json` **perpetuo**: no vence,
  no se renueva, no hay cuotas anuales.
- Incluye todo el ERP operativo. El módulo BI queda fuera: el menú
  "Inteligencia de Negocios" no aparece.
- **Terminales:** cada sucursal puede operar **una terminal activa**. Al
  intentar registrar o emparejar una segunda terminal en la misma sucursal,
  el sistema explica que requiere el pack Premium. Desactivar una terminal
  libera el lugar para otra.

### 3.3 ERP Core + BI Premium

- La compra completa: todo lo del Core **más** el paquete de inteligencia de
  negocios (dashboards, analíticas relacionales, rentabilidad, pronósticos,
  auditoría) y **terminales sin límite** en todas las sucursales.
- También perpetuo: se paga una vez.

## 4. Cómo se activa o cambia de paquete

| Situación | Qué hacer |
|:----------|:----------|
| Compró (o evaluación vencida) | El proveedor envía el `license.json` por correo. Un administrador lo carga desde **Configuración → Licencia → "Instalar licencia…"**, o desde la pantalla de bloqueo si el sistema ya está bloqueado. Aplica al instante, sin reiniciar ni visitar el servidor. |
| Quiere subir de Core a Core+BI | Igual: se carga la nueva licencia y el módulo BI (y las terminales extra) se habilitan en caliente. |
| Cambió de computadora (laptop → servidor) | La licencia viaja con la instalación; no hay que re-emitirla (ver [`GUIDE_MIGRACION_LAPTOP_A_SERVIDOR.md`](GUIDE_MIGRACION_LAPTOP_A_SERVIDOR.md)). |
| Instalación nueva con licencia ya comprada | El instalador acepta `--license` + `--license-public-key` (ver [`GUIDE_INSTALACION_NUEVO_CLIENTE.md`](GUIDE_INSTALACION_NUEVO_CLIENTE.md) §5). |

Nadie en la instalación puede fabricar ni alterar una licencia: el archivo va
firmado digitalmente por el proveedor y el sistema lo verifica antes de
aplicarlo.

## 5. Preguntas frecuentes

- **¿La evaluación tiene alguna función recortada?** No: durante los 90 días el
  sistema es idéntico al pack Premium, terminales incluidas. Es el producto
  real, para evaluarlo de verdad.
- **¿Qué pasa si dejamos pasar los 90 días?** El sistema se bloquea, pero los
  datos quedan intactos. Al cargar la licencia comprada todo vuelve al segundo.
- **¿El Core vence algún día?** No. Las licencias compradas (Core y Premium)
  son perpetuas: no hay renovación ni suscripción.
- **¿Podemos tener 2 terminales en la sucursal principal con Core?** No: Core
  incluye 1 terminal activa por sucursal. Las terminales adicionales necesitan
  el pack Premium. Puede desactivarse una terminal para habilitar otra.
- **¿Y con terminales en varias sucursales?** El límite es por sucursal: cada
  sucursal Core tiene su propia terminal.
- **¿Se pierde el BI si cargamos una licencia Core habiendo usado el BI en la evaluación?** El
  menú BI se oculta y sus consultas dejan de responder, pero los datos
  operativos siguen intactos: el BI es lectura/analítica, no almacena
  transacciones.

## 6. Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v1.0 | 2026-09-29 | Primera versión: catálogo Evaluación / Core / Core+BI Premium con el límite de terminales por sucursal (REQ_BIPACK v2.2, RN-10) | renpereiradx + agente |
