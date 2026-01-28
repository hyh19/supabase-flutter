# SupabaseStreamBuilder

The `SupabaseStreamBuilder` class provides real-time streaming capabilities for Supabase database queries, combining PostgREST API calls with WebSocket-based real-time subscriptions to deliver live-updating data streams.

## Core Components

### Helper Classes and Types

```dart 9:24:packages/supabase/lib/src/supabase_stream_builder.dart
class _StreamPostgrestFilter {
  _StreamPostgrestFilter({
    required this.column,
    required this.value,
    required this.type,
  });

  /// Column name of the eq filter
  final String column;

  /// Value of the eq filter
  final dynamic value;

  /// Type of the filer being applied
  final PostgresChangeFilterType type;
}
```

The `_StreamPostgrestFilter` class represents a filter condition that can be applied to both PostgREST queries and real-time subscriptions, ensuring consistency between initial data loading and subsequent updates.

```dart 26:33:packages/supabase/lib/src/supabase_stream_builder.dart
class _Order {
  _Order({
    required this.column,
    required this.ascending,
  });
  final String column;
  final bool ascending;
}
```

The `_Order` class encapsulates sorting configuration for stream data.

```dart 35:45:packages/supabase/lib/src/supabase_stream_builder.dart
class RealtimeSubscribeException implements Exception {
  RealtimeSubscribeException(this.status, [this.details]);

  final RealtimeSubscribeStatus status;
  final Object? details;

  @override
  String toString() {
    return 'RealtimeSubscribeException(status: $status, details: $details)';
  }
}
```

`RealtimeSubscribeException` provides structured error handling for real-time subscription failures, including connection timeouts and channel errors.

```dart 47:47:packages/supabase/lib/src/supabase_stream_builder.dart
typedef SupabaseStreamEvent = List<Map<String, dynamic>>;
```

`SupabaseStreamEvent` is a type alias representing a list of database records as maps, which is the standard format for stream emissions.

## Main Class Architecture

### SupabaseStreamBuilder Class

The `SupabaseStreamBuilder` extends Dart's `Stream<SupabaseStreamEvent>` class, providing a reactive interface for database streaming.

```dart 49:97:packages/supabase/lib/src/supabase_stream_builder.dart
class SupabaseStreamBuilder extends Stream<SupabaseStreamEvent> {
  final PostgrestQueryBuilder _queryBuilder;

  final RealtimeClient _realtimeClient;

  final String _realtimeTopic;

  RealtimeChannel? _channel;

  final String _schema;

  final String _table;

  /// Used to identify which row has changed
  final List<String> _uniqueColumns;

  final _log = Logger('supabase.supabase');

  /// StreamController for `stream()` method.
  BehaviorSubject<SupabaseStreamEvent>? _streamController;

  /// Contains the combined data of postgrest and realtime to emit as stream.
  SupabaseStreamEvent _streamData = [];

  /// `eq` filter used for both postgrest and realtime
  _StreamPostgrestFilter? _streamFilter;

  /// Which column to order by and whether it's ascending
  _Order? _orderBy;

  /// Count of record to be returned
  int? _limit;

  /// Flag that the stream has at least one time been subscribed to realtime
  bool _wasSubscribed = false;
```

Key fields include:

- `_queryBuilder`: PostgREST query builder for initial data fetching
- `_realtimeClient`: WebSocket client for real-time updates
- `_streamData`: In-memory cache of current stream data
- `_streamFilter`: Optional filter applied to both queries and subscriptions
- `_orderBy` and `_limit`: Sorting and pagination configuration

### Stream Configuration Methods

```dart 99:119:packages/supabase/lib/src/supabase_stream_builder.dart
  /// Orders the result with the specified [column].
  ///
  /// When `ascending` value is true, the result will be in ascending order.
  ///
  /// ```dart
  /// supabase.from('users').stream(primaryKey: ['id']).order('username', ascending: false);
  /// ```
  SupabaseStreamBuilder order(String column, {bool ascending = false}) {
    _orderBy = _Order(column: column, ascending: ascending);
    return this;
  }

  /// Limits the result with the specified `count`.
  ///
  /// ```dart
  /// supabase.from('users').stream(primaryKey: ['id']).limit(10);
  /// ```
  SupabaseStreamBuilder limit(int count) {
    _limit = count;
    return this;
  }
