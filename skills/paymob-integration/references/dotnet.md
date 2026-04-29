# Paymob Integration -- .NET / C#

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
dotnet add package System.Net.Http.Json
```

## PaymobClient Class

```csharp
using System.Security.Cryptography;
using System.Text;
using System.Net.Http.Headers;

public class PaymobClient {
    private readonly string _secretKey;
    private readonly string _publicKey;
    private readonly string _hmacSecret;
    private readonly string _baseUrl;
    private readonly HttpClient _http;

    public PaymobClient(IConfiguration config, HttpClient http) {
        _secretKey  = config["PAYMOB_SECRET_KEY"]!;
        _publicKey  = config["PAYMOB_PUBLIC_KEY"]!;
        _hmacSecret = config["PAYMOB_HMAC_SECRET"]!;
        _baseUrl    = config["PAYMOB_BASE_URL"] ?? "https://accept.paymob.com";
        _http = http;
        _http.BaseAddress = new Uri(_baseUrl);
        _http.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Token", _secretKey);
    }

    public async Task<(string Id, string ClientSecret)> CreateIntentionAsync(object payload) {
        var response = await _http.PostAsJsonAsync("/v1/intention/", payload);
        response.EnsureSuccessStatusCode();
        var body = await response.Content.ReadFromJsonAsync<Dictionary<string, object>>();
        return (body!["id"].ToString()!, body["client_secret"].ToString()!);
    }

    public string CheckoutUrl(string clientSecret) =>
        $"{_baseUrl}/unifiedcheckout/?publicKey={_publicKey}&clientSecret={clientSecret}";

    public async Task<JsonElement> RefundAsync(long transactionId, int amountCents) {
        var response = await _http.PostAsJsonAsync("/api/acceptance/void_refund/refund",
            new { transaction_id = transactionId, amount_cents = amountCents });
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<JsonElement>();
    }

    public async Task<JsonElement> VoidAsync(long transactionId) {
        var response = await _http.PostAsJsonAsync("/api/acceptance/void_refund/void",
            new { transaction_id = transactionId });
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<JsonElement>();
    }

    public bool ValidateTransactionHmac(Dictionary<string, object> obj, string received) {
        var fields = new[] {
            obj["amount_cents"], obj["created_at"], obj["currency"],
            obj["error_occured"], obj["has_parent_transaction"],
            obj["id"], obj["integration_id"], obj["is_3d_secure"],
            obj["is_auth"], obj["is_capture"], obj["is_refunded"],
            obj["is_standalone_payment"], obj["is_voided"],
            ((Dictionary<string,object>)obj["order"])["id"],
            obj["owner"], obj["pending"],
            ((Dictionary<string,object>)obj["source_data"])["pan"],
            ((Dictionary<string,object>)obj["source_data"])["sub_type"],
            ((Dictionary<string,object>)obj["source_data"])["type"],
            obj["success"],
        };
        var concat = string.Concat(fields.Select(f => f?.ToString() ?? ""));
        using var hmac = new HMACSHA512(Encoding.UTF8.GetBytes(_hmacSecret));
        var computed = Convert.ToHexString(hmac.ComputeHash(Encoding.UTF8.GetBytes(concat))).ToLower();
        return CryptographicOperations.FixedTimeEquals(
            Encoding.UTF8.GetBytes(computed), Encoding.UTF8.GetBytes(received));
    }
}
```

## ASP.NET Core Controller

```csharp
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api")]
public class PaymobController : ControllerBase {
    private readonly PaymobClient _paymob;
    private readonly IConfiguration _config;

    public PaymobController(PaymobClient paymob, IConfiguration config) {
        _paymob = paymob; _config = config;
    }

    [HttpPost("checkout")]
    public async Task<IActionResult> Checkout([FromBody] CheckoutRequest req) {
        var amountCents = (int)Math.Round(req.Amount * 100);
        const string NA = "NA";
        var payload = new {
            amount = amountCents, currency = "EGP",
            payment_methods = new[] { int.Parse(_config["PAYMOB_INTEGRATION_ID_CARD"]!) },
            billing_data = new {
                first_name = req.FirstName ?? NA, last_name = req.LastName ?? NA,
                email = req.Email, phone_number = "+20000000000",
                apartment = NA, floor = NA, street = NA, building = NA,
                shipping_method = NA, postal_code = NA, city = NA,
                country = "EG", state = NA
            },
            customer = new { first_name = NA, last_name = NA, email = req.Email },
            items = Array.Empty<object>(),
            notification_url = Url.Action("Webhook", "Paymob", null, Request.Scheme),
            redirection_url = $"{Request.Scheme}://{Request.Host}/payment/complete"
        };
        var (_, clientSecret) = await _paymob.CreateIntentionAsync(payload);
        return Ok(new { client_secret = clientSecret, checkout_url = _paymob.CheckoutUrl(clientSecret) });
    }

    [HttpPost("paymob/webhook")]
    public IActionResult Webhook([FromBody] WebhookPayload payload, [FromQuery] string hmac) {
        var obj = payload.Obj;
        if (!_paymob.ValidateTransactionHmac(obj, hmac))
            return Unauthorized(new { error = "Invalid HMAC" });
        if (obj.TryGetValue("success", out var success) && success?.ToString() == "True")
            Console.WriteLine($"Payment succeeded");
        return Ok(new { received = true });
    }
}

public record CheckoutRequest(decimal Amount, string Email, string? FirstName, string? LastName);
public record WebhookPayload(Dictionary<string, object> Obj);
```

## Program.cs (Dependency Injection)

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddHttpClient<PaymobClient>();
builder.Services.AddControllers();
var app = builder.Build();
app.MapControllers();
app.Run();
```
