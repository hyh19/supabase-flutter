# GoTrueClient Session Management

This document explains the session management functionality in the `GoTrueClient` class, including session lifecycle, token refresh, session recovery, and signing out.

## Session Lifecycle Overview

Session management in GoTrueClient involves several key operations:

1. **Session Creation**: Occurs after successful authentication via `signUp`, `signInWithPassword`, `verifyOTP`, etc.
2. **Session Storage**: The current session is stored in `_currentSession` and optionally persisted to async storage
3. **Session Refresh**: Access tokens are automatically refreshed before expiration
4. **Session Recovery**: Sessions can be recovered from stored JSON on app restart
5. **Session Termination**: Sessions are cleared on sign out or authentication failures

## Session Storage

### Internal Session Storage

```dart 1194:1198:lib/src/gotrue_client.dart
  /// set currentSession and currentUser
  void _saveSession(Session session) {
    _log.finest('Saving session: $session');
    _log.fine('Saving session');
    _currentSession = session;
  }
```

The `_saveSession` method stores the session in the private `_currentSession` field. This is called whenever a new session is established or refreshed.

### Session Removal

```dart 1200:1203:lib/src/gotrue_client.dart
  void _removeSession() {
    _log.fine('Removing session');
    _currentSession = null;
  }
```

The `_removeSession` method clears the current session, typically called during sign out or when token refresh fails.

## Sign Out Operations

### signOut

The `signOut` method handles signing out users with configurable scope:

```dart 841:873:lib/src/gotrue_client.dart
  /// Signs out the current user, if there is a logged in user.
  ///
  /// [scope] determines which sessions should be logged out.
  ///
  /// If using [SignOutScope.others] scope, no [AuthChangeEvent.signedOut] event is fired!
  Future<void> signOut({
    SignOutScope scope = SignOutScope.local,
  }) async {
    _log.info('Signing out user with scope: $scope');
    final accessToken = currentSession?.accessToken;

    if (scope != SignOutScope.others) {
      _removeSession();
      await _asyncStorage?.removeItem(
          key: '${Constants.defaultStorageKey}-code-verifier');
      notifyAllSubscribers(AuthChangeEvent.signedOut);
    }

    if (accessToken != null) {
      try {
        await admin.signOut(accessToken, scope: scope);
      } on AuthException catch (error) {
        // ignore 401s since an invalid or expired JWT should sign out the current session
        // ignore 403s since user might not exist anymore
        // ignore 404s since user might not exist anymore
        if (error.statusCode != '401' &&
            error.statusCode != '403' &&
            error.statusCode != '404') {
          rethrow;
        }
      }
    }
  }
```

The `signOut` method supports three different scopes defined by the `SignOutScope` enum:

```dart 105:114:lib/src/constants.dart
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

**Scope behaviors:**

- **`local` (default)**: Signs out only the current session, emits `signedOut` event
- **`global`**: Signs out all sessions across all devices, emits `signedOut` event
- **`others`**: Signs out all other sessions except the current one, does NOT emit `signedOut` event

The method gracefully handles exceptions:

- 401 (invalid/expired token): Ignored, as the token is already invalid
- 403/404 (user not found): Ignored, as the user may have been deleted
- Other errors: Rethrown

## Session Recovery

### setInitialSession

Sets the initial session from a stored JSON string, typically called on app startup:

```dart 1006:1017:lib/src/gotrue_client.dart
  /// Set the initial session to the session obtained from local storage
  Future<void> setInitialSession(String jsonStr) async {
    final session = Session.fromJson(json.decode(jsonStr));
    if (session == null) {
      // sign out to delete the local storage from supabase_flutter
      await signOut();
      throw notifyException(AuthException('Initial session is missing data.'));
    }

    _currentSession = session;
    notifyAllSubscribers(AuthChangeEvent.initialSession);
  }
