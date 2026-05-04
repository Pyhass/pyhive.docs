---
title: pyhive-integration
description: Python library for interfacing with the Hive smart home platform — async and sync, standalone or with Home Assistant.
sidebar: main
---

# pyhive-integration

**pyhive-integration** is a Python library for interfacing with the [Hive](https://www.hivehome.com/) smart home platform. It is the canonical Python client for the Hive API from version 2.0.0 onwards.

The library ships two packages from the same source:

| Import | Style | Use case |
|---|---|---|
| `from apyhiveapi import Hive, Auth` | Async (`async`/`await`) | Home Assistant, asyncio applications |
| `from pyhiveapi import Hive, Auth` | Synchronous | Scripts, tools, non-async contexts |

Both packages expose an identical API. The sync package is auto-generated from the async source at build time using [unasync](https://github.com/python-trio/unasync) — do not mix imports from both packages in the same program.

---

## Supported Devices

| Device | HA entity type | Hive types | Capabilities |
|---|---|---|---|
| **Heating** (thermostat, TRV) | `climate` | `heating`, `trvcontrol` | Current/target temperature, mode (SCHEDULE / MANUAL / OFF), boost, heat-on-demand, schedule |
| **Hot Water** | `water_heater` | `hotwater` | Mode (SCHEDULE / ON / OFF), boost, state |
| **Lights** | `light` | `warmwhitelight`, `tuneablelight`, `colourtuneablelight` | On/off, brightness, colour temperature, RGB colour |
| **Smart Plugs** | `switch` | `activeplug` | On/off, power usage (watts) |
| **Motion Sensor** | `binary_sensor` | `motionsensor` | Motion detected / clear |
| **Contact Sensor** | `binary_sensor` | `contactsensor` | Open / closed |
| **Hub / Sense** | `binary_sensor` | `hub`, `sense` | Smoke/CO, dog bark, glass break detection |
| **Actions** | `switch` | `action` | Enable / disable Hive automations |

---

## Requirements

- Python 3.10 or later
- Hive account (account owner — guest accounts are not supported)

---

## Quick links

- [Getting Started](getting-started) — installation and first session
- [Authentication](authentication) — login, 2FA, device registration
- [Home Assistant Integration](home-assistant) — entity creation and update patterns
- [Standalone Async](standalone-async) — full async usage guide
- [Standalone Sync](standalone-sync) — full sync usage guide
- [API Reference](api/heating) — per-device method reference
- [Exceptions](exceptions) — error handling reference
- [Migration Guide](migration) — upgrading from pre-2.0.0

---

## Installation

```bash
pip install pyhive-integration
```

---

## Architecture overview

```
pyhive-integration (PyPI)
├── apyhiveapi/          ← async package (source of truth, in src/)
│   ├── Hive             ← public entry point; inherits HiveSession
│   ├── Auth             ← AWS Cognito SRP authentication
│   ├── API              ← async HTTP client (HiveApiAsync)
│   └── device modules   ← heating, hotwater, light, switch, sensor, hub, action
└── pyhiveapi/           ← sync package (auto-generated from apyhiveapi via unasync)
    └── (identical API, no await keywords)
```

The `Hive` class is the single entry point. After `start_session()`, device modules are available as attributes (`hive.heating`, `hive.light`, etc.) and the full device list is in `hive.device_list`.
