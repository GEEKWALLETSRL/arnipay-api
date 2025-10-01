# Documentación de la API de Enlaces de Pago

[English Version](../payment-links.md)

Esta documentación proporciona detalles para utilizar la API de Enlaces de Pago para crear y gestionar enlaces de pago de forma programática.

## Autenticación

Todas las solicitudes a la API requieren autenticación basada en firma utilizando sus credenciales de Comercio. Los siguientes encabezados deben incluirse con cada solicitud:

| Nombre del Encabezado | Descripción |
|-------------|-------------|
| `X-Client-ID` | Su ID de cliente de Comercio (formato UUID) |
| `X-Timestamp` | Marca de tiempo Unix actual como entero (segundos desde el 1 de enero de 1970 UTC) |
| `X-Signature` | Firma HMAC-SHA256 para verificación de solicitud |

Las solicitudes expiran a los 15 minutos desde el valor indicado en `X-Timestamp`.

**Cadena canónica y generación de firma (usada para API y webhooks):**

1. Construya los componentes canónicos:
   1. Método HTTP en mayúsculas (por ejemplo `GET`, `POST`)
   2. URI (solo path + query; sin esquema/host), p. ej. `/api/v1/payment?id=123`
   3. Marca de tiempo Unix (mismo entero que en el encabezado `X-Timestamp`)
   4. Identificador estable (mismo valor que `X-Client-ID`)
   5. Hash SHA-256 del cuerpo en base64: `base64(sha256(raw_body))`. Para solicitudes sin cuerpo use `base64(sha256(""))`

2. Una los componentes con saltos de línea `"\n"` para formar la cadena canónica.

3. Calcule la firma con HMAC-SHA256 usando su clave secreta:
   ```php
   $metodo = strtoupper($metodoSolicitud);
   $uri = $rutaConQuery; // path + query, sin esquema/host
   $marcaTiempo = (string) $timestampUnix;
   $idCliente = $xClientId;
   $cuerpoCrudo = $cuerpoCrudoSolicitud; // exactamente como se envía por la red
   $hashCuerpo = base64_encode(hash('sha256', $cuerpoCrudo, true));
   $canonica = implode("\n", [$metodo, $uri, $marcaTiempo, $idCliente, $hashCuerpo]);
   $firma = hash_hmac('sha256', $canonica, $clavePrivada);
   ```

4. Envíe y firme cuerpos JSON sin escapes extra usando `json_encode` con `JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE` para que el cuerpo crudo coincida con lo que verifica el servidor.

**Notas de Seguridad:**
- Las solicitudes son válidas solo dentro de los 15 minutos del `X-Timestamp` para prevenir ataques de repetición
- Su secreto nunca se transmite por la red
- El hash del cuerpo garantiza la integridad del payload incluso con encabezados idénticos

Puede encontrar o regenerar su ID de cliente y clave privada en su configuración de Comercio.

## Endpoints de la API

### Crear un Enlace de Pago

Crea un nuevo enlace de pago asociado a su cuenta de Comercio.

**Endpoint:** `POST https://arnipay.com.py/api/v1/payment`

**Encabezados:**
- `X-Client-ID`: Su ID de cliente de Comercio
- `X-Timestamp`: Marca de tiempo Unix actual (segundos)
- `X-Signature`: Firma de la solicitud (HMAC-SHA256 sobre cadena canónica)
- `Content-Type: application/json`

**Parámetros del Cuerpo de la Solicitud:**

| Parámetro | Tipo | Requerido | Descripción |
|-----------|------|----------|-------------|
| `price` | número | Sí | El precio del ítem (mínimo 1) |
| `title` | cadena | Sí | Título del enlace de pago (máx 255 caracteres) |
| `description` | cadena | No | Descripción opcional del enlace de pago |
| `image` | cadena (URL) | No | URL opcional a una imagen para el enlace de pago |
| `payment_methods` | array de cadenas | No | Lista opcional de métodos de pago (si se omite, se permiten todos los métodos) |
| `reference` | cadena | No | Código de referencia opcional (máx 255 caracteres) |
| `start_date` | fecha | No | Fecha opcional en la que el enlace se activa |
| `expiration_date` | fecha | No | Fecha de vencimiento opcional (debe ser posterior a start_date) |
| `approved_redirection_url` | cadena (URL) | No | URL opcional para redirigir después de un pago exitoso |
| `failed_redirection_url` | cadena (URL) | No | URL opcional para redirigir después de un pago fallido |
| `process_redirection_url` | cadena (URL) | No | URL opcional para redirigir durante el procesamiento del pago |

