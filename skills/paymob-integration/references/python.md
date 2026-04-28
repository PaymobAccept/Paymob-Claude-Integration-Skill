# Paymob Integration — Python / Django / Flask

## Environment Variables

```env
PAYMOB_API_KEY=your_api_key
PAYMOB_SECRET_KEY=sk_live_xxxxxxx
PAYMOB_HMAC_SECRET=your_hmac_secret
PAYMOB_INTEGRATION_ID_CARD=123456
PAYMOB_INTEGRATION_ID_WALLET=789012
PAYMOB_BASE_URL=https://accept.paymob.com
PAYMOB_IFRAME_ID=12345
```

## Paymob Service Class (Framework-Agnostic)

```python
import hashlib
import hmac
import os
import requests
from dataclasses import dataclass
from typing import Optional


@dataclass
class PaymobConfig:
    api_key: str
    secret_key: str
    hmac_secret: str
    base_url: str = "https://accept.paymob.com"
    integration_id_card: Optional[int] = None
    integration_id_wallet: Optional[int] = None
    iframe_id: Optional[str] = None

    @classmethod
    def from_env(cls):
        return cls(
            api_key=os.environ["PAYMOB_API_KEY"],
            secret_key=os.environ["PAYMOB_SECRET_KEY"],
            hmac_secret=os.environ["PAYMOB_HMAC_SECRET"],
            base_url=os.environ.get("PAYMOB_BASE_URL", "https://accept.paymob.com"),
            integration_id_card=int(os.environ.get("PAYMOB_INTEGRATION_ID_CARD", 0)) or None,
            integration_id_wallet=int(os.environ.get("PAYMOB_INTEGRATION_ID_WALLET", 0)) or None,
            iframe_id=os.environ.get("PAYMOB_IFRAME_ID"),
        )


class PaymobClient:
    def __init__(self, config: PaymobConfig):
        self.config = config
        self.session = requests.Session()

    # ── Intention API (Recommended) ──

    def create_intention(self, amount_cents: int, currency: str,
                         payment_methods: list[int], billing_data: dict,
                         items: list[dict], notification_url: str,
                         redirection_url: str) -> dict:
        """Create a payment intention using the modern API."""
        response = self.session.post(
            f"{self.config.base_url}/v1/intention/",
            json={
                "amount": amount_cents,
                "currency": currency,
                "payment_methods": payment_methods,
                "items": items,
                "billing_data": billing_data,
                "notification_url": notification_url,
                "redirection_url": redirection_url,
            },
            headers={
                "Authorization": f"Token {self.config.secret_key}",
                "Content-Type": "application/json",
            },
        )
        response.raise_for_status()
        return response.json()

    # ── Legacy 3-Step Flow ──

    def authenticate(self) -> str:
        """Step 1: Get auth token."""
        resp = self.session.post(
            f"{self.config.base_url}/api/auth/tokens",
            json={"api_key": self.config.api_key},
        )
        resp.raise_for_status()
        return resp.json()["token"]

    def create_order(self, auth_token: str, amount_cents: int,
                     merchant_order_id: str, currency: str = "EGP") -> int:
        """Step 2: Register an order."""
        resp = self.session.post(
            f"{self.config.base_url}/api/ecommerce/orders",
            json={
                "auth_token": auth_token,
                "delivery_needed": "false",
                "amount_cents": str(amount_cents),
                "currency": currency,
                "merchant_order_id": merchant_order_id,
                "items": [],
            },
        )
        resp.raise_for_status()
        return resp.json()["id"]

    def get_payment_key(self, auth_token: str, order_id: int,
                        amount_cents: int, integration_id: int,
                        billing_data: dict, currency: str = "EGP") -> str:
        """Step 3: Generate payment key."""
        default_billing = {
            "first_name": "NA", "last_name": "NA",
            "email": "NA", "phone_number": "NA",
            "apartment": "NA", "floor": "NA", "street": "NA",
            "building": "NA", "shipping_method": "NA",
            "postal_code": "NA", "city": "NA", "country": "EG", "state": "NA",
        }
        default_billing.update(billing_data)

        resp = self.session.post(
            f"{self.config.base_url}/api/acceptance/payment_keys",
            json={
                "auth_token": auth_token,
                "amount_cents": str(amount_cents),
                "expiration": 3600,
                "order_id": order_id,
                "billing_data": default_billing,
                "currency": currency,
                "integration_id": integration_id,
            },
        )
        resp.raise_for_status()
        return resp.json()["token"]

    # ── HMAC Validation ──

    def validate_hmac(self, transaction: dict, received_hmac: str) -> bool:
        fields = [
            str(transaction["amount_cents"]),
            transaction["created_at"],
            transaction["currency"],
            str(transaction["error_occured"]),
            str(transaction["has_parent_transaction"]),
            str(transaction["id"]),
            str(transaction["integration_id"]),
            str(transaction["is_3d_secure"]),
            str(transaction["is_auth"]),
            str(transaction["is_capture"]),
            str(transaction["is_refunded"]),
            str(transaction["is_standalone_payment"]),
            str(transaction["is_voided"]),
            str(transaction["order"]["id"]),
            str(transaction["owner"]),
            str(transaction["pending"]),
            transaction["source_data"]["pan"],
            transaction["source_data"]["sub_type"],
            transaction["source_data"]["type"],
            str(transaction["success"]),
        ]

        concatenated = "".join(fields)
        calculated = hmac.new(
            self.config.hmac_secret.encode(),
            concatenated.encode(),
            hashlib.sha512,
        ).hexdigest()

        return hmac.compare_digest(calculated, received_hmac)

    # ── Post-Payment ──

    def refund(self, auth_token: str, transaction_id: int, amount_cents: int) -> dict:
        resp = self.session.post(
            f"{self.config.base_url}/api/acceptance/void_refund/refund",
            json={
                "auth_token": auth_token,
                "transaction_id": str(transaction_id),
                "amount_cents": amount_cents,
            },
        )
        resp.raise_for_status()
        return resp.json()


## Django View Example

```python
# views.py
import json
import uuid
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
from django.views.decorators.http import require_POST

