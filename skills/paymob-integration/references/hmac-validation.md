# Paymob HMAC Validation -- Complete Reference

Always use **SHA-512**. Paymob sends HMAC as a query parameter on both webhook notification and redirect URLs.

## Transaction Webhook (POST) HMAC

**20 fields concatenated in this exact order** from obj.*:

| # | Field |
|---|-------|
| 1 | obj.amount_cents |
| 2 | obj.created_at |
| 3 | obj.currency |
| 4 | obj.error_occured |
| 5 | obj.has_parent_transaction |
| 6 | obj.id |
| 7 | obj.integration_id |
| 8 | obj.is_3d_secure |
| 9 | obj.is_auth |
| 10 | obj.is_capture |
| 11 | obj.is_refunded |
| 12 | obj.is_standalone_payment |
| 13 | obj.is_voided |
| 14 | obj.order.id |
| 15 | obj.owner |
| 16 | obj.pending |
| 17 | obj.source_data.pan |
| 18 | obj.source_data.sub_type |
| 19 | obj.source_data.type |
| 20 | obj.success |

## Transaction Redirect (GET) HMAC

**Same 20 fields** but from query parameters. Key differences:

| POST field | GET query param |
|------------|-----------------|
| obj.id | id |
| obj.order.id | order_id |
| All others | Same name, no obj. prefix |

## Card Token HMAC

**8 fields concatenated** in this order:

| # | Field |
|---|-------|
| 1 | card_subtype |
| 2 | created_at |
| 3 | email |
| 4 | id |
| 5 | masked_pan |
| 6 | merchant_id |
| 7 | order_id |
| 8 | 	oken |

## Subscription HMAC

The concatenation string is: "{trigger_type}for{subscription_data.id}"

HMAC is sent **in the request body** (not as query param).

## Node.js Implementation

```typescript
import crypto from 'crypto';

const HMAC_SECRET = process.env.PAYMOB_HMAC_SECRET!;

// POST webhook - from req.body.obj
function validateTransactionPostHmac(obj: any, received: string): boolean {
  const fields = [
    obj.amount_cents, obj.created_at, obj.currency,
    obj.error_occured, obj.has_parent_transaction,
    obj.id, obj.integration_id, obj.is_3d_secure,
    obj.is_auth, obj.is_capture, obj.is_refunded,
    obj.is_standalone_payment, obj.is_voided,
    obj.order.id, obj.owner, obj.pending,
    obj.source_data.pan, obj.source_data.sub_type,
    obj.source_data.type, obj.success,
  ];
  const concat = fields.map(String).join('');
  const computed = crypto.createHmac('sha512', HMAC_SECRET).update(concat).digest('hex');
  return crypto.timingSafeEqual(Buffer.from(computed), Buffer.from(received));
}

// GET redirect - from req.query
function validateTransactionGetHmac(query: Record<string, string>, received: string): boolean {
  const fields = [
    query.amount_cents, query.created_at, query.currency,
    query.error_occured, query.has_parent_transaction,
    query.id, query.integration_id, query.is_3d_secure,
    query.is_auth, query.is_capture, query.is_refunded,
    query.is_standalone_payment, query.is_voided,
    query.order_id, query.owner, query.pending,
    query.source_data_pan, query.source_data_sub_type,
    query.source_data_type, query.success,
  ];
  const concat = fields.map(String).join('');
  const computed = crypto.createHmac('sha512', HMAC_SECRET).update(concat).digest('hex');
  return crypto.timingSafeEqual(Buffer.from(computed), Buffer.from(received));
}

// Card token
function validateCardTokenHmac(data: any, received: string): boolean {
  const fields = [
    data.card_subtype, data.created_at, data.email, data.id,
    data.masked_pan, data.merchant_id, data.order_id, data.token,
  ];
  const concat = fields.map(String).join('');
  const computed = crypto.createHmac('sha512', HMAC_SECRET).update(concat).digest('hex');
  return crypto.timingSafeEqual(Buffer.from(computed), Buffer.from(received));
}
```

## Python Implementation

```python
import hashlib, hmac as hmac_mod

def validate_transaction_post_hmac(obj: dict, received: str, hmac_secret: str) -> bool:
    fields = [
        obj['amount_cents'], obj['created_at'], obj['currency'],
        obj['error_occured'], obj['has_parent_transaction'],
        obj['id'], obj['integration_id'], obj['is_3d_secure'],
        obj['is_auth'], obj['is_capture'], obj['is_refunded'],
        obj['is_standalone_payment'], obj['is_voided'],
        obj['order']['id'], obj['owner'], obj['pending'],
        obj['source_data']['pan'], obj['source_data']['sub_type'],
        obj['source_data']['type'], obj['success'],
    ]
    s = ''.join(str(f) for f in fields)
    computed = hmac_mod.new(hmac_secret.encode(), s.encode(), hashlib.sha512).hexdigest()
    return hmac_mod.compare_digest(computed, received)

def validate_transaction_get_hmac(query: dict, received: str, hmac_secret: str) -> bool:
    fields = [
        query.get('amount_cents'), query.get('created_at'), query.get('currency'),
        query.get('error_occured'), query.get('has_parent_transaction'),
        query.get('id'), query.get('integration_id'), query.get('is_3d_secure'),
        query.get('is_auth'), query.get('is_capture'), query.get('is_refunded'),
        query.get('is_standalone_payment'), query.get('is_voided'),
        query.get('order_id'), query.get('owner'), query.get('pending'),
        query.get('source_data_pan'), query.get('source_data_sub_type'),
        query.get('source_data_type'), query.get('success'),
    ]
    s = ''.join(str(f) for f in fields)
    computed = hmac_mod.new(hmac_secret.encode(), s.encode(), hashlib.sha512).hexdigest()
    return hmac_mod.compare_digest(computed, received)

def validate_card_token_hmac(data: dict, received: str, hmac_secret: str) -> bool:
    fields = [
        data['card_subtype'], data['created_at'], data['email'], data['id'],
        data['masked_pan'], data['merchant_id'], data['order_id'], data['token'],
    ]
    s = ''.join(str(f) for f in fields)
    computed = hmac_mod.new(hmac_secret.encode(), s.encode(), hashlib.sha512).hexdigest()
    return hmac_mod.compare_digest(computed, received)
```

## PHP Implementation

```php
function validateTransactionPostHmac(array \, string \, string \): bool {
    \ = [
        \['amount_cents'], \['created_at'], \['currency'],
        \['error_occured'], \['has_parent_transaction'],
        \['id'], \['integration_id'], \['is_3d_secure'],
        \['is_auth'], \['is_capture'], \['is_refunded'],
        \['is_standalone_payment'], \['is_voided'],
        \['order']['id'], \['owner'], \['pending'],
        \['source_data']['pan'], \['source_data']['sub_type'],
        \['source_data']['type'], \['success'],
    ];
    \ = implode('', array_map('strval', \));
    \ = hash_hmac('sha512', \, \);
    return hash_equals(\, \);
}
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using SHA-256 | Always use SHA-512 |
| Wrong field order | Follow exact 20-field order above |
| Using POST fields for GET redirect | Use id/order_id not obj.id/obj.order.id |
| Not using timing-safe compare | Use crypto.timingSafeEqual / hmac.compare_digest / hash_equals |
| Logging the HMAC secret | Never log secrets |
| Trusting success=true without HMAC | Always validate HMAC first |
