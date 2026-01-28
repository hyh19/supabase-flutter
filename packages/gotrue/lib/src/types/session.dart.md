# Session Class

The `Session` class represents an authenticated user session in the GoTrue authentication system. It encapsulates all the token and user information needed to maintain and validate an authenticated state.

## Overview

When a user logs in, the GoTrue server returns a session object containing authentication tokens and user details. The `Session` class provides a structured way to handle this data, including JSON serialization, expiration checking, and immutable copy operations.

## Class Structure

### Instance Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `accessToken` | `String` | Yes | The JWT access token used for authenticated API requests |
| `tokenType` | `String` | Yes | The type of token (typically "bearer") |
| `user` | `User` | Yes | The authenticated user's information |
| `expiresIn` | `int?` | No | Seconds until the token expires from issuance time |
| `refreshToken` | `String?` | No | Token used to obtain a new access token |
| `providerToken` | `String?` | No | OAuth provider's access token (for OAuth flows) |
| `providerRefreshToken` | `String?` | No | OAuth provider's refresh token (for OAuth flows) |

### Derived Properties

| Property | Type | Description |
|----------|------|-------------|
| `expiresAt` | `int?` | Unix timestamp of when the token expires, extracted from the JWT payload |

## Key Methods

### fromJson

```dart
static Session? fromJson(Map<String, dynamic> json)
```

Creates a `Session` instance from a JSON map.

**Behavior:**

- Returns `null` if `access_token` is missing (invalid session)
- Parses all token fields from the JSON object
- Constructs a `User` object from the nested user data

**Example:**

```dart
final session = Session.fromJson(jsonResponse);
if (session != null) {
  print('User: ${session.user.email}');
}
```

### toJson

```dart
Map<String, dynamic> toJson()
```

Converts the session to a JSON-serializable map.

**Includes:**

- All token fields
- The `expires_at` derived property (not `expiresIn`)
- Nested user object

**Use Case:** Persisting session to local storage or transmitting over the network.

### copyWith

```dart
Session copyWith({
  String? accessToken,
  int? expiresIn,
  String? refreshToken,
  String? tokenType,
  String? providerToken,
  String? providerRefreshToken,
  User? user,
})
```

Creates a new `Session` instance with optional field replacements.

**Behavior:**

- Only replaces fields that are explicitly provided
- Keeps all other fields unchanged from the original instance

**Example:**

```dart
final updatedSession = session.copyWith(
  accessToken: newAccessToken,
  refreshToken: newRefreshToken,
);
```

### isExpired (Getter)

```dart
bool get isExpired
```

Determines whether the session token has expired or will expire soon.

**Logic:**

- Returns `false` if `expiresAt` is `null` (cannot determine expiration)
- Adds a 10-second buffer via `Constants.expiryMargin` to account for network latency
- Compares the expiration time against the current time

**Example:**

```dart
if (session.isExpired) {
  // Trigger token refresh or re-authentication
  await authClient.refreshSession();
}
```

## Token Expiration

The expiration handling uses JWT token claims:

1. **Extraction:** The `expiresAt` value is extracted from the JWT payload's `exp` claim using `Jwt.parseJwt()`.
2. **Safety Margin:** A configurable buffer (via `Constants.expiryMargin`) prevents edge cases where the token expires mid-request due to latency.
3. **Fallback:** If JWT parsing fails, `expiresAt` becomes `null` and `isExpired` returns `false`.

## OAuth Provider Tokens

For OAuth-based authentication (e.g., Google, GitHub sign-in), the session includes additional tokens:

- `providerToken`: The OAuth provider's access token (can be used to call provider APIs)
- `providerRefreshToken`: The OAuth provider's refresh token (to renew the provider token)

These are stored separately from the Supabase-issued tokens.

## Equality and Hashing

The class implements value-based equality:

```dart
@override
bool operator ==(Object other)

@override
int get hashCode
```

Two sessions are equal if all their properties match. This enables proper use in collections like `Set` and `Map`.

## Usage Example

```dart
// Parse a session from API response
final session = Session.fromJson(response.data)!;

// Check if session is still valid
if (!session.isExpired) {
  // Make authenticated API calls using accessToken
  final response = await supabaseClient.from('profiles').select().execute();
}

// Refresh the session when expired
if (session.isExpired) {
  final newSession = await client.refreshSession(session.refreshToken!);
  // newSession is a new Session instance with fresh tokens
}

// Persist session for later use
final json = session.toJson();
await storage.write('session', json);

// Restore session
final restoredSession = Session.fromJson(storedJson)!;
```

## See Also

- [User class](./user.dart.md) - User information within a session
- [AuthClient](../gotrue_client.dart.md) - Authentication client that manages sessions
- [Constants](./constants.dart.md) - Configuration values like expiry margin
