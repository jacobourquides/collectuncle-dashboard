# Collectuncle — Dashboard Financiero

Dashboard estático (GitHub Pages) para el negocio de importación y reventa de coleccionables de Collectuncle. Sin backend: lee datos en vivo desde Google Sheets vía gviz JSONP.

**URL dashboard:** `jacobourquides.github.io/collectuncle-dashboard`
**Fuente de datos:** Google Sheets **nativo** (Sheet ID: `1lZsWSFdUYrH-9k1cZAig2PBWecX8EsRaD3Pg5ADOm50`)

> El Sheet debe estar compartido como **"Cualquiera con el enlace → Lector"**. Si está en "Restringido", gviz no puede leerlo y el dashboard cae a datos de muestra.

---

## Arquitectura de datos

El dashboard lee cuatro pestañas del Sheet: `Ventas`, `Items_Ordenes`, `Inventario` y `Gastos`.

Las dos primeras forman un **modelo de dos niveles** (orden y producto), porque una orden puede contener varios productos de distintas categorías. Antes se forzaba una sola categoría por orden, lo que distorsionaba el análisis por categoría. `Inventario` y `Gastos` son fuentes independientes: stock en existencia y egresos del negocio.

### Pestaña `Ventas` (nivel ORDEN)

Una fila por orden completa. Aquí viven los totales de la orden. Los datos empiezan en la **fila 4** (encabezados en fila 2, sub-encabezados en fila 3).

Columnas (índice gviz A4:V = 0-based):
- `OrderID` (A, idx 0) — identificador único de la orden (valor fijo, no fórmula)
- `Fecha` (B, idx 1)
- `Producto / Descripción` (C, idx 2)
- `Estado Pago` (D, idx 3) — válidos: `Venta`, `Venta - Pendiente liq.` (se aplica trim). Se excluye `Apartado`.
- `Piezas` (E, idx 4)
- `Plataforma` (F, idx 5) — Mercado Libre, FB Ads, etc.
- `Cliente` (G, idx 6)
- `Precio Venta` (H, idx 7) — total de la orden
- `Cargos Plataforma` (L, idx 11), `Envío Meli` (M, idx 12), `Impuestos` (N, idx 13) — costos Meli a nivel de orden
- `Costo Inventario` (P, idx 15)
- `Envío` (Q, idx 16)
- `Costo Embalaje` (R, idx 17) — $36 fijo por orden
- `Ganancia Neta` (S, idx 18), `Margen %` (T, idx 19)
- `Categoria` (U, idx 20) — categoría principal de la orden
- `Multiproducto` (Y, idx 24) — vacío = producto único; `IsMulti_1prod` = un producto con varias piezas; `IsMulti_Nprod` = varios productos distintos

> **IMPORTANTE:** el mapeo de índices se corrió +1 cuando se agregó la columna `OrderID` en A. El dashboard ya está corregido a esta estructura. Si se agregan/quitan columnas, hay que reajustar los índices en `processGvizData`.

### Pestaña `Items_Ordenes` (nivel PRODUCTO)

Una fila por cada producto individual dentro de una orden. Permite análisis correcto por categoría. Datos desde fila 2.

Columnas (índice gviz A2:K = 0-based):
- `OrderID` (A, idx 0) — vincula con Ventas (clave foránea)
- `ItemNum` (B, idx 1) — 1, 2, 3... secuencial dentro de cada orden
- `Producto` (C, idx 2)
- `Piezas` (D, idx 3)
- `Categoria` (E, idx 4) — una de: `Funko`, `TCG - Deportes`, `TCG - Animacion`, `ThrillJoy`, `Otros`
- `Fecha` (F, idx 5) — traída de Ventas (columna temporal)
- `Precio_Item` (G, idx 6) — precio de venta **por unidad**
- `Costo_Inventario_Item` (H, idx 7) — costo de inventario **por unidad**
- (I, idx 8) — columna vacía separadora
- `Precio_Total_Item` (J, idx 9) — `=Precio_Item * Piezas`
- `Costo_Total_Item` (K, idx 10) — `=Costo_Inventario_Item * Piezas`

> El dashboard usa **J y K (totales de línea)** para el prorrateo por categoría, no las columnas unitarias.

### Pestaña `Inventario` (nivel PRODUCTO EN STOCK)

Una fila por SKU en existencia. Alimenta toda la sección de Inventario y las alertas de liquidación. Datos desde la **fila 3**.

