---
title: Sensors API
description: API reference for hive.sensor (Sensor) — motion sensors, contact sensors, and diagnostic sensors.
sidebar: api
---

# Sensors — `hive.sensor`

The `hive.sensor` attribute is a `Sensor` instance. It covers **motion sensors**, **contact sensors**, and diagnostic sensors.

Devices are found in `hive.device_list["binary_sensor"]` and `hive.device_list["sensor"]` after `start_session()`.

---

## Sensor types

| `device.hive_type` | `device_list` key | Description |
|---|---|---|
| `"motionsensor"` | `"binary_sensor"` | Hive motion detector |
| `"contactsensor"` | `"binary_sensor"` | Hive door/window contact sensor |
| `"Availability"` | `"sensor"` | Device online/offline diagnostic |
| `"Connectivity"` | `"sensor"` | Hub connectivity diagnostic |

---

## Quick reference

| Method | Returns | Description |
|---|---|---|
| `get_sensor(device)` | `Device` | Refresh and return full sensor state |
| `get_state(device)` | `bool \| None` | Sensor state (motion active or contact open) |
| `online(device)` | `bool \| None` | Whether the device is online |

---

## `get_sensor(device)`

Fetches the latest state and populates `device.status`, `device.device_data`, and `device.attributes`.

```python
device = await hive.sensor.get_sensor(device)

device.status["state"]            # bool — meaning depends on sensor type
device.device_data["online"]       # bool
device.attributes                  # dict — battery, online status
```

Returns the cached device state if `should_use_cached_data()` is `True`.

### State values by sensor type

| `device.hive_type` | `device.status["state"]` | Meaning |
|---|---|---|
| `"motionsensor"` | `True` | Motion detected |
| `"motionsensor"` | `False` | No motion |
| `"contactsensor"` | `True` | Open |
| `"contactsensor"` | `False` | Closed |
| `"Availability"` | `True` | Device is online |
| `"Availability"` | `False` | Device is offline |

---

## `get_state(device)` → `bool | None`

Returns the sensor state:
- **Contact sensor:** `True` = open, `False` = closed
- **Motion sensor:** `True` (active) or `False` (clear) based on `props.motion.status`

```python
state = await hive.sensor.get_state(device)

if device.hive_type == "contactsensor":
    print("Door is", "open" if state else "closed")
elif device.hive_type == "motionsensor":
    print("Motion", "detected" if state else "clear")
```

---

## `online(device)` → `bool | None`

Returns `True` if the device is online, `False` if offline. Reads from `devices[device_id].props.online`.

```python
is_online = await hive.sensor.online(device)
```

---

## Full sensor example

```python
for device in device_list.get("binary_sensor", []):
    device = await hive.sensor.get_sensor(device)
    print(f"\n{device.ha_name} ({device.hive_type})")
    print(f"  State  : {device.status['state']}")
    print(f"  Online : {device.device_data.get('online')}")
    if device.attributes:
        print(f"  Battery: {device.attributes.get('battery_level')}%")
```

---

## Diagnostic sensors

Diagnostic sensors (`Availability`, `Connectivity`) appear in `device_list["sensor"]`. They monitor the online/offline status of physical devices:

```python
for device in device_list.get("sensor", []):
    device = await hive.sensor.get_sensor(device)
    print(f"{device.ha_name}: {device.status['state']}")
```

---

## Backwards-compatible aliases

| Alias | Current name |
|---|---|
| `getSensor(device)` | `get_sensor(device)` |

---

## Sync usage

```python
from pyhiveapi import Hive

hive = Hive(username="...", password="...")
device_list = hive.start_session(config)

for device in device_list.get("binary_sensor", []):
    device = hive.sensor.get_sensor(device)
    print(f"{device.ha_name}: state={device.status['state']}")
```
