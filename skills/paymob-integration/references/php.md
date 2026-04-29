# Paymob Integration -- PHP

## Environment Variables (.env)

```env
PAYMOB_SECRET_KEY=sk_live_xxxxxxx
PAYMOB_PUBLIC_KEY=pk_live_xxxxxxx
PAYMOB_HMAC_SECRET=your_hmac_secret
PAYMOB_INTEGRATION_ID_CARD=123456
PAYMOB_BASE_URL=https://accept.paymob.com
```

## Installation

```bash
composer require guzzlehttp/guzzle vlucas/phpdotenv
```

## PaymobClient Class

```php
<?php
use GuzzleHttp\Client;

class PaymobClient {
    private string \;
    private string \;
    private string \;
    private string \;
    private Client \;

    public function __construct() {
        \->secretKey  = getenv('PAYMOB_SECRET_KEY');
        \->publicKey  = getenv('PAYMOB_PUBLIC_KEY');
        \->hmacSecret = getenv('PAYMOB_HMAC_SECRET');
        \->baseUrl    = getenv('PAYMOB_BASE_URL') ?: 'https://accept.paymob.com';
        \->http = new Client([
            'base_uri' => \->baseUrl,
            'headers'  => [
                'Authorization' => "Token {\->secretKey}",
                'Content-Type'  => 'application/json',
            ],
        ]);
    }

    public function createIntention(array \): array {
        \ = \->http->post('/v1/intention/', ['json' => \]);
        \ = json_decode(\->getBody(), true);
        return ['id' => \['id'], 'client_secret' => \['client_secret']];
    }

    public function checkoutUrl(string \): string {
        return "{\->baseUrl}/unifiedcheckout/?publicKey={\->publicKey}&clientSecret={\}";
    }

    public function refund(int \, int \): array {
        \ = \->http->post('/api/acceptance/void_refund/refund',
            ['json' => ['transaction_id' => \, 'amount_cents' => \]]);
        return json_decode(\->getBody(), true);
    }

    public function void(int \): array {
        \ = \->http->post('/api/acceptance/void_refund/void',
            ['json' => ['transaction_id' => \]]);
        return json_decode(\->getBody(), true);
    }

    public function validateTransactionHmac(array \, string \): bool {
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
        \ = hash_hmac('sha512', \, \->hmacSecret);
        return hash_equals(\, \);
    }
}
```

## Laravel Controller

```php
<?php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class PaymobController extends Controller {
    protected PaymobClient \;

    public function __construct() {
        \->paymob = new PaymobClient();
    }

    public function checkout(Request \): JsonResponse {
        \->validate(['amount' => 'required|numeric', 'email' => 'required|email']);
        \ = (int) round(\->amount * 100);
        \ = 'NA';
        \ = \->paymob->createIntention([
            'amount'           => \,
            'currency'         => 'EGP',
            'payment_methods'  => [(int) getenv('PAYMOB_INTEGRATION_ID_CARD')],
            'billing_data'     => [
                'first_name' => \->get('first_name', \),
                'last_name'  => \->get('last_name', \),
                'email'      => \->email,
                'phone_number' => '+20000000000',
                'apartment' => \, 'floor' => \, 'street' => \,
                'building'  => \, 'shipping_method' => \,
                'postal_code' => \, 'city' => \,
                'country'   => 'EG', 'state' => \,
            ],
            'customer'         => ['first_name' => \, 'last_name' => \, 'email' => \->email],
            'items'            => [],
            'notification_url' => url('/api/paymob/webhook'),
            'redirection_url'  => url('/payment/complete'),
        ]);
        return response()->json([
            'client_secret' => \['client_secret'],
            'checkout_url'  => \->paymob->checkoutUrl(\['client_secret']),
        ]);
    }

    public function webhook(Request \): JsonResponse {
        \ = \->all();
        \  = \['obj'] ?? [];
        \ = \->query('hmac', '');
        if (!\->paymob->validateTransactionHmac(\, \)) {
            return response()->json(['error' => 'Invalid HMAC'], 401);
        }
        if (\['success'] ?? false) {
            \Log::info("Payment {\['id']} succeeded");
        }
        return response()->json(['received' => true]);
    }
}
```

## Symfony Controller

```php
<?php
namespace App\Controller;

use Symfony\Component\HttpFoundation\{Request, JsonResponse};
use Symfony\Component\Routing\Annotation\Route;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;

class PaymobController extends AbstractController {
    #[Route('/api/checkout', methods: ['POST'])]
    public function checkout(Request \): JsonResponse {
        \ = json_decode(\->getContent(), true);
        \ = new \PaymobClient();
        \ = 'NA';
        \ = \->createIntention([
            'amount' => (int)round((float)\['amount'] * 100),
            'currency' => 'EGP',
            'payment_methods' => [(int)getenv('PAYMOB_INTEGRATION_ID_CARD')],
            'billing_data' => array_fill_keys(['first_name','last_name','email','phone_number',
                'apartment','floor','street','building','shipping_method','postal_code','city','country','state'],
                'NA') + ['country' => 'EG', 'email' => \['email']],
            'customer' => ['first_name' => \, 'last_name' => \, 'email' => \['email']],
            'items' => [],
            'notification_url' => \->generateUrl('paymob_webhook', [], 0),
            'redirection_url'  => \->generateUrl('payment_complete', [], 0),
        ]);
        return \->json(['checkout_url' => \->checkoutUrl(\['client_secret'])]);
    }
}
```
