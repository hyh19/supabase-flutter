# GoTrueClientSignInProvider Extension

The `GoTrueClientSignInProvider` extension adds OAuth and SSO authentication capabilities to the `GoTrueClient` class. This extension provides methods for third-party authentication, enterprise SSO, and identity linking.

## Extension Overview

```dart 236:368:packages/supabase_flutter/lib/src/supabase_auth.dart
extension GoTrueClientSignInProvider on GoTrueClient {
```

This extension leverages methods from the underlying Gotrue client (`getOAuthSignInUrl`, `getSSOSignInUrl`, `getLinkIdentityUrl`) and adds Flutter-specific functionality like URL launching and platform-specific handling.

## OAuth Authentication with `signInWithOAuth()`

```dart 254:285:packages/supabase_flutter/lib/src/supabase_auth.dart
Future<bool> signInWithOAuth(
  OAuthProvider provider, {
  String? redirectTo,
  String? scopes,
  LaunchMode authScreenLaunchMode = LaunchMode.platformDefault,
  Map<String, String>? queryParams,
}) async {
  final res = await getOAuthSignInUrl(
    provider: provider,
    redirectTo: redirectTo,
    scopes: scopes,
    queryParams: queryParams,
  );
  final uri = Uri.parse(res.url);

  LaunchMode launchMode = authScreenLaunchMode;

  // `Platform.isAndroid` throws on web, so adding a guard for web here.
  final isAndroid = !kIsWeb && Platform.isAndroid;

  // Google login has to be performed on external browser window on Android
  if (provider == OAuthProvider.google && isAndroid) {
    launchMode = LaunchMode.externalApplication;
  }

  final result = await launchUrl(
    uri,
    mode: launchMode,
    webOnlyWindowName: '_self',
  );
  return result;
}
```

### Parameters

| Parameter | Type | Purpose |
|-----------|------|---------|
| `provider` | `OAuthProvider` | The third-party provider (google, github, facebook, etc.) |
| `redirectTo` | `String?` | Deep link URL to return to app after OAuth |
| `scopes` | `String?` | Additional OAuth scopes to request |
| `authScreenLaunchMode` | `LaunchMode` | How to launch the auth screen |
| `queryParams` | `Map<String, String>?` | Additional query parameters for the OAuth URL |

### Return Value

The method returns a `bool` indicating whether the URL was launched successfully. Note that this doesn't indicate whether authentication succeeded—only that the OAuth flow was initiated.

### Platform-Specific Handling

```dart 272:277:packages/supabase_flutter/lib/src/supabase_auth.dart
// `Platform.isAndroid` throws on web, so adding a guard for web here.
final isAndroid = !kIsWeb && Platform.isAndroid;

// Google login has to be performed on external browser window on Android
if (provider == OAuthProvider.google && isAndroid) {
  launchMode = LaunchMode.externalApplication;
}
```

Google OAuth on Android requires launching in an external browser window (`LaunchMode.externalApplication`) rather than an embedded web view. This is a security requirement from Google.

### Usage Example

```dart
await supabase.auth.signInWithOAuth(
  OAuthProvider.google,
  redirectTo: 'myapp://auth/callback',
  scopes: 'email profile',
);
```

## Single Sign-On with `signInWithSSO()`

```dart 308:326:packages/supabase_flutter/lib/src/supabase_auth.dart
Future<bool> signInWithSSO({
  String? providerId,
  String? domain,
  String? redirectTo,
  String? captchaToken,
  LaunchMode launchMode = LaunchMode.platformDefault,
}) async {
  final ssoUrl = await getSSOSignInUrl(
    providerId: providerId,
    domain: domain,
    redirectTo: redirectTo,
    captchaToken: captchaToken,
  );
  return await launchUrl(
    Uri.parse(ssoUrl),
    mode: launchMode,
    webOnlyWindowName: '_self',
  );
}
```

### Parameters

| Parameter | Type | Purpose |
|-----------|------|---------|
| `providerId` | `String?` | UUID of the SSO Identity Provider |
| `domain` | `String?` | Organization domain to auto-detect the IdP |
| `redirectTo` | `String?` | Deep link URL for callback |
| `captchaToken` | `String?` | CAPTCHA verification token if required |
| `launchMode` | `LaunchMode` | How to launch the SSO screen |

### How SSO Works

1. **Domain-based**: Provide a domain (e.g., 'company.com') to auto-detect the registered IdP
2. **Direct Provider**: Provide the IdP UUID for organization-specific login pages
3. The user is redirected to the identity provider's authorization page
4. Upon success, the user is redirected back via the deep link

### Usage Example

```dart
// Using domain
await supabase.auth.signInWithSSO(
  domain: 'company.com',
  redirectTo: 'myapp://auth/sso-callback',
);

// Using provider ID
await supabase.auth.signInWithSSO(
  providerId: 'sso-provider-uuid',
);
```

## Nonce Generation with `generateRawNonce()`

```dart 328:331:packages/supabase_flutter/lib/src/supabase_auth.dart
String generateRawNonce() {
  final random = Random.secure();
  return base64Url.encode(List<int>.generate(16, (_) => random.nextInt(256)));
}
```

