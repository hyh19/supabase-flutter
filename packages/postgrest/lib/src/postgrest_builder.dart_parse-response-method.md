# Response Parsing Method in PostgrestBuilder

## Overview

The `_parseResponse` method is a core component of the PostgREST client responsible for processing HTTP responses from the PostgREST API. It handles both successful responses and error cases, performing content type detection, JSON parsing, data transformation, and type conversion.

## Method Signature

```dart 180:181:packages/postgrest/lib/src/postgrest_builder.dart
  /// Parse request response to json object if possible
  Future<T> _parseResponse(http.Response response, String method) async {
```

The method takes an HTTP response object and the HTTP method used, returning a generic type `T` that represents the final processed response data.

## Response Processing Flow

### Success Response Handling (Status Codes 200-299)

#### 1. Initial Body Parsing

```dart 182:205:packages/postgrest/lib/src/postgrest_builder.dart
    if (response.statusCode >= 200 && response.statusCode <= 299) {
      Object? body;
      int? count;

      if (response.request!.method != METHOD_HEAD) {
        if (response.bodyBytes.isEmpty) {
          body = null;
        } else if (response.request!.headers['Accept'] == 'text/csv') {
          body = response.body;
        } else if (_headers['Accept'] != null &&
            _headers['Accept']!.contains('application/vnd.pgrst.plan+text')) {
          body = response.body;
        } else {
          try {
            if ((response.contentLength ?? 0) > 10000 && _isolate != null) {
              body = await _isolate!.decode(response.body);
            } else {
              body = jsonDecode(response.body);
            }
          } on FormatException catch (_) {
            body = null;
          }
        }
      }
```

The method first checks if the response is successful. For non-HEAD requests, it parses the response body based on content type:

- **Empty body**: Sets `body` to `null`
- **CSV responses**: Keeps the raw string body
- **PostgREST plan responses**: Keeps the raw string body
- **JSON responses**: Uses JSON decoding, with performance optimization for large responses (>10KB) by using an isolate for parsing

#### 2. Single Row Handling (Maybe Single Workaround)

```dart 207:226:packages/postgrest/lib/src/postgrest_builder.dart
      // Workaround for https://github.com/supabase/supabase-flutter/issues/560
      if (_maybeSingle && method.toUpperCase() == 'GET' && body is List) {
        if (body.length > 1) {
          final exception = PostgrestException(
            // https://github.com/PostgREST/postgrest/blob/a867d79c42419af16c18c3fb019eba8df992626f/src/PostgREST/Error.hs#L553
            code: '406',
            details:
                'Results contain ${body.length} rows, application/vnd.pgrst.object+json requires 1 row',
            hint: null,
            message: 'JSON object requested, multiple (or no) rows returned',
          );

          _log.finest('$exception for request $_url');
          throw exception;
        } else if (body.length == 1) {
          body = body.first;
        } else {
          body = null;
        }
      }
```

This section implements a workaround for issue #560. When `_maybeSingle` is true and the response is a list, it enforces single-row semantics:

- **Multiple rows**: Throws a PostgrestException with code 406
- **Single row**: Extracts the first element from the list
- **No rows**: Sets body to null

#### 3. Content Range Parsing

```dart 228:233:packages/postgrest/lib/src/postgrest_builder.dart
      final contentRange = response.headers['content-range'];
      if (contentRange != null && contentRange.length > 1) {
        count = contentRange.split('/').last == '*'
            ? null
            : int.parse(contentRange.split('/').last);
      }
```

Parses the `Content-Range` header to extract the total count of records. The header format is typically `items start-end/total`, where:

- `*` indicates unknown total count
- A number indicates the actual total count

#### 4. Type Conversion

```dart 235:249:packages/postgrest/lib/src/postgrest_builder.dart
      body as dynamic;
      final S converted;

      if (R == PostgrestList) {
        body = PostgrestList.from(body);
      } else if (R == PostgrestMap) {
        body = PostgrestMap.from(body);
      } else if (R == _Nullable<PostgrestMap>) {
        if (body != null) {
          body = PostgrestMap.from(body);
        }
      } else if (R == int) {
        if (count != null) body = count;
      }
      body as R;
```