```

This method:

1. Parses the JSON string into a `Session` object
2. If parsing fails, signs out and throws an exception
3. Stores the session and emits `initialSession` event
4. Does not trigger auto-refresh (that's handled separately)

### recoverSession

Recovers a session from a stringified JSON, with automatic refresh if expired:

```dart 1019:1055:lib/src/gotrue_client.dart
  /// Recover session from stringified [Session].
  Future<AuthResponse> recoverSession(String jsonStr) async {
    try {
      final session = Session.fromJson(json.decode(jsonStr));
      if (session == null) {
        _log.warning("Can't recover session from string, session is null");
        await signOut();
        throw notifyException(
          AuthException('Current session is missing data.'),
        );
      }

      if (session.isExpired) {
        _log.fine('Session from recovery is expired');
        final refreshToken = session.refreshToken;
        if (_autoRefreshToken && refreshToken != null) {
          return await _callRefreshToken(refreshToken);
        } else {
          await signOut();
          throw notifyException(AuthException('Session expired.'));
        }
      } else {
        final shouldEmitEvent = _currentSession == null ||
            _currentSession!.user.id != session.user.id;
        _saveSession(session);

        if (shouldEmitEvent) {
          notifyAllSubscribers(AuthChangeEvent.tokenRefreshed);
        }

        return AuthResponse(session: session);
      }
    } catch (error, stackTrace) {
      notifyException(error, stackTrace);
      rethrow;
    }
  }
```

The recovery logic:

1. Parses the session from JSON
2. If null, signs out and throws an exception
3. If expired:
   - If auto-refresh is enabled and refresh token exists, attempts refresh
   - Otherwise, signs out and throws an exception
4. If not expired:
   - Determines if event should be emitted (different user or no current session)
   - Saves the session
   - Emits `tokenRefreshed` event if needed
   - Returns the session

### setSession

Sets a session directly from a refresh token:

```dart 750:756:lib/src/gotrue_client.dart
  /// Sets the session data from refresh_token and returns the current session.
  Future<AuthResponse> setSession(String refreshToken) async {
    if (refreshToken.isEmpty) {
      throw AuthSessionMissingException('Refresh token cannot be empty');
    }
    return await _callRefreshToken(refreshToken);
  }
```

This is a convenience method that uses `_callRefreshToken` to establish a session from a refresh token.

## Token Refresh Mechanism

### refreshSession

Public method to manually refresh the current session:

```dart 621:636:lib/src/gotrue_client.dart
  /// Returns a new session, regardless of expiry status.
  /// Takes in an optional [refreshToken]. If not provided, then refreshSession() will attempt to retrieve it from the current session.
  /// If no refresh token is available (neither provided nor in current session), an error will be thrown.
  Future<AuthResponse> refreshSession([String? refreshToken]) async {
    _log.info('Refresh session');

    final currentSessionRefreshToken =
        refreshToken ?? _currentSession?.refreshToken;

    if (currentSessionRefreshToken == null) {
      _log.warning("Can't refresh session, no refresh token found.");
      throw AuthSessionMissingException();
    }

    return await _callRefreshToken(currentSessionRefreshToken);
  }
```

The method:

1. Uses provided refresh token or falls back to current session's refresh token
2. Throws `AuthSessionMissingException` if no refresh token is available
3. Delegates to `_callRefreshToken` for the actual refresh operation

### _callRefreshToken

Internal method that orchestrates token refresh with deduplication:

```dart 1269:1317:lib/src/gotrue_client.dart
  /// Generates a new JWT.
  ///
  /// To prevent multiple simultaneous requests it catches an already ongoing request by using the global [_refreshTokenCompleter].
  /// If that's not null and not completed it returns the future of the ongoing request.
  Future<AuthResponse> _callRefreshToken(String refreshToken) async {
    // Refreshing is already in progress
    if (_refreshTokenCompleter != null) {
      _log.finer("Don't call refresh token, already in progress");
      return _refreshTokenCompleter!.future;
    }

    try {
      _refreshTokenCompleter = Completer<AuthResponse>();

      // Catch any error in case nobody awaits the future
      _refreshTokenCompleter!.future.then(
        (_) => null,
        onError: (_, __) => null,
      );
      _log.fine('Refresh access token');

      final data = await _refreshAccessToken(refreshToken);

      final session = data.session;

      if (session == null) {
        throw AuthSessionMissingException();
      }

      _saveSession(session);
      notifyAllSubscribers(AuthChangeEvent.tokenRefreshed);

      _refreshTokenCompleter?.complete(data);
      return data;
    } on AuthException catch (error, stack) {
      if (error is! AuthRetryableFetchException) {
        _removeSession();
        notifyAllSubscribers(AuthChangeEvent.signedOut);
      } else {
        notifyException(error, stack);
      }

      _refreshTokenCompleter?.completeError(error);

      rethrow;
    } catch (error, stack) {
      _refreshTokenCompleter?.completeError(error);
      notifyException(error, stack);
      rethrow;
    } finally {
      _refreshTokenCompleter = null;
    }
  }
