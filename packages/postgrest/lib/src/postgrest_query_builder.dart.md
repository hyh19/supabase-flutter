# PostgrestQueryBuilder Class

## Overview

The `PostgrestQueryBuilder` class is a core component of the Supabase PostgREST client, providing a fluent interface for building database queries. It serves as the entry point for constructing SQL-like operations (SELECT, INSERT, UPSERT, UPDATE, DELETE) with a chainable API that allows stacking filters before execution.

## Architecture

```dart 13:13:packages/postgrest/lib/src/postgrest_query_builder.dart
class PostgrestQueryBuilder<T> extends RawPostgrestBuilder<T, T, T> {
```

The class extends `RawPostgrestBuilder<T, T, T>`, inheriting the base functionality for HTTP request building and response handling. The generic type `T` represents the expected return type from the database operations.

## Constructor

```dart 15:31:packages/postgrest/lib/src/postgrest_query_builder.dart
PostgrestQueryBuilder({
  required Uri url,
  String? method,
  Map<String, String>? headers,
  String? schema,
  Client? httpClient,
  YAJsonIsolate? isolate,
}) : super(
        PostgrestBuilder(
          url: url,
          method: method,
          headers: headers ?? {},
          schema: schema,
          httpClient: httpClient,
          isolate: isolate,
        ),
      );
```

The constructor initializes the query builder with:

- **url**: The base URI for the PostgREST endpoint
- **method**: HTTP method (optional, defaults based on operation)
- **headers**: Custom HTTP headers
- **schema**: Database schema name
- **httpClient**: Custom HTTP client for requests
- **isolate**: JSON parsing isolate for performance optimization

## Core Query Methods

### SELECT Operations

```dart 43:62:packages/postgrest/lib/src/postgrest_query_builder.dart
PostgrestFilterBuilder<PostgrestList> select([String columns = '*']) {
  // Remove whitespaces except when quoted
  var quoted = false;
  final re = RegExp(r'\s');
  final cleanedColumns = columns.split('').map((c) {
    if (re.hasMatch(c) && !quoted) {
      return '';
    }
    if (c == '"') {
      quoted = !quoted;
    }
    return c;
  }).join();

  final url = overrideSearchParams('select', cleanedColumns);
  return PostgrestFilterBuilder(_copyWithType(
    url: url,
    method: METHOD_GET,
  ));
}
```

The `select` method performs read operations on tables/views. Key features:

- **Column Selection**: Accepts a column specification string (defaults to `*` for all columns)
- **Whitespace Handling**: Intelligently removes whitespace while preserving quoted column names
- **Chaining**: Returns a `PostgrestFilterBuilder<PostgrestList>` for applying filters
- **HTTP Method**: Uses GET requests

**Usage Examples:**

```dart
// Select all columns
supabase.from('users').select()

// Select specific columns
supabase.from('users').select('id, name, email')

// Select with quoted column names (handles spaces)
supabase.from('users').select('"user id", name')
```

### INSERT Operations

```dart 88:110:packages/postgrest/lib/src/postgrest_query_builder.dart
PostgrestFilterBuilder<T> insert(
  Object values, {
  bool defaultToNull = true,
}) {
  final newHeaders = {..._headers};
  newHeaders['Prefer'] = '';

  if (!defaultToNull) {
    newHeaders['Prefer'] = 'missing=default';
  }

  Uri url = _url;
  if (values is List) {
    url = _setColumnsSearchParam(values);
  }

  return PostgrestFilterBuilder(_copyWith(
    method: METHOD_POST,
    headers: newHeaders,
    body: values,
    url: url,
  ));
}
```

The `insert` method performs data insertion operations:

- **Single/Multiple Records**: Accepts either a single object or list of objects
- **Default Value Handling**: `defaultToNull` parameter controls whether missing fields use NULL or default values
- **HTTP Headers**: Uses `Prefer` header to control insertion behavior
- **Bulk Operations**: Automatically extracts column names for bulk inserts
- **HTTP Method**: Uses POST requests

