---
title: Labs preview features
description: Ship a finished feature that people can opt into through Home Assistant Labs, collect feedback, and graduate it to standard.
sidebarLabel: Labs
sidebarGroup: Changing behavior
order: 3
---

Home Assistant Labs lets people opt into a preview feature before it becomes standard. Users find it under **Settings** > **System** > **Labs**, where they can turn each feature on and off.

A Labs feature is finished and tested. What is still open is its design: how it looks, how it works, and what it should do, shaped by feedback from people using it at home. That makes Labs different from a beta, which tests the stability of a whole release.

Preview features are:

- **Off by default**: users turn them on themselves.
- **Production ready**: fully tested, not an experiment or half done.
- **Subject to change**: they can change, grow, or be removed based on feedback.
- **Reversible**: users can turn them off again at any time, without a restart.

## When to use Labs

Labs fits:

- Big changes or redesigns of the interface
- Significant changes in architecture
- Features where feedback from users will shape the final design

Labs does not fit:

- Settings that should simply be part of the integration. Use the options of the integration for those.
- Small changes that can go straight into a release
- Features that cannot be turned on and off while Home Assistant runs
- Features with known critical bugs, or that are not finished. Those stay in development.

## Add a preview feature

### 1. Declare it in the manifest

Add the feature to `preview_features` in the `manifest.json` of your integration:

```json
{
  "domain": "example",
  "name": "Example",
  "preview_features": {
    "new_dashboard": {
      "feedback_url": "https://community.home-assistant.io/t/...",
      "learn_more_url": "https://www.home-assistant.io/integrations/example",
      "report_issue_url": "https://github.com/home-assistant/core/issues/new"
    }
  }
}
```

- `feedback_url`: where people give feedback, like a forum topic or a GitHub discussion.
- `learn_more_url`: the documentation of the feature.
- `report_issue_url`: where people report bugs.

All three are optional, but add them: without them, the feedback you want has nowhere to go.

Then run hassfest, which adds the feature to `homeassistant/generated/labs.py`:

```shell
python3 -m script.hassfest
```

### 2. Add the text

Name and describe the feature in `strings.json`:

```json
{
  "preview_features": {
    "new_dashboard": {
      "name": "New dashboard",
      "description": "Replaces the Example panel with a new dashboard."
    }
  }
}
```

Say what the feature does, and mention if it may grow based on feedback. You can use Markdown.

If turning the feature on or off has consequences people should know about, like a migration that stays when they turn it off again, explain that in the description. You can also add `enable_confirmation` and `disable_confirmation`, which replace the standard confirmation messages. Use them to remind people of the consequences at the moment they act. They do not replace the description.

### 3. Check the flag

In Python, check whether the feature is on:

```python
from homeassistant.components.labs import async_is_preview_feature_enabled


async def async_setup_entry(
    hass: HomeAssistant, entry: ExampleConfigEntry
) -> None:
    """Set up Example from a config entry."""
    if async_is_preview_feature_enabled(hass, DOMAIN, "new_dashboard"):
        await async_setup_new_dashboard(hass)
    else:
        await async_setup_classic_panel(hass)
```

Keep the preview code and the standard code clearly apart, like this. Passing a `preview=True` flag deep into your code makes it hard to remove either one later.

In the frontend, `fetchLabFeatures` from `src/data/labs.ts` returns the features and whether they are enabled.

### 4. React when it is turned on or off

Preview features turn on and off while Home Assistant runs, without a restart. That is a hard requirement. Subscribe to changes with `async_subscribe_preview_feature`, which only calls your listener for your own feature:

```python
from homeassistant.components.labs import (
    EventLabsUpdatedData,
    async_subscribe_preview_feature,
)


async def async_setup_entry(
    hass: HomeAssistant, entry: ExampleConfigEntry
) -> None:
    """Set up Example from a config entry."""

    async def _async_preview_toggled(
        event_data: EventLabsUpdatedData,
    ) -> None:
        if event_data["enabled"]:
            await async_setup_new_dashboard(hass)
        else:
            await async_remove_new_dashboard(hass)

    entry.async_on_unload(
        async_subscribe_preview_feature(
            hass, DOMAIN, "new_dashboard", _async_preview_toggled
        )
    )
```

Things that work well at runtime: loading and unloading parts of the interface, creating and removing entities, starting and stopping background tasks, and registering event listeners. Things that do not: changing how Home Assistant starts, changing integrations that only load at startup, and anything that needs a change in `configuration.yaml`.

For more complex cases, you can listen to the `EVENT_LABS_UPDATED` event directly.

### 5. Prepare for feedback

Before you release the feature, create the place for feedback: a topic on the [community forum](https://community.home-assistant.io/), or a GitHub discussion or issue. Link it in `feedback_url`. Document the feature on the page of your integration on [home-assistant.io](https://www.home-assistant.io/): what it does, what feedback you are looking for, and how to turn it on.

A [My Home Assistant](/glossary/my-link/) link takes people straight to your feature in Labs, and highlights it. That works well in release notes, documentation, and forum posts:

```text
https://my.home-assistant.io/redirect/labs/?domain=example&preview_feature=new_dashboard
```

The [Kitchen Sink](https://github.com/home-assistant/core/tree/dev/homeassistant/components/kitchen_sink) integration has a complete example, with a preview feature that creates a repair issue.

## Test both states

Test your integration with the feature on and off, and test switching it. To start with the feature on, fill the storage of Labs before you set up the integration:

```python
async def test_new_dashboard(
    hass: HomeAssistant,
    hass_storage: dict[str, Any],
    mock_config_entry: MockConfigEntry,
) -> None:
    """Test the integration with the new dashboard enabled."""
    hass_storage["core.labs"] = {
        "version": 1,
        "data": {
            "preview_feature_status": [
                {"domain": DOMAIN, "preview_feature": "new_dashboard"},
            ]
        },
    }
    mock_config_entry.add_to_hass(hass)
    await hass.config_entries.async_setup(mock_config_entry.entry_id)
```

To switch it in a test, send a `labs/update` message with the `hass_ws_client` fixture:

```python
client = await hass_ws_client(hass)
await client.send_json_auto_id(
    {
        "type": "labs/update",
        "domain": DOMAIN,
        "preview_feature": "new_dashboard",
        "enabled": True,
    }
)
assert (await client.receive_json())["success"]
```

## While it is in preview

- Watch the feedback, and act on it. Fix bugs quickly.
- Changes that break things are allowed in a preview, but avoid them when you can. If you must, migrate the data and settings of users automatically. People rely on this in their homes, preview or not.
- Keep the description accurate as the feature changes.
- Do not leave a feature in Labs forever. Plan from the start how it graduates, or how it goes away.

## Graduate or remove

When the feature is ready to become standard:

1. Remove it from `preview_features` in `manifest.json`.
2. Remove the checks of the flag, so the feature is always on.
3. Run `python3 -m script.hassfest`.
4. Update the documentation, and announce it in the release notes.

When it did not work out:

1. Remove it from `preview_features` in `manifest.json`.
2. Remove the code and the text in `strings.json`.
3. Run `python3 -m script.hassfest`.
4. Announce the removal in the release notes, and explain why.
