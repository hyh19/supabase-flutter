# GoTrueClient Event System and Broadcasting

This document explains the event system and broadcasting functionality in the `GoTrueClient` class, including how authentication state changes are communicated to subscribers.

## Event System Overview

The GoTrueClient uses a publish-subscribe pattern to notify applications about authentication state changes. This allows multiple parts of an application to react to authentication events without tight coupling.

## AuthState and AuthChangeEvent

### AuthState

The `AuthState` class represents an authentication state change event:

```dart 65:67:lib/src/gotrue_client.dart
  final _onAuthStateChangeController = BehaviorSubject<AuthState>();
  final _onAuthStateChangeControllerSync =
      BehaviorSubject<AuthState>(sync: true);
```

The `AuthState` contains:

- The event type that occurred
- The current session (if any)
- Whether the event came from a broadcast (other tab)

### AuthChangeEvent

The `AuthChangeEvent` enum defines all possible authentication state changes:

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

**Event types:**

| Event | Description | When emitted |
|-------|-------------|--------------|
| `initialSession` | Initial session restored | On app startup when recovering session |
| `passwordRecovery` | Password recovery flow | User clicks recovery link or verifies OTP |
| `signedIn` | User signed in | After successful authentication |
| `signedOut` | User signed out | After sign out or token refresh failure |
| `tokenRefreshed` | Token was refreshed | After successful token refresh |
| `userUpdated` | User data updated | After `updateUser()` or identity change |
| `mfaChallengeVerified` | MFA challenge passed | After successful MFA verification |

## Stream Controllers

### Primary Stream Controller

```dart 65:66:lib/src/gotrue_client.dart
  final _onAuthStateChangeController = BehaviorSubject<AuthState>();
```

The main stream controller for authentication state changes. This is the primary stream that applications should subscribe to.

### Synchronous Stream Controller

```dart 66:67:lib/src/gotrue_client.dart
  final _onAuthStateChangeControllerSync =
      BehaviorSubject<AuthState>(sync: true);
```

A synchronous version of the stream controller, used internally for immediate event delivery. The `sync: true` parameter ensures events are delivered synchronously to subscribers.

## Public Stream Access

### onAuthStateChange

The primary stream for subscribing to authentication state changes:

```dart 83:84:lib/src/gotrue_client.dart
  Stream<AuthState> get onAuthStateChange =>
      _onAuthStateChangeController.stream;
```

**Usage example:**

```dart
supabase.auth.onAuthStateChange.listen((data) {
  final AuthChangeEvent event = data.event;
  final Session? session = data.session;
  if (event == AuthChangeEvent.signedIn) {
    // Handle sign in
  }
});
```

### onAuthStateChangeSync

Internal synchronous stream for immediate event delivery:

```dart 87:89:lib/src/gotrue_client.dart
  @internal
  Stream<AuthState> get onAuthStateChangeSync =>
      _onAuthStateChangeControllerSync.stream;
```

This stream is marked as `@internal` and should not be used directly by application code.

## Event Notification

### notifyAllSubscribers

The main method for broadcasting authentication state changes:

```dart 1324:1340:lib/src/gotrue_client.dart
  /// For internal use only.
  ///
  /// [broadcast] is used to determine if the event should be broadcasted to
  /// other tabs.
  @internal
  void notifyAllSubscribers(
    AuthChangeEvent event, {
    Session? session,
    bool broadcast = true,
  }) {
    session ??= currentSession;
    if (broadcast && event != AuthChangeEvent.initialSession) {
      _broadcastChannel?.postMessage({
        'event': event.jsName,
        'session': session?.toJson(),
      });
    }
    final state = AuthState(event, session, fromBroadcast: !broadcast);
    _log.finest('onAuthStateChange: $state');
    _onAuthStateChangeController.add(state);
    _onAuthStateChangeControllerSync.add(state);
  }
```

The method:

1. Uses the current session if none is provided
2. Posts the event to the broadcast channel (for cross-tab communication)
3. Creates an `AuthState` object with the event and session
4. Adds the state to both stream controllers

**Parameters:**

- `event`: The type of authentication event that occurred
- `session`: Optional session to include with the event
- `broadcast`: Whether to broadcast to other browser tabs (default: true)

### notifyException

Reports exceptions to the error stream:

```dart 1344:1351:lib/src/gotrue_client.dart
  /// For internal use only.
  @internal
  Object notifyException(Object exception, [StackTrace? stackTrace]) {
    _log.warning('Notifying exception', exception, stackTrace);
    _onAuthStateChangeController.addError(
      exception,
      stackTrace ?? StackTrace.current,
    );
    return exception;
  }
```

This method adds errors to the stream controller, allowing subscribers to catch authentication-related exceptions.

## Broadcast Channel (Web Only)

### Cross-Tab Communication

On web platforms, the `BroadcastChannel` API is used to synchronize authentication state across multiple browser tabs:

```dart 95:98:lib/src/gotrue_client.dart
  /// Proxy to the web BroadcastChannel API. Should be null on non-web platforms.
  BroadcastChannel? _broadcastChannel;

  StreamSubscription? _broadcastChannelSubscription;
```

