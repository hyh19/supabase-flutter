# Auth Response Types Documentation

This file defines the response types used throughout the GoTrue authentication client. These classes represent various authentication outcomes, from successful logins to OAuth flows and email link generation.

## Class Overview

### AuthResponse

The `AuthResponse` class represents a generic authentication response that may or may not contain session and user information. This is the primary response type returned by most authentication methods.

```dart 5:18:packages/gotrue/lib/src/types/auth_response.dart
class AuthResponse {
  final Session? session;
  final User? user;

  AuthResponse({
    this.session,
    User? user,
  }) : user = user ?? session?.user;

  /// Instanciates an `AuthResponse` object from json response.
  AuthResponse.fromJson(Map<String, dynamic> json)
      : session = Session.fromJson(json),
        user = User.fromJson(json) ?? Session.fromJson(json)?.user;
}
```

**Key Design Points:**

- **Optional Properties**: Both `session` and `user` are nullable, reflecting that some auth operations may return only one or the other.
- **Smart User Assignment**: The constructor uses a fallback mechanism where `user` defaults to `session?.user` if not explicitly provided.
- **JSON Deserialization**: The `fromJson` constructor handles parsing API responses, extracting both session and user data from the same JSON structure.

---

### OAuthResponse

The `OAuthResponse` class encapsulates the result of initiating an OAuth authentication flow.

```dart 21:30:packages/gotrue/lib/src/types/auth_response.dart
class OAuthResponse {
  final OAuthProvider provider;
  final String url;

  /// Instanciates an `OAuthResponse` object from json response.
  const OAuthResponse({
    required this.provider,
    required this.url,
  });
}
```

**Key Design Points:**

- **Immutable Design**: Uses `const` constructor for compile-time safety and performance.
- **Required Fields**: Both `provider` and `url` are required parameters, ensuring valid OAuth responses.
- **Provider Enum**: The `OAuthProvider` enum identifies which OAuth provider was used (Google, GitHub, etc.).

---

### UserResponse

The `UserResponse` class is a simple wrapper for responses containing user information.

```dart 33:37:packages/gotrue/lib/src/types/auth_response.dart
class UserResponse {
  final User? user;

  UserResponse.fromJson(Map<String, dynamic> json) : user = User.fromJson(json);
}
```

**Use Case**: Typically returned by user profile retrieval or update operations where no session data is involved.

---

### ResendResponse

The `ResendResponse` class handles responses for resending verification codes, primarily used for phone OTP resending.

```dart 39:46:packages/gotrue/lib/src/types/auth_response.dart
class ResendResponse {
  /// Only set for phone resend
  String? messageId;

  ResendResponse({
    this.messageId,
  });
}
```

**Key Design Points:**

- **Nullable messageId**: The `messageId` is only populated for phone resend operations, not for email resends.
- **Flexibility**: The class structure allows for future expansion if additional resend-related data needs to be tracked.

---

### AuthSessionUrlResponse

The `AuthSessionUrlResponse` class represents responses containing both a session and OAuth redirect information.

```dart 48:56:packages/gotrue/lib/src/types/auth_response.dart
class AuthSessionUrlResponse {
  final Session session;
  final String? redirectType;

  const AuthSessionUrlResponse({
    required this.session,
    required this.redirectType,
  });
}
```

**Use Case**: Returned after OAuth callback processing where the session is established and the redirect type (e.g., "code" for PKCE, "token" for implicit flow) needs to be communicated.

---

### GenerateLinkResponse

The `GenerateLinkResponse` class encapsulates the result of generating an email magic link or OTP verification link.

```dart 58:65:packages/gotrue/lib/src/types/auth_response.dart
class GenerateLinkResponse {
  final GenerateLinkProperties properties;
  final User user;

  GenerateLinkResponse.fromJson(Map<String, dynamic> json)
      : properties = GenerateLinkProperties.fromJson(json),
        user = User.fromJson(json)!;
}
```

**Key Design Points:**

- **Non-nullable Properties**: Both `properties` and `user` are required.
- **Defensive Programming**: Uses `!` assertion on `User.fromJson()` assuming the API always returns user data in this context.

---

### GenerateLinkProperties

The `GenerateLinkProperties` class contains all the details needed to construct and send verification links to users.

