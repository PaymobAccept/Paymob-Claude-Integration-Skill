---
name: paymob-integration
description: >
  Guides developers through integrating Paymob payment gateway into their applications.
  Covers both the modern Intention API and legacy Accept API flows for accepting card payments,
  mobile wallets (Vodafone Cash, Orange Money, etc.), kiosk payments, and cash collection.
  Supports all major tech stacks: Node.js/TypeScript, Python/Django/Flask, PHP/Laravel,
  .NET/C#, Ruby, and frontend frameworks (React, Next.js, Vue, Flutter).
  Use this skill whenever the user mentions Paymob, WeAccept, payment integration in Egypt
  or MENA region, or needs help with Paymob's authentication, order registration, payment keys,
  intention APIs, HMAC validation, webhooks/callbacks, iframes, or checkout experiences.
  Also trigger when the user asks about accepting payments in Egyptian Pound (EGP),
  Pakistani Rupee (PKR), or other Paymob-supported currencies.
---

# Paymob Integration Guide

You are an expert developer assistant specializing in Paymob payment gateway integration. Your job is to help developers integrate Paymob's Accept Payments product into their applications — from initial setup through production deployment.

## When to Use This Skill

This skill activates whenever a developer needs help with:
- Setting up Paymob payment acceptance (cards, wallets, kiosk, cash)
- Authenticating with Paymob APIs
- Creating payment intentions or orders
- Generating payment keys and embedding iframes
- Handling webhooks/callbacks and HMAC validation
- Debugging Paymob integration issues
- Migrating between Paymob API versions

## Architecture Overview

Paymob offers **two API generations**. The newer **Intention API** is the recommended path for new integrations, while the **Legacy Accept API** is still widely used and documented. Both are fully supported.

### Intention API (Recommended for New Projects)

The modern flow is simpler — a single API call creates a payment intention:

```
Backend → POST /v1/intention/ → Paymob returns client_secret → Frontend renders checkout
```

**Base URLs:**
- Production: `https://accept.paymob.com` (Egypt) — check Paymob docs for your region
- Staging: `https://next-stg.paymobsolutions.com`

**Authentication:** Add your secret key in the `Authorization` header:
```
Authorization: Token sk_live_xxxxxxxxxxxxx
```

**Create Intention Request:**
```http
POST /v1/intention/
Content-Type: application/json
Authorization: Token {secret_key}

{
  "amount": 10000,
  "currency": "EGP",
  "payment_methods": [123456],
  "items": [
    {
      "name": "Product Name",
      "amount": 10000,
      "description": "Product description",
      "quantity": 1
    }
  ],
  "billing_data": {
    "first_name": "John",
    "last_name": "Doe",
    "email": "john@example.com",
    "phone_number": "+201234567890",
    "apartment": "NA",
    "floor": "NA",
    "street": "NA",
    "building": "NA",
    "shipping_method": "NA",
    "postal_code": "NA",
    "city": "NA",
    "country": "EG",
    "state": "NA"
  },
  "customer": {
    "first_name": "John",
    "last_name": "Doe",
    "email": "john@example.com"
  },
  "notification_url": "https://yoursite.com/api/paymob/webhook",
  "redirection_url": "https://yoursite.com/payment/complete"
}
```

**Response** returns `client_secret` and `id` — use `client_secret` to render the checkout on the frontend.

**`payment_methods`** accepts Integration IDs (integers) or integration names (strings like `"card"`, `"wallet"`). The status (Live/Test) of the IDs must match the secret key type.

### Legacy Accept API (3-Step Flow)

The classic flow requires three sequential API calls:

**Step 1 — Authentication:**
```http
POST https://accept.paymob.com/api/auth/tokens
Content-Type: application/json

{
  "api_key": "your_api_key_here"
}
```
Returns `{ "token": "auth_token_here" }`.

**Step 2 — Order Registration:**
```http
POST https://accept.paymob.com/api/ecommerce/orders
Content-Type: application/json

{
  "auth_token": "auth_token_from_step_1",
  "delivery_needed": "false",
  "amount_cents": "10000",
  "currency": "EGP",
  "merchant_order_id": "order_123",
  "items": [
    {
      "name": "Product",
      "amount_cents": "10000",
      "description": "Description",
      "quantity": "1"
    }
  ]
}
```
Returns `{ "id": 12345, ... }` — the order ID.

**Step 3 — Payment Key Request:**
```http
POST https://accept.paymob.com/api/acceptance/payment_keys
Content-Type: application/json

{
  "auth_token": "auth_token_from_step_1",
  "amount_cents": "10000",
  "expiration": 3600,
  "order_id": 12345,
  "billing_data": {
    "first_name": "John",
    "last_name": "Doe",
    "email": "john@example.com",
    "phone_number": "+201234567890",
    "apartment": "NA",
    "floor": "NA",
    "street": "NA",
    "building": "NA",
    "shipping_method": "NA",
    "postal_code": "NA",
    "city": "NA",
    "country": "EG",
    "state": "NA"
  },
  "currency": "EGP",
  "integration_id": 123456,
  "lock_order_when_paid": "false"
}
```
Returns `{ "token": "payment_key_token" }`.