Columnas (índice gviz A3:N = 0-based):
- `ID` (A, idx 0) — identificador del producto; si viene vacío la fila se descarta
- `Marca` (B, idx 1) — Topps, Panini, Funko, etc.
- `Categoria` (C, idx 2) — categoría de inventario (`TCG`, `Funko`, …); **no** es la misma taxonomía que la columna `Categoria` de Ventas/Items
- `Nombre` (E, idx 4)
- `CostoUnit` (H, idx 7), `PrecioVenta` (I, idx 8), `Cantidad` (J, idx 9)
- `EstadoStock` (K, idx 10) — se normaliza por coincidencia de texto a `OK` / `BAJO` / `AGOTADO` / `OTRO`
- `ValorInventario` (L, idx 11) — capital parado en ese SKU
- `Margen` (M, idx 12)

> **Margen usa `.v`, no `.f`.** gviz devuelve `.f` como `"5%"` y `.v` como `0.05`. Leer `.f` multiplicaría el porcentaje dos veces. Es la única columna del proyecto donde se prefiere `.v` sobre `.f`.

### Pestaña `Gastos`

Rango `A2:C200`. `A` = fecha, `B` = categoría, `C` = monto.

- Se suman todas las categorías **excepto** `Compra de Inventario` → `window.GASTOS_OPS` (gastos operativos, los que restan en el P&L). El desglose queda en `window.GASTOS_DETAIL`.
- Las filas con categoría `Publicidad Meta` o `Publicidad Meli` (comparación en minúsculas, con trim) se guardan además en `window.GASTOS_ADS` como `{fecha, canal, monto}` para poder deducirlas por canal y por período.

---

## Cómo el dashboard combina las pestañas

El dashboard carga **4 fuentes** por JSONP en cadena secuencial: **Items → Inventario → Gastos → Ventas** (Items primero para que `ITEMS_BY_ORDER` esté listo cuando Ventas arma el desglose; Ventas al final porque su llegada dispara el render).

- **Métricas de ORDEN** (KPIs, plataforma, evolución semanal, P&L): se calculan sobre `Ventas` directamente.
- **Métricas por CATEGORÍA** (donut, tarjetas de tipo, comparativa, insights ROI/margen): usan `Items_Ordenes`.
- **Sección de Inventario y alertas de liquidación**: usan `Inventario`, independiente de las ventas.
- **P&L y deducción de publicidad por canal**: usan `Gastos`.

### Prorrateo de ganancia neta por categoría (Método B)

Para una orden multi-categoría, la ganancia neta (que es un dato de orden) se reparte entre las categorías de sus productos **proporcional a la ganancia bruta de cada producto** (`Precio_Total_Item − Costo_Total_Item`).

- Fallback a **Método A** (proporcional al precio) si algún producto tiene ganancia bruta ≤ 0, para evitar porcentajes negativos.
- Venta e inversión se reparten proporcional al precio/costo real de línea.
- Validado: el prorrateo conserva los totales exactamente (suma del desglose = total de la orden, diff $0). Solo 7 órdenes son multi-categoría real.

Funciones clave en el código: `processItemsData` (llena `ITEMS_BY_ORDER`), `buildCatBreakdown(r)` (arma el desglose por orden), `aggByCategory(rows)` (suma el desglose de un conjunto de órdenes).

---

## Sección de Inventario

Cuatro tarjetas alimentadas por el array `INVENTARIO`. **Todas son interactivas**: al hacer click despliegan una tabla de detalle en el contenedor `#inv-detail`, debajo de la sección. El detalle funciona como toggle — click en el mismo filtro que ya está abierto lo cierra (`INV_FILTER_ACTIVE`). La tabla muestra máximo 200 productos y avisa cuando trunca.

### A) Capital por categoría — `renderInvCapital()`

Donut del valor de inventario (columna L) agrupado por categoría, ordenado de mayor a menor. Debajo, el capital total y el total de piezas. Es la única de las cuatro que no abre detalle al hacer click.

### B) Salud del stock — `renderInvSalud()`

Barras horizontales con el conteo y porcentaje de productos en `OK` / `BAJO` / `AGOTADO`. Cada barra abre la lista de ese estado ordenada por valor descendente (`filterInvEstado`).

### C) Capital atascado — `renderInvAtascado()`

Top 10 de SKUs con **3 o más piezas** en stock, ordenados por valor de inventario — dónde está parado el dinero en un solo artículo. El título abre la lista completa de SKUs con 3+ piezas (`filterInvAtascado`).

### D) Auditor de datos — `renderInvAuditor()`

Cuenta los productos con datos incompletos o mal categorizados y los agrupa por tipo de problema. El título abre la tabla con el detalle de cada falla (`filterInvAuditor`), pensada para corregir directamente en la pestaña `Inventario` del Sheet.

