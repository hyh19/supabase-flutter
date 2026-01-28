# SupabaseAuth Class Deep Dive

The `SupabaseAuth` class is the core authentication management component in the Supabase Flutter SDK. It acts as a mixin that integrates with Flutter's widget system to handle authentication state, deep link processing, and session persistence across app lifecycle events.

## Class Overview

```dart 15:16:packages/supabase_flutter/lib/src/supabase_auth.dart
class SupabaseAuth with WidgetsBindingObserver {
```

The class uses two mixins/patterns:

- `WidgetsBindingObserver`: Allows the class to observe application lifecycle events (foreground/background transitions)
- The class manages both persistent session storage and real-time deep link handling for OAuth callbacks

## Core Member Variables

### Storage and Configuration

```dart 18:25:packages/supabase_flutter/lib/src/supabase_auth.dart
late LocalStorage _localStorage;
late AuthFlowType _authFlowType;
late bool _autoRefreshToken;
static bool _initialDeeplinkIsHandled = false;
```

| Variable | Purpose |
|----------|---------|
| `_localStorage` | Abstraction for persisting sessions (uses SharedPreferences on mobile, localStorage on web) |
| `_authFlowType` | Determines OAuth flow type: implicit or PKCE (Proof Key for Code Exchange) |
| `_autoRefreshToken` | Flag to enable automatic JWT token refresh before expiration |
| `_initialDeeplinkIsHandled` | Ensures initial deep link is processed only once per app session |

### Subscription Management

```dart 29:33:packages/supabase_flutter/lib/src/supabase_auth.dart
StreamSubscription<AuthState>? _authSubscription;
StreamSubscription<Uri?>? _deeplinkSubscription;
final _appLinks = AppLinks();
final _log = Logger('supabase.supabase_flutter');
```

- `_authSubscription`: Listens to Gotrue's authentication state changes to persist sessions
- `_deeplinkSubscription`: Monitors incoming deep links for OAuth callbacks
- `_appLinks`: Instance of `app_links` package for universal deep link handling
- `_log`: Logger instance for debugging and monitoring

## Initialization Process

The `initialize()` method sets up the entire authentication infrastructure:

```dart 40:83:packages/supabase_flutter/lib/src/supabase_auth.dart
Future<void> initialize({
  required FlutterAuthClientOptions options,
}) async {
  _localStorage = options.localStorage!;
  _authFlowType = options.authFlowType;
  _autoRefreshToken = options.autoRefreshToken;

  _authSubscription = Supabase.instance.client.auth.onAuthStateChange.listen(
    (data) {
      _onAuthStateChange(data.event, data.session);
    },
    onError: (error, stackTrace) {},
  );

  await _localStorage.initialize();

  final hasPersistedSession = await _localStorage.hasAccessToken();
  var shouldEmitInitialSession = true;
  if (hasPersistedSession) {
    final persistedSession = await _localStorage.accessToken();
    if (persistedSession != null) {
      try {
        await Supabase.instance.client.auth
            .setInitialSession(persistedSession);
        shouldEmitInitialSession = false;
      } catch (error, stackTrace) {
        _log.warning(
            'Error while setting initial session', error, stackTrace);
      }
    }
  }
  if (shouldEmitInitialSession) {
    Supabase.instance.client.auth
        // ignore: invalid_use_of_internal_member
        .notifyAllSubscribers(AuthChangeEvent.initialSession);
  }
  _widgetsBindingInstance?.addObserver(this);

  if (options.detectSessionInUri) {
    await _startDeeplinkObserver();
  }
}
```

### Initialization Flow

1. **Store Configuration**: Extract options for storage type, auth flow, and refresh behavior
2. **Subscribe to Auth Changes**: Listen for auth state changes to persist sessions automatically
3. **Initialize Storage**: Set up the local storage backend
4. **Check for Persisted Session**: Verify if a previous session exists
5. **Restore or Emit**: Either restore the persisted session or emit a null session event
6. **Register Lifecycle Observer**: Listen for app foreground/background transitions
7. **Start Deep Link Observer**: Begin monitoring for OAuth callbacks if enabled

### Session Recovery

