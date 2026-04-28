# Paymob Integration — PHP / Laravel

## Laravel Integration

### Installation & Config

```bash
composer require guzzlehttp/guzzle
```

Add to `.env`:
```env
PAYMOB_API_KEY=your_api_key
PAYMOB_SECRET_KEY=sk_live_xxxxxxx
PAYMOB_HMAC_SECRET=your_hmac_secret
PAYMOB_INTEGRATION_ID_CARD=123456
PAYMOB_INTEGRATION_ID_WALLET=789012
PAYMOB_IFRAME_ID=12345
PAYMOB_BASE_URL=https://accept.paymob.com
```

Create `config/paymob.php`:
```php
<?php
return [
    'api_key' => env('PAYMOB_API_KEY'),
    'secret_key' => env('PAYMOB_SECRET_KEY'),
    'hmac_secret' => env('PAYMOB_HMAC_SECRET'),
    'integration_id_card' => env('PAYMOB_INTEGRATION_ID_CARD'),
    'integration_id_wallet' => env('PAYMOB_INTEGRATION_ID_WALLET'),
    'iframe_id' => env('PAYMOB_IFRAME_ID'),
    'base_url' => env('PAYMOB_BASE_URL', 'https://accept.paymob.com'),
];
```

### Service Class

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\Http;

class PaymobService
{
    protected string $baseUrl;
    protected string $secretKey;
    protected string $apiKey;
    protected string $hmacSecret;

    public function __construct()
    {
        $this->baseUrl = config('paymob.base_url');
        $this->secretKey = config('paymob.secret_key');
        $this->apiKey = config('paymob.api_key');
        $this->hmacSecret = config('paymob.hmac_secret');
    }

    // ── Intention API (Recommended) ──

    public function createIntention(
        int $amountCents,
        string $currency,
        array $paymentMethods,
        array $billingData,
        array $items,
        string $notificationUrl,
        string $redirectionUrl
    ): array {
        $response = Http::withHeaders([
            'Authorization' => "Token {$this->secretKey}",
        ])->post("{$this->baseUrl}/v1/intention/", [
            'amount' => $amountCents,
            'currency' => $currency,
            'payment_methods' => $paymentMethods,
            'items' => $items,
            'billing_data' => $billingData,
            'notification_url' => $notificationUrl,
            'redirection_url' => $redirectionUrl,
        ]);

        $response->throw();
        return $response->json();
    }

    // ── Legacy 3-Step Flow ──

    public function authenticate(): string
    {
        $response = Http::post("{$this->baseUrl}/api/auth/tokens", [
            'api_key' => $this->apiKey,
        ]);
        $response->throw();
        return $response->json('token');
    }

    public function createOrder(string $authToken, int $amountCents, string $merchantOrderId): int
    {
        $response = Http::post("{$this->baseUrl}/api/ecommerce/orders", [
            'auth_token' => $authToken,
            'delivery_needed' => 'false',
            'amount_cents' => (string) $amountCents,
            'currency' => 'EGP',
            'merchant_order_id' => $merchantOrderId,
            'items' => [],
        ]);
        $response->throw();
        return $response->json('id');
    }

    public function getPaymentKey(
        string $authToken,
        int $orderId,
        int $amountCents,
        int $integrationId,
        array $billingData = []
    ): string {
        $defaultBilling = [
            'first_name' => 'NA', 'last_name' => 'NA',
            'email' => 'NA', 'phone_number' => 'NA',
            'apartment' => 'NA', 'floor' => 'NA', 'street' => 'NA',
            'building' => 'NA', 'shipping_method' => 'NA',
            'postal_code' => 'NA', 'city' => 'NA', 'country' => 'EG', 'state' => 'NA',
        ];

        $response = Http::post("{$this->baseUrl}/api/acceptance/payment_keys", [
            'auth_token' => $authToken,
            'amount_cents' => (string) $amountCents,
            'expiration' => 3600,
            'order_id' => $orderId,
            'billing_data' => array_merge($defaultBilling, $billingData),
            'currency' => 'EGP',
            'integration_id' => $integrationId,
        ]);
        $response->throw();
        return $response->json('token');
    }