`auditarInventario()` marca un producto cuando cumple alguna de estas reglas (un producto puede acumular varias fallas):

| Regla | Condición |
|---|---|
| `sin precio` | `PrecioVenta` vacío o 0 |
| `sin costo` | `CostoUnit` vacío o 0 |
| `categoría "X" (debería ser TCG)` | la marca está en `MARCAS_TCG` pero la categoría no es `TCG` |
| `N en stock sin precio` | `Cantidad > 0` y `PrecioVenta` vacío o 0 |

**`MARCAS_TCG`** — marcas que siempre deben caer en categoría `TCG`: `Topps`, `Topps NOW`, `Panini`, `Pokemon`, `Upper Deck`, `Pulse`. La comparación es exacta en minúsculas contra la columna `Marca`, así que una variante de escritura en el Sheet (`Topps Now`, `panini prizm`, un espacio de más) **no** dispara la regla. Al agregar marcas nuevas de cartas hay que sumarlas a esa constante.

### Piezas a liquidar — `renderAlerts()`

Fuera de las cuatro tarjetas, esta lista sale también de `INVENTARIO`. Candidatos: productos con stock y costo > 0 que además tienen **precio por debajo del costo** (pérdida real) o **margen menor a 5%** (crítico). Se ordenan primero las pérdidas reales por monto perdido (`(costo − precio) × cantidad`), luego los críticos por margen ascendente. Muestra los primeros 30; el título abre la lista completa (`filterInvLiquidar`).

---

## Deducción de publicidad por canal

Las categorías `Publicidad Meta` y `Publicidad Meli` de `Gastos` se descuentan del canal que les corresponde: **Meta → FB Ads**, **Meli → Mercado Libre**.

`adsDelPeriodo(canal, desde, hasta)` suma el gasto de un canal dentro del rango de fechas activo. Con el período en `all` (sin rango) suma todo. El rango no viene del selector: se deriva de la primera y última fecha de las órdenes ya filtradas, para que ventas y publicidad se recorten igual.

Afecta dos lugares:

- **Gráfica de plataformas** (`renderPlat`): la barra es `ganancia neta − publicidad del canal` (`gNeta`), y el orden de las plataformas se calcula sobre ese valor, no sobre la ganancia bruta.
- **Insights** (`renderInsights`): el margen por canal y el múltiplo "FB Ads = N× más rentable" usan el margen post-publicidad. Cuando hay gasto publicitario en el período, ambos textos lo indican con el monto descontado por canal.

> Los márgenes por canal del dashboard **no** coinciden con los de la columna `Ganancia Neta` del Sheet: esa columna no conoce la publicidad. La diferencia es intencional.

---

## Reglas de negocio críticas

1. **Precio y costo son SIEMPRE por unidad** en las columnas G/H de Items_Ordenes. Las columnas de total (J/K) multiplican por piezas.
2. **Gastos a nivel de orden** (envío, cargos de plataforma, impuestos, embalaje) viven solo en `Ventas`, nunca se reparten entre productos en Items_Ordenes.
3. **Categorías:** cinco válidas. `Multicategoria` no es una categoría de producto — las órdenes con productos de distinta categoría se dividen en varias filas.
4. **Impuesto Texas (8.25%)** aplica sobre precio + envío, no solo precio. eBay lo cobra según dirección de destino (PO Box en Hidalgo, TX).
5. **ThrillJoy** se excluye intencionalmente de la comparación TCG vs Funko en los insights.
6. **Publicidad Meta/Meli** se descuenta del canal correspondiente en la gráfica de plataformas y en los insights, pero **no** de la `Ganancia Neta` a nivel de orden — no hay forma de atribuir el gasto publicitario a una venta individual.
7. **La categoría de `Inventario` es independiente** de la de `Ventas`/`Items_Ordenes`. El auditor valida contra `TCG` (la de Inventario), no contra `TCG - Deportes` / `TCG - Animacion`.

---

## Sincronización de Items_Ordenes (Apps Script)

`Items_Ordenes` se regenera con un script de Apps Script (menú **Collectuncle → Sincronizar Items_Ordenes ahora**). Requiere que el archivo sea **Google Sheets nativo** (los .xlsx no tienen Apps Script).

Qué hace:
- Regenera las filas **automáticas** (single-product + `IsMulti_1prod`) desde Ventas, copiando la categoría de la columna U.
- **Conserva** las filas manuales `IsMulti_Nprod` (varios productos distintos, capturadas a mano).
- Recalcula precio/costo unitario = total de Ventas ÷ piezas.
- Limpia valores de error (`#REF!`, etc.) y rellena categorías vacías desde Ventas.
- Crea `Items_Backup` (snapshot pre-sync) y `Sync_Log` (bitácora de cambios de precio).

