---
title: Store runtime data in runtime_data
description: "Bronze rule runtime-data: runtime data is stored in ConfigEntry.runtime_data, with a typed config entry."
sidebarLabel: runtime-data
sidebarGroup: Bronze
order: 17
---

This is a Bronze rule. In `quality_scale.yaml`, it is called `runtime-data`.

Data that the integration needs while a [config entry](/glossary/config-entry/) is loaded, like the API client or the [coordinator](/glossary/coordinator/), must be stored in `entry.runtime_data`. Not in `hass.data`. Give the config entry a type that says what is in `runtime_data`. When the integration also follows [strict-typing](/core/quality-scale/strict-typing/), that typed config entry is required, and you use it everywhere you handle the entry.

## Why it matters

`runtime_data` belongs to one config entry, and it is gone when the entry is unloaded. There is no shared dictionary to keep clean, and no keys that collide with other integrations. Because the config entry is typed, your editor and mypy know exactly what `entry.runtime_data` is, and catch mistakes before your users do. And every integration stores its data the same way, which makes the code easier to read for the next person.

## Example

Define the typed config entry, and assign `runtime_data` in `async_setup_entry`:

```python
type MyConfigEntry = ConfigEntry[MyClient]


async def async_setup_entry(hass: HomeAssistant, entry: MyConfigEntry) -> None:
    """Set up My integration from a config entry."""
    client = MyClient(entry.data[CONF_HOST])
    entry.runtime_data = client

    await hass.config_entries.async_forward_entry_setups(entry, PLATFORMS)
```

A platform reads it back, fully typed:

```python
async def async_setup_entry(
    hass: HomeAssistant,
    entry: MyConfigEntry,
    async_add_entities: AddConfigEntryEntitiesCallback,
) -> None:
    """Set up the sensors."""
    client = entry.runtime_data
```

A coordinator that gets the config entry should use the typed entry too:

```python
class MyCoordinator(DataUpdateCoordinator[MyData]):
    """Fetch data from the device."""

    config_entry: MyConfigEntry
```

When more than one object needs to live in `runtime_data`, put them in a dataclass and use that as the type.

## How it is checked

- Hassfest, when the rule is `done`: `async_setup_entry` in `__init__.py` must assign `entry.runtime_data`.
- Hassfest, when [strict-typing](/core/quality-scale/strict-typing/) is also `done`: the config entry must be typed, with a type alias whose name ends in `ConfigEntry`. This is checked for `async_setup_entry`, `async_unload_entry`, `async_migrate_entry`, `async_remove_entry`, and `async_remove_config_entry_device` in `__init__.py`, for `async_setup_entry` of every platform, for the diagnostics functions, and for `async_get_options_flow`.
- Pylint, for every integration with a config flow: `home-assistant-use-runtime-data` warns when the integration stores data in `hass.data[DOMAIN]`.
- Pylint: `home-assistant-coordinator-untyped-config-entry` warns when a coordinator does not use the typed config entry of its integration.

## Exceptions

If the integration does not have any runtime data to store, this rule does not apply.

## Related rules

- [strict-typing](/core/quality-scale/strict-typing/): use strict typing.
- [test-before-setup](/core/quality-scale/test-before-setup/): check the connection during setup.
