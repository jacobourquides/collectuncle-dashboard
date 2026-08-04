# Cotizador: utilidad e historial — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Que el cotizador calcule precio de venta y utilidad en pesos a partir de un porcentaje de markup, y que guarde cada cotización en un historial navegable.

**Architecture:** Todo el cambio vive en `index.html`, dentro del panel `#panel-cotizador` (markup ~409-488) y del IIFE en ES5 que lo controla (~490-567). Se extrae una función pura `computeQuote(inputs)` como única definición de la fórmula, se persisten solo los inputs en `localStorage`, y el historial se pinta recalculando con esa misma función.

**Tech Stack:** HTML + CSS + JavaScript ES5 en un solo archivo estático. Sin build, sin npm, sin framework, sin dependencias. `localStorage` para persistencia.

## Global Constraints

- **Un solo archivo.** Todo el código va en `index.html`. No crear archivos JS, CSS ni `package.json`.
- **ES5 estricto.** El IIFE del cotizador usa `var`, `function(){}`, concatenación de strings. Nada de `const`/`let`, arrow functions, template literals, `class`, spread ni `Array.prototype.find`. Seguir ese estilo.
- **No tocar el dashboard.** No modificar `ALL_ROWS`, `ITEMS_BY_ORDER`, `INVENTARIO`, `GASTOS_OPS`, `GASTOS_DETAIL`, `GASTOS_ADS`, `renderAll`, `loadAndRender` ni ninguna función de gviz. El cotizador es un bloque independiente.
- **Nunca abrir con `file://`.** Siempre `python3 -m http.server 8000` y navegar a `http://localhost:8000`. Con `file://` gviz falla y el dashboard cae a datos de muestra.
- **Integridad del archivo.** Antes de cada commit: `grep -c index_files index.html` debe dar `0` y `grep -c docs.google.com index.html` debe dar más de `0`.
- **Markup, no margen.** El porcentaje que captura el usuario se suma al costo: `venta = costo × (1 + p/100)`. El margen sobre venta es solo un dato informativo derivado.
- **Guardar solo inputs.** En `localStorage` nunca se escriben resultados calculados, únicamente los campos que el usuario capturó, más `id` y `ts`.
- **Escapar el nombre del producto** antes de insertarlo en `innerHTML`.
- **Rama:** `feat/cotizador-utilidad-historial`. Commit tras cada tarea, y `git push` después de cada commit.

## File Structure

| Archivo | Responsabilidad | Cambio |
|---|---|---|
| `index.html` ~445 | `.section-label` del cotizador | Se le agrega el botón del historial |
| `index.html` ~411-443 | `<style>` del panel | Reglas nuevas para venta estimada y drawer |
| `index.html` ~447-470 | Tarjeta "Datos de la compra" | Input de utilidad |
| `index.html` ~471-486 | Tarjeta "Desglose" | Bloque "Venta estimada" |
| `index.html` ~487 | Fin del panel | Markup del drawer y su backdrop |
| `index.html` ~490-567 | IIFE del cotizador | `computeQuote`, capa de guardado, autoguardado, drawer |

Cuatro tareas, cada una con su ciclo de verificación y su commit:

1. `computeQuote` puro y `calc()` reescrito sobre él — sin cambio visible, es la base.
2. Input de utilidad, bloque "Venta estimada" y copiado extendido — primera funcionalidad visible.
3. Capa de guardado, autoguardado y "Nueva cotización" — persiste sin UI que lo muestre.
4. Drawer del historial — cargar y borrar.

---

### Task 1: `computeQuote` puro y `calc()` reescrito

Refactor sin cambio visible. Aísla la fórmula para que la usen tanto el desglose como el historial. Se incluye desde ya `utilPct` en la firma para no reescribir la función en la Tarea 2; mientras el input no exista, `n()` devuelve `0` y la venta iguala al costo.

**Files:**
- Modify: `index.html:496` (endurecer el helper `n`)
- Modify: `index.html:498-516` (partir `calc()` en `readInputs` + `computeQuote` + render)
- Test: verificación manual en el navegador (no hay framework de tests en el repo)

