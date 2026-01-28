# PostgrestBuilder Class

## Overview

The `PostgrestBuilder` class is the core foundation of the PostgREST Dart client, serving as a base builder for constructing and executing database queries against a PostgREST API. It implements the `Future<T>` interface, allowing it to be used directly as an asynchronous operation that can be awaited.

```dart 30:36:packages/postgrest/lib/src/postgrest_builder.dart
/// The base builder class.
///
/// [T] for the overall return type, so `PostgrestResponse<S>` or [S]
///
/// When using [_converter], [S] is the input and [R] is the output
/// Otherwise [S] and [R] are the same
@immutable
class PostgrestBuilder<T, S, R> implements Future<T> {
```

## Generic Type Parameters

The class uses three generic type parameters that work together to provide flexible type handling:

- **T**: The overall return type. This is either `PostgrestResponse<S>` (when count is requested) or `S` (the converted data type).
- **S**: The source data type before conversion.
- **R**: The raw data type from the API response. When a converter is used, `S` is the input to the converter and `R` is the output. When no converter is used, `S` and `R` are the same type.

## Key Properties

The builder maintains several important configuration properties:

```dart 38:48:packages/postgrest/lib/src/postgrest_builder.dart
  final Object? _body;
  final Headers _headers;
  final bool _maybeSingle;
  final String? _method;
  final String? _schema;
  final Uri _url;
  final PostgrestConverter<S, R>? _converter;
  final Client? _httpClient;
  final YAJsonIsolate? _isolate;
  final CountOption? _count;
  final _log = Logger('supabase.postgrest');
```

- `_body`: The request body data for POST/PUT/PATCH operations
- `_headers`: HTTP headers including authentication and content-type
- `_maybeSingle`: Flag for single-row queries that may return zero or one result
- `_method`: HTTP method (GET, POST, PUT, PATCH, DELETE, HEAD)
- `_schema`: Database schema name for multi-schema databases
- `_url`: The complete URI for the API request
- `_converter`: Optional data converter for transforming API responses
- `_httpClient`: Custom HTTP client (defaults to standard http package)
- `_isolate`: JSON parsing isolate for handling large responses efficiently
- `_count`: Count option for getting row counts alongside data

## Constructor and CopyWith Pattern

The constructor initializes all properties with sensible defaults:

```dart 50:70:packages/postgrest/lib/src/postgrest_builder.dart
  PostgrestBuilder({
    required Uri url,
    required Headers headers,
    String? schema,
    String? method,
    Object? body,
    Client? httpClient,
    YAJsonIsolate? isolate,
    CountOption? count,
    bool maybeSingle = false,
    PostgrestConverter<S, R>? converter,
  })  : _maybeSingle = maybeSingle,
        _method = method,
        _converter = converter,
        _schema = schema,
        _url = url,
        _headers = headers,
        _httpClient = httpClient,
        _isolate = isolate,
        _count = count,
        _body = body;
```

The `_copyWith` method enables the builder pattern by creating modified copies:

```dart 72:96:packages/postgrest/lib/src/postgrest_builder.dart
  PostgrestBuilder<T, S, R> _copyWith({
    Uri? url,
    Headers? headers,
    String? schema,
    String? method,
    Object? body,
    Client? httpClient,
    YAJsonIsolate? isolate,
    CountOption? count,
    bool? maybeSingle,
    PostgrestConverter<S, R>? converter,
  }) {
    return PostgrestBuilder<T, S, R>(
      url: url ?? _url,
      headers: headers ?? _headers,
      schema: schema ?? _schema,
      method: method ?? _method,
      body: body ?? _body,
      httpClient: httpClient ?? _httpClient,
      isolate: isolate ?? _isolate,
      count: count ?? _count,
      maybeSingle: maybeSingle ?? _maybeSingle,
      converter: converter ?? _converter,
    );
  }
```

## Core Execution Logic

### Request Execution

The `_execute()` method orchestrates the entire HTTP request process:

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

Key aspects of the execution:

1. **Count Header**: Automatically adds count preference headers when count is requested
2. **Schema Handling**: Sets appropriate profile headers for multi-schema support
3. **Content-Type**: Automatically sets JSON content-type for non-GET/HEAD requests
4. **HTTP Methods**: Supports all standard REST operations
5. **Logging**: Logs request details at finest level for debugging

### Response Parsing

The `_parseResponse()` method handles complex response parsing with multiple data type conversions:

```dart 180:301:packages/postgrest/lib/src/postgrest_builder.dart
  /// Parse request response to json object if possible
  Future<T> _parseResponse(http.Response response, String method) async {
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

      final contentRange = response.headers['content-range'];
      if (contentRange != null && contentRange.length > 1) {
        count = contentRange.split('/').last == '*'
            ? null
            : int.parse(contentRange.split('/').last);
      }

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
      } else {
        error = PostgrestException(
          code: '${response.statusCode}',
          message: response.body,
          details: 'Error in Postgrest response for method HEAD',
          hint: response.reasonPhrase,
        );
      }

      _log.finest('$error from request: $_url');
      _log.fine('$error from request');

      throw error;
    }
  }
```

