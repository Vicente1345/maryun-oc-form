# Pruebas del formulario de confirmación de OC

`index.html` es un único archivo autocontenido, sin dependencias externas: funciona en
GitHub Pages sin build. Todo lo que muestra sale del **hash** de la URL (base64 de un JSON
en UTF-8).

## Cómo levantarlo

```sh
cd C:\Users\user\Desktop\maryun-oc-form
python -m http.server 8765
```

Luego abrir `http://localhost:8765/index.html#<HASH>` con los hashes de más abajo.

> **Nota sobre el entorno.** En `file://`, `localhost` y `127.0.0.1` (y con `?demo=1`) la
> página es «entorno de prueba»: si **no** hay hash muestra el payload de demostración con
> una banda naranja que lo advierte. En cualquier otro host —producción— un enlace sin hash
> muestra el panel de error, nunca datos de ejemplo. Para probar ese caso sirve
> `http://127.0.0.2:8765/index.html` (mismo servidor, host que no es «de prueba»).

## Cómo regenerar un hash

```js
// node
const b64 = o => Buffer.from(JSON.stringify(o), 'utf8').toString('base64');
console.log(b64({ v:2, seg:"SEG-2026-00214", proveedor:"…", ordenes:[ /* … */ ] }));
```

El decodificador tolera **base64url** (`-` `_`), relleno ausente y hashes que el cliente de
correo dejó percent-encoded. Rechaza longitudes `%4 == 1` y bytes que no son UTF-8 válido
(`TextDecoder` con `fatal:true`).

---

## Los dos contratos

El **folio `seg`** decide el modo. Nada más lo decide.

| | sin `seg` válido | con `seg` válido (`SEG-AAAA-NNNNN`) |
|---|---|---|
| Modo | **LEGADO** | **v2** |
| Interfaz | «¿Recibió conforme?» Sí / No | 4 estados + casilla «parcial» + correo de contacto |
| Destino | `POWER_AUTOMATE_URL` (firmada, en producción) | `URL_SEGUIMIENTO` (**vacía** hasta que exista el flujo F10) |
| `estado` | `confirmado` / `problema` | `Despachado` / `EnPreparacion` / `Retrasado` / `SinInformacion` |
| `fecha_entrega` en blanco | `"Por definir"` | `""` |

Un `seg` con formato inválido (p. ej. `SEG-26-214`) se **ignora** y cae a modo legado: nunca
se postea al canal equivocado.

---

## Casos

### (a) v2 con 2 OC — 2 tarjetas y folio visible

```
eyJ2IjoyLCJzZWciOiJTRUctMjAyNi0wMDIxNCIsInByb3ZlZWRvciI6IkRpc3RyaWJ1aWRvcmEgRWplbXBsbyBTLkEuIiwib3JkZW5lcyI6W3sibnVtZXJvX29jIjoiNDU4MzAiLCJwcm9kdWN0byI6IkNhYm8gUFAgMTYgbW0geCAyMjAgbSIsInZhcmlhbnRlIjoiTmVncm8iLCJjYW50aWRhZCI6IjQwIHJvbGxvcyIsImZlY2hhX2xpbWl0ZSI6IjA4LTA4LTIwMjYifSx7Im51bWVyb19vYyI6IjQ1NzcwIiwicHJvZHVjdG8iOiJCb3lhIEVWQSAzMDAgbW0iLCJ2YXJpYW50ZSI6IiIsImNhbnRpZGFkIjoiMjAwIHVuIiwiZmVjaGFfbGltaXRlIjoiMTUtMDgtMjAyNiJ9XX0=
```

**Esperado:** badge `Folio: SEG-2026-00214`; `2 órdenes` / `2 OC`; dos tarjetas (45830, 45770);
los 4 estados; casilla «parcial»; caja «Su correo de contacto»; la OC 45770 **no** muestra
«Variante» porque viene vacía.

Cuerpo que arma (con Despachado + «+3 días» + parcial en la 1.ª y Sin información en la 2.ª):

```json
{"v":2,"seg":"SEG-2026-00214","proveedor":"Distribuidora Ejemplo S.A.","respondido_por":"juan@proveedor.cl","fecha_respuesta":"…","respuestas":[{"numero_oc":"45830","producto":"Cabo PP 16 mm x 220 m","estado":"Despachado","fecha_entrega":"2026-08-24","parcial":true,"observaciones":""},{"numero_oc":"45770","producto":"Boya EVA 300 mm","estado":"SinInformacion","fecha_entrega":"","parcial":false,"observaciones":""}]}
```

Con `URL_SEGUIMIENTO` vacía **no se hace ninguna petición**: aparece la pantalla verde con el
pie «No pudimos confirmar la recepción automática…» y el botón *Copiar mi respuesta*.

### (b) Acentos, ñ y Ø