**Interfaces:**
- Consumes: los helpers existentes `$`, `n`, `mxn`, `usd`, `LS` del IIFE.
- Produces:
  - `readInputs()` → `{prod, price, ship, taxPct, poboxPct, tdcCard, tdcPobox, utilPct}` — `prod` es string ya con `.trim()`; el resto son números.
  - `computeQuote(q)` → `{imp, sub, com, cardMXN, poboxMXN, total, venta, utilidad, margen, hasTdc}` — pura, sin tocar el DOM. Las tareas 3 y 4 la llaman con objetos leídos de `localStorage`, no del DOM.

- [ ] **Step 1: Anotar los números de referencia (el "test")**

Con `python3 -m http.server 8000` corriendo, abrir `http://localhost:8000`, ir al panel Cotizador y capturar:

| Campo | Valor |
|---|---|
| Precio | `43` |
| Envío | `8` |
| Tax | `8.25` |
| Comisión PO Box | `16` |
| TDC tarjeta | `19.50` |
| TDC PO Box | `21.00` |

El desglose debe mostrar exactamente esto, y son los valores que no pueden cambiar con el refactor:

```
Precio                          $43.00
Envío                            $8.00
Impuesto (8.25%)                 $4.21
Subtotal (cargo a tarjeta)      $55.21
  × 19.5 = $1,076.55
Comisión PO Box (16%)            $8.83
  × 21 = $185.50
TOTAL a cobrar              $1,262.04
```

- [ ] **Step 2: Endurecer el helper `n`**

`readInputs` va a leer `cot-util`, que todavía no existe en el DOM. Hoy `n` explotaría con `TypeError`. Reemplazar la línea 496:

```js
    var n=function(id){var v=parseFloat($(id).value);return isNaN(v)?0:v;};
```

por:

```js
    var n=function(id){var el=$(id);if(!el)return 0;var v=parseFloat(el.value);return isNaN(v)?0:v;};
```

- [ ] **Step 3: Reescribir `calc()` sobre las dos funciones nuevas**

Reemplazar el bloque completo de `function calc(){...}` (líneas 498-516) por:

```js
    function readInputs(){
      return {
        prod:String($('cot-prod').value||'').trim(),
        price:n('cot-price'), ship:n('cot-ship'),
        taxPct:n('cot-tax'), poboxPct:n('cot-pobox'),
        tdcCard:n('cot-tdc-card'), tdcPobox:n('cot-tdc-pobox'),
        utilPct:n('cot-util')
      };
    }
    // Única definición de la fórmula. Pura: no lee ni escribe el DOM.
    // La usan calc() y el render del historial, que la llama con objetos de localStorage.
    function computeQuote(q){
      var imp=(q.price+q.ship)*(q.taxPct/100);
      var sub=q.price+q.ship+imp;
      var com=sub*(q.poboxPct/100);
      var cardMXN=sub*q.tdcCard, poboxMXN=com*q.tdcPobox;
      var total=cardMXN+poboxMXN;
      var venta=total*(1+(q.utilPct||0)/100);
      var utilidad=venta-total;
      return {imp:imp,sub:sub,com:com,cardMXN:cardMXN,poboxMXN:poboxMXN,
              total:total,venta:venta,utilidad:utilidad,
              margen:venta>0?(utilidad/venta*100):0,
              hasTdc:!!(q.tdcCard||q.tdcPobox)};
    }
    function calc(){
      var q=readInputs(), r=computeQuote(q);
      $('cot-o-price').textContent=usd.format(q.price);
      $('cot-o-ship').textContent=usd.format(q.ship);
      $('cot-o-tax-lbl').textContent='Impuesto ('+q.taxPct+'%)';
      $('cot-o-tax').textContent=usd.format(r.imp);
      $('cot-o-sub').textContent=usd.format(r.sub);
      $('cot-o-sub-mxn').innerHTML=q.tdcCard?('× '+q.tdcCard+' = <strong>'+mxn.format(r.cardMXN)+'</strong>'):'ingresa TDC tarjeta →';
      $('cot-o-pobox-lbl').textContent='Comisión PO Box ('+q.poboxPct+'%)';
      $('cot-o-pobox').textContent=usd.format(r.com);
      $('cot-o-pobox-mxn').innerHTML=q.tdcPobox?('× '+q.tdcPobox+' = <strong>'+mxn.format(r.poboxMXN)+'</strong>'):'ingresa TDC PO Box →';
      $('cot-o-total').textContent=r.hasTdc?mxn.format(r.total):'—';
    }
```

- [ ] **Step 4: Verificar que nada cambió**

