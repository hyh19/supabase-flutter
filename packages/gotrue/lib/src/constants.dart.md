# constants.dart

## Overview

The `constants.dart` file serves as the central configuration and type definition hub for the GoTrue Dart client library. It establishes fundamental constants used throughout the authentication system, defines API version specifications, and declares enums that represent authentication events, OTP types, and session management behaviors. This file provides a single source of truth for all static configuration values and type-safe enumerations that the rest of the library relies upon.

## Constants Class

The `Constants` class encapsulates all static configuration values used by the GoTrue client. These constants define default behaviors, time-based thresholds, and identifier strings that govern how the authentication client operates. By centralizing these values, the library ensures consistent behavior across all components while allowing for easy customization through client configuration.

### Default Configuration Values

```dart 5:30:lib/src/constants.dart
class Constants {
  static const String defaultGotrueUrl = 'http://localhost:9999';
  static const String defaultAudience = '';
  static const Map<String, String> defaultHeaders = {
    'X-Client-Info': 'gotrue-dart/$version',
  };
  static const int defaultExpiryMargin = 60 * 1000;

  /// storage key prefix to store code verifiers
  static const String defaultStorageKey = 'supabase.auth.token';

  /// The margin to use when checking if a token is expired.
  static const expiryMargin = Duration(seconds: 30);

  /// Current session will be checked for refresh at this interval.
  static const autoRefreshTickDuration = Duration(seconds: 10);

  /// A token refresh will be attempted this many ticks before the current session expires.
  static const autoRefreshTickThreshold = 3;

  /// The name of the header that contains API version.
  static const apiVersionHeaderName = 'x-supabase-api-version';

  /// The TTL for the JWKS cache.
  static const jwksTtl = Duration(minutes: 10);
}
```

The `defaultGotrueUrl` provides a placeholder URL (`http://localhost:9999`) that serves as the fallback endpoint when no custom GoTrue server URL is configured. This value reflects the common development scenario where developers run GoTrue locally during development and testing phases. In production environments, this value should be overridden with the actual GoTrue server endpoint.

The `defaultAudience` is an empty string that represents the default JWT audience claim. The audience claim in JWT tokens identifies the intended recipients of the token, and an empty string indicates that no specific audience validation is required by default. Applications that require audience validation can specify their own audience when initializing the client.

The `defaultHeaders` map establishes the default HTTP headers that the GoTrue client includes in all API requests. The `X-Client-Info` header contains the library name and version number, which helps server administrators identify the client library making requests. This header is valuable for debugging, analytics, and ensuring proper compatibility between client and server versions.

The `defaultExpiryMargin` is set to 60,000 milliseconds (60 seconds) and represents the margin used when calculating token expiration. This value is part of the legacy configuration system and has been superseded by the more explicit `expiryMargin` duration constant.

### Token Refresh Configuration

```dart 17:23:lib/src/constants.dart
  /// The margin to use when checking if a token is expired.
  static const expiryMargin = Duration(seconds: 30);

  /// Current session will be checked for refresh at this interval.
  static const autoRefreshTickDuration = Duration(seconds: 10);

  /// A token refresh will be attempted this many ticks before the current session expires.
  static const autoRefreshTickThreshold = 3;
```

The `expiryMargin` of 30 seconds defines the buffer time before a token is considered expired. When checking token validity, the client considers a token expired if it will expire within this margin, allowing sufficient time to proactively refresh the token before it becomes invalid.

The `autoRefreshTickDuration` of 10 seconds determines how frequently the client checks whether a session needs refreshing. This polling-based approach ensures that token refresh happens in a timely manner without consuming excessive system resources. The 10-second interval balances responsiveness with efficiency.

The `autoRefreshTickThreshold` of 3, combined with the tick duration, means that a token refresh attempt will occur when the token expires in 3 * 10 = 30 seconds or less. This calculation ensures that refresh operations have adequate time to complete before the token actually expires, accounting for potential network latency and processing time.

### Storage and API Configuration

```dart 13:14:lib/src/constants.dart
  /// storage key prefix to store code verifiers
  static const String defaultStorageKey = 'supabase.auth.token';
```

The `defaultStorageKey` defines the prefix used when storing authentication tokens in persistent storage. The value `supabase.auth.token` follows a naming convention that clearly identifies the purpose of the stored data while maintaining compatibility with the Supabase ecosystem. This key is used as a prefix for storing various authentication-related items including code verifiers for PKCE flow.

```dart 25:29:lib/src/constants.dart
  /// The name of the header that contains API version.
  static const apiVersionHeaderName = 'x-supabase-api-version';

  /// The TTL for the JWKS cache.
  static const jwksTtl = Duration(minutes: 10);
```