```
eyJ2IjoyLCJzZWciOiJTRUctMjAyNi0wMDc3NyIsInByb3ZlZWRvciI6IlNlw7FvcsOtbyBNYXLDrXRpbW8gw5F1w7FvYSBTLkEuIOKAlCBQZcOxYWZsb3IiLCJvcmRlbmVzIjpbeyJudW1lcm9fb2MiOiI0NTk5OSIsInByb2R1Y3RvIjoiQ2FibyBkZSBueWxvbiDDmCAxMiBtbSDCq3RyZW56YWRvwrsiLCJ2YXJpYW50ZSI6IsOYMTIgwrcgYXp1bCIsImNhbnRpZGFkIjoiMyByb2xsb3MiLCJmZWNoYV9saW1pdGUiOiIzMC0wOS0yMDI2In1dfQ==
```

**Esperado, sin un solo carácter roto:** `Señorío Marítimo Ñuñoa S.A. — Peñaflor` y
`Cabo de nylon Ø 12 mm «trenzado»`.

### (c) Payload LEGADO — interfaz «¿Recibió conforme?»

```
eyJwcm92ZWVkb3IiOiJDb21lcmNpYWwgQW50aWd1YSBMdGRhLiIsImNvcnJlb19wcm92ZWVkb3IiOiJ2ZW50YXNAYW50aWd1YS5jbCIsInRlbGVmb25vX3ZlbmRlZG9yIjoiKzU2OTEyMzQ1Njc4IiwiY29ycmVvX3ZlbmRlZG9yIjoidmJlY2VycmFAbWFyeXVuLmNsIiwiY29ycmVvX3Byb3ZlZWRvcl9hbGVydGEiOiJ2YmVjZXJyYUBtYXJ5dW4uY2wiLCJudW1lcm9fdmVuZGVkb3IiOiJWLTAwMSIsIm9yZGVuZXMiOlt7Im51bWVyb19vYyI6IjQ1ODMwIiwicHJvZHVjdG8iOiJDYWJvIFBQIDE2IG1tIiwidmFyaWFudGUiOiIiLCJjYW50aWRhZCI6IiIsImZlY2hhX2xpbWl0ZSI6IiJ9LHsibnVtZXJvX29jIjoiNDU3NzAiLCJwcm9kdWN0byI6IkJveWEgRVZBIDMwMCBtbSIsInZhcmlhbnRlIjoiIiwiY2FudGlkYWQiOiIiLCJmZWNoYV9saW1pdGUiOiIifV19
```

**Esperado:** sin folio, sin caja de correo, sin casilla «parcial»; etiqueta
`¿RECIBIÓ CONFORME?` con `✅ Sí, conforme` / `❌ No, hay problema`.

Con la 1.ª en «Sí, conforme» + atajo *Hoy*, y la 2.ª en «No, hay problema» + chip
«Sin stock», el cuerpo sale **idéntico al de producción, clave por clave y en el mismo
orden**:

```json
{"proveedor":"Comercial Antigua Ltda.","correo_proveedor":"ventas@antigua.cl","telefono_vendedor":"+56912345678","correo_vendedor":"vbecerra@maryun.cl","fecha_respuesta":"…","correo_proveedor_alerta":"vbecerra@maryun.cl","numero_vendedor":"V-001","respuestas":[{"numero_oc":"45830","producto":"Cabo PP 16 mm","estado":"confirmado","fecha_entrega":"2026-08-21","observaciones":""},{"numero_oc":"45770","producto":"Boya EVA 300 mm","estado":"problema","fecha_entrega":"Por definir","observaciones":"Sin stock"}]}
```

Este es el **invariante duro**: es lo que el flujo legado y `Respuestas_OC.xlsx` esperan hoy.
Si cambia, se rompe la compatibilidad hacia atrás.

### (d) Hash corrupto — panel de error

```
eyJzZWciOiJTRUctMjAyNi0wMDIxNCIsInByb3ZlZWRvciI6IlNlw7FvcsOtbyIsIm9yZGVuZXMiOlt7Im51bWVyb19vYyI6Ij
```

**Esperado:** «No pudimos leer este enlace» y **nada más** en pantalla —se ocultan aviso,
barra de proveedor, sección de OC y botón de envío—. Nunca se muestra el demo con un hash
presente.

### (e) Folio inválido → cae a modo legado

```
eyJzZWciOiJTRUctMjYtMjE0IiwicHJvdmVlZG9yIjoiUHJvdmVlZG9yIENvbiBGb2xpbyBNYWxvIiwib3JkZW5lcyI6W3sibnVtZXJvX29jIjoiNDU4MzAiLCJwcm9kdWN0byI6IkNhYm8gUFAifV19
```

**Esperado:** interfaz «¿Recibió conforme?», sin badge de folio, destino `POWER_AUTOMATE_URL`.
En consola queda el aviso de que el folio no cumple el patrón.

### (f) Forma vieja `ocs` con ítems string + una OC sin número

