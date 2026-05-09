---
title: Auth & API Examples
description: Examples for authenticating with the Hive API — first login, SMS 2FA, device login, and token management in both async and sync.
sidebar: main
---

# Auth & API Examples

---

## First login — Async (with SMS 2FA)

```python
import asyncio
from apyhiveapi import Auth, SMS_REQUIRED

async def first_login():
    auth = Auth(username="you@example.com", password="yourpassword")
    result = await auth.login()

    if result.get("ChallengeName") == SMS_REQUIRED:
        code = input("SMS code: ")
        result = await auth.sms_2fa(code, result)
        # Device registration is automatic after sms_2fa.
        # Save these credentials for future device logins (no SMS needed).
        device_data = await auth.get_device_data()
        print("Device group key:", device_data[0])
        print("Device key:",       device_data[1])
        print("Device password:",  device_data[2])

    if "AuthenticationResult" in result:
        ar = result["AuthenticationResult"]
        tokens = {
            "token":        ar["IdToken"],
            "refreshToken": ar["RefreshToken"],
            "accessToken":  ar["AccessToken"],
        }
        print("Tokens:", tokens)
        return tokens

asyncio.run(first_login())
```

---

## First login — Sync

```python
from pyhiveapi import Auth, SMS_REQUIRED

auth = Auth(username="you@example.com", password="yourpassword")
result = auth.login()

if result.get("ChallengeName") == SMS_REQUIRED:
    code = input("SMS code: ")
    result = auth.sms_2fa(code, result)
    device_data = auth.get_device_data()
    print("Device data:", device_data)

if "AuthenticationResult" in result:
    ar = result["AuthenticationResult"]
    tokens = {
        "token":        ar["IdToken"],
        "refreshToken": ar["RefreshToken"],
        "accessToken":  ar["AccessToken"],
    }
```

---

## Device login — Async (no SMS, for subsequent logins)

Once you have saved device credentials from the first login, pass them to `start_session` via the `device_data` key — the session handles the device login internally.

```python
import asyncio
from apyhiveapi import Hive

async def device_login(tokens, device_group_key, device_key, device_password):
    hive = Hive(username="you@example.com", password="yourpassword")
    device_list = await hive.start_session({
        "tokens": tokens,
        "username": "you@example.com",
        "password": "yourpassword",
        "device_data": [device_group_key, device_key, device_password],
    })
    print("Devices:", list(device_list.keys()))
    return hive, device_list

asyncio.run(device_login(tokens, dg_key, d_key, d_password))
```

---

## Device login — Sync

```python
from pyhiveapi import Hive

hive = Hive(username="you@example.com", password="yourpassword")
device_list = hive.start_session({
    "tokens": tokens,
    "username": "you@example.com",
    "password": "yourpassword",
    "device_data": [device_group_key, device_key, device_password],
})
```

---

## Checking for SMS requirement

```python
from apyhiveapi import Auth, SMS_REQUIRED
from apyhiveapi.helper.hive_exceptions import (
    HiveInvalidUsername,
    HiveInvalidPassword,
    HiveApiError,
)

async def safe_login():
    auth = Auth(username="you@example.com", password="yourpassword")

    try:
        result = await auth.login()
    except HiveInvalidUsername:
        return None, "Invalid email address"
    except HiveInvalidPassword:
        return None, "Invalid password"
    except HiveApiError:
        return None, "Cannot reach Hive — check your internet connection"

    needs_sms = result.get("ChallengeName") == SMS_REQUIRED
    return result, None, needs_sms
```

---

## Handling a bad SMS code

```python
from apyhiveapi.helper.hive_exceptions import HiveInvalid2FACode

async def submit_sms(auth, code, session):
    try:
        return await auth.sms_2fa(code, session)
    except HiveInvalid2FACode:
        print("Wrong code — please try again")
        return None
```

---

## Token structure

Auth tokens returned from `login()` or `sms_2fa()` have this structure under `result["AuthenticationResult"]`:

| Key | Description |
|---|---|
| `IdToken` | The bearer token used in API requests |
| `AccessToken` | OAuth access token |
| `RefreshToken` | Long-lived token used to obtain new tokens |
| `ExpiresIn` | Token lifetime in seconds (typically 3600) |

When passing tokens to `start_session`, you can pass the dict directly from `login()` result, or extract and reshape into `{"token": ..., "refreshToken": ..., "accessToken": ...}`.

Both formats are accepted by `start_session` / `update_tokens`.
