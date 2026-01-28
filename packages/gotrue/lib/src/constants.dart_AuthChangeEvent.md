# AuthChangeEvent Enum and Extension

The `AuthChangeEvent` enum and its accompanying extension `AuthChangeEventExtended` define a set of authentication-related events that can occur within the GoTrue client library. These events are used to notify applications when significant authentication state changes happen, enabling reactive UI updates and business logic execution.

## AuthChangeEvent Enum

The `AuthChangeEvent` enum represents all possible authentication state change events that the Supabase client can emit. Each enum value is associated with a string identifier called `jsName`, which corresponds to the event name used in JavaScript environments (particularly for web applications and the Supabase JavaScript client).

### Enum Values and Their Meanings

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

**initialSession** (`INITIAL_SESSION`) — This event is emitted when the client first initializes and loads any existing session from storage. It represents the initial state of the authentication system before any user interaction occurs. Applications typically listen for this event to restore user sessions and determine if a user is already logged in when the application starts.

**passwordRecovery** (`PASSWORD_RECOVERY`) — This event is emitted when a user initiates the password recovery flow. It indicates that the user has requested a password reset email and is likely on a password reset page. Applications can use this event to show appropriate recovery UI components and prepare for password updates.

**signedIn** (`SIGNED_IN`) — This event is emitted when a user successfully signs into their account. This occurs after login operations such as email/password authentication, OAuth authentication, or magic link authentication complete successfully. It represents a transition to an authenticated state.

**signedOut** (`SIGNED_OUT`) — This event is emitted when a user signs out of their account. It indicates that the current session has been terminated and the client is transitioning to an unauthenticated state. Applications use this event to clear user-specific data, update UI to show login screens, and perform cleanup operations.

**tokenRefreshed** (`TOKEN_REFRESHED`) — This event is emitted when the client's access token has been successfully refreshed. Authentication tokens typically have limited lifespans, and the client automatically refreshes them before expiration. This event notifies listeners that a fresh token is now in use without requiring user re-authentication.

**userUpdated** (`USER_UPDATED`) — This event is emitted when user profile information or attributes are modified. This can include changes to user metadata, email addresses, or other user-specific data. Applications can respond to this event by refreshing cached user information and updating the UI accordingly.

**userDeleted** (empty string) — This enum value was deprecated and is marked for potential removal in future versions. It was intended to represent user deletion events but was never actually implemented or used in the system. The empty string value for `jsName` reflects that this event never had a proper JavaScript counterpart.

**mfaChallengeVerified** (`MFA_CHALLENGE_VERIFIED`) — This event is emitted when a user successfully completes a multi-factor authentication (MFA) challenge. MFA adds an extra layer of security by requiring users to verify their identity through a second factor (such as a code from an authenticator app). This event indicates that the MFA verification was successful and the authentication flow can proceed.

### Custom Enum Property

The enum includes a custom `jsName` property that stores the JavaScript-compatible string representation of each event. This design choice allows the Dart enum to map directly to the event names used in the JavaScript Supabase client, ensuring consistency across different platform implementations.

```dart 51:52:lib/src/constants.dart
  final String jsName;
  const AuthChangeEvent(this.jsName);
```

The constructor takes the `jsName` as a parameter and assigns it to the final property. This pattern is common in Dart when enum values need to carry additional metadata beyond their name.

## AuthChangeEventExtended Extension

The `AuthChangeEventExtended` extension adds static utility methods to the `AuthChangeEvent` enum, providing functionality for parsing string values back into enum instances.

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

### fromString Method

The `fromString` method converts a string value into a corresponding `AuthChangeEvent` enum value. It iterates through all available enum values and compares each one's `name` property against the input string.

**Method signature and implementation:**

```dart 56:63:lib/src/constants.dart
  static AuthChangeEvent? fromString(String? val) {
    for (final event in AuthChangeEvent.values) {
      if (event.name == val) {
        return event;
      }
    }
    return null;
  }
```

The method accepts a nullable `String` parameter (`String? val`), allowing it to safely handle null inputs without throwing exceptions. If the input matches any enum value's name, that enum value is returned. If no match is found, the method returns `null`.

This approach uses the enum's automatic `name` property (which Dart provides for all enums) rather than the custom `jsName` property. The `name` property returns the identifier used in the source code (e.g., `initialSession`, `signedIn`), while `jsName` contains the JavaScript-compatible string (e.g., `INITIAL_SESSION`, `SIGNED_IN`).

### Why Use an Extension Instead of a Class Method

Dart extensions provide a clean way to add functionality to existing types without modifying their original definition or creating wrapper classes. By using an extension, the GoTrue library keeps the enum definition simple while providing convenient utility methods that can be called directly on enum values.

The `static` keyword on the `fromString` method means it can be called without an instance: `AuthChangeEvent.fromString('signedIn')`. This is the typical pattern for factory-like methods that create enum values from external representations.

## Usage Patterns

Applications typically subscribe to auth change events using the GoTrue client's subscription mechanism. When any of these events occur, registered listeners are invoked with the corresponding event type, allowing the application to respond appropriately.

For converting string values back to enum instances (such as when deserializing event data from JSON), the `fromString` method provides a safe parsing mechanism:

```dart
final eventString = 'signedIn';
final event = AuthChangeEvent.fromString(eventString);

if (event != null) {
  // Handle the authenticated event
}
```

The nullable return type (`AuthChangeEvent?`) requires callers to handle the case where the input string does not correspond to a valid enum value, promoting defensive programming practices.
