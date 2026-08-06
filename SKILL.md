---
name: polypay-sdk-integration
description: Integrate PolyPay payment acceptance and merchant automation into production applications. Use when Codex needs to add or repair PolyPay hosted checkout, JavaScript browser checkout, PHP SDK or server REST API integration, order cancellation or void synchronization, webhook verification, sandbox testing, x402 agent payments, Notification Center channels or templates, PolyPay MCP access, or WordPress, WooCommerce, WHMCS, and Shopify payment flows. Also trigger for legacy PonponPay integration requests.
---

# PolyPay Production Integration

Integrate the smallest complete PolyPay flow that matches the host application. Preserve existing architecture and prove the result without exposing credentials or charging real funds by default.

## Resolve the Integration Mode

Inspect the host framework, runtime, order model, existing payment abstraction, callback routes, package manager, and test commands before editing.

Treat these references as a tested capability snapshot. Before coding against a mutable API, compare them with the installed package types and current official PolyPay documentation. Prefer, in order: the target application's locked dependency types, current official documentation, then this Skill's references. If they conflict, follow the newer authoritative contract and report the Skill drift instead of guessing.

Choose one primary mode:

- Use hosted checkout for normal customer payments. Let PolyPay own payment-method selection unless the merchant already knows both currency and network.
- Use `@polypay/sdk/browser` for browser checkout with a Public Key and server-generated signed parameters.
- Use `polypay/php-sdk` for PHP server-side hosted checkout, direct orders, status queries, webhook verification, and x402.
- Use PolyPay's canonical merchant REST endpoints with a server-only API Key for ordinary Node.js order operations. Treat `/api/v1/pay/sdk/...` routes as legacy compatibility aliases rather than the default for new integrations. Do not initialize `PolyPayClient` with an API Key; that client is browser-only and accepts a Public Key.
- Use `@polypay/sdk/x402` only for server-side JavaScript/TypeScript x402 resources.
- Reuse the official platform plugin or its conventions for WordPress, WooCommerce, WHMCS, or Shopify.
- Use PolyPay MCP when the user wants an AI client to operate merchant resources rather than embed checkout code.

## Load Only the Required References

- Read [references/checkout.md](references/checkout.md) for hosted checkout, Public Key, API Key, direct-order, locale, and sandbox flows.
- Read [references/webhooks.md](references/webhooks.md) whenever payment state changes local data.
- Read [references/x402.md](references/x402.md) for paid API or Agent resource requests.
- Read [references/notifications.md](references/notifications.md) for channels, templates, delivery, or wallet events.
- Read [references/mcp.md](references/mcp.md) for AI-client access to merchant tools.
- Read [references/platforms.md](references/platforms.md) for framework and commerce-platform constraints.

Read every reference relevant to the request before implementation. Do not load unrelated references.

## Enforce Production Safety

- Keep API Keys in server-only secret storage. Never place them in browser code, mobile code, URLs, logs, examples with real values, or committed files.
- Prefer API-key-independent Ed25519 Webhook v2 verification for new integrations. Keep legacy API Key HMAC verification only when preserving an existing integration, and never downgrade from failed v2 verification to v1.
- Allow Public Keys in browser code only after the merchant configures the exact allowed domains.
- Generate hosted-checkout signatures on a trusted server and pass signed parameters to the browser. Never invent or omit the required signature.
- Treat verified webhooks or an authenticated server-side reconciliation query as the source of truth. Never mark an order paid from a redirect, popup close, client poll result alone, or Agent message.
- Make webhook handling and fulfillment idempotent using stable event, trade, merchant-order, transaction, or nonce identifiers.
- Use Sandbox first for checkout and order flows. Use mocks and protocol test vectors for x402 until the user explicitly authorizes a low-value production settlement. Do not perform a production charge, settlement, webhook resend, or external write without that authorization.
- Preserve the platform trade ID and merchant order ID mapping.
- When the host platform cancels, voids, or replaces an unpaid payment attempt, synchronize that transition through PolyPay's server-side cancellation API. Reconcile ambiguous responses before retrying, and never overwrite a paid state with a local cancellation.
- Validate callback and redirect URLs, use HTTPS in production, and avoid open redirects.
- Keep x402 settlement and merchant API Keys out of public bundles.

## Implement the Complete Slice

1. Confirm the selected integration mode and required identifiers.
2. Add only the required dependency and environment-variable names.
3. Implement order or checkout creation, redirect, and error handling.
4. Implement server-side cancellation synchronization for unpaid attempts when the host can cancel, void, or replace an order.
5. Implement verified, replay-resistant, idempotent payment-state handling when local state changes.
6. Reconcile ambiguous or delayed states through an authenticated server-side query.
7. Preserve localized user-facing copy and the host application's existing patterns.
8. Add focused tests for success, cancellation races, invalid signature, duplicate delivery, expired order, and missing configuration where applicable.
9. Run the repository's typecheck, tests, lint, or build commands that cover the changed paths.

## Require Release Evidence

Before reporting completion, verify as many of these as the environment permits:

- A Sandbox order or hosted-checkout URL is created with the expected merchant order ID.
- Cancelling an unpaid attempt converges both systems to cancelled, and creating another same-amount attempt is not changed solely by a stale allocation from the cancelled order.
- The default checkout URL is locale-neutral; an explicit locale produces a localized path.
- No secret appears in generated client assets, source control, logs, or returned URLs.
- A valid webhook changes state exactly once, while invalid, expired, or replayed signatures do not.
- Redirect success without verified payment does not mark the order paid.
- x402 returns HTTP 402 before payment and protected content only after successful verification and settlement.
- Required checks pass and the manual verification path is documented.

If credentials or a running environment are unavailable, complete static validation and report the exact unverified runtime steps. Do not claim end-to-end success without evidence.

## Stop and Escalate

Stop and ask for direction when:

- The requested design exposes an API Key or settlement capability to an untrusted client.
- The host lacks a stable merchant order identifier or a safe idempotency boundary.
- The user wants client redirects to be the paid-state authority.
- The requested currency, network, x402 scheme, or platform capability is not supported by the current reference.
- Completing the task requires a real production payment or a new external credential not provided by the user.

## Report the Result

Report the selected mode, changed files, dependencies, environment-variable names, tests run, manual Sandbox path, and any remaining production verification. Never print credential values.
