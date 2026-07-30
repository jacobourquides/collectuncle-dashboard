# Collectuncle — Dashboard Financiero

Dashboard estático (GitHub Pages) para el negocio de importación y reventa de coleccionables de Collectuncle. Sin backend: lee datos en vivo desde Google Sheets vía gviz JSONP.

**URL dashboard:** `jacobourquides.github.io/collectuncle-dashboard`
**Fuente de datos:** Google Sheets **nativo** (Sheet ID: `1lZsWSFdUYrH-9k1cZAig2PBWecX8EsRaD3Pg5ADOm50`)

> El Sheet debe estar compartido como **"Cualquiera con el enlace → Lector"**. Si está en "Restringido", gviz no puede leerlo y el dashboard cae a datos de muestra.

---

## Arquitectura de datos (modelo de dos niveles)

El negocio usa un modelo de dos niveles, porque una orden puede contener varios productos de distintas categorías. Antes se forzaba una sola categoría por orden, lo que distorsionaba el análisis por categoría.

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

---

## Cómo el dashboard combina las dos pestañas

El dashboard carga **3 fuentes** por JSONP en cadena secuencial: **Items → Gastos → Ventas** (Items primero para que `ITEMS_BY_ORDER` esté listo cuando Ventas arma el desglose).

- **Métricas de ORDEN** (KPIs, plataforma, evolución semanal, P&L): se calculan sobre `Ventas` directamente.
- **Métricas por CATEGORÍA** (donut, tarjetas de tipo, comparativa, insights ROI/margen): usan `Items_Ordenes`.

### Prorrateo de ganancia neta por categoría (Método B)

Para una orden multi-categoría, la ganancia neta (que es un dato de orden) se reparte entre las categorías de sus productos **proporcional a la ganancia bruta de cada producto** (`Precio_Total_Item − Costo_Total_Item`).

- Fallback a **Método A** (proporcional al precio) si algún producto tiene ganancia bruta ≤ 0, para evitar porcentajes negativos.
- Venta e inversión se reparten proporcional al precio/costo real de línea.
- Validado: el prorrateo conserva los totales exactamente (suma del desglose = total de la orden, diff $0). Solo 7 órdenes son multi-categoría real.

Funciones clave en el código: `processItemsData` (llena `ITEMS_BY_ORDER`), `buildCatBreakdown(r)` (arma el desglose por orden), `aggByCategory(rows)` (suma el desglose de un conjunto de órdenes).

---

## Reglas de negocio críticas

1. **Precio y costo son SIEMPRE por unidad** en las columnas G/H de Items_Ordenes. Las columnas de total (J/K) multiplican por piezas.
2. **Gastos a nivel de orden** (envío, cargos de plataforma, impuestos, embalaje) viven solo en `Ventas`, nunca se reparten entre productos en Items_Ordenes.
3. **Categorías:** cinco válidas. `Multicategoria` no es una categoría de producto — las órdenes con productos de distinta categoría se dividen en varias filas.
4. **Impuesto Texas (8.25%)** aplica sobre precio + envío, no solo precio. eBay lo cobra según dirección de destino (PO Box en Hidalgo, TX).
5. **ThrillJoy** se excluye intencionalmente de la comparación TCG vs Funko en los insights.

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

- **Carga de datos:** gviz JSONP siempre invoca `window.google.visualization.Query.setResponse()` sin importar el parámetro `responseHandler`. Esto impide cargar múltiples hojas por JSONP en paralelo (colisión de callback). Solución: carga secuencial reasignando el callback entre cada hoja (Items → Gastos → Ventas).
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

- **Sección "Piezas a liquidar":** actualmente tiene alertas hardcodeadas en el código (Trunks, Jim Halpert, etc.), no vienen del Sheet. Conectar a datos reales o quitar.
- **Deducción de Meta Ads** del canal FB (bloqueado: requiere que Efraín divida la categoría `Publicidad` de Gastos en subcategorías Meta vs Meli). Al hacerlo, Gastos pasa a `A2:D1000`.
- **Pestaña `Preventas / Apartados`** (aún no conectada). Riesgo de doble conteo: productos que aparecen tanto en tabs VIP/Preventas como en Ventas/Items. Clarificar si una preventa "se mueve" a Ventas al liquidarse o queda en ambas.
- **Consolidar tabs de clientes VIP** (Jose Chavez, Andre Cavazos, Hugo Lozano, Daniel Nolasco) en Preventas/Apartados.
- **Alertas WhatsApp** (Meta Cloud API o Twilio) / email (Gmail SMTP).
- **Automatización de captura** de ventas vía Mercado Libre API.
- **Protección de rangos** de fórmulas (G/I en Items) — el Sheet nativo ya lo permite.
- **Formato de moneda** en columna K de Items (cosmético; valores correctos, sin símbolo $).

---

## Archivos de referencia

- `index.html` — el dashboard completo (HTML/CSS/JS en un archivo, Chart.js inlined).
- Script de sync — instalado en Apps Script del Sheet nativo (Extensions → Apps Script).
- Sheet viejo `.xlsx` (ID `17DzG3o...`) — **abandonado**, solo respaldo histórico. La fuente de verdad es el Sheet nativo.
