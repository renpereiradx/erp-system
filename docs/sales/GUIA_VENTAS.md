# GUIA_VENTAS — Qué decirle a un cliente para venderle el sistema

## 1. Ficha

| Campo | Valor |
|:------|:------|
| Documento | Guía comercial de venta (playbook) |
| Código | SALES-001 |
| Versión | v1.5 |
| Fecha | 2026-10-31 |
| Marca de producto | **CDGC** (Code Dynamic · Gestión Comercial) — el producto se nombra "CDGC" en todo material; el descriptor "Sistema de Gestión Comercial" acompaña, no reemplaza. Arena Connect pasa a **CDAC** (Code Dynamic · Arena Connect) |
| Proveedor | **Code Dynamic** — estudio de desarrollo de software (contacto: +595 995 639 582 · codedynamicdev@gmail.com) |
| Audiencia | Owner / comercial (leer antes de cada reunión) |
| Fuente de verdad funcional | `docs/requirements/` + código (todo lo listado acá existe HOY) |

---

## 2. Posicionamiento (la regla de oro)

**El producto es un "sistema de gestión comercial integral" para comercios del Paraguay.**

- ✅ Decir: "sistema de gestión", "sistema de gestión integral", "el sistema que ordena
  la operación comercial: vender, comprar, stock, facturación electrónica, reportes".
- ✅ Decir si preguntan "¿es un ERP?": no afirmar la categoría (prometer "ERP" es
  prometer contabilidad). "Es un sistema de gestión integral: todo queda integrado en
  un solo sistema. Está pensado para comercios como el suyo, sin la complejidad de un
  SAP."
- ❌ NO decir: "ERP completo", "es un ERP", "le reemplaza al contador", "tiene
  contabilidad", "tiene nómina", "maneja todo el negocio". Hoy NO tiene contabilidad
  general ni sueldos (ver §7), y "todo" invita exactamente a esa pregunta.
- Los nombres de los paquetes (Evaluación / **ERP Core** / **ERP Core + BI Premium**)
  sí llevan "ERP": son nombres de edición del licenciamiento, no promesas de alcance,
  y se mantienen.

**Por qué importa:** si vendemos "ERP completo", la primera pregunta del cliente es
"¿y la contabilidad?" y la respuesta "todavía no" mancha todo lo demás. Si vendemos
"gestión comercial integral", esa pregunta no aparece — y cuando llegue la contabilidad,
pasamos a decir "ERP completo" sin cambiar nada más.

---

## 3. Elevator pitch (30 segundos, para decir literal)

> "Es un sistema de gestión para comercios. Ventas en el mostrador que cobran en segundos,
> compras a proveedores, inventario con control por sucursal — usted sabe cuánto stock hay
> en cada local y en total, sin llamar por teléfono —, factura electrónica de SIFEN, caja y
> cuentas de clientes; y en una sola pantalla ve qué vendió, qué le conviene reponer y quién
> le debe.
> Se instala en su propia computadora — sus datos quedan en su negocio — y no paga
> mensualidad: se compra una vez. Y puede probarlo gratis por 30 días, completo, en una
> sucursal, antes de decidir."

Versión ultra-corta (10 segundos, para WhatsApp):

> "Vendé, controlá el inventario de todas tus sucursales y facturá electrónicamente desde un
> solo sistema. Instalado en tu computadora, sin mensualidad. Probalo gratis 30 días."

---

## 4. Para quién es (perfil ideal)

**Cliente ideal:**
- Comercio / retail / distribuidora en Paraguay (venta de productos, con o sin variantes: talles, colores).
- 1 a 10 sucursales, 1 a 10 cajas/terminales.
- Hoy controla en Excel, en cuadernos, o en un sistema viejo que no da más.
- Ya factura electrónicamente (o le urge empezar por SIFEN).
- Mueve guaraníes (y a veces dólares — hay multi-moneda con tasas de cambio).

**Señales de compra (calificar rápido con esto):**
1. "Tengo stock distinto en cada sucursal y no sé cuánto tengo en total."
2. "Al final del día no sé cuánto vendí ni cuánto quedó en caja."
3. "Mi sistema no hace factura electrónica / me cobran por comprobante."
4. "No sé quién me debe hasta que llamo cliente por cliente."
5. "Vendo por WhatsApp y por el local y se me cruza todo."

Si el prospecto busca contabilidad como requisito principal → no es cliente HOY
(ver §7 cómo responder igual sin cerrar la puerta).

