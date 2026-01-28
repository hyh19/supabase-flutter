# GoTrueClient Authentication Methods

This document explains the various authentication methods provided by the `GoTrueClient` class for signing up and signing in users using different authentication strategies.

## Sign Up Methods

### signInAnonymously

Creates a new anonymous user without requiring any credentials. This is useful for allowing users to explore an application before committing to creating an account.

```dart 147:176:lib/src/gotrue_client.dart
  /// Creates a new anonymous user.
  ///
  /// Returns An `AuthResponse` with a session where the `is_anonymous` claim
  /// in the access token JWT is set to true
  Future<AuthResponse> signInAnonymously({
    Map<String, dynamic>? data,
    String? captchaToken,
  }) async {
    final response = await _fetch.request(
      '$_url/signup',
      RequestMethodType.post,
      options: GotrueRequestOptions(
        headers: _headers,
        body: {
          'data': data ?? {},
          'gotrue_meta_security': {'captcha_token': captchaToken},
        },
      ),
    );

    final authResponse = AuthResponse.fromJson(response);

    final session = authResponse.session;
    if (session != null) {
      _saveSession(session);
      notifyAllSubscribers(AuthChangeEvent.signedIn);
    }

    return authResponse;
  }
```

The method sends a POST request to the `/signup` endpoint with optional user data and a captcha token. If successful, it saves the session and notifies all subscribers of the `signedIn` event.

### signUp

Creates a new user account with either an email or phone number and a password. The method supports PKCE (Proof Key for Code Exchange) flow for enhanced security.

```dart 178:263:lib/src/gotrue_client.dart
  /// Creates a new user.
  ///
  /// Be aware that if a user account exists in the system you may get back an
  /// error message that attempts to hide this information from the user.
  /// This method has support for PKCE via email signups. The PKCE flow cannot be used when autoconfirm is enabled.
  ///
  /// Returns a logged-in session if the server has "autoconfirm" ON, but only a user if the server has "autoconfirm" OFF
  ///
  /// [email] is the user's email address
  ///
  /// [phone] is the user's phone number WITH international prefix
  ///
  /// [password] is the password of the user
  ///
  /// [data] sets [User.userMetadata] without an extra call to [updateUser]
  ///
  /// [channel] Messaging channel to use (e.g. whatsapp or sms)
  Future<AuthResponse> signUp({
    String? email,
    String? phone,
    required String password,
    String? emailRedirectTo,
    Map<String, dynamic>? data,
    String? captchaToken,
    OtpChannel channel = OtpChannel.sms,
  }) async {
    assert((email != null && phone == null) || (email == null && phone != null),
        'You must provide either an email or phone number');

    late final Map<String, dynamic> response;

    if (email != null) {
      String? codeChallenge;

      if (_flowType == AuthFlowType.pkce) {
        assert(_asyncStorage != null,
            'You need to provide asyncStorage to perform pkce flow.');
        final codeVerifier = generatePKCEVerifier();
        await _asyncStorage!.setItem(
            key: '${Constants.defaultStorageKey}-code-verifier',
            value: codeVerifier);
        codeChallenge = generatePKCEChallenge(codeVerifier);
      }

      response = await _fetch.request(
        '$_url/signup',
        RequestMethodType.post,
        options: GotrueRequestOptions(
          headers: _headers,
          redirectTo: emailRedirectTo,
          body: {
            'email': email,
            'password': password,
            'data': data,
            'gotrue_meta_security': {'captcha_token': captchaToken},
            'code_challenge': codeChallenge,
            'code_challenge_method': codeChallenge != null ? 's256' : null,
          },
        ),
      );
    } else if (phone != null) {
      final body = {
        'phone': phone,
        'password': password,
        'data': data,
        'gotrue_meta_security': {'captcha_token': captchaToken},
        'channel': channel.name,
      };
      final fetchOptions = GotrueRequestOptions(headers: _headers, body: body);
      response = await _fetch.request('$_url/signup', RequestMethodType.post,
          options: fetchOptions) as Map<String, dynamic>;
    } else {
      throw AuthException(
          'You must provide either an email or phone number and a password');
    }

    final authResponse = AuthResponse.fromJson(response);

    final session = authResponse.session;
    if (session != null) {
      _saveSession(session);
      notifyAllSubscribers(AuthChangeEvent.signedIn);
    }

    return authResponse;
  }
```

