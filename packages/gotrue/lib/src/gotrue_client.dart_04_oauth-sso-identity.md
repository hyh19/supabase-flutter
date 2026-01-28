# GoTrueClient OAuth, SSO, and Identity Management

This document explains the OAuth, SSO (Single Sign-On), and identity management functionality in the `GoTrueClient` class.

## OAuth Authentication

### getOAuthSignInUrl

Generates the URL for initiating an OAuth sign-in flow with a third-party provider.

```dart 317:331:lib/src/gotrue_client.dart
  /// Generates a link to log in an user via a third-party provider.
  Future<OAuthResponse> getOAuthSignInUrl({
    required OAuthProvider provider,
    String? redirectTo,
    String? scopes,
    Map<String, String>? queryParams,
  }) async {
    return _getUrlForProvider(
      provider,
      url: '$_url/authorize',
      redirectTo: redirectTo,
      scopes: scopes,
      queryParams: queryParams,
    );
  }
```

The method constructs the authorization URL by delegating to the internal `_getUrlForProvider` method. The URL includes:

- The provider name
- Optional scopes (e.g., `email`, `profile`)
- Optional redirect URL after authentication
- Optional custom query parameters

### _getUrlForProvider

Internal method that constructs the OAuth URL with optional PKCE support.

```dart 1151:1191:lib/src/gotrue_client.dart
  /// Returns the OAuth sign in URL constructed from the [url] parameter.
  Future<OAuthResponse> _getUrlForProvider(
    OAuthProvider provider, {
    required String url,
    required String? scopes,
    required String? redirectTo,
    required Map<String, String>? queryParams,
    bool skipBrowserRedirect = false,
  }) async {
    final urlParams = {'provider': provider.snakeCase};
    if (scopes != null) {
      urlParams['scopes'] = scopes;
    }
    if (redirectTo != null) {
      urlParams['redirect_to'] = redirectTo;
    }
    if (queryParams != null) {
      urlParams.addAll(queryParams);
    }
    if (_flowType == AuthFlowType.pkce) {
      assert(_asyncStorage != null,
          'You need to provide asyncStorage to perform pkce flow.');
      final codeVerifier = generatePKCEVerifier();
      await _asyncStorage!.setItem(
        key: '${Constants.defaultStorageKey}-code-verifier',
        value: codeVerifier,
      );

      final codeChallenge = generatePKCEChallenge(codeVerifier);
      final flowParams = {
        'flow_type': _flowType.name,
        'code_challenge': codeChallenge,
        'code_challenge_method': 's256',
      };
      urlParams.addAll(flowParams);
    }
    if (skipBrowserRedirect) {
      urlParams['skip_http_redirect'] = 'true';
    }
    final oauthUrl = '$url?${Uri(queryParameters: urlParams).query}';
    return OAuthResponse(provider: provider, url: oauthUrl);
  }
```

For PKCE flow:

1. Generates a random code verifier
2. Stores the verifier in async storage
3. Creates a code challenge from the verifier
4. Includes the challenge in the authorization URL

The `skipBrowserRedirect` parameter is useful for mobile apps that want to handle the OAuth response directly instead of relying on browser redirects.

## Single Sign-On (SSO)

### getSSOSignInUrl

Obtains a URL to perform a single-sign on using an enterprise Identity Provider. This is used for SAML/OIDC-based enterprise authentication.