Recargar `http://localhost:8000`, capturar de nuevo los seis valores del Step 1 y confirmar que el desglose es idéntico, renglón por renglón, incluyendo `TOTAL a cobrar = $1,262.04`.

Borrar los dos TDC: el total debe volver a `—`. Abrir la consola del navegador: cero errores.

- [ ] **Step 5: Commit**

```bash
grep -c index_files index.html    # 0
grep -c docs.google.com index.html # > 0
git add index.html
git commit -m "refactor: extract pure computeQuote from cotizador calc()"
git push
```

---

### Task 2: Campo de utilidad y bloque "Venta estimada"

**Files:**
- Modify: `index.html:411-443` (`<style>` del panel: reglas de venta estimada)
- Modify: `index.html:465-468` (input nuevo en "Datos de la compra")
- Modify: `index.html:480` (bloque nuevo tras `.ctotal`)
- Modify: `index.html` IIFE (persistencia del `%`, render de la venta, texto copiado)
- Test: verificación manual en el navegador

**Interfaces:**
- Consumes: `readInputs()` y `computeQuote(q)` de la Tarea 1.
- Produces:
  - Input `#cot-util` en el DOM — la Tarea 4 lo escribe al cargar una cotización.
  - `savePrefs()` — reemplaza a `saveTdc()`; persiste los dos TDC y el `%` de utilidad.

- [ ] **Step 1: Anotar los resultados esperados (el "test")**

Con los mismos datos de la Tarea 1 más **Utilidad deseada = 40**, el bloque nuevo debe mostrar exactamente:

```
TOTAL a cobrar                      $1,262.04
─── Venta estimada ──────────────────────────
Precio de venta (+40%)              $1,766.86
Utilidad estimada                     $504.82
Margen sobre venta                      28.57%
```

Comprobación a mano: `1,262.04 × 1.4 = 1,766.86`, `1,766.86 − 1,262.04 = 504.82`, `504.82 / 1,766.86 = 28.57%`.

- [ ] **Step 2: Agregar el CSS**

Dentro del `<style>` del panel, justo después de la regla `#panel-cotizador .ctotal .amt{...}` (línea 433), insertar:

```css
      #panel-cotizador .cventa{margin-top:12px;padding:14px 16px;border-radius:var(--radius);background:var(--surface2);border:1px solid var(--border)}
      #panel-cotizador .cventa .vh{font-size:10px;font-weight:500;color:var(--text-3);text-transform:uppercase;letter-spacing:0.06em;margin-bottom:10px}
      #panel-cotizador .cventa .crow{border-bottom:1px dashed var(--border)}
      #panel-cotizador .cventa .crow:last-of-type{border-bottom:none}
      #panel-cotizador .cventa .big .k{font-weight:600;color:var(--text)}
      #panel-cotizador .cventa .big .v{font-size:19px;color:var(--gold)}
      #panel-cotizador .cventa .soft .k,#panel-cotizador .cventa .soft .v{font-size:11px;color:var(--text-3);font-weight:400}
```

- [ ] **Step 3: Agregar el input de utilidad**

Después del `<div class="two">` de los TDC (que cierra en la línea 468) y antes del `<div class="cnote">`, insertar:

```html
        <label>Utilidad deseada <span class="hint">% sobre el costo</span></label>
        <input id="cot-util" type="number" step="0.01" min="0" value="40">
```

- [ ] **Step 4: Agregar el bloque de salida**

Después del `<div class="ctotal">...</div>` (línea 480) y antes de `<div class="cfoot">`, insertar:

```html
        <div class="cventa">
          <div class="vh">Venta estimada</div>
          <div class="crow big"><span class="k" id="cot-o-venta-lbl">Precio de venta (+40%)</span><span class="v" id="cot-o-venta">—</span></div>
          <div class="crow"><span class="k">Utilidad estimada</span><span class="v" id="cot-o-utilidad">—</span></div>
          <div class="crow soft"><span class="k">Margen sobre venta</span><span class="v" id="cot-o-margen">—</span></div>
        </div>
```

- [ ] **Step 5: Pintar la venta en `calc()`**

Al final de `calc()`, después de la línea de `cot-o-total`, agregar:

```js
      $('cot-o-venta-lbl').textContent='Precio de venta (+'+q.utilPct+'%)';
      $('cot-o-venta').textContent=r.hasTdc?mxn.format(r.venta):'—';
      $('cot-o-utilidad').textContent=r.hasTdc?mxn.format(r.utilidad):'—';
      $('cot-o-margen').textContent=r.hasTdc?(r.margen.toFixed(2)+'%'):'—';
```

