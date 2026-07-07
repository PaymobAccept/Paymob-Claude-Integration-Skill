# Paymob MCP Server (live merchant access)

Paymob runs an **official, first-party MCP server** that lets an AI agent act on a merchant's real Paymob account — create payment intentions and payment links, pull transactions/balances, export reports, and more — instead of only generating code. It complements this skill: use the **skill's code references** to build the merchant's app, and use the **MCP server** for live account actions, testing, and reconciliation from inside the agent.

- **Endpoint:** `https://mcp.paymob.com/mcp`
- **Transport:** Streamable HTTP (remote MCP; no local install)
- **Server:** "Paymob Payment System" (verified live at time of writing)
- **Auth:** bring-your-own Paymob API credentials, configured **in-session** (see below). Connecting requires no credentials — the server does nothing until you provide yours.

> This is a hosted service run by Paymob. You send it your own Paymob API key + secret key, and it acts on your account. Treat it with the same care as your dashboard login: start in **test mode** (`is_live: false`), and understand that it includes money-movement tools (see the security note).

---

## Connecting

### Claude Code — via this plugin (automatic)

This repo ships a `.mcp.json` at its root, so when the `paymob-integration` plugin is installed/enabled, Claude Code registers the `paymob` MCP server automatically. Approve it when prompted.

### Claude Code — standalone (without the plugin)

```bash
claude mcp add --transport http paymob https://mcp.paymob.com/mcp
```

### Any client via `.mcp.json` (Cursor, VS Code, Claude Code project scope, …)

Add to the project's `.mcp.json` (or the client's MCP config):

```json
{
  "mcpServers": {
    "paymob": {
      "type": "http",
      "url": "https://mcp.paymob.com/mcp"
    }
  }
}
```

`type` may also be written as `streamable-http` (an alias for `http`) if you're copying from other tooling. An entry with a `url` but **no `type`** is a configuration error.

### Cursor

Add the same object to `.cursor/mcp.json` (project) or `~/.cursor/mcp.json` (global).

---

## Authenticating (per session)

The server follows a "configure, test, use" flow — its own `get_help` and `get_usage_examples` tools document it live:

1. **Configure credentials** — provide your Paymob **API key** and **secret key** (from Dashboard → Developers → API Keys) to the server's credential-setup tool (`set_api_credentials`). Use **test** credentials first.
2. **Test connectivity** — `test_api_connectivity` verifies the credentials work.
3. **Discover methods** — `get_available_payment_methods` (pass `currency`, `is_live: false`) lists the Integration IDs available to you.

Then call any action tool. Prefer the `elicit_*` tools for guided, step-by-step setup that fills required fields correctly.

---

## Tool catalog (~25 tools)

**Guided setup (start here):** `elicit_payment_intention`, `elicit_payment_link`, `elicit_scaling_payment_link`, `elicit_transaction_filters`, `elicit_transfer_filters` — walk you through required fields before you call the matching create/query tool.

**Create & manage payments (write):**
- `create_payment_intention` / `update_payment_intention` — Intention (Flash) API: `amount` (cents), `currency`, `payment_methods` (Integration IDs), `billing_data`, optional `items`, `special_reference`, `redirection_url`, `notification_url`.
- `create_payment_link` — single-customer shareable link.
- `create_scaling_payment_link` — multi-customer link (crowdfunding/donations), with optional usage limits.
- `create_invoice` — invoice for payment collection.

**Read / reporting (read-only):**
- `get_merchant_transactions`, `get_filtered_transactions` (date/amount/currency/status/…), `export_transactions` (Excel).
- `get_merchant_transfers`, `get_filtered_transfers`.
- `get_merchant_balances` — available balances per account/currency.
- `get_merchant_payment_links`, `get_payment_link_by_token`.
- `get_invoices`.
- `get_available_payment_methods`, `test_api_connectivity`.

**Money movement (write — high impact):**
- `request_instant_settlement` — moves funds from your Paymob balance to your bank account.

**Support & help:**
- `create_support_ticket` — open a support ticket (terminal/payment issues).
- `get_help`, `get_usage_examples` — the server's own live docs; call these first if a tool's fields are unclear.

> The exact tool list is versioned by Paymob and may change. Call `tools/list` (or `get_help`) against the live server for the current set rather than treating this catalog as fixed.

---

## Security

- **Credentials are yours.** The server acts with your Paymob API key + secret key. Never paste **live** keys until you've validated the flow in **test mode** (`is_live: false`).
- **It can move money.** `request_instant_settlement` transfers funds to your bank; `create_payment_intention`, `create_payment_link`, and `create_invoice` create real payment obligations when `is_live: true`. Confirm intent before invoking these in live mode, and don't let an unattended agent call them without review.
- **Don't commit secrets.** The `.mcp.json` in this repo contains only the public endpoint URL — no keys. Credentials are supplied at runtime via the credential-setup tool, not stored in the repo.
- **Prefer read-only for automation.** For dashboards/reconciliation jobs, lean on the `get_*` / `export_*` tools, which are read-only.

## MCP server vs. code references — when to use which

| Use the **MCP server** when… | Use the **skill's code references** when… |
|---|---|
| Testing your account interactively (create a test intention/link, then pull it back) | Building the merchant's actual app/backend |
| Reconciling: pulling transactions, balances, transfers, exports | Implementing the HMAC-verified webhook (the source of truth) |
| Ad-hoc ops: issue a payment link, open a support ticket, request a settlement | Shipping production checkout, mobile SDK, or subscriptions code |

The MCP server does **not** replace your webhook: payment status in your app must still come from the HMAC-verified callback (`references/hmac-verification.md`), reconciled with Transaction Inquiry (`references/transaction-inquiry.md`) — which the MCP `get_*` tools make easy to do by hand.