```dart 579:619:lib/src/gotrue_client.dart
  /// Obtains a URL to perform a single-sign on using an enterprise Identity
  /// Provider. The redirect URL is implementation and SSO protocol specific.
  ///
  /// You can use it by providing a SSO domain. Typically you can extract this
  /// domain by asking users for their email address. If this domain is
  /// registered on the Auth instance the redirect will use that organization's
  /// currently active SSO Identity Provider for the login.
  ///
  /// If you have built an organization-specific login page, you can use the
  /// organization's SSO Identity Provider UUID directly instead.
  Future<String> getSSOSignInUrl({
    String? providerId,
    String? domain,
    String? redirectTo,
    String? captchaToken,
  }) async {
    assert(
      providerId != null || domain != null,
      'providerId or domain has to be provided.',
    );

    String? codeChallenge;
    String? codeChallengeMethod;
    if (_flowType == AuthFlowType.pkce) {
      assert(_asyncStorage != null,
          'You need to provide asyncStorage to perform pkce flow.');
      final codeVerifier = generatePKCEVerifier();
      await _asyncStorage!.setItem(
          key: '${Constants.defaultStorageKey}-code-verifier',
          value: codeVerifier);
      codeChallenge = generatePKCEChallenge(codeVerifier);
      codeChallengeMethod = codeVerifier == codeChallenge ? 'plain' : 's256';
    }

    final res = await _fetch.request('$_url/sso', RequestMethodType.post,
        options: GotrueRequestOptions(
          body: {
            if (providerId != null) 'provider_id': providerId,
            if (domain != null) 'domain': domain,
            if (redirectTo != null) 'redirect_to': redirectTo,
            if (captchaToken != null)
              'gotrue_meta_security': {'captcha_token': captchaToken},
            'skip_http_redirect': true,
            'code_challenge': codeChallenge,
            'code_challenge_method': codeChallengeMethod,
          },
          headers: _headers,
        ));

    return res['url'] as String;
  }
```

The method:

1. Requires either a `providerId` (UUID) or `domain` (organization domain)
2. Supports PKCE flow with both S256 and plain methods
3. Makes a POST request to `/sso` endpoint
4. Returns the SSO authorization URL

## Session from URL

### getSessionFromUrl

Parses a callback URL (from OAuth or SSO redirect) and establishes a session.

```dart 758:839:lib/src/gotrue_client.dart
  /// Gets the session data from a magic link or oauth2 callback URL
  Future<AuthSessionUrlResponse> getSessionFromUrl(
    Uri originUrl, {
    bool storeSession = true,
  }) async {
    var url = originUrl;
    if (originUrl.hasQuery) {
      final decoded = originUrl.toString().replaceAll('#', '&');
      url = Uri.parse(decoded);
    } else {
      final decoded = originUrl.toString().replaceAll('#', '?');
      url = Uri.parse(decoded);
    }

    final errorDescription = url.queryParameters['error_description'];
    final errorCode = url.queryParameters['error_code'];
    final error = url.queryParameters['error'];
    if (errorDescription != null) {
      throw AuthException(
        errorDescription,
        statusCode: errorCode,
        code: error,
      );
    }

    if (_flowType == AuthFlowType.pkce) {
      final authCode = originUrl.queryParameters['code'];
      if (authCode == null) {
        throw AuthPKCEGrantCodeExchangeError(
            'No code detected in query parameters.');
      }
      return await exchangeCodeForSession(authCode);
    }

    final accessToken = url.queryParameters['access_token'];
    final expiresIn = url.queryParameters['expires_in'];
    final refreshToken = url.queryParameters['refresh_token'];
    final tokenType = url.queryParameters['token_type'];
    final providerToken = url.queryParameters['provider_token'];
    final providerRefreshToken = url.queryParameters['provider_refresh_token'];

    if (accessToken == null) {
      throw AuthException('No access_token detected.');
    }
    if (expiresIn == null) {
      throw AuthException('No expires_in detected.');
    }
    if (refreshToken == null) {
      throw AuthException('No refresh_token detected.');
    }
    if (tokenType == null) {
      throw AuthException('No token_type detected.');
    }

    final user = (await getUser(accessToken)).user;
    if (user == null) {
      throw AuthException('No user found.');
    }

    final session = Session(
      providerToken: providerToken,
      providerRefreshToken: providerRefreshToken,
      accessToken: accessToken,
      expiresIn: int.parse(expiresIn),
      refreshToken: refreshToken,
      tokenType: tokenType,
      user: user,
    );

    final redirectType = url.queryParameters['type'];

    if (storeSession == true) {
      _saveSession(session);
      if (redirectType == 'recovery') {
        notifyAllSubscribers(AuthChangeEvent.passwordRecovery);
      } else {
        notifyAllSubscribers(AuthChangeEvent.signedIn);
      }
    }

    return AuthSessionUrlResponse(session: session, redirectType: redirectType);
  }
```

The method handles two scenarios:

**PKCE Flow:**

1. Extracts the authorization code from query parameters
2. Calls `exchangeCodeForSession` to exchange the code for tokens

**Implicit Flow (legacy):**