```dart 88:102:packages/supabase_flutter/lib/src/supabase_auth.dart
Future<void> recoverSession() async {
  try {
    final hasPersistedSession = await _localStorage.hasAccessToken();
    if (hasPersistedSession) {
      final persistedSession = await _localStorage.accessToken();
      if (persistedSession != null) {
        await Supabase.instance.client.auth.recoverSession(persistedSession);
      }
    }
  } on AuthException catch (error, stackTrace) {
    _log.warning(error.message, error, stackTrace);
  } catch (error, stackTrace) {
    _log.warning("Error while recovering session", error, stackTrace);
  }
}
```

The `recoverSession()` method is called lazily after initialization. It attempts to restore a session from local storage without emitting an initial session event. This is useful when the session needs to be recovered at a specific point in the app's lifecycle rather than immediately during initialization.

### Difference from `setInitialSession()`

| Aspect | `setInitialSession()` | `recoverSession()` |
|--------|----------------------|-------------------|
| Called During | `initialize()` | Lazily by `Supabase` instance |
| Emits Events | No (unless failed) | No |
| Purpose | Initial app startup | Subsequent session restoration |
| Timing | Synchronous with init | Asynchronous/lazy |

## Session Persistence via Auth State Change

```dart 130:136:packages/supabase_flutter/lib/src/supabase_auth.dart
void _onAuthStateChange(AuthChangeEvent event, Session? session) {
  if (session != null) {
    _localStorage.persistSession(jsonEncode(session.toJson()));
  } else if (event == AuthChangeEvent.signedOut) {
    _localStorage.removePersistedSession();
  }
}
```

This callback handles two scenarios:

1. **Session Exists**: Persist the session data to local storage for future restoration
2. **Signed Out**: Remove the persisted session when user logs out

## Deep Link Processing

### Auth Callback Detection

```dart 139:145:packages/supabase_flutter/lib/src/supabase_auth.dart
bool _isAuthCallbackDeeplink(Uri uri) {
  return (uri.fragment.contains('access_token') &&
          _authFlowType == AuthFlowType.implicit) ||
      (uri.queryParameters.containsKey('code') &&
          _authFlowType == AuthFlowType.pkce) ||
      (uri.fragment.contains('error_description'));
}
```

The method determines if a deep link is an OAuth callback based on:

| Condition | Auth Flow | What It Matches |
|-----------|-----------|-----------------|
| Fragment contains `access_token` | Implicit | Fragment-based token delivery |
| Query param contains `code` | PKCE | Authorization code delivery |
| Fragment contains `error_description` | Both | Error notifications |

### Deep Link Observer Management

```dart 148:162:packages/supabase_flutter/lib/src/supabase_auth.dart
Future<void> _startDeeplinkObserver() async {
  _log.fine('Starting deeplink observer');
  _handleIncomingLinks();
  await _handleInitialUri();
}

void _stopDeeplinkObserver() {
  if (_deeplinkSubscription != null) {
    _log.fine('Stopping deeplink observer');
    _deeplinkSubscription?.cancel();
  }
}
```

The observer is started to listen for:

1. **Incoming Links**: Deep links received while app is running
2. **Initial URI**: The deep link that launched the app (web only)

### Handling Incoming Links

```dart 166:181:packages/supabase_flutter/lib/src/supabase_auth.dart
void _handleIncomingLinks() {
  if (!kIsWeb) {
    _deeplinkSubscription = _appLinks.uriLinkStream.listen(
      (Uri? uri) {
        if (uri != null) {
          _handleDeeplink(uri);
        }
      },
      onError: (Object err, StackTrace stackTrace) {
        _onErrorReceivingDeeplink(err, stackTrace);
      },
    );
  }
}
```

On native platforms, the `uriLinkStream` provides continuous updates when deep links arrive. On web, this is not needed since the initial URI handling covers the OAuth redirect.

### Initial URI Handling

```dart 190:211:packages/supabase_flutter/lib/src/supabase_auth.dart
Future<void> _handleInitialUri() async {
  if (_initialDeeplinkIsHandled) return;
  _initialDeeplinkIsHandled = true;

  if (kIsWeb) {
    try {
      final Uri? uri = await _appLinks.getInitialLink();
      if (uri != null) {
        await _handleDeeplink(uri);
      }
    } on PlatformException catch (err, stackTrace) {
      _onErrorReceivingDeeplink(err.message ?? err, stackTrace);
    } on FormatException catch (err, stackTrace) {
      _onErrorReceivingDeeplink(err.message, stackTrace);
    } catch (err, stackTrace) {
      _onErrorReceivingDeeplink(err, stackTrace);
    }
  }
}
```

