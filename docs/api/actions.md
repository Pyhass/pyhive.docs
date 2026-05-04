---
title: Actions API
description: API reference for hive.action (HiveAction) — enabling and disabling Hive automation actions.
sidebar: api
---

# Actions — `hive.action`

The `hive.action` attribute is a `HiveAction` instance. It manages **Hive automation actions** — rules you configure in the Hive app that trigger devices automatically (e.g. "turn on the hall light when motion is detected").

Action entities appear in `hive.device_list["switch"]` after `start_session()`, with `device.hive_type == "action"`.

---

## Quick reference

| Method | Returns | Description |
|---|---|---|
| `get_action(device)` | `Device \| "REMOVE"` | Refresh and return action state |
| `get_state(device)` | `bool \| None` | Whether the action is currently enabled |
| `set_status_on(device)` | `bool` | Enable the action |
| `set_status_off(device)` | `bool` | Disable the action |

---

## `get_action(device)`

Fetches the latest action state. Returns the updated `Device` on success, or the string `"REMOVE"` if the action no longer exists in the Hive API (it was deleted in the app).

```python
result = await hive.action.get_action(device)

if result == "REMOVE":
    # This action was deleted — remove the entity
    pass
else:
    device = result
    enabled = device.status["state"]   # bool
```

Returns the cached device state if `should_use_cached_data()` is `True`.

---

## `get_state(device)` → `bool | None`

Returns `True` if the action is enabled, `False` if disabled.

```python
enabled = await hive.action.get_state(device)
```

---

## `set_status_on(device)` → `bool`

Enables the action so it will fire when its trigger conditions are met.

```python
success = await hive.action.set_status_on(device)
```

Returns `True` on success, `False` on failure.

---

## `set_status_off(device)` → `bool`

Disables the action so it will not fire even when trigger conditions are met.

```python
success = await hive.action.set_status_off(device)
```

---

## Full actions example

```python
# Actions appear in device_list["switch"] with hive_type == "action"
for device in device_list.get("switch", []):
    if device.hive_type != "action":
        continue

    result = await hive.action.get_action(device)
    if result == "REMOVE":
        print(f"Action {device.ha_name} was deleted")
        continue

    device = result
    print(f"Action: {device.ha_name} — {'enabled' if device.status['state'] else 'disabled'}")

    # Toggle: enable if disabled, disable if enabled
    if device.status["state"]:
        await hive.action.set_status_off(device)
    else:
        await hive.action.set_status_on(device)
```

---

## How actions work internally

Actions are stored in `session.data.actions` (a dict keyed by action ID). When `set_status_on()` or `set_status_off()` is called, the library:

1. Copies the full action data from `session.data.actions`
2. Updates the `enabled` field
3. Calls `session.api.set_action(action_id, json.dumps(data))` to write back the full object
4. Refreshes device data on success

This means the full action configuration is preserved — only the `enabled` flag changes.

---

## Backwards-compatible aliases

| Alias | Current name |
|---|---|
| `getAction(device)` | `get_action(device)` |
| `setStatusOn(device)` | `set_status_on(device)` |
| `setStatusOff(device)` | `set_status_off(device)` |

---

## Sync usage

```python
from pyhiveapi import Hive

hive = Hive(username="...", password="...")
device_list = hive.start_session(config)

for device in device_list.get("switch", []):
    if device.hive_type == "action":
        result = hive.action.get_action(device)
        if result != "REMOVE":
            device = result
            print(f"{device.ha_name}: {device.status['state']}")
            hive.action.set_status_on(device)
```
