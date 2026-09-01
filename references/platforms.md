# Platform Guidance

## Next.js and React

- Use `@polypay/sdk/browser` only in client-side code to open a server-created `checkout_url`.
- Create Hosted Checkout with a server-only API Key in a route handler or server action.
- Keep API Key REST calls and `@polypay/sdk/x402` in server-only modules.
- Never place PolyPay payment credentials or order-creation parameters in public environment variables.
- Trigger redirects from explicit user actions and preserve the app's localization and error boundaries.

## PHP and Laravel

- Install `polypay/php-sdk` with Composer when available.
- Keep API Keys in environment-backed configuration.
- Use the SDK for hosted checkout, direct orders, reconciliation, webhook verification, and x402.
- Exempt only the callback route from CSRF and update orders inside database transactions.
- Use shared nonce storage for horizontally scaled webhook handlers.

## WordPress and WooCommerce

- Prefer the official PolyPay plugin and WordPress settings APIs.
- Store credentials as protected options and never render them in pages or logs.
- Preserve WooCommerce HPOS compatibility, order notes, callback conventions, and gateway status transitions.
- Support standalone shortcode behavior only when WooCommerce is not the requested path.

## WHMCS

- Prefer the official PolyPay gateway layout and WHMCS settings storage.
- Preserve invoice idempotency and callback validation before calling `addInvoicePayment` or equivalent state changes.
- Use a Sandbox API Key for test invoices and clear `templates_c/` after template or config changes when rendering is stale.

## Shopify

- Reuse the existing bridge service conventions when maintaining an installed integration.
- Verify Shopify Webhook HMAC separately from PolyPay callback verification.
- Prefer PolyPay hosted checkout for new flows instead of building another payment-method selector.
- Keep Shopify Admin tokens and PolyPay API Keys server-side, and make Shopify order-paid updates idempotent.

## Existing Applications

Follow the host application's router, dependency injection, HTTP client, order model, migrations, logging, localization, and test style. Modify the smallest complete slice; do not replace a working payment abstraction without explicit direction.
