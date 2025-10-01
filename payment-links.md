# Payment Links API Documentation

[Versión en Español](es/payment-links.md)

This documentation provides details for using the Payment Links API to create and manage payment links programmatically.

## Authentication

All API requests require signature-based authentication using your Commerce credentials. The following headers must be included with every request:

| Header Name | Description |
|-------------|-------------|
| `X-Client-ID` | Your Commerce client ID (UUID format) |
| `X-Timestamp` | Current Unix timestamp as an integer (seconds since January 1, 1970 UTC) |
| `X-Signature` | HMAC-SHA256 signature for request verification |

Requests expire 15 minutes from the time indicated by `X-Timestamp`.

**Canonical string and signature generation (used for both API and webhooks):**

1. Build the canonical components:
   1. Uppercased HTTP method (e.g. `GET`, `POST`)
   2. URI (path + query only; no scheme/host), e.g. `/api/v1/payment?id=123`
   3. Unix timestamp (same integer value as in the `X-Timestamp` header)
   4. Stable identifier (same value as `X-Client-ID`)
   5. Base64-encoded SHA-256 hash of the raw request body: `base64(sha256(raw_body))`. For requests without a body use `base64(sha256(""))`

2. Join the components with newline characters `"\n"` to form the canonical string.

3. Compute the signature with HMAC-SHA256 using your secret key:
   ```php
   $method = strtoupper($requestMethod);
   $uri = $requestPathWithQuery; // path + query, no scheme/host
   $timestamp = (string) $unixTimestamp;
   $clientId = $xClientId;
   $rawBody = $requestRawBody; // exactly as sent over the wire
   $bodyHash = base64_encode(hash('sha256', $rawBody, true));
   $canonical = implode("\n", [$method, $uri, $timestamp, $clientId, $bodyHash]);
   $signature = hash_hmac('sha256', $canonical, $privateKey);
   ```

4. Send JSON bodies without extra escaping using `json_encode` with `JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE` so the raw body matches what the server verifies.

**Security Notes:**
- Requests are valid only within 15 minutes of the `X-Timestamp` to prevent replay attacks
- Your secret is never transmitted over the network
- The body hash ensures integrity of the payload even with identical headers

You can find or regenerate your client ID and private key in your Commerce settings.

## API Endpoints

### Create a Payment Link

Creates a new payment link associated with your Commerce account.

**Endpoint:** `POST https://arnipay.com.py/api/v1/payment`

**Headers:**
- `X-Client-ID`: Your Commerce client ID
- `X-Timestamp`: Current Unix timestamp (seconds)
- `X-Signature`: Request signature (HMAC-SHA256 over canonical string)
- `Content-Type: application/json`

**Request Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `price` | number | Yes | The price of the item (minimum 1) |
| `title` | string | Yes | Title of the payment link (max 255 chars) |
| `description` | string | No | Optional description of the payment link |
| `image` | string (URL) | No | Optional URL to an image for the payment link |
| `payment_methods` | array of strings | No | Optional list of payment methods (if omitted, all methods are allowed) |
| `reference` | string | No | Optional reference code (max 255 chars) |
| `start_date` | date | No | Optional date when the link becomes active |
| `expiration_date` | date | No | Optional expiration date (must be after start_date) |
| `approved_redirection_url` | string (URL) | No | Optional URL to redirect after successful payment |
| `failed_redirection_url` | string (URL) | No | Optional URL to redirect after failed payment |
| `process_redirection_url` | string (URL) | No | Optional URL to redirect during payment processing |

**Example Request:**

```json
{
  "price": 150000,
  "title": "Premium Subscription",
  "description": "1 year access to all premium content",
  "payment_methods": ["qr", "tigo"],
  "reference": "SUB-2025",
  "approved_redirection_url": "https://example.com/success",
  "failed_redirection_url": "https://example.com/failed"
}
```

**Success Response (201 Created):**

```json
{
  "status": "success",
  "message": "Payment link created successfully",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "url": "https://arnipay.com.py/checkout/550e8400-e29b-41d4-a716-446655440000",
    "commerce_id": 123,
    "title": "Premium Subscription",
    "price": 150000,
    "created_at": "2025-03-10T09:00:00Z"
  }
}
```

**Error Responses:**

- `401 Unauthorized`: Invalid or missing authentication credentials
- `422 Unprocessable Entity`: Validation errors in the request data

### Get a Specific Payment Link

Retrieves detailed information about a specific payment link.

**Endpoint:** `GET https://arnipay.com.py/api/v1/payment/{id}`

**Headers:**
- `X-Client-ID`: Your Commerce client ID
- `X-Timestamp`: Current Unix timestamp (seconds)
- `X-Signature`: Request signature (HMAC-SHA256 over canonical string)

**Parameters:**
- `id`: The UUID of the payment link

**Success Response (200 OK):**

