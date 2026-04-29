# Paymob Subscriptions

Paymob supports recurring billing through the Subscription Plans API. Subscriptions are available for selected merchants -- contact Paymob to enable this feature.

## Authentication

Subscription plan management uses **Bearer token** (retrieved via auth endpoint):

```bash
POST {base_url}/api/auth/tokens
Content-Type: application/json
{"api_key": "your_api_key"}
# Returns: {"token": "...", ...}
```

All other subscription operations use **Token auth** (secret key):
```
Authorization: Token {secret_key}
```

## Create Subscription Plan

```bash
POST {base_url}/api/acceptance/subscription_plans
Authorization: Bearer {auth_token}
Content-Type: application/json

{
  "name": "Monthly Pro",
  "amount_cents": 50000,
  "currency": "EGP",
  "interval": "month",
  "interval_count": 1
}
```

Response includes id (plan ID) to use when creating subscriptions.

## Create Subscription (via Intention API)

Add ecurring to your intention payload:

```json
{
  "amount": 50000,
  "currency": "EGP",
  "payment_methods": [123456],
  "billing_data": { ... },
  "customer": { ... },
  "items": [],
  "notification_url": "https://yoursite.com/webhook",
  "redirection_url": "https://yoursite.com/complete",
  "recurring": {
    "subscription_plan": 789,
    "auto_debit": true
  }
}
```

Authorization: Token {secret_key}

## Subscription Webhook (HMAC)

Paymob sends subscription events to your 
otification_url.

HMAC string: "{trigger_type}for{subscription_data.id}"

HMAC is in the **request body** (not query param).

```typescript
function validateSubscriptionHmac(body: any, hmacSecret: string): boolean {
  const str = ${body.trigger_type}for;
  const computed = crypto.createHmac('sha512', hmacSecret).update(str).digest('hex');
  return crypto.timingSafeEqual(Buffer.from(computed), Buffer.from(body.hmac));
}
```

```python
def validate_subscription_hmac(body: dict, hmac_secret: str) -> bool:
    s = f"{body['trigger_type']}for{body['subscription_data']['id']}"
    computed = hmac.new(hmac_secret.encode(), s.encode(), hashlib.sha512).hexdigest()
    return hmac.compare_digest(computed, body['hmac'])
```

## Cancel Subscription

```bash
POST {base_url}/api/acceptance/subscriptions/{subscription_id}/cancel
Authorization: Token {secret_key}
```

## Get Subscription Status

```bash
GET {base_url}/api/acceptance/subscriptions/{subscription_id}
Authorization: Token {secret_key}
```

## Trigger Types

| 	rigger_type | Description |
|----------------|-------------|
| charge | Recurring charge executed |
| cancel | Subscription cancelled |
| ailure | Charge attempt failed |
| etry | Charge retry |

## Notes

- First payment is always customer-initiated (CIT) through the checkout flow
- Subsequent payments are merchant-initiated (MIT) auto-debits
- Always validate HMAC before processing subscription events
- Store subscription_data.id for future management operations
