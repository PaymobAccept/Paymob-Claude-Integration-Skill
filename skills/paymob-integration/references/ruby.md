# Paymob Integration — Ruby / Rails

## Configuration

```ruby
# config/initializers/paymob.rb
module Paymob
  mattr_accessor :api_key, :secret_key, :hmac_secret, :base_url,
                 :integration_id_card, :integration_id_wallet, :iframe_id

  self.api_key = ENV['PAYMOB_API_KEY']
  self.secret_key = ENV['PAYMOB_SECRET_KEY']
  self.hmac_secret = ENV['PAYMOB_HMAC_SECRET']
  self.base_url = ENV.fetch('PAYMOB_BASE_URL', 'https://accept.paymob.com')
  self.integration_id_card = ENV['PAYMOB_INTEGRATION_ID_CARD']&.to_i
  self.integration_id_wallet = ENV['PAYMOB_INTEGRATION_ID_WALLET']&.to_i
  self.iframe_id = ENV['PAYMOB_IFRAME_ID']
end
```

## Service Class

```ruby
# app/services/paymob_service.rb
require 'net/http'
require 'json'
require 'openssl'

class PaymobService
  # Intention API (recommended)
  def create_intention(amount_cents:, currency: 'EGP', payment_methods:,
                       billing_data:, items:, notification_url:, redirection_url:)
    post('/v1/intention/', {
      amount: amount_cents,
      currency: currency,
      payment_methods: payment_methods,
      items: items,
      billing_data: billing_data,
      notification_url: notification_url,
      redirection_url: redirection_url,
    }, auth_header: { 'Authorization' => "Token #{Paymob.secret_key}" })
  end

  # Legacy: Step 1
  def authenticate
    response = post('/api/auth/tokens', { api_key: Paymob.api_key })
    response['token']
  end

  # Legacy: Step 2
  def create_order(auth_token:, amount_cents:, merchant_order_id:, currency: 'EGP')
    response = post('/api/ecommerce/orders', {
      auth_token: auth_token,
      delivery_needed: 'false',
      amount_cents: amount_cents.to_s,
      currency: currency,
      merchant_order_id: merchant_order_id,
      items: [],
    })
    response['id']
  end

  # Legacy: Step 3
  def get_payment_key(auth_token:, order_id:, amount_cents:, integration_id:,
                      billing_data: {}, currency: 'EGP')
    defaults = {
      first_name: 'NA', last_name: 'NA', email: 'NA', phone_number: 'NA',
      apartment: 'NA', floor: 'NA', street: 'NA', building: 'NA',
      shipping_method: 'NA', postal_code: 'NA', city: 'NA', country: 'EG', state: 'NA',
    }

    response = post('/api/acceptance/payment_keys', {
      auth_token: auth_token,
      amount_cents: amount_cents.to_s,
      expiration: 3600,
      order_id: order_id,
      billing_data: defaults.merge(billing_data),
      currency: currency,
      integration_id: integration_id,
    })
    response['token']
  end

  # HMAC Validation
  def valid_hmac?(transaction, received_hmac)
    fields = [
      transaction['amount_cents'], transaction['created_at'], transaction['currency'],
      transaction['error_occured'], transaction['has_parent_transaction'],
      transaction['id'], transaction['integration_id'], transaction['is_3d_secure'],
      transaction['is_auth'], transaction['is_capture'], transaction['is_refunded'],
      transaction['is_standalone_payment'], transaction['is_voided'],
      transaction.dig('order', 'id'), transaction['owner'], transaction['pending'],
      transaction.dig('source_data', 'pan'), transaction.dig('source_data', 'sub_type'),
      transaction.dig('source_data', 'type'), transaction['success'],
    ]

    concatenated = fields.map(&:to_s).join
    calculated = OpenSSL::HMAC.hexdigest('SHA512', Paymob.hmac_secret, concatenated)

    ActiveSupport::SecurityUtils.secure_compare(calculated, received_hmac)
  end

  private

  def post(path, body, auth_header: {})
    uri = URI("#{Paymob.base_url}#{path}")
    http = Net::HTTP.new(uri.host, uri.port)
    http.use_ssl = true

    request = Net::HTTP::Post.new(uri)
    request['Content-Type'] = 'application/json'
    auth_header.each { |k, v| request[k] = v }
    request.body = body.to_json

    response = http.request(request)
    JSON.parse(response.body)
  end
end
```

## Controller

```ruby
# app/controllers/paymob_controller.rb
class PaymobController < ApplicationController
  skip_before_action :verify_authenticity_token, only: [:webhook]

  def checkout
    service = PaymobService.new
    amount_cents = (params[:amount].to_f * 100).to_i

    result = service.create_intention(
      amount_cents: amount_cents,
      currency: 'EGP',
      payment_methods: [Paymob.integration_id_card],
      billing_data: default_billing.merge(params.permit(:first_name, :last_name, :email).to_h),
      items: [{ name: 'Order', amount: amount_cents, quantity: 1 }],
      notification_url: paymob_webhook_url,
      redirection_url: payment_complete_url,
    )

    render json: { client_secret: result['client_secret'] }
  end

  def webhook
    transaction = params[:obj] || params
    received_hmac = params[:hmac].to_s

    service = PaymobService.new
    unless service.valid_hmac?(transaction.to_unsafe_h, received_hmac)
      return render json: { error: 'Invalid HMAC' }, status: :unauthorized
    end

    if transaction[:success] == true
      # Order.find_by(merchant_id: transaction.dig(:order, :merchant_order_id))
      #   &.update!(status: :paid, transaction_id: transaction[:id])
    end

    render json: { received: true }
  end

  private

  def default_billing
    { first_name: 'NA', last_name: 'NA', email: 'NA', phone_number: 'NA',
      apartment: 'NA', floor: 'NA', street: 'NA', building: 'NA',
      shipping_method: 'NA', postal_code: 'NA', city: 'NA', country: 'EG', state: 'NA' }
  end
end
```
