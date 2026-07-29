# Collectuncle — Dashboard Financiero

Dashboard estático (GitHub Pages) para el negocio de importación y reventa de coleccionables de Collectuncle. Sin backend: lee datos en vivo desde Google Sheets vía gviz JSONP.

**URL dashboard:** `jacobourquides.github.io/collectuncle-dashboard`
**Fuente de datos:** Google Sheets (Sheet ID: `17DzG3oSOTXosHNpp-mW97u1WHAwzf-T6`)

---

## Estructura de datos

El negocio migró de un modelo de "una fila = una orden" a un modelo de dos niveles, porque una orden puede contener varios productos de distintas categorías. Antes se forzaba una sola categoría por orden, lo que distorsionaba el análisis.

### Pestaña `Ventas` (nivel ORDEN)

Una fila por orden completa. Aquí viven los datos que aplican a toda la orden sin importar cuántos productos tenga.

Columnas clave:
- `OrderID` (A) — identificador único de la orden (valor fijo, no fórmula)
- `Fecha` (B)
- `Estado Pago` — válidos: `Venta`, `Venta - Pendiente liq.` (se aplica trim). Se excluye `Apartado`.
- `Plataforma` — Mercado Libre, FB Ads, Recompra, Venta directa
- `Precio Venta` — total de la orden
- `Cargos Plataforma`, `Envío`, `Impuestos`, `Costo Embalaje` ($36 fijo por orden) — **gastos a nivel de orden, NO se dividen por producto**
- `Ganancia Neta`, `Margen %`
- `isMulti` (columna auxiliar) — marca el tipo de orden: vacío = producto único; `isMulti_1prod` = un producto con varias piezas; `isMulti_Nprod` = varios productos distintos

### Pestaña `ITEMS_ORDENES` (nivel PRODUCTO)

Una fila por cada producto individual dentro de una orden. Permite análisis correcto por categoría.

Columnas:
- `OrderID` — vincula con Ventas (clave foránea)
- `ItemNum` — 1, 2, 3... secuencial dentro de cada orden
- `Producto`
- `Piezas`
- `Categoria` — una de: `Funko`, `TCG - Deportes`, `TCG - Animacion`, `ThrillJoy`, `Otros`
- `Fecha` — traída de Ventas vía VLOOKUP (columna temporal, se puede borrar)
- `Precio_Item` — precio de venta **por unidad** (sin envío, cargos, impuestos ni embalaje)
- `Costo_Inventario_Item` — costo de inventario **por unidad**
- `Precio_Total_Item` — fórmula `=Precio_Item * Piezas` (NO editar)
- `Costo_Total_Item` — fórmula `=Costo_Inventario_Item * Piezas` (NO editar)

---

## Reglas de negocio críticas

1. **Precio y costo son SIEMPRE por unidad** en ITEMS_ORDENES. La columna de total lo multiplica por piezas automáticamente. Nunca meter el total de línea en las columnas unitarias (infla el número al doble).

2. **Gastos a nivel de orden** (envío, cargos de plataforma, impuestos, embalaje) viven solo en `Ventas`, nunca se reparten entre productos en ITEMS_ORDENES.

3. **Categorías:** cinco válidas. `Multicategoria` fue eliminada — las órdenes con productos de distinta categoría se dividen en varias filas, cada una con su categoría real.

4. **Impuesto Texas (8.25%)** aplica sobre precio + envío, no solo precio. eBay lo cobra según dirección de destino (PO Box en Hidalgo, TX), no según estado del vendedor.

5. **ThrillJoy** se excluye intencionalmente de la comparación TCG vs Funko en los insights.

---

## Notas técnicas del dashboard

- **Carga de datos:** gviz JSONP siempre invoca `window.google.visualization.Query.setResponse()` sin importar el parámetro `responseHandler`. Esto impide cargar múltiples hojas por JSONP en paralelo (colisión de callback). Solución: carga secuencial (Gastos primero, luego Ventas, reasignando el callback en medio).
- `fetch()` sobre gviz falla por CORS tanto en local (`file://`) como en GitHub Pages. Usar el patrón JSONP.
- **FX feed:** `open.er-api.com/v6/latest/USD` — keyless, CORS abierto, confiable. (`api.frankfurter.app` resultó menos confiable.)
- **localStorage** funciona en el deploy de GitHub Pages, pero puede no persistir en entornos de preview sandboxed — comportamiento esperado, no es bug.
- **Charting:** Chart.js (inlined). **Iconos:** Tabler icons. **Tema:** dark/light con acentos dorados (`--gold`, `--teal`, `--surface`).

---

## Pendientes / roadmap

- Deducción de costo de Meta Ads del canal FB (bloqueado: requiere que Efraín divida la categoría `Publicidad` de Gastos en subcategorías Meta vs Meli). Al hacerlo, el rango de Gastos pasa a `A2:D1000`.
- Pestaña `Preventas` (aún no conectada)
- Alertas WhatsApp (Meta Cloud API o Twilio) / email (Gmail SMTP)
- Automatización de captura de ventas vía Mercado Libre API
- Migrar el archivo de `.xlsx` a Google Sheets nativo (habilita protección de rangos y mejor compatibilidad gviz)