**Ejemplo de Solicitud:**

```json
{
  "price": 150000,
  "title": "Suscripción Premium",
  "description": "Acceso de 1 año a todo el contenido premium",
  "payment_methods": ["qr", "tigo"],
  "reference": "SUB-2025",
  "approved_redirection_url": "https://example.com/success",
  "failed_redirection_url": "https://example.com/failed"
}
```

**Respuesta Exitosa (201 Created):**

```json
{
  "status": "success",
  "message": "Enlace de pago creado exitosamente",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "url": "https://arnipay.com.py/checkout/550e8400-e29b-41d4-a716-446655440000",
    "commerce_id": 123,
    "title": "Suscripción Premium",
    "price": 150000,
    "created_at": "2025-03-10T09:00:00Z"
  }
}
```

**Respuestas de Error:**

- `401 Unauthorized`: Credenciales de autenticación inválidas o faltantes
- `422 Unprocessable Entity`: Errores de validación en los datos de la solicitud

### Obtener un Enlace de Pago Específico

Recupera información detallada sobre un enlace de pago específico.

**Endpoint:** `GET https://arnipay.com.py/api/v1/payment/{id}`

**Encabezados:**
- `X-Client-ID`: Su ID de cliente de Comercio
- `X-Timestamp`: Marca de tiempo Unix actual (segundos)
- `X-Signature`: Firma de la solicitud (HMAC-SHA256 sobre cadena canónica)

**Parámetros:**
- `id`: El UUID del enlace de pago

**Respuesta Exitosa (200 OK):**

```json
{
  "status": "success",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "url": "https://yourdomain.com/checkout/550e8400-e29b-41d4-a716-446655440000",
    "commerce_id": 123,
    "title": "Suscripción Premium",
    "price": 150000,
    "description": "Acceso de 1 año a todo el contenido premium",
    "enabled": true,
    "payment_methods": ["qr", "tigo"],
    "reference": "SUB-2025",
    "stock": null,
    "quantity": null,
    "start_date": null,
    "expiration_date": null,
    "created_at": "2025-03-10T09:00:00Z",
    "updated_at": "2025-03-10T09:00:00Z",
    "is_paid": false
  }
}
```

**Respuestas de Error:**

- `401 Unauthorized`: Credenciales de autenticación inválidas o faltantes
- `404 Not Found`: Enlace de pago no encontrado

## Manejo de Errores

La API devuelve códigos de estado HTTP estándar para indicar éxito o fracaso:

- `200 OK`: Solicitud exitosa (para solicitudes GET)
- `201 Created`: Recurso creado exitosamente (para solicitudes POST)
- `401 Unauthorized`: Autenticación fallida
- `404 Not Found`: Recurso solicitado no encontrado
- `422 Unprocessable Entity`: Errores de validación
- `500 Internal Server Error`: Error del lado del servidor

Las respuestas de error incluyen un cuerpo JSON con detalles:

```json
{
  "status": "error",
  "message": "Descripción del mensaje de error",
  "errors": {
    "nombre_del_campo": ["Mensaje de error de validación"]
  }
}
```

## Paginación

Los endpoints de lista pueden implementar paginación en el futuro. La implementación actual devuelve todos los resultados sin paginación.

## Gestión de Enlaces de Pago

- Los enlaces creados a través de la API tendrán un campo `source` establecido en `"api"`.
- Estos enlaces son completamente funcionales pero no se muestran en la interfaz de usuario para evitar desorden.
- Los pagos para enlaces creados por API serán visibles en el historial de actividad/pago con una insignia de API.

## Notificaciones Webhook

Nuestro sistema puede notificar a su aplicación sobre eventos de pago en tiempo real utilizando notificaciones webhook. Esto permite que su aplicación reciba actualizaciones sobre pagos sin consultar la API constantemente.

### Eventos de Webhook

Los siguientes eventos desencadenan notificaciones webhook:

| Evento | Descripción |
|-------|-------------|
| `payment.completed` | Un pago se ha completado exitosamente |
| `payment.failed` | Un pago ha fallado |
| `payment.pending` | Un pago está pendiente de procesamiento |

### Formato de Carga Útil del Webhook

