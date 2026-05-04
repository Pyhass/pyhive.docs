---
title: Authentication
description: How to authenticate with the Hive API using AWS Cognito SRP — first login, SMS 2FA, device registration, and device login for subsequent sessions.
sidebar: main
---

# Authentication

pyhive-integration authenticates against the Hive platform using **AWS Cognito SRP** (Secure Remote Password). Passwords are never sent in plaintext — the SRP protocol performs a cryptographic key exchange.

---

## Authentication flows

There are two flows depending on whether your device is already registered with Hive:

### First login (SMS 2FA)

Used the very first time, or on a new device. Hive will send an SMS code to your registered phone number.

```
Auth.login()
  → ChallengeName == "SMS_MFA"  →  Auth.sms_2fa(code, session)
                                         ↓
                                  Device registered automatically
                                         ↓
                                  Save device_group_key, device_key, device_password
```

### Subsequent logins (device login — no SMS)

Once your device is registered, subsequent logins use the saved device credentials and do not require SMS:

```
Hive.start_session(config with device_data)
  → internal device login (HiveAuthAsync.device_login())
  → tokens returned
```

---

## Step-by-step: first login

### Async

```python
import asyncio
from apyhiveapi import Auth, SMS_REQUIRED
from apyhiveapi.helper.hive_exceptions import (
    HiveInvalidUsername,
    HiveInvalidPassword,
    HiveInvalid2FACode,
    HiveApiError,
)

async def first_login():
    auth = Auth(username="you@example.com", password="yourpassword")

    try:
        result = await auth.login()
    except HiveInvalidUsername:
        print("Invalid username.")
        return
    except HiveInvalidPassword:
        print("Invalid password.")
        return
    except HiveApiError:
        print("API error — check your internet connection.")
        return

    if result.get("ChallengeName") == SMS_REQUIRED:
        code = input("Enter the SMS code sent to your phone: ")
        try:
            result = await auth.sms_2fa(code, result)
        except HiveInvalid2FACode:
            print("Invalid code — try again.")
            return

    # Device registration happens automatically after sms_2fa().
    # Save these three values securely for future logins.
    device_data = auth.get_device_data()
    print("device_group_key:", device_data[0])
    print("device_key:",       device_data[1])
    print("device_password:",  device_data[2])

    # Extract tokens for start_session
    if "AuthenticationResult" in result:
        tokens = {
            "token":        result["AuthenticationResult"]["IdToken"],
            "refreshToken": result["AuthenticationResult"]["RefreshToken"],
            "accessToken":  result["AuthenticationResult"]["AccessToken"],
        }
        return tokens, device_data

asyncio.run(first_login())
```

### Sync

```python
from pyhiveapi import Auth, SMS_REQUIRED

auth = Auth(username="you@example.com", password="yourpassword")
result = auth.login()

if result.get("ChallengeName") == SMS_REQUIRED:
    code = input("Enter the SMS code: ")
    result = auth.sms_2fa(code, result)

device_data = auth.get_device_data()
```

---

## Step-by-step: subsequent logins (device login)

Pass the saved device credentials to `start_session`. The library handles the device login flow internally — no SMS required.

```python
# Async
from apyhiveapi import Hive

hive = Hive(username="you@example.com", password="yourpassword")
device_list = await hive.start_session({
    "tokens": tokens,           # tokens from previous login or refresh
    "username": "you@example.com",
    "password": "yourpassword",
    "device_data": [            # saved from first login
        device_group_key,
        device_key,
        device_password,
    ],
})
```

If the device is no longer remembered by Cognito (e.g. it was removed from your Hive account), `start_session` will raise `HiveInvalidDeviceAuthentication` and you must fall back to a full SMS login.

---

## Token lifecycle

Tokens are automatically managed by the session:

| Event | What happens |
|---|---|
| Token is at 90% of its lifetime | Proactive silent refresh via `hive_refresh_tokens()` |
| Refresh token expired | Falls back to device login (3 attempts with backoff) |
| Device login also fails | Raises `HiveReauthRequired` — user interaction needed |
| `HiveReauthRequired` raised | Caller must prompt for new credentials / SMS code |

The 90% threshold means the library refreshes tokens before they expire during an API call, not after. You do not need to manage token expiry yourself.

---

## The `SMS_REQUIRED` constant

`SMS_REQUIRED` is a string constant exported from both packages:

```python
from apyhiveapi import SMS_REQUIRED   # async
from pyhiveapi import SMS_REQUIRED    # sync

print(SMS_REQUIRED)  # "SMS_MFA"
```

Use it to detect the SMS challenge without hardcoding the string `"SMS_MFA"`.

---

## `Auth` constructor

```python
Auth(
    username: str,
    password: str,
    device_group_key: str = None,   # from get_device_data()[0]
    device_key: str = None,         # from get_device_data()[1]
    device_password: str = None,    # from get_device_data()[2]
)
```

| Parameter | Required | Description |
|---|---|---|
| `username` | Yes | Hive account email address |
| `password` | Yes | Hive account password |
| `device_group_key` | No | Device group key from a previous registration |
| `device_key` | No | Device key from a previous registration |
| `device_password` | No | Device password from a previous registration |

> **Important:** Only the Hive account owner is supported. Guest accounts cannot authenticate.

---

## Exceptions during authentication

See [Exceptions](../exceptions) for the full reference. The auth-specific exceptions are:

| Exception | When raised |
|---|---|
| `HiveInvalidUsername` | `login()` — username not found |
| `HiveInvalidPassword` | `login()` — password incorrect |
| `HiveInvalid2FACode` | `sms_2fa()` — wrong code entered |
| `HiveInvalidDeviceAuthentication` | Device login — device not registered |
| `HiveReauthRequired` | All retries exhausted, or SMS required on device login |
| `HiveApiError` | Network error or unexpected API response |
