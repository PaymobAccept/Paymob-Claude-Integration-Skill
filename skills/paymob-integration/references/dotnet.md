# Paymob Integration — .NET / C#

## Quick Start

### NuGet Package (Community SDK)

```bash
dotnet add package Xdot.Paymob.CashIn.DependencyInjection
```

### Configuration

```csharp
// Startup.cs / Program.cs
services.AddPaymobCashIn(config =>
{
    config.ApiKey = Configuration["Paymob:ApiKey"];
    config.Hmac = Configuration["Paymob:HmacSecret"];
});
```

```json
// appsettings.json
{
  "Paymob": {
    "ApiKey": "your_api_key",
    "SecretKey": "sk_live_xxxxxxx",
    "HmacSecret": "your_hmac_secret",
    "IntegrationIdCard": 123456,
    "IntegrationIdWallet": 789012,
    "IframeId": "12345",
    "BaseUrl": "https://accept.paymob.com"
  }
}
```

## Using the Xdot.Paymob SDK

The SDK provides `IPaymobCashInBroker` with these methods:

```csharp
// Create an order
var orderRequest = CashInCreateOrderRequest.CreateOrder(amountCents: 10000);
var orderResponse = await _broker.CreateOrderAsync(orderRequest);

// Generate payment key
var billingData = new CashInBillingData(
    firstName: "John",
    lastName: "Doe",
    phoneNumber: "+201234567890",
    email: "john@example.com"
);

var paymentKeyRequest = new CashInPaymentKeyRequest(
    integrationId: 123456,
    orderId: orderResponse.Id,
    billingData: billingData,
    amountCents: 10000
);

var paymentKeyResponse = await _broker.RequestPaymentKeyAsync(paymentKeyRequest);

// Generate iframe URL
string iframeSrc = _broker.CreateIframeSrc(iframeId: "12345", token: paymentKeyResponse.PaymentKey);

// Mobile wallet payment
var walletRequest = new CashInWalletPayRequest(paymentKeyResponse.PaymentKey, "01234567890");
var walletResponse = await _broker.CreateWalletPayAsync(walletRequest);

// Kiosk payment
var kioskRequest = new CashInKioskPayRequest(paymentKeyResponse.PaymentKey);
var kioskResponse = await _broker.CreateKioskPayAsync(kioskRequest);
// kioskResponse.BillReference — customer uses this at kiosk

// HMAC validation
bool isValid = _broker.Validate(transaction, hmacFromQueryString);
```

## Manual Integration (Without SDK)

```csharp
using System.Net.Http;
using System.Net.Http.Json;
using System.Security.Cryptography;
using System.Text;

public class PaymobService
{
    private readonly HttpClient _http;
    private readonly PaymobConfig _config;

    public PaymobService(HttpClient http, IOptions<PaymobConfig> config)
    {
        _http = http;
        _config = config.Value;
    }

    // Intention API
    public async Task<IntentionResponse> CreateIntentionAsync(
        int amountCents, string currency, int[] integrationIds,
        BillingData billing, Item[] items,
        string notificationUrl, string redirectionUrl)
    {
        var request = new HttpRequestMessage(HttpMethod.Post,
            $"{_config.BaseUrl}/v1/intention/");

        request.Headers.Add("Authorization", $"Token {_config.SecretKey}");
        request.Content = JsonContent.Create(new
        {
            amount = amountCents,
            currency,
            payment_methods = integrationIds,
            items,
            billing_data = billing,
            notification_url = notificationUrl,
            redirection_url = redirectionUrl,
        });

        var response = await _http.SendAsync(request);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<IntentionResponse>();
    }

    // HMAC Validation
    public bool ValidateHMAC(TransactionCallback transaction, string receivedHMAC)
    {
        var fields = new[]
        {
            transaction.AmountCents.ToString(),
            transaction.CreatedAt,
            transaction.Currency,
            transaction.ErrorOccured.ToString().ToLower(),
            transaction.HasParentTransaction.ToString().ToLower(),
            transaction.Id.ToString(),
            transaction.IntegrationId.ToString(),
            transaction.Is3dSecure.ToString().ToLower(),
            transaction.IsAuth.ToString().ToLower(),
            transaction.IsCapture.ToString().ToLower(),
            transaction.IsRefunded.ToString().ToLower(),
            transaction.IsStandalonePayment.ToString().ToLower(),
            transaction.IsVoided.ToString().ToLower(),
            transaction.Order.Id.ToString(),
            transaction.Owner.ToString(),
            transaction.Pending.ToString().ToLower(),
            transaction.SourceData.Pan,
            transaction.SourceData.SubType,
            transaction.SourceData.Type,
            transaction.Success.ToString().ToLower(),
        };

        var concatenated = string.Join("", fields);
        using var hmac = new HMACSHA512(Encoding.UTF8.GetBytes(_config.HmacSecret));
        var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(concatenated));
        var calculated = BitConverter.ToString(hash).Replace("-", "").ToLower();

        return calculated == receivedHMAC;
    }
}
```

### Webhook Controller

```csharp
[ApiController]
[Route("api/paymob")]
public class PaymobController : ControllerBase
{
    private readonly PaymobService _paymob;

    public PaymobController(PaymobService paymob) => _paymob = paymob;

    [HttpPost("webhook")]
    public IActionResult Webhook([FromQuery] string hmac, [FromBody] WebhookPayload payload)
    {
        var transaction = payload.Obj;

        if (!_paymob.ValidateHMAC(transaction, hmac))
            return Unauthorized(new { error = "Invalid HMAC" });

        if (transaction.Success)
        {
            // Update order in database
        }

        return Ok(new { received = true });
    }
}
```

## Key Types

```csharp
public record PaymobConfig
{
    public string ApiKey { get; init; }
    public string SecretKey { get; init; }
    public string HmacSecret { get; init; }
    public string BaseUrl { get; init; } = "https://accept.paymob.com";
    public int IntegrationIdCard { get; init; }
    public int IntegrationIdWallet { get; init; }
    public string IframeId { get; init; }
}

public record IntentionResponse(int Id, string ClientSecret);

public record TransactionCallback
{
    public int AmountCents { get; init; }
    public string CreatedAt { get; init; }
    public string Currency { get; init; }
    public bool ErrorOccured { get; init; }
    public bool HasParentTransaction { get; init; }
    public int Id { get; init; }
    public int IntegrationId { get; init; }
    public bool Is3dSecure { get; init; }
    public bool IsAuth { get; init; }
    public bool IsCapture { get; init; }
    public bool IsRefunded { get; init; }
    public bool IsStandalonePayment { get; init; }
    public bool IsVoided { get; init; }
    public OrderRef Order { get; init; }
    public int Owner { get; init; }
    public bool Pending { get; init; }
    public SourceDataRef SourceData { get; init; }
    public bool Success { get; init; }
}
```