---

## 5. Qué vender: los módulos en lenguaje de cliente

Cada fila: **qué decir** (beneficio) → qué es en realidad (para que usted se ubique).

| Qué decir al cliente | Qué es por dentro |
|:--|:--|
| "Cobrá en segundos, incluso en hora pico: el ticket sale impreso y el saldo queda en la cuenta del cliente. También pedidos de mostrador, presupuestos imprimibles y reservas con agenda." | Módulo Ventas: POS con carrito, atajos de teclado, presupuestos, reservas acopladas |
| "Cuánto stock hay en cada local y en total, sin llamar por teléfono. Transferís mercadería entre sucursales con control de cada movimiento." | Inventario multi-sucursal: stock con alcance por sucursal, transferencias con permisos, movimientos |
| "Comprás a proveedores con órdenes y recibís la mercadería que actualiza el stock. Llevás lo que les debés y hasta cuánto te queda pagar." | Compras: órdenes, requisiciones, pagos a proveedores, cuentas por pagar con antigüedad |
| "Factura electrónica de SIFEN desde el mismo sistema, con los libros fiscales listos para su contador." | SIFEN: facturación electrónica, libros legales IVA compras/ventas, tasas de IVA |
| "Caja: apertura, movimientos, cierre del día. Sabés lo que entró y lo que salió, en cada sucursal." | Finanzas operativas: caja, movimientos, flujo de caja |
| "Sabés al instante quién te debe y desde cuándo, con el estado de cuenta de cada cliente." | Cuentas por cobrar: aging, perfil de crédito por cliente, estado de cuenta |
| "Si fabricás o armás kits, el sistema te descuenta los ingredientes y te da el producto terminado." | Manufactura + conversión de unidades |
| "El sistema tiene manejo de reservas con agenda: apartás mercadería para un cliente y, al concretar, ya es una venta sin re-cargar nada." | Reservas: stock reservado + agenda, acopladas a ventas (ReserveSvc/ScheduleSvc) |
| "Reportes que antes pedías a un contador: qué vendió cada vendedor, qué productos dejan más ganancia, mapa de calor de ventas." | Pack BI Premium: dashboards, drill-downs relacionales, rentabilidad, pronósticos |
| "Cada empleado ve solo lo que le corresponde: su sucursal, sus permisos, y todo queda auditado." | Usuarios, roles, sucursales, auditoría completa |

---

## 6. Los 5 diferenciales (por qué elegirnos a nosotros)

1. **Sin mensualidad.** Se compra una vez; la licencia es perpetua. No hay cuota anual
   ni renovación. (Competencia SaaS cobra todos los meses, para siempre.)
2. **Sus datos quedan en su negocio.** Se instala en su propia computadora/servidor.
   Si mañana no trabajara más con nosotros, el sistema y los datos siguen suyos.
   (Nube de terceros = sus datos en la computadora de otro.)
3. **Hecho para Paraguay.** Guaraníes, factura electrónica SIFEN nativa, IVA, libros
   fiscales, todo en español.
4. **Multi-sucursal de verdad.** Inventario, ventas, caja y permisos por sucursal, con
   transferencias controladas entre locales — no "otra copia del sistema" por local.
5. **Prueba completa de 30 días.** El sistema entero, sin recortes ni funciones
   demo, gratis. Si no le convence, no compró nada.

---

## 7. Objeciones frecuentes — respuesta lista

**"¿Es un ERP? ¿Tiene contabilidad?"**
> "Es un sistema de gestión integral: ventas, compras, stock, facturación electrónica,
> caja y reportes, todo integrado. La contabilidad con asientos la lleva su contador/a:
> el sistema le entrega los libros fiscales de IVA (compras y ventas) y los reportes de
> resultados y flujo de caja, que es lo que un contador necesita. Un módulo contable
> completo está en nuestro plan de desarrollo."

*(No prometer fecha. No decir "ya casi está".)*

**"¿Y sueldos / nómina?"**
> "Hoy no incluye nómina. Su planilla la maneja aparte y el sistema le da todo lo
> operativo del negocio."

**"¿Dónde viven mis datos? ¿Están en la nube?"**
> "En su propia computadora, dentro de su negocio. No dependen de un servidor ajeno
> ni de que nosotros estemos de pie. Si algún día quiere mirar el negocio desde su
> casa, hay una conexión privada opcional (VPN), pero los datos nunca salen del
> negocio salvo la factura electrónica, que por ley viaja a SIFEN."

