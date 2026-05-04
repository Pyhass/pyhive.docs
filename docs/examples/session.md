---
title: Full Session Examples
description: Complete worked examples for using a pyhive-integration session with all device types — async and sync.
sidebar: main
---

# Full Session Examples

---

## Full async session — all device types

```python
import asyncio
from apyhiveapi import Auth, Hive, SMS_REQUIRED


async def main():
    # ── Authentication ────────────────────────────────────
    auth = Auth(username="you@example.com", password="yourpassword")
    result = await auth.login()

    if result.get("ChallengeName") == SMS_REQUIRED:
        code = input("SMS code: ")
        result = await auth.sms_2fa(code, result)
        device_data = auth.get_device_data()
        # Persist device_data for future runs
    else:
        device_data = None  # Load from storage if available

    # ── Session ───────────────────────────────────────────
    hive = Hive(username="you@example.com", password="yourpassword")
    config = {
        "tokens": result,
        "username": "you@example.com",
        "password": "yourpassword",
    }
    if device_data:
        config["device_data"] = device_data

    device_list = await hive.start_session(config)

    # ── Heating ───────────────────────────────────────────
    print("\n── Heating ──")
    for device in device_list.get("climate", []):
        device = await hive.heating.get_climate(device)
        s = device.status
        print(f"  {device.ha_name}:")
        print(f"    Current temp  : {s['current_temperature']}°")
        print(f"    Target temp   : {s['target_temperature']}°")
        print(f"    Mode          : {s['mode']}")
        print(f"    Boost         : {s['boost']}")
        print(f"    Min/max       : {device.min_temp}° – {device.max_temp}°")

        schedule = await hive.heating.get_schedule_now_next_later(device)
        if schedule:
            print(f"    Schedule now  : {schedule['now']}")

    # ── Hot water ─────────────────────────────────────────
    print("\n── Hot Water ──")
    for device in device_list.get("water_heater", []):
        device = await hive.hotwater.get_water_heater(device)
        s = device.status
        print(f"  {device.ha_name}:")
        print(f"    Mode   : {s['current_operation']}")

        boost = await hive.hotwater.get_boost(device)
        bt    = await hive.hotwater.get_boost_time(device)
        print(f"    Boost  : {boost}" + (f" ({bt} min remaining)" if bt else ""))

    # ── Lights ────────────────────────────────────────────
    print("\n── Lights ──")
    for device in device_list.get("light", []):
        device = await hive.light.get_light(device)
        s = device.status
        print(f"  {device.ha_name} ({device.hive_type}):")
        print(f"    State      : {'on' if s['state'] else 'off'}")
        print(f"    Brightness : {s.get('brightness')}")
        if "color_temp" in s:
            print(f"    Color temp : {s['color_temp']} mireds")
        if "mode" in s:
            print(f"    Color mode : {s['mode']}")
        if s.get("hs_color"):
            print(f"    Color      : {s['hs_color']}")

    # ── Smart plugs ───────────────────────────────────────
    print("\n── Smart Plugs ──")
    for device in device_list.get("switch", []):
        if device.hive_type != "activeplug":
            continue
        device = await hive.switch.get_switch(device)
        s = device.status
        print(f"  {device.ha_name}:")
        print(f"    State : {'on' if s['state'] else 'off'}")
        print(f"    Power : {s.get('power_usage')} W")

    # ── Sensors ───────────────────────────────────────────
    print("\n── Sensors ──")
    for device in device_list.get("binary_sensor", []):
        device = await hive.sensor.get_sensor(device)
        s = device.status
        type_label = device.hive_type
        print(f"  {device.ha_name} ({type_label}): {s.get('state')}")

    for device in device_list.get("sensor", []):
        device = await hive.sensor.get_sensor(device)
        print(f"  {device.ha_name} (diagnostic): {device.status.get('state')}")

    # ── Actions ───────────────────────────────────────────
    print("\n── Actions ──")
    for device in device_list.get("switch", []):
        if device.hive_type != "action":
            continue
        result = await hive.action.get_action(device)
        if result == "REMOVE":
            continue
        device = result
        print(f"  {device.ha_name}: {'enabled' if device.status['state'] else 'disabled'}")


asyncio.run(main())
```

