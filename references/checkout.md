# Checkout and Orders

Check current official documentation at `https://polypay.ai/en/docs/javascript-sdk` and `https://polypay.ai/en/docs/php-sdk` when package types or endpoints may have changed.

## Select the API Surface

Use hosted checkout for the default customer-payment path.

| Host | Normal checkout | Direct order |
| --- | --- | --- |
| Browser JavaScript | `@polypay/sdk/browser` opens a server-created opaque `checkout_url` | Not supported; create orders on the merchant server |
| PHP server | `polypay/php-sdk` `createCheckoutUrl()` | `createOrder()` |
| Node.js server | PolyPay REST API with server-only API Key | PolyPay REST API with server-only API Key |

`PolyPayClient` and the browser order APIs were removed in JavaScript SDK 2.0. Use `@polypay/sdk/x402` only for Node.js x402 server routes.

## Implement Browser Hosted Checkout

1. Call `POST /api/v1/pay/order/checkout` on the merchant server with `X-API-Key`.
2. Set amount, merchant order ID, callback URL, redirect URL, locale, and any preselected payment method on the server.
3. Return only the opaque `checkout_url` to the browser.
4. Call `PolyPayCheckout.redirect(checkoutUrl)` or `openModal(checkoutUrl)` from a user-triggered action.
5. Confirm payment through a verified Webhook or authenticated server reconciliation.

Do not reconstruct, parse, modify, or persist secret-bearing parameters from the returned URL. The server API selects the locale; omitting it lets Hosted Checkout use the default language behavior.

## Implement Server API Key Checkout

Keep the API Key in a server-only environment variable. For PHP, prefer the SDK rather than hand-written HTTP. For Node.js, call the documented REST API because the browser client is not a general API Key client.

Use the canonical merchant endpoints for new server integrations:

- `POST /api/v1/pay/order/checkout` for a signed hosted checkout URL.
- `POST /api/v1/pay/order/add` for a direct order when currency and network are known.
- `POST /api/v1/pay/order/detail` for authenticated reconciliation. Provide at least one of `trade_id` or `mch_order_id`; when both are present, `trade_id` takes precedence.
- `POST /api/v1/pay/order/cancel` for cancelling a pending order. Provide at least one of `trade_id` or `mch_order_id`; when both are present, `trade_id` takes precedence. The API Key compatibility path is `POST /api/v1/pay/sdk/order/cancel`.

The legacy `POST /api/v1/pay/order/cancel-by-trade-id` and `/api/v1/pay/sdk/order/cancel-by-trade-id` paths remain available for existing integrations. Do not select them for new integrations.

Authenticate server REST requests with `X-API-Key`. Preserve the returned `trade_id`, `payment_url`, merchant order ID, and expiry data in the host application's existing order model.

## Implement Native Android or iOS Checkout

Use `PolyPayAi/android-sdk` for Kotlin/Android and `PolyPayAi/ios-sdk` for SwiftUI or UIKit when the user asks for native payment UI. The native SDKs contain both the payment-method selection page and the payment page; they do not embed Hosted Checkout in a WebView.

1. The merchant server calls `POST /api/v1/pay/order/checkout` with its server-only API Key and omits both `currency` and `network`.
2. Give the app only the returned HTTPS `checkout_url`, which must point to `/pay/{tradeId}` on an exact allowlisted checkout host.
3. The SDK uses public checkout-scoped endpoints to read the placeholder, load enabled payment methods, submit the selected currency/network, display exact payment details, and observe confirmation state. Android uses server-locked wallet parameters to open compatible EVM wallets and provides explicit address/amount copy actions; iOS currently retains its address QR and copy flow.
4. Android returns a `PolyPayCheckoutResult`; iOS emits a `PolyPayCheckoutOutcome`. Neither contract contains a `paid` outcome. Reconcile the trade ID on the merchant server before fulfillment.

Do not use a browser token flow in a native app, expose an API Key, or accept arbitrary API/checkout hosts. Never invent a payment URI for an unsupported network: use explicit manual address, amount, and network copy actions until a compatible native signing integration exists.

The canonical merchant business endpoints under `/api/v1/pay` accept either a merchant dashboard JWT or an API Key. Use `Authorization: Bearer <jwt>` for an interactive merchant dashboard session and `X-API-Key: <key>` for trusted server automation. Account identity and security operations remain JWT-only. The explicit `/api/v1/pay/sdk/...` API Key routes are legacy compatibility aliases; preserve them in existing integrations, but do not select them for new integrations when a canonical merchant endpoint exists.

## Synchronize Local Cancellations

Treat cancellation as part of the payment lifecycle, not as a local-only status change.

1. Preserve the mapping from the host order or payment attempt to PolyPay `trade_id` and `mch_order_id`.
2. When the host cancels, voids, or replaces an unpaid attempt, call `POST /api/v1/pay/order/cancel` from the trusted server with `X-API-Key` and either `{"trade_id":"..."}` or `{"mch_order_id":"..."}`. Do not send `actual_amount`; cancellation no longer uses an amount to locate the order.
3. Prefer `trade_id` because it identifies one PolyPay payment attempt precisely. Use `mch_order_id` when the host does not have the PolyPay trade ID.
4. Cancel only pending orders. If payment confirmation races with cancellation, preserve the verified paid state and continue the normal fulfillment or refund workflow instead of forcing cancelled locally.
5. If the cancellation response is lost, retried, or reports that the state already changed, query order detail and treat a remote cancelled state as converged, a paid state as authoritative, and an expired state as terminal.

PolyPay allocates an exact receiving wallet and payable amount to a pending order. If the host cancels only its local order, the obsolete PolyPay order can remain payable and its allocation can remain occupied until expiry. A later order for the same amount may therefore receive an adjusted payable amount; for stablecoin payments this is commonly an additional `0.01`, but the increment is configuration- and currency-dependent. Do not promise customers that the requested and payable amounts will always match.

## Use Sandbox First

- Create a Sandbox API Key with the `sk_sandbox_` prefix.
- Keep the API host unchanged; the key selects Sandbox versus Production.
- Configure a Sandbox webhook separately.
- Create an order and simulate payment states in the merchant dashboard.
- Test paid, expired, cancelled, invalid callback, and duplicate callback paths.
- Cancel an unpaid order through the API, then create another same-currency, same-network, same-amount order and verify that the cancelled order does not leave a stale allocation that changes the new payable amount.
- Never copy a Sandbox key into browser code.

Browser checkout uses the same server-created API Key flow in Sandbox and Production; the API Key environment selects the target.

## Production Gates

- Require a unique merchant order ID no longer than the product limit.
- Validate amount, URLs, currency, and network before calling PolyPay.
- Prevent duplicate checkout creation for the same logical attempt unless retry behavior is intentional.
- Synchronize local cancellation, void, and replacement events for unpaid attempts through the PolyPay cancellation API.
- Store amounts with decimal-safe types.
- Do not treat polling or redirect results as authoritative paid state.
