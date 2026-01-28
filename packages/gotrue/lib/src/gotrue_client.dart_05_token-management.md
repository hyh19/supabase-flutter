# GoTrueClient Token Management and JWT Verification

This document explains the token management and JWT (JSON Web Token) verification functionality in the `GoTrueClient` class.

## Token Management Overview

The `GoTrueClient` manages tokens through several mechanisms:

1. **Token Storage**: Sessions contain access tokens, refresh tokens, and provider tokens
2. **Automatic Refresh**: Tokens are refreshed before expiration using a periodic timer
3. **Manual Refresh**: The `refreshSession()` method allows manual token refresh
4. **JWT Verification**: The `getClaims()` method verifies and extracts JWT claims
5. **JWKS Caching**: JSON Web Key Set is cached for efficient signature verification

## JWKS (JSON Web Key Set) Management

### JWKS Caching

The client maintains an in-memory cache of the JWKS to avoid repeated network requests:

```dart 62:63:lib/src/gotrue_client.dart
  JWKSet? _jwks;
  DateTime? _jwksCachedAt;
```

- **`_jwks`**: Cached JWK Set containing public keys for JWT verification
- **`_jwksCachedAt`**: Timestamp when the JWKS was cached

The cache has a TTL (Time To Live) of 10 minutes:

```dart 28:29:lib/src/constants.dart
  /// The TTL for the JWKS cache.
  static const jwksTtl = Duration(minutes: 10);
```

### _fetchJwk

Fetches a specific JWK (JSON Web Key) for JWT signature verification:

```dart 1353:1390:lib/src/gotrue_client.dart
  Future<JWK?> _fetchJwk(String kid, JWKSet suppliedJwks) async {
    // try fetching from the supplied jwks
    final jwk = suppliedJwks.keys.firstWhereOrNull((jwk) => jwk.kid == kid);
    if (jwk != null) {
      return jwk;
    }

    final now = DateTime.now();

    // try fetching from cache
    final cachedJwk = _jwks?.keys.firstWhereOrNull((jwk) => jwk.kid == kid);

    // jwks exists and it isn't stale
    if (cachedJwk != null &&
        _jwksCachedAt != null &&
        _jwksCachedAt!.add(Constants.jwksTtl).isAfter(now)) {
      return cachedJwk;
    }

    // jwk isn't cached in memory so we need to fetch it from the well-known endpoint
    final jwksResponse = await _fetch.request(
      '$_url/.well-known/jwks.json',
      RequestMethodType.get,
      options: GotrueRequestOptions(headers: _headers),
    );

    final jwks = JWKSet.fromJson(jwksResponse as Map<String, dynamic>);

    if (jwks.keys.isEmpty) {
      return null;
    }

    _jwks = jwks;
    _jwksCachedAt = now;

    // find the signing key
    return jwks.keys.firstWhereOrNull((jwk) => jwk.kid == kid);
  }
```

The method:

1. First checks if the key is in the supplied JWKS (for immediate verification)
2. Then checks the in-memory cache if it's still valid
3. If not cached, fetches from the `/.well-known/jwks.json` endpoint
4. Updates the cache and returns the matching key

## JWT Verification

### getClaims

Extracts and verifies JWT claims from an access token. This is more efficient than `getUser()` for simply reading claims:

```dart 1411:1456:lib/src/gotrue_client.dart
  /// Extracts the JWT claims present in the access token by first verifying the
  /// JWT against the server's JSON Web Key Set endpoint
  /// `/.well-known/jwks.json` which is often cached, resulting in significantly
  /// faster responses. Prefer this method over [getUser] which always
  /// sends a request to the Auth server for each JWT.
  ///
  /// If the project is not using an asymmetric JWT signing key (like ECC or
  /// RSA) it always sends a request to the Auth server (similar to [getUser]) to verify the JWT.
  ///
  /// For JWTs signed with asymmetric algorithms (RS256, ES256, etc.), the JWKS
  /// is fetched from the server on the first call and cached for subsequent calls.
  /// The cache is refreshed automatically after 10 minutes.
  ///
  /// [jwt] An optional specific JWT you wish to verify, not the one you
  ///       can obtain from [currentSession].
  /// [options] Various additional options that allow you to customize the
  ///           behavior of this method.
  ///
  /// Returns a [GetClaimsResponse] containing the JWT claims, or throws an [AuthException] on error.
  Future<GetClaimsResponse> getClaims([
    String? jwt,
    GetClaimsOptions? options,
  ]) async {
    String token = jwt ?? '';

    if (token.isEmpty) {
      final session = currentSession;
      if (session == null) {
        throw AuthSessionMissingException('No session found');
      }
      token = session.accessToken;
    }

    // Decode the JWT to get the payload
    final decoded = decodeJwt(token);

    // Validate expiration unless allowExpired is true
    if (!(options?.allowExpired ?? false)) {
      validateExp(decoded.payload.exp);
    }

    final signingKey =
        (decoded.header.alg.startsWith('HS') || decoded.header.kid == null)
            ? null
            : await _fetchJwk(decoded.header.kid!, _jwks ?? JWKSet(keys: []));

    // If symmetric algorithm, fallback to getUser()
    if (signingKey == null) {
      await getUser(token);
      return GetClaimsResponse(
          claims: decoded.payload,
          header: decoded.header,
          signature: decoded.signature);
    }

    try {
      JWT.verify(token, signingKey.rsaPublicKey);
      return GetClaimsResponse(
          claims: decoded.payload,
          header: decoded.header,
          signature: decoded.signature);
    } catch (e) {
      throw AuthInvalidJwtException('Invalid JWT signature: $e');
    }
  }
```

The method performs the following steps:

1. **Token Selection**: Uses provided JWT or falls back to current session's access token
2. **JWT Decoding**: Decodes the JWT to extract header and payload
3. **Expiration Validation**: Checks the `exp` claim unless `allowExpired` is true
4. **Key Selection**:
   - For symmetric algorithms (HS256, etc.): Skips JWKS verification
   - For asymmetric algorithms (RS256, ES256, etc.): Fetches the signing key
5. **Signature Verification**: Verifies the JWT signature using the public key
6. **Fallback**: If no signing key is available, falls back to `getUser()` for verification

### JWT Structure

The JWT types are defined in `jwt.dart`:

```dart 39:92:lib/src/types/jwt.dart
/// JWT Payload structure with standard claims
class JwtPayload {
  /// Issuer - identifies principal that issued the JWT
  final String? iss;

  /// Subject - identifies the subject of the JWT
  final String? sub;

  /// Audience - identifies recipients that the JWT is intended for
  final dynamic aud;

  /// Expiration time - timestamp after which the JWT must not be accepted
  final int? exp;

  /// Not Before - timestamp before which the JWT must not be accepted
  final int? nbf;

  /// Issued At - timestamp when the JWT was issued
  final int? iat;

  /// JWT ID - unique identifier for the JWT
  final String? jti;

  /// Additional claims stored in the payload
  final Map<String, dynamic> claims;
```

```dart 6:37:lib/src/types/jwt.dart
/// JWT Header structure
class JwtHeader {
  /// Algorithm used to sign the JWT (e.g., 'RS256', 'ES256', 'HS256')
  final String alg;

  /// Key ID - identifies which key was used to sign the JWT
  final String? kid;

  /// Token type - typically 'JWT'
  final String? typ;
```

### GetClaimsResponse

The response contains the decoded JWT components:

```dart 134:150:lib/src/types/jwt.dart
/// Response from getClaims method
class GetClaimsResponse {
  /// JWT claims from the payload
  final JwtPayload claims;

  /// JWT header
  final JwtHeader header;

  /// JWT signature
  final List<int> signature;

  GetClaimsResponse({
    required this.claims,
    required this.header,
    required this.signature,
  });
}
```

### GetClaimsOptions

Options to customize the behavior of `getClaims()`:

```dart 152:161:lib/src/types/jwt.dart
/// Options for getClaims method
class GetClaimsOptions {
  /// If set to `true`, the `exp` claim will not be validated against the current time.
  /// This allows you to extract claims from expired JWTs without getting an error.
  final bool allowExpired;

  const GetClaimsOptions({
    this.allowExpired = false,
  });
}
```

