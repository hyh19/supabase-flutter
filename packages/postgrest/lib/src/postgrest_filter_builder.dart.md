# PostgrestFilterBuilder Class

## Overview

The `PostgrestFilterBuilder<T>` class is a core component of the Supabase Flutter SDK's PostgREST client, providing a fluent API for building database filter queries. It enables developers to construct complex SQL-like filter operations through method chaining, which are then translated into URL query parameters that PostgREST can understand.

This class is part of the query builder pattern used throughout the PostgREST client, where each method returns a new instance of the builder with updated query parameters, allowing for immutable and chainable query construction.

## Class Hierarchy

```dart 3:3:packages/postgrest/lib/src/postgrest_filter_builder.dart
class PostgrestFilterBuilder<T> extends PostgrestTransformBuilder<T> {
```

The inheritance hierarchy is:

- `PostgrestFilterBuilder<T>` → `PostgrestTransformBuilder<T>` → `RawPostgrestBuilder<T, T, T>` → `PostgrestBuilder<T, S, R>`

This layered architecture allows different builder types to handle different aspects of query construction while maintaining a consistent API.

## Core Architecture

### Fluent API Pattern

All filter methods return a new instance of `PostgrestFilterBuilder<T>`, enabling method chaining:

```dart
supabase
  .from('users')
  .select()
  .eq('status', 'active')
  .gt('age', 18)
  .like('name', '%john%')
```

### URL Parameter Construction

The class internally builds query parameters that PostgREST interprets. For example:

- `eq('username', 'john')` becomes `?username=eq.john`
- `gt('age', 18)` becomes `?age=gt.18`

Complex filters can result in multiple query parameters being appended to the URL.

## Filter Method Categories

### Basic Comparison Filters

#### Equality and Inequality

```dart 62:88:packages/postgrest/lib/src/postgrest_filter_builder.dart
PostgrestFilterBuilder<T> eq(String column, Object value) {
  final Uri url;
  if (value is List) {
    url = appendSearchParams(column, 'eq.{${_cleanFilterArray(value)}}');
  } else {
    url = appendSearchParams(column, 'eq.$value');
  }
  return copyWithUrl(url);
}

PostgrestFilterBuilder<T> neq(String column, Object value) {
  final Uri url;
  if (value is List) {
    url = appendSearchParams(column, 'neq.{${_cleanFilterArray(value)}}');
  } else {
    url = appendSearchParams(column, 'neq.$value');
  }
  return copyWithUrl(url);
}
```

These methods support both single values and arrays of values, automatically formatting them appropriately for PostgREST.

#### Comparison Operators

```dart 98:136:packages/postgrest/lib/src/postgrest_filter_builder.dart
PostgrestFilterBuilder<T> gt(String column, Object value) {
  return copyWithUrl(appendSearchParams(column, 'gt.$value'));
}

PostgrestFilterBuilder<T> gte(String column, Object value) {
  return copyWithUrl(appendSearchParams(column, 'gte.$value'));
}

PostgrestFilterBuilder<T> lt(String column, Object value) {
  return copyWithUrl(appendSearchParams(column, 'lt.$value'));
}

PostgrestFilterBuilder<T> lte(String column, Object value) {
  return copyWithUrl(appendSearchParams(column, 'lte.$value'));
}
```

Standard SQL comparison operators for greater than, less than, and their equals variants.

### Pattern Matching

#### LIKE Operations

```dart 146:212:packages/postgrest/lib/src/postgrest_filter_builder.dart
PostgrestFilterBuilder<T> like(String column, String pattern) {
  return copyWithUrl(appendSearchParams(column, 'like.$pattern'));
}

PostgrestFilterBuilder likeAllOf(String column, List<String> patterns) {
  return copyWithUrl(
      appendSearchParams(column, 'like(all).{${patterns.join(',')}}'));
}

PostgrestFilterBuilder ilike(String column, String pattern) {
  return copyWithUrl(appendSearchParams(column, 'ilike.$pattern'));
}
```

- `like`: Case-sensitive pattern matching using SQL LIKE syntax
- `ilike`: Case-insensitive pattern matching
- `likeAllOf`/`ilikeAllOf`: Match all patterns in a list
- `likeAnyOf`/`ilikeAnyOf`: Match any pattern in a list

#### Regular Expressions

```dart 487:501:packages/postgrest/lib/src/postgrest_filter_builder.dart
PostgrestFilterBuilder<T> matchRegex(String column, String pattern) {
  return copyWithUrl(appendSearchParams(column, 'match.$pattern'));
}

PostgrestFilterBuilder<T> imatchRegex(String column, String pattern) {
  return copyWithUrl(appendSearchParams(column, 'imatch.$pattern'));
}
```

PostgreSQL regular expression matching with case-sensitive (`match`) and case-insensitive (`imatch`) variants.

### Array and Collection Filters

#### Membership Testing

