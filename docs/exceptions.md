---
title: Exceptions
description: Complete reference for all pyhive-integration custom exceptions — when they are raised and how to handle them.
sidebar: main
---

# Exceptions

All custom exceptions are defined in `apyhiveapi.helper.hive_exceptions` (and identically in `pyhiveapi.helper.hive_exceptions`).

```python
from apyhiveapi.helper.hive_exceptions import HiveReauthRequired, HiveApiError
# or
from pyhiveapi.helper.hive_exceptions import HiveReauthRequired, HiveApiError
```

---

## Exception reference

### `HiveApiError`

**Base class for API errors.**

Raised when the Hive API returns an unexpected response or a network error occurs. `HiveAuthError` inherits from this class.

| When raised | `Auth.login()`, any device `set_*` method, `get_devices()` |
|---|---|
| How to handle | Log the error and retry, or surface to user |

```python
from apyhiveapi.helper.hive_exceptions import HiveApiError

try:
    await hive.heating.set_target_temperature(device, 21.0)
except HiveApiError as e:
    print(f"API call failed: {e}")
```

---

### `HiveAuthError`

**Subclass of `HiveApiError`. HTTP 401 or 403 response.**

Raised when the API returns an authentication error after a token refresh was attempted. The session handles this internally by calling `_retry_login()` — you only see it if all retries fail.

| When raised | Any authenticated API call after token refresh |
|---|---|
| How to handle | Handled internally; surfaces as `HiveReauthRequired` if retries fail |

---

### `HiveRefreshTokenExpired`

**The refresh token is expired.**

Raised by the auth layer when the refresh token itself has expired (not just the access token). The session handles this by falling back to `_retry_login()`.

| When raised | `hive_refresh_tokens()` — refresh token rejected by Cognito |
|---|---|
| How to handle | Handled internally; surfaces as `HiveReauthRequired` if retries fail |

---

### `HiveFailedToRefreshTokens`

**Token refresh API call failed.**

Raised when the token refresh request itself fails (network error, API unavailable).

| When raised | `hive_refresh_tokens()` — refresh request failed |
|---|---|
| How to handle | Handled internally; surfaces as `HiveReauthRequired` if retries fail |

---

### `HiveReauthRequired`

**Full reauthentication required. User interaction needed.**

This is the most important exception for application code to handle. It is raised when all automatic recovery attempts are exhausted and the session can no longer refresh its tokens without user input (e.g. SMS 2FA, new password).

| When raised | All token refresh retries fail; SMS 2FA challenge during device login; invalid credentials |
|---|---|
| How to handle | Prompt the user to log in again; re-run the auth flow |

```python
from apyhiveapi.helper.hive_exceptions import HiveReauthRequired

try:
    device_list = await hive.start_session(config)
except HiveReauthRequired:
    # Tokens are invalid and no device login is possible.
    # Re-authenticate the user from scratch.
    tokens = await fresh_login()
    device_list = await hive.start_session({"tokens": tokens, ...})
```

In Home Assistant, this should trigger a reauthentication config flow:
```python
except HiveReauthRequired:
    config_entry.async_start_reauth(hass)
```

---

### `HiveUnknownConfiguration`

**Session started with missing or invalid configuration.**

Raised by `start_session()` when a non-empty config dict is provided but `tokens` is not present.

| When raised | `start_session({...})` without a `tokens` key |
|---|---|
| How to handle | Ensure you pass `{"tokens": tokens, ...}` to `start_session()` |

```python
# Correct — pass tokens
await hive.start_session({"tokens": auth_tokens, "username": "...", "password": "..."})

# Raises HiveUnknownConfiguration — no tokens
await hive.start_session({"username": "...", "password": "..."})

# Fine — empty dict triggers file mode or existing session
await hive.start_session({})
```

---

### `HiveInvalidUsername`

**Username (email) not found in Hive's system.**

| When raised | `Auth.login()` |
|---|---|
| How to handle | Show an error asking the user to check their email address |

---

### `HiveInvalidPassword`

**Incorrect password.**

| When raised | `Auth.login()` |
|---|---|
| How to handle | Show an error asking the user to check their password |

---

### `HiveInvalid2FACode`

**The SMS 2FA code entered is incorrect.**

| When raised | `Auth.sms_2fa(code, session)` |
|---|---|
| How to handle | Ask the user to re-enter the code; optionally resend |

---

### `HiveInvalidDeviceAuthentication`

**Device is not registered with Hive.**

Raised during the device login flow when the saved device credentials are not recognised by Cognito. This happens if the device was removed from the Hive account or the credentials were corrupted.

| When raised | Internal device login flow in `start_session()` |
|---|---|
| How to handle | Fall back to a full SMS login and re-register the device |

---

### `NoApiToken`

**No API token present.**

| When raised | Low-level API client — token missing before making a request |
|---|---|
| How to handle | Ensure `start_session()` completes before calling device methods |

---

### `FileInUse`

**Internal: fixture file is already open.**

| When raised | File-based testing mode edge case |
|---|---|
| How to handle | Not expected in normal usage |

---

## Exception hierarchy

```
Exception
├── HiveApiError
│   └── HiveAuthError
├── HiveRefreshTokenExpired
├── HiveFailedToRefreshTokens
├── HiveReauthRequired
├── HiveUnknownConfiguration
├── HiveInvalidUsername
├── HiveInvalidPassword
├── HiveInvalid2FACode
├── HiveInvalidDeviceAuthentication
├── NoApiToken
└── FileInUse
```

---

## Handling exceptions in a complete flow

```python
from apyhiveapi import Auth, Hive, SMS_REQUIRED
from apyhiveapi.helper.hive_exceptions import (
    HiveInvalidUsername,
    HiveInvalidPassword,
    HiveInvalid2FACode,
    HiveApiError,
    HiveReauthRequired,
    HiveUnknownConfiguration,
)

async def run():
    auth = Auth(username="you@example.com", password="yourpassword")

    try:
        result = await auth.login()
    except HiveInvalidUsername:
        return "Check your email address."
    except HiveInvalidPassword:
        return "Check your password."
    except HiveApiError:
        return "Cannot reach Hive — check your connection."

    if result.get("ChallengeName") == SMS_REQUIRED:
        code = input("SMS code: ")
        try:
            result = await auth.sms_2fa(code, result)
        except HiveInvalid2FACode:
            return "Invalid code."

    hive = Hive(username="you@example.com", password="yourpassword")
    try:
        device_list = await hive.start_session({"tokens": result, ...})
    except HiveUnknownConfiguration:
        return "Tokens missing from config."
    except HiveReauthRequired:
        return "Reauthentication required."

    # Use the session
    for device in device_list.get("climate", []):
        try:
            device = await hive.heating.get_climate(device)
        except HiveReauthRequired:
            return "Session expired — log in again."
```