paymob = PaymobClient(PaymobConfig.from_env())


@require_POST
def create_checkout(request):
    body = json.loads(request.body)
    amount_cents = int(body["amount"] * 100)

    result = paymob.create_intention(
        amount_cents=amount_cents,
        currency="EGP",
        payment_methods=[paymob.config.integration_id_card],
        billing_data={
            "first_name": body.get("first_name", "NA"),
            "last_name": body.get("last_name", "NA"),
            "email": body.get("email", "NA"),
            "phone_number": body.get("phone", "NA"),
            "apartment": "NA", "floor": "NA", "street": "NA",
            "building": "NA", "shipping_method": "NA",
            "postal_code": "NA", "city": "NA", "country": "EG", "state": "NA",
        },
        items=[{"name": "Order", "amount": amount_cents, "quantity": 1}],
        notification_url=f"{request.build_absolute_uri('/api/paymob/webhook/')}",
        redirection_url=body.get("redirect_url", "/payment/complete"),
    )

    return JsonResponse({
        "client_secret": result["client_secret"],
        "intention_id": result["id"],
    })


@csrf_exempt
@require_POST
def paymob_webhook(request):
    data = json.loads(request.body)
    transaction = data.get("obj", data)
    received_hmac = request.GET.get("hmac", "")

    if not paymob.validate_hmac(transaction, received_hmac):
        return JsonResponse({"error": "Invalid HMAC"}, status=401)

    if transaction.get("success"):
        # Update order status in your database
        order_id = transaction["order"]["merchant_order_id"]
        # Order.objects.filter(merchant_id=order_id).update(status="paid")

    return JsonResponse({"received": True})
```

## Flask Example

```python
from flask import Flask, request, jsonify

app = Flask(__name__)
paymob = PaymobClient(PaymobConfig.from_env())

@app.route("/api/checkout", methods=["POST"])
def checkout():
    body = request.get_json()
    amount_cents = int(body["amount"] * 100)

    result = paymob.create_intention(
        amount_cents=amount_cents,
        currency="EGP",
        payment_methods=[paymob.config.integration_id_card],
        billing_data={...},  # same as Django example
        items=[{"name": "Order", "amount": amount_cents, "quantity": 1}],
        notification_url=f"{request.host_url}api/paymob/webhook",
        redirection_url="/payment/complete",
    )
    return jsonify(client_secret=result["client_secret"])

@app.route("/api/paymob/webhook", methods=["POST"])
def webhook():
    transaction = request.get_json().get("obj", request.get_json())
    received_hmac = request.args.get("hmac", "")

    if not paymob.validate_hmac(transaction, received_hmac):
        return jsonify(error="Invalid HMAC"), 401

    if transaction.get("success"):
        # process successful payment
        pass

    return jsonify(received=True)
```

## FastAPI Example

```python
from fastapi import FastAPI, Request, Query, HTTPException

app = FastAPI()
paymob = PaymobClient(PaymobConfig.from_env())

@app.post("/api/paymob/webhook")
async def webhook(request: Request, hmac_value: str = Query(alias="hmac")):
    body = await request.json()
    transaction = body.get("obj", body)

    if not paymob.validate_hmac(transaction, hmac_value):
        raise HTTPException(status_code=401, detail="Invalid HMAC")

    if transaction.get("success"):
        # process payment
        pass

    return {"received": True}
```
```
