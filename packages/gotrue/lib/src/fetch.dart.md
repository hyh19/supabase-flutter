# GotrueFetch

The `GotrueFetch` class is responsible for making HTTP requests to the GoTrue authentication API. It handles all network communication, error processing, and response parsing for authentication-related operations.

## Overview

```dart
class GotrueFetch {
  final Client? httpClient;

  const GotrueFetch([this.httpClient]);
}
```

This class provides a clean abstraction over HTTP communication, encapsulating:

- Request construction and execution
- Error handling with appropriate exception types
- Response parsing and decoding
- JWT token injection

## Request Method Types

```dart
enum RequestMethodType { get, post, put, delete }
```

The `RequestMethodType` enum defines the supported HTTP methods for GoTrue API requests. Each method corresponds to a standard HTTP verb used in RESTful API interactions.

| Method | Usage |
|--------|-------|
| `get` | Retrieve resources (e.g., fetching user session) |
| `post` | Create resources (e.g., sign up, sign in) |
| `put` | Update resources (e.g., update user attributes) |
| `delete` | Remove resources (e.g., sign out) |

## Status Code Handling

### Checking Success

```dart
bool isSuccessStatusCode(int code) {
  return code >= 200 && code <= 299;
}
```

The `isSuccessStatusCode()` method determines if an HTTP response indicates success. All 2xx status codes (200-299) are considered successful, following the HTTP specification. This method is used to decide whether to process the response or handle it as an error.

## Error Message Extraction

### Extracting Error Messages

```dart
String _getErrorMessage(dynamic error) {
  if (error is Map) {
    return error['msg'] ??
        error['message'] ??
        error['error_description'] ??
        error['error']?.toString() ??
        error.toString();
  }
  return error.toString();
}
```

The `_getErrorMessage()` method extracts human-readable error messages from various response formats. It checks multiple possible key names in order of priority:

1. `msg` - Custom message key
2. `message` - Standard message key
3. `error_description` - OAuth-style description
4. `error` - Generic error key
5. Fallback to `toString()` representation

### Extracting Error Codes

```dart
String? _getErrorCode(dynamic error, String key) {
  if (error is Map) {
    final dynamic errorCode = error[key];
    if (errorCode is String) {
      return errorCode;
    }
  }
  return null;
}
```

The `_getErrorCode()` method retrieves structured error codes from error responses. It extracts the error code using a specified key and validates that the result is a string.

## Error Handling

### Main Error Handler

```dart
AuthException _handleError(dynamic error) {
  if (error is! Response) {
    throw AuthRetryableFetchException(message: error.toString());
  }
  final response = error;

  // If the status is 500 or above, it's likely a server error,
  // and can be retried.
  if (response.statusCode >= 500) {
    throw AuthRetryableFetchException(
      message: response.body,
      statusCode: response.statusCode.toString(),
    );
  }

  // ... more error handling
}
```

The `_handleError()` method processes HTTP error responses and converts them into appropriate `AuthException` types. This method implements a sophisticated error classification system:

**Non-Response Errors:**
When the error is not an HTTP Response object, it's treated as a retryable network or infrastructure error.

**Server Errors (5xx):**
HTTP 5xx status codes indicate server-side issues. These are classified as `AuthRetryableFetchException` since they may be temporary and suitable for retry with exponential backoff.

**Empty Response Bodies:**
Responses with status codes but empty bodies trigger `AuthUnknownException` with a descriptive message.

**JSON Parsing Errors:**
If the response body cannot be decoded as JSON, an `AuthUnknownException` is thrown.

**Weak Password Detection:**
The handler includes special logic for password validation errors:

```dart
// Legacy support for weak password errors, when there were no error codes
if (data is Map &&
    data['weak_password'] is Map &&
    data['weak_password']['reasons'] is List &&
    (data['weak_password']['reasons'] as List).isNotEmpty &&
    (data['weak_password']['reasons'] as List)
        .whereNot((element) => element is String)
        .isEmpty) {
  throw AuthWeakPasswordException(
    message: _getErrorMessage(data),
    statusCode: response.statusCode.toString(),
    reasons: List<String>.from(data['weak_password']['reasons']),
  );
}
```

This code handles the weak password response in two scenarios:

1. **Legacy format**: No explicit error code, but contains `weak_password.reasons`
2. **Modern format**: Explicit `weak_password` error code with optional reasons

**API Version Handling:**
Error code extraction adapts based on the API version:

```dart
final responseApiVersion = ApiVersion.fromResponse(response);

if (responseApiVersion?.isSameOrAfter(ApiVersions.v20240101) ?? false) {
  errorCode = _getErrorCode(data, 'code');
} else {
  errorCode = _getErrorCode(data, 'error_code');
}
```

