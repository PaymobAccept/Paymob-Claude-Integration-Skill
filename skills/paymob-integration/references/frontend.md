# Paymob Integration — Frontend (React, Next.js, Vue)

The frontend's role in Paymob integration is to render the checkout UI after the backend creates a payment intention or payment key.

## Approach 1: Unified Checkout (Intention API)

Paymob's unified checkout is a hosted page that handles all payment methods. After your backend creates an intention and returns a `client_secret`:

```jsx
// React component
function PaymobCheckout({ clientSecret, publicKey }) {
  const checkoutUrl = `https://accept.paymob.com/unifiedcheckout/?publicKey=${publicKey}&clientSecret=${clientSecret}`;

  return (
    <div>
      <h2>Complete Your Payment</h2>
      <a href={checkoutUrl} target="_blank" rel="noopener noreferrer">
        Pay Now
      </a>
      {/* Or embed as iframe */}
      <iframe
        src={checkoutUrl}
        width="100%"
        height="600"
        style={{ border: 'none' }}
        title="Paymob Checkout"
      />
    </div>
  );
}
```

## Approach 2: Iframe (Legacy API)

After your backend returns a `payment_key`:

```jsx
// React component
function PaymobIframe({ paymentKey, iframeId }) {
  const iframeSrc = `https://accept.paymob.com/api/acceptance/iframes/${iframeId}?payment_token=${paymentKey}`;

  return (
    <iframe
      src={iframeSrc}
      width="100%"
      height="500"
      style={{ border: 'none' }}
      title="Paymob Payment"
    />
  );
}
```

## React + Next.js Full Example

```tsx
// app/checkout/page.tsx (Next.js App Router)
'use client';

import { useState } from 'react';

export default function CheckoutPage() {
  const [loading, setLoading] = useState(false);
  const [checkoutUrl, setCheckoutUrl] = useState<string | null>(null);
  const [error, setError] = useState<string | null>(null);

  async function handleCheckout() {
    setLoading(true);
    setError(null);

    try {
      const res = await fetch('/api/checkout', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          amount: 150.00,
          items: [{ name: 'Premium Plan', price: 150, quantity: 1 }],
          customer: {
            firstName: 'Ahmed',
            lastName: 'Hassan',
            email: 'ahmed@example.com',
          },
        }),
      });

      if (!res.ok) throw new Error('Failed to create checkout session');

      const data = await res.json();
      setCheckoutUrl(data.iframeUrl);

      // Alternative: redirect to hosted checkout
      // window.location.href = data.iframeUrl;
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Something went wrong');
    } finally {
      setLoading(false);
    }
  }

  return (
    <div style={{ maxWidth: 600, margin: '0 auto', padding: 20 }}>
      <h1>Checkout</h1>

      {!checkoutUrl ? (
        <button onClick={handleCheckout} disabled={loading}>
          {loading ? 'Processing...' : 'Pay 150 EGP'}
        </button>
      ) : (
        <iframe
          src={checkoutUrl}
          width="100%"
          height="600"
          style={{ border: 'none', borderRadius: 8 }}
          title="Paymob Payment"
        />
      )}

      {error && <p style={{ color: 'red' }}>{error}</p>}
    </div>
  );
}
```

### Next.js API Route (Backend)

```typescript
// app/api/checkout/route.ts
import { NextResponse } from 'next/server';

export async function POST(request: Request) {
  const body = await request.json();
  const amountCents = Math.round(body.amount * 100);

  // Create intention
  const response = await fetch(`${process.env.PAYMOB_BASE_URL}/v1/intention/`, {
    method: 'POST',
    headers: {
      'Authorization': `Token ${process.env.PAYMOB_SECRET_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      amount: amountCents,
      currency: 'EGP',
      payment_methods: [Number(process.env.PAYMOB_INTEGRATION_ID_CARD)],
      items: body.items.map((i: any) => ({
        name: i.name,
        amount: Math.round(i.price * 100),
        quantity: i.quantity,
      })),
      billing_data: {
        first_name: body.customer.firstName,
        last_name: body.customer.lastName,
        email: body.customer.email,
        phone_number: '+20000000000',
        apartment: 'NA', floor: 'NA', street: 'NA',
        building: 'NA', shipping_method: 'NA',
        postal_code: 'NA', city: 'NA', country: 'EG', state: 'NA',
      },
      notification_url: `${process.env.NEXT_PUBLIC_APP_URL}/api/paymob/webhook`,
      redirection_url: `${process.env.NEXT_PUBLIC_APP_URL}/payment/complete`,
    }),
  });

  if (!response.ok) {
    const error = await response.text();
    return NextResponse.json({ error }, { status: 500 });
  }

  const data = await response.json();

  return NextResponse.json({
    clientSecret: data.client_secret,
    iframeUrl: `https://accept.paymob.com/unifiedcheckout/?publicKey=${process.env.PAYMOB_PUBLIC_KEY}&clientSecret=${data.client_secret}`,
  });
}
```

## Vue.js Example

```vue
<template>
  <div>
    <button @click="startPayment" :disabled="loading" v-if="!iframeUrl">
      {{ loading ? 'Processing...' : 'Pay Now' }}
    </button>
    <iframe
      v-if="iframeUrl"
      :src="iframeUrl"
      width="100%"
      height="600"
      frameborder="0"
    />
  </div>
</template>

<script setup>
import { ref } from 'vue';

const loading = ref(false);
const iframeUrl = ref(null);

async function startPayment() {
  loading.value = true;
  try {
    const res = await fetch('/api/checkout', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ amount: 100, items: [{ name: 'Item', price: 100, quantity: 1 }] }),
    });
    const data = await res.json();
    iframeUrl.value = data.iframeUrl;
  } finally {
    loading.value = false;
  }
}
</script>
```

## Handling Redirect Callbacks

After payment, Paymob redirects the user to your `redirection_url` with query parameters:

```tsx
// app/payment/complete/page.tsx
'use client';

import { useSearchParams } from 'next/navigation';

export default function PaymentComplete() {
  const params = useSearchParams();
  const success = params.get('success') === 'true';
  const transactionId = params.get('id');

  return (
    <div>
      {success ? (
        <div>
          <h1>Payment Successful!</h1>
          <p>Transaction ID: {transactionId}</p>
        </div>
      ) : (
        <div>
          <h1>Payment Failed</h1>
          <p>Please try again or contact support.</p>
        </div>
      )}
    </div>
  );
}
```

## Security Reminders

- **Never put secret keys in frontend code** — only use the public key on the client side
- **Always validate webhooks on the backend** — don't rely on redirect query params alone for order fulfillment
- **The redirect callback is for UX only** — the webhook is the source of truth for payment status
