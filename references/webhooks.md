# Payment Webhooks

Check the current contract at `https://polypay.ai/en/docs/webhook-security` before implementing a new verifier.

## Prefer Official Verification

For new PHP integrations, use `polypay/php-sdk` `webhookV2(expectedMerchantId, environment)->handle()`. It fetches PolyPay public keys from `GET /api/v1/pay/public/webhook-jwks`; merchants do not configure or choose a signing key. Do not hand-roll verification when the official handler is available.

Webhook v2 includes:

- `X-Webhook-Signature-Version: v2`
- `X-Webhook-Key-Id`
- `X-Webhook-Merchant-Id`
- `X-Webhook-Environment`
- `X-Webhook-Signature-V2` as unpadded Base64url Ed25519 bytes
- the shared `X-Timestamp` and `X-Nonce` headers

Use the headers as follows:

| Header | Meaning |
|---|---|
| `Content-Type` | Must describe the delivered body as `application/json`; it is not a signing-key selector |
| `User-Agent` | Identifies the sender as PolyPay for observability only; never treat it as authentication |
| `X-Timestamp` | Unix seconds used for the 300-second freshness check and signed by both protocols |
| `X-Nonce` | Per-delivery random value consumed once to prevent replay; signed by both protocols |
| `X-Webhook-Signature-Version` | Selects the verification protocol; require exactly `v2` when using the v2 handler |
| `X-Webhook-Key-Id` | `kid` used to select the Ed25519 public JWK; it is an identifier, not a secret |
| `X-Webhook-Merchant-Id` | Signed merchant audience; compare with the receiver's server-side expected merchant ID |
| `X-Webhook-Environment` | Signed `production` or `sandbox` audience; compare with the receiver's expected environment |
| `X-Webhook-Signature-V2` | Unpadded Base64url Ed25519 signature over the exact v2 payload below |
| `X-Key-Prefix` | Legacy v1 API Key prefix used only to select a server-controlled candidate key; it is sensitive metadata even though it is not the full key |
| `X-Signature` | Legacy lowercase hexadecimal HMAC-SHA256 signature |

Never authenticate from `User-Agent`, `X-Key-Prefix`, merchant ID, or environment alone. Authentication requires a valid signature over the unchanged raw body plus all protocol-bound audience fields.

Build the exact UTF-8 bytes without JSON reserialization:

```text
payload = "polypay-webhook-v2" + "\n"
        + kid + "\n"
        + timestamp + "\n"
        + nonce + "\n"
        + merchant_id + "\n"
        + environment + "\n"
        + raw_body
```

Select the Ed25519 JWK by `kid` and require `kty=OKP`, `crv=Ed25519`, `alg=EdDSA`, and `use=sig`. Validate a 300-second timestamp window, atomically consume the nonce in shared storage, and compare the signed merchant ID and environment with server-side expected values. Refresh JWKS once for an unknown `kid`, then reject it. Never fall back to v1 after any v2 error.

## Preserve Legacy HMAC v1

PolyPay dual-signs deliveries during migration. Existing PHP integrations may keep using `webhook()->handle()` unchanged; it verifies only the legacy API Key signature and ignores v2 headers. This compatibility path is useful for already deployed merchants but is not the default for new integrations.

The signed callback includes `X-Key-Prefix`, `X-Timestamp`, `X-Nonce`, and `X-Signature`. Verify the current contract as follows:

```text
key_hash = lowercase_hex(SHA256(api_key))
payload = timestamp + "\n" + nonce + "\n" + raw_body
expected = lowercase_hex(HMAC_SHA256(key=key_hash, data=payload))
```

- Require `X-Key-Prefix` to equal the first 12 API Key characters.
- Require a Unix-seconds timestamp within 300 seconds of server time.
- Require a 16–128 character alphanumeric nonce.
- Consume `SHA256(timestamp + "|" + nonce)` once with at least a 600-second shared TTL.
- Compare the 64-character hexadecimal signature in constant time.

During API Key rotation, select only a server-controlled candidate key by prefix. Never accept a verification key supplied by the callback request. A merchant with multiple API Keys should migrate to v2 instead of guessing which API Key signed a callback.

For high-traffic or multi-instance PHP deployments, replace file-based nonce storage with a shared `NonceStorageInterface` implementation such as Redis.