## Checkout Experiences

After obtaining the payment key (legacy) or client secret (intention), present the checkout:

### Hosted Checkout / Iframe (Card Payments)
```html
<iframe
  src="https://accept.paymob.com/api/acceptance/iframes/{iframe_id}?payment_token={payment_key}"
  width="100%"
  height="500"
  frameborder="0"
></iframe>
```
Get the `iframe_id` from your Paymob Dashboard → Developers → iFrames.

### Mobile Wallet Payments
```http
POST https://accept.paymob.com/api/acceptance/payments/pay
Content-Type: application/json

{
  "source": {
    "identifier": "01234567890",
    "subtype": "WALLET"
  },
  "payment_token": "{payment_key}"
}
```
Returns a `redirect_url` — redirect the user to complete wallet authorization.

### Kiosk Payments
```http
POST https://accept.paymob.com/api/acceptance/payments/pay
Content-Type: application/json

{
  "source": {
    "identifier": "AGGREGATOR",
    "subtype": "AGGREGATOR"
  },
  "payment_token": "{payment_key}"
}
```
Returns a `bill_reference` number — the customer uses this at any kiosk (Aman, Masary, etc.) to pay.

## Webhook & Callback Handling

Paymob sends two callbacks after payment processing:

1. **Transaction Processed Callback** (POST) — server-to-server notification
2. **Transaction Response Callback** (GET) — browser redirect with query params

### HMAC Validation (Critical for Security)

Every callback must be validated using HMAC to ensure it's genuinely from Paymob and hasn't been tampered with.

**How HMAC works:**
1. Extract specific fields from the callback in a defined order
2. Concatenate them into a single string
3. Hash using HMAC-SHA512 with your HMAC secret
4. Compare with the `hmac` value in the callback

**Fields to concatenate (in this exact order):**
```
amount_cents, created_at, currency, error_occured, has_parent_transaction,
id, integration_id, is_3d_secure, is_auth, is_capture, is_refunded,
is_standalone_payment, is_voided, order.id, owner,
pending, source_data.pan, source_data.sub_type, source_data.type, success
```

Read `references/hmac-validation.md` for language-specific HMAC implementations.

## Credentials & Dashboard Setup

Developers need these from the Paymob Dashboard:
- **API Key** — for legacy authentication (Settings → Account Info)
- **Secret Key** — for Intention API auth header
- **Public Key** — for frontend/client-side usage
- **HMAC Secret** — for webhook validation (Developers → HMAC)
- **Integration ID(s)** — one per payment method (Developers → Payment Integrations)
- **iFrame ID** — for hosted card checkout (Developers → iFrames)

## Important Notes

- **Amounts are always in cents/piasters** — multiply by 100 (e.g., 100 EGP = 10000 cents)
- **Billing data fields are required** even if not relevant — use "NA" for unused fields
- **Test vs Live keys** must match Integration ID status
- **Currency must match** the Integration ID's configured currency
- **All connections use TLS** — validate Paymob's certificate to prevent MITM attacks

## Post-Payment Operations

- **Refund:** `POST /api/acceptance/void_refund/refund` with `auth_token`, `transaction_id`, `amount_cents`
- **Void:** `POST /api/acceptance/void_refund/void` with `auth_token`, `transaction_id`
- **Get Transaction:** `GET /api/acceptance/transactions/{id}` with auth header
- **Capture (auth-only):** `POST /api/acceptance/capture` with `auth_token`, `transaction_id`, `amount_cents`

## Stack-Specific Guidance

When helping a developer, ask which stack they're using and consult the relevant reference file:

- **Node.js / TypeScript** → Read `references/nodejs.md`
- **Python / Django / Flask** → Read `references/python.md`
- **PHP / Laravel** → Read `references/php.md`
- **.NET / C#** → Read `references/dotnet.md`
- **Ruby / Rails** → Read `references/ruby.md`
- **Frontend (React, Next.js, Vue)** → Read `references/frontend.md`
- **Mobile (Flutter, React Native)** → Read `references/mobile.md`

## Troubleshooting Common Issues

When a developer encounters an error, check these first:

| Symptom | Likely Cause |
|---------|-------------|
| "Duplicate Merchant Transaction ID" | Reusing `merchant_order_id` — generate unique IDs |
| 401 / Authentication failed | Wrong API key, or mixing test/live credentials |
| HMAC mismatch | Wrong HMAC secret, wrong field order, or encoding issue |
| Amount mismatch error | `amount_cents` differs between order registration and payment key |
| iframe not loading | Wrong iframe ID or expired payment key (default 1hr expiry) |
| Wallet redirect fails | Invalid phone number format or wrong integration ID |
| "Integration not found" | Integration ID doesn't exist or doesn't match key status (test/live) |

## Testing

Paymob provides test credentials and test card numbers:
- **Test Card (success):** `5123456789012346`, Expiry: any future date, CVV: `123`
- **Test Card (decline):** Check Paymob docs for current decline test cards
- Use test secret key + test integration IDs for sandbox testing
- Webhook testing: use ngrok or similar to expose localhost

Always remind developers to switch to live credentials before going to production, and to never expose secret keys in frontend code.
