# GoTrueClient Core Architecture

This document explains the core architecture of the `GoTrueClient` class, which is the main API client for interacting with a GoTrue authentication server in a Supabase Flutter application.

## Class Overview

The `GoTrueClient` class is the central component for handling all authentication-related operations in the Supabase Flutter SDK. It manages user sessions, handles authentication flows, and provides methods for various authentication operations.

```dart 39:57:lib/src/gotrue_client.dart
class GoTrueClient {
  /// Namespace for the GoTrue API methods.
  /// These can be used for example to get a user from a JWT in a server environment or reset a user's password.
  late final GoTrueAdminApi admin;

  /// Namespace for the GoTrue MFA API methods.
  late final GoTrueMFAApi mfa;

  /// The session object for the currently logged in user or null.
  Session? _currentSession;

  final String _url;
  final Map<String, String> _headers;
  final Client? _httpClient;
  late final GotrueFetch _fetch = GotrueFetch(_httpClient);

  late bool _autoRefreshToken;

  Timer? _autoRefreshTicker;

  /// Completer to combine multiple simultaneous token refresh requests.
  Completer<AuthResponse>? _refreshTokenCompleter;

  JWKSet? _jwks;
  DateTime? _jwksCachedAt;
```

## Key Properties

### Session Management

- **`_currentSession`**: Stores the currently authenticated user's session. This includes the access token, refresh token, user information, and token expiration details. The session is private and accessed through the public getter `currentSession`.

### HTTP and Network

- **`_url`**: The base URL of the GoTrue instance. Defaults to `http://localhost:9999` if not provided.
- **`_headers`**: HTTP headers sent with every request. Includes default headers and any custom headers provided during initialization.
- **`_httpClient`**: Optional custom HTTP client. If not provided, a default client is used internally.
- **`_fetch`**: The `GotrueFetch` instance that handles all HTTP requests. It is initialized once during construction.

### Authentication Flow

- **`_autoRefreshToken`**: Boolean flag indicating whether the client should automatically refresh access tokens before they expire. Defaults to `true`.
- **`_autoRefreshTicker`**: Timer that periodically checks if a token refresh is needed.
- **`_refreshTokenCompleter`**: Used to combine multiple simultaneous token refresh requests into a single operation, preventing race conditions.

### Security

- **`_jwks`**: Cached JSON Web Key Set used for JWT signature verification.
- **`_jwksCachedAt`**: Timestamp indicating when the JWKS was cached. Used to implement caching with a TTL (Time To Live).

### API Namespaces

```dart 42:45:lib/src/gotrue_client.dart
  late final GoTrueAdminApi admin;
  late final GoTrueMFAApi mfa;
```

The client exposes two namespace objects for organizing related API methods:

- **`admin`**: Provides administrative API methods for server-side operations like resetting passwords, listing users, and managing sessions.
- **`mfa`**: Provides Multi-Factor Authentication API methods for managing MFA factors and challenges.

### Event Broadcasting

```dart 65:67:lib/src/gotrue_client.dart
  final _onAuthStateChangeController = BehaviorSubject<AuthState>();
  final _onAuthStateChangeControllerSync =
      BehaviorSubject<AuthState>(sync: true);
```

Two `BehaviorSubject` streams are maintained for broadcasting authentication state changes:

- **`onAuthStateChange`**: The primary stream for subscribers to receive authentication state changes. This stream is asynchronous.
- **`onAuthStateChangeSync`**: A synchronous version of the auth state change stream, used for internal purposes where immediate delivery is required.

### Broadcast Channel

```dart 95:98:lib/src/gotrue_client.dart
  /// Proxy to the web BroadcastChannel API. Should be null on non-web platforms.
  BroadcastChannel? _broadcastChannel;

  StreamSubscription? _broadcastChannelSubscription;
```

On web platforms, a `BroadcastChannel` is used to synchronize authentication state across multiple browser tabs. This allows a user signed in on one tab to be automatically recognized on other tabs.

### Async Storage

```dart 69:70:lib/src/gotrue_client.dart
  /// Local storage to store pkce code verifiers.
  final GotrueAsyncStorage? _asyncStorage;
```

An optional async storage implementation for storing PKCE code verifiers. This is required for the PKCE (Proof Key for Code Exchange) authentication flow.

### Flow Type

```dart 91:91:lib/src/gotrue_client.dart
  final AuthFlowType _flowType;
```

The authentication flow type determines how the client handles the OAuth/PKCE flow. The default is `AuthFlowType.pkce`, which is more secure than the older implicit flow.

## Public Getters

```dart 138:145:lib/src/gotrue_client.dart
  /// Getter for the headers
  Map<String, String> get headers => _headers;

  /// Returns the current logged in user, asociated to [currentSession] if any;
  User? get currentUser => _currentSession?.user;

  /// Returns the current session, if any;
  Session? get currentSession => _currentSession;
```

The client provides three read-only getters:

- **`headers`**: Returns the current HTTP headers being used.
- **`currentUser`**: Returns the currently authenticated user, or `null` if no session exists.
- **`currentSession`**: Returns the current session object, or `null` if not authenticated.