1. Extracts tokens from URL parameters
2. Validates all required parameters are present
3. Fetches user details using the access token
4. Creates a session object
5. Stores the session and emits appropriate event

### exchangeCodeForSession

Exchanges an authorization code for a session in PKCE flow.

```dart 333:377:lib/src/gotrue_client.dart
  /// Verifies the PKCE code verifyer and retrieves a session.
  Future<AuthSessionUrlResponse> exchangeCodeForSession(String authCode) async {
    assert(_asyncStorage != null,
        'You need to provide asyncStorage to perform pkce flow.');

    final codeVerifierRawString = await _asyncStorage!
        .getItem(key: '${Constants.defaultStorageKey}-code-verifier');
    if (codeVerifierRawString == null) {
      throw AuthException('Code verifier could not be found in local storage.');
    }
    final codeVerifier = codeVerifierRawString.split('/').first;
    final eventName = codeVerifierRawString.split('/').last;
    final redirectType = AuthChangeEventExtended.fromString(eventName);

    final Map<String, dynamic> response = await _fetch.request(
      '$_url/token',
      RequestMethodType.post,
      options: GotrueRequestOptions(
        headers: _headers,
        body: {
          'auth_code': authCode,
          'code_verifier': codeVerifier,
        },
        query: {
          'grant_type': 'pkce',
        },
      ),
    );

    await _asyncStorage.removeItem(
        key: '${Constants.defaultStorageKey}-code-verifier');

    final authSessionUrlResponse = AuthSessionUrlResponse(
        session: Session.fromJson(response)!, redirectType: redirectType?.name);

    final session = authSessionUrlResponse.session;
    _saveSession(session);
    if (redirectType == AuthChangeEvent.passwordRecovery) {
      notifyAllSubscribers(AuthChangeEvent.passwordRecovery);
    } else {
      notifyAllSubscribers(AuthChangeEvent.signedIn);
    }

    return authSessionUrlResponse;
  }
```

The method:

1. Retrieves the stored code verifier from async storage
2. Parses the event name from the stored value (for determining redirect type)
3. Sends the authorization code and code verifier to the server
4. Clears the stored verifier (one-time use)
5. Creates a session and notifies subscribers

## Identity Management

### getUserIdentities

Retrieves all identities linked to the current user.

```dart 912:916:lib/src/gotrue_client.dart
  /// Gets all the identities linked to a user.
  Future<List<UserIdentity>> getUserIdentities() async {
    final res = await getUser();
    return res.user?.identities ?? [];
  }
```

The method fetches the current user and returns their linked identities.

### linkIdentityWithIdToken

Links a new identity to the current user using an ID token from an OAuth provider.

```dart 930:967:lib/src/gotrue_client.dart
  /// Link an identity to the current user using an ID token.
  ///
  /// [provider] is the OAuth provider
  ///
  /// [idToken] is the ID token from the OAuth provider
  ///
  /// [accessToken] is the access token from the OAuth provider
  ///
  /// [nonce] is the nonce used for the OAuth flow
  ///
  /// [captchaToken] is the verification token received when the user
  /// completes the captcha on the app.
  Future<AuthResponse> linkIdentityWithIdToken({
    required OAuthProvider provider,
    required String idToken,
    String? accessToken,
    String? nonce,
    String? captchaToken,
  }) async {
    final response = await _fetch.request(
      '$_url/token',
      RequestMethodType.post,
      options: GotrueRequestOptions(
        headers: _headers,
        jwt: _currentSession?.accessToken,
        body: {
          'provider': provider.snakeCase,
          'id_token': idToken,
          'nonce': nonce,
          'gotrue_meta_security': {'captcha_token': captchaToken},
          'access_token': accessToken,
          'link_identity': true,
        },
        query: {'grant_type': 'id_token'},
      ),
    );

    final authResponse = AuthResponse.fromJson(response);

    if (authResponse.session == null) {
      throw AuthException(
        'An error occurred on token verification.',
      );
    }

    _saveSession(authResponse.session!);
    notifyAllSubscribers(AuthChangeEvent.userUpdated);

    return authResponse;
  }
```

This allows users to link multiple identity providers to a single account. The method:

1. Sends the ID token to the server with `link_identity: true`
2. Validates the response contains a session
3. Updates the current session with the new identity
4. Emits `userUpdated` event

