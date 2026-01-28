# PostgrestTransformBuilder

## Overview

The `PostgrestTransformBuilder<T>` class is a core component of the Supabase Flutter PostgREST client library. It extends `RawPostgrestBuilder<T, T, T>` and provides a fluent API for applying transformation operations to database queries. This class enables developers to modify query results through operations like filtering, ordering, limiting, and formatting.

Located in the PostgREST package, this builder follows the builder pattern used throughout the Supabase client libraries, allowing method chaining to construct complex database queries in a readable and intuitive way.

## Class Structure

```dart 3:3:packages/postgrest/lib/src/postgrest_transform_builder.dart
class PostgrestTransformBuilder<T> extends RawPostgrestBuilder<T, T, T> {
```

The class is generic with type parameter `T`, representing the expected return type of the query. It inherits from `RawPostgrestBuilder<T, T, T>`, maintaining type consistency across the transformation chain.

## Key Methods

### Select Operations

#### `select([String columns = '*'])`

Performs horizontal filtering with SELECT, allowing specification of which columns to retrieve from the database.

```dart 26:52:packages/postgrest/lib/src/postgrest_transform_builder.dart
PostgrestTransformBuilder<PostgrestList> select([String columns = '*']) {
  // Remove whitespaces except when quoted
  var quoted = false;
  final re = RegExp(r'\s');
  final cleanedColumns = columns.split('').map((c) {
    if (re.hasMatch(c) && !quoted) {
      return '';
    }
    if (c == '"') {
      quoted = !quoted;
    }
    return c;
  }).join();
  final newHeaders = {..._headers};

  final url = overrideSearchParams('select', cleanedColumns);
  if (newHeaders['Prefer'] != null) {
    newHeaders['Prefer'] = '${newHeaders['Prefer']},';
  }
  newHeaders['Prefer'] = '${newHeaders['Prefer']}return=representation';
  return PostgrestTransformBuilder<PostgrestList>(
    _copyWithType(
      url: url,
      headers: newHeaders,
    ),
  );
}
```

**Features:**

- Cleans whitespace from column specifications while preserving quoted strings
- Automatically adds `return=representation` header for data retrieval
- Changes return type to `PostgrestList` for list results

**Usage Examples:**

```dart
// Select inserted data
supabase.from('users').insert().select('id, messages')

// Select inserted data with count
supabase.from('users').insert().select('id, messages').count(CountOption.exact)
```

### Ordering Operations

#### `order(String column, {bool ascending = false, bool nullsFirst = false, String? referencedTable})`

Orders query results by the specified column with configurable sort direction and null handling.

```dart 72:84:packages/postgrest/lib/src/postgrest_transform_builder.dart
PostgrestTransformBuilder<T> order(
  String column, {
  bool ascending = false,
  bool nullsFirst = false,
  String? referencedTable,
}) {
  final key = referencedTable == null ? 'order' : '$referencedTable.order';
  final existingOrder = _url.queryParameters[key];
  final value = '${existingOrder == null ? '' : '$existingOrder,'}'
      '$column.${ascending ? 'asc' : 'desc'}.${nullsFirst ? 'nullsfirst' : 'nullslast'}';
  final url = overrideSearchParams(key, value);
  return PostgrestTransformBuilder(copyWithUrl(url));
}
```

**Features:**

- Supports ascending/descending order
- Configurable null value positioning
- Handles referenced table ordering for joined queries
- Allows multiple order clauses by chaining

**Usage Examples:**

```dart
// Simple descending order
supabase.from('users').select().order('username', ascending: false)

// Order with nulls first
supabase.from('users').select().order('username', ascending: false, nullsFirst: true)

// Order referenced table column
supabase.from('users')
  .select('messages(*)')
  .order('channel_id', referencedTable: 'messages', ascending: false)
```

### Limiting Operations

#### `limit(int count, {String? referencedTable})`

Limits the number of rows returned by the query.

```dart 100:105:packages/postgrest/lib/src/postgrest_transform_builder.dart
PostgrestTransformBuilder<T> limit(int count, {String? referencedTable}) {
  final key = referencedTable == null ? 'limit' : '$referencedTable.limit';

  final url = appendSearchParams(key, '$count');
  return PostgrestTransformBuilder(copyWithUrl(url));
}
```

**Usage Example:**

```dart
supabase.from('users').select().limit(10)
```