Es **manual** (botón), NO `onEdit`, para que el Sheet no se ponga lento con cada tecla.

Notas de ejecución:
- La primera vez pide autorización OAuth ("app no verificada" → Advanced → Allow; normal para scripts personales).
- El editor a veces muestra "Exceeded maximum execution time" pero el script **sí completa** (el `alert()` no cierra limpio el debugger; es cosmético). El mensaje "Sincronización lista" es la confirmación real.

---

## Notas técnicas del dashboard

- **Carga de datos:** gviz JSONP siempre invoca `window.google.visualization.Query.setResponse()` sin importar el parámetro `responseHandler`. Esto impide cargar múltiples hojas por JSONP en paralelo (colisión de callback). Solución: carga secuencial reasignando el callback entre cada hoja (Items → Inventario → Gastos → Ventas). Cada eslabón llama a `cbNext()` tanto al terminar como en `onerror`: si una hoja falla, la cadena sigue con las demás; si un eslabón nuevo se olvida de llamarlo, la cadena se cuelga y nunca carga Ventas. Hay timeouts de rescate a 9s (Ventas) y 12s (global) que disparan `useFallback()`.
- `fetch()` sobre gviz falla por CORS tanto en local (`file://`) como en GitHub Pages. Usar el patrón JSONP (inyección de `<script>`).
- **El dashboard SOLO funciona servido por HTTP (GitHub Pages), nunca abierto como archivo local `file://`** (gviz bloquea CORS desde file:// → cae a datos de muestra).
- **Conversión de texto blindada:** gviz a veces devuelve categorías u otros campos como no-string. Todas las conversiones usan `String(...??'')` antes de `.trim()` para no romper el parseo.
- **FX feed (Cotizador):** `open.er-api.com/v6/latest/USD` — keyless, CORS abierto.
- **localStorage** funciona en GitHub Pages, puede no persistir en previews sandboxed (esperado).
- **Charting:** Chart.js (inlined). **Iconos:** Tabler. **Tema:** dark/light con acentos dorados (`--gold`, `--teal`, `--surface`).

### Deploy: cómo NO romper el archivo (importante)

Al subir `index.html` a GitHub, **nunca** hacer "descargar → abrir en navegador → Guardar como". El navegador lo re-guarda como "página web completa", inyecta una línea `<script src="index_files/json.txt">` y reescribe las URLs de gviz a rutas locales — el dashboard queda roto (errores `Not allowed to load local resource` / `Cannot read properties of undefined (reading 'rows')`).

**Forma correcta:** editar `index.html` directamente en GitHub (lápiz → seleccionar todo → pegar el código nuevo → Commit), o abrir el archivo con un editor de texto (TextEdit) y copiar-pegar desde ahí. Verificar antes de commitear: buscar `index_files` (NO debe aparecer) y `docs.google.com` (SÍ debe aparecer).

---

## Pendientes / roadmap

- **Normalizar marcas del auditor:** `MARCAS_TCG` compara texto exacto, así que variantes de escritura en el Sheet se escapan de la validación. Falta normalizar (quitar acentos/espacios, comparar por prefijo) o cerrar la captura de `Marca` a una lista.
- **Pestaña `Preventas / Apartados`** (aún no conectada). Riesgo de doble conteo: productos que aparecen tanto en tabs VIP/Preventas como en Ventas/Items. Clarificar si una preventa "se mueve" a Ventas al liquidarse o queda en ambas.
- **Consolidar tabs de clientes VIP** (Jose Chavez, Andre Cavazos, Hugo Lozano, Daniel Nolasco) en Preventas/Apartados.
- **Alertas WhatsApp** (Meta Cloud API o Twilio) / email (Gmail SMTP).
- **Automatización de captura** de ventas vía Mercado Libre API.
- **Protección de rangos** de fórmulas (G/I en Items) — el Sheet nativo ya lo permite.
- **Formato de moneda** en columna K de Items (cosmético; valores correctos, sin símbolo $).

---

## Archivos de referencia

- `index.html` — el dashboard completo (HTML/CSS/JS en un archivo, Chart.js inlined).
- `CLAUDE.md` — guía de arquitectura para Claude Code (cadena de carga, estado global, trampas del parseo).
- Script de sync — instalado en Apps Script del Sheet nativo (Extensions → Apps Script).
- Sheet viejo `.xlsx` (ID `17DzG3o...`) — **abandonado**, solo respaldo histórico. La fuente de verdad es el Sheet nativo.
