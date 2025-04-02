# API Documentation

[Versión en Español](es/README.md)

Welcome to the API documentation for our payment processing system.

## Available Documentation

- [Payment Links API Documentation](payment-links.md) - Complete documentation of the Payment Links API endpoints, authentication, request/response formats, and best practices.
- [Payment Links Code Examples](payment-links-examples.md) - Practical code examples in multiple programming languages for integrating with the Payment Links API.

## Authentication

All API endpoints require authentication using your Commerce credentials:

- `X-Client-ID`: Your Commerce client ID (UUID format)
- `X-Private-Key`: Your Commerce private key

You can find these credentials in your Commerce settings page, or regenerate them if needed.

## API Base URL

The base URL for all API endpoints is:

```
https://arnipay.com.py/api/v1/
```

## Webhooks

Our payment system can send real-time notifications to your server using webhooks when payment events occur. Set up your webhook URL and configuration in the Commerce settings page to receive these notifications.

Key webhook features:
- Real-time payment notifications (completed, failed, pending)
- Secure delivery with HMAC-SHA256 signature verification
- Configurable retry mechanism
- Detailed event payloads

For detailed information about webhooks and implementation examples, see:
- [Webhook Configuration Documentation](payment-links.md#webhook-notifications)
- [Webhook Implementation Examples](payment-links-examples.md#webhook-examples)

## Rate Limiting

Our API implements rate limiting to protect the service. The current limits are:

- 60 requests per minute per client
- 1000 requests per day per client

If you exceed these limits, you'll receive a `429 Too Many Requests` response.

## Support

If you have any questions or need assistance integrating with our API, please contact our support team at api-support@example.com. 

## SDK

We provide official SDKs to help you integrate with our payment system quickly and easily in your preferred programming language:

Each SDK includes comprehensive documentation, examples, and error handling to streamline your integration process.

### JavaScript SDK

A JavaScript/TypeScript library with full TypeScript support for both browser and Node.js environments.

Our JavaScript SDK is available through the following channels:

- **GitHub**: [github.com/GEEKWALLETSRL/arnipay-sdk-js](https://github.com/GEEKWALLETSRL/arnipay-sdk-js)
- **npm**: Install via npm:
  ```bash
  npm install @geekwallet/gw-sdk-js
  ```

### PHP SDK

A PHP library that provides simple interfaces for creating payment links, fetching payment information, and handling webhooks.

Our PHP SDK is available through the following channels:

- **GitHub**: [github.com/GEEKWALLETSRL/arnipay-sdk-php](https://github.com/GEEKWALLETSRL/arnipay-sdk-php)
- **Composer**: Install via Composer:
  ```bash
  composer require geekwalletsrl/arnipay-sdk-php
  ```

