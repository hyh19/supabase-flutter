# Local Storage System in Supabase Flutter

This document provides an in-depth explanation of the local storage system used in the Supabase Flutter SDK for persisting user sessions and authentication data.

## Overview

The `local_storage.dart` file defines a comprehensive abstraction layer for session persistence across different platforms (iOS, Android, Web). It implements the Strategy design pattern to provide platform-specific storage solutions while maintaining a consistent API.

## Architecture Overview

```text
LocalStorage (Abstract Base)
├── EmptyLocalStorage (No-op implementation)
├── SharedPreferencesLocalStorage (Main session storage)
└── SharedPreferencesGotrueAsyncStorage (PKCE flow storage)
```

## Core Components

### 1. Session Persistence Key

```dart
const supabasePersistSessionKey = 'SUPABASE_PERSIST_SESSION_KEY';
```

This constant defines the key used to store the user's session data in the local storage. It is primarily used for migration purposes from the legacy Hive storage system and is no longer actively used in the current implementation.

### 2. LocalStorage Abstract Class

The `LocalStorage` abstract class defines the contract for all storage implementations. It provides five core methods that every storage solution must implement:

#### Key Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `initialize()` | `Future<void> initialize()` | Prepares the storage backend for operations |
| `hasAccessToken()` | `Future<bool> hasAccessToken()` | Checks if a session exists |
| `accessToken()` | `Future<String?> accessToken()` | Retrieves the stored access token |
| `removePersistedSession()` | `Future<void> removePersistedSession()` | Clears the current session |
| `persistSession()` | `Future<void> persistSession(String persistSessionString)` | Saves session data |

### 3. Platform-Specific Implementation Strategy

The file uses conditional imports to handle platform differences:

```dart
import './local_storage_stub.dart'
    if (dart.library.js_interop) './local_storage_web.dart' as web;
```

This pattern ensures that:

- **Web Platform**: Uses browser's localStorage API via `local_storage_web.dart`
- **Native Platforms** (iOS/Android): Uses SharedPreferences
- **Stub Platform**: Provides no-op implementations for testing

## Implementation Details

### EmptyLocalStorage

The `EmptyLocalStorage` class is a no-op (no operation) implementation that discards all data. Use this when you want to disable session persistence entirely.

**Use Cases:**

- Security-sensitive applications where session persistence is prohibited
- Testing environments
- Guest mode functionality

**Implementation Characteristics:**

- All methods return immediately without any actual storage operations
- `hasAccessToken()` always returns `false`
- `accessToken()` always returns `null`

### SharedPreferencesLocalStorage

This is the primary storage implementation for native mobile platforms. It wraps the Flutter `SharedPreferences` plugin to persist authentication sessions.

#### Constructor

```dart
SharedPreferencesLocalStorage({required this.persistSessionKey});
```

The constructor requires a `persistSessionKey` parameter that defines the storage key for the session data.

#### Platform Detection

```dart
static const _useWebLocalStorage =
    kIsWeb && bool.fromEnvironment("dart.library.js_interop");
```

This static constant determines at compile time whether to use web-based storage:

- `kIsWeb`: Flutter's web platform detection
- `bool.fromEnvironment("dart.library.js_interop")`: Checks for JavaScript interoperability availability

#### Initialization Process

```dart
Future<void> initialize() async {
  if (!_useWebLocalStorage) {
    WidgetsFlutterBinding.ensureInitialized();
    _prefs = await SharedPreferences.getInstance();
  }
}
```

The initialization process:

1. Checks if running on native platform (not web)
2. Ensures Flutter binding is initialized (required for plugin access)
3. Retrieves the SharedPreferences instance asynchronously

**Note:** Web platforms skip this initialization since they use a different storage mechanism.

#### Token Management Methods

All methods in this class follow a consistent pattern:

1. **Check platform** using `_useWebLocalStorage`
2. **Delegate to appropriate handler**:
   - Web: Delegate to `web` module functions
   - Native: Use `_prefs` instance directly

**Example Flow:**

```dart
Future<bool> hasAccessToken() async {
  if (_useWebLocalStorage) {
    return web.hasAccessToken(persistSessionKey);
  }
  return _prefs.containsKey(persistSessionKey);
}
```

### SharedPreferencesGotrueAsyncStorage

This class extends `GotrueAsyncStorage` and provides async storage specifically designed for the PKCE (Proof Key for Code Exchange) authentication flow.

#### PKCE Flow Background

The PKCE flow requires storing:

