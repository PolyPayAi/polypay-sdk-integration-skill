# PolyPay Production Integration Skill

[English](#english) · [简体中文](#简体中文)

Give coding agents a production-oriented workflow for integrating [PolyPay](https://polypay.ai) payments, Webhooks, x402 resources, merchant notifications, and MCP access.

> `README.md` is for human readers. [`SKILL.md`](SKILL.md) is the authoritative instruction source for agents.

## English

### What this Skill does

The Skill inspects the target application's runtime, payment architecture, order model, and security boundary before selecting the smallest complete PolyPay integration. It helps an agent implement and verify:

- Hosted Checkout created with server-side API Keys
- JavaScript redirect/popup helpers for opaque checkout URLs and server-side REST integrations
- PHP SDK checkout, order queries, Webhook verification, and x402
- Signed, replay-resistant, idempotent Webhook processing
- Sandbox checkout and reconciliation flows
- x402-paid APIs and Agent resources
- Notification Center events, templates, and delivery channels
- Least-privilege PolyPay MCP access
- WordPress, WooCommerce, WHMCS, and Shopify payment flows
- Existing PolyPay integrations that need to migrate or remain compatible

### Install

Ask your coding agent to install the Skill from this repository:

```text
Install and use the PolyPay integration Skill from:
https://github.com/PolyPayAi/polypay-sdk-integration-skill
```

For a manual Codex installation, clone it into the Codex skills directory:

```bash
git clone https://github.com/PolyPayAi/polypay-sdk-integration-skill.git \
  ~/.codex/skills/polypay-sdk-integration
```

Use the equivalent skills directory when your Agent uses another installation location. Start a new Agent session after installation so the Skill can be discovered.

### Example prompts

```text
Use $polypay-sdk-integration to add PolyPay Hosted Checkout to this Next.js app.
Keep the API Key server-only and verify the payment through an idempotent Webhook.
```

```text
Use $polypay-sdk-integration to protect this API with x402.
Validate the request in a test environment and do not perform a production settlement.
```

```text
Use $polypay-sdk-integration to review this existing PolyPay integration for
secret exposure, unsigned callbacks, replay attacks, and duplicate fulfillment.
```

### Production guardrails

The Skill instructs agents to:

- keep API Keys and settlement credentials out of browsers, URLs, logs, and source control;
- create Hosted Checkout on the server and expose only the opaque checkout URL to browsers;
- treat verified Webhooks or authenticated server reconciliation as the payment source of truth;
- reject expired or replayed callbacks and fulfill each order only once;
- preserve PolyPay trade IDs and merchant order IDs;
- test checkout in Sandbox before production;
- stop before real charges, settlements, or other external writes unless explicitly authorized.

### Repository structure

```text
.
├── SKILL.md              # Agent workflow and safety requirements
├── agents/openai.yaml    # Skill UI metadata
└── references/           # Capability-specific integration guidance
    ├── checkout.md
    ├── webhooks.md
    ├── x402.md
    ├── notifications.md
    ├── mcp.md
    └── platforms.md
```

The Agent loads only the references needed for the requested integration. When installed package types or current official documentation conflict with a bundled reference, the newer authoritative contract wins and the Agent should report the drift.

### Documentation

- [PolyPay documentation](https://polypay.ai/en/docs)
- [API Key integration](https://polypay.ai/en/docs/apikey)
- [JavaScript SDK](https://polypay.ai/en/docs/javascript-sdk)
- [PHP SDK](https://polypay.ai/en/docs/php-sdk)
- [x402 integration](https://polypay.ai/en/docs/x402-integration)

## 简体中文

### 这是什么

这是面向 Codex、Cursor 等编程 Agent 的 PolyPay 生产级接入 Skill。它会先检查项目运行时、订单模型、支付抽象和安全边界，再选择 Hosted Checkout、JavaScript、PHP SDK、服务端 REST、x402、MCP 或官方平台插件中的最小正确方案。

它重点指导 Agent 完成：

- 服务端创建 Hosted Checkout，浏览器只接收并打开不透明支付链接
- Hosted Checkout、订单创建、跳转和服务端对账
- Webhook 原始正文验签、时间窗口、防重放和幂等履约
- Sandbox 验证以及成功、失败、过期、重复回调等测试
- x402 付费 API、通知中心、MCP 和电商平台插件接入
- WordPress、WooCommerce、WHMCS、Shopify 及 PolyPay 兼容场景

### 安装与使用

可以直接让编程 Agent 从本仓库安装：

```text
请从下面的 GitHub 仓库安装并使用 PolyPay integration Skill：
https://github.com/PolyPayAi/polypay-sdk-integration-skill
```

Codex 也可以手动安装：

```bash
git clone https://github.com/PolyPayAi/polypay-sdk-integration-skill.git \
  ~/.codex/skills/polypay-sdk-integration
```

安装后新建 Agent 会话，然后使用类似提示词：

```text
使用 $polypay-sdk-integration 为这个 Next.js 项目接入 PolyPay Hosted Checkout。
API Key 必须只保留在服务端，并通过验签且幂等的 Webhook 确认支付结果。
```

### 安全原则

Skill 默认要求使用 Sandbox，不会把前端跳转或轮询结果当作支付成功依据，也不会在未获得明确授权时发起真实扣款、x402 结算、Webhook 重发或其他生产写操作。

Agent 执行时以 [`SKILL.md`](SKILL.md) 为准，并按任务需要读取 `references/` 中的专项资料。详细产品能力和接口参数请以 [PolyPay 官方文档](https://polypay.ai/zh/docs) 为准。
