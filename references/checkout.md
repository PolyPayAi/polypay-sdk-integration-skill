# Checkout and Orders

Check current official documentation at `https://polypay.ai/en/docs/javascript-sdk` and `https://polypay.ai/en/docs/php-sdk` when package types or endpoints may have changed.

## Select the API Surface

Use hosted checkout for the default customer-payment path.

| Host | Normal checkout | Direct order |
| --- | --- | --- |
| Browser JavaScript | `@polypay/sdk/browser` with Public Key and signed parameters | `PolyPayClient` with Public Key and short-lived session token |
| PHP server | `polypay/php-sdk` `createCheckoutUrl()` | `createOrder()` |
| Node.js server | PolyPay REST API with server-only API Key | PolyPay REST API with server-only API Key |

Do not pass an API Key to `PolyPayClient`; it accepts `publicKey` and runs only in a browser. Use `@polypay/sdk/x402` only for Node.js x402 server routes.

## Implement Browser Hosted Checkout

1. Generate or retrieve signed checkout parameters on a trusted server.
2. Pass `publicKey`, `amount`, `timestamp`, `signature`, and stable `orderId` to the browser.
3. Pass `notifyUrl` for server-to-server state updates and `redirectUrl` only for user navigation.
4. Call `PolyPayCheckout.redirectToHostedCheckout()` from a user-triggered browser action.
5. Omit currency and network to let PolyPay show payment-method selection. Include both only when already selected.

The JavaScript SDK requires `signature`; never substitute a placeholder in finished code. Its default URL is `https://checkout.polypay.ai/checkout`, which detects browser language and falls back to English. Pass an explicit locale only to pin `https://checkout.polypay.ai/{locale}/checkout`.

## Implement Server API Key Checkout

Keep the API Key in a server-only environment variable. For PHP, prefer the SDK rather than hand-written HTTP. For Node.js, call the documented REST API because the browser client is not a general API Key client.

Use these API Key endpoints when direct REST access is necessary:

- `POST /api/v1/pay/sdk/order/checkout` for a signed hosted checkout URL.
- `POST /api/v1/pay/sdk/order/add` for a direct order when currency and network are known.
- `POST /api/v1/pay/sdk/order/detail` for authenticated reconciliation.

Authenticate server REST requests with `X-API-Key`. Preserve the returned `trade_id`, `payment_url`, merchant order ID, and expiry data in the host application's existing order model.

## Use Sandbox First

- Create a Sandbox API Key with the `sk_sandbox_` prefix.
- Keep the API host unchanged; the key selects Sandbox versus Production.
- Configure a Sandbox webhook separately.
- Create an order and simulate payment states in the merchant dashboard.
- Test paid, expired, cancelled, invalid callback, and duplicate callback paths.
- Never copy a Sandbox key into browser code.

Browser Public Key Mode is intended for production checkout. Prefer API Key Sandbox flows for repeatable integration tests.

## Production Gates

- Require a unique merchant order ID no longer than the product limit.
- Validate amount, URLs, currency, and network before calling PolyPay.
- Prevent duplicate checkout creation for the same logical attempt unless retry behavior is intentional.
- Store amounts with decimal-safe types.
- Do not treat polling or redirect results as authoritative paid state.