The initial URI handling is web-specific because native platforms deliver the initial URI through the stream. The method uses a flag to ensure it's only processed once, as `getInitialLink()`/`getInitialUri()` is designed for single-use on app launch.

### Deep Link Processing

```dart 214:228:packages/supabase_flutter/lib/src/supabase_auth.dart
Future<void> _handleDeeplink(Uri uri) async {
  if (!_isAuthCallbackDeeplink(uri)) return;

  _log.finest('handle deeplink uri: $uri');
  _log.info('handle deeplink uri');

  try {
    await Supabase.instance.client.auth.getSessionFromUrl(uri);
  } on AuthException catch (error, stackTrace) {
    Supabase.instance.client.auth.notifyException(error, stackTrace);
  } catch (error, stackTrace) {
    _log.warning('Error while getSessionFromUrl', error, stackTrace);
  }
}
```

When an auth callback deep link is received:

1. Validate it's an OAuth callback
2. Extract session data from the URL
3. Handle authentication exceptions by notifying subscribers
4. Log any other errors without throwing

## App Lifecycle Management

```dart 115:128:packages/supabase_flutter/lib/src/supabase_auth.dart
@override
void didChangeAppLifecycleState(AppLifecycleState state) {
  switch (state) {
    case AppLifecycleState.resumed:
      if (_autoRefreshToken) {
        Supabase.instance.client.auth.startAutoRefresh();
      }
    case AppLifecycleState.detached:
    case AppLifecycleState.paused:
      if (kIsWeb || Platform.isAndroid || Platform.isIOS) {
        Supabase.instance.client.auth.stopAutoRefresh();
      }
    default:
  }
}
```

The lifecycle observer controls token refresh behavior:

| State | Action | Platform | Reason |
|-------|--------|----------|--------|
| `resumed` | Start auto-refresh | All | App is active, tokens can refresh |
| `paused` | Stop auto-refresh | Web, iOS, Android | App in background, network limited |
| `detached` | Stop auto-refresh | Android, iOS | Activity destroyed, no network |

### Why Stop Auto-Refresh in Background?

- **Battery Conservation**: Network requests drain battery when backgrounded
- **Connection Limits**: Background apps have limited network access
- **Token Expiry**: Background tokens may expire before app resumes

## Resource Cleanup

```dart 105:112:packages/supabase_flutter/lib/src/supabase_auth.dart
void dispose() {
  if (!kIsWeb && Platform.environment.containsKey('FLUTTER_TEST')) {
    _initialDeeplinkIsHandled = false;
  }
  _authSubscription?.cancel();
  _stopDeeplinkObserver();
  _widgetsBindingInstance?.removeObserver(this);
}
```

The `dispose()` method:

1. Resets the deep link flag for tests on native platforms
2. Cancels the auth state subscription
3. Stops the deep link observer
4. Removes the lifecycle observer

## Authentication Flow Diagram

```text
┌─────────────────────────────────────────────────────────────┐
│                    App Launch                                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  SupabaseAuth.initialize()                                   │
│  ├─ Set up auth state listener                              │
│  ├─ Initialize local storage                                │
│  ├─ Check for persisted session                             │
│  ├─ Restore session or emit initialSession                  │
│  ├─ Register lifecycle observer                             │
│  └─ Start deep link observer (if detectSessionInUri)        │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
┌─────────────────────────┐   ┌─────────────────────────────────┐
│ Has Persisted Session   │   │ No Persisted Session            │
│ └─ setInitialSession()  │   │ └─ emit initialSession (null)   │
└─────────────────────────┘   └─────────────────────────────────┘
              │                               │
              └───────────────┬───────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  App Running - Monitoring Events                             │
│  ├─ Auth state changes → persist/remove session             │
│  ├─ Deep links → process OAuth callbacks                    │
│  ├─ Lifecycle: resumed → start auto-refresh                 │
│  └─ Lifecycle: paused → stop auto-refresh                   │
└─────────────────────────────────────────────────────────────┘
```

## Key Design Patterns

1. **Observer Pattern**: Uses `WidgetsBindingObserver` to react to lifecycle changes
2. **Stream Subscription**: Manages subscriptions for auth state and deep links
3. **Resource Management**: Properly disposes of subscriptions and observers
4. **Platform-Specific Handling**: Different behavior for web vs native platforms
5. **Error Isolation**: Errors in callbacks don't crash the app, they're logged
