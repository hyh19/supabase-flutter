# PostgREST Types and Exceptions

This file defines the core type definitions, exceptions, and enums used throughout the PostgREST client library. It provides the foundational types for handling API responses, error management, and query configuration options.

## Type Definitions

```dart 1:6:packages/postgrest/lib/src/types.dart
typedef Headers = Map<String, String>;
typedef PostgrestConverter<S, T> = S Function(T data);
typedef PostgrestList = List<PostgrestMap>;
typedef PostgrestMap = Map<String, dynamic>;
typedef PostgrestListResponse = PostgrestResponse<PostgrestList>;
typedef PostgrestMapResponse = PostgrestResponse<PostgrestMap>;
```

The file begins with several type aliases that simplify working with PostgREST data structures:

- **`Headers`**: A convenience type for HTTP headers represented as a string-to-string map
- **`PostgrestConverter<S, T>`**: A generic converter function type that transforms data from type `T` to type `S`. Useful for custom data transformation logic
- **`PostgrestList`**: A list of `PostgrestMap` objects, representing multiple records from a database query
- **`PostgrestMap`**: A dynamic map representing a single database record, where keys are column names and values are the corresponding data
- **`PostgrestListResponse`** and **`PostgrestMapResponse`**: Predefined response types for common query patterns (multiple records vs single record)

## PostgrestException Class

```dart 8:49:packages/postgrest/lib/src/types.dart
/// A Postgrest response exception
class PostgrestException implements Exception {
  final String message;
  final String? code;
  final Object? details;
  final String? hint;

  const PostgrestException({
    required this.message,
    this.code,
    this.details,
    this.hint,
  });

  factory PostgrestException.fromJson(
    Map<String, dynamic> json, {
    String? message,
    int? code,
    String? details,
  }) {
    return PostgrestException(
      message: (json['message'] ?? message) as String,
      code: (json['code'] ?? '$code') as String?,
      details: (json['details'] ?? details),
      hint: json['hint'] as String?,
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'message': message,
      'code': code,
      'details': details,
      'hint': hint,
    };
  }

  @override
  String toString() {
    return 'PostgrestException(message: $message, code: $code, details: $details, hint: $hint)';
  }
}
```

The `PostgrestException` class provides comprehensive error handling for PostgREST operations:

**Properties:**

- **`message`**: Required human-readable error description
- **`code`**: Optional error code (typically a string representation of HTTP status codes)
- **`details`**: Optional additional error information (can be any object type)
- **`hint`**: Optional suggestion for resolving the error

**Key Features:**

- **JSON Serialization**: Both `fromJson()` factory and `toJson()` methods for converting between JSON and exception objects
- **Flexible Construction**: The `fromJson()` factory allows overriding values with external parameters
- **Standard Exception Interface**: Implements Dart's `Exception` interface
- **Debug-Friendly**: Comprehensive `toString()` implementation for debugging

This exception class handles various error scenarios including network failures, authentication issues, database constraints, and malformed queries.

## PostgrestResponse Class

```dart 51:77:packages/postgrest/lib/src/types.dart
/// A Postgrest response
class PostgrestResponse<T> {
  const PostgrestResponse({
    required this.data,
    required this.count,
  });

  final T data;

  final int count;

  factory PostgrestResponse.fromJson(Map<String, dynamic> json) =>
      PostgrestResponse<T>(
        data: json['data'] as T,
        count: json['count'] as int,
      );

  Map<String, dynamic> toJson() => {
        'data': data,
        'count': count,
      };

  @override
  String toString() {
    return 'PostgrestResponse(data: $data, count: $count)';
  }
}
```

The `PostgrestResponse<T>` class is a generic wrapper for all PostgREST API responses:

**Generic Type Support:**

- The type parameter `T` allows for type-safe responses
- Common instantiations include `PostgrestResponse<PostgrestList>` for multiple records and `PostgrestResponse<PostgrestMap>` for single records

**Properties:**

- **`data`**: The actual response data of type `T`
- **`count`**: The number of records affected or returned (useful for pagination and result tracking)

**Serialization Support:**

- `fromJson()` factory for creating responses from JSON data
- `toJson()` method for converting responses back to JSON

## Query Configuration Enums

### CountOption Enum

```dart 79:89:packages/postgrest/lib/src/types.dart
/// Returns count as part of the response when specified.
enum CountOption {
  /// Exact but slow count algorithm. Performs a `COUNT(*)` under the hood.
  exact,

  /// Approximated but fast count algorithm. Uses the Postgres statistics under the hood.
  planned,

  /// Uses exact count for low numbers and planned count for high numbers.
  estimated,
}
```

The `CountOption` enum provides different strategies for counting query results:

- **`exact`**: Performs a full `COUNT(*)` operation for precise results (slower but accurate)
- **`planned`**: Uses PostgreSQL's statistics for approximate counts (faster but less precise)
- **`estimated`**: Hybrid approach that uses exact counts for small result sets and planned counts for larger ones

This enum allows developers to balance between accuracy and performance based on their specific use cases.

### Deprecated ReturningOption Enum

```dart 91:98:packages/postgrest/lib/src/types.dart
// coverage:ignore-[start]
/// Returns count as part of the response when specified.
@Deprecated('Not used anywhere. Will be removed in the next major version.')
enum ReturningOption {
  minimal,
  representation,
}
// coverage:ignore-[end]
```

The `ReturningOption` enum is marked as deprecated and scheduled for removal. It was previously used for controlling the amount of data returned in responses but is no longer utilized in the codebase. The coverage ignore comments indicate this code is excluded from test coverage requirements.

### TextSearchType Enum

```dart 100:111:packages/postgrest/lib/src/types.dart
/// The type of tsquery conversion to use on [query].
enum TextSearchType {
  /// Uses PostgreSQL's plainto_tsquery function.
  plain,

  /// Uses PostgreSQL's phraseto_tsquery function.
  phrase,

  /// Uses PostgreSQL's websearch_to_tsquery function.
  /// This function will never raise syntax errors, which makes it possible to use raw user-supplied input for search, and can be used with advanced operators.
  websearch,
}
```

The `TextSearchType` enum controls how text search queries are processed in PostgreSQL:

- **`plain`**: Uses `plainto_tsquery()` - converts natural language text to a tsquery, normalizing words and removing stopwords
- **`phrase`**: Uses `phraseto_tsquery()` - preserves word order and proximity for phrase searches
- **`websearch`**: Uses `websearch_to_tsquery()` - accepts raw user input with advanced operators (AND, OR, quotes, etc.) without throwing syntax errors

This enum enables different text search behaviors ranging from simple keyword matching to complex boolean search expressions.

## Usage Patterns

These types work together to provide a robust foundation for PostgREST client operations:

1. **Error Handling**: `PostgrestException` provides consistent error reporting across all API calls
2. **Type Safety**: Generic `PostgrestResponse<T>` ensures type-safe data handling
3. **Query Configuration**: Enums like `CountOption` and `TextSearchType` allow fine-tuned control over database operations
4. **Data Transformation**: `PostgrestConverter` enables custom data mapping and transformation logic

The type definitions promote code reusability and maintainability while providing clear contracts for API interactions.
