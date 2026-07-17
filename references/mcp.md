# PolyPay MCP

Check the current contract at `https://polypay.ai/en/docs/mcp` before configuring scopes or tool allowlists.

Use MCP when an AI client must query or operate merchant resources. Do not embed MCP credentials in a browser application.

## Connect

- Use Streamable HTTP at `https://api.polypay.ai/mcp`.
- Authenticate with `Authorization: Bearer <api_key>` or `X-API-Key`.
- Store the API Key in the AI client's secure configuration.

Relevant scopes include:

- `mcp:read` for merchant profile, orders, Webhooks, x402, and docs.
- `mcp:orders:write` for order creation or cancellation.
- `mcp:webhooks:write` for Webhook configuration or redelivery.
- `mcp:x402:write` for x402 Resource changes.
- `mcp:admin` for administrator access; avoid it unless required.

Available tools include merchant profile, order list/create, Webhook configuration and redelivery, x402 Resource operations, and documentation search.

## Enforce Least Privilege

Restrict scopes and allowed tools to the task. Configure daily call limits, write limits, and IP allowlists where appropriate. Every write tool requires an `idempotency_key`; reuse the same key only for an identical retry.

Review the merchant MCP audit log for tool name, result, summary, IP, and User-Agent after testing. Do not claim a write succeeded without reading the tool result.
