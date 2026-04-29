# Paymob Integration Plugin for Claude

A Claude Code / Cowork plugin that gives Claude expert-level knowledge of Paymob payment gateway integration. It guides developers step-by-step through accepting payments via cards, mobile wallets, BNPLs, Apple Pay, Google Pay, kiosk, and bank installments across any tech stack.

## What It Does

When you ask Claude for help integrating Paymob, this plugin provides:

- **Complete Intention API knowledge** — the official payment creation flow with Unified Checkout and Pixel SDK
- **Working code in your stack** — Node.js/TypeScript, Python/Django/Flask/FastAPI, PHP/Laravel, .NET/C#, Ruby/Rails, React/Next.js/Vue, iOS/Android/Flutter/React Native
- **HMAC webhook validation** — all 3 types (transaction, card token, subscription), exact field orders, SHA-512 implementations
- **Core features** — subscriptions, saved cards (CIT/MIT), Auth/Cap, split features, convenience fees
- **Multi-region support** — Egypt, Saudi Arabia, UAE, Oman with correct base URLs
- **Troubleshooting** — instant diagnosis of common errors like integration ID mismatches, auth failures, HMAC mismatches
- **Security best practices** — credential handling, test vs live environments, and production readiness checks

## Installation

### Claude Code (CLI)

```bash
claude plugin install --git https://github.com/PaymobAccept/Paymob-Claude-Integration-Skill
```

### Cowork (Desktop)

Install from the plugin marketplace, or point to this repo as a custom marketplace source.

### Local Development

```bash
git clone https://github.com/PaymobAccept/Paymob-Claude-Integration-Skill.git
claude --plugin-dir ./Paymob-Claude-Integration-Skill
```

## Usage

Once installed, Claude automatically activates the skill when you ask about Paymob. Try prompts like:

- "Help me integrate Paymob card payments in my Next.js app"
- "Add Vodafone Cash wallet payments to my Laravel backend"
- "My Paymob HMAC validation keeps failing — here's my code..."
- "Set up Paymob subscriptions with Python/FastAPI"
- "Integrate Apple Pay with Paymob in my React Native app"
- "What credentials do I need from the Paymob dashboard?"

## Plugin Structure

```
paymob-integration-plugin/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── skills/
│   └── paymob-integration/
│       ├── SKILL.md             # Main skill (Intention API, checkout, webhooks, features)
│       └── references/
│           ├── nodejs.md        # Node.js / TypeScript / NestJS
│           ├── python.md        # Python / Django / Flask / FastAPI
│           ├── php.md           # PHP / Laravel
│           ├── dotnet.md        # .NET / C#
│           ├── ruby.md          # Ruby / Rails
│           ├── frontend.md      # React / Next.js / Vue / Pixel SDK
│           ├── mobile.md        # iOS / Android / Flutter / React Native native SDKs
│           ├── hmac-validation.md   # HMAC implementations (all 3 types, all languages)
│           ├── subscriptions.md     # Subscription plans, management, webhooks
│           └── saved-cards.md       # Card tokens, CIT, MIT flows
├── universal-prompt.md          # Cross-platform prompt (ChatGPT, Gemini, etc.)
├── LICENSE
└── README.md
```

## Payment Methods Covered

| Method | Regions | Notes |
|--------|---------|-------|
| **Cards** (Visa, MC, Amex, MADA, OmanNet) | EGY, KSA, UAE, OMN | 3DS, Moto, Card-on-File, Auth/Cap |
| **Mobile Wallets** (Vodafone Cash, Orange Cash, StcPay, etc.) | EGY, KSA | — |
| **BNPLs** (Valu, Tabby, Tamara, Souhoola, Sympl, etc.) | EGY, KSA, UAE | 15+ providers |
| **Apple Pay** | EGY, KSA, UAE, OMN | Requires certificates |
| **Google Pay** | KSA, UAE, OMN | Not yet in Egypt |
| **Bank Installments** | EGY | Live IDs only |
| **Kiosk** (Aman, Masary) | EGY | No refund support |

## Supported Regions

| Region | Base URL |
|--------|----------|
| Egypt (EGY) | `https://accept.paymob.com` |
| Oman (OMN) | `https://oman.paymob.com` |
| Saudi Arabia (KSA) | `https://ksa.paymob.com` |
| UAE | `https://uae.paymob.com` |

## Contributing

Contributions are welcome! If Paymob releases new APIs or you have improvements for a specific tech stack:

1. Fork this repo
2. Create a branch (`git checkout -b feature/add-go-support`)
3. Edit the relevant reference file or add a new one
4. Submit a pull request

## Cross-Platform Version

This repo includes a universal system prompt (`universal-prompt.md`) that works with ChatGPT, Gemini, Cursor, Windsurf, and any other AI assistant. Paste it into your AI's system prompt or custom instructions.

## License

MIT — see [LICENSE](LICENSE) for details.