The method enforces that exactly one of `email` or `phone` must be provided. For email signups with PKCE flow, it generates a code verifier and challenge pair. The code verifier is stored in async storage and sent as a challenge in the request.

**Key behaviors:**

- If server has "autoconfirm" ON, returns a logged-in session immediately
- If server has "autoconfirm" OFF, only returns a user (email confirmation required)
- PKCE flow cannot be used when autoconfirm is enabled

## Sign In Methods

### signInWithPassword

Authenticates an existing user with their email or phone number and password.

```dart 265:315:lib/src/gotrue_client.dart
  /// Log in an existing user with an email and password or phone and password.
  Future<AuthResponse> signInWithPassword({
    String? email,
    String? phone,
    required String password,
    String? captchaToken,
  }) async {
    late final Map<String, dynamic> response;

    if (email != null) {
      response = await _fetch.request(
        '$_url/token',
        RequestMethodType.post,
        options: GotrueRequestOptions(
          headers: _headers,
          body: {
            'email': email,
            'password': password,
            'gotrue_meta_security': {'captcha_token': captchaToken},
          },
          query: {'grant_type': 'password'},
        ),
      );
    } else if (phone != null) {
      response = await _fetch.request(
        '$_url/token',
        RequestMethodType.post,
        options: GotrueRequestOptions(
          headers: _headers,
          body: {
            'phone': phone,
            'password': password,
            'gotrue_meta_security': {'captcha_token': captchaToken},
          },
          query: {'grant_type': 'password'},
        ),
      );
    } else {
      throw AuthException(
        'You must provide either an email, phone number, a third-party provider or OpenID Connect.',
      );
    }

    final authResponse = AuthResponse.fromJson(response);

    if (authResponse.session?.accessToken != null) {
      _saveSession(authResponse.session!);
      notifyAllSubscribers(AuthChangeEvent.signedIn);
    }
    return authResponse;
  }
```

The method sends a POST request to the `/token` endpoint with `grant_type=password`. If authentication succeeds and a session is returned, it saves the session and emits a `signedIn` event.

### signInWithIdToken

Allows signing in with an ID token issued by supported providers like Apple, Google, Facebook, Kakao, and Keycloak. This is an OpenID Connect (OIDC) based authentication method.

```dart 379:426:lib/src/gotrue_client.dart
  /// Allows signing in with an ID token issued by supported providers.
  /// Common supported providers include Apple, Google, Facebook, Kakao, and Keycloak.
  /// The [idToken] is verified for validity and a new session is established.
  ///
  /// If the ID token contains an `at_hash` claim, then [accessToken] must be
  /// provided to compare its hash with the value in the ID token.
  ///
  /// If the ID token contains a `nonce` claim, then [nonce] must be
  /// provided to compare its hash with the value in the ID token.
  ///
  /// [captchaToken] is the verification token received when the user
  /// completes the captcha on the app.
  Future<AuthResponse> signInWithIdToken({
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
        body: {
          'provider': provider.snakeCase,
          'id_token': idToken,
          'nonce': nonce,
          'gotrue_meta_security': {'captcha_token': captchaToken},
          'access_token': accessToken,
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
    notifyAllSubscribers(AuthChangeEvent.signedIn);

    return authResponse;
  }
```

This method supports additional security validations:

- If the ID token contains an `at_hash` claim, the `accessToken` must be provided for hash comparison
- If the ID token contains a `nonce` claim, the `nonce` parameter must be provided for validation

### signInWithOtp

Sends a one-time password (OTP) or magic link to the user's email or phone for authentication without requiring a password.