**"¿Necesito internet?"**
> "Solo para la factura electrónica, que es obligación legal. La venta, el stock y
> los reportes funcionan en su red local, sin internet."

**"¿Y si se rompe la computadora?"**
> "El sistema hace su copia de respaldo en el servidor y hay una guía paso a paso
> para reponerlo o pasar a otra máquina. La licencia viaja con la instalación: no se
> paga de nuevo." *(Recomendar siempre copia externa: pendrive o nube del cliente.)*

**"¿Cuánto cuesta?"**
> "Depende del paquete: hay uno básico y uno con inteligencia de negocios incluida.
> Le preparo la propuesta según cuántas sucursales y cajas tiene." *(Pasar a §8 y
> definir precio en llamada/visita posterior. No mandar precio por escrito sin
> propuesta formal.)*

**"Estoy con Excel / otro sistema, ¿cómo migro?"**
> "Le mostramos el funcionamiento del sistema de punta a punta. La carga de sus datos
> —productos, precios y stock inicial— la hace usted con su gente; si prefiere delegarla,
> el soporte de carga de inventario se cotiza aparte." *(Nunca prometer carga inicial
> incluida: es un servicio aparte con precio.)*

**"¿Y si no me gusta?"**
> "Lo prueba 30 días gratis, completo. Si a mitad de camino decide que no, sus datos
> se exportan y no debe nada."

**"¿Mi gente lo va a poder usar? No son de computadora."**
> "Sí. La caja se usa en segundos y cualquiera de su gente la opera desde el primer día,
> sin capacitación. Hay guías escritas paso a paso para no técnicos, y las terminales se
> configuran una sola vez."

---

## 8. Paquetes y licencias (lo que se puede vender hoy)

| | **Evaluación** | **ERP Core** | **ERP Core + BI Premium** |
|:--|:--|:--|:--|
| Qué es | El sistema completo para probar | La compra estándar | El paquete más completo |
| Duración | 30 días gratis | **Perpetua** | **Perpetua** |
| Sucursales | **1** (para la prueba) | **Según la licencia** — cuota declarada al emitir (REQ_BIPACK v2.5) | **Según la licencia** — cuota declarada al emitir |
| Módulos de gestión | Todos | Todos | Todos |
| BI (dashboards, analítica) | Sí, durante la prueba | No | Sí |
| Terminales POS | Sin límite | 1 activa por sucursal | Sin límite |
| Precio | Gratis | *definir* | *definir* |

Cómo se entrega (para contarlo con seguridad):
- La evaluación arranca sola al instalar: sin licencia, sin llamar a nadie.
- Al comprar se envía un archivo de licencia firmado digitalmente; se carga desde
  Configuración → Licencia y aplica al instante, sin reinstalar.
- De Core a Premium se sube después cargando la nueva licencia (se habilita en caliente).
- La cuota de sucursales va dentro de la licencia firmada: una sucursal adicional = re-emisión
  con cupo mayor, cargada desde la web, sin reinstalar (REQ_BIPACK v2.5).
- La licencia viaja si cambia la máquina (laptop → servidor), no se re-emite.

*Nota interna:* precios a definir por el owner. Sugerencia de modelo: Core = precio base
según la cuota de sucursales; Premium = adicional fijo; instalación incluida en el primer
año; el soporte de carga de inventario se cotiza aparte (nunca prometerlo incluido).

---

## 9. La demo de 20 minutos (ruta de oro)

Preparación: instalación de demo con datos realistas (productos con SKU, 2 sucursales,
clientes con saldos, ventas de la semana), impresora configurada, JWT de admin a mano.

| # | Paso | Qué remarcar |
|:--|:--|:--|
| 1 | Login y elección de sucursal | "Cada sucursal ve SU inventario; el dueño ve todo." |
| 2 | Venta rápida: buscar por nombre y por código, carrito, cobrar (teclas 1-9 = formas de pago) | "Más rápido que anotar en un cuaderno." |
| 3 | Imprimir ticket con saldo pendiente | "Cuenta corriente sin planillas." |
| 4 | Presupuesto: crear, imprimir, convertir en venta | "El cliente que hoy 'lo pienso' queda registrado." |
| 5 | Inventario: ver por sucursal + transferir | "Sé qué hay en cada local, sin llamar por teléfono." |
| 6 | Compra a proveedor y recepción | "Recibo mercadería y el stock se actualiza solo." |
| 7 | Caja: apertura y cierre con diferencia | "Cierra caja en 2 minutos, no en 40." |
| 8 | Cliente moroso: aging + estado de cuenta | "Quién le debe y desde cuándo, en una pantalla." |
| 9 | BI: dashboard de ventas + drill-down hasta el detalle | "De 'vendimos más o menos' a 'este producto, en esta sucursal, esta vendedora'." |
| 10 | Configuración → Licencia (días restantes) | "Los 30 días corren desde que instala. El sistema completo." |

