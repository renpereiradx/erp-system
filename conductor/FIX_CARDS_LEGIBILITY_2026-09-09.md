# Cards de productos legibles + mapeo de stock coherente (2026-09-09)

> Pedido: mejorar la legibilidad de las cards de /catalogo y /pedidos y alinear
> el stock de base/variantes con cómo lo mapea la página de productos.
> Commits: backend `08d0ea0`, frontend `db5860d` (branch dev).

## Qué se arregló

**Legibilidad (captura del usuario):** las filas de variante apretaban nombre,
SKU, precio y stock en una sola línea — nombres truncados a la mitad ("CAMISET
ADIS..."), SKUs partidos ("CC2Y5J-VERDE-/XL") y el stock encimado. Ahora:

- Filas de variante a dos líneas: nombre (bold, con tooltip) + SKU mono abajo a
  la izquierda; precio + chip de stock a la derecha, columnas que no se doblan.
- Chip de stock: verde con cantidad, **"Sin stock"** en rojo cuando es 0 (antes
  "STOCK: 0" confundía con dato faltante).
- La fila P.V.P./stock ahora hace wrap en cards angostas (antes recortaba el
  chip en el borde).

**Mapeo de stock (referencia: ProductDetailsModal de /productos):** el admin
muestra "Total / Base (sin variantes) / En variantes". Las cards lo replicaban
mal porque mezclaban alcances: el `stock_quantity` del buscador es GLOBAL
(todas las sucursales; camisetas: 67) y las variantes enriquecidas son POR
SUCURSAL activa (NEGRO-M=1 en MAIN). Restar daba una "base" falsa (66).

**Solución backend mínima:** `GET /products/{id}/stock-summary?branch_id=`
(catalog, gate products:read del subrouter) → `{base_stock, variants_stock,
total_stock}` en UNA consulta y UN alcance. Con las camisetas reales:
global 44 base + 23 variantes = 67 ✓; MAIN: 41 + 1 = 42 ✓.

## Comportamiento final de las cards

- Chip de la tarjeta: total de la **sucursal activa** (con summary) o el
  proyectado global como fallback (backend viejo sin el endpoint — la FE cae
  en silencio, verificado en tests).
- Fila "Producto base" en pedidos: SU stock (`base_stock`), no el total que
  mezcla variantes.
- Línea de desglose en catálogo: "Base: 41 · En variantes: 1".
- Productos sin variantes: igual que siempre, sin llamadas extra.

## Estado de despliegue

- El FE ya está visible por HMR en :5173 (con fallback hasta reiniciar BE).
- **Rebuild + restart del backend** para que las cards usen el summary por
  sucursal; sin él siguen mostrando el total global (coherente, menos fino).
- Verificado E2E: endpoint con credenciales de caja y vendor en :5051
  (products:read alcanza); visual de /catalogo y picker de /pedidos como
  vendor limpio.
- Gates: BE build/test/lint ✓ · FE tsc 0, vitest 614/614, build, lint:design ✓.