### getLinkIdentityUrl

Gets the URL to initiate linking an identity with an OAuth provider.

```dart 969:990:lib/src/gotrue_client.dart
  /// Returns the URL to link the user's identity with an OAuth provider.
  Future<OAuthResponse> getLinkIdentityUrl(
    OAuthProvider provider, {
    String? redirectTo,
    String? scopes,
    Map<String, String>? queryParams,
  }) async {
    final urlResponse = await _getUrlForProvider(
      provider,
      url: '$_url/user/identities/authorize',
      redirectTo: redirectTo,
      scopes: scopes,
      queryParams: queryParams,
      skipBrowserRedirect: true,
    );
    final res = await _fetch.request(urlResponse.url, RequestMethodType.get,
        options: GotrueRequestOptions(
          headers: _headers,
          jwt: _currentSession?.accessToken,
        ));
    return OAuthResponse(provider: provider, url: res['url']);
  }
```

The method:

1. Generates the authorization URL for the identity link endpoint
2. Makes an authenticated request to get the actual OAuth URL
3. Returns the URL for initiating the OAuth flow

### unlinkIdentity

Unlinks an identity from a user, preventing future sign-ins with that identity.

```dart 992:1004:lib/src/gotrue_client.dart
  /// Unlinks an identity from a user by deleting it.
  ///
  /// The user will no longer be able to sign in with that identity once it's unlinked.
  Future<void> unlinkIdentity(UserIdentity identity) async {
    await _fetch.request(
      '$_url/user/identities/${identity.identityId}',
      RequestMethodType.delete,
      options: GotrueRequestOptions(
        headers: headers,
        jwt: _currentSession?.accessToken,
      ),
    );
  }
```

The method makes an authenticated DELETE request to remove the specified identity.

## User Management

### getUser

Fetches the current user's details, either from the provided JWT or the current session.

```dart 706:721:lib/src/gotrue_client.dart
  /// Gets the current user details from current session or custom [jwt]
  Future<UserResponse> getUser([String? jwt]) async {
    if (jwt == null && currentSession?.accessToken == null) {
      throw AuthSessionMissingException();
    }
    final options = GotrueRequestOptions(
      headers: _headers,
      jwt: jwt ?? currentSession?.accessToken,
    );
    final response = await _fetch.request(
      '$_url/user',
      RequestMethodType.get,
      options: options,
    );
    return UserResponse.fromJson(response);
  }
```

The method:

1. Requires either a JWT parameter or existing session
2. Makes a GET request to `/user` endpoint with the JWT
3. Returns a `UserResponse` containing user details

### reauthenticate

Sends a reauthentication OTP to the user's email or phone number. This is typically used when sensitive operations require recent authentication.

```dart 638:655:lib/src/gotrue_client.dart
  /// Sends a reauthentication OTP to the user's email or phone number.
  ///
  /// Requires the user to be signed-in.
  Future<void> reauthenticate() async {
    final session = currentSession;
    if (session == null) {
      throw AuthSessionMissingException();
    }

    final options =
        GotrueRequestOptions(headers: headers, jwt: session.accessToken);

    await _fetch.request(
      '$_url/reauthenticate',
      RequestMethodType.get,
      options: options,
    );
  }
```

## Summary

The OAuth and identity management features include:

| Feature | Method | Purpose |
|---------|--------|---------|
| OAuth URL | `getOAuthSignInUrl()` | Generate OAuth authorization URL |
| SSO URL | `getSSOSignInUrl()` | Generate SSO authorization URL |
| Session from URL | `getSessionFromUrl()` | Parse callback and create session |
| Code exchange | `exchangeCodeForSession()` | Exchange PKCE code for tokens |
| List identities | `getUserIdentities()` | Get all linked identities |
| Link identity | `linkIdentityWithIdToken()` | Link new identity via ID token |
| Link identity URL | `getLinkIdentityUrl()` | Get URL for OAuth identity link |
| Unlink identity | `unlinkIdentity()` | Remove linked identity |
| Get user | `getUser()` | Fetch user details |
| Reauthenticate | `reauthenticate()` | Send reauth OTP |

These methods enable comprehensive identity management including:

- Third-party OAuth authentication
- Enterprise SSO integration
- Multi-identity account linking
- Identity unlinking for account security