    // ── HMAC Validation ──

    public function validateHMAC(array $transaction, string $receivedHMAC): bool
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
        $calculated = hash_hmac('sha512', $concatenated, $this->hmacSecret);

        return hash_equals($calculated, $receivedHMAC);
    }

    // ── Refund / Void ──

    public function refund(string $authToken, int $transactionId, int $amountCents): array
    {
        $response = Http::post("{$this->baseUrl}/api/acceptance/void_refund/refund", [
            'auth_token' => $authToken,
            'transaction_id' => (string) $transactionId,
            'amount_cents' => $amountCents,
        ]);
        $response->throw();
        return $response->json();
    }

    public function void(string $authToken, int $transactionId): array
    {
        $response = Http::post("{$this->baseUrl}/api/acceptance/void_refund/void", [
            'auth_token' => $authToken,
            'transaction_id' => (string) $transactionId,
        ]);
        $response->throw();
        return $response->json();
    }

    public function getIframeUrl(string $paymentKey): string
    {
        $iframeId = config('paymob.iframe_id');
        return "{$this->baseUrl}/api/acceptance/iframes/{$iframeId}?payment_token={$paymentKey}";
    }
}
```

### Controller

```php
<?php

namespace App\Http\Controllers;

use App\Services\PaymobService;
use Illuminate\Http\Request;
use Illuminate\Support\Str;

class PaymobController extends Controller
{
    public function __construct(protected PaymobService $paymob) {}

    public function checkout(Request $request)
    {
        $validated = $request->validate([
            'amount' => 'required|numeric|min:1',
            'first_name' => 'nullable|string',
            'last_name' => 'nullable|string',
            'email' => 'nullable|email',
            'phone' => 'nullable|string',
        ]);

        $amountCents = (int) round($validated['amount'] * 100);

        $result = $this->paymob->createIntention(
            amountCents: $amountCents,
            currency: 'EGP',
            paymentMethods: [(int) config('paymob.integration_id_card')],
            billingData: [
                'first_name' => $validated['first_name'] ?? 'NA',
                'last_name' => $validated['last_name'] ?? 'NA',
                'email' => $validated['email'] ?? 'NA',
                'phone_number' => $validated['phone'] ?? 'NA',
                'apartment' => 'NA', 'floor' => 'NA', 'street' => 'NA',
                'building' => 'NA', 'shipping_method' => 'NA',
                'postal_code' => 'NA', 'city' => 'NA', 'country' => 'EG', 'state' => 'NA',
            ],
            items: [['name' => 'Order', 'amount' => $amountCents, 'quantity' => 1]],
            notificationUrl: route('paymob.webhook'),
            redirectionUrl: route('payment.complete'),
        );

        return response()->json([
            'client_secret' => $result['client_secret'],
        ]);
    }

    public function webhook(Request $request)
    {
        $data = $request->json()->all();
        $transaction = $data['obj'] ?? $data;
        $receivedHMAC = $request->query('hmac', '');

        if (!$this->paymob->validateHMAC($transaction, $receivedHMAC)) {
            return response()->json(['error' => 'Invalid HMAC'], 401);
        }

        if ($transaction['success'] === true) {
            // Update order in database
            // Order::where('merchant_id', $transaction['order']['merchant_order_id'])
            //     ->update(['status' => 'paid', 'transaction_id' => $transaction['id']]);
        }

        return response()->json(['received' => true]);
    }
}
```

### Routes

```php
// routes/api.php
Route::post('/checkout', [PaymobController::class, 'checkout']);
Route::post('/paymob/webhook', [PaymobController::class, 'webhook'])->name('paymob.webhook');
```

## Community Packages

- `baklysystems/laravel-paymob` — Laravel package for legacy API
- `samir-hussein/paymob` — PHP package with both native and Laravel support
- `paymob/php-library` — Official PHP library (check Packagist for availability)