- [ ] **Step 6: Persistir el porcentaje y escuchar el input**

Reemplazar el bloque de `saveTdc` y sus listeners (líneas 548-552) por:

```js
    // Memoria de los TDC y del % de utilidad (persisten en el navegador)
    function savePrefs(){LS.set('cu_tdc_card',$('cot-tdc-card').value);LS.set('cu_tdc_pobox',$('cot-tdc-pobox').value);LS.set('cu_util_pct',$('cot-util').value);}
    (function(){var c=LS.get('cu_tdc_card'),p=LS.get('cu_tdc_pobox'),u=LS.get('cu_util_pct');if(c)$('cot-tdc-card').value=c;if(p)$('cot-tdc-pobox').value=p;if(u)$('cot-util').value=u;})();
    $('cot-tdc-card').addEventListener('input',savePrefs);
    $('cot-tdc-pobox').addEventListener('input',savePrefs);
    $('cot-util').addEventListener('input',savePrefs);
    ['cot-price','cot-ship','cot-tax','cot-pobox','cot-tdc-card','cot-tdc-pobox','cot-util','cot-prod'].forEach(function(id){$(id).addEventListener('input',calc);});
```

En la línea 543 hay otra llamada a `saveTdc()` dentro del handler de "usar" del tipo de cambio de mercado. Cambiarla a `savePrefs()`.

- [ ] **Step 7: Extender el texto copiado**

Reemplazar el cuerpo del listener de `cot-copy` (línea 554) por:

```js
      var rq=readInputs(), rr=computeQuote(rq);
      var t='Cotización Collectuncle\nProducto: '+(rq.prod||'(sin nombre)')+
        '\nPrecio: '+usd.format(rq.price)+' | Envío: '+usd.format(rq.ship)+
        '\nTOTAL: '+$('cot-o-total').textContent+
        '\nPrecio de venta: '+$('cot-o-venta').textContent+
        '\nUtilidad: '+$('cot-o-utilidad').textContent+' ('+rq.utilPct+'%)';
```

- [ ] **Step 8: Verificar**

Recargar y comprobar en `http://localhost:8000`:

1. Con los datos del Step 1: los cuatro renglones coinciden exactamente.
2. Cambiar utilidad a `0`: venta = `$1,262.04`, utilidad = `$0.00`, margen = `0.00%`.
3. Cambiar utilidad a `100`: venta = `$2,524.09`, margen = `50.00%`.
4. Borrar ambos TDC: total, venta y utilidad muestran `—`, nunca `$0.00`.
5. Poner utilidad en `35`, recargar la página: el campo vuelve en `35`.
6. Copiar cotización: el portapapeles trae las líneas de precio de venta y utilidad.
7. Consola sin errores.

- [ ] **Step 9: Commit**

```bash
grep -c index_files index.html && grep -c docs.google.com index.html
git add index.html
git commit -m "feat: add markup input and estimated sale price to cotizador"
git push
```

---

### Task 3: Capa de guardado y autoguardado

Sin UI todavía: al terminar esta tarea las cotizaciones se guardan y se verifican desde la consola del navegador. La Tarea 4 les pone cara.

**Files:**
- Modify: `index.html` IIFE (funciones nuevas antes de `loadFx`)
- Modify: `index.html` IIFE (listeners de input, `parseOrden`, botón `cot-reset`)
- Modify: `index.html:483` (etiqueta del botón)
- Modify: `index.html:487` y su listener (toast parametrizable)
- Test: verificación manual con la consola del navegador

**Interfaces:**
- Consumes: `readInputs()`, `computeQuote(q)` de la Tarea 1.
- Produces:
  - `quotesLoad()` → array de cotizaciones, `ts` descendente; `[]` si no hay o si el JSON está corrupto.
  - `quotesSave(q)` → hace upsert por `q.id`, fija `q.ts = Date.now()`, recorta a `MAX_QUOTES`.
  - `quotesDelete(id)`.
  - `newId()` → string único.
  - `showToast(msg)` → toast con texto variable.
  - `renderHist()` → **stub vacío en esta tarea**, implementado en la Tarea 4. Se llama ya desde `autoSave`.
  - Variable `ACTIVE_ID` — id de la cotización en edición, o `null`.
  - `esc(s)` → escapa `& < > "` para `innerHTML`.

