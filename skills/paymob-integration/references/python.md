# Paymob Integration -- Python

## Environment Variables

```env
PAYMOB_SECRET_KEY=sk_live_xxxxxxx
PAYMOB_PUBLIC_KEY=pk_live_xxxxxxx
PAYMOB_HMAC_SECRET=your_hmac_secret
PAYMOB_INTEGRATION_ID_CARD=123456
PAYMOB_BASE_URL=https://accept.paymob.com
```

## Installation

```bash
pip install requests python-dotenv
```

## PaymobClient Class

```python
import os, hashlib, hmac as hmac_mod, requests
from typing import List, Dict, Any, Optional

class PaymobClient:
    def __init__(self):
        self.secret_key  = os.environ['PAYMOB_SECRET_KEY']
        self.public_key  = os.environ['PAYMOB_PUBLIC_KEY']
        self.hmac_secret = os.environ['PAYMOB_HMAC_SECRET']
        self.base_url    = os.environ.get('PAYMOB_BASE_URL', 'https://accept.paymob.com')
        self.session = requests.Session()
        self.session.headers.update({
            'Authorization': f'Token {self.secret_key}',
            'Content-Type': 'application/json',
        })

    def create_intention(self, amount: int, currency: str, payment_methods: List[int],
                         billing_data: dict, customer: dict, items: list,
                         notification_url: str, redirection_url: str, **extra) -> dict:
        payload = dict(amount=amount, currency=currency, payment_methods=payment_methods,
                       billing_data=billing_data, customer=customer, items=items,
                       notification_url=notification_url, redirection_url=redirection_url, **extra)
        r = self.session.post(f'{self.base_url}/v1/intention/', json=payload)
        r.raise_for_status()
        d = r.json()
        return {'id': d['id'], 'client_secret': d['client_secret']}

    def checkout_url(self, client_secret: str) -> str:
        return f'{self.base_url}/unifiedcheckout/?publicKey={self.public_key}&clientSecret={client_secret}'

    def refund(self, transaction_id: int, amount_cents: int) -> dict:
        r = self.session.post(f'{self.base_url}/api/acceptance/void_refund/refund',
                              json={'transaction_id': transaction_id, 'amount_cents': amount_cents})
        r.raise_for_status(); return r.json()

    def void(self, transaction_id: int) -> dict:
        r = self.session.post(f'{self.base_url}/api/acceptance/void_refund/void',
                              json={'transaction_id': transaction_id})
        r.raise_for_status(); return r.json()

    def capture(self, transaction_id: int, amount_cents: int) -> dict:
        r = self.session.post(f'{self.base_url}/api/acceptance/capture',
                              json={'transaction_id': transaction_id, 'amount_cents': amount_cents})
        r.raise_for_status(); return r.json()

    def validate_transaction_hmac(self, obj: dict, received_hmac: str) -> bool:
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
        computed = hmac_mod.new(self.hmac_secret.encode(), s.encode(), hashlib.sha512).hexdigest()
        return hmac_mod.compare_digest(computed, received_hmac)
```

## Django View

```python
import json
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt

paymob = PaymobClient()

def create_checkout(request):
    data = json.loads(request.body)
    amount_cents = round(float(data['amount']) * 100)
    NA = 'NA'
    intention = paymob.create_intention(
        amount=amount_cents, currency='EGP',
        payment_methods=[int(os.environ['PAYMOB_INTEGRATION_ID_CARD'])],
        billing_data={'first_name': data.get('first_name', NA), 'last_name': data.get('last_name', NA),
                      'email': data.get('email', NA), 'phone_number': '+20000000000',
                      'apartment': NA, 'floor': NA, 'street': NA, 'building': NA,
                      'shipping_method': NA, 'postal_code': NA, 'city': NA, 'country': 'EG', 'state': NA},
        customer={'first_name': data.get('first_name', NA), 'last_name': data.get('last_name', NA), 'email': data.get('email', NA)},
        items=[], notification_url='https://yoursite.com/api/paymob/webhook/',
        redirection_url='https://yoursite.com/payment/complete/',
    )
    return JsonResponse({'client_secret': intention['client_secret'],
                         'checkout_url': paymob.checkout_url(intention['client_secret'])})

@csrf_exempt
def paymob_webhook(request):
    body = json.loads(request.body)
    obj = body.get('obj', {})
    received_hmac = request.GET.get('hmac', '')
    if not paymob.validate_transaction_hmac(obj, received_hmac):
        return JsonResponse({'error': 'Invalid HMAC'}, status=401)
    if obj.get('success'):
        print(f"Payment {obj['id']} succeeded")
    return JsonResponse({'received': True})
```

