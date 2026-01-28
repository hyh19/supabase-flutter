# signInWithOAuth Method Explanation

The `signInWithOAuth` method initiates an OAuth authentication flow with a third-party provider. It is defined as an extension method on `GoTrueClient`, making it available through the `supabase.auth` API.

## Method Signature

```dart 254:260:packages/supabase_flutter/lib/src/supabase_auth.dart
Future<bool> signInWithOAuth(
  OAuthProvider provider, {
  String? redirectTo,
  String? scopes,
  LaunchMode authScreenLaunchMode = LaunchMode.platformDefault,
  Map<String, String>? queryParams,
}) async {
```

The method is asynchronous and returns a `Future<bool>` indicating whether the OAuth URL was successfully launched.

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `provider` | `OAuthProvider` | The third-party OAuth provider to authenticate with (e.g., `OAuthProvider.google`, `OAuthProvider.github`) |
| `redirectTo` | `String?` | Optional deep link URL where the user will be redirected after successful authentication |
| `scopes` | `String?` | Optional additional OAuth scopes to request from the provider |
| `authScreenLaunchMode` | `LaunchMode` | Controls how the auth screen is launched (default: `LaunchMode.platformDefault`) |
| `queryParams` | `Map<String, String>?` | Optional additional query parameters to include in the OAuth URL |

## Implementation Flow

### Step 1: Generate OAuth Sign-In URL

```dart 261:266:packages/supabase_flutter/lib/src/supabase_auth.dart
final res = await getOAuthSignInUrl(
  provider: provider,
  redirectTo: redirectTo,
  scopes: scopes,
  queryParams: queryParams,
);
final uri = Uri.parse(res.url);
```

The method calls `getOAuthSignInUrl` (provided by the underlying Gotrue client) to construct the OAuth authorization URL. This URL includes necessary parameters such as the client ID, redirect URI, scopes, and a state parameter for security.

### Step 2: Determine Launch Mode

```dart 269:277:packages/supabase_flutter/lib/src/supabase_auth.dart
LaunchMode launchMode = authScreenLaunchMode;

// `Platform.isAndroid` throws on web, so adding a guard for web here.
final isAndroid = !kIsWeb && Platform.isAndroid;

// Google login has to be performed on external browser window on Android
if (provider == OAuthProvider.google && isAndroid) {
  launchMode = LaunchMode.externalApplication;
}
```

The launch mode determines how the OAuth authorization page is displayed to the user. Several important considerations apply:

- **Platform Detection**: The code uses `!kIsWeb && Platform.isAndroid` to safely detect Android devices. This guard is necessary because `Platform.isAndroid` throws an exception when running on web platforms.

- **Google OAuth Special Case**: Google requires authentication to occur in an external browser window on Android, not in an embedded web view. This is a security requirement from Google to prevent token interception. Therefore, when `provider` is `OAuthProvider.google` on Android, the launch mode is forced to `LaunchMode.externalApplication`.

### Step 3: Launch the URL

```dart 279:284:packages/supabase_flutter/lib/src/supabase_auth.dart
final result = await launchUrl(
  uri,
  mode: launchMode,
  webOnlyWindowName: '_self',
);
return result;
```

The `launchUrl` function from the `url_launcher` package opens the OAuth authorization page. On web platforms, `webOnlyWindowName: '_self'` ensures the OAuth flow replaces the current page rather than opening a new tab or window.

## Return Value Behavior

The method returns a `bool` indicating whether the URL was successfully launched. This does **not** indicate whether authentication succeeded. The return value will be `false` if:

- The user cancelled the operation
- No suitable app is available to handle the URL
- The URL launch fails for any reason

Authentication success should be determined by listening to the `onAuthStateChanged` stream, which emits auth state change events when the OAuth flow completes and the session is established.

## Usage Example

```dart
await supabase.auth.signInWithOAuth(
  OAuthProvider.google,
  redirectTo: 'myapp://auth/callback',
  scopes: 'email profile openid',
);
```

## OAuth Flow Sequence

```text
┌─────────────────────────────────────────────────────────────────┐
│  signInWithOAuth() called                                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  getOAuthSignInUrl() generates authorization URL               │
│  (includes client_id, redirect_uri, scopes, state)             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Platform checks performed                                      │
│  └─ Google on Android → external browser required              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  launchUrl() opens authorization page in browser               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  User authenticates with provider (e.g., enters Google creds)  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Provider redirects to redirectTo URL with auth code           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  App receives deep link, extracts session                      │
│  └─ onAuthStateChanged emits Authenticated state               │
└─────────────────────────────────────────────────────────────────┘
```

## Platform-Specific Behavior

| Platform | Launch Mode | Special Considerations |
|----------|-------------|------------------------|
| **iOS** | Uses `authScreenLaunchMode` (default) or external browser | Google OAuth works in either mode |
| **Android** | Default or external browser | **Google OAuth must use external browser** |
| **Web** | In-app browser | `webOnlyWindowName: '_self'` replaces current page |

## Comparison with Related Methods

| Method | Purpose | User State |
|--------|---------|------------|
| `signInWithOAuth` | Authenticate with consumer OAuth providers (Google, GitHub, etc.) | Not authenticated |
| `signInWithSSO` | Authenticate with enterprise identity providers (SAML, OIDC) | Not authenticated |
| `linkIdentity` | Link additional OAuth identity to existing account | Already authenticated |

## Error Handling Notes

Since the method only returns whether the URL was launched, proper error handling should involve:

1. Checking the return value for URL launch failures
2. Setting up a listener on `onAuthStateChanged` to detect successful authentication
3. Handling the deep link callback to process the OAuth response

## Implementation Details Summary

1. **Asynchronous Operation**: The method is `async` and must be awaited
2. **URL Generation**: Delegated to underlying Gotrue client's `getOAuthSignInUrl`
3. **Platform Safety**: Uses `kIsWeb` guard before accessing `Platform.isAndroid`
4. **Google Requirement**: Forces external browser for Google on Android
5. **Web Behavior**: Uses `_self` window name to replace current page on web
6. **Return Value**: Only indicates URL launch success, not authentication success