This method handles:

1. **Multiple Content Types**: JSON, CSV, and execution plans
2. **Performance Optimization**: Uses isolates for large JSON responses (>10KB)
3. **Maybe Single Logic**: Special handling for queries expecting zero or one row
4. **Count Parsing**: Extracts row counts from Content-Range headers
5. **Type Conversion**: Automatic conversion to PostgrestList/PostgrestMap types
6. **Converter Application**: Applies custom data converters when provided
7. **Response Wrapping**: Wraps data in PostgrestResponse when count is requested

## Maybe Single Error Handling

The `_handleMaybeSingleError` method provides special error handling for queries that expect zero or one result:

```dart 306:329:packages/postgrest/lib/src/postgrest_builder.dart
  /// When [_maybeSingle] is true, check whether error details contain
  /// 'Results contain 0 rows' then
  /// return PostgrestResponse with null data
  T _handleMaybeSingleError(
    http.Response response,
    PostgrestException error,
  ) {
    if (error.details is String &&
        error.details.toString().contains('Results contain 0 rows')) {
      if (_count != null && response.request!.method != METHOD_HEAD) {
        if (_converter != null) {
          return PostgrestResponse<S>(data: _converter!(null as R), count: 0)
              as T;
        } else {
          return null as T;
        }
      } else {
        if (_converter != null) {
          return _converter!(null as R) as T;
        } else {
          return null as T;
        }
      }
    } else {
      throw error;
    }
  }
```

This converts "no rows found" errors into successful null responses for single-row queries.

## Query Parameter Utilities

The class provides utilities for manipulating URL query parameters:

### Append Search Params

```dart 335:340:packages/postgrest/lib/src/postgrest_builder.dart
  /// Get new Uri with updated queryParams
  /// Uses lists to allow multiple values for the same key
  ///
  /// [url] may be used to update based on a different url than the current one
  Uri appendSearchParams(String key, String value, [Uri? url]) {
    final searchParams =
        Map<String, dynamic>.from((url ?? _url).queryParametersAll);
    searchParams[key] = [...searchParams[key] ?? [], value];
    return (url ?? _url).replace(queryParameters: searchParams);
  }
```

Allows adding multiple values for the same query parameter key.

### Override Search Params

```dart 345:349:packages/postgrest/lib/src/postgrest_builder.dart
  /// Get new Uri with overridden queryParams
  ///
  /// [url] may be used to update based on a different url than the current one
  Uri overrideSearchParams(String key, String value) {
    final searchParams = Map<String, dynamic>.from(_url.queryParametersAll);
    searchParams[key] = value;
    return _url.replace(queryParameters: searchParams);
  }
```

Replaces existing values for a query parameter key.

### Filter Array Cleaning

```dart 352:358:packages/postgrest/lib/src/postgrest_builder.dart
  /// Convert list filter to query params string
  String _cleanFilterArray(List filter) {
    if (filter.every((element) => element is num)) {
      return filter.map((s) => '$s').join(',');
    } else {
      return filter.map((s) => '"$s"').join(',');
    }
  }
```

Formats filter arrays appropriately, quoting strings but not numbers.

## Future Interface Implementation

Since `PostgrestBuilder` implements `Future<T>`, it provides all the standard Future methods:

```dart 360:373:packages/postgrest/lib/src/postgrest_builder.dart
  @override
  Stream<T> asStream() {
    final controller = StreamController<T>.broadcast();

    then((value) {
      controller.add(value);
    }).catchError((Object error, StackTrace stack) {
      controller.addError(error, stack);
    }).whenComplete(() {
      controller.close();
    });

    return controller.stream;
  }
```

The implementation delegates to the `_execute()` method through the `then()` method, providing the full Future API including `catchError`, `timeout`, `whenComplete`, and `asStream`.

## Key Design Patterns

### Builder Pattern

The class uses immutable builders with `_copyWith` to create modified instances, enabling fluent query construction like `from().select().eq().limit()`.

### Type Safety with Generics

Complex generic type system (`<T, S, R>`) provides compile-time type safety while allowing runtime flexibility with converters.

### Performance Optimization

Uses isolates for parsing large JSON responses to prevent UI blocking on mobile platforms.

### Error Recovery

Special handling for "maybe single" queries converts certain errors into successful null responses.

### Extensibility

The base class is extended by specialized builders (filter, query, transform, etc.) through part files, allowing modular feature addition.

This architecture provides a robust, type-safe, and performant foundation for the PostgREST client's query building capabilities.