```

The refresh mechanism includes:

1. **Deduplication**: If a refresh is already in progress, returns the existing future
2. **Error handling**: Non-retryable errors trigger sign out, retryable errors are logged
3. **Completion**: The completer is always resolved in the finally block
4. **Session update**: Saves the new session and notifies subscribers

### _refreshAccessToken

Internal method that performs the actual token refresh HTTP request:

```dart 1113:1148:lib/src/gotrue_client.dart
  /// Generates a new JWT.
  /// [refreshToken] A valid refresh token that was returned on login.
  Future<AuthResponse> _refreshAccessToken(String refreshToken) async {
    final startedAt = DateTime.now();
    var attempt = 0;
    return await retry<AuthResponse>(
      // Make a GET request
      () async {
        attempt++;
        _log.fine('Attempt $attempt to refresh token');
        final options = GotrueRequestOptions(
            headers: _headers,
            body: {'refresh_token': refreshToken},
            query: {'grant_type': 'refresh_token'});
        final response = await _fetch
            .request('$_url/token', RequestMethodType.post, options: options);
        final authResponse = AuthResponse.fromJson(response);
        return authResponse;
      },
      retryIf: (e) {
        // Do not retry if the next retry comes after the next tick.
        final nextBackOff =
            Duration(milliseconds: (200 * pow(2, attempt - 1).floor()));

        return e is AuthRetryableFetchException &&
            (DateTime.now().millisecondsSinceEpoch +
                    nextBackOff.inMilliseconds -
                    startedAt.millisecondsSinceEpoch) <
                Constants.autoRefreshTickDuration.inMilliseconds;
      },
      maxDelay: Duration(seconds: 10),
      randomizationFactor: 0,

      // Max interval between retries is 10 sec, so just set the maxAttempts
      // to something that will yield a more than 10 sec interval.
      maxAttempts: 999,
    );
  }
```

The method uses exponential backoff with the `retry` package:

1. Makes a POST request to `/token` with `grant_type=refresh_token`
2. Retries on `AuthRetryableFetchException` with exponential backoff
3. Stops retrying if the next retry would exceed the auto-refresh tick duration
4. Has a maximum of 999 attempts with 10-second maximum delay

## Automatic Token Refresh

### startAutoRefresh

Starts the automatic token refresh timer:

```dart 1059:1070:lib/src/gotrue_client.dart
  /// Starts an auto-refresh process in the background. Close to the time of expiration a process is started to
  /// refresh the session. If refreshing fails it will be retried for as long as necessary.
  void startAutoRefresh() async {
    stopAutoRefresh();

    _log.fine('Starting auto refresh');
    _autoRefreshTicker = Timer.periodic(
      Constants.autoRefreshTickDuration,
      (Timer t) => _autoRefreshTokenTick(),
    );

    await Future.delayed(Duration.zero);
    await _autoRefreshTokenTick();
  }
