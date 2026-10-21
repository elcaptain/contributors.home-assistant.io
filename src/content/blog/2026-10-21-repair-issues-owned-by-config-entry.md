---
title: Repair issues can belong to a config entry
description: Link a repair issue to its config entry, and Home Assistant deletes it when the entry is removed.
date: 2026-10-21
authors:
  - Erik Montnémery
areas:
  - core
---

You can now link a repair issue to the config entry it belongs to, by passing the new `config_entry_id` argument to `async_create_issue` or `create_issue`. Home Assistant deletes the issue when the config entry is removed, just like it does with the devices and entities of the entry.

```python
from homeassistant.helpers import issue_registry as ir

ir.async_create_issue(
    hass,
    DOMAIN,
    f"outdated_firmware_{entry.entry_id}",
    config_entry_id=entry.entry_id,
    is_fixable=False,
    severity=ir.IssueSeverity.WARNING,
    translation_key="outdated_firmware",
)
```

Set `config_entry_id` on issues that are about a single config entry. If your integration has code that deletes those issues when the entry is removed, you can drop it. Of the 150 core integrations that create repair issues, 11 clean up such issues themselves, and 63 leave them behind after the entry is gone. We'll update the core integrations in follow-up pull requests.

A few things to keep in mind:

- Issues are still identified by their domain and `issue_id`, so the `issue_id` must stay unique within your domain. When several config entries can raise the same kind of issue, put the entry ID in the `issue_id`.
- The config entry must exist. Passing an unknown ID raises `HomeAssistantError`.
- Creating an issue that already exists replaces its `config_entry_id`, also when you leave `config_entry_id` out.
- Issues that belong to a config entry that no longer exists are deleted when Home Assistant starts.

The reauthentication issue and the orphaned ignored config entry issue that Home Assistant creates itself now belong to their config entry too.

This is implemented in core [PR #186197](https://github.com/home-assistant/core/pull/186197), and lands in Home Assistant Core 2026.11. For the details, see [issues that belong to a config entry](/core/integrations/repairs/#issues-that-belong-to-a-config-entry).
