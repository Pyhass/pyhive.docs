---
title: Migration Guide
description: Migrating from the legacy pyhiveapi package to pyhive-integration v2.0.0 — what changed and how to update your code.
sidebar: main
---

# Migration Guide

This guide covers changes between the legacy `pyhiveapi` package (pre-2.0.0) and the current `pyhive-integration` v2.0.0.

---

## Package rename

The PyPI distribution name changed. The module names are unchanged.

| Old (PyPI) | New (PyPI) |
|---|---|
| `pyhiveapi` | `pyhive-integration` |

**Update your `requirements.txt` / `pyproject.toml`:**

```diff
- pyhiveapi
+ pyhive-integration
```

**Imports are unchanged:**

```python
# These still work — module names have not changed
from pyhiveapi import Hive, Auth      # sync
from apyhiveapi import Hive, Auth     # async
```

---

## Python version

The minimum Python version is now **3.10**. Python 3.8 and 3.9 are no longer supported.

---

## Method naming — snake_case is now preferred

All public methods have snake_case names. The old camelCase names still work as aliases but are considered deprecated. Update your code to use the new names:

| Old (camelCase) | New (snake_case) |
|---|---|
| `hive.startSession(config)` | `hive.start_session(config)` |
| `hive.updateData(device)` | `hive.update_data(device)` |
| `hive.heating.setMode(device, mode)` | `hive.heating.set_mode(device, mode)` |
| `hive.heating.setTargetTemperature(device, temp)` | `hive.heating.set_target_temperature(device, temp)` |
| `hive.heating.setBoostOn(device, mins, temp)` | `hive.heating.set_boost_on(device, mins, temp)` |
| `hive.heating.setBoostOff(device)` | `hive.heating.set_boost_off(device)` |
| `hive.hotwater.setMode(device, mode)` | `hive.hotwater.set_mode(device, mode)` |
| `hive.hotwater.setBoostOn(device, mins)` | `hive.hotwater.set_boost_on(device, mins)` |
| `hive.hotwater.setBoostOff(device)` | `hive.hotwater.set_boost_off(device)` |
| `hive.light.turnOn(device, ...)` | `hive.light.turn_on(device, ...)` |
| `hive.light.turnOff(device)` | `hive.light.turn_off(device)` |
| `hive.switch.turnOn(device)` | `hive.switch.turn_on(device)` |
| `hive.switch.turnOff(device)` | `hive.switch.turn_off(device)` |
| `hive.action.setStatusOn(device)` | `hive.action.set_status_on(device)` |
| `hive.action.setStatusOff(device)` | `hive.action.set_status_off(device)` |

---

## Device attribute naming — snake_case is now preferred

`Device` fields are now snake_case. Dict-style access with old camelCase keys still works through automatic key translation:

| Old (camelCase) | New (snake_case) |
|---|---|
| `device["hiveID"]` | `device.hive_id` |
| `device["hiveName"]` | `device.hive_name` |
| `device["hiveType"]` | `device.hive_type` |
| `device["haName"]` | `device.ha_name` |
| `device["haType"]` | `device.ha_type` |
| `device["deviceData"]` | `device.device_data` |
| `device["parentDevice"]` | `device.parent_device` |

The old dict-style access (`device["hiveName"]`) is kept for backwards compatibility and will continue to work, but direct attribute access (`device.hive_name`) is preferred.

---

## `device_list` — new key name

The property was renamed. The old name is kept as an alias:

| Old | New |
|---|---|
| `hive.deviceList` | `hive.device_list` |

---

## `SMS_REQUIRED` constant

The `HiveSmsRequired` exception used in older versions has been replaced with the `SMS_REQUIRED` string constant:

```diff
- from pyhiveapi import HiveSmsRequired
- if tokens.get("ChallengeName") == "SMS_MFA":

+ from pyhiveapi import SMS_REQUIRED
+ if tokens.get("ChallengeName") == SMS_REQUIRED:
```

---

## Camera support removed

Camera device support was removed in v2.0.0. If your code references `hive.camera` or `device_list["camera"]`, remove those references.

---

## `updateInterval()` is now a no-op

The `updateInterval(new_interval)` method no longer sets the scan interval. It always returns `True` and does nothing. This alias is kept for backwards compatibility with Home Assistant integrations that call it.

To control the scan interval, set it directly on the session config:

```python
from datetime import timedelta
hive.config.scan_interval = timedelta(seconds=60)
```

---

## Auth flow changes

### Device registration

In v2.0.0, **device registration happens automatically** after a successful `sms_2fa()` call. You no longer need to call a separate `device_registration()` method.

```diff
- result = await auth.sms_2fa(code, session)
- await auth.device_registration("MyDevice")
- print(auth.device_group_key)

+ result = await auth.sms_2fa(code, session)
+ device_data = auth.get_device_data()
+ # device_data is [device_group_key, device_key, device_password]
```

### Passing device data to `start_session`

In older versions, device keys were passed as constructor arguments. In v2.0.0, pass them as a list in the `start_session` config:

```diff
- session = Hive(
-     username="...", password="...",
-     deviceGroupKey="...", deviceKey="...", devicePassword="..."
- )
- session.deviceLogin()
- session.startSession()

+ hive = Hive(username="...", password="...")
+ await hive.start_session({
+     "tokens": tokens,
+     "username": "...",
+     "password": "...",
+     "device_data": [device_group_key, device_key, device_password],
+ })
```

---

## Dependency changes

| Package | Status |
|---|---|
| `boto3`, `botocore` | Still required (AWS Cognito SRP) |
| `aiohttp` | Still required (async HTTP) |
| `requests` | Still required (sync HTTP) |
| `pyquery` | Still required |
| `loguru` | Added in v2.0.0 |

---

## Summary checklist

- [ ] Update `requirements.txt`: `pyhiveapi` → `pyhive-integration`
- [ ] Ensure Python 3.10+
- [ ] Update `hive.startSession()` → `hive.start_session()`
- [ ] Update `hive.deviceList` → `hive.device_list`
- [ ] Update device method names to snake_case
- [ ] Update `device["hiveName"]` etc. to `device.hive_name`
- [ ] Replace `HiveSmsRequired` with `SMS_REQUIRED` constant
- [ ] Remove any camera device references
- [ ] Update device registration to use `auth.get_device_data()` and pass as `device_data` list to `start_session`