```dart 428:503:lib/src/gotrue_client.dart
  /// Log in a user given a User supplied OTP received via mobile.
  ///
  /// If the `{{ .ConfirmationURL }}` variable is specified in the email template, a magiclink will be sent.
  ///
  /// If the `{{ .Token }}` variable is specified in the email template, an OTP will be sent.
  ///
  /// If you're using phone sign-ins, only an OTP will be sent. You won't be able to send a magiclink for phone sign-ins.
  ///
  /// If [shouldCreateUser] is set to false, this method will not create a new user. Defaults to true.
  ///
  /// [emailRedirectTo] can be used to specify the redirect URL embedded in the email link
  ///
  /// [data] can be used to set the user's metadata, which maps to the `auth.users.user_metadata` column.
  ///
  /// [captchaToken] Verification token received when the user completes the captcha on the site.
  ///
  /// [channel] Messaging channel to use (e.g. whatsapp or sms)
  Future<void> signInWithOtp({
    String? email,
    String? phone,
    String? emailRedirectTo,
    bool? shouldCreateUser,
    Map<String, dynamic>? data,
    String? captchaToken,
    OtpChannel channel = OtpChannel.sms,
  }) async {
    if (email != null) {
      String? codeChallenge;
      if (_flowType == AuthFlowType.pkce) {
        assert(_asyncStorage != null,
            'You need to provide asyncStorage to perform pkce flow.');
        final codeVerifier = generatePKCEVerifier();
        await _asyncStorage!.setItem(
            key: '${Constants.defaultStorageKey}-code-verifier',
            value: codeVerifier);
        codeChallenge = generatePKCEChallenge(codeVerifier);
      }
      await _fetch.request(
        '$_url/otp',
        RequestMethodType.post,
        options: GotrueRequestOptions(
          headers: _headers,
          redirectTo: emailRedirectTo,
          body: {
            'email': email,
            'data': data ?? {},
            'create_user': shouldCreateUser ?? true,
            'gotrue_meta_security': {'captcha_token': captchaToken},
            'code_challenge': codeChallenge,
            'code_challenge_method': codeChallenge != null ? 's256' : null,
          },
        ),
      );
      return;
    }
    if (phone != null) {
      final body = {
        'phone': phone,
        'data': data ?? {},
        'create_user': shouldCreateUser ?? true,
        'gotrue_meta_security': {'captcha_token': captchaToken},
        'channel': channel.name,
      };
      final fetchOptions = GotrueRequestOptions(headers: _headers, body: body);

      await _fetch.request(
        '$_url/otp',
        RequestMethodType.post,
        options: fetchOptions,
      );
      return;
    }
    throw AuthException(
      'You must provide either an email, phone number, a third-party provider or OpenID Connect.',
    );
  }
```

The method behavior depends on the email template configuration:

- With `{{ .ConfirmationURL }}` in the template: sends a magiclink
- With `{{ .Token }}` in the template: sends an OTP
- For phone: always sends an OTP

The `shouldCreateUser` parameter controls whether a new user is created if one doesn't exist.

## OTP Verification

### verifyOTP

Verifies an OTP (One-Time Password) received via email or mobile phone, and establishes a session upon successful verification.

```dart 505:567:lib/src/gotrue_client.dart
  /// Log in a user given a User supplied OTP received via mobile.
  ///
  /// [phone] is the user's phone number WITH international prefix
  ///
  /// [token] is the token that user was sent to their mobile phone
  ///
  /// [tokenHash] is the token used in an email link
  Future<AuthResponse> verifyOTP({
    String? email,
    String? phone,
    String? token,
    required OtpType type,
    String? redirectTo,
    String? captchaToken,
    String? tokenHash,
  }) async {
    // For recovery type with tokenHash, only tokenHash and type are required
    final isRecoveryWithTokenHash =
        type == OtpType.recovery && tokenHash != null;

    if (!isRecoveryWithTokenHash) {
      assert(
          ((email != null && phone == null) ||
                  (email == null && phone != null)) ||
              (tokenHash != null),
          '`email` or `phone` needs to be specified.');
      assert(token != null || tokenHash != null,
          '`token` or `tokenHash` needs to be specified.');
    } else {
      // For recovery with tokenHash, email/phone should not be provided
      assert(email == null && phone == null,
          'For recovery type with tokenHash, only tokenHash and type should be provided.');
    }

    final body = {
      // For recovery type with tokenHash, exclude email/phone
      if (!isRecoveryWithTokenHash && email != null) 'email': email,
      if (!isRecoveryWithTokenHash && phone != null) 'phone': phone,
      if (token != null) 'token': token,
      'type': type.snakeCase,
      'redirect_to': redirectTo,
      'gotrue_meta_security': {'captchaToken': captchaToken},
      if (tokenHash != null) 'token_hash': tokenHash,
    };
    final fetchOptions = GotrueRequestOptions(headers: _headers, body: body);
    final response = await _fetch
        .request('$_url/verify', RequestMethodType.post, options: fetchOptions);

    final authResponse = AuthResponse.fromJson(response);

    if (authResponse.session == null) {
      throw AuthException(
        'An error occurred on token verification.',
      );
    }

    _saveSession(authResponse.session!);
    notifyAllSubscribers(type == OtpType.recovery
        ? AuthChangeEvent.passwordRecovery
        : AuthChangeEvent.signedIn);

    return authResponse;
  }
```