- **Code Verifier**: A cryptographic random string generated during authentication initiation
- **Code Challenge**: A hash of the code verifier, sent to the server

Both values must persist across the authentication redirect cycle.

#### Initialization Pattern

```dart
SharedPreferencesGotrueAsyncStorage() {
  _initialize();
}

final Completer<void> _initializationCompleter = Completer();

Future<void> _initialize() async {
  WidgetsFlutterBinding.ensureInitialized();
  _prefs = await SharedPreferences.getInstance();
  _initializationCompleter.complete();
}
```

This implementation uses a `Completer` to handle asynchronous initialization:

1. Constructor triggers `_initialize()` immediately
2. `_initialize()` starts async SharedPreferences loading
3. The `Completer` allows methods to await initialization completion
4. All storage operations wait for the completer before proceeding

#### Storage Interface Methods

| Method | Operation | Storage Type |
|--------|-----------|--------------|
| `getItem()` | Retrieve value by key | `_prefs.getString(key)` |
| `setItem()` | Store key-value pair | `_prefs.setString(key, value)` |
| `removeItem()` | Delete key-value pair | `_prefs.remove(key)` |

All methods:

1. Await the initialization completer
2. Perform the requested operation
3. Return the result or await completion of the operation

## Platform-Specific Behavior

### Web Platform

On web platforms, the system delegates to `local_storage_web.dart` which uses the browser's `localStorage` API. This is necessary because:

- SharedPreferences is not available in browser environments
- Browser sandbox restrictions require using web-specific APIs
- Cross-tab synchronization is handled natively by the browser

### Native Platforms (iOS/Android)

Native platforms use SharedPreferences directly:

- Data is stored as key-value pairs in the device's shared preferences
- Persistence survives app restarts
- Data is cleared when the app is uninstalled

## Usage Examples

### Disabling Session Persistence

```dart
await Supabase.initialize(
  url: 'https://your-project.supabase.co',
  anonKey: 'your-anon-key',
  authOptions: AuthClientOptions(
    authFlowType: AuthFlowType.pkce,
  ),
  // Use EmptyLocalStorage to disable persistence
);
```

### Custom Session Key

```dart
final storage = SharedPreferencesLocalStorage(
  persistSessionKey: 'my_app_custom_session_key',
);
await storage.initialize();
```

### Accessing Stored Session

```dart
final storage = SharedPreferencesLocalStorage(
  persistSessionKey: 'supabase_session',
);

bool hasSession = await storage.hasAccessToken();
String? token = await storage.accessToken();

// When logging out
await storage.removePersistedSession();
```

## Error Handling Considerations

1. **SharedPreferences Loading Failures**: The `initialize()` method may throw if SharedPreferences cannot be loaded
2. **Storage Quota Exceeded**: Web localStorage may throw when quota is exceeded
3. **Concurrent Access**: SharedPreferences handles concurrent reads/writes internally
4. **Platform Mismatches**: Ensure platform detection logic matches your deployment target

## Performance Characteristics

| Operation | Native (iOS/Android) | Web |
|-----------|---------------------|-----|
| Initialize | ~50-200ms (first time) | Immediate |
| Read | <1ms | <1ms |
| Write | ~5-50ms | ~5-50ms |
| Delete | ~5-20ms | ~5-20ms |

## Security Implications

1. **Data Storage**: Access tokens are stored in plaintext
2. **Access Control**: SharedPreferences are accessible to any app with root access on Android
3. **Web Storage**: localStorage is accessible via JavaScript on the same domain
4. **Data Recovery**: Cleared when device is reset or app is uninstalled

For high-security applications, consider:

- Using `EmptyLocalStorage` for sensitive sessions
- Implementing additional encryption
- Using secure storage plugins for sensitive data

## Migration Considerations

The `supabasePersistSessionKey` constant exists for migration from Hive to SharedPreferences:

- Legacy apps may have data stored with this key
- Migration logic would read from Hive and write to SharedPreferences
- New implementations should use custom keys to avoid conflicts

## Related Components

- **SupabaseAuth**: Uses LocalStorage for session restoration on app start
- **AuthFlowType.pkce**: Requires SharedPreferencesGotrueAsyncStorage for code verifier storage
- **SharedPreferences Plugin**: Underlying storage mechanism for native platforms

## Summary

The local storage system provides a robust, platform-aware abstraction for persisting authentication data. By separating concerns into abstract base classes and platform-specific implementations, the system maintains flexibility while providing optimal performance on each platform. The dual implementation strategy (session storage + PKCE storage) supports both implicit and PKCE authentication flows comprehensively.