#### `range(int from, int to, {String? referencedTable})`

Limits results to rows within a specified range (inclusive).

```dart 124:134:packages/postgrest/lib/src/postgrest_transform_builder.dart
PostgrestTransformBuilder<T> range(int from, int to,
    {String? referencedTable}) {
  final keyOffset =
      referencedTable == null ? 'offset' : '$referencedTable.offset';
  final keyLimit =
      referencedTable == null ? 'limit' : '$referencedTable.limit';

  var url = appendSearchParams(keyOffset, '$from');
  url = appendSearchParams(keyLimit, '${to - from + 1}', url);
  return PostgrestTransformBuilder(copyWithUrl(url));
}
```

**Usage Example:**

```dart
// Get rows 1-10 (equivalent to LIMIT 10 OFFSET 1)
supabase.from('users').select().range(1, 10)
```

### Single Row Operations

#### `single()`

Retrieves exactly one row from the result set. The query must be structured to return exactly one row.

```dart 146:155:packages/postgrest/lib/src/postgrest_transform_builder.dart
PostgrestTransformBuilder<PostgrestMap> single() {
  final newHeaders = {..._headers};
  newHeaders['Accept'] = 'application/vnd.pgrst.object+json';

  return PostgrestTransformBuilder(
    _copyWithType(
      headers: newHeaders,
    ),
  );
}
```

**Usage Example:**

```dart
final user = await supabase
  .from('users')
  .select()
  .eq('id', userId)
  .single();
```

#### `maybeSingle()`

Retrieves at most one row from the result set. Unlike `single()`, this method allows for zero results.

```dart 162:179:packages/postgrest/lib/src/postgrest_transform_builder.dart
PostgrestTransformBuilder<PostgrestMap?> maybeSingle() {
  // Temporary fix for https://github.com/supabase/supabase-flutter/issues/560
  // Issue persists e.g. for `.insert([...]).select().maybeSingle()`
  final newHeaders = {..._headers};

  if (_method?.toUpperCase() == 'GET') {
    newHeaders['Accept'] = 'application/json';
  } else {
    newHeaders['Accept'] = 'application/vnd.pgrst.object+json';
  }

  return PostgrestTransformBuilder(
    _copyWithType(
      maybeSingle: true,
      headers: newHeaders,
    ),
  );
}
```

**Features:**

- Handles both GET and non-GET operations with appropriate headers
- Includes a temporary fix for issue #560 related to insert operations

### Response Format Operations

#### `csv()`

Returns query results in CSV format, skipping object parsing for performance.

```dart 188:197:packages/postgrest/lib/src/postgrest_transform_builder.dart
PostgrestTransformBuilder<String> csv() {
  final newHeaders = {..._headers};
  newHeaders['Accept'] = 'text/csv';

  return PostgrestTransformBuilder(
    _copyWithType(
      headers: newHeaders,
    ),
  );
}
```

**Usage Example:**

```dart
final csvData = await supabase.from('users').select().csv();
```

#### `geojson()`

Enables GeoJSON format support for PostGIS data types.

```dart 242:247:packages/postgrest/lib/src/postgrest_transform_builder.dart
ResponsePostgrestBuilder<Map<String, dynamic>, Map<String, dynamic>,
    Map<String, dynamic>> geojson() {
  final newHeaders = {..._headers};
  newHeaders['Accept'] = 'application/geo+json;';
  return ResponsePostgrestBuilder(_copyWithType(headers: newHeaders));
}
```

**Note:** Requires PostGIS extension to be enabled on the Supabase instance.

### Aggregation Operations

#### `count([CountOption count = CountOption.exact])`

Performs a count query alongside the main query to get the total number of matching rows.

```dart 215:220:packages/postgrest/lib/src/postgrest_transform_builder.dart
ResponsePostgrestBuilder<PostgrestResponse<T>, T, T> count(
    [CountOption count = CountOption.exact]) {
  return ResponsePostgrestBuilder(
    _copyWithType(count: count),
  );
}
```

**Features:**

- Returns `PostgrestResponse<T>` containing both data and count
- Count respects filters but ignores modifiers like limit

**Usage Example:**

```dart
final response = await supabase
  .from('users')
  .select()
  .count(CountOption.exact);

final users = response.data;
final totalCount = response.count;
```

### Utility Operations

#### `head()`

Performs a HEAD request that returns no data but provides metadata.

