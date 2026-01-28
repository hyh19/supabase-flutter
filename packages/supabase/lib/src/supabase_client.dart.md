# SupabaseClient - Core Client for Supabase Flutter SDK

## Overview

The `SupabaseClient` class is the central coordinator and entry point for the Supabase Flutter SDK. It provides a unified interface to interact with all Supabase services including authentication, database operations, real-time subscriptions, file storage, and edge functions.

```dart 42:42:packages/supabase/lib/src/supabase_client.dart
class SupabaseClient {
```

This class follows the facade pattern, initializing and orchestrating multiple specialized clients while providing a single, consistent API for developers.

## Architecture

### Service Clients Coordination

The `SupabaseClient` initializes and manages five core service clients:

```dart 56:64:packages/supabase/lib/src/supabase_client.dart
  GoTrueClient? _authInstance;

  /// Supabase Functions allows you to deploy and invoke edge functions.
  late final FunctionsClient functions;

  /// Supabase Storage allows you to manage user-generated content, such as photos or videos.
  late final SupabaseStorageClient storage;
  late final RealtimeClient realtime;
  late final PostgrestClient rest;
```

Each client handles a specific aspect of the Supabase ecosystem:

- **`GoTrueClient`**: Authentication and user management
- **`FunctionsClient`**: Serverless edge functions execution
- **`SupabaseStorageClient`**: File upload/download and storage management
- **`RealtimeClient`**: WebSocket-based real-time subscriptions
- **`PostgrestClient`**: REST API interactions with the database

### URL Structure and Endpoints

The client constructs service-specific URLs from the base Supabase URL:

```dart 131:135:packages/supabase/lib/src/supabase_client.dart
        _restUrl = '$supabaseUrl/rest/v1',
        _realtimeUrl = '$supabaseUrl/realtime/v1'.replaceAll('http', 'ws'),
        _authUrl = '$supabaseUrl/auth/v1',
        _storageUrl = '$supabaseUrl/storage/v1',
        _functionsUrl = '$supabaseUrl/functions/v1',
```

This creates the standard Supabase service endpoints:

- REST API: `https://project.supabase.co/rest/v1`
- Realtime: `wss://project.supabase.co/realtime/v1`
- Auth: `https://project.supabase.co/auth/v1`
- Storage: `https://project.supabase.co/storage/v1`
- Functions: `https://project.supabase.co/functions/v1`

## Initialization and Configuration

### Constructor Parameters

The client accepts comprehensive configuration options:

```dart 117:142:packages/supabase/lib/src/supabase_client.dart
  /// {@macro supabase_client}
  SupabaseClient(
    String supabaseUrl,
    String supabaseKey, {
    PostgrestClientOptions postgrestOptions = const PostgrestClientOptions(),
    AuthClientOptions authOptions = const AuthClientOptions(),
    StorageClientOptions storageOptions = const StorageClientOptions(),
    FunctionsClientOptions functionsOptions = const FunctionsClientOptions(),
    RealtimeClientOptions realtimeClientOptions = const RealtimeClientOptions(),
    this.accessToken,
    Map<String, String>? headers,
    Client? httpClient,
    YAJsonIsolate? isolate,
  })
```

**Key Parameters:**

- `supabaseUrl` & `supabaseKey`: Core credentials from Supabase dashboard
- `accessToken`: Optional function for third-party authentication systems
- `headers`: Custom HTTP headers for all requests
- `httpClient`: Custom HTTP client for advanced networking needs
- `isolate`: JSON parsing isolate for performance optimization

### Client Initialization Sequence

During construction, the client follows a specific initialization order:

```dart 143:162:packages/supabase/lib/src/supabase_client.dart
    _authInstance = _initSupabaseAuthClient(
      autoRefreshToken: authOptions.autoRefreshToken,
      gotrueAsyncStorage: authOptions.pkceAsyncStorage,
      authFlowType: authOptions.authFlowType,
    );
    _authHttpClient =
        AuthHttpClient(_supabaseKey, httpClient ?? Client(), _getAccessToken);
    rest = _initRestClient();
    functions = _initFunctionsClient();
    storage = _initStorageClient(storageOptions.retryAttempts);
    realtime = _initRealtimeClient(options: realtimeClientOptions);
    if (accessToken == null) {
      _log.config(
          'Initialize SupabaseClient v$version with no custom access token');
      _listenForAuthEvents();
    } else {
      _log.config(
          'Initialize SupabaseClient v$version with custom access token');
    }
```

This ensures proper dependency injection and authentication setup.

## Authentication Integration

### Auth State Management

The client deeply integrates with authentication through several mechanisms:

