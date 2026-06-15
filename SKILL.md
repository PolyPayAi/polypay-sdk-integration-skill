---
name: ponponpay-sdk-integration
description: Integrate PolyPay / PonponPay payments, hosted checkout, SDKs, webhooks, x402, and Notification Center features into an existing application. Use when a user asks to add PolyPay payment acceptance, JavaScript SDK, PHP SDK, API Key Mode, Public Key Mode, webhook verification, WHMCS or WordPress payment flows, wallet transaction notifications, or bilingual notification templates.
---

# PolyPay SDK Integration

Use this skill to integrate PolyPay payment and notification features into an existing codebase with minimal, idiomatic changes.

## Product Names

- Public-facing brand: PolyPay.
- Repository and legacy package names may still use PonponPay.
- Do not rename existing project identifiers unless the user explicitly asks.

## Core Rules

- Prefer hosted checkout for merchant checkout flows.
- Do not build a merchant-side payment-method selection page by default.
- Treat webhook verification as the source of truth for paid status.
- Never put an API Key in browser code, mobile client code, logs, or committed files.
- Public Key Mode is allowed in frontend code only when the merchant has configured the allowed domain.
- Add `.env.example` entries for required variables, but never write real secrets.
- Keep edits aligned with the host project structure, router, state management, styling, and validation tools.
- Run the available typecheck, build, lint, or test commands after changes.

## Integration Modes

### Frontend / Public Key Mode

Use this for browser-only checkout initiation.

- Install `@polypay/sdk` if the project uses npm, pnpm, yarn, or bun.
- Read the public key from a public env var such as `NEXT_PUBLIC_POLYPAY_PUBLIC_KEY`.
- Create a hosted checkout session or URL through the JavaScript SDK.
- Redirect users to `https://checkout.polypay.ai/{locale}/checkout` or the hosted URL returned by the SDK/API.
- Do not expose API Key Mode from frontend code.

### Server / API Key Mode

Use this for trusted server-side order creation, webhook verification, and status reconciliation.

- PHP projects should use `polypay/php-sdk` when Composer is available.
- JavaScript/TypeScript server projects should keep the API Key in server-only env vars.
- Create direct orders only when the server already knows the required `currency` and `network`.
- Otherwise request a hosted checkout URL and let PolyPay collect the payment method details.
- Persist the platform trade/order id and merchant order id mapping.

### Webhooks

Add a webhook endpoint when payment state changes matter to the host application.

- Verify the webhook signature before changing local order state.
- Make the handler idempotent by checking event id, trade id, or merchant order id.
- Update paid/cancelled/expired state only after verification succeeds.
- Return a 2xx response only after the event has been accepted.
- Log useful diagnostic context without logging secrets.

## Notification Center

Use Notification Center when the user asks for Telegram, WhatsApp, WeCom, browser push, in-app, wallet transaction, webhook failure, x402, order, or subscription notifications.

### Template Model

Notification templates are scoped by:

- merchant id
- environment
- channel type
- notification type
- locale

Supported template locales:

- `en`
- `zh`

Status values:

- `1` means enabled
- `2` means disabled

When toggling a template status, update only the `status` field and preserve the merchant's current `title_template` and `content_template`.

### Event Types

Use these event type ids:

- `order_paid`
- `order_expired`
- `order_cancelled`
- `webhook_failed`
- `x402_payment`
- `wallet_transaction`
- `subscription_expiring`
- `subscription_expired`

### Wallet Transaction Templates

Use `wallet_transaction` for wallet income and expense notifications.

Common variables:

- `cash_flow`: localized income/expense text
- `amount`
- `token`
- `network`
- `address_from`
- `address_from_short`
- `address_from_link`
- `address_to`
- `address_to_short`
- `address_to_link`
- `transaction_hash`
- `transaction_hash_short`
- `transaction_hash_link`
- `block_time`
- `note`

English default:

```text
Wallet {{cash_flow}}
✅✅✅ #{{cash_flow}}
Amount: {{amount}} {{token}}
From: {{address_from}}
To: {{address_to}}
Tx Hash: {{transaction_hash}}
Time: {{block_time}}
On-chain note: {{note}}
```

Chinese default:

```text
钱包{{cash_flow}}
✅✅✅ #{{cash_flow}}
支付金额：{{amount}} {{token}}
发出地址：{{address_from}}
接收地址：{{address_to}}
交易Hash：{{transaction_hash}}
交易时间：{{block_time}}
链上转账备注：{{note}}
```

### Template UI Expectations

If adding or modifying a template management page:

- Include a language selector for `en` and `zh`.
- List templates for the selected language only.
- Show status as an action button, not static text.
- Clicking the status button toggles enabled/disabled.
- Keep edit and delete actions separate from status toggling.
- Preview should render with the selected locale.
- Use localized labels for event names, channel names, language names, enabled, and disabled.

## Platform-Specific Guidance

### Next.js / React

- Keep SDK calls that need API keys in route handlers or server actions.
- Use `NEXT_PUBLIC_` only for Public Key Mode values.
- Keep checkout redirects in user-triggered flows.
- Use existing UI components and i18n patterns.

### PHP / Laravel

- Use Composer when available.
- Keep API keys in `.env`.
- Put webhook verification in a controller or route that bypasses CSRF only for the webhook path.
- Use existing order models and transaction boundaries.

### WordPress / WHMCS

- Store merchant configuration in the platform's settings system.
- Do not hardcode API endpoints, API keys, or merchant credentials.
- Preserve existing hook and template conventions.
- Clear WHMCS `templates_c/` after template/config changes if rendering does not update.

## Implementation Checklist

1. Identify the host framework and package manager.
2. Locate existing order/payment/webhook abstractions.
3. Choose Public Key Mode, API Key Mode, or hosted checkout based on the security boundary.
4. Add env examples without real secrets.
5. Implement checkout creation and redirect.
6. Implement webhook verification and idempotent order updates.
7. Add Notification Center templates/channels only if requested.
8. Add or update bilingual copy for user-facing UI.
9. Run the repository's validation commands.
10. Report changed files, required env vars, and manual test steps.

## Refuse Or Escalate

Stop and ask for direction when:

- The user wants API keys committed to source control.
- The requested flow requires browser-side API Key usage.
- The host project lacks enough order identifiers to reconcile webhooks safely.
- Payment status would be marked paid from frontend redirects without webhook verification.