```

The method:

1. Stops any existing refresh timer
2. Creates a periodic timer that ticks every 10 seconds (configurable via `Constants.autoRefreshTickDuration`)
3. Triggers an immediate check after initialization

### stopAutoRefresh

Stops the automatic token refresh timer:

```dart 1072:1077:lib/src/gotrue_client.dart
  /// Stops an active auto refresh process running in the background (if any).
  void stopAutoRefresh() {
    _log.fine('Stopping auto refresh');
    _autoRefreshTicker?.cancel();
    _autoRefreshTicker = null;
  }
```

### _autoRefreshTokenTick

The periodic check that determines if a token refresh is needed:

```dart 1079:1109:lib/src/gotrue_client.dart
  Future<void> _autoRefreshTokenTick() async {
    try {
      final now = DateTime.now();
      final refreshToken = _currentSession?.refreshToken;
      if (refreshToken == null) {
        return;
      }

      final expiresAt = _currentSession?.expiresAt;
      if (expiresAt == null) {
        return;
      }

      final expiresInTicks =
          (DateTime.fromMillisecondsSinceEpoch(expiresAt * 1000)
                      .difference(now)
                      .inMilliseconds /
                  Constants.autoRefreshTickDuration.inMilliseconds)
              .floor();

      _log.finer('Access token expires in $expiresInTicks ticks');

      // Only tick if the next tick comes after the retry threshold
      if (expiresInTicks <= Constants.autoRefreshTickThreshold) {
        await _callRefreshToken(refreshToken);
      }
    } catch (error) {
      // Do nothing. JS client prints here, but error is already tracked via
      // [notifyException]
    }
  }
```

The tick logic:

1. Gets the current time and session's refresh token
2. Calculates how many ticks until expiration
3. If tokens expire within the threshold (3 ticks = 30 seconds by default), triggers refresh
4. Silently catches errors (they're tracked via `notifyException`)

### Configuration Constants

```dart 18:23:lib/src/constants.dart
  /// Current session will be checked for refresh at this interval.
  static const autoRefreshTickDuration = Duration(seconds: 10);

  /// A token refresh will be attempted this many ticks before the current session expires.
  static const autoRefreshTickThreshold = 3;
```

- **autoRefreshTickDuration**: How often to check if refresh is needed (10 seconds)
- **autoRefreshTickThreshold**: How many ticks before expiration to trigger refresh (3 ticks = 30 seconds)

## User Attributes Update

### updateUser

Updates the current user's attributes:

```dart 723:748:lib/src/gotrue_client.dart
  /// Updates user data, if there is a logged in user.
  Future<UserResponse> updateUser(
    UserAttributes attributes, {
    String? emailRedirectTo,
  }) async {
    final accessToken = currentSession?.accessToken;
    if (accessToken == null) {
      throw AuthSessionMissingException();
    }

    final body = attributes.toJson();
    final options = GotrueRequestOptions(
      headers: _headers,
      body: body,
      jwt: accessToken,
      redirectTo: emailRedirectTo,
    );
    final response = await _fetch.request('$_url/user', RequestMethodType.put,
        options: options);
    final userResponse = UserResponse.fromJson(response);

    _currentSession = currentSession?.copyWith(user: userResponse.user);
    notifyAllSubscribers(AuthChangeEvent.userUpdated);

    return userResponse;
  }
```

The method:

1. Requires an authenticated session
2. Sends a PUT request to `/user` with the updated attributes
3. Updates the current session with the new user data
4. Emits `userUpdated` event

## Summary

Session management in GoTrueClient encompasses:

| Operation | Method | Purpose |
|-----------|--------|---------|
| Sign out | `signOut()` | Terminates session with configurable scope |
| Initial session | `setInitialSession()` | Sets session from stored JSON on startup |
| Recover session | `recoverSession()` | Recovers and optionally refreshes session |
| Set session | `setSession()` | Sets session from refresh token |
| Refresh session | `refreshSession()` | Manually refreshes the access token |
| Update user | `updateUser()` | Updates user attributes |
| Auto-refresh | `startAutoRefresh()` / `stopAutoRefresh()` | Controls automatic token refresh |

The automatic refresh mechanism ensures users stay authenticated by refreshing tokens before expiration, with exponential backoff for retryable failures.
