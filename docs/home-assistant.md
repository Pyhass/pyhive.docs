---
title: Home Assistant Integration
description: How pyhive-integration integrates with Home Assistant — auth flow, entity creation, polling, and reading device state.
sidebar: main
---

# Home Assistant Integration

pyhive-integration is the library used by the [Hive Custom Component](https://github.com/Pyhive/Pyhiveapi) for Home Assistant. This page describes how the integration uses the library so that integration developers can follow the same patterns.

---

## Overview

The integration uses the **async** package (`apyhiveapi`) since Home Assistant is built on `asyncio`. The sync package is not used in Home Assistant contexts.

---

## Config entry setup

On initial setup (config flow), the integration:

1. Creates an `Auth` instance and calls `auth.login()`
2. If `ChallengeName == SMS_REQUIRED`, presents the user with an SMS code input
3. Calls `auth.sms_2fa(code, result)` with the entered code
4. Saves the device credentials from `auth.get_device_data()` to the config entry
5. Saves the auth tokens to the config entry
6. Stores `username` and `password` for token refresh fallback

```python
from apyhiveapi import Auth, SMS_REQUIRED

auth = Auth(username=username, password=password)
result = await auth.login()

if result.get("ChallengeName") == SMS_REQUIRED:
    result = await auth.sms_2fa(sms_code, result)

tokens = result["AuthenticationResult"]
device_data = await auth.get_device_data()
# Store tokens and device_data in config entry
```

---

## Session startup (`async_setup_entry`)

On entry setup, the integration creates a `Hive` instance and calls `start_session()` with all saved credentials. This fetches the device list and populates `hive.device_list`.

```python
from apyhiveapi import Hive

hive = Hive(username=username, password=password)

device_list = await hive.start_session({
    "tokens": stored_tokens,
    "username": username,
    "password": password,
    "device_data": stored_device_data,  # [group_key, key, password]
})
```

`device_list` is a dict keyed by HA entity type. The integration passes the relevant sub-list to each platform (`climate`, `light`, `switch`, etc.) via `hass.config_entries.async_forward_entry_setups`.

---

## Entity creation

Each entry in `device_list["climate"]` (or `"light"`, `"switch"`, etc.) corresponds to one Home Assistant entity. A typical platform setup:

```python
async def async_setup_entry(hass, config_entry, async_add_entities):
    hive = hass.data[DOMAIN][config_entry.entry_id]
    devices = hive.device_list["climate"]
    async_add_entities([HiveClimateEntity(hive, device) for device in devices], True)
```

The `Device` instance is passed to each entity and retained as `self.device`. All state reads go through the device module (`hive.heating`, `hive.light`, etc.) using the device as a key.

---

## Update flow

Home Assistant entities call `hive.update_data(device)` in their update methods. This is rate-limited to the scan interval (default 120 s) — multiple entities calling it concurrently will result in only one actual API call:

```python
class HiveClimateEntity(ClimateEntity):
    async def async_update(self):
        await self.hive.update_data(self.device)
        self.device = await self.hive.heating.get_climate(self.device)
```

`get_climate()` (and equivalent methods for other device types) updates `device.status`, `device.device_data`, and `device.attributes` in place, then returns the updated device. The method respects the session cache — if a poll is slow or in-flight it returns the last known state.

---

## Reading device state

After `get_*` is called, state is available from `device.status`:

```python
# Heating
device = await hive.heating.get_climate(device)
current_temp  = device.status["current_temperature"]   # float
target_temp   = device.status["target_temperature"]    # float
mode          = device.status["mode"]                  # "SCHEDULE" | "MANUAL" | "OFF"
action        = device.status["action"]                # bool (working)
boost_status  = device.status["boost"]                 # "ON" | "OFF"

# Light
device = await hive.light.get_light(device)
state         = device.status["state"]                 # bool
brightness    = device.status["brightness"]            # int 0-255
color_temp    = device.status.get("color_temp")        # int mireds (tunable/colour lights)
color_mode    = device.status.get("mode")              # "COLOUR" | "WHITE"
hs_color      = device.status.get("hs_color")          # tuple (colour lights in COLOUR mode)

# Switch / Smart Plug
device = await hive.switch.get_switch(device)
state         = device.status["state"]                 # bool
power_usage   = device.status.get("power_usage")       # float watts (activeplug only)
```

---

## HA state attributes

`device.attributes` is populated by `session.attr.state_attributes()` and contains additional data for HA entity attributes:

```python
# For climate devices:
device.attributes["battery_level"]    # int | None
device.attributes["online"]           # bool
device.attributes["mode"]             # str

# For binary sensors:
device.attributes["battery_level"]    # int | None
device.attributes["online"]           # bool
```

---

## Token refresh and reauthentication

Token refresh is fully automatic — you do not need to handle it explicitly. However, if all refresh attempts fail (e.g. the user changes their Hive password), `update_data()` will raise `HiveReauthRequired`. The integration should catch this and trigger a re-auth config flow:

```python
from apyhiveapi.helper.hive_exceptions import HiveReauthRequired

try:
    await self.hive.update_data(self.device)
except HiveReauthRequired:
    self.hive.config_entry.async_start_reauth(hass)
```

---

## Passing an existing websession

If Home Assistant already has an `aiohttp.ClientSession`, pass it to avoid creating a second one:

```python
from aiohttp import ClientSession

hive = Hive(
    username=username,
    password=password,
    websession=hass.helpers.aiohttp_client.async_get_clientsession(hass),
)
```

---

## Device key access patterns

`Device` supports both attribute access and dict-style access. The dict style is provided for backwards compatibility with older integration code that used `device["hiveName"]` etc.:

```python
device.ha_name          # preferred
device["haName"]        # legacy — still works
device["hiveName"]      # legacy — maps to device.hive_name
```

See [Session — Dict-style access](session#dict-style-access) for the full key map.