```

These methods provide fluent configuration of sorting and pagination, returning `this` to enable method chaining.

### Stream Lifecycle Management

```dart 143:156:packages/supabase/lib/src/supabase_stream_builder.dart
  /// Sets up the stream controller and calls the method to get data as necessary
  void _setupStream() {
    _streamController ??= BehaviorSubject(
      onListen: () {
        _getStreamData();
      },
      onCancel: () {
        _log.fine('stream controller for table: $_table got closed');
        _channel?.unsubscribe();
        _streamController?.close();
        _streamController = null;
      },
    );
  }
```

The `_setupStream()` method initializes a `BehaviorSubject` that manages the stream lifecycle:

- `onListen`: Triggers initial data fetching when subscribers connect
- `onCancel`: Cleans up resources when the last subscriber disconnects

## Data Fetching and Synchronization

### Initial Data Loading

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

The `_getPostgrestData()` method fetches initial data via PostgREST API, applying the same filters, sorting, and limits that will be used for real-time updates. This ensures data consistency between the initial load and subsequent real-time changes.

### Real-time Subscription Setup

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

The `_getStreamData()` method orchestrates the real-time streaming setup:

1. **Filter Translation**: Converts internal filters to real-time subscription filters
2. **Channel Setup**: Creates a WebSocket channel for the specific table and schema
3. **Event Handling**: Processes INSERT, UPDATE, and DELETE events from the database
4. **Subscription Management**: Handles connection lifecycle including reconnections

### Change Event Processing

For each database change event, the system:

- **INSERT**: Adds new records to the in-memory cache
- **UPDATE**: Locates existing records by primary key and updates them
- **DELETE**: Removes records from the cache by primary key matching

```dart 288:300:packages/supabase/lib/src/supabase_stream_builder.dart
  bool _isTargetRecord({
    required Map<String, dynamic> record,
    required PostgresChangePayload payload,
  }) {
    late final Map<String, dynamic> targetRecord;
    if (payload.eventType == PostgresChangeEvent.update) {
      targetRecord = payload.newRecord;
    } else if (payload.eventType == PostgresChangeEvent.delete) {
      targetRecord = payload.oldRecord;
    }
    return _uniqueColumns
        .every((column) => record[column] == targetRecord[column]);
  }
```

The `_isTargetRecord()` method identifies which cached record corresponds to a real-time change by comparing primary key values.

## Data Ordering and Limiting

```dart 302:316:packages/supabase/lib/src/supabase_stream_builder.dart
  void _sortData() {
    final orderModifier = _orderBy!.ascending ? 1 : -1;
    _streamData.sort((a, b) {
      final columnA = a[_orderBy!.column];
      final columnB = b[_orderBy!.column];

      if (columnA is num && columnB is num) {
        return orderModifier * columnA.compareTo(columnB);
      } else if (columnA is String && columnB is String) {
        return orderModifier * columnA.compareTo(columnB);
      } else {
        return 0;
      }
    });
  }
```

The `_sortData()` method maintains sorted order for numeric and string columns, ensuring the stream emits data in the correct order after each update.

```dart 318:328:packages/supabase/lib/src/supabase_stream_builder.dart
  /// Will add new data to the stream if streamController is not closed
  void _addStream() {
    if (_orderBy != null) {
      _sortData();
    }
    if (!(_streamController?.isClosed ?? true)) {
      final emitData =
          (_limit != null ? _streamData.take(_limit!) : _streamData).toList();
      _streamController!.add(emitData);
    }
  }
```

The `_addStream()` method emits the current state of the data to subscribers, applying sorting and limiting as configured.

## Stream Interface Implementation

### Core Stream Methods

```dart 127:141:packages/supabase/lib/src/supabase_stream_builder.dart
  @override
  StreamSubscription<SupabaseStreamEvent> listen(
    void Function(SupabaseStreamEvent event)? onData, {
    Function? onError,
    void Function()? onDone,
    bool? cancelOnError,
  }) {
    _setupStream();
    return _streamController!.stream.listen(
      onData,
      onError: onError,
      onDone: onDone,
      cancelOnError: cancelOnError,
    );
  }