---

## Full sync session — all device types

```python
from pyhiveapi import Auth, Hive, SMS_REQUIRED


# ── Authentication ────────────────────────────────────────
auth = Auth(username="you@example.com", password="yourpassword")
result = auth.login()

if result.get("ChallengeName") == SMS_REQUIRED:
    code = input("SMS code: ")
    result = auth.sms_2fa(code, result)
    device_data = auth.get_device_data()
else:
    device_data = None

# ── Session ───────────────────────────────────────────────
hive = Hive(username="you@example.com", password="yourpassword")
config = {"tokens": result, "username": "you@example.com", "password": "yourpassword"}
if device_data:
    config["device_data"] = device_data

device_list = hive.start_session(config)

# ── Heating ───────────────────────────────────────────────
for device in device_list.get("climate", []):
    device = hive.heating.get_climate(device)
    print(f"{device.ha_name}: {device.status['current_temperature']}° / target {device.status['target_temperature']}°")
    hive.heating.set_target_temperature(device, 21.0)
    hive.heating.set_mode(device, "SCHEDULE")

# ── Hot water ─────────────────────────────────────────────
for device in device_list.get("water_heater", []):
    device = hive.hotwater.get_water_heater(device)
    print(f"{device.ha_name}: {device.status['current_operation']}")
    hive.hotwater.set_mode(device, "SCHEDULE")

# ── Lights ────────────────────────────────────────────────
for device in device_list.get("light", []):
    device = hive.light.get_light(device)
    print(f"{device.ha_name}: {'on' if device.status['state'] else 'off'}")
    hive.light.turn_on(device, brightness=200, color_temp=None, color=None)
    hive.light.turn_off(device)

# ── Smart plugs ───────────────────────────────────────────
for device in device_list.get("switch", []):
    if device.hive_type == "activeplug":
        device = hive.switch.get_switch(device)
        print(f"{device.ha_name}: {device.status.get('power_usage')} W")
        hive.switch.turn_on(device)
        hive.switch.turn_off(device)

# ── Sensors ───────────────────────────────────────────────
for device in device_list.get("binary_sensor", []):
    device = hive.sensor.get_sensor(device)
    print(f"{device.ha_name}: {device.status.get('state')}")

# ── Actions ───────────────────────────────────────────────
for device in device_list.get("switch", []):
    if device.hive_type == "action":
        result = hive.action.get_action(device)
        if result != "REMOVE":
            device = result
            print(f"Action {device.ha_name}: {device.status['state']}")
```

---

## Periodic polling loop (async)

```python
import asyncio
from apyhiveapi import Hive

async def polling_loop(hive, device_list):
    while True:
        for device in device_list.get("climate", []):
            await hive.update_data(device)
            device = await hive.heating.get_climate(device)
            print(f"[{device.ha_name}] {device.status['current_temperature']}°")
        await asyncio.sleep(30)
```

---

## Controlling devices after session start

```python
# Get the first heating zone
heating_device = device_list["climate"][0]

# Read latest state
heating_device = await hive.heating.get_climate(heating_device)

# Set temperature
await hive.heating.set_target_temperature(heating_device, 21.5)

# Set mode
await hive.heating.set_mode(heating_device, "MANUAL")

# Boost for 30 minutes at 22°C
await hive.heating.set_boost_on(heating_device, mins=30, temp=22.0)

# Cancel boost
await hive.heating.set_boost_off(heating_device)

# Hot water
hw_device = device_list["water_heater"][0]
await hive.hotwater.set_boost_on(hw_device, mins=60)
await hive.hotwater.set_boost_off(hw_device)

# Light
light_device = device_list["light"][0]
await hive.light.turn_on(light_device, brightness=200, color_temp=None, color=None)
await hive.light.turn_off(light_device)

# Smart plug
plug_device = next(d for d in device_list["switch"] if d.hive_type == "activeplug")
await hive.switch.turn_on(plug_device)
await hive.switch.turn_off(plug_device)
```
