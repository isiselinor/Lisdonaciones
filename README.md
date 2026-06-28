# 🤝 Sistema de Gestión de Donaciones

Sistema de transparencia para un fondo de donaciones. Todo el dinero (en cualquier moneda)
se consolida en un **único fondo en dólares**.

- **Backend:** Google Apps Script (`Code.gs`), vinculado a tu planilla de respuestas. Expone una API web.
- **Frontend:** `index.html` estático que consume la API por **JSONP** (sirve para hosting estático tipo Netlify).

Monedas configuradas: **ARS, USD, VES**. Solo suman las donaciones **verificadas**.

---

## Archivos

| Archivo | Qué es | Dónde va |
|---|---|---|
| `Code.gs` | Backend de Apps Script | Se pega en el editor de Apps Script de tu planilla |
| `index.html` | Página pública | Se sube a Netlify |
| `netlify.toml` | Config de Netlify | Se sube junto a `index.html` |
| `README.md` | Esta guía | Solo referencia |

---

## PARTE A — Backend (Apps Script)

### A1. Abrir el editor
1. Abrí tu **planilla de respuestas** (la Hoja de cálculo donde caen las respuestas del formulario).
   - Si tu formulario todavía no envía a una hoja: en el Form → pestaña **Respuestas** → ícono de Sheets → **Crear hoja de cálculo**.
2. En esa planilla: menú **Extensiones → Apps Script**.

### A2. Pegar el código
1. Borrá lo que haya en `Código.gs`.
2. Pegá **todo** el contenido de `Code.gs`.
3. Guardá (💾).

### A3. Sembrar las hojas y el menú
1. Recargá la planilla (F5). Va a aparecer un menú nuevo: **🤝 Donaciones**.
   - Si no aparece, en Apps Script ejecutá la función `onOpen` una vez (te pedirá permisos: aceptá con tu cuenta).
2. En el menú **🤝 Donaciones** corré, en este orden:
   - **Crear/Reparar hoja Tasas** → crea la hoja `Tasas` con ARS/USD/VES.
   - **Crear/Reparar hoja Egresos** → crea la hoja `Egresos` con sus columnas.
   - **Actualizar Resumen ahora** → crea/recalcula la hoja `Resumen`.
   - **Instalar trigger automático (onFormSubmit)** → hace que el Resumen se recalcule solo con cada nueva donación.

### A4. Ajustar las TASAS (a mano)
En la hoja **Tasas**, columna *Unidades por 1 USD*:
- `ARS` → cuántos pesos por 1 dólar (ej.: dólar a 1180 → **1180**).
- `USD` → **1** (y USDT = 1).
- `VES` → cuántos bolívares por 1 dólar.
El sistema las lee en vivo: cambiás el número y los totales en USD se recalculan.

### A5. Publicar como App Web
1. En Apps Script, arriba a la derecha: **Implementar → Nueva implementación**.
2. Tipo (ícono de engranaje) → **Aplicación web**.
3. Configurá:
   - **Descripción:** Donaciones v1
   - **Ejecutar como:** Yo (tu cuenta)
   - **Quién tiene acceso:** **Cualquier persona** ← importante, si no, la web pública no puede leer.
4. **Implementar** → aceptá permisos.
5. Copiá la **URL de la aplicación web** (termina en `/exec`). Esa es tu `API_URL`.
   - También la podés ver luego desde el menú **🤝 Donaciones → Ver URL de la API web**.

### A6. Probar la API
Pegá en el navegador (cambiando la URL por la tuya):
- `TU_URL/exec?action=balance` → debería devolver un JSON con `totalRaisedUSD`, `totalSpentUSD`, `balanceUSD`.
- `TU_URL/exec?action=search&name=juan` → busca por nombre.

---

## PARTE B — Frontend (Netlify)

### B1. Configurar `index.html`
Abrí `index.html` y, cerca del final (bloque CONFIGURACIÓN), pegá tu URL:
```js
var API_URL = "https://script.google.com/macros/s/AKfyc.../exec";  // ← la tuya, termina en /exec
var FORM_URL = "https://forms.gle/R8m5dUpwm4v6Xgwr6";              // ← link público para donar
```
Guardá el archivo.

### B2. Subir a Netlify (lo más simple: arrastrar y soltar)
1. Entrá a https://app.netlify.com (creá cuenta gratis si no tenés).
2. **Add new site → Deploy manually** (o pestaña **Sites → Drag and drop**).
3. Arrastrá la carpeta que contiene `index.html` y `netlify.toml`.
4. Netlify te da una URL tipo `https://nombre-al-azar.netlify.app`. ¡Listo!
   - Opcional: **Site configuration → Change site name** para una URL más linda.

> Alternativa con Git: subí estos archivos a un repo de GitHub y en Netlify elegí **Import from Git**. Cada push se redepliega solo.

### B3. Verificar
Abrí tu URL de Netlify. Deberías ver:
- Botón **💚 Quiero donar** (abre el formulario en pestaña nueva).
- **Total global en dólares** (recaudado − egresos = saldo).
- Tarjetas por moneda, buscador por nombre y la lista de gastos.

---

## Reimplementar cuando cambies el código (MUY IMPORTANTE)

Apps Script **no** actualiza la versión publicada automáticamente. Cada vez que edites `Code.gs`:

1. Guardá los cambios.
2. **Implementar → Gestionar implementaciones**.
3. En tu implementación web, clic en el **lápiz (Editar)**.
4. En **Versión**, elegí **Versión nueva**.
5. **Implementar**.

> La **URL `/exec` no cambia** al crear versión nueva (mientras edites la misma implementación), así que **no** tenés que tocar `index.html`. Si en cambio creás una implementación *desde cero*, te da una URL nueva y ahí sí actualizá `API_URL`.

---

## Cómo funciona (resumen técnico)

- **Columnas por encabezado:** el código busca *Nombre completo*, *Método de donación*, *Monto*, etc. por su texto. Podés reordenar columnas sin romper nada.
- **Monedas:** se detectan por palabras clave en *Método de donación*. Se evalúan en orden USD → VES → ARS, así "bolívares" gana sobre el "pesos" genérico.
- **Verificada:** suma solo si la celda dice `SI`, `SÍ`, `X` o `VERIFICADA` (sin distinguir mayúsculas/acentos). Vacío / `NO` / `pendiente` no suma.
- **Parser de montos:** entiende `14.399,00` (europeo), `50000`, `50.000` (miles), `10.5` y `10,5` (decimales).
- **Consolidado en USD:** cada moneda se convierte con la tasa de la hoja *Tasas*. El **gasto siempre se muestra en USD** como cifra principal (sale del fondo único, no se separa por moneda).

## Endpoints de la API (`?action=`)
`totals` · `egresos` · `gastos` · `balance` · `rates` · `search&name=...`
Con `&callback=` responde **JSONP**; sin él, **JSON**.

---

Sistema creado por **Isis Elinor** · Responsable de la recaudación: **@listheproducer**
