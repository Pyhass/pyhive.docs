---
title: Getting Started
description: Install pyhive-integration and run your first Hive session in under 5 minutes.
sidebar: main
---

# Getting Started

## Installation

```bash
pip install pyhive-integration
```

Requires Python 3.10 or later.

---

## Async quick start

```python
import asyncio
from apyhiveapi import Auth, Hive, SMS_REQUIRED

async def main():
    # 1. Authenticate
    auth = Auth(username="you@example.com", password="yourpassword")
    tokens = await auth.login()

    # Handle SMS two-factor authentication if required
    if tokens.get("ChallengeName") == SMS_REQUIRED:
        code = input("Enter the SMS code sent to your phone: ")
        tokens = await auth.sms_2fa(code, tokens)
        # Save device credentials for future logins (no SMS required next time)
        device_data = await auth.get_device_data()
        print("Save these for future logins:", device_data)

    # 2. Start a session
    hive = Hive(username="you@example.com", password="yourpassword")
    device_list = await hive.start_session({
        "tokens": tokens,
        "username": "you@example.com",
        "password": "yourpassword",
    })

    # 3. Inspect discovered devices
    for entity_type, devices in device_list.items():
        for device in devices:
            print(f"{entity_type}: {device.ha_name} ({device.hive_type})")

asyncio.run(main())
```

---

## Sync quick start

```python
from pyhiveapi import Auth, Hive, SMS_REQUIRED

# 1. Authenticate
auth = Auth(username="you@example.com", password="yourpassword")
tokens = auth.login()

if tokens.get("ChallengeName") == SMS_REQUIRED:
    code = input("Enter the SMS code sent to your phone: ")
    tokens = auth.sms_2fa(code, tokens)
    device_data = auth.get_device_data()
    print("Save these for future logins:", device_data)

# 2. Start a session
hive = Hive(username="you@example.com", password="yourpassword")
device_list = hive.start_session({
    "tokens": tokens,
    "username": "you@example.com",
    "password": "yourpassword",
})

# 3. List devices
for entity_type, devices in device_list.items():
    for device in devices:
        print(f"{entity_type}: {device.ha_name}")
```

> **Note:** Never mix `apyhiveapi` and `pyhiveapi` imports in the same program. Pick one and use it consistently.

---

## Subsequent logins (device login — no SMS required)

After the first SMS login, save the device credentials returned by `auth.get_device_data()`. On subsequent runs, pass them in the config to skip SMS:

```python
# Async
hive = Hive(username="you@example.com", password="yourpassword")
device_list = await hive.start_session({
    "tokens": tokens,
    "username": "you@example.com",
    "password": "yourpassword",
    "device_data": [device_group_key, device_key, device_password],
})
```

See [Authentication](authentication) for the full login flow.

---

## File-based testing (no live credentials)

Set `username="use@file.com"` to load device state from bundled JSON fixture files instead of calling the Hive API. Useful for development and testing without real credentials:

```python
hive = Hive(username="use@file.com", password="")
device_list = await hive.start_session({})
```

All read methods work as normal against the fixture data. Write methods (set temperature, turn on, etc.) are not applied to a real device.

---

## What's next?

- [Authentication](authentication) — full auth flow including SMS 2FA and device registration
- [Session](session) — session lifecycle, polling, and the `device_list` structure
- [API Reference](api/heating) — all device modules and their methods
- [Examples](examples/session) — complete worked examples for all device types