```json
{
  "status": "success",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "url": "https://yourdomain.com/checkout/550e8400-e29b-41d4-a716-446655440000",
    "commerce_id": 123,
    "title": "Premium Subscription",
    "price": 150000,
    "description": "1 year access to all premium content",
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

**Error Responses:**

- `401 Unauthorized`: Invalid or missing authentication credentials
- `404 Not Found`: Payment link not found

## Error Handling

The API returns standard HTTP status codes to indicate success or failure:

- `200 OK`: Request successful (for GET requests)
- `201 Created`: Resource created successfully (for POST requests)
- `401 Unauthorized`: Authentication failed
- `404 Not Found`: Requested resource not found
- `422 Unprocessable Entity`: Validation errors
- `500 Internal Server Error`: Server-side error

Error responses include a JSON body with details:

```json
{
  "status": "error",
  "message": "Error message description",
  "errors": {
    "field_name": ["Validation error message"]
  }
}
```

## Pagination

List endpoints may implement pagination in the future. The current implementation returns all results without pagination.

## Payment Link Management

- Links created via the API will have a `source` field set to `"api"`.
- These links are fully functional but are not displayed in the UI interface to prevent cluttering.
- Payments for API-created links will be visible in the activity/payment history with an API badge.

## Webhook Notifications

Our system can notify your application about payment events in real-time using webhook notifications. This allows your application to receive updates about payments without polling the API.

### Webhook Events

The following events trigger webhook notifications:

| Event | Description |
|-------|-------------|
| `payment.completed` | A payment has been successfully completed |
| `payment.failed` | A payment has failed |
| `payment.pending` | A payment is pending processing |

### Webhook Payload Format

Webhook notifications are sent as HTTP POST requests to your configured webhook URL with a JSON payload:

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

### Webhook Configuration

To receive webhook notifications, configure your webhook URL and settings in your Commerce settings. The following configurations are available:

#### Required Settings

- **Webhook URL**: The endpoint on your server that will receive webhook notifications.
- **Webhook Secret**: A secret key used to verify that webhook notifications are coming from our system.

#### Optional Settings

- **Webhook Max Attempts**: The number of times the system will attempt to deliver a webhook in case of failure (default: 5, max: 10).

### Webhook Security

Webhook requests are signed using the same canonical string rules as API requests and include these headers:

- `X-Client-ID`
- `X-Timestamp`
- `X-Signature`
- `X-Webhook-ID` (unique identifier for the webhook delivery)

To verify a webhook:

1. Read the raw body exactly as received and compute `base64(sha256(raw_body))`
2. Build the canonical string using: uppercased HTTP method, request URI (path + query), `X-Timestamp`, `X-Client-ID`, and the base64 body hash
3. Compute the HMAC-SHA256 with your webhook secret and compare to `X-Signature`

Example verification in PHP:

```php
$rawBody = file_get_contents('php://input');
$timestamp = $_SERVER['HTTP_X_TIMESTAMP'] ?? '';
$clientId = $_SERVER['HTTP_X_CLIENT_ID'] ?? '';
$signature = $_SERVER['HTTP_X_SIGNATURE'] ?? '';
$method = strtoupper($_SERVER['REQUEST_METHOD'] ?? 'POST');
$uri = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
$query = $_SERVER['QUERY_STRING'] ?? '';
if ($query !== '') {
    $uri .= '?' . $query;
}
$bodyHash = base64_encode(hash('sha256', $rawBody, true));
$canonical = implode("\n", [$method, $uri, $timestamp, $clientId, $bodyHash]);
$expected = hash_hmac('sha256', $canonical, $webhookSecret);

if (!hash_equals($expected, $signature)) {
    http_response_code(403);
    exit('Invalid signature');
}
```

### Webhook Retry Mechanism

If your server responds with a non-2xx status code, we will retry sending the webhook notification according to your configured max attempts. Retries follow an exponential backoff strategy:

- 1st retry: 1 minute after initial attempt
- 2nd retry: 5 minutes after 1st retry
- 3rd retry: 15 minutes after 2nd retry
- 4th retry: 30 minutes after 3rd retry
- 5th retry: 60 minutes after 4th retry

### Webhook Best Practices

1. **Verify Signatures**: Always verify webhook signatures to ensure authenticity.
2. **Process Idempotently**: Design your webhook handler to be idempotent, as the same notification might be delivered multiple times.
3. **Respond Quickly**: Your webhook endpoint should respond within a few seconds to avoid timeouts.
4. **Use HTTPS**: Always use HTTPS for your webhook URL to ensure secure communication.
5. **Implement Error Handling**: Log any errors for debugging purposes but respond with a successful status code if you've received the webhook (even if you encountered errors processing it).

## Using in Production

For production use:
1. Securely store your client_id and private_key
2. Implement proper error handling
3. Consider implementing rate limiting on your side to prevent overloading the API
4. Always validate the success status in the response 