- [ ] **Step 1: Definir el "test" — la secuencia de consola**

Al terminar la tarea, esta secuencia en la consola debe dar los resultados anotados:

```js
JSON.parse(localStorage.getItem('cu_cotizaciones') || '[]')
```

| Acción en la UI | Resultado esperado |
|---|---|
| Llenar todo menos el producto, esperar 1s | `[]` — sin nombre no se guarda |
| Escribir "Funko Batman" letra por letra | **un solo** objeto, con `prod: "Funko Batman"` |
| Cambiar utilidad a 50, esperar 1s | el **mismo** objeto (mismo `id`), con `utilPct: 50` |
| Clic en "Nueva cotización", llenar otra, esperar | dos objetos, el nuevo primero |

Cada objeto debe traer exactamente estas llaves y ninguna más: `id`, `ts`, `prod`, `price`, `ship`, `taxPct`, `poboxPct`, `tdcCard`, `tdcPobox`, `utilPct`. Que no aparezca `total`, `venta` ni `utilidad`: solo se guardan inputs.

- [ ] **Step 2: Parametrizar el toast**

El toast hoy tiene el texto fijo "Cotización copiada" en el HTML y hace falta para avisar fallos de guardado. En la línea 487 dejar el elemento vacío:

```html
    <div class="cot-toast" id="cot-toast"></div>
```

Y en el IIFE, junto a los demás helpers (después de la definición de `LS`, línea 497), agregar:

```js
    function showToast(msg){var el=$('cot-toast');el.textContent=msg;el.classList.add('show');setTimeout(function(){el.classList.remove('show');},1900);}
    function esc(s){return String(s==null?'':s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;');}
```

En el listener de `cot-copy`, reemplazar el `.then(...)` que manipula el toast a mano por:

```js
      navigator.clipboard.writeText(t).then(function(){showToast('Cotización copiada');});
```

- [ ] **Step 3: Escribir la capa de guardado**

Insertar en el IIFE, justo antes de `function loadFx(){`:

```js
    // ---- Historial de cotizaciones (localStorage) ----
    var QKEY='cu_cotizaciones', MAX_QUOTES=200;
    var ACTIVE_ID=null, idSeq=0, saveTimer=null, warnedLS=false;

    function newId(){return String(Date.now())+'-'+(idSeq++);}

    function quotesLoad(){
      try{
        var raw=LS.get(QKEY);
        if(!raw)return[];
        var a=JSON.parse(raw);
        return Object.prototype.toString.call(a)==='[object Array]'?a:[];
      }catch(e){return[];}
    }
    // Escritura directa: LS.set traga el error y aquí sí importa saberlo.
    function quotesWrite(arr){
      try{localStorage.setItem(QKEY,JSON.stringify(arr));return true;}
      catch(e){
        if(!warnedLS){warnedLS=true;showToast('Este navegador no permite guardar el historial');}
        return false;
      }
    }
    function quotesSave(q){
      var arr=quotesLoad(),i,found=-1;
      for(i=0;i<arr.length;i++){if(arr[i].id===q.id){found=i;break;}}
      q.ts=Date.now();
      if(found>=0)arr[found]=q;else arr.push(q);
      arr.sort(function(a,b){return b.ts-a.ts;});
      if(arr.length>MAX_QUOTES)arr=arr.slice(0,MAX_QUOTES);
      quotesWrite(arr);
      return q;
    }
    function quotesDelete(id){
      var arr=quotesLoad(),out=[],i;
      for(i=0;i<arr.length;i++)if(arr[i].id!==id)out.push(arr[i]);
      quotesWrite(out);
    }
    // La Tarea 4 la implementa; autoSave ya la llama.
    function renderHist(){}

    // Autoguardado: 800ms tras el último cambio, y solo si la cotización
    // tiene nombre y un costo real. Mientras se edita se actualiza la
    // cotización activa, así teclear el nombre no crea una entrada por letra.
    function scheduleSave(){
      if(saveTimer)clearTimeout(saveTimer);
      saveTimer=setTimeout(autoSave,800);
    }
    function autoSave(){
      var q=readInputs(), r=computeQuote(q);
      if(!q.prod||!(r.total>0))return;
      q.id=ACTIVE_ID||newId();
      quotesSave(q);
      ACTIVE_ID=q.id;
      renderHist();
    }
```

