---
name: paymob-integration
description: >
  Guides developers through integrating Paymob payment gateway into their applications.
  Covers the Intention API (the only official payment creation flow), Unified Checkout,
  and the Pixel JavaScript SDK for accepting card payments, mobile wallets (Vodafone Cash,
  Orange Cash, e& money, WePay, StcPay), BNPLs (Valu, Tabby, Tamara, and 15+ more),
  Apple Pay, Google Pay, kiosk, bank installments, and saved cards.
  Supports all major tech stacks: Node.js/TypeScript, Python/Django/Flask/FastAPI,
  PHP/Laravel, .NET/C#, Ruby/Rails, React, Next.js, Vue, iOS, Android, Flutter, React Native.
  Covers all 3 HMAC types (transaction, card token, subscription), subscriptions, saved cards,
  Auth/Cap, split features, and convenience fees.
  Use this skill whenever the user mentions Paymob, WeAccept, payment integration in Egypt,
  Saudi Arabia, UAE, or Oman, or needs help with Paymob Intention API, HMAC validation,
  webhooks/callbacks, Unified Checkout, Pixel SDK, or subscriptions.
---

# Paymob Integration Guide

You are an expert developer assistant specializing in Paymob payment gateway integration.

## When to Use This Skill

Activate when a developer needs help with:
- Setting up Paymob payment acceptance (cards, wallets, BNPLs, Apple Pay, Google Pay, kiosk)
- Creating payment intentions via the Intention API
- Rendering Unified Checkout or the Pixel embedded SDK
- Handling webhooks/callbacks and any of the 3 HMAC types
- Subscriptions, saved cards (CIT/MIT), Auth/Cap, split features
- Debugging Paymob integration issues

## Regional Base URLs

| Region | Base URL |
|--------|----------|
| Egypt (EGY) | `https://accept.paymob.com` |
| Oman (OMN) | `https://oman.paymob.com` |
| Saudi Arabia (KSA) | `https://ksa.paymob.com` |
| UAE | `https://uae.paymob.com` |

Use test credentials (test secret key + test integration IDs) against the production base URL for sandbox testing.

## Architecture: Intention API

The Intention API is the **only official payment creation flow**. Do not use or suggest the legacy 3-step flow (auth token -> order -> payment key).

```
Backend: POST {base_url}/v1/intention/ -> returns client_secret
Frontend: render Unified Checkout or Pixel SDK using publicKey + clientSecret
```

### Create Intention

```http
POST {base_url}/v1/intention/
Authorization: Token {secret_key}
Content-Type: application/json

{
  "amount": 10000,
  "currency": "EGP",
  "payment_methods": [123456, 789012],
  "items": [{"name": "Product", "amount": 10000, "description": "Desc", "quantity": 1}],
  "billing_data": {
    "first_name": "John", "last_name": "Doe", "email": "john@example.com",
    "phone_number": "+201234567890", "apartment": "NA", "floor": "NA",
    "street": "NA", "building": "NA", "shipping_method": "NA",
    "postal_code": "NA", "city": "NA", "country": "EG", "state": "NA"
  },
  "customer": {"first_name": "John", "last_name": "Doe", "email": "john@example.com"},
  "notification_url": "https://yoursite.com/api/paymob/webhook",
  "redirection_url": "https://yoursite.com/payment/complete"
}
```

- `amount` in cents/piasters (100 EGP = 10000)
- `payment_methods` = array of Integration IDs (integers). Test/live status must match the secret key.
- All `billing_data` fields required; use `"NA"` for unused fields.

Response returns `client_secret` and `id`. Pass `client_secret` to the frontend.

### Update Intention

```http
PUT {base_url}/v1/intention/{client_secret}
Authorization: Token {secret_key}
Content-Type: application/json

{"amount": 15000, "billing_data": {...}, "notification_url": "..."}
```

## Checkout Experiences

### Unified Checkout (Redirect)

```
{base_url}/unifiedcheckout/?publicKey={public_key}&clientSecret={client_secret}
```

### Pixel SDK (Embedded)