The `apiVersionHeaderName` specifies the HTTP header name (`x-supabase-api-version`) used to communicate API version information between client and server. This header allows the server to respond with version-appropriate data and enables clients to understand which API version is being used.

The `jwksTtl` of 10 minutes sets the cache duration for JSON Web Key Set (JWKS) data. The JWKS contains the public keys used to verify JWT signatures. Caching this data reduces network requests and improves performance while ensuring that key rotation is detected within a reasonable timeframe.

## ApiVersions Class

```dart 32:37:lib/src/constants.dart
class ApiVersions {
  static final v20240101 = ApiVersion(
    name: '2024-01-01',
    timestamp: DateTime.parse('2024-01-01T00:00:00.0Z'),
  );
}
```

The `ApiVersions` class provides named references to supported API versions. Currently, it defines only `v20240101` representing the API version released on January 1, 2024. Each API version is represented by an `ApiVersion` object containing a human-readable name and a timestamp indicating when that version was released.

This class serves as a registry of known API versions, making it easy for the client to reference specific versions without hardcoding version strings throughout the codebase. As the GoTrue API evolves with new versions, they can be added to this class to maintain a centralized record of supported versions.

## AuthChangeEvent Enum

```dart 39:53:lib/src/constants.dart
enum AuthChangeEvent {
  initialSession('INITIAL_SESSION'),
  passwordRecovery('PASSWORD_RECOVERY'),
  signedIn('SIGNED_IN'),
  signedOut('SIGNED_OUT'),
  tokenRefreshed('TOKEN_REFRESHED'),
  userUpdated('USER_UPDATED'),

  @Deprecated('Was never in use and might be removed in the future.')
  userDeleted(''),
  mfaChallengeVerified('MFA_CHALLENGE_VERIFIED');

  final String jsName;
  const AuthChangeEvent(this.jsName);
}
```

The `AuthChangeEvent` enum defines all possible authentication state changes that can occur during the application lifecycle. Each enum value is associated with a JavaScript-compatible string name (`jsName`) that is used when broadcasting events to listeners, particularly for JavaScript/Flutter web applications where the naming convention must match JavaScript expectations.

The enum includes the following events:

- **initialSession**: Triggered when the application starts and loads any existing session from storage. This event fires during initialization to notify listeners about pre-existing authentication state.
- **passwordRecovery**: Fired when a password recovery flow is initiated, typically after a user requests a password reset email.
- **signedIn**: Occurs when a user successfully signs in, whether through email/password, OAuth, or any other authentication method.
- **signedOut**: Triggered when a user signs out, either explicitly through a sign-out call or implicitly through token expiration.
- **tokenRefreshed**: Fired when a session token is successfully refreshed before expiration, indicating that the user's session has been extended.
- **userUpdated**: Occurs when user attributes are modified, such as when email, metadata, or other profile information is updated.
- **userDeleted**: Marked as deprecated, this event was intended to fire when a user account is deleted but was never properly implemented.
- **mfaChallengeVerified**: Triggered when a multi-factor authentication challenge is successfully completed, confirming the user's identity through an additional verification step.

### AuthChangeEventExtended Extension

```dart 55:64:lib/src/constants.dart
extension AuthChangeEventExtended on AuthChangeEvent {
  static AuthChangeEvent? fromString(String? val) {
    for (final event in AuthChangeEvent.values) {
      if (event.name == val) {
        return event;
      }
    }
    return null;
  }
}
```

The `AuthChangeEventExtended` extension adds a utility method `fromString` to the `AuthChangeEvent` enum. This method enables parsing a string value into the corresponding enum instance by comparing the input string against each enum value's `name` property. The method returns `null` if no matching event is found, making it safe to use with potentially unknown or future event types.

This extension is particularly useful when deserializing authentication events from network responses, storage, or inter-component communication where events are represented as strings rather than enum values.

## GenerateLinkType Enum

```dart 66:85:lib/src/constants.dart
enum GenerateLinkType {
  signup,
  invite,
  magiclink,
  recovery,
  emailChangeCurrent,
  emailChangeNew,
  unknown,
}

extension GenerateLinkTypeExtended on GenerateLinkType {
  static GenerateLinkType fromString(String? val) {
    for (final type in GenerateLinkType.values) {
      if (type.snakeCase == val) {
        return type;
      }
    }
    return GenerateLinkType.unknown;
  }
}
```

The `GenerateLinkType` enum represents the different types of authentication links that can be generated for users. These link types correspond to various email-based authentication flows supported by the GoTrue server.

