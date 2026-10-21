---
title: Entity registry
description: Look up the entities of your integration in the entity registry, and migrate them when their unique IDs change.
sidebarGroup: Devices and registries
order: 2
---

This page is about working with the entity registry from the code of your integration: looking up entries, and migrating them. How entities get into the registry, and what decides whether they start enabled, visible, or categorized, is on [the entity registry](/core/entities/registry/) in entities and platforms.

The entity registry remembers every entity that has a [unique ID](/glossary/unique-id/): its entity ID, the name and icon the user gave it, its area, whether it is disabled, and which config entry and device it belongs to. Your entities end up in it by themselves, as long as they have a unique ID.

## Get the registry

```python
from homeassistant.helpers import entity_registry as er

entity_registry = er.async_get(hass)
```

## Look up entities

```python
# The entity ID for a unique ID of your integration, or None
entity_id = entity_registry.async_get_entity_id(
    "sensor", DOMAIN, f"{serial_number}_temperature"
)

# The registry entry for an entity ID
entry = entity_registry.async_get(entity_id)

# All entities of a config entry
entries = er.async_entries_for_config_entry(
    entity_registry, config_entry.entry_id
)

# All entities of a device
entries = er.async_entries_for_device(entity_registry, device_id)
```

`async_get_entity_id` takes the entity domain (like `sensor`), the platform (your integration domain), and the unique ID. Use the entity ID it returns: never build an entity ID yourself, users can change them.

## Migrate unique IDs

Unique IDs must not change. When they have to anyway, for example because the old ones were not unique enough, migrate the registry entries, so users keep their entity IDs, names, and automations. Do it in `async_migrate_entry`, together with a [config entry version](/core/integrations/config-entries/#migrate-a-config-entry) bump:

```python
from typing import Any

from homeassistant.core import callback
from homeassistant.helpers import entity_registry as er


async def async_migrate_entry(
    hass: HomeAssistant, entry: ExampleConfigEntry
) -> None:
    """Migrate an old config entry."""
    if entry.version == 1:

        @callback
        def _migrate_unique_id(
            entity_entry: er.RegistryEntry,
        ) -> dict[str, Any] | None:
            """Prefix unique IDs with the serial number."""
            if entity_entry.unique_id.startswith(entry.unique_id):
                return None

            return {
                "new_unique_id": f"{entry.unique_id}_{entity_entry.unique_id}"
            }

        await er.async_migrate_entries(
            hass, entry.entry_id, _migrate_unique_id
        )

        hass.config_entries.async_update_entry(
            entry, version=2, minor_version=1
        )
```

This is a major version bump, so set `VERSION = 2` on your config flow. Older code that does not know the new unique IDs would not find its entities after a downgrade, and create them again. The example builds on the unique ID of the config entry, so it assumes every entry has one.

`async_migrate_entries` calls your function for every registry entry of the config entry. Return a dictionary of changes for `async_update_entity`, or `None` to leave the entry alone.

## Update or remove entries

`entity_registry.async_update_entity(entity_id, ...)` changes a registry entry, and `entity_registry.async_remove(entity_id)` removes it. Integrations rarely need them. Use them for cleanup, like removing entities of a feature your integration no longer offers. Settings that belong to the user, like names, areas, and whether an entity is disabled or hidden, are not yours to change.