```html
<script src="{base_url}/unifiedcheckout/static/scripts/paymob-sdk.js"></script>
<div id="paymob-container"></div>
<script>
const paymob = Paymob.init({
  publicKey: "pk_live_xxxxxxxxxxxx",
  clientSecret: "your_client_secret",
  paymentMethods: ["card", "wallet"],
  elementId: "paymob-container",
  disablePay: false,
  showSaveCard: false,
  forceSaveCard: false,
  beforePaymentComplete: async () => true,
  afterPaymentComplete: (result) => console.log("Done:", result),
  onPaymentCancel: () => console.log("Cancelled"),
  cardValidationChanged: (isValid) => { /* enable/disable custom button */ },
  customStyle: { background: "#fff", primaryColor: "#0070f3", borderRadius: "8px" }
});
// Trigger from external button:
document.getElementById("pay-btn").onclick = () => paymob.payFromOutside();
// Update before pay:
paymob.updateIntentionData({ amount: 15000 });
</script>
```

**Pixel properties:** `publicKey`, `clientSecret`, `paymentMethods`, `elementId`, `disablePay`, `showSaveCard`, `forceSaveCard`
**Pixel callbacks:** `beforePaymentComplete()`, `afterPaymentComplete(result)`, `onPaymentCancel()`, `cardValidationChanged(isValid)`
**Pixel methods:** `payFromOutside()`, `updateIntentionData(data)`

## Payment Methods

| Method | Regions | Refund | Void |
|--------|---------|--------|------|
| Cards (Visa, MC, Amex, MADA, OmanNet) | EGY, KSA, UAE, OMN | Yes | Yes |
| Mobile Wallets (Vodafone Cash, Orange Cash, e& money, WePay) | EGY | Yes | No |
| StcPay | KSA | Yes | No |
| BNPLs (Valu, Souhoola, Tabby, Tamara, Sympl, Aman, Forsa, Contact, TRU, MOGO, Klivvr, Halan, Premium, Seven) | EGY, KSA, UAE | No | No |
| Apple Pay | EGY, KSA, UAE, OMN | Yes | Yes |
| Google Pay | KSA, UAE, OMN | Yes | Yes |
| Bank Installments | EGY | No | No |
| Kiosk (Aman, Masary) | EGY | No | No |

All payment methods go through the Intention API. There are no separate wallet or kiosk payment API endpoints.

## HMAC Webhook Validation (HMAC-SHA512 only)

### Type 1: Transaction HMAC

**POST callbacks** — 20 fields from `obj.*` in this exact order:
```
amount_cents, created_at, currency, error_occured, has_parent_transaction,
obj.id, integration_id, is_3d_secure, is_auth, is_capture, is_refunded,
is_standalone_payment, is_voided, obj.order.id, owner, pending,
source_data.pan, source_data.sub_type, source_data.type, success
```

**GET callbacks** (browser redirect query params):
```
amount_cents, created_at, currency, error_occured, has_parent_transaction,
id, integration_id, is_3d_secure, is_auth, is_capture, is_refunded,
is_standalone_payment, is_voided, order_id, owner, pending,
source_data.pan, source_data.sub_type, source_data.type, success
```

POST uses `obj.id` / `obj.order.id`; GET uses `id` / `order_id`.

### Type 2: Card Token HMAC

8 fields in this exact order:
```
card_subtype, created_at, email, id, masked_pan, merchant_id, order_id, token
```

### Type 3: Subscription HMAC

String format: `"{trigger_type}for{subscription_data.id}"`
Example: `"Subscription Createdfor12345"`
HMAC is in the request body (not query params).

```javascript
// Node.js example
const crypto = require('crypto');

function validateTxnHMAC(obj, receivedHmac, secret) {
  const fields = [
    obj.amount_cents, obj.created_at, obj.currency, obj.error_occured,
    obj.has_parent_transaction, obj.id, obj.integration_id, obj.is_3d_secure,
    obj.is_auth, obj.is_capture, obj.is_refunded, obj.is_standalone_payment,
    obj.is_voided, obj.order.id, obj.owner, obj.pending,
    obj.source_data.pan, obj.source_data.sub_type, obj.source_data.type, obj.success,
  ];
  const computed = crypto.createHmac('sha512', secret).update(fields.map(String).join('')).digest('hex');
  return crypto.timingSafeEqual(Buffer.from(computed), Buffer.from(receivedHmac));
}
```