```dart 67:92:packages/gotrue/lib/src/types/auth_response.dart
class GenerateLinkProperties {
  /// The email link to send to the user.
  /// The action_link follows the following format: auth/v1/verify?type={verification_type}&token={hashed_token}&redirect_to={redirect_to}
  final String actionLink;

  /// The raw email OTP.
  /// You should send this in the email if you want your users to verify using an OTP instead of the action link.
  final String emailOtp;

  /// The hashed token appended to the action link.
  final String hashedToken;

  /// The URL appended to the action link.
  final String redirectTo;

  /// The verification type that the email link is associated to.
  final GenerateLinkType verificationType;

  GenerateLinkProperties.fromJson(Map<String, dynamic> json)
      : actionLink = json['action_link'] ?? '',
        emailOtp = json['email_otp'] ?? '',
        hashedToken = json['hashed_token'] ?? '',
        redirectTo = json['redirect_to'] ?? '',
        verificationType =
            GenerateLinkTypeExtended.fromString(json['verification_type']);
}
```

**Property Details:**

| Property | Description |
|----------|-------------|
| `actionLink` | The complete verification URL with token and parameters. Format: `auth/v1/verify?type={type}&token={token}&redirect_to={url}` |
| `emailOtp` | The raw OTP code for email-based verification (alternative to links) |
| `hashedToken` | The token component used in the verification link |
| `redirectTo` | The URL to redirect users to after verification |
| `verificationType` | The type of verification (signup, recovery, email change, etc.) |

**Default Values**: All string properties default to empty strings if not present in the JSON response.

---

## Extension: ToSnakeCase

```dart 94:111:packages/gotrue/lib/src/types/auth_response.dart
extension ToSnakeCase on Enum {
  String get snakeCase {
    final a = 'a'.codeUnitAt(0), z = 'z'.codeUnitAt(0);
    final A = 'A'.codeUnitAt(0), Z = 'Z'.codeUnitAt(0);
    final result = StringBuffer()..write(name[0].toLowerCase());
    for (var i = 1; i < name.length; i++) {
      final char = name.codeUnitAt(i);
      if (A <= char && char <= Z) {
        final pChar = name.codeUnitAt(i - 1);
        if (a <= pChar && pChar <= z) {
          result.write('_');
        }
      }
      result.write(name[i].toLowerCase());
    }
    return result.toString();
  }
}
```

**Purpose**: Converts Dart enum names to snake_case format for API compatibility.

**Algorithm Breakdown:**

1. **First Character**: Always lowercase for the start of the snake_case string.
2. **Subsequent Characters**: For each uppercase letter, check if the previous character was lowercase. If so, insert an underscore before the uppercase letter.
3. **Conversion**: All characters are converted to lowercase in the result.

**Example Conversions:**

| Enum Name | Snake Case |
|-----------|------------|
| `GenerateLinkType.signup` | `signup` |
| `OAuthProvider.google` | `oauth_provider.google` (extension method returns `oauth_provider` for `OAuthProvider`) |
| `EmailOtpType.magicLink` | `email_otp_type.magic_link` |

---

## Usage Examples

### Parsing an Auth Response

```dart
final response = AuthResponse.fromJson(jsonData);
print(response.session?.accessToken);
print(response.user?.email);
```

### Handling OAuth Flow

```dart
final oauthResponse = OAuthResponse(
  provider: OAuthProvider.google,
  url: 'https://accounts.google.com/oauth...',
);
// Redirect user to oauthResponse.url
```

### Generating Email Links

```dart
final linkResponse = GenerateLinkResponse.fromJson(jsonData);
print(linkResponse.properties.actionLink);
print(linkResponse.properties.emailOtp);
```

---

## Class Hierarchy

```text
Enum
    └── ToSnakeCase extension

AuthResponse
    ├── Session?
    └── User?

OAuthResponse
    ├── OAuthProvider
    └── String

UserResponse
    └── User?

ResendResponse
    └── String? (messageId)

AuthSessionUrlResponse
    ├── Session
    └── String? (redirectType)

GenerateLinkResponse
    ├── GenerateLinkProperties
    └── User

GenerateLinkProperties
    ├── String (actionLink)
    ├── String (emailOtp)
    ├── String (hashedToken)
    ├── String (redirectTo)
    └── GenerateLinkType (verificationType)
```

---

## Design Patterns Used

1. **Factory Pattern**: `fromJson` static methods for object construction from API responses.
2. **Null Safety**: Leveraging Dart's null safety with nullable types and fallback logic.
3. **Extension Methods**: Adding utility methods to existing types (Enum).
4. **Immutability**: Using `const` constructors where possible for performance and safety.
5. **Composition**: Nested response types (e.g., `GenerateLinkResponse` contains `GenerateLinkProperties`).