## Flask View

```python
from flask import Flask, request, jsonify
app = Flask(__name__)
paymob = PaymobClient()

@app.route('/api/checkout', methods=['POST'])
def create_checkout():
    data = request.get_json()
    amount_cents = round(float(data['amount']) * 100)
    NA = 'NA'
    intention = paymob.create_intention(
        amount=amount_cents, currency='EGP',
        payment_methods=[int(os.environ['PAYMOB_INTEGRATION_ID_CARD'])],
        billing_data={'first_name': NA, 'last_name': NA, 'email': data.get('email', NA),
                      'phone_number': '+20000000000', 'apartment': NA, 'floor': NA,
                      'street': NA, 'building': NA, 'shipping_method': NA,
                      'postal_code': NA, 'city': NA, 'country': 'EG', 'state': NA},
        customer={'first_name': NA, 'last_name': NA, 'email': data.get('email', NA)},
        items=[], notification_url='https://yoursite.com/webhook',
        redirection_url='https://yoursite.com/complete',
    )
    return jsonify({'checkout_url': paymob.checkout_url(intention['client_secret'])})

@app.route('/api/paymob/webhook', methods=['POST'])
def webhook():
    body = request.get_json()
    obj = body.get('obj', {})
    received_hmac = request.args.get('hmac', '')
    if not paymob.validate_transaction_hmac(obj, received_hmac):
        return jsonify({'error': 'Invalid HMAC'}), 401
    if obj.get('success'):
        print(f"Payment {obj['id']} succeeded")
    return jsonify({'received': True})
```

## FastAPI View

```python
from fastapi import FastAPI, Request, HTTPException
from pydantic import BaseModel

app = FastAPI()
paymob = PaymobClient()

class CheckoutRequest(BaseModel):
    amount: float
    email: str
    first_name: str = 'NA'
    last_name: str = 'NA'

@app.post('/api/checkout')
async def create_checkout(req: CheckoutRequest):
    amount_cents = round(req.amount * 100)
    NA = 'NA'
    intention = paymob.create_intention(
        amount=amount_cents, currency='EGP',
        payment_methods=[int(os.environ['PAYMOB_INTEGRATION_ID_CARD'])],
        billing_data={'first_name': req.first_name, 'last_name': req.last_name,
                      'email': req.email, 'phone_number': '+20000000000',
                      'apartment': NA, 'floor': NA, 'street': NA, 'building': NA,
                      'shipping_method': NA, 'postal_code': NA, 'city': NA,
                      'country': 'EG', 'state': NA},
        customer={'first_name': req.first_name, 'last_name': req.last_name, 'email': req.email},
        items=[], notification_url='https://yoursite.com/webhook',
        redirection_url='https://yoursite.com/complete',
    )
    return {'checkout_url': paymob.checkout_url(intention['client_secret'])}

@app.post('/api/paymob/webhook')
async def webhook(request: Request):
    body = await request.json()
    obj = body.get('obj', {})
    received_hmac = request.query_params.get('hmac', '')
    if not paymob.validate_transaction_hmac(obj, received_hmac):
        raise HTTPException(status_code=401, detail='Invalid HMAC')
    if obj.get('success'):
        print(f"Payment {obj['id']} succeeded")
    return {'received': True}
```