**Qué NO demostrar todavía** (limitaciones conocidas):
- Inutilización de comprobantes SIFEN (fallo conocido pendiente de fix).
- No prometer impresión desde celular (móvil = consulta/aprobación por navegador, sin imprimir).
- No prometer backup automático a nube (el respaldo es local + copia externa manual).

---

## 10. Checklist antes de la primera reunión

- [ ] Sé cuántas sucursales y cajas tiene el prospecto.
- [ ] Sé si ya factura electrónico (¿tiene certificado SIFEN?).
- [ ] Sé con qué controla hoy (Excel / papel / sistema viejo).
- [ ] Sé si el requisito crítico es contabilidad → ajustar expectativas ANTES (§7).
- [ ] Demo con datos realistas probada de punta a punta (§9), incluida la impresora.
- [ ] Flyer PDF e imagen promocional listos para enviar después de la reunión
      (`assets/`).
- [ ] Definí el precio a ofertar ANTES de entrar (no improvisarlo frente al cliente).

---

## 11. Roadmap honesto (cómo presentarlo sin prometer)

| En desarrollo / planificado | Cómo decirlo |
|:--|:--|
| Módulo contable (asientos, balance) | "Está en nuestro plan de desarrollo" — sin fecha |
| Nómina / sueldos | "No descartado, hoy no está en el paquete" |
| Respaldos automáticos a la nube | "Mientras tanto: copia externa recomendada, guía incluida" |

Regla: el roadmap se cuenta **solo si el cliente lo pregunta**, nunca se abre con él.
Nunca atar una fecha ("este año", "el mes que viene") — vender lo que existe hoy.

---

## 12. Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v1.5 | 2026-10-31 | Marca de producto: el sistema se nombra **CDGC** (Code Dynamic · Gestión Comercial) en todo el material de venta; el descriptor "Sistema de Gestión Comercial" acompaña a la marca. Los nombres de edición (Evaluación / ERP Core / ERP Core + BI Premium) no cambian en esta versión | renpereiradx + agente |
| v1.4 | 2026-10-31 | Alineado con REQ_BIPACK v2.5: cuota de sucursales por licencia (§8 nueva fila "Sucursales": Evaluación = 1, Core/Premium según licencia; entrega por re-emisión con cupo mayor) y evaluación de 30 días en 1 sucursal aclarada en el pitch. Migración: la carga inicial NO va incluida — solo se muestra el funcionamiento del sistema; soporte de carga de inventario se cotiza aparte (§7 + nota interna §8). §7 "mi gente": respuesta directa (se opera desde el primer día, sin capacitación). §5: ventas e inventario en lenguaje de resultado — eliminados "código de barras o por nombre" y "se descuenta solo" (copy trivial) | renpereiradx + agente |
| v1.3 | 2026-10-07 | Terminología: "stock" → "inventario (stock)" en todo el material de venta (módulo del flyer ahora "Inventario (stock)"); reservas más visibles: en el listado de capacidades del flyer y §5 reformulado como "manejo de reservas con agenda"; fix acento "electrónicamente" en pitch corto | renpereiradx + agente |
| v1.2 | 2026-10-07 | Evaluación 90→30 días (REQ_BIPACK v2.3); proveedor **Code Dynamic** con contacto real en ficha; módulo Reservas como fila propia en §5 (verificado: stock reservado + agenda acopladas a ventas); flyer regenerado (30 días, contacto real, módulos Reservas y Multi-moneda) | renpereiradx + agente |
| v1.0 | 2026-10-07 | Primera versión: posicionamiento, pitch, módulos, objeciones, demo, paquetes | agente + renpereiradx |
| v1.1 | 2026-10-07 | Endurecido el no-ERP: la respuesta a "¿es un ERP?" ya no afirma la categoría; prohibido "maneja todo el negocio"; flyer regenerado con titular y pack Core sin sobre-promesa (PDF/PNG/HTML actualizados) | agente + renpereiradx |
