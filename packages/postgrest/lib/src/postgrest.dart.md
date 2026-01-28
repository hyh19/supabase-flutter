# PostgrestClient - PostgREST API Client

The `PostgrestClient` class is the core component of the PostgREST Dart client library, providing an ORM-like interface for interacting with PostgreSQL databases through RESTful APIs. This client enables Flutter and Dart applications to perform database operations using a fluent, chainable API that mirrors SQL query patterns.

## Class Overview

The `PostgrestClient` serves as the main entry point for all database interactions, handling HTTP requests, authentication, and JSON parsing with performance optimizations through isolate-based processing.

```dart 8:40:packages/postgrest/lib/src/postgrest.dart
/// A PostgREST api client written in Dartlang. The goal of this library is to make an "ORM-like" restful interface.
class PostgrestClient {
  final String url;
  final Map<String, String> headers;
  final String? _schema;
  final Client? httpClient;
  final YAJsonIsolate _isolate;
  final bool _hasCustomIsolate;
  final _log = Logger('supabase.postgrest');

  /// To create a [PostgrestClient], you need to provide an [url] endpoint.
  ///
  /// You can also provide custom [headers] and [_schema] if needed
  /// ```dart
  /// PostgrestClient(REST_URL)
  /// PostgrestClient(REST_URL, headers: {'apikey': 'foo'})
  /// ```
  ///
  /// [httpClient] is optional and can be used to provide a custom http client
  ///
  /// [isolate] is optional and can be used to provide a custom isolate, which is used for heavy json computation
  PostgrestClient(
    this.url, {
    Map<String, String>? headers,
    String? schema,
    this.httpClient,
    YAJsonIsolate? isolate,
  })  : _schema = schema,
        headers = {...defaultHeaders, if (headers != null) ...headers},
        _isolate = isolate ?? (YAJsonIsolate()..initialize()),
        _hasCustomIsolate = isolate != null {
    _log.config('Initialize PostgrestClient with url: $url, schema: $_schema');
    _log.finest('Initialize with headers: $headers');
  }
```

## Core Properties

- **`url`**: The base REST API endpoint URL
- **`headers`**: HTTP headers including authentication and content-type headers
- **`_schema`**: Optional database schema name for multi-schema databases
- **`httpClient`**: Optional custom HTTP client for advanced networking needs
- **`_isolate`**: JSON parsing isolate for performance optimization
- **`_hasCustomIsolate`**: Flag to track if isolate was provided externally

## Constructor Parameters

The constructor accepts several optional parameters for customization:

- **`headers`**: Custom HTTP headers (merged with default headers)
- **`schema`**: Database schema name for multi-tenant applications
- **`httpClient`**: Custom HTTP client implementation
- **`isolate`**: Custom JSON parsing isolate

Default headers include content-type and other necessary headers for PostgREST communication.

## Authentication Methods

### Legacy Authentication (Deprecated)

```dart 42:47:packages/postgrest/lib/src/postgrest.dart
  /// Authenticates the request with JWT.
  @Deprecated("Use setAuth() instead")
  PostgrestClient auth(String token) {
    headers['Authorization'] = 'Bearer $token';
    return this;
  }
```

The legacy `auth()` method is deprecated in favor of the more flexible `setAuth()` method.

### Modern Authentication

```dart 49:57:packages/postgrest/lib/src/postgrest.dart
  PostgrestClient setAuth(String? token) {
    _log.finest("setAuth with: $token");
    if (token != null) {
      headers['Authorization'] = 'Bearer $token';
    } else {
      headers.remove('Authorization');
    }
    return this;
  }
```

The `setAuth()` method provides proper JWT token management:

- Sets Bearer token when provided
- Removes authorization header when token is null
- Returns the client instance for method chaining

## Core Database Operations

### Table Operations

```dart 59:69:packages/postgrest/lib/src/postgrest.dart
  /// Perform a table operation.
  PostgrestQueryBuilder<void> from(String table) {
    final url = '${this.url}/$table';
    return PostgrestQueryBuilder<void>(
      url: Uri.parse(url),
      headers: {...headers},
      schema: _schema,
      httpClient: httpClient,
      isolate: _isolate,
    );
  }
```

The `from()` method initiates table-level operations by creating a `PostgrestQueryBuilder`. This is the primary entry point for building database queries using a fluent interface pattern.

Example usage:

```dart
// Basic table access
supabase.from('users')

