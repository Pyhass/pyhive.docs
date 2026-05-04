---
title: Heating API
description: API reference for hive.heating (Climate) — temperature, mode, boost, heat-on-demand, and schedule methods.
sidebar: api
---

# Heating — `hive.heating`

The `hive.heating` attribute is a `Climate` instance. It covers **heating zones** (`heating`, `trvcontrol`) and is the module used for thermostats and TRV controls.

Devices are found in `hive.device_list["climate"]` after `start_session()`.

---

## Quick reference

| Method | Returns | Description |
|---|---|---|
| `get_climate(device)` | `Device` | Refresh and return full heating state |
| `get_current_temperature(device)` | `float \| None` | Current measured temperature |
| `get_target_temperature(device)` | `float \| None` | Target (set point) temperature |
| `get_mode(device)` | `str \| None` | Current mode |
| `get_state(device)` | `str \| None` | `"ON"` or `"OFF"` (heating/not heating) |
| `get_current_operation(device)` | `bool \| None` | Whether the boiler is currently working |
| `get_boost_status(device)` | `str` | `"ON"` or `"OFF"` |
| `get_boost_time(device)` | `int \| None` | Boost minutes remaining |
| `get_heat_on_demand(device)` | `bool \| None` | Heat on Demand active |
| `get_operation_modes()` | `list` | `["SCHEDULE", "MANUAL", "OFF"]` |
| `get_min_temperature(device)` | `int` | Minimum target temperature |
| `get_max_temperature(device)` | `int` | Maximum target temperature |
| `get_schedule_now_next_later(device)` | `dict \| None` | Schedule slots |
| `minmax_temperature(device)` | `dict \| None` | Today's min/max readings |
| `set_target_temperature(device, temp)` | `bool` | Set target temperature |
| `set_mode(device, mode)` | `bool` | Set heating mode |
| `set_boost_on(device, mins, temp)` | `bool \| None` | Start boost |
| `set_boost_off(device)` | `bool` | Stop boost |
| `set_heat_on_demand(device, state)` | `bool` | Enable/disable Heat on Demand |

---

## `get_climate(device)`

Fetches the latest state for a heating device and populates `device.status`, `device.device_data`, `device.min_temp`, `device.max_temp`, and `device.attributes`.

```python
device = await hive.heating.get_climate(device)

device.min_temp               # float — minimum target temperature
device.max_temp               # float — maximum target temperature
device.status["current_temperature"]   # float
device.status["target_temperature"]    # float
device.status["mode"]                  # "SCHEDULE" | "MANUAL" | "OFF"
device.status["action"]                # bool (boiler working)
device.status["boost"]                 # "ON" | "OFF"
device.attributes                      # dict — battery, online status etc.
```

Returns the cached device state if `should_use_cached_data()` is `True` (poll in-flight or last poll was slow).

---

## `get_current_temperature(device)` → `float | None`

Returns the current measured temperature in degrees. Also updates `session.data.minMax` with today's min/max tracking.

```python
temp = await hive.heating.get_current_temperature(device)
print(f"{temp}°")
```

---

## `get_target_temperature(device)` → `float | None`

Returns the current target (set point) temperature.

```python
target = await hive.heating.get_target_temperature(device)
```

---

## `get_mode(device)` → `str | None`

Returns the current heating mode. If boost is active, returns the pre-boost mode.

| Value | Meaning |
|---|---|
| `"SCHEDULE"` | Following the programmed schedule |
| `"MANUAL"` | Manual temperature hold |
| `"OFF"` | Heating off |

```python
mode = await hive.heating.get_mode(device)
```

---

## `get_state(device)` → `str | None`

Returns `"ON"` if the current temperature is below the target (heating is active), `"OFF"` otherwise.

```python
state = await hive.heating.get_state(device)
```

---

## `get_current_operation(device)` → `bool | None`

Returns `True` if the boiler is currently running (working), `False` if idle.

```python
working = await hive.heating.get_current_operation(device)
```

---

## `get_boost_status(device)` → `str`

Returns `"ON"` if boost is currently active, `"OFF"` otherwise.

```python
boost = await hive.heating.get_boost_status(device)
```

---

## `get_boost_time(device)` → `int | None`

Returns the number of minutes remaining on the boost, or `None` if boost is off.

```python
remaining = await hive.heating.get_boost_time(device)
if remaining:
    print(f"Boost ends in {remaining} minutes")
```

