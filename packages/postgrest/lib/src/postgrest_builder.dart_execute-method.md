# The `_execute` Method in PostgrestBuilder

The `_execute` method is the core HTTP request execution engine of the PostgREST builder pattern. It handles all HTTP operations (GET, POST, PUT, PATCH, DELETE, HEAD) for database queries and mutations, managing headers, request bodies, and response parsing.

## Overview

```dart 104:178:packages/postgrest/lib/src/postgrest_builder.dart
Future<T> _execute() async {
  final String? method = _method;

  if (_count != null) {
    if (_headers['Prefer'] != null) {
      final oldPreferHeader = _headers['Prefer'];
      _headers['Prefer'] = '$oldPreferHeader,count=${_count!.name}';
    } else {
      _headers['Prefer'] = 'count=${_count!.name}';
    }
  }

  try {
    if (method == null) {
      throw ArgumentError(
        'Missing table operation: select, insert, update or delete',
      );
    }

    final uppercaseMethod = method.toUpperCase();
    late http.Response response;

    if (_schema == null) {
      // skip
    } else if ([METHOD_GET, METHOD_HEAD].contains(method)) {
      _headers['Accept-Profile'] = _schema!;
    } else {
      _headers['Content-Profile'] = _schema!;
    }
    if (method != METHOD_GET && method != METHOD_HEAD) {
      _headers['Content-Type'] = 'application/json';
    }
    final bodyStr = jsonEncode(_body);
    _log.finest("Request: $uppercaseMethod $_url");

    if (uppercaseMethod == METHOD_GET) {
      response = await (_httpClient?.get ?? http.get)(
        _url,
        headers: _headers,
      );
    } else if (uppercaseMethod == METHOD_POST) {
      response = await (_httpClient?.post ?? http.post)(
        _url,
        headers: _headers,
        body: bodyStr,
      );
    } else if (uppercaseMethod == METHOD_PUT) {
      response = await (_httpClient?.put ?? http.put)(
        _url,
        headers: _headers,
        body: bodyStr,
      );
    } else if (uppercaseMethod == METHOD_PATCH) {
      response = await (_httpClient?.patch ?? http.patch)(
        _url,
        headers: _headers,
        body: bodyStr,
      );
    } else if (uppercaseMethod == METHOD_DELETE) {
      response = await (_httpClient?.delete ?? http.delete)(
        _url,
        headers: _headers,
      );
    } else if (uppercaseMethod == METHOD_HEAD) {
      response = await (_httpClient?.head ?? http.head)(
        _url,
        headers: _headers,
      );
    }

    return _parseResponse(response, method);
  } catch (error) {
    rethrow;
  }
}
```

## Key Components

### 1. Count Header Configuration

The method first handles count preferences by modifying the `Prefer` header. This tells PostgREST to include row counts in the response.

```dart 107:114:packages/postgrest/lib/src/postgrest_builder.dart
if (_count != null) {
  if (_headers['Prefer'] != null) {
    final oldPreferHeader = _headers['Prefer'];
    _headers['Prefer'] = '$oldPreferHeader,count=${_count!.name}';
  } else {
    _headers['Prefer'] = 'count=${_count!.name}';
  }
}
```

The `_count` field is of type `CountOption` which has three values:

- `exact`: Performs a `COUNT(*)` operation (slow but accurate)
- `planned`: Uses PostgreSQL statistics (fast but approximate)
- `estimated`: Uses exact count for low numbers, planned count for high numbers

### 2. Method Validation

```dart 116:121:packages/postgrest/lib/src/postgrest_builder.dart
if (method == null) {
  throw ArgumentError(
    'Missing table operation: select, insert, update or delete',
  );
}
```

This ensures that a valid HTTP method has been set through the builder pattern before execution.

### 3. Schema Header Configuration

```dart 126:132:packages/postgrest/lib/src/postgrest_builder.dart
if (_schema == null) {
  // skip
} else if ([METHOD_GET, METHOD_HEAD].contains(method)) {
  _headers['Accept-Profile'] = _schema!;
} else {
  _headers['Content-Profile'] = _schema!;
}
```

PostgREST uses schema headers to specify which database schema to operate on:

- `Accept-Profile`: Used for read operations (GET, HEAD) to specify which schema to read from
- `Content-Profile`: Used for write operations (POST, PUT, PATCH, DELETE) to specify which schema to write to

### 4. Content-Type Header

```dart 133:135:packages/postgrest/lib/src/postgrest_builder.dart
if (method != METHOD_GET && method != METHOD_HEAD) {
  _headers['Content-Type'] = 'application/json';
}
```

Sets the content type to JSON for all write operations, as PostgREST expects JSON payloads for data manipulation.

### 5. HTTP Request Execution

The method supports all standard HTTP methods used by PostgREST:

```dart 139:172:packages/postgrest/lib/src/postgrest_builder.dart
if (uppercaseMethod == METHOD_GET) {
  response = await (_httpClient?.get ?? http.get)(
    _url,
    headers: _headers,
  );
} else if (uppercaseMethod == METHOD_POST) {
  response = await (_httpClient?.post ?? http.post)(
    _url,
    headers: _headers,
    body: bodyStr,
  );
// ... other methods
```

- **GET**: Used for SELECT queries
- **POST**: Used for INSERT operations
- **PUT**: Used for UPSERT operations
- **PATCH**: Used for UPDATE operations
- **DELETE**: Used for DELETE operations
- **HEAD**: Used for existence checks or metadata requests

The method uses either a custom HTTP client (`_httpClient`) if provided, or falls back to the default `http` package client.

### 6. Response Parsing

After executing the HTTP request, the response is passed to `_parseResponse` for processing:

```dart 174:174:packages/postgrest/lib/src/postgrest_builder.dart
return _parseResponse(response, method);
```

## Response Processing (`_parseResponse`)

The `_parseResponse` method handles the HTTP response and converts it into the appropriate Dart types.

### Success Response Handling

For successful responses (status codes 200-299):

1. **Body Parsing**: Handles different content types and sizes:
   - Empty responses
   - CSV data
   - Plan text responses
   - Large JSON responses (uses isolate for performance)
   - Regular JSON responses

2. **Maybe Single Workaround**: A special handling for single-row requests that may return multiple rows, throwing an exception if more than one row is returned.

3. **Count Extraction**: Parses the `Content-Range` header to extract row counts.

4. **Type Conversion**: Applies converters and type transformations based on the generic type parameters.

5. **Return Value**: Returns either a `PostgrestResponse<T>` (when count is requested) or just the data.

### Error Response Handling

For error responses, it:

1. Attempts to parse JSON error responses into `PostgrestException` objects
2. Falls back to generic error creation for non-JSON responses
3. Handles special cases for HEAD requests
4. Logs errors and throws exceptions

## Integration with Builder Pattern

The `_execute` method is called internally by the various builder classes in the PostgREST client:

- `PostgrestQueryBuilder`: For SELECT queries
- `PostgrestRpcBuilder`: For stored procedure calls
- `PostgrestFilterBuilder`: For filtered queries
- `PostgrestTransformBuilder`: For data transformations

Each builder configures the method, URL, headers, and body before calling `_execute()`.

## Error Handling

The method uses a simple try-catch that rethrows all errors, allowing calling code to handle specific exceptions. This design allows for flexible error handling at higher levels while keeping the core execution logic clean.

## Logging

The method includes logging at the `finest` level for request details, which helps with debugging network issues during development.

## Performance Considerations

- Uses JSON isolates for large response parsing (>10KB)
- Supports custom HTTP clients for advanced use cases
- Minimal overhead in header processing and method routing
