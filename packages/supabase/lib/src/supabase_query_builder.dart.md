# SupabaseQueryBuilder

The `SupabaseQueryBuilder` class extends the `PostgrestQueryBuilder` to provide real-time streaming capabilities for Supabase database queries. This class bridges the gap between static PostgREST queries and dynamic real-time updates from Supabase's real-time engine.

## Class Overview

`SupabaseQueryBuilder` is a specialized query builder that inherits all the database querying functionality from `PostgrestQueryBuilder` while adding real-time streaming capabilities through the `stream` method.

```dart 3:3:packages/supabase/lib/src/supabase_query_builder.dart
class SupabaseQueryBuilder extends PostgrestQueryBuilder {
```

## Constructor Parameters

The class requires several key components for real-time functionality:

- `url`: The base URL for the PostgREST API endpoint
- `realtime`: An instance of `RealtimeClient` for WebSocket connections
- `headers`: Optional HTTP headers (defaults to empty)
- `schema`: The database schema name
- `table`: The specific table name for queries
- `httpClient`: Custom HTTP client for requests
- `incrementId`: A unique identifier for managing real-time subscriptions
- `isolate`: JSON parsing isolate for performance optimization

```dart 9:24:packages/supabase/lib/src/supabase_query_builder.dart
  SupabaseQueryBuilder(
    String url,
    RealtimeClient realtime, {
    super.headers = const {},
    required String super.schema,
    required String table,
    super.httpClient,
    required int incrementId,
    required super.isolate,
  })  : _realtime = realtime,
        _schema = schema,
        _table = table,
        _incrementId = incrementId,
        super(
          url: Uri.parse(url),
        );
```

## Private Fields

The class maintains several private fields for managing real-time subscriptions:

- `_realtime`: The WebSocket client for real-time updates
- `_schema`: Database schema name used in subscription topics
- `_table`: Table name used in subscription topics  
- `_incrementId`: Unique identifier for subscription management

```dart 4:7:packages/supabase/lib/src/supabase_query_builder.dart
  final RealtimeClient _realtime;
  final String _schema;
  final String _table;
  final int _incrementId;
```

## Stream Method

The primary feature of `SupabaseQueryBuilder` is the `stream` method, which creates a real-time data stream that combines initial PostgREST query results with ongoing database changes.

### Method Signature

```dart 46:46:packages/supabase/lib/src/supabase_query_builder.dart
  SupabaseStreamFilterBuilder stream({required List<String> primaryKey}) {
```

### Parameters

- `primaryKey`: A list of column names that serve as the primary key for the table. This is required for the real-time engine to properly identify which records to update, insert, or delete when changes occur.

### Key Features

1. **Real-time Updates**: Automatically receives database changes via WebSocket connections
2. **Primary Key Management**: Uses specified primary key columns to match incoming updates with existing records
3. **Automatic Lifecycle Management**: Handles connection lifecycle and data refetching when needed
4. **Query Filtering**: Supports additional filtering methods like `eq`, `neq`, `lt`, `lte`, `gt`, `gte`, `order`, and `limit`

### Usage Examples

#### Basic Usage

```dart
// Stream all chat messages with primary key 'id'
supabase.from('chats').stream(primaryKey: ['id']).listen(_onChatsReceived);
```

#### Filtered Streaming

```dart
// Stream chat messages for a specific room, ordered by creation time, limited to 20 records
supabase.from('chats')
  .stream(primaryKey: ['id'])
  .eq('room_id', '123')
  .order('created_at')
  .limit(20)
  .listen(_onChatsReceived);
```

### Real-time Topic Construction

The stream creates a subscription topic using the format: `schema:table:incrementId`. This topic is used by Supabase's real-time engine to route database change events to the correct subscribers.

```dart 51:51:packages/supabase/lib/src/supabase_query_builder.dart
      realtimeTopic: '$_schema:$_table:$_incrementId',
```

### Error Handling Requirements

The documentation emphasizes proper error handling when using streams:

> Make sure to provide `onError` and `onDone` callbacks to `Stream.listen` to handle errors and completion of the stream.

### Realtime Setup Requirements

Real-time streaming requires proper database replication setup:

> Realtime is disabled by default for new tables. You can turn it on by managing replication.

## Integration with PostgREST

Since `SupabaseQueryBuilder` extends `PostgrestQueryBuilder`, it inherits all standard database querying capabilities:

- `select()` - Retrieve data with column selection
- `insert()` - Add new records
- `update()` - Modify existing records  
- `delete()` - Remove records
- `upsert()` - Insert or update based on conditions
- Various filter methods (`eq`, `neq`, `gt`, `lt`, etc.)
- Transform methods (`order`, `limit`, `range`, etc.)

## Return Type

The `stream` method returns a `SupabaseStreamFilterBuilder`, which provides additional filtering capabilities specific to real-time streams and ultimately creates the `Stream` of data.

## Architectural Role

`SupabaseQueryBuilder` serves as the bridge between:

- **PostgREST**: For initial data fetching and static queries
- **Realtime Engine**: For dynamic updates via WebSocket connections
- **Application Layer**: Providing reactive data streams to UI components

This design enables developers to build reactive applications where UI components automatically update when underlying database data changes, without manual polling or complex state management.
