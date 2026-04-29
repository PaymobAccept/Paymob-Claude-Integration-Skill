# Paymob Saved Cards -- CIT / MIT

Paymob supports saving card tokens for future payments (saved cards / one-click checkout).

## Customer-Initiated Token (CIT)

The first payment must be customer-initiated. To tokenize the card, include payment_methods with the integration ID configured for tokenization.

### Create Intention with Tokenization

```json
{
  "amount": 10000,
  "currency": "EGP",
  "payment_methods": [123456],
  "billing_data": { ... },
  "customer": { "first_name": "...", "last_name": "...", "email": "user@example.com" },
  "items": [],
  "notification_url": "https://yoursite.com/webhook",
  "redirection_url": "https://yoursite.com/complete",
  "extras": { "save_card": true }
}
```

Authorization: Token {secret_key}

## Card Token Webhook

After successful tokenization, Paymob sends a card token webhook with these fields:

| Field | Description |
|-------|-------------|
| 	oken | Card token for future MIT payments |
| masked_pan | Masked card number (e.g., 512345XXXXXX1234) |
| card_subtype | Visa / MasterCard |
| email | Customer email |
| order_id | Associated order ID |
| merchant_id | Your merchant ID |
| id | Token record ID |
| created_at | Timestamp |

### Card Token HMAC (SHA-512)

8 fields concatenated in this order:
card_subtype + created_at + email + id + masked_pan + merchant_id + order_id + token

```typescript
function validateCardTokenHmac(data: any, hmacSecret: string): boolean {
  const fields = [
    data.card_subtype, data.created_at, data.email, data.id,
    data.masked_pan, data.merchant_id, data.order_id, data.token,
  ];
  const concat = fields.map(String).join('');
  const computed = crypto.createHmac('sha512', hmacSecret).update(concat).digest('hex');
  return crypto.timingSafeEqual(Buffer.from(computed), Buffer.from(data.hmac ?? ''));
}
```

```python
def validate_card_token_hmac(data: dict, hmac_secret: str) -> bool:
    fields = [
        data['card_subtype'], data['created_at'], data['email'], data['id'],
        data['masked_pan'], data['merchant_id'], data['order_id'], data['token'],
    ]
    s = ''.join(str(f) for f in fields)
    computed = hmac.new(hmac_secret.encode(), s.encode(), hashlib.sha512).hexdigest()
    return hmac.compare_digest(computed, data.get('hmac', ''))
```

## Merchant-Initiated Token (MIT)

Use the saved token to charge the customer without their interaction:

```bash
POST {base_url}/api/acceptance/payments/pay
Authorization: Token {secret_key}
Content-Type: application/json

{
  "source": {
    "identifier": "CARD_TOKEN_HERE",
    "subtype": "TOKEN"
  },
  "payment_token": "INTENTION_CLIENT_SECRET_HERE"
}
```

### MIT Flow

1. Create intention normally (same create_intention API call)
2. Instead of redirecting customer, call /api/acceptance/payments/pay with token
3. Handle response -- success: true means charged

```typescript
async function chargeWithToken(token: string, clientSecret: string) {
  const response = await axios.post(
    ${BASE_URL}/api/acceptance/payments/pay,
    {
      source: { identifier: token, subtype: 'TOKEN' },
      payment_token: clientSecret,
    },
    { headers: { Authorization: Token  } }
  );
  return response.data;
}
```

## Auth / Capture Flow

Use is_auth: true in intention for two-step authorization + capture:

```json
{
  "amount": 10000,
  "currency": "EGP",
  "payment_methods": [123456],
  "is_auth": true,
  ...
}
```

Then capture later:

```bash
POST {base_url}/api/acceptance/capture
Authorization: Token {secret_key}
Content-Type: application/json

{"transaction_id": 12345, "amount_cents": 10000}
```

## Security Notes

- Store only 	oken and masked_pan -- never store raw card data
- Always validate card token HMAC before trusting the token
- MIT charges must comply with PCI DSS requirements
- Verify the email in token webhook matches your customer record