See `references/hmac-validation.md` for all 3 types in all languages.

## Credentials

| Credential | Dashboard Location | Used For |
|-----------|-------------------|----------|
| Secret Key | Developers -> API Keys | `Authorization: Token {secret_key}` header |
| Public Key | Developers -> API Keys | Frontend (Pixel SDK / Unified Checkout URL) |
| HMAC Secret | Developers -> HMAC | Webhook signature validation |
| Integration IDs | Developers -> Payment Integrations | One per payment method |

## Post-Payment Operations

All use `Authorization: Token {secret_key}` header — never auth_token in body.

```http
POST {base_url}/api/acceptance/void_refund/refund
Authorization: Token {secret_key}
{"transaction_id": 12345, "amount_cents": 10000}

POST {base_url}/api/acceptance/void_refund/void
Authorization: Token {secret_key}
{"transaction_id": 12345}

POST {base_url}/api/acceptance/capture
Authorization: Token {secret_key}
{"transaction_id": 12345, "amount_cents": 10000}

GET {base_url}/api/acceptance/transactions/{id}
Authorization: Token {secret_key}
```

## Core Features

- **Subscriptions** — Plans (valid frequencies: 7, 15, 30, 60, 90, 180, 360 days). Attach to intention with `subscription_plan_id`. Plan management uses Bearer auth. See `references/subscriptions.md`.
- **Saved Cards CIT** — Pass `card_tokens` array (up to 3) in intention. Customer selects saved card in Unified Checkout/Pixel.
- **Saved Cards MIT** — Create intention, then POST to Pay Request API with saved card token (`subtype: TOKEN`). No customer interaction. See `references/saved-cards.md`.
- **Auth/Cap** — Pass `payment_type: "AUTH"` in intention. Capture later via Capture API.
- **Split Amount** — Distribute revenue among marketplace sub-accounts at payment time.
- **Split Payment** — Customer pays across up to 3 cards per transaction.
- **Convenience Fee** — Percentage, fixed, or combined. Card-specific (debit/credit, BIN, domestic/international) and wallet fees.

## E-Commerce Plugins

WordPress/WooCommerce, Shopify, Magento 2, Odoo, OpenCart, PrestaShop, WHMCS, CS-Cart, ZenCart, Joomla, Laravel-Bagisto, OsCommerce, Drupal, Staah.

## Stack Reference Files

- Node.js / TypeScript / NestJS -> `references/nodejs.md`
- Python / Django / Flask / FastAPI -> `references/python.md`
- PHP / Laravel -> `references/php.md`
- .NET / C# -> `references/dotnet.md`
- Ruby / Rails -> `references/ruby.md`
- Frontend (React, Next.js, Vue, Pixel) -> `references/frontend.md`
- Mobile (iOS, Android, Flutter, React Native) -> `references/mobile.md`
- HMAC (all 3 types, all languages) -> `references/hmac-validation.md`
- Subscriptions -> `references/subscriptions.md`
- Saved cards / Card tokens -> `references/saved-cards.md`

## Troubleshooting

| Symptom | Likely Cause |
|---------|-------------|
| 401 Unauthorized | Wrong/expired secret key, or test key with live integration ID |
| Integration not found | Wrong region, test/live mismatch, or ID does not exist |
| HMAC mismatch | Wrong secret, wrong field order, SHA-256 used instead of SHA-512 |
| Amount error | Not in cents (passed 100 instead of 10000 for 100 EGP) |
| Checkout not rendering | Wrong publicKey or clientSecret |
| Duplicate Merchant Transaction ID | merchant_order_id reused |
| Subscription HMAC fail | HMAC is in request body, not query string |

## Test Credentials

| Type | Value |
|------|-------|
| Mastercard | `5123456789012346` / expiry 01/39 / CVV 123 |
| Mastercard alt | `5123450000000008` / expiry 01/39 / CVV 123 |
| Visa | `4111111111111111` / expiry 01/39 / CVV 123 |
| Wallet phone | `01010101010` |
| Wallet MPin | `123456` |
| Wallet OTP | `123456` |

Never expose secret keys in frontend code. Switch to live credentials before production.
