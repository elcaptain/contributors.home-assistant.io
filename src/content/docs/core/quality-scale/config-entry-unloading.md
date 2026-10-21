---
title: Support unloading the config entry
description: "Silver rule config-entry-unloading: the integration can be unloaded and reloaded without a restart."
sidebarLabel: config-entry-unloading
sidebarGroup: Silver
order: 2
---

This is a Silver rule. In `quality_scale.yaml`, it is called `config-entry-unloading`.

The integration must implement `async_unload_entry` in its `__init__.py`. When a [config entry](/glossary/config-entry/) is unloaded, the integration unloads its platforms and cleans up everything it set up: listeners, subscriptions, background tasks, and open connections.

## Why it matters

Unloading is what makes it possible to reload, disable, or remove an integration while Home Assistant keeps running. Without it, the user has to restart Home Assistant to apply a change or get rid of the integration. And whatever is not cleaned up keeps running in the background, using memory and connections for nothing.

## Example

Unload the platforms, and clean up when that succeeds:

```python
async def async_unload_entry(
    hass: HomeAssistant, entry: MyConfigEntry
) -> None:
    """Unload a config entry."""
    await hass.config_entries.async_unload_platforms_or_raise(
        entry, PLATFORMS
    )
    await entry.runtime_data.client.disconnect()
```

`async_unload_platforms_or_raise` raises `ConfigEntryError` when a platform fails to unload, so the cleanup after it only runs when unloading succeeded.

Often, you do not need to keep track of the cleanup yourself. Register it during setup with `entry.async_on_unload`, and Home Assistant calls it for you:

```python
async def async_setup_entry(
    hass: HomeAssistant, entry: MyConfigEntry
) -> None:
    """Set up My integration from a config entry."""
    client = MyClient(entry.data[CONF_HOST])
    entry.async_on_unload(client.subscribe(_handle_event))
    ...
```

The callbacks registered with `entry.async_on_unload` run when `async_unload_entry` succeeds, and also whenever `async_setup_entry` fails: when it raises, like `ConfigEntryNotReady`, `ConfigEntryAuthFailed`, or `ConfigEntryError`. You still need `async_unload_entry` itself: without it, the entry cannot be unloaded at all.

## How it is checked

- Pylint, when the rule is `done`: `__init__.py` must define `async_unload_entry` (`home-assistant-missing-config-entry-unloading`).
- Code that tests never run shows up in the [test coverage](/core/quality-scale/test-coverage/), so a test that unloads the entry is usually needed anyway.
- Whether everything is cleaned up is checked in review.

## Exceptions

There are no exceptions to this rule.

## Related rules

- [entity-event-setup](/core/quality-scale/entity-event-setup/): subscribe to events in the entity lifecycle.
