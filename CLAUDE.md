# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Qué es esto

Dashboard financiero estático de un solo archivo (`index.html`, ~1440 líneas: HTML + CSS + JS + Chart.js inlined) servido por GitHub Pages en `jacobourquides.github.io/collectuncle-dashboard`. No hay backend, build, package.json, dependencias ni tests. Los datos se leen en vivo desde un Google Sheet vía gviz JSONP.

El `README.md` documenta el modelo de datos del Sheet, las reglas de negocio y el script de Apps Script que sincroniza `Items_Ordenes`. **Léelo antes de tocar cualquier lógica de parseo o de cálculo** — este archivo no lo repite.

## Comandos

No hay build ni tests. El flujo es: editar `index.html` → servir por HTTP → commit.

```bash
python3 -m http.server 8000    # previsualizar en http://localhost:8000
```

**Nunca abras `index.html` como `file://`**: gviz bloquea el origen y el dashboard cae silenciosamente a `useFallback()` (datos de muestra hardcodeados en la línea ~700). Si ves "Datos de muestra" en el badge superior, la carga real falló.

Antes de commitear, verifica que el archivo no fue corrompido por un "Guardar como" del navegador:

```bash
grep -c index_files index.html    # debe ser 0
grep -c docs.google.com index.html # debe ser > 0
```

## Arquitectura

### Cadena de carga (crítica)

gviz JSONP **siempre** llama a `window.google.visualization.Query.setResponse()`, ignorando el parámetro `responseHandler`. Por eso no se pueden cargar varias hojas en paralelo: colisionan en el mismo callback. La solución en `loadAndRender()` (línea ~1425) es una cadena secuencial que reasigna `setResponse` antes de cada `<script>` inyectado:

```
loadItems → loadInventario → loadGastos → loadVentas
```

El orden importa: `ITEMS_BY_ORDER` debe estar poblado antes de que `processGvizData` arme el `catBreakdown` de cada orden. Si agregas una hoja, encadénala igual y añade su `cbNext()` tanto en `onerror` como en el handler; un eslabón que no llame a `cbNext()` cuelga toda la cadena. Hay timeouts de rescate a 9s (Ventas) y 12s (global) que disparan `useFallback()`.

`fetch()` no sirve aquí — gviz falla por CORS. Siempre inyección de `<script>`.

### Estado global

Cuatro variables globales alimentan todo el render; no hay framework ni estado reactivo:

- `ALL_ROWS` — órdenes de la hoja `Ventas` (nivel ORDEN), cada una con un `catBreakdown` prorrateado
- `ITEMS_BY_ORDER` — `{OrderID: [{cat, precio, costo}]}` de `Items_Ordenes` (nivel PRODUCTO)
- `INVENTARIO` — productos de la hoja `Inventario`
- `window.GASTOS_OPS` / `GASTOS_DETAIL` / `GASTOS_ADS` — de la hoja `Gastos`; `GASTOS_ADS` guarda publicidad con fecha y canal (`meta` | `meli`) para deducirla por período en `adsDelPeriodo()`

`renderAll(period)` (línea ~1320) reejecuta todos los renders de golpe. Cualquier cambio de filtro, período o tema pasa por ahí.

### Los dos niveles y el prorrateo

Una orden puede tener productos de varias categorías. Los KPIs, plataforma, evolución semanal y P&L se calculan sobre `Ventas` (nivel orden); todo lo que va por categoría (donut, tarjetas de tipo, insights ROI/margen) sale de `Items_Ordenes` vía `buildCatBreakdown(r)` + `aggByCategory(rows)`.

`buildCatBreakdown` reparte la ganancia neta de la orden entre categorías proporcional a la **ganancia bruta de línea** (Método B), con fallback a proporcional al precio si alguna línea tiene bruto ≤ 0. El prorrateo conserva los totales exactos — si lo modificas, verifica que `suma(catBreakdown) == total de la orden`.

### Índices de columnas gviz

Todo el parseo es posicional por índice 0-based (`processGvizData`, `processItemsData`, `processGastosData`, `processInventarioData`), con los rangos codificados en las URLs (`A4:V500`, `A2:K1000`, `A2:C200`, `A3:N1000`). **Agregar o quitar una columna en el Sheet rompe el dashboard en silencio**: hay que reajustar índices y rango a la vez. El mapeo completo está en el README.

Detalles que ya causaron bugs y no hay que revertir:
- Toda conversión de texto usa `String(x ?? '')` antes de `.trim()` — gviz devuelve no-strings.
- Para montos se prefiere `.f` sobre `.v` (`c[i]?.f ?? c[i]?.v`) porque `.f` trae el valor formateado del Sheet.
- Excepción: el margen de `Inventario` (col M) usa `.v` a propósito — `.f` viene como `"5%"` y multiplicaría dos veces.
- Se usan las columnas J/K de `Items_Ordenes` (totales de línea), no G/H (unitarios).

### UI

Tres paneles (`showPanel`): Dashboard, Estado de Resultados (`renderPnL`), Cotizador. El Cotizador es un bloque de JS independiente (ES5, líneas ~531-761) que no toca el estado del dashboard: calcula costos de importación en USD→MXN, deriva precio de venta y utilidad, persiste preferencias e historial en `localStorage` y consulta `open.er-api.com/v6/latest/USD` (keyless, CORS abierto).

### Cotizador

`computeQuote(inputs)` es la **única** definición de la fórmula: pura, sin tocar el DOM. La usan `calc()` para pintar el desglose y `renderHist()` para pintar cada renglón del historial. Si cambias la fórmula, cámbiala solo ahí.

El campo "Utilidad deseada" es **markup sobre el costo** (`venta = total × (1 + p/100)`), no margen sobre la venta. El margen que se muestra abajo es derivado e informativo — es el que se puede comparar con el "margen" que reporta el dashboard, que se calcula al revés.

Claves de `localStorage`: `cu_tdc_card`, `cu_tdc_pobox`, `cu_util_pct` (preferencias) y `cu_cotizaciones` (historial, máx. 200). **El historial guarda solo inputs, nunca resultados** — así una corrección de la fórmula se refleja en todo el historial sin migrar datos. No agregues campos calculados al objeto que se persiste.

El autoguardado tiene debounce de 800 ms y actualiza `ACTIVE_ID`, la cotización en edición; por eso teclear el nombre no crea una entrada por letra. Dos trampas que ya se resolvieron y no hay que revertir:

- Asignar `.value` por código **no** dispara el evento `input`. `parseOrden()`, el botón "usar" del tipo de cambio y `loadQuote()` tienen que llamar al autoguardado (o cancelarlo) a mano.
- `loadQuote()` y el botón "Nueva cotización" hacen `clearTimeout(saveTimer)`. Sin eso, un guardado ya agendado se dispara después y sobrescribe la cotización que se acaba de desligar.

El nombre del producto se inserta con `innerHTML` en el historial, así que pasa por `esc()`.

Tema dark/light por `data-theme` en `<html>` + variables CSS (`--gold`, `--teal`, `--surface`). Cambiar de tema fuerza un `renderAll` completo porque los colores de Chart.js se leen del tema en cada render.

## Markup de placeholder

Varios contenedores (`#kpi-row`, `#tipo-cards`, `#franq-bars`, `#month-pills`) traen HTML hardcodeado con cifras de una corrida vieja. Es lo que se ve mientras carga y lo reemplaza el render correspondiente. **No es fuente de verdad** — no leas números de ahí ni asumas que reflejan el estado actual del Sheet. Los contenedores de Inventario (`#inv-salud`, `#inv-atascado`, `#inv-auditor`, `#alerts-list`) sí nacen vacíos.