- [ ] **Step 4: Disparar el autoguardado desde los inputs**

Reemplazar la línea de listeners que quedó en la Tarea 2:

```js
    ['cot-price','cot-ship','cot-tax','cot-pobox','cot-tdc-card','cot-tdc-pobox','cot-util','cot-prod'].forEach(function(id){$(id).addEventListener('input',calc);});
```

por:

```js
    function onEdit(){calc();scheduleSave();}
    ['cot-price','cot-ship','cot-tax','cot-pobox','cot-tdc-card','cot-tdc-pobox','cot-util','cot-prod'].forEach(function(id){$(id).addEventListener('input',onEdit);});
```

En `parseOrden`, la última línea es `calc();`. Cambiarla a `calc();scheduleSave();` — pegar una orden llena los campos por código, así que el evento `input` de esos campos no se dispara.

En el handler de "usar" del tipo de cambio de mercado (línea ~543), que hoy hace `savePrefs();calc();`, dejar `savePrefs();onEdit();`.

- [ ] **Step 5: Convertir "Limpiar" en "Nueva cotización"**

En la línea 483, cambiar el botón:

```html
          <button class="btn btn-theme" id="cot-reset"><i class="ti ti-plus"></i> Nueva cotización</button>
```

Y en su listener, después de la línea `$('cot-tdc-card').value=c||'';$('cot-tdc-pobox').value=p||'';`, agregar:

```js
      var u=LS.get('cu_util_pct');$('cot-util').value=u||'40';
      if(saveTimer){clearTimeout(saveTimer);saveTimer=null;}
      ACTIVE_ID=null;
      renderHist();
```

El `clearTimeout` importa: sin él, un guardado ya agendado se dispararía después del reset y escribiría campos vacíos sobre la cotización que se acaba de desligar.

- [ ] **Step 6: Verificar**

Recargar y correr la tabla completa del Step 1, revisando `localStorage` en la consola tras cada acción. Además:

- Pegar una orden de eBay en el textarea con un producto ya nombrado: se guarda también (el parser dispara `scheduleSave`).
- Consola sin errores.

- [ ] **Step 7: Commit**

```bash
grep -c index_files index.html && grep -c docs.google.com index.html
git add index.html
git commit -m "feat: autosave cotizaciones to localStorage with active-quote semantics"
git push
```

---

### Task 4: Drawer del historial

**Files:**
- Modify: `index.html:411-443` (`<style>`: reglas del drawer)
- Modify: `index.html:445` (botón dentro del `.section-label`)
- Modify: `index.html:487` (markup del drawer y backdrop)
- Modify: `index.html` IIFE (`renderHist` real, `loadQuote`, apertura/cierre, delegación de eventos)
- Test: verificación manual en el navegador

**Interfaces:**
- Consumes: `quotesLoad()`, `quotesDelete(id)`, `computeQuote(q)`, `esc(s)`, `ACTIVE_ID`, `calc()`.
- Produces: `renderHist()` real (sustituye al stub), `loadQuote(id)`, `openDrawer()`, `closeDrawer()`, `agoTxt(ts)`.

- [ ] **Step 1: Definir el "test" — el recorrido de UI**

Al terminar, este recorrido debe funcionar sin tocar la consola:

1. Guardar dos cotizaciones distintas → el botón dice `Mis cotizaciones (2)`.
2. Abrir el drawer → dos renglones, el más reciente arriba, cada uno con nombre, precio de venta entre paréntesis, antigüedad, % y costo.
3. Clic en el renglón viejo → se cargan sus ocho campos, el drawer se cierra, el desglose corresponde.
4. Reabrir → ese renglón está resaltado.
5. Cambiar el % y esperar → sigue habiendo dos cotizaciones, no tres.
6. Clic en `×` → pide confirmación; al aceptar queda un renglón y el botón dice `(1)`.
7. `Escape` y clic en el backdrop cierran el drawer.
8. Recargar la página → el historial sigue ahí.

- [ ] **Step 2: Agregar el CSS del drawer**

Al final del `<style>` del panel, antes de `</style>` (línea 444):

