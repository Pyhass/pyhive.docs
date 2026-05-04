---
title: Hub / Sense API
description: API reference for hive.hub (HiveHub) — smoke/CO, dog bark, and glass break detection from the Hive Hub and Hive Sense.
sidebar: api
---

# Hub / Sense — `hive.hub`

The `hive.hub` attribute is a `HiveHub` instance. It covers detection sensors built into the **Hive Hub** and **Hive Sense** devices (`hub`, `sense` types).

Hub sensor entities appear in `hive.device_list["binary_sensor"]` after `start_session()`.

---

## Available sensors

| Sensor | `device.hive_type` | Description |
|---|---|---|
| Smoke / CO | `"SMOKE_CO"` | Smoke and carbon monoxide alert |
| Dog bark | `"DOG_BARK"` | Dog bark detection |
| Glass break | `"GLASS_BREAK"` | Glass break detection |

Hub sensors are created automatically when a Hub or Sense device is discovered during `start_session()`.

---

## Quick reference

| Method | Returns | Description |
|---|---|---|
| `get_smoke_status(device)` | `int \| None` | Smoke/CO alert: `1` = active, `0` = clear |
| `get_dog_bark_status(device)` | `int \| None` | Dog bark: `1` = detected, `0` = clear |
| `get_glass_break_status(device)` | `int \| None` | Glass break: `1` = detected, `0` = clear |

---

## `get_smoke_status(device)` → `int | None`

Returns `1` if smoke or CO is detected, `0` if clear.

```python
status = await hive.hub.get_smoke_status(device)
if status == 1:
    print("WARNING: Smoke or CO detected!")
```

---

## `get_dog_bark_status(device)` → `int | None`

Returns `1` if a dog bark was detected, `0` if clear.

```python
status = await hive.hub.get_dog_bark_status(device)
```

---

## `get_glass_break_status(device)` → `int | None`

Returns `1` if a glass break was detected, `0` if clear.

```python
status = await hive.hub.get_glass_break_status(device)
```

---

## Hub sensor example

Hub sensors are routed through `hive.sensor.get_sensor()` for state updates — `hive.hub` methods are called internally. To read hub sensor state, use the sensor module:

```python
for device in device_list.get("binary_sensor", []):
    if device.hive_type in ("SMOKE_CO", "DOG_BARK", "GLASS_BREAK"):
        device = await hive.sensor.get_sensor(device)
        print(f"{device.ha_name}: {device.status['state']}")
```

Direct calls to hub methods are used when you have a specific hub device and want to query a detection type:

```python
# Find the hub Sense product device
for device in device_list.get("binary_sensor", []):
    if device.hive_type == "SMOKE_CO":
        active = await hive.hub.get_smoke_status(device)
        print(f"Smoke/CO: {active}")
```

---

## `const.py` mappings

Hub status values are mapped by `HIVETOHA["Hub"]` in `const.py`:

```python
HIVETOHA["Hub"] = {
    "Status": {True: 1, False: 0},
    "Smoke":  {True: 1, False: 0},
    "Dog":    {True: 1, False: 0},
    "Glass":  {True: 1, False: 0},
}
```

All hub sensors return `1` when active, `0` when clear.

---

## Sync usage

```python
from pyhiveapi import Hive

hive = Hive(username="...", password="...")
device_list = hive.start_session(config)

for device in device_list.get("binary_sensor", []):
    if device.hive_type == "SMOKE_CO":
        device = hive.sensor.get_sensor(device)
        print(f"Smoke/CO: {device.status['state']}")
```