```dart 355:363:packages/supabase/lib/src/supabase_client.dart
  void _listenForAuthEvents() {
    // ignore: invalid_use_of_internal_member
    _authStateSubscription = auth.onAuthStateChangeSync.listen(
      (data) async {
        await _handleTokenChanged(data.event, data.session?.accessToken);
      },
      onError: (error, stack) {},
    );
  }
```

### Token Propagation

When authentication state changes, tokens are automatically propagated to all services:

```dart 365:385:packages/supabase/lib/src/supabase_client.dart
  Future<void> _handleTokenChanged(AuthChangeEvent event, String? token) async {
    if (event == AuthChangeEvent.initialSession ||
        event == AuthChangeEvent.tokenRefreshed ||
        event == AuthChangeEvent.signedIn) {
      try {
        await realtime.setAuth(token);
      } on FormatException catch (e) {
        if (e.message.contains('InvalidJWTToken')) {
          // The exception is thrown by RealtimeClient when the token is
          // expired for example on app launch after the app has been closed
          // for a while.
        } else {
          rethrow;
        }
      }
    } else if (event == AuthChangeEvent.signedOut) {
      // Token is removed

      await realtime.setAuth(_supabaseKey);
    }
  }
```

### AuthHttpClient Wrapper

All HTTP requests are wrapped through `AuthHttpClient` which automatically injects JWT tokens:

```dart 11:11:packages/supabase/lib/src/supabase_client.dart
import 'auth_http_client.dart';
```

```dart 148:149:packages/supabase/lib/src/supabase_client.dart
    _authHttpClient =
        AuthHttpClient(_supabaseKey, httpClient ?? Client(), _getAccessToken);
```

### Third-Party Authentication Support

The client supports custom authentication systems:

```dart 67:67:packages/supabase/lib/src/supabase_client.dart
  final Future<String?> Function()? accessToken;
```

When provided, this disables the built-in auth client and uses external token management.

## Core API Methods

### Database Operations

The `from()` method provides the primary interface for database queries:

```dart 174:187:packages/supabase/lib/src/supabase_client.dart
  /// Perform a table operation.
  SupabaseQueryBuilder from(String table) {
    final url = '$_restUrl/$table';
    return SupabaseQueryBuilder(
      url,
      realtime,
      headers: {...rest.headers, ...headers},
      schema: _postgrestOptions.schema,
      table: table,
      httpClient: _authHttpClient,
      incrementId: _incrementId.increment(),
      isolate: _isolate,
    );
  }
```

This creates a query builder that combines REST API calls with real-time capabilities.

### Schema Selection

For multi-schema databases:

```dart 189:204:packages/supabase/lib/src/supabase_client.dart
  /// Select a schema to query or perform an function (rpc) call.
  ///
  /// The schema needs to be on the list of exposed schemas inside Supabase.
  SupabaseQuerySchema schema(String schema) {
    final newRest = rest.schema(schema);
    return SupabaseQuerySchema(
      counter: _incrementId,
      restUrl: _restUrl,
      headers: headers,
      schema: schema,
      isolate: _isolate,
      authHttpClient: _authHttpClient,
      realtime: realtime,
      rest: newRest,
    );
  }
```

### Remote Procedure Calls

For calling database functions:

```dart 206:214:packages/supabase/lib/src/supabase_client.dart
  /// {@macro postgrest_rpc}
  PostgrestFilterBuilder<T> rpc<T>(
    String fn, {
    Map<String, dynamic>? params,
    get = false,
  }) {
    rest.headers.addAll({...rest.headers, ...headers});
    return rest.rpc(fn, params: params, get: get);
  }
```

### Real-Time Channels

Channel management for real-time features:

```dart 216:225:packages/supabase/lib/src/supabase_client.dart
  /// Creates a Realtime channel with Broadcast, Presence, and Postgres Changes.
  RealtimeChannel channel(String name,
      {RealtimeChannelConfig opts = const RealtimeChannelConfig()}) {
    return realtime.channel(name, opts);
  }

  /// Returns all Realtime channels.
  List<RealtimeChannel> getChannels() {
    return realtime.getChannels();
  }
```

## Header Management

### Dynamic Header Updates

The client supports runtime header modifications with proper propagation:

```dart 79:114:packages/supabase/lib/src/supabase_client.dart
  /// To apply the new headers in existing realtime channels, manually unsubscribe and resubscribe these channels.
  set headers(Map<String, String> headers) {
    _headers.clear();
    _headers.addAll({
      ...Constants.defaultHeaders,
      ...headers,
    });

    rest.headers
      ..clear()
      ..addAll(_headers);

    functions.headers
      ..clear()
      ..addAll(_headers);

    storage.headers
      ..clear()
      ..addAll(_headers);

    if (accessToken == null) {
      auth.headers
        ..clear()
        ..addAll({
          ...Constants.defaultHeaders,
          ..._getAuthHeaders(),
          ...headers,
        });
    }

    // To apply the new headers in the realtime client,
    // manually unsubscribe and resubscribe to all channels.
    realtime.headers
      ..clear()
      ..addAll(_headers);
  }
```

## Token Management

### Access Token Resolution

The client intelligently resolves access tokens from multiple sources:

```dart 241:270:packages/supabase/lib/src/supabase_client.dart
  /// Get either the custom access token from [accessToken] or the supabase one
  /// from [_authInstance]
  Future<String?> _getAccessToken() async {
    if (accessToken != null) {
      return await accessToken!();
    }

    final authInstance = _authInstance!;

    if (authInstance.currentSession?.isExpired ?? false) {
      try {
        await authInstance.refreshSession();
      } catch (error, stackTrace) {
        final expiresAt = authInstance.currentSession?.expiresAt;
        if (expiresAt != null) {
          // Failed to refresh the token.
          final isExpiredWithoutMargin = DateTime.now()
              .isAfter(DateTime.fromMillisecondsSinceEpoch(expiresAt * 1000));
          if (isExpiredWithoutMargin) {
            // Throw the error instead of making an API request with an expired token.
            _log.warning(
              'Access token is expired and refreshing failed, aborting api request',
              error,
              stackTrace,
            );
            rethrow;
          }
        }
      }
    }
    return authInstance.currentSession?.accessToken;
  }
```

This includes automatic token refresh and intelligent error handling.

## Service Client Initialization

### Auth Client Setup

```dart 280:297:packages/supabase/lib/src/supabase_client.dart
  GoTrueClient _initSupabaseAuthClient({
    bool? autoRefreshToken,
    required GotrueAsyncStorage? gotrueAsyncStorage,
    required AuthFlowType authFlowType,
  }) {
    final authHeaders = {...headers};
    authHeaders['apikey'] = _supabaseKey;
    authHeaders['Authorization'] = 'Bearer $_supabaseKey';

    return GoTrueClient(
      url: _authUrl,
      headers: authHeaders,
      autoRefreshToken: autoRefreshToken,
      httpClient: _httpClient,
      asyncStorage: gotrueAsyncStorage,
      flowType: authFlowType,
    );
  }
```

### REST Client Setup

```dart 299:307:packages/supabase/lib/src/supabase_client.dart
  PostgrestClient _initRestClient() {
    return PostgrestClient(
      _restUrl,
      headers: {...headers},
      schema: _postgrestOptions.schema,
      httpClient: _authHttpClient,
      isolate: _isolate,
    );
  }
```

### Other Service Clients

Functions, storage, and realtime clients follow similar patterns with their specific configurations.

## Resource Management

### Disposal and Cleanup

The client properly manages resource lifecycle:

```dart 272:278:packages/supabase/lib/src/supabase_client.dart
  Future<void> dispose() async {
    _log.fine('Dispose SupabaseClient');
    await realtime.disconnect();
    await _authStateSubscription?.cancel();
    await _isolate.dispose();
    _authInstance?.dispose();
  }
```

This ensures clean shutdown of all services and prevents resource leaks.

## Key Features

1. **Unified API**: Single entry point for all Supabase services
2. **Automatic Authentication**: JWT tokens automatically injected into requests
3. **Real-time Integration**: Seamless combination of REST and WebSocket operations
4. **Flexible Configuration**: Extensive customization options for all services
5. **Third-party Auth Support**: Compatible with external authentication systems
6. **Resource Management**: Proper cleanup and disposal of resources
7. **Error Handling**: Intelligent token refresh and error recovery
8. **Performance Optimization**: JSON parsing in separate isolate

## Usage Patterns

### Basic Initialization

```dart
final supabase = SupabaseClient(
  'https://your-project.supabase.co',
  'your-anon-key',
);
```

### Database Query

```dart
final data = await supabase.from('users').select('id, name');
```

### Real-time Subscription

```dart
final channel = supabase.channel('users');
channel.onPostgresChanges(
  event: PostgresChangeEvent.all,
  schema: 'public',
  table: 'users',
  callback: (payload) => print('Change received: $payload'),
).subscribe();
```

### Authentication

```dart
await supabase.auth.signUp(
  email: 'user@example.com',
  password: 'password',
);
```

The `SupabaseClient` serves as the foundation of the Supabase Flutter SDK, providing developers with a powerful, flexible, and easy-to-use interface for building applications with Supabase.
