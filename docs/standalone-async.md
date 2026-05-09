---
title: Standalone — Async
description: How to use pyhive-integration as a standalone async Python library without Home Assistant.
sidebar: main
---

# Standalone — Async

Use `apyhiveapi` when your application is already built on `asyncio` (or any asyncio-compatible framework like FastAPI, aiohttp, etc.).

---

## Full example

The following example covers the complete flow: first-time login, starting a session, reading state for all device types, and controlling devices.

```python
import asyncio
from apyhiveapi import Auth, Hive, SMS_REQUIRED
from apyhiveapi.helper.hive_exceptions import HiveReauthRequired, HiveInvalidUsername


async def main():
    # ──────────────────────────────────────────────
    # 1.  Authentication (first-time login)
    # ──────────────────────────────────────────────
    auth = Auth(username="you@example.com", password="yourpassword")
    result = await auth.login()

    if result.get("ChallengeName") == SMS_REQUIRED:
        code = input("SMS code: ")
        result = await auth.sms_2fa(code, result)
        device_data = await auth.get_device_data()
        # Persist these for future runs (see step 3 below)
        print("Device credentials:", device_data)

    tokens = {
        "token":        result["AuthenticationResult"]["IdToken"],
        "refreshToken": result["AuthenticationResult"]["RefreshToken"],
        "accessToken":  result["AuthenticationResult"]["AccessToken"],
    }

    # ──────────────────────────────────────────────
    # 2.  Start session
    # ──────────────────────────────────────────────
    hive = Hive(username="you@example.com", password="yourpassword")
    device_list = await hive.start_session({
        "tokens": tokens,
        "username": "you@example.com",
        "password": "yourpassword",
        # "device_data": [group_key, key, password],  # include for subsequent logins
    })

    # ──────────────────────────────────────────────
    # 3.  Heating
    # ──────────────────────────────────────────────
    for device in device_list.get("climate", []):
        device = await hive.heating.get_climate(device)
        print(f"\n=== Heating: {device.ha_name} ===")
        print(f"  Current temp : {device.status['current_temperature']}°")
        print(f"  Target temp  : {device.status['target_temperature']}°")
        print(f"  Mode         : {device.status['mode']}")
        print(f"  Boost        : {device.status['boost']}")

        # Set target temperature
        await hive.heating.set_target_temperature(device, 21.0)

        # Change mode
        await hive.heating.set_mode(device, "SCHEDULE")  # SCHEDULE | MANUAL | OFF

        # Boost for 30 minutes at 22°C
        await hive.heating.set_boost_on(device, mins=30, temp=22.0)

        # Turn boost off
        await hive.heating.set_boost_off(device)

    # ──────────────────────────────────────────────
    # 4.  Hot water
    # ──────────────────────────────────────────────
    for device in device_list.get("water_heater", []):
        device = await hive.hotwater.get_water_heater(device)
        print(f"\n=== Hot Water: {device.ha_name} ===")
        print(f"  Mode  : {device.status['current_operation']}")

        schedule = await hive.hotwater.get_schedule_now_next_later(device)
        print(f"  Schedule now: {schedule}")

        # Boost for 60 minutes
        await hive.hotwater.set_boost_on(device, mins=60)
        await hive.hotwater.set_boost_off(device)

        # Set mode
        await hive.hotwater.set_mode(device, "ON")   # ON | OFF | SCHEDULE

    # ──────────────────────────────────────────────
    # 5.  Lights
    # ──────────────────────────────────────────────
    for device in device_list.get("light", []):
        device = await hive.light.get_light(device)
        print(f"\n=== Light: {device.ha_name} ({device.hive_type}) ===")
        print(f"  State      : {'On' if device.status['state'] else 'Off'}")
        print(f"  Brightness : {device.status.get('brightness')}")
        print(f"  Color temp : {device.status.get('color_temp')} mireds")
        print(f"  Color mode : {device.status.get('mode')}")

        # Turn on
        await hive.light.turn_on(device, brightness=None, color_temp=None, color=None)

        # Set brightness (0-255)
        await hive.light.turn_on(device, brightness=128, color_temp=None, color=None)

        # Set colour temperature in mireds (tuneablelight / colourtuneablelight)
        await hive.light.turn_on(device, brightness=None, color_temp=250, color=None)

        # Set RGB colour (colourtuneablelight)
        # color is a list: [hue 0-360, saturation 0-100, value 0-100]
        await hive.light.turn_on(device, brightness=None, color_temp=None, color=[240, 80, 100])

        # Turn off
        await hive.light.turn_off(device)

    # ──────────────────────────────────────────────
    # 6.  Smart plugs
    # ──────────────────────────────────────────────
    for device in device_list.get("switch", []):
        if device.hive_type == "activeplug":
            device = await hive.switch.get_switch(device)
            print(f"\n=== Plug: {device.ha_name} ===")
            print(f"  State       : {'On' if device.status['state'] else 'Off'}")
            print(f"  Power usage : {device.status.get('power_usage')} W")

            await hive.switch.turn_on(device)
            await hive.switch.turn_off(device)

    # ──────────────────────────────────────────────
    # 7.  Sensors
    # ──────────────────────────────────────────────
    for device in device_list.get("binary_sensor", []):
        device = await hive.sensor.get_sensor(device)
        print(f"\n=== Sensor: {device.ha_name} ({device.hive_type}) ===")
        print(f"  State  : {device.status.get('state')}")

    # ──────────────────────────────────────────────
    # 8.  Actions
    # ──────────────────────────────────────────────
    for device in device_list.get("switch", []):
        if device.hive_type == "action":
            device = await hive.action.get_action(device)
            if device == "REMOVE":
                continue  # action was deleted from Hive app
            print(f"\n=== Action: {device.ha_name} ===")
            print(f"  Enabled : {device.status['state']}")

            # Enable / disable the action
            await hive.action.set_status_on(device)
            await hive.action.set_status_off(device)

    # ──────────────────────────────────────────────
    # 9.  Force an immediate poll (optional)
    # ──────────────────────────────────────────────
    await hive.force_update()


asyncio.run(main())
```

---

## Subsequent logins (device credentials saved)

Once you have saved `device_data` from the first login, pass it to `start_session` to avoid SMS on every run:

```python
async def subsequent_login(tokens, device_group_key, device_key, device_password):
    hive = Hive(username="you@example.com", password="yourpassword")
    return await hive.start_session({
        "tokens": tokens,
        "username": "you@example.com",
        "password": "yourpassword",
        "device_data": [device_group_key, device_key, device_password],
    })
```

---

## Error handling

```python
from apyhiveapi.helper.hive_exceptions import (
    HiveReauthRequired,
    HiveApiError,
    HiveInvalidUsername,
    HiveInvalidPassword,
)

try:
    await hive.force_update()
except HiveReauthRequired:
    # Tokens expired and all refresh attempts failed.
    # Re-run your login flow and create a new session.
    print("Reauthentication required — please log in again.")
except HiveApiError as e:
    print(f"Hive API error: {e}")
```

---

## Polling in a long-running process

For scripts that run continuously, call `update_data(device)` periodically. It respects the 120-second scan interval automatically:

```python
import asyncio
from apyhiveapi import Hive

async def monitor(hive, device):
    while True:
        await hive.update_data(device)
        device = await hive.heating.get_climate(device)
        print(f"Current temp: {device.status['current_temperature']}°")
        await asyncio.sleep(30)  # check every 30s; update_data handles rate-limiting
```