```dart 237:240:packages/postgrest/lib/src/postgrest_filter_builder.dart
PostgrestFilterBuilder<T> inFilter(String column, List values) {
  return copyWithUrl(
      appendSearchParams(column, 'in.(${_cleanFilterArray(values)})'));
}
```

Tests if a column value is contained within a list of values. The `_cleanFilterArray` helper method properly formats different value types:

```dart 352:358:packages/postgrest/lib/src/postgrest_builder.dart
String _cleanFilterArray(List filter) {
  if (filter.every((element) => element is num)) {
    return filter.map((s) => '$s').join(',');
  } else {
    return filter.map((s) => '"$s"').join(',');
  }
}
```

Numbers are joined with commas, while other types (strings, etc.) are quoted and then joined.

#### Containment Operations

```dart 267:281:packages/postgrest/lib/src/postgrest_filter_builder.dart
PostgrestFilterBuilder<T> contains(String column, Object value) {
  final Uri url;
  if (value is String) {
    // range types can be inclusive '[', ']' or exclusive '(', ')' so just
    // keep it simple and accept a string
    url = appendSearchParams(column, 'cs.$value');
  } else if (value is List) {
    // array
    url = appendSearchParams(column, 'cs.{${_cleanFilterArray(value)}}');
  } else {
    // json
    url = appendSearchParams(column, 'cs.${json.encode(value)}');
  }
  return copyWithUrl(url);
}
```

The `contains` and `containedBy` methods handle different data types:

- **Arrays**: Check if one array contains another
- **Ranges**: Check range containment (e.g., `[1,10)` contains `[2,5)`)
- **JSON**: Check if a JSON object contains specified key-value pairs

### Range-Specific Operations

```dart 332:382:packages/postgrest/lib/src/postgrest_filter_builder.dart
PostgrestFilterBuilder<T> rangeLt(String column, String range) {
  return copyWithUrl(appendSearchParams(column, 'sl.$range'));
}

PostgrestFilterBuilder<T> rangeGt(String column, String range) {
  return copyWithUrl(appendSearchParams(column, 'sr.$range'));
}

PostgrestFilterBuilder<T> rangeGte(String column, String range) {
  return copyWithUrl(appendSearchParams(column, 'nxl.$range'));
}

PostgrestFilterBuilder<T> rangeLte(String column, String range) {
  return copyWithUrl(appendSearchParams(column, 'nxr.$range'));
}

PostgrestFilterBuilder<T> rangeAdjacent(String column, String range) {
  return copyWithUrl(appendSearchParams(column, 'adj.$range'));
}
```

These operators work specifically with PostgreSQL range types:

- `sl`/`sr`: Strictly left/right of another range
- `nxl`/`nxr`: Does not extend left/right of another range
- `adj`: Adjacent to another range

### Logical Operations

#### NOT Operation

```dart 18:36:packages/postgrest/lib/src/postgrest_filter_builder.dart
PostgrestFilterBuilder<T> not(String column, String operator, Object? value) {
  final Uri url;
  if (value is List) {
    if (operator == "in") {
      url = appendSearchParams(
        column,
        'not.$operator.(${_cleanFilterArray(value)})',
      );
    } else {
      url = appendSearchParams(
        column,
        'not.$operator.{${_cleanFilterArray(value)}}',
      );
    }
  } else {
    url = appendSearchParams(column, 'not.$operator.$value');
  }
  return copyWithUrl(url);
}
```

Negates any filter operation by prefixing it with `not.`.

#### OR Operation

```dart 46:50:packages/postgrest/lib/src/postgrest_filter_builder.dart
PostgrestFilterBuilder<T> or(String filters, {String? referencedTable}) {
  final key = referencedTable != null ? '$referencedTable.or' : 'or';
  final url = appendSearchParams(key, '($filters)');
  return copyWithUrl(url);
}
```

Combines multiple conditions with OR logic. The `referencedTable` parameter allows OR conditions to be applied to related tables in joins.

### Advanced Filters

#### Text Search

```dart 413:433:packages/postgrest/lib/src/postgrest_filter_builder.dart
PostgrestFilterBuilder<T> textSearch(
  String column,
  String query, {
  /// The text search configuration to use.
  String? config,

  /// The type of tsquery conversion to use on [query].
  TextSearchType? type,
}) {
  var typePart = '';
  if (type == TextSearchType.plain) {
    typePart = 'pl';
  } else if (type == TextSearchType.phrase) {
    typePart = 'ph';
  } else if (type == TextSearchType.websearch) {
    typePart = 'w';
  }
  final configPart = config == null ? '' : '($config)';
  return copyWithUrl(
      appendSearchParams(column, '${typePart}fts$configPart.$query'));
}
```

Full-text search using PostgreSQL's text search capabilities:

```dart 101:111:packages/postgrest/lib/src/types.dart
enum TextSearchType {
  /// Uses PostgreSQL's plainto_tsquery function.
  plain,

  /// Uses PostgreSQL's phraseto_tsquery function.
  phrase,

  /// Uses PostgreSQL's websearch_to_tsquery function.
  /// This function will never raise syntax errors, which makes it possible to use raw user-supplied input for search, and can be used with advanced operators.
  websearch,
```

#### Null Handling

```dart 224:226:packages/postgrest/lib/src/postgrest_filter_builder.dart
PostgrestFilterBuilder<T> isFilter(String column, bool? value) {
  return copyWithUrl(appendSearchParams(column, 'is.$value'));
}
```

Specifically handles `null`, `true`, and `false` values, which regular equality operators don't handle properly.

#### Match Multiple Columns

```dart 473:477:packages/postgrest/lib/src/postgrest_filter_builder.dart
PostgrestFilterBuilder<T> match(Map<String, Object> query) {
  var url = _url;
  query.forEach((k, v) => url = appendSearchParams(k, 'eq.$v', url));
  return copyWithUrl(url);
}
```

Applies equality filters to multiple columns in a single operation.

#### Generic Filter

```dart 443:462:packages/postgrest/lib/src/postgrest_filter_builder.dart
PostgrestFilterBuilder<T> filter(
    String column, String operator, Object? value) {
  final Uri url;
  if (value is List) {
    if (operator == "in") {
      url = appendSearchParams(
        column,
        '$operator.(${_cleanFilterArray(value)})',
      );
    } else {
      url = appendSearchParams(
        column,
        '$operator.{${_cleanFilterArray(value)}}',
      );
    }
  } else {
    url = appendSearchParams(column, '$operator.$value');
  }
  return copyWithUrl(url);
}
```

A generic filter method that accepts any operator as a string, providing maximum flexibility.

## Implementation Details

### URL Building Mechanism

The core of the filtering system is the `appendSearchParams` method:

```dart 335:340:packages/postgrest/lib/src/postgrest_builder.dart
Uri appendSearchParams(String key, String value, [Uri? url]) {
  final searchParams =
      Map<String, dynamic>.from((url ?? _url).queryParametersAll);
  searchParams[key] = [...searchParams[key] ?? [], value];
  return (url ?? _url).replace(queryParameters: searchParams);
}
```

This method accumulates multiple filter values for the same column, allowing complex queries like filtering by multiple values on the same column.

### Method Chaining and Immutability

Each filter method creates a new instance of `PostgrestFilterBuilder<T>` with updated URL parameters:

```dart 6:8:packages/postgrest/lib/src/postgrest_filter_builder.dart
@override
PostgrestFilterBuilder<T> copyWithUrl(Uri url) =>
    PostgrestFilterBuilder(_copyWith(url: url));
```

This ensures immutability and allows for safe method chaining without side effects.

### Header Management

```dart 519:523:packages/postgrest/lib/src/postgrest_filter_builder.dart
@override
PostgrestFilterBuilder<T> setHeader(String key, String value) {
  return PostgrestFilterBuilder(
    _copyWith(headers: {..._headers, key: value}),
  );
}
```

Allows setting custom HTTP headers for the request, which can be useful for authentication or custom PostgREST configurations.

## Usage Examples

### Basic Filtering

```dart
// Find active users older than 18
final users = await supabase
  .from('users')
  .select()
  .eq('status', 'active')
  .gt('age', 18);

// Find users with names containing "john" (case insensitive)
final johns = await supabase
  .from('users')
  .select()
  .ilike('name', '%john%');
```

### Complex Queries

```dart
// Users who are either online OR have a specific role
final users = await supabase
  .from('users')
  .select()
  .or('status.eq.ONLINE,role.eq.admin');

// Users with tags containing both 'urgent' and 'backend'
final issues = await supabase
  .from('issues')
  .select()
  .contains('tags', ['urgent', 'backend']);
```

### Array and JSON Operations

```dart
// Issues assigned to specific users
final issues = await supabase
  .from('issues')
  .select()
  .overlaps('assigned_users', [1, 2, 3]);

// Users with specific address data
final users = await supabase
  .from('users')
  .select()
  .contains('address', {'city': 'New York'});
```

### Text Search

```dart
// Full-text search in articles
final articles = await supabase
  .from('articles')
  .select()
  .textSearch('content', 'database optimization', 
      type: TextSearchType.websearch);
```

## Integration with PostgREST

The `PostgrestFilterBuilder` translates method calls into PostgREST-compatible URL query parameters. PostgREST is a standalone web server that turns PostgreSQL databases directly into RESTful APIs, and it understands these query parameter formats natively.

For example, a complex query like:

```dart
supabase.from('users').select().eq('status', 'active').gt('age', 18)
```

Becomes a URL like:

```text
GET /users?status=eq.active&age=gt.18
```

PostgREST then converts these parameters into appropriate SQL WHERE clauses, providing a type-safe, fluent interface for database queries without requiring raw SQL knowledge.
