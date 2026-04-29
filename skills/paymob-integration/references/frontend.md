# Paymob Integration -- Frontend (React / Next.js / Vue)

## Checkout Flow (All Frameworks)

1. POST to your backend /api/checkout with cart data
2. Receive clientSecret from backend
3. Redirect user to Unified Checkout URL **or** embed Pixel SDK

## Unified Checkout (Redirect)

```js
// Get clientSecret from backend, then redirect:
const PUBLIC_KEY = 'pk_live_xxxxxxx';
const BASE_URL   = 'https://accept.paymob.com'; // your region's base URL

function redirectToCheckout(clientSecret) {
  window.location.href = ${BASE_URL}/unifiedcheckout/?publicKey=&clientSecret=;
}
```

## Pixel SDK (Embedded)

Include the Pixel SDK script:

```html
<script src="https://accept.paymob.com/v1/pay/sdk.js"></script>
```

Then initialize:

```js
Paymob.init({
  publicKey: 'pk_live_xxxxxxx',
  clientSecret: clientSecret, // from your backend
  onPaymentComplete: ({ id, success, amount }) => {
    if (success) redirectToSuccess(id);
    else showError('Payment failed');
  },
});
Paymob.pay({ selector: '#pay-button' });
```

## React -- Checkout Component

```jsx
import { useState } from 'react';

export default function CheckoutButton({ amount, email }) {
  const [loading, setLoading] = useState(false);

  const handleCheckout = async () => {
    setLoading(true);
    try {
      const res = await fetch('/api/checkout', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ amount, email }),
      });
      const { checkout_url } = await res.json();
      window.location.href = checkout_url;
    } catch (err) {
      console.error('Checkout error:', err);
    } finally {
      setLoading(false);
    }
  };

  return (
    <button onClick={handleCheckout} disabled={loading}>
      {loading ? 'Processing...' : Pay EGP }
    </button>
  );
}
```

## React -- Payment Complete Page

```jsx
import { useEffect, useState } from 'react';

export default function PaymentComplete() {
  const [status, setStatus] = useState('loading');

  useEffect(() => {
    // Paymob appends result params to redirect URL
    const params = new URLSearchParams(window.location.search);
    const success = params.get('success');
    setStatus(success === 'true' ? 'success' : 'failed');
  }, []);

  return (
    <div>
      {status === 'success' && <h2>Payment Successful!</h2>}
      {status === 'failed' && <h2>Payment Failed. Please try again.</h2>}
      {status === 'loading' && <p>Checking payment status...</p>}
    </div>
  );
}
```

## Next.js -- API Route (App Router)

```ts
// app/api/checkout/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function POST(req: NextRequest) {
  const { amount, email } = await req.json();
  const amountCents = Math.round(amount * 100);
  const NA = 'NA';

  const res = await fetch(${process.env.PAYMOB_BASE_URL}/v1/intention/, {
    method: 'POST',
    headers: {
      'Authorization': Token ,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      amount: amountCents, currency: 'EGP',
      payment_methods: [Number(process.env.PAYMOB_INTEGRATION_ID_CARD)],
      billing_data: { first_name: NA, last_name: NA, email, phone_number: '+20000000000',
                      apartment: NA, floor: NA, street: NA, building: NA,
                      shipping_method: NA, postal_code: NA, city: NA, country: 'EG', state: NA },
      customer: { first_name: NA, last_name: NA, email },
      items: [],
      notification_url: ${process.env.APP_URL}/api/paymob/webhook,
      redirection_url:  ${process.env.APP_URL}/payment/complete,
    }),
  });
  const data = await res.json();
  const checkoutUrl = ${process.env.PAYMOB_BASE_URL}/unifiedcheckout/?publicKey=&clientSecret=;
  return NextResponse.json({ checkout_url: checkoutUrl });
}
```

## Vue 3 -- Checkout Composable

```ts
// composables/useCheckout.ts
import { ref } from 'vue';

export function useCheckout() {
  const loading = ref(false);
  const error   = ref('');

  const checkout = async (amount: number, email: string) => {
    loading.value = true;
    error.value = '';
    try {
      const res = await fetch('/api/checkout', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ amount, email }),
      });
      const { checkout_url } = await res.json();
      window.location.href = checkout_url;
    } catch (e) {
      error.value = 'Checkout failed';
    } finally {
      loading.value = false;
    }
  };

  return { checkout, loading, error };
}
```

```vue
<!-- PaymentButton.vue -->
<template>
  <button @click="checkout(amount, email)" :disabled="loading">
    {{ loading ? 'Processing...' : 'Pay Now' }}
  </button>
  <p v-if="error" class="error">{{ error }}</p>
</template>

<script setup lang="ts">
import { useCheckout } from '@/composables/useCheckout';
const { checkout, loading, error } = useCheckout();
defineProps<{ amount: number; email: string }>();
</script>
```

## Notes

- Never expose PAYMOB_SECRET_KEY on the frontend
- PUBLIC_KEY (pk_live_*) is safe to use in frontend code
- Always validate HMAC on the server-side webhook, never client-side
- The 
otification_url must be publicly accessible (use ngrok for local dev)