```css
      #cot-hist-btn{margin-left:auto;text-transform:none;letter-spacing:0}
      #cot-drawer-back{position:fixed;inset:0;background:rgba(0,0,0,0.45);opacity:0;pointer-events:none;transition:opacity 0.2s;z-index:90}
      #cot-drawer-back.open{opacity:1;pointer-events:auto}
      #cot-drawer{position:fixed;top:0;right:0;bottom:0;width:340px;max-width:88vw;background:var(--surface);border-left:1px solid var(--border);transform:translateX(101%);transition:transform 0.22s;z-index:91;display:flex;flex-direction:column}
      #cot-drawer.open{transform:none}
      #cot-drawer .dh{display:flex;align-items:center;padding:16px 18px;border-bottom:1px solid var(--border)}
      #cot-drawer .dh .t{font-size:13px;font-weight:600;color:var(--text)}
      #cot-drawer .dh button{margin-left:auto;background:none;border:none;color:var(--text-2);cursor:pointer;font-size:20px;line-height:1;padding:0 4px;font-family:inherit}
      #cot-drawer .dl{overflow-y:auto;flex:1;padding:6px 0}
      #cot-drawer .qi{display:flex;align-items:flex-start;gap:8px;padding:11px 18px;cursor:pointer;border-left:3px solid transparent}
      #cot-drawer .qi:hover{background:var(--surface2)}
      #cot-drawer .qi.active{border-left-color:var(--gold);background:var(--surface2)}
      #cot-drawer .qi .qb{flex:1;min-width:0}
      #cot-drawer .qi .qt{font-size:13px;font-weight:600;color:var(--text);line-height:1.35;word-break:break-word}
      #cot-drawer .qi .qt em{font-style:normal;color:var(--teal);font-variant-numeric:tabular-nums}
      #cot-drawer .qi .qs{font-size:11px;color:var(--text-3);margin-top:3px;font-variant-numeric:tabular-nums}
      #cot-drawer .qi .qd{background:none;border:none;color:var(--text-3);cursor:pointer;font-size:16px;line-height:1;padding:0 2px;font-family:inherit}
      #cot-drawer .qi .qd:hover{color:var(--red)}
      #cot-drawer .empty{padding:22px 18px;font-size:12px;color:var(--text-3);text-align:center;line-height:1.5}
```

- [ ] **Step 3: Agregar el botón**

`.section-label` ya es `display:flex` con `align-items:center`, así que el botón entra dentro del mismo div y se empuja a la derecha con `margin-left:auto`. Reemplazar la línea 445:

```html
    <div class="section-label">Cotizador de importación · eBay → PO Box McAllen
      <button class="btn btn-theme" id="cot-hist-btn"><i class="ti ti-history"></i> Mis cotizaciones (<span id="cot-hist-count">0</span>)</button>
    </div>
```

- [ ] **Step 4: Agregar el markup del drawer**

Después del `<div class="cot-toast" id="cot-toast"></div>` (línea 487) y antes del `</div>` que cierra el panel:

```html
    <div id="cot-drawer-back"></div>
    <div id="cot-drawer">
      <div class="dh"><span class="t">Mis cotizaciones</span><button id="cot-drawer-close" title="Cerrar">×</button></div>
      <div class="dl" id="cot-hist-list"></div>
    </div>
```

- [ ] **Step 5: Implementar `renderHist` y la carga**

Reemplazar el stub `function renderHist(){}` de la Tarea 3 por:

```js
    function agoTxt(ts){
      var ms=Date.now()-ts, d=Math.floor(ms/86400000), h=Math.floor(ms/3600000);
      if(d>=1)return d===1?'ayer':'hace '+d+' días';
      if(h>=1)return h===1?'hace 1 hora':'hace '+h+' horas';
      return 'hace unos minutos';
    }
    function renderHist(){
      var arr=quotesLoad(), el=$('cot-hist-list'), html='', i, q, r;
      $('cot-hist-count').textContent=arr.length;
      if(!arr.length){el.innerHTML='<div class="empty">Aún no hay cotizaciones guardadas.<br>Ponle nombre a un producto y se guarda sola.</div>';return;}
      for(i=0;i<arr.length;i++){
        q=arr[i];r=computeQuote(q);
        html+='<div class="qi'+(q.id===ACTIVE_ID?' active':'')+'" data-id="'+esc(q.id)+'">'+
          '<div class="qb">'+
            '<div class="qt">'+esc(q.prod)+' <em>('+mxn.format(r.venta)+')</em></div>'+
            '<div class="qs">'+agoTxt(q.ts)+' · '+(q.utilPct||0)+'% utilidad · costo '+mxn.format(r.total)+'</div>'+
          '</div>'+
          '<button class="qd" data-del="'+esc(q.id)+'" title="Borrar">×</button>'+
        '</div>';
      }
      el.innerHTML=html;
    }
    function loadQuote(id){
      var arr=quotesLoad(),i,q=null;
      for(i=0;i<arr.length;i++)if(arr[i].id===id){q=arr[i];break;}
      if(!q)return;
      if(saveTimer){clearTimeout(saveTimer);saveTimer=null;}
      $('cot-prod').value=q.prod||'';
      $('cot-price').value=q.price;
      $('cot-ship').value=q.ship;
      $('cot-tax').value=q.taxPct;
      $('cot-pobox').value=q.poboxPct;
      $('cot-tdc-card').value=q.tdcCard||'';
      $('cot-tdc-pobox').value=q.tdcPobox||'';
      $('cot-util').value=q.utilPct;
      $('cot-paste').value='';$('cot-paste-status').textContent='';
      ACTIVE_ID=q.id;
      calc();renderHist();closeDrawer();
    }
```

