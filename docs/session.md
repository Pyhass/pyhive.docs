---
title: Session
description: The Hive session — constructor, start_session, polling, device_list structure, the Device dataclass, caching, and file-based testing.
sidebar: main
---

# Session

The `Hive` class is the single entry point for the library. It manages authentication, polling, and access to all device modules.

---

## Constructor

```python
from apyhiveapi import Hive  # or: from pyhiveapi import Hive

hive = Hive(
    username: str = None,
    password: str = None,
    websession = None,      # optional: existing aiohttp.ClientSession
)
```

| Parameter | Description |
|---|---|
| `username` | Hive account email address |
| `password` | Hive account password |
| `websession` | Optional `aiohttp.ClientSession` to reuse (e.g. from Home Assistant) |

Creating a `Hive` instance does not make any network calls. Call `start_session()` to authenticate and load devices.

---

## `start_session(config)`

Authenticates the session, fetches device data from the Hive API, and returns the populated `device_list`.

```python
device_list = await hive.start_session(config)
```

### Config dict

| Key | Type | Description |
|---|---|---|
| `tokens` | `dict` | Auth tokens from `Auth.login()` |
| `username` | `str` | Hive account email (can override constructor value) |
| `password` | `str` | Hive account password |
| `device_data` | `list` | `[device_group_key, device_key, device_password]` from a previous registration |

**Minimal config (first login):**
```python
device_list = await hive.start_session({
    "tokens": tokens,
    "username": "you@example.com",
    "password": "yourpassword",
})
```

**With device credentials (subsequent logins — no SMS):**
```python
device_list = await hive.start_session({
    "tokens": tokens,
    "username": "you@example.com",
    "password": "yourpassword",
    "device_data": [device_group_key, device_key, device_password],
})
```

### Raises

| Exception | Condition |
|---|---|
| `HiveUnknownConfiguration` | `config` dict provided but `tokens` key is missing |
| `HiveReauthRequired` | API returned no devices — re-authentication required |

---

## Device modules

After `start_session()`, the following modules are available on the `Hive` instance:

| Attribute | Class | Covers |
|---|---|---|
| `hive.heating` | `Climate` | Thermostats, TRV controls |
| `hive.hotwater` | `WaterHeater` | Hot water |
| `hive.light` | `Light` | All Hive light types |
| `hive.switch` | `Switch` | Smart plugs, actions, Heat on Demand |
| `hive.sensor` | `Sensor` | Motion and contact sensors |
| `hive.hub` | `HiveHub` | Hub / Sense detection |
| `hive.action` | `HiveAction` | Hive automation actions |

---

## `device_list`

`hive.device_list` is a dict keyed by Home Assistant entity type. It is populated by `start_session()`.

```python
{
    "parent":        [...],   # physical Hive hub devices
    "binary_sensor": [...],   # motion sensors, contact sensors, hub sensors
    "climate":       [...],   # heating zones, TRVs
    "light":         [...],   # lights
    "sensor":        [...],   # diagnostic sensors (battery, connectivity)
    "switch":        [...],   # smart plugs, Heat on Demand, actions
    "water_heater":  [...],   # hot water
}
```

Each entry in these lists is a `Device` instance.

---

## The `Device` dataclass

Every device is represented as a `Device` instance (from `apyhiveapi.helper.hivedataclasses`).

### Fields

| Field | Type | Description |
|---|---|---|
| `hive_id` | `str` | Unique ID used in Hive API calls (product/action ID) |
| `hive_name` | `str` | Device name from the Hive API |
| `hive_type` | `str` | Hive device type (e.g. `"heating"`, `"tuneablelight"`) |
| `ha_type` | `str` | Home Assistant entity type (e.g. `"climate"`, `"light"`) |
| `ha_name` | `str` | Display name for the HA entity |
| `device_id` | `str` | Hive device ID (used for online/offline checks) |
| `device_data` | `dict` | Device properties from the Hive API |
| `parent_device` | `str \| None` | ID of the parent hub device |
| `is_group` | `bool` | Whether this is a heating group |
| `category` | `str \| None` | HA entity category (e.g. `"diagnostic"`) |
| `temperature_unit` | `str \| None` | `"C"` or `"F"` from account settings |
| `status` | `dict \| None` | Cached state values (populated by `get_*` module methods) |
| `attributes` | `dict \| None` | HA state attributes |
| `min_temp` | `float \| None` | Minimum temperature (heating devices) |
| `max_temp` | `float \| None` | Maximum temperature (heating devices) |

### Dict-style access

`Device` supports both attribute-style and dict-style access. Legacy camelCase keys are automatically translated:

```python
device.hive_id          # attribute access (preferred)
device["hiveID"]        # dict access — translated to hive_id
device.ha_name          # attribute access
device["haName"]        # dict access — translated to ha_name
```

Full key map: `hiveID → hive_id`, `hiveName → hive_name`, `hiveType → hive_type`,
`haName → ha_name`, `haType → ha_type`, `deviceData → device_data`,
`parentDevice → parent_device`.

---

## Polling

### `update_data(device)`

Rate-limited poll. Fetches latest device state from the Hive API if the scan interval has elapsed (default 120 seconds). Used by Home Assistant update coordinators.

```python
updated = await hive.update_data(device)
# Returns True if a poll was performed, False if still within the scan interval
```

If another poll is already in progress, the call returns immediately without waiting.

### `force_update()`

Bypasses the scan interval and immediately polls the API. Intended for power users and one-shot scripts.

```python
success = await hive.force_update()
# Returns True on success, False if a poll was already in progress
```

### Scan interval

Default scan interval is 120 seconds. It is set on `hive.config.scan_interval` as a `timedelta`. Home Assistant controls this via the integration config — `updateInterval()` is a no-op alias retained for backwards compatibility.

---

## Cache behaviour

To avoid returning stale data when a poll is in-flight or running slowly, device modules check `should_use_cached_data()` before returning live data:

```python
hive.should_use_cached_data()
# Returns True when:
#   • the last poll was slow (> 3 seconds), OR
#   • a different task is currently holding the update_lock
```

When the cache is active, `get_*` methods return the last known device state from `entity_cache` rather than computing fresh values. Cached state is written by `set_cached_device(device)` after a successful `get_*` call.

---

## Session config

`hive.config` is a `SessionConfig` dataclass with the following runtime state:

| Attribute | Type | Description |
|---|---|---|
| `file` | `bool` | `True` when using file-based testing mode |
| `home_id` | `str \| None` | Hive home ID |
| `user_id` | `str \| None` | Hive user ID |
| `username` | `str \| None` | Account email |
| `last_update` | `datetime` | Timestamp of the last successful poll |
| `scan_interval` | `timedelta` | Poll frequency (default 120 s) |
| `battery` | `list` | Device IDs being battery-monitored |
| `mode` | `list` | Product IDs being mode-tracked |

---

## File-based testing

Pass `username="use@file.com"` to load device state from bundled fixture files (`src/data/data.json`) instead of making live API calls. `start_session()` works normally but no real devices are needed:

```python
hive = Hive(username="use@file.com", password="")
device_list = await hive.start_session({})
```

All read methods return data from the fixtures. Write methods silently fail (they check `device.device_data["online"]` which will be `False` for most fixture devices).

---

## Backwards-compatible aliases

| New (preferred) | Old alias |
|---|---|
| `start_session(config)` | `startSession(config)` |
| `update_data(device)` | `updateData(device)` |
| `device_list` | `deviceList` |