### _mayStartBroadcastChannel

Initializes the broadcast channel on web platforms:

```dart 1205:1253:lib/src/gotrue_client.dart
  void _mayStartBroadcastChannel() {
    if (const bool.fromEnvironment('dart.library.js_interop')) {
      // Used by the js library as well
      final broadcastKey =
          "sb-${Uri.parse(_url).host.split(".").first}-auth-token";

      assert(_broadcastChannel == null,
          'Broadcast channel should not be started more than once.');
      try {
        _broadcastChannel = web.getBroadcastChannel(broadcastKey);
        _broadcastChannelSubscription =
            _broadcastChannel?.onMessage.listen((messageEvent) {
          final rawEvent = messageEvent['event'];
          _log.finest('Received broadcast message: $messageEvent');
          _log.info('Received broadcast event: $rawEvent');
          final event = switch (rawEvent) {
            // This library sends the js name of the event to be comptabile with
            // the js library, so we need to convert it back to the dart name
            'INITIAL_SESSION' => AuthChangeEvent.initialSession,
            'PASSWORD_RECOVERY' => AuthChangeEvent.passwordRecovery,
            'SIGNED_IN' => AuthChangeEvent.signedIn,
            'SIGNED_OUT' => AuthChangeEvent.signedOut,
            'TOKEN_REFRESHED' => AuthChangeEvent.tokenRefreshed,
            'USER_UPDATED' => AuthChangeEvent.userUpdated,
            'MFA_CHALLENGE_VERIFIED' => AuthChangeEvent.mfaChallengeVerified,
            // This case should never happen though
            _ => AuthChangeEvent.values
                .firstWhereOrNull((event) => event.name == rawEvent),
          };

          if (event != null) {
            Session? session;
            if (messageEvent['session'] != null) {
              session = Session.fromJson(messageEvent['session']);
            }
            if (session != null) {
              _saveSession(session);
            } else {
              _removeSession();
            }
            notifyAllSubscribers(event, session: session, broadcast: false);
          }
        });
      } catch (error, stackTrace) {
        _log.warning('Failed to start broadcast channel', error, stackTrace);
        // Ignoring
      }
    }
  }
```

The method:

1. Checks if running in a web environment using `dart.library.js_interop`
2. Creates a broadcast channel key based on the GoTrue URL host
3. Listens for messages from other tabs
4. Converts JavaScript-style event names back to Dart enum values
5. Updates the local session and notifies subscribers

**Broadcast channel key format:**

```
sb-{host-first-part}-auth-token
```

For example, for `https://example.supabase.co`, the key would be `sb-example-auth-token`.

## Event Flow Examples

### Sign In Event Flow

```dart
// User successfully signs in
_saveSession(session);
notifyAllSubscribers(AuthChangeEvent.signedIn);
// Subscribers receive:
// AuthState(event: AuthChangeEvent.signedIn, session: Session)
```

### Token Refresh Flow

```dart
// Token is refreshed
_saveSession(session);
notifyAllSubscribers(AuthChangeEvent.tokenRefreshed);
// Subscribers receive:
// AuthState(event: AuthChangeEvent.tokenRefreshed, session: Session)
```

### Sign Out Flow

```dart
// User signs out
_removeSession();
notifyAllSubscribers(AuthChangeEvent.signedOut);
// Subscribers receive:
// AuthState(event: AuthChangeEvent.signedOut, session: null)
```

### Cross-Tab Sign Out

When user signs out in one tab:

1. `signOut()` is called
2. `_removeSession()` clears the local session
3. `notifyAllSubscribers()` posts to broadcast channel
4. Other tabs receive the message and:
   - Clear their local sessions
   - Call `notifyAllSubscribers(event, broadcast: false)` to notify their subscribers

## Disposing Resources

### dispose

Cleans up stream controllers and subscriptions:

```dart 1255:1263:lib/src/gotrue_client.dart
  @mustCallSuper
  void dispose() {
    _onAuthStateChangeController.close();
    _onAuthStateChangeControllerSync.close();
    _broadcastChannel?.close();
    _broadcastChannelSubscription?.cancel();
    _refreshTokenCompleter?.completeError(AuthException('Disposed'));
    _autoRefreshTicker?.cancel();
  }
```

This method should be called when the client is no longer needed to prevent memory leaks.

## Summary

The event system provides:

| Component | Purpose |
|-----------|---------|
| `onAuthStateChange` | Primary stream for auth state changes |
| `onAuthStateChangeSync` | Internal synchronous stream |
| `notifyAllSubscribers()` | Broadcast events to subscribers |
| `notifyException()` | Report errors to error stream |
| `BroadcastChannel` | Cross-tab synchronization on web |
| `dispose()` | Clean up resources |

Key features:

- **Multi-tab sync**: Changes in one tab are reflected in all open tabs
- **Dual streams**: Async and sync streams for different use cases
- **Error handling**: Exceptions are propagated through the error stream
- **Resource cleanup**: Proper disposal prevents memory leaks
