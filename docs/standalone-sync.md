---
title: Standalone — Sync
description: How to use pyhive-integration as a standalone synchronous Python library — scripts, tools, and non-asyncio contexts.
sidebar: main
---

# Standalone — Sync

Use `pyhiveapi` when your code is synchronous — scripts, CLI tools, Jupyter notebooks, or any context where you do not want to manage `asyncio` event loops.

The sync package is auto-generated from the async source and exposes an identical API without `await` keywords.

> **Note:** Never import from both `pyhiveapi` and `apyhiveapi` in the same program. Pick one.

---

## Full example

```python
from pyhiveapi import Auth, Hive, SMS_REQUIRED

# ──────────────────────────────────────────────
# 1.  Authentication
# ──────────────────────────────────────────────
auth = Auth(username="you@example.com", password="yourpassword")
result = auth.login()

if result.get("ChallengeName") == SMS_REQUIRED:
    code = input("SMS code: ")
    result = auth.sms_2fa(code, result)
    device_data = auth.get_device_data()
    print("Device credentials:", device_data)

tokens = {
    "token":        result["AuthenticationResult"]["IdToken"],
    "refreshToken": result["AuthenticationResult"]["RefreshToken"],
    "accessToken":  result["AuthenticationResult"]["AccessToken"],
}

# ──────────────────────────────────────────────
# 2.  Start session
# ──────────────────────────────────────────────
hive = Hive(username="you@example.com", password="yourpassword")
device_list = hive.start_session({
    "tokens": tokens,
    "username": "you@example.com",
    "password": "yourpassword",
    # "device_data": [group_key, key, password],  # for subsequent runs
})

# ──────────────────────────────────────────────
# 3.  Heating
# ──────────────────────────────────────────────
for device in device_list.get("climate", []):
    device = hive.heating.get_climate(device)
    print(f"\n=== Heating: {device.ha_name} ===")
    print(f"  Current temp : {device.status['current_temperature']}°")
    print(f"  Target temp  : {device.status['target_temperature']}°")
    print(f"  Mode         : {device.status['mode']}")

    hive.heating.set_target_temperature(device, 21.0)
    hive.heating.set_mode(device, "SCHEDULE")
    hive.heating.set_boost_on(device, mins=30, temp=22.0)
    hive.heating.set_boost_off(device)

# ──────────────────────────────────────────────
# 4.  Hot water
# ──────────────────────────────────────────────
for device in device_list.get("water_heater", []):
    device = hive.hotwater.get_water_heater(device)
    print(f"\n=== Hot Water: {device.ha_name} ===")
    print(f"  Mode : {device.status['current_operation']}")

    hive.hotwater.set_boost_on(device, mins=60)
    hive.hotwater.set_boost_off(device)
    hive.hotwater.set_mode(device, "SCHEDULE")

# ──────────────────────────────────────────────
# 5.  Lights
# ──────────────────────────────────────────────
for device in device_list.get("light", []):
    device = hive.light.get_light(device)
    print(f"\n=== Light: {device.ha_name} ===")
    print(f"  State : {'On' if device.status['state'] else 'Off'}")

    hive.light.turn_on(device, brightness=None, color_temp=None, color=None)
    hive.light.turn_on(device, brightness=200, color_temp=None, color=None)
    hive.light.turn_off(device)

# ──────────────────────────────────────────────
# 6.  Smart plugs
# ──────────────────────────────────────────────
for device in device_list.get("switch", []):
    if device.hive_type == "activeplug":
        device = hive.switch.get_switch(device)
        print(f"\n=== Plug: {device.ha_name} — {device.status.get('power_usage')} W ===")
        hive.switch.turn_on(device)
        hive.switch.turn_off(device)

# ──────────────────────────────────────────────
# 7.  Sensors
# ──────────────────────────────────────────────
for device in device_list.get("binary_sensor", []):
    device = hive.sensor.get_sensor(device)
    print(f"\n=== Sensor: {device.ha_name} — {device.status.get('state')} ===")

# ──────────────────────────────────────────────
# 8.  Actions (Hive automations)
# ──────────────────────────────────────────────
for device in device_list.get("switch", []):
    if device.hive_type == "action":
        result = hive.action.get_action(device)
        if result == "REMOVE":
            continue
        print(f"\n=== Action: {device.ha_name} — enabled={device.status['state']} ===")
        hive.action.set_status_on(device)
        hive.action.set_status_off(device)
```

---

## Subsequent logins

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

## Error handling

```python
from pyhiveapi.helper.hive_exceptions import HiveReauthRequired, HiveApiError

try:
    hive.heating.set_target_temperature(device, 20.0)
except HiveReauthRequired:
    print("Re-authentication required — run the login flow again.")
except HiveApiError as e:
    print(f"API error: {e}")
```

---

## How the sync package works

The `pyhiveapi` package is generated from `apyhiveapi` source code at build time using [unasync](https://github.com/python-trio/unasync). The build step:

- Renames `apyhiveapi` → `pyhiveapi` throughout the generated files
- Replaces `async def` → `def`, removes all `await` keywords
- Replaces `asyncio` threading primitives with `threading` equivalents

You should never edit the generated sync files. The source of truth is always the async source in `src/`.