// Query building
supabase.from('users').select('id, name').eq('active', true)
```

### Schema Selection

```dart 71:82:packages/postgrest/lib/src/postgrest.dart
  /// Select a schema to query or perform an function (rpc) call.
  ///
  /// The schema needs to be on the list of exposed schemas inside Supabase.
  PostgrestClient schema(String schema) {
    return PostgrestClient(
      url,
      headers: {...headers},
      schema: schema,
      httpClient: httpClient,
      isolate: _isolate,
    );
  }
```

The `schema()` method creates a new client instance configured for a specific database schema, enabling multi-schema database operations.

### Remote Procedure Calls (RPC)

```dart 84:112:packages/postgrest/lib/src/postgrest.dart
  /// {@template postgrest_rpc}
  /// Performs a stored procedure call.
  ///
  /// [fn] is the name of the function to call.
  ///
  /// [params] is an optional object to pass as arguments to the function call.
  ///
  /// When [get] is set to `true`, the function will be called with read-only
  /// access mode.
  ///
  /// {@endtemplate}
  ///
  /// ```dart
  /// supabase.rpc('get_status', params: {'name_param': 'supabot'})
  /// ```
  PostgrestFilterBuilder<T> rpc<T>(
    String fn, {
    Map? params,
    bool get = false,
  }) {
    final url = '${this.url}/rpc/$fn';
    return PostgrestRpcBuilder(
      url,
      headers: {...headers},
      schema: _schema,
      httpClient: httpClient,
      isolate: _isolate,
    ).rpc(params, get);
  }
```

The `rpc()` method enables calling PostgreSQL stored procedures and functions through the REST API. It supports both read-write and read-only operations.

Parameters:

- **`fn`**: Function name in the database
- **`params`**: Optional parameters to pass to the function
- **`get`**: When true, performs read-only access

## Resource Management

```dart 114:119:packages/postgrest/lib/src/postgrest.dart
  Future<void> dispose() async {
    _log.fine("dispose PostgrestClient");
    if (!_hasCustomIsolate) {
      return _isolate.dispose();
    }
  }
```

The `dispose()` method properly cleans up resources, particularly the JSON parsing isolate when it was created internally by the client.

## Usage Patterns

### Basic Initialization

```dart
// Simple initialization
final client = PostgrestClient('https://your-project.supabase.co/rest/v1');

// With authentication
final client = PostgrestClient(
  'https://your-project.supabase.co/rest/v1',
  headers: {'apikey': 'your-anon-key'}
);

// With custom HTTP client
final client = PostgrestClient(
  'https://your-project.supabase.co/rest/v1',
  httpClient: MyCustomClient()
);
```

### Authentication Flow

```dart
// Set authentication token
client.setAuth('jwt-token-here');

// Remove authentication
client.setAuth(null);
```

### Database Operations

```dart
// Select data
final users = await client.from('users').select('id, name');

// Insert data
final result = await client.from('users').insert({'name': 'John'});

// Update data
await client.from('users').update({'name': 'Jane'}).eq('id', 1);

// Delete data
await client.from('users').delete().eq('id', 1);

// Call stored procedure
final result = await client.rpc('get_user_count');
```

### Schema Operations

```dart
// Use specific schema
final adminClient = client.schema('admin');
final users = await adminClient.from('system_users').select();
```

## Architectural Patterns

The `PostgrestClient` implements several key architectural patterns:

1. **Fluent Interface**: Method chaining for building queries
2. **Builder Pattern**: Progressive query construction through specialized builders
3. **Immutable Configuration**: Schema method returns new client instances
4. **Resource Management**: Proper cleanup through dispose pattern
5. **Performance Optimization**: Isolate-based JSON parsing for heavy computations

## Error Handling

The client relies on the underlying HTTP client and PostgREST API error responses. Common error scenarios include:

- Network connectivity issues
- Authentication failures
- Invalid query syntax
- Database constraint violations
- Schema access permissions

## Integration with Supabase

This client is designed to work seamlessly with the broader Supabase ecosystem:

- **Authentication**: Integrates with Supabase Auth for automatic token injection
- **Real-time**: Complements Supabase Realtime for live data updates
- **Storage**: Works alongside Supabase Storage for file operations
- **Edge Functions**: Enables calling serverless functions

The client serves as the foundation for the `SupabaseClient.from()` method, providing a unified interface across all Supabase services.
