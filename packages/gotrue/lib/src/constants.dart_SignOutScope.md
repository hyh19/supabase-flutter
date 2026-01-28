# SignOutScope Enum

## Overview

The `SignOutScope` enum is defined in the constants file and determines the scope or range of sessions that should be terminated when a user signs out of the application. This enum provides three distinct options for controlling logout behavior, allowing developers to implement different sign-out strategies based on their application's authentication requirements.

The enum is particularly useful in scenarios where users may have multiple active sessions across different devices or platforms, and you need granular control over which sessions are terminated during the logout process.

## Enum Values

### global

```dart 107:107:lib/src/constants.dart
  global,
```

The `global` option signs out all sessions associated with the user's account. When this scope is selected, the authentication system terminates every active session across all devices and platforms where the user might be logged in. This provides the most comprehensive logout experience and is typically used when:

- The user is signing out for security reasons (e.g., suspicious activity detected)
- The user explicitly requests to log out of all devices
- The application needs to ensure complete session termination (e.g., when changing passwords)
- Account security is a priority and you want to force re-authentication everywhere

### local

```dart 110:110:lib/src/constants.dart
  local,
```

The `local` option signs out only the current session where the sign-out operation is being performed. This means that if the user is logged in on multiple devices, only the device from which they initiated the sign-out will be logged out. Other sessions on different devices remain active and unaffected.

This scope is appropriate when:

- The user is simply logging out from one device (e.g., logging out of a shared computer)
- You want to preserve the user's session on their primary device
- The logout is routine and not security-related
- Users often use the application on multiple devices simultaneously

### others

```dart 113:113:lib/src/constants.dart
  others,
```

The `others` option signs out all sessions except the current one. This means that all other active sessions on different devices will be terminated, while the current session remains active. This is particularly useful for:

- Security scenarios where you suspect other sessions may be compromised
- Forcing re-authentication on unknown or untrusted devices
- Maintaining the current session while cleaning up all other sessions
- Situations where the user wants to keep their current device logged in but log out everywhere else

**Important Note**: When using the `others` scope, there is no `AuthChangeEvent.signedOut` event fired on the current session. This is because the current session remains active and is not terminated. The event would only be triggered for the other sessions that are being signed out.

## Usage Context

This enum is typically used in conjunction with the sign-out functionality of the GoTrue client. The scope parameter allows developers to control the breadth of the logout operation, providing flexibility in how authentication sessions are managed across different contexts.

When implementing sign-out functionality, developers should consider:

1. **Security requirements**: If security is paramount, prefer `global` to ensure complete session termination
2. **User experience**: For routine logouts, `local` provides a smoother experience by not affecting other devices
3. **Selective termination**: Use `others` when you need to maintain the current session while revoking access to all other devices

## Relationship with AuthChangeEvent

The `SignOutScope` enum interacts with the `AuthChangeEvent` enum, specifically the `signedOut` event. When sessions are terminated, the authentication system may fire `AuthChangeEvent.signedOut` events to notify the application of the state change. However, as documented for the `others` scope, this event is not fired for the current session since it remains active.
