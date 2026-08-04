# Notification Center

Query the current supported-channel, default-template, and template-variable APIs before writing mutable notification metadata.

Do not confuse Notification Webhook channels with payment Webhook callbacks. Payment callbacks update order state; Notification Webhook channels deliver merchant-facing messages.

## Current Model

Templates are scoped by merchant, environment, channel type, notification type, and locale. Template locales are `en` and `zh`. Template status `1` means enabled and `2` means disabled; status is event-scoped across locales. When toggling status, preserve the current title and content.

Current customizable event types include:

- `order_paid`, `order_expired`, `order_cancelled`
- `webhook_failed`, `x402_payment`
- `wallet_income`, `wallet_expense`

`wallet_transaction` is accepted only as a compatibility input and is remapped by `cash_flow`. Use `wallet_income` and `wallet_expense` for new template management.

`subscription_expiring` and `subscription_expired` are system-managed. Do not create, edit, delete, or override merchant templates for them.

## Supported Channels

Use the supported-channel API instead of hardcoding availability. Current channel types include in-app, browser push, Telegram, WhatsApp, generic Webhook, WeCom, Discord, and Feishu.

Treat bot tokens, access tokens, and webhook URLs as secrets. Preserve stored masked values when an edit form leaves a sensitive field unchanged.

## Integrate Templates

1. Query default templates and supported variables rather than copying static defaults.
2. Keep `en` and `zh` content separate.
3. Validate every `{{variable}}` against the supported-variable response.
4. Preview with the selected locale and channel.
5. Keep create, edit, delete, and status-toggle actions distinct.
6. Test channel configuration before enabling production delivery.

Wallet templates support linked and shortened address and transaction fields, address names, token and network icons, block time, note, and parameterized short or timezone forms. Prefer values returned by the variables endpoint because this set evolves.

## Delivery Reliability

- Keep event emission idempotent where repeated upstream events are possible.
- Inspect delivery logs before retrying.
- Retry Telegram automatically only when the request is known to be replay-safe: DNS or connection failure before the request is written, HTTP 408, HTTP 429 (honoring `retry_after`), or HTTP 5xx.
- Do not automatically retry Telegram after the request was written but no response was confirmed, or after a successful response could not be read or parsed. Record these outcomes as `uncertain` for manual review because `sendMessage` has no client idempotency key.
- Treat permanent Telegram HTTP 4xx responses as failed without automatic retry.
- Avoid automatic retry loops that duplicate downstream messages.
- Respect environment separation, quotas, channel status, template status, and system-managed events.