This utility generates a cryptographically secure random nonce for use with OAuth flows.

### Technical Details

| Aspect | Value |
|--------|-------|
| Length | 16 bytes (32 characters in base64url) |
| Randomness | `Random.secure()` for cryptographic safety |
| Encoding | Base64 URL-safe (no `+` or `/` characters) |

### Usage Context

```dart
final nonce = supabase.auth.generateRawNonce();
// Use in PKCE flow or state parameter
```

## Identity Linking with `linkIdentity()`

```dart 335:366:packages/supabase_flutter/lib/src/supabase_auth.dart
Future<bool> linkIdentity(
  OAuthProvider provider, {
  String? redirectTo,
  String? scopes,
  LaunchMode authScreenLaunchMode = LaunchMode.platformDefault,
  Map<String, String>? queryParams,
}) async {
  final res = await getLinkIdentityUrl(
    provider,
    redirectTo: redirectTo,
    scopes: scopes,
    queryParams: queryParams,
  );
  final uri = Uri.parse(res.url);

  LaunchMode launchMode = authScreenLaunchMode;

  // `Platform.isAndroid` throws on web, so adding a guard for web here.
  final isAndroid = !kIsWeb && Platform.isAndroid;

  // Google login has to be performed on external browser window on Android
  if (provider == OAuthProvider.google && isAndroid) {
    launchMode = LaunchMode.externalApplication;
  }

  final result = await launchUrl(
    uri,
    mode: launchMode,
    webOnlyWindowName: '_self',
  );
  return result;
}
```

This method links an additional OAuth identity to an existing logged-in user account. It supports the PKCE flow for secure linking.

### Use Cases

- Link multiple OAuth providers to one account
- Connect social accounts to existing authenticated sessions
- Migrate users between authentication methods

### Usage Example

```dart
// Current user can link their Google account
await supabase.auth.linkIdentity(
  OAuthProvider.google,
  redirectTo: 'myapp://auth/link-callback',
);
```

## URL Launch Configuration

### LaunchMode Options

| Mode | Platform | Behavior |
|------|----------|----------|
| `platformDefault` | All | Uses system default behavior |
| `externalApplication` | Mobile | Opens in external browser/app |
| `inAppWebView` | Mobile | Opens in embedded web view |
| `inAppBrowser` | Web | Opens in app-managed browser |

### Web-Specific: `webOnlyWindowName`

```dart
webOnlyWindowName: '_self',
```

On web, this parameter controls how the OAuth window behaves:

- `'_self'`: Replaces the current page (used here for seamless flow)
- `'_blank'`: Opens in new tab/window

## OAuth Flow Comparison

| Aspect | OAuth (signInWithOAuth) | SSO (signInWithSSO) | Link Identity |
|--------|------------------------|---------------------|---------------|
| **User State** | Not authenticated | Not authenticated | Already authenticated |
| **Provider** | Consumer (Google, GitHub) | Enterprise (SAML, OIDC) | Consumer (same as OAuth) |
| **Flow** | Standard OAuth 2.0 | SAML/OIDC Protocol | OAuth with linking |
| **Result** | New session created | New session created | New identity added |

## Error Handling Pattern

All methods return a `bool` indicating URL launch success, not authentication success:

```dart
final launched = await launchUrl(uri, mode: launchMode);
if (!launched) {
  // URL launch failed (user cancelled, no browser, etc.)
}
// OAuth flow continues in browser...
// Auth success is detected via onAuthStateChange listener
```

## Complete Authentication Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│  User Initiates OAuth                                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  signInWithOAuth() / signInWithSSO() / linkIdentity()           │
│  ├─ Generate OAuth URL from Gotrue                              │
│  ├─ Check platform requirements (Android Google special case)   │
│  └─ Determine launch mode                                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  launchUrl()                                                     │
│  ├─ Opens browser/Auth screen                                   │
│  └─ Returns success/failure of URL launch                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  User Completes Authentication in Browser                        │
│  ├─ Enters credentials at IdP                                   │
│  ├─ Authorizes access                                           │
│  └─ Redirects with auth code or token                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  App Receives Deep Link                                          │
│  ├─ appLinks detects OAuth callback                             │
│  └─ _handleDeeplink() processes URL                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Session Established                                             │
│  ├─ getSessionFromUrl() extracts session                        │
│  ├─ onAuthStateChange fires with new session                    │
│  └─ SupabaseAuth persists session to storage                    │
└─────────────────────────────────────────────────────────────────┘
```

## Key Implementation Details

1. **URL-Based Flow**: The extension doesn't handle tokens directly—it launches URLs and lets deep links handle token extraction
2. **Platform Guards**: `!kIsWeb && Platform.isAndroid` pattern prevents errors on web where `Platform.isAndroid` throws
3. **Google Special Case**: Android requires external browser for Google OAuth due to their security policy
4. **PKCE Support**: All methods can be used with PKCE flow for enhanced security
5. **No Direct Token Access**: Return values indicate URL launch success, not auth success—listen to `onAuthStateChange` for auth results
