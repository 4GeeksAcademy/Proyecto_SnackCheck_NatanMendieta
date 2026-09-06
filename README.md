# SnackCheck

SnackCheck es un flujo automatizado desarrollado en **n8n** que permite consultar y evaluar la información nutricional de un producto a partir de su código de barras.

El flujo recibe un código de barras mediante un webhook y utiliza la API pública de **Open Food Facts** para obtener datos como:

* Nombre y marca del producto.
* Nutri-Score.
* Calorías, grasas, azúcares, proteínas y sal.
* Información nutricional disponible por cada 100 g.

A partir de estos datos, SnackCheck genera un veredicto nutricional sencillo y en español. Si el producto no existe, el código de barras es inválido o la información disponible es insuficiente, el flujo devuelve un mensaje explicativo.

## Funcionamiento general

El proceso sigue estos pasos:

1. El usuario envía un código de barras al webhook de n8n.
2. El flujo valida que el código tenga un formato correcto.
3. n8n consulta el producto en Open Food Facts.
4. Se comprueba si existen datos nutricionales suficientes.
5. El flujo analiza la información obtenida.
6. Se devuelve un veredicto o un mensaje de error en formato JSON.

## Configuración en n8n

El workflow debe incluir, como mínimo, los siguientes nodos:

1. **Webhook:** recibe el código de barras mediante una solicitud `POST`.
2. **Validación:** comprueba que el código de barras tenga un formato válido.
3. **HTTP Request:** consulta la API de Open Food Facts.
4. **Procesamiento:** analiza los datos nutricionales y genera el resultado.
5. **Respond to Webhook:** devuelve la respuesta final al usuario.

## Endpoints

### URL de prueba

Esta dirección se utiliza mientras el workflow se encuentra en modo de prueba dentro de n8n:

```http
POST https://<tu-n8n>/webhook-test/nutrition-check
```

Antes de enviar la solicitud, se debe seleccionar **Listen for test event** o **Execute workflow** en n8n.

### URL activa

Esta dirección se utiliza cuando el workflow está activado:

```http
POST https://<tu-n8n>/webhook/nutrition-check
```

Para utilizarla, el workflow debe estar guardado y marcado como **Active** en n8n.

## Cuerpo de la solicitud

La solicitud debe incluir el código de barras del producto en formato JSON:

```json
{
  "barcode": "8076809513753"
}
```

También debe enviarse el siguiente encabezado:

```http
Content-Type: application/json
```

## Configuración del nodo HTTP Request

El nodo **HTTP Request** debe consultar la API de Open Food Facts utilizando el código recibido por el webhook.

### Método

```http
GET
```

### URL

```text
https://world.openfoodfacts.org/api/v2/product/{{ $json.body.barcode }}.json?fields=product_name,brands,nutriscore_grade,nutriments
```

Dependiendo de la estructura de los nodos anteriores, el código también podría obtenerse mediante:

```javascript
{{ $json.barcode }}
```

La consulta no necesita autenticación ni credenciales.

## Guion de pruebas y evidencias

### TC-001 — Consulta funcional

**Solicitud:**

```json
{
  "barcode": "8076809513753"
}
```

**Resultado esperado:**

* Código de estado: `200 OK`.
* La respuesta contiene el nombre del producto.
* La respuesta contiene un veredicto nutricional en español.

---

### TC-002 — Datos nutricionales insuficientes

**Solicitud:**

```json
{
  "barcode": "8859287400223"
}
```

**Resultado esperado:**

* Código de estado: `400 Bad Request`.
* La respuesta informa que no existen datos suficientes.
* La respuesta contiene una sugerencia en español.

---

### TC-003 — Producto no encontrado

**Solicitud:**

```json
{
  "barcode": "123456789"
}
```

**Resultado esperado:**

* Código de estado: `400 Bad Request`.
* La respuesta informa que el producto no fue encontrado.
* La respuesta contiene una sugerencia en español.

---

### TC-004 — Formato de código de barras inválido

**Solicitud:**

```json
{
  "barcode": "12345ABCDE"
}
```

**Resultado esperado:**

* Código de estado: `400 Bad Request`.
* La respuesta informa que el código de barras no tiene un formato válido.
* La respuesta contiene una sugerencia en español.

## API externa

SnackCheck utiliza la API pública de Open Food Facts:

```http
GET https://world.openfoodfacts.org/api/v2/product/{barcode}.json?fields=product_name,brands,nutriscore_grade,nutriments
```

Donde `{barcode}` debe reemplazarse por el código de barras que se desea consultar.

Por ejemplo:

```http
GET https://world.openfoodfacts.org/api/v2/product/8076809513753.json?fields=product_name,brands,nutriscore_grade,nutriments
```

La API devuelve una respuesta en formato JSON que posteriormente es procesada por los nodos del workflow.