```dart 232:234:packages/postgrest/lib/src/postgrest_transform_builder.dart
PostgrestBuilder<void, void, void> head() {
  return _copyWithType(method: METHOD_HEAD);
}
```

**Usage Examples:**

```dart
// Check if table has data
supabase.from("users").select().head()

// Check if RPC function exists
supabase.rpc("my_function").head()
```

#### `maxAffected(int value)`

Sets the maximum number of rows that can be affected by UPDATE or DELETE operations.

```dart 261:279:packages/postgrest/lib/src/postgrest_transform_builder.dart
PostgrestTransformBuilder<T> maxAffected(int value) {
  final newHeaders = {..._headers};

  // Add handling=strict and max-affected headers
  if (newHeaders['Prefer'] != null) {
    var preferHeader = newHeaders['Prefer']!;
    if (!preferHeader.contains('handling=strict')) {
      preferHeader += ',handling=strict';
    }
    if (!preferHeader.contains('max-affected=')) {
      preferHeader += ',max-affected=$value';
    }
    newHeaders['Prefer'] = preferHeader;
  } else {
    newHeaders['Prefer'] = 'handling=strict,max-affected=$value';
  }

  return PostgrestTransformBuilder(_copyWith(headers: newHeaders));
}
```

**Features:**

- Requires PostgREST v13 or higher
- Only works with PATCH and DELETE operations
- Fails the query if the limit is exceeded

**Usage Examples:**

```dart
// Update maximum 5 inactive users
supabase.from('users')
  .update({'active': false})
  .eq('status', 'inactive')
  .maxAffected(5)

// Delete maximum 10 inactive users
supabase.from('users')
  .delete()
  .eq('active', false)
  .maxAffected(10)
```

#### `explain()`

Obtains the EXPLAIN plan for the query, useful for performance debugging.

```dart 298:319:packages/postgrest/lib/src/postgrest_transform_builder.dart
PostgrestBuilder<String, String, String> explain({
  bool analyze = false,
  bool verbose = false,
  bool settings = false,
  bool buffers = false,
  bool wal = false,
}) {
  final options = [
    if (analyze) 'analyze',
    if (verbose) 'verbose',
    if (settings) 'settings',
    if (buffers) 'buffers',
    if (wal) 'wal',
  ].join('|');

  // An Accept header can carry multiple media types but postgrest-js always sends one
  final forMediatype = _headers['Accept'] ?? 'application/json';
  final newHeaders = {..._headers};
  newHeaders['Accept'] =
      'application/vnd.pgrst.plan+text; for="$forMediatype"; options=$options;';
  return _copyWithType(headers: newHeaders);
}
```

**Features:**

- Requires enabling explain on the Supabase instance (development only)
- Various options for detailed analysis
- Returns the query execution plan as text

## Implementation Patterns

### Method Chaining

All transformation methods return a new `PostgrestTransformBuilder` instance, enabling fluent method chaining:

```dart
supabase.from('users')
  .select('id, name, email')
  .order('name', ascending: true)
  .limit(10)
  .single()
```

### Header Management

The class extensively uses HTTP headers to control PostgREST behavior:

- `Accept` headers for response format (JSON, CSV, GeoJSON)
- `Prefer` headers for query preferences (return representation, handling strict, max-affected)

### URL Parameter Building

Query parameters are built using helper methods:

- `overrideSearchParams()` - Replaces existing parameters
- `appendSearchParams()` - Adds to existing parameters
- `copyWithUrl()` - Creates new instances with modified URLs

### Type Safety

The generic type system ensures type safety throughout the query building process:

- `select()` changes return type to `PostgrestList`
- `single()` changes return type to `PostgrestMap`
- `count()` returns `ResponsePostgrestBuilder<PostgrestResponse<T>, T, T>`

## Error Handling Considerations

- `single()` will throw an error if the result contains more than one row
- `maybeSingle()` is safer for operations that might return zero or one results
- `maxAffected()` will cause query failure if the limit is exceeded
- Network and parsing errors should be handled by the caller

## Performance Considerations

- `csv()` skips JSON parsing for better performance with large datasets
- `head()` provides metadata without transferring data
- `explain()` should only be used in development environments
- Consider using `range()` for pagination instead of `limit()` + `offset()` for large datasets

This class forms the foundation of the PostgREST query transformation API, providing developers with a powerful and intuitive way to construct database queries in Flutter applications using Supabase.