Newer API versions use the `code` key, while older versions use `error_code`.

## Request Execution

### Public Request Method

```dart
Future<dynamic> request(
  String url,
  RequestMethodType method, {
  GotrueRequestOptions? options,
}) async {
  final headers = options?.headers ?? {};

  // Set the API version header if not already set
  if (!headers.containsKey(Constants.apiVersionHeaderName)) {
    headers[Constants.apiVersionHeaderName] = ApiVersions.v20240101.name;
  }

  if (options?.jwt != null) {
    headers['Authorization'] = 'Bearer ${options!.jwt}';
  }

  final qs = options?.query ?? {};
  if (options?.redirectTo != null) {
    qs['redirect_to'] = options!.redirectTo!;
  }
  Uri uri = Uri.parse(url);
  uri = uri.replace(queryParameters: {...uri.queryParameters, ...qs});

  return await _handleRequest(
      method: method, uri: uri, options: options, headers: headers);
}
```

The `request()` method is the main entry point for making API calls. It handles:

1. **API Version Header**: Automatically injects the API version header if not provided
2. **JWT Authentication**: Adds Bearer token to Authorization header when JWT is available
3. **Query Parameters**: Merges existing query parameters with additional options
4. **Redirect Handling**: Adds `redirect_to` parameter for email/link verification flows
5. **URI Construction**: Parses and rebuilds the URL with all query parameters

### Internal Request Handler

```dart
Future<dynamic> _handleRequest({
  required RequestMethodType method,
  required Uri uri,
  required GotrueRequestOptions? options,
  required Map<String, String> headers,
}) async {
  final bodyStr = json.encode(options?.body ?? {});

  if (method != RequestMethodType.get) {
    headers['Content-Type'] = 'application/json';
  }
  Response response;
  try {
    switch (method) {
      case RequestMethodType.get:
        response = await (httpClient?.get ?? get)(uri, headers: headers);
        break;
      case RequestMethodType.post:
        response = await (httpClient?.post ?? post)(uri, headers: headers, body: bodyStr);
        break;
      case RequestMethodType.put:
        response = await (httpClient?.put ?? put)(uri, headers: headers, body: bodyStr);
        break;
      case RequestMethodType.delete:
        response = await (httpClient?.delete ?? delete)(uri, headers: headers, body: bodyStr);
        break;
    }
  } catch (e) {
    // fetch failed, likely due to a network or CORS error
    throw AuthRetryableFetchException(message: e.toString());
  }

  if (!isSuccessStatusCode(response.statusCode)) {
    throw _handleError(response);
  }

  if (options?.noResolveJson == true) {
    return response.body;
  }

  try {
    final bodyString = utf8.decode(response.bodyBytes);
    if (bodyString.isEmpty) {
      return <String, dynamic>{};
    }
    return json.decode(bodyString);
  } catch (error) {
    throw _handleError(error);
  }
}
```

The `_handleRequest()` method performs the actual HTTP communication:

**Request Building:**

- JSON-encodes the request body for non-GET requests
- Sets `Content-Type: application/json` header for methods that send data

**HTTP Execution:**

- Uses the provided `httpClient` if available, otherwise falls back to top-level `get`, `post`, `put`, `delete` functions
- Each HTTP method is handled in a dedicated case branch
- Network errors (connection refused, DNS failure, etc.) are caught and converted to `AuthRetryableFetchException`

**Response Processing:**

- Non-success status codes trigger `_handleError()` for proper exception conversion
- If `noResolveJson` is true, returns the raw response body as a string
- Otherwise, attempts UTF-8 decoding followed by JSON parsing
- Empty response bodies return an empty map
- JSON parsing failures trigger `_handleError()`

## Exception Types

The module defines several exception types for different error scenarios:

| Exception | Usage | Retryable |
|-----------|-------|-----------|
| `AuthRetryableFetchException` | Network errors, server errors (5xx) | Yes |
| `AuthApiException` | Client errors (4xx) with known codes | No |
| `AuthWeakPasswordException` | Password validation failures | No |
| `AuthUnknownException` | Unexpected error formats | No |

## Usage Example

```dart
// Create fetch instance with custom HTTP client
final fetch = GotrueFetch(customHttpClient);

// Make a POST request with JWT authentication
final response = await fetch.request(
  'https://example.com/auth/v1/signup',
  RequestMethodType.post,
  options: GotrueRequestOptions(
    body: {'email': 'user@example.com', 'password': 'secure123'},
    jwt: 'user-jwt-token',
  ),
);
```

## Thread Safety

The `GotrueFetch` class is designed to be stateless and thread-safe. The optional `httpClient` parameter allows sharing HTTP client instances across multiple `GotrueFetch` instances for connection pooling.