Asignar `.value` por código no dispara el evento `input`, así que cargar una cotización no re-agenda un guardado. El `clearTimeout` cancela cualquier guardado pendiente de lo que se estaba editando antes.

- [ ] **Step 6: Cablear la apertura, el cierre y los clics**

Justo antes de la línea final `loadFx();calc();` del IIFE, agregar:

```js
    function openDrawer(){renderHist();$('cot-drawer').classList.add('open');$('cot-drawer-back').classList.add('open');}
    function closeDrawer(){$('cot-drawer').classList.remove('open');$('cot-drawer-back').classList.remove('open');}
    $('cot-hist-btn').addEventListener('click',openDrawer);
    $('cot-drawer-close').addEventListener('click',closeDrawer);
    $('cot-drawer-back').addEventListener('click',closeDrawer);
    document.addEventListener('keydown',function(e){if(e.key==='Escape')closeDrawer();});
    $('cot-hist-list').addEventListener('click',function(e){
      var t=e.target, del=t.getAttribute?t.getAttribute('data-del'):null;
      if(del){
        e.stopPropagation();
        if(confirm('¿Borrar esta cotización?')){
          quotesDelete(del);
          if(ACTIVE_ID===del)ACTIVE_ID=null;
          renderHist();
        }
        return;
      }
      var row=t;
      while(row&&row!==this&&!(row.className&&/(^|\s)qi(\s|$)/.test(row.className)))row=row.parentNode;
      if(row&&row!==this)loadQuote(row.getAttribute('data-id'));
    });
```

Y cambiar la última línea del IIFE de `loadFx();calc();` a:

```js
    loadFx();calc();renderHist();
```

para que el conteo del botón sea correcto desde que carga la página.

- [ ] **Step 7: Verificar**

Correr el recorrido completo de los ocho puntos del Step 1 en `http://localhost:8000`. Además:

- Guardar una cotización con el nombre `Funko <b>test</b> & "raro"`: el renglón muestra el texto literal, sin negritas ni markup roto.
- Cambiar entre tema claro y oscuro con el drawer abierto: los colores del drawer siguen al tema.
- Angostar la ventana a ~400px: el drawer ocupa 88% y sigue usable.
- El resto del dashboard carga bien: el badge superior **no** dice "Datos de muestra", KPIs y gráficas pobladas.
- Consola sin errores.

- [ ] **Step 8: Commit**

```bash
grep -c index_files index.html && grep -c docs.google.com index.html
git add index.html
git commit -m "feat: add slide-out quote history drawer to cotizador"
git push
```

---

## Cierre

- [ ] **Actualizar `CLAUDE.md`**

La sección "UI" describe el Cotizador como "un bloque de JS independiente (ES5, líneas ~490-567) que no toca el estado del dashboard: calcula costos de importación en USD→MXN, persiste tipos de cambio en `localStorage`". Extenderla: ahora también calcula precio de venta y utilidad por markup, y persiste el historial de cotizaciones bajo `cu_cotizaciones`. Anotar que `computeQuote` es la única definición de la fórmula y que en `localStorage` se guardan solo inputs. Corregir el rango de líneas.

- [ ] **Commit y push del `CLAUDE.md`**

- [ ] **Abrir el PR** hacia `main` desde `feat/cotizador-utilidad-historial`.
