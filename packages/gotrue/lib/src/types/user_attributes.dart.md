# User Attributes Classes

This document explains the `UserAttributes` and `AdminUserAttributes` classes defined in `packages/gotrue/lib/src/types/user_attributes.dart`.

## Overview

These classes represent the user attributes that can be set or modified when creating or updating user accounts in Supabase. They provide type-safe data structures with JSON serialization and proper equality comparisons.

---

## UserAttributes Class

The `UserAttributes` class is the base class for user attribute management. It contains the core attributes that a user can update about themselves.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `email` | `String?` | The user's email address |
| `phone` | `String?` | The user's phone number |
| `password` | `String?` | The user's password |
| `nonce` | `String?` | The nonce sent for reauthentication if the user's password is being updated. Call `reauthenticate()` to obtain the nonce first |
| `data` | `Object?` | Custom JSON object to store user-specific metadata. Maps to `auth.users.user_metadata` column |

### Constructor

```dart
UserAttributes({
  this.email,
  this.phone,
  this.password,
  this.nonce,
  this.data,
}) : assert(data == null || data is List || data is Map);
```

The constructor includes an assertion to ensure that `data` is either `null`, a `List`, or a `Map` - ensuring it's valid JSON-serializable data.

### Methods

#### toJson()

```dart
Map<String, dynamic> toJson() {
  return {
    if (email != null) 'email': email,
    if (phone != null) 'phone': phone,
    if (nonce != null) 'nonce': nonce,
    if (password != null) 'password': password,
    if (data != null) 'data': data,
  };
}
```

Converts the instance to a JSON map. Only includes non-null values, making it efficient for API requests.

#### operator ==()

```dart
@override
bool operator ==(Object other) {
  if (identical(this, other)) return true;
  if (other is! UserAttributes) return false;

  final mapEquals = const DeepCollectionEquality().equals;

  return other.email == email &&
      other.phone == phone &&
      other.password == password &&
      other.nonce == nonce &&
      mapEquals(other.data, data);
}
```

Compares two `UserAttributes` instances. Uses `DeepCollectionEquality` from the `collection` package to properly compare nested collections (like `data`).

#### hashCode

```dart
@override
int get hashCode {
  return email.hashCode ^
      phone.hashCode ^
      password.hashCode ^
      nonce.hashCode ^
      data.hashCode;
}
```

Generates a hash code using XOR operations on all properties, ensuring proper behavior in hash-based collections like `Set` and `Map`.

---

## AdminUserAttributes Class

The `AdminUserAttributes` class extends `UserAttributes` and adds administrative properties that only a service role can modify.

### Additional Properties

| Property | Type | Description |
|----------|------|-------------|
| `userMetadata` | `Map<String, dynamic>?` | User metadata. Maps to `auth.users.user_metadata` column. Only service role can modify |
| `appMetadata` | `Map<String, dynamic>?` | Application-specific metadata. Maps to `auth.users.app_metadata` column. Only service role can modify (e.g., identity providers, roles, access control) |
| `emailConfirm` | `bool?` | If true, confirms the user's email address. Only service role can modify |
| `phoneConfirm` | `bool?` | If true, confirms the user's phone number. Only service role can modify |
| `banDuration` | `String?` | Duration for which the user is banned. Format: strict sequence with unit suffix ("ns", "us", "ms", "s", "m", "h"). Example: "300ms", "2h45m". Use "none" to lift the ban |

### Constructor

```dart
AdminUserAttributes({
  super.email,
  super.phone,
  super.password,
  super.data,
  this.userMetadata,
  this.appMetadata,
  this.emailConfirm,
  this.phoneConfirm,
  this.banDuration,
});
```

Uses Dart 2.17+ super constructor parameters to inherit properties from `UserAttributes`.

### Methods

#### toJson()

```dart
@override
Map<String, dynamic> toJson() {
  return {
    if (email != null) 'email': email,
    if (phone != null) 'phone': phone,
    if (password != null) 'password': password,
    if (data != null) 'data': data,
    if (userMetadata != null) 'user_metadata': userMetadata,
    if (appMetadata != null) 'app_metadata': appMetadata,
    if (emailConfirm != null) 'email_confirm': emailConfirm,
    if (phoneConfirm != null) 'phone_confirm': phoneConfirm,
    if (banDuration != null) 'ban_duration': banDuration,
  };
}
```

Overrides the parent method to include admin-specific properties with snake_case keys (matching Supabase API conventions).

#### operator ==()

```dart
@override
bool operator ==(Object other) {
  if (identical(this, other)) return true;
  if (other is! AdminUserAttributes) return false;

  final mapEquals = const DeepCollectionEquality().equals;

  return mapEquals(other.userMetadata, userMetadata) &&
      mapEquals(other.appMetadata, appMetadata) &&
      other.emailConfirm == emailConfirm &&
      other.phoneConfirm == phoneConfirm &&
      other.banDuration == banDuration;
}
```

Only compares the new properties added by `AdminUserAttributes` (inherited properties are compared by parent class via `super`).

#### hashCode

```dart
@override
int get hashCode {
  return super.hashCode ^
      userMetadata.hashCode ^
      appMetadata.hashCode ^
      emailConfirm.hashCode ^
      phoneConfirm.hashCode ^
      banDuration.hashCode;
}
```

XORs the parent's hash code with the new properties' hash codes.

---

## Usage Examples

### Creating UserAttributes

```dart
final userAttrs = UserAttributes(
  email: 'user@example.com',
  password: 'securePassword123',
  data: {'firstName': 'John', 'lastName': 'Doe'},
);

final json = userAttrs.toJson();
// {'email': 'user@example.com', 'password': 'securePassword123', 'data': {'firstName': 'John', 'lastName': 'Doe'}}
```

### Creating AdminUserAttributes

```dart
final adminAttrs = AdminUserAttributes(
  email: 'admin@example.com',
  userMetadata: {'role': 'moderator'},
  appMetadata: {'permissions': ['read', 'write', 'delete']},
  emailConfirm: true,
  banDuration: '2h',
);

final json = adminAttrs.toJson();
// {'email': 'admin@example.com', 'user_metadata': {'role': 'moderator'}, 'app_metadata': {'permissions': ['read', 'write', 'delete']}, 'email_confirm': true, 'ban_duration': '2h'}
```

### Comparing Instances

```dart
final attrs1 = UserAttributes(email: 'test@example.com');
final attrs2 = UserAttributes(email: 'test@example.com');

print(attrs1 == attrs2); // true
print(attrs1.hashCode == attrs2.hashCode); // true

final attrs3 = UserAttributes(email: 'different@example.com');
print(attrs1 == attrs3); // false
```

---

## Key Design Patterns

1. **Conditional Serialization**: Uses `if (value != null)` pattern to exclude null values from JSON output
2. **Deep Collection Equality**: Uses `DeepCollectionEquality` for proper comparison of nested collections
3. **Inheritance for Extensibility**: `AdminUserAttributes` extends `UserAttributes` to add admin-only fields
4. **JSON Convention Compliance**: Uses snake_case for JSON keys, matching Supabase API conventions
5. **Service Role Separation**: Clear separation between user-modifiable and admin-only attributes