**Usage Examples:**

```dart
// Insert single record
await supabase.from('messages').insert({
  'message': 'Hello',
  'user_id': 123
});

// Insert multiple records
await supabase.from('messages').insert([
  {'message': 'Hello', 'user_id': 123},
  {'message': 'World', 'user_id': 456}
]);
```

### UPSERT Operations

```dart 143:178:packages/postgrest/lib/src/postgrest_query_builder.dart
PostgrestFilterBuilder<T> upsert(
  Object values, {
  String? onConflict,
  bool ignoreDuplicates = false,
  bool defaultToNull = true,
}) {
  final newHeaders = {..._headers};
  newHeaders['Prefer'] =
      'resolution=${ignoreDuplicates ? 'ignore' : 'merge'}-duplicates';

  if (!defaultToNull) {
    newHeaders['Prefer'] = '${newHeaders['Prefer']!},missing=default';
  }

  Uri url = _url;

  if (values is List) {
    url = _setColumnsSearchParam(values);
  }

  if (onConflict != null) {
    url = url.replace(
      queryParameters: {
        'on_conflict': onConflict,
        ...url.queryParameters,
      },
    );
  }

  return PostgrestFilterBuilder<T>(_copyWith(
    method: METHOD_POST,
    headers: newHeaders,
    body: values,
    url: url,
  ));
}
```

The `upsert` method combines INSERT and UPDATE operations:

- **Conflict Resolution**: `onConflict` parameter specifies the UNIQUE constraint column(s)
- **Duplicate Handling**: `ignoreDuplicates` controls whether to ignore or merge duplicate records
- **Default Values**: Same `defaultToNull` behavior as INSERT
- **Query Parameters**: Uses `on_conflict` URL parameter for conflict resolution
- **HTTP Headers**: Uses `Prefer` header with resolution strategy

**Usage Examples:**

```dart
// Upsert with conflict resolution
await supabase.from('users').upsert({
  'id': 1,
  'email': 'user@example.com',
  'name': 'John Doe'
}, onConflict: 'email');

// Ignore duplicates
await supabase.from('logs').upsert(
  {'event': 'login', 'user_id': 123},
  ignoreDuplicates: true
);
```

### UPDATE Operations

```dart 200:209:packages/postgrest/lib/src/postgrest_query_builder.dart
PostgrestFilterBuilder<T> update(Map values) {
  final newHeaders = {..._headers};
  newHeaders['Prefer'] = '';

  return PostgrestFilterBuilder<T>(_copyWith(
    method: METHOD_PATCH,
    headers: newHeaders,
    body: values,
  ));
}
```

The `update` method performs data modification operations:

- **Selective Updates**: Accepts a Map of field-value pairs to update
- **HTTP Method**: Uses PATCH requests
- **Filter Required**: Typically used with WHERE conditions (applied via chaining)

**Usage Examples:**

```dart
// Update records matching conditions
await supabase
    .from('users')
    .update({'status': 'active'})
    .eq('id', 123);

// Update multiple fields
await supabase
    .from('profiles')
    .update({
      'name': 'Jane Doe',
      'updated_at': DateTime.now().toIso8601String()
    })
    .eq('user_id', 123);
```

### DELETE Operations

```dart 231:238:packages/postgrest/lib/src/postgrest_query_builder.dart
PostgrestFilterBuilder<T> delete() {
  final newHeaders = {..._headers};
  newHeaders['Prefer'] = '';
  return PostgrestFilterBuilder<T>(_copyWith(
    method: METHOD_DELETE,
    headers: newHeaders,
  ));
}
```

The `delete` method performs data removal operations:

- **No Parameters**: The actual deletion criteria are specified through chaining
- **HTTP Method**: Uses DELETE requests
- **Filter Required**: Must be combined with WHERE conditions

**Usage Examples:**

```dart
// Delete specific record
await supabase
    .from('messages')
    .delete()
    .eq('id', 123);

// Delete with multiple conditions
await supabase
    .from('sessions')
    .delete()
    .eq('user_id', 123)
    .lt('created_at', '2024-01-01');
```