## Stream Access

```dart 83:89:lib/src/gotrue_client.dart
  /// Receive a notification every time an auth event happens.
  ///
  /// ```dart
  /// supabase.auth.onAuthStateChange.listen((data) {
  ///   final AuthChangeEvent event = data.event;
  ///   final Session? session = data.session;
  ///   if(event == AuthChangeEvent.signedIn) {
  ///     // handle signIn event
  ///   }
  /// });
  /// ```
  Stream<AuthState> get onAuthStateChange =>
      _onAuthStateChangeController.stream;

  /// Don't use this, it's for internal use only.
  @internal
  Stream<AuthState> get onAuthStateChangeSync =>
      _onAuthStateChangeControllerSync.stream;
```

The `onAuthStateChange` stream is the primary way for applications to subscribe to authentication state changes. It emits `AuthState` objects containing the event type and the associated session.

## Constructor

```dart 100:136:lib/src/gotrue_client.dart
  GoTrueClient({
    String? url,
    Map<String, String>? headers,
    bool? autoRefreshToken,
    Client? httpClient,
    GotrueAsyncStorage? asyncStorage,
    AuthFlowType flowType = AuthFlowType.pkce,
  })  : _url = url ?? Constants.defaultGotrueUrl,
        _headers = {
          ...Constants.defaultHeaders,
          ...?headers,
        },
        _httpClient = httpClient,
        _asyncStorage = asyncStorage,
        _flowType = flowType {
    _autoRefreshToken = autoRefreshToken ?? true;

    final gotrueUrl = url ?? Constants.defaultGotrueUrl;
    _log.config(
        'Initialize GoTrueClient v$version with url: $_url, autoRefreshToken: $_autoRefreshToken, flowType: $_flowType, tickDuration: ${Constants.autoRefreshTickDuration}, tickThreshold: ${Constants.autoRefreshTickThreshold}');
    _log.finest('Initialize with headers: $_headers');
    admin = GoTrueAdminApi(
      gotrueUrl,
      headers: _headers,
      httpClient: httpClient,
    );
    mfa = GoTrueMFAApi(
      client: this,
      fetch: _fetch,
    );
    if (_autoRefreshToken) {
      startAutoRefresh();
    }

    _mayStartBroadcastChannel();
  }
```

The constructor initializes all properties and performs the following setup:

1. Sets the URL and headers
2. Initializes the HTTP fetch mechanism
3. Creates the `GoTrueAdminApi` namespace
4. Creates the `GoTrueMFAApi` namespace
5. Starts automatic token refresh if enabled
6. Initializes the web broadcast channel (if running in a browser)

## Logging

```dart 93:93:lib/src/gotrue_client.dart
  final _log = Logger('supabase.auth');
```

The client uses the `logging` package for structured logging with different levels (info, fine, finer, finest, config, warning).

## Initialization Sequence

When a `GoTrueClient` is instantiated, the following initialization steps occur:

1. The URL and headers are set up with defaults
2. The fetch mechanism is initialized
3. If `autoRefreshToken` is true (default), `startAutoRefresh()` is called
4. The broadcast channel is initialized if running in a browser environment

The automatic token refresh mechanism runs a timer that checks every 10 seconds whether a token refresh is needed. If the token will expire within the next 3 ticks (30 seconds), a refresh is triggered.

## Related Types

### Session

The `Session` class represents an authenticated user's session:

```dart 6:16:lib/src/types/session.dart
class Session {
  final String? providerToken;
  final String? providerRefreshToken;
  final String accessToken;
  final int? expiresIn;
  final String? refreshToken;
  final String tokenType;
  final User user;
```

### User

The `User` class contains all user-related information:

```dart 6:54:lib/src/types/user.dart
class User {
  final String id;
  final Map<String, dynamic> appMetadata;
  final Map<String, dynamic>? userMetadata;
  final String aud;
  // ... many more fields
  final bool isAnonymous;
```

### AuthChangeEvent

The `AuthChangeEvent` enum defines all possible authentication state changes:

```dart 39:53:lib/src/constants.dart
enum AuthChangeEvent {
  initialSession('INITIAL_SESSION'),
  passwordRecovery('PASSWORD_RECOVERY'),
  signedIn('SIGNED_IN'),
  signedOut('SIGNED_OUT'),
  tokenRefreshed('TOKEN_REFRESHED'),
  userUpdated('USER_UPDATED'),
  mfaChallengeVerified('MFA_CHALLENGE_VERIFIED');
```

## Summary

The `GoTrueClient` is a comprehensive authentication client that:

- Manages the complete authentication lifecycle
- Handles multiple authentication methods (email, phone, OAuth, SSO, etc.)
- Provides automatic token refresh to maintain sessions
- Broadcasts authentication state changes to subscribers
- Synchronizes state across browser tabs via BroadcastChannel
- Organizes functionality into namespace objects (`admin`, `mfa`)
- Supports both PKCE and implicit authentication flows