## Handle State Safely

1. Read the raw body before JSON reserialization changes it.
2. Verify signature freshness and consume the nonce.
3. Parse the verified event.
4. Deduplicate by `(environment, event_id)`, then find the local order using the persisted PolyPay trade ID or merchant order ID.
5. Apply only valid state transitions inside a transaction.
6. Record a stable deduplication identifier.
7. Return 2xx only after the event is durably accepted.

Do not trust a transaction hash, browser redirect, callback URL query, or status string until the callback is verified. Reconcile suspicious or missing transitions with an authenticated server-side order query.

## Understand the Payment Payload

Treat fiat accounting and crypto settlement as two separate dimensions. Do not compare a local fiat currency such as USD with the callback's `currency`, because `currency` identifies the crypto asset.

| Field | Meaning | Integration use |
|---|---|---|
| `order_no` | Merchant order ID originally supplied when creating the PolyPay order | Find the merchant's local order; do not confuse it with PolyPay's `trade_id` |
| `status` | Numeric PolyPay order state | `0` checkout pending, `1` waiting, `2` paid, `3` expired, `4` cancelled, `5` manual recharge, `6` confirming, `7` admin marked paid |
| `amount` | Crypto asset quantity due or paid | Compare with the expected token quantity using decimal-safe arithmetic |
| `fiat_amount` | Original fiat order total denominated in USD | Compare with the local USD order total; do not compare it with a crypto quantity |
| `exchange_rate` | Locked value of one crypto unit in USD | Reconcile `amount`, `fiat_amount`, and the order's pricing snapshot |
| `currency` | Uppercase crypto asset code such as `USDT` | Compare with the expected payment asset, not the invoice fiat currency |
| `currency_name` | Lowercase crypto asset name such as `usdt` | Display or compatibility alias; use `currency` for canonical comparisons |
| `network` | Blockchain network such as `Tron`, `Ethereum`, `Polygon`, or `Solana` | Compare with the payment attempt's expected chain |
| `contract_addr` | Token contract or mint address; empty for a native or unknown asset | Bind token transfers to the expected asset on the selected network |
| `hash` | On-chain transaction hash | Audit and deduplicate a chain payment; it can be empty for status `7` because an admin-marked payment may have no chain transaction |
| `wallet_address` | PolyPay receiving wallet address | Reconcile the destination after verification; treat it as sensitive in logs and screenshots |
| `environment` | `production` or `sandbox` | Compare with the signed header and the receiver's server-side expected environment |
| `event_id` | Stable identifier for one business event | Deduplicate by `(environment, event_id)` before applying state changes |

The callback does not currently include a separate `fiat_currency` field: `fiat_amount` is defined as USD. If a host platform stores another invoice currency, persist the pricing or conversion snapshot used when the PolyPay order was created rather than treating `currency` as that fiat currency.

## Return Actionable Failures

Return `2xx` only after durable acceptance. For a permanent business rejection, use a `4xx` status and identify the exact differing field whenever it is safe to disclose it to the authenticated merchant's delivery log. For example:

```json
{
  "code": "payment_currency_mismatch",
  "message": "payment currency mismatch",
  "field": "currency",
  "expected": "USDT",
  "actual": "USDC",
  "event_id": "evt_example"
}
```

Do not collapse amount and currency failures into one opaque message when the receiver knows which comparison failed. Keep expected and actual values free of secrets or customer data. A `409` business conflict should be corrected or reconciled before resend; repeating the identical callback cannot repair a deterministic mapping mismatch. Reserve `5xx` for transient receiver failures that may succeed on retry.

## Framework Boundaries

- Exempt only the webhook route from CSRF where the framework requires it; do not disable CSRF globally.
- Keep the route publicly reachable over HTTPS but protect it with signature verification and rate limits.
- Avoid logging raw authorization headers, API Keys, signatures, full request bodies containing sensitive data, or unmasked customer data.
- Acknowledge verified callbacks promptly; defer slow downstream work to a queue after durable acceptance.

## Required Tests

Cover a valid callback, wrong signature, expired timestamp, malformed nonce, nonce replay, unknown order, duplicate delivery, and an invalid state transition. Assert that only one fulfillment or paid transition occurs.
