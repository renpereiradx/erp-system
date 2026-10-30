# GUIDE_PAQUETES_CLIENTE — Paquetes disponibles para el cliente

## 1. Ficha

| Campo | Valor |
|:------|:------|
| Módulo | Catálogo de paquetes / ediciones del producto |
| Código | GUIDE-PAQUETES |
| Versión | v1.1 |
| Estado | Vigente |
| Fecha | 2026-10-15 |
| Audiencia | Comercial, soporte y clientes |
| Requerimiento | [`REQ_BIPACK_LICENCIAMIENTO_BI.md`](../requirements/REQ_BIPACK_LICENCIAMIENTO_BI.md) (v2.2 — RN-07/08/09/10) |

---

## 2. Los paquetes en una tabla

| | **Evaluación** | **ERP Core** | **ERP Core + BI Premium** |
|:--|:--|:--|:--|
| **Qué es** | El sistema completo, gratis, para conocerlo antes de pagar | La compra estándar: todo el ERP operativo | La compra completa: ERP + inteligencia de negocios |
| **Duración** | **30 días** desde la primera instalación | **Perpetua** (no vence nunca) | **Perpetua** (no vence nunca) |
| **Módulos ERP** (ventas, compras, inventario, producción, fiscal/SIFEN, caja, clientes, proveedores) | ✔ Todos | ✔ Todos | ✔ Todos |
| **Módulo BI** (dashboards, analíticas de ventas/compras/inventario, rentabilidad, pronósticos, auditoría) | ✔ Todos durante la evaluación | ✘ No incluido | ✔ Todos |
| **Sucursales activas** | **1 (la principal)** | 1 incluida; cada adicional se paga | 1 incluida; cada adicional se paga |
| **Terminales POS por sucursal** | Sin límite | **1 terminal activa por sucursal** | Sin límite |
| **Precio** | Gratis | Ver comercial | Ver comercial |
| **Requiere archivo de licencia** | No — arranca solo | Sí (`license.json` edición Core) | Sí (`license.json` edición Core+BI) |

> El límite de terminales es por SUCURSAL: cada sucursal de una instalación Core
> opera una terminal activa. Con el pack Premium se agregan todas las que el
> negocio necesite (RN-BIPACK-10).
>
> **Sucursales (v2.5, RN-BIPACK-12):** toda instalación —Evaluación, Core o
> Core+BI— opera **1 sucursal activa** (la principal). Cada sucursal adicional
> se paga y se habilita al instante con una licencia re-emitida (el cliente la
> carga desde Configuración → Licencia, sin reinstalar). La cuota es
> independiente del pack BI: existe **Core multi-sucursal** sin inteligencia
> de negocios. Desactivar una sucursal libera su lugar.

## 3. Cómo funciona cada paquete

### 3.1 Evaluación (sin licencia)

- Se instala el sistema y **arranca solo**, sin licencia y sin llamar al
  proveedor: todo funciona (Core + BI) durante **30 días corridos** desde la
  primera instalación.
- El reloj no se reinicia: reinstalar o borrar la licencia no devuelve días.
- En los últimos 10 días aparece un aviso en pantalla con los días
  restantes.
- **Sucursales:** durante la evaluación opera **1 sucursal** (la principal),
  igual que la compra base. Para probar con varias (o abrirlas en serio) se
  paga la cuota correspondiente y se carga la licencia — ver §4.
- **Al vencer:** el sistema se bloquea por completo (no se pierde ningún dato).
  Se desbloquea al cargar la licencia comprada — ver §4.
- En Configuración → Licencia se ve el estado y los días restantes.

### 3.2 ERP Core (licencia sin BI)

- La compra estándar. Se entrega un `license.json` **perpetuo**: no vence,
  no se renueva, no hay cuotas anuales.
- Incluye todo el ERP operativo. El módulo BI queda fuera: el menú
  "Inteligencia de Negocios" no aparece.
- **Terminales:** cada sucursal puede operar **una terminal activa**. Al
  intentar registrar o vincular una segunda terminal en la misma sucursal,
  el sistema explica que requiere el pack Premium. Desactivar una terminal
  libera el lugar para otra.
- **Sucursales:** incluye **1 sucursal activa** (la principal). Core
  multi-sucursal existe: cada sucursal adicional se paga y se habilita al
  instante cargando la licencia re-emitida (ver §4). Desactivar una
  sucursal libera su lugar.

### 3.3 ERP Core + BI Premium

- La compra completa: todo lo del Core **más** el paquete de inteligencia de
  negocios (dashboards, analíticas relacionales, rentabilidad, pronósticos,
  auditoría) y **terminales sin límite** en todas las sucursales.