---

## `get_heat_on_demand(device)` → `bool | None`

Returns whether Heat on Demand (AutoBoost) is currently active on a Hive thermostat.

```python
hot = await hive.heating.get_heat_on_demand(device)
```

---

## `get_operation_modes()` → `list`

Returns the list of available operation modes. Static — does not make an API call.

```python
modes = await hive.heating.get_operation_modes()
# ["SCHEDULE", "MANUAL", "OFF"]
```

---

## `get_min_temperature(device)` → `int`

Returns the minimum allowed target temperature. For `nathermostat` devices, reads `props.minHeat`. For all other devices, returns `5`.

```python
min_temp = await hive.heating.get_min_temperature(device)
```

---

## `get_max_temperature(device)` → `int`

Returns the maximum allowed target temperature. For `nathermostat` devices, reads `props.maxHeat`. For all other devices, returns `32`.

```python
max_temp = await hive.heating.get_max_temperature(device)
```

---

## `get_schedule_now_next_later(device)` → `dict | None`

Returns the schedule slots for now, next, and later. Only returns data when `mode == "SCHEDULE"` and the device is online.

```python
schedule = await hive.heating.get_schedule_now_next_later(device)
if schedule:
    print("Now:",   schedule["now"])
    print("Next:",  schedule["next"])
    print("Later:", schedule["later"])
```

---

## `minmax_temperature(device)` → `dict | None`

Returns today's min and max measured temperatures from the session cache.

```python
minmax = await hive.heating.minmax_temperature(device)
if minmax:
    print(f"Today: {minmax['TodayMin']}° – {minmax['TodayMax']}°")
    print(f"Since restart: {minmax['RestartMin']}° – {minmax['RestartMax']}°")
```

---

## `set_target_temperature(device, new_temp)` → `bool`

Sets the target temperature. Requires the device to be online.

```python
success = await hive.heating.set_target_temperature(device, 21.0)
```

| Parameter | Type | Description |
|---|---|---|
| `device` | `Device` | Heating device |
| `new_temp` | `float` | Target temperature in degrees |

Returns `True` on success, `False` if the device is offline or the API call failed.

---

## `set_mode(device, new_mode)` → `bool`

Sets the heating mode.

```python
success = await hive.heating.set_mode(device, "SCHEDULE")
```

| Mode | Description |
|---|---|
| `"SCHEDULE"` | Follow the programmed schedule |
| `"MANUAL"` | Hold at current target temperature |
| `"OFF"` | Turn heating off |

---

## `set_boost_on(device, mins, temp)` → `bool | None`

Starts a heating boost for `mins` minutes at `temp` degrees.

```python
success = await hive.heating.set_boost_on(device, mins=30, temp=22.0)
```

| Parameter | Type | Description |
|---|---|---|
| `device` | `Device` | Heating device |
| `mins` | `int` | Boost duration in minutes (must be > 0) |
| `temp` | `float` | Boost temperature (must be within min/max range) |

Returns `None` if `temp` or `mins` are out of range, `True` on success, `False` on API failure.

---

## `set_boost_off(device)` → `bool`

Cancels an active boost and restores the previous mode.

```python
success = await hive.heating.set_boost_off(device)
```

Reads `props.previous.mode` from the device to determine the pre-boost state and restores it automatically.

---

## `set_heat_on_demand(device, state)` → `bool`

Enables or disables the Heat on Demand (AutoBoost) feature on a Hive thermostat.

```python
success = await hive.heating.set_heat_on_demand(device, "ENABLED")
success = await hive.heating.set_heat_on_demand(device, "DISABLED")
```

---

## Backwards-compatible aliases

| Alias | Current name |
|---|---|
| `setMode(device, mode)` | `set_mode(device, mode)` |
| `setTargetTemperature(device, temp)` | `set_target_temperature(device, temp)` |
| `setBoostOn(device, mins, temp)` | `set_boost_on(device, mins, temp)` |
| `setBoostOff(device)` | `set_boost_off(device)` |
| `getClimate(device)` | `get_climate(device)` |

---

## Sync usage

All methods are identical without `await`:

```python
from pyhiveapi import Hive

hive = Hive(username="...", password="...")
device_list = hive.start_session(config)

for device in device_list.get("climate", []):
    device = hive.heating.get_climate(device)
    print(device.status["current_temperature"])
    hive.heating.set_target_temperature(device, 21.0)
```