The method supports different OTP types:

- `signup`: Email/phone signup confirmation
- `sms`: SMS OTP verification
- `magiclink`: Magic link verification
- `recovery`: Password recovery
- `emailChange`: Email change confirmation
- `phoneChange`: Phone change confirmation

For password recovery with `tokenHash`, the method has special handling where email/phone should not be provided.

## Resend Confirmation

### resend

Resends an existing signup confirmation email, email change email, SMS OTP, or phone change OTP.

```dart 657:704:lib/src/gotrue_client.dart
  /// Resends an existing signup confirmation email, email change email, SMS OTP or phone change OTP.
  ///
  /// For [type] of [OtpType.signup] or [OtpType.emailChange] [email] must be
  /// provided, and for [type] or [OtpType.sms] or [OtpType.phoneChange],
  /// [phone] must be provided
  Future<ResendResponse> resend({
    String? email,
    String? phone,
    required OtpType type,
    String? emailRedirectTo,
    String? captchaToken,
  }) async {
    assert((email != null && phone == null) || (email == null && phone != null),
        '`email` or `phone` needs to be specified.');
    if (email != null) {
      assert([OtpType.signup, OtpType.emailChange].contains(type),
          'email must be provided for type ${type.name}');
    }
    if (phone != null) {
      assert([OtpType.sms, OtpType.phoneChange].contains(type),
          'phone must be provided for type ${type.name}');
    }

    final body = {
      if (email != null) 'email': email,
      if (phone != null) 'phone': phone,
      'type': type.snakeCase,
      'gotrue_meta_security': {'captcha_token': captchaToken},
    };

    final options = GotrueRequestOptions(
      headers: _headers,
      body: body,
      redirectTo: emailRedirectTo,
    );

    final response = await _fetch.request(
      '$_url/resend',
      RequestMethodType.post,
      options: options,
    );

    if ((response as Map).containsKey(['message_id'])) {
      return ResendResponse(messageId: response['message_id']);
    } else {
      return ResendResponse();
    }
  }
```

The method validates that the correct contact information is provided for each OTP type and returns a `ResendResponse` containing the message ID if available.

## Password Reset

### resetPasswordForEmail

Initiates a password reset flow by sending a recovery email to the user.

```dart 875:910:lib/src/gotrue_client.dart
  /// Sends a reset request to an email address.
  Future<void> resetPasswordForEmail(
    String email, {
    String? redirectTo,
    String? captchaToken,
  }) async {
    String? codeChallenge;
    if (_flowType == AuthFlowType.pkce) {
      assert(_asyncStorage != null,
          'You need to provide asyncStorage to perform pkce flow.');
      final codeVerifier = generatePKCEVerifier();
      await _asyncStorage!.setItem(
        key: '${Constants.defaultStorageKey}-code-verifier',
        value: '$codeVerifier/${AuthChangeEvent.passwordRecovery.name}',
      );
      codeChallenge = generatePKCEChallenge(codeVerifier);
    }

    final body = {
      'email': email,
      'gotrue_meta_security': {'captcha_token': captchaToken},
      'code_challenge': codeChallenge,
      'code_challenge_method': codeChallenge != null ? 's256' : null,
    };

    final fetchOptions = GotrueRequestOptions(
      headers: _headers,
      body: body,
      redirectTo: redirectTo,
    );
    await _fetch.request(
      '$_url/recover',
      RequestMethodType.post,
      options: fetchOptions,
    );
  }
```

This method stores the code verifier with the `passwordRecovery` event name so that when the user clicks the recovery link, the correct event can be triggered after code exchange.

## Summary

The `GoTrueClient` provides comprehensive authentication methods covering:

| Method | Purpose | Returns |
|--------|---------|---------|
| `signInAnonymously` | Create anonymous user | AuthResponse with session |
| `signUp` | Create new account | AuthResponse with session/user |
| `signInWithPassword` | Email/phone + password | AuthResponse with session |
| `signInWithIdToken` | OIDC provider login | AuthResponse with session |
| `signInWithOtp` | Send OTP/magiclink | void |
| `verifyOTP` | Verify OTP code | AuthResponse with session |
| `resend` | Resend confirmation | ResendResponse |
| `resetPasswordForEmail` | Initiate password reset | void |

Each method supports captcha verification and PKCE flow where applicable, with proper session management and event notification.
