---
title: Lights API
description: API reference for hive.light (Light) — state, brightness, colour temperature, RGB colour, and turn on/off methods.
sidebar: api
---

# Lights — `hive.light`

The `hive.light` attribute is a `Light` instance. It covers all Hive light types.

Devices are found in `hive.device_list["light"]` after `start_session()`.

---

## Light types

| `device.hive_type` | Supports |
|---|---|
| `warmwhitelight` | On/off, brightness |
| `tuneablelight` | On/off, brightness, colour temperature |
| `colourtuneablelight` | On/off, brightness, colour temperature, RGB colour |

Check `device.hive_type` to know which methods are applicable before calling colour-specific methods.

---

## Quick reference

| Method | Returns | Description |
|---|---|---|
| `get_light(device)` | `Device` | Refresh and return full light state |
| `get_state(device)` | `bool \| None` | `True` = on, `False` = off |
| `get_brightness(device)` | `int \| None` | Brightness 0–255 |
| `get_min_color_temp(device)` | `int \| None` | Minimum colour temperature (mireds) |
| `get_max_color_temp(device)` | `int \| None` | Maximum colour temperature (mireds) |
| `get_color_temp(device)` | `int \| None` | Current colour temperature (mireds) |
| `get_color(device)` | `tuple \| None` | Current colour as RGB tuple |
| `get_color_mode(device)` | `str \| None` | `"COLOUR"` or `"WHITE"` |
| `turn_on(device, brightness, color_temp, color)` | `bool` | Turn on (with optional parameters) |
| `turn_off(device)` | `bool` | Turn off |
| `set_status_on(device)` | `bool` | Turn on (no parameters) |
| `set_status_off(device)` | `bool` | Turn off |
| `set_brightness(device, n_brightness)` | `bool` | Set brightness |
| `set_color_temp(device, color_temp)` | `bool` | Set colour temperature |
| `set_color(device, new_color)` | `bool` | Set colour (HSV) |

---

## `get_light(device)`

Fetches the latest state and populates `device.status`, `device.device_data`, and `device.attributes`.

```python
device = await hive.light.get_light(device)

device.status["state"]           # bool — True = on
device.status["brightness"]      # int 0–255
device.status.get("color_temp")  # int mireds (tuneablelight / colourtuneablelight)
device.status.get("mode")        # "COLOUR" | "WHITE" (colourtuneablelight)
device.status.get("hs_color")    # tuple (h, s) — present when mode == "COLOUR"
```

Returns the cached device state if `should_use_cached_data()` is `True`.

---

## `get_state(device)` → `bool | None`

Returns `True` if the light is on, `False` if off.

```python
on = await hive.light.get_state(device)
```

---

## `get_brightness(device)` → `int | None`

Returns the current brightness as a value from 0 to 255 (converted from the Hive API's 0–100 scale).

```python
brightness = await hive.light.get_brightness(device)
```

---

## `get_min_color_temp(device)` → `int | None`

Returns the minimum supported colour temperature in **mireds**.

> **Note:** The Hive API stores colour temperature as Kelvin. The library converts automatically: `mireds = round(1_000_000 / kelvin)`.

```python
min_ct = await hive.light.get_min_color_temp(device)
```

---

## `get_max_color_temp(device)` → `int | None`

Returns the maximum supported colour temperature in **mireds**.

```python
max_ct = await hive.light.get_max_color_temp(device)
```

---

## `get_color_temp(device)` → `int | None`

Returns the current colour temperature in **mireds**.

```python
ct = await hive.light.get_color_temp(device)
```

---

## `get_color(device)` → `tuple | None`

Returns the current colour as an RGB tuple `(r, g, b)` where each value is 0–255. The Hive API stores colour as HSV; the library converts automatically.

```python
rgb = await hive.light.get_color(device)
if rgb:
    r, g, b = rgb
```

---

## `get_color_mode(device)` → `str | None`

Returns the current colour mode for `colourtuneablelight` devices.

| Value | Meaning |
|---|---|
| `"COLOUR"` | RGB colour mode |
| `"WHITE"` | White/colour temperature mode |

```python
mode = await hive.light.get_color_mode(device)
```

---

## `turn_on(device, brightness, color_temp, color)` → `bool`

Turns the light on. Pass parameters to simultaneously set a property; pass `None` for parameters you do not want to change.

```python
# Turn on (no change to brightness/colour)
await hive.light.turn_on(device, brightness=None, color_temp=None, color=None)

# Turn on at 50% brightness (128/255)
await hive.light.turn_on(device, brightness=128, color_temp=None, color=None)

# Turn on at 250 mireds colour temperature
await hive.light.turn_on(device, brightness=None, color_temp=250, color=None)

# Turn on in colour mode — color is [hue 0-360, saturation 0-100, value 0-100]
await hive.light.turn_on(device, brightness=None, color_temp=None, color=[240, 80, 100])
```

| Parameter | Type | Description |
|---|---|---|
| `device` | `Device` | Light device |
| `brightness` | `int \| None` | Brightness 0–255, or `None` to leave unchanged |
| `color_temp` | `int \| None` | Colour temperature in mireds, or `None` |
| `color` | `list \| None` | `[hue, saturation, value]` (HSV), or `None` |

**Priority:** If `brightness` is set, only brightness is changed. If `color_temp` is set (and `brightness` is `None`), only colour temperature is changed. If `color` is set (and both others are `None`), only colour is changed. If all are `None`, only the on/off state is set.

---

## `turn_off(device)` → `bool`

Turns the light off.

```python
await hive.light.turn_off(device)
```

---

## `set_status_on(device)` → `bool`

Turns the light on without changing any other properties. Equivalent to `turn_on(device, None, None, None)`.

---

## `set_status_off(device)` → `bool`

Turns the light off. Equivalent to `turn_off(device)`.

---

## `set_brightness(device, n_brightness)` → `bool`

Sets the brightness. Also turns the light on if it was off.

```python
await hive.light.set_brightness(device, 200)
```

| Parameter | Type | Description |
|---|---|---|
| `n_brightness` | `int` | Brightness value 0–255 |

---

## `set_color_temp(device, color_temp)` → `bool`

Sets the colour temperature in mireds. For `colourtuneablelight` devices, also sets `colourMode` to `"WHITE"`.

```python
await hive.light.set_color_temp(device, 370)  # warm white
await hive.light.set_color_temp(device, 154)  # cool white
```

---

## `set_color(device, new_color)` → `bool`

Sets the colour in HSV format. Only applicable to `colourtuneablelight` devices. Sets `colourMode` to `"COLOUR"` automatically.

```python
# color = [hue 0-360, saturation 0-100, value 0-100]
await hive.light.set_color(device, [0, 100, 100])    # red
await hive.light.set_color(device, [120, 100, 100])  # green
await hive.light.set_color(device, [240, 100, 100])  # blue
```

---

## Backwards-compatible aliases

| Alias | Current name |
|---|---|
| `turnOn(device, brightness, color_temp, color)` | `turn_on(device, brightness, color_temp, color)` |
| `turnOff(device)` | `turn_off(device)` |
| `getLight(device)` | `get_light(device)` |

---

## Sync usage

```python
from pyhiveapi import Hive

hive = Hive(username="...", password="...")
device_list = hive.start_session(config)

for device in device_list.get("light", []):
    device = hive.light.get_light(device)
    print(f"{device.ha_name}: {'on' if device.status['state'] else 'off'}")
    hive.light.turn_on(device, brightness=200, color_temp=None, color=None)
    hive.light.turn_off(device)
```