### COUNT Operations

```dart 255:260:packages/postgrest/lib/src/postgrest_query_builder.dart
PostgrestFilterBuilder<int> count([CountOption option = CountOption.exact]) {
  return PostgrestFilterBuilder<int>(_copyWithType(
    method: METHOD_HEAD,
    count: option,
  ));
}
```

The `count` method performs counting operations:

- **Count Options**: Accepts different counting strategies (exact, planned, estimated)
- **Return Type**: Returns `PostgrestFilterBuilder<int>` for count results
- **HTTP Method**: Uses HEAD requests (no body returned)
- **Performance**: Optimized for counting without fetching data

**Usage Examples:**

```dart
// Get exact count
int exactCount = await supabase.from('users').count();

// Get planned count (faster but approximate)
int plannedCount = await supabase.from('users').count(CountOption.planned);

// Count with filters
int activeUsers = await supabase
    .from('users')
    .count()
    .eq('status', 'active');
```

## Helper Methods

### Column Search Parameter Setting

```dart 240:249:packages/postgrest/lib/src/postgrest_query_builder.dart
Uri _setColumnsSearchParam(List values) {
  final newValues = PostgrestList.from(values);
  final columns = newValues.fold<List<String>>(
      [], (value, element) => value..addAll(element.keys));
  if (newValues.isNotEmpty) {
    final uniqueColumns = {...columns}.map((e) => '"$e"').join(',');
    return appendSearchParams("columns", uniqueColumns);
  }
  return _url;
}
```

This private method extracts column names from bulk operations:

- **Column Extraction**: Gathers all unique column names from a list of records
- **Deduplication**: Uses Set to ensure unique column names
- **URL Parameters**: Adds `columns` query parameter for bulk operations
- **Quoting**: Properly quotes column names

## Method Overrides

### Header Setting

```dart 263:273:packages/postgrest/lib/src/postgrest_query_builder.dart
PostgrestQueryBuilder<T> setHeader(String key, String value) {
  return PostgrestQueryBuilder(
    url: _url,
    headers: {..._headers, key: value},
    httpClient: _httpClient,
    method: _method,
    schema: _schema,
    isolate: _isolate,
  );
}
```

Overrides the base class method to return the correct type:

- **Type Safety**: Returns `PostgrestQueryBuilder<T>` instead of the base type
- **Immutability**: Creates new instance with updated headers
- **Fluent Interface**: Maintains method chaining capability

## Design Patterns

### Builder Pattern

The class implements a fluent builder pattern where each method returns a builder that can be further chained:

```dart
supabase
  .from('users')
  .select('id, name')
  .eq('status', 'active')
  .order('name')
  .limit(10)
```

### Immutable Builders

Each method creates a new builder instance rather than modifying the existing one, enabling safe method chaining without side effects.

### Type Safety

The generic type `T` provides compile-time type checking for the expected return types from different operations.

## HTTP Integration

The class integrates with PostgREST's HTTP API:

- **RESTful Design**: Maps CRUD operations to appropriate HTTP methods
- **Header Control**: Uses HTTP headers for operation-specific preferences
- **URL Parameters**: Leverages query parameters for advanced features
- **JSON Payloads**: Sends/receives JSON data for request/response bodies

## Error Handling

While the class itself doesn't implement error handling, it works with the broader PostgREST client ecosystem that includes:

- HTTP status code interpretation
- Network error handling
- Database constraint violation handling
- Authentication error management

## Performance Considerations

- **JSON Isolation**: Optional use of isolates for JSON parsing in large responses
- **Streaming**: Support for streaming responses in the broader ecosystem
- **Connection Reuse**: HTTP client reuse for efficient connection management
- **Query Optimization**: PostgREST handles server-side query optimization

This class serves as the foundation for type-safe, fluent database operations in the Supabase Flutter ecosystem, providing developers with an intuitive interface that maps closely to SQL operations while leveraging the power of PostgREST's RESTful API design.
