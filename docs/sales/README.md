# docs/sales/ — Material comercial y de ventas

Este directorio contiene el material para **vender el producto**: qué decir, a quién,
qué prometer y qué no. Es el espejo comercial de `docs/requirements/` (el QUÉ hace
el sistema) y `docs/guides/` (el CÓMO se instala).

| Documento | Pregunta que responde | Audiencia |
|:----------|:----------------------|:----------|
| [`GUIA_VENTAS.md`](GUIA_VENTAS.md) | ¿QUÉ decirle a un cliente para venderle? (pitch, módulos, objeciones, demo) | Comercial / owner |
| [`assets/flyer_promocional.pdf`](assets/flyer_promocional.pdf) | ¿QUÉ enviarle? Flyer de una página para compartir/imprimir | Clientes |
| [`assets/flyer_promocional.png`](assets/flyer_promocional.png) | Ídem, imagen para WhatsApp/redes (render 2×) | Clientes |
| [`assets/flyer_promocional.html`](assets/flyer_promocional.html) | Fuente editable del flyer (cambiar contacto/colores → regenerar) | Owner / agente |

Reglas de este directorio:

1. **Solo afirmaciones verificables**: todo lo que figura acá existe HOY en el producto
   (verificado contra código/REQ). Lo que está en roadmap se dice como roadmap, sin fecha.
2. **Sin precios en los assets**: los precios se conversan, no se envían por adelantado
   (el flyer remite a contacto). Cuando se definan, viven en un archivo aparte que NO se
   comparte con clientes.
3. El material hereda el posicionamiento vigente: **sistema de gestión comercial
   integral** — la categoría "ERP" se usa en los nombres de edición (ERP Core,
   ERP Core + BI Premium) pero nunca se promete "ERP completo" (ver GUIA_VENTAS §1).
4. Idioma: español, lenguaje de cliente (sin tecnicismos internos).
