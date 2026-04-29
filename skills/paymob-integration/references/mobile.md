# Paymob Integration -- Mobile (React Native / Flutter)

## Overview

Mobile apps should call your backend to create the intention. Never expose the secret key in mobile apps.

Flow:
1. Mobile app sends cart data to your backend
2. Backend creates intention, returns clientSecret + checkout URL
3. Mobile opens WebView or browser with checkout URL

## React Native

### Installation

```bash
npm install axios react-native-webview
# iOS
cd ios && pod install
```

### Checkout Hook

```tsx
import { useState } from 'react';
import axios from 'axios';

export function usePaymobCheckout() {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const initiateCheckout = async (amount: number, email: string) => {
    setLoading(true);
    setError(null);
    try {
      const { data } = await axios.post('https://yourbackend.com/api/checkout', {
        amount, email,
      });
      return data.checkout_url as string;
    } catch (e: any) {
      setError(e.message || 'Checkout failed');
      return null;
    } finally {
      setLoading(false);
    }
  };

  return { initiateCheckout, loading, error };
}
```

### WebView Payment Screen

```tsx
import React, { useState } from 'react';
import { View, Button, ActivityIndicator, Alert } from 'react-native';
import WebView from 'react-native-webview';
import { usePaymobCheckout } from './usePaymobCheckout';

export default function CheckoutScreen({ amount, email, navigation }) {
  const { initiateCheckout, loading } = usePaymobCheckout();
  const [checkoutUrl, setCheckoutUrl] = useState<string | null>(null);

  const startCheckout = async () => {
    const url = await initiateCheckout(amount, email);
    if (url) setCheckoutUrl(url);
  };

  const handleNavChange = (navState: any) => {
    // Detect redirect to your completion URL
    if (navState.url.includes('/payment/complete')) {
      const params = new URL(navState.url).searchParams;
      if (params.get('success') === 'true') {
        Alert.alert('Success', 'Payment completed!');
      } else {
        Alert.alert('Failed', 'Payment failed. Please try again.');
      }
      navigation.goBack();
    }
  };

  if (checkoutUrl) {
    return (
      <WebView
        source={{ uri: checkoutUrl }}
        onNavigationStateChange={handleNavChange}
        startInLoadingState
        renderLoading={() => <ActivityIndicator />}
      />
    );
  }

  return (
    <View>
      {loading ? <ActivityIndicator /> : (
        <Button title={Pay EGP } onPress={startCheckout} />
      )}
    </View>
  );
}
```

### Deep Link Handling (optional)

```tsx
// Register deep link in app.json / AndroidManifest
// Then handle redirect:
import { Linking } from 'react-native';

Linking.addEventListener('url', ({ url }) => {
  if (url.includes('payment/complete')) {
    const params = new URL(url).searchParams;
    const success = params.get('success') === 'true';
    // navigate to result screen
  }
});
```

## Flutter

### pubspec.yaml

```yaml
dependencies:
  http: ^1.2.0
  webview_flutter: ^4.7.0
  url_launcher: ^6.2.0
```

### Paymob Service

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;

class PaymobService {
  final String backendUrl;
  PaymobService({required this.backendUrl});

  Future<String> createCheckoutUrl({
    required int amountCents,
    required String email,
  }) async {
    final response = await http.post(
      Uri.parse('/api/checkout'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode({'amount': amountCents, 'email': email}),
    );
    if (response.statusCode != 200) {
      throw Exception('Checkout failed: ');
    }
    final data = jsonDecode(response.body);
    return data['checkout_url'] as String;
  }
}
```

### Flutter WebView Screen

```dart
import 'package:flutter/material.dart';
import 'package:webview_flutter/webview_flutter.dart';

class PaymobCheckoutScreen extends StatefulWidget {
  final String checkoutUrl;
  final VoidCallback onSuccess;
  final VoidCallback onFailure;
  const PaymobCheckoutScreen({
    required this.checkoutUrl, required this.onSuccess, required this.onFailure, super.key
  });

  @override State<PaymobCheckoutScreen> createState() => _State();
}

class _State extends State<PaymobCheckoutScreen> {
  late final WebViewController _controller;

  @override
  void initState() {
    super.initState();
    _controller = WebViewController()
      ..setJavaScriptMode(JavaScriptMode.unrestricted)
      ..setNavigationDelegate(NavigationDelegate(
        onNavigationRequest: (request) {
          if (request.url.contains('/payment/complete')) {
            final uri = Uri.parse(request.url);
            if (uri.queryParameters['success'] == 'true') {
              widget.onSuccess();
            } else {
              widget.onFailure();
            }
            Navigator.pop(context);
            return NavigationDecision.prevent;
          }
          return NavigationDecision.navigate;
        },
      ))
      ..loadRequest(Uri.parse(widget.checkoutUrl));
  }

  @override
  Widget build(BuildContext context) => Scaffold(
    appBar: AppBar(title: const Text('Payment')),
    body: WebViewWidget(controller: _controller),
  );
}
```

### Flutter Checkout Flow

```dart
final service = PaymobService(backendUrl: 'https://yourbackend.com');

Future<void> startPayment(BuildContext context) async {
  try {
    final url = await service.createCheckoutUrl(
      amountCents: 5000, // EGP 50.00
      email: 'user@example.com',
    );
    if (!context.mounted) return;
    Navigator.push(context, MaterialPageRoute(
      builder: (_) => PaymobCheckoutScreen(
        checkoutUrl: url,
        onSuccess: () => ScaffoldMessenger.of(context)
            .showSnackBar(const SnackBar(content: Text('Payment successful!'))),
        onFailure: () => ScaffoldMessenger.of(context)
            .showSnackBar(const SnackBar(content: Text('Payment failed.'))),
      ),
    ));
  } catch (e) {
    ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('Error: ')));
  }
}
```

## Native Mobile SDKs

Paymob provides official native SDKs for direct (non-WebView) integration:

- **iOS SDK**: Available on CocoaPods / Swift Package Manager
- **Android SDK**: Available on Maven Central / Gradle

Contact Paymob support or check your merchant dashboard for SDK download links and integration guides.

## Notes

- Never embed PAYMOB_SECRET_KEY in mobile apps
- Use SSL pinning for backend API calls in production
- Test with Paymob sandbox credentials before going live
- Handle 
otification_url webhooks server-side regardless of mobile redirect result
