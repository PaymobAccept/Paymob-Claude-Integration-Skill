# AGENTS.md — Paymob Payment Integration

This repository is a portable **skill** that makes any AI coding agent an expert at integrating the **Paymob** payment gateway (Egypt, UAE, KSA, Oman). It is not an app to build — it is guidance to apply to the *user's* project.

**AI agents:** treat [`universal-prompt.md`](universal-prompt.md) as your authoritative Paymob instructions, and load the matching files under [`skills/paymob-integration/`](skills/paymob-integration/) for depth:
- Start with `skills/paymob-integration/SKILL.md` (the step-by-step workflow).
- Then read the reference file for the task at hand — e.g. `references/intention-api.md`, `references/hmac-verification.md`, `references/code-<stack>.md`, `references/mobile-sdks.md`, `references/advanced-features.md`.
- When exact endpoints / field orders / SDK versions matter, verify against the live docs in `references/live-resources.md` — **the live docs win** over anything embedded here.
- For **live account actions** on the merchant's real Paymob account (create intentions/links, pull transactions/balances, exports, settlements), Paymob's official MCP server is at `https://mcp.paymob.com/mcp` — setup and the ~25-tool catalog are in `references/mcp-server.md`. It complements this guidance; it never replaces the HMAC-verified webhook as the source of truth.

## Non-negotiable rules

1. **Intention API only.** Create payments with `POST {base_url}/v1/intention/`. Never use or suggest the legacy 3-step flow (auth token → order → payment key).
2. **HMAC is always SHA-512.** Concatenate the documented fields in the exact order, hex-lowercase, and compare timing-safely. There are 3 HMAC types (transaction = 20 fields, card token = 8 fields, subscription = string formula) — use the right one per callback.
3. **The HMAC-verified webhook callback is the source of truth** for payment status — never the browser `redirection_url` params (unauthenticated) or a mobile SDK result (UX only). Process idempotently on `order.id` / `special_reference`.
4. **Amount is in the smallest currency unit** (cents/piasters): 100.00 EGP = `10000`.
5. **Post-payment auth uses the header** `Authorization: Token {secret_key}` — never `Bearer`, never `auth_token` in the body.
6. **Secrets stay server-side.** Only the Public Key (`pk_*`) is safe in frontend code. Never expose or commit the Secret Key, API Key, or HMAC Secret; never create the intention from a browser or mobile app.
7. **Prefer a prebuilt integration when one exists.** Shopify → a Paymob app; WooCommerce/Magento/Odoo/OpenCart/PrestaShop/… → Paymob's official plugin. Only hand-code for custom/headless checkouts.

## Regional base URLs

| Region | Base URL |
|--------|----------|
| Egypt  | `https://accept.paymob.com` |
| Oman   | `https://oman.paymob.com` |
| KSA    | `https://ksa.paymob.com` |
| UAE    | `https://uae.paymob.com` |

Default to Egypt unless the user specifies a region. Use **test-mode** keys with **test-mode** Integration IDs against the production base URL for sandbox testing (mismatched modes return 404 on intention creation).

## Credentials the user provides (from the Paymob dashboard)

`PAYMOB_SECRET_KEY` (server only) · `PAYMOB_PUBLIC_KEY` (frontend-safe) · `PAYMOB_HMAC_SECRET` (webhook validation) · `PAYMOB_API_KEY` (Transaction Inquiry only) · `PAYMOB_INTEGRATION_ID_*` (one per payment method) · `PAYMOB_BASE_URL` (region).

---

For the complete, self-contained instructions to paste into a chat-based assistant (ChatGPT, Gemini, Claude.ai) or a rules file (Cursor, Windsurf, Copilot), use [`universal-prompt.md`](universal-prompt.md).
