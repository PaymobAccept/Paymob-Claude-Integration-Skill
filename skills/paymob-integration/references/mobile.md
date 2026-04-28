# Paymob Integration — Mobile (Flutter, React Native)

## Flutter

### Packages

```yaml
# pubspec.yaml
dependencies:
  http: ^1.1.0
  webview_flutter: ^4.4.0
  # Or use the community package:
  # easy_paymob: ^latest
```

### Service

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;

class PaymobService {
  static const String baseUrl = 'https://accept.paymob.com';
  final String secretKey;

  PaymobService({required this.secretKey});

  /// Create payment intention (recommended)
  Future<Map<String, dynamic>> createIntention({
    required int amountCents,
    required String currency,
    required List<int> paymentMethods,
    required Map<String, String> billingData,
    required List<Map<String, dynamic>> items,
    required String notificationUrl,
    required String redirectionUrl,
  }) async {
    final response = await http.post(
      Uri.parse('$baseUrl/v1/intention/'),
      headers: {
        'Authorization': 'Token $secretKey',
        'Content-Type': 'application/json',
      },
      body: jsonEncode({
        'amount': amountCents,
        'currency': currency,
        'payment_methods': paymentMethods,
        'items': items,
        'billing_data': billingData,
        'notification_url': notificationUrl,
        'redirection_url': redirectionUrl,
      }),
    );

    if (response.statusCode != 200 && response.statusCode != 201) {
      throw Exception('Failed to create intention: ${response.body}');
    }
    return jsonDecode(response.body);
  }
}
```

### WebView Checkout

```dart
import 'package:flutter/material.dart';
import 'package:webview_flutter/webview_flutter.dart';

class PaymobCheckoutScreen extends StatefulWidget {
  final String clientSecret;
  final String publicKey;

  const PaymobCheckoutScreen({
    required this.clientSecret,
    required this.publicKey,
  });

  @override
  State<PaymobCheckoutScreen> createState() => _PaymobCheckoutScreenState();
}

class _PaymobCheckoutScreenState extends State<PaymobCheckoutScreen> {
  late final WebViewController _controller;

  @override
  void initState() {
    super.initState();
    final url = 'https://accept.paymob.com/unifiedcheckout/'
        '?publicKey=${widget.publicKey}'
        '&clientSecret=${widget.clientSecret}';

    _controller = WebViewController()
      ..setJavaScriptMode(JavaScriptMode.unrestricted)
      ..setNavigationDelegate(NavigationDelegate(
        onNavigationRequest: (request) {
          // Detect redirect to your redirection_url
          if (request.url.contains('payment/complete')) {
            final uri = Uri.parse(request.url);
            final success = uri.queryParameters['success'] == 'true';
            Navigator.pop(context, success);
            return NavigationDecision.prevent;
          }
          return NavigationDecision.navigate;
        },
      ))
      ..loadRequest(Uri.parse(url));
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Payment')),
      body: WebViewWidget(controller: _controller),
    );
  }
}
```

### Using easy_paymob Package

```dart
import 'package:easy_paymob/easy_paymob.dart';

// Initialize
final paymob = EasyPaymob(
  apiKey: 'your_api_key',
  iframeId: 12345,
  integrationId: 123456,
);

// In your widget
final result = await paymob.payWithCard(
  context: context,
  amountInCents: '10000',
  currency: 'EGP',
  billingData: BillingData(
    firstName: 'John',
    lastName: 'Doe',
    email: 'john@example.com',
    phone: '+201234567890',
  ),
);

if (result.success) {
  // Payment succeeded
  print('Transaction ID: ${result.transactionId}');
}
```

## React Native

### WebView Approach

```bash
npm install react-native-webview
```

```tsx
import React, { useState } from 'react';
import { View, Button, ActivityIndicator } from 'react-native';
import { WebView } from 'react-native-webview';

interface Props {
  clientSecret: string;
  publicKey: string;
  onComplete: (success: boolean) => void;
}

export function PaymobCheckout({ clientSecret, publicKey, onComplete }: Props) {
  const [loading, setLoading] = useState(true);
  const checkoutUrl = `https://accept.paymob.com/unifiedcheckout/?publicKey=${publicKey}&clientSecret=${clientSecret}`;

  return (
    <View style={{ flex: 1 }}>
      {loading && <ActivityIndicator size="large" style={{ position: 'absolute', top: '50%', alignSelf: 'center' }} />}
      <WebView
        source={{ uri: checkoutUrl }}
        onLoadEnd={() => setLoading(false)}
        onNavigationStateChange={(navState) => {
          // Detect redirect to your success/failure URL
          if (navState.url.includes('payment/complete')) {
            const url = new URL(navState.url);
            const success = url.searchParams.get('success') === 'true';
            onComplete(success);
          }
        }}
        javaScriptEnabled
        domStorageEnabled
      />
    </View>
  );
}
```

### Usage in Screen

```tsx
export function CheckoutScreen() {
  const [checkoutData, setCheckoutData] = useState<{ clientSecret: string } | null>(null);

  async function startPayment() {
    // Call your backend to create an intention
    const res = await fetch('https://your-api.com/api/checkout', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ amount: 150, currency: 'EGP' }),
    });
    const data = await res.json();
    setCheckoutData(data);
  }

  if (checkoutData) {
    return (
      <PaymobCheckout
        clientSecret={checkoutData.clientSecret}
        publicKey="pk_live_xxxxx"
        onComplete={(success) => {
          setCheckoutData(null);
          if (success) {
            // Navigate to success screen
          } else {
            // Show error
          }
        }}
      />
    );
  }

  return <Button title="Pay 150 EGP" onPress={startPayment} />;
}
```

## Important Notes for Mobile

1. **Backend creates the intention** — never put secret keys in mobile apps
2. **Use WebView for checkout** — Paymob's unified checkout works well in embedded WebViews
3. **Detect completion via URL redirect** — watch for navigation to your `redirection_url`
4. **Wallet payments on mobile** — the redirect from wallet apps (Vodafone Cash, etc.) may need deep linking setup
5. **Always validate payment server-side** — use webhooks as the source of truth, not client-side redirect params
