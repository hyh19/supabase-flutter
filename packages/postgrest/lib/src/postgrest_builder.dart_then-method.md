# The `then` Method in PostgREST Builder

## Overview

The `then` method is an override of Dart's `Future.then()` method, providing enhanced error handling and type safety for PostgREST query execution. This method serves as the bridge between the fluent query builder API and the actual HTTP request execution.

## Method Signature

```dart 380:384:packages/postgrest/lib/src/postgrest_builder.dart
  @override
  Future<U> then<U>(
    FutureOr<U> Function(T value) onValue, {
    Function? onError,
  }) async {
```

## Parameters

- **`onValue`**: A required callback function that transforms the successful response of type `T` into a result of type `U`
- **`onError`**: An optional error handler function that can accept either:
  - `Function(Object)` - receives only the error object
  - `Function(Object, StackTrace)` - receives both error and stack trace

## Error Handler Validation

The method performs strict validation of the `onError` callback to ensure type safety:

```dart 385:394:packages/postgrest/lib/src/postgrest_builder.dart
    if (onError != null &&
        onError is! Function(Object, StackTrace) &&
        onError is! Function(Object)) {
      throw ArgumentError.value(
        onError,
        "onError",
        "Error handler must accept one Object or one Object and a StackTrace"
            " as arguments, and return a value of the returned future's type",
      );
    }
```

This validation prevents runtime errors by ensuring the error handler has one of the two accepted signatures. The error message provides clear guidance on the expected function signatures.

## Execution Flow

### Successful Execution

```dart 396:398:packages/postgrest/lib/src/postgrest_builder.dart
    try {
      final response = await _execute();
      return onValue(response);
    }
```

When the query executes successfully:

1. Calls the private `_execute()` method to perform the actual HTTP request
2. Passes the response (of type `T`) to the `onValue` callback
3. Returns the transformed result (of type `U`)

### Error Handling

```dart 399:425:packages/postgrest/lib/src/postgrest_builder.dart
    } catch (error, stack) {
      final FutureOr<U> result;
      if (onError != null) {
        if (onError is Function(Object, StackTrace)) {
          result = onError(error, stack);
        } else if (onError is Function(Object)) {
          result = onError(error);
        } else {
          throw ArgumentError.value(
            onError,
            "onError",
            "Error handler must accept one Object or one Object and a StackTrace"
                " as arguments, and return a value of the returned future's type",
          );
        }
        // Give better error messages if the result is not a valid
        // FutureOr<R>.
        try {
          return result;
        } on TypeError {
          throw ArgumentError(
              "The error handler of Future.then"
                  " must return a value of the returned future's type",
              "onError");
        }
      }
      rethrow;
    }
```

The error handling logic:

1. **Catches any exception** thrown during execution with both error and stack trace
2. **Checks if error handler is provided** - if not, rethrows the error
3. **Determines error handler signature** and calls it appropriately:
   - For handlers accepting `(Object, StackTrace)`, passes both parameters
   - For handlers accepting `(Object)`, passes only the error
4. **Validates return type** - if the error handler returns an incompatible type, throws a `TypeError` with a descriptive message
5. **Returns the error handler result** if validation passes

## Design Rationale

### Type Safety Emphasis

This implementation goes beyond standard `Future.then()` by adding runtime type checking for error handlers. This prevents subtle bugs where error handlers might return incorrect types, leading to confusing runtime errors later in the execution chain.

### Enhanced Error Messages

The method provides detailed error messages that help developers understand:

- What signatures are acceptable for error handlers
- What types the error handler must return
- Where the validation failed

### Backward Compatibility

The method maintains full compatibility with standard Dart `Future.then()` usage while adding the PostgREST-specific execution logic.

## Usage Example

```dart
// Basic usage with success transformation
final result = await supabase
    .from('users')
    .select('name, email')
    .eq('active', true)
    .then((response) => response.data as List<dynamic>);

// Usage with error handling
final result = await supabase
    .from('users')
    .select('name, email')
    .then(
      (response) => response.data,
      onError: (error, stackTrace) {
        print('Query failed: $error');
        return []; // Return empty list on error
      },
    );
```

## Integration with Builder Pattern

This method completes the fluent interface of the PostgREST query builder. The builder methods (`select()`, `eq()`, `limit()`, etc.) construct the query, and `then()` executes it and handles the response. This allows for a clean, chainable API similar to:

```dart
supabase.from('table').select('columns').where('condition').then((result) => /* handle result */);
```

The method's generic type parameters `<T, U>` allow for flexible response transformation while maintaining type safety throughout the query execution pipeline.
