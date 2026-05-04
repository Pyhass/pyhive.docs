---
title: Hot Water API
description: API reference for hive.hotwater (WaterHeater) — mode, boost, state, and schedule methods.
sidebar: api
---

# Hot Water — `hive.hotwater`

The `hive.hotwater` attribute is a `WaterHeater` instance. It covers hot water cylinders (`hotwater` type).

Devices are found in `hive.device_list["water_heater"]` after `start_session()`.

---

## Quick reference

| Method | Returns | Description |
|---|---|---|
| `get_water_heater(device)` | `Device` | Refresh and return full hot water state |
| `get_mode(device)` | `str \| None` | Current mode |
| `get_state(device)` | `str \| None` | `"ON"` or `"OFF"` |
| `get_boost(device)` | `str` | `"ON"` or `"OFF"` |
| `get_boost_time(device)` | `int \| None` | Boost minutes remaining |
| `get_operation_modes()` | `list` | `["SCHEDULE", "ON", "OFF"]` |
| `get_schedule_now_next_later(device)` | `dict \| None` | Schedule slots |
| `set_mode(device, mode)` | `bool` | Set hot water mode |
| `set_boost_on(device, mins)` | `bool` | Start boost |
| `set_boost_off(device)` | `bool` | Stop boost |

---

## `get_water_heater(device)`

Fetches the latest state and populates `device.status`, `device.device_data`, and `device.attributes`.

```python
device = await hive.hotwater.get_water_heater(device)

device.status["current_operation"]   # "SCHEDULE" | "ON" | "OFF"
device.device_data["online"]          # bool
device.attributes                     # dict
```

Returns the cached device state if `should_use_cached_data()` is `True`.

---

## `get_mode(device)` → `str | None`

Returns the current hot water mode. If boost is active, returns the pre-boost mode.

| Value | Meaning |
|---|---|
| `"SCHEDULE"` | Following the programmed schedule |
| `"ON"` | Continuously on |
| `"OFF"` | Hot water off |

```python
mode = await hive.hotwater.get_mode(device)
```

---

## `get_state(device)` → `str | None`

Returns `"ON"` or `"OFF"` based on the current status. In schedule mode, checks the current schedule slot and boost status to determine the actual state.

```python
state = await hive.hotwater.get_state(device)
```

---

## `get_boost(device)` → `str`

Returns `"ON"` if boost is currently active, `"OFF"` otherwise.

```python
boost = await hive.hotwater.get_boost(device)
```

---

## `get_boost_time(device)` → `int | None`

Returns the number of minutes remaining on the boost, or `None` if boost is off.

```python
remaining = await hive.hotwater.get_boost_time(device)
if remaining:
    print(f"Boost ends in {remaining} minutes")
```

---

## `get_operation_modes()` → `list`

Returns the list of available operation modes. Static — does not make an API call.

```python
modes = await hive.hotwater.get_operation_modes()
# ["SCHEDULE", "ON", "OFF"]
```

---

## `get_schedule_now_next_later(device)` → `dict | None`

Returns the schedule slots for now, next, and later. Only returns data when `mode == "SCHEDULE"`.

```python
schedule = await hive.hotwater.get_schedule_now_next_later(device)
if schedule:
    print("Now:",   schedule["now"])
    print("Next:",  schedule["next"])
    print("Later:", schedule["later"])
```

---

## `set_mode(device, new_mode)` → `bool`

Sets the hot water mode.

```python
success = await hive.hotwater.set_mode(device, "SCHEDULE")
```

| Mode | Description |
|---|---|
| `"SCHEDULE"` | Follow the programmed schedule |
| `"ON"` | Turn hot water on continuously |
| `"OFF"` | Turn hot water off |

Returns `True` on success, `False` on API failure.

---

## `set_boost_on(device, mins)` → `bool`

Starts a hot water boost for `mins` minutes.

```python
success = await hive.hotwater.set_boost_on(device, mins=60)
```

| Parameter | Type | Description |
|---|---|---|
| `device` | `Device` | Hot water device |
| `mins` | `int` | Boost duration in minutes (must be > 0) |

Requires the device to be online. Returns `True` on success.

---

## `set_boost_off(device)` → `bool`

Cancels an active boost and restores the previous mode.

```python
success = await hive.hotwater.set_boost_off(device)
```

Reads `props.previous.mode` from the device and restores the pre-boost state.

---

## Backwards-compatible aliases

| Alias | Current name |
|---|---|
| `setMode(device, mode)` | `set_mode(device, mode)` |
| `setBoostOn(device, mins)` | `set_boost_on(device, mins)` |
| `setBoostOff(device)` | `set_boost_off(device)` |
| `getWaterHeater(device)` | `get_water_heater(device)` |

---

## Sync usage

```python
from pyhiveapi import Hive

hive = Hive(username="...", password="...")
device_list = hive.start_session(config)

for device in device_list.get("water_heater", []):
    device = hive.hotwater.get_water_heater(device)
    print(device.status["current_operation"])
    hive.hotwater.set_boost_on(device, mins=30)
    hive.hotwater.set_boost_off(device)
```