## JWK (JSON Web Key)

The `JWK` class represents a JSON Web Key for asymmetric cryptography:

```dart 186:274:lib/src/types/jwt.dart
class JWK {
  /// The "kty" (key type) parameter identifies the cryptographic algorithm
  /// family used with the key, such as "RSA" or "EC".
  final String kty;

  /// The "key_ops" (key operations) parameter identifies the cryptographic
  /// operations for which the key is intended to be used.
  final List<String> keyOps;

  /// The "alg" (algorithm) parameter identifies the algorithm intended for
  /// use with the key.
  final String? alg;

  /// The "kid" (key ID) parameter is used to match a specific key.
  final String? kid;

  /// Additional arbitrary properties of the JWK.
  final Map<String, dynamic> _additionalProperties;

  /// Converts this [JWK] to a JSON map and extracts the RSA public key
  RSAPublicKey get rsaPublicKey {
    final bytes = utf8.encode(json.encode(toJson()));
    return RSAPublicKeyBytes(bytes);
  }
}
```

The `rsaPublicKey` getter is used for JWT signature verification.

## Token Types in Session

The `Session` class contains multiple token types:

```dart 6:16:lib/src/types/session.dart
class Session {
  final String? providerToken;
  final String? providerRefreshToken;
  final String accessToken;
  final int? expiresIn;
  final String? refreshToken;
  final String tokenType;
  final User user;
```

- **`accessToken`**: The JWT used for API authentication
- **`refreshToken`**: Token used to obtain new access tokens
- **`providerToken`**: OAuth provider access token (for making API calls to the provider)
- **`providerRefreshToken`**: OAuth provider refresh token
- **`tokenType`**: Typically "bearer"

## Token Expiration Handling

### Session Expiration Check

The `Session` class provides an `isExpired` property:

```dart 71:79:lib/src/types/session.dart
  /// Returns 'true` if the token is expired or will expire in the next 10 seconds.
  ///
  /// The 10 second buffer is to account for latency issues.
  bool get isExpired {
    if (expiresAt == null) return false;
    return DateTime.now().add(Constants.expiryMargin).isAfter(
          DateTime.fromMillisecondsSinceEpoch(expiresAt! * 1000),
        );
  }
```

The expiry margin is 30 seconds:

```dart 16:17:lib/src/constants.dart
  /// The margin to use when checking if a token is expired.
  static const expiryMargin = Duration(seconds: 30);
```

### Automatic Refresh Timing

The automatic refresh mechanism uses tick-based checking:

```dart 19:23:lib/src/constants.dart
  /// Current session will be checked for refresh at this interval.
  static const autoRefreshTickDuration = Duration(seconds: 10);

  /// A token refresh will be attempted this many ticks before the current session expires.
  static const autoRefreshTickThreshold = 3;
```

- Tick duration: 10 seconds
- Refresh threshold: 3 ticks (30 seconds before expiration)

## Summary

Token management and JWT verification in GoTrueClient includes:

| Feature | Method/Property | Purpose |
|---------|-----------------|---------|
| JWKS Fetch | `_fetchJwk()` | Retrieve signing key for JWT verification |
| JWT Claims | `getClaims()` | Extract and verify JWT claims |
| Session Refresh | `refreshSession()` | Manually refresh access token |
| Auto-refresh | `startAutoRefresh()` / `_autoRefreshTokenTick()` | Automatic token refresh |
| Expiration Check | `session.isExpired` | Check if token is expired |
| Token Types | `Session` class | Access, refresh, and provider tokens |
| JWKS Cache | `_jwks`, `_jwksCachedAt` | In-memory caching of signing keys |

Key benefits of the JWT verification approach:

- **Efficient**: JWKS is cached for 10 minutes, avoiding repeated fetches
- **Secure**: RS256/ES256 signatures are verified against server's public keys
- **Flexible**: Supports both symmetric (HS256) and asymmetric algorithms
- **Configurable**: Can extract claims from expired tokens when needed
