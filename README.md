# Paymob Integration Skill for AI Agents

![version](https://img.shields.io/badge/version-3.1.0-blue)
![license](https://img.shields.io/badge/license-MIT-green)
![works with](https://img.shields.io/badge/works%20with-Claude%20%C2%B7%20Cursor%20%C2%B7%20Windsurf%20%C2%B7%20Copilot%20%C2%B7%20Codex-8A2BE2)
![regions](https://img.shields.io/badge/regions-EGY%20%C2%B7%20UAE%20%C2%B7%20KSA%20%C2%B7%20OMN-orange)

Give any AI coding agent expert, **workflow-driven** knowledge of the [Paymob](https://paymob.com) payment gateway across **Egypt, UAE, KSA, and Oman**. The agent routes the developer by platform, walks through onboarding, and produces correct, copy-ready code for accepting cards, mobile wallets, BNPLs, Apple Pay, Google Pay, kiosk, and bank installments — on any tech stack.

Ships as a Claude Code / Cowork plugin **and** as a portable prompt (`universal-prompt.md`) + `AGENTS.md` so it works in Cursor, Windsurf, GitHub Copilot, OpenAI Codex, ChatGPT, Gemini, and more.

---

## Works with any AI agent

The guidance is plain Markdown + code, so it's agent-agnostic. Pick the entrypoint your tool understands:

| Agent / tool | How to use |
|---|---|
| **Claude Code** / **Cowork** | Install as a plugin (see below) — auto-activates as an Agent Skill |
| **OpenAI Codex**, **Aider**, **Zed**, **Gemini CLI**, **Jules** | Copy [`AGENTS.md`](AGENTS.md) into your project root — these tools read it automatically |
| **Cursor** | Add [`universal-prompt.md`](universal-prompt.md) as a rule (`.cursor/rules/paymob.mdc`) or `@`-reference it in chat |
| **Windsurf** | Drop the prompt into `.windsurf/rules/paymob.md` (or `.windsurfrules`) |
| **GitHub Copilot** | Paste the prompt into `.github/copilot-instructions.md` |
| **Cline**, **Continue**, **Roo** | Add `universal-prompt.md` as a rules / context file |
| **ChatGPT**, **Gemini**, **Claude.ai**, any chat UI | Paste [`universal-prompt.md`](universal-prompt.md) into the system prompt / custom instructions |

> Single source of truth: the full skill lives in `skills/paymob-integration/`. `universal-prompt.md` is the portable distillation, and `AGENTS.md` is a concise cross-agent router to it — so there's no drift between versions.

---

## What it does

When you ask the agent for help integrating Paymob, it provides:

- **Platform routing first** — detects Shopify, an official e-commerce plugin platform (WooCommerce, Magento, Odoo, OpenCart, PrestaShop, …), or a custom build, and steers you to the fastest correct path instead of hand-coding when a prebuilt integration already exists.
- **Guided onboarding** — merchant-status check, dashboard credential collection, and sandbox-first testing before go-live.
- **Complete Intention API knowledge** — the only official payment-creation flow, with Unified Checkout (redirect) and the Pixel SDK (embedded).
- **Native Mobile SDK flow** — iOS / Android / Flutter / React Native, keeping the Secret Key off the device.
- **Corrected, copy-ready code in your stack** — Node.js/TypeScript/NestJS, Python/Django/Flask/FastAPI, PHP/Laravel, .NET/C#, Ruby/Rails, and React/Next.js/Vue.
- **All 3 HMAC types** — transaction, card token, and subscription — with exact field orders, SHA-512, and timing-safe comparison.
- **Reconciliation** — a Transaction Inquiry fallback for callbacks that never arrive, stuck "pending" orders, and admin lookups.
- **Core & advanced features** — subscriptions, saved cards (CIT/MIT), Auth/Capture, refund/void, split features, convenience fees.
- **Live-doc discipline** — points at Paymob's `llms.txt` index, developer docs, Integration Wizard, and community forum so the agent can confirm anything that may have changed.

---

## Live access — Paymob MCP server

Beyond *generating* code, agents can act on a merchant's **real Paymob account** through Paymob's official [MCP server](https://mcp.paymob.com/mcp) — creating payment intentions and links, pulling transactions/balances, exporting reports, requesting settlements, and opening support tickets (~25 tools). Great for interactive testing and reconciliation.

This plugin **bundles** it: the repo ships a root [`.mcp.json`](.mcp.json), so enabling the plugin registers the `paymob` server automatically. To add it to any other MCP client:

```json
{
  "mcpServers": {
    "paymob": { "type": "http", "url": "https://mcp.paymob.com/mcp" }
  }
}
```

Or in Claude Code standalone:

```bash
claude mcp add --transport http paymob https://mcp.paymob.com/mcp
```

You authenticate **in-session** with your own Paymob API credentials (test mode first — it includes money-movement tools). It complements, but does **not** replace, the HMAC-verified webhook as the source of truth. Full setup, the tool catalog, and security notes: [`references/mcp-server.md`](skills/paymob-integration/references/mcp-server.md).

---

## Installation

### Claude Code (CLI)

```bash
claude plugin install --git https://github.com/PaymobAccept/Paymob-AI-Integration-Skill
```

### Cowork (Desktop)

Install from the plugin marketplace, or point to this repo as a custom marketplace source.

### Other agents (Cursor, Windsurf, Copilot, Codex, …)

```bash
git clone https://github.com/PaymobAccept/Paymob-AI-Integration-Skill.git
```

Then follow the [Works with any AI agent](#works-with-any-ai-agent) table — copy `AGENTS.md` or `universal-prompt.md` into your own project at the location your tool expects.

### Local development (Claude Code)

```bash
claude --plugin-dir ./Paymob-AI-Integration-Skill
```

---

## Usage

Once installed, the agent activates on any Paymob request — or even a generic regional payment request ("add a payment gateway to my UAE store"). Try prompts like:

- "Help me integrate Paymob card payments in my Next.js app"
- "Add Vodafone Cash wallet payments to my Laravel backend"
- "My Paymob HMAC validation keeps failing — here's my code..."
- "Set up Paymob subscriptions with Python/FastAPI"
- "Integrate Apple Pay with Paymob in my React Native app"
- "Reconcile a Paymob order that's stuck pending"
- "Add Paymob to my Shopify / WooCommerce store"

---

## Repository structure

```
Paymob-AI-Integration-Skill/
├── AGENTS.md                          # Cross-agent entrypoint (Codex, Aider, Zed, Gemini CLI, …)
├── universal-prompt.md                # Portable prompt (Cursor, Windsurf, Copilot, ChatGPT, Gemini, …)
├── .mcp.json                          # Bundled Paymob MCP server (auto-registers when the plugin is enabled)
├── .claude-plugin/
│   └── plugin.json                    # Claude Code plugin manifest (v3.2.0)
├── skills/
│   └── paymob-integration/
│       ├── SKILL.md                   # Workflow backbone: platform routing → onboarding → web/mobile → testing
│       └── references/
│           ├── shopify-apps.md        # Paymob Shopify apps (on-site / off-site / BNPL) + install path
│           ├── intention-api.md       # Create Intention spec, Unified Checkout, common errors
│           ├── mobile-sdks.md         # Native SDK flow (iOS / Android / Flutter / React Native)
│           ├── hmac-verification.md   # Transaction HMAC: field order, SHA-512, worked example
│           ├── transaction-inquiry.md # Pull-based status checks / reconciliation
│           ├── test-credentials.md    # Sandbox cards, wallets, OTPs
│           ├── advanced-features.md   # Subscriptions, saved cards (CIT/MIT), Auth/Cap, refund/void, split, fees
│           ├── live-resources.md      # llms.txt, dev docs, Integration Wizard, community — when/how to use
│           ├── mcp-server.md          # Official Paymob MCP server: connect, authenticate, tool catalog, security
│           ├── code-nodejs.md         # Node.js / TypeScript / Express / NestJS
│           ├── code-python.md         # Python / Django / Flask / FastAPI
│           ├── code-php.md            # PHP / Laravel
│           ├── code-dotnet.md         # .NET / C# / ASP.NET
│           ├── code-ruby.md           # Ruby / Rails
│           └── code-frontend.md       # React / Next.js / Vue + Unified Checkout / Pixel SDK
├── LICENSE
└── README.md
```

---

## Payment methods covered

| Method | Regions | Notes |
|--------|---------|-------|
| **Cards** (Visa, MC, Amex, MADA, OmanNet) | EGY, KSA, UAE, OMN | 3DS, MOTO, Card-on-File, Auth/Cap |
| **Mobile Wallets** (Vodafone Cash, Orange Cash, e& money, WePay, StcPay) | EGY, KSA | — |
| **BNPLs** (Valu, Tabby, Tamara, Souhoola, Sympl, and more) | EGY, KSA, UAE | 15+ providers |
| **Apple Pay** | EGY, KSA, UAE, OMN | Requires certificates |
| **Google Pay** | KSA, UAE, OMN | Not yet in Egypt |
| **Bank Installments** | EGY | Live IDs only |
| **Kiosk** (Aman, Masary) | EGY | No refund support |

## Supported regions

| Region | Base URL |
|--------|----------|
| Egypt (EGY) | `https://accept.paymob.com` |
| Oman (OMN) | `https://oman.paymob.com` |
| Saudi Arabia (KSA) | `https://ksa.paymob.com` |
| UAE | `https://uae.paymob.com` |

---

## Security

- Only the **Public Key** (`pk_*`) is safe in frontend code. The **Secret Key**, **API Key**, and **HMAC Secret** are server-side only — never commit them or ship them in a mobile binary.
- **The HMAC-verified webhook callback is the source of truth** for payment status — never the browser redirect params or a mobile SDK result.
- Always verify HMAC with **SHA-512** and a timing-safe comparison, and process callbacks **idempotently** (key on `order.id` / `special_reference`).

## Staying current

Specs embedded here are known-good as of **June 2026**. Paymob changes endpoints, field orders, and SDK versions on its own schedule — the skill instructs the agent to cross-check the live docs ([`references/live-resources.md`](skills/paymob-integration/references/live-resources.md), especially the machine-readable `llms.txt` index) and lets the live docs win on any disagreement.

## Support & resources

- 📚 Developer docs — https://developers.paymob.com/
- 🧭 Integration Wizard (roadmap, runnable samples, HMAC/webhook tester) — https://wizard.paymob.com/
- 💬 Community forum — https://community.paymob.com/
- ✉️ Support — support@paymob.com

## Contributing

Contributions are welcome! If Paymob releases new APIs or you have improvements for a specific tech stack:

1. Fork this repo
2. Create a branch (`git checkout -b feature/add-go-support`)
3. Edit the relevant reference file (or add a new one) — keep `SKILL.md`, `universal-prompt.md`, and `AGENTS.md` in sync
4. Submit a pull request

## License

MIT — see [LICENSE](LICENSE) for details.
