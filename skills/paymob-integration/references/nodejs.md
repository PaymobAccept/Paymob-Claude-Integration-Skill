# Paymob Integration -- Node.js / TypeScript

## Environment Variables (.env)

```env
PAYMOB_SECRET_KEY=sk_live_xxxxxxx
PAYMOB_PUBLIC_KEY=pk_live_xxxxxxx
PAYMOB_HMAC_SECRET=your_hmac_secret
PAYMOB_INTEGRATION_ID_CARD=123456
PAYMOB_BASE_URL=https://accept.paymob.com
APP_URL=https://yoursite.com
```

## Installation

```bash
npm install express axios crypto dotenv
```

## Intention API Service (TypeScript)

```typescript
import axios from 'axios';

const BASE = process.env.PAYMOB_BASE_URL!;
const SECRET = process.env.PAYMOB_SECRET_KEY!;

export async function createIntention(data: {
  amount: number; currency: string; paymentMethods: number[];
  customer: { firstName: string; lastName: string; email: string; phone?: string };
  items: { name: string; amount: number; quantity: number }[];
  notificationUrl: string; redirectionUrl: string;
}) {
  const response = await axios.post(
    ${BASE}/v1/intention/,
    {
      amount: data.amount,
      currency: data.currency,
      payment_methods: data.paymentMethods,
      items: data.items,
      billing_data: {
        first_name: data.customer.firstName,
        last_name: data.customer.lastName,
        email: data.customer.email,
        phone_number: data.customer.phone || '+20000000000',
        apartment: 'NA', floor: 'NA', street: 'NA',
        building: 'NA', shipping_method: 'NA',
        postal_code: 'NA', city: 'NA', country: 'EG', state: 'NA',
      },
      customer: {
        first_name: data.customer.firstName,
        last_name: data.customer.lastName,
        email: data.customer.email,
      },
      notification_url: data.notificationUrl,
      redirection_url: data.redirectionUrl,
    },
    { headers: { Authorization: Token , 'Content-Type': 'application/json' } }
  );
  return { id: response.data.id, clientSecret: response.data.client_secret };
}

export async function updateIntention(clientSecret: string, updates: Record<string, unknown>) {
  const response = await axios.put(
    ${BASE}/v1/intention/,
    updates,
    { headers: { Authorization: Token  } }
  );
  return response.data;
}
```

## Express Checkout Route

```typescript
import express from 'express';
import crypto from 'crypto';

const router = express.Router();

router.post('/api/checkout', async (req, res) => {
  const { amount, items, customer } = req.body;
  const intention = await createIntention({
    amount: Math.round(amount * 100),
    currency: 'EGP',
    paymentMethods: [Number(process.env.PAYMOB_INTEGRATION_ID_CARD)],
    customer,
    items: items.map((i: any) => ({ name: i.name, amount: Math.round(i.price * 100), quantity: i.quantity })),
    notificationUrl: ${process.env.APP_URL}/api/paymob/webhook,
    redirectionUrl: ${process.env.APP_URL}/payment/complete,
  });
  res.json({
    clientSecret: intention.clientSecret,
    checkoutUrl: ${process.env.PAYMOB_BASE_URL}/unifiedcheckout/?publicKey=&clientSecret=,
  });
});

router.post('/api/paymob/webhook', express.json(), (req, res) => {
  const obj = req.body.obj;
  const receivedHMAC = req.query.hmac as string;
  if (!validateTransactionHMAC(obj, receivedHMAC)) {
    return res.status(401).json({ error: 'Invalid HMAC' });
  }
  if (obj.success === true) {
    console.log('Payment', obj.id, 'succeeded for order', obj.order.id);
  }
  res.status(200).json({ received: true });
});

function validateTransactionHMAC(obj: any, received: string): boolean {
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
  const computed = crypto.createHmac('sha512', process.env.PAYMOB_HMAC_SECRET!).update(fields.map(String).join('')).digest('hex');
  return crypto.timingSafeEqual(Buffer.from(computed), Buffer.from(received));
}

export default router;
```

## Post-Payment Operations

```typescript
const headers = { Authorization: Token  };

async function refundTransaction(transactionId: number, amountCents: number) {
  return (await axios.post(${BASE}/api/acceptance/void_refund/refund, { transaction_id: transactionId, amount_cents: amountCents }, { headers })).data;
}
async function voidTransaction(transactionId: number) {
  return (await axios.post(${BASE}/api/acceptance/void_refund/void, { transaction_id: transactionId }, { headers })).data;
}
async function captureTransaction(transactionId: number, amountCents: number) {
  return (await axios.post(${BASE}/api/acceptance/capture, { transaction_id: transactionId, amount_cents: amountCents }, { headers })).data;
}
async function getTransaction(transactionId: number) {
  return (await axios.get(${BASE}/api/acceptance/transactions/, { headers })).data;
}
```

## Useful Packages

- xios -- HTTP client
- crypto -- Built-in Node.js HMAC module
- express / astify / @nestjs/common -- Web framework