```
eyJwcm92ZWVkb3IiOiJQcm92ZWVkb3IgRm9ybWF0byBWaWVqbyIsIm9jcyI6WyI0NTgzMCIsIjQ1NzcwIix7InByb2R1Y3RvIjoiU2luIG7Dum1lcm8gZGUgT0MiLCJjYW50aWRhZCI6IjUgdW4ifV19
```

**Esperado:** 2 tarjetas, y el panel naranja de advertencia: *«El enlace traía 3 línea(s) y 1
no se pudo(pudieron) mostrar porque venía(n) sin número de OC…»*. Una OC nunca se descarta
en silencio.

### (g) Base64 de longitud inválida (`%4 == 1`) — rechazo estricto

```
eyJwcm92ZWVkb3IiOiJ4Iiwib3JkZW5lcyI6W3sibnVtZXJvX29jIjoiMSJ9XX0AA
```

**Esperado:** panel de error (no se intenta decodificar a medias).

### (h) Sin hash

* En `localhost` / `file://` → demo con la banda «MODO DEMOSTRACIÓN».
* En `http://127.0.0.2:8765/index.html` (producción) → panel «Falta la información del enlace».

---

## Comprobaciones que no dependen del hash

| Qué | Cómo | Esperado |
|---|---|---|
| **`hashchange`** | Con la pestaña abierta, `location.hash = '<otro hash>'` desde la consola | La página **se recarga** y muestra las OC del enlace nuevo (antes se quedaba con las del anterior) |
| **Inyección desde la URL** | Payload con `proveedor: "<img src=x onerror=alert(1)>"` | Se ve el texto literal; 0 nodos inyectados (todo va por `textContent`) |
| **Chips** | Marcar y desmarcar los 5 | El textarea queda vacío; ninguna línea huérfana |
| **Chips vs. texto del proveedor** | Escribir «Confirmo que no hay retraso alguno» y alternar «Retraso» | La línea del proveedor **sobrevive** (antes se borraba) |
| **Guarda de truncado** | Llenar observaciones a 295 caracteres y pulsar un chip | Avisa que no queda espacio y **no** agrega media línea |
| **Fecha opcional (solo legado)** | Chip «Cuenta bloqueada» o «Sin stock» | El campo fecha y sus atajos se deshabilitan, el `*` desaparece y el payload lleva `"Por definir"` |
| **Formato de fecha** | Navegador sin `type=date`: escribir «22 de agosto» | Se rechaza pidiendo `aaaa-mm-dd` |
| **Accesibilidad** | Tabular por el formulario | Foco visible de 3 px; radios enfocables y operables con flechas; `role="radiogroup"`; sin ids duplicados; sin `<div>` dentro de `<label>` |
| **Móvil** | Viewport 375 px | Sin desborde horizontal; radios/chips/atajos de 44 px; campos a 16 px (iOS no hace zoom) |

### Envío observable

`enviarConReintento(url, cuerpo)` — verificable en consola sustituyendo `window.fetch`:

| Escenario | Intentos | Resultado | Pie de la pantalla verde |
|---|---|---|---|
| A responde 2xx con CORS | 1 (`application/json`) | `confirmado` | «Recepción confirmada por nuestro sistema» |
| A rechazado (CORS/red) | 2 → B es `no-cors` + `text/plain`, **mismo cuerpo** | `enviado` | «…No pudimos leer la confirmación del servidor» + botón copiar |
| A responde 4xx/5xx | 1 (no reintenta: es un NO del servidor) | `fallido` | «No pudimos confirmar la recepción automática» + botón copiar |
| A y B fallan | 2 | `fallido` | idem |
| URL de destino vacía | **0 peticiones** | `fallido` | idem |

El **intento B es exactamente lo que hace hoy producción** (`mode:"no-cors"`,
`Content-Type: text/plain`, mismo `JSON.stringify(payload)`), más un tope de 15 s.
`fecha_respuesta` se genera **una sola vez** antes de los dos intentos, así que si alguna vez
llegaran los dos cuerpos son reconocibles como el mismo envío.

---

## Validación de sintaxis

```sh
# extraer el <script> inline y verificarlo
node --check <(sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d')
```

## Antes de publicar

- [ ] `POWER_AUTOMATE_URL` conserva la firma completa (`sp` / `sv` / `sig`) y **no** lleva el
      segmento `/cu/04/`. Está verificada con un POST real: responde **HTTP 202**.
- [ ] El payload legado conserva `correo_vendedor` y `telefono_vendedor` con esos nombres
      exactos, y `fecha_entrega` cae a `"Por definir"`.
- [ ] `URL_SEGUIMIENTO` sigue vacía mientras el flujo **F10** no exista.
- [ ] Cero dependencias externas: sin `src`/`href` remotos, sin CDN, sin fuentes.
- [ ] `proceso.jpeg` sigue en el repo (lo usa el correo, no el HTML) y el HTML **no** lo
      referencia.