- **Sucursales:** igual que Core — incluye **1 activa** y cada adicional se
  paga (la cuota de sucursales y el pack BI se compran por separado).
- También perpetuo: se paga una vez.

## 4. Cómo se activa o cambia de paquete

| Situación | Qué hacer |
|:----------|:----------|
| Compró (o evaluación vencida) | El proveedor envía el `license.json` por correo. Un administrador lo carga desde **Configuración → Licencia → "Instalar licencia…"**, o desde la pantalla de bloqueo si el sistema ya está bloqueado. Aplica al instante, sin reiniciar ni visitar el servidor. |
| Quiere subir de Core a Core+BI | Igual: se carga la nueva licencia y el módulo BI (y las terminales extra) se habilitan en caliente. |
| **Quiere abrir otra sucursal** | Se paga la sucursal adicional y el proveedor envía una licencia re-emitida. Un administrador la carga en **Configuración → Licencia → "Instalar licencia…"** y la nueva sucursal se puede crear al instante (sin reinstall ni visita técnica). La card de Licencia muestra _Sucursales activas: N de M_. |
| **Se muda de local** | No se paga nada extra: crear la sucursal nueva y **desactivar** la vieja (Configuración → Sucursales). La cuota cuenta solo sucursales activas, así que el lugar queda libre. Ojo: antes de desactivar, ningún usuario debe tener la vieja como su sucursal predeterminada. Las terminales de la vieja dejan de funcionar (aviso "sucursal desactivada") — vincularlas de nuevo a la nueva. |
| **No puede crear/reactivar una sucursal** (aviso amarillo en Sucursales o error _"admite N sucursal(es) activa(s)"_) | La instalación llegó a su cuota. Llamar al proveedor: o se compra otra sucursal (licencia re-emitida), o se desactiva una sucursal que ya no se usa para liberar el lugar. Nadie del lado del cliente puede subir el límite: el número viaja firmado dentro de la licencia. |
| Cambió de computadora (laptop → servidor) | La licencia viaja con la instalación; no hay que re-emitirla (ver [`GUIDE_MIGRACION_LAPTOP_A_SERVIDOR.md`](GUIDE_MIGRACION_LAPTOP_A_SERVIDOR.md)). |
| Instalación nueva con licencia ya comprada | El instalador acepta `--license` + `--license-public-key` (ver [`GUIDE_INSTALACION_NUEVO_CLIENTE.md`](GUIDE_INSTALACION_NUEVO_CLIENTE.md) §5). |

Nadie en la instalación puede fabricar ni alterar una licencia: el archivo va
firmado digitalmente por el proveedor y el sistema lo verifica antes de
aplicarlo.

## 5. Preguntas frecuentes

- **¿La evaluación tiene alguna función recortada?** No: durante los 30 días el
  sistema es idéntico al pack Premium, terminales incluidas. Es el producto
  real, para evaluarlo de verdad. **Sucursales:** la evaluación opera 1
  sucursal (la principal), igual que la compra base — las sucursales extra se
  habilitan pagando, con el sistema ya comprado o durante la evaluación
  (queda activo al instalar la licencia).
- **¿Qué pasa si dejamos pasar los 30 días?** El sistema se bloquea, pero los
  datos quedan intactos. Al cargar la licencia comprada todo vuelve al segundo.
- **¿El Core vence algún día?** No. Las licencias compradas (Core y Premium)
  son perpetuas: no hay renovación ni suscripción.
- **¿Podemos abrir una segunda sucursal sin comprar el BI?** Sí: la cuota de
  sucursales es independiente del pack (Core multi-sucursal). Cada sucursal
  adicional se paga una vez y se habilita al instante cargando la licencia
  re-emitida; sin pack BI, cada sucursal mantiene su límite de 1 terminal
  POS activa.
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

| v1.2 | 2026-10-30 | **Cuota de sucursales (REQ_BIPACK v2.5, RN-12):** toda instalación opera 1 sucursal activa; cada adicional se paga y se habilita con licencia re-emitida cargada desde la web. Core multi-sucursal sin BI. Evaluación = 1 sucursal (cierra la fuga del trial) | renpereiradx + agente |

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v1.1 | 2026-10-15 | §3.2: vocabulario "registrar o vincular" (antes "emparejar"), alineado con la UI (Configuración → Terminales / Este dispositivo) | renpereiradx + agente |
| v1.0 | 2026-09-29 | Primera versión: catálogo Evaluación / Core / Core+BI Premium con el límite de terminales por sucursal (REQ_BIPACK v2.2, RN-10) | renpereiradx + agente |
