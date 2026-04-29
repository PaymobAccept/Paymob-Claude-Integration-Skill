# Paymob Integration -- Ruby / Ruby on Rails

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
gem install faraday
# or in Gemfile:
gem 'faraday'
```

## PaymobClient Class

```ruby
require 'faraday'
require 'json'
require 'openssl'

class PaymobClient
  def initialize
    @secret_key  = ENV.fetch('PAYMOB_SECRET_KEY')
    @public_key  = ENV.fetch('PAYMOB_PUBLIC_KEY')
    @hmac_secret = ENV.fetch('PAYMOB_HMAC_SECRET')
    @base_url    = ENV.fetch('PAYMOB_BASE_URL', 'https://accept.paymob.com')
    @conn = Faraday.new(url: @base_url) do |f|
      f.request :json
      f.response :json
      f.headers['Authorization'] = "Token #{@secret_key}"
    end
  end

  def create_intention(payload)
    response = @conn.post('/v1/intention/', payload)
    raise "Paymob error: #{response.body}" unless response.success?
    { id: response.body['id'], client_secret: response.body['client_secret'] }
  end

  def checkout_url(client_secret)
    "#{@base_url}/unifiedcheckout/?publicKey=#{@public_key}&clientSecret=#{client_secret}"
  end

  def refund(transaction_id, amount_cents)
    r = @conn.post('/api/acceptance/void_refund/refund',
                   { transaction_id: transaction_id, amount_cents: amount_cents })
    r.body
  end

  def void(transaction_id)
    r = @conn.post('/api/acceptance/void_refund/void',
                   { transaction_id: transaction_id })
    r.body
  end

  def validate_transaction_hmac(obj, received)
    fields = [
      obj['amount_cents'], obj['created_at'], obj['currency'],
      obj['error_occured'], obj['has_parent_transaction'],
      obj['id'], obj['integration_id'], obj['is_3d_secure'],
      obj['is_auth'], obj['is_capture'], obj['is_refunded'],
      obj['is_standalone_payment'], obj['is_voided'],
      obj.dig('order', 'id'), obj['owner'], obj['pending'],
      obj.dig('source_data', 'pan'), obj.dig('source_data', 'sub_type'),
      obj.dig('source_data', 'type'), obj['success']
    ]
    concat = fields.map(&:to_s).join
    computed = OpenSSL::HMAC.hexdigest('sha512', @hmac_secret, concat)
    ActiveSupport::SecurityUtils.secure_compare(computed, received) rescue computed == received
  end
end
```

## Rails Controller

```ruby
class PaymentsController < ApplicationController
  skip_before_action :verify_authenticity_token, only: [:webhook]
  before_action :set_paymob

  def checkout
    amount_cents = (params[:amount].to_f * 100).round
    na = 'NA'
    intention = @paymob.create_intention(
      amount: amount_cents,
      currency: 'EGP',
      payment_methods: [ENV['PAYMOB_INTEGRATION_ID_CARD'].to_i],
      billing_data: {
        first_name: params[:first_name] || na, last_name: params[:last_name] || na,
        email: params[:email], phone_number: '+20000000000',
        apartment: na, floor: na, street: na, building: na,
        shipping_method: na, postal_code: na, city: na, country: 'EG', state: na
      },
      customer: { first_name: na, last_name: na, email: params[:email] },
      items: [],
      notification_url: paymob_webhook_url,
      redirection_url: payment_complete_url
    )
    render json: {
      client_secret: intention[:client_secret],
      checkout_url: @paymob.checkout_url(intention[:client_secret])
    }
  end

  def webhook
    body = JSON.parse(request.body.read)
    obj  = body['obj'] || {}
    hmac = params[:hmac]
    unless @paymob.validate_transaction_hmac(obj, hmac)
      return render json: { error: 'Invalid HMAC' }, status: :unauthorized
    end
    if obj['success']
      Rails.logger.info "Payment #{obj['id']} succeeded"
    end
    render json: { received: true }
  end

  private

  def set_paymob
    @paymob = PaymobClient.new
  end
end
```

## Sinatra App

```ruby
require 'sinatra'
require 'json'

paymob = PaymobClient.new

post '/api/checkout' do
  content_type :json
  data = JSON.parse(request.body.read)
  amount_cents = (data['amount'].to_f * 100).round
  na = 'NA'
  intention = paymob.create_intention(
    amount: amount_cents, currency: 'EGP',
    payment_methods: [ENV['PAYMOB_INTEGRATION_ID_CARD'].to_i],
    billing_data: { first_name: na, last_name: na, email: data['email'],
                    phone_number: '+20000000000', apartment: na, floor: na,
                    street: na, building: na, shipping_method: na,
                    postal_code: na, city: na, country: 'EG', state: na },
    customer: { first_name: na, last_name: na, email: data['email'] },
    items: [], notification_url: 'https://yoursite.com/webhook',
    redirection_url: 'https://yoursite.com/complete'
  )
  { checkout_url: paymob.checkout_url(intention[:client_secret]) }.to_json
end
```
