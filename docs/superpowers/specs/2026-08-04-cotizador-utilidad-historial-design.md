# Cotizador: utilidad deseada e historial de cotizaciones

Fecha: 2026-08-04
Estado: aprobado, pendiente de implementar

## Problema

El cotizador (`index.html`, panel `#panel-cotizador`) calcula cuánto cuesta traer un
producto de eBay hasta México: precio, envío, impuesto, comisión del PO Box y los dos
tipos de cambio. Termina en un **TOTAL a cobrar** en pesos, que es el costo.

Efraín necesita dos cosas que hoy no existen:

1. Saber a qué precio vender y cuánto ganaría, a partir de un porcentaje de utilidad
   que él define.
2. Conservar las cotizaciones que ha hecho y poder volver a cualquiera de ellas.

## Decisiones tomadas

| Decisión | Elección | Motivo |
|---|---|---|
| Significado del % | **Markup sobre el costo** (`venta = costo × (1 + p)`) | Es como Efraín lo piensa y lo describió |
| Persistencia | **`localStorage`** | No hay backend; gviz es solo lectura. Mismo mecanismo que ya usan los TDC |
| Disparo del guardado | **Automático**, con debounce | Nunca se pierde un cálculo |
| Edición | **Actualiza la cotización activa** | Evita que el autoguardado llene la lista de borradores |

El markup y el margen sobre venta no son lo mismo. 40% de markup equivale a 28.6% de
margen. El dashboard reporta *margen*, así que el desglose muestra ambos para que los
números sean comparables, pero el único campo que Efraín captura es el markup.

## Alcance

Todo vive dentro del IIFE en ES5 de las líneas ~490-567 de `index.html`, más el markup
del panel (~409-488) y su bloque `<style>`. No se toca el estado global del dashboard
(`ALL_ROWS`, `ITEMS_BY_ORDER`, `INVENTARIO`, `GASTOS_*`) ni la cadena de carga de gviz.

## Diseño

### `computeQuote(inputs)` — el núcleo

Función pura, sin efectos ni lectura del DOM. Recibe los ocho inputs y devuelve todos
los derivados:

```js
computeQuote({price, ship, taxPct, poboxPct, tdcCard, tdcPobox, utilPct})
// →
{
  imp,        // (price + ship) * taxPct/100
  sub,        // price + ship + imp            (cargo a la tarjeta, USD)
  com,        // sub * poboxPct/100            (comisión PO Box, USD)
  cardMXN,    // sub * tdcCard
  poboxMXN,   // com * tdcPobox
  total,      // cardMXN + poboxMXN            (costo puesto en México, MXN)
  venta,      // total * (1 + utilPct/100)     (MXN)
  utilidad,   // venta - total                 (MXN)
  margen,     // utilidad / venta * 100        (%, informativo)
  hasTdc      // bool: hay al menos un TDC capturado
}
```

Cuando `hasTdc` es falso, `total`, `venta` y `utilidad` se muestran como `—`; no se
pintan ceros, que se leerían como un resultado real. Cuando `venta` es 0, `margen` es 0
(no `NaN`).

Es la única definición de la fórmula. La usan tanto `calc()` para pintar el desglose
como el render del historial para pintar cada renglón. Extraerla es lo que permite
guardar solo inputs.

### Campo de utilidad

Input nuevo al final de la tarjeta "Datos de la compra":

- `id="cot-util"`, `type="number"`, `step="0.01"`, `min="0"`, default `40`
- Etiqueta: `Utilidad deseada` · hint `% sobre el costo`
- Se persiste en `localStorage` bajo `cu_util_pct` junto a los TDC, porque Efraín usará
  casi siempre el mismo porcentaje. `saveTdc()` pasa a guardarlo también.

### Bloque "Venta estimada"

Va en la tarjeta de desglose, después de `.ctotal`. Tres renglones calculados, ninguno
editable:

| Renglón | Elemento | Formato |
|---|---|---|
| Precio de venta sugerido | `#cot-o-venta` | MXN, destacado |
| Utilidad estimada | `#cot-o-utilidad` | MXN |
| Margen sobre venta | `#cot-o-margen` | %, texto chico y gris |

La etiqueta del precio de venta incluye el porcentaje vigente, igual que ya hacen las
etiquetas de impuesto y comisión: `Precio de venta (+40%)`.

### Capa de guardado

Tres funciones y nada más entre la UI y el almacenamiento:

```js
quotesLoad()        // → array de cotizaciones, más reciente primero; [] si falla
quotesSave(quote)   // upsert por id; recorta a MAX_QUOTES; actualiza ts
quotesDelete(id)
```

Clave `cu_cotizaciones`, JSON array. `MAX_QUOTES = 200`: al exceder, se descartan las
más viejas. Cada entrada guarda **solo inputs**, nunca resultados:

```js
{
  id,        // string único
  ts,        // Date.now() de la última actualización
  prod,      // nombre del producto (título de la lista)
  price, ship, taxPct, poboxPct, tdcCard, tdcPobox, utilPct
}
```

