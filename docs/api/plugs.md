---
title: Smart Plugs API
description: API reference for hive.switch (Switch) — state, power usage, turn on/off for smart plugs, actions, and Heat on Demand entities.
sidebar: api
---

# Smart Plugs & Switches — `hive.switch`

The `hive.switch` attribute is a `Switch` instance. It covers **smart plugs**, **Heat on Demand** entities, and **Hive actions** — all of which appear as switch entities in `hive.device_list["switch"]`.

Devices are found in `hive.device_list["switch"]` after `start_session()`.

---

## Device types in `device_list["switch"]`

| `device.hive_type` | Description |
|---|---|
| `"activeplug"` | Hive smart plug |
| `"Heating_Heat_On_Demand"` | Heat on Demand toggle for a thermostat |
| `"action"` | Hive automation action (enable/disable) |

Check `device.hive_type` to distinguish between plug, action, and Heat on Demand entities.

---

## Quick reference

| Method | Returns | Description |
|---|---|---|
| `get_switch(device)` | `Device` | Refresh and return full switch state |
| `get_switch_state(device)` | `bool \| None` | Current state (routes through HoD/action if applicable) |
| `get_state(device)` | `bool \| None` | Smart plug state (`True` = on) |
| `get_power_usage(device)` | `float \| None` | Current power usage in watts (activeplug only) |
| `turn_on(device)` | `bool` | Turn on (routes through HoD if applicable) |
| `turn_off(device)` | `bool` | Turn off (routes through HoD if applicable) |
| `set_status_on(device)` | `bool` | Turn smart plug on directly |
| `set_status_off(device)` | `bool` | Turn smart plug off directly |

---

## `get_switch(device)`

Fetches the latest state and populates `device.status` and `device.device_data`.

```python
device = await hive.switch.get_switch(device)

device.status["state"]               # bool — True = on / enabled
device.status.get("power_usage")     # float watts (activeplug only)
device.device_data["online"]          # bool
```

Returns the cached device state if `should_use_cached_data()` is `True`.

---

## `get_switch_state(device)` → `bool | None`

Returns the current state. Automatically routes to the correct underlying method based on device type:

- `activeplug` → calls `get_state()`
- `Heating_Heat_On_Demand` → calls `session.heating.get_heat_on_demand()`

```python
state = await hive.switch.get_switch_state(device)
```

---

## `get_state(device)` → `bool | None`

Returns `True` if the smart plug is on, `False` if off. Use `get_switch_state()` when you do not know the device type.

```python
state = await hive.switch.get_state(device)
```

---

## `get_power_usage(device)` → `float | None`

Returns the current power consumption in watts for `activeplug` devices. Returns `None` if the device type does not support power monitoring.

```python
watts = await hive.switch.get_power_usage(device)
if watts is not None:
    print(f"Power usage: {watts} W")
```

---

## `turn_on(device)` → `bool`

Turns the device on. Automatically routes based on type:

- `activeplug` → `set_status_on()`
- `Heating_Heat_On_Demand` → `session.heating.set_heat_on_demand(device, "ENABLED")`

```python
await hive.switch.turn_on(device)
```

---

## `turn_off(device)` → `bool`

Turns the device off. Automatically routes based on type:

- `activeplug` → `set_status_off()`
- `Heating_Heat_On_Demand` → `session.heating.set_heat_on_demand(device, "DISABLED")`

```python
await hive.switch.turn_off(device)
```

---

## `set_status_on(device)` → `bool`

Turns the smart plug on directly (no routing for HoD/actions).

---

## `set_status_off(device)` → `bool`

Turns the smart plug off directly.

---

## Smart plug example

```python
for device in device_list.get("switch", []):
    if device.hive_type != "activeplug":
        continue

    device = await hive.switch.get_switch(device)
    print(f"{device.ha_name}: {'on' if device.status['state'] else 'off'}")
    print(f"  Power: {device.status.get('power_usage')} W")

    await hive.switch.turn_on(device)
    await hive.switch.turn_off(device)
```

---

## Heat on Demand example

```python
for device in device_list.get("switch", []):
    if device.hive_type == "Heating_Heat_On_Demand":
        device = await hive.switch.get_switch(device)
        enabled = device.status["state"]
        print(f"Heat on Demand: {'enabled' if enabled else 'disabled'}")

        await hive.switch.turn_on(device)   # enable HoD
        await hive.switch.turn_off(device)  # disable HoD
```

---

## Backwards-compatible aliases

| Alias | Current name |
|---|---|
| `turnOn(device)` | `turn_on(device)` |
| `turnOff(device)` | `turn_off(device)` |
| `getSwitch(device)` | `get_switch(device)` |

---

## Sync usage

```python
from pyhiveapi import Hive

hive = Hive(username="...", password="...")
device_list = hive.start_session(config)

for device in device_list.get("switch", []):
    if device.hive_type == "activeplug":
        device = hive.switch.get_switch(device)
        print(f"{device.ha_name}: {device.status.get('power_usage')} W")
        hive.switch.turn_on(device)
        hive.switch.turn_off(device)
```