Las notificaciones webhook se envían como solicitudes HTTP POST a su URL de webhook configurada con una carga útil JSON:

```json
{
  "event": "payment.completed",
  "timestamp": "2025-03-10T15:30:45Z",
  "data": {
    "link_id": "550e8400-e29b-41d4-a716-446655440000",
    "payment_id": "12345",
    "status": "paid",
    "payment_method": "qr",
    "amount": 150000,
    "payment_details": {
      "payment_date": "2025-03-10T15:30:40Z"
    }
  }
}
```

### Configuración de Webhook

Para recibir notificaciones webhook, configure su URL de webhook y ajustes en su configuración de Comercio. Las siguientes configuraciones están disponibles:

#### Ajustes Requeridos

- **URL de Webhook**: El endpoint en su servidor que recibirá las notificaciones webhook.
- **Secreto de Webhook**: Una clave secreta utilizada para verificar que las notificaciones webhook provienen de nuestro sistema.

#### Ajustes Opcionales

- **Máximo de Intentos de Webhook**: El número de veces que el sistema intentará entregar un webhook en caso de fallo (predeterminado: 5, máximo: 10).

### Seguridad de Webhook

Las solicitudes de webhook se firman usando las mismas reglas de cadena canónica que las solicitudes de API e incluyen estos encabezados:

- `X-Client-ID`
- `X-Timestamp`
- `X-Signature`
- `X-Webhook-ID` (identificador único del webhook entregado)

Para verificar un webhook:

1. Lea el cuerpo crudo exactamente como se recibió y calcule `base64(sha256(raw_body))`
2. Construya la cadena canónica con: método HTTP en mayúsculas, URI (path + query), `X-Timestamp`, `X-Client-ID` y el hash del cuerpo en base64
3. Calcule el HMAC-SHA256 con su secreto de webhook y compare con `X-Signature`

Ejemplo de verificación en PHP:

```php
$cuerpoCrudo = file_get_contents('php://input');
$timestamp = $_SERVER['HTTP_X_TIMESTAMP'] ?? '';
$clientId = $_SERVER['HTTP_X_CLIENT_ID'] ?? '';
$firma = $_SERVER['HTTP_X_SIGNATURE'] ?? '';
$metodo = strtoupper($_SERVER['REQUEST_METHOD'] ?? 'POST');
$uri = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
$query = $_SERVER['QUERY_STRING'] ?? '';
if ($query !== '') {
    $uri .= '?' . $query;
}
$hashCuerpo = base64_encode(hash('sha256', $cuerpoCrudo, true));
$canonica = implode("\n", [$metodo, $uri, $timestamp, $clientId, $hashCuerpo]);
$esperada = hash_hmac('sha256', $canonica, $webhookSecret);

if (!hash_equals($esperada, $firma)) {
    http_response_code(403);
    exit('Firma inválida');
}
```

### Mecanismo de Reintento de Webhook

Si su servidor responde con un código de estado que no es 2xx, reintentaremos enviar la notificación webhook de acuerdo con su máximo de intentos configurado. Los reintentos siguen una estrategia de retroceso exponencial:

- 1er reintento: 1 minuto después del intento inicial
- 2do reintento: 5 minutos después del 1er reintento
- 3er reintento: 15 minutos después del 2do reintento
- 4to reintento: 30 minutos después del 3er reintento
- 5to reintento: 60 minutos después del 4to reintento

### Mejores Prácticas para Webhooks

1. **Verificar Firmas**: Siempre verifique las firmas de webhook para garantizar la autenticidad.
2. **Procesar de Forma Idempotente**: Diseñe su manejador de webhook para ser idempotente, ya que la misma notificación podría entregarse varias veces.
3. **Responder Rápidamente**: Su endpoint de webhook debe responder en unos pocos segundos para evitar tiempos de espera.
4. **Usar HTTPS**: Siempre use HTTPS para su URL de webhook para garantizar una comunicación segura.
5. **Implementar Manejo de Errores**: Registre cualquier error para propósitos de depuración, pero responda con un código de estado exitoso si ha recibido el webhook (incluso si encontró errores al procesarlo).

## Uso en Producción

Para uso en producción:
1. Almacene de forma segura su client_id y private_key
2. Implemente un manejo adecuado de errores
3. Considere implementar límites de tasa en su lado para evitar sobrecargar la API
4. Siempre valide el estado de éxito en la respuesta 
