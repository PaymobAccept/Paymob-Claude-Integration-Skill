# Paymob Integration — Node.js / TypeScript

## Quick Start with Express

### Installation

```bash
npm install express axios crypto dotenv
```

### Environment Variables (.env)

```env
PAYMOB_API_KEY=your_api_key
PAYMOB_SECRET_KEY=sk_live_xxxxxxx
PAYMOB_PUBLIC_KEY=pk_live_xxxxxxx
PAYMOB_HMAC_SECRET=your_hmac_secret
PAYMOB_INTEGRATION_ID_CARD=123456
PAYMOB_INTEGRATION_ID_WALLET=789012
PAYMOB_INTEGRATION_ID_KIOSK=345678
PAYMOB_IFRAME_ID=12345
PAYMOB_BASE_URL=https://accept.paymob.com
```

### Intention API (Recommended)

```typescript
import axios from 'axios';

const PAYMOB_BASE = process.env.PAYMOB_BASE_URL;

interface PaymentIntentionRequest {
  amount: number;        // in cents
  currency: string;
  paymentMethods: number[];
  customer: { firstName: string; lastName: string; email: string };
  items: { name: string; amount: number; quantity: number }[];
  notificationUrl: string;
  redirectionUrl: string;
}

export async function createIntention(data: PaymentIntentionRequest) {
  const response = await axios.post(
    `${PAYMOB_BASE}/v1/intention/`,
    {
      amount: data.amount,
      currency: data.currency,
      payment_methods: data.paymentMethods,
      items: data.items,
      billing_data: {
        first_name: data.customer.firstName,
        last_name: data.customer.lastName,
        email: data.customer.email,
        phone_number: '+20000000000',
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
    {
      headers: {
        Authorization: `Token ${process.env.PAYMOB_SECRET_KEY}`,
        'Content-Type': 'application/json',
      },
    }
  );

  return {
    intentionId: response.data.id,
    clientSecret: response.data.client_secret,
  };
}
```

### Legacy 3-Step Flow

```typescript
import axios from 'axios';

const PAYMOB_BASE = process.env.PAYMOB_BASE_URL;

// Step 1: Authenticate
async function authenticate(): Promise<string> {
  const { data } = await axios.post(`${PAYMOB_BASE}/api/auth/tokens`, {
    api_key: process.env.PAYMOB_API_KEY,
  });
  return data.token;
}

// Step 2: Create Order
async function createOrder(authToken: string, amountCents: number, merchantOrderId: string) {
  const { data } = await axios.post(`${PAYMOB_BASE}/api/ecommerce/orders`, {
    auth_token: authToken,
    delivery_needed: 'false',
    amount_cents: amountCents.toString(),
    currency: 'EGP',
    merchant_order_id: merchantOrderId,
    items: [],
  });
  return data.id;
}

// Step 3: Get Payment Key
async function getPaymentKey(
  authToken: string,
  orderId: number,
  amountCents: number,
  integrationId: number,
  billingData: Record<string, string>
) {
  const { data } = await axios.post(`${PAYMOB_BASE}/api/acceptance/payment_keys`, {
    auth_token: authToken,
    amount_cents: amountCents.toString(),
    expiration: 3600,
    order_id: orderId,
    billing_data: {
      first_name: billingData.firstName || 'NA',
      last_name: billingData.lastName || 'NA',
      email: billingData.email || 'NA',
      phone_number: billingData.phone || 'NA',
      apartment: 'NA', floor: 'NA', street: 'NA',
      building: 'NA', shipping_method: 'NA',
      postal_code: 'NA', city: 'NA', country: 'EG', state: 'NA',
    },
    currency: 'EGP',
    integration_id: integrationId,
  });
  return data.token;
}
```

### Express Routes

```typescript
import express from 'express';
import crypto from 'crypto';

const router = express.Router();

// Create checkout session
router.post('/api/checkout', async (req, res) => {
  try {
    const { amount, items, customer } = req.body;
    const amountCents = Math.round(amount * 100);

    // Using Intention API
    const intention = await createIntention({
      amount: amountCents,
      currency: 'EGP',
      paymentMethods: [Number(process.env.PAYMOB_INTEGRATION_ID_CARD)],
      customer,
      items: items.map((i: any) => ({
        name: i.name,
        amount: Math.round(i.price * 100),
        quantity: i.quantity,
      })),
      notificationUrl: `${process.env.APP_URL}/api/paymob/webhook`,
      redirectionUrl: `${process.env.APP_URL}/payment/complete`,
    });

    res.json({
      clientSecret: intention.clientSecret,
      iframeUrl: `https://accept.paymob.com/unifiedcheckout/?publicKey=${process.env.PAYMOB_PUBLIC_KEY}&clientSecret=${intention.clientSecret}`,
    });
  } catch (error) {
    console.error('Checkout error:', error);
    res.status(500).json({ error: 'Payment initialization failed' });
  }
});

// Webhook handler
router.post('/api/paymob/webhook', express.json(), (req, res) => {
  const transaction = req.body.obj;
  const receivedHMAC = req.query.hmac as string;

  if (!validateHMAC(transaction, receivedHMAC)) {
    console.error('HMAC validation failed');
    return res.status(401).json({ error: 'Invalid HMAC' });
  }

  if (transaction.success === true) {
    // Payment successful — update your order in DB
    console.log(`Payment ${transaction.id} succeeded for order ${transaction.order.id}`);
    // await updateOrderStatus(transaction.order.merchant_order_id, 'paid');
  } else {
    // Payment failed
    console.log(`Payment ${transaction.id} failed: ${transaction.data?.message}`);
  }

  res.status(200).json({ received: true });
});

function validateHMAC(transaction: any, receivedHMAC: string): boolean {
  const fields = [
    transaction.amount_cents, transaction.created_at, transaction.currency,
    transaction.error_occured, transaction.has_parent_transaction,
    transaction.id, transaction.integration_id, transaction.is_3d_secure,
    transaction.is_auth, transaction.is_capture, transaction.is_refunded,
    transaction.is_standalone_payment, transaction.is_voided,
    transaction.order.id, transaction.owner, transaction.pending,
    transaction.source_data.pan, transaction.source_data.sub_type,
    transaction.source_data.type, transaction.success,
  ];

  const concatenated = fields.map(String).join('');
  const calculated = crypto
    .createHmac('sha512', process.env.PAYMOB_HMAC_SECRET!)
    .update(concatenated)
    .digest('hex');

  return calculated === receivedHMAC;
}

export default router;
```

### NestJS Service Pattern

```typescript
import { Injectable, HttpException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { HttpService } from '@nestjs/axios';
import { firstValueFrom } from 'rxjs';

@Injectable()
export class PaymobService {
  private readonly baseUrl: string;
  private readonly secretKey: string;

  constructor(
    private readonly http: HttpService,
    private readonly config: ConfigService,
  ) {
    this.baseUrl = this.config.get('PAYMOB_BASE_URL');
    this.secretKey = this.config.get('PAYMOB_SECRET_KEY');
  }

  async createPaymentIntention(amount: number, currency: string, integrationIds: number[]) {
    const { data } = await firstValueFrom(
      this.http.post(`${this.baseUrl}/v1/intention/`, {
        amount,
        currency,
        payment_methods: integrationIds,
        // ... billing_data, items, etc.
      }, {
        headers: { Authorization: `Token ${this.secretKey}` },
      })
    );
    return data;
  }
}
```

## Useful npm Packages

- `paymob-node` — Community SDK (check npm for latest)
- `axios` — HTTP client
- `crypto` — Built-in Node.js module for HMAC
- `express` / `fastify` / `@nestjs/common` — Web framework