The enum includes:

- **signup**: Used when generating an email link for new user registration. Clicking this link completes the sign-up process and creates a new user account.
- **invite**: Generated when an existing user is invited to join an organization or when an admin creates an invitation for a new user.
- **magiclink**: A passwordless authentication link that allows users to sign in without credentials by clicking a link sent to their email.
- **recovery**: Used for password reset flows, allowing users to create a new password after proving ownership of their email address.
- **emailChangeCurrent**: Generated when a user requests to change their email address; this link confirms the change from the current email address.
- **emailChangeNew**: Generated alongside emailChangeCurrent to confirm the change from the new email address.
- **unknown**: A fallback type used when the server returns an unrecognized link type.

### GenerateLinkTypeExtended Extension

The `GenerateLinkTypeExtended` extension provides the `fromString` method that parses string values into `GenerateLinkType` enum instances. Unlike the `AuthChangeEventExtended` parsing, this method uses `snakeCase` comparison rather than the enum's `name` property. This is because the server returns link types in snake_case format (e.g., `email_change_current`) while the enum uses camelCase naming convention (e.g., `emailChangeCurrent`).

If no matching type is found, the method returns `unknown` rather than `null`, providing a safe default for handling unrecognized link types.

## OtpType Enum

```dart 87:96:lib/src/constants.dart
enum OtpType {
  sms,
  phoneChange,
  signup,
  invite,
  magiclink,
  recovery,
  emailChange,
  email
}
```

The `OtpType` enum defines the various one-time password (OTP) delivery scenarios supported by the GoTrue client. Each type corresponds to a specific authentication context that requires OTP verification.

- **sms**: Standard SMS OTP delivery for phone number verification.
- **phoneChange**: OTP sent when a user requests to change their phone number, verifying ownership of the new number.
- **signup**: OTP sent during the registration process to verify the user's phone number.
- **invite**: OTP delivered when a user is invited via phone number and needs to accept the invitation.
- **magiclink**: OTP alternative to magic links for phone-based passwordless authentication.
- **recovery**: OTP sent for phone-based account recovery flows.
- **emailChange**: OTP sent when changing an email address (distinct from phone-based changes).
- **email**: Standard email OTP delivery for email verification.

## OtpChannel Enum

```dart 99:102:lib/src/constants.dart
/// Messaging channel to use (e.g. whatsapp or sms)
enum OtpChannel {
  sms,
  whatsapp,
}
```

The `OtpChannel` enum specifies the messaging delivery channel for OTP codes. Currently, it supports SMS and WhatsApp as delivery mechanisms. This enum allows developers to specify their preferred channel when sending OTPs, though the availability of each channel depends on the GoTrue server configuration and the user's capabilities.

## SignOutScope Enum

```dart 104:115:lib/src/constants.dart
/// Determines which sessions should be logged out.
enum SignOutScope {
  /// All sessions by this account will be signed out.
  global,

  /// Only this session will be signed out.
  local,

  /// All other sessions except the current one will be signed out. When using others, there is no [AuthChangeEvent.signedOut] event fired on the current session!
  others,
}
```

The `SignOutScope` enum controls the scope of sign-out operations, determining which sessions are terminated when a user signs out. This enum is crucial for applications that support multiple simultaneous sessions across different devices or browsers.

- **global**: Signs out the user from all sessions across all devices. This provides complete account security by terminating every active session, useful when a user suspects unauthorized access or when they want to ensure no other devices remain logged in.
- **local**: Signs out only the current session, leaving other active sessions untouched. This is appropriate for routine sign-outs where the user simply wants to log out of the current device without affecting their sessions on other devices.
- **others**: Signs out all sessions except the current one. This enables a user to clear potentially compromised sessions while maintaining their current authenticated state. Importantly, when using this scope, no `SIGNED_OUT` event is fired for the current session since it remains active.

## Architectural Significance

The `constants.dart` file plays a foundational role in the GoTrue client architecture. It establishes type-safe enums that prevent invalid state transitions and provide compile-time checking for authentication-related operations. The constants provide sensible defaults while maintaining flexibility for customization.

The separation of concerns within this file is notable: configuration constants are grouped in the `Constants` class, API versioning is handled separately, and authentication-related enums provide a comprehensive vocabulary for authentication state management. This organization makes the codebase more maintainable and easier to understand.

Extensions on enums provide convenient parsing utilities that bridge the gap between string representations used in storage and network communication, and the type-safe enum values used in application code. This pattern reduces boilerplate code throughout the library and centralizes parsing logic in a single location.
