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

## Framework Boundaries

- Exempt only the webhook route from CSRF where the framework requires it; do not disable CSRF globally.
- Keep the route publicly reachable over HTTPS but protect it with signature verification and rate limits.
- Avoid logging raw authorization headers, API Keys, signatures, full request bodies containing sensitive data, or unmasked customer data.
- Acknowledge verified callbacks promptly; defer slow downstream work to a queue after durable acceptance.

## Required Tests

Cover a valid callback, wrong signature, expired timestamp, malformed nonce, nonce replay, unknown order, duplicate delivery, and an invalid state transition. Assert that only one fulfillment or paid transition occurs.
