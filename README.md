# Paymob Integration Plugin for Claude

A Claude Code / Cowork plugin that gives Claude expert-level knowledge of Paymob payment gateway integration. It guides developers step-by-step through accepting payments via cards, mobile wallets, kiosk, and cash collection across any tech stack.

## What It Does

When you ask Claude for help integrating Paymob, this plugin provides:

- **Complete API knowledge** — both the modern Intention API and the legacy 3-step Accept API
- **Working code in your stack** — Node.js/TypeScript, Python/Django/Flask/FastAPI, PHP/Laravel, .NET/C#, Ruby/Rails, React/Next.js/Vue, Flutter/React Native
- **HMAC webhook validation** — the exact field order, common pitfalls (the `error_occured` typo, SHA-512 not SHA-256), and ready-to-use implementations
- **Troubleshooting** — instant diagnosis of common errors like duplicate merchant IDs, auth failures, and amount mismatches
- **Security best practices** — credential handling, test vs live environments, and production readiness checks

## Installation

### Claude Code (CLI)

```bash
claude plugin install --git https://github.com/ajeeb24/paymob-integration-plugin
```

### Cowork (Desktop)

Install from the plugin marketplace, or point to this repo as a custom marketplace source.

### Local Development

```bash
git clone https://github.com/ajeeb24/paymob-integration-plugin.git
claude --plugin-dir ./paymob-integration-plugin
```

## Usage

Once installed, Claude automatically activates the skill when you ask about Paymob. Try prompts like:

- "Help me integrate Paymob card payments in my Next.js app"
- "Add Vodafone Cash wallet payments to my Laravel backend"
- "My Paymob HMAC validation keeps failing — here's my code..."
- "Set up Paymob kiosk payments with Python/FastAPI"
- "What credentials do I need from the Paymob dashboard?"

## Plugin Structure

```
paymob-integration-plugin/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── skills/
│   └── paymob-integration/
│       ├── SKILL.md             # Main skill (API flows, checkout, webhooks)
│       └── references/
│           ├── nodejs.md        # Node.js / TypeScript / NestJS
│           ├── python.md        # Python / Django / Flask / FastAPI
│           ├── php.md           # PHP / Laravel
│           ├── dotnet.md        # .NET / C#
│           ├── ruby.md          # Ruby / Rails
│           ├── frontend.md      # React / Next.js / Vue
│           ├── mobile.md        # Flutter / React Native
│           └── hmac-validation.md  # HMAC implementations (all languages)
├── LICENSE
└── README.md
```

## Payment Methods Covered

| Method | Flow |
|--------|------|
| **Card payments** | Iframe / Unified Checkout |
| **Mobile wallets** | Vodafone Cash, Orange Money, Etisalat Cash |
| **Kiosk** | Aman, Masary (bill reference) |
| **Cash collection** | Agent-based collection |

## API Versions

| API | When to Use |
|-----|-------------|
| **Intention API** | New projects — single API call, simpler flow |
| **Legacy Accept API** | Existing codebases — 3-step auth → order → payment key |

## Supported Regions

Egypt (EGP), Pakistan (PKR), UAE, Saudi Arabia, Oman, Jordan, and other Paymob-supported markets.

## Contributing

Contributions are welcome! If Paymob releases new APIs or you have improvements for a specific tech stack:

1. Fork this repo
2. Create a branch (`git checkout -b feature/add-go-support`)
3. Edit the relevant reference file or add a new one
4. Submit a pull request

## Cross-Platform Version

This repo also includes a universal system prompt (`universal-prompt.md`) that works with ChatGPT, Cursor, Windsurf, Gemini, or any other LLM. See the file for setup instructions per platform.

## License

MIT