Guardar solo inputs significa que si la fórmula se corrige en el futuro, todo el
historial refleja la corrección sin migrar datos.

El `id` no puede depender de `Date.now()` solo: dos guardados en el mismo milisegundo
colisionarían. Se usa `Date.now() + '-' + (counter++)`.

`localStorage` puede lanzar en Safari privado. El wrapper `LS` existente ya traga el
error con try/catch; se agrega un aviso discreto, **una sola vez por sesión**, cuando
una escritura falla. Hoy fallaría en silencio y Efraín creería tener su historial a
salvo.

### Autoguardado

- Debounce de **800 ms** después del último cambio en cualquier input del cotizador.
- Una cotización se guarda solo si `prod` no está vacío **y** `total > 0`. Sin nombre no
  hay título posible para la lista.
- `ACTIVE_ID` es la cotización que se está editando. Si es `null` al momento de guardar,
  se crea una entrada nueva y su id pasa a ser `ACTIVE_ID`. Si no, se actualiza esa.
- Consecuencia buscada: teclear el nombre letra por letra produce **una** entrada cuyo
  título va cambiando, no una por letra.

### Historial (drawer)

Botón `Mis cotizaciones (N)` junto al `.section-label` del panel, con el conteo en vivo.
Abre un panel deslizable desde la derecha.

Se eligió drawer sobre una tercera columna del `.cot-grid` porque ese grid ya colapsa a
una sola columna abajo de 820px; una tercera columna se volvería ahí un muro vertical
entre los datos y el desglose.

Cada renglón:

```
Funko Batman #144  ($1,680.00)                    ×
hace 2 días · 40% utilidad · costo $1,200.00
```

El precio entre paréntesis es `venta`, recalculado con `computeQuote` al pintar.

- Orden: `ts` descendente.
- Click en el renglón → carga los inputs, fija `ACTIVE_ID`, recalcula y cierra el drawer.
- El renglón de `ACTIVE_ID` va resaltado.
- `×` borra, con `confirm()` previo: es destructivo y no hay deshacer. No debe disparar
  la carga del renglón (`stopPropagation`).
- Lista vacía: mensaje "Aún no hay cotizaciones guardadas".
- Cierre: botón ×, clic en el backdrop, o `Escape`.

### Botones del pie

- `Limpiar` pasa a ser **`Nueva cotización`**: limpia los campos, restaura los defaults y
  pone `ACTIVE_ID = null`. Es el único modo de empezar una entrada nueva desde cero.
  Conserva los TDC y el % de utilidad guardados, como ya hace hoy con los TDC.
- `Copiar cotización` agrega dos líneas al texto copiado:

  ```
  Precio de venta: $1,680.00
  Utilidad: $480.00 (40%)
  ```

  Es justo lo que Efraín le mandaría a un cliente.

## Verificación

No hay tests ni build en el repo. La verificación es manual, sobre
`python3 -m http.server 8000` (nunca `file://`, ver CLAUDE.md):

1. **Aritmética.** Precio 43, envío 8, tax 8.25%, PO Box 16%, TDC 19.50 y 21.00,
   utilidad 40%. Verificar contra cálculo a mano que venta = total × 1.4 y que
   utilidad = venta − total.
2. **Markup vs margen.** Con 40%, el margen mostrado debe ser 28.57%.
3. **Sin TDC.** Total, venta y utilidad muestran `—`, no `$0.00`.
4. **Utilidad 0%.** Venta = total, utilidad = $0.00, margen = 0%.
5. **Nace una sola entrada.** Teclear "Funko Batman" letra por letra con TDC y precio
   cargados deja exactamente una cotización en la lista.
6. **Sin nombre no guarda.** Llenar todo menos el producto: la lista sigue en 0.
7. **Actualizar la activa.** Cargar una del historial, cambiar el %, esperar: se
   actualiza esa misma, el conteo no sube.
8. **Nueva cotización.** Tras `Nueva cotización`, guardar crea una entrada adicional.
9. **Ida y vuelta.** Guardar, recargar la página, abrir el drawer, cargar la cotización:
   los ocho campos vuelven idénticos.
10. **Borrar.** `×` + confirmar quita el renglón y baja el conteo; cancelar no borra.
11. **No hay regresión.** El resto del dashboard carga igual: badge sin "Datos de
    muestra", KPIs y gráficas pobladas.
12. **Archivo íntegro.** `grep -c index_files index.html` → 0 y
    `grep -c docs.google.com index.html` → mayor que 0.

## Fuera de alcance

- Sincronizar cotizaciones al Google Sheet o entre dispositivos. Se descartó a
  conciencia: exigiría un Apps Script desplegado como web app con escritura. Si Efraín
  lo pide después, el cambio queda acotado a `quotesLoad` / `quotesSave` / `quotesDelete`.
- Buscar o filtrar dentro del historial.
- Convertir cotizaciones en filas de `Ventas` o de `Inventario`.
