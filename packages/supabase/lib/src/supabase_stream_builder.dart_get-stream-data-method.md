# `_getStreamData()` Method Explanation

The `_getStreamData()` method is the core implementation of real-time data streaming in Supabase Flutter. It establishes a WebSocket connection through Supabase's realtime service and synchronizes data changes with an in-memory cache, providing a reactive stream of database changes.

## Overview

This method serves as the bridge between PostgREST (for initial data fetching) and Supabase Realtime (for live updates). It handles the complete lifecycle of setting up real-time subscriptions, processing database change events, and maintaining data consistency between the initial query results and subsequent real-time updates.

```dart 158:236:packages/supabase/lib/src/supabase_stream_builder.dart
Future<void> _getStreamData() async {
  final currentStreamFilter = _streamFilter;
  _streamData = [];
  PostgresChangeFilter? realtimeFilter;
  if (currentStreamFilter != null) {
    realtimeFilter = PostgresChangeFilter(
      type: currentStreamFilter.type,
      column: currentStreamFilter.column,
      value: currentStreamFilter.value,
    );
  }

  _channel = _realtimeClient.channel(_realtimeTopic);

  _channel!
      .onPostgresChanges(
          event: PostgresChangeEvent.all,
          schema: _schema,
          table: _table,
          filter: realtimeFilter,
          callback: (payload) {
            switch (payload.eventType) {
              case PostgresChangeEvent.insert:
                final newRecord = payload.newRecord;
                _streamData.add(newRecord);
                _addStream();
                break;
              case PostgresChangeEvent.update:
                final updatedIndex = _streamData.indexWhere(
                  (element) =>
                      _isTargetRecord(record: element, payload: payload),
                );

                final updatedRecord = payload.newRecord;
                if (updatedIndex >= 0) {
                  _streamData[updatedIndex] = updatedRecord;
                } else {
                  _streamData.add(updatedRecord);
                }
                _addStream();
                break;
              case PostgresChangeEvent.delete:
                final deletedIndex = _streamData.indexWhere(
                  (element) =>
                      _isTargetRecord(record: element, payload: payload),
                );
                if (deletedIndex >= 0) {
                  /// Delete the data from in memory cache if it was found
                  _streamData.removeAt(deletedIndex);
                  _addStream();
                }
                break;
              default:
                break;
            }
          })
      .subscribe((status, [error]) {
    switch (status) {
      case RealtimeSubscribeStatus.subscribed:
        // Reload all data after a reconnect from postgrest
        // First data from postgrest gets loaded before the realtime connect
        if (_wasSubscribed) {
          _getPostgrestData();
        }
        _wasSubscribed = true;
        break;
      case RealtimeSubscribeStatus.closed:
        _streamController?.close();
        break;
      case RealtimeSubscribeStatus.timedOut:
        _addException(RealtimeSubscribeException(status, error));
        break;
      case RealtimeSubscribeStatus.channelError:
        _addException(RealtimeSubscribeException(status, error));
        break;
    }
  });
  _getPostgrestData();
}
```

## Method Flow

### 1. Filter Setup and Channel Creation

The method begins by capturing the current stream filter and clearing the in-memory data cache:

```dart 158:168:packages/supabase/lib/src/supabase_stream_builder.dart
final currentStreamFilter = _streamFilter;
_streamData = [];
PostgresChangeFilter? realtimeFilter;
if (currentStreamFilter != null) {
  realtimeFilter = PostgresChangeFilter(
    type: currentStreamFilter.type,
    column: currentStreamFilter.column,
    value: currentStreamFilter.value,
  );
}
```

This ensures a clean state for the new stream session. If filters are applied (via methods like `eq()`, `neq()`, etc.), they are converted to realtime-compatible filters.

### 2. Realtime Channel Setup

```dart 170:170:packages/supabase/lib/src/supabase_stream_builder.dart
_channel = _realtimeClient.channel(_realtimeTopic);
```

Creates a dedicated realtime channel for this stream. The `_realtimeTopic` is generated in the `SupabaseQueryBuilder.stream()` method using the pattern `schema:table:incrementId`, ensuring unique topics for different table streams.

### 3. PostgreSQL Change Event Subscription

The core of the method sets up listeners for all PostgreSQL change events:

```dart 172:213:packages/supabase/lib/src/supabase_stream_builder.dart
_channel!
    .onPostgresChanges(
        event: PostgresChangeEvent.all,
        schema: _schema,
        table: _table,
        filter: realtimeFilter,
        callback: (payload) {
          // Handle INSERT, UPDATE, DELETE events
        })
```

This subscribes to all change events (`PostgresChangeEvent.all`) for the specified table and schema, with optional filtering.

### 4. Change Event Processing

The method handles three types of database operations:

#### INSERT Events

```dart 180:183:packages/supabase/lib/src/supabase_stream_builder.dart
case PostgresChangeEvent.insert:
  final newRecord = payload.newRecord;
  _streamData.add(newRecord);
  _addStream();
  break;
```

New records are simply added to the in-memory cache and the stream is updated.

#### UPDATE Events

```dart 185:197:packages/supabase/lib/src/supabase_stream_builder.dart
case PostgresChangeEvent.update:
  final updatedIndex = _streamData.indexWhere(
    (element) =>
        _isTargetRecord(record: element, payload: payload),
  );

  final updatedRecord = payload.newRecord;
  if (updatedIndex >= 0) {
    _streamData[updatedIndex] = updatedRecord;
  } else {
    _streamData.add(updatedRecord);
  }
  _addStream();
  break;
```

Updates are more complex - the method finds the existing record using `_isTargetRecord()` (which compares primary key values), replaces it if found, or adds it if it's a new record that matches the current filters.

#### DELETE Events

```dart 199:208:packages/supabase/lib/src/supabase_stream_builder.dart
case PostgresChangeEvent.delete:
  final deletedIndex = _streamData.indexWhere(
    (element) =>
        _isTargetRecord(record: element, payload: payload),
  );
  if (deletedIndex >= 0) {
    /// Delete the data from in memory cache if it was found
    _streamData.removeAt(deletedIndex);
    _addStream();
  }
  break;
```

Deletes remove the record from the cache if it exists.

### 5. Subscription Status Handling

The method monitors the realtime connection status:

```dart 214:234:packages/supabase/lib/src/supabase_stream_builder.dart
.subscribe((status, [error]) {
  switch (status) {
    case RealtimeSubscribeStatus.subscribed:
      // Reload all data after a reconnect from postgrest
      // First data from postgrest gets loaded before the realtime connect
      if (_wasSubscribed) {
        _getPostgrestData();
      }
      _wasSubscribed = true;
      break;
    case RealtimeSubscribeStatus.closed:
      _streamController?.close();
      break;
    case RealtimeSubscribeStatus.timedOut:
      _addException(RealtimeSubscribeException(status, error));
      break;
    case RealtimeSubscribeStatus.channelError:
      _addException(RealtimeSubscribeException(status, error));
      break;
  }
});
```

**Subscribed Status**: Triggers a full data reload via `_getPostgrestData()` after reconnections to ensure data consistency.

**Closed Status**: Properly closes the stream controller.

**Error States**: Converts realtime errors into stream exceptions.

### 6. Initial Data Loading

```dart 235:235:packages/supabase/lib/src/supabase_stream_builder.dart
_getPostgrestData();
```

Finally, initiates the initial data fetch from PostgREST to populate the stream with current data before realtime updates begin.

## Key Design Patterns

### Hybrid Architecture

Combines REST API (PostgREST) for initial data with WebSocket (Realtime) for live updates, ensuring immediate data availability while maintaining real-time synchronization.

### In-Memory Caching

Maintains a local copy of data (`_streamData`) that gets updated reactively, reducing the need for constant API calls.

### Primary Key-Based Record Matching

Uses `_uniqueColumns` (primary keys) to identify which records to update or delete, ensuring data integrity during concurrent operations.

### Connection Resilience

Handles reconnection scenarios by reloading data from PostgREST when the realtime connection is re-established.

## Error Handling

The method relies on the subscription status callback and `_addException()` method to propagate realtime connection errors to the stream consumers. PostgREST errors are handled in the `_getPostgrestData()` method.

## Performance Considerations

- Filters are applied both at the PostgREST level and realtime level for efficiency
- Only relevant change events are processed based on table/schema/filter criteria
- In-memory operations are optimized for real-time updates
- Stream emissions are batched and ordered when necessary

This method essentially creates a reactive, real-time view of database changes while maintaining the familiar Stream API interface that Flutter developers expect.