Performs type-specific conversions based on the generic type `R`:

- **PostgrestList**: Wraps the body in a PostgrestList
- **PostgrestMap**: Wraps the body in a PostgrestMap
- **Nullable PostgrestMap**: Conditionally wraps non-null body
- **int**: Uses the count value instead of the body

#### 5. Custom Conversion and Response Construction

```dart 251:264:packages/postgrest/lib/src/postgrest_builder.dart
      if (_converter != null) {
        converted = _converter!(body);
      } else {
        converted = body as S;
      }

      if (_count != null && method != METHOD_HEAD) {
        return PostgrestResponse<S>(
          data: converted,
          count: count!,
        ) as T;
      } else {
        return converted as T;
      }
```

Applies custom converters if available, then constructs the final response:

- **With count**: Returns a PostgrestResponse containing both data and count
- **Without count**: Returns the converted data directly

### Error Response Handling (Status Codes 300+)

#### 1. Error Parsing for Non-HEAD Requests

```dart 265:286:packages/postgrest/lib/src/postgrest_builder.dart
    } else {
      late PostgrestException error;
      if (response.request!.method != METHOD_HEAD) {
        try {
          final errorJson = jsonDecode(response.body) as Map<String, dynamic>;
          error = PostgrestException.fromJson(
            errorJson,
            message: response.body,
            code: response.statusCode,
            details: response.reasonPhrase,
          );

          if (_maybeSingle) {
            return _handleMaybeSingleError(response, error);
          }
        } catch (_) {
          error = PostgrestException(
            message: response.body,
            code: '${response.statusCode}',
            details: response.reasonPhrase,
          );
        }
      }
```

For error responses, attempts to parse JSON error details. Falls back to basic exception creation if JSON parsing fails. Special handling for maybe-single errors is delegated to `_handleMaybeSingleError`.

#### 2. Error Handling for HEAD Requests

```dart 287:294:packages/postgrest/lib/src/postgrest_builder.dart
      } else {
        error = PostgrestException(
          code: '${response.statusCode}',
          message: response.body,
          details: 'Error in Postgrest response for method HEAD',
          hint: response.reasonPhrase,
        );
      }
```

HEAD requests don't have a response body, so a basic exception is created with status information.

#### 3. Error Logging and Throwing

```dart 296:300:packages/postgrest/lib/src/postgrest_builder.dart
      _log.finest('$error from request: $_url');
      _log.fine('$error from request');

      throw error;
    }
```

Logs the error at different levels and throws the PostgrestException.

## Key Design Patterns

### Performance Optimization

- Uses isolates for large JSON parsing (>10KB) to avoid blocking the main thread
- Conditional parsing based on content type to minimize unnecessary processing

### Error Handling

- Comprehensive error parsing with fallbacks
- Structured exception types with detailed information
- Special handling for edge cases like single-row requests

### Type Safety

- Generic type parameters (`T`, `S`, `R`) ensure type safety throughout the conversion process
- Runtime type checking and conversion for PostgREST-specific types

### Content Negotiation

- Respects Accept headers for different response formats (CSV, plan text, JSON)
- Handles various PostgREST response types appropriately

## Integration Points

This method integrates with several other components:

- **Isolate for JSON parsing**: Uses `yet_another_json_isolate` package for performance
- **Custom converters**: Applies user-defined data transformation functions
- **PostgREST types**: Converts to `PostgrestList`, `PostgrestMap` wrapper types
- **Logging**: Uses the internal logger for debugging and error tracking
- **Exception handling**: Creates structured `PostgrestException` instances

## Edge Cases Handled

1. **Empty responses**: Gracefully handles empty response bodies
2. **Malformed JSON**: Falls back to null body instead of crashing
3. **Single-row enforcement**: Implements strict single-row semantics when requested
4. **Content range parsing**: Handles both known and unknown total counts
5. **HEAD request errors**: Special handling for requests without response bodies
6. **Type conversion failures**: Uses dynamic casting with proper error handling

This method serves as the central response processing hub, ensuring consistent handling of all PostgREST API responses while maintaining performance and type safety.