```

The `listen()` method provides the standard Stream interface, delegating to the internal `BehaviorSubject`.

```dart 337:338:packages/supabase/lib/src/supabase_stream_builder.dart
  @override
  bool get isBroadcast => true;
```

The stream is broadcast-capable, allowing multiple subscribers to listen simultaneously.

### Advanced Stream Operations

```dart 341:380:packages/supabase/lib/src/supabase_stream_builder.dart
  @override
  Stream<E> asyncMap<E>(
      FutureOr<E> Function(SupabaseStreamEvent event) convert) {
    // Copied from [Stream.asyncMap]

    final controller = BehaviorSubject<E>();

    controller.onListen = () {
      StreamSubscription<SupabaseStreamEvent> subscription = listen(null,
          onError: controller.addError, // Avoid Zone error replacement.
          onDone: controller.close);
      FutureOr<void> add(E value) {
        controller.add(value);
      }

      final addError = controller.addError;
      final resume = subscription.resume;
      subscription.onData((SupabaseStreamEvent event) {
        FutureOr<E> newValue;
        try {
          newValue = convert(event);
        } catch (e, s) {
          controller.addError(e, s);
          return;
        }
        if (newValue is Future<E>) {
          subscription.pause();
          newValue.then(add, onError: addError).whenComplete(resume);
        } else {
          controller.add(newValue as dynamic);
        }
      });
      controller.onCancel = subscription.cancel;
      if (!isBroadcast) {
        controller
          ..onPause = subscription.pause
          ..onResume = resume;
      }
    };
    return controller.stream;
  }
```

The `asyncMap()` method enables asynchronous transformations of stream events, supporting both synchronous and asynchronous conversion functions.

## Filtering Capabilities

The `SupabaseStreamFilterBuilder` class extends `SupabaseStreamBuilder` with comprehensive filtering methods:

```dart 13:27:packages/supabase/lib/src/supabase_stream_filter_builder.dart
  /// Filters the results where [column] equals [value].
  ///
  /// Only one filter can be applied to `.stream()`.
  ///
  /// ```dart
  /// supabase.from('users').stream(primaryKey: ['id']).eq('name', 'Supabase');
  /// ```
  SupabaseStreamBuilder eq(String column, Object value) {
    _streamFilter = _StreamPostgrestFilter(
      type: PostgresChangeFilterType.eq,
      column: column,
      value: value,
    );
    return this;
  }
```

All filter methods follow the same pattern: creating a `_StreamPostgrestFilter` instance and returning `this` for method chaining. Supported filter types include equality, inequality, range comparisons, and inclusion filters.

## Error Handling

```dart 330:335:packages/supabase/lib/src/supabase_stream_builder.dart
  /// Will add error to the stream if streamController is not closed
  void _addException(Object error, [StackTrace? stackTrace]) {
    if (!(_streamController?.isClosed ?? true)) {
      _streamController?.addError(error, stackTrace ?? StackTrace.current);
    }
  }
```

The `_addException()` method safely propagates errors to stream subscribers when the controller is still active.

## Usage Pattern

The typical usage involves chaining configuration methods:

```dart
final stream = supabase
    .from('messages')
    .stream(primaryKey: ['id'])
    .eq('room_id', roomId)
    .order('created_at', ascending: true)
    .limit(50);

stream.listen((messages) {
  // Handle real-time message updates
});
```

This creates a filtered, ordered, limited stream that automatically handles:

1. Initial data loading via PostgREST
2. Real-time subscription setup
3. Automatic data synchronization
4. Error handling and reconnection logic

## Key Design Principles

1. **Consistency**: Filters and sorting apply to both initial queries and real-time updates
2. **Resource Management**: Automatic cleanup when subscribers disconnect
3. **Error Resilience**: Structured error handling with reconnection capabilities
4. **Performance**: In-memory caching with efficient primary key lookups
5. **Reactive**: Full Stream interface compliance with RxDart integration
6. **Type Safety**: Strong typing with Dart's type system throughout
