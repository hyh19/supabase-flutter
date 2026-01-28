# The `_getPostgrestData()` Method

## Overview

The `_getPostgrestData()` method is a private asynchronous method in the `SupabaseStreamBuilder` class that fetches initial data from PostgREST and prepares it for streaming. This method is crucial in the stream builder architecture as it provides the baseline data that gets combined with realtime updates.

## Purpose

This method serves as the bridge between the initial PostgREST query and the realtime streaming functionality. It:

1. Constructs and executes a PostgREST query with all configured filters, ordering, and limits
2. Processes the query results into the stream format
3. Updates the internal stream data cache
4. Emits the data to stream subscribers
5. Handles errors gracefully by cleaning up connections

## Method Signature

```dart 238:286:packages/supabase/lib/src/supabase_stream_builder.dart
Future<void> _getPostgrestData() async {
  PostgrestFilterBuilder<PostgrestList> query = _queryBuilder.select();
  if (_streamFilter != null) {
    switch (_streamFilter!.type) {
      case PostgresChangeFilterType.eq:
        query = query.eq(_streamFilter!.column, _streamFilter!.value);
        break;
      case PostgresChangeFilterType.neq:
        query = query.neq(_streamFilter!.column, _streamFilter!.value);
        break;
      case PostgresChangeFilterType.lt:
        query = query.lt(_streamFilter!.column, _streamFilter!.value);
        break;
      case PostgresChangeFilterType.lte:
        query = query.lte(_streamFilter!.column, _streamFilter!.value);
        break;
      case PostgresChangeFilterType.gt:
        query = query.gt(_streamFilter!.column, _streamFilter!.value);
        break;
      case PostgresChangeFilterType.gte:
        query = query.gte(_streamFilter!.column, _streamFilter!.value);
        break;
      case PostgresChangeFilterType.inFilter:
        query = query.inFilter(_streamFilter!.column, _streamFilter!.value);
        break;
    }
  }
  PostgrestTransformBuilder<PostgrestList>? transformQuery;
  if (_orderBy != null) {
    transformQuery =
        query.order(_orderBy!.column, ascending: _orderBy!.ascending);
  }
  if (_limit != null) {
    transformQuery = (transformQuery ?? query).limit(_limit!);
  }

  try {
    final data = await (transformQuery ?? query);
    final rows = SupabaseStreamEvent.from(data);
    _streamData = rows;
    _addStream();
  } catch (error, stackTrace) {
    _addException(error, stackTrace);
    // In case the postgrest call fails, there is no need to keep the
    // realtime connection open
    _channel?.unsubscribe();
    _streamController?.close();
  }
}
```

## Code Flow

### 1. Query Initialization

```dart 239:239:packages/supabase/lib/src/supabase_stream_builder.dart
PostgrestFilterBuilder<PostgrestList> query = _queryBuilder.select();
```

Starts with a basic `SELECT` query using the configured `PostgrestQueryBuilder`. This creates the foundation query that will be progressively modified with filters, ordering, and limits.

### 2. Filter Application

```dart 240:263:packages/supabase/lib/src/supabase_stream_builder.dart
if (_streamFilter != null) {
  switch (_streamFilter!.type) {
    case PostgresChangeFilterType.eq:
      query = query.eq(_streamFilter!.column, _streamFilter!.value);
      break;
    case PostgresChangeFilterType.neq:
      query = query.neq(_streamFilter!.column, _streamFilter!.value);
      break;
    case PostgresChangeFilterType.lt:
      query = query.lt(_streamFilter!.column, _streamFilter!.value);
      break;
    case PostgresChangeFilterType.lte:
      query = query.lte(_streamFilter!.column, _streamFilter!.value);
      break;
    case PostgresChangeFilterType.gt:
      query = query.gt(_streamFilter!.column, _streamFilter!.value);
      break;
    case PostgresChangeFilterType.gte:
      query = query.gte(_streamFilter!.column, _streamFilter!.value);
      break;
    case PostgresChangeFilterType.inFilter:
      query = query.inFilter(_streamFilter!.column, _streamFilter!.value);
      break;
  }
}
```

Applies the configured stream filter to the query. The filter types correspond to standard SQL comparison operators:

- `eq`: equals (`=`)
- `neq`: not equals (`!=` or `<>`)
- `lt`: less than (`<`)
- `lte`: less than or equal (`<=`)
- `gt`: greater than (`>`)
- `gte`: greater than or equal (`>=`)
- `inFilter`: IN clause for multiple values

Only one filter can be applied at a time, as enforced by the `SupabaseStreamFilterBuilder` methods.

### 3. Transform Operations

```dart 265:272:packages/supabase/lib/src/supabase_stream_builder.dart
PostgrestTransformBuilder<PostgrestList>? transformQuery;
if (_orderBy != null) {
  transformQuery =
      query.order(_orderBy!.column, ascending: _orderBy!.ascending);
}
if (_limit != null) {
  transformQuery = (transformQuery ?? query).limit(_limit!);
}
```

Applies ordering and limiting transformations to the query:

- **Ordering**: Sorts results by the specified column in ascending or descending order
- **Limiting**: Restricts the number of returned records

The `transformQuery` variable accumulates these transformations, ensuring they're applied in the correct order.

### 4. Query Execution

```dart 274:278:packages/supabase/lib/src/supabase_stream_builder.dart
try {
  final data = await (transformQuery ?? query);
  final rows = SupabaseStreamEvent.from(data);
  _streamData = rows;
  _addStream();
}
```

Executes the constructed query and processes the results:

1. **Query Execution**: Awaits the PostgREST query execution
2. **Data Conversion**: Converts the raw PostgREST response to `SupabaseStreamEvent` format (a list of maps)
3. **Stream Update**: Updates the internal `_streamData` cache
4. **Emission**: Calls `_addStream()` to emit the data to stream subscribers

### 5. Error Handling

```dart 279:285:packages/supabase/lib/src/supabase_stream_builder.dart
catch (error, stackTrace) {
  _addException(error, stackTrace);
  // In case the postgrest call fails, there is no need to keep the
  // realtime connection open
  _channel?.unsubscribe();
  _streamController?.close();
}
```

Handles any errors that occur during the PostgREST query:

- **Exception Emission**: Adds the error to the stream for subscribers to handle
- **Resource Cleanup**: Unsubscribes from the realtime channel and closes the stream controller, preventing unnecessary resource consumption when the initial data fetch fails

## Integration with Realtime Updates

This method is called in two scenarios within the `SupabaseStreamBuilder`:

1. **Initial Load**: Called at the end of `_getStreamData()` to fetch the baseline data before setting up realtime subscriptions
2. **Reconnection**: Called when the realtime connection is re-established after a disconnect (when `_wasSubscribed` is true) to reload all data

The method ensures that the stream always starts with the most current data from PostgREST, which then gets updated in real-time as changes occur in the database.

## Performance Considerations

- The method is asynchronous to avoid blocking the main thread
- Error handling ensures resources are properly cleaned up on failure
- The query construction is efficient, building upon the existing PostgREST query builder pattern
- Only fetches the data once during initialization, relying on realtime updates thereafter

## Thread Safety

Since this method modifies the `_streamData` field and calls `_addStream()`, it should be called from the same context where the stream controller was created to ensure thread safety with the reactive stream operations.
