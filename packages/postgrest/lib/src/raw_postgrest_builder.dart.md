# RawPostgrestBuilder Class

## Overview

The `RawPostgrestBuilder` class is a specialized wrapper around the `PostgrestBuilder` class that provides enhanced type flexibility for response conversion. It serves as an intermediate builder that allows changing the generic type parameters while maintaining the same underlying query building functionality.

## Class Declaration

```dart 4:4:packages/postgrest/lib/src/raw_postgrest_builder.dart
class RawPostgrestBuilder<T, S, R> extends PostgrestBuilder<T, S, R> {
```

The class uses three generic type parameters:

- `T`: The input data type
- `S`: The transformed data type
- `R`: The raw response type

This design allows for flexible type transformations during the query building process.

## Constructor

```dart 5:17:packages/postgrest/lib/src/raw_postgrest_builder.dart
  RawPostgrestBuilder(PostgrestBuilder<T, S, R> builder)
      : super(
          url: builder._url,
          method: builder._method,
          headers: builder._headers,
          schema: builder._schema,
          body: builder._body,
          httpClient: builder._httpClient,
          count: builder._count,
          isolate: builder._isolate,
          maybeSingle: builder._maybeSingle,
          converter: builder._converter,
        );
```

The constructor takes an existing `PostgrestBuilder` instance and copies all its internal state. This pattern allows wrapping existing builders while preserving their configuration.

## Type-Safe Copy Method

```dart 20:42:packages/postgrest/lib/src/raw_postgrest_builder.dart
  RawPostgrestBuilder<O, P, Q> _copyWithType<O, P, Q>({
    Uri? url,
    Headers? headers,
    String? schema,
    String? method,
    Object? body,
    Client? httpClient,
    YAJsonIsolate? isolate,
    CountOption? count,
    bool? maybeSingle,
  }) {
    return RawPostgrestBuilder<O, P, Q>(PostgrestBuilder(
      url: url ?? _url,
      headers: headers ?? _headers,
      schema: schema ?? _schema,
      method: method ?? _method,
      body: body ?? _body,
      httpClient: httpClient ?? _httpClient,
      isolate: isolate ?? _isolate,
      count: count ?? _count,
      maybeSingle: maybeSingle ?? _maybeSingle,
    ));
  }
```

This private method creates a new `RawPostgrestBuilder` instance with potentially different generic types (`O`, `P`, `Q`). Unlike the base class's `_copyWith` method, this version omits the converter parameter, allowing for type changes without converter constraints.

## Header Management

```dart 45:49:packages/postgrest/lib/src/raw_postgrest_builder.dart
  @override
  RawPostgrestBuilder<T, S, R> setHeader(String key, String value) {
    return PostgrestFilterBuilder(
      _copyWithType(headers: {..._headers, key: value}),
    );
  }
```

The `setHeader` method overrides the base implementation to return a `PostgrestFilterBuilder` instead of maintaining the `RawPostgrestBuilder` type. This allows transitioning to filter operations after setting headers.

## Response Conversion

```dart 61:75:packages/postgrest/lib/src/raw_postgrest_builder.dart
  PostgrestBuilder<U, U, R> withConverter<U>(
      PostgrestConverter<U, R> converter) {
    return PostgrestBuilder(
      url: _url,
      headers: _headers,
      schema: _schema,
      method: _method,
      body: _body,
      isolate: _isolate,
      httpClient: _httpClient,
      count: _count,
      maybeSingle: _maybeSingle,
      converter: converter,
    );
  }
```

The `withConverter` method is the key feature of this class. It allows converting raw responses into type-safe objects using a `PostgrestConverter`. The method returns a `PostgrestBuilder<U, U, R>` where:

- `U` is the converted type
- Both input and transformed types are the same (`U, U`) since the conversion happens at the response level
- `R` remains the raw response type

## Usage Example

```dart
// Example usage pattern
List<User> users = await postgrest
    .from('users')
    .select()
    .withConverter(
      (users) => users.map(User.fromJson).toList(),
    );
```

## Design Purpose

The `RawPostgrestBuilder` serves as a bridge between raw query building and type-safe response handling. It allows the PostgREST client to:

1. Maintain query building capabilities while preparing for type conversion
2. Change generic type parameters without losing query state
3. Provide a clean transition to converted response builders
4. Support both raw responses and type-safe object conversion

This design enables flexible API usage where developers can choose between raw JSON responses or strongly-typed Dart objects based on their needs.
