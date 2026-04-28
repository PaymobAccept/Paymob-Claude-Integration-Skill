# HMAC Validation Reference

HMAC validation is the most critical security step. Every callback from Paymob must be verified before processing.

## Field Extraction Order

Extract these fields from the transaction callback object, **in this exact order**, and concatenate their string values:

```
amount_cents
created_at
currency
error_occured          (note: Paymob uses this spelling, not "error_occurred")
has_parent_transaction
id
integration_id
is_3d_secure
is_auth
is_capture
is_refunded
is_standalone_payment
is_voided
order.id               (nested: transaction.order.id)
owner
pending
source_data.pan        (nested: transaction.source_data.pan)
source_data.sub_type   (nested: transaction.source_data.sub_type)
source_data.type       (nested: transaction.source_data.type)
success
```

## Node.js Implementation

```javascript
const crypto = require('crypto');

function validateHMAC(transaction, receivedHMAC, hmacSecret) {
  const fields = [
    transaction.amount_cents,
    transaction.created_at,
    transaction.currency,
    transaction.error_occured,
    transaction.has_parent_transaction,
    transaction.id,
    transaction.integration_id,
    transaction.is_3d_secure,
    transaction.is_auth,
    transaction.is_capture,
    transaction.is_refunded,
    transaction.is_standalone_payment,
    transaction.is_voided,
    transaction.order.id,
    transaction.owner,
    transaction.pending,
    transaction.source_data.pan,
    transaction.source_data.sub_type,
    transaction.source_data.type,
    transaction.success,
  ];

  const concatenated = fields.join('');
  const calculatedHMAC = crypto
    .createHmac('sha512', hmacSecret)
    .update(concatenated)
    .digest('hex');

  return calculatedHMAC === receivedHMAC;
}
```

## Python Implementation

```python
import hmac
import hashlib

def validate_hmac(transaction: dict, received_hmac: str, hmac_secret: str) -> bool:
    fields = [
        str(transaction['amount_cents']),
        transaction['created_at'],
        transaction['currency'],
        str(transaction['error_occured']),
        str(transaction['has_parent_transaction']),
        str(transaction['id']),
        str(transaction['integration_id']),
        str(transaction['is_3d_secure']),
        str(transaction['is_auth']),
        str(transaction['is_capture']),
        str(transaction['is_refunded']),
        str(transaction['is_standalone_payment']),
        str(transaction['is_voided']),
        str(transaction['order']['id']),
        str(transaction['owner']),
        str(transaction['pending']),
        transaction['source_data']['pan'],
        transaction['source_data']['sub_type'],
        transaction['source_data']['type'],
        str(transaction['success']),
    ]

    concatenated = ''.join(fields)
    calculated = hmac.new(
        hmac_secret.encode('utf-8'),
        concatenated.encode('utf-8'),
        hashlib.sha512
    ).hexdigest()

    return hmac.compare_digest(calculated, received_hmac)
```

## PHP Implementation

```php
function validateHMAC(array $transaction, string $receivedHMAC, string $hmacSecret): bool
{
    $fields = [
        $transaction['amount_cents'],
        $transaction['created_at'],
        $transaction['currency'],
        $transaction['error_occured'],
        $transaction['has_parent_transaction'],
        $transaction['id'],
        $transaction['integration_id'],
        $transaction['is_3d_secure'],
        $transaction['is_auth'],
        $transaction['is_capture'],
        $transaction['is_refunded'],
        $transaction['is_standalone_payment'],
        $transaction['is_voided'],
        $transaction['order']['id'],
        $transaction['owner'],
        $transaction['pending'],
        $transaction['source_data']['pan'],
        $transaction['source_data']['sub_type'],
        $transaction['source_data']['type'],
        $transaction['success'],
    ];

    $concatenated = implode('', array_map('strval', $fields));
    $calculated = hash_hmac('sha512', $concatenated, $hmacSecret);

    return hash_equals($calculated, $receivedHMAC);
}
```

## C# / .NET Implementation

```csharp
using System.Security.Cryptography;
using System.Text;

public static bool ValidateHMAC(Transaction transaction, string receivedHMAC, string hmacSecret)
{
    var fields = new string[]
    {
        transaction.AmountCents.ToString(),
        transaction.CreatedAt,
        transaction.Currency,
        transaction.ErrorOccured.ToString().ToLower(),
        transaction.HasParentTransaction.ToString().ToLower(),
        transaction.Id.ToString(),
        transaction.IntegrationId.ToString(),
        transaction.Is3dSecure.ToString().ToLower(),
        transaction.IsAuth.ToString().ToLower(),
        transaction.IsCapture.ToString().ToLower(),
        transaction.IsRefunded.ToString().ToLower(),
        transaction.IsStandalonePayment.ToString().ToLower(),
        transaction.IsVoided.ToString().ToLower(),
        transaction.Order.Id.ToString(),
        transaction.Owner.ToString(),
        transaction.Pending.ToString().ToLower(),
        transaction.SourceData.Pan,
        transaction.SourceData.SubType,
        transaction.SourceData.Type,
        transaction.Success.ToString().ToLower(),
    };

    var concatenated = string.Join("", fields);
    using var hmac = new HMACSHA512(Encoding.UTF8.GetBytes(hmacSecret));
    var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(concatenated));
    var calculated = BitConverter.ToString(hash).Replace("-", "").ToLower();

    return calculated == receivedHMAC;
}
```

## Common Pitfalls

1. **Boolean serialization** — Paymob sends `"true"/"false"` as strings. Make sure your concatenation produces lowercase `"true"` or `"false"`, not `"True"` or `"1"`.
2. **The typo `error_occured`** — Paymob uses this spelling (missing 'r'). Don't "fix" it to `error_occurred`.
3. **Field order matters** — The fields MUST be concatenated in the exact order listed above.
4. **Nested fields** — `order.id`, `source_data.pan`, `source_data.sub_type`, and `source_data.type` are nested objects.
5. **Hash algorithm** — Use HMAC-SHA512, not SHA256 or SHA1.
